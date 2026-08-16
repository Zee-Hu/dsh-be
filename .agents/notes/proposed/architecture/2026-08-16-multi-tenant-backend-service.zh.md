# Agent Note: 多租户后端服务 —— 可信头身份、Postgres 持久化与 RBAC 授权

Status: proposed

[English](2026-08-16-multi-tenant-backend-service.md) | 中文

## 问题

DeepSeek Harness 目前是一个单用户本地应用。一个 `dsh web` 进程只服务一个匿名用户；会话、设置、搜索索引和凭据环境都存在于本地 Harness home 中。除了浏览器信任的可达性栅栏之外，它既没有认证，也没有按用户隔离，更没有任何授权模型。

某部署希望把它作为一个后端服务运行在统一 IAM 网关和浏览器前端之后，初期只暴露 chat。有三个缺口挡住了这件事：

- **身份。** 没有任何请求携带调用方身份，运行时也没有按请求的主体概念。[浏览器信任边界](../../implemented/architecture/2026-07-28-api-browser-trust-boundary.md)明确只是可达性策略，不是认证。
- **持久化的共享存储。** 会话持久化、设置和搜索索引都是本地单实例产物（JSONL 文件、SQLite、设置文件）。它们必须能在重启后存活，并被部署作为一个逻辑存储来读取。
- **授权。** 一旦存在多个用户，谁能使用哪个 agent 预设、其下哪些工具和技能、以及某个工具能接触哪些资源，都需要基于角色的控制。代码库已明确说明 agent scope 和 `ctx.tools.restrict` 是可见性组合，**不是**权威边界，因此 RBAC 必须是一个独立的强制层。

## 提案

把应用改造成一个多租户后端，分成三个层次的关注点，按下面的阶段交付：

1. **身份** —— 一个可信的 `x-username` 请求头（由 IAM 网关注入并覆盖）标识调用方。它被规范化为带品牌的 `UserId`，在会话创建时写入 `SessionHeader.userId`，并作为之后每个数据隔离与授权决策的持久化权威来源被读取。
2. **数据隔离（who）** —— 每个会话级读取和变更都按 `userId` 过滤：list、resume、fork、export、search、history。这与授权是分开的。
3. **授权（what）** —— 一个新的 `ctx.rbac` seam 依据现有平台 RBAC 模型评估角色到权限的授予，在四个点位强制：工具执行管线、agent 预设准入、技能目录与加载器、以及每个工具的资源检查。

三条原则约束每一个实现阶段：

- **身份只在边界处确立。** `x-username` 在 HTTP 和 WebSocket 入口解析、校验、规范化一次；之后的一切都从 `SessionHeader` 读取持久化的 `userId`，而不是从请求级存储读取，因为一个 turn、它的流、后台 job 和 subagent 都存活于 HTTP 请求之外。
- **隔离与授权是两层。** `userId` scoping 回答“这个调用方能看到谁的数据”；RBAC 回答“这个调用方能用哪些 agent、工具、技能和资源”。它们分开强制，绝不能混为一谈。
- **RBAC 是权威边界。** `ctx.tools.restrict` 仍是模型可见性优化（隐藏未授权 schema、省 token、避免诱导模型调用它们）。allow/deny 决策落在 `tools/pre-execute` → guard 执行管线上，native 调用和 Code Mode 子调用都会经过它。

以下范围决策已与部署方达成一致，记录于此，以免后续阶段重新讨论：

- 直接持久化到 **Postgres**（没有 SQLite 过渡步骤）。
- **设置保持全局** —— 一份共享文档；只有会话及会话派生数据是按用户的。
- **缺失或格式非法的 `x-username` 一律 401/403 拒绝**，绝不映射到某个共享匿名用户。
- **角色在服务端解析**：由 RBAC 库按 `userId` 查得（IAM 网关不转发角色头）。
- **资源级授权归工具所有**：工具在自己的 `execute` 内咨询 `ctx.rbac`，而不是由通用 guard 从参数里猜资源名。
- **一把共享 API key** 通过环境注入；没有按用户的凭据面。
- **暂不容器化**；内存中的 live session 模型因此保持单实例。
- 对外 API 目前**只暴露 chat**；bash/fs/tools 对调用方不可达，这把执行沙箱隔离问题推迟到工具开放之时。

