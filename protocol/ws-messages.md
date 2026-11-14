# WS 消息协议

前后端唯一契约。**改这里必须同时改 `arcade_ftw_be/app/ws.py` 与 `arcade_ftw_fe/src/store/gameStore.ts`。**

## 端点

```
ws(s)://<host>/api/ws
```

路径带 `/api` 前缀：CloudFront 只用**一条 `/api/*` 行为**把游戏相关流量反代到后端 ALB，所以新增接口永远不需要改基础设施。`/health`、`/health/db` 是唯一留在根路径的例外（ALB 直连探活）。

前端用相对地址推导（生产 `wss://<cloudfront 域名>/api/ws`，本地开发由 Vite proxy 转发到 `localhost:5071`），因此**同一份代码在本地与线上都能跑**。

## 信封

所有消息统一为：

```json
{ "type": "<string>", "payload": { } }
```

前端类型定义：[`arcade_ftw_fe/src/store/gameStore.ts`](../../arcade_ftw_fe/src/store/gameStore.ts) 的 `WsMessage`。

## 已实现

| 方向 | type | payload | 说明 |
|---|---|---|---|
| server → client | `state` | 见下 | 连接建立时立即推一次；此后每次状态变化**扇出给所有连接** |
| server → client | `sabotage` | 见下 | Sabotage 引擎自己的快照：连接时推一次（若房间在 Sabotage 中），此后每次子阶段推进 / 暂停 / 恢复时扇出 |

**客户端不发消息。** 管理员控制走 HTTP（见 `arcade_ftw_be/app/game.py`）：可鉴权、能在 `/api/docs` 里直接点、失败有真实状态码。状态变化再由服务端广播。所以 WS 是**单向推送通道**。Sabotage 的投票同样走 HTTP（`POST /api/sabotage/{code}/vote`），WS 只负责把结果推回来。

`state` 的 payload：

```json
{
  "phase": "waiting | playing | paused | finished",
  "index": 13,
  "node_count": 17,
  "node": { "type": "quiz", "title": "Ultimate Quiz", "duration_sec": 84, "questions": 3, "seconds_each": 25, "answer_sec": 20, "score": "ultimate" },
  "flow": [ { "type": "intro", "title": "Welcome", "duration_sec": 10 }, "…整条 rundown…" ],
  "remaining_ms": 16780,
  "question": {
    "index": 0,
    "count": 3,
    "seconds_each": 25,
    "remaining_ms": 14320
  },
  "server_time_ms": 1791733393351
}
```

`flow` 是**整条流程**（同 `flow.py` 的节点列表），让管理台能按 section 给每个跳转按钮打标签，而不必在前端复制一份流程。客户端可以忽略它。

> 前端**不要用 `server_time_ms` 去算截止时刻**——用 `remaining_ms` 自己在本地换算。给的是时长而不是时刻，所以机器时钟有偏差只会整体平移，不会改变节点长度。

`question` 是**引擎按节点时钟算出的当前题号**（`app/engine.py` 的 `_question()`），不是设备上报的：

- 只有声明了 `questions` + `seconds_each` 的节点才有值，其余节点为 `null`。
- `waiting` / `finished` 时固定给第 1 题、`remaining_ms: 0`，避免前端在无时钟状态下算出越界题号。
- 大屏靠它展示题面，所有 iPad 的题号因此天然一致；选项仍只留在 iPad 上。

### `sabotage` 的 payload

Sabotage 是唯一一个「主流程节点里还套着自己的时钟」的游戏，所以它的快照单独走一条消息，而不是塞进 `state`：

```json
{
  "code": "round-3",
  "phase": "attack",
  "paused": false,
  "round": 1,
  "attacker": "red",
  "defender": "blue",
  "remaining_ms": 15200,
  "budgets": { "red": 100000, "blue": 100000 },
  "scores": { "red": 0, "blue": 0 },
  "spent": [],
  "scenario": { "attacker": "red", "defender": "blue", "attack_title": "...", "attack_prompt": "...", "defence_title": "...", "defence_prompt": "...", "learning": "..." },
  "voting_team": "red",
  "options": ["deepfake_video", "cloned_voice", "..."],
  "tie_candidates": [],
  "attack_card": null,
  "defence_card": null,
  "last_result": null
}
```

