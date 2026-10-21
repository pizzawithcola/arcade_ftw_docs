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

**客户端不发消息。** 管理员控制走 HTTP（见 `arcade_ftw_be/app/game.py`）：可鉴权、能在 `/api/docs` 里直接点、失败有真实状态码。状态变化再由服务端广播。所以 WS 是**单向推送通道**。

`state` 的 payload：

```json
{
  "phase": "idle | playing | paused | finished",
  "index": 2,
  "node_count": 8,
  "node": { "type": "quiz", "title": "Quiz round 1", "duration_sec": 46, "questions": 2, "seconds_each": 20 },
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

> 前端**不要用 `server_time_ms` 去算截止时刻**——用 `remaining_ms` 自己在本地换算。给的是时长而不是时刻，所以机器时钟有偏差只会整体平移，不会改变节点长度。

`question` 是**引擎按节点时钟算出的当前题号**（`app/engine.py` 的 `_question()`），不是设备上报的：

- 只有声明了 `questions` + `seconds_each` 的节点才有值，其余节点为 `null`。
- `idle` / `finished` 时固定给第 1 题、`remaining_ms: 0`，避免前端在无时钟状态下算出越界题号。
- 大屏靠它展示题面，所有 iPad 的题号因此天然一致；选项仍只留在 iPad 上。

## 待实现

| 方向 | type | 说明 |
|---|---|---|
| server → client | `question` / `result` | 题目下发与结算（payload 由题型 handler 的 `render()` 决定）|
| client → server | `answer` | 玩家提交答案（届时 WS 才会变成双向）|

## 流程节点

当前主线（8 个节点）：

```json
{ "type": "intro", "title": "Welcome",       "duration_sec": 10 }
{ "type": "game",  "game": "hallucination",  "title": "Hallucination", "duration_sec": 66, "questions": 2, "seconds_each": 30 }
{ "type": "quiz",  "title": "Quiz round 1",  "duration_sec": 46, "questions": 2, "seconds_each": 20 }
{ "type": "game",  "game": "data_shadow",    "title": "Data Shadow",   "duration_sec": 51 }
{ "type": "quiz",  "title": "Quiz round 2",  "duration_sec": 46, "questions": 2, "seconds_each": 20 }
{ "type": "game",  "game": "sabotage",       "title": "Sabotage",      "duration_sec": 30 }
{ "type": "quiz",  "title": "Ultimate Quiz", "duration_sec": 46, "questions": 2, "seconds_each": 20 }
{ "type": "outro", "title": "Wrap up",       "duration_sec": 10 }
```

配置源：`arcade_ftw_be/app/flow.py` 的 `ROUND_FLOW`。**改流程只改配置，不改引擎。**

### 节点的两个身份字段

- **`type` = 类别**（`intro` / `game` / `quiz` / `outro`），是引擎唯一关心的字段。
- **`game` = 具体游戏**，只出现在 `game` 节点上。三个游戏外观完全不同，但共用一个类别——所以**前端在自己那一层按 `game` 分派界面**，`type` 保持宽泛。

前端用两个 class 落地这件事：`stage--game`（共用的游戏版式）**加** `stage--hallucination`（该游戏专属的钩子）。只给类别 class 的话，三个游戏会共用一个不可区分的钩子。

**`title` 纯展示**：观众看它，代码永远不拿它做分支。正因如此，两轮普通 quiz 和最后的 Ultimate Quiz 可以都是 `type: "quiz"`，而代码不需要区分它们。
