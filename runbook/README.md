# 活动当天 Runbook

> ⬜ 待补充。活动前必须把下面的清单填满并演练一遍。

## 公开访问点

```
https://d39s9y5ma27s5.cloudfront.net        # 站点
https://d39s9y5ma27s5.cloudfront.net/api/*  # API（同源，无 CORS）
https://d39s9y5ma27s5.cloudfront.net/health # 健康检查
```

> ⚠️ **ALB 已不再对外可达**（安全组只允许 CloudFront 的源站 IP）。所有验证、排查一律走上面这个域名，直连 ALB 会超时。

## 活动前

- [ ] 后端部署到最新镜像，确认所有 task 健康
- [ ] 确认 `curl https://<上面域名>/health/db` 返回 200（**不能只看 `/health`**——它刻意不查库）
- [ ] 数据库快照已建
- [ ] 前端产物已上线，`assets/index-*.js` 哈希确认更新
- [ ] 大屏（kiosk）自愈与硬重载已开启
- [ ] 实机验证玩家链路：登录 → `/play` **自动跟随当前轮**（不扫码）→ 等待室只显示三行
- [ ] 实机验证大屏：`/display` 自动跟随当前轮；等待室人数与名墙随玩家 `join` 更新

## 活动中

- [ ] **禁止部署 / 重启后端容器**（会清空内存态房间）
- [ ] 管理员控制路径不影响进行中的房间

### 操作台动作（现场备忘）

| 想做的事 | 按哪个 | ⚠️ 注意 |
|---|---|---|
| 开演 | `start` | 只用于 waiting / finished（从头开演）；**暂停中跳节点后不要按** |
| 暂停 | `pause` | 冻结剩余时间，暂停多久都不影响 |
| 继续 | `resume` | |
| **暂停后跳到某节点再继续** | **`resume`** | ⚠️ **别按 `start`**，它会把整个节目弹回第 1 节点；控制台不会禁用该按钮 |
| 调整节奏 | `go/{n}` | 跳转后大屏显示该节点**满时长** |
| 结束 | `finish` | |
| 排练完，准备正式开演 | `reset` → 等观众 → `start` | `reset` 后大屏显示「待开演」 |

## 活动后

- [ ] 数据备份 / 归档
- [ ] 回补文档与本次事故记录

## 数据库变更

改动 schema 的常规路径：

```bash
# 本地
docker compose up -d
alembic revision --autogenerate -m "add xxx"
# 检查生成的脚本，确认 downgrade() 可用
alembic upgrade head && alembic downgrade -1 && alembic upgrade head
pytest
git push          # CI 会先跑迁移，成功后才滚动更新
```

> ⚠️ **迁移失败会阻断部署**（这是刻意的）：旧 revision 继续服务，不会出现「代码已上新、schema 未生效」。
> ⚠️ **迁移不可自动回滚**：代码可以回滚，schema 不能。回滚需人工执行 `alembic downgrade`，且要先确认数据兼容。

## 凭据轮换（管理员密码 / SecretKey）

两者都活在**任务定义的环境变量**里，所以「改密码」= 改栈参数 → 新 task definition revision → ECS 滚动更新。它们**不在 Secrets Manager**（只有 RDS 主密码由 RDS 托管），所以**每一次轮换都是一次部署**。

