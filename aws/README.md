# AWS

## 隔离策略

**除 VPC / 子网外全部新建**，资源名统一 `arcade-ftw-` 前缀。共享 VPC 的任何改动**必须先问用户**。

## 资源清单

IaC 模板：`arcade_ftw_be/infra/cloudformation.yml` —— 只覆盖**后端**资源。
**已部署**：CloudFormation 栈 **`arcade-ftw`**（`ap-south-1`，账号 `866730904414`，`CREATE_COMPLETE`）。

| 资源 | 标识 | 状态 |
|---|---|---|
| ECR repo | `arcade-ftw-backend` → `866730904414.dkr.ecr.ap-south-1.amazonaws.com/arcade-ftw-backend` | ✅ 已建（**暂无镜像**）|
| ECS Cluster | `arcade-ftw-cluster` | ✅ 已建 |
| ECS Task | `arcade-ftw-task`（0.5 vCPU / 1GB）| ✅ 已建 |
| ECS Service | `arcade-ftw-service` | ✅ ACTIVE，**desiredCount = 0**（见下）|
| ALB | `arcade-ftw-alb` → `arcade-ftw-alb-1071955694.ap-south-1.elb.amazonaws.com` | ✅ active（1a + 1b），**idle timeout 3600s** |
| Target Group | `arcade-ftw-tg`（HTTP:8000，健康检查 `/health`）| ✅ 已建 |
| 安全组 | `arcade-ftw-alb-sg` / `arcade-ftw-ecs-sg`（8000 仅允许来自 ALB）| ✅ 已建 |
| IAM roles | `arcade-ftw-ecs-task-role` / `arcade-ftw-ecs-task-execution-role` | ✅ 已建 |
| 日志组 | `/ecs/arcade-ftw-backend`（保留 30 天）| ✅ 已建 |
| VPC / 子网 | 旧系统共用 `vpc-0dda76b286c45fc27`（**只引用**）| ✅ 未改动 |
| RDS | 暂不建（placeholder 阶段）| ⬜ 未定义 |
| S3 + CloudFront | 属**前端 repo** 的范围 | ⬜ 未定义 |

### 当前为什么 `desiredCount = 0`

ECR 里还没有镜像，且 **CI 未接**（GitHub secrets 未配）。若设成 1，任务拉不到镜像会反复失败。要上线：

```bash
# 1) 本地构建并推送（必须 amd64）
docker build --platform linux/amd64 -t $ECR_URI:$TAG . && docker push $ECR_URI:$TAG
# 2) 拉起服务
aws ecs update-service --cluster arcade-ftw-cluster --service arcade-ftw-service --desired-count 1
```

> 栈参数 `DesiredCount`（默认 1）就是为此存在：**首次建栈必须用 0**，否则 CloudFormation 会等 ECS 达到稳定态而**挂住并回滚**。

### 建栈时实际踩到的两个坑（旧模板同源）

1. **ECS Service 依赖顺序错误**：ECS 拒绝「Target Group 尚未挂到 ALB」的服务，而 `!Ref AlbTargetGroup` 并不隐含 Listener 依赖 → 必须显式 `DependsOn: AlbHttpListener`。旧模板没有这行，说明它**从未通过 CloudFormation 成功部署**。
2. **`DesiredCount: 1` + 空 ECR = 栈挂死**：首次建栈必须先把 desired count 设为 0。

**命名注意**：AWS 的 ALB / Target Group 名称**不允许下划线**，因此资源名前缀统一用连字符 `arcade-ftw-`（仓库名仍为 `arcade_ftw_*`）。

## 部署要点（沿用旧系统教训）

1. **后端镜像必须 `--platform linux/amd64`**（本地 Mac 构建时）。GitHub runner 本身是 amd64，CI 不受影响。
2. **任务子网必须落在 ALB 已启用的 AZ**（旧系统曾因此约 1/3 概率部署失败）。
3. **任务子网需有 egress**：旧 VPC 无 NAT 网关，故 `AssignPublicIp: ENABLED` 必须保持。
4. **CloudFront 200 伪装**：行为白名单 + 错误重写没配好，任何后端错误都会变成 `200 text/html`（前端那步处理）。
5. **调 API 带尾斜杠**（否则裸路径落 S3）。
6. 用 `curl -w '%{content_type}'` 区分 `application/json` vs `text/html` 来排查。
7. 旧模板里 `!Ref` 传给 `TaskRoleArn` / `ExecutionRoleArn` 是**错的**（`Ref` 返回 role 名而非 ARN，需 `!GetAtt X.Arn`）；新模板已修正。
8. 基础镜像使用 `public.ecr.aws/docker/library/python:3.12-slim`，避开 Docker Hub 匿名拉取限流（否则 CI 会随机失败）。
9. 本地校验工具：`actionlint`（workflow）+ `hadolint`（Dockerfile），`brew install actionlint hadolint`。

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

- [x] 目标账号 / 区域：`866730904414` / `ap-south-1`
- [x] 部署 `infra/cloudformation.yml` → 栈 `arcade-ftw` 已 `CREATE_COMPLETE`
- [ ] **推送首个镜像并把 `desiredCount` 提到 1**（当前为 0，服务尚不对外提供服务）
- [ ] 配置 GitHub secrets（`AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY`）以接通 CI
- [ ] 建 Secrets Manager，把 `API_KEY` 从明文 env 迁走
- [ ] 建 RDS + 引入 Alembic（数据库改动目前**无处可部署**）
- [ ] 前端侧 CloudFront 分发（含 API 行为白名单 + 403/404 错误重写）

