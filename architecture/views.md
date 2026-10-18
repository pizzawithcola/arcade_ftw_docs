# 三个视图与房间

阶段 1 的成果：把过去「一个页面扮演三个角色」拆成三条独立的 UI 路线，并把 **room** 放进 API 形状。

## 路由

| 路由 | 视图 | 谁能进 |
|---|---|---|
| `/` | 玩家入口（昵称 + 活动）| 所有人 |
| `/r/:code` | User View：等待室 + 游戏 | 玩家令牌 |
| `/display/:code` | Display View：大屏展示 | admin（含只读 display 令牌）|
| `/admin` | Admin Panel：房间控制与监视 | admin（**必须 full scope**）|

`/status` 保留为诊断页。

## 为什么要拆

拆分前 `GamePage` 同时承担：玩家作答、大屏展示、操作员控制。任何持 admin 令牌的页面都会渲染 Operator 面板——**大屏上出现控制按钮**是不可接受的。拆分后控制只存在于 `/admin`。

## 房间

**即使只跑一场，room 也在 API 路径里**：

```
/api/rooms/{code}/state
/api/rooms/{code}/start | pause | resume | finish | reset | go/{index}
/api/rooms/{code}/join            # 玩家把自己记进该房间
/api/rooms/{code}/players         # 房间名单（玩家只见昵称，admin/display 见总分）
/api/players/{id}/score           # 分数读写（管理员）
/api/rooms                        # 房间 CRUD（POST 建、PATCH 改、DELETE 删）
/api/ws?room={code}
```

默认房间 `main`。理由：并行活动是小概率事件，但真发生时才改，就要动每一处 fetch、WS 协议和前端路由，而那天通常是活动当天。

引擎按房间隔离（`app/rooms.py` 的 `RoomRegistry`），每个房间有自己的时钟、进度和广播目标。**未知房间返回 404**（不会静默新建，避免打错 URL 就出现一场空节目）。

### 房间是数据

房间现在有行：`rooms(code PK, event, label, created_at)`，`main` 由迁移 seed。启动时 `RoomRegistry.load()` 读这张表建引擎；**读库失败只记警告并退回默认房间**——否则 DB 抖动会让任务启动失败、被 ALB 反复重启，反而把内存里的节目全清掉。

成员归属记在 `players.room_code`（可空，`ON DELETE SET NULL`）。登录就建 `players` 行，**进房间才写 `room_code`**：`POST /api/rooms/{code}/join`，玩家端每次进页面都调，所以刷新 / 换 Wi-Fi 后仍然在原房间。删房间只是把成员置空，不会删人。

> `GET /api/rooms` 的两个数字不要混：`players` 是**库里加入该房间的人数**，`clients` 是**当前 WS 连接数**。iPad 熄屏、切后台都会让 `clients` 掉下去，而 `players` 不变。

> ⚠️ 引擎仍是**进程内内存态**。两个房间在一个进程里可以并行，但**一次部署会把所有房间清零**。

### 房间名单（阶段 1.5）

`GET /api/rooms/{code}/players` 给**任何已登录身份**（`require_signed_in`），但**按身份裁剪内容**：

| 身份 | 拿到什么 |
|---|---|
| admin / display | `player_id`、`nickname`、`event`、`total_score` |
| 玩家 | **只有 `nickname`** |

玩家等待室要显示「房间里有谁」，所以这条必须对玩家开放；但分数和活动是给操作员和大屏的，
否则每台 iPad 都在轮询一张排行榜。裁剪在同一个接口里做，前端不需要第二个端点。

- 读法是 `players LEFT JOIN scores`，按 `total_score` 倒序；**没有分数行的人也在名单上，读作 0**——人从进房间那一刻就该看得见，而不是等打完第一局。
- 等待室每 **3 秒** 轮询一次（名单不走 WS：socket 传的是节目，不是宾客名单），自己的名字高亮。
- 昵称在房间内唯一（重名自动编号），所以可以作为列表的 key。

### 管理接口（阶段 2 的第一块）

房间、玩家、分数都能增删改查。全部走 HTTP + `require_admin` / `require_display`，在 `/api/docs` 里可以直接点。

