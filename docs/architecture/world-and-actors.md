# 世界与角色运行时架构

本文描述当前源码中的 `GameWorld`、玩家和敌人实现，并说明它们与已落地 `GodActionHarness` 的交界。只把源码中实际存在的接口记为当前能力。

## 文件职责

| 文件 | 职责 | 不应承接的职责 |
| --- | --- | --- |
| `Assets/Scripts/GameWorld.cs` | 创建本局场景对象，持有世界级服务引用和胜负状态，累计击杀，提供死亡、胜利、重开入口；同文件还包含出口门、相机跟随和程序化原型美术。 | 不承担地形格子的读写实现（由 `GridTerrain` 负责），不解析法术程序（由 `ProgramCaster` 负责），不应把 UI 文案散落在世界状态代码中。 |
| `Assets/Scripts/PlayerController.cs` | 玩家刚体的键鼠输入、地面检测、跳跃、朝向、射击调用，以及玩家生命值与受伤入口。 | 不直接实现弹体或法术效果；射击委托给同对象上的 `ProgramCaster`。 |
| `Assets/Scripts/EnemyChaser.cs` | 敌人生命与死亡、追逐玩家、地面/障碍检测、跳跃越障、接触伤害、受击击退。 | 不直接改变地形或结算全局胜负；死亡通过 `GameWorld.RegisterEnemyKill()` 登记。 |

## 创建与生命周期

### 世界启动

`GameWorld.Awake()` 设置静态 `Instance`；重复世界对象会被销毁。`Start()` 是当前原型的组合根：重置 `Time.timeScale` 和 2D 重力，创建五层背景、初始化 `GridTerrain`，再创建玩家、法术施放器、爆炸调度器、四只分布在路线上的敌人、顶层出口、相机和 HUD/编辑器/设置/神谕服务。这个工程当前由代码动态构造关卡对象，不依赖场景里预摆好的玩家与敌人。

重要顺序：地形先初始化，玩家和敌人再依据 `Terrain.FloorY(x)` 放置；玩家创建完毕后添加 `ProgramCaster`；最后 `Dialogue.BeginRun()` 开始本局模型服务状态。新增依赖某对象的组件时，应确认创建顺序和引用已就绪，而不是依赖 `FindObjectOfType` 偶然找到对象。

### 每帧及物理调用

```text
GameWorld.Start
  ├─ 初始化 GridTerrain
  ├─ 创建 Player + ProgramCaster
  ├─ 创建 EnemyChaser × 4 / Skyspine GoalPortal
  └─ 创建 HUD、ProgramEditor、设置、GodDialogueService、GodActionHarness、GodChatWindow

PlayerController.Update ──输入采样/瞄准/鼠标射击──> ProgramCaster.TryCast
PlayerController.FixedUpdate ──落地检测/水平速度/跳跃──> Rigidbody2D
EnemyChaser.FixedUpdate ──落地与障碍检测/追逐和越障──> Rigidbody2D
EnemyChaser 碰撞/触发 ──冷却后接触伤害──> PlayerController.TakeDamage
敌人死亡 ──> GameWorld.RegisterEnemyKill ──> GoalPortal.Refresh
玩家生命归零 ──> GameWorld.OnPlayerDeath ──> ProgramCaster.StopAll
神谕请求 ──> GodActionHarness 快照/暂停 ──> GodDialogueService.RequestDecision
  ──有限候选评分──> GodActionHarness 重校验 ──> ProgramCaster 注册组合
  ──> 观察目标受击及爆炸完成
玩家进入出口且击杀数达标 ──> GameWorld.WinRun ──> ProgramCaster.StopAll
```

`GameHud.Update()` 接收 F3 并调用 `ActionHarness.RequestFaultlineCharge()`；HUD 也提供双语按钮入口，并根据玩家位置显示五层中的当前层名。`GameWorld.Update()` 和 `Restart()` 都会在动作 harness 忙碌（决策或观察操作结果）时拒绝重开，避免异步请求和本局状态在中途被卸载。`CameraFollow2D.LateUpdate()` 读取玩家 Transform，将镜头水平和垂直限制在地形可玩范围；地图高度增加后，相机可以随玩家逐层上升。

## 权威状态与不变量