## 身份与请求主体

新增一个 `identity/` 包，定义 `UserId`（按跨边界 id 惯例使用 `Branded`），其构造负责规范化 `x-username`：trim、校验安全字符集（如 `[A-Za-z0-9._-]+`）、拒绝空值或超长值。存储前规范化使该值成为稳定的分区键。

一个请求级 `ctx.principal` 服务，由 `AsyncLocalStorage` 背书，向同一请求内的调用方暴露当前 `UserId`。它只用于把身份从 HTTP/WebSocket 边界带到会话创建或恢复的那一刻；它不是任何存活于请求之外的事物的运行时权威来源。

`@deepseek-ai/dsh-client-connection` 的 host 半边 —— 唯一的 `/api` Fetch 桥（`http-bridge.ts`）和 WebSocket 升级路径（`websocket-downlink.ts`）—— 在 RPC 分发之前读取并校验 `x-username`，并为整个请求或连接绑定一个主体。WebSocket 在升级时绑定其主体；下行帧不带 header，因此该绑定在连接生命周期内固定。缺失或格式非法的输入在任何分发之前就回答 401/403。

由于后端信任该 header，安全前提由部署方负责：后端只能通过 IAM 网关可达，且网关必须覆盖（而非仅仅转发）`x-username`，使直接调用方无法伪造它。

## Postgres 持久化

三个 provider 增量复用现有 seam，而不是引入新契约。

`@deepseek-ai/dsh-session-persistence-postgres` 实现 `SessionPersistence`（`ctx.sessionPersistence`），带共享的 `SCHEMA_VERSION` 迁移，镜像 SQLite 后端的行模型：一张 `sessions` 表存 `SessionHeader` 字段外加 `user_id`，一张 `session_events` 表每行一条 `SessionEvent`（`session_id, seq, type, time, data jsonb, source_event_seqs, surface_op`）。它必须通过共享的 `runPersistenceContract` 套件，包括连续 seq 追加和中断 turn 恢复，使语义与 JSONL、SQLite 后端一致。

`SessionHeader` 增加 `userId: UserId`，经 `CreateSessionOptions.meta` 透传。这是一次持久化格式变更：按 pre-release 立场，`SESSION_FORMAT_VERSION` 和 Postgres 的 `SCHEMA_VERSION` 单调递增，不提供迁移路径。

`@deepseek-ai/dsh-settings-postgres` 实现 `SettingsProvider`（`ctx.settings`），基于一份全局文档，在服务组合中替换 `settings-file`。组合配置仍留在 `cordis.yml`；只有用户可编辑子集存进该存储，这正是 settings seam 早已规定的。

搜索/血缘读模型在较后阶段迁移到 Postgres 支撑的 `ctx.sessionQuery` provider；单实例 MVP 阶段 SQLite FTS5 provider 仍可接受。

## RBAC 授权

新增一个 `rbac/` 组，遵循能力缝惯例（Service Definition、Provider、Consumer）。

`@deepseek-ai/dsh-rbac` 定义 `ctx.rbac`：

```ts type-equiv
interface RbacService {
  /** 对一个动作及可选资源的授予判定；`ask` 可交给 `ctx.approval` 或降级为 deny。 */
  authorize(subject: UserId, action: string, resource?: ResourceRef): Promise<'allow' | 'deny' | 'ask'>
  /** 批量拉取授予某主体的 id 集合，用于列表过滤与可见性掩码。 */
  listGranted(subject: UserId, kind: 'preset' | 'tool' | 'skill'): Promise<string[]>
}
```

`@deepseek-ai/dsh-rbac-postgres` 依据平台现有的 user→role→permission 表评估 `authorize`，按 `userId` 解析，带短 TTL 缓存。`authorize` 是权威调用，绝不能服务过期授予；`listGranted` 是预计算便利，其过期只影响可见性，不影响强制。

权限是从粗到细的字符串动作：`agent:<presetId>`、`tool:<name>`、`skill:<id>`、以及 `db:table:<table>:<op>`。把授予编码为动作字符串，使得新增工具、技能、资源只是数据变更而非代码变更。

