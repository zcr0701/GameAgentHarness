# 可破坏地形架构

本文以 `Assets/Scripts/GridTerrain.cs` 当前实现为准，说明地形状态、Tilemap/碰撞投影以及其他模块应如何请求地形变化。

## 模块职责与边界

`GridTerrain` 是地形格子的唯一权威所有者。其私有 `Material[,] cells` 存放每个单元的材料状态：`Air`、`Rock`、`Sand`、`Bedrock`。数组、材料枚举和底层 `SetCell` 都是私有成员；其他脚本没有也不应获得直接读写数组的途径。

`Tilemap` 只负责把权威格子投影为可见瓦片和 2D 碰撞形状。Tilemap 上创建了 `TilemapRenderer` 与 `TilemapCollider2D`。外部模块若绕过 `GridTerrain` 直接调用 Tilemap 写瓦片，会制造“屏幕/碰撞显示为空、逻辑仍是实体”或反向不一致。

### 哪些模块不能直接改格子

- `TerrainPulse`、`BlastScheduler`、`ProgramCaster`、`PlayerController`、`EnemyChaser`、AI/神谕执行器、测试和 UI 都不得直接访问或新增一份平行 `cells` 数组。
- 玩法模块只能调用地形提供的受控操作；若新操作需要独特规则，应在 `GridTerrain` 中定义边界明确的方法，由它负责同时更新数据和 Tilemap。
- 当前 `WorldToCell`、`CellCenter`、`TryDamageCell`、`DrillSegment` 是 `internal`，可供同程序集中的法术系统等调用；它们不是可跨程序集的公共 API。即使可以调用，调用方也不应自行操作 Tilemap 或保存第二份地形真相。
- `SetCell` 只允许地形内部使用；不要把它改成 public 作为通用快捷方式。这样会把材料规则、边界策略和投影一致性责任推给所有调用方。

## 数据和投影初始化

格网为 `1024 × 160`，每格边长 `0.5` 世界单位；地图总尺寸约 `512 × 80`，扣除边界后的可玩范围约 `510 × 78`。原点仍以世界中心对称计算，`PlayableMinX/MaxX/MinY/MaxY` 从网格边界动态推导，回响使用 `ClampPlayablePoint` 将爆心限制在可玩区域。`GameWorld.Start()` 创建 `GridTerrain` 后调用 `Initialize()`。初始化顺序是：创建 Grid/Tilemap 和 TilemapCollider2D，生成材质瓦片，生成权威材料数据，重建沙粒索引，最后 `PaintAll()` 一次性将约 16.4 万个格子投影到同一个 Tilemap。当前没有按镜头分区加载。

生成器写入底层起伏岩层、地图边界基岩、五个房间基准地板、四段上升阶梯、固定种子平台、五处可挖岩门和各层沙堆。五层地板行是井口 `FloorCell(x)`、32、56、80、104；阶梯平台宽 5 格，中心左右各 2 格，末端与上层房间地板重叠接合。路线平台/沙堆/岩门使用固定随机种子 `84173` 或固定格子坐标，令相同版本和配置可重复生成，便于测试复现和人工定位。`SetInitial()` 限制初始结构写入范围。`TileFor()` 按材料返回对应瓦片；岩石/沙子的外观变化由坐标确定，因而同格重绘不产生随机材质跳变。`Air` 对应 `null` tile，基岩不参与挖掘。

## 修改接口与调用流

| 接口 | 可见性 | 当前作用 | 返回/边界 |
| --- | --- | --- | --- |
| `Initialize()` | public | 初始化 Tilemap、材料表并绘制完整初始地形。 | 应由世界启动阶段调用一次；源码没有重复初始化保护。 |
| `FloorY(float worldX)` | public | 返回给定 X 附近的基准地表高度，供出生点/地形定位使用。 | 依据程序化 `FloorCell` 地表，不会追踪被挖洞或沙子滑落后的即时表面。 |
| `DamageCircle(Vector2 center, float radius, int damage = 1)` | public | 将圆形范围映射到候选格子；对圆内格子尝试挖除。 | 仅正半径、正 damage 有效；返回是否至少改变一格。当前 `damage` 仅作为有效性开关，尚非多段耐久值。 |
| `WorldToCell(Vector2)` / `CellCenter(int,int)` | internal | 世界坐标与格子索引/中心互转，供同程序集算法使用。 | `WorldToCell` 不钳制范围；调用者必须检查边界。 |
| `TryDamageCell(int,int)` | internal | 只允许挖除 Rock/Sand，成功后置 Air 并更新 Tilemap。 | 超界、Air、Bedrock 均返回 false。 |
| `DamageCellsBatch(Vector2Int[], int)` | internal | 对一批候选格逐个执行边界/材料检查，在 `cells` 与沙索引更新后，一次 `Tilemap.SetTiles` 提交已改变的格子。 | 单批最多 96 个候选；越界、Air、Bedrock 会跳过。返回实际改变格数。 |
| `DrillSegment(Vector2 start, Vector2 end, float radius)` | internal | 沿线按小于半格的采样间距，重复调用圆形挖掘，避免只破坏线段端点。 | 返回该线段是否有任一挖掘发生。 |

