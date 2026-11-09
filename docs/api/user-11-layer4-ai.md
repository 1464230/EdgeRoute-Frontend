# 四层转发与 AI(用户端)
> 路由总数 16,来源 EdgeUserPortal/internal/web/server.go

说明:
- 所有错误统一返回 `{"error":"消息"}`;未登录统一 `401` `请先登录`。
- "需登录+CSRF写保护" = 路由包了 `sensitiveUserWrite`(未登录 401;管理员协助会话 403 `管理员协助会话不能修改用户资源或安全配置`;平台授权失效 403 只读提示),并且处理函数内用 `validCSRF` 比对请求体字段 `csrfToken`(不匹配 403 `CSRF 校验失败`)。CSRF 令牌放在 JSON 请求体,不是请求头。
- 请求体使用 `decodeAPIJSON`:上限 64KiB、禁止未知字段、只允许单个 JSON 对象。
- 四层功能受套餐门控:`/api/layer4` 系列 GET 与 export 要求套餐含 `server.tcp` 或 `server.udp` 任一功能,否则 `403` `当前套餐未开通四层转发功能`;创建/更新项目按 `network` 要求对应的 `server.tcp` / `server.udp`,否则 `403` `当前套餐未开通X功能`;套餐功能读取失败 `502` `读取套餐功能失败`。
- `POST/PATCH/DELETE /api/layer4/projects*` 与 `POST /api/layer4/projects/{id}/status` 在请求体解码失败时返回 `400` `请求格式无效`(提交 eaa88f1 前曾直接 return,客户端收到空 200,已修复)。
- "公共结构"见表后。

