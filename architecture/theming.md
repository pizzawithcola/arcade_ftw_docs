# 主题：深色（默认）与浅色，后端统一开关

> 状态：**计划已定，尚未动工**（2026-11-16）。修改任何东西之前先读「铁律」一节。
>
> 起因：stakeholder 要求在保留深色模式的前提下支持「白底 + 红白」的浅色模式，并保留科技质感。

---

## 0. 铁律（不可违反）

1. **默认必须是深色。** 深色是现场在用的东西，浅色是新增选项。任何改动如果让默认观感变了，就是回归。
2. **深色的样式表不重写。** 浅色通过**追加覆盖层**实现（`[data-theme='light']` 作用域），而不是把现有规则改成「两边都要」。
3. **e2e 断言颜色的用例必须继续通过。** 仓库里有大量形如 `rgb(255, 82, 87)` 的断言，它们断言的是深色。所以测试默认跑在深色下。
4. **一次只改一个页面。** 每页改完截图对比深色前后，深色不一致就回退。
5. **主题是全场统一的**，由后端存储，不由设备决定。大屏、iPad、控制台必须同色。

---

## 1. 已确定的产品决策

| 议题 | 决定 |
|---|---|
| 开关位置 | **后端** `app_settings`，全场统一 |
| Data Shadow 的假帖子 | **跟随变白**（白卡片 + 墨色文字） |
| AI Cyber Battle 紫/青队色 | 白底下加深（`#8e0f8e` / `#07707f`）；本次活动大概率不用 Battle，最后一页再做 |
| 页面 40px 网格 | 浅色下**关掉**，只保留 HUD 内部的点阵（`.hud__matrix`） |
| 图标资源（带烘焙红光的 PNG） | 先按现状看效果，不重做 |
| 主题粒度 | 全场统一 |
| 顺序 | 后定；原则是**逐页面推进** |

---

## 2. 技术骨架

### 2.1 后端

- `app_settings` 新增键 `theme`，值 `dark` | `light`，缺省 = `dark`（无行也视为 dark）。
- `GET /api/settings` 增加 `theme` 字段。
- `PUT /api/settings` 接受 `theme`（沿用现有的部分更新语义，只改传了的字段）。
- 校验：只接受 `dark` / `light`，其它 422；未知值降级为 `dark`。
- `app/manager.py` 增加 `broadcast_all(message)`（现有只有按房间广播）：主题一变，所有在线屏幕立刻切换，不用等刷新。

```jsonc
// WS 广播载荷（新增消息类型）
{ "type": "theme", "payload": { "theme": "light" } }
```

### 2.2 前端

- `<html data-theme="dark|light">`，唯一开关点。`index.css` 与覆盖层都按这个属性选择。
- **首屏防闪**：`index.html` 加一段内联脚本，在 hydration 之前从 `localStorage` 读出缓存值写到 `<html>`；随后 app 启动再向 `GET /api/settings` 覆写（服务端才是真相）。
- 切主题：admin 写 `PUT /api/settings { theme }` → 后端广播 → 每台设备收到后改 `data-theme` 并写 localStorage。
- `color-scheme` 跟随：`index.css`、`views/auth.css`、`pages/StatusPage.css` 三处的 `color-scheme: dark` 改为按主题取值（影响原生下拉、滚动条、表单控件）。
- 缓存键：`localStorage['arcade_ftw_theme']`，只存最后一次已知值，服务端仍然是真相。
- **不做** `?theme=` 临时参数：Design 页的预览直接在自己容器上写 `data-theme="light"`，用的是同一套规则，不需要第二套机制。

### 2.3 覆盖层组织（关键）

浅色的所有规则集中在 `src/design/light.css`，一个文件、一个作用域：

```css
[data-theme='light'] { /* token 重映射 */ }
[data-theme='light'] .hud__frame { /* 光晕改阴影 */ }
```

深色文件**一行不动**。这样任何浅色问题都不会反向影响深色，回滚 = 删掉这个文件 + 去掉开关。

---

## 3. 动工前的准备：把颜色收敛到 token