标准修改链路：

```text
玩法系统确定效果中心/线段
  → 调用 GridTerrain.DamageCircle / DrillSegment / TryDamageCell
  → 地形检查边界及材料是否可挖
  → 私有写入路径更新 cells[x,y] 与派生沙索引
  → 单格操作用 Tilemap.SetTile；爆炸批次用 Tilemap.SetTiles
  → TilemapCollider2D 更新相应碰撞投影
  → Unity 物理步观察更新后的碰撞
```

实际使用点包括：`TerrainPulse` 使用线段钻掘及圆形开挖；`BlastScheduler` 逐帧扫描爆炸区域，把圆内格子收集到最多 96 格的缓冲区，再调用 `DamageCellsBatch`；`GameWorld` smoke 通过公开 `DamageCircle` 验证地形改动。调用方选择作用范围和时机，材料资格及写入仍归 `GridTerrain`。

## 物理同步与模拟边界

- `SetCell()` 先改私有数据，再调用 `tilemap.SetTile()`，让渲染和碰撞来自同一个投影。不要只修改 `cells` 或只修改 Tilemap。
- Unity 对 TilemapCollider2D 的形状更新及物理查询存在物理步时序。刚创建或大量改地形后，如需立即验证射线/角色接地，测试应跨过 `WaitForFixedUpdate`，而不是假定 `SetTile()` 调用返回时所有物理查询都已经完成。
- 当前直接挂载 `TilemapCollider2D`，源码没有设置 `CompositeCollider2D`，也没有手写独立地形碰撞网格。新增碰撞优化需同步评估角色接地、钻脉贯穿和爆炸后通行性。
- `FixedUpdate()` 每累计约 0.10 秒移动一轮沙子；沙格位置由 `sandCellIndices` 维护。每轮只复制当前沙粒索引到复用的 `sandScanBuffer` 并排序，不扫描整张 1024×160 网格；行优先顺序保证下方先处理，单颗沙子每轮最多移动一格。横向扫描方向按 tick 交替，移动优先直落再向左右斜落。所有加入、移除沙子的状态变化仍经 `SetCellData` 更新数组与索引，单格移动再经 `SetCell` 更新 Tilemap。
- `TerrainPulse` 使用新版 `Physics2D.CircleCast` 固定结果数组和默认可查询层筛选器；Unity 返回按距离排序的命中，满 64 项时回退完整查询。普通路径不创建命中数组，但满缓冲回退仍可能分配。
- 爆炸地形任务由 `BlastScheduler` 跨帧分段处理：每帧地形遍历预算最多 320 格、每轮每个 job 最多收集 96 个圆内候选格；TerrainJob 与 `GridTerrain` 复用批量缓冲数组，改动格子通过一次 `Tilemap.SetTiles` 投影。这样避免每个爆炸格都单独调用 `SetTile`，并限制大半径爆炸单帧地形工作的上限。预算统计的是扫描格数，不是实际被破坏格数。
- 游戏结束时沙子更新停止。当前没有其他粒子材料、液体、气体，也没有回滚或网络复制层；不要把未来材料模拟能力假定为已实现。

## 扩展约束

1. 新增可破坏材料时，明确它是否可被 `TryDamageCell` 移除、如何映射 Tile、是否参与重力/扩散、如何与碰撞同步；材料规则集中在 `GridTerrain`。
2. 新增伤害规则时，当前 `damage` 并非耐久强度。若要不同材料多次受击，应先定义权威耐久数据和公开操作语义，再改 `TryDamageCell`，不能让投射物私自减少某个影子耐久表。
3. 新增面积效果，可扩展 `DamageCircle` 周边的受控操作或在其上组合；注意按格中心距离筛圆与按格 AABB 判定会有不同边缘效果，改规则时用固定场景验证。
4. 新增线形效果应集中采样/覆盖策略，保证连续覆盖，不要在每个法术脚本复制不同的格子遍历并直接写 Tilemap。
5. 对 AI 或 Agent Harness，只暴露如“请求指定半径挖掘”这类经过规则审查的受限接口；由更高层执行器校验预算和许可，再调用地形 API。模型不应拿到可写数组、Tilemap 引用或任意材质赋值能力。

