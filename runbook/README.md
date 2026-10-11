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
# 部署基础设施（幂等）
aws cloudformation deploy \
  --template-file infra/cloudformation.yml \
  --stack-name arcade-ftw \
  --capabilities CAPABILITY_NAMED_IAM \
  --parameter-overrides VpcId=... PublicSubnet1=... PublicSubnet2=... \
                         TaskSubnet1=... TaskSubnet2=... ApiKey=...

# 查看服务与滚动状态
aws ecs describe-services --cluster arcade-ftw-cluster --services arcade-ftw-service

# 看日志
aws logs tail /ecs/arcade-ftw-backend --follow
```

CI 是主路径（`push main` 自动部署）；上面的命令用于首次建栈和排障。

