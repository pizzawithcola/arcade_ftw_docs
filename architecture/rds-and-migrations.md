# RDS 与数据库迁移链路

> **状态：已实施并验证（2026-10-11）**。CI run #6 端到端成功：构建 → 推 ECR → 注册 revision → 一次性任务跑 `alembic upgrade head` → 滚动更新 → 验证 `/health` 与 `/health/db`，全程 **4 分 37 秒**。
> CloudWatch 里可见迁移任务的输出：`Running upgrade -> 0d85f32b238d, initial players and scores`。
> 决策已记入 [ADR 索引](./decisions/README.md)。

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

当前 schema（迁移 `5f2d983893bf` 之后）：

```sql
rooms(code, event, label, created_at)
        -- 主键 (code)；迁移 seed 了 'main'
        -- 房间的真相来源：启动时 Registry 读这张表建引擎

players(id, nickname, event, room_code, created_at)
        -- UNIQUE (event, nickname)
        -- room_code 可空，FK → rooms(code) ON DELETE SET NULL，已建索引

scores(player_id, hallucination_score, data_score, sabotage_score, ultimate_score,
       total_score, updated_at)
        -- 主键 (player_id)，FK → players(id) ON DELETE CASCADE
        -- total_score 是数据库生成列 = 四个分数之和
        -- 四个分项 = 四个分区（游戏 + 其后的 quiz），见 views.md

score_events(player_id, event_key, points, created_at)
        -- 主键 (player_id, event_key)，FK → players(id) ON DELETE CASCADE
        -- 分数上报的幂等记录；event_key = 房间:节点序号:题号|total
        -- 只有真正插入的那条才累加 scores，重复上报是 no-op
```

**房间从「内存里的默认值」变成数据**：`RoomRegistry` 启动时读 `rooms` 建每个房间的引擎，
`main` 由迁移 seed；`players.room_code` 记录成员归属，玩家登录就建行、进房间才写归属。
删房间把成员置空（`ON DELETE SET NULL`）而不是删人——人是先于房间存在的。

> ⚠️ **这里与既有决策相反，是有意的改动。** 早期规则是「计分用长表 / JSONB，不硬编码
> `game1~5_score`」，理由是加新环节不必改表。现在**明确改成宽表**：演出固定四场游戏，
> 被问的查询是「按 total 排名的排行榜」，宽表让这类查询不必 GROUP BY。
> **代价：加第五个游戏需要一次迁移。**
>
> `total_score` 用 `GENERATED ALWAYS AS (...) STORED` 而不是应用层求和——这样
> **total 永远不可能和四个分项不一致**，无论哪条代码路径写入。
> 直接 `UPDATE total_score` 会被数据库拒绝（实测）。

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

## 5. 已定决策（实施的实际取值）

1. **实例规格**：`db.t4g.micro`（**不用** `db.t3.micro`）。
2. **迁移方式**：CI 里跑**一次性 ECS 任务**（`run-task` + container override `["alembic","upgrade","head"]`），失败即中止、不动线上。
3. **连接方式**：async SQLAlchemy 2 + asyncpg。
4. **首个迁移**：只建 `players` 与 `scores`（当时是长表，已由 `c4f1a9b7e2d5` 取代为宽表）。

### 实测得到的细节

| 项 | 实际值 |
|---|---|
| 实例 | `arcade-ftw-db`，PostgreSQL **16.15**，`PubliclyAccessible=false`，`MultiAZ=false`，存储加密 ✅，备份保留 7 天，删除保护 ✅ |
| 子网组 | 复用现有 `default-vpc-0dda76b286c45fc27`（**未新建**，共享 VPC 未改动）|
| 安全组 | `arcade-ftw-rds-sg`，5432 仅允许来自 `arcade-ftw-ecs-sg` |
| 端点 | `arcade-ftw-db.ctao8iuy2toz.ap-south-1.rds.amazonaws.com:5432` |
| 密钥 | RDS 托管，ARN `.../secret:rds!db-57437a62-...-iRC7r4`，JSON 键为 `['password','username']` |
| 迁移 revision | 当前 head `e8b3d5c1a712`（rooms 与成员归属）；首个迁移 `0d85f32b238d` 含可用的 `downgrade()` |

> ⚠️ **`EngineVersion` 必须给具体小版本**（如 `16.15`）——只写 `16` 会被 RDS 拒绝（`deprecated ... cannot be used to create new RDS DB instances`）。已做成参数便于将来升级。
> ⚠️ **`PreferredBackupWindow` 只接受 `hh24:mi-hh24:mi`**，不带星期前缀；**维护窗口**才用 `ddd:hh24:mi-ddd:hh24:mi` 形式。

## 6. 已完成

- [x] 模板加 RDS + `arcade-ftw-rds-sg` + 执行角色 SecretsManager 权限 + 任务定义 `DB_*` env/secret
- [x] 后端加 SQLAlchemy 2 async + asyncpg + Alembic，写首个 migration
- [x] `config.py` 拼装 DSN、加 `DB_*` 变量
- [x] CI 加「一次性迁移任务」步骤 + IAM 权限（`ecs:RunTask` / `ecs:DescribeTasks` / `iam:PassRole`）
- [x] 本地 docker-compose 起 Postgres（宿主 **5433**，避开开发机自己的 5432），跑通迁移与回滚
- [x] 部署验证：`/health` 与 `/health/db` 均 200
- [x] 回补 [aws/README.md](../aws/README.md) 与 [runbook](../runbook/README.md)

## 7. 已知限制

- **迁移不可自动回滚**：代码能回滚，schema 不能。迁移脚本必须自带可用的 `downgrade()`，回滚靠人工决定。
- 部署时间从 3.5 分钟增加到 **约 4.5–5 分钟**（多一次迁移任务）。
- `rooms` / `players.room_code` 已有 API 使用（建房间 / join / 名单）。`scores` 有三个写入方：玩家上报（`POST /api/me/score`，按题幂等）与管理员设值 / 增量（`PUT` / `adjust`）。逐题**答案**没有保存，所以只能做分数累加，做不了选项分布。
- 应用启动**不建表**，schema 只由迁移创建。
