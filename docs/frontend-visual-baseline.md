# 前端视觉改造基线（仅美化，不改主逻辑与接口）

> 2026-11-25 盘点。只读分析，未改动任何源文件。目的：在「不改主逻辑和接口」的前提下，明确每个界面能改到什么程度、哪些东西是硬契约。

## 1. 界面清单

| # | 界面 | 入口 | 规模 | 现有视觉 |
|---|---|---|---|---|
| 1 | Dashboard Web 面板 | `dashboard/public/{index.html,style.css,app.js}` + `dashboard/server.js` | 338 / 467 / 671 | 深色 oklch，单页 5 个 `<section>`，SSE 实时 |
| 2 | Desktop 客户端 WebView | `desktop/web/{index.html,style.css,app.js}`（打包时由 `desktop/scripts/collect-resources.sh:41-49` 复制） | 425 / 805 / 1227 | 深色 hex，7 个 `<section>`，Tauri `invoke` IPC |
| 3 | Update Console | `update-server/public/{index.html,style.css,app.js}` | 151 / 293 / 374 | slate + indigo 深色，4 个 view |
| 4 | 订阅运营后台 + 用户侧 | `subscription-server/src/main/resources/{templates,static}` | 6 模板 / 3 CSS | 后台深绿 rail + 柠檬绿；用户侧极简单栏浅色 |
| 5 | LogLens 日志工作台 | `LogLens/loglens-ui/src/main/resources/ui/*` | 压缩单行 | Georgia + 黄铜色，三栏 + 虚拟滚动 |
| 6 | LogLens 离线报告 | `LogLens/loglens-viewer/.../HtmlExporter.java:32` 内联 `<style>` | 1 行 | 同 LogLens 暗色 |
| 7 | Android 原生端 | `android-app/app/src/main/res/**` + Kotlin | 611 行 layout | Material3，浅蓝 / 夜间暖橙 |

关键事实：**五套 Web CSS 之间没有共享令牌层**。类名词汇也分两派——dashboard 用 `.btn.btn-primary`，desktop 用 `.btn-sm` / `.btn-save`，update-server 又是 `.btn.btn-primary`。统一与否是第一个需要用户决定的问题。

## 2. 可以放心改（纯 CSS）

- 全部 CSS 变量：颜色、字号刻度、间距刻度、圆角、阴影、缓动与时长。
- 组件外观：卡片、按钮族、表单控件、开关、状态点、表格、toast、模态框、滚动条、焦点环、选中态。
- 布局微调：栅格 `minmax`、`max-width`、`min-height`、断点。
- 可新增选择器、伪元素、动画；可新增不参与逻辑的装饰类。

## 3. 需要动标记但不算改逻辑

- HTML 包裹层重排、加图标位、加分组标题。
- JS 模板字符串里的结构（`dashboard/public/app.js:429-460` server-card、`desktop/web/app.js:463-507` / `647-656` / `752` / `1086-1110`、`update-server/public/app.js:123-137` / `148-157`、`LogLens .../app.js:16`、`telemetry.js:86-93`）。
- 内联 SVG 图标替换（保留可被 CSS 接管的尺寸）。
- 条件：**保留全部 id、`data-*`、事件绑定、`th:*` 指令、`R.id`**。

## 4. 绝对不动

- `dashboard`：`api()` 的 `/api/*`、`EventSource('/api/events')`、`data-section` / `data-key` / `data-node-id`、`.field-group[hidden]` 显隐、`#btnSave.disabled`、`#changeIndicator.hidden`。
- `desktop`：全部 `invoke()` 命令与参数、49 个 id、`.section.active` 显隐、`#subscriptionPanel`（**刻意不带 `.section` 类**，补上会被 `switchSection` 永久隐藏）、`fill.style.width` 进度几何、canvas 二维码颜色（在 `app.js:967-974`）。
- `update-server`：`/admin/*` 端点、`section id="view-*"`、`nav[data-view]`、`#nav.hidden`、静态路径 `/admin/static/*`（`update-server/tests/static.test.mjs:88-119` 有断言）。
- `subscription-server`：全部 `th:*`、`th:classappend` 产生的裸类 `active` / `up`、`logs.js` 的 `.log-workspace` 根与 `root.dataset.*`、`[data-dataset]` / `[data-view]` / `aria-pressed`、`.log-trend` 柱状图内联高度。
- `LogLens`：30 个 id、筛选 `name`、`body.classList.toggle('light')`、`.severity` + 大写等级、`#spacer.style.height` / `#rows.style.transform`、`48px` 行高三处常量；`LocalWebServer.java:68` 的白名单与 CSP 意味着**新增 CSS/字体/图片文件属接口级改动**。
- `Android`：全部 `R.id`、Kotlin 引用的 `R.color` / `R.drawable` 名称、视图 `tag`、`menu` 项与 `showPage` 映射。

