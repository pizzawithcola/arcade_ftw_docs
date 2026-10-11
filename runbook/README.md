# 活动当天 Runbook

> ⬜ 待补充。活动前必须把下面的清单填满并演练一遍。

## 活动前

- [ ] 后端部署到最新镜像，确认所有 task 健康
- [ ] 前端产物已上线，`assets/index-*.js` 哈希确认更新
- [ ] 数据库快照已建
- [ ] 大屏（kiosk）自愈与硬重载已开启
- [ ] 二维码登录链路实机验证

## 活动中

- [ ] **禁止部署 / 重启后端容器**（会清空内存态房间）
- [ ] 管理员控制路径不影响进行中的房间

## 活动后

- [ ] 数据备份 / 归档
- [ ] 回补文档与本次事故记录

## 已知的部署操作

```bash
# 常规部署：走 CI（push main 即可），约 3.5 分钟。下面是排障/首次建栈用的。

# 基础设施（幂等；参数取值见 ../aws/README.md）
aws cloudformation deploy --region ap-south-1 \
  --template-file infra/cloudformation.yml \
  --stack-name arcade-ftw \
  --capabilities CAPABILITY_NAMED_IAM \
  --parameter-overrides VpcId=vpc-... PublicSubnet1=subnet-... PublicSubnet2=subnet-... \
                         TaskSubnet1=subnet-... TaskSubnet2=subnet-... ApiKey=... DesiredCount=1

# 服务与滚动状态
aws ecs describe-services --cluster arcade-ftw-cluster --services arcade-ftw-service --region ap-south-1

# 日志
aws logs tail /ecs/arcade-ftw-backend --follow

# 健康检查（大屏/API 域名）
curl -s -w ' %{http_code} %{content_type}\n' http://arcade-ftw-alb-1071955694.ap-south-1.elb.amazonaws.com/health
```

> ⚠️ `cloudformation deploy` 会**按栈里存的参数值**收敛：如果曾用 `DesiredCount=0` 建栈、后来又手动把服务拉到 1，下一次 deploy **必须显式带 `DesiredCount=1`**，否则会把服务打回 0。

排障要点见 [../aws/README.md](../aws/README.md) 的「排查手册」（CloudTrail 读 OIDC 失败、describe-stack-events 看栈失败）。

