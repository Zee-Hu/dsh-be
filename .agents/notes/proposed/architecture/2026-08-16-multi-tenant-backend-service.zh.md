# Agent Note: 多租户后端服务 —— 可信头身份与 Postgres 持久化

Status: proposed

[English](2026-08-16-multi-tenant-backend-service.md) | 中文

## 问题

DeepSeek Harness 目前是一个单用户本地应用。一个 `dsh web` 进程只服务一个匿名用户；会话、设置、搜索索引和凭据环境都存在于本地 Harness home 中。除了浏览器信任的可达性栅栏之外，它既没有认证，也没有按用户隔离，更没有任何授权模型。

某部署希望把它作为一个后端服务运行在统一 IAM 网关和浏览器前端之后，初期只暴露 chat。有三个缺口挡住了这件事：

- **身份。** 没有任何请求携带调用方身份，运行时也没有按请求的主体概念。[浏览器信任边界](../../implemented/architecture/2026-07-28-api-browser-trust-boundary.md)明确只是可达性策略，不是认证。
- **持久化的共享存储。** 会话持久化、设置和搜索索引都是本地单实例产物（JSONL 文件、SQLite、设置文件）。它们必须能在重启后存活，并被部署作为一个逻辑存储来读取。
- **按用户的数据隔离。** 一旦存在多个用户，会话级操作 —— list、resume、fork、export、search、history —— 就必须限制到会话的拥有者，否则一个用户会看到并操作另一个用户的对话。

## 提案

把应用改造成一个多租户后端，分成两个层次的关注点，按下面的阶段交付：

1. **身份** —— 一个可信的 `x-username` 请求头（由 IAM 网关注入并覆盖）标识调用方。它被规范化为带品牌的 `UserId`，在会话创建时写入 `SessionHeader.userId`，并作为之后每个数据隔离决策的持久化权威来源被读取。
2. **数据隔离（who）** —— 每个会话级读取和变更都按 `userId` 过滤：list、resume、fork、export、search、history。至于调用方能使用哪些 agent、工具、技能和资源的授权，是另一个关注点，不在本次迭代范围内。

两条原则约束每一个实现阶段：

- **身份只在边界处确立。** `x-username` 在 HTTP 和 WebSocket 入口解析、校验、规范化一次；之后的一切都从 `SessionHeader` 读取持久化的 `userId`，而不是从请求级存储读取，因为一个 turn、它的流、后台 job 和 subagent 都存活于 HTTP 请求之外。
- **每个会话级操作都按调用方的 `userId` 过滤。** list、resume、fork、export、search、history 这六个操作，各自从持久化的会话头部解析调用用户，并拒绝或省略任何不属于该用户的内容。

以下范围决策已与部署方达成一致，记录于此，以免后续阶段重新讨论：

- 直接持久化到 **Postgres**（没有 SQLite 过渡步骤）。
- **设置保持全局** —— 一份共享文档；只有会话及会话派生数据是按用户的。
- **缺失或格式非法的 `x-username` 一律 401/403 拒绝**，绝不映射到某个共享匿名用户。
- 一把**共享 API key** 通过环境注入；没有按用户的凭据面。
- **暂不容器化**；内存中的 live session 模型因此保持单实例。
- 对外 API 目前**只暴露 chat**；bash/fs/tools 对调用方不可达，这也把执行沙箱隔离问题推迟到工具开放之时。
- **对 agent、工具、技能和资源的基于角色的访问控制（RBAC）不在本次迭代范围内，予以推迟。** 本提案只强制"谁能读取并操作某会话的数据"，而不强制"某角色能调用哪些能力"。

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

## 对外 chat API

现有 `/api` RPC 面已经流式传输会话的 prompt 和 live 帧（`session.prompt` 加 mux 下行）。后端复用它，加上主体门和 `userId` scoping，而不是另造一个并行的 REST 面。协议版本字段只有在客户端独立于 host 发布时才引入到 `host.describe`；在那之前客户端与 host 如今天一样同船发布。

## 分阶段交付

