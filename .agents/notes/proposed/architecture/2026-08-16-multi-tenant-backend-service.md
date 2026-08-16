# Agent Note: Multi-tenant backend service — trusted-header identity, Postgres persistence, and RBAC authorization

Status: proposed

English | [中文](2026-08-16-multi-tenant-backend-service.zh.md)

## Problem

DeepSeek Harness today is a single-user local application. A `dsh web` process serves one anonymous user; sessions, settings, search indexes, and the credential environment all live in a local Harness home, and there is no authentication, no per-user isolation, and no authorization model beyond the browser-trust reachability fence.

A deployment wants to run it as a backend service behind a unified IAM gateway and a browser frontend, initially exposing only chat. Three gaps block that:

- **Identity.** No request carries a caller identity, and the runtime has no per-request principal. The [browser trust boundary](../../implemented/architecture/2026-07-28-api-browser-trust-boundary.md) is explicitly reachability policy, not authentication.
- **Durable shared storage.** Session persistence, settings, and the search index are local per-instance artifacts (JSONL files, SQLite, a settings file). They must survive restarts and be readable by the deployment as one logical store.
- **Authorization.** Once several users exist, who may use which agent preset, which tools and skills under it, and which resources a tool may touch all need role-based control. The codebase states plainly that agent scope and `ctx.tools.restrict` are visibility composition, **not** an authority boundary, so RBAC must be a separate enforcement layer.

## Proposal

Turn the application into a multi-tenant backend in three layered concerns, delivered in the phases below:

1. **Identity** — a trusted `x-username` request header (injected and overwritten by the IAM gateway) names the caller. It is normalized into a branded `UserId`, written into `SessionHeader.userId` at session creation, and read as the durable authority for every later data-isolation and authorization decision.
2. **Data isolation (who)** — every session-scoped read and mutation is filtered by `userId`: list, resume, fork, export, search, history. This is separate from authorization.
3. **Authorization (what)** — a new `ctx.rbac` seam evaluates role-to-permission grants against the existing platform RBAC model, enforced at four points: the tool execution pipeline, agent-preset admission, the skill catalog and loader, and per-tool resource checks.

Three principles bind every implementing phase:

- **Identity is established only at the boundary.** `x-username` is parsed, validated, and normalized once at the HTTP and WebSocket entry; everything downstream reads the durable `userId` from `SessionHeader`, never from request-scoped storage, because a turn, its stream, background jobs, and subagents outlive the HTTP request.
- **Isolation and authorization are two layers.** `userId` scoping answers "whose data may this caller see"; RBAC answers "which agents, tools, skills, and resources may this caller use". They are enforced separately and must not be conflated.
- **RBAC is an authority boundary.** `ctx.tools.restrict` stays a model-visibility optimization (hide unauthorized schemas, save tokens, avoid prompting the model to call them). The allow/deny decision lives on the `tools/pre-execute` → guard execution pipeline, which native calls and Code Mode sub-dispatch both traverse.

Scope decisions already agreed with the deployment, recorded here so later phases do not re-open them:

- Persist to **Postgres** directly (no SQLite transition step).
- **Settings stay global** — one shared document; only sessions and session-derived data are per-user.
- A **missing or malformed `x-username` is rejected** with 401/403, never mapped to a shared anonymous user.
- **Roles resolve server-side** from the RBAC store by `userId` (the IAM gateway does not forward role headers).
- **Resource-level authorization is tool-owned**: a tool consults `ctx.rbac` inside its own `execute`, rather than a generic guard guessing resource names from arguments.
- One **shared API key** arrives through the environment; there is no per-user credential surface.
- **Containerization is out of scope** for now; the in-memory live session model therefore stays single-instance.
- The API exposes **chat only** for now; bash/fs/tools stay unreachable to callers, which defers the execution-sandbox isolation question until tools are exposed.

## Identity and request principal

A new `identity/` package defines `UserId` (a `Branded` value, per the cross-boundary id convention) with construction that normalizes `x-username`: trim, validate against a safe charset such as `[A-Za-z0-9._-]+`, and reject an empty or oversized value. Normalization before storage is what makes the value a stable partition key.

A request-scoped `ctx.principal` service, backed by `AsyncLocalStorage`, exposes the current `UserId` to same-request callers. It exists only to carry identity from the HTTP/WebSocket boundary to the moment a session is created or resumed; it is not the runtime authority for anything that outlives the request.

The host half of `@deepseek-ai/dsh-client-connection` — the single `/api` Fetch bridge (`http-bridge.ts`) and the WebSocket upgrade path (`websocket-downlink.ts`) — reads and validates `x-username` before RPC dispatch and binds one principal for the whole request or connection. A WebSocket binds its principal at upgrade; downlink frames carry no header, so the binding is fixed for the connection's lifetime. Missing or malformed input answers 401/403 before any dispatch.

Because the backend trusts the header, the deployment owns the security precondition: the backend must be reachable only through the IAM gateway, and the gateway must overwrite (not merely forward) `x-username` so a direct caller cannot forge it.

