# Arcade FTW 系统手册（跨 session 共享 memory）

> ⚠️ **开工前请先完整读完本文件。** 本文件是 `arcade_ftw` 的权威来源，自包含，不依赖会话记忆。
>
> - 生成日期：2026-10-11
> - 最近同步：2026-10-19（文档对齐当前代码：三个真实游戏、房间/玩家/分数、等待室与题目索引）
> - 上一代系统（`cyber_ai_festival` / `cyber_ai_festival_be`）的权威文档在其前端 repo 的 `doc/SYSTEM_HANDBOOK.md`；**本文件不复制它的内容**，只引用。

---

## 1. 定位

Arcade FTW = 第二个版本的游戏。与上一代系统 **完全平行、完全隔离**，但继承其运维经验与红线。

玩法：把多个单人游戏串成一条主线，中间穿插 quiz，**所有玩家同步推进**；题型不只是选择题（含 trust/doubt、互动题）；管理员可暂停 / 结束 / **回退到任意节点**。

## 2. 仓库布局（polyrepo）

| repo | 技术栈 | 状态 |
|---|---|---|
| `arcade_ftw_fe` | React 19 + TypeScript + Vite + zustand | 已上线，迭代中 |
| `arcade_ftw_be` | FastAPI + Python 3.12 | 已上线，迭代中 |
| `arcade_ftw_docs` | 本 repo | — |

**为什么 polyrepo**：见 [architecture/decisions](./architecture/decisions)。CI 天然按 repo 隔离，不存在 monorepo「前端提交误触发后端部署 → 清空内存态房间」的风险。

## 3. 当前状态

主线可以完整跑通：登录 → 等待室 → 引擎自动推进 **13 个节点**（开场 Welcome、3 个 15s `opening`（3s 名字 + 12s 规则）+ Sabotage 节点内自带的 15s 规则卡、3 个 section 榜单 + 总榜、收尾）→ 结束，全长 **9m09s**；四个玩法界面已有真实实现（Hallucination / Quiz / Data Shadow / Sabotage），房间 / 玩家 / 分数 / 榜单已数据化，前后端已部署到 AWS。

已完成：

- 前端：React 19 + Vite + TS + zustand；五条 UI 路线（登录 / 玩家 / 大屏 / 管理台 / 公开榜单）；四个真实玩法（Hallucination、Quiz、Data Shadow、Sabotage）；等待室、跟随当前轮、题目索引与大屏题面、游戏内 section 榜单（大屏 + 平板）与公开总榜
- 后端：FastAPI 入口（`/health`、`/health/db`、`/api/*`）、按房间隔离的状态机（`app/engine.py`）、流程配置（`app/flow.py`）、WS 推送（`app/ws.py`）、房间 / 玩家 / 分数接口、HMAC 令牌鉴权
- 数据：PostgreSQL + Alembic，`rooms` / `players` / `scores` / `score_events` / `room_participants`
- 运维：CloudFormation 双栈、OIDC 零密钥 CI、CloudFront 同源部署

未完成：题库后端化与题型注册表接线、逐题答案存储与大屏聚合、AWS 剩余项（见第 6 节与 [TODO](./TODO.md)）。

## 4. 与上一代系统的关系

| 事项 | 结论 |
|---|---|
| 代码 | **不迁移**旧游戏代码；旧系统冻结 |
| 可复用 | UI 组件库 / 主题 / 部署管道模式 |
| AWS | **隔离优先**：除 VPC/子网外全部新建，资源名用 `arcade-ftw-` 前缀 |
| 数据库 | 全新实例，不碰旧库 |
| 文档 | 本 repo 为权威；旧系统文档不动 |

## 5. 红线与踩坑（沿用上一代教训）

1. **现有 repo 默认只读**——除非用户明确要求，不动 `cyber_ai_festival` / `cyber_ai_festival_be`。
2. **端口 1688 绝不可碰**（旧系统本地 dev 服务）；本项目 dev 端口为前端 **5070** / 后端 **5071**——验证或起临时服务时都不要占用它们，另起 169x 等空闲端口。
3. **WS 是进程内内存态**——单进程运行，禁止多 worker；换 task 会清空房间，活动期间禁止重启后端容器。
4. **后端镜像必须 `--platform linux/amd64`**（Mac arm64 直 build 会得到 arm64 镜像，Fargate 起不来）。
5. **CloudFront 的 SPA 回退会把「不存在的对象」变成 `200 + index.html`**——本系统只在 **S3 行为**上启用该函数，`/api/*` 的错误保持真实状态码；但任何「只看状态码」的健康检查仍可能被骗，判定成功必须校验 `content-type`。
6. **游戏接口一律挂在 `/api/*` 下**——CloudFront 用一条 `/api/*` 行为代理到 ALB，路径结尾带不带斜杠都能命中（前端实际都用裸路径，如 `/api/rooms`）；**根路径没有反向代理**，漏掉 `/api` 前缀会落到 S3。
7. 凡用 storage 做「跨刷新保持」的状态，必须列出所有**生命周期结束点**并逐个清理（旧系统踩过跨局残留）。
8. **依赖要 pin 版本**（旧系统因未 pin 被新版 SQLAlchemy 引爆过）。

