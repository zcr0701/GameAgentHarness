# TODO

更新时间：2026-09-25。代码已写但尚未经过本轮 Unity 构建/运行验证的项目仍保持未完成状态。

## P1：神谕 Agent Harness 与隐藏成就

### 当前代码，待 Unity 验收

- [ ] 编译并运行地下隐藏房间：检查密封空间、玩家进入才解锁“神的房间？/God's Room?”，NPC/法术不能触发。
- [ ] 检查普通成就、隐藏成就分类、未解锁时问号显示、`J` 打开/关闭、英文切换与 PlayerPrefs 跨重开持久化。
- [x] `-harnessSandboxSmokeTest`：确认错误的强敌前置条件被拒绝，预演通过时不改变实机地形、敌人数、门户、施法资源；过期 revision 也会拒绝。
- [x] `-creationStrongPortalSmokeTest` 使用本地 Q4_K_M 模型验收“无穷无尽的强大敌人”：先建门并观察刷新，再切强敌 profile，验证后续生成的强敌生命、伤害、移速和 3 秒刷新节奏。

### 下一阶段：完整 DSL、试炼与可撤销执行

- [ ] 实现 Python 语法子集 AST：`def`、`if/else`、`for`、有界 `while` 是语言语法，不是游戏原语；SemIf 只在写入游戏 API 调用时约束可用原语 ID。Unity 的专属 parser/validator 负责参数、类型和预算，不运行 CPython/JavaScript 或任意代码。
- [ ] 验证 SemIf 与 llama.cpp 是否能在游戏 API 调用位置把原语 ID 限定为当前 registry 项；不对 Python 语法的每个 token 做候选评分。测一次通过率、合法率和延迟；现有 `direct_messages` + 单 token 候选评分不能标记为该 DSL 输出头。
- [ ] 将目前单步数据投影预演升级为整份脚本在独立试炼上下文预演，断言全部通过后才提交同一 artifact/hash 到实机；实机仍按相同步骤实时验收。
- [ ] 为每个副作用原语实现版本化 undo/compensation journal 和失败回滚；无法恢复地形/角色状态的操作不得加入可撤销脚本候选。
- [ ] 控制 AST 节点、嵌套、重试、循环次数、等待时间、生成速率、存活实体、总时间及取消路径；不允许无限对象增长。

完整状态与接口见[神谕操作 Harness 架构](docs/architecture/ai-harness.md)、[操作原语目录](docs/architecture/action-primitives.md)和[成就机制](docs/EASTER_EGG_MECHANIC.md)。外部实现仅作架构研究，见[Pi Agent Harness 源码研究](docs/PI_AGENT_HARNESS_RESEARCH.md)。
