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
  "index": 2,
  "node_count": 13,
  "node": { "type": "quiz", "title": "Quiz round 1", "duration_sec": 46, "questions": 2, "seconds_each": 20, "score": "hallucination" },
  "flow": [ { "type": "intro", "title": "Welcome", "duration_sec": 10 }, "…整条 rundown…" ],
  "remaining_ms": 16780,
  "question": {
    "index": 0,
    "count": 2,
    "seconds_each": 20,
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

当前主线（13 个节点）：

```json
{ "type": "intro", "title": "Welcome",       "duration_sec": 10 }
{ "type": "game",  "game": "hallucination",  "title": "Hallucination", "duration_sec": 66, "questions": 2, "seconds_each": 30, "score": "hallucination" }
{ "type": "quiz",  "title": "Quiz round 1",  "duration_sec": 46, "questions": 2, "seconds_each": 20, "score": "hallucination" }
{ "type": "leaderboard", "section": "hallucination", "title": "Hallucination standings", "duration_sec": 5 }
{ "type": "game",  "game": "data_shadow",    "title": "Data Shadow",   "duration_sec": 51, "score": "data" }
{ "type": "quiz",  "title": "Quiz round 2",  "duration_sec": 46, "questions": 2, "seconds_each": 20, "score": "data" }
{ "type": "leaderboard", "section": "data",  "title": "Data Shadow standings", "duration_sec": 5 }
{ "type": "game",  "game": "sabotage",       "title": "Sabotage",      "duration_sec": 170, "score": "sabotage" }
{ "type": "leaderboard", "section": "sabotage", "title": "Sabotage standings", "duration_sec": 5 }
{ "type": "quiz",  "title": "Ultimate Quiz", "duration_sec": 46, "questions": 2, "seconds_each": 20, "score": "ultimate" }
{ "type": "leaderboard", "section": "ultimate", "title": "Ultimate standings", "duration_sec": 5 }
{ "type": "leaderboard", "section": "total", "title": "Activity Leaderboard", "duration_sec": 5 }
{ "type": "outro", "title": "Wrap up",       "duration_sec": 10 }
```

配置源：`arcade_ftw_be/app/flow.py` 的 `ROUND_FLOW`。**改流程只改配置，不改引擎。**

### 节点的两个身份字段

- **`type` = 类别**（`intro` / `brief` / `game` / `quiz` / `leaderboard` / `outro`），是引擎唯一关心的字段。
- **`game` = 具体游戏**，只出现在 `game` 节点上。三个游戏外观完全不同，但共用一个类别——所以**前端在自己那一层按 `game` 分派界面**，`type` 保持宽泛。

前端用两个 class 落地这件事：`stage--game`（共用的游戏版式）**加** `stage--hallucination`（该游戏专属的钩子）。只给类别 class 的话，三个游戏会共用一个不可区分的钩子。

**`score` = 计分分区**：只有计分节点才有，取值 `hallucination` / `data` / `sabotage` / `ultimate`。游戏与其后的 quiz 共用同一个分区，服务端据此决定玩家上报的分数写哪一列（详见 [views](../architecture/views.md)）。

**`section` = 榜单分区**：只出现在 `leaderboard` 节点上，决定该节点展示哪个分区的排名（`total` 是活动总榜占位）。大屏与玩家平板都渲染它。

**`title` 纯展示**：观众看它，代码永远不拿它做分支。正因如此，两轮普通 quiz 和最后的 Ultimate Quiz 可以都是 `type: "quiz"`，而代码不需要区分它们。

### `brief` 节点：每个游戏前的 15 秒玩法说明

四组玩法（Hallucination / Data Shadow / Sabotage / Ultimate Quiz）各有一个前置 `brief` 节点，`duration_sec: 15`：

```json
{ "type": "brief", "brief": "sabotage", "title": "How Sabotage works",
  "duration_sec": 15, "section": "sabotage" }
```

- **`brief` = 文案键**（`hallucination` / `data_shadow` / `sabotage` / `ultimate_quiz`）。引擎不认识它——只是一个普通的定时节点，到点自动前进；文案在前端 `arcade_ftw_fe/src/games/briefs.ts`，所以改措辞不用动后端。
- **`section` 只用于分组**：让操作台把这张说明卡和它介绍的 section 放进同一个跳转框，表现为 `Challenge` 左边的一个 `intro` 按钮。`brief` 节点**不带 `score`**，永远不参与计分。
- 两端都渲染：玩家平板一屏一列，大屏是同一段文字的大字号版本。
- 顺序是硬约束：`brief` 必须**紧邻其后的**目标节点，`tests/test_flow.py` 会校验这条相邻关系和 15 秒时长。
