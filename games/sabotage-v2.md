# Sabotage v2 —— 主题绑定 · 多维向量 · 共享预算

> **状态：已实现（40 张卡 + $100,000 共享预算，2026-10-30 更新）。**
> 前端与后端都已切到 v2；[sabotage.md](./sabotage.md) 描述的 v1 仅作历史存档。
>
> 实现落在 `arcade_ftw_be/app/sabotage_config.py`（全部数据）与 `app/sabotage.py`（状态机），
> 前端在 `arcade_ftw_fe/src/games/sabotage/`。设计稿（含每轮预算推演）是
> [sabotage_cards.xlsx](./sabotage_cards.xlsx)，**改数值先改这张表，再同步 `sabotage_config.py`**。
>
> `tests/test_sabotage.py` 守住四条平衡不变量（见第 5 节），改动任何价格/向量后必须重新跑那组测试。

---

## 1. 一句话规则

> 四轮，每队一张 **$100,000** 的卡。每轮有一个**目标维度**，只比那一列；但钱是通着用的——
> 这一轮花痛快，第四轮就没得打。

---

## 2. 已落地的流程（引擎总时长 243s，flow 节点占位 250s）

| # | 阶段 | 时长 | 说明 |
|---|---|---|---|
| 1 | `team_reveal` | 10s | **抽队伍在这里**：玩家看到自己是红还是蓝，以及本轮攻/守 |
| 2 | `brief` | 15s | 规则卡：共享预算、三维、单目标、最多 4 张 |
| 3–10 | `select` / `reveal` ×4 | 35s + 5s，然后 25s + 5s ×3 | R1 给 35 秒（同时是教学局 + 第一次规划预算），其余 25 秒。揭晓只有 5 秒，是全场最紧的一屏 |
| 11 | `final` | 5s | 终局分数 + 四轮复盘 + 双方剩余预算。整局 160s，flow 槽位 163s |

四轮目标维度：Data → People → Infra → **Data（知识库投毒）**；攻守交替，两队各攻两轮，
这是唯一能避免「多攻一轮的队伍靠结构优势获胜」的排法。

---

## 3. 三个维度

每张卡给一个三维向量，强度 0–3：

```
Data（数据 / 信息） · People（人员 / 意识） · Infra（基础设施 / 系统）
```

- **目标维度**决定本轮谁赢：`攻 > 守` 才算击穿，**平局算守方守住**。
- 另外两列决定「赢得有多难看」：每多领先一列，击穿等级 +1。
- 牌库每轮只有 5 张，最多买 4 张——**必须放弃一张**，这是取舍的第一层。

---

## 4. 卡牌（每轮 5 攻 + 5 守，共 40 张）

> 格式：`卡名（价格, Data, People, Infra）`。**加粗** = 本轮目标维度上的最强格。
>
> 卡名与描述都刻意用**非技术听众能一眼看懂**的词，且都是 Title Case。

### R1 · Phishing email（目标 **Data**）· RED 攻 / BLUE 守

- 攻方剧本：**Get a credential out of the organisation.**
- 守方剧本：**Keep the credential inside the organisation.**

**攻击卡**

| 卡 | 价格 | D | P | I |
|---|---|---|---|---|
| Fake Sign-In Page | $8,000 | **3** | 0 | 1 |
| Malicious Attachment | $13,000 | **3** | 1 | 1 |
| Personal Details | $4,000 | **2** | 2 | 0 |
| Fake Sender Name | $2,000 | **2** | 1 | 1 |
| Urgent Deadline | $1,000 | **1** | 1 | 0 |

**防守卡**

| 卡 | 价格 | D | P | I |
|---|---|---|---|---|
| Block Spam Emails | $13,000 | **3** | 0 | 2 |
| Verify the Sender | $6,000 | **3** | 0 | 1 |
| Scan the Link | $5,000 | **2** | 0 | 0 |
| Security Training | $3,000 | 1 | 2 | 0 |
| Report Phishing | $2,000 | **1** | 1 | 1 |

### R2 · CEO impersonation（目标 **People**）· BLUE 攻 / RED 守

- 攻方剧本：**Get a person to authorise something they should not.**
- 守方剧本：**Make authorisation survive a phone call.**

**攻击卡**

| 卡 | 价格 | D | P | I |
|---|---|---|---|---|
| Deepfake Video Call | $14,000 | 1 | **3** | 0 |
| Urgent Wire Transfer | $6,000 | 1 | **3** | 0 |
| Lookalike Website | $2,000 | 2 | 0 | 1 |
| Do Not Escalate | $4,000 | 0 | **2** | 0 |
| Vishing Call | $3,000 | 1 | **2** | 0 |

