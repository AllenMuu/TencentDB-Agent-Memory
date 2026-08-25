# TencentDB Agent Memory Project/Repository Scope 后续实施计划

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 在保持现有 V3 `team_id + agent_id + user_id` 行为兼容的前提下，引入统一的 Scoped Memory Runtime，使 Project、Repository、Task、Session 和 Actor 能够被明确建模、授权、检索和逐步迁移。

**Architecture:** 采用 fork-first、upstream-compatible 的增量路线。先在 SDK/Core 中建立与存储无关的 `MemoryContext`、`MemoryBackend`、`ScopeResolver` 和 `AccessPolicy`，再接入现有 Metadata Store、V3 Gateway 和 MemoryProxy；Project/Repository/Task 元数据与 memory 记录分层演进，旧路径通过 legacy context 保持原语义。所有 scoped recall 都在相似度排序前完成 tenant、scope 和 ACL 过滤。

**Tech Stack:** TypeScript、Node.js 18+ SDK、Node.js 22+ MemoryCore、Vitest、SQLite/MongoDB metadata adapters、现有 V3 HTTP/MCP gateway、MemoryProxy agent adapters。

**Spec:** `/Users/allenj/Documents/Codex/2026-08-25/referenced-chatgpt-conversation-this-is-an/outputs/tencentdb-agent-memory-implementation-handoff.md`

## Global Constraints

- 保留旧的 `teamId + agentId + userId` 构造方式，并将 `teamId` 在 Runtime 层映射为 `tenantId`。
- 不删除或重命名既有 `team_id`、`agent_id`、`user_id` 字段；历史数据无法确定 Project 归属时保持 legacy scope。
- 采用 `expand → dual-read → dual-write → backfill → cutover`，每个切换点由 feature flag 控制并可回滚。
- `agent_id` 只表示 Actor，不再默认决定跨 Agent 的 Memory Partition；共享必须由 Scope、visibility 和 ACL 同时允许。
- 向量检索必须先过滤 tenant/scope/ACL，再进行向量或相关性排序。
- header/body 冲突不得静默覆盖已认证身份；客户端不得单独决定 visibility。
- 不重写向量数据库或 embedding 管线，不把 Skill 设计为新的隔离层，不自动合并无法确定归属的旧 memory。
- 每个里程碑都必须有独立测试、兼容性说明、feature flag 和可拆分的提交边界。

## 现状与实施边界

当前仓库已有：

- V3 SDK：`sdk/memory-core/typescript/src/v3/client.ts`、`types.ts`、`http.ts`，包含严格隔离上下文和旧 API 调用路径。
- Metadata：`MemoryCore/src/metadata/types.ts`、`service/metadata-service.ts`、`store/interface.ts`、SQLite/MongoDB adapter 和 `/v3/meta` router。
- Task、Agent、Asset、ACL 等已有实体或服务能力；但 Task 当前没有 Project/Repository 关联，Asset 的 Skill 也没有 RFC 所需的 promote 来源链路。
- MemoryProxy：`MemoryProxy/src/agent-adapters/`、`memory/`、`knowledge/` 和 `meta/`，是 context 传播与外部 Agent 集成的主要入口。

本计划先完成 M0–M2 的可运行基础，再按 M3–M5 推进跨 Agent、Skill Asset 和 upstream 拆分。实现过程中若现有模块的实际导出名与计划不一致，应优先保留既有模块边界，并在同一任务的文件清单中修正引用，而不是进行无关重构。

## 文件地图

### 新增文件

