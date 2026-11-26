# Desktop 端纯样式改造可行性核实

> 2026-11-26。全部结论来自实测：在 1100×720 / 900×600 / 1440×900 三档视口下向真实页面注入候选 CSS，读取 `getComputedStyle` 计算值确认生效。
> 未修改 `desktop/web/` 下任何文件。

## 判定口径

- **纯 CSS**：只改 `desktop/web/style.css`（或其在 `<link>` 之后追加的样式块），`index.html` 与 `app.js` 一字不动。
- **需改标记**：必须动 `index.html` 或 `app.js` 里的模板字符串，但不改事件绑定 / `invoke()` / `id` / `data-*`。
- **改不了**：数据或状态机层面，改 CSS 无效。

---

## 一、纯 CSS 可以做（逐项实测通过）

### 1. 对比度 —— 全部达标，无需改结构

改 `:root` 两个变量即可。实测值：

| 组合 | 现状 | 改为 | 结果 |
|---|---|---|---|
| `--text-muted`（卡片 `#21242f`） | **3.20:1** | `#8b93a1` | **5.00:1** ✓ |
| `--text-muted`（输入 `#161922`） | 3.63:1 | `#8b93a1` | 5.67:1 ✓ |
| `--text-muted`（侧栏 `#1a1d27`） | 3.48:1 | `#8b93a1` | 5.43:1 ✓ |
| `--text-muted`（页面 `#0f1117`） | 3.90:1 | `#8b93a1` | 6.10:1 ✓ |
| `--text-muted` 备选 | — | `#9aa0a6` | 5.86 / 6.65 / 6.37 / 7.15 ✓ |
| `.nav-item.active` 白字 | **3.22:1** | 见下 | — |

`.nav-item.active` 要达标只有两条路，都纯 CSS：

- **改成实心深蓝**：`.nav-item.active{background:#2f5cbe;color:#fff}` → 实测 **6.19:1** ✓
- **改成浅底彩字**（dashboard 的做法）：`background` 用 accent 15% 叠在 `#1a1d27` 上得 `rgb(34,46,71)`，字用 `#4f8cff` → 实测 **4.21:1**，仍不达标；需再调字色到 `#6ba0ff` 左右才过 4.5。

**结论**：走深蓝实心最省事，一次改动同时修好 `.btn-save`（`style.css:603` 也是 `white on --accent`）。实测 `#2f5cbe` 对白字 6.19:1。

注意 `--accent` 本身还有 11 个消费点（描边、图标、focus 环、toggle 填充）。它作为**深底上的前景色**是 5.87:1，完全合格——**不要改 `--accent`，只改那些“accent 当背景”的地方**（`.nav-item.active:87`、`.btn-save:602`）。

### 2. 卡片与输入框同色 —— 一条声明解决

实测：`.server-card` 与 `.server-card .input` 计算值都是 `rgb(22, 25, 34)`（`--bg-input`）。加 `.server-card{background:var(--bg-card)}` 后实测变成 `rgb(33, 36, 47)`，**无需 `!important`**，层级天然够。节点卡内部六个字段立刻有了层次。

### 3. 节点卡 5+1 破格 —— 纯 CSS 可修，但要配断点

现状（1100 宽）：`.server-fields` 计算出 **5 列**，6 个字段排成 5+1，Key 独占第二行。

实测三档：

| 视口 | 卡片宽 | 当前列数 | 改 6 列后 | 字段宽 |
|---|---|---|---|---|
| 1440 | 1171 | 7（含 0px 空列） | 6 列 1 行 | 162px |
| 1100 | 831 | 5 | 6 列 1 行 | 127px |
| 900 | 631 | 4 | 6 列 → **94px**（`.select` 内最长选项 `chacha20-poly1305-v2` 实测需 142px，会截断） |

**结论**：`grid-template-columns: repeat(6, minmax(0,1fr))` 在 1440 与 1100 都是 1 行 6 格，很干净；**900 宽必须降到 3 列**（实测 94px → 195px，2 行 3+3）。一条 media query 即可，不碰任何标记。

### 4. 宽度失控 —— 纯 CSS 可修

实测：视口 1440 时 `.section` 与 `.server-card` 都是 **1171px**；加 `.content>.section{max-width:1040px}` 后实测降到 **1040px**，字段宽 184→162px，列数 7→6（空列消失）。

其余六页无任何 `max-width`（只有 `#sec-settings` 有 960px）。加一条规则统一收口即可。

### 5. 焦点可见 —— 纯 CSS 可修（且现在确实是零）

实测基线：`.btn-sm:focus` 的 `outline` 为 `none`；`.toggle-track` 的 `outline` 为 `none`。

