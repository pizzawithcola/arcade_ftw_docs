# Sabotage（AI Cyber Sabotage）

主流程第 6 个节点（`type: game`，`game: sabotage`）。
设计来源：`design/AI_Cyber_Sabotage_Developer_Storyboard_High_Resolution.pdf`（17 页）。目前**仍是占位屏**，本文是落地前的设计拆解。

> 状态：**设计已定稿（M0）+ 团队分配已实现（M1）**。玩法与效果矩阵见下；`flow.py` 的 sabotage 节点仍是 30s 占位，投票引擎（M2）与前端（M4）尚未实现，`sabotage_score` 尚无写入方。
>
> 模块进度：M0 ✅ 设计定稿 · M1 ✅ 团队分配 · M2 ⬜ 引擎+投票 · M3 ⬜ 流程接入 · M4 ⬜ 前端 · M5 ⬜ 结算联调

## 一句话

红蓝对抗的三轮攻防投票游戏：每轮全队用平板**私下投票**，多数票成为**全队唯一行动**；攻击卡是**两队共享、用过即废**的 6 张军火库，防守卡是**每轮可复用**的 6 张应对牌；团队胜负折算成个人分后进入 Ultimate Showdown。

## 玩法

| 规则 | 说明 |
|---|---|
| 参与资格 | 只有开始时状态为 `COMPETING` 的玩家分红蓝；**Learning Mode 全程只观察**，不投票、不计分、平板锁定 |
| 分队 | 竞技玩家尽量均分（20 人 → 10 红 / 10 蓝） |
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
分队揭晓 10s
  → R1 红攻（钓鱼邮件）× 蓝守 → 结算
  → R2 蓝攻（CEO 冒充）× 红守 → 结算
  → R3 红攻（AI 助手操纵）× 蓝守 → 结算
  → 团队结果 → 个人分发放 → 实时排行榜 → Ultimate Showdown（3 题）
```

每轮三段，每段投票 **20 秒**：

1. **ATTACK**：进攻方在未 `SPENT` 且买得起的攻击卡里选（R1 6 张 → R2 5 张 → R3 4 张）
2. **DEFEND**：防守方 6 张防守卡全可选，按预算禁用
3. **RESOLVE**：公布「攻击 vs 防守」配对，判定 `SABOTAGE SUCCESSFUL / ATTACK BLOCKED`，加减分与预算

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

### 真正的工作量在「节点内子阶段」

比双向通信更难的是：**现在引擎 1 节点 = 1 个时长**，而 Sabotage 一个节点内部要跑 10s 揭晓 + 6×20s 投票 + 3 次结算 + 团队结果 ≈ 2.5 分钟、约 12 个子阶段。

两条路：

- **拆节点**：把 sabotage 拆成多个 `flow.py` 节点（`sabotage_r1_attack` / `sabotage_r1_defend` / …），复用现有引擎时钟。改动小、风险低，但 `flow.py` 会变长，且「团队态」要挂在房间级（跨节点存续）。
- **引擎支持子阶段**：给 `Engine` 加一个 `substage` 概念。架构更干净，但动核心引擎，回归风险高。

**倾向先拆节点**：与「房间级 SabotageEngine 持有团队态」配合，flow 节点只负责计时，状态机逻辑放 SabotageEngine。

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
| 后端 | `app/sabotage.py` | ⬜ M2 | 房间级 SabotageEngine：budgets、shared_attack_spent、votes、团队分、状态机（T4_INIT → … → T5_TRANSITION） |
| 后端 | 投票接口 | ⬜ M2 | `POST /api/sabotage/{room}/vote`（玩家令牌，body: card_id）；可选 `GET /api/sabotage/{room}/state` |
| 后端 | 结算写分 | ⬜ M5 | 三轮后按团队总分对每个队员写同一个 `scores.sabotage_score` |
| 前端 | `games/sabotage/` | ⬜ M4 | 投票屏（选卡、已选高亮、倒计时）、团队揭晓屏、结果屏 |
| 前端 | 大屏 | ⬜ M4 | 攻击/防守情景、隐藏票数、截止揭晓、SPENT 卡片墙 |
| 配置 | `app/sabotage_config.py`（或 docs 约定） | ⬜ M2 | 见下「可配置项」 |

**不需要新建**：WS 通道、鉴权依赖、分数表结构（`sabotage_score` 已存在）、房间/玩家 CRUD。

## 可配置项（不硬编码）

设计 PDF 第 17 页明确要求以下都可配置：

| 类 | 项 | 现设计取值 |
|---|---|---|
| 剧本 | 三轮 attack/defend 文案 + learning 提示 | 钓鱼邮件 / CEO 冒充 / AI 助手操纵 |
| 攻击卡 | 6 张的 name / description / cost / success / fail | 见上表 |
| 防守卡 | 6 张同上 | 见上表 |
| 计时 | team reveal、每段投票、tie 重投 | 10s / 20s / 5s |
| 预算 | 每队初始 budget | $100k |
| **效果矩阵** | attack × defence 谁克谁 | **已定稿**：见上「效果矩阵」表 |
| 结算 | payout 规则、success/fail 的分值 | 胜利方 +100；blocked/partial/failed 见计分 |

**配置放哪**：理想是后端一份配置（`flow.py` 风格），前端渲染时从服务端拿；至少卡牌与效果矩阵必须服务端权威，因为团队分由服务端结算。前端**不能**自己算团队分——否则与「团队共享一个分数、服务端写库」矛盾。

## 时长预留

完整一轮：10s 揭晓 + 6×20s 投票 + 3×结算动画 + 团队结果 + payout，约 **2.5 分钟（150s）**。

- **当前 `flow.py` 里 sabotage 只有 30s**，是占位值，实现时必须重估（拆节点后由各子节点时长相加）。
- 本节先**只改文档不改成代码**，时长重构随实现一起做。

## 与现有系统的衔接

- `sabotage_score` 列已存在，但语义是「团队共享总分」，不是逐题累加；与第 1 项（分数上报）不同，本局分数由**服务端结算写库**，玩家不调用 `/api/me/score`。
- 需要区分 COMPETING / LEARNING 玩家，现有登录/名单没有这个维度。
- 团队态是进程内内存态，活动期间禁部署的既有红线同样适用。
