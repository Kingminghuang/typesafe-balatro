# 效果表编译器设计（Effect Table Compiler）

> 本文是 `/Users/huangqingming/Workspace/balatrollm/TYPESAFE_BALATRO_FEASIBILITY.md` §4.3 B 路径的落地设计：把 Balatro 卡牌的**本地化效果文案**离线编译成**结构化效果表**，供 `state.py` / `scoring.py` 在运行时查表打分，并用校准闭环回归验证。
>
> 目标仓库：`/Users/huangqingming/Workspace/typesafe-balatro`（当前为空仓库）
> 对应里程碑：M3（`/Users/huangqingming/Workspace/typesafe-balatro/scoring.py` + `/Users/huangqingming/Workspace/typesafe-balatro/actions.py`）与 M7（回归）

---

## 1. 目标与非目标

**目标**

1. 用模型把 `card.value.effect`（游戏 UI 刮出来的英文散文）转换成**可执行的数据**，一次编译、长期复用。
2. 编译产物必须**可断言**：能用测试验证，能在 PR 里 diff，能在 CI 里跑。
3. 产物必须能回答"**我现在算不准，是哪几张牌导致的**"——直接驱动 `estimate_confidence`。
4. 正确性由**游戏本身**裁决，而不是由人 review 裁决。

**非目标**

- ❌ 不追求 100% 覆盖全部卡牌。目标是**覆盖实际会遇到的**，长尾显式标 `unmodeled`。
- ❌ 不做通用表达式引擎。**没有 `eval()`，没有 DSL 解释器。** 见 §4.6。
- ❌ 不模拟随机性的具体取值（Lucky Card 之类），只建模**期望值**并标注 `stochastic`。
- ❌ 不在运行时调用任何模型。运行时只读表。

**核心设计原则**

> **模型写数据，代码做决策；编译离线，执行在线。**

---

## 2. 为什么这是"编译器"而不是"提示词"

对比一下两种做法：

| | 运行时提示词（§4.3 的方案 A） | 离线编译（本方案） |
| --- | --- | --- |
| 调用次数 | 每个决策 1 次 | **每张牌 1 次，之后永久缓存** |
| 输出形态 | 自由散文或 JSON，每次都可能不同 | **受 schema 约束的数据，签入仓库** |
| 可测试性 | 无法回归 | schema 测试 + 探针测试 + 覆盖度指标 |
| 出错表现 | 静默、不可复现 | **CI 失败 / 探针失败 / 误差分布偏移** |
| 成本 | 每局 $1–6 | **一次性几美元** |

关键点：**内容寻址**。记录的键是 `(card_key, sha256(effect_text))`。文案没变就永不重编；游戏更新导致文案变化时，哈希不匹配 → 自动标记为 stale → 触发重编。这让"游戏版本升级"从一个隐患变成一次可审计的 diff。

---

## 3. 第一层契约：语料（corpus）

编译器消费的原始素材。**这一步不需要模型，只需要游戏。**

```jsonc
// /Users/huangqingming/Workspace/typesafe-balatro/corpus/cards.jsonl  — 每行一张牌，追加写入，按 key 去重
{
  "key": "j_fibonacci",
  "label": "Fibonacci",
  "set": "JOKER",              // JOKER | TAROT | PLANET | SPECTRAL | VOUCHER | ENHANCED | DEFAULT | BOOSTER
  "rarity": "Uncommon",        // 仅 Joker 有
  "effect_text": "Each played Ace, 2, 3, 5, or 8 gives +8 Mult when scored",
  "cost_buy": 8,
  "cost_sell": 4,
  "source": "add_sweep",       // gamestate_harvest | add_sweep | localization
  "first_seen": "2026-09-21T10:00:00Z",
  "game_version": "1.0.1n",
  "collector_mod_version": "1.5.2"
}
```

**三条采集途径**（按优先级）：

