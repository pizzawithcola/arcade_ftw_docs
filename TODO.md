# Arcade FTW 待办清单

> 唯一共享的任务清单。完成一项就勾掉并回补相关文档；新增项请写清**范围**与**验收标准**。
>
> 最后更新：2026-10-22（分数接线完成）

## 基线（已完成，不在待办内）

- 引擎自主推进 + 四态状态机、按房间隔离
- 房间 / 玩家 / 分数数据化（`rooms` / `players` / `scores`）
- 三条 UI 路线：玩家 `/play`（跟随当前轮）、大屏 `/display`、管理台 `/admin`
- 三个真实玩法：Hallucination、Quiz、Data Shadow
- **分数上报**：玩家令牌 + 按题幂等，四个分区自动累加
- 大屏题面与大屏等待室、引擎下发当前题号
- AWS 双栈部署、OIDC 零密钥 CI、CloudFront 同源

## 1. 游戏分数接线（已完成）

**目标**：玩家答完题分数自动落库，排行榜不再依赖管理员手改。

**评分结构**（四列 = 四个分区，游戏与其后的 quiz 累加到同一列）：

| 分区（`scores` 列） | 计入的节点 |
|---|---|
| `hallucination_score` | Hallucination + Quiz round 1 |
| `data_score` | Data Shadow + Quiz round 2 |
| `sabotage_score` | Sabotage |
| `ultimate_score` | Ultimate Quiz |

**设计要点**：

- 节点在 `flow.py` 里声明 `score: <分区>`，服务端据此决定写哪一列；**前端不能指定列**。
- 玩家用**自己的令牌**上报（新增 `POST /api/me/score`），不再借用管理员接口。
- **按题幂等**：上报键 = `房间:节点序号:题号`（整节点提交用 `total`）。同键重复上报只生效一次，断网重试、页面重渲染都不会重复计分。
- 分数可为负（Hallucination 质疑侧、Data Shadow 误报），前端上报 0 分时直接跳过。

**验收标准**：

- [x] 三个游戏答完（或超时结算）后，`/api/players?room=` 能看到对应列增加
- [x] 同一题重复上报不重复加分（幂等）
- [x] 未加入该房间的玩家上报被拒绝（409）
- [x] 无 `score` 声明的节点上报被拒绝（422）
- [x] 后端测试覆盖：正常累计、幂等、跨房间拒绝、未声明节点拒绝
- [x] 文档同步：be README 路由表、`architecture/views.md`、`SYSTEM_HANDBOOK.md`

**实现落点**：

- `app/flow.py`：计分节点声明 `score` 分区
- `app/scores.py`：`POST /api/me/score`（玩家令牌）
- `app/models.py`：`ScoreEvent`（`(player_id, event_key)` 主键）
- `arcade_ftw_fe/src/hooks/useScoreReport.ts`：按题去重 + 一次重试
- 迁移 `5f2d983893bf`

**遗留**：分数仍由客户端计算（后端没有题目数据）。题目后端化后改成服务端计分，见第 3、4 项。

## 2. Sabotage 玩法

**目标**：第 6 个节点从占位屏变成真实游戏。完整设计拆解见 [`games/sabotage.md`](./games/sabotage.md)。

**核心结论**：

- 玩法：红蓝三轮攻防**投票**，多数票=全队唯一行动；攻击卡共享单次使用，防守卡每轮复用。
- **计分**：整局结算出**一个团队总分**，队内每个竞技成员**共享同一个数值**写入 `sabotage_score`（不均分；服务端统一写库，玩家不上报）。
- **效果矩阵**：6×6 已定稿（blocked / partial / failed 三档）；胜利方 payout +100。
- **时长**：约 **2.5 分钟（150s）**，需重构节点（10s 揭晓 + 6×20s 投票 + 结算），**先改文档不改代码**。
- **双向通信**：投票走 **HTTP + 玩家令牌**（复用 `/api/me/score` 模式），WS 保持单向只广播结果。
- **Learning**：本期不做，默认全员 competing。
- **风险点**：引擎现在「1 节点 = 1 时长」，Sabotage 有 ~12 个子阶段；倾向**拆 flow 节点**而非改核心引擎。

**M0 设计定稿** ✅

- [x] 玩法规则、6×6 效果矩阵、blocked/partial/failed 计分、payout（胜利方 +100）
- [x] 双向通信评估（HTTP 投票 + WS 广播）
- [x] 可配置项清单（见 `games/sabotage.md`）

**M1 团队分配** ✅

- [x] `players.team`（red/blue）+ 迁移 `7b3e4d8f1a2c`
- [x] `POST /api/rooms/{code}/teams/assign`：admin 触发，按 id 交替均分、可重跑、幂等
- [x] 名单 / `GET /api/auth/me` 带 team；玩家只见自己队，admin/display 见全员
- [x] join 进新房间清空 team
- [x] 测试：均分、奇数、重跑确定性、名单裁剪、me 带队、换房清队、鉴权（7 个）

**M2–M5 待办**

- [ ] M2 实现 `app/sabotage.py` 房间级状态机（T4_INIT → … → T5_TRANSITION）+ 投票接口 `POST /api/sabotage/{room}/vote`
- [ ] M3 拆 flow 节点（2.5 分钟）+ WS 广播 sabotage 子阶段
- [ ] M4 实现 `arcade_ftw_fe/src/games/sabotage/` + 大屏
- [ ] M5 团队结算写库（每队员同一 `sabotage_score`）+ E2E + 文档状态翻转

## 3. 题库后端化

**目标**：题目不再硬编码在前端，改题不需要发版。

- [ ] 建题库表与 `GET /api/questions?node=`（或按房间/轮次下发）
- [ ] 后端按题目数据**自动推导节点时长**，消除 `duration_sec` 与题目数量手工同步的隐患
- [ ] 三个 quiz 节点取不同题集（现在 round 1 / round 2 / ultimate 出同一套题）
- [ ] 前端 `content.ts` 降级为测试 fixture

## 4. 题型注册表接线

**目标**：`app/question_types/` 真正参与运行，而不是只有定义。

- [ ] 引擎/WS 按题型 `render()` 下发题目
- [ ] 实现 `trust_doubt`（Hallucination）与 `interactive`（Data Shadow）
- [ ] 与第 1 项合并考虑：答案上报后由题型 `score()` 计算，替代前端计分

## 5. Game Setting 分区

**目标**：管理台第三块从占位变成可用。

- [ ] 房间默认值（event / label 预设）
- [ ] 节点时长可视化调整（依赖第 3 项的后端流程配置）

## 6. AWS 收尾

- [ ] `SecretKey` / `AdminPassword` 迁到 Secrets Manager（现在在任务定义明文 env）
- [ ] 自定义域名 + ACM 证书（现在用 `*.cloudfront.net`）
- [ ] 评估两个 repo 是否转 private

## 已知取舍与红线（不要"顺手优化"）

- **宽表计分**：四列 + 数据库生成列 `total_score`，加分区需要迁移。这是已接受的决策。
- **引擎是进程内内存态**：单进程、禁止多 worker、活动期间禁止部署。
- **幂等键含节点序号**：admin 回退重跑同一节点时，同一题不会二次计分。
- **玩家上报分数由客户端计算**：服务端目前没有题目数据，无法独立核算；题目后端化（第 3 项）后才可能改成服务端计分。
