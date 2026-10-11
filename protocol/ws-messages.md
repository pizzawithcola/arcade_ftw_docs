# WS 消息协议

前后端唯一契约。**改这里必须同时改 `arcade_ftw_be/app/ws.py` 与 `arcade_ftw_fe/src/store/gameStore.ts`。**

## 信封

所有消息统一为：

```json
{ "type": "<string>", "payload": { } }
```

前端类型定义：[`arcade_ftw_fe/src/store/gameStore.ts`](../../arcade_ftw_fe/src/store/gameStore.ts) 的 `WsMessage<T>`。

## 已实现

| 方向 | type | payload | 说明 |
|---|---|---|---|
| server → client | `state` | `{ phase, current_index, current }` | 连接建立时立即推送一次状态快照 |
| client → server | `echo` | 原样回传 | ⚠️ **占位实现**，接入真实状态机后移除 |

`current` = 当前流程节点，见 [flow 配置](#流程节点)。

## 待实现

| 方向 | type | 说明 |
|---|---|---|
| server → client | `node` | 流程推进到新节点 |
| server → client | `question` | 下发题目（payload 由题型 handler 的 `render()` 决定）|
| server → client | `result` / `leaderboard` | 计分与排行 |
| client → server | `answer` | 玩家提交答案 |
| client → server | `admin` | 管理员控制（pause / resume / finish / go_to）|

## 流程节点

```json
{ "type": "game", "game": "hallucinate", "duration_sec": 180 }
{ "type": "quiz", "question_ids": [12, 33] }
```

配置源：`arcade_ftw_be/app/flow.py` 的 `ROUND_FLOW`。**改流程只改配置，不改引擎。**
