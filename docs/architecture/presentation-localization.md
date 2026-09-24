# 界面与本地化架构

> 范围：记录 `GameHud.cs`、`GameSettingsMenu.cs`、`GameLocalization.cs`、`GodChatWindow.cs` 的表现层边界。神谕动作的规则与执行归属见 [神谕操作 Harness](ai-harness.md)。事实以代码为准；文中“计划”表示当前尚未实现。

## 职责边界

- `GameLocalization` 是界面语言状态的单一来源：管理简体中文/英语、`PlayerPrefs` 持久化、双语文案选择、UI 字体和法术模块名称。
- `GameHud` 显示游戏内状态、操作提示、结算覆盖层，以及设置和神谕的入口。它不持有游戏状态，也不处理对话生成。
- `GameSettingsMenu` 显示设置覆盖层，目前只提供界面语言切换；负责打开时暂停以及关闭时恢复此前的时间缩放状态。
- `GodChatWindow` 显示神谕聊天界面、采集玩家输入，并把提交/取消/开关操作转交给 `GodDialogueService`。它不实现模型推理或游戏规则。
- 四个模块均使用 Unity IMGUI 绘制；没有引入独立 UI 框架或 UI 包。

## 代码文件映射

| 文件 | 类型/入口 | 主要依赖与输出 |
| --- | --- | --- |
| `Assets/Scripts/GameLocalization.cs` | 静态类 `GameLocalization`；`Current`、`UiFont`、`T`、`SetLanguage`、`ModuleName` | 依赖 `PlayerPrefs` 和 Unity 系统字体；为界面及法术模块名提供当前语言文本与字体 |
| `Assets/Scripts/GameHud.cs` | `MonoBehaviour`；`Update`、`Awake`、`OnGUI` | 从 `GameWorld.Instance` 读取玩家、击杀、施法器、设置、编辑器、对话和动作 Harness 状态；提供 F2 对话与 F3 登记组合入口 |
| `Assets/Scripts/GameSettingsMenu.cs` | `MonoBehaviour`；公开 `IsOpen`、`Toggle()` | 读写 `GameLocalization`；通过 `Time.timeScale` 暂停/恢复；遵守编辑器、对话和动作执行期间的互斥条件 |
| `Assets/Scripts/GodChatWindow.cs` | `MonoBehaviour`；`Update`、`OnGUI` | 读写 `GameWorld.Instance.Dialogue` 的开关、消息、状态、生成状态和操作方法；文本经 `GameLocalization` 显示 |

## 运行与输入流

### HUD 与设置

1. `GameHud.OnGUI()` 确认世界和玩家存在，并在设置、程序编辑器或神谕窗口打开时隐藏 HUD。
2. HUD 从 `GameWorld` 读取生命、击杀进度、能量、法术程序步数、持续施法状态及最后状态提示。底部显示键位提示；游戏结束时显示胜负文字和重开按钮。
3. HUD 右上角的设置按钮调用 `GameSettingsMenu.Toggle()`。神谕按钮调用 `GodDialogueService.ToggleOpen()`。
4. HUD 的“神谕组合”按钮与 F3 只调用 `GodActionHarness.RequestFaultlineCharge()`；HUD 不拼候选、不选目标、不提供动作参数。
5. 设置面板打开时保存当前 `Time.timeScale` 并设为 `0`。点击语言按钮立即调用 `GameLocalization.SetLanguage()`，切换状态并保存到 `PlayerPrefs`；关闭面板或按 Escape 时恢复时间缩放。
6. 如果程序编辑器仍打开，关闭设置后强制保持暂停（`timeScale = 0`）；否则恢复打开设置前的时间缩放值。

### 神谕窗口

1. `GodChatWindow.Update()` 在设置未打开时监听 F2；F2 通过服务切换窗口状态。打开窗口后清空尚未提交的输入框。窗口打开时 Escape 关闭对话。
2. `OnGUI()` 只在对话服务存在且 `IsOpen` 时绘制。聊天记录按消息角色区分“你/神谕”，使用可滚动区域并在用户处于底部时跟随新内容。
3. 输入上限为 500 个字符。输入框获得焦点时按 Return，或点击发送按钮，将去除首尾空白的非空文本交给 `Dialogue.Submit(text)`；提交后清空输入框。生成期间发送入口改为停止按钮，调用 `Dialogue.Cancel()`。
4. 状态行根据 `GodDialogueStatus` 显示加载、就绪、回应中、结构化决策中、不可用或错误的双语提示。生成和模型通信由服务负责，窗口不参与请求格式或推理。

## 公开协作接口

