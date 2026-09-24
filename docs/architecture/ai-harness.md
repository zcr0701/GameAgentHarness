# 神谕游戏操作 Harness

更新时间：2026-09-25。本文是本地模型与 Unity 之间的安全和协作契约。生产推理由 `GodDialogueServer.py` 加载本地 tokenizer，并调用 llama.cpp Qwen3.5-4B Q4_K_M GGUF；模型可以比较引擎提供的有限选项，但不能直接调用 Unity API。所有可选操作及上限见[游戏操作原语目录](action-primitives.md)。

## 当前模型评分链与 SemIf 的边界

当前 SemIf `direct_messages` 用于构造规范决策提示；服务端将提示交给 llama.cpp `/completion`，读取单 token 候选概率，仅对 Unity 声明且响应中有概率覆盖的候选进行归一化。候选覆盖不足、响应无效或概率非法时失败关闭。这不是单独训练的“防幻觉输出头”，也不是语义正确性证明；最终选择来自有限候选，不采样自由文本作为命令。此前 Transformers NF4 + SemIf `direct.score` 是历史生产基线，不能描述为当前实现。

性能参数属于推理适配层，不改变 Unity 的命令边界。当前聊天请求使用 Q4_K_M、`ngl=5`、`ctx=4096`、`parallel=1`、`threads=8`、`flash-attn=auto`，`cache_prompt=false`；`/decide` 也不复用 prompt cache，并仅生成一个候选槽 token。线程扫描里 8 线程的 96-token 贪心 decode 中位数为 14.61 token/s，高于 4/12 线程。聊天前缀缓存能降低多轮 prompt 时间，但当前 Qwen3.5 + llama.cpp 组合的贪心同输入回归未完全一致，因此禁用，直到原因和输出质量得到解释。`ngram-mod` 的局部吞吐收益约 3.4%，证据不足，不打开。完整条件和数值见[神谕本地对话服务](../GOD_DIALOGUE.md)。

这种选择把操作空间封闭在 Unity 预先给出的选项里，减少了任意命令、坐标与参数幻觉造成的执行风险；它不能保证选中的游戏判断正确，返回分数也未校准为决策置信度。因此 Unity 执行器必须重新检查事实、前置条件和资源。即使模型选择错误，也只能从当前有限候选中选错，而不能自创操作。

## 文件与职责

- `Assets/Scripts/GodDialogueService.cs`：启动/监测/结束本地 sidecar；异步执行聊天、旧消息摘要或结构化评分；将工作线程结果放入主线程队列；验证决策返回的 request ID、候选顺序、有限且归一化的概率，以及选项 ID 是否与最大分数一致。
- `Assets/StreamingAssets/LocalAI/GodDialogueServer.py`：仅监听 `127.0.0.1`；加载本地 tokenizer，并调用 llama.cpp Qwen3.5-4B Q4_K_M GGUF；`POST /chat` 生成只读对话，`POST /summarize` 压缩旧对话，`POST /decide` 使用 SemIf `direct_messages` 构造规范提示，再通过 llama.cpp `/completion` 读取单 token 候选概率并归一化；没有 Unity API 或动作执行器。
- `Assets/Scripts/GodActionHarness.cs`：生成候选、捕获权威快照、暂停状态、调用评分，复核并触发登记操作，再观察真实游戏结果；F3 组合与 `<作弊码>` 创世入口走分开的候选构造路径。
- `Assets/Scripts/GameHud.cs`：F3 与点击“神谕组合”是玩家入口，只显示 Harness 返回的双语状态。
- `Assets/Scripts/ProgramCaster.cs`：含 Unity 内定义的 `FaultlineChargeSequence` 和执行接口；它不从模型读取模块序列。
- `Assets/Scripts/BlastScheduler.cs`、`EnemyChaser.cs`：进行实际爆破、目标伤害、地形工作和反馈。

## 聊天上下文自动压缩

