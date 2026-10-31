# Quiz

主流程里 `type: quiz` 的节点**只剩最后一个**：Ultimate Quiz（Hallucination 和 Data Shadow 后面的两轮 quiz 已删除）。
代码：`arcade_ftw_fe/src/games/quiz/`。

节点靠 `key={nodeIndex}` 挂载；题库只有一份（`QUESTION_BANK`），因为只有一个 quiz 节点，不再有「按 section 选 deck」这层。

**没有开场卡**：节点不带 `intro_sec`，第一秒就是第一题。它前面的 `brief`（How the Ultimate Quiz works）已经完成了介绍，所以这一轮也没有名字卡。

## 玩法

4 选项单选题（A/B/C/D），每题**各自独立计时**，答完立刻揭晓：正确项标绿、错误选项标红，并给出**为什么**（`explain`）——让答对的人也知道自己为什么对。

这一轮**五道题，不是一份固定卷子**：每个位置从题库的某个分类里抽一道，两个 bonus 位置在后面。

| # | 抽自 | 池子 | 答对 |
|---|---|---|---|
| 1 | Data Shadow + Phishing email | 5 题 | **+100** |
| 2 | Hallucination | 5 题 | **+100** |
| 3 | CEO impersonation（Sabotage R2）| 3 题 | **+100** |
| 4 | Assistant and sources（R3 + R4）| 4 题 | **+200**（`×2`）|
| 5 | Hong Kong | 5 题 | **+300**（`×3`）|

满分 **800**（`perfectScore()` 从 slot 表推导，改 slot 不用改别处）。选错或超时 0 分。

**第 1 位把 Data Shadow 和钓鱼邮件合成了一个池子**：两者都是「信息在别人的指令下离开公司」，
所以共用一道题。代价是这一池里 3/5 是钓鱼题——**单局有 40% 的概率不考 Data Shadow**，这是合并
的固有结果，不是 bug。

**两个 bonus 故意放最后**：这一轮越往后越难、越值钱，落后的队伍还有得追。
前三题值 300，后两题值 500——最后一题一道顶三题。

## 抽题为什么必须是确定性的

平板和大屏各自算题。如果用 `Math.random()`，两台设备会抽出**不同的五道题**，玩家会对着
大屏背后那道题之外的另一道题作答。

所以 `draw.ts` 用 `room:nodeIndex` 当种子的 PRNG（FNV-1a → mulberry32），**同一房间同一节点
必然抽出同样的五道题**：

- 大屏和平板一致——这是唯一能让「屏幕上是哪道题」对得上的做法；
- 排练时来回跳节点不会重新洗牌；
- 两个房间之间仍然不同（4000 个房间实测出 2690 种不同组合，各分类 min/max 偏差 ≤22.5%）。

抽题时用 `used` 集合跳过本轮已经问过的题。目前五个池子互不重叠，但某道题以后换分类时
不能让它在一轮里出现两次。

## 字段命名沿用旧系统的题库 schema

`text` / `option_a`~`option_d` / `correct_option` / `time_limit` 与旧 Ultimate Showdown 的 `QuestionBase` 一致。

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
| Hallucination | 3 × 25s = 75s | **78s** | 3s |
| Ultimate Quiz | 5 × 25s = 125s | **128s** | 3s |

> ⚠️ **改题目数量或 `time_limit` 时，必须同步改 `flow.py` 的 `duration_sec`。** 节点时长本身有测试守住（`tests/test_flow.py::test_question_nodes_outlast_their_questions`，要求 `duration_sec > intro_sec + questions × seconds_each`），但「抽几道」和「节点声明几道」仍是两处手写，改 slot 表时两边都要动。

## 题库与 admin 展示

| 文件 | 内容 |
|---|---|
| `src/games/quiz/bank.ts` | **22 题题库**，按四个游戏写的，7 个分类 |
| `src/games/quiz/draw.ts` | slot 表 + 确定性抽题 |

题库不是按知识点分类，而是**按游戏分类**——每个分类对应观众刚刚玩过的那一局：

| 分类 | 题数 | 对应 |
|---|---|---|
| `data-shadow` | 2 | Data Shadow：什么算泄漏、什么能贴进 AI |
| `hallucination` | 5 | Hallucination：答案怎么坏、怎么验 |
| `phishing-email` | 3 | Sabotage R1（钓鱼邮件）；与 `data-shadow` 共用第 1 个 slot |
| `ceo-impersonation` | 3 | Sabotage R2（CEO 冒充）|
| `ai-assistant` | 2 | Sabotage R3（操纵助手）|
| `knowledge-base` | 2 | Sabotage R4（污染知识库）|
| `hk` | 5 | 香港本地法规与实务（PCPD、PDPO），只进 bonus 池 |

**每道题都带 `explain`**，并且尽量回指游戏——「这是 round 1 牌库里最便宜的那张卡」比
抽象的「钓鱼邮件有紧迫感」更能让观众想起来自己刚玩过什么。

答案分布刻意打散：A5 / B6 / C5 / D6。相邻同答案的问题已经不存在了——一轮五道题各来自不
同的池子，抽题顺序不再读题库文件的顺序。

**正确答案不能是最长的选项**：改前 20 题里有 11 题正确答案明显最长（最大差 37 字符），
玩家会学会「挑最长的念」。现在 0 题超标，长度差控制在 15 字符以内。

**为什么在仓库里而不是数据库里**：旧系统把题存在数据库，只能通过 API 看到，没有 diff、
没有 code review，这次是靠旧 ALB 还活着才捞回来。22 题的规模，硬编码在前端换来的是构建期
类型检查（选项少一个、答案写成 E、id 重复都编译不过）和零网络往返。

**admin 展示**：管理台 → `Game Setting`（`AdminPage.tsx` 的 `SettingsView`），两张表：
上面是「how it draws」——五个位置各自从哪个分类抽、池子多大、值多少分；下面是 22 题的
题库全表（分类徽章 + 加粗的正确答案 + `explain` + 时长）。表格只读，改题仍然改 `.ts` 文件后重新部署。

## 当前状态与限制

- **题目硬编码在前端**：`bank.ts` 22 道，分 7 个分类；这一轮从里面抽 5 道。
- **纯前端算分、后端落库**：每答一题就 `POST /api/scores` 上报，`section` 由节点的 `score` 决定（`ultimate`），所以名额落进 `ultimate_score` 列。
- **分数上限**：`perfectScore()` = **800**。结算页的满分是从 slot 表推导的，加 slot 不用改这两处。
