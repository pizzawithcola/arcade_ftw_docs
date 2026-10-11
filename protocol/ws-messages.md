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

前端类型定义：[`arcade_ftw_fe/src/store/gameStore.ts`](../../arcade_ftw_fe/src/store/gameStore.ts) 的 `WsMessage<T>`。

## 已实现

| 方向 | type | payload | 说明 |
|---|---|---|---|
| server → client | `state` | 见下 | 连接建立时立即推一次；此后每次状态变化**扇出给所有连接** |

**客户端不发消息。** 管理员控制走 HTTP（见 `arcade_ftw_be/app/game.py`）：可鉴权、能在 `/docs` 里直接点、失败有真实状态码。状态变化再由服务端广播。所以 WS 是**单向推送通道**。

`state` 的 payload：

```json
{
  "phase": "idle | playing | paused | finished",
  "index": 2,
  "node_count": 6,
  "node": { "type": "quiz", "title": "Quiz round 1", "duration_sec": 20 },
  "remaining_ms": 16780,
  "server_time_ms": 1791733393351
}
```

> 前端**不要用 `server_time_ms` 去算截止时刻**——用 `remaining_ms` 自己在本地换算。给的是时长而不是时刻，所以机器时钟有偏差只会整体平移，不会改变节点长度。

## 待实现

| 方向 | type | 说明 |
|---|---|---|
| server → client | `question` / `result` | 题目下发与结算（payload 由题型 handler 的 `render()` 决定）|
| client → server | `answer` | 玩家提交答案（届时 WS 才会变成双向）|

## 流程节点

```json
{ "type": "intro", "title": "Welcome", "duration_sec": 10 }
{ "type": "game", "game": "hallucinate", "title": "Hallucinate", "duration_sec": 30 }
{ "type": "quiz", "title": "Quiz round 1", "duration_sec": 20 }
{ "type": "outro", "title": "That's all", "duration_sec": 10 }
```

配置源：`arcade_ftw_be/app/flow.py` 的 `ROUND_FLOW`。**改流程只改配置，不改引擎。**

每个节点都带 `title`，前端**不依赖节点类型**也能渲染出一个像样的占位屏；`type` 决定前端用哪种界面。
