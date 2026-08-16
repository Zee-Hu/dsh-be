# AGENTS.md

DeepSeek Harness is a plugin-based agent harness on vendored Cordis: **everything is a plugin**. This is a personal fork; the upstream documentation, translation, CI, and quality-gate machinery has been removed. Only these engineering-correctness rules remain in force.

## Repository layout

```
vendor/      Vendored Cordis source
packages/    @deepseek-ai/dsh-<pkg> workspaces at packages/<group>/<pkg>/
apps/        Product assemblies: apps/cli owns the `dsh` bin, apps/web the GUI
examples/    Runnable cordis.yml leaves over packages/examples bundles
native/      @deepseek-ai/node-addon-landlock-run source of record
python/      Python SDK and bundled runtime
docs/        architecture, cordis-primer, glossary, defensive patterns, cookbook
```

## Commands

```sh
pnpm install            # pnpm workspaces, node ^22.19 || >=24
pnpm run clean          # remove build outputs and safe residue from deleted packages
pnpm run test           # vitest unit tests
pnpm run typecheck
pnpm run lint
pnpm run build          # tsc emits lib/types, tsdown bundles runtime
pnpm dsh --profile headless "task"   # run one task from source (needs DEEPSEEK_API_KEY)
pnpm run demo:cordis    # the agent modifies its own runtime (needs key)
pnpm run demo:acp       # ACP automation server (needs DEEPSEEK_API_KEY)
```

## Conventions (engineering correctness)

- Every npm package is `@deepseek-ai/dsh-<name>`; vendored packages are rescoped. ESM everywhere (`"type": "module"`). Use package names across packages and `.ts` in local relative imports.
- **Registrations are effects**: every contribution goes through `ctx.effect()` / `ctx.on()`; a registry's `register()` returns the disposer.
- **Runtime invariants assert owned relationships.** Check authoritative event streams or mutable data, not service or method presence, plugin metadata or effects, or fixed pure examples.
- **Typed events use declaration merging** and merge-extensible maps. A `SessionEventMap` member is required-on-read by default.
- **Switch on discriminant tags.** Closed unions end in `assertNever`; merge-extensible unions fall through a documented default.
- **Waterfall listeners MUST call `next()`** to delegate; returning without it short-circuits the chain.
- **Model-visible ⟺ logged**: anything that reaches a model request must be reconstructable from the session log; a new model-visible input requires a session event.
- **Plugins, not loop changes**: new behavior goes on documented extension points; changing `agent-loop` requires updating `docs/architecture.md`.
- **A capability seam comprises Service Definition / Service Provider / Consumer roles.** It is complete, never one role.
- **Plugin exports:** service packages default-export their service class; function plugins named-export `name` / `inject` / `Config` / `apply` and have no default export. Optional services use `ctx.get(name)`; reserve `ctx.<name>` for declared injections.
- **Explicit > implicit at package boundaries**: defaulting is an explicit `resolve(request): Spec` step in the owning implementation, never a hidden `?? default` inside `run()`.
- **No hardcoded tunables in plugins**: deployment-varying choices are validated `Config` fields changeable from cordis.yml. Protocol constants and security invariants stay fixed.
- **Misconfiguration fails loud** at load when self-contained, otherwise at the earliest resolvable point; never silently skip a missing referent.
- **Opaque cross-boundary ids are branded** (`Branded<B>` from `dsh-brand`), never bare `string`.
- **Trust TypeScript at typed same-process boundaries.** Add runtime validation only at parser/config, queued, model/tool JSON, durable/file, worker, process, and wire boundaries.
- **Model-facing contracts from the model's perspective.** Prompts, tool schemas, results, and diagnostics contain only task-relevant concepts.
- **Publish state only at its commit point.** Emit notifications and derive caches/views only after the operation succeeds, from one authoritative source.
- **Apply bounds to the complete result.** Enforce byte/token/item/time limits where the complete value is known.
- **Keep compiler faces explicit.** Host and Client type-check through `tsconfig.host.json` / `tsconfig.client.json`; never flatten them into one program.
- Files end with exactly one trailing newline.

## Secrets

Real-API tests and demos read `DEEPSEEK_API_KEY`, optional `DEEPSEEK_BASE_URL`, and root `.env`. cordis.yml allows `!!js` (never `!js`) under plugin `config`. Never commit credentials.

## References

- `docs/architecture.md` — read before changing `packages/`
- `docs/cordis-primer.md` — Cordis semantics
- `docs/defensive-patterns.md` — read before lifecycle, concurrency, subprocess, or teardown work
- `docs/glossary.md` — terminology
