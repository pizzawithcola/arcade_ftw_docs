# Quiz

主流程里 `type: quiz` 的节点（当前 3 个：Quiz round 1、Quiz round 2、Ultimate Quiz）。
代码：`arcade_ftw_fe/src/games/quiz/`。

三个 quiz 节点**共用同一个屏幕组件**、同一份题库。节点之间靠 `key={nodeIndex}` 重新挂载，所以上一轮的答案不会带进下一轮。

## 玩法

4 选项单选题（A/B/C/D），每题**各自独立计时**（`time_limit`，当前 20 秒）。

| 结果 | 得分 |
|---|---|
| 选对 | **+20** |
| 选错 | 0 |
| 超时未选 | 0 |

答完立刻揭晓：正确项标绿、错误选项标红，并给出**为什么**（`explain`）——让答对的人也知道自己为什么对。

## 字段命名沿用旧系统的题库 schema

`text` / `option_a`~`option_d` / `correct_option` / `time_limit` 与旧 Ultimate Showdown 的 `QuestionBase` 一致。题库上线后（`arcade_ftw_be` 加表 + `GET` 接口），`content.ts` 直接变成测试 fixture，**屏幕组件不用改**。

## 每道题一个倒计时

三道题就是三个独立的 20 秒，不是「整个节点一个总倒计时」。

- 计时锚定在 `questionStart`（挂载/切题的**时间戳**），由 `useTicker` 每 250ms 触发重渲染。
- **超时用派生值，不用 state**：`expired = picked === undefined && timeLeftMs === 0`。曾经用 `useEffect` + `setState` 实现，被 oxlint 的 `react(set-state-in-effect)` 抓出来——那是多余的额外渲染，且状态可以推导就不该存储。
- 超时**不会卡住节目**：直接进入揭晓并显示 `Time's up`，避免一块没人操作的屏幕让整场演出停摆。

## ⚠️ 节点时长必须 ≥ 题目总时长

引擎按 `flow.py` 的 `duration_sec` **自动推进**，到点就切下一个节点。所以节点时长必须覆盖「所有题目的计时之和 + 揭晓收尾」，否则引擎会在玩家答完之前把游戏切走。

当前配置：

| 游戏 | 题目数 × 每题 | 节点时长 | 余量 |
|---|---|---|---|
| Hallucination | 3 × 20s = 60s | **66s** | 6s |
| 每个 quiz | 2 × 20s = 40s | **46s** | 6s |

> ⚠️ **改题目数量或 `time_limit` 时，必须同步改 `flow.py` 的 `duration_sec`。** 目前这两处是**手工保持一致的**，没有代码校验。题库搬到后端后，应该让后端从题目数据算出节点时长，而不是继续手写。

## 当前状态与限制

- **题目硬编码在前端**（`content.ts`）：2 道，都与 hallucination 主题相关。
- **纯前端**：答案不落库、不上报、不做全场聚合。`scores` 表仍是零读写。
- **三个 quiz 节点共用同一份题目**——所以现在第二轮的 Quiz round 2 会和第一轮**出一样的题**。等题库落地后按节点取不同题集。