| 接口名称 | 方法 | 路径 | 主要请求参数 | 返回数据结构 | 功能说明 | 认证 |
|---|---|---|---|---|---|---|
| 四层转发概览 | GET | `/api/layer4` | 查询参数:<br>`protocol` string 可选,`tcp`\|`udp`<br>`port` int 可选,1000-40000<br>`range` string 可选,`24h`\|`7d`\|`30d`,默认 `24h` | `200` `apiLayer4Response`:<br>`range` string<br>`rules` array(见公共结构 Layer4Rule)<br>`trends` array(见 Layer4Trend)<br>`metricsAvailable` bool<br>`metricsMessage` string<br>`totalConnections` int64<br>`totalTrafficInBytes` int64<br>`totalTraficOutBytes` int64(源码字段名拼写即为 `Trafic`)<br>`healthyOrigins` int<br>`unhealthyOrigins` int<br>错误:`403` 套餐未开通四层;`502` `读取四层转发数据失败`(所有收集错误统一映射为该文案) | 汇总站点分组的 TCP/UDP 反代与独立四层项目,返回规则列表、连接/流量统计和趋势。独立项目 `groupId=0`、`groupName="独立转发"`。 | 需登录 |
| 四层项目列表 | GET | `/api/layer4/projects` | 无 | `200` `{"projects":[Layer4Project...]}`(结构见公共结构)<br>错误:`403` 套餐未开通四层;`502` | 列出当前用户的独立四层转发项目。 | 需登录 |
| 四层项目详情 | GET | `/api/layer4/projects/{id}` | 路径参数:`id` int 必填(项目编号) | `200` Layer4Project 对象<br>错误:`400` `项目编号无效`;`404` `四层项目不存在`;`502` | 读取单个独立四层项目。 | 需登录 |
| 四层项目诊断 | GET | `/api/layer4/projects/{id}/diagnostics` | 路径参数:`id` int 必填 | `200` 原样输出 RPC 返回的 JSON(`Cache-Control: no-store`)。参考 `serverconfigs.Layer4Diagnostics`:`unavailable`、`collectedAt`、`startedAt`、`active`、`accepted`、`trafficIn`、`trafficOut`、`reasons`(map[string]int64);待确认:RPC 是否直接返回该结构,源码此处按 `json.RawMessage` 透传<br>错误:`400` `项目编号无效`;`503` `当前版本不支持四层诊断`;`403` `无法读取该项目诊断，请确认权限和节点状态`(RPC 任意错误均映射为 403) | 读取项目的运行时诊断信息(活动连接、接入数、流量、拒绝原因分布)。 | 需登录 |
| 创建四层项目 | POST | `/api/layer4/projects` | JSON 请求体:<br>`csrfToken` string 必填<br>`name` string 必填,1-200<br>`network` string 必填,`tcp`\|`udp`<br>`port` int 必填,10000-40000<br>`userPlanId` int<br>`origins` array 必填,1-32 项,每项 `id`、`name`、`host` 必填、`port` 1-65535、`protocol` `tcp`\|`udp`\|`tls`、`enabled` bool、`backup` bool<br>`limits` object,`maxConnectionsPerIP` 0-100000、`newConnectionsPerSecond` 0-100000、`packetsPerSecond` 0-1000000<br>`schedulingCode` string<br>`connTimeout`/`readTimeout`/`idleTimeout` apiTimeDuration<br>`maxConns`/`maxIdleConns` int<br>`proxyProtocol` object | 成功 `201` 返回完整 Layer4Project;创建后查询不到时降级为 `{"id":id}`。<br>错误:`400`(经 `localizeLayer4Error` 本地化的校验文案);`403` CSRF/套餐未开通对应 tcp/udp 功能;`503` `当前 API 不支持项目防护设置`(`limits` 非空但 RPC 不支持)。<br>请求体解码失败时无响应(见说明)。 | 创建独立四层转发项目,端口由用户在 10000-40000 范围内指定。需要套餐含对应协议功能。 | 需登录+CSRF写保护 |
| 更新四层项目 | PATCH | `/api/layer4/projects/{id}` | 路径参数:`id` int 必填<br>JSON 请求体:同创建项目(`csrfToken`、`name`、`network`、`port`、`userPlanId`、`origins`、`limits`、`schedulingCode`、`connTimeout`、`readTimeout`、`idleTimeout`、`maxConns`、`maxIdleConns`、`proxyProtocol`) | 成功 `200` `{"ok":true}`。<br>错误:`400` `项目编号无效` / `至少需要一个主源站` / 其他校验文案;`403`;`404`;`503` `当前 API 不支持项目防护设置`。<br>请求体解码失败时无响应。 | 全量更新项目配置。`origins` 中至少要有一个非备用(`backup=false`)的主源站。 | 需登录+CSRF写保护 |
| 删除四层项目 | DELETE | `/api/layer4/projects/{id}` | 路径参数:`id` int 必填<br>JSON 请求体:`csrfToken` string 必填 | 成功 `200` `{"ok":true,"auditLogged":bool}`。<br>错误:`400` `项目编号无效`;`403`;`404`。<br>请求体解码失败时无响应。 | 删除项目。风险操作,不可恢复;`auditLogged` 表示审计日志是否写入成功。 | 需登录+CSRF写保护 |
| 启停四层项目 | POST | `/api/layer4/projects/{id}/status` | 路径参数:`id` int 必填<br>JSON 请求体:`csrfToken` string 必填;`isOn` bool(true 启用,false 停用) | 成功 `200` `{"ok":true}`。<br>错误:`400` `项目编号无效`;`403`。<br>请求体解码失败时无响应。 | 切换项目启用状态。 | 需登录+CSRF写保护 |
| 导出四层规则 CSV | GET | `/api/layer4/export` | 查询参数同 `/api/layer4`:`protocol`、`port`、`range` | `200` `text/csv`,下载文件名 `edgeroute-layer4-rules.csv`,列:分组、协议、源站地址、端口、状态、健康状态。<br>错误:`403` 套餐未开通四层;`502` `读取四层转发数据失败` | 按当前筛选条件导出四层规则列表。 | 需登录 |
| 批量启停四层规则 | POST | `/api/layer4/batch` | JSON 请求体:<br>`csrfToken` string 必填<br>`action` string 必填,`enable`\|`disable`<br>`rules` array 必填,1-50 项,每项 `groupId` int、`network` `tcp`\|`udp`、`originId` int | `200` `{"ok":bool,"failed":[{"groupId","network","originId","error"}]}`。部分失败也返回 200,以 `ok=false` 且 `failed` 非空表示。<br>错误:`400`;`403`。 | 批量启用/停用站点分组下的 TCP/UDP 源站规则,逐条执行并收集失败原因。 | 需登录+CSRF写保护 |
| 读取 AI 配置与状态 | GET | `/api/ai` | 无 | `200`:<br>`config` `{defenseMode,optimizationMode,updatedAt}`<br>`state` UserState(摘要:`status`、`lastDefenseAnalysisAt`、`lastOptimizationAt`、`lastError`、`metrics`、`siteMetrics`、`recommendations`、`changes`、`usageDay`、`requestsToday`、`siteRequestsToday`、`nextDefenseAnalysisAt`、`nextOptimizationAt`、`analysisRuns`、`manualTask`、`siteHealth`、`leases`、`autoWafRecoveries`)<br>`defenseAvailable` bool<br>`optimizationAvailable` bool<br>`autoAplyAvailable` bool(源码字段名拼写即为 `Aply`)<br>`autoExecutionEnabled` bool<br>`siteDomains` map[siteId]域名<br>错误:`403` `AI 功能未授权或暂不可用` / `AI 功能授权校验失败`;`502` | 读取当前用户的 AI 防御/优化模式、分析状态、建议和变更记录。 | 需登录 |
| 更新 AI 模式 | PUT | `/api/ai` | JSON 请求体:<br>`csrfToken` string 必填<br>`defenseMode` string,`off`\|`analyze`\|`review`\|`auto`,空视为 `off`<br>`optimizationMode` string,取值同上 | 成功 `200` `{"ok":true}`。<br>错误:`403`;`422`(直接输出 RPC 返回的错误信息);`400` 模式取值无效(待确认:具体文案)。 | 设置 AI 防御与优化的工作模式。 | 需登录+CSRF写保护 |
| 触发 AI 分析 | POST | `/api/ai/analyze` | JSON 请求体:<br>`csrfToken` string 必填<br>`category` string 必填,`defense`\|`optimization`<br>`serverId` int,≥0(0 表示全部站点)<br>`windowDays` int,`0`\|`1`\|`7`\|`30` | 成功 `202` `{"ok":true}`(异步执行)。<br>错误:`400` 参数无效;`403`;`422`(RPC 错误信息)。 | 手动发起一次 AI 分析任务,结果通过 `GET /api/ai` 的 `state.manualTask` / `recommendations` 获取。 | 需登录+CSRF写保护 |
| 应用 AI 建议 | POST | `/api/ai/recommendations/{recommendationID}/apply` | 路径参数:`recommendationID` string 必填,匹配 `[A-Za-z0-9-]` 且 ≤128<br>JSON 请求体:`csrfToken` string 必填 | 成功 `200` `{"ok":true}`。<br>错误:`400` `AI 记录标识无效`;`403`;`422`(RPC 失败文案)。 | 采纳并执行一条 AI 建议,生成对应变更记录。与 reject、rollback 共用 `handleAIAction`。 | 需登录+CSRF写保护 |
| 拒绝 AI 建议 | POST | `/api/ai/recommendations/{recommendationID}/reject` | 路径参数同上<br>JSON 请求体:`csrfToken` string 必填 | 成功 `200` `{"ok":true}`。<br>错误同上。 | 忽略一条 AI 建议。 | 需登录+CSRF写保护 |
| 回滚 AI 变更 | POST | `/api/ai/changes/{changeID}/rollback` | 路径参数:`changeID` string 必填,匹配 `[A-Za-z0-9-]` 且 ≤128<br>JSON 请求体:`csrfToken` string 必填 | 成功 `200` `{"ok":true}`。<br>错误:`400` `AI 记录标识无效`;`403`;`422`。 | 回滚一条已执行的 AI 变更。 | 需登录+CSRF写保护 |