**防守卡**

| 卡 | 价格 | D | P | I |
|---|---|---|---|---|
| Callback Verification | $9,000 | 0 | **3** | 0 |
| Payment Approval | $7,000 | 1 | **2** | 0 |
| Identity Verification | $6,000 | 0 | **2** | 0 |
| Manager Escalation | $5,000 | 0 | **2** | 1 |
| Hidden Address Check | $2,000 | 1 | 0 | 0 |

### R3 · Manipulating the AI assistant（目标 **Infra**）· RED 攻 / BLUE 守

- 攻方剧本：**Make the assistant do what its instructions forbid.**
- 守方剧本：**Keep the assistant inside its own rules.**

**攻击卡**

| 卡 | 价格 | D | P | I |
|---|---|---|---|---|
| Malicious AI Script | $9,000 | 1 | 0 | **3** |
| Hidden AI Script | $8,000 | 1 | 1 | **3** |
| Jailbreak the Assistant | $6,000 | 0 | 2 | **2** |
| Exfiltrate Data | $4,000 | 2 | 0 | **2** |
| False Rule | $2,500 | 1 | 1 | **1** |

**防守卡**

| 卡 | 价格 | D | P | I |
|---|---|---|---|---|
| Read-Only Access | $11,000 | 1 | 0 | **3** |
| Human in the Loop | $8,000 | 1 | 1 | **2** |
| Scan the File First | $6,000 | 2 | 0 | **3** |
| Assistant Restrictions | $2,000 | 1 | 0 | **1** |
| Human Approval | $1,500 | 1 | 1 | 0 |

### R4 · Poisoning the AI's sources（目标 **Data**）· BLUE 攻 / RED 守

- 攻方剧本：**Make the assistant tell a customer something false.**
- 守方剧本：**Prove every answer came from a trusted source.**

**攻击卡**

| 卡 | 价格 | D | P | I |
|---|---|---|---|---|
| Poison the Data Source | $14,000 | **3** | 1 | 0 |
| Malicious Insider Edit | $7,000 | **3** | 2 | 0 |
| Unchecked Auto-Sync | $4,500 | 0 | 0 | 2 |
| Outdated File Upload | $4,000 | **2** | 0 | 0 |
| Lookalike Website | $2,000 | **2** | 0 | 1 |

**防守卡**

| 卡 | 价格 | D | P | I |
|---|---|---|---|---|
| Verify the Source | $10,000 | **3** | 0 | 1 |
| Approved Sources Only | $7,000 | **3** | 0 | 2 |
| Human Approval | $5,000 | **2** | 2 | 0 |
| Approved Source List | $5,000 | 1 | 0 | 2 |
| Source Required | $1,500 | 0 | 0 | 1 |

> 两处刻意的重复：`Lookalike Website`（$2,000）在 R2 与 R4 是**同一张卡**，代码里共用一个 id；
> `Human Approval` 在 R3（$1,500）与 R4（$5,000）**向量不同**，是两个不同的 id。
>
> `Unchecked Auto-Sync` 是全场唯一目标格为 0 的卡（0,0,2）：它是纯粹的「侧翼陷阱」，
> 买它等于放弃本轮——留着是有意的诱饵。

---

## 5. 预算经济性（这一版的核心）

一队总共 **$100,000**，四轮通着花。每轮「打满目标格」的最省手牌如下：

| 轮 | 主题 | 目标 | 攻守 | 攻上限 | 守上限 | 攻方最佳手牌 | 守方最佳手牌 | 最贵四张（攻/守） |
|---|---|---|---|---|---|---|---|---|
| R1 | Phishing email | Data | RED 攻 / BLUE 守 | 10 | 9 | $27,000 | $26,000 | $27,000 / $27,000 |
| R2 | CEO impersonation | People | BLUE 攻 / RED 守 | 10 | 9 | $27,000 | $27,000 | $27,000 / $27,000 |
| R3 | Manipulating the AI assistant | Infra | RED 攻 / BLUE 守 | 10 | 9 | $27,000 | $27,000 | $27,000 / $27,000 |
| R4 | Poisoning the AI's sources | Data | BLUE 攻 / RED 守 | 10 | 9 | $27,000 | $27,000 | $29,500 / $27,000 |

推演：