| 操作 | 接口 | 关键约束 |
|---|---|---|
| 房间 增 | `POST /api/rooms` | code 必须匹配 `^[a-z0-9][a-z0-9-]{1,31}$`；**建库的同时立刻建引擎**，否则刚给你的房间码打开就是 404 |
| 房间 查 | `GET /api/rooms`、`GET /api/rooms/{code}` | 列表给 `players`（库里人数）与 `clients`（活的连接） |
| 房间 改 | `PATCH /api/rooms/{code}` | 只能改 `event` / `label`；**code 不可改**（已在 URL、已印的二维码、玩家归属里） |
| 房间 删 | `DELETE /api/rooms/{code}` | **playing / paused 时拒绝（409）**——删引擎会把房间里每台 iPad 晾在半局 |
| 玩家 增 | `POST /api/rooms/{code}/players` | 操作员替玩家建成员（设备掉了人还在名单上），重名照常编号 |
| 玩家 查 | `GET /api/players`、`GET /api/players/{id}` | 支持 `?room=` / `?event=` 过滤，带 `total_score` |
| 玩家 改 | `PATCH /api/players/{id}` | 改昵称 / 活动 / 房间；`room_code: null` = 移出房间，**省略该字段 = 不动** |
| 玩家 删 | `DELETE /api/players/{id}` | 分数随外键**级联删除** |
| 分数 改 | `PUT /api/players/{id}/score`、`POST /api/players/{id}/score/adjust` | 前者**设置**分项（未给的保持不变，首次写入建行），后者按**增量**加减；`total_score` 是生成列，不接受写入 |

分数为什么要两个接口：游戏上报的是「你得到 +20」，而运营手动纠错时想的是「他应该是 30」。
同一条路径上放 `delta` 字段会让「要设成 0」和「减 0」长得一样，所以拆成两个语义明确的动作。

> ⚠️ 这些接口目前**没有 Admin 面板 UI**，只有 Swagger（`/api/docs`）和 HTTP。UI 留给下一步。

## 令牌与角色

| scope | 主体 | 能读 | 能控制 |
|---|---|---|---|
| `full` | admin | ✅ | ✅ |
| `display` | admin | ✅ | ❌ |
| —（player）| `player:<id>` | 房间状态 | ❌ |

`POST /api/auth/admin` 接受 `scope: "display"`，签出只读令牌。这样**大屏可以整天保持登录**，而路过大屏的人无法暂停节目。

`require_admin` 要求 `scope === "full"`；`require_display` 接受两种 scope。

## Display View 的内容

- **idle**：大字「Join the game」+ **二维码**（指向 `/?room=<code>`）+ 房间号
- **playing / paused**：节点序号与标题、**大倒计时**、以及该轮可以静态呈现的内容
- **finished**：结束语

**大屏始终显示倒计时**——大屏上只有一个时钟，不像玩家屏幕会和游戏自己的计时器打架。

> ⚠️ 目前只有 **Data Shadow** 能在大屏显示真实内容（它是一条静态帖子）。Hallucination 和 Quiz 依赖「当前是第几题」的**设备内状态**，服务端不知道，所以大屏暂时只显示「Answer on your iPad」。等答案上报链路（阶段 3）落地，服务端才知道当前题目，大屏才能显示题面与分布。

## 玩家身份：一台设备一个人

因为是人手一台 iPad，登录身份**不再靠昵称唯一**：

- 客户端登录时带上**自己已有的令牌**；令牌有效且活动一致 → 认回同一个玩家（重开页面/刷新不丢）。
- 没有令牌且昵称已被占用 → 自动变成 `Alex (2)`、`Alex (3)`，**不会把两个人合并成一条分数**。
- 唯一键仍是 `(event, nickname)`，因为它约束的是「同一个活动内显示名不重复」。

这条不是锦上添花：人手一台时重名几乎必然发生，而错误合并要等到分数落库（阶段 3）才会暴露。

## 已知限制（阶段 1）

- **还没有答案上报**，所以大屏没有聚合数据；名单里的总分此刻恒为 0（`scores` 没有写入方）。
- **只有 `main` 一个房间**：建房间的接口已经可用，但没有 Admin 面板 UI，也没有「一屏看多个房间」之外的房间管理界面。
- 玩家分数仍只存在于页面内存，刷新即清零。