- 给按钮/输入/下拉统一加 `:focus-visible{outline:2px solid var(--accent);outline-offset:2px}` —— 实测生效。
- `.toggle input:focus-visible + .toggle-track{outline:...}` —— 先用 class 替代 `:focus-visible` 验证**兄弟选择链本身可用**（实测 `outline-style: solid`），说明该规则写法正确，只是因为无 `tabindex` 之外的阻碍才没生效。
- 坐标确认：`.input` 的 `--accent` 焦点环在卡片上是 **4.81:1**，在侧栏上是 **5.23:1**，符合非文本对比度 3:1 要求。

> 注意：`.nav-item` 是 `<li>`、`tabIndex = -1`，**CSS 无法给它键盘焦点**（`tabindex` 是属性）。这一项不属于纯 CSS。

### 6. 错误态矛盾 —— 纯 CSS 可修（但需要 `card-error` 存在）

现状：`app.js:196` 把 `error` 映射成 `status-indicator stopped`（灰），而 `:199` 给卡片加 `card-error`（红框），标签文字是“异常”。

实测注入 `.card-error .status-indicator.stopped{background:var(--error);box-shadow:0 0 6px var(--error)}` + `.card-error .status-label{color:var(--error)}` 后：圆点 `rgb(248,113,113)`、标签同色。**灰点 + 红框 +“异常”的三方矛盾被 CSS 消解**——因为卡片上的 `card-error` 本来就是 JS 正确给出的信号。

代价：圆点的 className 仍是 `stopped`，语义上不准确，但视觉上一致了。这是纯 CSS 能拿到的最大收益。

### 7. 数字跳动 —— 一行

实测 `.info-value` 的 `fontVariantNumeric` 从 `normal` 变 `tabular-nums` 生效。覆盖 `.info-value / .service-uptime / .log-time / .route-count / .update-progress-text`。

### 8. 日志断行 —— 一行

`.log-text{word-break:break-word;overflow-wrap:anywhere}` 实测从 `break-all` 变 `break-word`。URL 与 base64 不再被从中间劈开。

### 9. 无样式类补齐 —— 纯 CSS 可修

实测确认以下 6 个类在 `style.css` 里**完全没有规则**：`log-level`、`preset-desc`、`btn-remove`、`btn-import-ca`、`btn-clear-ca`、`version`。

实测给 `.preset-desc{color:var(--text-muted);font-size:12px}` 后计算色从继承的 `rgb(232,234,237)` 变成 `rgb(107,114,128)`——预设名称/描述/元信息三层立刻分开。

> 陷阱：`.log-level` 是 `<span>` 默认 `display:inline`，`min-width` 无效（实测宽度仍 96px 由内容决定）。要分栏必须写 `display:inline-block`——这仍在纯 CSS 范围内，实测可行。

### 10. 空 uptime —— 用 `:empty` 伪类处理，不动 JS

实测 `#uptime-proxy-local` 恒为空（`app.js` 全文无 `uptime` 赋值）。纯 CSS 两条路都通：

- `.service-uptime:empty{display:none}` —— 实测 `display:none`，空位消失。
- `.service-uptime:empty::before{content:"未运行"}` —— 实测生成内容，但**伪元素不进无障碍树**（实测 `innerText` 仍是空）。只能当视觉兜底。

### 11. 状态不只靠颜色 —— 纯 CSS 可加图形

实测 `.status-indicator::after{content:"●"}` 与 `.toast.error::before{content:"!"}` 均生效；toast 的 `::before` 实测 16px 圆形徽标成立。

但同样是伪元素：**对读屏无效**。如果只是给色觉障碍用户一个形状差异，纯 CSS 够；如果要真正播报，得动标记。

### 12. 设置页统一 —— 纯 CSS 可修

实测 `#sec-settings h2` 是 `28px / letter-spacing:1px`，其余页是 `20px / normal`。注入 `#sec-settings h2{font-size:20px;letter-spacing:normal}` 后实测对齐为 `20px / normal`；`.settings-eyebrow{display:none}` 实测可隐藏眉标；`#sec-settings{max-width:none}` 可去掉独立宽度。

全站圆角可从现在的 4/6/7/8/10/11/12/14 收敛到 2–3 档，纯 CSS。

### 13. 动效降级 —— 纯 CSS 可加

全站无 `prefers-reduced-motion`。实测媒体查询 `(prefers-reduced-motion: reduce)` 在当前 WebKit 可用，加一段全局降级即可。

### 14. 日志页双滚动 —— 纯 CSS 可改成弹性填充

现状 `height: calc(100vh - 180px)`，900×600 实测得 **420px**（`.content` 高 600，不滚动）。

