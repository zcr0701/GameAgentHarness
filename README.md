# GameAgentHarness / Ember Hollow

本仓库公开项目设计文档、研究资料与开发进度。Unity 游戏源码保存在单独的私有仓库 `zcr0701/UnityGameAgentHarness`。

Ember Hollow（暂定名）是一款 Unity 6、侧视 2D、可破坏地形的高组合性动作肉鸽原型。玩家通过“地脉程序”编辑器组合法术，并可与本地神谕 Agent Harness 交互。

## 当前进度（2026-09-25）

- **可玩关卡与法术：**1024×160 格、五层纵向关卡、四名哨兵、可破坏地形和组合式法术编辑器；中文默认，设置可切换英文。
- **本地模型：**Qwen3.5-4B Q4_K_M GGUF 通过 llama.cpp 本地服务运行；游戏接入对话、决策和旧对话自动压缩。当前验证配置为 `ngl=5`、`ctx=4096`、`threads=8`，额外显存约 0.8 GiB。
- **Agent Harness：**`<作弊码>` 请求只能从 Unity 登记的操作中选择。真实 Q4_K_M 模型已通过“创建传送门 → 配置强敌 → 验证后续刷新属性”的多步验收；每步记录脚本 hash、预演和实机回执。
- **试炼舱：**隐藏在同一 Unity 场景镜头外。当前对固定原语做单步数据投影预演，再在真实世界逐步执行和验收。
- **成就与语言：**普通/隐藏成就、本地原型存档、成就列表和中英切换已集成；尚未接入 Steamworks。

## 最近验证

2026-09-25 的 Unity Windows 构建成功，未发现 C# 编译警告或错误。普通玩法 smoke、扩展版 `-harnessSandboxSmokeTest`、真实本地模型 `-creationStrongPortalSmokeTest` 均通过；真实模型地图爆破与固定组合 smoke 也已通过。

## 后续工作

- 实现安全的 Python 语法子集 parser/interpreter。脚本中的 `def/if/for/while` 是语言结构；游戏操作、效果模块和权威查询单独登记为游戏原语。SemIf 仅在游戏 API 调用点约束可用原语 ID，不对整份脚本逐 token 评分。
- 将单步数据投影扩展为整段脚本的隔离试炼，并补齐可撤销事务记录。当前还没有完整物理沙盒或通用 undo 栈，部分破坏性原语不可完整撤销。
- 补齐各个创世原语的独立模型验收、实机 UI/碰撞检查和目标设备性能测量。

## 文档

- [设计总览](docs/DESIGN.md) · [五层关卡](docs/LEVEL_DESIGN.md) · [程序与法术系统](docs/PROGRAM_SYSTEM.md)
- [Agent Harness 架构](docs/architecture/ai-harness.md) · [游戏原语目录](docs/architecture/action-primitives.md) · [架构文档索引](docs/architecture/README.md)
- [Pi Harness 源码研究](docs/PI_AGENT_HARNESS_RESEARCH.md) · [Noita 设计研究](docs/NOITA_RESEARCH.md)
- [本地模型运行方案](docs/GOD_DIALOGUE.md) · [早期模型选型方案](本地模型选型与AI决策方案.md)
- [游戏本地化](docs/LOCALIZATION.md) · [成就机制](docs/EASTER_EGG_MECHANIC.md)
- [后续计划](TODO.md) · [协作记录](AGENTS.md) · [灵感备忘](灵感.MD)

“灵感备忘”只记录尚待讨论的想法，不代表已排期或承诺加入。
