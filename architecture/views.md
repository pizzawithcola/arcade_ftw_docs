# 三个视图与房间

阶段 1 的成果：把过去「一个页面扮演三个角色」拆成三条独立的 UI 路线，并把 **room** 放进 API 形状。

## 路由

| 路由 | 视图 | 谁能进 |
|---|---|---|
| `/` | 玩家入口（昵称 + 活动）| 所有人 |
| `/play` | User View：**自动进入当前轮**，等待室 + 游戏 | 玩家令牌 |
| `/r/:code` | User View 的固定版：钉在某个房间（排障用）| 玩家令牌 |
| `/display/:code` | Display View：大屏展示 | admin（含只读 display 令牌）|
| `/admin` | Admin Panel：侧边栏三块（Console / Data / Setting）| admin（**必须 full scope**）|

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

**room = 一轮游戏**。每轮由 admin 新建一个房间（`round-1`、`round-2`…，也可以自己给 code），玩家扫码加入后**绑定到那一轮**。所以房间列表是**历史**，最新创建的那个是「当前轮」。

房间有行：`rooms(code PK, event, label, created_at)`，`main` 由迁移 seed（第一轮之前的占位）。启动时 `RoomRegistry.load()` 读这张表建引擎；**读库失败只记警告并退回默认房间**——否则 DB 抖动会让任务启动失败、被 ALB 反复重启，反而把内存里的节目全清掉。

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
| 房间 增 | `POST /api/rooms` | code 可省略（自动 `round-N`），给了就必须匹配 `^[a-z0-9][a-z0-9-]{1,31}$`；**建库的同时立刻建引擎**，否则刚给你的房间码打开就是 404 |
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

### 当前轮（current）与大屏

`GET /api/rooms` 返回 `{current, rooms}`：`current` 是**最近创建**的房间（按 `created_at`），`rooms` 按新到旧排序。

- **大屏展示题目**：节点的 `questions` / `seconds_each` 让引擎算出**当前是第几题**（`snapshot.question`），大屏用这个索引渲染题面；选项只留在 iPad 上——六米外看不清四个选项。
- **大屏 `/display`（不带 code）自动跟随 `current`**：admin 点 `new room` 后大屏自己切到新房间，**URL 不用改**（每 3 秒轮询一次 `current`）。
- **大屏 `/display/:code` 仍然存在**，用于把某块屏固定在某一个房间（排障 / 复盘用）。
- 玩家侧**不再需要扫码**：登录后进 `/play`，自己问 `/api/rooms/current`（公开接口，只给 code + phase），跟随当前轮；每 3 秒问一次，换轮自动跟上。
- 本轮 `finished` → 玩家屏显示「That's a wrap」，**8 秒后清令牌回登录页**；下一轮开始时重新登录即进新房间。
- 一个设备在**轮次结束后才登录**（没参与过这一轮）不会被弹回登录页，而是停在「Between rounds」等下一轮 —— 否则会形成「登录 → 结束 → 再登录」的死循环。

> 玩家端同样每 3 秒问一次 `current`，换轮自动跟上；**不存在「换轮后要重扫」这一步**。

### Admin Panel 的结构

左侧是**分区侧边栏**，右侧是当前分区的内容：

| 分区 | 内容 | 数据来源 |
|---|---|---|
| **Arcade Console** | 单个控制面板：房间下拉 + **`new room`（开新一轮）** + `start/pause/resume/finish/reset` + 跳节点 | `GET /api/rooms`（2s 轮询） |
| **Arcade Data** | 一个 `Rooms / Players` **切换**，两边都是表格，并带 `+ new room` / `+ new player` 与每行的 `view / edit / delete` | `GET /api/rooms` + `GET /api/players?room=`（**仅此分区打开时** 3s 轮询） |
| **Game Setting** | 占位，留给房间默认值 / 节点时长 | — |

Arcade Data 的两张表：

| 表 | 列 |
|---|---|
| Rooms | Room、Label、Event、Phase、Node、Joined、Online、`view`（选中该房间并跳到 Players）；**最新一轮在最上面** |
| Players | Player、Event、Hallu、Data、Sabotage、Ultimate、Total（按 Total 倒序，无分数行读作 0） |

两张表**共用同一套表头、说明行和数字对齐**，切换时只有数据在变，视觉不跳。Rooms 表最后一列是动作列（表头留空只是 `aria-label`），不是数据列。

几个刻意的取舍：

- **没有独立的「房间列表」窗口**。以前左栏那排房间卡片占掉了半屏，而实际只有 `main` 一个房间。房间现在是 **Data 里的一张表**，需要选房间时用 Console 的下拉或表里的 `view`。
- **只用表格**：房间和玩家都是整齐的数字列，卡片塞不下 Joined / Online / 四个分项。
- **增删改在面板里做**：房间和玩家都能就地新增/编辑/删除（内联表单，不用弹窗），删除前 `confirm` 一次。分数仍然只能通过接口改 —— 改分比改名字危险得多。
- **令牌失效会退回登录**：轮询拿到 401 时面板/大屏回到登录页，而不是静默停止更新。
- **玩家表按分区打开才轮询**：名单没人看的时候不该每秒查一次库。
- **窄屏（< 52rem）时侧边栏变成一行 tab**，宽屏是 12rem 侧栏 + 内容区；表格在窄屏内横向滚动，不撑破页面。
- 分区状态**不持久化**，刷新回到 Console——操作员最常用的还是控制台。

## 令牌与角色

| scope | 主体 | 能读 | 能控制 |
|---|---|---|---|
| `full` | admin | ✅ | ✅ |
| `display` | admin | ✅ | ❌ |
| —（player）| `player:<id>` | 房间状态 | ❌ |

`POST /api/auth/admin` 接受 `scope: "display"`，签出只读令牌。这样**大屏可以整天保持登录**，而路过大屏的人无法暂停节目。

`require_admin` 要求 `scope === "full"`；`require_display` 接受两种 scope。

## Display View 的内容

- **idle = 等待室**：大字 **Waiting Room**、`N players joined`、以及**已加入玩家的名字墙**（2 秒轮询名单）。
  设备是我们发的 iPad，所以**没有二维码、没有网址、没有房间号**——大厅屏幕上不该出现需要人操作的东西。
  名字用胶囊格排布（`auto-fill` 自动列数），人多时区域内滚动，短名单自动垂直居中。
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
- **只有 `main` 一个房间**：建房间接口可用，但 Admin 面板里的房间操作仍是只读展示（没有「新建 / 删除」按钮）。
- **Admin 面板不能改分数**：房间/玩家的增删改已进面板，分数仍走 `/api/docs`。
- 玩家分数仍只存在于页面内存，刷新即清零。
