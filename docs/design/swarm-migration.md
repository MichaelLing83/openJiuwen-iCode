# iCode Swarm 设计分析与规划

> 分支：`feature/swarm` · 对照对象：本地 jiuwenswarm 源码树  
> 状态：设计工作底稿（非用户指南，未列入 `docs/index.yaml`）

## 0. 结论摘要

- **可迁移设计与能力面，不整包搬实现。** 不引入 openjiuwen `TeamAgent` / Gateway WS / `TeamManager` / `team.db` / pyzmq，不改 kernel。
- 两条实现路线，彼此正交：
  - **方案 A：预定义 Workflow 模板**——从内置的 Swarm 向工作流模板里选一个，填本次任务，按现有 Workflows 机制跑完一次 run。机制几乎全部现成，改动小（S）。
  - **方案 B：Team profile + `team_*` 工具**——选一个"团队"智能体配置，主会话当 Leader，用团队工具建任务、派给预注册的队员（子智能体），结果回到 Leader 汇总。需新增薄层（M～L）。
- **用户体验上**，jiuwenswarm 的核心是"组一支队伍，由 Leader 派工"，而不是"选一张流程图"。
  - 方案 A 加一层"团队外观"界面，能做到**看起来像**（名册、成员状态、成员记录），但**用起来不像**：成员和分工在模板里固定，运行期间不能和 Leader 对话。
  - 方案 B 加上团队界面，能做到**看起来像、用起来也基本像**；剩余差距是成员之间互发消息、成员跨任务记忆、分布式成员、群聊 @ 成员。
- 建议：先做 A 快速验证多角色流水线；要接近 jiuwenswarm 的产品形态，必须做 B；两者最后可以收进同一个"团队"入口（见 §7.4）。

---

## 1. jiuwenswarm 的 Swarm 是什么

### 1.1 用户视角

| 体验要素 | jiuwenswarm 的做法 |
| --- | --- |
| 进入方式 | Web 端模式选择器在「单 Agent 模式」和「集群模式」之间切换；TUI / IM 用 `/mode team`、`/mode code.team` |
| 团队组成 | 配置 `modes.team.<name>` 定义 Leader 与成员；可挂 AgentGroup（专家团名册）；运行时 Leader 可 `spawn_teammate` 加人 |
| 主交互面 | 和 Leader 对话；Leader 负责分解、调度、汇总，成员按角色并行处理 |
| 看到团队在干什么 | Team 面板（TUI `Ctrl+G`）：成员列表，选中成员查看其工作 |
| 确定性编排（可选） | 打开 `/swarmflow on` 后，Leader 调 `swarmflow()` 执行脚本（多为 Leader 当场生成，也可用已有脚本或 Swarm Skill）；`/swarmflows` 看运行树 |
| 人工介入 | 成员 / 工作流节点等待人工时回复（TUI 按 `h`） |
| 群聊 | SwarmChat：在群聊里 @ 专家团组建团队 |

要点：**用户不需要先选工作流模板。** 用户感知的是"我在和一支队伍的 Leader 协作"，工作流脚本只是 Leader 可选用的工具。

### 1.2 实际工作机制

- 每个会话由 `enrich_team_spec_for_swarm` 按 mode / config 把 rails、工具、子智能体等能力烘焙进成员 spec，再经 openjiuwen 构造运行对象。
- Leader 不走嵌套 `task_tool`；派工走团队原生工具（`build_team` / `spawn_teammate` / `create_task` / `claim_task` / `reassign_task`，以及 `org_*` 变体）。
- 团队状态与任务在 `TeamManager` + `team.db`；成员间通信经 messager，分布式部署用 zmq。
- 跨进程 / 冷启动靠 `build_context_seed` 重建上下文。

### 1.3 源码位置

| 区域 | 路径 |
| --- | --- |
| Swarm 装配 | `jiuwenswarm/agents/swarm/`（含 `DESIGN.md`、`assembly.py::enrich_team_spec_for_swarm`） |
| Team 运行时 | `jiuwenswarm/agents/harness/team/`（`team_manager.py`、`remote_member_bootstrap.py` 等） |
| team_helpers | `jiuwenswarm/server/runtime/agent_adapter/team_helpers.py` |
| 框架边界 | 外部包 `openjiuwen.agent_teams`（不在该仓内） |

