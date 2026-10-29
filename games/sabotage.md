# Sabotage（AI Cyber Sabotage）

> 📌 **本文档描述的是现行代码（v1）**：6 张共享攻击卡 + 6×6 效果矩阵 + 单选投票。
> 下一版重构的设计草案见 [sabotage-v2.md](./sabotage-v2.md)（18 卡主题绑定 + 三维向量 + 主维度判定，**尚未实现**）。

主流程第 8 个节点（`type: game`，`game: sabotage`，index 7，前面有 4 个游戏节点 + 3 个榜单节点）。
设计来源：`design/AI_Cyber_Sabotage_Developer_Storyboard_High_Resolution.pdf`（17 页）。

> 状态：**已完整实现（M0–M5）**——设计、团队分配、引擎/投票、流程接入、前端界面与结算写库全部就绪。
>
> 模块进度：M0 ✅ · M1 ✅ · M2 ✅ · M3 ✅ · M4 ✅ · M5 ✅

## 一句话

红蓝对抗的三轮攻防投票游戏：每轮全队用平板**私下投票**，多数票成为**全队唯一行动**；攻击卡是**两队共享、用过即废**的 6 张军火库，防守卡是**每轮可复用**的 6 张应对牌；团队胜负折算成个人分后进入 Ultimate Showdown。

## 玩法

| 规则 | 说明 |
|---|---|
| 参与资格 | 只有开始时状态为 `COMPETING` 的玩家分红蓝；**Learning Mode 全程只观察**，不投票、不计分、平板锁定 |
| 分队 | 竞技玩家尽量均分（20 人 → 10 红 / 10 蓝）；**人数组为奇数时红队多 1 人**（7 人 → 红 4 / 蓝 3） |
| 单人房 | **无法分队，整局跳过**：主流程直接越过该节点，该玩家拿 `SOLO_SCORE = 2700`（完美获胜分）。0 人房保持原行为（不启动、节点照常走完） |
| 投票 | 每人一台平板私下选 1 张卡；**截止前可改**，以最后一次为准 |
| 团队行动 | 票数最高的卡成为**唯一行动**，费用**只扣一次** |
| 实时票数 | 投票期间**完全隐藏**，截止才公布 |
| 平票 | 只对平票卡**重投 5 秒**；仍平则选**费用更低**的那张 |
| 预算 | 红蓝各 **$100k**，攻击与防守都从本队预算扣 |
| 攻击卡 | **双方共享、单次使用**；打过的卡对两队永久 `SPENT`，但仍显示并标注「被谁用过」 |
| 防守卡 | **每轮全部恢复**，不消耗，只受预算限制 |
| 买不起 | 仍显示但禁用，标 `NOT ENOUGH BUDGET`；`SPENT` 优先级高于预算判断 |

## 卡牌

**攻击（共享，单次）**

| 卡 | 费用 | 成功 | 失败 |
|---|---|---|---|
| AI Agent Attack | $35k | +1000 | −400 |
| Deepfake Video | $30k | +900 | −350 |
| Cloned Voice | $25k | +750 | −300 |
| Personalised Message | $20k | +650 | −250 |
| Fake Digital Identity | $20k | +600 | −250 |
| AI-Written Message | $10k | +300 | −100 |

**防守（每轮可复用）**

| 卡 | 费用 | 成功 | 失败 |
|---|---|---|---|
| Stop the Action | $20k | +750 | −150 |
| Verify Independently | $18k | +700 | −150 |
| Report & Escalate | $15k | +600 | −175 |
| Investigate Further | $8k | +300 | −275 |
| Ask Someone Else | $5k | +150 | −350 |
| Continue Carefully | $0 | +50 | −500 |

> 成败值取设计 PDF 第 5 页；「成功 / 失败」如何配对攻击判定，依赖**效果矩阵**（见下「可配置项」）。

## 流程

```
分队揭晓 8s
  → R1 红攻（钓鱼邮件）× 蓝守 → 结算
  → R2 蓝攻（CEO 冒充）× 红守 → 结算
  → R3 红攻（AI 助手操纵）× 蓝守 → 结算
  → 团队结果 → 个人分发放 → 实时排行榜 → Ultimate Showdown（3 题）
```

