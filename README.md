# NezhaLatency v2 —— 延迟 / 丢包面板（Komari 参考站风格，适配 Nezha v2）

把 [komari.alicloud.ru](https://komari.alicloud.ru/)（Komari Monitor + Nezha 主题）
的延迟 / 丢包展示方式，重做到 **Nezha v2** 上。本项目是单文件自定义代码
（`NezhaLatency.html`），直接粘贴进 Nezha v2 后台的自定义代码即可用。

> **适配目标：nezhahq/nezha v2（dashboard main，2026-11-20  commit `ff43a43`；
> 官方用户前端 hamster1963/nezha-dash-v2 v2.7.0，管理前端 v2.6.1）。**
> 注意：仪表盘本身的版本号在构建时注入，源码里默认是 `"debug"`，所以
> “v2.7.0”在这里指的是**用户前端**的版本（前端清单
> `service/singleton/frontend-templates.yaml` 锁定），不是仪表盘 release 号。
> 本项目**不适配 Nezha v1**（v1 的美化项目见 `~/workspace/nezha-beautify-v1/`，
> 两者接口与注入方式有差异，见“已知限制”）。

## 参考站（Komari）是怎么展示延迟 / 丢包的

参考站由任务方于 2026-11-20 21:55–21:57 实地查看（只读，未登录），摘要如下。
KomariBeautify 本身**没有**延迟功能（它的 5 个功能是欢迎气泡、资产统计、
3D 地球、页面保护、自定义弹窗），以下 UI 来自 Komari 核心 / 主题，
Komari 侧没有公开的延迟 API 文档可供对照（见“待验证”）：

| 位置 | 内容 |
|---|---|
| 首页总览 | 4 张卡片：服务器总数 25 / 在线 23 / 离线 2、总上下行流量、实时上下行速率；地区筛选（全部/美洲/亚太/欧洲）；世界地图；服务器卡片网格 |
| 服务器卡片底部 | 当前延迟 + 丢包率，以及**两条分段历史条**（绿/黄/红；左延迟、右丢包）。例：RakSmart 硅谷 182ms / 0.6%，RackNerd 洛杉矶 219ms / 46.5%，火山引擎北京 37ms / 0.0% |
| 服务器详情 `/server/{数字ID}` | 标签页：详情 / Network。Network 是延迟核心：固定 3 个探针服务 SCCM-v4、SCCU-v4、SCCT-v4（四川移动/联通/电信 IPv4）；时间范围 1h/6h/12h/24h/3d/7d/30d；Peak cut 开关；每服务显示平均延迟 / 平均丢包；**双轴折线图（左延迟 ms / 右丢包 %）**，图例可点击筛选。例：RakSmart 1h：SCCM-v4 179.8ms/1.0%，SCCU-v4 175.2ms/0.0%，SCCT-v4 177.0ms/0.0% |
| 离线处理 | 延迟/丢包显示 `-- / --`；探针不可达但 agent 仍上报 CPU 时丢包 100.0%（例：Akile 台北、阿里首尔 I 延迟 `--` / 丢包 100.0%） |
| 数据来源 | agent 约 1s 上报系统数据；延迟/丢包由外部探针 ping 服务器得到，历史最长 30 天。渲染出的 DOM 里看不到公开 API / WS / JSON，路由只有 `/`、`/server/{id}`、`/login` |

## 功能（本项目）

| # | 功能 | 说明 |
|---|---|---|
| 1 | 服务器列表：当前延迟 + 丢包 | 面板内表格：服务器名、国家码、在线状态、当前延迟、24h 平均延迟、24h 丢包。当前延迟取 24h 历史最新桶；丢包 = 100 − 正常率 |
| 2 | 首页卡片延迟条（Komari 截图形式） | 每张服务器卡片底部：一行“延迟 xxx ms　丢包率 x.x%”，下面左右两条分段条并排（左延迟、右丢包），每条 10 段，对应最近 24 小时分 10 桶，绿/黄/红。延迟绿线按 Komari 截图校正：183ms 为绿色，所以绿 <200ms · 黄 <300ms · 红 ≥300ms；丢包绿 <1% · 黄 <5%。只让条本身可点（`cursor: pointer` + “点击查看延迟详情”提示），点某张卡的条打开该服务器的详情面板；条的点击会阻止冒泡，不劫持整张卡片的点击（卡片本身点开仍是官方服务器详情页）。面板内的服务总览卡片也保留同款条（最近 10 天） |
| 3 | 详情 / Network 视图 + 1小时实时 | 点首页卡片条或面板内服务器行展开：每探针一个按钮，显示最近延迟、最低/最高延迟、平均延迟、平均丢包；双轴图（左延迟 ms / 右丢包 %，ECharts 5），时间范围 **1h（实时，默认）** / 24h / 7d / 30d，Peak cut 开关（按 P95 截断尖峰），图例点击开关延迟曲线，点探针只看它的丢包面积。1h 档是虚拟档：接口仍请求 period=1d，再在前端截取最近 60 分钟（TSDB 模式约 30s/点，约 120 个点），每 30 秒自动刷新；游客也能看 1h |
| 4 | 离线处理 | 服务器离线显示 `--`；探针不可达但服务器在线时丢包 100.0%；无数据时显示 `--`，不编造数字 |
| 5 | 自动刷新 | 默认每 30 秒刷新总览与历史（页面隐藏时暂停），右上角也有手动刷新按钮；WebSocket 断线 5 秒后重连 |

**不包含**（有意排除）：欢迎气泡、资产统计、3D 地球、页面保护。
保护组件是前端威慑、不是安全边界，v1 项目的 README 也这么写；延迟项目不需要它。

## 安装（Nezha v2）

### 方式一：script 引用（推荐，一行搞定）

在 Nezha v2 后台（`/dashboard/`）→ **设置** → **自定义代码**
（`custom_code`）里只粘贴这一行，保存即可：

```html
<script src="https://cdn.jsdelivr.net/gh/gugetang/NezhaLatency-v2@main/NezhaLatency.js"></script>
```

`NezhaLatency.js` 是由 `NezhaLatency.html` 用 `build-js.py` 生成的
同款单文件 JS：它自己注入样式和面板节点，再执行同样的逻辑，
和直接粘贴 HTML 效果一样。改配置时，在这行前面再加一行：

```html
<script>window.NezhaLatencyConfig = { refreshSec: 30 };</script>
<script src="https://cdn.jsdelivr.net/gh/gugetang/NezhaLatency-v2@main/NezhaLatency.js"></script>
```

注意：jsdelivr 对 `@main` 分支有缓存（最长约 12 小时），刚推上去的新版
可能不会立刻生效；等缓存过期，或改用固定提交号引用，例如
`https://cdn.jsdelivr.net/gh/gugetang/NezhaLatency-v2@2d817a2d/NezhaLatency.js`。
也可以用 jsdelivr 的 purge 链接清缓存：
`https://purge.jsdelivr.net/gh/gugetang/NezhaLatency-v2@main/NezhaLatency.js`。

### 方式二：直接粘贴 HTML

1. 登录 Nezha v2 后台（`/dashboard/`）→ **设置** → 找到**自定义代码**
   （`custom_code`，用户前台用）。
2. 把 `NezhaLatency.html` 的**全部内容**复制粘贴进去，保存。
3. 刷新用户前台首页，每张服务器卡片底部会出现延迟条（当前延迟 +
   丢包率 + 两条分段历史柱），点卡片条即打开该服务器的延迟详情面板。
   面板内点服务卡片可按服务筛选详情；点服务器行打开该服务器的 Network 详情。

> 注入机制（已验证）：`custom_code` / `custom_code_dashboard` 字段仍在 v2
> 里（`model/config.go:43-44`，属于 `ConfigForGuests`，游客也能通过
> `GET /api/v1/setting` 读到）。服务端**不改** `index.html`；用户前端
> （nezha-dash-v2 v2.7.0，`src/App.tsx:133-145,193`）先拉取设置，再用
> `InjectContext(customCode)` 按文档顺序注入：样式进 `head`，脚本按序执行，
> 其它节点进 `body`，节点带 `data-injected="true"`，变更时可卸载。
> 所以本文件必须是“样式 + 节点 + 脚本”一体的单文件。
> 另：v2 **没有**独立的服务监控页（官方前端只有首页可折叠的服务面板和
> 服务器详情页的 Network 图）；本项目是在此之外的 Komari 风格延迟面板。

## 配置（只改文件顶部 `CONFIG`）

```js
var NezhaLatencyConfig = {
  apiBase: '',                 // '' = 与当前页面同源（注入时请保持 ''）；独立测试可填 https://nezha.example.com（需服务端允许 CORS）
  wsServerUrl: '/api/v1/ws/server',
  serviceApi: '/api/v1/service',
  profileApi: '/api/v1/profile',
  refreshSec: 30,              // 自动刷新间隔（秒），0 = 不自动刷新
  ranges: [                    // v2 API 只支持 1d/7d/30d；1h 是虚拟档，接口请求 1d，前端截取最近 60 分钟
    { label: '1h',  value: '1h' },
    { label: '24h', value: '1d' },
    { label: '7d',  value: '7d' },
    { label: '30d', value: '30d' }
  ],
  defaultRange: '1h',
  thresholds: { latencyGreen: 200, latencyAmber: 300, lossGreen: 1, lossAmber: 5 }, // 延迟绿线按 Komari 截图校正：183ms 为绿色，取 200
  peakCut: { enabled: true, percentile: 95 },
  attachCards: true,           // 尽力而为：给官方前端的服务器卡片底部加延迟条（点条开详情；失败不影响面板）
  wsStaleSec: 90,              // last_active 超过这么多秒未更新视为离线
  maxDetailPoints: 600         // 图表单条线最多点数（超出降采样）
};
```

也可以在页面里先定义 `window.NezhaLatencyConfig` 再加载本文件，会与默认配置合并。
调试用：`window.NezhaLatency = { open, close, refresh, config, state }`。

## Nezha v2 API 实录（已从源码验证）

研究对象：`nezhahq/nezha` main，commit `ff43a43`（2026-11-20）。
**v2 没有 `/api/v2` 路由**，所有接口仍在 `api/v1` 下
（`cmd/dashboard/controller/controller.go:89`）。

| 接口 | 权限 | 用途 / 形状 |
|---|---|---|
| `GET /api/v1/ws/server` | 公开（`force_auth=false` 时游客可用；`force_auth=true` 时需 JWT/PAT） | WebSocket，每 2 秒推 `{now, online, servers: [StreamServer]}`；`StreamServer{id, name, public_note（仅首包）, display_index, host, state, country_code, last_active}`（`model/server_api.go:5-21`）。**这是游客拿服务器列表的唯一途径** |
| `GET /api/v1/service` | 公开，`nezha:service:read` | `ServiceResponse{services: map[uint64]ServiceResponseItem, cycle_transfer_stats}`；每项含 `service_name, current_up, current_down, total_up, total_down, delay[30], up[30], down[30]`（30 天日粒度）（`model/service_api.go:24-33`） |
| `GET /api/v1/service/:id/history` | 公开，`?period=1d\|7d\|30d`（默认 1d） | `ServiceHistoryResponse{service_id, service_name, servers: [{server_id, server_name, stats: {avg_delay, up_percent, total_up, total_down, data_points: [{ts, delay, status}]}}]}`（`model/service_history_api.go:12-37`）。**游客只能查 1d**，否则返回 `unauthorized: only 1d data available for guests`（`cmd/dashboard/controller/service.go:171-174`） |
| `GET /api/v1/server/:id/service` | 公开，`?period=1d\|7d\|30d` | 旧版接口：`[]ServiceInfos{monitor_id, server_id, monitor_name, server_name, display_index, created_at: [ms], avg_delay: []}`（`model/service_history_api.go:4-11`） |
| `GET /api/v1/server/:id/metrics` | 公开，`?metric=` 必填，`?period=1d\|7d\|30d` | 仅 TSDB 启用时有数据；本项目未使用 |
| `GET /api/v1/server` | **仅登录** | 游客不要用；服务器列表请走上面的 WebSocket |
| `GET /api/v1/service/list` | **仅登录** | 完整服务配置行；游客不要用 |
| `GET /api/v1/profile` | **仅登录（JWT，PAT 被禁）** | `200 + success:true` = 已登录；游客调用会 401，这是正常的。本文件只用它判断游客/会员 |
| `GET /api/v1/setting` | 游客可读（精简版） | 游客只能看到 `ConfigForGuests`（含 `custom_code`）；`version` / `frontend_templates` 不对游客开放 |

通用信封：`{success, data, error}`（`model/api.go:21-25`），HTTP 200 也可能
`success:false`，本文件按信封判断，不只看 HTTP 状态码。

### 服务探针模型（`model/service.go:246-272`）

`Service{name, type, target, duration, display_index, notify,
notification_group_id, cover, skip_servers, enable_trigger_task,
hide_for_guest, fail_trigger_tasks, recover_trigger_tasks, min_latency,
max_latency, latency_notify}`。其中 `type`：**1=HTTP GET，2=ICMP ping，
3=TCP ping**；`duration` 是探测间隔秒数（默认 30s）；`cover`：0=全部
服务器（除 `skip_servers`），1=仅列出的服务器；`min_latency` /
`max_latency` 只是**告警阈值**，不是历史数据。

### 延迟 / 丢包是怎么存的

- 主存 VictoriaMetrics TSDB：只写两个指标
  `nezha_service_delay{service_id, server_id}`（延迟毫秒）和
  `nezha_service_status{service_id, server_id}`（1=成功，0=失败），
  默认保留 30 天（`pkg/tsdb/`；`service/singleton/tsdb.go`）。
  **TSDB 只在配置了 `tsdb.data_path` 后才启用**，否则走 SQLite
  `ServiceHistory` 表（`model/service_history.go:9-19`：
  `service_id, server_id, avg_delay, up, down, data`）。
- **没有任何地方存丢包率**（`loss` 在模型、TSDB 指标、响应结构里
  全文检索为零）。丢包一律推导：`loss% = 100 - up_percent`，
  其中 `up_percent = up / (up + down) * 100`。
- 降采样（`pkg/tsdb/query.go:54-67`）：1d 约 30 秒/点，7d 约 30 分钟/点，
  30d 约 2 小时/点；每桶只算**平均**延迟，不保留最小/最大延迟。
  详情里的“最低/最高延迟”取的是该范围返回的点列里的最小/最大值，
  不是探针原始采样值。
- 探针写路径：TCP/ICMP 按 `AvgPingCount`（默认 2 次）取平均后写一次；
  HTTP 每次上报都写（`service/singleton/servicesentinel.go:860-908`）。

### 主题变量（nezha-dash-v2 v2.7.0，`src/index.css`）

Tailwind v4 + shadcn 风格，`:root` / `.dark` 上是 HSL 三元组变量：
`--background / --foreground / --card / --card-foreground / --primary /
--muted-foreground / --border / --destructive / --chart-1..--chart-10`，
由 `hsl(var(--background))` 这种形式消费。暗色是 `<html>` 上的
`.dark` / `.light` class（不是 data 属性），由 `localStorage["vite-ui-theme"]`
（`light|dark|system`，默认 `system`）+ `prefers-color-scheme` 决定。
本文件把它们映射成自己的 `--nl-*` 变量并带回退值，暗色变化时监听
`<html>` 的 class 变化并重绘图表。
根节点是 `<div id="root">`（React 19 + Vite 8，路由只有 `/` 和 `/server/:id`）。

## Komari → Nezha v2 对照表

| Komari 参考站 | Nezha v2 本项目 | 备注 |
|---|---|---|
| 首页卡片底部：当前延迟 + 丢包率 + 两条分段历史条 | 面板服务卡片：平均延迟 + 丢包 + 30 段延迟/正常率条；服务器表：当前延迟 + 丢包 | v2 的 30 段来自 `GET /api/v1/service` 的 30 天日数组 |
| 详情 Network：SCCM/SCCU/SCCT 三探针 | 详情：该服务器的全部探针（HTTP/ICMP/TCP，按服务配置来，不固定 3 个） | Nezha 的探针由管理员在 `/dashboard/service` 配置，名字/类型/目标都不同；Komari 的四川三运营商是参考站自己的配置 |
| 平均延迟 / 平均丢包（每探针） | 同上：平均延迟取历史均值；平均丢包 = 100 − 正常率 | v2 无丢包字段，均为推导值 |
| 双轴图（左延迟 ms / 右丢包 %） | ECharts 5 双轴图（左延迟、右丢包 0–100%）；单选探针显示丢包面积，多选只显示延迟线（与官方前端 `NetworkChart` 的行为一致） | 官方前端用 Recharts；本项目用 ECharts（图例筛选、dataZoom、大数据量降采样开箱即用） |
| 时间范围 1h/6h/12h/24h/3d/7d/30d | **1h（实时，虚拟档）/ 24h / 7d / 30d** | v2 `period` 只接受 `1d/7d/30d`，其它值接口直接报错（`pkg/tsdb/query.go:16-37`）。1h 是前端截取 1d 数据的最近 60 分钟，不是接口档；6h/12h/3d 仍没有 |
| Peak cut 开关 | 有，默认开，按 P95 截断延迟尖峰 | 官方前端也有 `window.ForcePeakCutEnabled`，本项目不依赖它，自己实现 |
| 图例可点击筛选 | ECharts 图例原生支持 | — |
| 离线 `-- / --`，探针不可达丢包 100.0% | 同上 | 在线判断见下 |
| 探针历史最长 30 天 | 同上（TSDB 默认保留 30 天） | 未配 TSDB 时走 SQLite，保留策略未在源码里确认，见“待验证” |

## 在线判断（实现说明）

`StreamServer.state` 是 `*HostState` 对象，不是布尔值（`model/server_api.go`），
所以本文件用 `last_active` 的新鲜度判断在线：`now - last_active ≤ 90 秒`
（`CONFIG.wsStaleSec`）视为在线；拿不到 `last_active` 时退化为“有 `state`
即在线”。WS 每 2 秒推一次，agent 约 1 秒上报一次，90 秒阈值留了足够余量。

## 已知限制

- **时间范围是 1h（实时，虚拟档）/ 24h / 7d / 30d**，没有 6h / 12h / 3d（v2 API 限制，见上）。1h 不是接口档，是前端从 1d 数据里截取最近 60 分钟；TSDB 模式下 1d 约 30s/点，1h 约 120 个点，SQLite 模式下点数可能更少。
- **游客可看 1h 和 24h**，7d / 30d 需要登录（v2 接口限制；游客点 7d/30d 会提示
  并自动切回 1h）。
- **丢包是推导值**，不是实测丢包字段：按桶统计（桶内失败记 100%，成功记
  0%），平均丢包 = 失败桶占比；1d 约 30 秒/桶，7d 约 30 分钟/桶，30d 约
  2 小时/桶，所以短时丢包会被桶平均稀释。
- **每桶只有平均延迟**，没有最小/最大延迟；详情里的最低/最高取自返回点列。
- **服务器卡片延迟条是尽力而为**：本文件尝试给官方前端的服务器卡片（`a[href^="/server/"]`）
  底部追加延迟条，官方前端改版后可能失效；失效不影响面板本身（面板内的服务
  器表是确定可用的）。卡片是 React 渲染、DOM 会重建，所以用 5 秒轮询补挂；
  首屏加载时自动拉一次数据，卡片条不依赖打开面板也能显示。
- **ECharts 靠 CDN**：jsdelivr 失败自动换 unpkg；两个都失败时图表区降级为
  聚合数据表（平均延迟/丢包/数据点数），不白屏。
- **TSDB 未启用时**：历史走 SQLite `ServiceHistory`，数据点可能比 TSDB 稀疏；
  `GET /api/v1/server/:id/metrics` 在 TSDB 关闭时返回空，本项目未使用该接口。
- **不适配 Nezha v1**：v1 项目（`nezha-beautify-v1`）用的 WS 与字段在 v2 里
  大部分仍在，但 v2 的服务聚合接口、游客范围限制、前端注入细节不同，
  请勿混用两个文件。
- **不包含欢迎气泡、资产统计、3D 地球、页面保护**（有意排除，见“功能”）。

## 修复记录：延迟面板点了打不开（2026-11-20）

**现象**：status.alicloud.ru（Nezha v2.6.2）上延迟按钮看得见，点多少次面板
都不弹出；`#nl-fab-summary` 为空、服务器卡片上也没有延迟角标。

**根因**：`NezhaLatency.html:320` / `NezhaLatency.js:77` 是
`var CFG = window.NezhaLatencyConfig;`，而外层先定义了已合并默认值的局部
变量 `var NezhaLatencyConfig = Object.assign({默认值…}, (window.NezhaLatencyConfig || {}));`
（HTML 第 291–315 行）。站上 custom_code 没有定义 `window.NezhaLatencyConfig`，
所以 `CFG` 是 `undefined`，内层顶层
`var state = { range: CFG.defaultRange, peakCut: CFG.peakCut.enabled, … }`
立即抛 `TypeError`，`boot()` / `init()` / 所有点击监听都没跑到；
`window.NezhaLatency` 也从未创建。按钮是外层 `inject()` 注入的（不依赖
`CFG`），所以看得见、点不动。数据和 ECharts 都不是瓶颈（`/api/v1/service`
返回 `success:true`、6 个探针，ECharts 5.5.0 正常加载）。

**修复**：内层改成用已合并默认值的局部变量，
`var CFG = NezhaLatencyConfig;`（加注释说明原因）。只改 HTML 再跑
`build-js.py` 重新生成 JS，不直接改 JS。验证：`node --check` 通过，
JS 里已无 `var CFG = window.NezhaLatencyConfig`。

## 改动记录：悬浮按钮移除，延迟改在卡片上展示（2026-11-21）

**用户要求**：延迟要展现在卡片上面，而不是外部按钮。

**改动**：
- 删掉右下角“延迟”悬浮按钮（`#nl-fab` 按钮节点、相关 CSS、点击绑定、
  摘要文字 `updateFab()`、`showFab` 配置项全部移除）；`build-js.py` 的防
  重复注入检查改成看 `#nl-modal-overlay` 是否已存在（原来是看 `#nl-fab`，
  不改的话每次重注都会重复注入）。
- 首页每张服务器卡片底部加延迟条（`attachCardStrips()`，复用面板服务卡片
  同款 `.nl-card-strip` 样式）：一行“延迟 xxx ms　丢包率 x.x%”，下面左右
  两条分段历史柱（左延迟、右丢包，各 10 段，对应最近 24 小时分 10 桶；
  延迟绿 <200ms · 黄 <300ms，丢包绿 <1% · 黄 <5%）。条的分桶数据在
  `loadServerLatency()` 里顺带算出（`histByServer[sid].buckets`，聚合
  `/api/v1/service/:id/history?period=1d` 的 `data_points`）。
- 只让条本身可点：条带 `cursor: pointer` + `title="点击查看延迟详情"`，
  点击 `preventDefault` + `stopPropagation` 后调 `openStripDetail(sid)`，
  先打开面板、等服务器数据就绪再 `openDetail(sid)`；不劫持整张卡片的点击，
  卡片本身点开仍是官方服务器详情页。
- 卡片条不依赖打开面板：`init()` 里首屏就调一次 `loadAll()` 拉数据；
  30 秒定时刷新改成始终跑（原来只在面板打开时跑），详情只在面板打开时刷新；
  React 重建卡片 DOM 后 5 秒轮询补挂，已存在的条只在数值变化时更新。
- 详情面板本身全部保留：`nl-modal-overlay`、1h/24h/7d/30d 范围、Peak cut、
  双轴图表、按探针平均、30 秒自动刷新、Esc 关闭、`window.NezhaLatency`
  API（open/close/refresh/config/state）都不动；CFG 取值仍是已合并默认值
  的局部变量（`var CFG = NezhaLatencyConfig;`）。

## 待验证 / Unknowns

以下内容在源码里无法确认，需要在真实 Nezha v2 实例上验证：

1. **仪表盘版本号**：本机 clone 无 tag，`Version` 默认 `"debug"`，无法确认
   main 分支就是 v2.7.0。`v2.7.0` 是前端清单锁定的**用户前端**版本。
   建议在真实实例上看 `GET /api/v1/setting`（管理员）返回的 `version` 字段核对。
2. **Komari 侧**：参考站的延迟 API 未公开、DOM 里无 JSON 可抓；Network 标签的
   探针名（SCCM/SCCU/SCCT）、范围档位、Peak cut 的具体行为均来自任务方实地
   查看，未在 Komari 源码里逐项核对。Komari 已迁移到 JSON-RPC
   `POST /api/rpc2`，其 ping 记录接口的方法名/参数未验证——本项目未使用
   任何 Komari 接口，所以不影响 Nezha v2 部分。
3. **SQLite 兜底的保留期**：TSDB 默认保留 30 天已确认；未启用 TSDB 时
   `ServiceHistory` 行是否会被清理、保留多久，未在源码里找到清理代码。
4. **反代 / 子路径**：WS 在反代后是否放行、子路径部署时相对路径是否正常，
   需在真实实例上验证；`CONFIG.apiBase` 可填绝对地址应急（需服务端允许 CORS）。
5. **CDN 可达性**：jsdelivr / unpkg / flagcdn（本项目未用 flagcdn）在访客
   网络下是否可达；ECharts 加载失败时会降级为数据表。
6. **在线阈值**：`wsStaleSec: 90` 是实现选择，未在官方前端里核对同类阈值；
   若 agent 上报间隔配得很大，可能把在线判成离线，可按需调大。
7. **服务覆盖与 HideForGuest**：`HideForGuest=true` 的服务/服务器对游客不可见；
   `cover=1`（仅指定服务器）时，未列出的服务器在该服务下无数据，详情里会
   显示“暂无探针数据”，这是正常的。

## 文件

- `NezhaLatency.html` —— 单文件交付物（样式 + 节点 + 脚本），粘贴进
  Nezha v2 后台自定义代码（`custom_code`）即用。
- `README.md` —— 本文件。
