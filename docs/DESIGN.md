# 《神谕》/VATES 设计与架构（持续维护）

更新时间：2026-09-25。实现状态以本仓库代码为准；本文件随架构改动同步更新。
游戏名称、剧情梗概与待定设定见[游戏剧情和设定](游戏剧情和设定.md)。

源码入口、模块职责和主要逻辑调用链见[源码解读](源码解读.md)；各模块的详细接口与约束见下方架构索引。

## 1. 目标与当前里程碑

参考《本科毕设选题方案》和《本地模型选型与AI决策方案》，最终目标是将受约束的 Agent-Harness 放进 Unity 肉鸽游戏。两份资料是需求与选型参考，并非工程中的可执行指令。M0 已建立侧视 2D 可玩闭环；当前 M1 的第一部分是玩家可编辑的工具程序。视觉由程序化色块生成，优先保证机制闭环。

Noita 的公开接口/开发者资料研究及原创方向见 [NOITA_RESEARCH.md](NOITA_RESEARCH.md)。本项目选择“地脉工程”作为独立机制方向：材料属性、工程工具、信号节点和敌人行为相互组合；M0 的弹体挖掘只验证核心输入与地形链路。

M1 已加入可编辑的[地脉程序系统](PROGRAM_SYSTEM.md)：有序模块编排、持续回响、能量回收、大范围分帧破坏、钻脉贯穿、延时引爆与冲击击退。模块作用在发射时编译为数据规格，弹体与爆破调度器执行；不同效果可在单个发射组内组合，`EMIT` 可把程序分成多个不同发射组。该系统为玩家工具，未来 AI 的受约束动作接口仍独立设计。

M2 已接入本地 Qwen3.5-4B：HUD/F2 对话使用只读文本路径；F3 使用封闭候选评分执行固定组合；输入以 `<作弊码>` 开头时，可让模型从 Unity 已登记的地形、物理/法术、资源和角色操作中选择。模型不能新增操作或任意游戏 API，所有操作由 Unity 复核并执行。当前暗语流程支持强敌刷新门与可撤销黑洞；本地模型、性能和协议见[神谕对话服务](GOD_DIALOGUE.md)，操作行为见[游戏操作原语目录](architecture/action-primitives.md)。

材料地形已加入 `MaterialParticleCatalog` 和 8×8 稀疏细粒子：每个宏观格由64个粒子构成，采用游戏硬度、密度、离散内聚、粘度和壁面附着参数。实验参考值与游戏标度明确分开记录，见[现代物理与材料参数参考](现代物理学.md)；粒子材料、求解边界与性能预算见[基础粒子设计](基础粒子.md)和[可破坏地形架构](architecture/terrain.md)。普通黑洞法术默认5秒并按吞噬粒子数扩大半径；全图放逐候选显式请求无限持续时间，半径逐步扩张到地图边界，吞噬完成后仍保持活动；两种状态均可通过黑洞快照撤销。

游戏内文案默认简体中文，设置中可切换英语，选择跨重开保存；新增文案必须同时提供中英两版。实现接口与验证见[游戏内语言规范](LOCALIZATION.md)。

### M0 验收与当前结果

- Unity 中打开场景可直接 Play，Windows 构建可启动。
- 玩家能在大型固定地图中移动、跳跃、射击并挖开岩土；部分沙粒在失去支撑后下落。当前地图为 1024×160 格、单格 0.5 Unity 单位，总尺寸约 512×80、可玩范围约 510×78 世界单位。固定种子生成五个高度层、四段阶梯、平台、沙堆和五处可破坏岩门；四名哨兵分布在井口、回响矿层、断炉工坊与天脊核心，全部击败后可进入顶层出口。蓝晶裂谷当前作为无敌人的穿越层。
- 敌人能追逐、造成伤害并被击败；界面显示生命、击败数、目标及重新开始方式。
- 所有运行时对象由场景中的 `GameWorld` 组装，不依赖外部美术或模型服务。