> **队伍怎么到玩家手上**：分队在 `auto_start_sabotage` 里**先写 `players.team` 再广播 reveal**；玩家设备进入 sabotage 节点后**轮询 `GET /api/auth/me`**（约 1 秒一次）读到自己的队。队名是个人隐私，因此**不放进房间级 WS 快照**。注意两条配套约束：`join` **只在换房间时清队**（同房间 rejoin 不清，否则会抹掉刚分的队）；前端 `fetchSession` 必须**带上 `team` 字段**。

每轮三段，每段投票 **20 秒**：

1. **ATTACK**：进攻方在未 `SPENT` 且买得起的攻击卡里选（R1 6 张 → R2 5 张 → R3 4 张）
2. **DEFEND**：防守方 6 张防守卡全可选，按预算禁用
3. **RESOLVE**：公布「攻击 vs 防守」配对，并按身份结算——**玩家平板显示自己阵营的结果**（攻成/攻败、守成/守破），**大屏显示本轮赢家**（`RED/BLUE WINS THE ROUND`），加减分与预算照旧

三轮剧本：钓鱼邮件 → CEO 冒充 → AI 助手操纵；正常打完**正好用掉 3 张攻击卡，剩 3 张未用**。

## 计分（团队共享一个分数）

> 与其余三个分区不同：sabotage **不是逐题累加**，而是整局结算出**一个团队总分**，队内每个竞技成员**共享同一个数值**写入 `scores.sabotage_score`。

### 效果矩阵（attack × defence，已定稿）

每格三档：✅ **blocked** / 🟡 **partial** / ❌ **failed**。

| 攻击 \ 防守 | Stop the Action | Verify Independently | Report & Escalate | Investigate Further | Ask Someone Else | Continue Carefully |
|---|---|---|---|---|---|---|
| Deepfake Video | 🟡 | ✅ | 🟡 | ✅ | ❌ | ❌ |
| Cloned Voice | 🟡 | ✅ | 🟡 | 🟡 | 🟡 | ❌ |
| Personalised Message | ❌ | 🟡 | 🟡 | ✅ | ❌ | ❌ |
| AI-Written Message | 🟡 | ❌ | ✅ | 🟡 | 🟡 | ❌ |
| Fake Digital Identity | 🟡 | ✅ | ❌ | ✅ | ❌ | ❌ |
| AI Agent Attack | ✅ | 🟡 | 🟡 | ❌ | ❌ | ❌ |

每张攻击都有至少一个 ✅ 最优解；`Continue Carefully` 全 ❌（呼应它 +50 / −500 的高风险低收益定位）。

### 每轮判定与得分

- ✅ **blocked**：防守得该防守卡的 **success** 分，攻击得 0
- ❌ **failed**：攻击得该攻击卡的 **success** 分，防守得该防守卡的 **fail** 分
- 🟡 **partial**：双方都得 0（部分止损，不引入小数）

### 团队总分与 payout

- 团队总分 = 三轮 attack/defence 判定得分累计之和
- 三轮结束：**胜利方 +100**，失败方 +0
- 队内每个竞技成员 `sabotage_score` = 本队最终总分（共享同一数值）
- 服务端在 `T4_RESOLVE → PAYOUT` 时统一写库（对每个队员 `UPDATE` 同一值），玩家不上报

### 单人房（跳过整局）

Sabotage 必须有**两队**，`start()` 里两个队都不能为空。所以房间里只有 **1 个玩家**时：

1. 该玩家 `sabotage_score = SOLO_SCORE`（= **2700**，`app/sabotage_config.py` 里的常量）
2. 不创建 sabotage 引擎，**主流程直接 `go_to(index + 1)` 越过该节点**，不让所有人干等 170 秒

`SOLO_SCORE` 的来历：把 6×6 矩阵、共享攻击牌、$100k 预算一起穷举（25,920 条合法路径），红队可达最高 **2600**，加上获胜奖金 **100** = **2700**，即「完美获胜」。**改动卡牌 / 费用 / 矩阵 / 预算后必须重新推导这个数**。

写库是**赋值不是累加**（`score.sabotage_score = SOLO_SCORE`），所以广播监听器在节点上被反复调用也不会叠加。

