# 游戏操作原语目录

更新时间：2026-09-25。本文区分玩家可编排的法术模块与神谕可调用的游戏操作；“原语”在两层中的权限和参数含义不同。当前暗语 Harness 逐步选择已登记 ID，没有执行任意 Python 脚本。

## 两层指令系统

| 层 | 谁在编排 | 输入形式 | 能改变什么 | 权限边界 |
| --- | --- | --- | --- | --- |
| 法术模块 | 玩家 | `ProgramEditor` 中最多 24 个有序 `ModuleKind` | 将模块编译成一次或多次 `PulseSpec` 发射 | `ProgramCaster` 统一编译、扣能量、运行令牌与取消；模块不能直接写 Tilemap |
| 世界操作 | Unity 构造候选；SemIf `direct_messages` 构造规范提示，llama.cpp `/completion` 返回单 token 候选概率 | 已登记的操作 ID；创世请求还要求玩家明确输入 `<作弊码>` | 调用 Unity 内固定、限界的玩法入口 | 仅对 Unity 声明且概率已覆盖的候选归一化；无覆盖则失败关闭。模型不能创建 ID、坐标、数值参数、模块数组或代码；Unity 复核快照并在主线程执行 |

生产后端是 `GodDialogueServer.py` 加载本地 tokenizer，并调用 llama.cpp Qwen3.5-4B Q4_K_M GGUF。SemIf `direct_messages` 构造规范决策提示，服务端读取 llama.cpp `/completion` 返回的单 token 候选概率，只在 Unity 实际发出的、且模型响应覆盖的有限候选中归一化；缺少候选覆盖或响应校验失败时拒绝决策。候选概率不是额外训练的防幻觉头、授权证明或校准置信度。此前 Transformers NF4 + `direct.score` 为历史基线。自然语言聊天仍然只返回文字。

## 玩家法术模块

每个修饰模块从左向右作用到下一个 `EMIT`；`EMIT` 生成一组脉冲并重置修饰状态。最多 24 项、最多 9 发、半径上限 24、伤害上限 64、击退上限 25；费用和持续工作由 `ProgramCaster`、`BlastScheduler` 约束。

| 分类 | 模块 ID | 组合维度 | 执行入口 |
| --- | --- | --- | --- |
| 发射/分叉 | `Emit`, `Fork` | 提交当前效果组；扇形增加脉冲数量 | `ProgramCaster.Compile` → `TerrainPulse.Spawn` |
| 空间范围 | `Expand` | 增加范围与钻孔宽度 | `PulseSpec.Radius` |
| 强度 | `Amplify` | 增加敌人伤害 | `PulseSpec.Damage` |
| 爆破/穿透 | `Blast`, `Drill` | 命中后范围爆破；钻穿可破坏材料 | `TerrainPulse` → `BlastScheduler` / `GridTerrain` |
| 时序/循环 | `Fuse`, `Echo` | 延迟爆破；爆破完成后回响 | `BlastScheduler` 有界事件队列 |
| 资源/动量 | `Recycle`, `Impulse` | 命中回收能量；给敌人冲量 | `ProgramCaster` / `EnemyChaser.ApplyKnockback` |

这套 DSL 是玩家的程序状态，不交给模型重写。新增模块必须同时更新编译器语义、成本、预览、编辑器与中英本地化，详见[程序编译器与编辑器](program-compiler.md)。

## 神谕受限脚本 DSL（目标架构，尚未实现）

用户要求直接使用 Python 语法子集组合多个游戏原语。脚本语法和游戏原语是两层：`def`、赋值、`if/else`、`for`、有界 `while` 属于脚本语言语法，由项目 DSL parser/interpreter 实现；它们不是游戏原语，不进入操作 registry。SemIf 只在插入游戏操作/查询/效果模块调用时约束合法 ID；不对脚本的每一个 token 做单 token 候选评分。Unity 再把脚本解析为类型化 AST、校验并通过静态 registry 执行。

