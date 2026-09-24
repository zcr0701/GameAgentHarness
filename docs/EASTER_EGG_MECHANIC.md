# 成就系统与隐藏房间成就

更新时间：2026-09-25。本文记录普通成就、隐藏成就的分类和“神的房间？”触发流程。架构入口见[神谕操作 Harness](architecture/ai-harness.md)。

## 玩家体验

- 成就页分为“普通成就”和“隐藏成就”，整体参考 Steam 的展示习惯。
- 未解锁的隐藏成就标题和说明显示为 `？？？/???` 与“尚未解锁/Undiscovered”；普通成就始终显示目标说明。
- 首次进入隐藏试炼房间时弹出双语解锁提示“神的房间？/God's Room?”，随后在成就页显示详情。
- 按 `J` 打开/关闭成就页，界面暂停游戏；成就分类、提示和描述均通过 `GameLocalization.T(中文, English)` 即时切换语言。

## 当前代码状态

`GameWorld` 在五层地图下方生成封闭的隐藏房间和玩家触发区；只有当前 `PlayerController` 可以调用 `TryTriggerRoomEasterEgg`。`AchievementService` 持有普通/隐藏目录并用 `PlayerPrefs` 去重和持久化，`AchievementToast` 显示解锁提示，`AchievementJournal` 显示列表。当前目录有“第一次胜利/First Victory”和隐藏项“神的房间？/God's Room?”。这是本地原型存档，**尚未接入 Steamworks 成就 API**。

本轮代码刚完成集成，以下 Unity 实际检查仍列为待办：

- 挖开岩顶进入后只解锁一次；普通 NPC、法术弹、神谕试炼操作都不能触发。
- 关闭并重新启动后成就仍保留；隐藏项在解锁前显示问号。
- 中文默认、切到英文后标题/描述/解锁弹窗即时变更，窗口布局没有裁切。
- 进入后可用回返传送门离开，不因发现成就困在密室。

## 与 AI 试炼隔离

物理房间只提供玩家探索和成就触发。`EasterEggRoomTrigger` 检查碰撞对象的父级 `PlayerController`，再由 `GameWorld` 核对它是当前玩家。AI 的 `HarnessSandboxChamber` 使用独立逻辑分区 ID 和数据投影，不复用玩家触发器，也不因解锁成就获得额外 Unity 权限。两者在同一 Scene，但其用途、状态与调用入口分离。

当前 AI 沙盒可以预演固定登记原语的单步状态投影；它不是物理房间里的 NPC/敌人演练，也不是一套独立 Tilemap/Physics2D。完整脚本 DSL、整段沙盒验收以及实机 undo journal 仍在 TODO 中。

## 对应实现

| 职责 | 代码 |
| --- | --- |
| 地图密室与触发/回返入口 | `GridTerrain.cs`、`GameWorld.cs`、`EasterEggRoomTrigger.cs`、`HarnessRoomExitPortal.cs` |
| 成就目录、存档和去重 | `AchievementService.cs` |
| 普通/隐藏成就列表 | `AchievementJournal.cs` |
| 解锁弹窗 | `AchievementToast.cs` |
| AI 独立预演上下文 | `HarnessSandboxChamber.cs`、`GodActionHarness.cs` |