## Postgres persistence

Three provider additions reuse existing seams rather than introducing new contracts.

`@deepseek-ai/dsh-session-persistence-postgres` implements `SessionPersistence` (`ctx.sessionPersistence`) with a shared `SCHEMA_VERSION` migration, mirroring the SQLite backend's row model: a `sessions` table holding the `SessionHeader` fields plus `user_id`, and a `session_events` table with one row per `SessionEvent` (`session_id, seq, type, time, data jsonb, source_event_seqs, surface_op`). It must pass the shared `runPersistenceContract` suite, including contiguous-seq append and interrupted-turn recovery, so its semantics match the JSONL and SQLite backends.

`SessionHeader` gains `userId: UserId`, threaded through `CreateSessionOptions.meta`. This is a durable-format change: per the pre-release stance, `SESSION_FORMAT_VERSION` and the Postgres `SCHEMA_VERSION` move monotonically without a migration path.

`@deepseek-ai/dsh-settings-postgres` implements `SettingsProvider` (`ctx.settings`) over one global document, replacing `settings-file` in the service composition. Composition config stays in `cordis.yml`; only the user-editable subset lives in this store, as the settings seam already prescribes.

The search/lineage read model moves to a Postgres-backed `ctx.sessionQuery` provider in a later phase; the SQLite FTS5 provider remains acceptable for a single-instance MVP.

## RBAC authorization

A new `rbac/` group follows the capability-seam convention (Service Definition, Provider, Consumer).

`@deepseek-ai/dsh-rbac` defines `ctx.rbac`:

```ts type-equiv
interface RbacService {
  /** Grant verdict for one action and optional resource; `ask` may defer to `ctx.approval` or degrade to deny. */
  authorize(subject: UserId, action: string, resource?: ResourceRef): Promise<'allow' | 'deny' | 'ask'>
  /** Batch the id set granted to a subject, for list filtering and visibility masks. */
  listGranted(subject: UserId, kind: 'preset' | 'tool' | 'skill'): Promise<string[]>
}
```

`@deepseek-ai/dsh-rbac-postgres` evaluates `authorize` against the platform's existing user→role→permission tables, resolved by `userId`, with a short-TTL cache. `authorize` is the authoritative call and must never serve a stale grant; `listGranted` is a precomputed convenience whose staleness only affects visibility, not enforcement.

Permissions are string actions from coarse to fine: `agent:<presetId>`, `tool:<name>`, `skill:<id>`, and `db:table:<table>:<op>`. Encoding grants as action strings keeps adding tools, skills, and resources a data change, not a code change.

Four enforcement points consume the seam:

1. A `tools/pre-execute` listener checks `authorize(userId, 'tool:' + name)`. This is the authoritative tool gate and covers native calls and Code Mode sub-dispatch, both of which traverse pre-execute and guards.
2. Agent-preset admission checks `authorize(userId, 'agent:' + presetId)` before `agentPresets.mount`, and `api/remotes` filters `agentPreset.list` by `listGranted(userId, 'preset')`.
3. The skill catalog filters by `listGranted(userId, 'skill')`, and the `dsh-tool-skill` loader refuses a skill the caller lacks before loading its content.
4. A tool whose operation touches resources resolves them in its own `execute` and calls `authorize(userId, 'db:table:' + table, { op })`; denial returns a model-visible error and is logged for audit.

Visibility is a separate, non-authoritative step: at agent setup, `listGranted(userId, 'tool')` drives a `ctx.tools.restrict` mask so the model does not see schemas it cannot use. Omitting or staleness of this mask must never weaken the guard.

The runtime `userId` for all four points comes from the durable `SessionHeader`, not `ctx.principal`, because enforcement happens during a turn, stream, job, or subagent run that outlives the HTTP request.

## Public chat API

The existing `/api` RPC face already streams a session's prompt and live frames (`session.prompt` plus the mux downlink). The backend reuses that face, adding the principal gate and `userId` scoping, rather than minting a parallel REST surface. A protocol version field is introduced on `host.describe` only when a client is released independently of the host; until then client and host ship together as today.

## Phased delivery

| Phase | Content | Primary packages |
|---|---|---|
| 0 | This proposal, reviewed before implementation | `.agents/notes/` |
| 1 | `UserId`, `ctx.principal`, and the `x-username` boundary gate in Connection (HTTP + WebSocket) | `identity/*`, `client/connection` |
| 2 | `SessionHeader.userId` through every backend and `session-persistence-postgres` | `core/session`, `session/*` |
| 3 | `settings-postgres` global document | `settings/*` |
| 4 | `ctx.rbac` seam, `rbac-postgres`, and the four enforcement points | `rbac/*`, `core/tools`, `preset/*`, `skill/*` |
| 5 | Authorization convergence in `api/remotes`: `userId` scoping plus RBAC admission on list/resume/fork/export/search, and `userId` in audit logs | `api/remotes`, `host/apiproxy` |
| 6 | Public chat API convergence and version negotiation | `host/*`, `api/*` |
| 7 | Postgres-backed session search/lineage provider (deferred past MVP) | `session-query/*` |