| 项 | 金额 |
|---|---|
| 每队预算（四轮共用） | **$100,000** |
| RED 每轮都打满的理想花费 | $108,000 |
| BLUE 每轮都打满的理想花费 | $107,000 |
| 前三轮打满已花 | $81,000 / $80,000 |
| 留给定决胜轮 | **$19,000 / $20,000** |
| 决胜轮打满需要 | $27,000 |
| **缺口** | **$8,000** |

也就是说：**四轮都打满在数学上不可能**。队伍必须提前决定「放弃哪一轮」或「哪一轮打七折」。
每轮最贵四张都落在 $25,000–$30,000（单轮最贵 $29,500），所以一轮不会吞掉整个预算。

**价格不是强度排名**：每副牌都存在「更贵但目标格不更强」的一对（例如 R1 攻的
`Malicious Attachment` $13,000 与 `Fake Sign-In Page` $8,000 都是 Data 3），
所以「买最贵的」不是策略。三张刻意的「便宜且致命」卡：
`Fake Sign-In Page` $8,000 (Data 3)、`Hidden AI Script` $8,000 (Infra 3)、
`Outdated File Upload` $4,000 (Data 2)。

---

## 6. 判定、计分与平衡不变量

### 判定

```
攻方组合向量 = Σ(所选攻击卡向量)
守方组合向量 = Σ(所选防守卡向量)

目标维度：攻 > 守  → 击穿（攻方赢本局）
          攻 ≤ 守  → 守住（守方赢本局，平局也算守住）
侧翼维度：逐维同规则比较，统计攻方领先的列数 flanks（0~2）
```

### 计分表（每轮恒定 **200** 分池，击穿永远比守住拿得多）

| 击穿 | flanks | 结果名 | 攻方 | 守方 |
|---|---|---|---|---|
| 否 | 0 | Full defence | 0 | **200** |
| 否 | 1 | Flank leak | 120 | 480 |
| 否 | 2 | Severe flank leak | 240 | 360 |
| 是 | 0 | Narrow breach | 320 | 280 |
| 是 | 1 | Effective breach | 420 | 180 |
| 是 | 2 | Total compromise | **200** | 0 |

**没买成牌就是没攻破，所以弃权一律算守方赢**：只有一边空手、或两边都空手，守方都拿满 200。早先「双方都空手 = 0–0 平局」的规则已经取消——那等于允许一支队伍把不想打的一轮停掉。

### 测试守住的四条不变量（`tests/test_sabotage.py`）

1. **攻守差恰好 1**：无限预算下，攻方目标上限 = 守方目标上限 + 1（10 vs 9）。差 1 是整个设计：
   多买一张干扰卡就打不穿，防守方买一张诱饵就能守住。
2. **单轮 25k–30k**：每副牌「打到上限最省的那套」价格必须落在 $25,000–$30,000。
3. **四轮打满超预算**：理想花费（RED $108,000 / BLUE $107,000）大于 $100,000，钱必须真实约束。
4. **价格倒挂存在**：每副牌必须有一对「更贵但目标格不更强」的卡，否则「买最贵的」成为最优解。

### 投票聚合（`aggregate`）

票数降序 → 同票按**本轮目标格强度**降序（**不按价格**，否则会奖励投贵卡）→
在**队伍剩余预算**内能装就装，装不下跳过并记进 `dropped`（`reason: "budget" | "slots"`）→
无人投票就是空手。

预算按**队伍**算：`remaining = $100,000 − Σ(已结束回合的实际花费)`，
`spent` 从 `history` 里回读而不是维护计数器，避免与复盘页对不上。

### 其余已落地的规则

- **不投票就弃权**：某队一张卡都没投（或钱已经不够买任何卡），本轮空手，直接输掉。
  这条来自第一次试玩：原本给静默方自动配最优手牌，结果不动手的一方赢了认真投票的一方。
- **暂停时跳到本节点也会开新局**，并冻结在抽队伍那一屏。
- **选卡期间 build 不广播**：广播是公开的，实时手牌等于把对手的答案递过去。
  每台设备用 `GET /api/sabotage/{code}/team` 取自己那一侧（含 `round_budget` / `budget` / `spent`）。
  钱是公开的：`GET /api/sabotage/{code}/state` 里的 `budget` 所有人都能看。

---

## 7. 界面上的钱

