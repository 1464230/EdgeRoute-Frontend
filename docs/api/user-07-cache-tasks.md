# 缓存任务(用户端)

> 路由总数 4,来源 EdgeUserPortal/internal/web/server.go

说明:错误统一为 `{"error":"消息"}`。"需登录+CSRF写保护"= 经 `sensitiveUserWrite`(未登录 401"请先登录";管理员协助会话 403;授权只读 403;记录操作日志),且 JSON 请求体须携带 `csrfToken`(失败 403"CSRF 校验失败")。创建缓存任务的接口位于站点模块(`POST /api/sites/{siteID}/cache/tasks`)。

| 接口名称 | 方法 | 路径 | 主要请求参数 | 返回数据结构 | 功能说明 | 认证 |
|---|---|---|---|---|---|---|
| 缓存任务列表 | GET | `/api/cache-tasks` | 查询参数:<br>`offset` int,≥0,默认 0<br>`type` string,可选,`purge`/`fetch`<br>`status` string,可选,`doing`/`success`/`failed`<br>`days` int,可选,取值 0/7/30/90<br>每页固定 20 条 | 200 `{"tasks":[CacheTask],"count":int}`(`CacheTask` 见公共结构)<br>错误:400"任务分页参数无效"/"任务筛选参数无效";503(RPC 不支持);502"读取任务记录失败" | 分页查询当前用户的缓存刷新/预热任务记录。 | 需登录 |
| 缓存任务额度 | GET | `/api/cache-task-quota` | 查询参数:`type` string 必填,`purge`/`fetch` | 200 `{"day":string,"maxKeysPerTask":int,"maxKeysPerDay":int,"usedKeys":int,"remainingKeys":int}`,响应头 `Cache-Control: no-store`<br>错误:400;503"额度暂不可用,请联系管理员升级服务";502 | 查询当日该类型任务的单次与每日 URL 额度及已用量。 | 需登录 |
| 缓存任务详情 | GET | `/api/cache-tasks/{taskID}` | 路径参数:`taskID` int64,须 >0 | 200 `{"task":CacheTask}`<br>错误:400"任务编号无效";404"任务不存在";503"任务记录暂不可用" | 读取单个任务及其逐 URL 状态。与重试接口共用处理函数 `apiCacheTask`,按请求方法分支。 | 需登录 |
| 重试缓存任务 | POST | `/api/cache-tasks/{taskID}/retry` | 路径参数:`taskID` int64,须 >0<br>JSON 请求体(上限 4KB):`csrfToken` string 必填 | 201 `{"id":int64,"count":int}`(新任务编号与 URL 数)<br>错误:400"任务编号无效";404"任务不存在";409"仅已结束的失败任务可以重试"/无可重试 URL;502;503 | 仅已结束且失败的任务可重试。优先重提失败的 key,无失败 key 则重提全部;会新建任务,不修改原任务。执行顺序:先查任务(404),再解码 body 和校验 CSRF。 | 需登录+CSRF写保护 |

## 公共结构

### CacheTask
- `id` int64
- `sequence?` int(可选)
- `type` string:`purge`/`fetch`
- `state` string:`waiting`/`running`/`success`/`failed`
- `createdAt` string
- `doneAt` string
- `keys` 数组,元素 `{"url":string,"state":string(waiting/running/success/failed),"error?":string}`

`createdAt`/`doneAt` 的具体时间格式及未完成时的取值:待确认,未核实格式化函数。