---

## 2. 当前 iCode 的工作方式

### 2.1 用户视角

iCode 现在有两种互相独立的使用方式。

**聊天模式**

- 和**一个**主智能体对话；用 `#` 或状态栏切换主智能体。
- 子智能体在 `/agents sub-agents` 里预先配置；主智能体按需调用，聊天里显示为子智能体工具卡片，可展开看详情。
- 子智能体失败时，TUI 弹出"重试 / 放弃"卡片。
- `todo_write` 维护主会话自己的待办清单。

**工作流模式**

- `/workflow` 打开选择器，选一个工作流（内置或项目内自定义 Python 文件）。
- 界面以**执行图**为主：节点、连线、状态；点击节点看输入、输出、智能体对话记录。
- 点"开始"，在对话框里填一次输入后启动；运行中可回答节点提出的问题、批准工具调用、重试失败节点、取消。
- 一个工作流会话可以保存多次运行（运行 1、运行 2……）；想换输入就再开一次 run。

### 2.2 实际工作机制

- **聊天回合**：kernel 循环驱动主智能体；子智能体是 profile 里静态的 `SubAgentRef`，构建时注册成工具；每次调用都是一个**全新的** child，经 `SubAgentToolShell` 运行（暂停 / 重试 / 放弃都在这里）。
- **工作流 run**：worker 加载作者写的 Python，产出**静态** manifest；`WorkflowCoordinator` → `load_workflow` → `admit_manifest` → `WorkflowRunner`；Scheduler 沿条件边路由。运行时不能向 manifest 插入新节点。
- **执行租约**：工作流 run 独占 `ExecutionLease`，运行期间聊天回合无法进入。
- **嵌套显示**：子智能体和工作流节点的视图按**整段**展示答案（`shows_whole_segments=True`）：多个 child 可并发，每段文本写完后才出现，不像主聊天那样逐字流式输出。

### 2.3 和 jiuwenswarm 的差距

iCode 今天是**层次委托**（主智能体 → 子智能体的树）加**静态 DAG**，不是"团队"：

- 没有"队伍"概念：子智能体是工具，不是有身份、有状态的成员。
- 没有多成员共享的任务状态：`todo_write` 只是单会话清单。
- 没有运行时扩编：子智能体名单在 profile 里固定。
- 聊天与工作流是两个模式，Leader 不能在对话中"顺手"启动一条流程再接着聊。

---

## 3. 问题域对照与复用判定

### 3.1 问题域 → iCode

| 问题域 | jiuwenswarm 语义 | iCode 侧落点 |
| --- | --- | --- |
| 角色分工 | Leader 协调、Teammate 执行 | Leader = 主会话 AgentProfile；Teammate = 有名字的子智能体调用 |
| 任务拆解与派工 | `create_task` / assign / claim、评审 | 方案 A：图上的固定节点；方案 B：任务板 + `team_*` 工具 |
| 结果聚合 | Leader 汇总、verification | 方案 A：`wf.join` / 汇总节点；方案 B：Leader 读任务板汇总 |
| 共享上下文 | team-workspace、skills、artifacts | 现有 profile / workspace；不做 per-session 全量烘焙 |
| 成员间通信 | messager / zmq | 先用"任务板 + 工具结果"，成员互发消息后置 |
| 动态编队 | `spawn_teammate` / AgentGroup | 方案 B：预注册成员模板上按需开 child |
| 失败与人工决策 | HITL、blocked | `SubAgentToolShell` 暂停 / 重试；工作流节点问答与重试 |
| LLM / 网络重试 | provider 层 | 直接复用现有重试通道 |
| 确定性多步编排 | Swarmflow | 映射到现有 Workflows（方案 A） |
| 跨进程成员 | `build_context_seed`、A2X | 需要时走 ACP 外部智能体；不搬 zmq |
| 可观测与 UI | Team 面板、运行树 | 现有嵌套视图 + 新团队视图（§7） |

