# 神谕本地对话服务

更新时间：2026-09-25。此页说明游戏内神谕对话的运行架构、模型来源、消息边界、本机启动条件，以及 GGUF Q4_K_M 生产后端的取舍。

## 玩家入口

- HUD 右上角“神谕 / ORACLE”按钮或 `F2` 打开聊天窗；`Esc` 关闭。
- HUD 右上角“神谕组合 / ORACLE COMBO”按钮或 `F3` 请求一次已登记组合评分；动作执行和校验与聊天分开。
- 打开时游戏模拟暂停，后台本地推理继续运行。
- 对话支持中英文。当前游戏语言会同时决定界面文字、系统提示和模型回复语言。
- 输入后逐段显示模型回复；生成中可点击“停止生成”。`R` 在聊天窗打开时不会重开关卡。
- 关卡重开会清空本局聊天记录，但保留已加载的模型进程，避免重复占用显存。

## 模型与本机依赖

游戏复用 `D:\WorkData\SemIf` 中 SemIf（OpenJev 接口研究项目）使用的 **Qwen3.5-4B**，不下载新的对话模型，也不把权重放入游戏仓库。当前生产权重为 `benchmarks/qwen3.5-4b-q4_k_m.gguf`（约 2.78 GB），由同目录 `benchmarks/llama-b11160/llama-server.exe` 推理；原 Safetensors、配置与分词器仍保留在模型目录中，供本地 tokenizer 和 SemIf 规范提示构造使用。

游戏侧用 SemIf `.venv` 启动轻量 Python HTTP 适配器；Transformers 仅加载本地 tokenizer，不加载模型权重到 CUDA。llama.cpp 以 `--gpu-layers 5`、单并行槽、4096 token context 启动。当前 RTX 4060 Laptop 8 GB 的实测总显存从约 1.9 GiB 基线升到约 2.8 GiB，增量约 0.8 GiB；开发配置保持在用户给定的约 1 GiB 增量内。显存数字仅适用于这台机器与当前二进制/上下文配置。

## Transformers 与 GGUF 选型