1. **`add` 端点批量注入**（最优）：调 `add(key=...)` 把牌直接塞进 `jokers`/`consumables` 区域，然后读 `gamestate` 拿到 `label` / `value.effect` / `rarity` / `cost`。这样**不需要真的遇到这张牌**，可以在一局废弃对局里扫完整个卡池。注意 `add` 需要 `--debug` + DebugPlus，且对状态与栏位有要求（见可行性报告附录 A）。
2. **顺手采集**：每次拿 `gamestate` 时，把见到的所有 card 记录进语料。零成本，覆盖自然增长。
3. **本地化文件**：游戏安装目录里的本地化文案是静态文件，可直接解析。覆盖最全，但需要处理键名映射与本地方言。

> ⚠️ 采集用的对局是**一次性**的，不要与计分实验共用进程（上游 #232 的状态泄漏，见可行性报告 §1.1）。

**语料完整性指标**：`corpus_coverage = len(corpus) / 已知卡池大小`。卡池大小从游戏 Wiki 或本地化文件取一个期望值，作为 CI 断言（低于阈值时告警，不阻断）。

---

## 4. 第二层契约：效果记录（effect record）

**这是本文档的核心。** 设计目标是：**封闭枚举 + 参数化 + 显式逃生舱**。

```jsonc
// /Users/huangqingming/Workspace/typesafe-balatro/effects/compiled/j_fibonacci.json
{
  "key": "j_fibonacci",
  "effect_text_hash": "sha256:9f2c…",      // 内容寻址，用于 stale 检测
  "schema_version": 1,
  "compiled": {
    "model": "gpt-5.2",                     // 编译器实现自由，但必须记录
    "prompt_version": "compile-v3",
    "at": "2026-09-21T11:02:00Z"
  },
  "review": {
    "status": "auto_validated",             // unreviewed | auto_validated | human_reviewed
    "note": null
  },
  "classified_as": "scaling_per_hand",      // §4.7 的原型桶，用于覆盖率统计
  "triggers": [
    {
      "site": "card_scored",                // 触发位点（封闭枚举，§4.1）
      "condition": {
        "all": [ { "card_rank_in": ["A", "2", "3", "5", "8"] } ]
      },
      "applies": "per_match",               // per_match | once
      "effect": {
        "kind": "add_mult",                 // 效果种类（封闭枚举，§4.2）
        "value": { "const": 8 }
      }
    }
  ],
  "stochastic": false,
  "unmodeled": null
}
```

一个 `unmodeled` 的例子（逃生舱，必须写明理由）：

```jsonc
// /Users/huangqingming/Workspace/typesafe-balatro/effects/compiled/j_blueprint.json
{
  "key": "j_blueprint",
  "effect_text_hash": "sha256:1a77…",
  "schema_version": 1,
  "compiled": { "model": "gpt-5.2", "prompt_version": "compile-v3", "at": "2026-09-21T11:02:00Z" },
  "review": { "status": "human_reviewed", "note": "order-dependent copy; evaluated at L3" },
  "classified_as": "copy",
  "triggers": [],
  "stochastic": false,
  "unmodeled": {
    "reason": "Copies the ability of the Joker to the right; requires order-aware evaluation",
    "intended_layer": "L3"
  }
}
```

### 4.1 触发位点 `site`（封闭枚举）

| site | 含义 | 求值时机 |
| --- | --- | --- |
| `joker_static` | 常驻，无每手条件 | 每次计分手牌时一次 |
| `hand_played` | 打出的手牌整体触发一次 | 计分开始时 |
| `card_scored` | 逐张计分牌触发 | 每张计分牌 |
| `card_held` | 留在手里的牌触发（Steel、部分 Joker） | 计分末尾 |
| `card_discarded` | 弃牌时触发 | 弃牌动作后 |
| `blind_start` / `round_end` | 盲注/回合边界（经济类） | 状态跨越时 |
| `shop_enter` | 进商店时 | SHOP 状态 |

### 4.2 效果种类 `effect.kind`（封闭枚举）

| kind | 含义 | 备注 |
| --- | --- | --- |
| `add_chips` | 加筹码 | |
| `add_mult` | 加倍率 | |
| `multiply_mult` | 乘倍率（×1.5 / ×2 / …） | |
| `retrigger` | 重触发（本牌或全部计分牌） | |
| `add_money` | 加钱 | 影响利息与商店决策 |
| `set_hand_type` | 改变成牌判定（Four Fingers / Shortcut 类） | 影响 §5 的手牌评估 |
| `modify_card` | 永久改牌（Hiker 类） | L3 |
| `unmodeled` | 显式放弃 | 必须带 reason |

