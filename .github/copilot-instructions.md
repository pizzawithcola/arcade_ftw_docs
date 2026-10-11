# Arcade FTW 强制规则（框架）

> **本文件是 `arcade_ftw` 三个 repo 的规则主来源**（single source of truth）。
>
> 参考旧系统迁移而来：
> - `cyber_ai_festival/.github/copilot-instructions.md`（前端：回答前/中/后 checklist 风格）
> - `cyber_ai_festival_be/.github/copilot-instructions.md`（后端：编号式工程规则）
>
> 状态：**框架**。标 ⬜ 的条目待确认后补齐。

---

## 0. 适用范围与分发

| repo | 角色 |
|---|---|
| `arcade_ftw_fe` | 前端（React + Vite + TS）|
| `arcade_ftw_be` | 后端（FastAPI + Python 3.12）|
| `arcade_ftw_docs` | **本文件所在**，跨项目共享知识库 |

**分发约定**：本文件是主来源；`arcade_ftw_fe` / `arcade_ftw_be` 各自的 `.github/copilot-instructions.md` 从本文件**派生**，只保留该 repo 相关的章节（第 1–4 章 + 第 8 章红线）。**不要把本文件整份复制过去**，避免两份规则分叉。

---

## 1. 回答前（所有 repo 通用）

- [ ] **问候语**：每个非代码回复以「好的，老板！」开头 ⬜ 是否保留
- [ ] **先读文档**：开工前先读 `arcade_ftw_docs/SYSTEM_HANDBOOK.md`
- [ ] **确认作用域**：这次改动应该落在哪个 repo？（polyrepo，见第 4 章）

## 2. 回答中（所有 repo 通用）

- [ ] **语言**：全程中文交互，代码内注释用英文
- [ ] **不懂就问**：模棱两可必须先问清楚，除非上下文足够明确
- [ ] **冲突管理**：发现当前内容与之前不一致时，可能是用户手动微调了，**绝不能私自改回去**，必须先问用户
- [ ] **端口保护（铁律）**：用户 dev 服务端口**绝不允许关闭、kill、占用或改动**
  - 旧系统端口：**1688**（可能跑 `dev` 或 `dev:prod`）
  - `arcade_ftw` 自己的端口：前端 ⬜ 5173 / 后端 ⬜ 8010（待确认）
  - 需要浏览器验证时**另起独立端口**（`169x/170x`），验证完只清理**自己起的**进程
  - 若需清理端口，先 `lsof -nP -iTCP:<port>` 确认进程归属；不是自己起的一律先问用户
- [ ] **共享资源改动先问**：VPC / 子网是唯一与旧系统共享的 AWS 资源

## 3. 回答后（所有 repo 通用）

- [ ] **范围控制**：确认只修改了项目内文件，未触及项目外内容
- [ ] **质量控制**：以资深 QA 视角审视本次修改是否会影响其他功能
- [ ] **修改范畴**：完全符合本次需求，不过分满足，不遗漏需求
- [ ] **文档回补**：状态变更（建资源 / 部署 / 改流程 / 改协议）后回补 `arcade_ftw_docs`
- [ ] **端口检查**：确认本次验证用独立端口，1688 dev 服务未被触碰

## 4. polyrepo 边界

- 一次改动只提交到**对应的** repo；跨项目文档改动落 `arcade_ftw_docs`。
- 代码 repo 里只放**指针**，不复制共享文档。
- 协议（WS 消息 / 题型）变更**必须同时**改 `arcade_ftw_be`、`arcade_ftw_fe` 与 `arcade_ftw_docs/protocol/`。

---

## 5. 前端专属（`arcade_ftw_fe`）

⬜ 待补（技术栈约束、组件复用、状态管理边界…）

## 6. 后端专属（`arcade_ftw_be`）

继承旧系统：

1. 改动后**补对应测试用例**。
2. 改动后**更新 README**。
3. 测试**用临时环境**，绝不污染真实数据库。

`arcade_ftw` 专属：

- **WS 是进程内内存态**：单进程运行，**禁止多 worker**；活动期间**禁止重启后端容器**。
- 后端镜像**必须** `--platform linux/amd64`（Mac 是 arm64，否则 Fargate 起不来）。
- **依赖必须 pin 版本**（旧系统被未锁定的新版 SQLAlchemy 引爆过）。
- 计分**用长表 / JSONB**，不硬编码 `game1~5_score`。
- 计分函数写成**纯函数**，便于单测。

## 7. 文档专属（`arcade_ftw_docs`）

- 本 repo 是**唯一共享 memory**；改完系统同步更新，别让文档过期。
- 状态流转、资源清单、部署要点都记在这里（`aws/`、`architecture/`、`protocol/`）。

## 8. 红线清单（铁律，任何 repo 不得违反）

1. **现有 repo 默认只读**——除非用户明确要求，不动 `cyber_ai_festival` / `cyber_ai_festival_be`。
2. **部署命令写死资源名**——一律用 `arcade_ftw-` 前缀变量，绝不手打 cluster / service 名。
3. **共享资源（VPC / 子网）改动必须先问用户**。
4. **绝不调用旧系统的破坏性接口**（purge / 删除）；验证新系统时 URL 用新域名。
5. **旧系统活动期间冻结旧 ECS**——禁止 push 后端 / 重启容器。
6. **端口 1688 铁律**——绝不 kill / 占用 / 改动。
7. **CloudFront 200 伪装**——判定请求成功必须校验 `content-type`，不能只看 `res.ok`。
8. **调 API 一律带尾斜杠**（裸路径会落到 S3）。
