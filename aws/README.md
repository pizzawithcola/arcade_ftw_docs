# AWS

## 隔离策略

**除 VPC / 子网外全部新建**，资源名统一 `arcade-ftw-` 前缀。共享 VPC 的任何改动**必须先问用户**。

## 资源清单

IaC 模板：`arcade_ftw_be/infra/cloudformation.yml` —— 只覆盖**后端**资源。
**已部署**：CloudFormation 栈 **`arcade-ftw`**（`ap-south-1`，账号 `866730904414`，`UPDATE_COMPLETE`）。

| 资源 | 标识 | 状态 |
|---|---|---|
| ECR repo | `arcade-ftw-backend` → `866730904414.dkr.ecr.ap-south-1.amazonaws.com/arcade-ftw-backend` | ✅ 已有镜像（生命周期保留最近 10 个）|
| ECS Cluster | `arcade-ftw-cluster` | ✅ 已建 |
| ECS Task | `arcade-ftw-task`（0.5 vCPU / 1GB）| ✅ revision 3 |
| ECS Service | `arcade-ftw-service` | ✅ ACTIVE，`desiredCount = 1`，稳定运行 |
| ALB | `arcade-ftw-alb` → `arcade-ftw-alb-1071955694.ap-south-1.elb.amazonaws.com` | ✅ active（1a + 1b），**idle timeout 3600s** |
| Target Group | `arcade-ftw-tg`（HTTP:8000，健康检查 `/health`）| ✅ 已建，target `healthy` |
| 安全组 | `arcade-ftw-alb-sg` / `arcade-ftw-ecs-sg`（8000 仅允许来自 ALB）| ✅ 已建 |
| IAM roles | `arcade-ftw-ecs-task-role` / `arcade-ftw-ecs-task-execution-role` | ✅ 已建 |
| IAM OIDC provider | `token.actions.githubusercontent.com` | ✅ 已建（CI 用）|
| IAM 部署角色 | `arcade-ftw-github-deploy` | ✅ 已建（CI 用）|
| 日志组 | `/ecs/arcade-ftw-backend`（保留 30 天）| ✅ 已建 |
| VPC / 子网 | 旧系统共用 `vpc-0dda76b286c45fc27`（**只引用**）| ✅ 未改动 |
| RDS | 暂不建（下一步做，见下）| ⬜ 未定义 |
| S3 + CloudFront | 属**前端 repo** 的范围 | ⬜ 未定义 |

### 首次上线必须先 bootstrap

ECR 为空时不能直接把服务设成 1：任务拉不到镜像会反复失败。首次流程：

```bash
# 1) 本地构建并推送（必须 amd64）
docker build --platform linux/amd64 -t $ECR_URI:$TAG . && docker push $ECR_URI:$TAG
# 2) 拉起服务
aws ecs update-service --cluster arcade-ftw-cluster --service arcade-ftw-service --desired-count 1
```

之后再回到正常路径：**一切改动走 CI**（`git push main`）。

> 栈参数 `DesiredCount`（默认 1）就是为此存在：**首次建栈必须用 0**，否则 CloudFormation 会等 ECS 达到稳定态而**挂住并回滚**。

### 建栈时实际踩到的坑（旧模板同源或同源缺陷）