### Learning（暂不做）

本期决定：**先不做 learning，默认全员 competing**。不引入 `players.mode` 字段；将来要支持时再加 `mode` 列 + 迁移，并让 `assign_teams` 只分 `competing` 玩家。

## 双向通信评估

现状：WS 是**单向推送**（`ws.py` 收到文本直接丢弃，客户端不发言）；玩家身份走 HMAC Bearer token（`require_player`）；刚上线的 `POST /api/me/score` 已验证「玩家令牌 + HTTP 上报」这条链路。

Sabotage 需要玩家提交投票。两个方案：

| | A：HTTP 投票（推荐） | B：WS 双向 |
|---|---|---|
| 鉴权 | Bearer token 放 header，复用 `require_player` | WS 握手不能带 header，token 只能进 URL（会泄漏）或连接后再发消息 |
| 幂等 | 重复投同一卡 = 覆盖，天然幂等 | 要自己定消息协议与重连语义 |
| 基础设施 | CloudFront `/api/*` 已覆盖，零新增 | 无需新增，但 ws.py 要大改 |
| 服务端推送 | 截止与结果仍走现有 WS `state` 广播 | 同一通道，更顺 |
| 工作量 | **小**（复用分数上报模式） | **中到大** |

**结论：走 A。** 投票用 HTTP（带玩家 token），截止、票数与结果仍由引擎经 WS `state` 广播——WS 保持单向，不引入「token 进 URL」或新的消息协议。

### M2 已实现：引擎与接口

- **状态机**：`setup → team_reveal → (attack → defend → resolve) ×3 → team_result → payout → done`，单调时钟自推进（与主引擎同款，暂停语义一致）。
- **投票**：只当前行动方可投；截止前最后一次覆盖；多数票=唯一行动；平票进 5s 重投（只收平票卡），再平选**最便宜**；无人投票=全部候选平票。
- **SPENT / 预算**：攻击卡选中即全局 `spent`、扣费一次；防守卡只扣费不消耗；买不起的卡拒绝。
- **接口**：`POST /api/sabotage/{code}/start`（admin）、`POST /api/sabotage/{code}/vote`（玩家，body `{card}`）、`GET /api/sabotage/{code}/state`（公开快照，不暴露个人投票）。

### 真正的工作量在「节点内子阶段」

比双向通信更难的是：主引擎「1 节点 = 1 时长」，而 Sabotage 一个节点内部要跑 reveal 8s + 6×20s 投票 + 3 次结算 + 团队结果 ≈ 2.5 分钟、约 12 个子阶段。

**最终方案**：不动主引擎、也不拆 flow 节点——房间级 `SabotageEngine` 持有团队态并跑**自己的单调时钟**，主 flow 的 sabotage 节点只占 170s 的槽位等它打完。比「拆节点 / 给主引擎加 substage」更省回归风险。

### 问题清单

1. **团队态挂哪**：`shared_attack_spent`、budgets、votes 是**跨节点存续**的共享态，必须挂在房间级（类似 `RoomRegistry` 里的 engine），不能挂前端、也不能挂单个 flow 节点。
2. **计时与暂停**：6 段投票都要跟随引擎暂停（复用 `useNodeElapsed` 的 `endsAt / heldMs` 推导，已踩过坑）。
3. **并发投票**：单进程 asyncio，内存态 dict 天然无竞争；截止时聚合即可。
4. **改票语义**：截止前最后一次覆盖，服务端只存 `player_id → card_id`，不存历史。
5. **断线重连**：投票态在服务端，WS 重连只丢画面不丢票；HTTP 投票重复提交是覆盖而非累计。
6. **Learning 锁定**：本期不做 learning（全员 competing），见「计分 → Learning」。

## 需要新建什么

