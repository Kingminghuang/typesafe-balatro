# typesafe-balatro 可行性评估

- **目标**：以 `/Users/huangqingming/Workspace/balatrollm` 的基础设施为骨架，引入 Jev（TypeSafe System One）作为决策头，在 `/Users/huangqingming/Workspace/typesafe-balatro` 实现一个 Jev 玩 Balatro 的智能体与评测台。
- **方法论参照**：`/Users/huangqingming/Workspace/typesafe-mario/docs/jev-decision-flow.md`
- **评估方式**：阅读三边源码（balatrollm 全部 Python 模块 + 策略模板、typesafe-mario 的 `state/policy/actions/runner/cli`）+ 解包 `typesafe-sdk 0.7.0` wheel 逐字段核对 API + 官方文档核对配额/计价/上下文限制 + 检查本机环境。
- **结论日期基准**：评估时 `typesafe-balatro` 为空仓库（`git init` 后无提交）。

---

## 0. 结论（TL;DR）

**技术上完全可行，而且比 typesafe-mario 更容易落地**——但"容易"的部分不是重点，真正的工程量集中在 balatrollm 完全没有的那一层。

三个核心判断：

1. **回合制消掉了 Mario 项目最难的部分。** Mario 需要 `ThreadPoolExecutor` 异步流水线、延迟回灌（`last_inference_delay_frames`）、起跳死线反推，因为 60fps 不给模型思考时间。Balatro 是回合制，一次决策可以慢慢等——同步循环即可，typesafe-mario 最精巧的 1/3 代码直接不需要。
2. **"用 Jev 换掉 LLM"这件事本身很轻，真正的项目是"事实层"。** balatrollm 把原始 gamestate JSON 丢给 LLM，让模型自己去算牌型、算够不够分、算该不该攒钱。Jev 的官方定位是"一次一个 snap judgment"，明确不建议"分析并决定"。所以必须由代码先把事实算好（剩下的分差、成牌类型、估算分数、抽牌概率、经济账），Jev 只在语义层面做取舍。**这层事实抽取 + 候选枚举在 balatrollm 里没有任何对应物，要全新写，占项目工作量的一半以上。**
3. **Jev 的硬性约束恰好帮了大忙。** "答案必然落在给定 criteria 之内"意味着**非法动作在结构上不可能发生**——balatrollm 里那一整套"连续 3 次非法调用就中止"的自我修正循环、`error call` / `failed call` 双通道错误处理，可以大幅删除。这是净简化，不是净增负担。

**但有一条第 0 号判断必须放在最前面：如果目标是"跑基准实验"，那么最大的风险不在 Jev，而在 BalatroBot。** 上游有 4 个 2026-08-08 提交、至今无人回复的 open issue，描述的是**静默的数据损坏**——跨局 `won` 泄漏（把失败记成胜利）、`--fast` 破坏种子可复现性、长会话退化。而 balatrollm 的"端口池 + 长生命周期实例复用"执行模型**恰好就是踩这些坑的写法**（issue 正文点名了它）。所以**"借用 balatrollm 的大部分基础设施"这句话里，`/Users/huangqingming/Workspace/balatrollm/src/balatrollm/executor.py` 的执行模型是唯一必须推翻重做的部分**——骨架留，策略换。详见 §1.1。

可行性评级：

| 维度 | 评级 | 说明 |
| --- | --- | --- |
| 架构契合度 | ✅ 高 | 状态机循环 / 批量并行 / 落盘统计几乎原样可用 |
| Jev 能力匹配度 | ✅ 高 | 255 选项上限、64k 上下文、纯 JSON 输入、必落 criteria 内 |
| 成本与限流 | ✅ 无压力 | 单局约 $0.03；1200 req/min；还有 $5 免费额度 |
| 事实层工程量 | ⚠️ 中高 | 需要自己写 Balatro 的"算术层"（含计分估算），是主要成本 |
| **数据有效性** | 🔴 **需要主动防御** | 上游静默缺陷 + 长生命周期实例复用 = 基准数据可能整体不可信（R11） |
| 环境前置条件 | ⚠️ 当前为 0（macOS 已实测；Windows 待实测） | 评估机（macOS）没有 Steam / Balatro / balatrobot / API key，且 macOS 26 + M1 有未知数（R12）；Windows 侧检查清单已备好（[`environments/windows.md`](environments/windows.md)） |

**工作量估计**（已计入 R11 带来的实例生命周期改造与数据复现性验收）：

- **MVP（L1 估算，忽略 Joker 效果，标注 `low` 置信度）** ≈ **5–8 个工作日**——这条路径不需要效果表编译器，可以先跑起来。
- **能打（L2：效果表编译器 + Joker 感知的估算）** ≈ 再加 **5.5–6 天**（其中约 3.5 天不依赖游戏与 API key，可与 M1 并行），详见 [`EFFECT_TABLE_DESIGN.md`](/Users/huangqingming/Workspace/balatrollm/EFFECT_TABLE_DESIGN.md)。
- **完成度（结果可复现、能做对照实验）** ≈ **3–4 周**（单人）。

**这里有一个值得注意的顺序问题**：L1 的估算器"能跑但打不好"，而 Jev 拿到 `confidence: low` 时会退回到按原则走。所以 MVP 阶段**不要**因为分数难看就急着调 instructions——先确认整条链路通，再把效果表补上，让 `confidence` 升到 `medium/high`，性能提升才有可归因的来源。

---

## 1. 本机环境现状（按 OS 拆分检查）

本项目可能在 macOS 与 Windows 上开发，机器环境检查按 OS 拆成独立文件（换机器只改对应文件，不污染评估主文档的结论）：

| 平台 | 检查文件 | 状态 |
| --- | --- | --- |
| macOS（评估机：Apple M1 + macOS 26.6.2） | [`environments/macos.md`](environments/macos.md) | ✅ 已实测：Steam / Balatro / Mods 目录 / `balatrobot` / API key 全缺，仅 Python 可用 |
| Windows | [`environments/windows.md`](environments/windows.md) | ⬜ 待实测：清单与预期路径已对照上游源码备好，拿到机器后按首次验收清单跑一遍回填 |

（上游还提供 `linux.py` / `native.py` launcher，需要时照这个模式补 Linux 的检查文件即可。）

**这意味着**：写代码可以立刻开始（大部分模块可以离线单测，见 §7），但"跑通第一局"之前必须先完成 Balatro + BalatroBot 的安装——在哪台机器上都一样。这是唯一的硬门槛，建议**第一天就并行推进**，不要等到代码写完才发现环境不通。

macOS 与 Windows 的启动路径都已对照上游 `balatrobot/platforms/*.py` 核实（细节与复检命令分别见上面两个文件，结论一致）：macOS 用 `DYLD_INSERT_LIBRARIES` 注入 `liblovely.dylib` 后直接 exec LÖVE；Windows 用 Lovely 的 `version.dll` DLL 注入后直接 exec `Balatro.exe`；`--fast` / `--headless` 等开关两边都走 `BALATROBOT_*` 环境变量。

**结论**：balatrollm 的 `/Users/huangqingming/Workspace/balatrollm/src/balatrollm/executor.py` 里 `BalatroInstance(cfg, port=port)` 的**启动方式**在 macOS / Windows 上都可以保留（不必改成直接 spawn CLI）。

**但是，它的"实例复用"策略必须改**——这一点在评估后期查上游 issue 时发现了硬伤，详见下面 §1.1，它直接影响能不能做基准实验。

### 1.1 ⚠️ 三个上游未修复缺陷，直接命中 balatrollm 的执行模型

BalatroBot 上游有 4 个 **2026-08-08 提交、至今无回复** 的 open issue（#231–#235），其中两个是针对 balatrollm 这种"长生命周期实例 + 端口池复用"模式说的（issue 正文已逐条核对原文）：

