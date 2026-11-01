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
| Data Shadow | **两栏**：左边推文正文（没有 report panel，所以文字可以放大），右边两张图纵向排列、各占一档固定高度。整块必须放进框内——投影仪不滚动 |
| AI Cyber Battle | **五个阶段各有一块「仪表」，不是一段说明文字**（见下） |
| Ultimate | 题干 + **四个选项**（A/B/C/D 两列网格，与平板同一份选项、同一个字母）；每题最后 **5 秒**换成 **`STANDINGS`** 实时排行榜（`RoomLeaderboard`，section `ultimate`）；第 4、5 题前的 **3 秒 bonus 卡**（和 iPad 同一台老虎机：追光灯珠 + 滚轮落到 `2` / `3`，下面写 `Gain 200 instead of 100`；kicker 仍是 `question 4 / 5`） |
| `leaderboard` | 该 section 的完整榜单 |
| `outro` | 结束语 |

## AI Cyber Battle 的大屏（2026-11-01 重做）

原来五个阶段都是「居中一段话」，是全站最素的一屏：信息少、没有图形、字号像图注。
现在每块都由同样四样东西组成——**轮次轨、两队、三个维度、一个大到后排能读的数字**：

| 阶段 | 大屏内容 |
|---|---|
| 分队 `team_reveal` | `Find your colour` + 两队在各自颜色里的大面板（队名、人数、剩余预算条） |
| 规则 `brief` | 轮次轨 R1–R4 + **`$100,000` 大字** + 四枚事实胶囊（4 rounds / 3 columns / 1 shared budget / target column wins）+ 两队面板 |
| 选卡 `select` | 轮次轨 + `ROUND n OF 4` + 主题 + **三个维度胶囊（目标那个点亮）** + 攻/守两块各自颜色的剧本卡 + 两队剩余预算条 |
| 揭晓 `reveal` | 轮次轨 + **结果大字**（`Full defence` / `Effective breach`…）+ 三个维度画成**拔河条**（紫=攻、青=守，带数值与 BREACH/TIE/HELD，目标列高亮）+ **双方本轮的牌**（卡名 + 花费）+ 剩余预算 + 红色那句「The bet: …」 |
| 终局 `final` | `TEAM A WINS` / `DRAWN` + 两队总分大数字 + 各赢几轮 |

细节：

- **轮次轨**用 `sabotage.history` 着色：已打完的轮次按赢家颜色（平局/弃权用虚线框），当前轮亮红。
- **拔河条**按 `max(attack, defence)` 归一化，所以每一行里两条的相对长度就是这一列的真实差距。
- **双方手牌**只有结算阶段才公开（选卡阶段 `builds` 是空的，这是刻意的）。
- 大屏**不显示**任何人名（`teams` 里只有玩家 id），只报人数。
- 「Answer on your iPad →」这行兜底文案**只在选卡阶段出现**，分队/规则/揭晓/终局都不再显示。

## 两个「怎么算出来的」

- **当前第几题**：由节点时钟推导（`questionOnScreen`），广播只在换题时发一次，
  信广播的大屏会整场停在第 1 题。
- **什么时候切榜单**：`seconds_each × 1000 − remaining_ms ≥ 20000`，即每题的最后 5 秒，
  与平板「20 秒作答 + 5 秒揭晓」同一套切分。