- **本局权威**：`GameWorld.Instance` 是当前本局服务入口；`Kills`、`IsGameOver`、`HasWon` 是私有 setter 的世界状态。其他模块应调用世界方法，不应自行改写全局胜负或击杀数。
- **胜负规则**：`RequiredKills = 4`。四只哨兵位于灰烬井口（开局目标）、回响矿层、断炉工坊和天脊核心；蓝晶裂谷当前没有敌人。击杀最多累计到 4；顶层出口外观由击杀数刷新。`WinRun()` 还会再次检查击杀门槛；死亡或胜利后置 `IsGameOver` 并调用 `Caster.StopAll()`。
- **玩家权威**：`PlayerController.Health` 私有 setter，最大值 `MaxHealth = 5`。`TakeDamage` 忽略非正伤害、已死亡和本局已结束的情况；归零时通知 `GameWorld`。
- **敌人权威**：敌人内部 `health` 是私有值，初始值至少为 1；公开只读 `Health` 和 `IsAlive`，外部不能改写。`TakeDamage` 处理死亡并销毁对象，先向 `GameWorld` 登记击杀。HUD/AI 可读 getter 判断目标有效性，但生命值写入仍只能走受控伤害入口。
- **物理判定**：玩家落地射线只接受非触发、非自身、法线向上且属于 `GridTerrain` 的碰撞体；敌人的地面/障碍射线排除触发器、角色碰撞体。地形与物理同步应由地形模块维护，不要以角色检测补丁绕过地形碰撞同步。
- **输入暂停边界**：玩家明确在游戏结束、`ProgramEditor.IsOpen` 或 Harness 忙碌时清掉移动/跳跃输入；跳跃输入有 0.12 秒缓冲和 0.1 秒离地宽限。玩家与 Tilemap 地形共用摩擦/弹性均为 0 的运行时物理材质，消除默认摩擦导致贴墙吸附的风险。编辑器、设置、聊天和操作 Harness 执行时暂停物理。
- **神谕操作边界**：`GameWorld.ActionHarness` 是唯一模型决策执行桥；F3 固定组合之外，聊天前缀 `<作弊码>` 可以提交有限创世候选。Unity 筛选候选并重验玩家/关卡快照，模型不能生成坐标或数值参数。登记 ID 与上限见[游戏操作原语目录](action-primitives.md)；聊天未带前缀时仍只是文本对话。

## 主要接口

### `GameWorld`

- 只读访问器：`Instance`、`Terrain`、`Player`、`Caster`、`Blasts`、`Editor`、`Settings`、`Dialogue`、`ActionHarness`、`Kills`、`IsGameOver`、`HasWon`。
- 状态入口：`RegisterEnemyKill()`、`OnPlayerDeath()`、`WinRun()`、`Restart()`。
- `Restart()` 在 `ActionHarness.IsBusy` 时无操作；世界 Update 和 HUD 入口也避开动作忙碌期。
- 命令行 smoke：`-smokeTest` 执行基础集成流程，可选 `-capture` 截图、`-modelSmokeTest` 检查本地模型聊天；`-movementSmokeTest` 检查低摩擦材质和落地前跳跃缓冲；`-actionHarnessSmokeTest` 检查固定组合；`-creationActionSmokeTest` 验证刷怪门和首只敌人；`-mapDemolitionSmokeTest` 验证全图可破坏格清理且基岩保留（AI action 测试都需要本机模型）。Action smoke 单独执行，避免普通 smoke 改写目标、程序和地形状态。

### `PlayerController`

- 外部可读 `Health`、`MaxHealth`；外部伤害入口 `TakeDamage(int amount)`。
- 通过 `GetComponent<ProgramCaster>()` 发射编译程序。瞄准依赖 `Camera.main` 和鼠标屏幕坐标；没有相机时瞄准回退朝右。
- 空格按下进入 `jumpBufferDuration` 计时窗口，离开地面后保留 `coyoteDuration` 宽限；默认分别为 0.12 秒与 0.1 秒。地面射线长度、内缩量、移速、跳跃速度、冷却和枪口偏移均为 Inspector 序列化参数。
- 与 `GridTerrain` 的 TilemapCollider 共用运行时零摩擦/零弹性材质；由 `GameWorld` 设置，不应在角色脚本自行创建一份材质。