- `phase`：`setup / team_reveal / attack / defend / tie_break / resolve / team_result / payout / done`
- `remaining_ms`：当前子阶段的剩余时长；`paused` 时是冻结值（与主引擎同款语义）
- `options`：当前行动方可选的卡；**不暴露任何人的投票**
- `last_result`：最近一次 resolve 的结果（含 `outcome` / `attack_delta` / `defence_delta`，payout 后带 `winner`）

前端类型：[`arcade_ftw_fe/src/store/gameStore.ts`](../../arcade_ftw_fe/src/store/gameStore.ts) 的 `SabotageState`。主引擎的 `state` 与 Sabotage 的 `sabotage` **并行广播、互不覆盖**。

## 待实现

| 方向 | type | 说明 |
|---|---|---|
| server → client | `question` / `result` | 题目下发与结算（payload 由题型 handler 的 `render()` 决定）|
| client → server | `answer` | 玩家提交答案（届时 WS 才会变成双向）|

## 流程节点

主线是**按房间的模块集合生成的**，不是一份固定列表。默认（全开）13 个节点、528 秒；建房时传 `modules` 可以只留其中几个，节点数、总时长、满分随之变化，且**被隐藏的模块在建房时就不在 flow 里**——引擎没有"跳过"这个动作。

```json
// POST /api/rooms
{ "code": "R-111", "modules": ["welcome", "hallucination", "data", "ultimate", "total", "wrapup"] }
```

七个模块键：`welcome`（intro）、`hallucination` / `data` / `sabotage` / `ultimate`（各含它的 opening / 游戏 / 分节榜）、`total`（活动总榜）、`wrapup`（outro）。`modules` 省略或 `null` = 全开，兼容本改动之前的所有房间。

| 配置 | 节点 | 时长 | 满分 |
|---|---|---|---|
| 全开（HK FTW / HK Employee） | 13 | 8m48s / 528s | 2800 |
| 关 AI Cyber Battle（SG FTW） | 11 | 5m25s / 325s | 2000 |
| 关 Battle + 关 Ultimate | 9 | 3m46s / 226s | 1400 |

房间生效后**不可改**：要换配置只能新建房间。这是刻意的——节点索引是分数上报键（`room:node_index:question`）的一部分，运行中改流程会让一条迟到的上报写进别的游戏的列。

`state` 消息里相应多了两个字段：

```json
{ "modules": ["data", "hallucination", "total", "ultimate", "welcome", "wrapup"], "total_sec": 325.0 }
```

默认全开时的主线（13 个节点，共 528 秒 / 8m48s）：

```json
{ "type": "intro",   "title": "Welcome",           "duration_sec": 10, "requires": "welcome" }
{ "type": "opening", "game": "hallucination", "title": "AI Hallucination", "duration_sec": 15, "title_sec": 3, "section": "hallucination" }
{ "type": "game",    "game": "hallucination", "title": "Hallucination", "duration_sec": 90, "questions": 3, "seconds_each": 30, "answer_sec": 25, "score": "hallucination" }
{ "type": "leaderboard", "section": "hallucination", "title": "Hallucination standings", "duration_sec": 8, "requires": "hallucination" }
{ "type": "opening", "game": "data_shadow",   "title": "Data Shadow",      "duration_sec": 15, "title_sec": 3, "section": "data" }
{ "type": "game",    "game": "data_shadow",   "title": "Data Shadow",      "duration_sec": 60, "answer_sec": 50, "score": "data" }
{ "type": "leaderboard", "section": "data",        "title": "Data Shadow standings", "duration_sec": 8, "requires": "data" }
{ "type": "game",    "game": "sabotage",      "title": "AI Cyber Battle",         "duration_sec": 203, "score": "sabotage" }
{ "type": "leaderboard", "section": "sabotage",    "title": "AI Cyber Battle standings", "duration_sec": 8, "requires": "sabotage" }
{ "type": "opening", "game": "ultimate_quiz", "title": "Ultimate Showdown", "duration_sec": 15, "title_sec": 3, "section": "ultimate" }
{ "type": "quiz",    "title": "Ultimate Quiz", "duration_sec": 84, "questions": 3, "seconds_each": 25, "answer_sec": 20, "bonus_intro_sec": 3, "bonus_questions": [2, 3], "score": "ultimate" }
{ "type": "leaderboard", "section": "total",       "title": "Activity Leaderboard", "duration_sec": 10, "requires": "total" }
{ "type": "outro",   "title": "Wrap up",           "duration_sec": 10, "requires": "wrapup" }
```