## 公共结构

### Layer4Rule(`/api/layer4` 的 `rules[]`)
- `id` int64、`groupId` int64(独立项目为 0)、`groupName` string、`network` string(`tcp`\|`udp`)
- `originId` int64、`originName` string、`originHost` string、`portRange` string
- `enabled` bool、`serverCount` int、`healthyServers` int
- `connectionCount` int64、`trafficInBytes` int64、`trafficOutBytes` int64
- `failureRate` number 可选
- `healthStatus` string:`healthy`\|`degraded`\|`unavailable`\|`disabled`
- `healthCheckOn` bool、`autoRemove` bool

### Layer4Trend(`trends[]`)
- `time` string、`concurrent` int64、`trafficInBytes` int64、`trafficOutBytes` int64

### Layer4Project
- `id` int64、`name` string、`cname` string、`network` `tcp`\|`udp`、`port` int、`enabled` bool、`userPlanId` int64、`createdAt`
- `origins` array:`id`、`name`、`host`、`port`、`protocol`(`tcp`\|`udp`\|`tls`)、`enabled`、`backup`
- `limits` 可选:`maxConnectionsPerIP`、`newConnectionsPerSecond`、`packetsPerSecond`
- `schedulingCode` string、`connTimeout`/`readTimeout`/`idleTimeout` 可选 `{count,unit}`、`maxConns` int、`maxIdleConns` int、`proxyProtocol` 可选
- 待确认:`createdAt` 的具体时间格式与 `proxyProtocol` 子字段,本模块未逐字段核对。

### AI 模式取值
`off`(关闭)\|`analyze`(仅分析)\|`review`(需人工确认)\|`auto`(自动执行);`autoExecutionEnabled`/`autoAplyAvailable` 决定 `auto` 是否可用。
