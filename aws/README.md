# AWS

## 隔离策略

**除 VPC / 子网外全部新建**，资源名统一 `arcade-ftw-` 前缀。共享 VPC 的任何改动**必须先问用户**。

## 资源清单

IaC 模板已写好（**尚未部署**）：`arcade_ftw_be/infra/cloudformation.yml` —— 只覆盖**后端**资源。

| 资源 | 标识 | 状态 |
|---|---|---|
| ECR repo | `arcade-ftw-backend` | ✅ 模板已定义（含生命周期策略保留最近 10 个镜像）|
| ECS Cluster | `arcade-ftw-cluster` | ✅ 模板已定义 |
| ECS Task | `arcade-ftw-task`（0.5 vCPU / 1GB，`DesiredCount: 1`）| ✅ 模板已定义 |
| ECS Service | `arcade-ftw-service` | ✅ 模板已定义 |
| ALB | `arcade-ftw-alb`（**idle timeout 3600s**，WS 必需）| ✅ 模板已定义 |
| Target Group | `arcade-ftw-tg`（健康检查 `/health`）| ✅ 模板已定义 |
| 安全组 | `arcade-ftw-alb-sg` / `arcade-ftw-ecs-sg`（8000 仅允许来自 ALB）| ✅ 模板已定义 |
| IAM roles | `arcade-ftw-ecs-task-role` / `arcade-ftw-ecs-task-execution-role` | ✅ 模板已定义 |
| 日志组 | `/ecs/arcade-ftw-backend`（保留 30 天）| ✅ 模板已定义 |
| VPC / 子网 | 旧系统共用，**仅作参数传入** | ✅ 参数占位 |
| RDS | 暂不建（placeholder 阶段）| ⬜ 未定义 |
| S3 + CloudFront | 属**前端 repo** 的范围 | ⬜ 未定义 |

**命名注意**：AWS 的 ALB / Target Group 名称**不允许下划线**，因此资源名前缀统一用连字符 `arcade-ftw-`（仓库名仍为 `arcade_ftw_*`）。

## 部署要点（沿用旧系统教训）

1. **后端镜像必须 `--platform linux/amd64`**（本地 Mac 构建时）。GitHub runner 本身是 amd64，CI 不受影响。
2. **任务子网必须落在 ALB 已启用的 AZ**（旧系统曾因此约 1/3 概率部署失败）。
3. **任务子网需有 egress**：旧 VPC 无 NAT 网关，故 `AssignPublicIp: ENABLED` 必须保持。
4. **CloudFront 200 伪装**：行为白名单 + 错误重写没配好，任何后端错误都会变成 `200 text/html`（前端那步处理）。
5. **调 API 带尾斜杠**（否则裸路径落 S3）。
6. 用 `curl -w '%{content_type}'` 区分 `application/json` vs `text/html` 来排查。
7. 旧模板里 `!Ref` 传给 `TaskRoleArn` / `ExecutionRoleArn` 是**错的**（`Ref` 返回 role 名而非 ARN，需 `!GetAtt X.Arn`）；新模板已修正。

## CI

`.github/workflows/deploy.yml`：`push main` → 构建 → ECR → **拉线上 task definition 再改镜像**（不依赖仓库里的 `fixed-task-def.json`，避免 drift）→ `update-service` → 验证 `/health`。

- 凭据目前用静态 AK/SK（沿用旧做法），文件内留了 OIDC 切换样板。
- 用 `check-creds` 前置守卫：未配置 secret 时**只告警不失败**，避免 AWS 还没建就一片红。
- `concurrency` 串行化，避免两次部署互相打断。

## 红线

- 绝不调用旧系统的破坏性接口（purge / 删除）。
- 验证新系统时 URL 用新域名。
- VPC / 子网是**唯一共享资源**，只引用不修改。

## 待办

- [ ] 确认目标账号 / 区域（模板默认 `ap-south-1`）
- [ ] 实际部署 `infra/cloudformation.yml`（当前只看文件，未创建资源）
- [ ] 建 Secrets Manager，把 `API_KEY` 从明文 env 迁走
- [ ] 前端侧 CloudFront 分发（含 API 行为白名单 + 403/404 错误重写）