- `sdk/memory-core/typescript/src/scoped/types.ts`：跨 SDK/Core 共享的 Actor、Scope、Context、MemoryRecord、Project/Repository/Task 输入输出类型。
- `sdk/memory-core/typescript/src/scoped/policy.ts`：scope cascade、visibility、legacy fallback 的纯函数策略。
- `sdk/memory-core/typescript/src/scoped/client.ts`：兼容旧配置的 Scoped Memory Client 以及 Project/Repository/Task、remember/recall/promote 方法。
- `sdk/memory-core/typescript/src/scoped/http-headers.ts`：HTTP/MCP scope header 的解析、序列化和冲突检测。
- `MemoryCore/src/scoped/runtime.ts`：Core 侧 Runtime façade，负责 context 归一化、backend 调用和 feature flag。
- `MemoryCore/src/scoped/backend.ts`：`MemoryBackend`、`ScopeResolver`、`AccessPolicy` 接口及 legacy adapter contract。
- `MemoryCore/src/scoped/tencentdb-backend.ts`：TencentDB metadata/memory 存储的 backend adapter。
- `MemoryCore/src/scoped/feature-flags.ts`：scoped read、cascade、dual-write 的显式开关。
- `MemoryCore/src/scoped/migrations/`：增量 schema、backfill、迁移检查和回滚记录。
- `MemoryCore/src/scoped/*.test.ts`、`sdk/memory-core/typescript/src/scoped/*.test.ts`：契约、策略、客户端和兼容性测试。

### 主要修改文件

- `sdk/memory-core/typescript/src/v3/types.ts`、`src/v3/client.ts`、`src/v3/index.ts`：增加可选 project/repository/task/session 配置和导出，保持原方法签名可用。
- `MemoryCore/src/metadata/types.ts`、`store/interface.ts`、`metadata-store.contract.ts`、`service/metadata-service.ts`：Project、Repository、Task 关联、生命周期和 ACL 契约。
- `MemoryCore/src/metadata/router/v3-meta-schemas.ts`、`v3-meta-router.ts`：Project/Repository/Task 关联接口及请求身份校验。
- `MemoryCore/scripts/db/sqlite-init.sql`、MongoDB 初始化/adapter 文件：增量表、字段和索引。
- `MemoryCore/src/gateway/types.ts`、`v2-router.ts`、相关 handler：context 传播、旧请求适配和 recall 前置过滤。
- `MemoryProxy/src/agent-adapters/*.ts`、`src/identity.ts`、`src/memory/memory-bridge.ts`、`src/knowledge/core-client.ts`：从 Agent runtime 解析并传播 project/repository/task context。
- `README.md`、`README_CN.md` 或对应模块 README：兼容配置、header、feature flag 和迁移运行说明。

## 里程碑与任务

### M0：基线锁定与 Scoped Runtime 抽象

目标是建立可独立测试的接口和 legacy 行为适配，不启用 scoped read、cascade 或 dual-write。

### Task 0.1：固定现有 V3 行为

**Files:**

- Create: `sdk/memory-core/typescript/src/v3/client.compatibility.test.ts`
- Modify: `sdk/memory-core/typescript/src/v3/client.ts`（仅在测试暴露现有行为确有缺口时修改）
- Test: `sdk/memory-core/typescript/src/v3/client.compatibility.test.ts`

**Interfaces:**

- Consumes: 现有 `MemoryClient`、`V3IsolationContext`、`V3HttpTransport`。
- Produces: 可证明旧配置、旧请求 body、session 写入校验、`withIsolation()` 和旧异常类型未改变的回归套件。

- [ ] 用 fake `Transport` 记录 POST body，覆盖旧构造方式只发送 `team_id/agent_id/user_id`，可选 `task_id/session_id` 仍按现状处理。
- [ ] 覆盖 `addConversation()` 缺少 session 时抛出 `ParamError`，以及 `deleteConversation()` 不继承构造函数 session 的现有安全语义。
- [ ] 覆盖 `withIsolation()` 返回新 client 且不修改原 client。
- [ ] 运行 `cd sdk/memory-core/typescript && npm test -- --runInBand`；预期所有已有测试和新增兼容测试通过。
- [ ] 记录当前 gateway、MemoryProxy 和 metadata contract 的基线命令及结果，作为后续每个里程碑的比较基准。

### Task 0.2：定义 Runtime 类型和后端接口

**Files:**

- Create: `sdk/memory-core/typescript/src/scoped/types.ts`
- Create: `MemoryCore/src/scoped/backend.ts`
- Create: `MemoryCore/src/scoped/feature-flags.ts`
- Create: `sdk/memory-core/typescript/src/scoped/types.test.ts`
- Create: `MemoryCore/src/scoped/backend.test.ts`

**Interfaces:**