### 3.2 复用判定

**直接复用**

- `EventBus` + `Invocation*` 事件：团队状态发新事件，保持 `BoundEmitter` / origin 链
- `InvocationOrigin`：队员用 `kind=sub_agent`，parent 为当前回合
- `SubAgentToolShell`：队员 = 有名字的 child，暂停 / 重试 / 放弃沿用
- 重试、响应校验、压缩：每个队员仍走现有调用语义
- Workflows 整条运行链（方案 A 的全部机制）

**部分复用**

- `SubAgentTools`：按 profile 静态注册；方案 B 的 `spawn_teammate` 需要预注册成员模板，或新增受控的运行时注册
- 三条构建路径的工具装配：给 Leader 加 `team_*` 工具包，不动 kernel
- `ExecutionLease`：团队长回合与工作流独占的关系要重新定义（§7.4）
- `todo_write`：只是单会话清单，不能当多成员任务板

**须新写**

- Leader 团队工具面（`create_task` / assign·claim / `spawn_teammate` / `team_board_read`）
- 任务板（方案 B 才需要；见 §5.3）
- 团队视图界面（§7）

**不引入**

- openjiuwen `TeamAgent`、Gateway WS、`TeamManager`、`team.db`、messager / zmq

---

## 4. 方案 A：预定义 Workflow 模板

### 4.1 做法

1. 内置若干 Swarm 向工作流模板（例如"修 bug"、"实现功能"、"代码评审"）。
2. 用户选一个模板，填本次任务描述。
3. 按现有路径 admit → `WorkflowRunner` → Scheduler 执行一次 run。

"选模板"不是每个聊天回合都要做：一次 run 内部可以有很多节点、多轮智能体调用、问答与重试，都属于同一次 run。换任务类型才重新选模板。

### 4.2 模板结构：固定拓扑 + 槽位填充

拓扑在模板里固定，每次 run 只填数据、走已有分支，不在运行时加节点。

| 槽位 | iCode 映射 | 每次 run 填什么 |
| --- | --- | --- |
| Leader | 规划 / 拆解用的智能体节点 | 任务输入 `WorkflowValue.text` + 指令 |
| 子任务 | 并行智能体节点 | fan-out / join |
| 聚合 | Python combine 或汇总智能体节点 | `wf.join(..., combine=)` |
| 分支 | switch / when 边 | 深度、是否评审、失败重试等**枚举值** |
| 状态 | `WorkflowValue.data` | 阶段间传递的结构化状态 |

已确认过的工作流再次运行，可复用 `spec_digest` / 确认记录，不必重复确认。

与 jiuwenswarm 的对照：Swarmflow 多为脚本驱动（Leader 可当场生成脚本），本身也不是"往运行中的图里插节点"。方案 A 的关键取舍是**不把每会话全量烘焙当默认**，而不是复制一套动态图引擎。

### 4.3 软件工程任务适不适合模板化

**大体适合。**

- 常见阶段可命名：理解需求与代码库 → 定位 / 设计 → 实现或修改 → 验证 → 总结交付。
- **模板友好**：feature / bugfix / refactor / 补测试 / review / 文档同步。
- **需要动态性的例外**：纯调研、选型、安全探索、需求严重模糊、0→1 架构摸索、跨仓依赖梳理、性能根因未知。这类任务更适合由自主的 Leader 派工（方案 B）或主会话里的子智能体处理。

### 4.4 和当前 iCode 的差别

**机制上差别很小。** 现有内置示例 `demo-workflow` 已经是"规划 → 多个读者并行 → 合并 → 撰写 / 评审"的形状。方案 A 本质上是**把工程类流程产品化、预置成模板**，不是新运行时。对用户来说，它更接近"内置工程工作流"，而不是 jiuwenswarm 意义上的"团队"。

---

## 5. 方案 B：Team profile + `team_*` 工具

### 5.1 用户侧

