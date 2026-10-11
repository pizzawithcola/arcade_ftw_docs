# RDS 与数据库迁移链路

> **状态：proposed（待用户确认）**。本文件描述「让数据库改动也能一条命令部署」的方案。
> 决策定稿后请更新状态与 [ADR 索引](./decisions/README.md)。

## 1. 目标

| 目标 | 说明 |
|---|---|
| 数据库改动可部署 | `git push main` 一次，schema 迁移与代码一起上线 |
| 完全隔离 | 不碰旧库 `cyber-ai-festival-db-recovered` |
| 不污染真实库 | 本地测试绝不连生产 RDS（旧后端铁律）|
| 不引入明文凭据 | 不用密码明文 env |

## 2. 现状（实测）

| 事实 | 值 |
|---|---|
| 旧 RDS | `cyber-ai-festival-db-recovered`，`db.t3.micro`，PostgreSQL **16.13**，20GB，单 AZ，`PubliclyAccessible=False` |
| 所在 VPC | `vpc-0dda76b286c45fc27`（**与旧系统共享，只引用**）|
| 现有 DB subnet group | `default-vpc-0dda76b286c45fc27`，覆盖该 VPC 全部 3 个子网 |
| 该 VPC 的子网 | **全部是公有子网**（`MapPublicIpOnLaunch=True`），**没有私有子网** |
| ECS 任务子网 | `subnet-01b3aa683790a0e8c`(1a) / `subnet-045ccd04e547efddb`(1b) |

**关键结论**：沿用旧系统的成熟做法——RDS 放进**已有的 default DB subnet group**、`PubliclyAccessible=False`、由安全组限制来源。**不需要新建子网，不需要改动共享 VPC。**

> 公有子网里的 RDS 只要 `PubliclyAccessible=False` 且安全组只放行 ECS 安全组，就不会从公网可达。同 VPC 内的 ECS 任务通过私有 IP 直连，**不依赖 NAT**。

## 3. 方案

### 3.1 基础设施（加入 `arcade_ftw_be/infra/cloudformation.yml`）

| 资源 | 设计 |
|---|---|
| `arcade-ftw-db` | PostgreSQL 16（与旧库同大版本），规格见「待定」|
| 凭据 | **RDS 托管的 Secrets Manager 主密码**（`ManageMasterUserPassword: true`），不留明文 |
| 安全组 | 新建 `arcade-ftw-rds-sg`，**5432 仅放行来自 `arcade-ftw-ecs-sg`** |
| DB subnet group | 复用 `default-vpc-0dda76b286c45fc27`（只引用）|
| 备份 | 保留 7 天，维护窗口错开旧库 |
| 删除保护 | 开启 |

### 3.2 应用侧连接

不用把整条 URL 塞进 secret（CFN 无法拼接密码）。改为：

- **普通环境变量**：`DB_HOST`（`!GetAtt DbInstance.Endpoint.Address`）、`DB_PORT`、`DB_NAME`、`DB_USER`
- **Secret**：`DB_PASSWORD`（取自 RDS 托管密钥的 `password` 键，`FromSecret: <secretArn>:password::`）
- 应用在 `config.py` 里**拼装 DSN**

执行角色需加 `secretsmanager:GetSecretValue`（限定到该密钥 ARN）。

### 3.3 ORM 与迁移

- **SQLAlchemy 2 async + asyncpg**（旧手册 §13.5 的建议，避免同步调用阻塞事件循环）
- **Alembic** 管理 schema：`alembic.ini` + `app/migrations/`
- 计分表按既有决策用**长表**，不硬编码 `game1~5_score`：

```sql
players(id, nickname, country, created_at)
scores(player_id, stage_key, score, updated_at)   -- 主键 (player_id, stage_key)
```

### 3.4 迁移的执行方式（推荐：CI 里跑一次性 ECS 任务）

```
git push main
  → 构建 & 推 ECR
  → aws ecs run-task（同镜像，command = ["alembic","upgrade","head"]）→ 等退出码 0
  → 拉线上 task definition 改镜像 → update-service → 验证 /health
```

**为什么不选另外两种**：
- ❌ **CI 直连数据库**：RDS `PubliclyAccessible=False`，GitHub runner 根本连不上。
- ❌ **容器启动时自动迁移**：滚动更新时新旧容器会短暂并存（max 200%），两个进程并发迁移会打架。

CI 角色需补权限：`ecs:RunTask`、`ecs:DescribeTasks`、`iam:PassRole`。

### 3.5 本地开发与测试

- **本地用 Docker 起 Postgres**（Docker Desktop 已就绪），不连生产库。
- 测试必须指向本地库 / 临时库——**绝不污染真实数据库**（旧后端铁律）。

## 4. 影响与代价

| 项 | 值 |
|---|---|
| 新增成本 | 约 **$15–16/月**（实例 + 20GB gp3 存储）|
| 部署耗时 | 从 3.5 分钟增加到约 **5–6 分钟**（多一次迁移任务）|
| 风险 | 迁移失败会**阻断部署**（先于 update-service），这是刻意的 |
| 回滚 | 代码可回滚；**迁移不可自动回滚**，需另写 downgrade 或手工处理 |

## 5. 待定（需用户确认）

1. **实例规格**：`db.t4g.micro`（Graviton，更便宜）还是与旧系统一致的 `db.t3.micro`？
2. **迁移方式**：确认采用「CI 跑一次性 ECS 任务」？
3. **连接方式**：确认 async SQLAlchemy + asyncpg？
4. **首次迁移内容**：只建 `players` / `scores`，还是同时把房间/答题表也建起来？

## 6. 落地步骤（确认后执行）

- [ ] 模板加 RDS + 安全组 + 执行角色 SecretsManager 权限
- [ ] 后端加 SQLAlchemy 2 async + asyncpg + Alembic，写首个 migration
- [ ] `config.py` 拼装 DSN、加 `DB_*` 变量
- [ ] CI 加「一次性迁移任务」步骤 + IAM 权限
- [ ] 本地 docker-compose 起 Postgres，跑通迁移与回滚
- [ ] 部署验证：`/health` + 一次真实读写冒烟
- [ ] 回补 [aws/README.md](../aws/README.md) 与 [runbook](../runbook/README.md)