现状：`var(--c-*)` 引用 **951 处**，硬编码颜色约 **40 处**。这 40 处不收敛，浅色就一定会漏。

### 3.1 新增 token（P0 已完成，实际落了 23 个）

每个 token 都是**一个颜色**，不是「带透明的颜色」：规则自己写 `color-mix(in srgb, var(--c-glow) 70%, transparent)` 保留各自的浓度。这样浅色只要把 token 本身设成 `transparent`，七十处光晕就一起消失，不用逐条改。

| token | 用途 | 深色值 | 浅色值 |
|---|---|---|---|
| `--c-on-brand` | 压在品牌红上的文字 | `#f4f4f6` | `#ffffff` |
| `--c-shadow` | 所有投影、内阴影、扫描线 | `#000000` | `#1a1614` |
| `--c-scrim` | 遮罩/压暗层 | `#000000` | `#ffffff` |
| `--c-well` | 凹陷：输入框、滑杆槽、徽章、预览屏 | `#000000` | `#eae6e1` |
| `--c-glow` | 红光的本色 | `#db0011` | `transparent` |
| `--c-glow-deep` | 标题背后的深红洗色 | `#af000e` | `transparent` |
| `--c-halo` | 白色光晕（小标记的可读性） | `#f4f4f6` | `transparent` |
| `--c-spark` | 亮线的白热核心 | `#ffffff` | `#ffffff` |
| `--c-doubt` / `-pale` | 滑杆「怀疑」侧 | `#f2a900` / `#ffe27a` | `#8a5a00` / `#f6e2b0` |
| `--c-trust` / `-pale` | 滑杆「信任」侧 | `#12b35c` / `#a8f5c8` | `#0f7a43` / `#b7e8cd` |
| `--c-cat-*` (6 组) + `-ink` | 题库分类徽章 | 现有 6 色 | 同色相加深 |

`color-scheme` 也跟着走：`:root { color-scheme: dark }` 放在 `tokens.css`，`[data-theme='light']` 里写 `light`，同元素、后者胜。**页内样式表不许再写 `:root { color-scheme }`**——`StatusPage.css` 曾经这么写，`inherit` 会把整站钉回 `normal`。

页面网格（`--c-grid-line`）在浅色下取 `transparent`：纸上是方格纸的感觉，不是这台机器的质感。

### 3.2 硬编码清单（逐个替换）

| 文件 | 处 | 内容 | 处理 |
|---|---|---|---|
| `views/hud.css` | 17 | 多为 `mask-image` 的 `#000`（**与主题无关，不改**）；3 处 `rgba(244,244,246,…)` 白色发光 | 改 `--c-glow` / `--c-on-brand` |
| `views/admin/AdminPage.css` | 7 | 6 个题型分类色 + 1 处 | 改 `--c-cat-*` |
| `games/hallucination/HallucinationGame.css` | 7 | 槽 `#070707`、判定红绿 | 改 `--c-well` / `--c-win` / `--c-lose` |
| `views/display/DisplayPage.css` | 6 | LinkedIn 蓝 `#0a66c2`（**保留**，它是「一个真帖子」）、`#000 22%` | 帖子蓝保留；压暗层改 `--c-scrim` |
| `games/dataShadow/DataShadowGame.css` | 5 | 帖子相关 | 走 `--c-paper*` 已有 token |
| `games/Welcome.css` / `TitleCard.css` / `QuizGame.css` | 12 | 光晕 rgba | 改 `--c-glow` |
| `views/user/PlayPage.css` | 1 | `drop-shadow(rgba(219,0,17,.85))` | 改 `--c-glow` |
| `pages/StatusPage.css` | 1 | `color-scheme: dark` | 跟随主题 |

### 3.3 三类系统性问题（跨页面，先解决）