此前 512×56 基线已通过 Unity Windows 构建及三敌关卡 smoke。2026-09-24 新增 1024×160 五层关卡后，Windows 构建无 C# 警告/错误；路线 smoke 先后发现宽台阶重叠遮挡，以及上层房间地板提前开始导致末级被挡、垂直高差达 4 格。最终将落脚面收窄为 5 格、将房间地板起点对齐阶梯终点，并在 smoke 中断言相邻落脚面高差不超过 2 格。实际玩家进程 `-smokeTest` 输出 `EMBER_HOLLOW_SMOKE_PASS`，覆盖地图跨度、五层/四段阶梯落脚面和高差、出生/出口、敌人分布、地形碰撞同步、钻脉、回响取消、大范围破坏、四敌清除和胜利状态。独立真实模型 F3 smoke 输出 `GOD_ACTION_HARNESS_SMOKE_PASS`，现行 Q4_K_M GGUF 后端选择登记组合并命中锁定目标。2026-09-25 新增的移动 smoke 与真实模型暗语刷怪门、全图爆破 smoke 通过；全图测试清除 10,250 个可破坏格并保留 5,740 个基岩格。神谕模型的后端、验证与历史基线见[神谕对话服务](GOD_DIALOGUE.md)；手动连续跑完整关、跳跃手感、敌人导航、性能中位数/P95 和连续多局稳定性仍待验证。

## 2. 当前架构

```text
Prototype.unity
  └─ GameWorld（运行时组装/胜负状态）
      ├─ GridTerrain（宏观材料、8×8稀疏细粒子、双 Tilemap 投影、挖掘/填充/材料更新）
      │   └─ MaterialParticleCatalog（材料粒子的数值参数）
      ├─ BlackHoleSpell（限时吸附、直线粒子表现、实体吞噬、状态快照撤销）
      ├─ PlayerController（输入与角色物理）
      ├─ EnemyChaser × N（追逐与伤害）
      ├─ ProgramCaster（程序编译、能量、存档和运行 token）
      ├─ ProgramEditor（暂停中的有序模块编辑）
      ├─ GameSettingsMenu / GameLocalization（设置、语言切换与双语文案）
      ├─ GodChatWindow（暂停中的神谕对话 UI，F2 打开）
      ├─ GodActionHarness（F3 组合与暗语创世操作；有限候选、快照复核、调用登记原语并检查结果）
      ├─ GodDialogueService（本地 Python sidecar；分开承载流式对话和候选打分）
      │   └─ GodDialogueServer.py（仅绑定 127.0.0.1；本地 tokenizer + llama.cpp Qwen3.5-4B Q4_K_M GGUF）
      ├─ TerrainPulse（工具脉冲、直接命中与爆破提交）
      ├─ BlastScheduler（回响队列、敌人伤害、分帧地形工作）
      ├─ EnemySpawnPortal（创世刷怪门的生成间隔与存活敌人数上限）
      ├─ BlastVisual（瞬时效果，仅表现）
      ├─ CameraFollow2D（镜头跟随）
      └─ GameHud（状态与操作提示）
```

`GameWorld` 只协调关卡生命周期与胜负；战斗和地形逻辑留在各自组件。`GridTerrain` 是地形状态的唯一写入点。`ProgramCaster` 把有序模块编译成发射规格，`TerrainPulse` 命中后向 `BlastScheduler` 提交爆破；调度器按帧预算调用地形单元接口，不直接改 Tilemap。玩家与敌人通过明确的 `TakeDamage` 接口处理伤害。

### 关键接口（当前）

| 所有者 | 接口 | 用途 |
| --- | --- | --- |
| `GameWorld` | `Instance`, `Player`, `Terrain`, `IsGameOver`, `RegisterEnemyKill()`, `OnPlayerDeath()`, `Restart()` | 关卡协调 |
| `GridTerrain` | `DamageCircle(Vector2, float, int)`, `DrillSegment(Vector2, Vector2, float)`, `TryDamageCell(int, int)`, `FloorY(float)` | 地形修改与出生位置；单元写入仅由地形模块负责 |
| `PlayerController` | `Health`, `MaxHealth`, `TakeDamage(int)` | 玩家生存状态 |
| `EnemyChaser` | `TakeDamage(int)` | 敌人受击 |
| `ProgramCaster` | `Modules`, `Preview`, `TryAdd`, `Move`, `RemoveAt`, `LoadPreset`, `TryCast`, `StopAll` | 编排与编译、能量结算、取消 |
| `TerrainPulse` | `Spawn(Vector2, Vector2, Transform source, PulseSpec)` | 用显式发射者与规格生成脉冲，避免从空间位置猜测归属 |
| `BlastScheduler` | `QueueBlast(Vector2, Vector2, PulseSpec)` | 有界处理回响、范围伤害与地形工作 |
| `GodDialogueService` | `ToggleOpen()`, `Close()`, `Submit(string)`, `Cancel()`；只读 `Messages`, `Status`, `IsGenerating` | 对话状态、暂停恢复、启动本地模型 sidecar 与逐段收取文本；不执行游戏命令 |
| `GodDialogueService` | `RequestDecision(GodDecisionRequest, callback)` | 通过单独的 `/decide` 端点打分 Unity 提交的封闭候选，并检查响应选项与 argmax 一致 |
| `GodActionHarness` | `RequestFaultlineCharge()`、`RequestCreativeAction(playerText, goal)`、`IsBusy`、`StatusText` | F3 固定组合和暗语创世候选使用独立评分流程；重验证后由 Unity 执行登记操作并确认结果 |
| `ProgramCaster` | `TryCastFaultlineCharge(origin, direction)`、`FaultlineChargeEnergyCost` | 使用共用编译器执行 Unity 内固定的组合，不改写玩家已保存程序 |
| `GodDialogueServer.py` | `GET /health`, `POST /chat`, `POST /cancel`, `POST /decide` | 仅回环地址上的模型状态、llama.cpp 文本生成/取消和受限候选评分；没有游戏命令 API |

