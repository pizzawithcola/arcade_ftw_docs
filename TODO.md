# Arcade FTW 待办清单

> 唯一共享的任务清单。完成一项就勾掉并回补相关文档；新增项请写清**范围**与**验收标准**。
>
> 最后更新：2026-11-16（新增浅色主题计划，见 `architecture/theming.md`）

## 基线（已完成，不在待办内）

- 引擎自主推进 + 四态状态机、按房间隔离
- 房间 / 玩家 / 分数数据化（`rooms` / `players` / `scores` / `score_events` / `room_participants`）
- **房间一次性生命周期**：`finish` 关闭房间（`closed_at`）并清空成员绑定；当前轮 = 唯一未关闭房间；参与历史独立留存
- 三条 UI 路线：玩家 `/play`（跟随当前轮）、大屏 `/display`、管理台 `/admin`（极简控制台：无房间选择器）
- 四个真实玩法：Hallucination、Data Shadow、Sabotage、Ultimate Quiz
- **Sabotage v2**：四轮攻防、40 张卡、共享预算与服务端结算（见第 2 项）
- **分数上报**：玩家令牌 + 按题幂等，四个分区自动累加
- 大屏题面与大屏等待室、引擎下发当前题号
- **榜单**：公开总榜 `/leaderboard`（前 10）+ 游戏内 section 榜（每个 section 结束后 5 秒，大屏+平板）+ 最终榜（`section: total`，垒箱子动效）
- AWS 双栈部署、OIDC 零密钥 CI、CloudFront 同源

## 0.5 浅色主题（计划，未动工）

完整计划见 [`architecture/theming.md`](./architecture/theming.md)。摘要：

- 开关在后端 `app_settings.theme`（`dark` 默认 / `light`），全场统一；改完广播所有在线屏幕。
- 浅色只在 `[data-theme='light']` 作用域下覆盖 token，**深色样式表一行不改**。
- 动工前先补 9 个 token、替换 40 处硬编码颜色、解决「光晕/红底文字/`--c-black` 当暗色」三类问题。
- 顺序：P0 地基 → P1 玩家侧 → P2 大屏侧 → P3 游戏内页 → P4 控制台（Battle 最后，可能不做）。
- 验收单位是**页面**，每页改完与深色前后截图比对；e2e 继续跑在深色下。

**下一步**：确认后从 P0 开始。

## 1. 游戏分数接线（已完成）

**目标**：玩家答完题分数自动落库，排行榜不再依赖管理员手改。

**评分结构**（四列 = 四个分区，每个分区现在只有一个节点在往里写）：

| 分区（`scores` 列） | 计入的节点 |
|---|---|
| `hallucination_score` | Hallucination |
| `data_score` | Data Shadow |
| `sabotage_score` | Sabotage |
| `ultimate_score` | Ultimate Quiz |

> Hallucination 与 Data Shadow 后面的两轮 quiz 已删除，所以这两列不再由「游戏 + quiz」累加。

**设计要点**：

- 节点在 `flow.py` 里声明 `score: <分区>`，服务端据此决定写哪一列；**前端不能指定列**。
- 玩家用**自己的令牌**上报（新增 `POST /api/me/score`），不再借用管理员接口。
- **按题幂等**：上报键 = `房间:节点序号:题号`（整节点提交用 `total`）。同键重复上报只生效一次，断网重试、页面重渲染都不会重复计分。
- 分数可为负（Hallucination 质疑侧、Data Shadow 误报），前端上报 0 分时直接跳过。

**验收标准**：

- [x] Hallucination、Data Shadow、Ultimate Quiz 答完（或超时结算）后，`/api/players?room=` 能看到对应列增加
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

## 2. Sabotage 玩法（已完成）

当前玩法是四轮 v2：40 张牌（每轮 5 攻 + 5 守），红蓝两队各有四轮共用的 `$100,000` 预算。玩家私下提交最多 4 张牌，服务端聚合队伍选择并按 Data / People / Infra 三维向量结算。

- 主流程 Sabotage 节点为 163 秒；Sabotage 子引擎正常运行 160 秒（10 秒分队揭晓 + 15 秒规则 + 首轮 35 秒选卡 + 后三轮各 25 秒 + 四次 5 秒揭晓 + 5 秒终局）。
- 自动分队、投票、暂停/恢复、服务端团队计分、单人房跳过均已接入；单人房的 `SOLO_SCORE` 为 800。
- 玩法规则与当前配置见 [`games/sabotage-v2.md`](./games/sabotage-v2.md)；旧版 6×6 矩阵只作历史记录，见 [`games/sabotage.md`](./games/sabotage.md)。

## 3. 题库后端化

**目标**：题目不再硬编码在前端，改题不需要发版。

- [ ] 建题库表与 `GET /api/questions?node=`（或按房间/轮次下发）
- [ ] 后端按题目数据**自动推导节点时长**，消除 `duration_sec` 与题目数量手工同步的隐患
- [x] ~~三个 quiz 节点取不同题集~~ —— quiz 节点只剩 Ultimate 一个，当前从 **20 题题库按三个 slot 各抽 1 题**（`draw.ts`；100/200/300，满分 600）
- [x] 题库只有一份（`bank.ts`），旧的多 deck 结构与 `content.ts` 已删除

## 4. 题型注册表接线

**目标**：`app/question_types/` 真正参与运行，而不是只有定义。

- [ ] 引擎/WS 按题型 `render()` 下发题目
- [ ] 实现 `trust_doubt`（Hallucination）与 `interactive`（Data Shadow）
- [ ] 与第 1 项合并考虑：答案上报后由题型 `score()` 计算，替代前端计分

## 5. Game Setting 分区

**现状**：`Game Setting` 已只读展示 Ultimate Quiz 的抽题规则与完整题库；编辑设置尚未实现。

- [ ] 节点时长可视化调整（依赖第 3 项的后端流程配置）
- [ ] 活动预设（`event` 列表、房间初始配置）

## 6. AWS 收尾

- [ ] `SecretKey` / `AdminPassword` 迁到 Secrets Manager（现在在任务定义明文 env）
- [ ] 自定义域名 + ACM 证书（现在用 `*.cloudfront.net`）
- [ ] 评估两个 repo 是否转 private

## 已知取舍与红线（不要"顺手优化"）

- **宽表计分**：四列 + 数据库生成列 `total_score`，加分区需要迁移。这是已接受的决策。
- **引擎是进程内内存态**：单进程、禁止多 worker、活动期间禁止部署。
- **幂等键含节点序号**：admin 回退重跑同一节点时，同一题不会二次计分。
- **玩家上报分数由客户端计算**：服务端目前没有题目数据，无法独立核算；题目后端化（第 3 项）后才可能改成服务端计分。