1. 选一个 Team profile（和现在用 `#` 选 Code 等智能体一样），例如"工程协作队"。
2. Profile 里写好成员模板：主会话就是 Leader；另有若干队员人设（实现、测试、评审……）。
3. 照常发任务。Leader 自己决定建哪些任务、派给谁、要不要再拉一个队员、最后怎么汇总。
4. 界面上看到主对话、队员卡片和任务板的更新，而不是一张固定执行图。

### 5.2 系统侧

**Team profile**：仍是 `AgentProfile`，多一段成员配置（若干 `SubAgentRef` 人设）。构建主智能体时，在普通工具之外再挂上 `team_*` 工具包（走现有三条构建路径的装配）。

**`team_*` 工具（Leader 调用）**

| 工具 | 作用 |
| --- | --- |
| `create_task` | 在任务板写一条任务：id、描述、状态 |
| `assign` / claim 类 | 指定或认领执行人 |
| `spawn_teammate` | 按预注册模板开一个 child（经现有 `SubAgentToolShell`） |
| `team_board_read` | 读任务板，用于汇总 |

**执行**：仍是普通的主聊天回合。Leader 调工具 → 队员 child 跑完、结果回到 Leader → Leader 再派工或收尾。不新建运行时，不改 kernel。

### 5.3 任务板是否必须

**不必迁入 jiuwenswarm 的 `TeamManager` / `team.db`。** 只有走"多成员 assign / claim"时才需要某种共享任务状态（`service/team_board`），而且不必用 sqlite：内存存储加会话持久化即可。若 MVP 只让 Leader 直接派工、不做认领，可以先不建任务板，用工具结果回传代替。

### 5.4 和当前 iCode 的差别

| | 当前 iCode 聊天 | 方案 B |
| --- | --- | --- |
| 选什么 | 普通智能体配置 | 带成员名册的 Team 配置 |
| 子智能体从哪来 | profile 里静态注册的工具 | 仍预注册，但 Leader 显式组队、派工 |
| 任务状态 | `todo_write` 单会话清单 | 多成员可派、可认领的任务板 |
| 协作语义 | "帮我查 / 改"，子智能体跑完交回 | 建任务 → 分人 → 执行 → 汇总 |
| 体感 | 带几个固定助手的单人聊天 | 主智能体像小组长 |

---

## 6. 三方对比：当前 iCode / 方案 A / 方案 B

### 6.1 用户使用视角

| 维度 | 当前 iCode | 方案 A：Workflow 模板 | 方案 B：Team profile |
| --- | --- | --- | --- |
| 起手动作 | 选智能体，开始聊天；或切到工作流模式选一个工作流 | 选一个 Swarm 模板，填任务 | 选一个 Team 配置，开始聊天 |
| 用户心智 | "我和一个助手对话" / "我跑一张流程图" | "我跑一条预置的多角色流水线" | "我和一支队伍的 Leader 协作" |
| 主界面 | 聊天记录；或执行图 | 执行图（节点即角色） | 聊天记录 + 队员卡片 + 任务板 |
| 谁决定分工 | 主智能体临时决定调哪个子智能体；工作流里由图决定 | 模板作者（图） | Leader 模型（在预配的成员与工具范围内） |
| 中途追加要求 | 聊天里随时可以；工作流运行中只能回答节点提问 | 只能回答节点提问；新要求 = 新 run | 随时对 Leader 说；回合中还可插话改方向 |
| 看到"谁在干什么" | 子智能体卡片；节点状态 | 节点状态 + 节点对话记录 | 队员状态 + 任务板 + 队员记录 |
| 结果可预期性 | 中 | **高**（流程固定） | 中（依赖 Leader 决策质量） |
| 适合的任务 | 通用 | 流程收敛的工程任务 | 结构事先不明、需要灵活派工的任务 |
| 接近 jiuwenswarm 的程度 | 低 | 低～中（加界面后外观可接近，§7） | 中～高（加界面后，§7） |

### 6.2 实际工作机制