### 地形坐标与运行原则

- 固定 1024×160 二维宏观格，单元尺寸 0.5 Unity 单位。每个宏观格按8×8、64个粒子细分，细粒子边长0.0625；完整方块压缩表示，变化格稀疏展开。`GridTerrain` 保存权威材料状态，宏观/细粒子 Tilemap 负责投影和碰撞。世界边界、出口位置以及相机水平/垂直跟随范围读取地形可玩边界；五个区域和阶梯的格子坐标见[五层关卡设计](LEVEL_DESIGN.md)。
- 沙粒更新维护沙格索引集合并复用排序缓冲，每个沙粒 tick 只访问现存沙格，按格子索引排序以保持稳定的单次下落顺序；地图仍为静态尺寸，地形显示与碰撞由 Tilemap 提供。
- 范围爆破仍由 `BlastScheduler` 按每帧最多 320 个地形格的共享预算分帧执行；每批最多收集 96 个候选格，再通过 `GridTerrain.DamageCellsBatch` 成批更新格子并调用一次 `Tilemap.SetTiles`。预算限制工作量，不是帧率承诺；尚未记录独立性能跑分。
- `TerrainPulse` 使用新版 `Physics2D.CircleCast`、筛选器与复用的 64 项命中数组；Unity 返回按距离排序的结果，正常路径不再分配或手动排序。缓冲装满时回退到 `CircleCastAll` 保留完整命中，极端拥挤场景下该路径可能分配内存。
- 基础材料为岩石、沙、水、岩浆和不可破坏基岩；材料参数表控制细粒子的硬度、相对质量、内聚、粘度及壁面附着。更新预算限制展开区域的单轮计算量。
- 东行路线有一处高岩壁，玩家需用切割脉冲开洞；敌人被岩壁暂时隔开属于关卡节奏设计。
- 地形修改必须检查地图边界和材料类型；输入、弹道与 AI 都不能直接操作底层数组。
- 地形与战斗都在 Unity 主线程执行，避免 Tilemap/Physics2D 跨线程访问。
- Tilemap 碰撞可能晚于格子数据一个更新阶段完成；当前 smoke 检查了下一物理步的碰撞刷新。后续若需要同帧查询修改结果，查询材料数组，或在明确的固定步边界等待碰撞更新。

## 3. 面向 AI 的边界

聊天仍只接收只读局面摘要并生成文字。生产后端由 `GodDialogueServer.py` 加载本地 tokenizer，并调用 llama.cpp 加载 Qwen3.5-4B Q4_K_M GGUF。F3 操作使用分开的 `/decide` 协议：SemIf `direct_messages` 构造规范提示，再读取 llama.cpp `/completion` 的单 token 候选概率；仅对 Unity 声明且模型概率覆盖的候选归一化，缺少覆盖或响应无效时失败关闭。该评分不是正确性证明，也没有校准为置信度。Unity HTTP 契约、候选表、快照复核与 Harness 前置检查仍是动作安全边界。

神谕操作包含 F3 固定组合与需要 `<作弊码>` 明确触发的有限创世候选。在评分时暂停局面；Unity 对照玩家、关卡和队列快照重新校验，随后执行登记操作并观察真实结果。创世请求最多10步；每步生成原语专属 Unity 验收回执，再携带新快照供模型评估未尝试的子目标。失败、取消、超时、仍有活动效果或安全门不通过时停止并报告部分结果。黑洞以地形/生物快照撤销；通用可撤销交易账本尚未实现。操作目录见[游戏操作原语](architecture/action-primitives.md)，Harness 边界和 Pi 参考见[神谕操作架构](architecture/ai-harness.md)，逐段源码研究见[Pi 代码研究](PI_AGENT_HARNESS_RESEARCH.md)。