- Consumes: 旧 `V3IsolationContext` 的 team/agent/user 三元组。
- Produces: `MemoryActor`、`MemoryScope`、`MemoryContext`、`MemoryRecord`、`ProjectEntity`、`RepositoryEntity`、`TaskEntity`、`MemoryBackend`、`ScopeResolver`、`AccessPolicy`。

- [ ] 在 `types.ts` 中定义 `ScopeType = global | user | team | project | repository | task | session`，并将 `tenantId`、Actor、scope、parentScopes、policyVersion、traceId 分开表达。
- [ ] 为 `MemoryBackend` 固定最小方法：`createProject`、`getProject`、`attachRepository`、`createTask`、`remember`、`recall`、`promoteAsset`；每个方法都显式接收 `MemoryContext` 或等价的已解析参数。
- [ ] 为 `AccessPolicy` 固定 `canRead(context, record)`、`canWrite(context, target)` 和 `resolveVisibility(input)`；禁止 policy 从未认证 body 推导身份。
- [ ] 为 feature flags 固定 `scopedRead`、`cascade`、`dualWrite` 三个开关，默认全部关闭，支持从环境变量读取并在日志中输出版本。
- [ ] 用纯单元测试验证：旧三元组可归一化为 legacy user/team scope；空 tenant、空 actor user、非法 scope 组合被拒绝；默认 visibility 不会由客户端输入直接提升。
- [ ] 运行 `cd sdk/memory-core/typescript && npm run build && npm test` 与 `cd MemoryCore && npm test -- --run`。

### Task 0.3：实现 legacy context 和 TencentDB backend adapter

**Files:**

- Create: `MemoryCore/src/scoped/runtime.ts`
- Create: `MemoryCore/src/scoped/tencentdb-backend.ts`
- Create: `MemoryCore/src/scoped/runtime.test.ts`
- Modify: `MemoryCore/src/gateway/types.ts`、`MemoryCore/src/gateway/v2-router.ts`

**Interfaces:**

- Consumes: Task 0.2 的 Runtime contract、现有 metadata service/store、现有 V3 gateway handler。
- Produces: `normalizeLegacyContext(input)`、`ScopedMemoryRuntime.remember()`、`ScopedMemoryRuntime.recall()`，以及不改变默认行为的 TencentDB adapter。

- [ ] 实现 `normalizeLegacyContext({ team_id, agent_id, user_id, task_id?, session_id? })`：`tenantId = team_id`，Actor 保留 user/agent，scope 默认使用 legacy user/team scope。
- [ ] adapter 初期只委托已有读写路径或返回明确的 unsupported error；不能在没有 schema/ACL 保障时伪造 scoped 数据。
- [ ] 在 gateway 入口增加可选 runtime 注入点，旧请求不经过 scoped read/cascade，旧 response envelope 保持不变。
- [ ] 测试 legacy 请求与新 runtime 双方得到相同的旧路径调用参数，测试 backend 失败不会吞掉错误或扩大 recall 范围。
- [ ] 运行 `cd MemoryCore && npm run build && npm test -- --run`，并保存 baseline 对比结果。

## M1：Project / Repository / Task 一等实体

### Task 1.1：增加实体模型、schema 和索引

**Files:**

- Modify: `MemoryCore/src/metadata/types.ts`
- Modify: `MemoryCore/src/metadata/store/interface.ts`
- Modify: `MemoryCore/src/metadata/store/metadata-store.contract.ts`
- Modify: `MemoryCore/scripts/db/sqlite-init.sql`
- Modify: `MemoryCore/src/metadata/store/sqlite-adapter.ts`
- Modify: `MemoryCore/src/metadata/store/mongodb-adapter.ts`
- Create: `MemoryCore/src/metadata/store/project-repository-task.test.ts`

**Interfaces:**

- Consumes: 现有 `TeamEntity`、`TaskEntity`、metadata store contract。
- Produces: `ProjectEntity`、`RepositoryEntity`、`ProjectRepositoryEntity` 或等价关联、扩展后的 `TaskEntity.project_id/repository_id`，以及 CRUD/list/archive/attach 方法。