| 分类 | 示例 ID/语法 | 职责 |
| --- | --- | --- |
| 游戏动作原语 | `portal.create`、`portal.set_profile`、`terrain.explode` | Unity registry 的稳定 ID；强类型参数、前置条件、范围/预算和 undo policy |
| 游戏查询原语 | `portal.exists`、`portal.spawn_count`、`enemy.profile` | 只读权威 Unity 状态；为分支与验收提供类型化值/证据 |
| 效果模块原语 | `Fork`、`Drill`、`Blast`、`Echo`、`Impulse` | 通过允许的模块适配器编译为现有 `PulseSpec`，仍由正式法术系统执行 |
| 脚本语言语法（不是游戏原语） | `def`、赋值、`if/else`、`for`、`while`、`return` | Python 语法子集与项目解释器控制流；静态上限嵌套、迭代和运行时间 |
| Harness 运行时内建（不是游戏原语） | `wait`、`assert`、`accept`、`undo_task` | 推进/观察任务并控制验收或事务回滚；受同一任务时限、取消和 journal 约束 |

目标流程是 **受约束 AST 预检 → 同场景隔离试炼上下文运行完整脚本 → 全部断言通过 → 用同一份脚本在玩家世界正式执行 → 实机验收和可撤销 journal**。第一阶段 `HarnessSandboxChamber` 已有单步数据投影预演和独立 `SandboxPartitionId`，但没有单独的 Tilemap、物理/实体/队列副本，也还不能在实机副作用发生前预演整段多步脚本。更完整的分区试炼舱仍需实现；无论哪一阶段，试炼数据都不得进入实机敌人数/击杀/HUD/存档。

仅提交不可变脚本 artifact，不把试炼舱里已改动的地形/物件移动到实机。正式执行前重验输入世界版本，发现快照过期则重新预演；实机结果也要复验，因为玩家世界会与沙盒状态出现差异。沙盒预演不替代实机撤销；所有会改状态的正式节点必须提供真实 `undo/compensation` handler 和 journal，否则不得进入可撤销 registry。无限周期行为必须有停止路径、生成速率和存活上限，并能撤销已生成的实体。

当前 Unity `/decide` 仍只在固定候选 ID 中选择，创世流程最多10步。`HarnessSandboxChamber` 把 Unity 固定参数编译成不可变单步 artifact/hash，在数据投影上预演后，再由实机执行器重放并分别验收；它还没有 Python 语法子集解释器、完整多步隔离试炼或通用撤销栈。本节描述目标设计，不得把单步数据预演写成完整物理沙盒。

## 神谕世界操作

