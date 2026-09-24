# Noita 技术研究与本项目独立设计推导

更新时间：2026-09-24。目标是研究**公开**的工程接口与设计取舍，提炼适合本项目的原则；这里不收录、移植或改写 Noita 的代码、素材、法术或关卡内容。事实与本项目推断分列。

## 可核实的公开范围

- Nolla Games 的[公开模组仓库](https://github.com/NollaGames/noita-modding)包含模组文档。它不是 Falling Everything 引擎源码仓库。[Lua 文档页](https://github.com/NollaGames/noita-modding/blob/master/Lua-scripting.md)指向游戏安装目录中的 API 文档；[入门页](https://github.com/NollaGames/noita-modding/blob/master/Getting-started.md)提到随游戏提供的示例模组与数据工具。
- 开发者 Petri Purho 在[访谈](https://www.gamedeveloper.com/game-platforms/road-to-the-igf-nolla-games-i-noita-i-)中说明：引擎用 C++，刚体使用 Box2D，脚本使用 Lua，程序生成使用 Wang Tile 工具。公开资料未提供足以审查其核心像素模拟实现的完整 C++ 源码，因此我们不能声称已分析其私有引擎内部代码。
- [GDC 讲座页](https://gdcvault.com/browse/gdc-19/play/1025695)列出了大规模沙粒模拟、连续世界、可破坏刚体与涌现玩法等主题；这是研究方向，不等于公开实现细节。

## 公开接口能说明什么

[官方模组基础文档](https://github.com/NollaGames/noita-modding/blob/master/Modding-basics.md)列有 `ModLuaFileAppend(to_filename, from_filename)`、`ModMagicNumbersFileAdd(filename)` 和 `ModMaterialsFileAdd(filename)`。从**接口表面**可确认：扩展点被分成脚本追加、全局参数配置和材料配置。为什么这么分层，官方短文档未直接解释；我们推断这让内容扩展与引擎核心相隔离，且可让材料作为数据配置参与多个系统。不能由函数名反推底层 C++ 的具体实现。

开发者访谈中的另一个要点是玩法验证：物理模拟复杂度本身不会自动产生有趣的战斗，倒塌建筑的试验经常只留下阻路碎石。因此本项目将每一种材料反应都绑定到玩家可感知的决策（开路、设陷阱、改变敌人路线），并用小关卡原型测试其可读性。[来源：开发者访谈](https://www.gamedeveloper.com/game-platforms/road-to-the-igf-nolla-games-i-noita-i-)

## 我们的原创方向：地脉工程

这是**本项目提案**，并非对 Noita 内容的概述。角色使用工程工具探索可变地下生态，核心组合由“材料属性 × 地形结构 × 工具模块 × 敌人行为”产生，而不是复现魔杖/法术系统。

1. **材料属性**：每格材料逐步具有承载、渗透、导流、传振等少量属性。M0 只有岩石、沙、边界；未来按玩法需求增加，不追求全图逐像素拟真。
2. **工具模块**：玩家组合“切槽、压实、导流、激振”等工程动作。动作影响地形属性或传播路径，形成开路、落砂、导流、诱敌等连锁结果。M0 的单发挖掘弹只是输入与地形闭环验证，不是最终工具系统。
3. **地脉与敌人**：关卡放置可被地形改变的信号节点；敌人对压力/声波/路径变化做反应。玩家可用同一组工具构筑路线或陷阱。
4. **肉鸽变化**：每局随机组合材料分布、节点位置、敌人生态和工具模块。随机化优先改变可推演的因果关系，而非仅随机数值。
5. **AI 边界**：后续本地 AI 只能从引擎给出的合法工程动作候选中排序/选择。状态只读、动作校验与主线程执行分离，和设计文档中的 Agent-Harness 目标对齐。

### 下一次原型验证

先实现两种工具效果与一条环境连锁，例如“切槽移除支撑 → 沙粒下落 → 压力节点被触发”。观察玩家是否能预测并主动利用该链路；若不可读，优先改反馈和关卡布置，再增加模拟复杂度。