- [ ] 增加 Project 的稳定 ID、tenant/team、owner、name、description、status、policy、timestamps；归档是状态变更，不是物理删除。
- [ ] 增加 Repository canonical identity、provider/host/organization/repository、project association、location/version metadata；本地绝对路径只能作为 metadata。
- [ ] 为 Task 增加可空 `project_id` 和 `repository_id`，创建与更新时检查引用存在且 repository 属于该 project。
- [ ] 新增 RFC 指定的复合索引：tenant+project+scope、tenant+project+repository、tenant+task、tenant+visibility、tenant+actor。
- [ ] 对 SQLite 和 MongoDB 的 store contract 增加同一套 CRUD、引用完整性、归档和 canonical identity 测试。
- [ ] 运行 `cd MemoryCore && npm test -- --run`；SQLite 与可用的 MongoDB contract 都必须通过，未配置 MongoDB 时记录跳过原因。

### Task 1.2：增加 Metadata Service 和 `/v3/meta` API

**Files:**

- Modify: `MemoryCore/src/metadata/service/metadata-service.ts`
- Modify: `MemoryCore/src/metadata/router/v3-meta-schemas.ts`
- Modify: `MemoryCore/src/metadata/router/v3-meta-router.ts`
- Modify: `MemoryCore/src/metadata/service/permission-checker.ts`
- Create: `MemoryCore/src/metadata/router/project-repository-task.test.ts`

**Interfaces:**

- Consumes: Task 1.1 的 store 方法和现有 `V3AuthContext`。
- Produces: Project create/get/update/archive/list、Repository attach/list、Task create/update/list with project/repository linkage，并返回统一 envelope。

- [ ] 为所有新增请求定义 Zod schema：非空 ID、名称长度、canonical identity 组成、状态转换和分页边界。
- [ ] 将 owner/member/team role ACL 复用现有 permission checker；未授权 get/list 不返回实体是否存在、repository 列表或 task 归属信息。
- [ ] 明确归档规则：active project 才能新建 task/attach repository；已有 memory 不删除，recall 由 policy 决定是否可见。
- [ ] 增加 header/body 冲突测试，确保已认证 user/team 不能被 body 静默替换。
- [ ] 运行 Metadata router 与 store contract 测试，并用一个 HTTP-level 测试验证成功、校验失败、未授权和 not-found envelope。

## M2：Scoped write/read 与迁移兼容

### Task 2.1：扩展 MemoryRecord 和增量迁移

**Files:**

- Modify: `MemoryCore/src/gateway/types.ts`
- Modify: 现有 memory persistence schema/adapter（按实际代码路径定位）
- Create: `MemoryCore/src/scoped/migrations/001-scoped-memory-expand.ts`
- Create: `MemoryCore/src/scoped/migrations/001-scoped-memory-expand.test.ts`
- Create: `MemoryCore/src/scoped/migrations/README.md`

**Interfaces:**

- Consumes: Task 0.2 的 `MemoryRecord` 和 Task 1.1 的 entities。
- Produces: 可重复执行的 expand migration、旧字段映射为 legacy scope、新字段的 nullable 写入，以及 migration audit/rollback metadata。

- [ ] 增加 `scope_type`、`scope_id`、`project_id`、`repository_id`、`task_id`、`actor_user_id`、`actor_agent_id`、`visibility`、`source_memory_id`、`version` 等字段，旧字段全部保留。
- [ ] 为旧数据生成稳定、可解释的 legacy scope；无法识别 Project/Repository/Task 的记录不自动归并。
- [ ] migration 必须支持重复运行、分批处理、断点记录、失败重试和 dry-run 统计；禁止一次性无界扫描生产数据。
- [ ] 测试重复执行幂等、半途失败恢复、未知归属保持 legacy、rollback flag 能关闭新读路径。
- [ ] 在文档中写明 expand、backfill、cutover、回滚命令和每一步的监控指标。

### Task 2.2：实现显式 scoped write/read 和 dual-read/dual-write

**Files:**

- Modify: `MemoryCore/src/scoped/runtime.ts`
- Modify: `MemoryCore/src/scoped/tencentdb-backend.ts`
- Create: `MemoryCore/src/scoped/scoped-memory.test.ts`
- Modify: `MemoryCore/src/gateway/*memory*` 相关 handler/schema

