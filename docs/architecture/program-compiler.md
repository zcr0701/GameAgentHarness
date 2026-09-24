# 法术程序编译器与编辑器

更新时间：2026-09-24。此页面向接手 `ProgramCaster.cs`、`ProgramEditor.cs` 与法术模块语义的开发者。

## 文件职责

- `Assets/Scripts/ProgramCaster.cs`：持有玩家编辑的模块序列、按序编译、能量收费、施放、回响世代令牌、存档，以及若干玩法状态提示。
- `Assets/Scripts/ProgramEditor.cs`：暂停时展示与编辑模块序列；不保存第二份程序状态，所有改动直接调用 `ProgramCaster`。
- `Assets/Scripts/GameLocalization.cs`：模块名和程序相关静态 UI 文案的中英映射。
- `Assets/Scripts/GodActionHarness.cs`：引用一个不可变的已注册组合入口；它不通过 UI 编辑器改玩家程序，细节见 [神谕操作 Harness](ai-harness.md)。

## 设计模型

玩家程序是至多 24 个 `ModuleKind` 组成的有序序列。`Compile` 按顺序读取修饰模块，状态累计到下一个 `Emit`；每个 `Emit` 产出一个独立 `Emission`，然后重置修饰状态。当前模块轴包括分叉、扩张、增幅、爆破、回响、回收、钻脉、引信、冲击与发射。

编译结果是运行时的 `PulseSpec + Count + Cost` 列表，而不是可执行脚本或任意 C#。`TryCast` 先编译玩家序列；`TryCastFaultlineCharge` 编译 Unity 内部固定的 `FaultlineChargeSequence`，施放后恢复玩家序列的编译结果。该操作不会写入玩家 `modules` 列表或 PlayerPrefs。

程序的组合含义由编译器唯一解释。例如 `Fork → Drill → Blast → Fuse → Expand → Impulse → Emit` 会得到三道钻脉爆破脉冲、0.55 秒引信、半径按一次扩张计算、5 点冲击，并按编译规则收费。`FaultlineChargeEnergyCost` 由同一个编译器动态计算，避免动作 Harness 复制成本公式。

## 调用流

```text
ProgramEditor 输入
  → ProgramCaster.TryAdd / Move / RemoveAt / LoadPreset
  → SaveProgram（仅玩家程序）

玩家鼠标发射
  → ProgramCaster.TryCast
  → Compile(modules)
  → 能量预检与扣除
  → TerrainPulse.Spawn

神谕注册组合
  → ProgramCaster.TryCastFaultlineCharge
  → Compile(FaultlineChargeSequence)
  → 同一施法校验与生成路径
  → Compile(modules) 恢复当前预览
```

## 公开协作接口

- `Modules`：只读访问器；修改必须走 `TryAdd`、`RemoveAt`、`Move` 或 `LoadPreset`。
- `Preview`：由当下玩家程序重新编译得到摘要；访问时会更新内部临时编译列表。
- `TryCast(Vector2 origin, Vector2 direction)`：运行当前玩家序列。
- `TryCastFaultlineCharge(Vector2 origin, Vector2 direction)`：运行固定登记组合；方向由调用者提供，当前由 `GodActionHarness` 计算并锁定目标。
- `FaultlineChargeEnergyCost`：固定登记组合通过同一编译器计算出的总消耗。
- `StopAll()`：递增 `RunToken`，使已排队的回响和相关法术失效；任何变更程序序列的编辑都应通过它取消旧链。
- `TrySpendEcho(float)` / `RestoreEnergy(float)`：仅供爆破调度器、钻脉回收等执行结算使用。

## 必须保持的不变量

1. `ProgramCaster` 是玩家模块序列、能量和回响令牌的唯一所有者；编辑器和 AI 不复制这些可变状态。
2. 费用必须在生成弹体之前一次性校验并扣除；失败不应生成部分弹体或扣除部分费用。
3. 每个 `Emit` 重置修饰状态，修改顺序或重置规则时要同时更新预览、存档和 smoke 断言。
4. 编译得到的 `PulseSpec` 会复制给每个弹体，弹体保留施法器引用和当时的 `RunToken`；不要由坐标推断施法者。
5. 当前面向模型的操作只能指向预注册组合 ID，不能让模型传任意 `ModuleKind[]`、伤害、半径、数量、世界坐标或价格。
6. 改变玩家已保存程序仍须通过现有编辑操作；神谕的临时组合不得覆盖存档或当前模块预览。

## 扩展与验证

- 增加模块时，同时更新 `ModuleKind` 编译语义、费用、预览描述、`GameLocalization.ModuleName`、程序编辑器调色板和双语提示。
- 任何可由 AI 选择的新组合都应先登记为 Unity 内部定义，再加到有限候选表；不能暴露数组反序列化入口。
- Unity 玩家进程检查覆盖模块顺序、回响取消、钻脉/引信/冲击组合、大范围费用限制。操作 Harness smoke 还应确认临时组合施放成功且 `Modules` 序列不变。
- 修改费用、模块上限或组合定义后，运行 Windows 构建、常规 `-smokeTest` 和真实模型的 `-actionHarnessSmokeTest`。

## 常见跑偏点

- 在 `ProgramEditor` 中维护第二套模块列表或单独保存神谕程序。
- 只改模块名或 UI 按钮，却遗漏编译器、费用和预览语义。
- 在模型端拼装组合数组，或者把单 token 候选评分误认为执行校验。
- 增加模块后仍用旧的固定耗能常量；本项目用 `FaultlineChargeEnergyCost` 从编译器计算。
- 将所有 `Emit` 修饰理解成“影响整串程序”；当前规则是修饰到最近的下一个 `Emit`。