| ID | 分类 | 固定行为与硬上限 | 起止/取消 | Unity 验收证据 | Unity 所有者 |
| --- | --- | --- | --- | --- | --- |
| `faultline_charge` | 组合战斗 | 固定 `Fork → Drill → Blast → Fuse → Expand → Impulse → Emit`；目标由 Unity 锁定，最远 20 单位；费用由共用编译器计算 | 命中/8 秒观测超时；`hold` 不产生副作用 | 锁定目标受伤/击败、新增爆破且队列清空 | `GodActionHarness` → `ProgramCaster` |
| `demolish_map` | 地形改造 | 在可玩边界按最多 30 单位间距布点，逐点提交半径 24、伤害 64 的爆破；毁坏可破坏材料，保留边界基岩；调度器限制每帧地形工作 | 操作进行时按 `X` 停止后续爆破；当前爆破批次完成 | 覆盖爆破全完成、可破坏格归零且基岩计数不变 | `GodActionHarness` → `BlastScheduler` → `GridTerrain` |
| `drill_bore` | 穿透/爆破 | 半径 3.4、伤害 16、击退 14；从玩家附近向右发射，文本明确向左时改为向左；带钻脉与回收 | 单次弹体，最长观察 4 秒；按 `X` 取消 | 脉冲结束、新增爆破、可破坏地形减少且队列清空 | `GodActionHarness` → `TerrainPulse` |
| `shockwave` | 范围/物理 | 半径 18、伤害 64、击退 25 的单次爆破 | 观察队列完成；按 `X` 中止 | 新增爆破且待处理工作清空 | `GodActionHarness` → `BlastScheduler` |
| `echo_storm` | 循环/法术效果 | 半径 12、伤害 8，回响并回收能量；沿现有有界爆破队列持续传播 | 持续运行，按 `X` 停止后续回响 | 至少两次新连锁爆破，且 RunToken 未变、后续链仍有工作 | `GodActionHarness` → `ProgramCaster` / `BlastScheduler` |
| `infinite_energy` | 资源规则 | 暂时跳过玩家法术和回响的能量扣除；不修改储能上限或玩家已保存程序 | 持续生效，按 `X` 关闭 | `ProgramCaster.CheatInfiniteEnergyEnabled` 为真 | `ProgramCaster` |
| `endless_enemy_portal` | 角色/生成 | 放在当前层安全落脚点；每 3 秒尝试生成一只敌人，同场存活上限 8 | 持续生成，按 `X` 关闭传送门 | 传送门仍活动且本次观察到刷新；已有门需有历史刷新记录 | `GameWorld` → `EnemySpawnPortal` |
| `set_portal_strong_profile` | 角色/属性 | 将现有刷怪门设为强敌 profile；存活上限和刷新间隔沿用门户注册参数 | 门关闭时拒绝；按 `X` 可关闭持续生成 | 至少观察一只新刷出的强敌，并核对生命、接触伤害、移速与刷新间隔 | `GodActionHarness` → `EnemySpawnPortal` / `EnemyChaser` |
| `create_black_hole_spell` | 地形/材料/法术 | 在地图内指定中心与初始半径创建黑洞；每块按8×8展开为64粒子；按每帧4096宏观格预算扫描 | 准备阶段不改地形；随后 `set_black_hole_duration` 启动。普通法术默认半径5.5 Unity units | 准备状态、初始半径、8×8规格和无副作用投影回执 | `GodActionHarness` → `GameWorld` → `BlackHoleSpell` |
| `set_black_hole_duration` | 地形/材料/吸附 | 启动黑洞；吞噬粒子使半径按 `0.05 × sqrt(已吞噬粒子数)` 增长，无视材料硬度吞噬作用范围内材料/基岩与生物 | 普通法术默认5秒、上限60秒；全图放逐目标将时长设为无限，并以12 Unity units/s扩张到地图边界。有限结束后或按 `X` 撤销；无限吞噬完成后仍活动，可用快照撤销 | 有限版实时检查局部材料/基岩吸收、增长半径和撤销快照；无限版额外检查全图格/粒子/基岩/生物归零；与 snapshot 撤销结果对照 | `GodActionHarness` → `BlackHoleSpell` → `GridTerrain` |
| `undo_last_black_hole` | 撤销/快照 | 恢复上一次黑洞保存的全地形材料/粒子快照、敌人位置/启用状态并恢复门户 | 单次撤销；没有可用黑洞快照时拒绝 | 还原前后宏观格/粒子/生物数量一致，Tilemap/碰撞同步 | `GodActionHarness` → `BlackHoleSpell.Undo` → `GridTerrain.RestoreSnapshot` |
| `fill_rock_blocks` / `fill_sand_blocks` / `fill_water_blocks` / `fill_magma_blocks` / `fill_bedrock_blocks` | 试炼填充/材料 | 向镜头外隐藏 Harness 房填入指定材料块；数量和范围固定受限 | 当前仅定位在 Harness 房；此类填充没有独立的通用撤销原语 | Sandbox 投影数量、实机隐藏房材料数量和对应材料索引 | `GodActionHarness` → `GameWorld` → `GridTerrain.FillBlocks` |
| `fill_particle_cluster` | 试炼填充/粒子 | 在隐藏 Harness 房生成受限材料粒子簇；本例按一个块64粒子展开 | 仅隐藏 Harness 房；可由后续黑洞操作整体快照撤销 | 细粒子数量、材料 ID 与占据掩码验证 | `GodActionHarness` → `GameWorld` → `GridTerrain.FillParticle` |
| `spawn_harness_creature` | 试炼填充/生物 | 在隐藏 Harness 房生成有限、已注册强度档的测试生物 | 仅隐藏 Harness 房；当前没有独立的通用删除/撤销 ID | 生物活动状态、位置和强度档查询 | `GodActionHarness` → `GameWorld` → `EnemyChaser` |
| `hold` | 无操作/拒绝执行 | 不执行游戏操作 | 立即结束 | 引擎世界状态无副作用；存在已完成步骤时整项任务标记部分完成 | `GodActionHarness` |

