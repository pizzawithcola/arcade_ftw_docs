# 状态机

实现：`arcade_ftw_be/app/engine.py` 的 `GameState`。

## 阶段

`idle → playing ⇄ paused → finished`

```python
class Phase(str, Enum):
    IDLE = "idle"
    PLAYING = "playing"
    PAUSED = "paused"
    FINISHED = "finished"
```

## 位置

`current_index` 指向 `ROUND_FLOW` 中的当前节点；`current` 属性返回该节点（越界返回 `None`）。

## 转移

| 方法 | 效果 |
|---|---|
| `start()` | `→ playing`，`current_index = 0` |
| `pause()` | `playing → paused` |
| `resume()` | `paused → playing` |
| `finish()` | `any → finished` |
| `go_to(index)` | 跳转到任意节点（**管理员回退**用），越界忽略 |
| `next()` | `current_index + 1` |
| `snapshot()` | 供 WS 下发的状态快照 |

## 设计约束

- 管理员控制全部是**状态转换事件**，不要用 if/else 补丁。
- 状态是**进程内内存态**：单进程运行，禁止多 worker；换 task 会清空（见手册红线 #3）。

## 待定

- 计时（`duration_sec`）到点自动推进：尚未实现
- 玩家级状态（每人答案、得分）与全局状态的边界：尚未定
