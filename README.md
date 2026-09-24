# Ember Hollow（暂定名）

Unity 6000 侧视 2D 可破坏地形动作原型。当前已有可玩的单人闭环和“地脉程序”编辑模式，之后逐步接入受约束的本地 AI 决策系统。

## 当前开发进度（2026-09-25）

- 已完成五层关卡、可破坏地形、组合式法术编辑器、中英双语设置，以及本地 Qwen3.5-4B Q4_K_M GGUF 对话/决策服务和旧对话自动压缩。
- `<作弊码>` 创世操作使用 Unity 登记的有限原语。Agent Harness 已支持逐步决策、固定参数预演、真实世界执行与状态验收；真实模型 smoke 已验证“创建传送门 → 切换强敌配置 → 验证后续刷出强敌”。
- 同一 Unity 场景镜头外设有隐藏试炼舱。当前是固定原语的单步数据投影预演，不是独立物理世界，也不能预演任意多步脚本。
- 普通成就与隐藏成就已接入本地原型存档；Steamworks 尚未接入。
- Python 风格语法子集 parser/interpreter、整段脚本隔离试炼和通用撤销账本仍是后续工作。`def/if/for/while` 属于脚本语法；游戏操作和查询单独登记为原语，SemIf 只约束游戏 API 调用点。

最新 Windows 构建、普通玩法 smoke、扩展版 `-harnessSandboxSmokeTest` 和真实 Q4_K_M 模型 `-creationStrongPortalSmokeTest` 均通过。详细现状、接口边界和待办见[Agent Harness 架构](docs/architecture/ai-harness.md)、[游戏原语目录](docs/architecture/action-primitives.md)及[TODO](TODO.md)。

当前关卡为 1024×160 格，单格 0.5 Unity 单位，地图约 512×80、可玩范围约 510×78 世界单位。玩家沿灰烬井口、回响矿层、蓝晶裂谷、断炉工坊和天脊核心逐层上升，经过四段阶梯、五处可破坏岩门，并击败四名哨兵后抵达顶层出口。首名哨兵仍在开局神谕组合的搜索距离内。地形使用单个 Tilemap；沙粒按沙格索引更新，爆破按帧预算批量写入。目标设备上的帧耗时基线仍待采集。关卡剖面、坐标和实测状态见[五层关卡设计](docs/LEVEL_DESIGN.md)。

## 打开与运行

公开 Demo 的双语操作说明见 [Demo 说明](docs/DEMO_README.md)。

1. 用 Unity 6000.0.28f1c1 打开本目录。
2. 打开 `Assets/Scenes/Prototype.unity`，点击 Play。
3. A/D 移动，空格跳跃，鼠标瞄准，按住左键运行当前程序；Tab 打开或关闭程序编辑器，X 停止持续回响，R 重开。右上角“设置”可切换界面语言。

游戏内文案首次启动默认简体中文；设置中可切换为英语，并自动保存选择。新增玩家可见文本须同时提供中英两版，规范见[游戏内语言文档](docs/LOCALIZATION.md)。

编辑器可添加、调序、删除分叉、扩张、增幅、爆破、回响、回能、钻脉、延时、震荡和发射模块，也可载入基础、持续回响、大范围破城与掘地震波预设。地形方块由8×8、64个细粒子构成，材料含岩石、沙、水和岩浆，并使用数值硬度、密度、内聚力、粘度和壁面附着参数。黑洞可在限时内把方块、粒子和生物沿直线吸入；保存地形/生物快照后可撤销。模块排列自动保存在本机。高范围与高伤害有上限，持续回响通过逐帧调度执行，可随时按 X 取消。

`Ember Hollow/Create Prototype Scene` 可重建场景；批处理入口为 `GameProjectSetup.BuildWindows`，输出在 `Builds/EmberHollow.exe`。玩家可按 `F2` 和神谕对话，按 `F3` 请求“地脉冲锋”登记组合；在聊天开头输入 `<作弊码>` 可请求地图爆破、刷怪门、强敌配置、黑洞、材料/粒子填充、冲击波、回响风暴、无限能量或钻脉。所有操作都只能从 Unity 提供的有限候选中选择，并在执行前重新检查局面。

玩家程序加 `-smokeTest` 会自动检查地图跨度、出生/出口与敌人分布、挖掘与碰撞刷新、程序持续回响与取消、大范围破坏、敌人与胜利状态。`-movementSmokeTest` 检查跳跃缓冲和零摩擦材质；`-creationStrongPortalSmokeTest` 用本机模型验证“无穷无尽的强大敌人”；`-creationBlackHoleSmokeTest` 用本机模型验收“将世界放逐到虚空”；`-harnessSandboxSmokeTest` 检查隔离预演；`-blackHolePhysicsSmokeTest` 验证64粒子/块、材料硬度阈值、黑洞无视硬度吞噬基岩及地形/生物快照撤销；`-mapDemolitionSmokeTest` 验证地图清理及基岩边界保留。模块设计文档入口为 [docs/architecture/README.md](docs/architecture/README.md)。

工程架构、当前范围、接口与后续计划见 [设计文档](docs/DESIGN.md)；模块语义与执行预算见 [程序系统文档](docs/PROGRAM_SYSTEM.md)。参考作品的公开技术资料与本作原创方向见 [研究笔记](docs/NOITA_RESEARCH.md)。子代理分工与经验记录见 [AGENTS.md](AGENTS.md)。

最初的课题设想存档于[本科毕设选题方案](docs/references/本科毕设选题方案.md)，该文档记录研究提案，不代表当前实现状态。

Pi Agent Harness 官方源码研究与本项目的架构映射见 [Pi 代码研究](docs/PI_AGENT_HARNESS_RESEARCH.md)。

源码文件入口、模块职责、主要调用链和测试定位见[源码解读](docs/源码解读.md)。发布给外部试玩的 Demo 仅含 Windows 运行时文件，不打包 Unity/C#/Python 项目源码，也不含本地模型权重；神谕功能需开发环境中的 sidecar 与模型服务。