**Interfaces:**

- Consumes: migration 后的 schema、Project/Repository/Task metadata、feature flags。
- Produces: `remember({ context, content, kind, visibility })` 和 `recall({ context, query, cascade, limit })` 的显式 scoped 行为；旧客户端仍走 legacy read。

- [ ] write 时校验 scope 与 Project/Repository/Task 关联，服务端重算 visibility；写入 provenance，包括 source actor、source memory、version 和 trace。
- [ ] recall 先构造 SQL/vector filter（tenant、允许的 scope、ACL、visibility），再调用现有相关性/向量排序；禁止先全库召回后在应用层过滤。
- [ ] dual-write 失败时记录结构化错误、retryable 状态和 trace，不破坏旧写路径；dual-read 结果去重并优先返回有 scope/provenance 的记录。
- [ ] 增加四组测试：旧客户端一致性、同 Project 不同 Agent 可按 policy 共享、不同 Project 默认隔离、Task 可读上层但 task-private 不反向泄漏。
- [ ] 运行 Core 全量测试、SDK 全量测试，并执行一次带 feature flags 的本地端到端 smoke flow。

### Task 2.3：扩展 TypeScript SDK，同时保持旧 API

**Files:**

- Create: `sdk/memory-core/typescript/src/scoped/client.ts`
- Create: `sdk/memory-core/typescript/src/scoped/http-headers.ts`
- Create: `sdk/memory-core/typescript/src/scoped/client.test.ts`
- Modify: `sdk/memory-core/typescript/src/v3/types.ts`
- Modify: `sdk/memory-core/typescript/src/v3/client.ts`
- Modify: `sdk/memory-core/typescript/src/v3/index.ts`

**Interfaces:**

- Consumes: Task 0.2 的类型、Task 2.2 的 HTTP/runtime contract。
- Produces: `MemoryClientConfig.projectId?`、`repositoryId?`、`taskId?`、`scopePolicy?`，以及 `createProject`、`getProject`、`attachRepository`、`createTask`、`remember`、`recall`、`promoteAsset` 的 typed client API。

- [ ] 新增 Scoped client 的 overload/构造方式，但旧 `new MemoryClient({ teamId, agentId, userId, ... })` 和旧方法调用 body 不变。
- [ ] header helper 固定 RFC 中的 `X-Memory-*` 字段；body 仅用于显式 scope，不能覆盖认证 user/tenant，冲突返回明确 `ParamError` 或服务端 400。
- [ ] 客户端只传请求意图，不允许把 visibility 直接提升为 global/team；服务端 policy 是最终裁决者。
- [ ] 测试类型导出、旧请求快照、新请求 header/body、冲突检测、超时/业务错误传播。
- [ ] 运行 `cd sdk/memory-core/typescript && npm run build && npm test`。

## M3：Scope Cascade、ACL 和跨 Agent 传播

### Task 3.1：实现可解释的 ScopeResolver 和 cascade

**Files:**

- Modify: `MemoryCore/src/scoped/backend.ts`
- Modify: `MemoryCore/src/scoped/runtime.ts`
- Create: `MemoryCore/src/scoped/cascade.ts`
- Create: `MemoryCore/src/scoped/cascade.test.ts`

**Interfaces:**

- Consumes: `MemoryContext.parentScopes`、AccessPolicy、Task 2 的 filtered recall。
- Produces: 默认顺序 `task/session exact → repository → project → team/user → global` 的可配置 resolver，以及每条 recall result 的 scope/visibility/source_actor/provenance。

- [ ] 将 cascade 解析为有序、去重的候选 scope 列表；每个候选包含原因、继承方向和 policy version。
- [ ] Task 读取 Project/Repository 只在 policy 明确允许时发生；Project 读取 Task-private 默认拒绝。
- [ ] partial failure 返回已授权且可解释的结果，并附带每个失败 scope 的 retryable/error code；不能因为某个 scope 失败而扩大到 global。
- [ ] 测试 session 结束后的 promote/retain 策略、不同 Project 同 Agent 隔离、cascade 关闭时只命中 exact scope。

### Task 3.2：贯通 MCP、MemoryProxy 和 Native Adapter context