当前 `HarnessSandboxChamber` 已在唯一 `GameWorld` 中建立逻辑试炼上下文，对固定原语执行单步数据投影预演，再把同一个不可变 step/hash 交给实机并分别验收；它尚无独立 Tilemap、Physics2D 或完整多步脚本预演。下一阶段直接采用 Python 语法子集：`def/if/for/while` 是语言结构，不是游戏原语；SemIf 只在插入游戏 API 调用时约束原语 ID，Unity parser/预算校验后先在隔离上下文验收整段脚本，再提交同一 artifact 到实机。每个副作用节点还需可运行的逆操作和 journal；地图破坏等现有原语尚不能完整撤销。Pi/Codex 沙盒仅提供工程边界的研究思路。地图下方的密室代码现已触发 Steam 风格隐藏成就“神的房间？”，普通胜利成就与隐藏成就列表使用 `PlayerPrefs` 原型存档，尚未接 Steamworks；Unity 运行验收见根目录 [TODO.md](../TODO.md)。真实模型已通过门户刷新、强敌配置、后续强敌属性和刷新间隔的双步 smoke；完整 DSL、成就 UI/物理碰撞还待验。完整状态见[神谕操作架构](architecture/ai-harness.md)和[成就机制](EASTER_EGG_MECHANIC.md)。

## 4. 决策记录

| 日期 | 决策 | 原因 |
| --- | --- | --- |
| 2026-09-24 | 在 `D:\UnityGameAgentHarness` 新建 Unity 6000 工程 | 当前工作区为空，避免改动其他 Unity 工程 |
| 2026-09-24 | 先做无外部依赖的可玩切片，再接本地 AI | 优先验证移动、碰撞、地形修改和战斗反馈 |
| 2026-09-24 | 使用程序化色块与 Tilemap 表达像素地形 | 无需等待美术资源，格子可局部修改 |
| 2026-09-24 | 将后续原创玩法聚焦“地脉工程” | 与参考作品保持机制表达上的独立性，并为合法动作原语提供可组合对象 |
| 2026-09-24 | 地脉程序采用按序修饰、`EMIT` 提交、回响与能量回收 | 玩家能像写短程序一样组合效果，重复爆破和高范围伤害有明确执行上限与取消入口 |
| 2026-09-24 | 大范围破坏交给 `BlastScheduler` 分帧执行 | 限制单帧爆破事件与地形单元数量，持续回响时保持游戏可响应 |
| 2026-09-24 | 游戏内简体中文默认、英语可选，并持久化设置 | 同时服务中文和国际玩家，统一管理未来新增文案与字体 |
| 2026-09-24 | 增加钻脉、延时和冲击作为独立组合轴 | 钻脉改变弹体与地层的相互作用；延时改变爆破时序；冲击改变敌人位移，三者可和范围、分叉、回响、伤害交叉组合 |
| 2026-09-24（历史基线） | 对话复用 `D:\WorkData\SemIf` 的 Qwen3.5-4B checkpoint，通过独立本机推理 sidecar 流式返回；游戏侧协议不绑定模型格式 | 当时生产后端为 bitsandbytes NF4 4-bit；相关速度与算子观察仅属历史基线。现行生产后端已切换为本地 tokenizer + llama.cpp Q4_K_M GGUF，动作评分链见“面向 AI 的边界”与[神谕对话服务](GOD_DIALOGUE.md) |
| 2026-09-24 | 对话模型只接受只读局面摘要并返回文本，不具备游戏工具调用 | 符合“生成—选择分离”边界；先打通感知和专属剧情回应，动作原语与组合校验仍单独实现 |
| 2026-09-24 | 将可玩路线扩至 512×56 格，并按沙粒数量、爆破批次和复用碰撞结果控制运行时工作 | 玩家可从短小起始区域探索约 254 世界单位宽的路线；避免沙模拟每轮扫全图，并减少爆破 Tilemap 写入调用与弹体碰撞分配 |
| 2026-09-25 | 同场景逻辑沙盒第一阶段按固定原语逐步预演、实机复验；目标 DSL 借 Python 语法形状但由 SemIf 约束输出并交 Unity registry 解释，整段先验收与 undo journal 仍待实现。 | 保留唯一 `GameWorld`，用上下文/快照隔开试炼和实机；不能将候选打分称为 SemIf 输出头，也不能将当前数据投影称为完整物理沙盒。 |
| 2026-09-25 | 隐藏房间首次进入解锁持久化隐藏成就；成就页区分普通与隐藏项，未解锁隐藏项显示问号。 | 玩家触发器只认当前 Player；AI 沙盒 ID 与成就触发权限分离。当前是 PlayerPrefs 原型存档，Unity 展示与重启验收待完成，Steamworks 接入未做。 |
| 2026-09-24 | 将地图扩为 1024×160 五层关卡，加入四段连接阶梯、五处材料门、四名哨兵和顶层出口；相机边界改为动态跟随地形高度 | 横向与垂直探索空间同时增加，并用逐层地标和明确清敌目标构成完整原型关卡；smoke 揭示宽平台遮挡、房间地板提前开始及末级高差过大，最终收窄平台、将房间起点对齐阶梯终点并加入每级高差断言。普通玩家和真实模型 F3 回归通过；性能只保留已有沙索引/爆破预算设计，尚无目标设备帧耗时实测 |
| 2026-09-25 | 创世暗语参考 Pi 的 tool result loop，加入最多三步、每步 Unity 验收后再决定下一步的 Harness 流程 | 仅传入有限原语 ID 与新鲜权威快照；执行回执记录动作 ID、通过状态和引擎证据；失败、取消、超时或残留法术/地形工作时停止，不接受模型自报成功。Pi 源码研究见[PI_AGENT_HARNESS_RESEARCH.md](PI_AGENT_HARNESS_RESEARCH.md)，验收条件见[操作原语目录](architecture/action-primitives.md) |
| 2026-09-25 | 本地对话超过 12 条时自动压缩旧历史，保留最近 8 条原文并将短摘要仅作为聊天背景 | 维持 4096-token 服务端预算与现有 GPU 显存上限；归档用本地 `/summarize` 串行生成，摘要失败时回退到最近原始消息，不作为当前世界事实或 Harness 决策输入。真实模型压缩 smoke 和完整普通玩法 smoke 通过；实现细节见[神谕对话服务](GOD_DIALOGUE.md)与[AI Harness 架构](architecture/ai-harness.md) |
| 2026-09-25 | 将材料粒子规格固定为宏观方块8×8、共64粒；现实试样密度/强度作为材料代理的参考，普通伤害按游戏阈值处理，黑洞无视硬度。 | 保留宏观格权威数据，用稀疏细粒子掩码、有限更新预算和黑洞专属 snapshot undo 支持玩法；有限物理 smoke 验证 Rock/玄武岩代理阈值、五种材料局部吞噬与精确撤销。真实 Q4_K_M 无限放逐 smoke 依次验收创建/持续时间，41.87秒内吞噬16,067格、1,028,288粒子、5,740格基底与4只生物，半径从5.5扩到507.9并通过实时验收。地图爆破 smoke 验证72次覆盖爆破清除10,327格并保留5,740格基底。通用操作 undo journal 与目标硬件性能基线仍待完成。现实数值、条件、映射与限制见[现代物理与材料参数参考](现代物理学.md)。 |

