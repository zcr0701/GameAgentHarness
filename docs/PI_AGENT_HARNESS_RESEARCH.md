# Pi Agent Harness 代码研究

更新时间：2026-09-25。本文根据 Pi 官方仓库 `main` 分支的 Agent Loop、Coding Agent 工具、Plan Mode 示例、Session Manager、Compaction 源码及文档整理，供本项目的 AI 操作架构设计参考。Pi 是研究对象，不是本游戏的运行时依赖；本文描述的是阅读到的实现原则，不复制其代码、工具或产品行为。

## 研究范围

- `packages/agent/src/agent-loop.ts`：核心消息循环、事件顺序、参数校验、执行前后钩子、工具结果、失败和后续循环。
- `packages/agent/src/types.ts`：`prepareNextTurn`、`prepareRequest`、`finishTurn` 等可替换回合边界。
- `packages/coding-agent/src/core/tools/{index,edit,write,bash}.ts`：模型可用的代码探索、精确编辑、整文件写入和命令运行工具。
- `packages/coding-agent/src/core/tools/file-mutation-queue.ts`：同一文件的并发写入串行化。
- `packages/coding-agent/src/core/session-manager.ts`：JSONL 会话树、工具结果、压缩边界、自定义状态与上下文编辑条目。
- `packages/coding-agent/src/core/compaction/compaction.ts`：压缩阈值、保留近期上下文和摘要边界选择。
- `packages/coding-agent/examples/extensions/plan-mode/index.ts`：可选 Plan Mode 扩展示例；注意它不是核心 Agent Loop 内建的工程规划保证。
- `packages/coding-agent/src/core/system-prompt.ts`：Coding Agent 系统提示及项目上下文装配。
- `packages/coding-agent/docs/extensions.md`：扩展生命周期、类型化工具、工具并行及事件钩子。
- `packages/coding-agent/docs/compaction.md`：长会话压缩、持久会话记录与结构化摘要。
- `packages/coding-agent/docs/containerization.md`：进程/工具的隔离边界与暴露资源的风险。

源码会随上游更新。链接指向官方仓库当前 `main`；需要基于特定提交复查时，应先固定 commit SHA，再核对下面的流程是否变化。

## 本次研究结论：复杂工程能力来自什么

Pi 并没有一条神奇的“给出大型需求 → 自动写对复杂项目”函数。复杂工作主要由四个部件共同实现：

1. **多回合工具循环**让模型读代码、搜索、编辑、运行测试，再读取真实工具输出并继续推理。
2. **Coding Agent 工具集**把读、精确替换、写文件和执行命令接进统一的工具协议。
3. **活动分支会话与上下文压缩**保留工具结果和工作进度，在长任务里控制下一次模型实际看到的历史。
4. **Project Context、skills、extensions 与可选计划扩展**给不同工程补上规则和专项知识。