| Issue | 问题 | 对本项目的后果 |
| --- | --- | --- |
| [#232](https://github.com/coder/balatrobot/issues/232) | **跨局状态泄漏**：`menu` + `start` 不能完全重置 run 状态。`G.GAME.won` 会跨局残留——**实例上赢过一次之后，后续任何一局失败都会报 `won: true`**（16 个实例复现，给了种子与对照）。另有 `last_tarot_planet`、`pool_flags`、`used_jokers` 三处泄漏 | ★★ issue 正文**点名 balatrollm**："the executor's port pool starts instances once... `bot.py` ends a run as won when `gamestate["won"]` is truthy — so a loss ... can be recorded and published as a **win**"。后果有两层：① **胜率统计直接失真**；② 泄漏 2–4 让**固定种子不可复现**（商店/消耗品结果取决于该实例上**之前跑过的种子**）——而固定种子对照正是 §6.2 实验设计的地基 |
| [#234](https://github.com/coder/balatrobot/issues/234) | **`--fast` 不是保语义的加速**：10× 速度下 SMODS 发牌事件竞态，**任意一次 Tarot/Spectral 卡包关闭之后，牌堆顺序被静默打乱**，之后每次发牌都和种子应该给出的不一样。原文："For benchmark data this is a silent validity problem" | ★ 之前我写的"加速只需加环境变量"需要加限定：**`--fast` 只能用于开发/演示，不能用于计分实验**。要么接受慢速，要么接受种子失效 |
| [#235](https://github.com/coder/balatrobot/issues/235) | **长会话退化**：同一实例连跑约 70 局后，单次动作延迟涨到约 3 倍，随后 `discard` 因 `G.buttons` 为 nil 而崩 | 批量脚本必须**定期回收实例**，不能一次开 8 个实例跑几百局 |
| [#231](https://github.com/coder/balatrobot/issues/231) | `cash_out` 在回合结算行写入前就触发 → **奖励少收，有时为 $0** | 影响 `round.chips` 的读取时机，直接波及 §4.1 的"估算 vs 实际"校准闭环 |

另外两条工程约束（来自上游源码与 issue）：

- **多实例共享同一份 Balatro 存档目录**（macOS：`~/Library/Application Support/Balatro`；Windows：`%APPDATA%\Balatro`），`balatrobot` 的启动器**没有任何存档隔离/文件锁**。叠加 #232 的泄漏，**并行度越高，数据可信度越低**。
- **必须 pin `balatrobot>=1.5.2`**：1.5.2 才修掉并发连接导致游戏崩溃 / 响应静默 0 字节的问题（#193、#196）。注意 **balatrollm 自己的 `/Users/huangqingming/Workspace/balatrollm/pyproject.toml` 还 pin 在 `>=1.4.1`**，直接把它的依赖声明抄过来会踩到这个坑。

**因此执行模型要改成**：把"实例复用"改成**一局一进程**（或至少每 N 局回收 + 每局前显式校验状态），并且**不要相信 `won` 字段**——用 `state == "GAME_OVER"` 加上最终 ante/round 做交叉验证；计分实验关掉 `--fast`。这会把吞吐量显著压低（可能只有原计划的 1/3–1/2），是本评估里**对工作量与实验规模影响最大的一条修正**。

安装链条为：Steam → Balatro → Lovely → Steamodded → balatrobot mod → `uv tool install balatrobot`。`/Users/huangqingming/Workspace/evalatro`（同工作区）已经把这套流程写成了脚本，可直接照抄步骤省去踩坑（Windows 机器的安装细节与首次验收见 [`environments/windows.md`](environments/windows.md)）。

---

## 2. 两边能借什么：模块级盘点

### 2.1 balatrollm 的基础设施（3310 行，逐个判定）

| 文件 | 行数 | 处置 | 具体说明 |
| --- | --- | --- | --- |
| `/Users/huangqingming/Workspace/balatrollm/src/balatrollm/client.py` | 71 | ✅ **原样复用** | JSON-RPC 2.0 异步客户端，`call(method, params)`，与决策层零耦合 |
| `/Users/huangqingming/Workspace/balatrollm/src/balatrollm/cli.py` | 133 | ✅ **基本复用** | 换环境变量前缀与参数名即可 |
| `/Users/huangqingming/Workspace/balatrollm/src/balatrollm/config.py` | 279 | ⚠️ 小改 | 三层配置（env < yaml < cli）+ 笛卡尔积任务生成逻辑完整可搬；`model` 列表改成 `policy` 列表，接 `TYPESAFE_*` |
| `/Users/huangqingming/Workspace/balatrollm/src/balatrollm/executor.py` | 114 | ⚠️ **骨架可用，执行模型必须重做** | 端口池 + `asyncio.Queue` + 任务并发的骨架可以照搬，启动方式在 macOS 上也能保留；**但"一个实例跑完整批任务"的复用策略正是上游 #232/#235 描述的出错模式，必须改成"一局一进程"或定期回收**，见 §1.1 |
| `/Users/huangqingming/Workspace/balatrollm/src/balatrollm/collector.py` | 453 | ⚠️ 中改 | 目录布局 `/Users/huangqingming/Workspace/typesafe-balatro/runs/v{version}/{strategy}/{vendor}/{model}/{ts}_{deck}_{stake}_{seed}/`、`stats.json`、`batch.json`、`latest.json`、`previous.json`、`screenshots/` 全部保留；`requests.jsonl`/`responses.jsonl`（OpenAI Batch API 格式）换成 Jev 专用的 `decisions.jsonl` |
| `/Users/huangqingming/Workspace/balatrollm/src/balatrollm/bot.py` | 408 | ⚠️ 大改 | `_run_game_loop` 的状态分发骨架（`SELECTING_HAND`/`SHOP`/`SMODS_BOOSTER_OPENED` 问模型，`ROUND_EVAL` 硬编码 `cash_out`，`BLIND_SELECT` 硬编码 `select`，`GAME_OVER` 结束）完整保留；`_get_llm_response` / `_execute_tool_call` 换成 `policy.choose` + `actions.materialize`；错误处理两条通路可删掉大半。**另外循环开头的 `if gamestate.get("won")` 判定必须改**——它正是上游 #232 点名的失真路径（见 §1.1） |
| `/Users/huangqingming/Workspace/balatrollm/src/balatrollm/llm.py` | 144 | ❌ **替换** | 退化成 `/Users/huangqingming/Workspace/typesafe-balatro/jev.py`：`TypeSafeClient` 封装 + 超时/重试/连续失败中止语义（这部分模式值得照抄） |
| `/Users/huangqingming/Workspace/balatrollm/src/balatrollm/strategy.py` | 159 | ⚠️ 重构 | Jinja 渲染器 → playbook 装载器（instructions / criteria 生成规则 / manifest） |
| `/Users/huangqingming/Workspace/balatrollm/src/balatrollm/strategies/*/STRATEGY.md.jinja` | 730 | ⚠️ 转化 | 720 行"Balatro 教科书"的**内容**很有价值，但形态要从散文变成结构化 `instructions`；也可保留为人类可读文档 |
| `/Users/huangqingming/Workspace/balatrollm/src/balatrollm/strategies/*/GAMESTATE.md.jinja` | 335 | ❌ **废弃** | Jev 直接吃 JSON，不需要把状态渲染成 Markdown 散文（typesafe-mario README 明确："TypeSafe accepts JSON directly, so there is no need to flatten telemetry into prose"） |
| `/Users/huangqingming/Workspace/balatrollm/src/balatrollm/strategies/*/TOOLS.json` | 356 | ⚠️ 重构 | 按状态分桶的**思路**保留，内容从"工具 schema"变成"候选计划枚举规则" |
| `/Users/huangqingming/Workspace/balatrollm/src/balatrollm/strategies/*/MEMORY.md.jinja` | 34 | ⚠️ 转化 | "最近 10 条动作 + 上次错误" → 变成 state 里由代码计算的 `recent_control` / `run_history` 事实块 |
| `/Users/huangqingming/Workspace/balatrollm/src/balatrollm/views.py` + `/Users/huangqingming/Workspace/balatrollm/views/*.html` | 45+ | ⚠️ 可选 | 低优先级，最后做，展示 `decisions.jsonl` |
| `/Users/huangqingming/Workspace/balatrollm/src/balatrollm/strategies/*/manifest.json` | 11 | ✅ 复用 | playbook 元数据格式照搬 |

**粗算**：约 40% 可直接搬或基本照搬，约 35% 需要中等改造，约 25% 必须新写——而必须新写的那 25%（事实层 + 候选层）恰恰是决定这个 agent 强弱的全部。

**唯一的例外是 `/Users/huangqingming/Workspace/balatrollm/src/balatrollm/executor.py`**：它的**并发骨架**可以搬，但**实例生命周期策略必须推翻重做**（一局一进程 / 定期回收），因为 balatrollm 的"一个实例跑完整批任务"正是上游已知缺陷的触发条件（§1.1、R11）。这是"借基础设施"这件事里唯一一个**不能照抄的部分**。

### 2.2 Jev 侧的真实能力边界（已核对 SDK 源码与官方文档）

这部分是评估里最关键的"弹药"，逐条都经过核实（解包 `typesafe_sdk-0.7.0-py3-none-any.whl` 读源码 + 官方文档）：

| 项 | 实际值 | 对本项目的含义 |
| --- | --- | --- |
| 模型 | `jev-1.13.0`（别名 `jev-latest` / `jev-preview`） | 返回体 `model` 字段会回报版本号，**可固定版本号保证实验可复现** |
| 计价 | **$42 / Btok 输入**（≈ $0.042 / Mtok），**输出 token 免费**；注册用户有 **$5 免费额度 ≈ 1.2 亿 token** | 单次决策约 2–4k 输入 token ≈ **$0.0001**；整局约 300 次决策 ≈ **$0.03**；100 局横评 ≈ **$3**。**免费额度足够跑完整个项目** |
| 延迟 | 约 **70–500ms，典型 ~100ms**（官方/第三方数据，待实测） | 单局墙钟时间由游戏动画主导，不由推理主导 |
| 限流 | 250,000 tokens/s；**1,200 requests/min**（官方声明会动态调整）；超限返回 429，过载返回 529 | 4–8 个并行实例绰绰有余 |
| 上下文 | 单请求 64k token；`state` + 最长单个问题 32k；**一次调用只能带一个 `state`**（多状态 = 多次调用） | 状态 JSON 放 2–5k token，余量 10 倍以上；每次决策正好对应一次调用 |
| **Choice 选项上限** | **255 个**（官方明说"给出完整列表而不是精简清单"；官方 cookbook 里真实出现过 **182 选项**和 **218 选项**的 Choice） | ★ 候选计划集 5–25 个非常舒服；即使激进到把几百个原始组合全列出来也在可行范围内（但不建议，见 §3） |
| **Score 档位上限** | **2–10 档**（少于 2 档或多于 10 档会被拒） | 危险度/经济压力这类仪表盘 Score 定 3–5 档即可 |
| 提问数上限 | 无文档硬上限，受 64k 预算约束（建议把同 state 的所有问题塞进一次请求） | 一次请求同时要 `plan` + `noul` + `score`，符合官方"speculative fan-out"模式（官方实测：13 个问题合并 1 次调用比 13 次调用**便宜 12.2 倍、快 10.0 倍**） |
| 输入形态 | 纯文本：字符串 / JSON object / 数组；**无图** | 与"文本化 gamestate、不看截图"的路线天然吻合 |
| 语言 | **英文是主要训练语言**；CJK 被接受但精度更低 | ★ 事实 JSON 的 key 与文本 value 必须全英文 |
| 返回结构 | `Choice` → `choice` + `probabilities` + `confidence`；`Noul` → `noul`(0–1)；`Score` → `score`(期望值，可为小数) + `probabilities` + `confidence` | 无 reasoning 文本 → 可解释性只能靠概率分布 + 置信度 + 额外探针问题 |
| **强制约束** | **"Every answer is constrained to the options you supplied"** | ★★ **非法动作在结构上不可能发生**——这是相对 balatrollm 最大的可靠性升级 |
| `instructions` / `criteria` 形态 | 字符串、**对象、数组均可**（对象可用任意自定义字段名，如 `what`/`not_for`/`examples`） | ★ 候选描述可以携带机器可读事实，而不是一段散文 |
| SDK | **同步 `TypeSafeClient` + 异步 `AsyncTypeSafeClient` 都有**；依赖 `httpx2` + `pydantic` + `tenacity`（注意：balatrollm 用的是 `httpx`，两者会并存，不冲突） | 可原生接入 balatrollm 的 asyncio executor：**必须用 `AsyncTypeSafeClient`**，同步客户端会阻塞事件循环 |
| 重试 | `RetryPolicy`（默认 `max_retries=2`、0.5→5.0s 退避 + 0.25 抖动、重试 408/429/5xx、遵守 `retry-after`、**单次调用总预算 30s**、单次 HTTP 操作超时 10s） | 网络层可靠性不用自己写；"最多 ~30s 后必然抛出"这一上界正好对应 balatrollm 的 `llm_abort` 降级语义 |
| **离线开发能力** | ① SDK 支持注入 `httpx` client / `transport` → **可用 mock transport 做零成本单测**；② TypeSafe **官方**提供了 [`system-one-adapter-python`](https://github.com/typesafe-ai/system-one-adapter-python)（同样接口，后端换成 OpenAI/Anthropic 模型）；③ 社区有 MIT 协议的自托管替身 [`jeff`](https://github.com/logan-markewich/jeff)（GLiFormer-400M，实现同一个 `POST /v1/systemone`，`TypeSafeClient(api_key="devkey", base_url="http://localhost:8000")` 即可接入官方 SDK）；④ 无官方 mock 模式 | ★ 这是**风险 R1 的关键缓解**：整条 Jev 通路可以在没有 API key、没有 Balatro 的情况下开发与 CI 测试。**注意**：`jeff` 精度明显低于 Jev（其自测 JevBench 66.9 vs Jev 75.3），只能用来验契约，**不能用来做效果结论** |
| **官方公布的弱项** | **不做算术、不做计数**、字面阅读、日期排序不可靠、多跳推理弱、state 越冗长精度越掉、容易被矛盾指令带偏；**不生成任何文本** | ★★ 这是"事实层必须由代码承担"的最强论据——分差、成牌、抽牌概率、利息，全部不能指望 Jev 去算 |
| 概率语义陷阱 | ① `Noul` **没有 confidence**；② `P(是) + P(否) ≠ 1`（官方示例出现 0.72 + 0.47）；③ **Choice 的概率是相对的，不能和 Noul 的概率互相比较**（同一命题，Noul 给 0.22、Choice 给 0.01）；④ Choice 的并列（tie）处理未文档化 | 设计上必须：不要在不同问题类型之间复用阈值；不要假设 Noul 与"取反 Noul"互补；低 confidence 时按"不确定"处理而不是假设有稳定的 tie-break |
| 确定性 | **没有 `temperature` / `seed` / `top_p` 参数**；部分 Noul 在多次重复间有 ~0.005–0.008 的标准差，多数问题完全可复现 | 实验可复现性靠"固定模型版本 + 固定种子集 + 多次重复取分布"，而不是靠单次确定性 |
| 可观测性 | `response.request_id`（来自 `x-typesafe-request-id`）、`response.model`、`usage.input_tokens/output_tokens`；**延迟不在响应体内**，需自行计时；`TYPESAFE_LOG_LEVEL=debug` 会记录**未脱敏**的请求/响应体 | 落盘时记录 request_id + model + 自计时延迟；不要把 debug 日志带 key 输出到公开 CI |
| 异常 | `TypeSafeRateLimitError`(429) / `TypeSafeUnprocessableEntityError`(422) / `TypeSafeAuthenticationError`(401) / `TypeSafeAPITimeoutError` 等，层级完整 | 可以照 balatrollm 的 `llm_abort` 模式做降级分类 |

**两个必须记住的"形状约束"**（来自官方 primitives 文档）：

- 一次一个问题，问"**一秒内能做出的判断**"。"分析这个局面并决定最佳打法"是反面教材——那需要慢思考，官方建议拆成小问题、在代码里合成。**这就是为什么事实层不是可选项：它是让 Jev 的问题保持"snap judgment"的前提。**
- 同一请求内的问题**互相独立**，前一个答案不会成为后一个的上下文。跨问题依赖必须发第二次请求（官方认可的两段式用法正是"用第一个 Choice 的答案决定第二个请求提供哪些选项"——恰好就是"先选策略类别、再选具体计划"的层级设计）。

### 2.3 typesafe-mario 提供的方法论模板

`/Users/huangqingming/Workspace/typesafe-mario/docs/jev-decision-flow.md` 里那套三层分工（**代码算事实 → 提示词教规则 → Jev 做取舍**）可以整体平移，但要按 Balatro 的形态重写：

| Mario 概念 | Balatro 对应物 | 备注 |
| --- | --- | --- |
| `MarioSnapshot.to_state()` | `GameSnapshot.to_state()` | 结构照搬：objective / 局况 / 手牌 / 威胁 / 近期反馈 |
| `navigation_features()` / `threat_features()` | 分差、成牌评估、抽牌概率、经济账 | 语义完全不同，需重写 |
| `jump_must_start_this_decision`（时序结论） | `blind_clearable_with_remaining_hands`、`must_play_now` | "算好结论给模型"的精髓要保留 |
| `terrain.observation_reliability` | **`estimate_confidence`**（计分估算的可信度） | ★ 最漂亮的平移：模型知道自己的数字什么时候不可信 |
| `reaction_timing.last_inference_delay_frames` | ❌ 无对应物 | 回合制无延迟压力，整块删掉 |
| `recent_control`（上一动作结果） | 上一决策的**估算 vs 实际**误差 | ★ 保留并升格为"校准闭环"：估算偏差回灌下一次状态 |
| `Choice` 7 个宏动作 | 候选计划枚举（见 §3） | 从静态枚举变成按局面生成，但**语义分层的思路一致** |
| `Noul` / `Score` 交叉印证 | 同样保留（见 §3.3） | |
| `/Users/huangqingming/Workspace/typesafe-mario/src/typesafe_mario/dashboard.py` 实时仪表盘 | 可选，低优先级 | balatrollm 的 `/Users/huangqingming/Workspace/balatrollm/src/balatrollm/views.py` 覆盖了大部分需求 |
| "无脚本兜底覆盖"原则 | 同样坚持：候选与事实由代码给，选择权归 Jev | |

---

## 3. 核心难题：把 Balatro 的组合动作空间压成 Jev 的 Choice

这是整个评估里唯一需要真正动脑子的地方，也是 Mario 经验**不能**直接照抄的地方。

### 3.1 问题有多大

Mario 的动作空间是 **7 个静态宏动作**，所以 criteria 可以硬编码。

Balatro 不是。balatrollm 的 `/Users/huangqingming/Workspace/balatrollm/src/balatrollm/strategies/*/TOOLS.json` 里 `SELECTING_HAND` 状态下的动作是 `play(cards=[...])`——从 8 张手牌里选 1–5 张：

```
C(8,1)+C(8,2)+C(8,3)+C(8,4)+C(8,5) = 8+28+56+70+56 = 218 种出牌
加上同样多的弃牌组合                                = 218 种弃牌
加上 use / sell / rearrange
≈ 450+ 个原始动作
```

balatrollm 把这个难题甩给 LLM（让它自己推理该选哪几张）。Jev 不能这么用——它不做慢思考。

**需要澄清一个可能的误解**：Jev 的 Choice 上限是 255 个选项，官方 cookbook 里确实出现过 218 个选项的 Choice。所以"把 218 种出牌全列出来让 Jev 选"在**技术上**是可行的。不建议这么做，理由是：

1. 218 张选项里绝大多数是高度相似的卡牌子集（`[0,2,3]` vs `[0,2,4]`），而 Jev 的强项是在**语义不同**的选项之间做情境化权衡，不是在近重复项里做精细排序；
2. 每个候选都携带"估算分数、是否过关、抽牌概率"这类事实，218 条结构化描述会把 state 撑大（官方另有一节专门讲 state 变长后的精度漂移）；
3. 让模型直接选卡牌子集，等于把"组合优化"塞回给模型——这正是应该由代码（候选枚举）承担的部分。

### 3.2 解法：代码枚举"计划"，Jev 在计划间取舍

**代码负责把 218 种出牌归并成 5–10 个语义不同的"计划"，Jev 只在这些计划之间做情境化取舍。**

这正好是 Mario 设计的严格推广：Mario 的 7 个宏动作是**人预先设计**的对按键空间的语义抽象；Balatro 的候选计划是**代码按局面生成**的对卡牌组合空间的语义抽象。"按键组合由 JoypadSpace 展开" 对应 "具体牌张索引由 `materialize()` 展开"。

举例（`SELECTING_HAND`，某具体局面）：

| criteria key | 描述（结构化，节选） | 代码 materialize 成 |
| --- | --- | --- |
| `play_flush` | `{"kind":"play","hand_type":"flush","cards":[0,2,3,6,7],"est_chips":1320,"clears_blind":false,"chips_left_after":1530}` | `play(cards=[0,2,3,6,7])` |
| `play_two_pair` | `{"kind":"play","hand_type":"two_pair","ranks":["K","9"],"cards":[1,4,5,7],"est_chips":420,...}` | `play(cards=[1,4,5,7])` |
| `play_high_card_ace` | `{"kind":"play","hand_type":"high_card","cards":[2],"est_chips":85,...}` | `play(cards=[2])` |
| `discard_for_flush` | `{"kind":"discard","cards":[1,4,5],"target":"flush","outs":9,"deck_remaining":34,"draw_probability":0.42}` | `discard(cards=[1,4,5])` |
| `use_consumable_0` | `{"kind":"use","index":0,"target_cards":[0,2,3,6,7],"reason":"The Star converts 3 cards to hearts"}` | `use(consumable=0, cards=[...])` |
| `sell_joker_2` | `{"kind":"sell","index":2,"refund":5,"reason":"..."}` | `sell(joker=2)` |

按"成牌类型"归并的好处是：**成牌类型本身就是一个近似静态的标签集**（high_card / pair / two_pair / three_kind / straight / flush / full_house / four_kind / straight_flush + 各档弃牌原型 + 道具 + 卖出），因此可以在很大程度上保持 criteria 的键稳定，只有描述随局面变化。这既符合官方"选项名和描述都会送给模型"的说明，也规避了"每次调用都换一整套 criteria 会不会有问题"的不确定性。

### 3.3 各决策点的具体设计

| 游戏状态 | 主 Choice（驱动动作） | 交叉印证 Noul | 仪表盘 Score | 备注 |
| --- | --- | --- | --- | --- |
| `BLIND_SELECT` | `blind_choice`: {`play_blind`, `skip_for_tag`}，描述里带 tag 名称与效果（代码提取 `tag_name`/`tag_effect`） | `should_skip` | `blind_pressure` | ★ **这是相对 balatrollm 的净增量**：balatrollm 代码里写死"永不 skip"，等于砍掉了一整个策略维度（它自己的分析文档也承认这点） |
| `SELECTING_HAND` | `plan`: 候选计划集（§3.2），5–12 个 | `discard_is_better_than_playing` | `blind_danger` | 主角 |
| `SHOP` | `purchase`: {`buy_card_0..n`, `buy_voucher_0..n`, `buy_pack_0..n`, `reroll`, `sell_joker_i`, `use_consumable_i`, `leave`}，描述带"买完剩多少钱 / 还差多少钱到下一档利息" | `should_leave_shop` | `economy_pressure` | 候选数动态但总量 ≤ ~12 |
| `SMODS_BOOSTER_OPENED` | `pack_pick`: {`take_0..n`, `skip`}，Tarot 类需要的 `targets` 由代码预先算好 | `pack_is_worth_taking` | — | |
| `ROUND_EVAL` | ❌ 不问 Jev，硬编码 `cash_out`（与 balatrollm 一致，无决策空间） | | | |
| `GAME_OVER` | 结束，记 `lost` | | | |

一次决策 = **一次 `system_one` 调用、2–3 个问题**，完全符合官方"把同一 state 的所有问题塞进一次请求"的建议。

### 3.4 一次决策的实际请求长什么样

把上面的设计落到具体 payload（节选，实际字段更全）：

```jsonc
// POST https://api.typesafe.ai/v1/systemone  { "state": …, "model": "jev-1.13.0", "questions": … }
{
  "state": {
    "objective": "Clear Ante 8. A failed blind ends the run.",
    "run":   { "ante": 4, "round": 12, "money": 12, "deck": "RED", "stake": "WHITE" },
    "blind": { "name": "The Hook", "type": "boss", "effect": "Discards 2 random cards per hand played",
               "target_chips": 4000, "current_chips": 1150, "chips_needed": 2850,
               "hands_left": 2, "discards_left": 1, "failing_ends_run": true },
    "hand":  [ {"i":0,"rank":"K","suit":"hearts","chips":10}, {"i":1,"rank":"4","suit":"clubs","chips":4},
               {"i":2,"rank":"A","suit":"hearts","chips":11}, /* … */ ],
    "jokers": [ {"i":0,"name":"Jolly Joker","effect":"+8 Mult if played hand contains a Pair","sell":3} ],
    "economy": { "interest_step_dollars": 5, "interest_cap": 5, "reroll_cost": 5 },
    "candidates": [ /* §3.2 的候选计划，逐个带 est_chips / clears_blind / draw_probability */ ],
    "recent_control": {
      "last_action": {"kind":"play","hand_type":"pair","est_chips":180,"actual_chips":172,"estimate_error":-8},
      "stalled_decisions": 0
    },
    "estimate_confidence": "low"   // 估算器还没建 Joker 效果表 → 数字仅供参考
  },
  "questions": {
    "plan": {
      "type": "choice",
      "instructions": {
        "question": "Which plan should be committed to for this decision?",
        "pressure": "`blind.chips_needed` is what remains; `blind.hands_left` plays are left. If no candidate clears the blind, prefer the plan that maximizes progress while keeping a viable follow-up.",
        "estimate": "`estimate_confidence` is low: treat `est_chips` as an ordering hint, not a fact.",
        "resource": "Discards are a scarce resource; do not spend one for a marginal improvement."
      },
      "criteria": { "play_flush": { /* 结构化事实，见 §3.2 */ }, "discard_for_flush": { /* … */ } }
    },
    "discard_is_better_than_playing": {
      "type": "noul",
      "instructions": "Is spending a discard now better than playing the best available hand?"
    },
    "blind_danger": {
      "type": "score",
      "instructions": "How much trouble is this blind in?",
      "criteria": ["Comfortably on track", "Tight but manageable", "Must hit a strong hand now", "Effectively lost"]
    }
  }
}
```

返回（Jev 保证答案落在给定 criteria 内，且给出完整概率分布）：

```jsonc
{
  "model": "jev-1.13.0",
  "answers": {
    "plan":     { "type":"choice", "choice":"play_flush", "confidence":0.58,
                  "probabilities": {"play_flush":0.58,"discard_for_flush":0.27,"play_two_pair":0.11,"play_high_card_ace":0.04} },
    "discard_is_better_than_playing": { "type":"noul", "noul":0.30 },
    "blind_danger": { "type":"score", "score":1.8, "confidence":0.49,
                      "legend": {"0":"Comfortably on track","1":"Tight but manageable","2":"Must hit a strong hand now","3":"Effectively lost"},
                      "probabilities": {"0":0.05,"1":0.36,"2":0.42,"3":0.17} }
  },
  "usage": { "input_tokens": 2810, "output_tokens": 37 }
}
```

→ 代码据此执行 `play(cards=[0,2,3,6,7])`，并把整条记录写进 `decisions.jsonl`。

（注：SDK 把 `Score` 的 `legend` / `probabilities` 的键从线上的字符串强制转成整数，并有 `response.choices` / `.nouls` / `.scores` 三个按类型过滤的视图；但**最稳的读取方式是按问题 id 直接读 `response.answers`**，这也是 SDK 文档里的字面契约。）

**一个必须注意的语义陷阱**：Choice / Noul / Score 三者的概率**不可互换**。同一个命题，"Noul 问法"和"Choice 问法"会给出差异很大的数值（官方 jaggedness 示例：Noul 给 0.22，Choice 给 0.01），而且 `P(是) + P(否) ≠ 1`。所以 §3.3 里那些交叉印证的 Noul **只能作为独立的第二信号写进日志**（正如 Mario 的 Noul 只用于事后一致性分析），**绝不能用它去覆盖或校正 Choice 的结果**——这也正好与"不设脚本兜底覆盖"的原则一致。

---

## 4. 事实层：Balatro 版的 `takeoff_deadline`

这一层是项目的主体，balatrollm 里没有任何对应物。

### 4.1 必须由代码算的"算术"

这不是我的偏好，而是 Jev 的硬约束：官方"已知弱项"里明确写着 **不做算术、不做计数**，并在构建指南里要求"**把算术留在代码里**""把数值在代码里换算成有名字的档位（hot / comfortable / cold）再传进去"。对应 Mario 把"接触帧数、起跳死线、落地预测"全部固化成 Python 的做法，Balatro 版需要：

1. **局面算术**：`chips_needed` = 目标分 − 当前分；`hands_left` / `discards_left`；失败是否直接结束本局（Balatro 里基本是"是"）。
2. **手牌评估**：给定牌张子集 → 成牌类型。需要处理 enhancement、seal、edition、debuff，以及会改变成牌判定的 Joker（Four Fingers / Shortcut / Smeared Joker 等）。
3. **计分估算**：`chips = 基础筹码 + 牌面筹码`，`mult = 基础倍率 + 卡牌加成`，`score = chips × mult`——**含 Joker 效果**。这是最难的部分：Balatro 有上百个 Joker，行为各异。见 §4.2 的分层策略。
4. **抽牌概率**：弃牌后补到目标牌的期望概率（超几何分布，精确可算）。若 balatrobot 的 `cards` 区域暴露完整牌堆（`GameState.cards: CardArea`），剩余牌堆构成可直接得到，不需要自己追踪已出牌。
5. **经济算术**：利息规则（每 $5 存款 $1，基础封顶 $5，voucher 会改）、下一次 reroll 价格、买完之后是否跌破利息档位。
6. **校准算术**：上一次决策的"估算分 vs 实际分"误差回灌——这是 Mario 的 `recent_control` + 延迟回灌在回合制下的等价物，也是这套架构里最有价值的一个闭环。

### 4.2 分层的估算器（`estimate_confidence` 是关键设计）

不要一上来就写完整计分模拟器——那是数周到数月的工作量（近似于重写 Balatro 的评分引擎）。建议分三层，并用一个显式的事实字段告诉 Jev 当前数字的可信度：

| 层级 | 覆盖范围 | `estimate_confidence` |
| --- | --- | --- |
| L1（MVP） | 只算牌型 + 牌面筹码 + 手牌等级基础值，**忽略 Joker 效果** | `low` |
| L2 | 给最常见的 20–40 个 Joker 建效果表（加法/乘法/条件触发）。**这张表不要手工录**——用模型离线把 `value.effect` 文案编译成结构化效果，产物入库（见 §4.3） | `medium` |
| L3 | 覆盖复杂 Joker（Blueprint/Brainstorm 复制、retrigger、随状态变化） | `high` |

**为什么这个设计是对的**：`estimate_confidence` 是 Mario 的 `terrain.observation_reliability`（`high` / `low_airborne`）的直接对应物——代码告诉 Jev"这个数字现在可不可信"，Jev 据此决定是"照着数字選"还是"别太当真、按原则走"。同时它让 MVP 可以先上线：L1 的估算虽然不准，但因为标注了 `low`，Jev 不至于被错误的精确数字带偏。

**这里有一条必须守住的界线**：代码只提供**事实与可行性**（确定性的算术），Jev 负责**取舍与权衡**（语义判断）。如果估算器精确到可以直接取 argmax，Jev 就退化成装饰品，项目也就失去了意义。这条界线要写进 README。

### 4.3 事实层用代码还是模型？—— "交给 LLM"有三种含义，结论完全不同

先澄清歧义，因为这三件事经常被混为一谈：

| 含义 | 形态 | 结论 |
| --- | --- | --- |
| **A. 逐次调用** | 每个决策点先问一次 LLM，让它输出事实 JSON，再喂给 Jev | ❌ **不推荐**（四条独立理由，见下） |
| **B. 离线编译** | 用模型把游戏里的**散文**（Joker 的 `value.effect` 文案）编译成结构化效果表，产物入库；运行时代码只查表 | ✅ **推荐，而且是必需的** |
| **C. 完全替代** | 不写事实层，让 Jev 直接看原始 gamestate | ❌ 不可行（Jev 官方文档明示不做算术/计数） |

#### A 为什么不划算：四条独立理由

1. **与 Jev 的契约自相矛盾。** 官方构建指南要求"**把算术留在代码里**"，并把"不做算术、不做计数"列为已知弱项。若事实由 LLM 产出、Jev 只做取舍，那么整条链路就有**两个概率模型串联**，"事实"本身也变成了模型输出——而这条架构本来最大的卖点，恰恰是**状态是观测的一个可验证的确定性函数**。
2. **它抹掉了采用 Jev 的理由。** 采用 Jev 的核心动机是"便宜、快、能高频"（~100ms / $0.0001 每次）。前面挂一个 frontier LLM 之后，每次决策的成本与延迟回到 balatrollm 的量级（约 $0.005–0.02、数秒），**Jev 的优势荡然无存**——那还不如直接用 balatrollm。
3. **引入第二个静默错误源，而且误差是"相关"的。** 代码算错：确定性、可复现、可单测、会以**校准误差尖峰**立刻暴露。LLM 算错：非确定性、不可复现、**输出的 JSON 依然合法**（完全静默），而且**同一个模型会重复犯同一个错**——这种系统性偏差会把整局数据整体带偏，比随机误差毒得多。考虑到本项目已经把"数据有效性"列为头号风险（R11），再叠一个静默源是反向操作。
4. **它会毁掉 §6.2 的实验设计。** 2×2 消融要回答的是"性能差异来自**模型**还是来自**代码算好的事实**"。如果"事实"本身由另一个模型产出，这一格就同时变了两个因子，归因彻底失效。

**一个必须承认的反方观点**：手写代码估算器很难，尤其 L1（忽略 Joker）大概率**打不过**一个能读 Joker 文案的 LLM。这个反驳成立——但它比较的对象错了。正确的比较不是"LLM vs 半成品代码"，而是"LLM vs 代码 + 由 LLM 编译出来的效果表"。**LLM 是好的脚手架，坏的运行时。**

#### B 才是模型真正该待的位置：离线编译一次，代码运行千万次

关键观察（附录 A 已记）：`value.effect` **是从游戏 UI 树里刮下来的本地化文案，不是结构化数据**——例如 `"Played cards with a rank of 2, 3, 4, or 5 each give +30 Chips when scored"`。要让代码算分，必须有人把这段自然语言变成可执行的效果模型。这是整个事实层里**唯一真正需要语义理解**的地方，而它同时具备两个宝贵性质：

- **静态**：效果文案按 `card.key` 固定不变 → 编译结果可以**按 card key 缓存并入库**，不需要每次决策重算；
- **可验证**：§4.1 的校准闭环（估算分 vs 实际分）就是效果表的**自动测试**。表填错 → 误差分布立刻偏移。这比人工 review 上百个 Joker 可靠得多，而且随着对局数据积累会越来越准。

于是 L2 从"几天的手工录表"变成"一次离线编译 + 持续用对局数据回归"。**这才是把 LLM 放进这个项目的正确姿势：让模型写数据，不让模型做决策。**

> 📄 **具体设计见同目录的 [`EFFECT_TABLE_DESIGN.md`](/Users/huangqingming/Workspace/balatrollm/EFFECT_TABLE_DESIGN.md)**：效果记录的 schema（封闭枚举的触发位点/效果种类/单位/谓词）、六步编译器流水线、三层验证体系、以及用**差分探针**让游戏本体裁决效果表正确性的方案。

顺带一个更省事的选择：把 Joker 归类到**原型桶**（加法倍率 / 乘法倍率 / 经济 / 成长 / 重触发 / 复制）本质上是"**固定标签集上的选择**"——正好是 Jev 的 `Choice` 原生能力，可以复用同一个依赖、不必再引一个 vendor。但 Jev 不生成文本，**无法输出带参数的表达式**，所以需要抽取参数的场合仍要用 LLM 的结构化输出或手写表。（实践建议：原型桶用 Jev/规则，参数先手写，长尾再上 LLM 编译。）

#### 一条可操作的决策规则

> **能用一个独立程序复算/断言它对不对 → 代码。**
> **能离线跑一次、把结果缓存成产物 → 模型离线编译，代码在线执行。**
> **只有"人看了觉得对"才能验证、且必须逐次做 → 才是真正的模型位。**

按这条规则过一遍事实层，结论几乎没有悬念：

| 事实层成分 | 归属 | 依据 |
| --- | --- | --- |
| 分差 / 剩余手数 / 弃牌数 / 利息 / reroll 价 | **代码** | 整数算术，可断言 |
| 手牌评估（含 enhancement/seal/debuff/改判 Joker） | **代码** | 组合规则，可断言；但需处理边界（stone/wild/4 fingers） |
| 抽牌概率（超几何） | **代码** | 纯数学，且 `cards` 已给出剩余牌堆 |
| 计分公式装配（chips × mult、效果叠加顺序） | **代码** | 可断言 |
| **Joker/消耗品效果文案 → 结构化效果模型** | **模型离线编译** | 唯一真正需要语义的地方，静态可缓存，可被校准闭环验证 |
| 雾战遮罩（`state.hidden` 仍返回真实牌值） | **代码（强制）** | 是**公平性不变量**，模型有可能"泄漏"隐藏牌 |
| 校准闭环（估算误差统计） | **代码** | 数值比较 |
| 候选枚举与 materialize | **代码** | 需要对 218 种组合的穷举与召回率度量；模型既无优势又会破坏确定性 |
| 局面叙事 / 建牌意图摘要 | **倾向代码** | "当前 build 特征"可由 Joker+牌堆构成确定性算出；散文叙事会撑大 state 并触发 Jev 的"冗长即掉精度" |

#### 额外收获：把 A 变成一个实验臂，而不是架构

既然 §6.2 的实验台会把"状态生产者"抽象成接口，那么"LLM 产出事实"其实**只是第三个生产者**，加进来的边际成本很低。于是真正的设计空间是 **3 生产者 × 2 决策头**：

| | Jev 决策 | LLM 决策 |
| --- | --- | --- |
| **代码产出事实** | ★ 目标臂 | 消融臂：对 LLM 而言事实层值多少 |
| **LLM 产出事实** | "级联小模型"臂（贵，但可测） | = balatrollm 现状（LLM 在脑内隐式推断事实） |
| **原始 JSON** | ❌ 不可行 | balatrollm 基线 |

注意最后一行：**"LLM 隐式推断事实"这一臂其实已经存在**，就是 balatrollm 本身。所以"LLM 产出事实"不是一个需要争论的架构选择，而是**一次可以顺便量出来的对照**——它直接回答"手写事实层到底值多少"。把这层歧义变成实验，比在架构上赌一边更划算。

---

## 5. 目标架构

```
                          typesafe-balatro
┌───────────────────────────────────────────────────────────────────┐
│ 外层工程骨架  ← 沿用 balatrollm                                      │
│   cli.py   config.py   executor.py   collector.py   views.py       │
│   批量任务笛卡尔积 / 并行实例端口池 / runs 落盘 / stats+batch 统计     │
├───────────────────────────────────────────────────────────────────┤
│ 决策链路  ← 形状沿用 typesafe-mario                                  │
│   state.py     gamestate JSON ──► 事实 JSON（英文 key）              │
│   actions.py   事实 ──► 候选计划 + materialize 成 JSON-RPC 调用       │
│   policy.py    事实 + 候选 ──► system_one ──► Decision              │
│   runner.py    状态机循环（同步，回合制，无异步流水线）                 │
├───────────────────────────────────────────────────────────────────┤
│ 交界件  ← 新写，替换 balatrollm 的 llm.py + strategy.py              │
│   jev.py        AsyncTypeSafeClient 封装 + 重试/超时/降级            │
│   scoring.py    牌型评估 + 计分估算（L1→L3）                          │
│   playbooks/<name>/  instructions.json + candidates.py + manifest   │
└───────────────────────────────────────────────────────────────────┘
        │ JSON-RPC 2.0                        │ HTTPS POST /v1/systemone
        ▼                                     ▼
   BalatroBot (Lua mod, 127.0.0.1:12346)   api.typesafe.ai (jev-1.13.0)
```

**双亲来源小结**：
- 工程外壳（批量、并行、落盘、CLI、配置、统计）来自 **balatrollm**；
- 决策链路形状（parser → policy → runner，`state-demo` 子命令，`HeuristicPolicy` 离线冒烟策略）来自 **typesafe-mario**；
- 被替换掉的是 balatrollm 的 `/Users/huangqingming/Workspace/balatrollm/src/balatrollm/llm.py`（OpenAI 客户端）与 `/Users/huangqingming/Workspace/balatrollm/src/balatrollm/strategy.py` + Jinja 提示词装配（Jev 没有"提示词"，只有 `instructions` + `criteria`）。

**建议保留的 playbook 机制**：`/Users/huangqingming/Workspace/typesafe-balatro/playbooks/<name>/` 目录承载 `instructions`（每个决策点的规则说明）+ 候选生成参数 + `/Users/huangqingming/Workspace/typesafe-balatro/playbooks/<name>/manifest.json`。这样"换一个文件夹 = 换一种打法"的属性得以保留——而这正是 balatrollm 能做"策略 × 模型"批量横评的结构基础，也是 Jev 版最值得继承的工程属性。（`/Users/huangqingming/Workspace/balatrollm/src/balatrollm/strategies/*/STRATEGY.md.jinja` 那 730 行 Balatro 教科书的内容，可以拆成 instructions 的素材，同时保留为人类可读文档。）

---

## 6. 落盘与实验设计

### 6.1 数据结构替换

`/Users/huangqingming/Workspace/balatrollm/src/balatrollm/collector.py` 的目录布局与聚合统计（`stats.json` / `batch.json` / `previous.json` / `latest.json` / `screenshots/`）可以直接沿用，只需替换两个 JSONL 的 schema：

`requests.jsonl`（OpenAI Batch 格式）→ **`decisions.jsonl`**，每行一条：

```jsonc
{
  "decision_index": 42,
  "state": "SELECTING_HAND",
  "facts": { /* 发给 Jev 的完整 state，便于离线复盘 */ },
  "candidates": [ {"id":"play_flush", "...": "..."} ],
  "answers": {
    "plan":     {"choice":"play_flush","confidence":0.63,"probabilities":{...}},
    "discard_is_better_than_playing": {"noul":0.21},
    "blind_danger": {"score":1.7,"confidence":0.55}
  },
  "action": {"method":"play","params":{"cards":[0,2,3,6,7]}},
  "estimate": {"est_chips":1320,"confidence":"low"},
  "outcome":  {"actual_chips":1180,"estimate_error":-140,"cleared_blind":false},
  "latency_ms": 412,
  "usage": {"input_tokens":2810,"output_tokens":37},
  "request_id": "…"
}
```

这一条记录同时服务四个目的：数据复盘、置信度校准分析、估算器误差分析、以及 dashboard/views 可视化。

**⚠️ 落盘层的三条硬要求（源自 §1.1 的上游缺陷）：**

1. **不要相信 `won` 字段。** balatrollm 的 `collector._calculate_stats` 直接读 `gamestate["won"]`（`/Users/huangqingming/Workspace/balatrollm/src/balatrollm/collector.py:426`）——这正是上游 #232 点名的失真路径。新仓库必须以 `state == "GAME_OVER"` + 最终 ante/round + 是否真的打完 Boss Blind 做交叉校验，并把 `won_raw` 与 `won_verified` 都落盘，便于事后审计。
2. **记录实例身份**（端口 + 进程启动时间 + 本次是该实例的第几局），这样一旦出现跨局泄漏可以事后定位是哪几局受影响。
3. **`round.chips` 的读取时机要小心**（上游 #231：`cash_out` 可能在结算行写入前触发，导致少收甚至 $0）。校准闭环（估算 vs 实际）依赖这个数，取数点要固定并记录取值时刻的状态。

### 6.2 对照实验设计（这才是这个项目真正的产出）

因为 balatrollm 已经跑通"LLM + 原始 JSON"这一臂，把 policy 抽象成 Protocol（照 typesafe-mario 的 `Policy.choose`）后，可以天然形成 2×2 消融：

| | 原始 gamestate JSON | 代码算好的事实 JSON |
| --- | --- | --- |
| **LLM**（tool calling） | balatrollm 现状（现成基线） | 消融臂：隔离事实层的贡献 |
| **Jev** | ❌ 不可行（Jev 不做慢思考） | ★ 本项目目标臂 |

这张表能回答一个真正有价值的问题：**性能差异究竟来自模型，还是来自代码算好的事实？** 只报"Jev 打败了 LLM"是没有信息量的；能拆开这两个因子才有信息量。

主指标沿用 balatrollm 已有的：固定种子集上的胜率 + 到达的 ante/round。次指标：Jev 置信度与最终结果的校准度（confidence 高时真的更容易过盲注吗）、估算误差分布、决策延迟、平均 token。

> **⚠️ 但这套实验设计有两个前提条件，必须先满足（见 §1.1）：**
> 1. **一局一进程**。上游 #232 明确说：长生命周期实例 + `menu`/`start` 复用会让**固定种子不可复现**（商店/消耗品结果受同实例上之前跑过的种子影响）。固定种子对照是这张 2×2 表的地基，地基不稳则整张表无意义。
> 2. **不用 `--fast`**。上游 #234：`--fast` 在第一次 Tarot/Spectral 卡包之后就不再保种子。吞吐量换数据有效性——这个取舍必须显式做，不能默认开着 fast 图快。
>
> 这两条把批量吞吐压到原计划的 1/3–1/2 左右，排期时要按这个前提算。

**另有两条零成本的基线必须一起做**：
- `HeuristicPolicy`（纯代码，照 evalatro 的 `naiveDecide` 那种 20 行启发式）——**不花一分钱就能验证整条链路**，这是 typesafe-mario 里 `--policy heuristic` 的用法，务必保留。
- `ScriptedPolicy`（固定动作序列）——验证状态机推进与落盘。

---

## 7. 分阶段实施计划

| 阶段 | 内容 | 交付/验证方式 | 估时 |
| --- | --- | --- | --- |
| **M0 环境打通** | Steam + Balatro + Lovely + Steamodded + balatrobot mod + `uv tool install balatrobot`；用 curl 打通 `gamestate`。⚠️ 两个坑：① **mod 没有预编译发行包**（上游 29 个 release 全部零附件），"下载最新版"指的是下载**整仓库源码压缩包**并把根目录改名成 `~/Library/Application Support/Balatro/Mods/balatrobot/`（Windows 机器同理，目标是 `%APPDATA%\Balatro\Mods\balatrobot\`）；② **先裸跑一次游戏**——macOS 26.6.2 + M1 能否原生起来见 R12，Windows 机器按 [`environments/windows.md`](environments/windows.md) 的首次验收清单执行 | 手动 `curl` 拿到一份**真实 gamestate JSON 存档**（极其重要，后续离线开发的燃料）+ 记录实际可用的 Lovely/Steamodded 版本 | 0.5–1.5 天（多为下载等待与排障） |
| **M1 骨架** | 仓库脚手架（pyproject / ruff / pytest / CI 照 typesafe-mario）、搬 `/Users/huangqingming/Workspace/balatrollm/src/balatrollm/client.py`、`/Users/huangqingming/Workspace/typesafe-balatro/jev.py`（含 stub transport 注入点）、`HeuristicPolicy`、`state-demo` 子命令；**pin `balatrobot>=1.5.2`** | **不接游戏、不接 API** 也能跑单测；`state-demo` 打印事实 JSON；CI 用 stub transport，零成本 | 1–2 天 |
| **M2 事实层** | `/Users/huangqingming/Workspace/typesafe-balatro/state.py`：`SELECTING_HAND` / `BLIND_SELECT` 的事实抽取（分差、手牌、Joker、经济、近期反馈） | 用 M0 的真实 JSON 存档做离线快照测试（golden test，照 balatrollm `/Users/huangqingming/Workspace/balatrollm/tests/fixtures/` 的做法）；用 `set` 端点构造边界局面 | 2–3 天 |
| **M3 候选层 + 估算器** | `/Users/huangqingming/Workspace/typesafe-balatro/scoring.py`（L1 牌型评估）+ `/Users/huangqingming/Workspace/typesafe-balatro/actions.py`（候选计划枚举 + materialize）；**外加效果表编译器**（§4.3 的 B 路径，详见 [`EFFECT_TABLE_DESIGN.md`](/Users/huangqingming/Workspace/balatrollm/EFFECT_TABLE_DESIGN.md)，其中约 3.5 天可与 M1 并行、不依赖游戏） | 离线单测：给定手牌 → 候选集正确且能 materialize 成合法调用；效果表由**差分探针**（对着真实游戏的固定种子量出单张 Joker 的贡献）+ 校准闭环回归 | 2–4 天 + 效果表 5.5–6 天（**最硬的一段**） |
| **M4 Jev 策略层** | `/Users/huangqingming/Workspace/typesafe-balatro/policy.py`：问题构造 + 答案解析 + `decisions.jsonl` 落盘 | 先用 stub / 本地 `jeff` 打通契约，再接真实 API 跑通一局 | 1–2 天 |
| **M5 商店/卡包/消耗品** | SHOP、`SMODS_BOOSTER_OPENED` 的事实与候选 | 完整一局（含商店）跑通 | 1–2 天 |
| **M6 批量并行 + 统计** | 改造 `/Users/huangqingming/Workspace/typesafe-balatro/executor.py` / `/Users/huangqingming/Workspace/typesafe-balatro/collector.py` / `/Users/huangqingming/Workspace/typesafe-balatro/cli.py`；**重做实例生命周期：一局一进程或每 N 局回收；随机端口；单实例串行调用；`won` 交叉校验；调用级超时+重启（应对 #195/#198/#199）** | 固定种子集批量跑，产出 `stats.json` / `batch.json`；**先跑同一批种子两遍，确认结果可复现**（这是数据有效性的验收条件） | 2–3 天（因 R11 比原估多 1 天） |
| **M7 对照实验 + 调参** | 2×2 消融、instructions 迭代、估算器 L2 | 报告 | 3–5 天（持续） |

**关键的可测试性结论**：M1–M3 完全不依赖 Balatro、不依赖 TypeSafe API——只要有 M0 阶段存下来的**一份真实 gamestate JSON**，就能写完整的离线单测。因此强烈建议：

1. **M0 的第一件事就是把真实 gamestate 存档到仓库里**（脱敏后作为测试 fixture）；
2. **把真实 Jev 响应录下来当回放 fixture**（照 TypeSafe 自家 cookbook 的 `cooksafe.JsonCache` 做法：缓存真实响应，测试时以 `cache-only` 模式无 key 回放）。这样 `/Users/huangqingming/Workspace/typesafe-balatro/policy.py` 的解析逻辑、`decisions.jsonl` 的 schema、置信度阈值逻辑都能进入 CI，而且**测试是确定性的**——考虑到 Jev 本身没有 `seed`/`temperature`，这一点尤其重要；
3. CI 里用 mock transport 或录制的 fixture，**零成本、不需要 API key**。

---

## 8. 风险清单与缓解

| # | 风险 | 等级 | 缓解 |
| --- | --- | --- | --- |
| R1 | **本机零环境**：没有 Steam/Balatro/balatrobot/API key | 🔴 阻塞端到端 | 三条缓解：① 第一天并行推进 M0；② 先把一份真实 gamestate 存档入库，解耦 M1–M3；③ Jev 侧用 stub transport 或本地 `jeff` 自托管替身，**不需要 API key 也能开发与 CI** |
| R2 | **计分估算器**（含 Joker）是长尾工程，容易失控 | 🟠 高 | 三层策略 + `estimate_confidence`；MVP 只做 L1；绝不追求完整模拟器 |
| R3 | **Jev 精度随 state 变长而漂移**（官方 "Model jaggedness"） | 🟠 中 | 事实 JSON 控制在 2–5k token；对抗性地裁剪字段；用 `state-demo` 人工审阅"模型到底看到了什么" |
| R4 | **候选集质量决定上限**：漏掉最优计划，Jev 无从选择 | 🟠 中 | 候选生成要**召回优先**（宁可多给几个）；用 `decision.candidates` 落盘统计"事后最优在不在候选里"的召回率。官方对此有明确警告："**模型无法选择被省略的选项**"——这是有文档背书的失败模式，必须用指标盯着 |
| R5 | **Jev 无 reasoning 文本**，可解释性弱于 LLM | 🟡 低 | 概率分布 + 置信度落盘；必要时加探针 Noul/Score；这是可接受的取舍 |
| R6 | **英文约束**：CJK 精度较低 | 🟡 低 | 事实 JSON 与 playbook 全英文；人类文档可以中文 |
| R7 | **macOS 多实例**并行开窗口的资源开销（GPU/内存） | 🟡 中 | 先用 `--parallel 1` 打通；注意 `--headless` **不是无窗口**（只停止绘制，LÖVE 窗口与 GL/Metal 上下文仍要创建），所以仍然需要显示环境与可用 GPU |
| R8 | **限流与配额**官方声明"动态调整" | 🟡 低 | 记录 429 与 `retry-after`；批量跑时控制并发；成本本身可忽略 |
| R9 | **Jev 模型版本漂移**（`jev-latest` 别名会移动） | 🟡 低 | 实验时固定 `jev-1.13.0`，并记录返回体里的 `model` 字段 |
| R10 | **哲学风险**：事实层写得越聪明，Jev 越像装饰品 | 🟠 中 | 把 §4.2 那条界线写进 README，并用 2×2 消融量化"事实层"与"模型"各自的贡献 |
| **R11** | ★★ **BalatroBot 上游未修复的静默正确性缺陷**（#232 跨局 `won` 泄漏、#234 `--fast` 破坏种子、#235 长会话退化、#231 `cash_out` 少收；另有 #195/#198/#199 三种必挂场景） | 🔴 **高** | 一局一进程（或每 N 局回收）+ 每局前校验状态；**计分实验禁用 `--fast`**；不信任 `won`，用 `state`+ante/round 交叉校验；对已知挂死场景加超时与重试；**pin `balatrobot>=1.5.2`**（1.5.2 修了并发崩溃 #193/#196，而 balatrollm 自己还 pin 在 1.4.1）。这条是**对实验规模影响最大的一条** |
| **R12** | **本机平台未知数**：macOS 26.6.2 + Apple M1 上 Balatro 能否原生运行（2025 年 6 月有 macOS 26 beta 崩溃、需强制 Rosetta 的用户报告）；Lovely 0.9.0 与 Steamodded 26.829.0 均无与 balatrobot 1.5.2 的实测记录。Windows 侧同样待实测（见 [`environments/windows.md`](environments/windows.md)） | 🟠 中 | **M0 第一件事就是单跑一次游戏**（不接任何代码），确认能启动、能连上 `health`；若原生崩溃，退回 x86_64 `liblovely.dylib` + Rosetta；记录实际使用的 Lovely / Steamodded 版本到 README。Windows 机器首跑按 [`environments/windows.md`](environments/windows.md) 首次验收清单执行，结果回填该文件 |

### 待确认（需要实际接触 API / 游戏后才能定性）

**已排除的疑问**（本次评估中已经查清，不再是不确定项）：

- ~~macOS 是否支持~~ → `balatrobot` 有专门的 macOS launcher，内置正确默认路径；但**游戏不能用 Steam 客户端启动**（Steam 的已知 bug），launcher 会直接 exec LÖVE 并注入 `DYLD_INSERT_LIBRARIES`。
- ~~Windows 是否支持~~ → `balatrobot` 有专门的 Windows launcher（`platforms/windows.py`），内置正确默认路径（`C:\Program Files (x86)\Steam\steamapps\common\Balatro\Balatro.exe` + 同目录 `version.dll`），直接 exec `Balatro.exe`，Lovely 走 `version.dll` DLL 注入（无 `DYLD` 机制）；`BALATROBOT_*` 环境变量与 macOS 完全一致，executor 的启动方式结论同样成立（详见 [`environments/windows.md`](environments/windows.md)）。
- ~~能否加速游戏 / 无头运行~~ → `fast`（10x）、`headless`、`no_shaders`、`gamespeed`（默认 4）、`animation_fps`（默认 10）全是 `Config` 字段，可用 `BALATROBOT_*` 环境变量配置。**但见 §1.1：`--fast` 不能用于计分实验。**
- ~~动态 criteria 是否安全~~ → 官方明确"不做微调、同一套权重服务所有账号"，criteria 完全属于请求体，没有任何注册/编译产物，也没有缓存惩罚；官方 cookbook 里就有每次调用重建选项集的用法。
- ~~`cards` 是否暴露完整牌堆~~ → 是，`cards` 是**抽牌堆剩余牌**（Area over `G.deck`），每张是完整 Card 对象。**抽牌概率可以直接算，不需要自己追踪已出牌**（只需自己按 `key` 聚合构成）。
- ~~gamestate 字段表~~ → 已逐字段核对上游 `gamestate.lua` / `types.lua`，关键结论见 §4.1 后的"集成要点"。

**仍然待实测**：

- Jev 的实际单次延迟（文档/第三方一致给 70–500ms、典型 ~100ms，可信度较高，但需自己测一次）。
- 候选集频繁变动时，**答案的稳定性**是否下降（机制上无惩罚，但精度曲线要自己量；§3.2 的"稳定标签集"设计已是对冲）。
- 一局从开局到结束的实际墙钟时间（决定批量实验的规模上限）。BalatroBot 自身会按请求打印 `%s OK (%.0fms)`，以你的日志为准（注意：网上流传的 BalatroBench `time_avg_ms ≈ 34s` 是 **LLM** 延迟，不是游戏动作延迟）。
- 上游 issue 报告的三种**必挂场景**在你实际会碰到的频率：#195（`sell` 对 Invisible Joker 卡死）、#198（薄牌堆时 `buy` 卡死）、#199（Celestial 包首张是 Black Hole 时卡死）——需要给每次调用加超时与重启逻辑。
- **macOS 26.6.2 + M1 上 Balatro 能否原生运行**，以及 Lovely 0.9.0 / Steamodded 26.829.0 与 balatrobot 1.5.2 的组合是否可用（见 R12）。
- **Windows 首次实测**：目标 Windows 版本上 `Balatro.exe` + Lovely `version.dll` 注入能否顺利启动、`BALATROBOT_*` 环境变量行为是否与 macOS 一致（清单与验收步骤见 [`environments/windows.md`](environments/windows.md)）。

---

## 9. 需要你决策的问题

1. **Balatro 环境**：本机没有 Balatro，是否已有 Steam 账号/游戏，或准备购买？这决定 M0 能否启动。
   - 若暂时不想买：Jev 侧可以用 stub transport / 本地 `jeff` 自托管**零成本**推进 M1–M3；但 M5 之后的端到端验证仍然绕不开游戏本体。
   - 若不方便在这台机器装：可以只在**另一台机器**上跑一局、把真实 gamestate JSON 导出到本仓库当 fixture，本机专心做事实层与候选层（另一台若为 Windows，环境先按 [`environments/windows.md`](environments/windows.md) 检查）。
   - 另一条参考路径：`/Users/huangqingming/Workspace/evalatro`（同工作区）已经写好了 macOS 的完整安装脚本（`/Users/huangqingming/Workspace/evalatro/scripts/setup-local.mjs`），可以直接照抄安装步骤，省去踩坑。
2. **`typesafe-balatro` 与 balatrollm 的耦合方式**：是把 balatrollm 作为 pip 依赖直接 import（它 `/Users/huangqingming/Workspace/balatrollm/src/balatrollm/__init__.py` 确实导出了公共 API，可行但版本耦合），还是**把基础设施拷进新仓库并加 MIT 归属说明**（balatrollm 是 MIT，允许；改造量大，建议后者）？
3. **项目重心**：目标是"做出一个 Jev 玩 Balatro 的 agent"（偏 demo，可以只跑单车、配 dashboard），还是"做一个能做 2×2 消融的评测台"（偏 benchmark，需要批量并行 + collector）？后者工作量大约多 30%，但产出的结论有价值得多。
   - ⚠️ 这个选择会**显著改变成本结构**：demo 路线可以开 `--fast`、可以复用实例、可以信 `won`，一局几分钟；benchmark 路线必须按 §1.1 / §6.2 的约束做（禁用 `--fast`、一局一进程、`won` 交叉校验），**单局墙钟时间和机器负载都会明显上升**，能跑的实验规模要按这个前提倒推。
4. **是否同时接 LLM 基线**：若要做 §6.2 的消融，需要在新仓库里保留一条 LLM 通路（即保留 balatrollm 的 `/Users/huangqingming/Workspace/balatrollm/src/balatrollm/llm.py` 与 Jinja 体系）；若只想验证 Jev 路线，可以完全删掉，代码量减少约 1/5。
5. **输出语言**：事实 JSON / criteria 必须英文（Jev 精度），但仓库 README、文档、报告用中文还是英文？
6. **是否接受"数据有效性优先于吞吐量"**：这是 R11 逼出来的取舍。若你更看重快速看到结果，可以选择 demo 口径（开 `--fast`、复用实例），但**必须放弃"固定种子对照"这个卖点**，报告里也要说明；若要做严谨对照，就要接受慢。

---

## 附：一句话总结

> **这不是"把 LLM 换成小模型"，而是"把 LLM 现在兼职干的两件事拆开"——事实推断交给代码（新写、最难、最有价值），情境取舍交给 Jev（轻、快、便宜、且结构上不可能出非法动作）。balatrollm 能借的是外壳，typesafe-mario 能借的是形状，真正的项目内容在那层 Balatro 的事实算术上。**
>
> **而在这之前还有一层前提：BalatroBot 上游存在未修复的静默数据损坏问题，balatrollm 的实例复用模型正好踩在上面。做 demo 无所谓，做基准就必须先把这层地基换掉——这件事的优先级高于写任何 Jev 代码。**

---

## 附录 A：BalatroBot 集成要点（写 `/Users/huangqingming/Workspace/typesafe-balatro/state.py` / `/Users/huangqingming/Workspace/typesafe-balatro/actions.py` 时直接照这份）

来源为上游 `gamestate.lua` / `types.lua` / `openrpc.json` 逐文件核对，能省掉大量试错：

| 事项 | 结论 |
| --- | --- |
| **目标分在哪** | `chips_needed` **不是现成字段**，要算 `blinds.<current>.score − round.chips`；`round` 里只有 `hands_left / hands_played / discards_left / discards_used / reroll_cost / chips` |
| **手牌等级** | `hands` 是以牌型名为键的 map，等级在 `hands.<Name>.level`，另有 `chips` / `mult` / `played` / `played_this_round` / `example`（`[card_key, 是否计分]` 对） |
| **缺字段陷阱** | 布尔字段**缺省时被省略而不是 `false`**（`state.debuff`、`modifier.eternal` 等），所有读取必须用 `.get()` |
| **卡牌稳定标识** | `card.id` = `sort_id`，是跨调用追踪同一张牌的稳定标识（比索引可靠——索引会随弃牌/抽牌漂移） |
| **索引语义** | API 全部 **0-based**；`play` 的牌数上限是 `hand.highlighted_limit`（通常 5） |
| **雾战** | `state.hidden` 表示牌面朝下（Boss Blind 的 fog）；`evalatro` 特意在 summarizer 里自己遮罩，因为 **balatrobot 仍会返回真实牌值**——要打公平就必须自己遮 |
| **牌堆** | `cards` = 抽牌堆**剩余**牌（不是全牌堆），按 `key` 聚合即得剩余构成 → 抽牌概率可直接算 |
| **过渡态要轮询** | `HAND_PLAYED` / `DRAW_TO_HAND` / `NEW_ROUND` 是动画过渡态，只能 `gamestate` 轮询；balatrollm 的 `case _: sleep + 轮询` 兜底分支正是为此，务必保留 |
| **写测试的利器** | `set` 端点无状态门禁，可直接改 `money`/`chips`/`ante`/`round`/`hands`/`discards`，还能 `shop` 重刷；`add` 可注入任意 joker/消耗品/卡牌。**离线快照测试与"把局面直接摆到 ante 8"都靠它** |
| **协议限制** | 单请求体上限 **64 KB**、HTTP/1.1 `POST /`、每响应 `Connection: close`、**单客户端**（1.5.2 起拒绝并发连接）。所以**每个实例的调用必须串行**，不要对同一端口并发发请求 |
| **端口** | 端口在重启后仍被占用约 **20–30s**；上游测试用 12346–23456 的随机端口，balatrollm 用连续端口——**改成随机端口更稳** |
| **截图** | `headless` 下 `screenshot` 不可用；需要截图就改用 `render_on_api`（与 headless 互斥） |