```bash
# 只改 AdminPassword，其余参数一律沿用栈里现存的值。
# 这样 SecretKey 完全不经过命令行与 shell 历史。
python3 - <<'PY'
import json
keep = ["NamePrefix","DesiredCount","CloudFrontPrefixListId","TaskSubnet2","GitHubRepoIds",
        "DbSubnetGroupName","TaskSubnet1","AdminUsername","SecretKey","VpcId","PublicSubnet2",
        "DbName","DbInstanceClass","DbEngineVersion","GitHubBranch","PublicSubnet1",
        "GitHubRepo","CertificateArn"]
params = [{"ParameterKey": k, "UsePreviousValue": True} for k in keep]
params.append({"ParameterKey": "AdminPassword", "ParameterValue": "新密码"})
json.dump(params, open("/tmp/pw.json", "w"), indent=1)
PY

aws cloudformation update-stack --stack-name arcade-ftw --region ap-south-1 \
  --use-previous-template --capabilities CAPABILITY_NAMED_IAM \
  --parameters file:///tmp/pw.json
rm -f /tmp/pw.json

aws cloudformation wait stack-update-complete --stack-name arcade-ftw --region ap-south-1
aws ecs wait services-stable --cluster arcade-ftw-cluster --services arcade-ftw-service --region ap-south-1

# 验证：旧密码 401 / 新密码 200
D=https://d39s9y5ma27s5.cloudfront.net
curl -s -o /dev/null -w '旧密码 %{http_code}\n' -X POST "$D/api/auth/admin" \
  -H 'Content-Type: application/json' -d '{"username":"admin","password":"旧密码"}'
curl -s -o /dev/null -w '新密码 %{http_code}\n' -X POST "$D/api/auth/admin" \
  -H 'Content-Type: application/json' -d '{"username":"admin","password":"新密码"}'
```

> ⚠️ **轮换会替换容器**，内存里的引擎状态归零（回 `waiting`，进度全丢）。**活动进行中绝对不要轮换。**
> ⚠️ 轮换 `SecretKey` 会让**所有已签发的令牌立即失效**（在场玩家需重新登录，管理员需重新登录）。
> ⚠️ 栈更新会把服务指回模板里的 `:latest`（见 [../aws/README.md](../aws/README.md) 第 22 条）：内容相同，但不再是 CI 固定的版本号。想恢复精确版本，轮换后再跑一次 CI。
> ⚠️ `update-stack` **必须带 `--capabilities CAPABILITY_NAMED_IAM`**，否则报 `InsufficientCapabilitiesException`。

## 已知的部署操作

```bash
# 常规部署：走 CI（push main 即可），约 3.5 分钟。下面是排障/首次建栈用的。

# 基础设施（幂等；参数取值见 ../aws/README.md）
# 现存的完整参数集可以直接读出来，避免手抄漏项：
aws cloudformation describe-stacks --stack-name arcade-ftw --region ap-south-1 \
  --query 'Stacks[0].Parameters'
aws cloudformation deploy --region ap-south-1 \
  --template-file infra/cloudformation.yml \
  --stack-name arcade-ftw \
  --capabilities CAPABILITY_NAMED_IAM \
  --parameter-overrides NamePrefix=arcade-ftw VpcId=vpc-... \
                         PublicSubnet1=subnet-... PublicSubnet2=subnet-... \
                         TaskSubnet1=subnet-... TaskSubnet2=subnet-... \
                         DbSubnetGroupName=... DbName=arcade_ftw DbEngineVersion=16.15 \
                         DbInstanceClass=db.t4g.micro CloudFrontPrefixListId=pl-9aa247f3 \
                         GitHubRepo=pizzawithcola/arcade_ftw_be GitHubBranch=main \
                         GitHubRepoIds=pizzawithcola@68038650/arcade_ftw_be@1414195943 \
                         AdminUsername=admin AdminPassword=... SecretKey=... \
                         CertificateArn= DesiredCount=1

# 服务与滚动状态
aws ecs describe-services --cluster arcade-ftw-cluster --services arcade-ftw-service --region ap-south-1

# 日志
aws logs tail /ecs/arcade-ftw-backend --follow

# 健康检查（一律走 CloudFront：ALB 已不对公网开放）
curl -s -w ' %{http_code} %{content_type}\n' https://d39s9y5ma27s5.cloudfront.net/health
```

> ⚠️ `cloudformation deploy` 会**按栈里存的参数值**收敛：如果曾用 `DesiredCount=0` 建栈、后来又手动把服务拉到 1，下一次 deploy **必须显式带 `DesiredCount=1`**，否则会把服务打回 0。

排障要点见 [../aws/README.md](../aws/README.md) 的「排查手册」（CloudTrail 读 OIDC 失败、describe-stack-events 看栈失败）。