**故意不设** `eval` / `script` / `formula` 这类自由字段。见 §4.6。

### 4.3 幅值 `effect.value`

三种形态，闭集：

```jsonc
{ "const": 8 }                                       // 常数
{ "per_unit": { "unit": "money", "step": 5,
                "amount": 1, "cap": 5 } }            // 每 unit 每 step 给 amount，可封顶
{ "table": { "Pair": 8, "Two Pair": 12 } }           // 按手牌类型查表的常数
```

### 4.4 单位 `unit`（封闭枚举 —— 这是"代码算好的事实"的词汇表）

编译器只能引用这些**求值器已经能算出来的**量。这份表就是编译器的词汇表，也是文档的一部分：

| unit | 语义 |
| --- | --- |
| `money` | 当前金钱 |
| `hands_left` / `discards_left` | 本回合剩余手数/弃牌数 |
| `hand_level` | 当前成牌类型的等级 |
| `deck_size` | 抽牌堆剩余张数 |
| `jokers_owned` / `empty_joker_slots` | Joker 数量 / 空槽 |
| `cards_discarded_this_round` | 本回合已弃牌数 |
| `cards_played_this_round` | 本回合已出牌数 |
| `scoring_cards` | 本次计分牌张数 |
| `matching_cards` | 命中 `condition` 的牌张数（配合 `per_match`） |
| `ante` / `round` | 当前 ante / 回合序号 |

### 4.5 条件 `condition`（封闭谓词）

叶子谓词全部从固定词汇表取，支持 `all` / `any` / `not` 组合：

```jsonc
{ "all": [
    { "hand_type_in": ["Flush", "Straight"] },
    { "any": [
        { "card_rank_in": ["A", "K"] },
        { "card_enhancement_in": ["GLASS"] }
    ]},
    { "not": { "joker_count_ge": 5 } }
]}
```

叶子谓词清单（初版）：

- 牌相关：`card_rank_in` / `card_suit_in` / `card_enhancement_in` / `card_seal_in` / `card_edition_in` / `card_is_stone` / `card_is_wild`
- 手牌相关：`hand_type_in` / `hand_contains`（Pair/Two Pair/…）/ `played_count_eq|ge|le` / `scoring_card_count_ge`
- 局面相关：`joker_count_ge|le` / `money_ge|le` / `hands_left_le` / `discards_left_le` / `ante_ge` / `deck_size_le` / `deck_suit_count_ge`

**每条谓词都必须有对应的求值实现**，由 `/Users/huangqingming/Workspace/typesafe-balatro/tests/test_conditions.py` 强制：谓词词汇表与实现一一对应，缺一即 CI 失败。

### 4.6 为什么不用表达式字符串 / 为什么要闭集

一个很自然的诱惑是让模型直接输出 `"score += 8 * count_rank(cards, ['A','2','3','5','8'])"` 然后 `eval`。**必须拒绝**，三个理由：

1. **安全**：`eval` 执行模型生成的代码，等于把任意代码执行权交给一个概率模型。
2. **不可验证**：字符串无法被 schema 校验，编译器可以把操作符写错而你毫无察觉。
3. **会越界**：模型会发明求值器不支持的算子，运行时才崩——错误从"编译期"推迟到"第 200 次决策时"。

闭集把模型的输出变成**可执行的数据**而不是**代码**。代价是表达力，但表达力不够时还有 `unmodeled` 逃生舱——而逃生舱是**可度量**的（覆盖率指标），比静默的错误表达式好得多。

### 4.7 原型桶 `classified_as`（用于覆盖率统计与长尾管理）

`add_mult` / `add_chips` / `multiply_mult` / `retrigger` / `copy` / `economy` / `scaling_per_hand` / `scaling_per_round` / `hand_type_modifier` / `card_modifier` / `random` / `conditional_big`

这个字段**不参与计分**，只用于：统计"我们覆盖了哪几类机制"、给编译器提示（"先归类，再细化"）、以及发现"某个桶全军覆没"这类系统性问题。

---

## 5. 编译器流水线