| 维度 | 当前 iCode | 方案 A | 方案 B |
| --- | --- | --- | --- |
| 执行载体 | 聊天回合（kernel 循环）；工作流 run | 工作流 run | 聊天回合 |
| 编排方式 | 模型自主调用工具；或静态 DAG | 静态 DAG + 枚举分支 | 模型自主调用 `team_*` 工具 |
| 成员是什么 | 子智能体工具，每次调用都是新 child | 智能体节点（每次激活一个 shell） | 预注册成员模板，每次派工开一个新 child |
| 成员能否运行时增加 | 否 | 否（不能插节点） | 能，但限于预注册模板（或受控注册） |
| 共享状态 | `todo_write`（单会话） | `WorkflowValue.data` | 任务板 |
| 成员跨任务记忆 | 无（child 每次全新） | 节点内多轮可以；跨 run 无 | 默认无（child 每次全新），要另做 |
| 成员之间通信 | 无 | 只经图上的数据流 | 只经任务板 / 工具结果 |
| 执行租约 | 回合或 run 各自持有；run 独占 | run 独占，运行期间不能聊天 | 回合持有，和普通聊天一样 |
| 失败处理 | 子智能体暂停 / 重试；节点重试 | 节点重试 | 子智能体暂停 / 重试 |

### 6.3 实现改动难度

量级是相对本仓现状的工程估计（含测试与文档），不是排期承诺。  
**S** ≈ 数人日～约 1–2 周；**M** ≈ 数周；**L** ≈ 跨多层，需与 TUI / i18n 多轮联调。

| 维度 | 当前 iCode | 方案 A | 方案 B |
| --- | --- | --- | --- |
| 核心机制改动 | —（基线） | 几乎无：新增模板文件 | 新增薄层：工具包、profile 扩展、事件、可选任务板 |
| 新运行时 | — | 否 | 否 |
| 动 kernel | — | 否 | 否 |
| 改动面广度 | — | 窄：workflows 内置模板、文档、测试 | 宽：service、orchestration、事件、三条构建路径、TUI、i18n |
| 粗代码量（含测试） | — | 数百～1k+ 行 | 数 k 行 |
| 量级（只含基本界面） | — | **S** | **M～L** |
| 再加"类 jiuwenswarm 团队界面"（§7） | — | 再加 **M** | 再加 **M**（合计偏 **L**） |
| 主要风险 | — | 低：机制成熟；风险在模板质量 | 中：Leader 派工质量、任务板一致性、TUI 性能约束 |

---

## 7. 能否靠界面让方案 A / B 获得类似 jiuwenswarm 的体验

### 7.1 要复刻的体验要素

从 §1.1 归纳，jiuwenswarm 体验由以下要素构成：

1. 一键进入"团队模式"
2. 看得见的团队名册（谁在队里、什么角色）
3. 成员状态与成员详情（Team 面板）
4. 以和 Leader 对话为主交互面
5. 任务 / 进度视图
6. 人工介入（回答问题、批准）
7. 运行中可扩编（新成员出现）
8. 群聊里 @ 成员
9. 成员之间协作、分布式成员

### 7.2 方案 A + 界面

可以做的界面改造：

- **团队入口**：在应用模式选择器里加"团队"，内部等于"工作流模式 + 只列 Swarm 模板的选择器"。
- **名册视图**：从 manifest 的智能体节点及其智能体配置推出成员列表（角色名、模型、状态）作为主视图，执行图退为次级页签。可能需要给模板加"角色显示名"元数据。
- **成员详情**：直接复用现有节点详情与对话记录。
- **Leader 输出**：把规划 / 汇总节点的输出放进类似聊天的主区域。
- **人工介入**：现有节点问答与审批已经具备。

做不到的（机制限制，靠界面绕不过去）：

- **不能和 Leader 对话**：run 独占执行租约，运行期间没有聊天回合；追加要求只能等节点提问，或开新 run。
- **分工不是 Leader 决定的**：拓扑是模板写死的，界面再像团队，"Leader"也只是一个固定节点。
- **不能扩编**：不能向运行中的 manifest 插节点。
- **@ 成员、成员互聊**：没有对应机制。

