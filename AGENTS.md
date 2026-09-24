# Agent 协作与分派记录

本文件是项目持续维护的协作记忆。每次分派、集成或返工后更新结论；以实际编译、运行和代码审查结果判断能力边界，不按模型名称预设质量。

## 角色与分派准则

- **gpt-6-sol／主协调者**：定范围与接口，处理跨系统设计、Unity 工程/构建集成、非局部错误、性能与安全边界，以及最终验证。需要判断根因或改动多模块时由 Sol 负责。
- **gpt-6-luna**：处理接口已冻结、影响范围小、可独立编译审查的模块。适合单脚本行为、样式/文案、小工具和机械性改动。分派时写清文件所有权、已存在接口和完成标准。
- 同时改同一文件、需求仍模糊、涉及地形状态一致性/物理同步/AI 安全校验或连续失败的任务，不直接交给 Luna；先由 Sol 拆分或亲自完成。
- **Harness DSL 分层规则**：Python 语法子集中的 `def`、赋值、`if/for/while` 是脚本语法，不是游戏原语。registry 只含受类型/范围校验的游戏操作、效果模块适配器和权威查询；SemIf 只约束这些 API 调用点的合法 ID，不为脚本每个 token 做单 token 候选评分。Unity 始终用项目 parser 解析并校验整份脚本。
- Luna 产出先审查，再用 Unity 编译与运行验证。若问题局限且修复明确，可让原 Luna 继续；若必须重构接口或跨模块排查，转给 Sol。
- **本地化长期规则**：所有新增玩家可见文本默认简体中文，并在同一次改动中提供完整英语。统一经 `GameLocalization.T(中文, English)` 显示；设置中即时切换，保存选择。界面改动需检查两种语言的截图或实机显示，避免仅翻译某一页面。

## 当前模块归属