## 性能设计与不变量

- 地图扩大后仍保留一份 `Material[Width, Height]` 作为逻辑权威。可见瓦片、TilemapCollider2D 和 `sandCellIndices` 都是从该状态同步维护的投影/索引，不可各自决定材料结果。
- 沙索引的一致性不变量：`sandCellIndices` 恰好包含所有且仅包含当前 `Sand` 格子的 row-major 索引 `y * Width + x`。初始化后重建一次；之后材料变化必须经 `SetCellData` 同步增删。不要从外部直接改数组，也不要只增删集合。
- 批量爆破先在调度器中生成候选格，再由 `DamageCellsBatch` 再次检查边界与可破材料。候选集合不代表成功破坏；返回值才是实际改变数。逻辑数据与沙索引先更新，随后 Tilemap 批量投影，保证性能优化没有创建第二份权威状态。
- 固定种子让初始长地图可复现，但不是存档随机种子系统；若以后提供种子配置/版本迁移，应显式保存生成参数并维护兼容策略。
- `DamageCellsBatch` 的 96 格上限、调度器的 320 格扫描预算和每帧最多两个爆破事件共同限制单帧工作。调高地图规模或破坏半径时，应保持分帧处理；如需再优化，先采集实际帧耗时、Tilemap collider 重建耗时和沙粒数量分布，再考虑区块 Tilemap、增量碰撞或沙区唤醒等方案。
- 目前没有按镜头范围卸载地形，也没有区块化数据/渲染/碰撞；扩大地图后整张 Tilemap 仍一次初始化和绘制。不能把上述候选预算描述成完整的世界分区或动态加载系统。

## 验证方式

- 初始化后确认边界为不可挖基岩、地板/平台为岩石、沙堆显示正确，角色出生点落在基准地面附近。
- 调用圆形挖掘：确认返回值反映是否真的挖掉 Rock/Sand；重复挖掘已空格、边界基岩或无效半径不应伪报变化。
- 调用钻掘线段：跨越多格并检查路径连续；该实现以 `CellSize * 0.25` 为最大采样步长，回归时特别验证复合 Tilemap collider 内部不漏挖。
- 物理步后进行射线/移动检查：地形被挖后玩家可通过空洞，未挖处仍可站立；沙子移动后原格与新格的显示和碰撞一致。
- 使用项目现有玩家 smoke 检查地形修改/同步和钻脉穿透；若单独添加地形测试，不要在验证过程中直接改 Tilemap 绕过 `GridTerrain`。
- 对五层长地图回归，应检查 1024×160 边界、固定种子生成可复现、五层房间地板、四段阶梯每个 5 格宽落脚面、层间接合、平台/沙堆/材料门、玩家上下边界与回响中心钳位。玩家进程 smoke 已验证尺寸、层/阶梯落脚面存在及四敌关卡闭环；连续手动跳跃体验仍需人工验证。对性能回归，应记录沙粒数量及每轮处理量、爆炸候选扫描量/批次数与单帧耗时，并确认批量破坏后 Tilemap/碰撞在物理步后吻合。目标设备性能尚未测量；除非测试日志明确记录，不应据文档推断帧率。

## 常见错误

- **直接写数组或 Tilemap**：破坏唯一权威和视觉/碰撞一致性。所有写入必须回到 `GridTerrain`。
- **把 `FloorY` 当成当前可站立表面查询**：它是生成地形时的基准曲线，不能报告挖掘后的洞、平台或动态沙面。
- **把 `damage` 理解成扣除耐久量**：现实现里它只判断是否大于零，符合条件的 Rock/Sand 一次就变 Air。
- **认为基岩是普通岩石**：`Bedrock` 不可由当前 `TryDamageCell` 移除；地图边界依赖此保护。
- **把 `WorldToCell` 当作已限制坐标**：函数可能返回负索引或超过网格范围，任何后续索引前都要校验。
- **在物理步前断言碰撞结果**：Tilemap 数据已改不代表当前帧物理查询已重建；跨固定步复验。
- **从调用方缓存地形真相**：调用方可保留一次操作结果用于调度/日志，但不可维护另一份会影响游戏规则的占用状态。