```
① 采集     /Users/huangqingming/Workspace/typesafe-balatro/tools/harvest_corpus.py     (需要游戏)   →  /Users/huangqingming/Workspace/typesafe-balatro/corpus/cards.jsonl
② 编译     /Users/huangqingming/Workspace/typesafe-balatro/tools/compile_effects.py    (需要 LLM)   →  /Users/huangqingming/Workspace/typesafe-balatro/effects/compiled/*.json
③ 校验     /Users/huangqingming/Workspace/typesafe-balatro/tools/validate_effects.py   (纯离线)     →  退出码 + 覆盖度报告
④ 出审阅件 /Users/huangqingming/Workspace/typesafe-balatro/tools/render_review.py      (纯离线)     →  /Users/huangqingming/Workspace/typesafe-balatro/effects/review/*.md
⑤ 探针     /Users/huangqingming/Workspace/typesafe-balatro/tools/probe_effects.py      (需要游戏)   →  /Users/huangqingming/Workspace/typesafe-balatro/effects/probes/*.json
⑥ 运行时   scoring.load_effects()      (读表)       →  estimate + confidence
```

### ① 采集

输入：一个废弃对局 + `add` 扫描清单。输出：`/Users/huangqingming/Workspace/typesafe-balatro/corpus/cards.jsonl`（按 `key` 去重，追加式）。
不需要模型。可在 M0 之后立刻跑，先把手头能见到的牌录下来。

### ② 编译

```python
# 伪代码
for card in corpus:
    if cached(card.key, hash(card.effect_text)):   # 内容寻址，命中即跳过
        continue
    record = llm.compile(
        system=COMPILE_PROMPT,                      # 含完整 schema + 词表 + 反例
        user=card.effect_text,
        context={"key": card.key, "label": card.label, "set": card.set},
        response_format=EFFECT_SCHEMA,              # ★ 结构化输出，schema 违约结构上不可能
    )
    ok, errors = validate_record(record)
    if not ok:
        record = llm.compile(..., repair_feedback=errors)   # ★ 把校验错误回灌，最多 N 轮
    write(record)
```

两个关键点：

- **`response_format` 用 JSON Schema 约束解码**（OpenAI structured outputs 一类）。这跟 Jev "答案必落在 criteria 内"是同一个技巧：**把一类错误从"事后检查"变成"结构上不可能"**。
- **校验错误回灌重试**，最多 2–3 轮。这是 balatrollm 那个"把失败原因塞进下一轮 prompt"的模式，只不过发生在离线编译期而不是运行期——代价低得多，且结果被固化下来。

### ③ 校验（纯离线，进 CI，不需要 key）

必须通过的**不变量**：

- 所有 `site` / `kind` / `unit` / 谓词名都在封闭枚举内；
- `per_unit.step > 0`，`cap` 若存在则 ≥ 0；
- 数值有限且在合理范围（如 `|add_mult| ≤ 1e4`、`|multiply_mult| ≤ 100`）——**防止模型幻觉出 10^9 这种数**；
- `triggers` 非空 **或** `unmodeled` 非空（不允许两者皆空）；
- `unmodeled` 必须带 `reason` 与 `intended_layer`；
- 每条记录的 `effect_text_hash` 与语料一致；
- 无重复 `key`；每条都有 `compiled` 溯源信息。

**覆盖度报告**（每次 CI 打印，可设阈值告警）：

```
effects total         : 187
  modeled             : 141  (75.4%)
  unmodeled           :  46
by archetype          : copy 6, random 9, card_modifier 11, ...
by set                : JOKER 148/152, PLANET 12/12, TAROT 22/22, VOUCHER 9/32
stale (hash mismatch) : 0
```

### ④ 出审阅件

把 JSON 渲染成人类可读的 Markdown 表格（`/Users/huangqingming/Workspace/typesafe-balatro/effects/review/j_fibonacci.md`）：

```markdown
# Fibonacci  `j_fibonacci`
> Each played Ace, 2, 3, 5, or 8 gives +8 Mult when scored

| site | condition | applies | effect |
|---|---|---|---|
| card_scored | all(card_rank_in[A,2,3,5,8]) | per_match | add_mult +8 |
```