当前创世操作的自然语言匹配只决定哪些已注册 ID 有资格进入本次候选表；参数固定或夹紧在 Unity 代码中。不可匹配、否定或空目标不应获得世界操作候选。一个请求最多执行10步，模型每轮只选择一个未尝试候选；单步数据投影预演通过后交给实机执行器，实机回执用于决定下一步。失败、取消、超时、未尝试候选或仍有活动法术时停止并显示部分结果。模型的自述不算成功。黑洞具备地形/敌人快照撤销；刷怪门/无限能量可关闭；填充操作当前仅限隐藏房但没有独立撤销原语；地图爆破、普通挖掘和敌人伤害没有通用持久化 undo journal。不可撤销项尚未达到用户要求的通用事务架构，后续应在扩大 DSL 前补齐。扩展流程与可撤销准入条件见[神谕操作 Harness](ai-harness.md)。

## 通用调用契约

1. 聊天文本不能自动触发操作；`<作弊码>` 只在输入开头时进入创世请求入口。
2. Unity 记录玩家对象/生命/位置、施法 `RunToken`、待处理地形工作与游戏状态；评分期间暂停模拟。
3. 响应必须匹配本次 request ID 和 Unity 原先提交的候选；选择后再校验局面与 UI 状态。无效或过期响应零执行。
4. 每个原语使用已有领域 API：法术走 `ProgramCaster`、`TerrainPulse`、`BlastScheduler`；材料修改仍只由 `GridTerrain` 写入；角色由 `GameWorld` 组装。
5. 长操作必须有明确停止路径和工作上限。`X` 停止当前链、无限能量或敌人传送门；重新开始在 Harness 空闲后可用。
6. 玩家可见新状态统一使用 `GameLocalization.T(中文, English)`；中文为默认语言。

## 已完成验证

新增黑洞回归：`-blackHolePhysicsSmokeTest` 验证8×8/64粒子、岩石7次1点伤害保留64粒且第8次只破坏1粒；普通5秒黑洞在有限范围吞噬岩石、沙、水、岩浆、基岩样本和生物，随后快照精确撤销。`-creationBlackHoleSmokeTest` 用本地 Q4_K_M 模型处理“帮我将这个世界放逐到虚空”，验收 create→duration 两步、全图/基岩/粒子/生物归零和实时回执。

2026-09-25 Unity Windows 构建无 C# 错误/警告；`-movementSmokeTest` 通过。当前 llama.cpp Q4_K_M GGUF 生产后端的玩家进程 `-smokeTest -modelSmokeTest -actionHarnessSmokeTest` 通过，选择 `faultline_charge` 并命中锁定目标；standalone `/chat`、`/decide`、`/cancel`、`/shutdown` 检查通过。模型运行配置 `ngl=5`、`ctx=4096`、`parallel=1`、`flash-attn=auto`，额外显存约 0.8 GiB。真实模型 `-creationActionSmokeTest` 选择 `endless_enemy_portal`，等待 3 秒后场上敌人从 4 增至 5 并关闭传送门。真实模型 `-creationStrongPortalSmokeTest` 先生成并验证门户，再切换强敌 profile 并验证后续刷新；两只门属敌人均为强敌（最小生命 12、接触伤害 2、移速 3.2），刷新间隔 3 秒，双步骤都有 sandbox/live 回执和 hash。`-harnessSandboxSmokeTest` 覆盖预演隔离、强敌前置条件和 stale revision fail-closed。真实模型 `-mapDemolitionSmokeTest` 选择 `demolish_map`，将可破坏格从 10,250 清到 0，基岩格保持 5,740。其余创世操作的代码有固定参数与执行路径，但还没有各自的独立真实模型 smoke。此前 Transformers NF4 / SemIf `direct.score` 的结果属于历史基线。

## 扩展与验证

增加原语时，先补目录行和冻结的 ID/参数/授权边界，再实现 Unity 规则、快照字段、取消语义、结果事件与双语文本。验证非法候选、过期快照、未选择项零副作用、上限和取消；若涉及地形或物理，运行真实玩家进程而非只测 HTTP 响应。分派边界见根目录 `AGENTS.md`，完整 Harness 调用图见[神谕操作 Harness](ai-harness.md)。

