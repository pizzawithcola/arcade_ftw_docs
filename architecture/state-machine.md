# 状态机与流程引擎

实现：`arcade_ftw_be/app/engine.py`。流程配置：`app/flow.py`。

## 决策：引擎自主推进

**主线由引擎按时间自己推进，管理员只能干预。** 这是刻意的：

- 现实里没人能手动按钮推完几十上百个节点；
- 「所有人同时进行」要求推进时刻是**唯一权威**的，不能依赖某个人点得准。

管理员的动作全部是**状态转换**，不是补丁：

| 动作 | 效果 |
|---|---|
| `start` | `idle/finished → playing`，**无条件回到第 0 个节点** |
| `pause` | `playing → paused`，**冻结剩余时间** |
| `resume` | `paused → playing`，从冻结处继续 |
| `finish` | `any → finished` |
| `go/{index}` | 跳到任意节点；也是从 `finished` / `idle` 复活的方式 |
| `reset` | `any → idle`，回到「干净待演」态（排练完交台用）|

## 阶段

```text
  idle ──start──▶ playing ⇄ paused
    ▲                │         │
    │                └───┬─────┘
  reset ◀───────────────┴──▶ finished
```

## 转移表（已逐格实测）

| 当前 ↓ / 动作 → | `start` | `pause` | `resume` | `finish` | `go/{i}` | `reset` |
|---|---|---|---|---|---|---|
| **`idle`** | → `playing` idx=0 | 无操作 | 无操作 | → `finished` | → `playing` idx=i | 无操作 |
| **`playing`** | **无操作**（不重置节点）| → `paused` 冻结剩余 | 无操作 | → `finished` | `playing` idx=i，**满时长** | → `idle` |
| **`paused`** | → `playing` **idx 归 0** ⚠️ | 无操作 | → `playing` 精确接续 | → `finished` | `paused` idx=i，**满时长** | → `idle` |
| **`finished`** | → `playing` idx=0 | 无操作 | 无操作 | 无操作（幂等）| → `playing` idx=i | → `idle` |

时间驱动（非人为）：`playing` 倒计时归零 → 自动进下一节点并重新装填；已是最后一个节点 → `finished`。

**两个不变量**：

1. runner 任务在 `finished` 时主动退出；引擎抛未捕获异常时也会落到 `finished`（不会静默卡死）。`reset` 会**取消**该任务——否则一个活着的 runner 会永远停在 wake 事件上等不到唤醒，变成泄漏的协程。
2. `idle` 在实现 `reset` 之前**不可达**（只在模块加载时赋值一次），所以排练到底后没有任何办法回到「干净待演」态。

### ⚠️ 操作员必须知道的：`go/{i}` 之后按 `resume`，不要按 `start`

`start` 的语义是「从第 1 节点开演」，它会**无条件把 `index` 归 0**。所以：

```
暂停中跳到第 4 节点 → 按 start  → 弹回第 1 节点 ⚠️
暂停中跳到第 4 节点 → 按 resume → 正确播放第 4 节点 ✅
```

前端据此刻意把 `start` **只在 `idle` / `finished` 时可点**（`playing` 期间 `start` 本来就是纯无操作），让这个陷阱无法从界面上触发。但从 `/docs` 或 curl 直接打 API 仍会命中，**用 API 操作时请自己注意**。

### `reset` 用在哪

排练循环：`reset` → `start` 从头再来，**一个键**。以前的替代路径是 `finish` → `start`，且只能用在演完之后。

`idle` 在现场的表现是那句「Waiting for the show to start…」——**不是**停在第 1 节点的 0:00。前端在 `phase === 'idle'` 时就走等待屏，否则 reset 出来的画面会像一个卡住的节目。

## 三条时钟设计（都踩过坑）

1. **单调时钟**：倒计时基于 `time.monotonic()`，宿主调系统时间不会让节目跳。
2. **暂停 = 冻结剩余时间**，而不是挪动一个 deadline。这样暂停多久，恢复后的剩余时间都完全一致。
3. **快照给的是 `remaining_ms`（时长），不是绝对截止时刻**。前端据此在本机换算倒计时——**机器时钟有偏差只会整体平移显示，不会改变节点长度**。

## 快照字段（WS `state` 的 payload）

```json
{
  "phase": "playing",
  "index": 2,
  "node_count": 8,
  "node": { "type": "quiz", "title": "Quiz round 1", "duration_sec": 20 },
  "remaining_ms": 16780,
  "server_time_ms": 1791733393351
}
```

## 广播

引擎每次状态变化都通知监听器，监听器把快照**扇出**给所有 WS 连接（`app/manager.py`）。这就是「所有人同步推进」的实现方式——不是每个客户端各自轮询。

## 已知的坑（都已在代码与测试中体现）

1. **新节点的广播必须带非零剩余时间**：最初 `_arm()` 是在守卫任务里延迟执行的，导致新节点广播出去时 `remaining_ms` 是 `0`，客户端会**永远卡在 0:00**。现在 `start()` / `go_to()` / 推进时都**先装填时钟再通知**，并有回归测试。
2. **后台标签页的倒计时会冻结**：浏览器节流隐藏标签页的 `setInterval`，`now` 变旧 → 剩余时间**偏大且不动**。修法是 `visibilitychange` / `focus` 时立刻刷新（`useTicker`）。排查时不要误判为引擎问题——**先和服务端 `/api/game/state` 对拍**。
3. **暂停中跳转曾把屏幕停在 0:00**：`go_to()` 原来无条件把剩余时间清零，于是「暂停 → 跳转」后大屏显示 `0:00`，看起来像节点没有时长。现在若跳转后仍是 `paused`，就把**新节点的满时长**装进剩余时间。注意 `resume()` 里写的是 `self._remaining or self._duration()`——剩余为 0 时会回退到满时长，所以这个修复也让「跳转后 resume」播放的是**目标节点**的时长，而不是第 1 节点的。
4. **内存态**：引擎是进程内单例。单进程运行、禁止多 worker；**替换容器 = 重置整场节目**。

## 待定

- 玩家级状态（每人答案、得分）与全局状态的边界。
- 节点结束是否需要「收尾动作」（例如 quiz 到点自动结算）。