聊天历史和 Harness 的权威局面状态分开管理。待发送聊天超过 12 条消息时，`GodDialogueService` 将最早的完整问答轮次移交 `/summarize`，最多每批 4 条，并要求本地模型最多返回 96 token 的压缩记忆；每批串行归纳，下一批会收到前一版记忆。随后 Unity 只从聊天上下文移除已归档的消息，保留最近 8 条原文，再将压缩记忆与当前主线程构建的世界快照放进同一条聊天 system 消息。

压缩记忆明确标为可能不完整的旧对话背景。服务端不会把它交给 SemIf `/decide`，也不会将其当成敌人数、生命、位置或资源的权威事实。聊天超限仍按 4096-token 服务端边界报错，不做静默截断。取消会打断活动的摘要或流式回复；摘要失败时本轮改用最近 12 条原始消息，保留原历史供下轮重试。新一局清空摘要。`-contextCompactionSmokeTest` 用本机真实模型验证分批摘要、保留窗口和后续聊天。

## 当前闭环

```text
玩家按 F3 / 点击神谕组合，或在对话开头输入 <作弊码>
  → Unity 检查胜负、暂停层、模型就绪、最近存活目标、范围、能量与空队列
  → 捕获本局请求 ID、目标对象/位置/生命、玩家位置/生命、能量、RunToken、队列/爆破计数
  → 暂停模拟，按入口构造候选（F3: faultline_charge/hold；创世: 当前登记子目标）
  → SemIf direct_messages 构造规范提示；llama.cpp /completion 返回单 token 候选概率
  → Unity 验证响应属于原候选且与 argmax 分数一致，再次检查权威快照
     ├─ F3：执行固定 Faultline Charge → 等待命中 → 验收锁定目标伤害/爆破/队列
     └─ <作弊码>：编译登记 HarnessScriptStep/hash → Sandbox 数据投影预演
             → 通过后执行同一 step → 验收真实局面 → 以新快照进入下一步
  → HUD 提示成功/暂缓/拒绝/未命中；控制台记录 request ID、选项和验收证据
```

F3 当前只提供 `faultline_charge` 与不执行的 `hold`。聊天开头明确含 `<作弊码>` 时，Unity 根据请求文本筛出一组已注册的创世候选，再走相同的受限概率评分链；当前目录包括地图爆破、刷怪门、冲击波、回响风暴、无限能量和钻脉。所有参数仍由 Unity 固定，模型响应不包含坐标、`ModuleKind` 数组、伤害/半径/能源字段。创世入口支持 Unity 主控的最多 3 步有界流程，不是通用工具调用或自由构造计划；确切行为和限制见原语目录。

## 受 Pi 公开 Agent Harness 启发的做法

完整的代码路径阅读与原则映射见 [Pi Agent Harness 代码研究](../PI_AGENT_HARNESS_RESEARCH.md)。本节只记录本模块的直接应用。

Pi 是架构研究对象，不是本项目的运行时依赖。参考的是控制流与执行记录方式，而非照搬其编码工具：

