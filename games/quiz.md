# Quiz

主流程里 `type: quiz` 的节点**只剩最后一个**：Ultimate Quiz（Hallucination 和 Data Shadow 后面的两轮 quiz 已删除）。
代码：`arcade_ftw_fe/src/games/quiz/`。

节点靠 `key={nodeIndex}` 挂载；题库只有一份（`QUIZ_QUESTIONS`），因为只有一个 quiz 节点，不再有「按 section 选 deck」这层。

**没有开场卡**：节点不带 `intro_sec`，第一秒就是第一题。它前面的 15s `brief`（How the Ultimate Quiz works）已经完成了介绍，所以这一轮也没有 5s 名字卡。

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

五道题就是五个独立的 20 秒，不是「整个节点一个总倒计时」。

- **倒计时跟随引擎暂停**（这是实测抓到的 bug）。裸 `Date.now()` 会让游戏内的倒计时在暂停期间继续走，题被烧掉。现在从引擎状态推导（`useNodeElapsed`）：`remainingMs = endsAt ?? heldMs` 相减得到已用时间——`endsAt` 与 `heldMs` 本来就是暂停感知的，所以不需要第二个时钟。
- 计时锚定在 `questionStart`（切题时的**已用时间快照**），由 `useTicker` 每 250ms 触发重渲染。
- **超时用派生值，不用 state**：`expired = picked === undefined && timeLeftMs === 0`。曾经用 `useEffect` + `setState` 实现，被 oxlint 的 `react(set-state-in-effect)` 抓出来——那是多余的额外渲染，且状态可以推导就不该存储。
- 超时**不会卡住节目**：直接进入揭晓并显示 `Time's up`，避免一块没人操作的屏幕让整场演出停摆。

## ⚠️ 节点时长必须 ≥ 题目总时长

引擎按 `flow.py` 的 `duration_sec` **自动推进**，到点就切下一个节点。所以节点时长必须覆盖「所有题目的计时之和 + 揭晓收尾」，否则引擎会在玩家答完之前把游戏切走。

当前配置：

| 游戏 | 题目数 × 每题 | 节点时长 | 余量 |
|---|---|---|---|
| Hallucination | 3 × 30s = 90s | **96s** | 6s |
| Ultimate Quiz | 5 × 25s = 125s | **130s** | 5s |

> ⚠️ **改题目数量或 `time_limit` 时，必须同步改 `flow.py` 的 `duration_sec`。** 节点时长本身有测试守住（`tests/test_flow.py::test_question_nodes_outlast_their_questions`，要求 `duration_sec > intro_sec + questions × seconds_each`），但「题库里有几道题」和「节点声明几道」仍是两处手写，改题库时两边都要动。

## 当前状态与限制

- **题目硬编码在前端**（`content.ts`）：5 道，全部是港/监管 + AI 主题（PCPD 框架、PDPO 六原则、生成式 AI 隐私、深伪付款、引用核查）。
- **纯前端算分、后端落库**：每答一题就 `POST /api/scores` 上报，`section` 由节点的 `score` 决定（`ultimate`），所以名额落进 `ultimate_score` 列。
- **分数上限**：5 题 × 20 = **100**。结算页的满分是从题数推导的，加题不用改这两处。