**Files:**

- Modify: `MemoryCore/src/gateway/v2-router.ts`、相关 MCP/HTTP handler
- Modify: `MemoryProxy/src/identity.ts`
- Modify: `MemoryProxy/src/memory/memory-bridge.ts`
- Modify: `MemoryProxy/src/knowledge/core-client.ts`
- Modify: `MemoryProxy/src/agent-adapters/types.ts`、`default.ts`、`claude-code.ts`、`codex.ts`、`pi` adapter（如存在对应入口）
- Create: `MemoryProxy/src/scoped-context.test.ts`

**Interfaces:**

- Consumes: SDK header helper、Runtime normalizer、现有 Agent adapter identity。
- Produces: 三条集成路径使用同一 `MemoryContext` 语义，并保留 agent/provider/run/location/worktree provenance。

- [ ] 从运行时配置、仓库远程 URL、工作树信息和显式请求中解析 canonical repository identity；绝对路径只进入 location metadata。
- [ ] 在 MCP/HTTP 入口解析 `X-Memory-Tenant-Id`、user、agent、project、repository、task、session、trace headers，并与认证上下文比较。
- [ ] body 允许补充非身份 scope 信息；身份冲突拒绝；visibility 只接受服务端 policy 计算结果。
- [ ] 测试 Claude/Codex/Pi 的同 Project 多 Agent 共享、不同 Project 隔离、同仓库不同 clone 归一化、fork/mirror 默认分离。
- [ ] 运行 `cd MemoryProxy && npm run typecheck && npm test -- --run`，再运行 Core gateway 集成测试。

## M4：Skill Asset 晋升

### Task 4.1：把 Skill promote 建模为版本化资产

**Files:**

- Modify: `MemoryCore/src/metadata/types.ts`
- Modify: `MemoryCore/src/metadata/store/interface.ts`
- Modify: `MemoryCore/src/metadata/service/metadata-service.ts`
- Modify: `MemoryCore/src/gateway/skill-schemas.ts`、`skill-handlers.ts`
- Create: `MemoryCore/src/scoped/skill-promotion.ts`
- Create: `MemoryCore/src/scoped/skill-promotion.test.ts`

**Interfaces:**

- Consumes: `MemoryRecord.kind = skill`、source scope/actor/provenance、现有 Asset lifecycle。
- Produces: `promoteAsset({ memoryId, targetScope, conditions, version })`、demote/revoke 能力，以及来源、置信度、适用条件、版本和回滚关系。

- [ ] 明确 Task observation → Project asset → Team asset → Global asset 的显式晋升边界；Skill 本身不成为新的 scope type。
- [ ] 晋升前校验调用者对 source 和 target scope 的 read/share/write 权限，记录 reviewer/approval 和 policy version。
- [ ] 新版本不覆盖旧版本；撤销通过 status/version relation 完成，保留 source memory 和审计记录。
- [ ] 测试无权限晋升、条件不满足、版本并存、demote/revoke 后 recall 不再命中、回滚恢复旧版本。
- [ ] 运行 Core skill route、metadata contract 和 scoped promotion 测试。

## M5：迁移切换、性能验证和 upstream 拆分

### Task 5.1：完成 backfill、观测和可回滚 cutover

**Files:**

- Modify: `MemoryCore/src/scoped/feature-flags.ts`
- Modify: `MemoryCore/src/scoped/migrations/*`
- Create: `MemoryCore/scripts/scoped-memory-backfill.ts`
- Create: `MemoryCore/scripts/scoped-memory-verify.ts`
- Create: `docs/tencentdb-agent-memory-scoped-migration-runbook.md`

**Interfaces:**

- Consumes: M2 的 dual-read/dual-write、M3 的 cascade、metadata entities。
- Produces: 可重跑 backfill、计数/抽样校验、cutover 前后兼容性报告、单开关回滚路径。

