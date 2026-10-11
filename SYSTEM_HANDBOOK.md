# Arcade FTW 系统手册（跨 session 共享 memory）

> ⚠️ **开工前请先完整读完本文件。** 本文件是 `arcade_ftw` 的权威来源，自包含，不依赖会话记忆。
>
> - 生成日期：2026-10-11
> - 上一代系统（`cyber_ai_festival` / `cyber_ai_festival_be`）的权威文档在其前端 repo 的 `doc/SYSTEM_HANDBOOK.md`；**本文件不复制它的内容**，只引用。

---

## 1. 定位

Arcade FTW = 第二个版本的游戏。与上一代系统 **完全平行、完全隔离**，但继承其运维经验与红线。

玩法：把多个单人游戏串成一条主线，中间穿插 quiz，**所有玩家同步推进**；题型不只是选择题（含 trust/doubt、互动题）；管理员可暂停 / 结束 / **回退到任意节点**。

## 2. 仓库布局（polyrepo）

| repo | 技术栈 | 状态 |
|---|---|---|
| `arcade_ftw_fe` | React 19 + TypeScript + Vite + zustand | 骨架已起 |
| `arcade_ftw_be` | FastAPI + Python 3.12 | 骨架已起 |
| `arcade_ftw_docs` | 本 repo | — |

**为什么 polyrepo**：见 [architecture/decisions](./architecture/decisions)。CI 天然按 repo 隔离，不存在 monorepo「前端提交误触发后端部署 → 清空内存态房间」的风险。

## 3. 当前状态（阶段 0：骨架）

已完成：

- 前端：Vite + React 19 + TS 脚手架，zustand 中央 store（`src/store/gameStore.ts`）
- 后端：FastAPI 入口（`/health`、CORS）、状态机（`app/engine.py`）、流程配置（`app/flow.py`）、WS 端点（`app/ws.py`）、题型注册表（`app/question_types/`）
- 前后端依赖已安装、构建/接口/WS 均已验证通过

未完成：主线流程、多题型、管理员控制、数据库、AWS。

## 4. 与上一代系统的关系

| 事项 | 结论 |
|---|---|
| 代码 | **不迁移**旧游戏代码；旧系统冻结 |
| 可复用 | UI 组件库 / 主题 / 部署管道模式 |
| AWS | **隔离优先**：除 VPC/子网外全部新建，资源名用 `arcade_ftw-` 前缀 |
| 数据库 | 全新实例，不碰旧库 |
| 文档 | 本 repo 为权威；旧系统文档不动 |

## 5. 红线与踩坑（沿用上一代教训）

1. **现有 repo 默认只读**——除非用户明确要求，不动 `cyber_ai_festival` / `cyber_ai_festival_be`。
2. **端口 1688 绝不可碰**（用户本地 dev 服务）；验证另起 1690/8010 等独立端口。
3. **WS 是进程内内存态**——单进程运行，禁止多 worker；换 task 会清空房间，活动期间禁止重启后端容器。
4. **后端镜像必须 `--platform linux/amd64`**（Mac arm64 直 build 会得到 arm64 镜像，Fargate 起不来）。
5. **CloudFront 会把 403/404 重写成 `200 + index.html`**——后端错误会伪装成 `200 text/html`；判定成功必须校验 content-type，不能只看 `res.ok`。
6. **调 API 一律带尾斜杠**（旧系统行为白名单不含裸路径，裸 `/users` 会落到 S3）。
7. 凡用 storage 做「跨刷新保持」的状态，必须列出所有**生命周期结束点**并逐个清理（旧系统踩过跨局残留）。
8. **依赖要 pin 版本**（旧系统因未 pin 被新版 SQLAlchemy 引爆过）。

## 6. 待办 / 未决

- [ ] `SYSTEM_HANDBOOK.md` 在旧系统里叫 "香港 FTW"，本项目管理名已改为 `arcade_ftw`——旧文档 Part B 已过时（需用户同意才能改旧 repo）
- [ ] `packages/shared`（机器可读的协议类型）尚未建；按旧手册 §11，起步用「TS 类型 + Pydantic 手写对应」，稳定后升级 JSON Schema
- [ ] AWS 资源尚未创建（见 [aws/](./aws)）
- [ ] `.github/copilot-instructions.md`（红线）尚未写入两个代码 repo

---

*改完系统后请同步更新本文件。*