**目的**：让"模型写的东西"进入代码评审流程。JSON diff 没人看得懂，Markdown diff 一眼就能看出模型理解错了。这一步是整条流水线里最便宜也最有效的质量闸门。

### ⑤ 探针（下一节详述）

### ⑥ 运行时

```python
table = load_effects(EFFECTS_DIR)          # 进程启动时加载一次

def estimate(hand, play_cards, facts) -> Estimate:
    chips, mult = base_hand(hand_type, level)          # 来自 hands.<Name>.chips/mult
    unmodeled = []
    for card in scoring_cards:
        chips += card_chips(card); mult += card_mult(card)
        apply_card_edition_seal(card)                   # Foil/Holo/Poly, Red seal 重触发
    for joker in facts.jokers:                          # ★ 从左到右，顺序有意义
        rec = table.get(joker.key)
        if rec is None or rec.unmodeled:
            unmodeled.append(joker.key); continue
        for trig in rec.triggers:
            if matches(trig.condition, hand, facts):
                c, m, x = eval_effect(trig, hand, facts)
                chips, mult = apply(chips, mult, c, m, x)
    return Estimate(chips=chips, mult=mult, score=chips * mult,
                    confidence=confidence_from(unmodeled, table),
                    unmodeled_jokers=unmodeled)
```

---

## 6. 验证体系：三层，从便宜到昂贵

| 层 | 需要 | 判据 | 频率 |
| --- | --- | --- | --- |
| **L-A schema 校验** | 无 | 封闭枚举 / 不变量全过 | 每次 CI |
| **L-B 手算对照** | 无 | 人工构造的 30–50 个 `(局面, 期望分)` 用例 | 每次 CI |
| **L-C 差分探针** | 游戏 | 预测贡献 vs 实测贡献 | 手动 / 每晚 |

### L-B：手工对照用例

从最简开始，覆盖已知的计分规则组合：裸牌型、单张 enhancement、单张 edition、单张 seal、单 Joker。这些用例既是 `/Users/huangqingming/Workspace/typesafe-balatro/scoring.py` 的测试，**也是编译器 prompt 的 few-shot 素材**——同一份数据两用。这是最划算的一层：写一次，既测代码又教模型。

### L-C：差分探针（本设计的核心创新点）

**思路**：不去猜一张 Joker 值多少分，而是**量出它**。因为种子固定时发牌是确定的，所以可以做差分：

```
probe(joker_key):
  1. 固定 seed S、deck RED、stake WHITE，起一个全新进程
  2. 走到第一个 SELECTING_HAND，记下手牌 H
  3. 打出一手固定的基线牌 B（取确定性规则，如"点数最大的一张"）
     → actual_baseline   （chips_after − chips_before）
  ── 回到第 2 步的同一状态 ──
  4. add(key=joker_key) 注入这张牌，再打出同一手 B
     → actual_with
  5. predicted = scoring.predict(joker_key, hand=H, play=B)
  6. observed  = actual_with − actual_baseline
  7. 断言 |predicted − observed| ≤ tol
```

**为什么这个设计很漂亮**：

- **不需要控制手牌**。发牌由种子决定，两次跑拿到的是同一手牌，所以差分是干净的。
- **不需要打完整局**。一局只要走到第一次出牌（约 10 个动作），150 张 Joker ≈ 300 次短局。
- **基线效应自动抵消**。牌型基础分、牌面筹码、enhancement、edition、seal、held-in-hand 效果在两次跑里完全相同，差分里只剩下这张 Joker 的贡献——**正好是我们要验的那一项**。
- **它测的是真实游戏，不是我们的理解**。这把"效果表对不对"的裁决权交给了 Balatro 本体，彻底摆脱"人 review 上百张牌"。

**两种实现方式**（先试第一种）：

1. **save / load 回放**（省一半开销）：`save()` → 出牌 → 记录 → `load()` → `add(joker)` → 出同一手牌 → 记录。同一进程、同一发牌，差分最干净。
2. **两次全新进程**（回退方案）：`save`/`load` 的可靠性未验证，不行就退回两进程对比。

**必须遵守的前提**（否则探针本身不可信）：