| 位置 | 显示 |
|---|---|
| 平板选卡页·上半区左栏 | 场景卡：`ROUND 1 OF 4 · RED ATTACKS · TARGET · DATA · YOU ARE ATTACKER` + 主题名（大白字）+ 一句话剧本 |
| 平板选卡页·上半区右栏 | 实时开销卡：`LIVE SPEND $91,000 LEFT FOR ALL FOUR ROUNDS` + `Earlier rounds $0 / This round $9,000` + 三维进度条 + `2 cards of 4 · 3/4 voted` |
| 上半区（竖屏）| 左右两栏改为上下堆叠 |
| 平板卡牌 | 上部 icon（统一 74px 高，保留原画配色）/ 中部卡名 + 描述 / 下部 Data·People·Infra 点数 + 价格 |
| 卡牌 icon | **全部 39 张卡的图都已接入**（四轮攻守各 5 张，共 39 个 id / 39 张图：`Lookalike Website` 两轮共用一张）；放在 `arcade_ftw_fe/public/sabotage_icons/<round>/<round><side>_<slug>.PNG`，在 `src/games/sabotage/cardIcons.ts` 里按 card id 登记；**一整副牌全部有图才显示图**（`deckHasArt`），半套图会被当成画面故障；现在八副牌都齐，规则留着给以后新加的轮次 |
| 平板结算页 | `RED spent $27,000 · BLUE spent $26,000` 与 `Left to spend — RED $73,000 · BLUE $74,000` |
| 平板终局页 | `Red finished with $19,000 · Blue with $20,000` |
| 大屏选卡 | `Left to spend — RED $100,000 · BLUE $100,000` |
| 大屏结算 | 本轮花费 + 双方剩余 |

金额一律格式化成 `$14,000`（`toLocaleString`），不出现裸数字。

### 角色配色（只有 Sabotage 这样）

整屏颜色 = **你这一轮的职责**，不是阵营：

| 谁 | `--c-brand` | `--c-brand-deep` | 效果 |
|---|---|---|---|
| 攻方看 | `#e21ce2` 品红紫 | `#820c82` | 边框、角标、标题页签、进度条、目标徽章、选中卡全部变紫 |
| 守方看 | `#16c6e2` 青蓝 | `#0a5566` | 同上，全部变青蓝（取自守方卡图，同色系）|
| 旁观 / 其他游戏 | `#db0011` 红 | `#af000e` | 品牌红，完全不变 |

实现方式是在 `.sab.is-attacker` / `.sab.is-defender` 里**重绑 `--c-brand` / `--c-brand-deep`**，
所以 `hud.css` 的边框、页签、箭头、光晕一次全部跟着变；为此把 `hud.css` 里 41 处手写的
`rgba(219,0,17,·)` / `rgba(175,0,14,·)` 换成了 `color-mix(in srgb, var(--c-brand) …, transparent)`，
在根节点数值完全等价，其他页面视觉零变化。

只有「阵营」相关的两处（开局抽队伍的队名、终局比分）继续用固定的红/浅色，走 `--sab-team-red`。

---

## 8. 开放问题

1. `Approved Sources Only` 与 `Approved Source List` 同在一副牌里，名字过于相似（内容层，暂缓）。
2. `Vishing Call` / `Exfiltrate Data` 仍是行话，非技术听众可能不熟（内容层，暂缓）。
3. R1 攻 `Fake Sign-In Page` $8,000 拿 Data 3，性价比过于集中（数据层，暂缓）。
4. 决胜轮缺口 $8,000 是否足够痛——要等真实试玩；想更紧就调低 `TOTAL_BUDGET`，
   想更松就调高（一个常量在 `sabotage_config.py`）。
5. 直播时是否公布对手的剩余预算（现在公布，双方信息对称）。

---

## 9. 与 v1 的差异对照

| 项 | v1 | v2（现行） |
|---|---|---|
| 攻击卡 | 6 张共享、单次使用 | 20 张，每轮独立 5 张 |
| 防守卡 | 6 张每轮复用 | 20 张，每轮独立 5 张 |
| 判定 | 6×6 手工矩阵三档（blocked/partial/failed） | 3 维向量逐维比较 + 目标维度胜负 |
| 计分 | 单卡 success/fail 分值累计 | 击穿 × 侧翼 6 档，恒定 600 分池 |
| 预算 | $100k，每轮重置，永远花不完 | **$100,000 四轮共用**，钱是真实的约束 |
| 卡价 | 10–45 | **$1,000–$14,000**（与真实投入挂钩） |
| 剧本 | 纯文案，不影响玩法 | 决定目标维度，直接影响打法 |
| 投票 | 单选 1 张 | 组合 1–4 张，预算内 |

---

*改动本文档时同步 [SYSTEM_HANDBOOK.md](../SYSTEM_HANDBOOK.md) 的进度条目与
[sabotage_cards.xlsx](./sabotage_cards.xlsx) 的设计表。*
