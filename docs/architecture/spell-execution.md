# 法术弹体、爆破调度与表现

更新时间：2026-09-24。负责从已编译的脉冲规格到命中、伤害、地形变化和视觉反馈的运行链。

## 文件边界

- `Assets/Scripts/TerrainPulse.cs`：创建并更新弹体，处理弹体寿命、钻脉路径、碰撞目标、直接伤害和爆破请求。
- `Assets/Scripts/BlastScheduler.cs`：有界处理引信事件、爆破伤害、回响和分帧地形任务；是爆破/地形工作队列的唯一调度者。
- `Assets/Scripts/BlastVisual.cs`：爆破的短生命周期显示，不承担碰撞、伤害或材料变化。
- `Assets/Scripts/ProgramCaster.cs`：提供 `PulseSpec`、费用与 `RunToken`，其编译器架构见 [程序编译器](program-compiler.md)。
- `Assets/Scripts/GridTerrain.cs`：实施格子破坏和钻脉；地形所有权见[地形模块](terrain.md)。

## 执行顺序

1. `ProgramCaster` 把一个或多个 `Emission` 变成带明确来源 Transform 的 `TerrainPulse`。
2. 弹体沿 `PulseSpec` 速度移动。钻脉弹体每帧沿线采样 `GridTerrain.DrillSegment`，再用 `Physics2D.CircleCast` 的复用结果数组处理遇到的单位或剩余实体碰撞。
3. 命中敌人或可爆破实体时，将命中点、飞行方向和 `PulseSpec` 提交给 `BlastScheduler.QueueBlast`。引信以 `Time.time + FuseSeconds` 计时。
4. 调度器每帧最多触发两个事件；确认当前施法器引用和 `RunToken` 有效后，显示 `BlastVisual`，增加完成计数，对范围内敌人执行伤害/冲击，并启动一个有界地形任务。
5. 地形任务每帧共享最多 320 格预算，每一步至多检查 96 格。完成后可排入下一代回响；回响收费、令牌与队列容量再次校验。

## 地形批写与碰撞查询

- `BlastScheduler` 每帧仍受 320 格共享地形预算约束；每个批次最多收集 96 个候选格。它将候选位置交给 `GridTerrain.DamageCellsBatch`，由地形组件集中修改材料数据，并使用一次 `Tilemap.SetTiles` 同步该批次的显示/碰撞投影。`BlastScheduler` 不直接写 Tilemap。
- `GridTerrain` 复用批次位置与 Tile 数组，空材料对应 `null` Tile，用于清除已破坏的格子。批写只改变可破坏材料，仍经由地形唯一写入点。
- `TerrainPulse` 使用新版 `Physics2D.CircleCast` 的静态 64 项结果数组和接触筛选器。Unity 按距离排列结果，因此无需额外排序；缓冲填满时回退到 `CircleCastAll` 保留完整命中语义，普通路径避免分配，极端命中密度下仍可能分配。筛选器跳过触发器并沿用默认可查询层，目标类型与发射来源仍由脚本做语义校验。
- 沙粒模拟由 `GridTerrain` 的沙格索引集合驱动，排序后的可复用扫描列表确保每个 tick 只遍历沙格且每粒最多移动一次；索引在格子材料变化时同步维护。
- 当前地图为 1024×160 格（单格 0.5 Unity 单位），并仍由一个 Tilemap 承载。上述预算限制单帧爆破地形工作量，不代表已完成性能基准测试；扩大范围、沙格密度或碰撞密度时仍需在目标设备上测量。

## 权威接口

- `TerrainPulse.Spawn(Vector2, Vector2, Transform source, PulseSpec payload)`：来源必须能唯一映射到一个玩家或敌人组件。
- `BlastScheduler.QueueBlast(Vector2 center, Vector2 direction, PulseSpec spec)`：拒绝无施法器请求或已达到 64 项的队列，并将半径、伤害、引信与冲击数值夹到游戏上限。
- `GridTerrain.DrillSegment(...)` / `DamageCircle(...)`：格子数据唯一写入接口。调度器只预算和调用，不直接改 Tilemap。
- `EnemyChaser.TakeDamage(int)` / `ApplyKnockback(...)` 与 `PlayerController.TakeDamage(int)`：单位生命和物理反馈接口。

`TerrainPulse` 对 `Drill + Blast` 组合有一次性终止规则：命中或弹体寿命到期时，只能提交一次爆破请求，随后将弹体标记为已消费并销毁。弹体的 `Update` 不能在过期后的每一帧重复排入爆破。

## 必须保持的不变量

1. Unity Physics2D、Tilemap、敌人、玩家与调度器只在 Unity 主线程/物理阶段访问；Python 推理线程不引用这些对象。
2. 爆破队列和地形任务都有上限。扩大地图或爆破范围时，重新估算单帧格数预算、排队数与取消行为。
3. 触发的爆破会比较 `PulseSpec.Token` 与发射器 `RunToken`。`StopAll()` 必须能取消等待引信与回响，避免旧程序继续改变世界。
4. 爆破半径、伤害、冲击、延时受规则层 clamp；未来来自模型的参数也不得绕过这些上限。本项目现在甚至不接收模型数值参数。
5. 钻脉与范围爆破通过 `GridTerrain` 唯一写入口更新数据，再更新显示和碰撞；禁止由粒子或视觉脚本改地形。
6. `CompletedBlasts` 表示爆破开始结算；只有地形队列清空后，`OutstandingWork` 才归零。测试若要求完整完成，需同时检查二者。
7. 爆破视觉失败不影响实际伤害与地形更新；视觉脚本不应成为规则依赖。

## 扩展与测试

- 新增弹体类型时，用 `PulseSpec` 或新的显式强类型载荷表达行为，不要在 `TerrainPulse` 里加入模型回复文本解析。
- 新增爆破工作时，对事件数量、每帧预算、失效令牌、重开/死亡清理逐项做边界测试。
- 现有玩家进程 smoke 验证钻脉穿透、延迟爆破、冲击速度、回响取消和大范围破坏。AI 组合 smoke 使用真实模型评分，验证一次性爆破、1–3 次爆破上限、锁定目标实际受伤和玩家程序未被改写；该用例曾发现弹体到期重复排队并触发 64 项队列/636 次爆破，修复后回归通过。
- Unity 修改后至少做一次 Windows 构建；跨物理和 Tilemap 改动继续做玩家进程检查及截图/材质状态核验。

## 常见跑偏点

- 在 `TerrainPulse` 中直接修改地形数组或让它绕开 `BlastScheduler` 分帧处理大范围任务。
- 用 `FindAnyObjectByType` 或世界坐标猜测弹体所有者；必须沿显式 `source` 与 `PulseSpec.Caster` 传递来源。
- 将回响当成普通弹体的下一帧操作；它受预算、能量、token 与地图边界共同约束。
- 认为弹体命中等于爆破已经完成，忽略引信和地形 job。
- 将 `BlastVisual` 当成伤害/地形代码的入口，造成视觉帧率变化影响规则结果。