| 阶段 | 内容 | 主要涉及包 |
|---|---|---|
| 0 | 本提案，实现前评审 | `.agents/notes/` |
| 1 | `UserId`、`ctx.principal`，以及 Connection 中的 `x-username` 边界门（HTTP + WebSocket） | `identity/*`、`client/connection` |
| 2 | 贯穿所有后端的 `SessionHeader.userId`，以及 `session-persistence-postgres` | `core/session`、`session/*` |
| 3 | `settings-postgres` 全局文档 | `settings/*` |
| 4 | `api/remotes` 中的数据隔离收敛：list/resume/fork/export/search/history 上的 `userId` scoping，审计日志带 `userId` | `api/remotes`、`host/apiproxy` |
| 5 | 对外 chat API 收敛与版本协商 | `host/*`、`api/*` |
| 6 | Postgres 支撑的会话搜索/血缘 provider（MVP 之后再做） | `session-query/*` |

每个阶段按仓库惯例补齐 per-file 覆盖率门和其包的 `./invariant` companion。

## 考虑过的替代方案

**SQLite 加持久卷，之后再迁移。** 被部署方否决：他们选择直接用 Postgres。持久化 seam 本可以让后续替换成本很低，但现在就需要共享数据库及其全文检索能力，而不是再做一次迁移。

**在后端内部处理 OIDC 或 JWT。** 被否决：IAM 网关已在可信内网上完成认证并断言身份。后端校验并规范化被断言的 header，而不是重做认证。

**新的版本化 REST/SSE chat 面（`/v1/chat`）。** 现阶段被否决：现有 `/api` RPC 面已经流式传输 prompt 和 live 帧，复用它并加上主体门和 scoping 能把前端改动降到最小。若日后需要一个独立于 web host 的契约，干净的 REST 面仍可作为选项。

## 验收标准

- 没有合法 `x-username` 的请求在分发前被 401/403 拒绝，HTTP 和 WebSocket 升级皆如此。
- `SessionHeader` 携带 `userId`，list、resume、fork、export、search、history 全部过滤到调用用户自己的会话。
- Postgres 持久化后端通过共享的 `SessionPersistence` 契约套件，设置 provider 把一份全局文档存进 Postgres。
- 审计日志在会话事实之外携带调用方 `userId`。
- 共享 API key 从环境解析；不存在按用户的凭据面。

## 风险

可信头模型只和网络前提一样强：若后端可被直接触达，任何调用方都能伪造 `x-username`。部署必须用 IAM 网关前置它（网关覆盖该 header），并把后端绑定到内网接口。

只存在于请求作用域的身份，会在 turn 于请求结束后继续时立刻失效。持久化的 `SessionHeader.userId` 是运行时权威来源；隔离检查绝不能读 `ctx.principal`，只能由创建或恢复会话的边界读取。

`SessionHeader` 变更是一次无迁移路径的持久化格式变更（pre-release 立场）。`SESSION_FORMAT_VERSION` 和 Postgres `SCHEMA_VERSION` 的递增意味着新构建无法打开现有的 JSONL 和 SQLite 存储；这只有在尚不存在外部部署时才是可接受的。

Postgres provider 必须精确复现 JSONL/SQLite 的持久化语义（连续 seq、中断 turn 恢复、revision 观察），否则 resume 和 search 会在各后端之间静默分叉；共享契约套件是缓解手段，冷路径行为需要显式测试。

推迟 RBAC 意味着没有按用户的能力边界：任何已认证用户原则上都能调用 host 组合所暴露的任何东西。chat-only 范围使这一点今天仍然安全。在向调用方暴露 bash、filesystem、subprocess 或其他工具之前，部署必须引入 RBAC 和远程沙箱 provider（例如现有的 [E2B 执行世界](../../implemented/architecture/2026-07-28-portable-execution-world-consumers.md)）；两者与工具本身一同推迟。

本提案不取代任何现存 Agent Note。它建立在以下 note 之上并与之交叉引用：[浏览器信任边界](../../implemented/architecture/2026-07-28-api-browser-trust-boundary.md)（主体门叠加于其上的可达性栅栏）、[会话持久化](../../implemented/architecture/2026-06-14-session-persistence.md)（Postgres 后端所扩展的 seam）、以及 [API Proxy 一元迁移](../../proposed/architecture/2026-08-10-unary-apiproxy-remote-migration.md)（本工作所收敛的 API 面）。
