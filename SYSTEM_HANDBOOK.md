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

主线可以完整跑通：登录 → 等待室 → 引擎自动推进 8 个节点 → 收尾；三个玩法界面已有真实实现（Hallucination / Quiz / Data Shadow），房间 / 玩家 / 分数已数据化，前后端已部署到 AWS。

已完成：

- 前端：React 19 + Vite + TS + zustand；四条 UI 路线（登录 / 玩家 / 大屏 / 管理台）；三个真实玩法（Hallucination、Quiz、Data Shadow）；等待室、跟随当前轮、题目索引与大屏题面
- 后端：FastAPI 入口（`/health`、`/health/db`、`/api/*`）、按房间隔离的状态机（`app/engine.py`）、流程配置（`app/flow.py`）、WS 推送（`app/ws.py`）、房间 / 玩家 / 分数接口、HMAC 令牌鉴权
- 数据：PostgreSQL + Alembic，`rooms` / `players` / `scores` / `score_events`
- 运维：CloudFormation 双栈、OIDC 零密钥 CI、CloudFront 同源部署

未完成：Sabotage 游戏界面、题库后端化与题型注册表接线、逐题答案存储与大屏聚合、AWS 剩余项（见第 6 节与 [TODO](./TODO.md)）。

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
- [ ] **Sabotage 仍是占位屏**（`src/games/` 下尚无该游戏），节点照常倒计时推进
- [ ] 题型注册表仍只有 `choice` 一个示例，且**尚未被引擎 / WS 调用**；题库无存储
- [x] 数据库：`rooms` / `players` / `scores` 三张表，房间是数据（`main` 由迁移 seed），成员记在 `players.room_code`
- [x] 房间 / 玩家 / 分数的增删改查接口（`/api/rooms`、`/api/players/{id}`、`/api/players/{id}/score`）；admin 面板可开新一轮
- [x] **room = 一轮游戏**：admin 新建 `round-N`，大屏 `/display` 与玩家 `/play` **都自动跟随当前轮**
- [x] 玩家不扫码：登录后在 `/play` 跟随当前轮；本轮 `finished` → 收尾屏 8 秒后回登录页
- [x] 大屏展示题目（索引由引擎按 `questions`/`seconds_each` 算出），选项只在玩家屏
- [x] 大屏空闲态 = **等待室**：Waiting Room + 已加入玩家名墙 + 人数（无二维码/网址/房间号，`qrcode.react` 已移除）
- [x] Admin Data 分区可对房间/玩家**新增、编辑、删除**（分数仍走接口）
- [x] `scores` 由游戏写入：玩家用自己令牌 `POST /api/me/score`，按题幂等累加到四个分区（`ScoreEvent` 表去重）
- [ ] 逐题**答案**未落库，所以大屏做不了选项分布；答案存下来后才能做全场聚合
- [ ] **强制规则不在本 repo**：位于 workspace 根的 `AI Arcade/.github/copilot-instructions.md`（刻意不入版本库）。若日后需要 GitHub 云端 Copilot 也遵守，再考虑在各 repo 放精简版

---

*改完系统后请同步更新本文件。*