- [ ] backfill 先 dry-run，输出可识别 project/repository/task、保持 legacy、冲突和失败重试数量。
- [ ] 仅对高置信度映射写入新 scope；低置信度和冲突记录进入审计清单，不自动合并。
- [ ] 在开启 scoped read 前验证所有 recall 结果都有 scope/provenance；在关闭 dual-write 前验证旧客户端仍可读。
- [ ] 增加指标：filtered-before-search、ACL deny、cascade partial failure、dual-write retry/failure、legacy fallback、recall latency、结果去重率。
- [ ] 演练 scoped read/cascade/dual-write 任一开关关闭后的恢复，记录数据一致性与旧客户端行为。

### Task 5.2：性能基准、真实流程验收和 PR 边界

**Files:**

- Create: `MemoryCore/scripts/bench-scoped-recall.ts`
- Create: `MemoryCore/scripts/e2e-scoped-memory.ts`
- Create: `docs/tencentdb-agent-memory-upstream-pr-breakdown.md`
- Modify: `README_CN.md`、相关 SDK/Core README

**Interfaces:**

- Consumes: 全部 M0–M4 能力和 feature flags。
- Produces: 性能基线、真实多 Agent/多 Project/多 Repository 验收报告、独立 upstream-compatible PR 列表。

- [ ] 基准至少比较 legacy recall、exact scoped recall、cascade recall；记录过滤耗时、排序耗时、总延迟、结果数和索引命中情况。
- [ ] e2e 流程覆盖：创建 Project、绑定同仓库不同 clone、创建 Task、Claude/Codex 写入、按 policy 共享、跨 Project 拒绝、Task-private 不向上泄漏、Skill promote/revoke。
- [ ] upstream 拆分为：纯 Runtime 抽象、兼容 SDK 扩展、metadata/schema、scoped read/write、cascade/ACL、Skill Asset、文档与测试；每组说明依赖、回滚方式和不含产品实验开关。
- [ ] 运行 `cd sdk/memory-core/typescript && npm run build && npm test`、`cd MemoryCore && npm run build && npm test -- --run`、`cd MemoryProxy && npm run typecheck && npm test -- --run`，并保存完整结果。
- [ ] 只有在所有验收标准满足后才关闭旧路径或提交 upstream PR；否则保留实验 flag 并在报告中列出剩余风险。

## 总体验收清单

- [ ] 旧客户端无 project 时行为与现有 V3 一致。
- [ ] 同 Project 不同 Agent 可按 policy 共享，且保留 actor provenance。
- [ ] 不同 Project 的同一 Agent 默认不可互读。
- [ ] Task 可以读取允许继承的 Project/Repository memory，Task-private 默认不能被上层读取。
- [ ] Session 结束后只能按显式策略 promote 或 retain。
- [ ] header/body 冲突明确拒绝或报错，无静默覆盖。
- [ ] 未授权 Project 无 metadata、向量结果或 cascade 失败信息泄漏。
- [ ] 同仓库不同 clone canonical identity 相同；fork/mirror 默认不同，可显式归入同一 Project。
- [ ] dual-write 失败可观测、可重试且不破坏旧路径；cascade 部分失败结果可解释。
- [ ] 所有 recall 结果带 scope 和 provenance，且 ACL 发生在检索前。
- [ ] scoped read、cascade、dual-write 均可通过 feature flag 关闭；迁移可重复、可监控、可回滚。
- [ ] MCP、MemoryProxy、Native Adapter 三条路径的 context 语义一致。
- [ ] 至少完成一次真实多 Agent、多 Project、多 Repository 流程验证，并产出性能与兼容性报告。

## 建议提交顺序

每个任务完成后单独提交，提交信息保持 upstream-compatible、可回滚：

1. `test: lock v3 isolation compatibility`
2. `feat: add scoped memory runtime contracts`
3. `feat: add legacy runtime and tencentdb backend adapter`
4. `feat: add project repository task metadata`
5. `feat: add scoped memory schema migration`
6. `feat: add scoped memory dual read write`
7. `feat: extend v3 sdk with scoped context`
8. `feat: add scope cascade and acl filtering`
9. `feat: propagate scoped context through adapters`
10. `feat: add versioned skill asset promotion`
11. `ops: add scoped migration verification and benchmarks`
12. `docs: document scoped memory rollout and upstream split`

每次提交前至少执行对应模块的 build/typecheck/test；涉及 schema 或 ACL 的提交必须附带失败场景测试和回滚说明。