### `EnemyChaser`

- 外部只读状态：`Health`、`IsAlive`；外部入口：`TakeDamage(int amount)`、`ApplyKnockback(Vector2 source, Vector2 directionHint, float impulse)`。
- `ApplyKnockback` 仅对动态刚体及正冲量生效，并限制冲量及短暂追逐抑制时长。
- `ShouldHoldTarget()` 由 `GodActionHarness` 查询：模型决策等待时暂缓敌人水平追逐；执行观测阶段锁住选定目标，直至确认其生命下降（或观察流程结束），减少瞄准期间目标移动带来的不稳定。
- `VelocityForSmoke` 是 `internal` 的 smoke 辅助观测，不是面向玩法系统的公共状态 API。
- `maxHealth`、`moveSpeed`、`stopDistance`、`jumpForce`、射线距离和接触伤害冷却由 Inspector 参数控制。

## 扩展建议

1. 新增玩家技能时，让输入层提交意图给技能/施放系统；不要从 `PlayerController` 内直接改地形或复制程序编译规则。
2. 新增敌人类型可复用伤害/追逐接口，但死亡登记必须保持恰好一次。若需要不同胜利条件，将规则集中放到 `GameWorld` 或专门的规则对象，不要让每个敌人自行判断通关。
3. 新增 AI/神谕动作时，沿用 `GodActionHarness` 的只读快照、封闭候选、确定性重校验和注册命令入口；世界端校验生命周期、目标有效性、距离、资源和当前局面。不得把自由生成内容变成权威状态，也不要让模型绕过 `ProgramCaster` 或 `GridTerrain`。
4. 新增玩家可见文案必须中英成对，并经 `GameLocalization.T(中文, English)` 选择；不要在角色逻辑中缓存只支持一种语言的显示文本。
5. 若增加玩家/敌人状态，明确唯一写入者和只读接口。HUD、模型提示和 smoke 检查都应读取该接口，避免各自维护互相矛盾的影子状态。

## 验证

- Unity 编译后启动场景：检查玩家和四只敌人的出生位置均基于已初始化地形，摄像机跟随并限制 1024×160 地图的水平/垂直边界。
- 实机输入：左右移动、落地跳跃、鼠标瞄准射击；验证编辑器打开时移动/跳跃被阻断、关闭后恢复。
- 伤害流程：敌人接触伤害受冷却控制；玩家生命归零后世界失败状态生效并停止施法队列。
- 胜利流程：击败四敌后天脊核心出口刷新，再进入出口触发胜利；未达到击杀数时不能通过 `WinRun()` 直接获胜。当前玩家 smoke 已覆盖四敌计数及胜利状态，玩家沿阶梯实际到达顶层的手动流程仍待验证。
- 回归命令行：Windows 玩家构建使用 `-smokeTest`；移动修复使用 `-movementSmokeTest`；模型聊天另加 `-modelSmokeTest`；固定组合、创世门与全图爆破分别使用独立 `-actionHarnessSmokeTest` / `-creationActionSmokeTest` / `-mapDemolitionSmokeTest`，等待决策、重校验及真实世界结果。AI smoke 会直接使用本地模型，应单独运行。

## 常见跑偏点

- 把 `GameWorld` 当成所有系统的实现文件继续堆代码：它是本局组合根和胜负权威，不应吞并法术编译、地形存储或 UI 交互。
- 以为击杀数上限就是胜利：仍需进入出口并由 `WinRun()` 检查门槛。
- 以为暂停时间会阻止所有 `Update()` 逻辑：`Time.timeScale = 0` 会停物理推进，但 Update 仍会运行；输入门禁要在拥有输入的脚本里明确实现。
- 通过公开 `EnemyChaser.Health` / `IsAlive` 读取敌人状态；不要尝试写入只读 getter 或再维护一份不同步的敌人生命副本。
- 用敌人射线命中任意刚体作为“障碍”：实现刻意排除了玩家、其他敌人和触发器；调整时需同时验证相互阻挡、越障和地面接触。
- 把有限候选评分说成模型绝不会判断错误：SemIf 限制选择范围，不证明所选操作正确。Unity harness 的候选、重校验和执行器才是操作边界；新增操作须先注册命令并加入确定性校验，不能开放任意模型命令。