## 5. 最容易踩的坑

1. **className 整串重写**：`dashboard/public/app.js:181,184`、`desktop/web/app.js:196,199` 每轮轮询都会重写 `status-indicator` / `service-card` 的整个 className。任何「追加自定义类」实现的状态视觉会被抹掉——状态只能由 CSS 消费既有状态类。
2. **固定高度魔法数**：`.log-terminal` 分别是 `calc(100vh - 220px)`（dashboard）与 `calc(100vh - 180px)`（desktop），与 topbar 高度、content padding、section-header 隐式绑定。改任一处就溢出或留白。
3. **开关几何四值耦合**：track / thumb / 内缩 / `translateX` 必须同比改动（dashboard 36×20/16/16，desktop 40×22/18/18）。
4. **`html{font-size:15px}`**（dashboard）驱动全部 rem 令牌，改它等于整体缩放。
5. **侧栏宽度联动**：`--sidebar-w` 与 `.main margin-left` 必须同步。
6. **desktop 侧栏 `-webkit-app-region: drag`** 覆盖整块，新增可点击元素必须显式 `no-drag`，否则点击被窗口拖拽吞掉。
7. **`subscription-server/static/admin/console.css` 第 3-9 行是超长压缩单行**，承载全部后台样式；`logs.css` 依赖它的变量名，改名会静默丢色。
8. **LogLens `style.css` 带 UTF-8 BOM**，且有三套并行色板（暗色 / `body.light` / HtmlExporter 离线报告），改一套不会同步其余两套。
9. **Android 大量样式绕过 XML**：`NodeRow.kt:81-156`、`MainActivity.kt:756-796`、`TrafficMetricView.kt:36-160`、`SubscriptionActivity.kt:19-27` 直接 `setTextColor` / `new MaterialCardView`；只改 `themes.xml` 对它们无效。
10. **update-server `.nav` 同时有 `display:flex` 与 `hidden`**，CSS 没有 `#nav[hidden]` 覆盖——既有隐患，美化时不要用 display 替换 hidden，需实测登录前是否隐藏。

## 6. 现存的视觉不一致（可顺手收口）

- 两套按钮词汇（`.btn.btn-primary` vs `.btn-sm` / `.btn-save`）。
- desktop 设置页 `style.css:117-148` 是独立设计的「已进化」版本（28px 标题、14px 圆角面板、44px 输入），与其余页面两套尺度。
- desktop 版本号写死 `v0.3.2`（`index.html:68,377`），而 `tauri.conf.json` 是 `0.3.5`。
- desktop 设置页有 `settings-eyebrow`（`SIMPLEPLANE / PREFERENCES`）这类标题上方的眉标。
- 状态普遍只靠颜色区分（`.status-indicator` 三态、`.toast` 四态仅左侧 3px 边框），对色觉障碍不友好，`toast` 无图标兜底。

## 7. 建议顺序

0. 先 settle 共享设计系统（颜色 / 字号层级 / 间距 / 圆角 / 阴影 / 缓动），并决定五个 Web 界面是统一还是各自保留身份。不做这步，后续必然漂移。
1. 可并行、风险最低：LogLens 两套变量层 + update-server 整份 CSS + Android `values`/`values-night` 成对色值与 `themes.xml`。
2. `dashboard`（先 CSS 层，再标记层）。
3. `desktop/web`（依赖第 0 步；先 CSS 层，并统一设置页与其它页）。
4. `subscription-server`（先用户侧三页，再格式化后台压缩 CSS，最后单独回归 logs 页）。
5. `Android`（先资源值层，再 layout / 程序化层）。

每个界面收尾都要跑对应回归：dashboard 五个 section 切换 + SSE 徽标 + server 卡片增删；desktop 3s 轮询重写 + form-row 隐藏 + 订阅面板 + 更新进度；update-server 登录后 nav + 表格行内类 + 上传进度；subscription `th:classappend` 的 `active`/`up` + logs 图表；Android 两个仪器测试 + `fontScale>=1.5` + 夜间模式。
