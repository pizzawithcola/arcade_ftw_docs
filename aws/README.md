# AWS

## 隔离策略

**除 VPC / 子网外全部新建**，资源名统一 `arcade_ftw-` 前缀。共享 VPC 的任何改动**必须先问用户**。

## 资源清单（尚未创建）

| 资源 | 标识 | 说明 |
|---|---|---|
| S3 bucket | ⬜ `arcade_ftw-frontend`（待定）| 前端静态文件 |
| CloudFront | ⬜ 新建分发 | **必须复制**行为白名单 + 403/404 → `200 /index.html` 错误重写 |
| ALB + Target Group | ⬜ 新建 | 不复用旧 ALB |
| ECS Cluster / Service / Task | ⬜ 新建 | **独立 service**（内存态，见红线 #3）|
| ECR repo | ⬜ `arcade_ftw-backend` | 后端镜像 |
| RDS | ⬜ 新实例（可先 SQLite 跑通）| 不碰旧库 |
| 安全组 | ⬜ 新建 | 避免误删旧 CloudFront 白名单 |
| 日志组 | ⬜ `/ecs/arcade_ftw-backend` | CloudWatch |
| 区域 / 账号 | 待定 | 旧系统为 `ap-south-1` / `866730904414` |

## 部署要点（沿用旧系统教训）

1. **后端镜像必须 `--platform linux/amd64`**（Mac 是 arm64）。
2. 后端 CI 需从零写对（旧系统 CI 是坏的，一直在手动部署）。
3. **CloudFront 200 伪装**：行为白名单 + 错误重写没配好，任何后端错误都会变成 `200 text/html`。
4. **调 API 带尾斜杠**（否则裸路径落 S3）。
5. 用 `curl -w '%{content_type}'` 区分 `application/json` vs `text/html` 来排查。

## 红线

- 绝不调用旧系统的破坏性接口（purge / 删除）。
- 验证新系统时 URL 用新域名。

## 待办

- [ ] 决定是否需要 IaC（Terraform / CDK）；旧手册建议先 IaC 化再复制环境