“Transformers 格式”常把两件事混在一起说：Transformers 是 Hugging Face 的 Python 模型加载/推理框架；本机模型的权重序列化文件是 safetensors，旁边还有模型配置、分词器和聊天模板。GGUF 则是一个包含张量与标准化元数据的二进制模型文件格式，通常由 llama.cpp 等 GGML 推理引擎读取；GGUF 文件本身不是推理引擎。[Transformers 自定义生成接口](https://huggingface.co/docs/transformers/generation_strategies)可直接改生成流程，[GGUF 格式说明](https://huggingface.co/docs/hub/gguf)描述其文件内容；[llama.cpp 模型文档](https://github.com/ggml-org/llama.cpp/blob/master/docs/models.md)说明该引擎要求 GGUF。

| 维度 | Transformers + safetensors（历史基线） | GGUF + llama.cpp（当前生产） |
| --- | --- | --- |
| 本机现状 | 原 SemIf NF4 路径保留，作为历史速度比较与模型参考；不再由游戏启动 | Unity 启动本地 Python 适配器及 llama.cpp 子进程，加载 Q4_K_M；聊天、候选评分、取消、关停及 F3 操作 smoke 已通过 |
| 定制模型行为 | 最方便深入访问模型对象、生成步骤、logits、缓存和 Python 自定义策略；适合研究决策、候选动作评分和特化生成 | 可在服务协议、采样器和受约束解码层定制；若要改张量执行或模型图，主要进入 C/C++/GGML 后端 |
| 推理部署 | Python/PyTorch 依赖较重；目前日志显示 Qwen3.5 的部分线性注意力算子回退到较慢的 PyTorch 参考实现 | 原生 C/C++ 服务和量化内核通常更适合轻量本地部署；具体速度取决于转换质量、量化档位、GPU 后端与模型版本，不能只凭格式判断 |
| 游戏引擎连接 | 历史路径同样使用本机 HTTP | Unity 仍使用稳定 loopback HTTP 契约；Python 适配层启动/监视/关闭 llama.cpp，正常退出时先请求 `/shutdown`，由适配层回收推理进程 |

生产后端已切换为 GGUF Q4_K_M。底层 Unity 协议保持后端无关：`/chat` 逐段回传纯文本；`/decide` 只对 Unity 送来的有限候选计算候选 token 概率并归一化，不生成自由动作文本。游戏动作链路仍是“局面快照 → 模型对受限候选评分 → 规则校验器核验版本、前置条件、成本与作用范围 → Unity 主线程执行类型化命令 → 以事件结果反馈”；概率格式本身不代替游戏规则校验。llama.cpp 的支持能力与验证状态见本页后续记录。[Qwen3.5 转换实现](https://github.com/ggml-org/llama.cpp/blob/master/conversion/qwen.py) · [llama.cpp 量化说明](https://github.com/ggml-org/llama.cpp/blob/master/tools/quantize/README.md) · [推测解码](https://github.com/ggml-org/llama.cpp/blob/master/docs/speculative.md)。

### token 生成速度建议（2026-09-25）

**当前策略：保持 Q4_K_M 质量档，用推理路径提速，不继续降到 Q4_0。** 已测 35-token 短请求中 Q4_K_M 解码中位数约为 Transformers NF4 的 2.10×；144 条直接候选集上，Q4_K_M 与 BF16 参考 argmax 一致 131/144（90.97%），高于 Q4_0 的 128/144（88.89%）。一致率是比较指标，不是正确率或游戏动作安全证明。

已部署的 llama.cpp 配置为 `--gpu-layers 5 --parallel 1 --ctx-size 4096 --flash-attn auto --threads 8`，保持 Q4_K_M 权重；玩家进程的一次短聊天实测 13.48 token/s、TTFT 0.487 秒，模型总显存增量约 0.8 GiB。GPU 分层不改变 GGUF 权重档位；Flash Attention 由后端自动选择实现，浮点执行可能不逐位相同，所以关键候选决策仍须通过 Harness 前置条件和动作结果检查。

在额外显存增量不超过 1 GiB 的约束下，已补测 CPU 线程数、对话前缀缓存和 `ngram-mod`。线程 4/8/12 的 96-token 贪心回复 decode 中位数分别为 14.31/14.61/13.17 token/s，固定提示输出哈希一致；8 线程是这组配置中吞吐最好的，12 线程更慢，生产配置继续保留 8。缓存探索测试的六轮 prompt 处理中位数是 629 ms 对 477 ms，但对话生成在第二轮后分叉，后续输入不再完全相同，故该差值只作方向性观察。更关键的是，贪心复测在同一轮、相同 token 输入下出现不同输出，因此暂不打开 `cache_prompt`，需先查清模型状态缓存与输出差异。`ngram-mod` 的单组六轮 seeded 样本较缓存组 decode 中位数快约 3.4%，但没有得到对无缓存基线的干净质量对照，尚不足以证明稳定收益或质量不退化，也不作为生产默认。

llama.cpp 的推测解码通过目标模型验证 draft token，`ngram-mod` 不需另载 draft 权重、但依赖上下文重复。它们仍可作为候选优化，但启用前要对照当前 `cache_prompt=false` 基线，重复测同一批中英文真实对话、生成 token 数、稳定 token/s、显存和人工质量；本轮结果不足以批准启用。若要显著提高量化保真度，可独立比较 Q5_K_M/Q6_K，但会增大权重文件和内存使用，且不是当前已验证的提速方向。[llama.cpp speculative decoding](https://github.com/ggml-org/llama.cpp/blob/master/docs/speculative.md) · [量化档位与误差说明](https://github.com/ggml-org/llama.cpp/blob/master/tools/quantize/README.md)

### 固定请求 A/B 与候选质量检查（2026-09-25）

本轮用 `tools/model_backend_benchmark.py` 比较固定短聊天请求，并用 `tools/evaluate_gguf_decisions.py` 检查 GGUF 候选评分。评测统一使用 `D:\WorkData\SemIf\.venv\Scripts\python.exe`。系统 Anaconda 的 NumPy/SciPy 二进制依赖不兼容，无法导入 Transformers，因此不能混用其结果。两个脚本是测量/离线评测工具，不是生产服务器。

**短聊天条件（迁移前基线）：**HF 与 llama.cpp 两端均为 35 个输入 token，每端 1 次 warmup + 5 轮计时；llama.cpp 设 `cache_prompt=false`、`enable_thinking=false`。`Temp`/`TMP` 均指向 `D:\WorkData\SemIf\benchmarks\tmp`。显存比较受额外占用不超过 1 GiB 的用户限制：Q4_0 使用 `-ngl 6`，Q4_K_M 使用 `-ngl 5`。当前游戏生产配置也使用 `-ngl 5`，另将并行槽限制为 1、上下文设为 4096。

| 后端/量化 | GPU offload | 额外显存 | Decode 中位数 | Decode P95 | Prefill | TTFT 中位数 | 完整轮次中位数 |
| --- | ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| Transformers NF4（HF） | 当前 NF4 加载路径 | 基线 | 6.769 token/s | 7.319 token/s | — | 0.222 s | 3.256 s |
| GGUF Q4_0 | `-ngl 6` | 约 +957 MiB | 14.421 token/s | 17.273 token/s | 43.58 token/s | — | 2.092 s |
| GGUF Q4_K_M | `-ngl 5` | 约 +981 MiB | 14.233 token/s | 14.418 token/s | 42.47 token/s | — | 2.159 s |

在这个 35-token 短请求集上，GGUF Q4_0 与 Q4_K_M 的 decode 中位数分别约为 HF NF4 的 **2.13×** 和 **2.10×**。端到端完整轮次的输出长度不完全相同，因此总耗时不是严格等长比较；35-token 输入也不能说明长历史、长生成或真实游戏对话一定更快。GGUF 的 partial offload prefill 明显较低，不应只凭 decode token/s 声称所有请求体验提升。额外显存数据只适用于这轮设备与 offload 配置，不保证其他设备同样低于 1 GiB。

当前 HF `/decide` 短测使用 147 个输入 token：前向中位数 **0.2155 秒**、P95 **0.2422 秒**，5/5 次选择 `blast`。另外，隔离的全 GPU 决策质量集使用 SemIf `authored144.jsonl`，以 BF16 `direct-authored144.jsonl` 作参考：Q4_0 选择一致 **128/144（88.89%）**，Q4_K_M 为 **131/144（90.97%）**；两档候选概率均齐全 **144/144**。这是对 BF16 benchmark 参考集的结果，**不是当前游戏 NF4 实机质量对比，也不是动作正确率或安全保证**。5/5 blast 是小型固定 `/decide` 样本，不代表普遍决策质量。

**质量结果的适用边界：**144 条候选集仍是全 GPU llama.cpp Q4_K_M 对 SemIf BF16 参考结果，不是当前 `-ngl 5` 生产配置相对 HF NF4 的实机质量对比，也不是动作正确率。已完成的生产切换验证覆盖 tokenizer 启动、短聊天流、取消、候选评分、Unity 编译和一轮真实 F3 固定组合命中；长对话、长上下文和多轮稳定性仍需继续积累数据。

### 游戏 AI 子进程缓存目录隔离

`GodDialogueService.StartSidecar()` 现为游戏启动的 AI 子进程设置 `TEMP`、`TMP`、`TRITON_CACHE_DIR`、`TORCH_EXTENSIONS_DIR`、`CUDA_CACHE_PATH` 和 `HF_HOME`，均指向模型根目录下 `benchmarks/runtime-cache`。该设置仅作用于游戏 AI 子进程，不修改系统环境变量。Unity 6000.0.28f1c1 Windows 构建已通过，日志为 `Builds/unity-model-cache-check.log`；运行时生成的缓存路径尚未由游戏进程实际验证。C 盘未实际清理；删除安装包的操作被系统策略拒绝，因此不得记录为已清理。

性能对比至少报告中位数和 P95 首 token 延迟、稳定 token/s、峰值显存、聊天质量、`/decide` 候选选择一致率、取消时间与 Unity 动作校验结果。当前已完成固定短请求 GGUF A/B 和一组离线候选质量测量；后续仍需真实请求长度、多轮稳定性、取消和完整 sidecar 生命周期回归，不预设 GGUF 一定更快或量化质量可接受。

游戏自动查找：

1. 模型目录：`D:\WorkData\SemIf`，包含 tokenizer 配置、`benchmarks/qwen3.5-4b-q4_k_m.gguf`、`benchmarks/llama-b11160/llama-server.exe` 和 `src/semif_phase1/core.py`。
2. Python tokenizer 环境：`<模型目录>\.venv\Scripts\python.exe`，运行时依赖 `transformers` 与 `tokenizers`；不再将 Transformers/PyTorch 权重加载到 GPU。
3. llama.cpp 负责 Q4_K_M 模型推理；当前本地二进制为 `benchmarks/llama-b11160` 下的 Windows CUDA 发行包。

要改模型目录，可在运行前设置 PlayerPrefs 字符串键 `EmberHollow.GodModelPath.v1`。新路径仍需保留上面列出的 Q4_K_M GGUF、llama.cpp 运行时、tokenizer、`.venv` 和 `src` 布局。

## 运行架构

```text
GameWorld
  ├─ GodChatWindow          IMGUI 窗口、F2/Esc、输入与流式文本显示
  ├─ GodActionHarness       F3 候选评分、快照、执行前复核与结果观察
  └─ GodDialogueService     对话历史、异步 HTTP 与两种响应校验
       └─ Python 子进程      Assets/StreamingAssets/LocalAI/GodDialogueServer.py
            ├─ 加载 SemIf 本地 tokenizer，构造规范对话/候选提示
            ├─ 127.0.0.1:<动态端口> 上的聊天/摘要/取消接口
            ├─ 有限候选 token 概率读取和归一化
            └─ llama-server 子进程
                 └─ Qwen3.5-4B GGUF Q4_K_M，ngl=5，ctx=4096，parallel=1
```

`GodDialogueService` 在本地找一个可用 TCP 端口，启动隐藏的 Python 适配器；适配器再为 llama-server 选择 loopback 端口，并轮询模型就绪状态。所有服务只绑定 `127.0.0.1`，不监听局域网，也不调用云服务。分词和模型加载在后台完成；Unity 主线程通过并发队列应用逐段文本，不从后台线程访问 Unity 对象。

服务对象通过 `DontDestroyOnLoad` 跨关卡重开保留，模型只加载一次。正常退出时 Unity 请求适配器 `/shutdown`，由适配器终止 llama-server 后退出；若等待超时 Unity 会回收 Python 适配器。`-smokeTest` 可单独关闭 AI 加载，避免非模型功能闭环检查占用显存。

## 消息协议与上下文

- 游戏端向 `POST /chat` 发送 system、user、assistant 三种纯文本消息；服务返回逐行 JSON（NDJSON），每行是 `token`、`done` 或 `error` 事件。
- `POST /cancel` 关闭当前 llama.cpp 流请求，让推理槽停止本次生成；模型加载本身不支持中途取消。
- `POST /summarize` 将旧聊天分批压成短记忆；它与 `/chat` 共用模型锁，支持同一 `/cancel`，不会接入 `/decide` 或操作 Harness。
- `POST /shutdown` 用于正常关机，适配器先取消活动生成并终止 llama-server。
- `POST /decide` 接收 SemIf `validate_row` 使用的 `id/state/question/options`，用 SemIf `direct_messages` 与本地 tokenizer 构造规范提示，调用 llama.cpp `/completion` 读取单 token答案槽概率，确认所有 Unity 候选均有分数后归一化并返回；不输出自由命令文本。候选概率语义为“只在已登记候选间条件归一化”，不校准为置信度。
- 每次提交会附带一个由主线程生成的只读局面摘要：当前生命、位置、剩余敌人、能量、已编排法术和本局胜负状态。
- 聊天可见记录上限为 80 条；模型上下文超过 12 条时自动归纳较早对话，压缩后保留最近 8 条原始消息。历史按完整玩家/神谕问答归档，每批最多 4 条，摘要最多生成 96 token；后续摘要会合并旧摘要和下一批历史。摘要只作为可能不完整的聊天背景，当前主线程生成的关卡快照和最近原文优先。玩家单条输入限制为 500 字符，服务端 `/chat` 最多接收 24 条、每条 5000 字符、整个 HTTP 请求最多 128 KiB。
- 每局重开清空压缩记忆。聊天 UI 在摘要生成时显示中英双语整理状态；停止生成会尝试中止当前摘要或回复，并保留可继续对话的 Ready 状态。压缩失败时本轮回退到最近 12 条原始消息，不丢弃历史，后续提交可再次尝试。
- llama.cpp 上下文固定为 4096 token；聊天提示最多 3936 token，为最多 160 个输出 token 留出空间；摘要后的请求仍做服务端 token 上限校验，不静默截断。若仍过长则本轮清晰报错。`/decide` 提示上限为 4095 token，并只生成一个候选答案槽 token。
- 自然语言聊天没有 tool schema、命令通道、Unity 对象引用或地形写入接口；聊天文本不会被解析成游戏动作。
- 决策候选由 Unity 构造。F3 当前仅提供登记的 `faultline_charge` 与 `hold`；明确输入 `<作弊码>` 时，候选由 Unity 的已登记原语与文本匹配规则确定。Unity 检查 request ID、选项次序、分数有效/归一化，并确认 selected ID 与最高分对应。条件选项分数未校准为决策置信度，也不代表判断正确。完整原语表见[游戏操作原语目录](architecture/action-primitives.md)。

## 游戏操作闭环

F3 后，`GodActionHarness` 检查游戏正在运行、模型 ready、覆盖层关闭、最近敌人存活且距离不超过 20、能量够用、弹体和爆破队列为空。Unity 记录目标对象/位置/生命、玩家位置/生命、能量、施法 `RunToken`、队列和爆破计数，并暂停模拟；随后只发送 `faultline_charge` / `hold` 两个候选。

若模型选中固定组合，Unity 主线程重新验证同一目标引用及 ID、生命/位置、玩家生命/位置、能量、距离、`RunToken`、工作队列、覆盖层与服务状态。所有条件通过后，才调用 `ProgramCaster.TryCastFaultlineCharge`。它在 Unity 内编译固定 `Fork → Drill → Blast → Fuse → Expand → Impulse → Emit` 序列，费用取自同一编译器，不改玩家程序和存档。被选目标在弹体命中前暂停追逐，避免模型推理和飞行造成目标坐标过期。

Harness 等待本次全部弹体结束、爆破地形 job 清空，并确认锁定目标生命下降或被击败后报告成功。若模型选择 hold、状态过期或规则校验失败，则不发射，游戏暂停状态恢复。模型得分不会绕过前置条件。

`<作弊码>` 创世请求按 Unity 文本规则形成封闭候选 ID，每轮模型只选择一个 ID。Unity 再验权威快照，在主线程执行原语，并以原语专属的引擎事实生成 `action_id / accepted / evidence` 收据。若仍有未尝试候选、模型服务 Ready、队列清空且未达到 3 步上限，Harness 捕获新玩家/资源/敌人/刷怪门/施法令牌/爆破队列状态，将收据放入下一轮 `/decide` 输入。task 总期限 150 秒；失败、`hold`、模型 `finish_task`、X 取消、超时、游戏结束或存在活动法术/地形工作都会停止流程。全部候选都跑完且每条收据通过才显示任务通过；待办被中断时如实显示部分结果。玩家可见计划/复核阶段、执行状态、Unity 验收摘要与停止原因，不展示隐藏思维链。

Pi Agent Loop 的源码路径将 tool call 校验、执行前钩子、执行、执行后收尾、tool result 消息和下一轮模型判断分开；本项目只借用这种“真实执行结果回到下一轮”的控制流，不引入任意工具名或代码执行。流程细节和限制见[AI Harness 架构](architecture/ai-harness.md)与[Pi 源码研究](PI_AGENT_HARNESS_RESEARCH.md)。

聊天和决策使用同一模型锁串行运行。决策时不接受聊天发送；玩家聊天上下文不进入动作评分。自然语言回答与动作候选在服务协议和执行路径上分开。

Python 服务可由开发者单独启动以排查模型问题：

```powershell
& 'D:\WorkData\SemIf\.venv\Scripts\python.exe' `
  'D:\UnityGameAgentHarness\Assets\StreamingAssets\LocalAI\GodDialogueServer.py' `
  --model-root 'D:\WorkData\SemIf' --port 39119
```

服务报告 `state=loading` 时正在读权重；`ready` 后可以调用 `/chat`。Unity 的生产路径自行选择动态端口，无需手动打开服务。

## 代码定位

| 文件 | 职责 |
| --- | --- |
| `Assets/Scripts/GodChatWindow.cs` | IMGUI 聊天窗口、键盘输入、自动滚动和双语 UI |
| `Assets/Scripts/GodDialogueService.cs` | 本地子进程、聊天/决策 HTTP、主线程队列与双协议响应校验 |
| `Assets/StreamingAssets/LocalAI/GodDialogueServer.py` | 本地 tokenizer、loopback 适配、Q4_K_M llama-server 子进程生命周期、NDJSON 聊天与受限候选评分 |
| `Assets/Scripts/GodActionHarness.cs` | F3 候选、创世多步 task、权威快照、暂停、规则复核、登记原语执行、收据和结果核对 |
| `Assets/Scripts/ProgramCaster.cs` | 固定登记组合的共用编译和执行；不改写玩家存档程序 |
| `Assets/Scripts/GameWorld.cs` | 服务/Harness 生命周期、专用操作 smoke 与重开边界 |
| `Assets/Scripts/GameHud.cs` | F2/F3 入口和本地化操作状态 |

## 当前限制与后续接口

此版本包含本地文字聊天、F3 固定组合与以 `<作弊码>` 触发的首批有限创世操作。Qwen3.5-4B 是 SemIf 使用的模型，没有复用 Jev 的未公开模型或实现。聊天文本生成与 direct-options 候选评分用途不同；共享模型锁保证二者不并行运行。

扩充操作时，先在 Unity 定义固定候选 ID、规则实现、成本/范围上限和实际完成证据。不要接收模型自由生成的 `ModuleKind[]`、伤害、半径或世界坐标。Pi Agent Harness 的结构研究与本项目边界见[神谕操作架构](architecture/ai-harness.md)。服务失败时，本地游戏和法术系统仍可运行。

## 验证记录

| 日期 | 检查 | 结果 |
| --- | --- | --- |
| 2026-09-25 | Harness 风格聊天上下文自动压缩 | Unity Windows 构建通过，日志 `Builds/unity-context-compaction-build.log`；真实模型玩家进程以 `-smokeTest -contextCompactionSmokeTest` 注入 7 轮旧对话，`/summarize` 两批均成功，归档 8 条并保留最近 8 条，之后 `/chat` 成功（387 输入 token、31 输出 token、12.50 token/s），同时完整 `EMBER_HOLLOW_SMOKE_PASS`。玩家日志 `Builds/player-context-compaction-smoke.log`。压缩记忆只进入聊天 system 背景；4096 context、ngl=5 与 1 GiB 显存约束均未提高。 |
| 2026-09-25 | Q4_K_M 生产后端迁移 | `GodDialogueServer.py` 由 Transformers NF4 权重加载改为本地 tokenizer + llama.cpp Q4_K_M；配置为 `ngl=5`、`ctx=4096`、`parallel=1`、Flash Attention auto。standalone `/chat` 流、`/decide` 候选概率、`/cancel`、`/shutdown` 通过；显存总占用由 2.0 GiB 基线升至约 2.8 GiB（增量约 0.8 GiB）。Unity Windows 构建通过，真实游戏 `-smokeTest -modelSmokeTest -actionHarnessSmokeTest` 选中 `faultline_charge` 并命中锁定目标，Python 与 llama-server 正常退出。最终构建日志 `Builds/unity-gguf-q4km-final-build.log`，玩家日志 `Builds/player-gguf-q4km-final-smoke.log`。 |
| 2026-09-25 | Q4_K_M 游戏配置短聊天测量 | standalone 19 输入 token、3 输出 token：TTFT 0.487 秒、解码约 13.48 token/s、总计 0.636 秒。单次短聊天不能代表长提示与多轮平均性能。 |
| 2026-09-25 | Q4_K_M 推理设置 A/B | 线程 4/8/12（1 warmup + 3 轮，78-token 提示、96-token 贪心输出）decode 中位数 14.31/14.61/13.17 token/s，输出哈希一致；保持 8 线程。探索性多轮 `cache_prompt` 对照 prompt 时间中位数 629/477 ms，但第二轮后输出/后续提示已分叉；另一次贪心复测同一轮相同输入的输出哈希不同，暂不启用。`ngram-mod` 单组 seeded 六轮 decode 约比缓存组快 3.4%，没有干净质量对照，不启用。VRAM 硬门槛为相对基线 1024 MiB；各次峰值增量约 935–1002 MiB。 |
| 2026-09-25 | 固定请求 HF NF4 与 GGUF Q4 聊天 A/B（生产迁移前基线） | 35 输入 token、1 warmup+5 轮；HF decode 中位数 6.769/P95 7.319 token/s、TTFT 中位数 0.222 秒、轮次中位数 3.256 秒；Q4_0 `-ngl 6`（+957 MiB）decode 14.421/P95 17.273、prefill 43.58、轮次 2.092 秒；Q4_K_M `-ngl 5`（+981 MiB）decode 14.233/P95 14.418、prefill 42.47、轮次 2.159 秒。仅短请求，端到端长度不同且 partial-offload prefill 较低。 |
| 2026-09-25 | GGUF direct-choice 质量与 HF `/decide` 短测（迁移前质量基线） | 全 GPU `authored144.jsonl` 对 BF16 `direct-authored144.jsonl`：Q4_0 128/144（88.89%），Q4_K_M 131/144（90.97%）；候选概率齐全均为 144/144。HF 147-token 请求前向中位数 0.2155 秒/P95 0.2422 秒，5/5 选择 blast。Q4 选择一致率参考为 BF16 benchmark，不是 `ngl=5` 游戏配置与 HF NF4 的实机质量对比，也不是正确率；5/5 blast 不代表普遍决策质量。 |
| 2026-09-25 | 离线 benchmark/evaluator 工具 | 新增 `tools/model_backend_benchmark.py` 与 `tools/evaluate_gguf_decisions.py`。必须用 `D:\WorkData\SemIf\.venv\Scripts\python.exe`；系统 Anaconda 因 NumPy/SciPy 二进制不兼容未能导入 Transformers。Luna 按冻结接口交付；Sol 发现 llama 的 `prompt_n` 位于 `timings.prompt_n` 而非顶层，Luna 修复后评测通过。 |
| 2026-09-25 | Unity AI 子进程临时与模型缓存路径隔离 | `GodDialogueService.StartSidecar()` 为游戏 AI 子进程设置 `TEMP`、`TMP`、`TRITON_CACHE_DIR`、`TORCH_EXTENSIONS_DIR`、`CUDA_CACHE_PATH`、`HF_HOME` 到 `modelRoot/benchmarks/runtime-cache`；只影响子进程。Unity 6000.0.28f1c1 Windows 构建成功，日志 `Builds/unity-model-cache-check.log`；运行时缓存路径尚未由游戏进程实际验证。C 盘没有清理，删除安装包被系统策略拒绝。 |
| 2026-09-24 | 本机模型目录、权重、CUDA Python 环境与 8 GB GPU 枚举 | 找到 SemIf Qwen3.5-4B safetensors 与 NF4 加载环境；游戏子进程成功加载模型并生成完整中文回复 |
| 2026-09-24 | Unity Windows 构建与 `-smokeTest -modelSmokeTest -capture` | 构建成功；本机模型启动、中文流式对话、暂停 UI、语言切换与既有玩法检查全部通过；记录到 `LOCAL_ORACLE_SMOKE_PASS` 与 `EMBER_HOLLOW_SMOKE_PASS` |
| 2026-09-24 | Qwen3.5 chat template 消息顺序修复 | 实测发现游戏端曾发两条 system 消息，模板要求 system 位于首位且仅一个；合并为一条并在服务端校验，端到端复验通过 |
| 2026-09-24 | 推理加速现状 | Transformers 日志提示 `causal_conv1d`、`flash-linear-attention` 优化算子未安装，相关运算使用 PyTorch 参考实现；需要基于固定硬件做性能基准，未修改 SemIf 环境 |
| 2026-09-24 | SemIf 有限候选评分与 Unity 操作 Harness | `/decide` 只对 Unity 给出的候选调用 `direct.score`；F3 操作通过真实模型评分、执行前快照复核、固定组合发射与锁定目标伤害检查。`-smokeTest -modelSmokeTest -actionHarnessSmokeTest` 输出 `GOD_ACTION_HARNESS_SMOKE_PASS`，模型选择 `faultline_charge`，爆破数在 1–3 且实际命中锁定目标，退出码 0 |
| 2026-09-24 | 本地聊天与玩法集成回归 | `-smokeTest -modelSmokeTest` 输出 `LOCAL_ORACLE_SMOKE_PASS` 和 `EMBER_HOLLOW_SMOKE_PASS`；SemIf Qwen3.5-4B NF4 完整回复中文，随后地形、钻脉、引信、冲击、回响取消、大范围破坏、敌人击败与胜利状态全部通过，退出码 0 |
| 2026-09-25 | 跳跃/摩擦与暗语创世原语 | Windows 构建、`-movementSmokeTest`、真实模型 `-creationActionSmokeTest` 和 `-mapDemolitionSmokeTest` 通过。确认 0.12 秒跳跃缓冲、0.1 秒离地宽限、玩家/地形零摩擦材质；刷怪门 3 秒一只、存活上限 8；全图测试将可破坏格 10,250 清到 0，基岩维持 5,740。 |
| 2026-09-25 | token 生成测量与初步结论 | sidecar 将 chat 首 token/解码速率和 `/decide` 前向用时回传 Unity；一次真实模型聊天测得输入 212、输出 35、TTFT 1.267 秒、6.84 token/s、总生成 6.237 秒；一次决策 258 输入 token、前向 1.0595 秒。测量仅为单次样本；运行日志确认 FLA/causal-conv1d 缺失并回退 PyTorch，Triton 未找到 CUDA Toolkit。先做隔离快算子兼容性测试，再 A/B GGUF；未动外部 SemIf 环境。 |
| 2026-09-24 | 玩法 smoke 稳定性修复 | 大范围测试此前复用前序开挖坐标，模型集成运行时出现“cast 已接受但 0.35 秒内未观察到爆破”。将测试发射点移到未被前序用例修改的地面，并为 smoke 断言增加逐项失败日志；最新 Windows 构建、普通玩法 smoke、聊天/玩法 smoke 和真实模型组合 smoke 全部通过 |