四个强制点位消费该 seam：

1. 一个 `tools/pre-execute` 监听器检查 `authorize(userId, 'tool:' + name)`。这是权威工具门，覆盖 native 调用和 Code Mode 子调用（两者都会经过 pre-execute 和 guard）。
2. agent 预设准入在 `agentPresets.mount` 之前检查 `authorize(userId, 'agent:' + presetId)`，`api/remotes` 用 `listGranted(userId, 'preset')` 过滤 `agentPreset.list`。
3. 技能目录按 `listGranted(userId, 'skill')` 过滤，`dsh-tool-skill` 加载器在加载内容之前拒绝调用方缺失的技能。
4. 一个操作会接触资源的工具，在自己的 `execute` 内解析这些资源并调用 `authorize(userId, 'db:table:' + table, { op })`；拒绝时返回模型可见错误并记入审计。

可见性是独立的、非权威的步骤：在 agent 装配时，`listGranted(userId, 'tool')` 驱动 `ctx.tools.restrict` 掩码，使模型看不到其无法使用的 schema。该掩码的缺失或过期绝不能削弱 guard。

四个点位的运行时 `userId` 都来自持久化的 `SessionHeader`，而非 `ctx.principal`，因为强制发生在存活于 HTTP 请求之外的 turn、流、job 或 subagent 运行期间。

## 对外 chat API

现有 `/api` RPC 面已经流式传输会话的 prompt 和 live 帧（`session.prompt` 加 mux 下行）。后端复用它，加上主体门和 `userId` scoping，而不是另造一个并行的 REST 面。协议版本字段只有在客户端独立于 host 发布时才引入到 `host.describe`；在那之前客户端与 host 如今天一样同船发布。

## 分阶段交付

| 阶段 | 内容 | 主要涉及包 |
|---|---|---|
| 0 | 本提案，实现前评审 | `.agents/notes/` |
| 1 | `UserId`、`ctx.principal`，以及 Connection 中的 `x-username` 边界门（HTTP + WebSocket） | `identity/*`、`client/connection` |
| 2 | 贯穿所有后端的 `SessionHeader.userId`，以及 `session-persistence-postgres` | `core/session`、`session/*` |
| 3 | `settings-postgres` 全局文档 | `settings/*` |
| 4 | `ctx.rbac` seam、`rbac-postgres`，以及四个强制点位 | `rbac/*`、`core/tools`、`preset/*`、`skill/*` |
| 5 | `api/remotes` 中的授权收敛：list/resume/fork/export/search 上的 `userId` scoping 加 RBAC 准入，审计日志带 `userId` | `api/remotes`、`host/apiproxy` |
| 6 | 对外 chat API 收敛与版本协商 | `host/*`、`api/*` |
| 7 | Postgres 支撑的会话搜索/血缘 provider（MVP 之后再做） | `session-query/*` |

每个阶段按仓库惯例补齐 per-file 覆盖率门和其包的 `./invariant` companion。

## 考虑过的替代方案

**SQLite 加持久卷，之后再迁移。** 被部署方否决：他们选择直接用 Postgres。持久化 seam 本可以让后续替换成本很低，但现在就需要共享数据库及其全文检索能力，而不是再做一次迁移。

**IAM 转发角色头，后端只读头。** 被否决：部署方选择按 `userId` 从 RBAC 库在服务端解析角色，让权限数据留在平台数据库，网关只与身份耦合。

**在 `ToolDefinition` 上加资源注解的通用 guard。** MVP 阶段被否决：资源名和 op 是工具拥有的语义；通用 `argPath` 提取器对参数结构脆弱。工具自查询 `ctx.rbac` 显式、可审计、单一拥有者。注解仍是日后可能的优化，而非强制模型。

**把 `ctx.tools.restrict` 当作安全边界。** 被否决：tools 契约把 `restrict` 记录为 live 可见性组合，而非权威边界。隐藏 schema 并不能阻止直接分发、嵌套调用或 Code Mode 子调用。权威门必须落在 pre-execute/guard 上。

**在后端内部处理 OIDC 或 JWT。** 被否决：IAM 网关已在可信内网上完成认证并断言身份。后端校验并规范化被断言的 header，而不是重做认证。