1. **光晕 → 阴影**：约 70 处 `box-shadow` / `drop-shadow`、33 处 `text-shadow`。白底上红色外发光会糊成粉雾。做法：把「发光」统一收进 `--c-glow`，浅色下置空；投影改 `--c-shadow`。
2. **红底上的字**：出厂假设「红底 + `--c-text`（白）」。浅色下 `--c-text` 是墨黑 → 红底上只有 2.6:1。所有「红底承载文字」的规则固定用 `--c-on-brand`。已知：`.hud__tab`、`.ldb__row.is-rank-1 .ldb__rank`、`.quiz__timer.is-urgent`、登录主按钮、`admin__new-round`。
3. **`--c-black` 被当「更暗」用**：6 处 `background: var(--c-black)`、6 处 `color-mix(... var(--c-black) …)` 做投影。浅色下它变纸白 → 投影变白色光斑。全部改 `--c-well` / `--c-shadow`。

---

## 4. 页面清单（施工与验收单位）

| # | 页面 | 涉及文件 | 浅色下必须处理 | 验收方式 |
|---|---|---|---|---|
| 1 | 登录 `/` | `views/auth.css`、`LoginPage` | 背景光晕、卡片描边、输入框 `color-scheme`、三个门、红按钮白字 | 截图对比 + 三活动下拉可用 |
| 2 | 玩家 `/play` `/r/:code` | `PlayPage.css`、`hud.css` | HUD 缺口/括号/tab/chevrons、进度菱形、Logout、等待室名单 | iPad 视口截图 |
| 3 | 等待室榜 / 公开榜 | `Leaderboard.css`、`LeaderboardPage.css`、`StandbyBoard` | 电平条、rail、第一名、图例 | 大屏 1920 截图 |
| 4 | 大屏 `/display` | `DisplayPage.css`(1640) | HUD、时钟、Data Shadow 帖子（变白）、四游戏分屏、SABOTAGE 战场 | 逐节点截图（13 节点） |
| 5 | Hallucination | `HallucinationGame.css` | 滑杆槽、判定红绿、glow | 滑杆两端截图 |
| 6 | Data Shadow | `DataShadowGame.css` | 白帖子、荧光笔、报告面板 | 选中/提交/揭晓三态 |
| 7 | AI Cyber Battle（最后） | `SabotageGame.css`(1122) | 62 处队色、6 色卡牌、三域图标 | **整页截图评审，可能不做** |
| 8 | Ultimate Quiz | `QuizGame.css` | 选项对/错态、奖励卡灯泡、tab | 答题 + 奖励卡 |
| 9 | Welcome / Opening / WrapUp | `Welcome.css`、`Opening.css`、`WrapUp.css` | 光晕、卡片 | 已用 Design 页预览验证方向 |
| 10 | Admin 控制台 | `AdminPage.css`(1159)、`EventReport.css`、`DesignPage.*` | 面板/表格/勾选、分类色、`--c-black` 3 处、**预览窗会自动跟随主题**（里面是真大屏） | 每个页签截图 |
| 11 | 状态页 `/status` | `StatusPage.css` | 独立一套 + `color-scheme` | 截图 |
| 12 | pelmet | `public/pelmet.html` | 独立交付物，内嵌字体 + 硬编码黑 | **本期排除**，必要时另出一版 |

---

## 5. 阶段划分

| 阶段 | 内容 | 产出 | 完成标志 |
|---|---|---|---|
| **P0 地基** ✅ | 23 个 token；40 处硬编码清零；39 处红光/10 处红底文字/20 处阴影改走 token；后端 `theme` 键 + `broadcast_all` + WS `theme`；`data-theme` + 防闪 + `color-scheme` + 控制台顶栏开关 | 开关可用、深色零变化 | 深色 15 屏截图比对；pytest 206、e2e 全绿 |
| **P1 玩家侧** | 页面 1、2、9 | iPad 全流程 | 逐页截图评审 |
| **P2 大屏侧** | 页面 3、4 | 大屏全节点 | 13 节点截图 |
| **P3 游戏内页** | 页面 5、6、8 | 三个游戏 | 关键态截图 |
| **P4 控制台 + 收尾** | 页面 10、11；Battle（可选，页面 7） | 控制台全页签 | 截图 + e2e |

**P0 是唯一「不先做就会返工」的阶段**，其余每阶段结束都可以停。

---

## 6. 风险与回滚