- **每个探针一个全新进程**——上游 #232 的跨局泄漏会让第二次测量带上第一次的残留（可行性报告 §1.1、R11）。
- **探针不要开 `--fast`**（#234 会破坏种子保真，而探针完全依赖种子保真）。
- 探针结果写入 `/Users/huangqingming/Workspace/typesafe-balatro/effects/probes/<key>.json`，包含 `seed` / `game_version` / `table_version`，便于在游戏更新后重跑对比。

对于**给钱不给分**的 Joker（经济类），同一套差分改成比较 `money` 而不是 `chips`——探针脚本按 `classified_as` 自动选择观测量。

### 探针的验收标准（写进 M3 的完成定义）

- 探针通过率 ≥ 90% 的"已建模 Joker"（`|predicted − observed| ≤ tol`，`tol` 取 0 或一个很小的相对误差）；
- 未通过的按 `classified_as` 归类分析：是**表错**（改表）还是**求值器错**（改代码）——这个区分本身就很有价值；
- 覆盖率 ≥ 首个 N 局实际遇到的 Joker 的 95%。

---

## 7. 校准闭环如何回归效果表

探针是**主动**验证（构造局面去问游戏），校准闭环是**被动**验证（从真实对局里收集残差）。两者互补。

**被动侧**：`decisions.jsonl` 里每条决策都记录 `(jokers_in_play, hand_type, est_chips, actual_chips)`。分析脚本按 `joker` 分组统计残差分布：

```jsonc
// /Users/huangqingming/Workspace/typesafe-balatro/reports/calibration_2026-09-21.json
{
  "table_version": "2026-09-21.1",
  "overall": { "n": 4128, "median_rel_err": 0.04, "p90_rel_err": 0.19 },
  "by_joker": [
    { "key": "j_fibonacci", "n": 143, "median_rel_err": 0.01, "flag": null },
    { "key": "j_misprint",  "n":  97, "median_rel_err": 0.31, "flag": "stochastic_expected" },
    { "key": "j_hiker",     "n":  22, "median_rel_err": 0.48, "flag": "review" }
  ]
}
```

**关键性质**：被动残差有**归因混淆**（多张 Joker 同时在场的误差分不开）。所以分工是：

- **主动探针**负责"单张牌对不对"（可归因，但需要构造局面）；
- **被动残差**负责"真实局面里整体准不准"（不可归因，但覆盖面广，且能抓到交互效应）。

一旦被动侧发现某张牌的残差异常，**自动为它排一个主动探针**去定位。这形成一个闭环：

```
被动残差发现异常  →  自动触发主动探针  →  定位是表错还是求值器错
        ↑                                            │
        └────────── 修表/修代码后重跑 ────────────────┘
```

这条闭环就是可行性报告 §4.1 第 6 条"校准算术"的完整形态，也是它值得被单列为一个模块的原因。

---

## 8. `estimate_confidence` 与"未建模清单"如何从表派生

**不做猜测，直接由表推导：**

```python
def confidence_from(unmodeled: list[str], table, hand_type) -> str:
    if unmodeled:
        return "low"                       # 有牌没建模 → 数字不可信
    if table.any_stochastic_in_play():     # 期望值建模，但单次结果会抖
        return "medium"
    if hand_type in table.hand_type_modifiers_in_play():
        return "medium"                    # 成牌判定被改过，边界风险高
    return "high"
```

并且在事实 JSON 里**显式列出是哪几张牌没建模**：

```jsonc
"estimate": {
  "chips": 1320, "mult": 24, "score": 31680,
  "confidence": "low",
  "unmodeled_jokers": ["j_blueprint", "j_brainstorm"]
}
```

这比一个笼统的 `low` 有用得多——**Jev 能看到"不确定来自蓝图的复制对象"，于是可以据此选择保守方案**。这是"代码给事实、模型做取舍"这条界线在估算器内部的具体落实，也是可行性报告 §4.2 里 `estimate_confidence` 的完整形态。

---

## 9. 版本化与实验卫生

三样东西必须进 `decisions.jsonl`，否则不同表版本的数据会被混在一起分析：

| 字段 | 来源 | 作用 |
| --- | --- | --- |
| `table_version` | `/Users/huangqingming/Workspace/typesafe-balatro/effects/compiled/_index.json` | 效果表版本 |
| `game_version` | 语料 | 游戏本体版本 |
| `estimator_version` | 代码常量 | 求值器实现版本 |