Each phase adds unit coverage to the per-file gate and its package's `./invariant` companion, per the repository conventions.

## Alternatives considered

**SQLite plus a persistent volume, then migrate later.** Rejected by the deployment: they chose Postgres directly. The persistence seam would have made a later swap cheap, but a shared database and its FTS story are needed now, not as a second migration.

**IAM forwards role headers; the backend only reads them.** Rejected: the deployment chose to resolve roles server-side from the RBAC store by `userId`, keeping permission data in the platform database and the gateway coupled only to identity.

**Generic guard with a resource annotation on `ToolDefinition`.** Rejected for the MVP: the resource name and op are tool-owned semantics; a generic `argPath` extractor is fragile against argument structure. Tool self-query of `ctx.rbac` is explicit, auditable, and single-owner. The annotation remains a possible later optimization, not the enforcement model.

**Use `ctx.tools.restrict` as the security boundary.** Rejected: the tools contract documents `restrict` as live visibility composition, not an authority boundary. Hiding a schema does not prevent a direct dispatch, a nested call, or a Code Mode sub-call. The authoritative gate must sit on pre-execute/guard.

**OIDC or JWT handling inside the backend.** Rejected: the IAM gateway already authenticates and asserts identity over a trusted internal network. The backend validates and normalizes the asserted header instead of re-implementing authentication.

**A new versioned REST/SSE chat surface (`/v1/chat`).** Rejected for now: the existing `/api` RPC face already streams prompts and live frames, so reusing it with the added principal gate and scoping minimizes frontend churn. A clean REST surface remains available if a contract independent of the web host is later required.

## Acceptance criteria

- A request without a valid `x-username` is rejected with 401/403 before dispatch, over both HTTP and WebSocket upgrade.
- `SessionHeader` carries `userId`, and list, resume, fork, export, search, and history are all filtered to the calling user's sessions.
- The Postgres persistence backend passes the shared `SessionPersistence` contract suite, and the settings provider stores one global document in Postgres.
- A role without `tool:<name>` cannot execute that tool — through a native call and through a Code Mode sub-call — even when a stale visibility mask still shows it.
- A role without `agent:<presetId>` cannot mount or select that preset, and the preset list omits it.
- A role without `skill:<id>` cannot load that skill, and the skill list omits it.
- A tool denies a resource action the role lacks (`db:table:<table>:<op>`), returning a model-visible error and an audit record carrying `userId`.
- Audit logs carry the caller `userId` alongside session and tool facts.
- The shared API key resolves from the environment; no per-user credential surface exists.

## Risks

The trusted-header model is only as strong as the network precondition: if the backend is directly reachable, any caller can forge `x-username`. The deployment must front it with the IAM gateway, which overwrites the header, and bind the backend to an internal interface.

Identity that lives only in request scope would break the first time a turn continues after the request ends. The durable `SessionHeader.userId` is the runtime authority; `ctx.principal` must never be read by enforcement, only by the boundary that creates or resumes a session.

The `SessionHeader` change is a durable-format change with no migration path (pre-release stance). The `SESSION_FORMAT_VERSION` and Postgres `SCHEMA_VERSION` bumps mean existing JSONL and SQLite stores cannot be opened by the new build; this is acceptable only because no external deployment exists yet.

RBAC must not be bypassable through the nested paths: Code Mode sub-dispatch, nested tool calls, and subagent tool use all funnel through the same pre-execute/guard pipeline, so the single gate covers them; any new dispatch path added later must route through it.

A stale `listGranted` cache may hide or reveal schemas wrongly, but must never change an enforcement verdict — `authorize` is the authority and must read the current grant. Visibility and enforcement divergence is a UX or token-cost issue, not a security one.

The Postgres provider must reproduce JSONL/SQLite durability semantics exactly (contiguous seq, interrupted-turn recovery, revision observation) or resume and search will silently diverge across backends; the shared contract suite is the mitigation, and cold-path behaviors need explicit tests.

Chat-only scope deliberately defers the multi-tenant execution-sandbox boundary. Exposing bash, filesystem, or subprocess tools to callers later reopens that question and will need the remote-sandbox providers (for example the existing [E2B execution world](../../implemented/architecture/2026-07-28-portable-execution-world-consumers.md)) before those tools reach an untrusted tenant.

This proposal does not supersede any active Agent Note. It builds on and cross-links: the [browser trust boundary](../../implemented/architecture/2026-07-28-api-browser-trust-boundary.md) (the reachability fence the principal gate sits above), [session persistence](../../implemented/architecture/2026-06-14-session-persistence.md) (the seam the Postgres backend extends), [per-session agent presets](../../implemented/architecture/2026-08-03-per-session-agent-presets.md) (the composition RBAC gates), the [API Proxy unary migration](../../proposed/architecture/2026-08-10-unary-apiproxy-remote-migration.md) (the API surface this work converges), and the [sandbox decision](../../implemented/feature/2026-07-06-sandbox.md) (the orthogonal execution-confinement axis).