1. **ECS Service 依赖顺序错误**：ECS 拒绝「Target Group 尚未挂到 ALB」的服务，而 `!Ref AlbTargetGroup` 并不隐含 Listener 依赖 → 必须显式 `DependsOn: AlbHttpListener`。旧模板没有这行，说明它**从未通过 CloudFormation 成功部署**。
2. **`DesiredCount: 1` + 空 ECR = 栈挂死**：首次建栈必须先把 desired count 设为 0。
3. **GitHub OIDC subject 变了**：带不可变数字 ID，旧写法永远匹配不上（详见 [CI](#ci) 一节）。

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
9. 本地校验工具：`actionlint`（workflow）+ `hadolint`（Dockerfile）+ `cfn-lint`（模板），见下节。
10. **本机网络对部分域名是白名单制**（实测）：`registry-1.docker.io`、`github.com`、`mirror.gcr.io` **不通**；`public.ecr.aws`、`ghcr.io`、`api.github.com`、`pypi.org`、`registry.npmjs.org` **通**。这会决定你能否本地构建/下载工具——遇到超时先怀疑它，别改代码。

## CI

`.github/workflows/deploy.yml`：`push main` → 构建 → ECR → **拉线上 task definition 再改镜像**（不依赖仓库里的 `fixed-task-def.json`，避免 drift）→ `update-service` → 验证 `/health`。

**凭据：GitHub OIDC，零长期密钥（无任何 GitHub secret）。**

栈里创建了：

| 资源 | 值 |
|---|---|
| OIDC provider | `arn:aws:iam::866730904414:oidc-provider/token.actions.githubusercontent.com` |
| 部署角色 | `arn:aws:iam::866730904414:role/arcade-ftw-github-deploy` |

信任策略**精确锁定**到（两种写法都接受，且**均无通配符**）：

```
repo:pizzawithcola/arcade_ftw_be:ref:refs/heads/main
repo:pizzawithcola@68038650/arcade_ftw_be@1414195943:ref:refs/heads/main
```

⚠️ **重要坑（实测）**：GitHub 现在在 OIDC subject 里带**不可变的数字 ID**，即
`repo:<owner>@<ownerId>/<repo>@<repoId>:ref:refs/heads/<branch>`。
只写人读形式 `repo:owner/repo:...` 会永远匹配不上，STS 返回
`AccessDenied: Not authorized to perform sts:AssumeRoleWithWebIdentity`。
排查方法：`aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=AssumeRoleWithWebIdentity`，从 `userIdentity.principalId` 直接读出真实 subject。

→ fork、其他仓库、PR 都**无法**冒充。权限仅限：推本 ECR 仓库、注册 task definition、更新本 service、`iam:PassRole` 到那两个 task role、读取 ALB。

> **为什么不用静态密钥**：账号里唯一的凭据是 `AdministratorAccess` 的 `JamieLiu8`，而本 repo 是 **public**——把 admin 密钥放进公开仓库 secrets，等于把**整个账号**（含旧活动系统）悬在泄漏风险上。OIDC 从根上消除了这个风险。

其他：`concurrency` 串行化，避免两次部署互相打断；`push` 与 `workflow_dispatch` 均可触发。

**实测**：一次完整部署（run #5）**约 3 分 30 秒**——OIDC 鉴权 → 构建 → 推 ECR → 改 task def → `update-service` → 验证 `/health`。

## 本地校验（改 IaC / CI 后必跑）

```bash
# CloudFormation 模板
cfn-lint infra/cloudformation.yml                       # pip install cfn-lint
aws cloudformation validate-template \
  --template-body file://infra/cloudformation.yml --region ap-south-1

# CI workflow / Dockerfile
actionlint .github/workflows/deploy.yml                 # brew install actionlint
hadolint Dockerfile                                     # brew install hadolint

# 镜像真机验证（必须 amd64）
docker build --platform linux/amd64 -t arcade-ftw-backend:local .
docker run --rm -p 1692:8000 arcade-ftw-backend:local
```

> 实测价值：`cfn-lint` 抓到 `TaskRoleArn` 用 `!Ref` 而非 `!GetAtt` 的真 bug；`actionlint`/`hadolint` 各抓到 5 处与 2 处真问题。**这套校验在本次会话里每次都抓到了东西**，别跳过。

## 排查手册

```bash
# STS / OIDC 失败：直接读真实 subject 与 errorCode
aws cloudtrail lookup-events --region ap-south-1 \
  --lookup-attributes AttributeKey=EventName,AttributeValue=AssumeRoleWithWebIdentity

# 栈失败：看哪个资源先炸
aws cloudformation describe-stack-events --stack-name arcade-ftw --region ap-south-1 \
  --query 'StackEvents[*].[Timestamp,LogicalResourceId,ResourceStatus,ResourceStatusReason]' --output text

# 服务/任务
aws ecs describe-services --cluster arcade-ftw-cluster --services arcade-ftw-service
aws ecs describe-tasks --cluster arcade-ftw-cluster --tasks $(aws ecs list-tasks --cluster arcade-ftw-cluster --query 'taskArns[0]' --output text)

# 容器日志
aws logs tail /ecs/arcade-ftw-backend --follow

# 区分后端 JSON 与 CloudFront/S3 的 HTML 伪装
curl -s -o /dev/null -w '%{http_code} %{content_type}\n' <url>
```

## 成本与停机

粗估（`ap-south-1`，未含流量）：Fargate 0.5 vCPU/1GB ≈ $24/月，ALB ≈ $16/月 + LCU ≈ **合计约 $40–50/月**。

```bash
# 只停计算（省 Fargate 费，ALB 仍计费）
aws ecs update-service --cluster arcade-ftw-cluster --service arcade-ftw-service --desired-count 0

# 全部拆除（⚠️ 会一并删掉 OIDC provider 与部署角色 → CI 会失败，直到重新建栈）
aws cloudformation delete-stack --stack-name arcade-ftw --region ap-south-1
```

## 红线

- 绝不调用旧系统的破坏性接口（purge / 删除）。
- 验证新系统时 URL 用新域名。
- VPC / 子网是**唯一共享资源**，只引用不修改。

## 待办

- [x] 目标账号 / 区域：`866730904414` / `ap-south-1`
- [x] 部署 `infra/cloudformation.yml` → 栈 `arcade-ftw`（含 OIDC provider + 部署角色）
- [x] 推送首个镜像并把 `desiredCount` 提到 1 —— 服务已对外可用
- [x] 接通 CI —— 改用 **GitHub OIDC**，无需任何 GitHub secret
- [ ] 建 Secrets Manager，把 `API_KEY` 从明文 env 迁走
- [ ] 建 RDS + 引入 Alembic（数据库改动目前**无处可部署**）—— 方案见 [architecture/rds-and-migrations.md](../architecture/rds-and-migrations.md)
- [ ] 前端侧 CloudFront 分发（含 API 行为白名单 + 403/404 错误重写）
- [ ] 考虑把 `arcade_ftw_be` 转为 private（目前 public）