结论：**外观相似度高，交互相似度低～中。** 适合当作"团队化的流水线"展示，不能替代 jiuwenswarm 的 Team 体验。

### 7.3 方案 B + 界面

方案 B 的机制本身就是"Leader 派工"，界面主要负责把它呈现成一支队伍：

- **团队入口**：Team 配置出现在 `#` 智能体列表里；也可以在模式选择器里加"团队"，直接选中一个 Team 配置。
- **名册面板**：从 Team 配置的成员列表生成；成员状态（空闲 / 运行中 / 暂停 / 失败）由子智能体调用事件和任务板事件驱动。
- **任务板面板**：订阅 `TeamTaskUpdated`。
- **成员详情**：复用现有子智能体卡片与详情。按整段显示的语义，成员面板随每段文本完成而更新。
- **主交互面**：Leader 就是主聊天，用户随时可以继续说话；回合中的插话改方向（steer）已经支持。
- **人工介入**：复用子智能体"重试 / 放弃"卡片与工具审批。

仍有差距（需要额外机制，不只是界面）：

| 差距 | 原因 | 弥补方式（若要做） |
| --- | --- | --- |
| 成员跨任务记忆 | 每次派工都是全新 child | 为每个成员保存独立历史，下次派工时续上；属于新机制 |
| 用户直接对某成员说话 / @ 成员 | 成员只是 Leader 回合内的 child，没有独立会话 | 先做成"@ 成员"= 给 Leader 的派工提示；直连成员需上一行的成员历史 |
| 成员之间互发消息 | 只有任务板和工具结果 | 后置；可用任务板留言模拟 |
| 分布式成员 | 不搬 zmq | 用 ACP 外部智能体作为远程成员 |

结论：**外观和主要交互都可以接近**（组队、Leader 派工、名册、任务板、成员详情、人工介入）；剩下的差距集中在成员持久化、成员间通信和分布式。

### 7.4 合并成一个"团队"入口

jiuwenswarm 的形态是"Team 模式 + 可选 Swarmflow"。iCode 可以对应地做成：一个"团队"入口，选中团队后有两种运行方式：

- **自主派工**（方案 B）：默认，和 Leader 对话。
- **流程模式**（方案 A）：Leader 或用户选一个模板跑一次 run。

**关键约束**：工作流 run 独占 `ExecutionLease`，并拒绝新的回合进入。Leader 不能在自己的回合**里面**启动一个模板 run 再接着往下跑。可选做法：

1. Leader 的工具只"提议"启动某模板；当前回合结束后，由界面交给工作流去跑，跑完把结果带回下一个回合。
2. 由用户在团队界面上手动启动模板，结果作为下一条消息的上下文。

两种做法都不改变租约规则。若要支持"回合内嵌套 run"，必须重新设计租约，不建议在 MVP 阶段做。

### 7.5 界面层的实现注意

- 新面板必须遵守 TUI 性能规则：显示 / 隐藏、焦点变化、小面板切换不能让 MainScreen 重新计算样式或重新布局；固定 chrome 用不触发布局的可见性切换；流式更新要节流，但最后一个状态一定要落地。
- 所有用户可见文案走 `msg()`，完成 i18n 全流程并更新目录 oracle。
- 无头模式与 ACP 如需看到团队事件：把事件类型加进无头事件白名单，并带上 `session_id`。
- 用户可见变化需要同步更新 `docs/en/` 与 `docs/zh-Hans/`。

---

## 8. 改动面明细

### 8.1 方案 A

| 部位 | 是否必改 | 做什么 |
| --- | --- | --- |
| `service/workflows/builtins/` | 是 | 新增 Swarm 向模板（及 manifest）：阶段、fan-out / join、可选评审边 |
| 工作流发现 / 目录 | 可选 | 标签或分组，把 Swarm 模板与示例区分开 |
| `orchestration/workflows/*` | 基本不改 | 仍走 coordinator → load → admit → runner |
| kernel / AgentEngine | 不改 | — |
| `app/tui`（基础） | 可选 | 让模板更好找 |
| `app/tui`（团队外观，§7.2） | 可选 | 团队入口、名册视图、Leader 输出区；可能需要模板角色元数据 |
| 用户文档（中英） | 用户可见则要 | 如何选模板、填任务、看结果 |
| 测试 | 是 | 模板加载 / admit、关键路径；团队外观需带真实对话记录的 TUI 性能测试 |