注入 `#sec-logs{display:flex;flex-direction:column;height:100%}` + `.log-terminal{flex:1;min-height:0;height:auto}` 后实测终端高 **480px**，多出 60px，且不再依赖魔法数。**纯 CSS 成立**，但要注意 `.section` 高度需 `height:100%` 链——实测该写法在本页有效。

---

## 二、需要改标记（技术上是“只改 class 与结构”，但不属于纯 CSS）

| 项 | 为什么 CSS 不行 | 改动量 |
|---|---|---|
| 假绿点 `Ready` | `.status-text` 文本由 DOM 决定；`app.js` 从未写它 | 属于逻辑问题（见三） |
| 空 uptime 的真实数值 | 无数据来源 | 属于逻辑问题 |
| `<h1>` 缺失 | 标签名是 DOM | 改 `index.html:77` 等 7 处 `h2` → `h1`，或补一条 |
| 导航键盘可达 | `tabindex` 是属性，CSS 无法添加 | 7 个 `<li>` 加 `role="link"` + `tabindex` + Enter/Space 处理（**涉及 JS**） |
| 模态无 `dialog` 语义 | 属性 | `index.html:273` + `app.js:1086` |
| 全站无 Escape | 键盘事件 | 属 JS |
| toast 无 `aria-live` | 属性 | `index.html:421` 一行 |
| 图标（月亮/盾牌/纸飞机/重复齿轮） | 内联 `<svg><path d>` | 替换 `index.html` 里 4 段 path + 1 段重复齿轮 |
| 版本号 `v0.3.2` → `0.3.5` | `textContent` | `index.html:68,377` |
| 重复齿轮图标 | 同一个 path 出现在 `:30` 与 `:62` | 换一处 path |

---

## 三、改不了（是数据/状态机问题，不是视觉）

1. **`Ready` 死徽标**：`app.js` 1227 行里没有一处碰 `#statusBadge`。CSS 只能让它不显眼，不能让它说真话。
2. **`stopping` / `error` 降级**：`app.js:196` 的三元表达式只有三路。CSS 能借 `card-error` 修好 `error` 的颜色，但 **`stopping` 完全没有任何卡片级类**，无从下手。
3. **双服务模型与单次 connect 不符**：停 TUN 顺带停代理，是后端行为。
4. **版本号三处不一致**：侧栏 `v0.3.2`、日志首行 `v0.3.2`、`tauri.conf.json` `0.3.5`。改文本是一行，但真正的“不漂移”需要构建期注入。
5. **删节点无确认、导入 YML 覆盖**：交互逻辑。
6. **`.server-card` 中 `Key` 字段被渲染为 `type="password"`**（`app.js:490`）而另外五个是明文——这是产品决策，不是样式。

---

## 四、两条硬约束（改 CSS 时必须守住）

**1. 不能靠“加自定义类”做状态视觉。**
`app.js:199` 每 3 秒执行 `card.className = 'service-card ' + state`，实测：给卡片加 `probe-extra` 后，模拟一次轮询，该类**被完全抹掉**。同理 `app.js:196` 重写状态圆点的全部 className。
→ 所有状态视觉只能写成 `.card-running {}` / `.status-indicator.running {}` 这类**消费既有状态类**的形式，或以 `card-X` 为祖先的派生选择器（实测 `.card-running .service-card-body{...}` 可用）。

**2. 可以给卡片加新类吗？**
只有 `app.js` 也会写的地方才行。`renderServerCards` 用 `card.className = 'server-card'`（`app.js:459`）——同样会被抹。所以**新类只能加在 JS 从不整体覆盖的元素上**，例如 `.section`、`.form-row`、`.settings-*`（由 `classList.toggle` 操作，是增量的）。

---

## 五、建议的纯 CSS 批次（零风险起步）

1. `:root` 三个变量：`--text-muted: #8b93a1`，并把 `.nav-item.active` / `.btn-save` 改为 `#2f5cbe`；`--border` 略提亮以支撑新密度。
2. `.server-card{background:var(--bg-card)}`。
3. `.server-fields` 断点：`≥1200px` 6 列、`≥820px` 4 列、其余 3 列。
4. `.content>.section{max-width:1040px}`（日志页 `.section-full` 除外）。
5. 全局 `:focus-visible` 环（按钮/输入/下拉/开关 track）。
6. `.card-error` 三处一致性（圆点、标签、边框）。
7. `tabular-nums` + `word-break: break-word`。
8. 六个无样式类补规则（`log-level` 记得 `inline-block`）。
9. `:empty` 处理空 uptime。
10. 设置页对齐全站 + 圆角收敛 + `prefers-reduced-motion`。
11. 日志页改 flex 填充（替代 `calc(100vh - 180px)`）。

这 11 条全部实测可在 CSS 内完成，且不触碰任何 JS 契约。