因此复杂度不是靠一份超长 prompt 承担，而是把工程变成可观察的循环：定位 → 改一小块 → 执行检查 → 读取错误 → 修复 → 再检查。模型负责提出下一步；Pi 负责让这一步有工具、有结果、有会话边界，具体测试命令与通过条件仍由项目规则和工程师设定。[How Pi Works](https://github.com/earendil-works/pi/blob/main/packages/coding-agent/docs/how-pi-works.md)

### Agent Loop 的实际控制顺序

阅读 `runAgentLoop` / `runLoop` 和工具调用路径后，调用顺序可以归纳为：

```text
输入消息追加到活动上下文
  → 准备下一轮消息、工具声明和上下文（可在此压缩）
  → 向模型发起流式请求
  → 收集 assistant 文本/工具调用
  → 找到当前运行时已注册的工具
  → 预处理 arguments，再按工具 schema 校验
  → beforeToolCall 策略钩子可阻止调用
  → 执行工具，发送 start/update/end 事件
  → afterToolCall 处理成功或失败结果
  → 将 toolResult 追加进上下文与会话记录
  → finishTurn / prepareNextTurn 决定结束还是继续请求模型
```

`agent-loop.ts` 将 `message.stopReason === "length"` 的工具消息视为参数可能被截断，整批不执行；正常调用则要先查工具是否存在，再执行参数预处理和 `validateToolArguments`，然后才进入执行前钩子及函数体。异常会被转换成错误工具结果，而不是从循环中消失。工具结果随后回到下一次 assistant 请求，因此模型能依据编译错误、测试失败或文件状态调整下一步。[Agent Loop 源码](https://github.com/earendil-works/pi/blob/main/packages/agent/src/agent-loop.ts)

Agent Loop 还有两层继续逻辑：一层处理当前 assistant 消息的工具调用及用户 steering；另一层处理 agent 本来准备结束后才到达的 follow-up 消息。`prepareNextTurn` 在工具完成后、下一次模型请求前有机会替换上下文/模型/消息；长耗时 compaction 就接在这个边界。`finishTurn` 则能要求继续一轮或结束。这类钩子让 Harness 能在每次推理前准备状态、在工具后验收，但不会替项目自动定义“测试通过”的业务规则。[Agent 类型与回合钩子](https://github.com/earendil-works/pi/blob/main/packages/agent/src/types.ts)

工具批次支持并行或顺序模式，带 `executionMode: "sequential"` 的工具会让批次顺序运行。文件修改也另有同文件 mutation queue：同一路径串行，不同文件仍可并行。结论是：**独立只读工作可以并行，互相依赖的改写/测试应串行**，否则会读到半完成状态。[Agent Loop 执行分支](https://github.com/earendil-works/pi/blob/main/packages/agent/src/agent-loop.ts)、[文件修改队列](https://github.com/earendil-works/pi/blob/main/packages/coding-agent/src/core/tools/file-mutation-queue.ts)

### Coding Agent 怎样落地“写复杂代码”

Pi 的编码工具集不是一个能任意调用 Node 内部对象的黑箱接口，而是模型可以调用的具名工具：`read`、`bash`、`edit`、`write`，扩展配置还能提供 `grep`、`find`、`ls` 等。

- **`edit` 是精确文本替换**：参数结构为文件路径和 `oldText/newText` 替换列表；`oldText` 必须唯一、不能互相重叠。执行时先读原文件，对原始内容应用变更，保持 BOM/行尾，再写回；返回 diff、统一 patch 和变更行号。工具提示要求改动块尽量小，把附近的相关修改合并处理。
- **`write` 用于新文件或完整重写**：路径与内容组成固定 schema；路径相对指定工作目录解析，父目录自动创建。
- **`bash` 用来检查和构建**：执行器流式返回命令输出并做输出预算/截断管理；模型可以根据失败输出再调用 read/edit/bash。
- **Schema 限制参数形状，执行器再做实际校验**：当前源代码中的 edit/write 等定义使用 provider 可用时偏好的 JSON Schema constrained sampling，但代码路径仍然先验证 arguments、解析工作目录并执行本地规则。Schema 不证明代码正确，也不替代 Unity/测试断言。

精确补丁工具能减少整文件重写时的漂移；写后返回 patch 让模型和 UI 看得见真实改动；bash 输出把编译器/测试的信息带回下一轮。复杂代码质量来自这些可循环的读写/验证能力及项目专属规范，而不是 edit 工具本身保证正确。[Coding Tool Registry](https://github.com/earendil-works/pi/blob/main/packages/coding-agent/src/core/tools/index.ts)、[Edit Tool](https://github.com/earendil-works/pi/blob/main/packages/coding-agent/src/core/tools/edit.ts)、[Write Tool](https://github.com/earendil-works/pi/blob/main/packages/coding-agent/src/core/tools/write.ts)、[Settings / Built-in Tools](https://github.com/earendil-works/pi/blob/main/packages/coding-agent/docs/settings.md)

### Plan Mode 是扩展示例，不是 Pi 内核的强制验收流程

官方 Plan Mode 示例先关闭内置 `edit/write`，再通过 `tool_call` 钩子把 shell 限制在只读命令 allowlist，要求模型做只读分析并产出编号计划。之后 UI 可以让用户选择执行、留在计划阶段或润色计划；执行时恢复完整工具，模型逐步执行并用 `[DONE:n]` 标记完成。扩展用 `pi.appendEntry("plan-mode", ...)` 持久化 todo/执行状态，恢复会话时重新载入并从本计划的执行边界后扫描进度。[Plan Mode 示例](https://github.com/earendil-works/pi/blob/main/packages/coding-agent/examples/extensions/plan-mode/index.ts)

这个示例的完成标记是从 assistant 文本中提取的 `[DONE:n]`，属于进度展示/扩展状态；它本身没有运行 Unity 测试来证明功能通过。游戏可以借鉴“规划阶段缩窄可写能力、计划显式列出、执行阶段逐项回报、状态可恢复”，但**验收一定用 Unity 的实时事实，不把模型写出的 DONE、成功话术或计划文本当作证明**。

## 对本游戏的关键设计决定：受约束的脚本 DSL

用户希望直接采用本地模型熟悉的 Python 语法子集，同时只允许调用登记好的游戏操作。脚本语言语法与游戏原语要分开：`def`、赋值、`if/else`、`for`、`while` 是 DSL 自带结构，不是游戏原语；诸如 `create_portal`、`set_portal_profile`、`explode_terrain` 才是 registry 中受限的游戏 API。项目自行实现子集 parser，不调用 CPython 执行模型输出。

```text
玩家目标
  → Unity 建立当前任务允许的游戏操作/查询/模块 registry 和 DSL schema
  → 模型按 Python 语法子集组织控制流；SemIf 只在游戏 API 调用点约束合法 primitive ID
  → Unity 解析、类型检查 AST，并把 API 调用映射到静态 registry
  → Unity 再校验关卡前置条件、参数边界、调用预算与效果预算
  → 事务/补偿层按序执行已授权节点
  → 每个节点返回结构化结果与可撤销凭证
  → 世界状态观察器检查目标断言
  → 未通过则把观测结果回交模型，要求选择新的合法修复 AST
```

未来脚本会混合**脚本语法**和**登记的游戏 API 调用**，例如：

```text
def create_strong_endless_enemies():
    portal = find_active_portal()
    if portal is None:
        portal = create_portal(interval=3.0, alive_limit=8)
    for attempt in range(2):
        wait(seconds=3.5)
        assert spawned_near(portal, profile="standard")
    set_portal_profile(portal, profile=STRONG)
    attempt = 0
    while attempt < 2:
        wait(seconds=3.5)
        assert spawned_near(portal, profile=STRONG)
        attempt += 1
    return accept(portal)
```

以上只是设计草图，**当前游戏没有实现 Python 语法子集 AST/脚本运行时**。`if/for/while/def` 由 grammar/parser 处理，不进 game primitive registry。SemIf 只负责在插入游戏 API 调用时把原语 ID 限制为当前任务合法项；参数仍由类型/schema 与 Unity 状态检查，循环、嵌套、等待、节点数和总时长有硬预算。语法、对象引用或任何 API 不合法都在实机执行前拒绝。危险或无界表达（无限循环、任意反射、任意 Python/C#、自由文件/网络/进程/Unity 对象访问）不进入语法。连续参数优先由 Unity 计算或量化，并受最小值/最大值和操作硬上限约束。

原语注册表不应只收录最终动作，可按职责分成：

| 分类 | 示例 | 注册内容 |
| --- | --- | --- |
| 游戏写操作原语 | `portal.create`、`terrain.explode`、`enemy.set_profile` | 稳定 ID、强类型参数 schema、前置条件、Unity 执行入口、影响范围、费用、错误码、undo 适配器 |
| 游戏查询原语 | `portal.spawn_count`、`enemy.profile`、`terrain.material_count` | 权威读取字段、采样窗口、值类型和观察证据；不改变游戏状态 |
| 效果模块原语 | `Fork`、`Drill`、`Blast`、`Echo`、`Impulse` | 允许的模块适配器 ID、编译语义、组合预算、执行/停止/撤销入口 |
| 脚本语言语法（不注册为游戏原语） | `def`、`if/else`、`for`、有界 `while`、赋值、`return` | DSL grammar/parser/interpreter 的语义与预算，不授予任何游戏写权限 |
| Harness 运行时内建 | `wait`、`assert`、`accept`、`undo_task` | 运行时调度与验收、取消和事务控制；受 task deadline 与 undo journal 管理 |

SemIf 约束的是脚本里的游戏 API 调用点，而不是把 Python 控制语法登记成动作，也不是暴露任意函数指针/函数名字符串。可见 ID 必须映射到静态 Unity registry entry；解释器只分派登记 ID，未知项一律拒绝。游戏当前玩家法术模块与创世动作目录见[游戏操作原语目录](architecture/action-primitives.md)。

### SemIf 输出约束和游戏端校验各守一层

当前生产 `/decide` 的真实链是：SemIf `direct_messages` 生成决策提示；llama.cpp 只推理一个答案槽 token；Python 适配层检查 tokenizer 确认槽 token 唯一，再只在 Unity 发出的选项 token 中归一化概率；Unity 再验证 response ID、候选顺序、归一化分数和 argmax。它目前只选一个固定候选 ID，参数固定，**还不是 SemIf 语法头生成多条脚本/typed AST**。

不要对脚本每个 token 反复使用 SemIf 单 token 候选评分。未来 DSL 生成时，SemIf 只在模型调用游戏原语的位置约束 ID（必要时约束有限枚举参数）；Python 语法子集中的 `if/loop/def` 按 grammar 解析执行。Unity parser/validator 再检查整份脚本。现有生产 `/decide` 的单 token 候选槽评分仅服务旧的单动作选择，不能拿来声称实现了受限脚本输出。实现前须确认 SemIf/llama.cpp 接口能在 API 调用字段限定动态 registry ID，并检查 tokenizer 对齐、截断恢复、合法率、一次通过率和时延；缺 ID 覆盖、解析或业务校验失败都在实机零副作用。

### 复合脚本与可撤销

循环、条件与高层函数必须能在 Harness journal 中逐节点跟踪：`task_id / node_id / node_kind / primitive_id? / typed_args / before_state_version / status / evidence / undo_token? / retry_count`。`node_kind` 记录脚本语法节点或游戏 API 调用；只有 API 调用节点有 `primitive_id`，只有有副作用的成功调用才有 `undo_token`。同一世界对象和地形写操作串行执行，节点超时/失败不会自动重放；只有明确声明幂等的查询或动作才可安全重试。

可撤销应以**补偿动作栈**实现：原语启动前记录足够的旧状态/资源凭证；成功后压入 `UndoRecord`；用户撤销 task 时先停止循环和待执行节点，再按逆序调用每个已成功节点的 Unity 端补偿函数。若某种地形/伤害操作没有精确恢复材料、生命、对象、队列和视觉状态的能力，则它不得假装支持强撤销：要么先建可恢复快照/事务日志，要么定义并在 UI 说明其“只能停止后续效果、不能恢复已造成结果”。循环撤销至少要能停止计时器、取消未运行迭代并撤销之前迭代产生的对象/改动。持久化 journal 才能覆盖进程崩溃后的恢复与撤销；目前游戏 task 记录只在本次 Unity 进程内存在，尚无崩溃恢复。

**原语加入门槛**：每个状态修改原语必须声明 `UndoPolicy` 和可运行的 undo/compensation handler，否则不能进入可撤销的脚本候选集。循环本身可以在用户请求下持续运行，但必须有速率上限、存活/队列/工作上限、明确的取消入口和可撤销迭代记录；它不能产生失控的对象增长。`if/assert/observe` 虽不修改世界，也要记录节点结果，供后续修复步骤复用。完整原语边界与当前实现状态应同步维护在[原语目录](architecture/action-primitives.md)及[AI Harness 架构说明](architecture/ai-harness.md)。

### 沙盒房间：先在副本验收，再向玩家世界提交

本项目的“调试房间/沙盒”是**同一 Unity 场景里、镜头外的一间隔离试炼舱**，不是新建整套 `GameWorld`，也不是操作系统权限沙箱。Pi 默认依赖宿主权限，容器/虚拟机边界由部署者提供；Codex 的 coding sandbox 则依赖操作系统限制命令的文件/网络访问。借鉴的是“执行环境应有真实边界”的原则，游戏内边界要靠世界分区、独立地形数据与执行上下文落实。[Pi 安全边界](https://github.com/earendil-works/pi/blob/main/packages/coding-agent/docs/containerization.md)、[OpenAI：在 Windows 上构建 Codex 沙盒](https://openai.com/index/building-codex-windows-sandbox/)

游戏内执行流程应加一个 staging 阶段：

```text
模型用受限输出头生成 DSL
  → 静态解析/类型/调用图/循环预算检查
  → 在同一场景镜头外启用隐藏 SandboxChamber，复制脚本读写范围所需的地形、实体和资源
  → 用同一 registry/解释器和一致的调度/物理规则，在 SandboxPartition 中运行脚本
  → 对 assert/观察节点采集结构化 trace、状态差异和失败原因
  ├─ 未通过：丢弃 sandbox 副本，将失败证据回交模型修正；有界重试，实机零副作用
  └─ 通过：冻结带哈希和状态版本的 ScriptArtifact，进入 commit 前复核
       → 实机状态/前置条件仍兼容才把同一份 DSL 交给真实世界执行器
       → 实机继续逐节点验收并写入 undo journal
       → 失败时逆序撤销已提交节点，或报告具体不可回滚边界
```

SandboxChamber 与玩家共处一个 Unity Scene 和真实 `GameWorld`，但必须有独立 `SandboxPartitionId` / context：一份单独的地形数据与 Tilemap、一组 sandbox 专属敌人/道具/调度队列、自己的玩家代理或目标代理、独立资源状态。它不计入实机敌人数/击杀/胜负，不写入玩家存档，也不出现在玩家摄像机与 HUD。爆破、命中、敌人查询与地形读写都必须从当前 context 的注册表取对象，禁止全场 `FindObjectsByType` 将测试实体混入实机结果。测试期间通过 `SimulationGate` 冻结实机控制器、敌人 AI、传送门、施法/爆破队列和地形更新；只让试炼舱继续模拟。房间与玩家区域保持大于所有原语最大影响半径的隔离距离，碰撞层/查询过滤器只允许试炼舱内部互相作用。

**不把沙盒中已经改变的 Unity 对象、物理体或单例直接搬进玩家地图。** 通过沙盒后转移的是经过验证的脚本 artifact，由正式执行器在真实地图重新执行；这样实体引用、运行队列与玩家存档仍由唯一真实 `GameWorld` 管理。commit 前检查 artifact 使用的世界快照版本/哈希，若地形、玩家位置、目标或资源已变化，应在新快照重新试炼或要求模型重订脚本，不能拿旧测试结论直接覆盖新局面。

沙盒必须复用正式执行原语与规则，不能维护一套“只为 smoke 通过”的假实现；物理步长、地形单元数据、刷怪上限、法术编译器、调度顺序和 RNG 种子都需可追踪。建议根据脚本读写 footprint 仅复制相关地图块/实体，越界访问则扩展快照或拒绝执行。验收使用真实断言，例如在观察窗口内至少两次刷怪、两个生成坐标接近传送门、间隔落在容差内、强敌 `MaxHealth/ContactDamage/MoveSpeed` 达标，并且下一个新刷出的敌人也符合强敌 profile；不能只因 AST 可解析或传送门对象存在就放行。

Unity 当前 `GameWorld.Instance` 可以继续作为唯一真实关卡协调者，但 `BlastScheduler`、敌人生成/查询和原语解释器必须显式接收 `WorldContext`。不创建第二个 `GameWorld` 或另一套完整地图；只在镜头外装配试炼舱及分区上下文。现有全场 `FindObjectsByType` 必须换成按 context/partition 查询，否则 sandbox 敌人会污染真实敌人数、击杀、AI 目标和验收证据。模型/沙盒只能拿到 Sandbox context 的写 API；Live executor 是唯一能更改玩家区域的入口。

沙盒通过只表示“此脚本在这个快照和模拟条件下通过预演”，不保证后续实机因敌人行动、玩家干预或随机事件而永不失败。因此要保留**预演门**和**实机验收/撤销账本**两道边界。用户输入 `<作弊码>` 已明确允许相应操作时，沙盒所有条件通过后可自动提交，不额外增加确认对话；UI 应显示预演结果、进入实机步骤、实际 pass/fail 和可撤销状态。

这比要求所有模拟效果从副本逐对象复制到实机更容易保证一致性，也与 Agent Harness 的“在边界环境测试 → 检查结果 → 将经过审查的工件提交正式环境”相似；Codex worktree/隔离命令本身不会自动证明业务逻辑正确，游戏侧仍要有特定断言和真实世界复核。[Codex 安全运行说明](https://openai.com/index/running-codex-safely/)、[Pi Agent Loop](https://github.com/earendil-works/pi/blob/main/packages/agent/src/agent-loop.ts)

## 核心代码路径

### 1. Agent Loop 把模型回合和工具回合拆开

`runAgentLoop` 发出 `agent_start`、`turn_start` 等事件，进入循环调用模型，并处理 assistant 消息。若消息含工具调用，循环会先执行工具，再把类型化 `toolResult` 放回上下文，随后才决定结束、继续、处理 steering/follow-up 消息或再问模型。代码还明确处理被 token 长度截断的工具调用：参数可能不完整时，整批调用报错，不执行工具。

这套拆分回答了一个关键问题：模型说要做事、工具开始做事、工具实际完成，是三种不同事实。调用 ID、工具名、参数、结果和错误沿事件传递，调用方可以观察生命周期，模型下一轮也能看到工具真实返回值。源码中先校验调用结构和参数，再运行 `beforeToolCall` 策略钩子；工具完成后由 `afterToolCall` 收尾，将成功或失败结果包装成工具消息放回上下文，再进入后续模型回合。截断或无效参数不会进入执行。参考 [Agent Loop：回合主循环与工具执行](https://github.com/earendil-works/pi/blob/main/packages/agent/src/agent-loop.ts)。

### 2. 工具接口是显式 schema 加执行函数

Pi 扩展工具声明名称、给模型看的说明、TypeBox 参数 schema 和 `execute()`；执行结果区分面向模型的 `content` 与给界面/状态恢复使用的 `details`。源码将工具准备、实际执行、执行后处理和结果事件分层；执行错误也会转成失败结果。共享可变状态的工具应顺序执行；同一 assistant 消息中的调用只有在彼此独立时才适合并行。

这使“模型能提出什么”和“宿主实际能执行什么”至少有一份可以检查的接口定义。但 schema 主要约束形状和类型，不能证明参数在当前世界里合理，因此游戏仍需规则层复核目标、版本、资源、距离和副作用范围。参考 [Pi 扩展文档：工具与事件](https://github.com/earendil-works/pi/blob/main/packages/coding-agent/docs/extensions.md) 及 [Agent Loop：工具调用准备、执行和收尾](https://github.com/earendil-works/pi/blob/main/packages/agent/src/agent-loop.ts)。

### 3. 事件钩子是生命周期边界，不是权限沙箱

Pi 提供 agent、message、tool 等阶段的事件和扩展点，执行前后可观察或改变允许的结果。扩展运行于 Pi 进程并继承该进程的操作系统权限；隔离要由容器、虚拟机或宿主策略落实。只把部分内置工具放进沙箱，不能自动约束仍在宿主机运行的扩展。

因此游戏不能因为“操作经过 Agent Harness”就假设安全。可执行能力必须由 Unity 的固定候选表、状态校验、主线程执行器和范围上限落实。参考 [Pi 扩展文档：生命周期和权限](https://github.com/earendil-works/pi/blob/main/packages/coding-agent/docs/extensions.md) 与 [Pi 容器化文档：隔离边界](https://github.com/earendil-works/pi/blob/main/packages/coding-agent/docs/containerization.md)。

### 4. 对话摘要不等于原始状态

Pi 会为长会话建立结构化压缩条目，记录摘要、保留边界和 token 数；它还允许扩展自定义压缩过程。摘要用于恢复模型工作上下文，原始会话记录仍与摘要条目分开保存。

源码将压缩策略作为可单独检查的计算逻辑：它按 token 预算选择需要摘要的旧消息，切分时保留近期消息尾部，并把摘要/压缩边界作为独立会话记录；设置文档还暴露自动压缩的保留预算。游戏状态的权威来源更严格：地形格子、角色生命、能量、任务队列和操作结果只能来自 Unity 运行时对象。摘要只能帮助模型阅读；Unity 必须从实时状态重新生成候选与前置条件。参考 [Pi Compaction 源码](https://github.com/earendil-works/pi/blob/main/packages/coding-agent/src/core/compaction/compaction.ts)、[Pi Compaction 文档](https://github.com/earendil-works/pi/blob/main/packages/coding-agent/docs/compaction.md) 和 [Pi 设置文档](https://github.com/earendil-works/pi/blob/main/packages/coding-agent/docs/settings.md)。

### 5. 本次复核重点：实际工具调用事件顺序

源码路径为：assistant 消息结束后循环检查工具调用结构和 arguments；策略钩子可拒绝执行；允许的调用才交给工具函数；执行后的内容/错误被包装成 `toolResult` 回到上下文；随后循环继续，让模型读取真实结果并决定结束或提出后续调用。工具调用的并行只适用于明确声明可并行的工作，依赖共享状态的写操作应串行。`afterToolCall` 是结果收尾入口，不能替代工具自身的参数和业务校验。

本项目 `/decide` 每轮只返回一个封闭候选，不暴露 Pi 的通用工具调用 API。现在借鉴“每步回执回到下一轮”的控制流，在 Unity 主控的最多10步循环中重新建快照、重新评分；不允许无界循环或并行写操作。游戏完成判据来自原语专属引擎事实，不来自模型的自然语言自述。

## 映射到 Ember Hollow

| Pi 的结构 | 本项目对应实现 | 设计理由与差异 |
| --- | --- | --- |
| 模型回合、工具调用、工具结果和下一回合有清楚边界 | `GodActionHarness` 的分步候选评分、复核、执行和 Unity 验收回执 | 评分通过不等于操作通过；每步生成有动作 ID、接受状态和引擎证据的回执，下一轮评分只接收该回执与新的 Unity 快照。 |
| 工具 schema 和已注册的 `execute()` | `GodDecisionOption` 封闭候选 ID；F3 固定组合和创世原语分别由 Unity 执行 | 模型不提供 C# 方法名、`ModuleKind[]`、世界坐标、伤害或爆破半径。Unity 自己计算方向、目标位置和费用。 |
| 执行前准备与执行后结果钩子 | `GodActionHarness` 捕获快照后重验，施放后核对实际游戏状态 | 检查同一个目标对象/ID、位置、生命、玩家状态、能量、`RunToken`、队列和覆盖层，防止模型等待期间的过期决定产生副作用。 |
| 顺序执行共享可变状态的工具 | 模型推理后台运行；评分期间暂停模拟；通过 Unity 主线程触碰游戏对象 | Tilemap、Physics2D、角色和施法器具有单一线程/单一写入者边界，不让 Python 或推理线程直接控制它们。 |
| 工具失败或取消有显式结果 | `GodActionHarness` 的步骤回执、超时、X 取消和任务结束状态 | 创世请求最多10步；已执行动作不会重放。中断/安全条件不满足时记录部分完成并停止，不做无界重试。 |
| 对话上下文与持久 session | 玩家聊天历史和创世 task 分开；同一创世 task 的回执只保存在当前 Unity 进程 | 玩家聊天内容不会成为通用动作指令；重开/崩溃恢复和跨会话 task 持久化当前未实现。 |

当前主线流程：

```text
Unity 检查并生成有限候选
  → SemIf direct_messages 构造规范提示，llama.cpp /completion 返回候选概率
  → Unity 验证 request ID、候选顺序、分数和当前快照
  → 固定登记组合在主线程执行
  → Unity 观察目标伤害、爆破计数和剩余工作
  → 向玩家报告已观测到的成功或失败
```

创世任务单独走有界多步流程：

```text
按玩家明确文本构造本任务的有限候选 ID
  → 模型每轮只选一个尚未尝试的候选
  → Unity 重验快照并执行这个登记原语
  → 用原语专属完成条件生成结构化验收回执
  → 若仍有候选且队列清空，将回执与新 Unity 快照送入下一轮评分
  → 最多10步；完成、暂缓、失败、超时、取消或安全门不满足时停止并报告部分/完整结果
```

当前 `direct_messages` 构造评分提示，llama.cpp `/completion` 取候选 token 概率并归一化；它不是另外训练的防幻觉头，也不是判断正确性的证明。候选集合封闭后，即使模型选错，也只能选错已有选项；是否允许产生副作用仍由 Unity 规则决定。

## 从源码提炼的设计原则

1. **把意图、调用和结果分开建模。** 每个阶段用明确状态和结果字段表达；不要用模型的一句自然语言同时代表计划、授权、执行成功。
2. **结构约束和规则校验各司其职。** schema/候选 ID 限制模型能表达的动作；业务规则负责检查当前世界是否允许动作。
3. **把共享状态的操作串行化。** 对会改变角色、地形、能量或队列的行为，在 Unity 主线程通过一个权威执行入口完成。
4. **错误必须变成可观察的结果。** 参数过期、资源不足、目标失效、执行异常、超时和真实动作失败分别报告，不把“请求已发出”显示成“操作已成功”。
5. **异步等待期间重新确认前置条件。** 候选生成时的状态可能过期；执行前重验捕获的对象和版本，状态不匹配则零副作用。
6. **把事实和摘要分开。** 模型上下文可被裁剪或总结，Unity 世界状态不可由总结回填或覆盖。
7. **权限需要由宿主兑现。** system prompt、候选 schema、Pi 风格生命周期或模型自律都不是安全边界；能做什么由游戏端注册和规则验证。

## 不引入的 Pi 能力

- 不把 shell、文件、终端、任意代码或通用工具转成神谕权限。
- 不让模型自由发出任意工具名、函数名、法术模块列表或数值参数。
- 不把 Pi 作为游戏 runtime、推理后端或 Unity 插件依赖。
- 不照搬其实现代码、提示词、产品 UI 或扩展实现；只将公开代码提炼为本项目自己的接口与测试要求。

## 下一步架构门槛

每个原语应交付：稳定 ID、Unity 强类型实现、游戏状态前置条件、资源/范围上限、幂等策略、取消与重开语义、可查询的完成/失败事件，以及针对过期快照和无效候选的零副作用测试。当前创世 Harness 已有 task/request ID、总期限、最多10步、取消与结构化回执；仍没有持久化 action journal、世界状态版本戳、崩溃恢复、通用工具注册表或自动重试。新黑洞原语已有专属地形/实体快照撤销，填充和试炼生物尚无独立撤销 ID。不要把有界 re-evaluation 误称为通用 Agent Harness，也不扩张模型自由调用面。

## 参考资料

- [Pi Agent Loop 源码](https://github.com/earendil-works/pi/blob/main/packages/agent/src/agent-loop.ts)
- [Pi 扩展文档](https://github.com/earendil-works/pi/blob/main/packages/coding-agent/docs/extensions.md)
- [Pi Compaction 文档](https://github.com/earendil-works/pi/blob/main/packages/coding-agent/docs/compaction.md)
- [Pi Compaction 源码](https://github.com/earendil-works/pi/blob/main/packages/coding-agent/src/core/compaction/compaction.ts)
- [Pi 设置文档](https://github.com/earendil-works/pi/blob/main/packages/coding-agent/docs/settings.md)
- [Pi 容器化文档](https://github.com/earendil-works/pi/blob/main/packages/coding-agent/docs/containerization.md)