> 四个要注意的变化（2026-10-31）：
> 1. **名字卡与规则卡已合并为 `opening`**：15 秒里前 `title_sec`（3 秒）是名字卡，后 12 秒是规则，控制台不再需要分别跳转。
> 2. **Sabotage 前面没有 `opening`**：它先抽队伍，然后由游戏自己在节点内播 15 秒规则卡，flow 层再加一张就把同一局介绍两次。
> 3. **Ultimate 后面没有分节榜**：分数直接进 10 秒活动总榜，避免同一批数字连播两次。
> 4. Hallucination 与 Data Shadow 之间的 `quiz` 节点仍然没有（只剩 Ultimate Quiz 一个 quiz），Ultimate Quiz 也不带 `intro_sec`，第一秒就是第一题。

> 5. **`answer_sec` 是「选手能操作的那段」**（2026-11-14 补）。一个节点的钟比它长：每题答完还有 5 秒 verdict，Data Shadow 交卷后还有 10 秒揭晓。大屏原来直接播节点钟，于是 Ultimate 每题大屏读 0:25、iPad 读 20s——全场唯一对不上的数字。现在大屏播 `answer_sec` 这一段，尾段单独倒数（正好是 iPad 上 verdict 的那几秒）。Sabotage 不带这个字段：两边都读 `sabotage.remaining_ms`，同一个来源。

配置源：`arcade_ftw_be/app/flow.py` 的 `ROUND_FLOW`。**改流程只改配置，不改引擎。**

### 节点的两个身份字段

- **`type` = 类别**（`intro` / `opening` / `game` / `quiz` / `leaderboard` / `outro`），是引擎唯一关心的字段。
- **`game` = 具体游戏**，出现在 `opening` 与 `game` 节点上。各游戏外观不同，所以**前端在自己那一层按 `game` 分派界面**，`type` 保持宽泛。

前端用两个 class 落地这件事：`stage--game`（共用的游戏版式）**加** `stage--hallucination`（该游戏专属的钩子）。只给类别 class 的话，三个游戏会共用一个不可区分的钩子。

**`score` = 计分分区**：只有计分节点才有，取值 `hallucination` / `data` / `sabotage` / `ultimate`。当前每个分区由一个计分节点写入；服务端据此决定玩家上报的分数写哪一列（详见 [views](../architecture/views.md)）。

**`section` = 玩法 / 榜单分区**：`opening` 节点用它把开场归入对应玩法；`leaderboard` 节点用它决定展示哪个分区的排名（`total` 是活动总榜）。大屏与玩家平板都渲染榜单。

**`title` 纯展示**：观众看它，代码永远不拿它做分支。正因如此，两轮普通 quiz 和最后的 Ultimate Quiz 可以都是 `type: "quiz"`，而代码不需要区分它们。

### `opening` 节点：名字与规则合成 15 秒

Hallucination、Data Shadow 与 Ultimate Quiz 各有一个 `opening` 节点，`duration_sec: 15`、`title_sec: 3`：前三秒显示名字卡，后十二秒逐条显示规则。Sabotage 在自动分队后由游戏内部展示规则卡，因此没有 flow 层的 `opening`。

- `game` 选择开场内容，`section` 供管理台归组；文案由前端 `arcade_ftw_fe/src/games/openings.ts` 提供。
- 开场节点不带 `score`，不参与计分；平板与大屏使用同一份规则内容。
