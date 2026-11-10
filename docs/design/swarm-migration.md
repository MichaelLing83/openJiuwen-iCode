# iCode Swarm 迁移分析与规划

> Branch: `feature/swarm` · 对照源仓：[JiuwenSwarm](https://gitcode.com) 本地树 · 议题：[MichaelLing83/openJiuwen-iCode#1](https://github.com/MichaelLing83/openJiuwen-iCode/issues/1)  
> 状态：分析纠偏后的工作底稿（非用户指南；未列入 `docs/index.yaml`）

## 总判断

**可迁移设计与能力面，不可整包搬实现。**

iCode / Chrys 今天是层次委托（`SubAgentTools` + `SubAgentToolShell`）+ Workflows DAG，不是 peer swarm。`src/chrys` 中无 `swarm` / `TeamAgent` / `agent_teams` / `build_team` 命中。

目标形态：在现有架构上落**薄层**团队机制——Leader = 主会话 `AgentProfile`；Teammate = `SubAgentRef` 模板 + 动态 invocation；任务真相在 `service/team_board`；确定性编排映射现有 Workflows。**禁止**引入 openjiuwen `TeamAgent` / Gateway WS / `TeamManager` / pyzmq。

默认路径：**固定 Swarm DAG 模板 + 槽位填充**；探索型任务留给正交的树状子代理环。

---

## 1. 源仓对照（问题域 → iCode）

源仓边界以 jiuwenswarm 为准（包路径前缀 `jiuwenswarm/`）：

| 区域 | 实际路径 |
| --- | --- |
| Swarm 装配 | `jiuwenswarm/agents/swarm/`（含 `DESIGN.md`、`enrich_team_spec_for_swarm`） |
| Team 运行时 | `jiuwenswarm/agents/harness/team/`（`TeamManager`、`team.db`、messager、zmq） |
| team_helpers | `jiuwenswarm/server/runtime/agent_adapter/team_helpers.py`（不与上两者同级） |
| 框架边界 | 外部包 `openjiuwen.agent_teams`（非本仓目录） |

| 问题域 | 语义要点 | iCode 侧含义 |
| --- | --- | --- |
| 角色分工 | Leader 协调、Teammate 执行；Leader 不走 nested `task_tool` | Leader = 主会话 AgentProfile；Teammate = 有名 child invocation |
| 任务拆解与派工 | `create_task` / assign / `claim`、评审 | 需任务板；`todo_write` 不够 |
| 结果聚合 | Leader 汇总、verification | Leader + 任务快照工具，或 DAG join / 聚合节点 |
| 共享上下文 | team-workspace、skills、artifacts | Profile / workspace；非 enrich 全链 |
| 成员间通信 | messager / 分布式 pyzmq | MVP：任务板 + 工具结果；peer 消息后置 |
| 动态编队 | 运行时 `spawn_teammate` / AgentGroup | 与固定 DAG **正交**；动态成员留在 `orchestration/team` + SubAgentTools |
| 失败与人类决策 | HITL、blocked | 对齐 `SubAgentToolShell` pause/retry |
| LLM / 网络重试 | provider 层 | **直接复用** WireRetryPolicy / StreamRetryLoop |
| 确定性多步编排 | Swarmflow（与自主派工正交） | **映射到现有 Workflows DAG** |
| 跨进程 / 冷启动 | `build_context_seed`、A2X | ACP / worker；勿先搬 pyzmq |
| 可观测与 UI | 看板、workflow 树 | EventBus 新 Team* 事件 + 现有嵌套视图 |

工具名以 jiuwenswarm 实际面为准：`create_task` / `claim_task` / `reassign_task` / `spawn_teammate`（及 `org_*` 变体）。**不要**写 `spawn_task`。

---

## 2. 复用判定

### 直接复用

- `EventBus` + `Invocation*`（`foundation/events/bus.py`）— 团队状态发新事件，保持 `BoundEmitter` / origin 链
- `InvocationOrigin` — teammate 用 `kind=sub_agent`，parent=turn
- `SubAgentToolShell` — Teammate = 有名 child，非 peer 对等多连接
- Wire / 验证 / Compaction 重试 — 每个 teammate 仍走现有 invocation 语义

### 部分复用

- `SubAgentTools` — 按 profile **静态**注册；缺任意运行时 spawn（MVP 的 `spawn_teammate` 需预注册 `SubAgentRef` 模板，或扩展注册 API）
- 三路径工具 / MCP 装配 — 可给 leader 增 `team_*` 工具包；勿动 kernel
- `ExecutionLease` — 单 workflow 独占；团队长回合需定义与 lease 关系
- `todo_write` — 单会话列表，**非**多成员 claim/assign 任务板
- Workflows DAG（`orchestration/workflows/coordinator.py` → `load_workflow` → `admit_manifest` → `WorkflowRunner`）— 映射 Swarmflow；节点用 `WorkflowAgentShell`
- ACP — 远程成员候选；非 Jiuwen pyzmq 位置透明

### 嵌套 UI 语义（纠偏）

子代理 / 嵌套视图使用 `shows_whole_segments=True`：**可并发**执行多个 child，答案段在**完成后整段**展示，不是主聊天那种 live token 流。勿写「嵌套流式投影」。

### 须新写 / 不复用

- openjiuwen `TeamAgent` — Chrys 不引入
- `team.db` / `TeamManager` — 薄 service 层 + 可选 sqlite
- Leader 团队原生工具面（`create_task` / assign / `spawn_teammate` 等）
- Peer messager — MVP 用任务板 + 工具结果即可

### 薄层原则

> Leader = 当前主会话 AgentProfile；Teammate = `SubAgentRef` 模板 + 动态 invocation；任务真相在 `service/team_board`，不经 kernel 新角色。禁止引入 TeamAgent 运行时或改 kernel/client 协议。

| 路径 | 层级 | 职责 |
| --- | --- | --- |
| `orchestration/team/` | 宿主 / 调度 | 薄编排；不新 Runner；主 turn 为 leader |
| `service/team_board/` | 状态 | 任务 id / assignee / status / body；`TeamTaskUpdated` → EventBus |
| `service/profiles/…` | 装配 | Team 模板：成员 YAML；进三 build 路径 |
| `orchestration/workflows/` | Swarmflow | 确定性图仍走现有 workflow worker |
| **不改** | 框架 | kernel loop、AgentEngine、transcript 不变量、分层 DAG |

MVP 闭环（示意）：用户选 Team profile → 主 turn 携带 `team_*` 工具 → `create_task` 写 board → `spawn_teammate` 经已注册的 SubAgent 开 child → 子代理 claim/完成写 board → 工具结果回 leader → EventBus 刷 TUI。聚合由 leader + `team_board_read` 完成。

---

## 3. 动态 swarm vs 固定模板

### 3.1 用现有 Workflows 承载确定性编排

iCode 工作流在 worker load 时执行作者 Python，产出**静态** manifest（`WorkflowBuilder.build` → `Workflow.manifest`）；运行时由 Scheduler 沿条件边路由，**而非** LLM 现场「画」新图。**不支持**运行时向 manifest 插入新 node id。

迁 Swarmflow：走 `WorkflowCoordinator` → `load_workflow` → `admit_manifest` → `WorkflowRunner`，**不**嵌 openjiuwen workflow VM。

动态成员与 DAG **正交**：留在 `orchestration/team` + `SubAgentTools`（双轨：确定性模板 + 自主子代理环）。

### 3.2 优先方向：固定参数化 DAG + 槽位填充

相对 jiuwenswarm 每会话 `enrich_team_spec_for_swarm` 全量烘焙 rails/tools、改写 DeepAgentSpec、再 `spec.build`：

**固定一份参数化 DAG（拓扑不变），每轮 run 只填任务内容**，而不是每次 enrich 全表或动态拼装图。

| 槽位 | iCode 映射 | 每任务填充什么 |
| --- | --- | --- |
| Leader | 单 agent 节点或主 turn 委派 | 指令后缀 + 任务输入 `WorkflowValue.text` |
| 子任务槽 | 并行 agent 节点或 sub_agent 工具环 | fan-out / join；并发受 `SubAgentRef.max_concurrency` |
| 聚合 | python combine 或 agent 汇总 | `wf.join(..., combine=)` 或最后 agent |
| 分支 | switch / when 边 | 深度、是否评审、失败重试等**枚举**，而非新节点 |
| 任务板 | （拟）`service/team_board` | 状态写在 `WorkflowValue.data`，或侧路 store |

二次及以后运行可复用已确认 `spec_digest` / ledger。

注：Jiuwen Swarmflow 多为脚本驱动（Leader 可生成 `scripts/workflow.py`），并非「运行时往 live manifest 插 node」。与 iCode「已 admit 的静态图」对照时，重点是**放弃 per-session 全量 enrich 为默认**，而不是误读为同一套动态图 VM。

### 3.3 软件工程任务是否更适合模板化？

**大体是的。**

- 常见阶段可命名：理解需求与代码库 → 定位/设计 → 实现或修改 → 验证 → 总结交付。
- **模板友好**：feature / bugfix / refactor / test / review / 文档同步。
- **保留动态性的例外**：纯调研、选型、安全审计探索、需求严重模糊、0→1 架构摸索、跨仓依赖梳理、性能根因未知 — 更依赖自主 Leader 或主会话内动态子代理。

因此：**默认以固定 Swarm workflow 模板为主路径**；未知留给节点内推理或正交子代理环；仅在探索型任务上加动态装配。

---

## 4. 相对 issue #1 的纠偏摘要

| 项 | 状态 | 说明 |
| --- | --- | --- |
| 缺失 Canvas / § 引用 | 以本文为准 | 原 `jiuwenswarm-swarm-analysis.canvas.tsx` 不在仓内 |
| `feature/swarm` 分支 | 已创建并推送 | 基于当时 `main` tip |
| `MichaelLing83/iCode#1` | 待处理 | 迁仓后仍 OPEN，应关闭并指向本仓 #1 |
| 源仓路径 shorthand | 见 §1 表 | 补全 `jiuwenswarm/`；`team_helpers` 位置 |
| 「嵌套流式」 | 见 §2 | 改为整段完成语义 |
| `spawn_task` | 见 §1 | 统一为 create/claim/spawn_teammate |

---

## 5. 迁移规划

### P0 · 固化边界与文档

- [x] 创建 `feature/swarm` 并推送
- [x] 本设计底稿入库（`docs/design/swarm-migration.md`）
- [ ] 修正 issue #1 中的路径 / 工具名 / 嵌套流式表述（或改指向本文）
- [ ] 关闭重复的 `MichaelLing83/iCode#1`，链接到本仓 #1

### P1 · 产品拍板 + 第一条可交付路径

拍板 MVP 入口（可二选一或并行切片）：

1. **固定 Swarm DAG 模板 + 槽位填充**，或  
2. **Leader 工具 + 进程内 teammates + team_board + EventBus**

落地项：

- [ ] `service/team_board` 骨架（store + `TeamTaskUpdated`）
- [ ] Leader `team_*` 工具包；装配进三 build 路径；teammate 经 `SubAgentToolShell`
- [ ] 首个软件工程 Swarm 模板：理解 → 定位/设计 → 实现 → 验证 → 汇总（fan-out/join + 可选评审边）

### P2 · 体验与运行时策略

- [ ] TUI / EventBus：任务板与嵌套成员视图（整段完成语义）
- [ ] `ExecutionLease`：团队长回合 vs workflow 独占；admission / rebuild 规则与测试

### P3 · 扩展

- [ ] 远程成员（若需要）：ACP sub-agent；不搬 pyzmq
- [ ] 探索型动态装配：仅对无法枚举拓扑的任务；默认仍是模板 + 节点内推理

### 硬约束（全程）

- 不引入 TeamAgent 运行时  
- 不改 kernel loop / client 协议 / transcript 不变量  
- 任务真相在 `team_board`，不经 kernel 新角色  

---

## 参考

- 议题：[openJiuwen-iCode#1](https://github.com/MichaelLing83/openJiuwen-iCode/issues/1)（上游镜像自 [openJiuwen-ai/iCode#239](https://github.com/openJiuwen-ai/iCode/issues/239)，已关闭）
- 源仓：`jiuwenswarm/agents/swarm/DESIGN.md`
- iCode 不变量：`AGENTS.md`（三 build 路径、`ExecutionLease`、`InvocationOrigin`、transcript）
