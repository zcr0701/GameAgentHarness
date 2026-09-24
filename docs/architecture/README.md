# Ember Hollow 架构索引

更新时间：2026-09-25。此目录是代码导航与设计边界的入口。阅读时以源码和自动检查为准；每个页面同时标记已实现内容与后续建议。

## 按模块阅读

| 模块 | 代码入口 | 架构说明 | 主要所有者 |
| --- | --- | --- | --- |
| 关卡与角色 | `GameWorld.cs`、`PlayerController.cs`、`EnemyChaser.cs` | [关卡与角色](world-and-actors.md) | Sol 定接口；清楚的单文件可交 Luna |
| 地形 | `GridTerrain.cs` | [地形数据与物理投影](terrain.md) | Sol；数据/碰撞一致性跨模块 |
| 法术程序编译 | `ProgramCaster.cs`、`ProgramEditor.cs` | [程序编译器与编辑器](program-compiler.md) | Sol 定语义；单文件 UI 可交 Luna |
| 法术命中与爆破 | `TerrainPulse.cs`、`BlastScheduler.cs`、`BlastVisual.cs` | [法术执行流水线](spell-execution.md) | Sol 负责跨模块执行与验证 |
| 本地模型与游戏操作 Harness | `GodDialogueService.cs`、`GodDialogueServer.py`、`GodActionHarness.cs`、`HarnessSandboxChamber.cs` | [神谕操作 Harness](ai-harness.md)、[游戏操作原语目录](action-primitives.md) | Sol 负责协议、授权边界与验证 |
| HUD、窗口与语言 | `GameHud.cs`、`GameSettingsMenu.cs`、`GameLocalization.cs`、`GodChatWindow.cs` | [表现层与本地化](presentation-localization.md) | 冻结接口后的单界面可交 Luna |
| 普通/隐藏成就与隐藏房间 | `AchievementService.cs`、`AchievementJournal.cs`、`AchievementToast.cs`、`EasterEggRoomTrigger.cs` | [成就机制](../EASTER_EGG_MECHANIC.md) | Sol 定成就和触发边界；冻结 UI 接口后的单文件表现可交 Luna |

## 新接手者推荐路径

1. 阅读根目录 [README.md](../../README.md)，建立运行和构建方式的概念。
2. 阅读 [DESIGN.md](../DESIGN.md)，了解当前游戏结构、里程碑和全局约束。
3. 按待改文件打开对应模块页；先核对模块页中的“当前实现”，不要把“扩展建议”误当成已有代码。
4. 涉及玩家可见文案时，遵循 [本地化规范](../LOCALIZATION.md)：中文默认，新增文本同时提供英文并使用 `GameLocalization.T`。
5. 涉及模型操作时，先读 [神谕操作 Harness](ai-harness.md)；模型排序不构成授权，最终规则检查和执行必须留在 Unity 主线程。
6. 涉及法术 DSL 或 AI 操作 ID 时，再读[游戏操作原语目录](action-primitives.md)，不要混淆两层权限。
7. 改完同步模块页、[DESIGN.md](../DESIGN.md) 与根目录 [AGENTS.md](../../AGENTS.md)，并记录实际构建/玩家进程检查结果。

待办与验收状态见根目录 [TODO.md](../../TODO.md)；普通/隐藏成就、密室触发与隔离规则见[机制说明](../EASTER_EGG_MECHANIC.md)。

## 外部 Agent Harness 研究

- [Pi Agent Harness 代码研究](../PI_AGENT_HARNESS_RESEARCH.md)：Pi 的核心回合/工具循环、扩展 schema 与权限边界、会话摘要，以及本项目对应的 Unity 约束和差异。

## 设计文档维护规则

- 一份模块页对应一个清晰的代码边界，写明职责、入口、调用关系、权威状态、不可破坏的不变量、扩展方式与验证路径。
- 跨文件流程可以在多个模块页中用链接交叉引用；数据所有权和写入权只在其唯一权威模块详细定义。
- 记录“当前实现”时引用仓库代码和实测；提案用“建议”或“后续”标注。
- 外部实现只作为架构研究材料，不复制其代码、素材或专有内容。Pi 的公开 Agent Harness 研究结论见 [神谕操作 Harness](ai-harness.md)。