## 6. 待办 / 未决

- [ ] `SYSTEM_HANDBOOK.md` 在旧系统里叫 "香港 FTW"，本项目管理名已改为 `arcade_ftw`——旧文档 Part B 已过时（需用户同意才能改旧 repo）
- [ ] `packages/shared`（机器可读的协议类型）尚未建；按旧手册 §11，起步用「TS 类型 + Pydantic 手写对应」，稳定后升级 JSON Schema
- [x] 后端 IaC（CloudFormation）已部署：栈 `arcade-ftw`（`ap-south-1` / `866730904414`），含 OIDC provider + 部署角色
- [x] 后端**已对外提供服务**（经 CloudFront）：`https://d39s9y5ma27s5.cloudfront.net`，`/health` 200、`/health/db` 200、`/api/ws` 通；ALB 已不对公网开放
- [x] CI 已接通并实测成功（OIDC，零密钥，一次部署约 4.5–5 分钟，含数据库迁移）
- [x] 数据库链路已打通：RDS `arcade-ftw-db` + Alembic，迁移在 CI 里于滚动更新前执行
- [x] 前端部署：S3 + CloudFront（站点与 API 同源），前端 CI 走 OIDC，真实浏览器验证三项全绿
- [x] **垂直切片已上线并实测**（2026-10-11）：登录（昵称 / 管理员）→ 开播 → **自动推进** → 广播到所有客户端 → 前端按节点渲染 + 倒计时（当时的占位屏已由真实游戏替换）；管理员可 pause / resume / finish / 跳节点
- [ ] AWS 剩余项（Secrets Manager 保存 `SecretKey` / `AdminPassword`、自定义域名 + ACM）见 [aws/](./aws)
- [x] 前端已有三个**真实玩法界面**：Hallucination（五档 trust/doubt）、Quiz（四选一）、Data Shadow（文本拖选）；节点壳按 `game` / `type` 分派
- [x] **Sabotage v2 已实现**（2026-10-30）：四轮、每轮独立 5 张牌库、三维向量（Data/People/Infra）+ 单目标判定、队伍组合投票聚合、抽队伍移到 Intro、每轮 10 秒结算、终局四轮复盘，见 [games/sabotage-v2.md](./games/sabotage-v2.md)；v1 的 6×6 矩阵存档在 [games/sabotage.md](./games/sabotage.md)
- [x] **Sabotage 共享预算 + 40 张卡定稿**（2026-10-30 晚）：卡表从 48 张收到 **40 张**（每轮 5 攻 + 5 守），预算从「每轮 100 点」改成**四轮共用的 $100,000**，卡价换成与真实投入挂钩的 $1,000–$14,000。四轮打满要 $108,000（RED）/ $107,000（BLUE），所以**必然有一轮买不满**——这是玩法核心。设计稿 [games/sabotage_cards.xlsx](./games/sabotage_cards.xlsx)，代码 `sabotage_config.py` 的 `TOTAL_BUDGET` / `CARDS` / `ROUNDS`，测试守四条不变量（攻守差 1、单轮 25k–30k、四轮超预算、价格倒挂）。选卡时长 R1 50s / 其余 35s，节点 250s
- [x] **Sabotage 单人房跳过**：1 个玩家时分不成两队，主流程直接越过该节点，该玩家拿 `SOLO_SCORE`（2700，完美获胜分），不再干等 170 秒
- [x] **Sabotage 队伍同步与重分队**：玩家设备进节点后轮询 `GET /api/auth/me` 拿自己的队；`go/{index}` / `reset` 先丢弃旧引擎，每次进节点 5 都按当前玩家重新随机分队、从头重跑（排练可反复快进/回退）
- [x] **Sabotage 结算文案按身份**：玩家平板显示自己阵营的结果（攻成/攻败、守成/守破），大屏显示本轮赢家（RED/BLUE WINS THE ROUND）
- [x] **游戏内 section 榜单**：Hallucination / Data Shadow / Sabotage 后面各插一个 5 秒 `leaderboard` 节点；Ultimate 后面**没有**分节榜——它的分数直接进 10 秒活动总榜，避免同一批数字连播两次。榜单按**累计总分**排序（不是该 section 单项），每行同时返回 `previous` / `score` / `total`，所以结算屏能演出「旧榜静置 1s → 数字滚动 → 卡片换位」；`GET /api/rooms/{code}/leaderboard?section=` 供两端读取
- [x] **每题固定周期**：Hallucination 20s 抢答 + 5s 判决 × **3 题**（节点 78s，末 3s 是总分页），Ultimate Quiz 20s + 5s × **5 题**（节点 128s，第 4 题 ×2 / 第 5 题 ×3，末 3s 是结算页）；判决页 5 秒后自动跳题，**没有 Next 按钮**。Ultimate 不带 `intro_sec`，第一秒就是第一题
- [x] **每个游戏一个 15s `opening`**（2026-10-31）：原来的 5s `title` 名字卡 + 15s `brief` 规则卡合并成**一个节点**，前 `title_sec: 3` 秒是名字卡、后 12 秒是规则，规则三条由节点时钟逐条浮现。`title` / `brief` 两个类型已从 flow 删除，`titles.ts` + `briefs.ts` 合成 `games/openings.ts`。Sabotage 仍然没有 flow 层开场（分队后游戏内自带 15s 规则卡）
- [x] **开场 Welcome（10s）**：标题 + 一句话 + 四张游戏卡逐张从下方浮现，全部动画在**第 5 秒**结束（最后一张卡 3.9s 起、4.78s 落定，页脚 4.2s），剩 5 秒读卡片。由节点时钟驱动（暂停会冻结在半途），大屏与 iPad 同一套
- [x] **题库重写为 25 题 + 每轮抽 6 题**（2026-10-31）：`arcade_ftw_fe/src/games/quiz/bank.ts` 改为**按游戏分类的 25 题**（Data Shadow 5 / Hallucination 5 / Phishing 3 / CEO 3 / Assistant 2 / Knowledge base 2 / HK 5），每题带 `explain` 并回指游戏，答案 A6/B6/C7/D6 且无相邻同答案、无「正确答案最长」偏差。`draw.ts` 用 `room:nodeIndex` 作种子的 PRNG 抽 6 题（大屏与平板必须一致），第 4 题 ×2、第 5 题 ×3，满分 800。管理台 `Game Setting` 两张表展示「how it draws」与 25 题全表，见 [games/quiz.md](./games/quiz.md)
- [x] **四款游戏统一到 3000 分**（2026-10-31）：Hallucination **600**（信任阶梯改成对称的 ±200/±100/0，每题上限 200——旧阶梯是 −10…+20，导致假话题只值真话题一半、整轮真实满分只有 40）、Data Shadow **800**（权重 ×8：4×120 + 4×80，误报 −80）、Sabotage **800**（每轮恒定 200 分池，攻方 `40 × (侧翼 + 3×突破)`）、Ultimate **800**（每题基数 20→100 → 100/100/100/200/300）。Sabotage 占比从 **88.6% 降到 26.7%**，同队成员之间第一次能分出高下，最后一轮也真的能翻盘
- [x] **Sabotage 每轮结束改成说话**（2026-10-31）：轮次 × 维度 × 四视角共 **48 句**（`games/sabotage/revealCopy.ts`），每侧只看到自己那 3 句（5–8 词），大屏保留中立数字并新增一句「这一轮赌的是什么」（四句）。弃权一律算守方赢（双方都弃权也从 0–0 平局改成守方拿满池），删掉 `No bids` / `bought no cards` 两句特例文案
- [x] **R2 的两条侧翼救活**（2026-10-31）：CEO 冒充轮的 Data 原本 92% 被攻穿、Infra 只有 0/1 两种值（56% 持平）。调整 10 张卡的向量（不动价格、不动 target、不动牌数，因为 39 张图与 40 个卡位一一对应）后：Data 33/29/38、Infra 53/23/24，六种轮级结果分布从「一枝独秀 43%」变成 23/22/20/17/11/6。测试新增「被单一结果决定超过 75% 的列」断言（目前只覆盖 R2）
- [x] **端到端测试套件**（2026-10-31）：`arcade_ftw_fe/e2e/` 两套，一条命令 `npm run test:e2e`（46 条）。`framework.test.mjs` 走真后端 + 前端代理的 HTTP/WS：13 节点与 549 秒、引擎的暂停/跳转/结算、四个 section 榜、Sabotage 分队与阶段、单人跳过、房间生命周期、静态资源；`ui.test.mjs` 用本机 Chrome（`playwright-core`，不下载浏览器）验屏幕：13 个节点都不落到 placeholder、欢迎页四卡是「编号+图标+名称」、opening 第 3 秒交棒且只 3–4 点、大屏与平板同一道题、第 5 题 ×3 与结算 `/800`、控制台表格、玩家登录进等待室。测试自建房间/玩家并自删。顺带修掉一个真实风险：`/api/rooms/current` 跟「最新未关闭房间」，而 `pytest` 曾留下 46 个未关闭测试房，会让玩家进错房间——现在 `conftest.py` 退出时清理
- [x] **一轮 quiz 题库**：Hallucination 与 Data Shadow 后面的两个 quiz 节点已删除，只剩 Ultimate Quiz；题库在 `src/games/quiz/bank.ts`（22 题），每轮抽 5 题，改 slot 表必须同时改 `flow.py` 的 `questions` / `duration_sec`（`tests/test_flow.py` 守住时长下限）
- [x] **Data Shadow 80s**（节点 86s），原来 50s 读不完段落
- [x] **大屏题号由本地时钟推导**：WS 只在题目变化时带 `question`，直接读最后一次推送会让大屏整节点停在第一题
- [x] **公开总榜**：`/leaderboard` 页面（前 10 名，自动刷新，纵向放不下时才变两列），主页有入口；`GET /api/leaderboard` 公开无鉴权，只暴露 rank / nickname / total_score
- [x] **总榜 placeholder**：Ultimate 榜之后一个 `section: "total"` 节点，先显示占位文案，设计待定
- [ ] 题型注册表仍只有 `choice` 一个示例，且**尚未被引擎 / WS 调用**；题库无存储
- [x] 数据库：`rooms` / `players` / `scores` / `score_events` / `room_participants`；房间是一次性数据（code + 创建时间即全部身份），当前房间绑定记在 `players.room_code`，参与历史记在 `room_participants`
- [x] 房间 / 玩家 / 分数的接口（`/api/rooms`、`/api/players/{id}`、`/api/players/{id}/score`）；admin 面板可开新一轮
- [x] **room = 一轮游戏，一次性**：admin 新建 `R-001`，`finish` 关闭房间（写 `closed_at`、清空成员绑定），下一轮必须新开；大屏 `/display` 与玩家 `/play` **都自动跟随当前轮**（最新未关闭房间）
- [x] 玩家不扫码：登录后在 `/play` 跟随当前轮；本轮 `finished` → 收尾屏 8 秒后回登录页
- [x] 大屏展示题目（索引由引擎按 `questions`/`seconds_each` 算出），选项只在玩家屏
- [x] 大屏空闲态 = **等待室**：Waiting Room + 已加入玩家名墙 + 人数（无二维码/网址/房间号，`qrcode.react` 已移除）
- [x] Admin Console 极简：**没有房间选择器**（当前轮 = 唯一未关闭房间），`new room` 一键开轮；Admin Data 的 Rooms 表**只读 + 删除**（列：Room / Created / Players / Status），Players 表可**新增、编辑、删除**（分数仍走接口）
- [x] Admin **Design** 页：只读展示**调色板**（Core 4 色 + Neutrals 7 色），值来自 `arcade_ftw_fe/src/design/tokens.ts`；**尚未**改成 CSS 变量，改 token 不影响渲染。主题已定为 **HSBC 红黑**（`#DB0011` + 黑 + 中性灰）；**全站组件仍是旧的薄荷配色，迁移未做**
- [x] `scores` 由游戏写入：玩家用自己令牌 `POST /api/me/score`，按题幂等累加到四个分区（`ScoreEvent` 表去重）
- [ ] 逐题**答案**未落库，所以大屏做不了选项分布；答案存下来后才能做全场聚合
- [ ] **强制规则不在本 repo**：位于 workspace 根的 `AI Arcade/.github/copilot-instructions.md`（刻意不入版本库）。若日后需要 GitHub 云端 Copilot 也遵守，再考虑在各 repo 放精简版

---

*改完系统后请同步更新本文件。*
