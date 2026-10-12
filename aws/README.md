# AWS

## 隔离策略

**除 VPC / 子网外全部新建**，资源名统一 `arcade-ftw-` 前缀。共享 VPC 的任何改动**必须先问用户**。

## 资源清单

**两个栈**：

| 栈 | 模板 | 覆盖 |
|---|---|---|
| `arcade-ftw`（后端）| `arcade_ftw_be/infra/cloudformation.yml` | ECR / ALB / ECS / RDS / IAM / 日志 / OIDC provider |
| `arcade-ftw-frontend`（前端）| `arcade_ftw_fe/infra/cloudformation.yml` | S3 / CloudFront / SPA 回退函数 / 前端 OIDC 角色 |

**已部署**：`ap-south-1`，账号 `866730904414`。

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
| RDS | `arcade-ftw-db`（`db.t4g.micro` / PG 16.15 / 非公网 / 存储加密 / 删除保护）| ✅ 已建 |
| RDS 安全组 | `arcade-ftw-rds-sg`（5432 仅来自 `arcade-ftw-ecs-sg`）| ✅ 已建 |
| DB 子网组 | 复用现有 `default-vpc-0dda76b286c45fc27`（**未新建**）| ✅ 只引用 |
| S3 bucket | `arcade-ftw-frontend`（私有 + OAC）| ✅ 已建 |
| CloudFront | `EJUERA5VLSVBB` → `d39s9y5ma27s5.cloudfront.net` | ✅ 已建 |
| SPA 回退函数 | CloudFront Function `arcade-ftw-spa-fallback` | ✅ 已建 |
| 前端 OIDC 角色 | `arcade-ftw-fe-github-deploy` | ✅ 已建 |

**公开访问点**：`https://d39s9y5ma27s5.cloudfront.net`（站点 + API 同源）。**ALB 不再对外可达**。

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
11. **`EngineVersion` 必须给具体 PostgreSQL 小版本**（如 `16.15`）；只写 `16` 会被 RDS 拒绝。
12. **`PreferredBackupWindow` 只接受 `hh24:mi-hh24:mi`**（不带星期）；星期前缀只对**维护窗口**有效。写错会导致建库失败。
13. **加 RDS 时要分两阶段部署**：先只建库、**先不把 DB 变量接进任务定义**，确认托管密钥的 `password` 键可用后，再接任务定义。否则一旦任务起不来，栈回滚会**把刚建好的 RDS 一起删掉**。（更普通的说法：任何会让 ECS 服务无法稳定的改动，都会连累同一栈里的数据库。）
14. **容器里要能拿到 `alembic.ini`**：Dockerfile 只 `COPY app ./app` 是不够的，迁移任务会立刻失败。已修。
15. **`cloudformation deploy` 按栈里存的参数收敛**：若曾用 `DesiredCount=0` 建栈、之后手工把服务拉到 1，下一次 deploy 必须显式带 `DesiredCount=1`，否则会把服务打回 0。
16. **不要改已命名安全组的 `GroupDescription`**：AWS 拒绝更新描述并触发**强制替换**，而替换要用同名 `GroupName` → 报 `Security Group ... already exists` 并失败。描述一旦写错就改名或放弃修正（本项目的 `arcade-ftw-alb-sg` 描述至今仍写着 "from the internet"）。
17. **CloudFront 源站前缀列表 `pl-9aa247f3` 有 46 条**，而安全组上限 60 条规则（按条目计）。所以**一个安全组只能引用它一次**——80 与 443 各引一次会报 `maximum number of rules per security group has been reached`。
18. **收紧 ALB 安全组会让 CI 的直连 curl 验证失效**。凡是要验证健康检查的地方（CI、人工排查）一律改走 CloudFront 域名。
19. **WebSocket 经 CloudFront 是「零配置」可用的**（已实测 `wss://<cf>/api/ws` 握手 101 并收到消息）。曾一度误判为 CloudFront 故障，实际是**新代码还没部署**——源站日志里那行 `"WebSocket /api/ws" 403` 是「没有匹配到路由」，不是网络问题。**排查顺序：先看源站日志，再怀疑 CDN。**
20. **CloudFront 分发创建很慢**（约 10–20 分钟），`cloudformation deploy` 要有耐心，别误判为卡死。
21. **两个栈通过 export/import 耦合**：前端栈 `Fn::ImportValue` 后端栈的 `arcade-ftw-AlbDns`，所以**前端栈存在期间后端栈无法单独删除**。
22. **CI 与栈更新是两个互不知道的写者，方向不对称，要分清**：
    - **CI 读线上的 task definition**，只替换镜像 tag → 所以**代码推送不会把改过的环境变量（如管理员密码）打回模板默认值**。
    - **反过来栈更新会赢**：`AWS::ECS::TaskDefinition` 是按模板重新生成一个 revision 并让服务指向它，于是服务从 CI 上次固定的 `<sha>` 回到模板里的 `:latest`。内容其实相同（CI 两个 tag 同推、digest 一致，已实测），但**不再是精确版本**——想要可追溯的版本号，栈更新后跑一次 CI（或直接 `update-service` 指回该 SHA 的 revision）。
23. **改单个参数别重发全部参数**：`update-stack --use-previous-template` + 逐个 `UsePreviousValue: true`，可以让 `SecretKey` 这类值**完全不经过命令行与 shell 历史**。注意 `update-stack` **必须显式带 `--capabilities CAPABILITY_NAMED_IAM`**，即使这次改动本身不碰 IAM。

### 刻意**不**做的事

- **不配置分发表级的 `403/404 → 200 /index.html`**（旧系统的做法）。那会让所有后端错误伪装成 `200 text/html`。这里用只在 S3 行为上生效的 CloudFront Function 做 SPA 回退，实测：`/api/nope` → **真实 404 JSON**，`/missing.js` → **真实 403**。

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
- [x] 建 RDS（`arcade-ftw-db`）+ Alembic 迁移链路 —— 数据库改动已可一键部署
- [x] 前端部署：S3 + CloudFront（站点与 API **同源**），前端 CI 走 OIDC
- [ ] 建 Secrets Manager 保存 `API_KEY`，把它从任务定义明文 env 迁走
- [ ] 考虑把 `arcade_ftw_be` / `arcade_ftw_fe` 转为 private（目前都是 public）
- [ ] 考虑给 CloudFront 配自定义域名 + ACM 证书（目前用 `*.cloudfront.net`）