### 8.2 方案 B

| 部位 | 是否必改 | 做什么 |
| --- | --- | --- |
| `service/team_board/`（新） | 需要认领 / 指派时 | 存储：id / assignee / status / body；读写接口 |
| `foundation/events/types.py` | 是 | 如 `TeamTaskUpdated`；TUI / ACP 订阅 |
| `service/tools/` + `packs.py` | 是 | `team_*` 工具包 |
| 三条构建路径 | 是（至少主会话） | Leader 挂团队工具；队员权限收窄 |
| `service/profiles/agents/` | 是 | Team 配置：成员列表、启用哪些团队工具；schema / loader |
| `orchestration/team/`（新，薄） | 建议 | 派工与生命周期胶水；不新建 Runner |
| `orchestration/sub_agents/` | 小改 | `spawn_teammate` 走 `SubAgentToolShell`；预注册模板或受控注册 |
| `app/tui/`（团队界面，§7.3） | 是 | 名册面板、任务板面板、成员状态 |
| i18n | 有界面文案则要 | 完整流程 |
| 无头 / ACP | 可选 | 注册团队事件 |
| `ExecutionLease` | 做 §7.4 时 | 明确团队回合与工作流的交接规则 |
| 测试 | 是 | 任务板、工具、配置装配、暂停 / 重试、TUI；量明显大于 A |
| 用户文档（中英） | 建议 | Team 配置用法；与工作流、普通子智能体的区别 |

---

## 9. 规划

### P0 · 定方向

- [x] 建 `feature/swarm` 分支
- [x] 设计底稿入库（本文）
- [ ] 产品拍板：MVP 先做 A、先做 B，还是并行切片（参考 §6、§7）

### P1 · 第一条可交付路径

- [ ] （A）首个软件工程模板：理解 → 定位 / 设计 → 实现 → 验证 → 汇总
- [ ] （B）Team 配置 schema + `team_*` 工具包（先 `spawn_teammate` + 派工，可暂不做认领）
- [ ] （B）需要认领 / 指派时再加 `service/team_board`

### P2 · 团队体验

- [ ] （A）团队外观：模式入口、名册视图、Leader 输出区（§7.2）
- [ ] （B）名册面板、任务板面板、成员状态（§7.3）
- [ ] 统一"团队"入口与 A / B 交接规则（§7.4，含 `ExecutionLease` 交接）

### P3 · 扩展

- [ ] 成员跨任务记忆（每成员独立历史）
- [ ] @ 成员：先做成给 Leader 的派工提示，再考虑直连
- [ ] 远程成员：ACP 外部智能体，不搬 zmq
- [ ] 探索型动态装配：只用于拓扑无法事先枚举的任务

### 硬约束（全程）

- 不引入 `TeamAgent` 运行时；不改 kernel 循环、client 协议、transcript 不变量
- 方案 B 的任务真相在任务板（或等价的薄存储），不经 kernel 新角色
- 不改变工作流 run 独占执行租约的规则，除非单独立项重新设计

---

## 参考

- jiuwenswarm：`jiuwenswarm/agents/swarm/DESIGN.md`；用户文档 `docs/zh/模式系统.md`、`docs/zh/TUI使用指南.md`、`docs/zh/TUI使用SwarmFlow指南.md`、`docs/zh/SwarmChat群聊专家团组织.md`
- iCode：`AGENTS.md`（三条构建路径、`ExecutionLease`、`InvocationOrigin`、TUI 性能规则、i18n）；用户文档 `docs/zh-Hans/guides/running/workflows.md`、`docs/zh-Hans/guides/configuration/agents.md`
