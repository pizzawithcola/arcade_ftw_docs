# 大屏（Display）

`arcade_ftw_fe/src/views/display/DisplayPage.tsx` · 样式 `DisplayPage.css`。
路由 `/display`（跟最新未关闭房间）或 `/display/:code`（钉住房间）。

**大屏只读**：所有作答都在平板上发生，屏幕只负责让整场看得见。以下规则同样适用于
`Opening.css` 里的 `.opening--display` 分支——**玩家平板的样式一律不动**。

## 逐屏内容

| 节点 | 大屏显示 |
|---|---|
| `intro`（Welcome） | 四张游戏卡（编号 + 图标 + 名称），副标题 `Four Games, One Winner!` |
| `opening` | 与平板同一份规则：`1/2/3` 编号小标题 + 三行规则。平板用 12 秒的节奏逐条浮现，大屏同样逐条，但字号放到后排可读（`clamp(1.15rem, 1.65vw, 1.9rem)`），整块在框内居中 |
| Hallucination | 完整对话（用户气泡 + AI 回答），**下方一行大字 `TRUST or DOUBT`**：`TRUST` 绿（`--c-win`）、`DOUBT` 琥珀（`--c-warning`）、中间 `or` 灰 |
| Data Shadow | **两栏**：左边推文正文（没有 report panel，所以文字可以放大），右边三张图纵向排列、各占一档固定高度。整块必须放进框内——投影仪不滚动 |
| Sabotage | 分队页「找同色的屏幕」、选卡页的目标与剩余预算、结算页本轮赢家 |
| Ultimate | 题干 + **四个选项**（A/B/C/D 两列网格，与平板同一份选项、同一个字母）；每题最后 **5 秒**换成 **`STANDINGS`** 实时排行榜（`RoomLeaderboard`，section `ultimate`） |
| `leaderboard` | 该 section 的完整榜单 |
| `outro` | 结束语 |

## 两个「怎么算出来的」

- **当前第几题**：由节点时钟推导（`questionOnScreen`），广播只在换题时发一次，
  信广播的大屏会整场停在第 1 题。
- **什么时候切榜单**：`seconds_each × 1000 − remaining_ms ≥ 20000`，即每题的最后 5 秒，
  与平板「20 秒作答 + 5 秒揭晓」同一套切分。