| 模块 | 初始负责者 | 状态 | 集成结论 |
| --- | --- | --- | --- |
| `PlayerController.cs` | gpt-6-luna | 已提交 | Unity 编译与玩家进程启动通过；键鼠手动体验待验 |
| `EnemyChaser.cs` | gpt-6-luna | 已提交 | 接地/障碍检测局部返工一次；编译与击败计数检查通过 |
| `TerrainPulse.cs`（原 `SpellProjectile.cs`） | gpt-6-luna，集成时由主协调者改名 | 已提交 | 命中规则局部返工一次；Unity 编译通过，手动弹道体验待验 |
| `GridTerrain.cs` | 主协调者 | M0 完成 | 地形修改检查与相机截图通过 |
| `GameWorld`、场景、构建、设计文档 | 主协调者 | M0 完成 | Windows 构建、运行时闭环检查通过 |
| `ProgramCaster.cs`、`BlastScheduler.cs`、跨模块命中接口 | 主协调者／gpt-6-sol | M1 程序闭环完成 | 编译、回响、取消、分帧地形与大范围 smoke 通过 |
| `BlastVisual.cs` | gpt-6-luna | M1 已集成 | 单文件视觉脚本一次交付；Unity 构建、玩家进程和截图通过 |
| `ProgramEditor.cs` | 主协调者 | M1 已集成 | 原 Luna 分派未进入执行阶段，主协调者补做；暂停/恢复自动检查与界面截图通过 |
| `GameLocalization.cs`、`GameSettingsMenu.cs`、双语状态接口 | 主协调者 | 中英双语已集成 | 默认中文、持久化、暂停恢复与即时切换由主协调者设计验证 |
| `GameHud.cs` 双语文案 | gpt-6-luna，主协调者集成 | 已集成 | Luna 单文件文案与设置入口一次交付；主协调者移除过时状态映射并检查界面截图 |
| `ProgramEditor.cs` 新模块调色板 | gpt-6-luna，主协调者集成 | 已完成 | 新增模块和双语提示，右栏独立滚动；中英文截图布局通过 |
| `ProgramCaster.cs` 钻脉/延时/震荡编译 | 主协调者／gpt-6-sol | 已完成 | 有序组合/累积上限及代价编译由 smoke 覆盖 |
| `TerrainPulse.cs`、`GridTerrain.cs` 钻脉执行 | 主协调者／gpt-6-sol | 已完成 | 首次碰撞仅挖掉入口格；改为按弹体线段采样并通过 smoke 验证贯穿货架 |
| `BlastScheduler.cs`、`EnemyChaser.cs` 延时与击退 | 主协调者／gpt-6-sol | 已完成 | 队列按延时触发、敌人施加冲量通过玩家进程 smoke |
| `GodChatWindow.cs` | gpt-6-luna，主协调者集成 | 已集成 | IMGUI 对话窗与双语交互；按评审修正状态双语映射和长回复高度/滚动，Unity 构建与两种语言截图通过 |
| `GodDialogueService.cs`、`GodDialogueServer.py`、模型/Unity 协议 | 主协调者／gpt-6-sol | Q4_K_M 与上下文自动压缩已集成 | 本地 tokenizer + llama.cpp Qwen3.5-4B GGUF Q4_K_M；Unity HTTP 契约不变。真实模型摘要 smoke 归档旧 8 条并保留最近 8 条，之后聊天及完整玩法 smoke 通过；生产设置保持 `ngl=5/ctx=4096/threads=8`，没有增加显存占用 |
| `GodDialogueServer.py` system 消息结构校验 | gpt-6-luna，主协调者复核 | 已集成 | 独立单文件校验最多一个且首位 system 消息；py_compile 与合法/非法输入检查通过 |
| `GodDialogueService.cs` `/decide` 协议、`GodDialogueServer.py` SemIf 接口 | 主协调者／gpt-6-sol | Q4_K_M 候选评分已集成 | SemIf `direct_messages` 构造规范 prompt；llama.cpp `/completion` 返回字母槽概率，适配层只对 Unity 候选归一化，缺失候选分数则失败关闭；Unity 候选顺序/归一化/argmax 校验与 F3 命中 smoke 通过 |
| `GodActionHarness.cs`、F3 HUD 与 `ProgramCaster` 固定组合接口 | 主协调者／gpt-6-sol | 已集成 | F3 只选 `faultline_charge` 或 `hold`；Unity 复核状态后执行固定组合并观察真实命中。最新构建上的真实模型玩家进程 smoke 通过，组合命中锁定目标且爆破计数 1–3 |
| `docs/architecture/*.md` 与 `docs/PI_AGENT_HARNESS_RESEARCH.md` 架构/研究说明 | 主协调者定边界；gpt-6-luna 编写 `world-and-actors.md`、`terrain.md` | 已集成 | Luna 两篇按冻结源码职责独立完成，主协调者复核 F3/Harness 与 `Health` 接口；Pi 独立研究页来自官方源码与文档，并链接到 AI Harness 与架构索引。 |
| 地图扩展与运行时性能工作量 (`GridTerrain.cs`、`BlastScheduler.cs`、`TerrainPulse.cs`、`GameWorld.cs`) | 主协调者／gpt-6-sol | 512×56 基线已验证，后续见下行 | 动态边界、沙格索引、96 格爆破 Tilemap 批写、固定数组 `CircleCast` 和分段敌人布局先在 512×56 单层路线验证；目标硬件帧耗时基线待测 |
| 五层关卡生成、相机纵向边界、四敌通关 (`GridTerrain.cs`、`GameWorld.cs`) | 主协调者／gpt-6-sol | 已集成 | 1024×160、五层基准地板、四段阶梯、五处可破坏岩门、四名哨兵及天脊出口；路线 smoke 先发现宽落脚平台遮挡，再发现房间地板提前开始导致末级遮挡和 4 格高差；Sol 将落脚平台收窄为 5 格、房间起点对齐阶梯终点，并以 smoke 断言相邻落脚面高差不超过 2 格；最新无 C# warning/error 的 Windows 构建、玩家 `-smokeTest` 和真实模型 `-actionHarnessSmokeTest` 均通过；手动跳跃/导航及目标硬件性能待验证 |
| `GameHud.cs` 当前层指示 | gpt-6-luna，主协调者集成 | 已集成 | Luna 单文件增加实时层号/中英文层名和五层四敌目标；主协调者构建、双语状态切换及整关 smoke 通过，手动截图/布局检查仍待实机确认 |
| `docs/LEVEL_DESIGN.md` 五层关卡说明 | gpt-6-luna，主协调者审阅 | 已集成 | Luna 按源码坐标写中英关卡剖面、敌人/出口、地形限制和待测事项；主协调者复核房间与阶梯接合坐标、每级高差不超过 2 格，并补入最新普通与真实模型 smoke 结论 |
| `docs/DESIGN.md`、`docs/architecture/spell-execution.md` 地图与性能更新 | 主协调者冻结代码事实；gpt-6-luna 更新文档 | 已集成 | Luna 独立补写设计概览和法术执行页；主协调者在碰撞查询换用 Unity 6 新接口后同步修正文档，并检查未将候选预算写成实测帧率 |
| `EnemySpawnPortal.cs` | gpt-6-luna 单脚本；Sol 负责世界生成入口与整合 | 已集成 | Luna 实现 3 秒刷新节奏和同场 8 只存活上限；Sol 接入安全落脚点/刷怪、关闭入口和双语反馈。真实 SemIf 模型创世 smoke 选择 `endless_enemy_portal`，3.5 秒内生成首只敌人并通过 |
| `<作弊码>` 世界操作及 `GodActionHarness.cs` 扩展 | 主协调者／gpt-6-sol | 首批原语已实现 | Unity 只给匹配的登记 ID 候选；加入地图爆破、刷怪门、冲击波、回响风暴、无限能量、钻脉；玩家对象/状态复核、固定参数、150 秒全局时限及 `X` 取消。真实模型刷怪门与地图爆破 smoke 通过；冲击波、回响、无限能量、钻脉需后续逐项 smoke/试玩 |
| `HarnessSandboxChamber.cs`、`GodActionHarness.cs` 单步预演与实机回执 | 主协调者／gpt-6-sol | Windows 构建、扩展版 `-harnessSandboxSmokeTest`、`-creationStrongPortalSmokeTest` 通过 | `oracle-sandbox:<task>` 数据投影预演已登记 step/hash；强敌前置缺失与过期 revision 均 fail-closed，预演零修改实机。最新 smoke 还验证隐藏房间成就触发/持久化、`J` 成就页暂停恢复及英文标签。真实 Q4_K_M 玩家进程按“门户实际刷新 → 切换强敌 → 检查后续强敌属性/刷新间隔”完成双步验收。仍不是完整 Unity 物理分区或整段脚本预演，也无通用撤销栈 |
| `AchievementService.cs`、`AchievementJournal.cs`、`AchievementToast.cs`、隐藏房间 | 主协调者设计与整合；`AchievementToast.cs`、`EasterEggRoomTrigger.cs` 为 Luna 单文件产出 | 构建与服务逻辑 smoke 通过；物理碰撞/UI 双语布局待人工验 | 普通胜利与隐藏“神的房间？”成就、问号隐藏项、J 列表、PlayerPrefs 持久化、提示和地下密室已集成；烟测检查密室壳及成就服务入口。原 toast 彩蛋稿因需求改为隐藏成就而删除；Steamworks 未接入 |
| `PlayerController.cs`、`GridTerrain.cs` 连按空格/墙边吸附修复 | 主协调者／gpt-6-sol | 自动检查通过 | 增加 0.12 秒跳跃缓冲、0.1 秒离地宽限；玩家胶囊与 TilemapCollider 共用零摩擦/零弹性材质。Unity `-movementSmokeTest` 验证落地前缓冲跳和材质一致；连续键盘/边缘手感仍待手动试玩 |
| `docs/architecture/action-primitives.md` 原语分类 | 主协调者／gpt-6-sol | 已留档 | 单独列出玩家法术 DSL 与神谕世界操作 ID，注明所属分类、固定参数、上限、执行 owner、取消与授权边界；已链接至架构索引、AI Harness 和设计总览 |
| `GodDialogueServer.py` / `GodDialogueService.cs` 模型后端与性能 | 主协调者／gpt-6-sol | Q4_K_M 生产后端已完成迁移 | 固定 35-token 历史短请求：HF NF4 decode 中位数 6.769/P95 7.319 token/s；Q4_K_M `-ngl 5` 约 +981 MiB、14.233/P95 14.418 token/s；144 条离线参考选择一致率 131/144（90.97%，参考为 BF16，不是正确率）。线程 4/8/12 实测 14.31/14.61/13.17 token/s，保留 8；前缀缓存探索测试第二轮后输入分叉，不能据中位数推断加速，同时贪心同输入输出未完全一致，未启用；ngram-mod 单组增益约 3.4%，没有干净质量对照，未启用。生产配置 `ngl=5/ctx=4096/parallel=1/flash-attn auto/threads=8`，模型总显存增量约 0.8 GiB；Unity 真实 F3 smoke 命中锁定目标。详细限制见 `docs/GOD_DIALOGUE.md` |
| `tools/model_backend_benchmark.py`、`tools/evaluate_gguf_decisions.py` | gpt-6-luna 冻结接口实现；主协调者复核 | 已交付并通过离线评测 | 使用 `D:\WorkData\SemIf\.venv\Scripts\python.exe`；Anaconda NumPy/SciPy 二进制不兼容，不能导入 Transformers。Sol 审查发现 llama `prompt_n` 位于 `timings.prompt_n`，Luna 修复后 benchmark/evaluator 通过。质量集是全 GPU GGUF 对 BF16 benchmark 参考，不是游戏 NF4 实机质量 |
| `GodDialogueService.StartSidecar()` 子进程缓存/关停 | 主协调者／gpt-6-sol | 构建与游戏进程验证通过 | Python 适配器及 llama-server 子进程继承指向 modelRoot 下 `benchmarks/runtime-cache` 的临时目录环境；llama 日志已写入 D 盘。Unity 正常退出请求 `/shutdown`，smoke 后未留 Python/llama-server 监听进程。C 盘未清理，安装包删除被系统策略拒绝 |