1. **清楚的操作阶段**：区分快照/评分、执行前复核、执行中、结果观察和结束状态；模型返回 ID 不等于游戏操作完成。Pi 的循环将 assistant 回复、tool start/result 与下一次循环作为不同事件处理。[Pi Agent Loop](https://github.com/earendil-works/pi/blob/main/packages/agent/src/agent-loop.ts)
2. **类型化工具与参数验证**：Pi 先把可用工具交给模型并校验调用参数；游戏也先构造有限的合法候选，然后独立验证目标、资源与 snapshot。当前功能只有固定候选 ID，没有任意 schema / 参数执行 API。[Pi Agent Loop](https://github.com/earendil-works/pi/blob/main/packages/agent/src/agent-loop.ts)
3. **执行前后边界**：Pi 的 agent 流程与扩展点让工具执行可被拒绝、失败并回传。游戏将决定性的规则放在 `GodActionHarness.Revalidate` 和 `ProgramCaster.TryCastFaultlineCharge`，而不是依赖 system prompt 或模型自律。[Pi Extensions](https://github.com/earendil-works/pi/blob/main/packages/coding-agent/docs/extensions.md)
4. **以真实状态为准**：Pi 支持长对话压缩；游戏如果需要跨操作摘要，也应从 Unity 当前状态重建，绝不将模型摘要当作事实来源。[Pi Compaction](https://github.com/earendil-works/pi/blob/main/packages/coding-agent/docs/compaction.md)
5. **不要借用宿主沙箱假设**：Pi 的容器化文档说明权限隔离来自宿主运行环境；本游戏权限必须由 Unity 的候选表与执行器落实。[Pi Containerization](https://github.com/earendil-works/pi/blob/main/packages/coding-agent/docs/containerization.md)
6. **实际结果反馈**：每个创世动作使用自己定义的 Unity 验收事实，产生 `action_id / accepted / evidence` 回执。若还有未尝试候选且法术/地形队列为空，Unity 捕获全新快照并将回执交给下一轮评分；已尝试操作不会重放。模型可以暂缓或结束，但只要仍有未验收的登记子目标，整项任务就按部分完成报告。

## 创世多步验收时序

```text
玩家以 <作弊码> 提交明确目标
  → Unity 文本规则生成最多 6 个封闭候选 ID，并保存原始目标与 150 秒 task deadline
  → 每轮暂停游戏，由模型只从本轮候选中选择一个原语
  → Unity 对玩家对象/位置/生命、RunToken、地形队列、UI 与服务状态重新校验
  → 生成不可变参数 step/hash，在 sandbox 数据投影上预演并检查本步断言
  → 通过后主线程执行相同 step，再观测实机断言；记录 sandbox/live 双证据
  → 若候选仍未尝试、最多 3 步且没有活动弹体/地形工作，则用新世界快照继续评分
  → 失败、hold、finish_task、超时、取消、游戏结束或安全门不通过都停止本任务
  → 汇总每一步回执；全部候选尝试且所有验收通过才把任务记为通过
```

`finish_task` 只能在后续评分轮选择，当前实现仍将未尝试候选视为未完成目标，因此此时整项任务报告部分完成；不让模型单方面把待做子目标宣称为完成。`X` 会取消创世 task、忽略迟到的模型结果、停止当前由 Caster 管理的链，并关闭传送门/无限能量持续效果。

## 目标架构：从当前数据预演升级到分区试炼舱

### 当前已集成的第一阶段

地图高处路线下方现在有一间封闭的地下房间。玩家挖开岩顶进入后，`EasterEggRoomTrigger` 只接受真实玩家，首次进入解锁本地持久化隐藏成就“神的房间？/God's Room?”；`J` 打开普通/隐藏成就列表，未解锁的隐藏项显示为问号。首次胜利是普通成就。`AchievementService` 目前用 `PlayerPrefs` 保存，不代表已经接入 Steam API。

AI 预演由 `HarnessSandboxChamber` 持有，和真实世界共处一个 Unity Scene 与唯一 `GameWorld`。每个任务获得独立的 `oracle-sandbox:<task>` 逻辑上下文；它从权威快照复制本次可检验的状态数据，在隔离投影上模拟已注册原语，不创建或修改真实敌人、地形、法术队列，也不参与胜负统计。地下房间供玩家发现成就，逻辑试炼上下文只供 Harness 使用，两者没有共享触发权限。这个第一阶段**不是一套独立 Unity 物理场景**：目前没有第二份运行中 Tilemap、敌人对象或物理调度器。

当前创世步骤按此顺序运行：

```text
<作弊码>目标
  → Unity 生成有限候选 ID、捕获权威快照和 StateRevision
  → SemIf / 本地模型只为本轮候选打分；评分时暂停游戏
  → Unity 重验快照，为已登记原语编译不可变 HarnessScriptStep 并生成 SHA-256
  → oracle-sandbox:<task> 基于独立数据投影运行预演断言
     ├─ 失败：丢弃 staged step，本步不触碰玩家世界
     └─ 通过：版本仍匹配时，把同一个不可变 step 交给 Unity 实机执行器
             → 观察实时游戏结果，保存 sandbox/live 双回执
             → 捕获新快照，再决定后续步骤
```

对于“无穷无尽的强大敌人”，Unity 先只提供 `endless_enemy_portal`；实机确认门户存在且至少一次刷新成功后，下一轮才提供 `set_portal_strong_profile`。第二步调整门户后，必须观察新生成敌人并检查生命、接触伤害、移速和刷新间隔。任一断言失败，目标不从剩余列表移除，最终任务不能报通过。现阶段仍是固定 ID 和 Unity 固定参数驱动的最多 3 步流程，并非模型生成多条任意脚本。

`StateRevision` 目前只在本次运行中检查快照新鲜度；每步有参数规范串、hash 和验收回执，但没有持久化 JSONL journal 或崩溃恢复。门户/无限能量等少数持续效果能关闭，地图挖掘、爆破和敌人受伤还没有完整逆操作；因此现状**不具备所有动作可撤销的保证**。以下可撤销准入规则是后续 DSL 的硬门槛。

根据游戏设计要求，Harness 在普通验收之外将增加一个位于同一 Unity Scene、摄像机视野外的 `SandboxChamber`。它不是第二个完整关卡或第二个 `GameWorld`，而是共享正式原语实现的隔离空间，包含专属地形投影/材料数据、临时实体和独立资源上下文。神谕先在其中执行脚本并检查断言；失败则清空临时分区，将真实失败证据回交模型修复，重试次数有上限，实机保持不变。

```text
自然语言目标 → 安全头生成受限 DSL → AST/类型/预算预检
  → 隐藏试炼舱运行与断言
     ├─ 失败：日志/状态差异回灌，修订脚本后再试；实机零副作用
     └─ 通过：冻结脚本与快照版本 → 检查真实世界是否仍匹配
             → 同一解释器在实机执行 → 实时验收 + 可撤销 journal
             → 条件变化或实机失败时拒绝/回滚，记录原因
```

测试期间锁住玩家输入，并通过 `SimulationGate` 暂停真实玩家、敌人 AI、沙粒、门户、施法与爆破工作；只推进 SandboxPartition。试炼舱的地形使用独立数据/Tilemap，实体和队列通过 `WorldContext` 查询，不能写入玩家地形，也不能参与真实胜负统计、存档或 HUD。布局在地图边界外/镜头不可达处，与实机保持大于全部原语影响半径的缓冲距离；渲染器默认不进入玩家摄像机，碰撞与射线查询按分区过滤。

试炼舱通过后只把不可变 DSL artifact（脚本 ID/hash、参数、断言、输入世界版本）提交给实机执行器，不搬运被测试生成的 Unity 对象或修改后的地形。实机再次运行同一脚本、实时观察结果并写入撤销记录，因为预演快照与正式交互之间可能发生状态差异。若实机快照版本不匹配则重测/重规划；若提交后中途失败则按 journal 撤销已完成步骤。用户 `<作弊码>` 是明确执行授权，sandbox 全部断言通过即可自动进入实机，不另加确认门。

目标中的独立地形 Tilemap、敌人与调度队列、正式 `WorldContext` 分区查询、实机 `SimulationGate`、共享解释器以及状态变化回滚目前尚未实现。当前数据投影足以在真正副作用前拒绝参数和前置条件不成立的固定原语，但不能模拟完整物理相互作用；其预演证据必须与玩家世界的实时验收证据分开记录。继续保留唯一 `GameWorld.Instance`，不创建第二个完整关卡。

## 目标脚本语言：受约束的 Python 语法子集

用户要求直接采用模型训练语料充分的 Python 语法来提高准确率和一次通过率。本项目实现一个**Python 语法子集**，用项目专属 DSL parser 解析这一子集；不加载 CPython、不执行任意 Python 源代码。游戏侧 registry 只登记可调用的游戏操作、效果模块适配器和权威状态查询；`def`、赋值、`if/else`、`for`、`while`、`assert` 等属于脚本语言本身的语法/运行时内建，不是游戏原语，也不混进游戏动作表。

首版 grammar 限于 `def`、局部赋值、游戏 registry API 调用、有限字面量、`if/else`、`for i in range(n)`、有界 `while`、`assert` 和 `return`。表达式只开放局部变量、注册 API 返回值、常量、布尔逻辑、指定比较（如 `is None`、`<`、`==`）和受限整数计数；拒绝 `import`、反射、动态函数名、任意属性访问/表达式、文件/网络/进程调用、递归、任意 eval。所有结构解析成有类型的 AST；游戏原语调用名必须来自当前任务 registry，参数 schema、对象引用、先决条件、执行预算和授权由 Unity 校验。循环与条件由解释器按语言语义执行并由 Harness 施加静态预算，不需要作为游戏原语注册。游戏函数 ID 未登记、参数类型不符、脚本截断、越权或预算超限一律失败关闭并零副作用。

“无限敌人”在游戏中表示一个**有界持续效果**，例如定时传送门、刷新速率上限、同屏存活上限和明确取消口，不能解释为无上限 CPU 循环或无限对象。期望的脚本表达示意如下，**不是当前模型生成格式或已实现的 DSL 语法**：

```python
def create_strong_endless_enemies():
    portal = find_active_portal()
    if portal is None:
        portal = create_portal(interval=3.0, alive_limit=8)
    assert portal_exists(portal)
    for attempt in range(2):
        wait(seconds=3.5)
        assert spawned_near(portal, profile="standard")
    set_portal_profile(portal, profile=STRONG)
    attempt = 0
    while attempt < 2:
        wait(seconds=3.5)
        assert spawned_near(portal, profile=STRONG)
        assert nearby_enemies_meet_profile(portal, health_at_least=12, damage_at_least=2)
        attempt += 1
    return accept(portal)
```

`wait`、`assert`、`accept` 是 Harness 运行时内建，不是游戏操作原语；`while` 必须同时有 Unity 验证的最大迭代数、deadline 和取消令牌。loop 次数、等待时长、嵌套深度、AST 节点数、总运行预算都由引擎硬性限制。模型可以在断言失败后根据真实证据选择登记的修正步骤，但最多重试若干次，不能让 Agent Harness 自行无限扩张。

### SemIf 只约束游戏原语调用点

不把 SemIf 用成对脚本每个 token 反复做单 token 候选评分。Python 语法由模型按熟悉的语言模式生成；只在模型插入“游戏能调用什么操作/查询/效果模块”时，让 SemIf 受限输出头把函数 ID 限定到本任务允许的 registry 项。游戏调用的枚举参数可同样来自有类型的有限表。循环、条件、函数定义和局部变量属于语言语法，不送入游戏原语 registry，也不消耗动作候选评分回合。

当前生产 `direct_messages` 只构造规范提示；已有 `/decide` 用 llama.cpp 对一个固定候选槽做单 token 评分，是 F3/旧创世入口的实现，**不会扩展为给 Python 脚本逐 token 评分**，也不是 SemIf 的原语约束头。后续须验证 SemIf 能在游戏函数调用点按本任务 registry 限定 ID，并与 llama.cpp tokenizer/解码接口对齐。随后 Unity 仍独立解析整份脚本，校验语法、类型、参数范围、前置条件、预算、世界版本和权限；合法的函数名不代表游戏操作合理。支持不足时应停止 DSL 生成或沿用单动作入口，不能把旧评分接口冒称新输出头。

### 可撤销操作准入

每个副作用原语必须登记 `UndoPolicy`、逆操作或补偿操作、可恢复的数据范围、幂等性、取消和失败语义。每个脚本节点至少记录 `task_id / node_id / primitive_id / typed_args / before_revision / after_revision / status / evidence / undo_token`；撤销按已提交节点的逆序运行，验证对象 ID 和版本仍匹配。回滚不能恢复完整状态时，必须报告未恢复节点，不得声称全部撤销。

例如关闭新建传送门并清理由其生成的敌人、恢复门户此前的 profile、关闭持续无限能量都要有对应 undo。地形挖掘要保存受影响格子的旧材料并恢复 Tilemap/碰撞；爆破/受伤目标要提供可重建的敌人与状态快照，或明确拒绝加入“可撤销脚本”候选集。**没有可运行且通过验证的 undo handler 的副作用原语，不能进入可撤销脚本 registry。**

### 原语验收事实

| 原语 | 通过条件（Unity 事实） |
| --- | --- |
| `demolish_map` | 完成全覆盖爆破；可破坏格归零；基岩计数不变。 |
| `endless_enemy_portal` | 门保持开启，且新门至少实际刷新一次；复用已有门时必须有历史刷新记录。 |
| `set_portal_strong_profile` | 门仍开启；至少一次后续刷新成功；门户所属敌人满足强敌 profile 的生命/接触伤害/移速阈值；新刷敌人计数与刷新间隔成立。 |
| `infinite_energy` | `ProgramCaster.CheatInfiniteEnergyEnabled == true`。 |
| `shockwave` | 爆破完成计数增加，队列与地形工作清空。 |
| `echo_storm` | 新增至少两次连锁爆破，RunToken 未变，链条仍有活动工作。 |
| `drill_bore` | 脉冲结束、爆破计数增加、可破坏地形减少、工作队列清空。 |

这些证据是操作完成与反馈依据，不是对隐藏思维链的展示。目标态下还会并列显示“试炼舱预演证据”与“玩家世界实机证据”；前者不能替代后者。玩家可见信息是计划/复核阶段、正在执行的步骤、实际验收摘要与停止原因。

## 关键安全和一致性规则

1. 模型进程只访问普通 JSON 请求和分数；它不加载 Unity DLL，不持有对象引用，不写文件或地形。
2. Python 侧选项数、ID 长度、描述、问题、状态和总请求体都有上限；SemIf `validate_row` 再校验其评分协议。
3. `GodDialogueService` 的线程只做 HTTP/CPU-GPU 推理；Unity 对象只由主线程 Harness 和游戏系统使用。
4. F3 只能在游戏活动且模型 `Ready` 时开始；决策中暂停模拟，阻止打开设置、编辑器、聊天或重开。
5. 动作被选中后，Unity 根据相同快照复核目标引用/ID、生命和位置、玩家生命与位置、能量、`RunToken`、队列、最大距离、关卡状态与覆盖层。任一项变化则零动作并恢复游戏。
6. 玩家在模型评分过程中不能移动或发射。已选目标在组合弹体命中前不追逐移动，否则弹体飞行延迟会让“锁定目标”变成过期世界事实。
7. 爆破完成以 Unity 观测为准：至少一发新爆破、锁定目标生命下降/目标被击败，且队列为空。模型自报成功不计为通过。
8. llama.cpp 返回的条件候选概率不是校准决策置信度；候选缺失覆盖时评分失败关闭，当前不以固定分数阈值作为授权。
9. 对话文本与游戏操作评分协议分开。自然语言聊天永远不经过命令提取或动作执行器。

## 扩展步骤

若增加操作：

1. 先由 Unity 给操作定义不可变 ID、确定性的实现与资源/范围上限。
2. 写该操作的前置条件、快照字段、取消/失效语义、实际完成信号和双语状态。
3. 只把人类可解释的有限候选及必要证据交给 `RequestDecision`。任何自由参数都应由 Unity 自己选择或在 Unity 规则层约束。
4. 新增执行前后验证，并证明未选中的选项没有副作用、无效响应零执行、状态版本过期零执行。
5. 对玩家进程加独立测试：真实本地评分、合法候选、重验证、实际游戏结果、玩家程序存档不变。
6. 再同步本文、[游戏操作原语目录](action-primitives.md)、模块索引、`DESIGN.md` 与 `AGENTS.md`。不要先向模型开放可任意组合模块、坐标、数值或 C# 方法的工具。

当前有运行期 `StateRevision` 和任务内 script hash/步骤回执，但没有持久化 JSONL action journal、崩溃恢复、动态通用 tool registry、完整物理分区、整段多步 DSL 预演或通用 undo 栈。创世循环严格受 3 步和 150 秒限制。`requestSequence`、步骤回执与 HUD 状态只服务当前会话。版本化持久化须保持 Unity 数据为事实来源，并先补齐逆操作协议。

## 验证清单

- Python 无需修改外部 SemIf 仓库：用 SemIf 虚拟环境做 `py_compile`，检查合法与非法候选行，玩家端通过 `/decide` 调用真实模型。
- Windows Unity 构建无错误。
- 普通 `-smokeTest` 通过，并确认没有模型依赖的冒烟检查仍可离线运行。
- Unity Windows 构建成功；standalone 服务的 `/chat`、`/decide`、`/cancel`、`/shutdown` 检查通过。
- 实际玩家进程 `-smokeTest -modelSmokeTest -actionHarnessSmokeTest` 输出 `GOD_ACTION_HARNESS_SMOKE_PASS`，选择 `faultline_charge` 并命中锁定目标。
- 实际玩家进程 `-smokeTest -modelSmokeTest -creationActionChainSmokeTest` 验证无限能量与刷怪门组成的双步骤请求、每一步的引擎回执、刷怪门真实刷新和任务级最终验收。
- 新增 Harness 沙盒隔离检查：预演失败时真实状态零变化；过期 revision 拒绝；强敌 profile 缺少活动传送门时 fail closed。
- `-creationStrongPortalSmokeTest` 用真实 Q4_K_M 模型完成强敌目标链：第一步门户刷新通过、第二步强敌配置和下一次刷新通过；最终观察 2 只门户敌人均为强敌，最小生命 12、接触伤害 2、移速 3.2、刷新间隔 3 秒，双步骤 hash 与 sandbox/live 回执齐全。
- 隐藏成就检查首次玩家进入、普通对象不触发、J 面板分类/隐藏问号、英语切换与本地存档跨重启；当前代码尚待 Unity 实机验收。
- 当前运行参数为 `ngl=5`、`ctx=4096`、`parallel=1`、`flash-attn=auto`，本次实际运行额外显存约 0.8 GiB；此为已验证配置记录，不等同于通用硬件性能承诺。
- 另做正常 `-smokeTest -modelSmokeTest` 检查聊天路径，保证文本生成和决策端点共享模型锁但协议不同。
- `-smokeTest -contextCompactionSmokeTest` 验证真实模型自动整理旧消息、只留最近 8 条原文，并完成后续 `/chat` 与普通游戏 smoke。
- `-harnessSandboxSmokeTest` 验证密室结构、隐藏成就触发与持久化、`J` 成就页暂停/恢复和英文标签、隔离数据预演、强敌缺少传送门前置条件时拒绝、过期 revision 拒绝和实机状态不变。该检查已在最新 Windows 构建上通过；实际碰撞和窗口视觉布局仍需人工试玩检查。
- 人工按 F3 并观察暂停/恢复、目标命中、暂缓/错误状态和中英文 HUD。模型端分数不是胜率指标。

## 参考链接

- [Pi Agent Loop 源码](https://github.com/earendil-works/pi/blob/main/packages/agent/src/agent-loop.ts)
- [Pi Agent 扩展文档](https://github.com/earendil-works/pi/blob/main/packages/coding-agent/docs/extensions.md)
- [Pi 上下文压缩文档](https://github.com/earendil-works/pi/blob/main/packages/coding-agent/docs/compaction.md)
- [Pi 容器化和权限边界说明](https://github.com/earendil-works/pi/blob/main/packages/coding-agent/docs/containerization.md)