## 5. 下一里程碑

M1 后续：扩充原创材料与节点反应，验证“切槽移除支撑 → 落砂 → 压力节点”；根据手动试玩调整回响速度、能量与画面可读性。M2 已包含只读聊天、固定组合、强敌传送门、黑洞/8×8材料粒子与有界验收循环。创世请求上限为10步；持久化操作日志、状态版本和通用逆操作账本仍待实现。性能方面继续测量现行 Q4_K_M GGUF 后端的首 token、稳定 token/s、显存和结构化决策质量；当前仍用 `ngl=5/threads=8/cache_prompt=false`。NF4 及其他 GGUF 测试均作为历史/对比基线，参考文档或单次生成不能代替本机基准。

## 6. 构建与自动检查

编辑器菜单 `Ember Hollow/Create Prototype Scene` 会重建原型场景并写入构建场景列表；`GameProjectSetup.BuildWindows` 可从 Unity 批处理执行，生成 `Builds/EmberHollow.exe`。玩家程序传入 `-smokeTest` 会运行玩法闭环检查；`-movementSmokeTest` 检查跳跃缓冲/低摩擦；`-actionHarnessSmokeTest` 验证固定组合；`-creationStrongPortalSmokeTest` 验证门户及强敌二步验收；`-creationBlackHoleSmokeTest` 用真实模型验收“放逐世界到虚空”；`-harnessSandboxSmokeTest` 检查隔离预演和隐藏成就服务入口；`-mapDemolitionSmokeTest` 验证清理全图和保留基岩；`-blackHolePhysicsSmokeTest` 验证8×8粒子展开、单粒子硬度、黑洞无视硬度吞噬、材料填充和撤销。地图检查运行完毕后进程以 0/1 退出并记录断言结果；附加 `-capture` 会输出玩法截图。架构说明入口见[架构文档索引](architecture/README.md)。神谕模型依赖项目外的本机 SemIf Python 环境和模型权重，配置详见 [GOD_DIALOGUE.md](GOD_DIALOGUE.md)。

