# 三个视图与房间

阶段 1 的成果：把过去「一个页面扮演三个角色」拆成三条独立的 UI 路线，并把 **room** 放进 API 形状。

## 路由

| 路由 | 视图 | 谁能进 |
|---|---|---|
| `/` | 玩家入口（昵称 + 活动）| 所有人 |
| `/play` | User View：**自动进入当前轮**；等待室三行字，开演后是游戏 | 玩家令牌 |
| `/r/:code` | User View 的固定版：钉在某个房间（排障用）| 玩家令牌 |
| `/display` | Display View：大屏展示，自动跟随当前轮 | admin（含只读 display 令牌）|
| `/display/:code` | Display View 的固定版：钉在某个房间（排障 / 复盘）| admin（含只读 display 令牌）|
| `/admin` | Admin Panel：侧边栏三块（Console / Data / Setting）| admin（**必须 full scope**）|
| `/leaderboard` | 公开总榜：前 10 名总分，自动刷新；默认单列，纵向放不下时才变两列 | 所有人（无鉴权）|

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
/api/me/score                     # 玩家上报自己某一题/整节点的得分
/api/rooms                        # 房间（POST 建、GET 查、DELETE 删；房间是一次性的，不可改）
/api/ws?room={code}
```

房间是一次性的：admin 每轮新建一个（自动 `R-001`），结束后 `finish` 把它关闭，下一轮再建新的。没有「默认房间」。

引擎按房间隔离（`app/rooms.py` 的 `RoomRegistry`），每个房间有自己的时钟、进度和广播目标。**未知房间返回 404**（不会静默新建，避免打错 URL 就出现一场空节目）。

### 房间是数据

**room = 一轮游戏，一次性**。每轮由 admin 新建一个房间（`R-001`、`R-002`…）。玩家登录后进 `/play`，自己跟随 `current` 并调用 `join`，从而**绑定到那一轮**；不再有扫码这一步。房间列表是**历史**：`closed_at` 为空的房间只有一个——最新创建的那一个，即「当前轮」；`finish` 之后它关闭，下一轮是全新的房间。

房间由 **code + 创建时间**标识。当前 API 创建房间只接受 `code`，不提供 `label` / `event` 编辑能力；模型中仍保留一个 nullable `Room.event` 字段，但新建流程不写入它。谁参加了这一轮记在独立的 `room_participants(room_code, player_id, joined_at)` 表里（复合主键、级联删除），**它不随房间关闭而清空**，所以历史房间永远能回答「这轮有谁」。

启动时 `RoomRegistry.load()` 读这张表为每个房间建引擎；**读库失败只记警告并退回内存 fallback `main`**——否则 DB 抖动会让任务启动失败、被 ALB 反复重启，反而把内存里的节目全清掉（该 fallback 只存在于进程内，不写库）。

成员归属另有一份「当前房间」绑定在 `players.room_code`（可空，`ON DELETE SET NULL`）：登录就建 `players` 行，**进房间才写 `room_code`**（`POST /api/rooms/{code}/join`，玩家端每次进页面都调）。`finish` 时把该房间所有成员的 `room_code` 和 `team` 一并清空——**关闭的房间不再绑定任何玩家**，所以下一场绝不可能「还看到上一场的玩家」。删房间则把成员置空，不会删人。

> `GET /api/rooms` 的两个数字不要混：`players` 是**加入该房间的人数**（来自 `room_participants`，房间关闭后仍在），`clients` 是**当前 WS 连接数**。iPad 熄屏、切后台都会让 `clients` 掉下去，而 `players` 不变。

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
- **玩家等待室只显示三行**：`AI Cyber Arcade` + 活动名、`Hi, <昵称>`、`Waiting for the host to start the show…`，屏幕居中，**没有名单、没有时钟、没有顶栏**。
  名单是给大厅屏幕（大屏）看的，玩家自己的 iPad 上列一遍别人的名字没有意义。
- 大屏的等待室每 **2 秒** 轮询一次名单（不走 WS：socket 传的是节目，不是宾客名单）。

### 分数上报（玩家自助）

**四个分区**，一个分区 = 一场游戏 + 紧随其后的 quiz：

| `scores` 列 | 计入的节点 |
|---|---|
| `hallucination_score` | Hallucination + Quiz round 1 |
| `data_score` | Data Shadow + Quiz round 2 |
| `sabotage_score` | Sabotage |
| `ultimate_score` | Ultimate Quiz |

节点在 `flow.py` 里用 `score: <分区>` 声明归属；**服务端据此决定写哪一列，客户端不能指定列**。

上报接口 `POST /api/me/score`：

```json
{ "room": "R-001", "node_index": 1, "question": 0, "points": 20 }
```

- **玩家令牌**（`require_player`）：管理员令牌会被拒绝，操作员不参与比赛。
- **必须已加入该房间**：`player.room_code` 不匹配 → 409。
- **节点必须声明了 `score`**：intro / outro / 未知节点 → 422。
- **按题幂等**：事件键由服务端拼成 `房间:节点序号:题号`（整节点提交用 `total`），写入 `score_events`（主键 `(player_id, event_key)`）与累加分区在同一事务里。重复上报返回 `applied: false`，分数不变。
- **允许负分**：Hallucination 的质疑侧与 Data Shadow 的误报都会扣分；0 分前端直接跳过。

为什么按「题」而不是「节点」幂等：一局有多题，用节点键会让整局只能计一次；用题号既能让每题各自计分，又能让重试和「回退重跑同一节点」都不重复计分。

> ⚠️ **分数由客户端计算**：题目目前在前端（`src/games/quiz/bank.ts`，20 题，每轮确定性抽 5 题），后端没有题目数据，所以只能信任设备上报（单条限 ±1000）。题库搬到后端后才有可能改成服务端计分。

### 榜单（游戏内 section 榜 + 公开总榜）

**游戏内 section 榜**：Hallucination、Data Shadow 与 Sabotage 结束后，主流程各插一个 5 秒 `leaderboard` 节点（`type: leaderboard`，`section: <分区>`）。Ultimate 后直接进入活动总榜，不再重复播分节榜。**大屏与玩家平板都渲染 section 榜**——房间按该 section 的分数排名，分数来自对应的 `scores` 列，服务端已落库。

数据接口 `GET /api/rooms/{code}/leaderboard?section=hallucination|data|sabotage|ultimate|total`：`require_signed_in`（玩家 / display / admin 都能读），每行返回 `{rank, nickname, score, previous, total}`。`score` 是当前分区得分，`previous` 是进入该分区前的累计分（最低为 0），`total` 是当前累计分；排名按 `total` 倒序，无 score 行读作 0。榜单是房间级数据，所以玩家平板看的是「自己房间」的排名，不是全网。

**公开总榜** `GET /api/leaderboard?limit=10`：**无鉴权**，跨房间按 `total_score` 倒序，只暴露 `rank / nickname / total_score`（不泄露 player_id / event / team / room），挂在观众后方的公开屏上。

两个榜单共用同一个 `Leaderboard` 组件：**默认单列，只有纵向放不下（超过窗口底部）时才切成两列**；短榜单无论横竖屏都是一列往下排。

视觉上它是「仪表」而不是「打印件」：

- 顶部一条会流动的红色光轨（`.ldb__rail`），整块板子背后一团红色辉光（`.ldb::before`）；
- 标题是两端切角的牌子（`clip-path` + 红色渐变），不是一行浮空的字；
- 面板（`.ldb__frame`）带角标、顶部高光边和内阴影，像一块嵌入式屏幕；
- `RANK / PLAYER / SCORE` 三列标签和行用**同一个 grid**，标签正对数据；
- 名次牌是斜切的 `01…10`；榜首那行底色泛红、名字加粗前面带 ▲；
- 每行背后按「该行分数 ÷ 榜首分数」铺一条**带刻度的红色电平条**（`--fill` + 重复渐变），
  一眼能看出分差，不用比对数字；0 分的行不画条（0% 的条仍会画出那条亮红右边，像是有一分）。

行高固定，所以两列切换的测量逻辑不受影响。

### 最终榜（`section: "total"`）：一叠箱子

活动总榜不是「渲染完就摆在那儿」，而是**一摞慢慢垒起来的箱子**（`Leaderboard` 的 `pile` 模式）：

- **从最后一名开始**：第 10 名先从屏幕上方落下；接着第 9 名压在上一条肩上落下，**已经落好的每条会被压得往下沉一点**（`PILE_PRESS_PX = 9`，240ms 回弹）。
- 依此类推向上垒，直到**前三名**出场——它们落下时**放光**（红色外发光）、**放大回弹**（`PILE_POP_MS`）、以及一道横扫的高光，名次越高越大声。
- 位置不是用 CSS 动画算的，而是 `pileAt(clock, index, count)` 每帧算出来的：落点、被压的位移、前三名的 pop 比例，都在一个纯函数里。原因很实际——同一帧里既要「自己下落」又要「被别人压」，两条 CSS 动画抢同一个 `transform`，后者会盖掉前者。
- 最终榜**永远单列**（两列不是一摞）；超过 **10 人**就退回普通榜单（`PILE_MAX_ROWS`），因为一列放不下，垒起来也没意义。
- `prefers-reduced-motion` 下只保留落位，不做放光和横扫。

数据来自同一个 `GET /api/rooms/{code}/leaderboard?section=total`（按 `total_score` 倒序），所以大屏和平板同时开始垒、垒出同一个结果。

### 管理接口（阶段 2 的第一块）

房间、玩家、分数都能增删改查。全部走 HTTP + `require_admin` / `require_display`，在 `/api/docs` 里可以直接点。

| 操作 | 接口 | 关键约束 |
|---|---|---|
| 房间 增 | `POST /api/rooms` | code 可省略（自动 `R-001`，零填充三位），给了就必须匹配 `^[A-Za-z0-9][A-Za-z0-9-]{1,31}$`；**建库的同时立刻建引擎**，否则刚给你的房间码打开就是 404 |
| 房间 查 | `GET /api/rooms`、`GET /api/rooms/{code}` | 列表给 `players`（该轮参与人数）与 `clients`（活的连接），并带 `created_at` / `closed_at` |
| 房间 删 | `DELETE /api/rooms/{code}` | **playing / paused 时拒绝（409）**——删引擎会把房间里每台 iPad 晾在半局 |
| 玩家 增 | `POST /api/rooms/{code}/players` | 操作员替玩家建成员（设备掉了人还在名单上），重名照常编号 |
| 玩家 查 | `GET /api/players`、`GET /api/players/{id}` | 支持 `?room=` / `?event=` 过滤，带 `total_score` |
| 玩家 改 | `PATCH /api/players/{id}` | 改昵称 / 活动 / 房间；`room_code: null` = 移出房间，**省略该字段 = 不动** |
| 玩家 删 | `DELETE /api/players/{id}` | 分数随外键**级联删除** |
| 分数 改 | `PUT /api/players/{id}/score`、`POST /api/players/{id}/score/adjust` | 前者**设置**分项（未给的保持不变，首次写入建行），后者按**增量**加减；`total_score` 是生成列，不接受写入 |

分数为什么要两个接口：游戏上报的是「你得到 +20」，而运营手动纠错时想的是「他应该是 30」。
同一条路径上放 `delta` 字段会让「要设成 0」和「减 0」长得一样，所以拆成两个语义明确的动作。

### 当前轮（current）与大屏

`GET /api/rooms` 返回 `{current, rooms}`：`current` 是**最新一个未关闭**（`closed_at IS NULL`）的房间，`rooms` 按新到旧排序（历史房间全在里面，带 `closed_at`）。

- **大屏展示题目**：节点的 `questions` / `seconds_each` 让引擎算出**当前是第几题**（`snapshot.question`），大屏用这个索引渲染题面；选项只留在 iPad 上——六米外看不清四个选项。
- **大屏 `/display`（不带 code）自动跟随 `current`**：admin 点 `new room` 后大屏自己切到新房间，**URL 不用改**（每 3 秒轮询一次 `current`）。
- **大屏 `/display/:code` 仍然存在**，用于把某块屏固定在某一个房间（排障 / 复盘用）。
- 玩家侧**不再需要扫码**：登录后进 `/play`，自己问 `/api/rooms/current`（公开接口，只给 code + phase），跟随当前轮；每 3 秒问一次，换轮自动跟上。
- 本轮 `finished` → 玩家屏显示「That's a wrap」，**8 秒后清令牌回登录页**；下一轮开始时重新登录即进新房间。

### 收尾屏（`type: outro`，Wrap up）

最后一个节点是**打字机效果的收尾页**（`src/games/WrapUp.tsx`），大屏和 iPad 是同一个组件、同一段文字：

> Thank you for your participation. AI is evolving fast, and the AI Cyber Arcade brings to life how we can use it safely. Embrace the change and **Remember to keep a human in the loop.**

- 逐字显示（**30ms/字**，约 6 秒打完，节点 10 秒留 4 秒回读）；进度由**节点时钟**驱动（`elapsedMs`），所以暂停时打字也会停。
- 最后一句是 `<strong class="wrapup__call">`，**红色 + 发光**；其余白色。这是全站唯一一处正文级红字——它出现在整场的最后一屏，字号够大（大屏 `clamp(1.6rem,3.2vw,2.6rem)`）。
- 光标是一个闪烁的方块，打完自动隐藏；`prefers-reduced-motion` 下不打字、不闪。
- 注意：chassis 是**三行 grid**（tab / bar / stage），所以这屏也必须给一个 `hud__bar`，否则 stage 落在中间那行、整个框会塌成文字高度（踩过）。
- **`finish` 同时清空该房间的成员**（`room_code` / `team` 置空）：一轮正式结束即散场，`start` 重开同一个房间得到的是空房间，不会残留上一场的玩家。玩家行与分数都保留（只解除房间归属），复盘查人不受影响。
- 一个设备在**轮次结束后才登录**（没参与过这一轮）不会被弹回登录页，而是停在「Between rounds」等下一轮 —— 否则会形成「登录 → 结束 → 再登录」的死循环。

> 玩家端同样每 3 秒问一次 `current`，换轮自动跟上；**不存在「换轮后要重扫」这一步**。

### Admin Panel 的结构

左侧是**分区侧边栏**，右侧是当前分区的内容：

| 分区 | 内容 | 数据来源 |
|---|---|---|
| **Arcade Console** | 单个控制面板：**Show control**（`start/pause/resume/finish/reset` + **16:9 的大屏实时预览**，`/display/{code}` 内嵌 iframe，高度约窗口 40%）→ 分隔线 → **Leaderboard Event** 切换（`All events` / 单个活动，决定大屏、等待室与公开榜排哪个活动）→ 分隔线 → **Node Control**（按 section 分组的跳节点按钮） | `GET /api/rooms`（2s 轮询，快照带 `flow`）+ iframe 自己的 WS + `GET/PUT /api/settings` |
| **Arcade Data** | 一个 `Rooms / Players` **切换**，两边都是表格；Rooms 是纯历史（只读 + 删除），Players 带 `+ new player` 与每行的 `edit / delete` | `GET /api/rooms` + `GET /api/players?room=`（**仅此分区打开时** 3s 轮询） |
| **Event Report** | **只读**报表：六张卡片（Registered / Played / Average / Best / Rounds / Joins）、Games（含平均分占满分的比例条）、Played per day、三场活动横向对比、前 10 榜单 | `GET /api/events`、`GET /api/events/{key}/summary`、`GET /api/events/{key}/leaderboard`（10s 轮询；横向对比顺带拉全部活动的 summary） |
| **Game Setting** | 只读展示 Ultimate Quiz 的抽题规则与完整题库；题库目前在前端维护 | `src/games/quiz/` |

Arcade Data 的两张表：

| 表 | 列 |
|---|---|
| Rooms | Room（当前轮只有薄荷绿左边条标记，无 `current` 文字）、Created、Players、Status（open / closed）、`delete`；**最新一轮在最上面**，关闭的房间是历史 |
| Players | Player、Event、Hallu、Data、Sabotage、Ultimate、Total（按 Total 倒序，无分数行读作 0） |

几个刻意的取舍：

- **控制台内嵌一块「小电视」**。Console 右上角是一个 `16:9` 的 `/display/{code}` iframe，高约窗口 **36%**，宽度等比缩放；窄屏（< 64rem）时移到控制按钮下方、宽度撑满。**iframe 按大屏真实尺寸 `1920×1080` 布局，再用 `transform: scale(containerWidth / 1920)` 整体缩小**——所以看到的是「大屏画面被按比例压小」，而不是「网页被塞进小视口导致字特别大、溢出」。看不清字不重要，知道屏幕在显示什么才重要。用 iframe 而不是复用组件，是因为大屏有自己的 WS 连接和 zustand store，同页复用会两个屏幕抢同一份状态。iframe 设了 `pointer-events: none`——它是画面，不能抢控制台的点击。代价是多一个 WS 连接，以及**它比控制台快最多 2 秒**（大屏走 WS 即时，控制台 `GET /api/rooms` 2 秒轮询），极短节点上可能看到两处差一个节点。
- **跳节点按 section 分组，不显示 1–13 的序号**。每个盒子是「这轮游戏」，里面是它的内容：`Hallucination → opening / challenge / ranking`、`Data → opening / challenge / ranking`、`Sabotage → challenge / ranking`、`Ultimate → opening / quiz`，外加 `Welcome`、`Activity → ranking`（最终榜，垒箱子动效）、`Wrap up`。操作员想的是「跳到 Sabotage 的挑战」，不是「跳到第 8 个节点」。分组由快照里的 `flow` 现场推导（`section` 或 `score` + `type`），**前端不硬编码流程**，改 `flow.py` 后按钮自动跟着变。
- **没有房间选择器**。房间是一次性的，**当前轮 = 唯一一个未关闭的房间**，操作员要操作的就是它，无需选择；上一场结束了就 `finish`，再 `new room` 开下一场。历史房间只作记录，不参与控制。
- **只用表格**：房间和玩家都是整齐的数字列，卡片塞不下 Created / 参与人数 / 四个分项。
- **房间只读 + 删除**：房间没有可编辑字段（code + 创建时间就是全部身份），所以 Rooms 表只有 `delete`；玩家仍可就地新增/编辑/删除（内联表单，不用弹窗），删除前 `confirm` 一次。分数仍然只能通过接口改 —— 改分比改名字危险得多。
- **令牌失效会退回登录**：轮询拿到 401 时面板/大屏回到登录页，而不是静默停止更新。
- **玩家表按分区打开才轮询**：名单没人看的时候不该每秒查一次库。
- **「排哪个活动」是控制，不是过滤**。它写在服务端 `settings.active_event`，改它等于改大屏、等待室和公开榜的内容，所以它和演出控制放在一起（Console 的 **Leaderboard Event**）。**Event Report 只读**，它自己的活动切换只决定这一页看谁，不影响任何屏幕——报表在演出中可以被随手打开看上一场。
- **窄屏（< 52rem）时侧边栏变成一行 tab**，宽屏是 12rem 侧栏 + 内容区；表格在窄屏内横向滚动，不撑破页面。
- 分区状态**不持久化**，刷新回到 Console——操作员最常用的还是控制台。

### Event Report 的报表口径

`GET /api/events/{key}/summary` 同时给两个人数，**它们回答的是不同的问题**：

| 字段 | 含义 | 读法 |
|---|---|---|
| `players` | 该活动注册过的人 | 分母 |
| `played` | **进过房间并上报过分数的人**，`scores` 行存在即算，**0 分也算** | 「有多少人真的玩了」 |
| `scored` | `total_score > 0` 的人 | 「有多少人拿到分」 |

`played` 的判据刻意**不是** `total_score > 0`：答完整场却一题没对的人也是玩了的人。
两者的差（`played - scored`）说明这一轮**跑过但没给分**；`players - played` 才是「注册了没玩」。

每日桶 `registrations_per_day` 里同时带 `count` 和 `played`；报表上的柱状图**画的是 `played`**，
因为活动关心的是每天真实上场的人数，不是注册量。

报表里**没有任何玩家数据的游戏不显示**（行和列都不显示）：没玩的游戏是「模块没开」，
不是「0 分」，后者是更糟也更错误的说法。

## 令牌与角色

| scope | 主体 | 能读 | 能控制 |
|---|---|---|---|
| `full` | admin | ✅ | ✅ |
| `display` | admin | ✅ | ❌ |
| —（player）| `player:<id>` | 房间状态 | ❌ |

`POST /api/auth/admin` 接受 `scope: "display"`，签出只读令牌。这样**大屏可以整天保持登录**，而路过大屏的人无法暂停节目。

`require_admin` 要求 `scope === "full"`；`require_display` 接受两种 scope。

## Display View 的内容

- **waiting = 等待室**：大字 **Waiting Room**、`N players joined`、以及**已加入玩家的名字墙**（2 秒轮询名单）。
  设备是我们发的 iPad，所以**没有二维码、没有网址、没有房间号**——大厅屏幕上不该出现需要人操作的东西。
  名字用胶囊格排布（`auto-fill` 自动列数），人多时区域内滚动，短名单自动垂直居中。
- **playing / paused**：节点序号与标题、**大倒计时**、以及该轮可以静态呈现的内容
- **finished**：结束语

**大屏始终显示倒计时**——大屏上只有一个时钟，不像玩家屏幕会和游戏自己的计时器打架。

> ✅ Hallucination 与 Quiz 的**题面**会显示在大屏上：引擎下发 `snapshot.question`（当前第几题），大屏据此取题。**选项仍只留在 iPad**，大屏不显示答案分布——那需要答案聚合，目前只有分数上报，没有逐题答案落库。

## 玩家身份：一台设备一个人

因为是人手一台 iPad，登录身份**不再靠昵称唯一**：

- 客户端登录时带上**自己已有的令牌**；令牌有效且活动一致 → 认回同一个玩家（重开页面/刷新不丢）。
- 没有令牌且昵称已被占用 → 自动变成 `Alex (2)`、`Alex (3)`，**不会把两个人合并成一条分数**。
- 唯一键仍是 `(event, nickname)`，因为它约束的是「同一个活动内显示名不重复」。

这条不是锦上添花：人手一台时重名几乎必然发生，而错误合并要等到分数落库（阶段 3）才会暴露。

## 已知限制

- **分数已自动落库**（`POST /api/me/score`，按题幂等）；但**逐题答案没有存**，所以做不了「全场选 A/B/C/D 分布」这类聚合。大屏显示题面靠引擎的题号，不靠答案。
- **Admin 面板不能改分数**：房间/玩家的增删改已进面板，分数仍走 `/api/docs`。
- **分数由客户端计算**：题目还在前端，后端无法独立核算。