## 经验与调整日志

| 日期 | 观察 | 下次分派调整 |
| --- | --- | --- |
| 2026-09-25 | 对话窗口超出 12 条消息时，由 Sol 在 Unity 服务与本地 Python 适配层实现批量摘要、滚动记忆、取消、重开清理和安全回退；最近 8 条保留原文，摘要只送入聊天、不送 `/decide`。真实 Q4_K_M 玩家进程两批 `/summarize`、`/chat`、8 条保留断言与全玩法 smoke 通过。 | 对话记忆、推理锁、取消与 Harness 决策隔离属于跨层状态一致性，继续由 Sol 设计和验证；UI 的单条双语状态可以在接口冻结后独立委派 Luna。 |
| 2026-09-25 | 用户将隐藏房间彩蛋改为 Steam 风格隐藏成就。Luna 按单文件边界交付双语 `AchievementToast` 与玩家触发器；原 toast 彩蛋组件已删除。Sol 负责成就持久化/列表、密室地形、试炼上下文和跨模块接线。Unity Windows build 无 C# warning/error；隔离 smoke 与真实模型强敌门户双步 smoke 通过。尚未验证实际碰撞/UI 截图或 Steam API。 | 只把纯提示和简单触发入口交给 Luna；存档、成就状态和 Unity 世界分区由 Sol 设计。将用户改方向后的旧稿及时删除，架构文档明确当前实现/规划及不可撤销原语限制。 |
| 2026-09-25 | 用户澄清脚本控制语法与游戏原语分层：`def/if/for/while` 属于本地模型熟悉的 Python 语法子集，不是游戏操作；SemIf 只需把实际插入的游戏 API 名限定在注册表，不能把现有单 token 候选评分扩展到脚本每个 token。 | 架构文档、原语目录、Pi 研究页和 TODO 同步使用“两层：Python 语法/游戏 API registry”的术语；SemIf 不负责业务正确性，Unity parser、状态断言和预算不省略。 |
| 2026-09-25 | 最新扩展版 `-harnessSandboxSmokeTest` 在 Windows 构建后通过；同一 smoke 验证隐藏房间成就触发/持久化、`J` 成就页暂停恢复和英文标签、沙盒单步预演不改实机、缺少传送门时拒绝强敌配置、世界版本过期时拒绝执行。 | Harness 单步投影边界继续写清楚，不称作完整物理沙盒；下一阶段单独实现受限 Python 语法子集 parser/interpreter，语法节点不加入游戏原语 registry，游戏 API 才受 SemIf 调用点约束。 |
| 2026-09-25 | Q4_K_M 后端上比较 `cache_prompt`、`ngram-mod` 和 CPU threads 4/8/12，逐项保持约 1 GiB VRAM 上限。8 threads 是测得 decode 吞吐最高值（14.61 token/s）；12 threads 反而降到 13.17。前缀缓存六轮探索数据在第二轮后输入分叉，不能作干净 A/B，且贪心同输入输出不完全一致；ngram-mod 仅有约 3.4% 单组提升且没有干净质量对照。 | 不把官方可用选项当作本地质量保证。先用同一真实游戏提示做重复速度、贪心一致性、候选选择和中英文本质量回归；无法解释输出变化的推理缓存不进入生产。模型性能设置由 Sol 决定，测试工具或报表可委托 Luna，结果由 Sol 复核。 |
| 2026-09-25 | HF NF4 → llama.cpp Q4_K_M 涉及 Unity 进程生命周期、双进程回收、聊天流解析、SemIf 候选概率语义与 Harness 安全边界；Sol 冻结适配接口并独立实现。Luna 适合接手接口已稳定后的模型架构文档同步。Python standalone 聊天/决策/取消/关停通过，Windows 构建和真实玩家 F3 锁定命中 smoke 通过；额外显存约 0.8 GiB。 | 跨进程推理、取消或候选语义调整继续由 Sol 做；接口冻结后可把不改代码的模块文档同步分给 Luna，再由 Sol 核对最终实现/文档一致性。 |
| 2026-09-24 | Luna 在清晰的单文件接口下迅速完成敌人与法术弹初稿；尚未完成 Unity 编译或运行验证。 | 保持单文件任务形式，集成结论出来前不扩大委派范围。 |
| 2026-09-24 | Unity 初次编译通过；敌人接地/障碍检测、弹体过时 API 与非地形命中各发现局部问题，原 Luna 按明确反馈完成修正。 | 这类明确、局部的修复继续交给 Luna；跨文件命名与整体运行验证由主协调者处理。 |
| 2026-09-24 | 第二次干净构建没有 C# 警告/错误，玩家进程自动闭环检查通过；离屏截图确认基础画面。 | Luna 适合继续承担边界清楚的单模块行为；手感调试、环境连锁机制和后续 AI 校验器由 Sol 设计并复验。 |
| 2026-09-24 | Sol 跨模块复核发现玩家接地误判风险和弹体发射者反查风险；Luna 按明确接口修好接地，主协调者将弹体改为显式传入发射者。复验构建与碰撞刷新 smoke 通过。 | 保留“Sol 找跨模块根因 → Luna 修局部行为 → Sol/主协调者整合复验”的分工；涉及对象归属与 AI 安全边界时由 Sol 定接口。 |
| 2026-09-24 | Luna 的 `BlastVisual` 在明确视觉接口下直接集成，通过构建和运行截图。另一个 Luna 编辑器任务和 Sol 架构任务长期停留在初始化，未产生代码。 | 保留可独立视觉/单脚本任务给 Luna；初始化停滞属于任务执行环境问题，不能据此推断模型能力。超时后由主协调者接手，避免阻塞主线。 |
| 2026-09-24 | 地脉程序需要同时处理编译、能量交易、命中、回响、队列预算和 Tilemap 同步，主协调者统一设计与集成；玩家进程 smoke 通过持续回响、取消和大范围破坏。 | 此类跨系统一致性工作由 Sol 负责；下次可将模块说明、纯视觉效果、孤立编辑器控件等冻结接口的任务交给 Luna，再由 Sol 运行验证。 |
| 2026-09-24 | 双语 HUD 单文件任务由 Luna 迅速完成；主协调者发现状态字符串已改为双语持有，因此删除了 Luna 额外写的旧英文状态映射，并统一设置暂停与编辑器接口。 | 继续将边界明确的文案/单界面任务交给 Luna；共享状态与跨界面同步由 Sol 定接口，集成时检查是否存在重复翻译逻辑。 |
| 2026-09-24 | 在冻结 `Drill/Fuse/Impulse` 编译接口与双语模块名后，Luna 独立更新程序编辑器；十种模块和预设占用纵向空间，因此采用独立可滚动调色板，底部预览固定。 | 这种单 UI 文件可以委托；主协调者继续负责跨文件玩法语义、命中行为、计时队列、敌人物理与集成构建。 |
| 2026-09-24 | 钻脉最初只按物理表面命中位置挖掘，复合 Tilemap collider 进入内部后不再报告下一处表面；运行检查复现后，将路径沿线采样委托给 `GridTerrain`，穿透 smoke 通过。 | 地形一致性和 Unity 物理交界仍归主协调者；失败先用玩家进程定位，再更改唯一写入口，不扩大 Luna 任务范围。 |
| 2026-09-24 | Luna 在 `GodChatWindow.cs` 的冻结单文件 UI 下快速交付；评审要求增加五种状态的双语映射，并修正长回复裁剪/自动滚动。另将服务端 system 顺序校验作为独立单文件任务交 Luna，一次完成且 Python 校验通过；主协调者在模型 smoke 中定位到游戏端重复 system，合并提示后端到端回复通过。 | Luna 继续负责单界面与纯输入校验；服务协议、聊天模板兼容、Unity 生命周期与实际模型错误由 Sol 定根因并整合复验。 |
| 2026-09-24 | 实际运行确认 Qwen3.5 由 Transformers + NF4 从本地 checkpoint 加载，完整中文流式对话通过。日志显示 `causal_conv1d` 与 `flash-linear-attention` 缺失，PyTorch 走参考实现。 | 不因 GGUF 标签推定性能；先建立同提示、同设备的延迟/吞吐/显存/质量基线，再评估安装优化算子或 GGUF 后端。未获直接环境改动授权时，不改 `D:\WorkData\SemIf` 虚拟环境。 |
| 2026-09-24 | Pi Agent Harness 阅读后，将事件阶段、类型化输入、执行前后校验、权威状态与结果反馈映射到独立 Unity Harness；没有引入任意 shell/文件工具。有限候选 `direct.score` 减少自由命令输出，但它不是新训练的防幻觉头，也不验证选择正确。 | 新增游戏动作时先定义 Unity 可执行原语、证据快照、复核规则与可观察完成事件；再让模型从合法候选中评分。先由 Sol 定安全/状态接口，再把单文件本地化或独立模块文档等任务交给 Luna。 |
| 2026-09-24 | 真实模型组合 smoke 首次发现 `Drill + Blast` 弹体寿命到期后每帧重复排队；当次队列顶满 64 项并累计 636 次爆破。Sol 在 `TerrainPulse` 终止路径增加一次性消费/销毁，重跑模型 smoke 后固定组合命中锁定目标，爆破数量回到 1–3。 | 弹体终止、排队和调度器是跨组件一致性边界，由 Sol 查因并修改；以后用真实运行计数检查多阶段/回响组合，不以 HTTP 决策成功代替游戏动作成功。 |
| 2026-09-24 | Luna 在冻结职责后交付关卡/角色与地形架构页；Sol 审核并补入 Harness 状态、角色生命查询、地形唯一写入和验证边界。模块索引链接到编译器、法术执行、AI Harness、表现本地化页面。 | 继续将源码可核对、无接口设计的模块说明交 Luna；跨模块调用图与“已实现/规划”边界由 Sol 集成审阅，逐次修改后更新本文件和架构索引。 |
| 2026-09-24 | Sol 从 Pi 官方 `agent-loop.ts`、扩展、压缩和容器化文档提炼出事件阶段、schema、共享状态串行、摘要与事实分离、宿主权限边界，独立写入 Pi 研究页并链接到 AI Harness。 | 外部架构研究由 Sol 核验源码语义并定本项目落地原则；Luna 可继续写冻结模块的独立说明，不让文档将上游能力误记成游戏现有功能。 |
| 2026-09-24 | Sol 负责地图尺寸、沙粒索引、爆破批写和碰撞缓冲的跨模块实现；Luna 在代码接口冻结后更新设计概览与法术执行页。首轮构建发现旧 `CircleCastNonAlloc` API 弃用警告，Sol 改用 Unity 6 固定结果数组重载，省去手工命中排序并同步修正文档。最终 Windows 构建无 C# 警告/错误，新增地图跨度/出生出口/敌人分布/F3 可达范围断言、完整玩法 smoke 和真实模型 F3 锁定命中 smoke 均通过；尚无目标硬件 profiler 基线。 | 性能结构与物理/地形一致性由 Sol 设计并集成；Luna 继续承担接口冻结后的架构文档任务。下一阶段先采集帧耗时、TilemapCollider 重建耗时和 GC，再决定是否区块化，不按结构性优化推断具体帧率收益。 |
| 2026-09-24 | 完整模型聊天回归首次在最后的大范围爆破检查失败：测试从前序地形用例已改动的 `x=-12` 发射点起射，导致 cast 成功但观察窗口内没有爆破。将大范围 smoke 改到未受前序操作影响的地面 `x=8`，并给各断言输出单项原因；重建后普通玩法、模型聊天+玩法、真实模型组合三个玩家进程检查全部通过。 | 自动检查的场景和状态必须彼此隔离。跨模块执行项保持 Sol 负责；对常规 smoke 加明确断言原因后，下一次能区分代码回归与测试夹具相互污染。 |
| 2026-09-24 | 五层地图由 Sol 负责地形、出生/出口、相机纵向边界与四敌胜负；路线检查先发现台阶平台互相遮挡，再发现房间地板遮住末级落脚面且高差达 4 格。Sol 将房间起点对齐阶梯终点、统一 5 格落脚宽度，并把相邻落脚面高差不超过 2 格设为 smoke 不变量。最新构建无 C# warning/error，普通关卡 smoke 和真实 SemIf 模型 F3 锁定目标命中 smoke 均通过。Luna 在冻结后的 HUD 层级接口和关卡结构上完成单文件 UI/文档。 | 地形生成、跳跃物理、相机与通关检查相互耦合，继续由 Sol 定接口和查跨模块根因；Luna 对稳定接口的状态标签、双语文案及事实性文档交付有效。手动路线体验、敌人导航和目标设备性能不能由结构 smoke 代替。 |
| 2026-09-25 | Luna 的刷怪门脚本按冻结的 `Initialize/StopPortal` 接口一次交付；Sol 集成安全生成点后，Unity Windows 构建、真实模型候选选择、门开启、3 秒刷新、首只敌人和关闭清理 smoke 通过。 | 此类独立计时/上限脚本适合 Luna；落地位置、敌人归属、胜负与 HUD 等跨文件状态仍由 Sol 设计和复验。 |
| 2026-09-25 | 地图爆破真实模型 smoke 选择 `demolish_map`，可破坏格 10,250→0、基岩 5,740→5,740；Unity 无 C# 警告/错误。 | 长时间世界操作必须检查实际材料结果、不可破坏边界和可取消/超时行为；模型只选候选不能作为通过证据。 |
| 2026-09-25 | 跳跃输入短按未跨越到落地物理帧会被丢弃；共享默认摩擦还会放大墙边接触。新增输入缓冲/离地宽限和统一零摩擦材质，`-movementSmokeTest` 通过。 | 物理交界由 Sol 修；自动 smoke 能验证缓冲与材料配置，仍需手动连续按空格/贴墙试玩来检查真实手感。 |
| 2026-09-25 | 本地模型已用 bitsandbytes NF4 4-bit + BF16 compute；实测 Windows 日志显示 Qwen3.5 的 FLA/causal-conv1d 快算子缺失并走 PyTorch fallback，Triton 未找到 CUDA Toolkit。单次聊天 6.84 token/s、TTFT 1.267 秒，单次 SemIf `/decide` 前向 1.0595 秒。 | 优化顺序是补齐同版本兼容的快算子并重复测量，然后独立比较 GGUF Q4/Q5；不再建议重复做 4-bit 量化，也不把 GGUF 文件名当作更快的证据。高定制 direct.score 尚需保持语义一致。 |
| 2026-09-25 | 为记录 token 速度新增 stream 指标包；首轮真实聊天 smoke 抓到嵌套 streamer 误引用外层 tokenizer，修正作用域后 Python 编译、Unity 构建、聊天和创世决策 smoke 通过。 | 保留端到端玩家进程 smoke；性能日志是辅助观测，不得改变原 token 流与决策响应协议。 |
| 2026-09-25 | Luna 按冻结 CLI/数据接口交付纯 HTTP benchmark 和 GGUF 候选 evaluator；Sol 审查发现 llama `prompt_n` 实际位于 `timings.prompt_n`，让 Luna 针对该局部数据读取问题修复，之后基准与评测通过。固定短请求数据显示 GGUF decode 更快，但只覆盖 35 输入 token；partial offload prefill 较低，输出长度不同，候选一致率来自全 GPU GGUF 对 BF16 benchmark 参考，不是生产 NF4 游戏质量。HF 仍是生产端，GGUF sidecar 取消/生命周期/Unity 集成未完成。 | 纯 HTTP 基准和单文件离线评测适合给 Luna，接口冻结后可按明确字段返工；跨后端 direct.score 语义、质量门槛、取消与进程生命周期由 Sol 设计、审查和集成。不要据短 prompt 的 token/s 直接宣布后端切换。 |
| 2026-09-25 | Sol 在 `GodDialogueService.StartSidecar()` 将临时文件和运行时缓存环境变量重定向到 modelRoot 的 `benchmarks/runtime-cache`，只对子进程生效；Unity 6000.0.28f1c1 Windows 构建成功，日志为 `Builds/unity-model-cache-check.log`，但游戏进程实际生成缓存路径尚未验证。尝试清理 C 盘安装包时被系统策略拒绝，没有实际清理。 | 环境路径隔离在构建后仍需由游戏进程验证实际写入位置；不要将构建成功等同于缓存路径运行时已验证，也不要把失败的清理尝试记录成已完成。 |
| 2026-09-25 | Sol 阅读 Pi 官方循环、工具、session manager 与 Plan Mode 资料后写下 Harness DSL 初稿；用户随后明确 Python `def/if/for/while` 是语言语法，不是游戏原语。文档已更正为两层设计：SemIf 只在游戏 API 调用点约束 registry ID；项目解析 Python 语法子集，Unity 独立做 AST/状态/预算校验。完整 DSL、整段沙盒和可逆事务账本仍属目标方案。 | 受限脚本语法、Unity 状态解释器和可逆事务属于跨模块架构，由 Sol 冻结接口并集成；文档必须分清语言语法与游戏操作 registry，不能扩展为 SemIf 对每个脚本 token 做单 token 评分。 |
| 2026-09-25 | 根据用户修正，预演区定为同一个 Unity Scene 内、镜头外的一间隐藏试炼舱，不再设计成第二个完整 `GameWorld`。研究与架构页已补上 `SandboxPartitionId/WorldContext`、试炼期间冻结实机、独立地形/实体注册表、通过后只提交脚本 artifact 并在实机重验，以及同场景全局查询会造成计数污染的风险。该设施仍属提案，未进入运行时代码。 | Sol 先解决跨系统世界分区、模拟冻结、原语解释器与撤销边界；接口确定后再拆成独立地形投影、registry 表格/校验器和单文件 HUD 等 Luna 可验收任务。 |

## 提交前检查

1. 文件所有权不重叠，设计文档已反映新接口和运行行为。
2. Unity 无编译错误；若可运行，验证移动、射击、地形破坏、敌人伤害、胜负与重开。
3. 记录 Luna 产出是否一次通过，若返工，写清是实现错误、接口变化还是集成问题。
4. 玩家可见新文案中英成对，中文为首次启动默认值；设置切换、状态提示和两种语言布局均已检查。