- `GameLocalization.Current`：当前 `GameLanguage`。枚举目前只有 `Chinese` 与 `English`。
- `GameLocalization.T(string chinese, string english)`：根据当前语言选取文本。所有新增玩家可见文案应提供成对中文和英文。
- `GameLocalization.SetLanguage(GameLanguage)`：切换并立即持久化语言；非法枚举值被忽略。
- `GameLocalization.UiFont`：供 IMGUI 样式使用的动态系统字体，按 Microsoft YaHei、SimHei、SimSun、Arial Unicode MS 顺序尝试。
- `GameLocalization.ModuleName(ModuleKind)`：法术模块统一名称映射。添加模块名称时在这里同时提供中英文。
- `GameHud` 依赖的协调接口：`GameWorld.Instance` 上的 `Player`、`Kills`、`RequiredKills`、`Caster`、`Settings`、`Editor`、`Dialogue`、`ActionHarness`、`IsGameOver`、`HasWon`、`Restart()`；施法器状态由 `ProgramCaster` 提供。
- `GameSettingsMenu.IsOpen` / `Toggle()`：供 HUD、程序编辑器等入口查询和切换设置面板。
- `GodChatWindow` 使用的对话服务接口：`IsOpen`、`IsGenerating`、`IsDeciding`、`Status`、`Messages`、`ToggleOpen()`、`Close()`、`Submit(string)`、`Cancel()`。消息以 `GodDialogueMessage` / `GodDialogueRole` 表示。更改聊天协议时同步检查窗口调用。
- `GameHud` 调用 Harness 的 `RequestFaultlineCharge()`，显示只读 `StatusText` 与 `IsBusy`；候选校验、程序编译和施法均不属于表现层。

## 必须保持的不变量

1. 首次启动的默认语言为简体中文；只接受中文/英语，语言选择保存在 `PlayerPrefs`，运行时切换立即作用于所有读取 `T` 的文案。
2. 玩家可见文案在同一改动中提供中文和英文，并统一调用 `GameLocalization.T`；不要在各 UI 脚本中复制语言选择或持久化逻辑。
3. 动态状态应保存为状态/数值，再由当前语言渲染，不能只保存已翻译的一种字符串。模块名经 `ModuleName` 统一映射。
4. 设置、编辑器、聊天和动作决策不得同时控制游戏。动作 Harness 忙碌时拒绝打开设置/聊天、程序编辑和关卡重开；HUD 本身在设置、编辑器或聊天覆盖层出现时隐藏。
5. 设置关闭时尊重程序编辑器的暂停状态；不能无条件把 `Time.timeScale` 恢复成 `1`。
6. 聊天窗口只负责展示和转发输入/取消。F3 由 HUD 转发到 `GodActionHarness`；不得从 IMGUI 绘制代码直接修改地形、生命或施法资源。动作接口和校验边界见专门的 Harness 文档。
7. IMGUI 控件绘制期间避免改变上述游戏状态；开关和命令由按钮/输入事件交给既有协调接口处理。

## 扩展方式

- **增加可见文案**：在对应控件处写成 `GameLocalization.T("简体中文", "English")`；检查插值字符串两种语言中的标签、单位和数值一致。
- **增加语言**：当前 `GameLanguage`、`T`、存储读取/校验和设置界面按钮都假设只有中文/英语。需要整体扩展这些位置，并明确旧存档和未知值的回退规则；当前实现没有多语言表或资源文件系统。
- **增加设置项**：将显示与状态归属明确后扩展 `GameSettingsMenu`，继续遵循暂停恢复规则；设置面板当前没有通用设置注册机制。
- **增加聊天展示状态**：在服务 `GodDialogueStatus` 与本窗口 `StatusText()` 保持一一映射，并提供双语文案。长回复应继续使用按内容测量的高度与滚动区域。
- **增加 HUD 状态**：从现有世界/系统接口读取并格式化，不让 HUD 成为状态权威；检查 HUD 与设置、编辑器、对话、动作执行和结算覆盖层的绘制关系。
- **增加神谕操作入口**：界面只负责触发一个已登记请求和显示状态；更新候选/前置条件/反馈时修改 `GodActionHarness` 及 [Harness 设计文档](ai-harness.md)，不要把执行逻辑加入 HUD。

## 验证方法

- Unity Windows 构建，检查无 C# 编译错误/警告。
- 运行检查：首次启动默认中文；切到英语立即更新；重启后选择仍保留；非法语言值回退中文。
- 设置检查：从 HUD 和程序编辑器进入；打开时暂停；Escape/返回后按原暂停状态恢复；设置与神谕互斥。
- HUD 检查：核对生命/击杀/能量/程序状态、设置入口、F2 对话与 F3 组合提示、动作状态、胜负覆盖层与重开按钮。
- 神谕检查：F2 开关、Escape 关闭、空白输入不发送、Return/发送按钮提交、生成中可取消、流式长回复可滚动且底部跟随正常。
- 分别检查中文和英文窗口截图，确认字体字形、按钮宽度、提示换行和长回复排版。多分辨率、非 Windows 字体回退及完整人工键鼠体验仍须单独验证。

## 常见跑偏点

- 在 UI 脚本写 `language == ... ? ... : ...` 的私有翻译表，导致新状态或页面漏翻译。
- 将设置或聊天窗口当成游戏状态服务；它们只是展示层，状态读取/命令要经既有系统接口。
- 设置关闭时无条件恢复 `timeScale = 1`，破坏编辑器等其他暂停原因。
- 让 F2/Escape 在设置覆盖层打开时仍同时操作神谕，破坏输入焦点和互斥关系。
- 把服务状态枚举直接 `ToString()` 作为玩家提示；新增状态必须补充中英文映射。
- 假设聊天回复固定一行，导致长消息被裁剪；继续保留动态高度、滚动和自动跟随行为。
- 将“神谕”标题当前中英文相同误判为本地化遗漏；这是现有有意呈现的双语品牌标题。
- 将模型候选分数当成游戏规则校验；表现层按钮只触发 Harness，候选和实际效果由规则模块确认。