`_index.json` 内容：

```jsonc
{
  "table_version": "2026-09-21.1",
  "schema_version": 1,
  "corpus_hash": "sha256:…",
  "counts": { "total": 187, "modeled": 141, "unmodeled": 46, "stale": 0 },
  "probe_summary": { "passed": 128, "failed": 13, "not_probed": 46 }
}
```

**规则**：`table_version` 变化后，之前的实验数据**不能**与新数据合并统计；要么固定版本跑完一批，要么在新版本上重跑基准。

---

## 10. 已知局限与逃生舱

| 局限 | 处理 |
| --- | --- |
| **复制类 Joker**（Blueprint / Brainstorm）依赖 Joker 排列顺序 | 显式 `unmodeled`，`intended_layer: L3`。求值器已按左→右顺序遍历，L3 时可插入 |
| **随机类**（Misprint / Lucky Card / 8 Ball）单次取值不确定 | 建期望值 + `stochastic: true` → 降级为 `medium` 置信度 |
| **永久改牌类**（Hiker 等）改变后续所有计分 | L3；先 `unmodeled` |
| **Joker 之间的交互**（如"每有一个空 Joker 槽"） | 条件词汇表里有 `empty_joker_slots`；更复杂的交互落 L3 |
| **游戏更新导致文案变化** | 内容寻址：文本哈希不匹配 → 记录标 `stale` → CI 告警 + 触发重编，产出一份可审计的 diff |
| **模型把一切都标 `unmodeled`** | 覆盖率是 CI 指标（§5 ③），且 `unmodeled` 必须写 reason 并进入人工审阅件。**覆盖率下降会在 PR 里可见** |
| **探针太慢跑不动全量** | 探针是"手动/每晚"层，不是 CI 门禁；CI 只跑 L-A / L-B 离线层 |

---

## 11. 工作量与里程碑

| 步骤 | 交付物 | 依赖 | 估时 |
| --- | --- | --- | --- |
| 契约 + 校验器 | `/Users/huangqingming/Workspace/typesafe-balatro/effects/schema.py` + `/Users/huangqingming/Workspace/typesafe-balatro/tools/validate_effects.py` + CI | 无（纯离线） | 0.5 天 |
| 采集器 | `/Users/huangqingming/Workspace/typesafe-balatro/tools/harvest_corpus.py` | 游戏（M0） | 0.5 天 |
| 编译器 | `/Users/huangqingming/Workspace/typesafe-balatro/tools/compile_effects.py` + prompt + 重试回灌 | LLM key | 1 天 |
| 审阅件渲染 | `/Users/huangqingming/Workspace/typesafe-balatro/tools/render_review.py` | 无 | 0.5 天 |
| 求值器接口 | `scoring.load_effects()` + `estimate()` | 无 | 1 天（与 M3 合并） |
| 手工对照用例 | `/Users/huangqingming/Workspace/typesafe-balatro/tests/fixtures/scoring_cases.json` | 无 | 0.5 天 |
| 差分探针 | `/Users/huangqingming/Workspace/typesafe-balatro/tools/probe_effects.py` | 游戏 | 1–1.5 天 |
| 校准回归脚本 | `/Users/huangqingming/Workspace/typesafe-balatro/tools/calibrate.py` | `decisions.jsonl` 有数据 | 0.5 天 |

**合计约 5.5–6 个工作日**，其中约 3.5 天不依赖游戏、不依赖 API key，可以和 M1 并行开工。

> **注意**：这套东西**不是**"额外负担"，它是可行性报告 §4.2 里 L2 层的实现方式。手工录 20–40 张 Joker 的表同样要花 1–2 天，而且会错、会过时、无法回归。差别在于：手工表错一次你不知道；编译表错一次探针会告诉你。

---

## 附：一句话总结

> **让模型把散文编译成受约束的数据，让游戏本体裁决数据对不对，让代码拿着数据去做确定性的算术。** 编译器负责"读懂"，探针负责"证伪"，校准闭环负责"长期免疫"，而 Jev 只在最后一步做它真正擅长的事——在语义不同的方案之间做取舍。