**新的版本化 REST/SSE chat 面（`/v1/chat`）。** 现阶段被否决：现有 `/api` RPC 面已经流式传输 prompt 和 live 帧，复用它并加上主体门和 scoping 能把前端改动降到最小。若日后需要一个独立于 web host 的契约，干净的 REST 面仍可作为选项。

## 验收标准

- 没有合法 `x-username` 的请求在分发前被 401/403 拒绝，HTTP 和 WebSocket 升级皆如此。
- `SessionHeader` 携带 `userId`，list、resume、fork、export、search、history 全部过滤到调用用户自己的会话。
- Postgres 持久化后端通过共享的 `SessionPersistence` 契约套件，设置 provider 把一份全局文档存进 Postgres。
- 缺少 `tool:<name>` 的角色无法执行该工具 —— 无论 native 调用还是 Code Mode 子调用 —— 即使过期的可见性掩码仍显示它。
- 缺少 `agent:<presetId>` 的角色无法挂载或选择该预设，且预设列表省略它。
- 缺少 `skill:<id>` 的角色无法加载该技能，且技能列表省略它。
- 工具拒绝角色缺失的资源动作（`db:table:<table>:<op>`），返回模型可见错误并产生带 `userId` 的审计记录。
- 审计日志在会话与工具事实之外携带调用方 `userId`。
- 共享 API key 从环境解析；不存在按用户的凭据面。

## 风险

可信头模型只和网络前提一样强：若后端可被直接触达，任何调用方都能伪造 `x-username`。部署必须用 IAM 网关前置它（网关覆盖该 header），并把后端绑定到内网接口。

只存在于请求作用域的身份，会在 turn 于请求结束后继续时立刻失效。持久化的 `SessionHeader.userId` 是运行时权威来源；强制点绝不能读 `ctx.principal`，只能由创建或恢复会话的边界读取。

`SessionHeader` 变更是一次无迁移路径的持久化格式变更（pre-release 立场）。`SESSION_FORMAT_VERSION` 和 Postgres `SCHEMA_VERSION` 的递增意味着新构建无法打开现有的 JSONL 和 SQLite 存储；这只有在尚不存在外部部署时才是可接受的。

RBAC 必须无法经由嵌套路径绕过：Code Mode 子分发、嵌套工具调用和 subagent 工具使用都汇入同一条 pre-execute/guard 管线，因此单一的门即可覆盖它们；日后新增的任何分发路径都必须经过它。

过期的 `listGranted` 缓存可能错误地隐藏或暴露 schema，但绝不能改变强制判定 —— `authorize` 是权威，必须读取当前授予。可见性与强制的分歧是 UX 或 token 成本问题，而非安全问题。

Postgres provider 必须精确复现 JSONL/SQLite 的持久化语义（连续 seq、中断 turn 恢复、revision 观察），否则 resume 和 search 会在各后端之间静默分叉；共享契约套件是缓解手段，冷路径行为需要显式测试。

chat-only 范围刻意推迟了多租户执行沙箱边界。日后向调用方暴露 bash、filesystem 或 subprocess 工具会重新打开这个问题，届时需要远程沙箱 provider（例如现有的 [E2B 执行世界](../../implemented/architecture/2026-07-28-portable-execution-world-consumers.md)）之后，这些工具才能触达不可信租户。

本提案不取代任何现存 Agent Note。它建立在以下 note 之上并与之交叉引用：[浏览器信任边界](../../implemented/architecture/2026-07-28-api-browser-trust-boundary.md)（主体门叠加于其上的可达性栅栏）、[会话持久化](../../implemented/architecture/2026-06-14-session-persistence.md)（Postgres 后端所扩展的 seam）、[按会话的 agent 预设](../../implemented/architecture/2026-08-03-per-session-agent-presets.md)（RBAC 所 gate 的组合）、[API Proxy 一元迁移](../../proposed/architecture/2026-08-10-unary-apiproxy-remote-migration.md)（本工作所收敛的 API 面）、以及 [沙箱决策](../../implemented/feature/2026-07-06-sandbox.md)（正交的执行约束轴）。