| 风险 | 说明 | 对策 |
|---|---|---|
| `color-mix` 220 处 | 「品牌红 × 透明」在白底会变粉雾，无法静态预判 | 逐页截图；必要时把某处 `color-mix` 换成显式 token |
| 深色被误伤 | 覆盖层写错选择器会反噬深色 | 覆盖层 100% 限定在 `[data-theme='light']` 下；每页改完前后截图比对 |
| e2e 颜色断言 | 18 条 UI 用例含硬编码色值 | 默认主题保持 dark，测试不变 |
| 首屏闪烁 | 服务端值到达前会闪深色 | `index.html` 内联脚本读缓存 |
| Battle 队色不可读 | 紫/青在白底接近失效 | 加深 + 整页评审；本期大概率不做 |
| 预览窗 | 控制台里嵌的是真大屏 | 会自动跟随主题，需在 P4 专门确认 |

**回滚**：删除 `theme-light.css` 的引用 + 移除 admin 开关 = 回到纯深色，后端 `theme` 键留着无害（缺省 dark）。

---

## 7. 与 Design 页预览的关系

`views/admin/LightPreview.tsx` 是**设计记录**，不是实现：它把真组件的 DOM 缩放进框里，用 `.theme-light` 覆盖 token 演示方向。正式实现后：

- 预览保留，继续作为「浅色应该长什么样」的参照；
- 但预览使用的 `.theme-light`（类作用域）会与正式实现的 `[data-theme='light']`（根作用域）并存 —— 实施时统一成一件事，避免两套；
- Design 页的说明文字随实施更新。

---

## 8. 参考

- Token 定义：`arcade_ftw_fe/src/design/tokens.css`
- 预览实现：`arcade_ftw_fe/src/views/admin/LightPreview.{tsx,css}`
- 设置存储：`arcade_ftw_be/app/settings_store.py`
- 视图形状：`architecture/views.md`

---

## 9. P0 实施记录（2026-11-17）

**改动落点**

| 层 | 文件 | 内容 |
|---|---|---|
| token | `src/design/tokens.css` | 23 个新 token + `:root { color-scheme: dark }` |
| 覆盖层 | `src/design/light.css` | 作用域从 `.theme-light` 改成 `[data-theme='light']`，补 roles / leans / 分类色，网格关闭 |
| 应用 | `src/theme.ts` | `applyTheme` / `readStoredTheme`，键 `arcade_ftw_theme` |
| 首屏 | `index.html` | 内联脚本按 localStorage 提前写 `data-theme` |
| 分发 | `src/store/gameStore.ts` | `applyMessage` 收 `{type:'theme'}`，大屏、iPad、状态页同一个入口 |
| 控制台 | `src/views/admin/AdminPage.tsx` + `.css` | 顶栏 `ThemeSwitch`（dark / light 两态），写入即广播 |
| 预览 | `src/views/admin/LightPreview.tsx` | 容器改用 `data-theme="light"`，与真实实现同一套规则 |
| 后端 | `app/settings_store.py` | `theme` 键、`read_theme`、422 校验、保存后 `broadcast_all` |
| 后端 | `app/manager.py` / `app/ws.py` | `broadcast_all`；连接建立时先发 `theme` 再发 `state` |

**深色零变化的证据**：15 屏（大屏 9 节点 + iPad 3 态 + 登录 + 公开榜 + 控制台）改动前后截图比对，逐像素差异最大 `2/255`、涉及 2.5% 的像素（`color-mix` 的取整），另加滑杆槽 `#070707 → #000000` 一处刻意的合并。比对脚本会遮掉三处天然不稳定的区域：大屏时钟、iPad 计时器、控制台里那块实时预览 iframe。

**踩到的坑**：`StatusPage.css` 顶上有一条 `:root { color-scheme: dark }`，改成 `inherit` 后会把整站钉在 `normal`（`inherit` 在根元素等于初始值），所以 `color-scheme` 现在只由 token 层拥有。

**P0 结束时浅色的样子**：结构对了、颜色对了，但页面级排版还没调——例如大屏的扫描线纹理在纸上过密、部分深红色块偏重。这些是 P1–P4 的内容，不是 P0 的缺陷。