| 层 | 项 | 状态 | 说明 |
|---|---|---|---|
| 数据 | `players.team` + 迁移 | ✅ M1 | 值 `red` / `blue`；join 清空、assign 时写入 |
| 后端 | 团队分配 `POST /api/rooms/{code}/teams/assign` | ✅ M1 | admin 触发，按 id 交替均分、可重跑、幂等 |
| 后端 | `app/sabotage.py` | ✅ M2 | 房间级 SabotageEngine：budgets、shared_attack_spent、votes、团队分、状态机（team_reveal → attack → defend → resolve ×3 → team_result → payout → done），单调时钟自推进 |
| 后端 | 投票接口 | ✅ M2 | `POST /api/sabotage/{room}/vote`（玩家令牌，body: card）；`/start`（admin）；`/state`（公开）|
| 后端 | 结算写分 | ✅ M5 | 引擎到 `done` 时把每个队员的 `sabotage_score` 写成各自团队总分（服务端结算，玩家不上报）|
| 前端 | `games/sabotage/` | ✅ M4 | 投票屏（选卡、已选高亮、倒计时）、团队揭晓屏、结果屏；卡牌元数据由快照 `cards` 下发，不硬编码 |
| 前端 | 大屏 | ✅ M4 | 攻击/防守情景、隐藏票数、截止揭晓、SPENT 卡片墙 |
| 配置 | `app/sabotage_config.py` | ✅ M2 | 卡牌、6×6 效果矩阵、剧本、计时、预算（改配置不改代码） |

**不需要新建**：WS 通道、鉴权依赖、分数表结构（`sabotage_score` 已存在）、房间/玩家 CRUD。

### 自动开赛与重开（admin 零操作）

主流程推进到 Sabotage 节点（`flow.py` 里 `game: "sabotage"`）时，房间的广播监听器调用 `auto_start_sabotage`：随机均分红蓝（写 `players.team`）→ 启动引擎 → 进入 8s 揭晓。

**每次 `go/{index}` 跳转或 `reset` 都会先丢弃旧的 sabotage 引擎**（连 runner 一起取消），所以跳到节点 5 永远是全新一局：按**当前房间玩家**重新随机分队、从 team_reveal 重跑。排练时反复快进/回退都能重新测试，不会复用上一局的分队或进度。现场不需要任何「Assign Teams / Start」操作。

## 可配置项（不硬编码）

设计 PDF 第 17 页明确要求以下都可配置：

| 类 | 项 | 现设计取值 |
|---|---|---|
| 剧本 | 三轮 attack/defend 文案 + learning 提示 | 钓鱼邮件 / CEO 冒充 / AI 助手操纵 |
| 攻击卡 | 6 张的 name / description / cost / success / fail | 见上表 |
| 防守卡 | 6 张同上 | 见上表 |
| 计时 | team reveal、每段投票、tie 重投 | 8s / 20s / 5s |
| 预算 | 每队初始 budget | $100k |
| **效果矩阵** | attack × defence 谁克谁 | **已定稿**：见上「效果矩阵」表 |
| 结算 | payout 规则、success/fail 的分值 | 胜利方 +100；blocked/partial/failed 见计分 |

**配置放哪**：理想是后端一份配置（`flow.py` 风格），前端渲染时从服务端拿；至少卡牌与效果矩阵必须服务端权威，因为团队分由服务端结算。前端**不能**自己算团队分——否则与「团队共享一个分数、服务端写库」矛盾。

## 时长预留

子阶段时长在 `sabotage_config.py` 的 `TIMINGS` 里，正常总时长 ≈ **2.5 分钟**：

`reveal 8s + 3×(attack 20s + defend 20s + resolve 4s) + team_result 6s + payout 5s ≈ 151s`，另加平票重投余量（每次 tie 5s，最多 3 次）。

- `flow.py` 的 sabotage 节点 `duration_sec` 已从 30s 调到 **170s**（含平票余量），Sabotage 打完后主节点到点自动推进到 Sabotage 榜单（5s），再到 Ultimate Quiz。
- **暂停联动**：操作员 pause/resume 主流程时，Sabotage 的子阶段时钟同步冻结/恢复（`game.py` 联动 + 广播）。
- Sabotage 结束后 `sabotage_score` 的写库在 M5 完成。

## 与现有系统的衔接

- `sabotage_score` 列已存在，但语义是「团队共享总分」，不是逐题累加；与第 1 项（分数上报）不同，本局分数由**服务端结算写库**，玩家不调用 `/api/me/score`。
- 需要区分 COMPETING / LEARNING 玩家，现有登录/名单没有这个维度。
- 团队态是进程内内存态，活动期间禁部署的既有红线同样适用。
