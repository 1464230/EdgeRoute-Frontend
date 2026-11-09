# EdgeRoute 后端接口总路由表

> 范围：用户端（EdgeUserPortal）与管理端（EdgeAdmin）对外 HTTP 接口，共 **700** 条路由、26 个模块。其中 697 条在各模块主表（带"模块"列），另有 3 条前端性能上报接口（`/api/metrics/web-vitals`、`/api/metrics/performance-timeline`、`/api/metrics/query`）位于第 14 模块末尾的补充小节，格式不同（无"模块"列）；`/api/metrics/query` 在第 8 模块主表中也出现一次，属同一路由的重复记录。
> 依据：现有后端源码的逐模块梳理（`docs/api/*.md`），本文件是对它们的合并与重排，未新增任何实现代码。
> 标注“待确认”的内容表示梳理时未读到对应源码或未实测，实现时不要当作事实，见文末附录 A。

## 0. 全局约定

| 项 | 用户端（`/api/*`、`/oauth/*`） | 管理端（`/api/admin/*`、`/setup/*`、`/recover/*`、`/csrf/token`） |
|---|---|---|
| 请求体 | JSON，`DisallowUnknownFields`（多一个字段即 400“请求格式无效”），默认上限 64KB，超限 413 | **不解析 JSON**；参数只走 query 与表单（`application/x-www-form-urlencoded` 或 multipart）；数组用重复 key；少数字段的值本身是 JSON 字符串（如 `*JSON`） |
| 成功响应 | 直接返回业务对象（创建类为 201） | `{"code":200,"message":"","data":{...}}`，下表“响应数据”只写 `data` 内容 |
| 失败响应 | `{"error":"消息"}`，HTTP 状态即错误码 | `this.Fail`：`{"code":400,"message":"...","data":...,"errors":null}`，HTTP 状态是否为 200 未实测；`actionutils.Fail`：gRPC 状态映射为 HTTP（InvalidArgument 400 / Unauthenticated 401 / PermissionDenied 403 / NotFound 404 / AlreadyExists 409 / ResourceExhausted 429 / Unavailable 503 / DeadlineExceeded 504 / 其他 500） |
| CSRF | 先 `GET /api/session` 取会话 Cookie 与 `csrfToken`，`csrfToken` 放在 JSON 请求体（不是请求头） | 令牌来自 `GET /api/admin/session` 的 `data.csrfToken`（30 分钟，绑定会话），POST 必带表单字段 `csrfToken`；失败为 **403 纯文本**，不是 JSON。例外：`/setup/*`、`/recover/*` 不要 CSRF，改用 `setupToken`；`node-join` 不要 CSRF |
| 认证 | 会话 Cookie；“公开”=无需登录，“登录”=需用户会话 | 会话 Cookie + 模块权限（Common / Server / Node / Setting / Admin / finance 等）；无会话 401，权限不足 403，系统未初始化 409 |
| 分页 | 以各接口说明为准 | `page`（<1 取 1）、`size`（<1 取 20，>100 取 100） |
| 前端性能上报 | — | `/api/metrics/*` 三条使用 JSON、独立错误格式 `{"code","message"}`、成功 204，见管理端“安装向导、找回、确认与 CSRF”模块 |

表格说明：“请求参数”单元格内已分行标注路径参数、查询参数、请求体字段及类型；“主要错误”从原响应里拆出，没有逐条错误的行写“见模块约定”，即该模块通用的 401/403/CSRF 规则。

### 0.1 更新类接口的"漏传保持原值"语义（接口重构后生效）

路由与字段名没有变化，变化的是更新类接口对缺省字段的处理。旧行为是漏传的布尔/数值被当作 `false`/`0` 写回，会静默关闭开关或清零阈值。新行为统一为"读当前值，只覆盖显式传入的字段"：

| 端 | 规则 |
|---|---|
| 用户端（JSON） | 更新体里的布尔/数值字段为可选；**不传 = 保持原值**，传 `false`/`0` = 明确设置。整块配置（如站点协议、WAF、分组 WAF）以当前配置为底，只覆盖传入的键；列表字段传入即整体替换。`DisallowUnknownFields` 不变 |
| 管理端（表单） | 字段是否存在按表单 key 判断：**key 不存在 = 保持原值**；`key=false`、`key=0` 才是明确设置。空列表在表单中不会被编码，所以列表字段只在"整块提交标志字段"也缺失时才继承旧值。老前端整块提交的行为不变 |
| 管理端（整块 JSON 字符串，如站点各 `*JSON`） | 以当前配置为底，对传入 JSON 的键做覆盖；未出现的键保持原值 |
| 密钥类字段 | 留空 = 保留已保存的密钥，不回填明文，也不会被清空 |
| 权限类字段 | `isSuper`/`isOn`/`canLogin`/`otpOn`/模块权限列表绝不因漏传被重置 |
| 类型切换 | 如防火墙动作切换 `type` 时，不继承旧类型的参数 |
| 一次性动作标志 | 如 `keepOldConfig`、`clear*`/`reset*` 不回填，只在显式传入时生效 |

仍为"整体替换/单字段开关"语义、无需回填的接口：各类 Create/Sort、`UpdateNodeIsOn`、`FinancePackageUpdate/PeriodUpdate/PriceUpdate`、`FinanceRegionalPriceUpdate`、`UpdateRuntimeSettings`（整块 ConfigJSON）、`UpdateSiteGlobalPages`、`WAFPolicySetToggle`、`UpdateNodeDNS`、`UpdateNodeIPAddressThresholds`、站点 redirect/request-script 的整体列表替换。

另有三处行为变化需前端注意：

- `POST /api/admin/maintenance/updates/apply` 与 `.../updates/user/apply` 的 `confirmation` 改为**必填**，值为 `UPDATE <版本>` / `UPDATE USER`，留空不再跳过确认。
- 带密钥的 GET（源站保护 reveal、api-nodes/install、user-nodes/install、otp-qrcode）要求超级管理员并写审计日志。
- `/recover/*` 需要口令；`/setup/status` 不再返回错误详情。
- 用户端 `POST /api/security/mfa/confirm` 新增必填 `currentPassword`；`POST /api/access-keys` 新增必填 `currentPassword`，账号已启用 MFA 时还必须带 `otpCode`。校验失败返回 403，连续失败会被限流（429）。这两个接口的响应带 `Cache-Control: no-store`（`/api/security/mfa/setup` 同）。
- 用户端登录按「IP+用户名」限流，注册、邮件重发、邮箱验证、关闭 MFA 各有独立限流键，超限返回 429。
- RPC 失败按 gRPC 状态码映射 HTTP 状态：PermissionDenied→403、NotFound→404、AlreadyExists→409、InvalidArgument→400、ResourceExhausted→429、DeadlineExceeded→504，其余仍为 502。
- 用户端审计日志改为写操作成功（HTTP<400）后记录，描述附结果状态码。
- 管理端 POST 写操作若处理器未自行记录日志，由 helper 兜底写一条操作日志（路径与结果，不含请求体）；数据库维护的读取与写入接口要求超级管理员。
- `POST /api/admin/node-join` 等匿名接口按来源 IP 限流（60 次/分钟）。

### 0.2 统一规范补充（全量整改后生效）

用户端：

- 请求体超过大小上限返回 **413**「请求体过大」，JSON 格式或未知字段返回 400「请求格式无效」。
- 分页参数 `offset`（0..1000000，默认 0）、`size`（1..100，默认 20）严格校验，非法值返回 **400**「分页参数无效」，不再静默回退默认值。适用于消息、已读消息、操作记录、工单列表。
- `/api` 下未匹配的路径返回 JSON `404 {"error":"接口不存在"}`，不再回落到 SPA 页面。
- 所有 JSON 响应默认带 `Cache-Control: no-store`（公开的 portal/openapi 等显式缓存的接口除外），`/api/session` 等返回 `csrfToken` 的响应不会被缓存。
- 上游 RPC 失败按 gRPC 状态码映射（见上），超时为 504；注册、AI、实名认证的错误同样经过敏感信息过滤。
- `POST /api/cache-tasks/{taskID}/retry` 先校验 CSRF 再访问上游；任务读取失败时只有明确的 NotFound 才返回 404，其他按上游状态映射。

管理端：

- 节点写操作（创建、更新、启停、删除，含 API 节点、用户节点）归 Node 模块；仅有 Server 模块权限的管理员不能再操作节点。
- 授权（license）激活/刷新/释放/离线、API 节点与用户节点的创建更新启停删除、数据库节点的保存启停删除状态、审计日志删除，要求**超级管理员**。
- `GET /api/admin/maintenance/health`、`/health/history`、`/doctor` 归 Setting 模块；`GET /api/admin/security-overview` 与 `GET /api/admin/security-events/notifications` 归 Server 模块。`/dashboard` 是 Common 组唯一保留的接口。
- `POST /api/admin/maintenance/transfers/start` 与 `/rollback` 新增必填 `confirmation`，值须等于 `planId`。
- 批量 ID 参数（转移 `nodeIds`、发布批次 `nodeIds`、证书 OCSP `certIds`、消息告警 `ids`）统一校验：元素须大于 0，自动去重，单次最多 500 个，超限返回错误。
- `POST /api/admin/messages/alert-state` 的 `state` 仅允许 `open` / `acknowledged` / `silenced`，`silencedUntil` 不得为负。
- ACME 任务的更新、删除、运行要求 `taskId` 大于 0。

## 目录

1. [管理端 · 认证与会话(管理端)](#m1)（11 条）
2. [管理端 · 网站与网站分组(管理端)](#m2)（66 条）
3. [管理端 · 节点与集群(管理端)](#m3)（53 条）
4. [管理端 · 节点与集群后半部分(管理端)](#m4)（53 条）
5. [管理端 · DNS 管理(管理端)](#m5)（22 条）
6. [管理端 · 证书与 ACME(管理端)](#m6)（27 条）
7. [管理端 · WAF/IP名单/缓存(管理端)](#m7)（46 条）
8. [管理端 · 日志与指标(管理端)](#m8)（26 条）
9. [管理端 · 财务(管理端)](#m9)（47 条）
10. [管理端 · 用户与管理员(管理端)](#m10)（21 条）
11. [管理端 · 工单/消息/通知(管理端)](#m11)（26 条）
12. [管理端 · 系统设置(管理端)](#m12)（32 条）
13. [管理端 · 运维维护(管理端)](#m13)（31 条）
14. [管理端 · 安装向导、找回、确认与 CSRF 令牌(管理端)](#m14)（24 条）
15. [用户端 · 认证与会话(用户端)](#m15)（17 条）
16. [用户端 · 安全设置(用户端)](#m16)（11 条）
17. [用户端 · 仪表盘与个人资料(用户端)](#m17)（12 条）
18. [用户端 · 消息与工单(用户端)](#m18)（12 条）
19. [用户端 · 计费与订单(用户端)](#m19)（17 条）
20. [用户端 · 站点管理(用户端)](#m20)（27 条）
21. [用户端 · 缓存任务(用户端)](#m21)（4 条）
22. [用户端 · 证书 / DNS 服务商 / ACME 自动签发(用户端)](#m22)（27 条）
23. [用户端 · 静态网站 Pages(用户端)](#m23)（22 条）
24. [用户端 · 站点分组配置(用户端)](#m24)（40 条）
25. [用户端 · 四层转发与 AI(用户端)](#m25)（16 条）
26. [用户端 · 系统与运维(用户端)](#m26)（7 条）

- [附录 A 待确认事项](#pending)
- [附录 B 疑似缺陷与字段偏差](#defects)

<a id="m1"></a>

## 1. 管理端 · 认证与会话(管理端)

**模块约定**

> 路由总数 11,来源 `EdgeAdmin/internal/web/actions/default/api/admin/init.go`(处理器:`auth.go`、`session.go`、`logout.go`、`node_enrollment_resource.go`、`dashboard.go`、`security_overview_resource.go`、`security_event_notification_resource.go`、`tls_baseline_resource.go`)
说明:
- 路径前缀 `/api/admin`。处理器基于 TeaGo action,参数来自 URL query 与 POST 表单(`application/x-www-form-urlencoded` 或 multipart),不解析 JSON body。字段名按"原名/首字母小写名"匹配,字符串值自动去除首尾空白,数值解析失败按 0 处理。
- 响应包络(`this.Success()`):`{"code":200,"message":"","data":{...}}`,可能带 `next`。失败(`this.Fail(msg)`):`{"code":400,"message":"...","data":{...},"errors":null}`,`code` 默认 400,`this.Fail` 不显式写 HTTP 状态码(HTTP 状态码是否为 200 待确认)。
- 失败(`actionutils.Fail`):先把 gRPC 状态映射为 HTTP 状态(InvalidArgument 400 / Unauthenticated 401 / PermissionDenied 403 / NotFound 404 / AlreadyExists 409 / ResourceExhausted 429 / Unavailable 503 / DeadlineExceeded 504 / 其他 500),`message` 为"服务端错误文案(脱敏后的详情)"。含密码、令牌等敏感词的错误详情会被替换为固定文案。
- CSRF:POST 接口(`node-join` 除外)的 params 声明了 `CSRF`,须带参数 `csrfToken`(query 或表单字段,不是 JSON body)。令牌由 `GET /api/admin/session` 的 `data.csrfToken` 下发,绑定当前会话 sid,有效期 30 分钟。校验失败返回 HTTP 403 纯文本 `表单已失效，请刷新页面后重试(001)`,不是 JSON 包络。
- 登录态:凭 Cookie 会话。受保护接口(`helpers.NewUserMustAuth(模块)`)依次检查:恶意请求/WAF 拦截(403)、IP 白名单(403)、系统未初始化(409,`{"code":409,"message":"system setup is required","data":null}`)、无会话(401,`{"code":401,"message":"authentication required","data":null}`)、会话指纹或区域变化(401 或重定向到 `/login/validate`)、管理员不存在(401)、模块权限不足(403,`{"code":403,"message":"permission denied","data":null}`)。
- 权限模块:`Common`(`AdminModuleCodeCommon`)、`Setting`(`AdminModuleCodeSetting`)、`Server`(`AdminModuleCodeServer`)。
- 本模块 `session`、`login`、`otp`、`logout`、`node-join`、`node-join/installer.py` 注册为公开路由(不经 `NewUserMustAuth`)。其中 `login`、`otp`、`logout` 仍要求 `csrfToken`;`session`、`node-join*` 不要求。

| 模块 | 接口名称 | 方法 | 路径 | 请求参数 | 响应数据 | 主要错误 | 功能说明 | 认证/权限 |
|---|---|---|---|---|---|---|---|---|
| admin-01-auth-session | 节点安装脚本 | GET | `/api/admin/node-join/installer.py` | 无参数 | 成功:非 JSON,直接输出 Python 脚本文本,响应头 `Content-Type: text/x-python; charset=utf-8`、`Cache-Control: no-store`。 | 失败:脚本渲染失败返回 HTTP 500,无 body。<br>脚本具体内容由 `nodeinstall.RenderScript` 渲染,待确认(未读到该函数源码) | 下发节点自助安装用的 Python 安装脚本(NodeJoinInstallerAction),脚本本身不含凭据,凭据由后续 `POST /node-join` 的请求头提交 | 公开,无需登录,无需 CSRF |
| admin-01-auth-session | 节点安装通道 | POST | `/api/admin/node-join` | URL 不得带 query(否则"安装凭据不能放在 URL 中")。<br>请求头:`Authorization: Bearer <64 位小写十六进制令牌>`(必填,恰好 1 个)、`X-EdgeRoute-Join-Client: <64 位小写十六进制>`(必填,恰好 1 个),不符合返回"安装凭据无效"。<br>请求体:JSON 单个对象,不超过 8192 字节,不允许未知字段。可用字段:`action`(string,必填,取值 metadata/download/claim/progress,否则"不允许此安装操作")、`arch`(string)、`name`(string)、`phase`(string)、`failureCode`(string)、`offset`(整数)。`id/clusterId/groupId/regionId/count/level/address` 会被服务端强制清零,`token/client` 以请求头为准 | 成功:不使用标准包络。`action=download` 返回 `application/octet-stream` 二进制;其余 action 直接输出 RPC 返回的原始 `PayloadJSON`(`application/json`)。响应头 `Cache-Control: no-store`、`X-Content-Type-Options: nosniff`。<br>`PayloadJSON` 的具体字段由控制面决定,可参考 `nodeenrollment.Result`:items/tokens/record/config/manifest/publicKey/size/sha256,实际返回哪些待确认。 | 失败:格式错误 HTTP 400;并发槽(8)占满 HTTP 429"安装请求过多，请稍后重试";控制面不可用 HTTP 503"控制面连接失败，请稍后重试";任何 RPC 错误一律 HTTP 403"安装任务无法继续，请在管理端检查令牌期限、任务状态、签名包和 API 连接配置" | 节点安装器使用的无登录通道,按 action 执行元数据读取、安装包下载、认领和进度上报(NodeJoinAction)。令牌有效期 15 分钟。注意:不使用 `ParentAction`,响应不是标准包络;所有 RPC 错误统一为 403,无法区分原因 | 公开,无需登录,无需 CSRF,凭安装令牌请求头鉴权 |
| admin-01-auth-session | 会话信息 | GET | `/api/admin/session` | 无参数 | `data`:`authenticated`(bool)、`setupRequired`(bool,系统是否待初始化)、`csrfToken`(string,供后续 POST 使用)、`loginToken`(string,供 `login` 使用,有效期 30 分钟)、`systemName`、`productName`、`version`、`showVersion`(bool)、`faviconFileId`、`logoFileId`、`menus`(菜单,结构待确认)。<br>已登录时另有 `admin:{id,name}`。 | 错误:无 | 前端启动时调用,获取登录状态、CSRF 令牌与登录令牌(SessionAction)。无会话 Cookie 时会写入会话 Cookie | 公开,无需登录,无需 CSRF |
| admin-01-auth-session | 登录 | POST | `/api/admin/login` | `username`(string,必填)、`password`(string,必填,不超过 72 字节)、`remember`(bool,默认 false,记住登录)、`loginToken`(string,必填,来自 `session`)、`csrfToken`(string,必填) | 成功:`data.requireOTP`(bool);需要二次验证时 `requireOTP=true` 且 `data.challenge`(32 位十六进制,有效期 5 分钟,绑定 IP 与指纹),不需要时已直接登录且 `requireOTP=false`。<br>错误信息:"请输入用户名和密码"、"登录页面已失效，请刷新后重试"、"登录尝试过于频繁，请稍后重试"(同一 IP 5 分钟内最多 20 次)、"管理服务暂时不可用"、"用户名或密码不正确"、以"登录失败次数过多"开头的锁定提示(原样透传);CSRF 失败为 403 纯文本 | 见模块约定 | 管理员用户名密码登录(LoginAction),开启 OTP 的账号需继续调用 `otp` | 公开,无需登录,需要 CSRF |
| admin-01-auth-session | 动态密码验证 | POST | `/api/admin/otp` | `challenge`(string,必填,32 位,来自 `login`)、`otpCode`(string,必填,长度 6 到 8)、`remember`(bool,默认 false)、`csrfToken`(string,必填) | 成功:`data` 为空对象。<br>错误信息:"动态密码验证请求无效，请重新登录"(challenge 或 otpCode 长度不符)、"验证尝试过于频繁，请稍后重试"、"动态密码验证已过期或登录环境已变化，请重新登录"、"动态密码不正确"、"动态密码错误次数过多，请重新登录"(累计失败 5 次后挑战作废) | 见模块约定 | 完成 OTP 二次验证并建立登录会话(OTPAction)。挑战 sid 前缀 `admin-otp-`,绑定 IP 与指纹 | 公开,无需登录,需要 CSRF |
| admin-01-auth-session | 退出登录 | POST | `/api/admin/logout` | `csrfToken`(string,必填) | 成功:`data` 为空对象。 | 错误:CSRF 失败 403 纯文本 | 清除当前登录会话(LogoutAction) | 公开(无登录检查),需要 CSRF |
| admin-01-auth-session | 仪表盘 | GET | `/api/admin/dashboard` | 无参数 | `data`:<br>计数:`countServers`、`countNodeClusters`、`countNodes`、`countOflineNodes`(原样拼写)、`countUsers`、`countAPINodes`、`countOfflineAPINodes`、`countDBNodes`。<br>流量:`todayTrafficBytes`、`todayCountIPs`、`todayRequests`、`accounting`(计费统计,结构待确认)。<br>`diskUsageWarning`(磁盘告警,结构待确认)。<br>`accessLogQueue`:`{pending,retry,capacity,dropped,usagePercent,isWarning}`。<br>`hourlyTrafficStats[]`:`{hour,bytes,countRequests,countAttackRequests}`。<br>`topDomainStats[]`:`{serverId,domain,countRequests,bytes}`。 | 错误:RPC 失败走 `actionutils.Fail` | 管理端首页统计(DashboardAction) | 登录 + Common |
| admin-01-auth-session | 安全概览 | GET | `/api/admin/security-overview` | `day`(string,可选,格式 `YY-MM-DD`,默认当天,格式错误提示"日期格式无效") | `data` 顶层键:`day`、`checkedAt`、`dataStatus`、`waf`、`traffic`、`logStatus`(`sampled`/`unavailable`)、`logSampleSize`、`logSampleHasMore`、`events`、`apiDiscovery`、`botSummary`、`threatIPs`、`eventHistory`、`wafPolicies`、`wafPolicyCount`、`wafPolicyStatus`。<br>`waf`:`available,dayFound,dailyLogs,dailyBlocks,dailyCaptcha,weeklyBlocks`;`hourly[]`:`{hour,countLogs,countCaptcha,countBlocks}`;`daily[]`:`{day,countLogs,countCaptcha,countBlocks}`;`groups[]`:`{id,name,count}`;`nodes[]`:`{nodeId,nodeName,countRequests,bytes,countAttackRequests,attackBytes}`;`domains[]`:`{serverId,domain,countRequests,bytes,countAttackRequests,attackBytes}`;`countries[]`:`{countryName,bytes,countRequests,percent,countAttackRequests,attackBytes}`(hourly/daily/nodes/domains/countries 直接序列化 pb 对象,值为 0 的字段按 omitempty 省略)。<br>`traffic`:`available,dayFound,requests,attacks,nodes,offlineNodes`。<br>`events[]`:`id,requestId,severity,action,sourceIP,host,path,method,status,ruleId,tags,errors,createdAt,firstSeen,lastSeen,count,requestIds`。<br>`apiDiscovery[]`(最多 100):`method,path,serverId,locationId,siteName,domain,requests,errors,errorRate,statuses,atRisk,protection,recommendation`。<br>`botSummary`:`available,humanLike,automated,suspicious,unknown,highVolumeIPs,method`。<br>`threatIPs[]`(最多 100):`ip,requests,blocked,region,source`。<br>`wafPolicies[]`:`id,name,description,isOn,mode,inbound,outbound`。<br>`eventHistory`:`status,days,events[{signature,firstSeen,lastSeen,count,severity,sourceIP,host,path,method,status,ruleId,action,errors,requestIds}],updatedAt,retentionDays` | 见模块约定 | 汇总指定日期的 WAF、流量、事件、API 发现、爬虫分类与威胁 IP(SecurityOverviewAction)。日志样本上限 1000 条。注意:GET 有副作用,会把事件写入本地安全事件历史,写入失败时 `eventHistory.status="write-failed"` | 登录 + Common |
| admin-01-auth-session | 安全事件通知设置 | GET | `/api/admin/security-events/notifications` | 无参数 | `data`:`status`(`available`/`unavailable`)、`enabled`(bool)、`minSeverity`(low/medium/high)、`minCount`、`cooldownMinutes`、`updatedAt`、`recipientScope`(取值含义待确认) | 见模块约定 | 读取安全事件通知配置(SecurityEventNotificationSettingsAction)。配置存于本地文件 `data/security-events/notifications.json`,非数据库 | 登录 + Common |
| admin-01-auth-session | 保存安全事件通知设置 | POST | `/api/admin/security-events/notifications` | `enabled`(bool,默认 false)、`minSeverity`(string,小写,取值 low/medium/high,为空默认 high)、`minCount`(int64,必填,范围 1 到 1000000000)、`cooldownMinutes`(int64,必填,范围 1 到 1440)、`csrfToken`(string,必填) | 成功:`data` 同 GET(`status,enabled,minSeverity,minCount,cooldownMinutes,updatedAt,recipientScope`)。<br>错误信息:"通知最低级别必须是 low、medium 或 high"、"通知触发次数无效"、"通知冷却时间必须在 1 到 1440 分钟之间"、"安全事件通知设置保存失败"。<br>注意:`minCount`、`cooldownMinutes` 缺省时按 0 处理,校验必然失败,必须显式传 | 见模块约定 | 保存安全事件通知配置(SecurityEventNotificationSettingsSaveAction) | 登录 + Setting |
| admin-01-auth-session | TLS 基线审计 | GET | `/api/admin/security/tls-baseline` | `page`(int,默认 1,小于 1 取 1)、`size`(int,默认 20,小于 1 取 20,大于 100 取 100)、`clusterId`(int64,可选,按集群过滤,0 为不过滤) | `data`:`items[]`:`{siteId,siteName,policyId,issues[{code,message}]}`(只返回存在问题的站点)、`auditedSites`(本页启用 HTTPS 的站点数)、`total`、`page`、`size`。 | 错误:RPC 失败走 `actionutils.Fail` | 审计站点 HTTPS 的 TLS 基线(TLSBaselineAction)。`issues[].code` 的取值枚举待确认 | 登录 + Server |

<a id="m2"></a>

## 2. 管理端 · 网站与网站分组(管理端)

**模块约定**

> 路由总数 66,来源 `EdgeAdmin/internal/web/actions/default/api/admin/init.go`(处理器:`sites.go`、`site_resource.go`、`site_mutations.go`、`site_settings_resource.go`、`site_settings_normalize.go`、`site_settings_load.go`、`site_web_settings_resource.go`、`site_delivery_settings_resource.go`、`site_content_settings_resource.go`、`site_request_settings_resource.go`、`site_security_settings_resource.go`、`site_runtime_settings_resource.go`、`site_rewrite_settings_resource.go`、`site_request_script_settings_resource.go`、`site_redirect_settings_resource.go`、`site_location_policy_resource.go`、`site_location_settings_resource.go`、`server_groups_resource.go`、`server_group_request_scripts_resource.go`)
说明:
- 路径前缀 `/api/admin`。处理器基于 TeaGo action,参数来自 URL query 与 POST 表单(`application/x-www-form-urlencoded` 或 multipart),**不解析 JSON body**。字段名按"原名/首字母小写名"匹配(`alias` 可改名),字符串自动去除首尾空白,`default` 在缺失或空串时生效,bool 接受 `on/true/yes/enabled`,数值解析失败按 0 处理,`[]string`/`[]int64` 用重复 key 传递。
- 响应包络。成功(`this.Success()`):`{"code":200,"message":"","data":{...}}`,本表"返回"一栏只写 `data` 内字段;"成功 data 为空"表示 `data` 无业务字段。失败(`this.Fail(msg)`):`{"code":400,"message":"...","data":...,"errors":null}`(`FailField` 时 `errors:[{param,messages}]`),`code` 默认 400;`failWithoutPanic` 未调用 `WriteHeader`,据此推断 HTTP 状态为 200(未实测,待确认)。失败(`actionutils.Fail`):gRPC 状态映射为 HTTP 状态(InvalidArgument 400 / Unauthenticated 401 / PermissionDenied 403 / NotFound 404 / AlreadyExists 409 / ResourceExhausted 429 / Unavailable 503 / DeadlineExceeded 504 / 其他 500),`message` 为脱敏后的服务端文案。表中"错误"列出的文案均为 `this.Fail` 的 message,除非另注。
- CSRF:本模块全部 POST 的 params 声明了 `CSRF`,须带参数 `csrfToken`(query 或表单字段)。令牌由 `GET /api/admin/session` 的 `data.csrfToken` 下发,绑定会话 sid,有效期 30 分钟。校验失败返回 HTTP 403 纯文本 `表单已失效，请刷新页面后重试(001)`,**不是 JSON 包络**。下表 POST 行不再逐条重复 `csrfToken`(均为必填 string)。
- 认证:本模块 66 条均为 `helpers.NewUserMustAuth(AdminModuleCodeServer)`,即需登录且具备 Server 模块权限。无会话 401 `{"code":401,"message":"authentication required","data":null}`;权限不足 403 `{"code":403,"message":"permission denied","data":null}`;系统未初始化 409。
- 分页(`page`/`size`):`page<1` 取 1,`size<1` 取 20,`size>100` 取 100。
- 作用域(`serverId`/`locationId`):除网站列表、创建和分组类接口外,`serverId` 为网站 ID(int64,>0 必填,否则 `invalid site id`/`invalid <resource> id`)。网站设置类接口的 `locationId` 为可选 int64:0(缺省)=网站级配置,>0=该网站下某条路由规则(Location)的配置;`locationId<0` 报 `invalid location id`;Location 不属于该网站报"路由规则不存在或不属于当前网站"。高级设置要求网站为 HTTP 家族,否则 `advanced website settings require an HTTP-family site`;网站不存在 `site not found`。
- `*JSON` 参数:参数值是一段 JSON 文本(放在表单字段里),须非空、≤256KiB,严格解析(禁止未知字段、只能有一个 JSON 对象),否则"请求内容为空或过大"/"请求只能包含一个 JSON 对象"。`isPrior`/`isOn` 为 bool。容量 `{count,unit}` 单位 byte/kb/mb/gb;时间 `{count,unit}` 单位见各行。
- 操作日志:变更类接口通过 `CreateLogInfo` 记录操作日志,不影响响应。
- 语言不统一:HTTPS 跳转、WebP、HLS、统计、分组请求脚本等的校验错误为英文前缀,其余多为中文;表中按源码原文给出。

| 模块 | 接口名称 | 方法 | 路径 | 请求参数 | 响应数据 | 主要错误 | 功能说明 | 认证/权限 |
|---|---|---|---|---|---|---|---|---|
| admin-02-sites | 网站列表 | GET | `/api/admin/sites` | `page` int64,默认 1<br>`size` int64,默认 20,上限 100<br>`keyword` string,可选<br>`clusterId` int64,可选<br>`groupId` int64,可选<br>`userId` int64,可选<br>`auditingFlag` int32,可选 | `items[]`:`id,name,isOn,typeName,cluster{id,name}`(可为 null)`,groups[{id,name}],user{id,name,username}`(可为 null)`,ports[{protocol,portRange}],firstDomain,countDomains,auditingState`(approved/pending/rejected)`,bandwidthBits,qps,attackQPS`<br>`total,page,size,countAuditing` | 错误:`actionutils.Fail` | 分页查询网站,支持关键字、集群、分组、用户、审核状态筛选。 | Server 模块 |
| admin-02-sites | 网站表单数据 | GET | `/api/admin/sites/form` | `serverId` int64,可选(缺省为新建表单) | `clusters[{id,name}]`(仅启用且非全局)<br>`groups[{id,name}]`(仅启用)<br>`users[{id,name,username,requirePlan}]`(≤1000,仅启用)<br>`site`:新建时 null,否则 `{id,name,description,isOn,type,clusterId,groupIds}` | 错误:`site not found` | 返回新建/编辑网站所需的下拉选项及(编辑时)网站基本信息。 | Server 模块 |
| admin-02-sites | 用户套餐选项 | GET | `/api/admin/sites/form/plans` | `userId` int64,可选;≤0 时返回空 | `plans[{id,name,planName,dayTo,clusterId}]`<br>`requirePlan` bool<br>`userId≤0` 返回 `plans:[]` | 错误:"所选用户不可用，请重新选择。" | 查询所选用户可用的套餐。 | Server 模块 |
| admin-02-sites | 网站详情 | GET | `/api/admin/sites/detail` | `serverId` int64,必填 | `site`:`id,name,description,type,typeName,isOn,createdTime`(`2006-01-02 15:04:05`)`,cluster,owner,groups,domains,cname,suportCNAME`(拼写按源码)`,protocols[{name,isOn,addresses[{host,portRange}]}],origins[{id,name,address,isOn,backup,weight}],featureStates[{code,name,state}]`(code:cache/waf/websocket/compression/accessLog/uam)`,bandwidthBits,qps,attackQPS` | 错误:`site not found` | 返回网站详情概览。`cluster/owner/groups/domains` 的元素字段待确认(未逐项核对)。 | Server 模块 |
| admin-02-sites | 网站设置读取 | GET | `/api/admin/sites/settings` | `serverId` int64,必填 | `site{id,name,type,cluster}`<br>`domains{items(ServerNameConfig 数组),isAuditing,auditingAt,auditingResult}`<br>`protocols{http{isOn,addresses[{protocol,host,portRange}]},https{isOn,addresses,sslPolicy{id,http2Enabled,http3Enabled,certificateCount}}}`<br>`reverseProxy`(见公共结构)<br>`web`:`compression,cache,websocket,accessLog`,另有 `waf,headers,requests,delivery,content,auth,fastcgi,locations,rewrites,scripts,redirects`(各子结构键以源码为准,部分待确认) | 错误:`invalid site id`/`site not found` | 一次性读取网站全部设置,供设置页渲染。认证策略读取时不回显密码/密钥(仅返回 username、`hasSecret`、`configured` 等标记);脚本 `code` 原样回显。 | Server 模块 |
| admin-02-sites | 路由规则策略读取 | GET | `/api/admin/sites/settings/locations/policies` | `serverId` int64,必填<br>`locationId` int64,必填(>0) | `site{id,name,type,cluster}`<br>`location{id,name,pattern,patternTypeName}`<br>`reverseProxy`<br>`web`(同上,但 `locations` 恒为 `[]`) | 错误:"路由规则不存在或不属于当前网站"(`this.Fail`),其余 `actionutils.Fail` | 读取某条路由规则自身的反向代理与 Web 策略设置。 | Server 模块 |
| admin-02-sites | 创建网站 | POST | `/api/admin/sites/create` | `userId` int64,可选<br>`userPlanId` int64,可选<br>`clusterId` int64,必填(由 RPC 校验)<br>`domains` []string,重复 key<br>`originAddrs` []string,重复 key<br>`enableWebsocket` bool,默认 false | 成功 `siteId` int64 | 错误:"用户与套餐选择无效，请重新选择。"(`userId<0` 或 `userPlanId<0` 或 `userId=0 且 userPlanId>0`);套餐校验类错误有专门中文文案 | 创建网站。疑似缺陷:操作日志 `defer` 在失败时 serverId 为 0。域名/源站的具体校验由 RPC 负责(待确认)。 | Server 模块 |
| admin-02-sites | 更新网站基础信息 | POST | `/api/admin/sites/update` | `serverId` int64,必填<br>`name` string<br>`description` string<br>`clusterId` int64<br>`groupIds` []int64,重复 key<br>`isOn` bool<br>`keepOldConfig` bool | 成功 data 为空 | 错误:`invalid site id`;名称/集群无本地校验,交 RPC | 更新网站名称、说明、所属集群、分组与启停。 | Server 模块 |
| admin-02-sites | 更新域名 | POST | `/api/admin/sites/settings/domains` | `serverId` int64,必填<br>`serverNamesJSON` string,必填:数组 `[{type,name,subNames[]}]`<br>`type`:`full/prefix/suffix/match`,空默认 `full`<br>`name` 转小写;`name` 与 `subNames` 不能同时为空;规则 ≤1000 条;重复项去重 | 成功 data 为空 | 错误:"域名配置格式无效"/"域名规则不能超过 1000 条"/"第 N 条域名规则无效/类型无效/不能为空"/"域名已被同一集群中的其他网站使用：…"/`site cluster not found` | 整体替换网站域名列表,并校验同集群内域名唯一。 | Server 模块 |
| admin-02-sites | 更新监听协议 | POST | `/api/admin/sites/settings/protocols` | `serverId` int64,必填<br>`httpIsOn` bool<br>`httpsIsOn` bool<br>`httpAddressesJSON` string:`[{protocol,host,portRange}]`<br>`httpsAddressesJSON` string,同上<br>端口 1–65535,范围 ≤128 个端口,每族 ≤16 条;HTTP 不含 443,HTTPS 不含 80 | 成功 data 为空 | 错误:"HTTP 和 HTTPS 至少需要启用一项"/"HTTP 监听地址无效：…"/"HTTPS 监听地址无效：…"/"监听地址格式无效" | 更新网站 HTTP/HTTPS 开关与监听地址;启用但未给地址时默认 80/443;检查地址冲突。 | Server 模块 |
| admin-02-sites | 更新反向代理 | POST | `/api/admin/sites/settings/reverse-proxy` | `serverId` int64,必填<br>`locationId` int64,默认 0<br>`isPrior,isOn` bool<br>`schedulingCode` string:`random/roundRobin/leastConn/hash/sticky`<br>`hashKey` string(hash 时必填,≤500,无换行)<br>`stickyType` string(sticky 时必填:`cookie/header/argument`)<br>`stickyParam` string(sticky 时必填,`^[A-Za-z0-9_-]{1,50}$`)<br>`requestHostType` int8:0=代理服务器 Host,1=源站 Host,2=自定义<br>`requestHost` string(type=2 必填)<br>`requestHostExcludingPort,stripPrefix,autoFlush,followRedirects,retry50X,retry40X` bool<br>`requestURI` string<br>`addHeaders` []string,白名单 `X-Real-IP/X-Forwarded-For/X-Forwarded-By/X-Forwarded-Host/X-Forwarded-Proto`<br>`connTimeoutJSON,readTimeoutJSON,idleTimeoutJSON` string:`{count,unit}`,unit `ms/second/minute/hour`,count≥0 且 ≤7 天<br>`maxConns,maxIdleConns` int,0–1000000<br>`originShieldIsOn,originShieldUseGlobal,originShieldOnlyCacheable` bool<br>`originShieldOriginId` int64<br>`originShieldMode` string:`direct/cloudflare`,空则沿用原值或 `direct`<br>`originShieldCloudflareHost,originShieldCloudflareProtocol` string | 成功 data 为空 | 错误:"回源调度算法无效"/"Hash 调度键值无效"/"Sticky 参数类型无效"/"Sticky 参数名称无效"/"回源 Host 类型无效"/"自定义回源 Host 无效"/"回源连接数限制无效"/"…格式无效"/"…数值无效"(超时)/"启用回源时至少需要一个主源站"/`site reverse proxy is not initialized`<br>L2:"L2 节点源站无效"/"L2 节点源站必须是启用的 HTTP/HTTPS 源站"/"L2 分层缓存尚未配置服务端安全令牌"/"L2 分层缓存配置无效：…" | 更新网站或路由规则的反向代理策略。回源分层(L2):`originShieldUseGlobal` 时写 `{isOn:true,useGlobal:true}`;`originShieldIsOn` 时须指定已启用的 HTTP/HTTPS 源站;两者都不开则写显式关闭(不继承全局)。 | Server 模块 |
| admin-02-sites | 新增源站 | POST | `/api/admin/sites/settings/origins/create` | `serverId` int64,必填<br>`locationId` int64,默认 0<br>`kind` string:`primary/backup`<br>`isOn` bool<br>`name` string,≤100<br>`description` string,≤200<br>`protocol` string(可编辑的 HTTP/HTTPS 族协议)<br>`host` string<br>`portRange` string<br>`weight` int,0–1000<br>`domains` []string<br>`requestHost` string<br>`followPort` bool<br>`http2Enabled` bool(仅 HTTPS 族协议生效)<br>`healthCheckJSON` string(间隔 1 秒–1 小时) | 成功 `originId` int64 | 错误:"源站类型无效"/"源站名称不能超过 100 个字符"/"源站备注不能超过 200 个字符"/"源站协议无效"/"源站地址无效"/"源站端口无效：…"/"源站权重需要在 0 到 1000 之间"/"单源站回源 Host 无效"/"健康检查设置过长"/"健康检查设置格式无效"/"健康检查间隔需要在 1 秒到 1 小时之间"/"同一类型的源站不能超过 32 个" | 在网站(或路由规则)的反向代理下新增主/备源站;引用写入失败时回滚(把新源站置为停用)。 | Server 模块 |
| admin-02-sites | 更新源站 | POST | `/api/admin/sites/settings/origins/update` | 同"新增源站",另加<br>`originId` int64,必填 | 成功 data 为空 | 错误:同"新增源站"校验,另有 `origin does not belong to this site`/"托管源站协议只能调整启用状态、名称和权重"/"启用回源时不能移动最后一个主源站" | 修改源站。托管源站(非 HTTP/HTTPS 族协议)只允许改启停、名称、权重。 | Server 模块 |
| admin-02-sites | 删除源站 | POST | `/api/admin/sites/settings/origins/delete` | `serverId` int64,必填<br>`locationId` int64,默认 0<br>`originId` int64,必填 | 成功 data 为空 | 错误:`origin does not belong to this site`/"启用回源时不能移除最后一个主源站" | 从反向代理移除源站引用,并把源站 `isOn` 置 false(软删除)。 | Server 模块 |
| admin-02-sites | 更新响应压缩 | POST | `/api/admin/sites/settings/compression` | `serverId` int64,必填<br>`locationId` int64,默认 0<br>`compressionJSON` string:`{isPrior,isOn,useDefaultTypes,types[],decompressData,minLength{count,unit},maxLength{count,unit},mimeTypes[],extensions[],exceptExtensions[],enablePartialContent}`<br>`types`:`brotli/gzip/zstd/deflate`<br>容量单位 `byte/kb/mb/gb`,count 0–1000000 | 成功 data 为空 | 错误:"响应压缩配置格式无效：…"/"启用自定义压缩算法时至少选择一种算法"/"最小内容长度不能大于最大内容长度"/"MIME 类型无效：…"/"压缩扩展名无效：…"/"例外扩展名无效：…"/"响应压缩配置无效：…"/"不支持的压缩算法：x" | 更新压缩设置。 | Server 模块 |
| admin-02-sites | 更新缓存 | POST | `/api/admin/sites/settings/cache` | `serverId` int64,必填<br>`locationId` int64,默认 0<br>`cacheJSON` string:`{isPrior,isOn,addStatusHeader,addAgeHeader,enableCacheControlMaxAge,useDefaultPolicy,purgeIsOn,rules[]}`<br>`rules[]`:`sourceIndex,isOn,isReverse,matchType`(`extension/all/index/prefix/exact/wildcard/regexp`)`,values[],isCaseInsensitive,life{count,unit(minute/hour/day/week),≤525600 分钟},ignoreQuery,status[](100–599),minSize,maxSize,skipCacheControlValues[],skipSetCookie,enableRequestCachePragma,allowChunkedEncoding,allowPartialContent,forcePartialContent,alwaysForwardRangeRequest,enableIfNoneMatch,enableIfModifiedSince,enableReadingOriginAsync`<br>规则 ≤50 条 | 成功 data 为空 | 错误:"缓存规则不能超过 50 条"/"缓存规则索引无效"/"不能修改高级缓存规则"/"缓存最小内容大小不能大于最大内容大小"/"缓存规则匹配值无效"/"扩展名缓存规则不能为空"/"缓存规则匹配类型无效"/"缓存有效期无效"/"缓存状态码无效：N" | 更新缓存规则。读取侧只以 `purgeKeyConfigured` 表示清除密钥是否已配置,不回显密钥。 | Server 模块 |
| admin-02-sites | 更新 WebSocket | POST | `/api/admin/sites/settings/websocket` | `serverId` int64,必填<br>`locationId` int64,默认 0<br>`websocketJSON` string:`{isPrior,isOn,handshakeTimeout{count,unit},allowAllOrigins,allowedOrigins[],requestSameOrigin,requestOrigin}`<br>握手超时 ≤24 小时;来源 ≤64 个,格式 `^[a-z0-9*._:-]+$` | 成功 data 为空 | 错误:握手超时单位无效/时间无效;来源格式无效(具体文案待确认) | 更新 WebSocket 设置。 | Server 模块 |
| admin-02-sites | 更新访问日志 | POST | `/api/admin/sites/settings/access-log` | `serverId` int64,必填<br>`locationId` int64,默认 0<br>`accessLogJSON` string:`{isPrior,isOn,fields[]int,status1,status2,status3,status4,status5,enableClientClosed,firewallOnly,storageOnly}`<br>可写 `fields` 码:1=Header,2=SentHeader,3=Arg,4=Cookie,5=Extend,6=Referer,7=UserAgent,8=RequestBody(9=ResponseBody 不允许)<br>`storagePolicyCount/advancedFieldCount` 为只读 | 成功 data 为空 | 错误:"不支持的访问日志字段：N" | 更新访问日志记录范围;非白名单字段码保留原值。 | Server 模块 |
| admin-02-sites | 更新 WAF | POST | `/api/admin/sites/settings/waf` | `serverId` int64,必填(先校验)<br>`locationId` int64,默认 0<br>`wafJSON` string(alias):`{isPrior,isOn,ignoreGlobalRules,defaultCaptcha(none/default),forceCaptcha,mode(defend/observe/bypass),maxRequestBodySize(0=默认,64KiB–16MiB),groups[{code,isOn}],browserChallenge{isOn,triggerMode(all/rate),minQPSPerIP,keyLife,challengeDificulty,challengeTimeout}}`<br>`minQPSPerIP` 1–100000(rate 时);`keyLife` 900–3600 秒;`challengeDificulty`(拼写按源码)14–22;`challengeTimeout` 60–180 秒 | 成功 data 为空 | 错误:"WAF 模式无效"/"默认人机验证方式无效"/"启用强制 CAPTCHA 前请先启用 WAF"/"WAF 请求内容检查上限必须在 64 KiB 至 16 MiB 之间"/"浏览器安全验证的触发方式无效"/"单 IP 触发 QPS 必须在 1 到 100000 之间"/"通行凭证有效期必须在 15 到 60 分钟之间"/"浏览器验证强度必须在 14 到 22 之间"/"浏览器验证超时必须在 60 到 180 秒之间"/"未知的 WAF 规则组：x"/"WAF 规则组重复：x"/"WAF 策略无效：…" | 更新 WAF 开关、模式与规则组。首次启用且无策略时自动创建名为"网站 WAF"的策略。 | Server 模块 |
| admin-02-sites | 更新 Header | POST | `/api/admin/sites/settings/headers` | `serverId` int64,必填(先校验)<br>`locationId` int64,默认 0<br>`headersJSON` string:`{request:{isPrior,isOn,setHeaders[{id,name,value}],deleteHeaders[]},response:{同 request,另有 addTrailers[],cors{isOn,allowMethods[],allowOrigin,allowCredentials,exposeHeaders[],maxAge(0–86400),requestHeaders[],optionsMethodOnly}}}`<br>各列表 ≤32 项;`name` ≤128 且为合法 token;`value` ≤2000 且无换行<br>禁止配置:connection/proxy-connection/keep-alive/transfer-encoding/content-length/upgrade/trailer/te/host<br>CORS 方法:GET/HEAD/POST/PUT/PATCH/DELETE/OPTIONS | 成功 data 为空<br>错误前缀"请求 Header：/响应 Header："<br>"请求 Header 策略不能配置响应 Trailer"/"Header x 重复"/"Header x 的值无效"/"Header N 不属于当前策略"/"启用 CORS 时需要填写允许来源"/"允许凭据时 CORS 来源不能为 *"/"CORS 方法无效：x"/"CORS 预检缓存时间无效" | 见模块约定 | 更新请求/响应 Header 策略与 CORS。 | Server 模块 |
| admin-02-sites | 更新请求限制 | POST | `/api/admin/sites/settings/request-limit` | `serverId` int64,必填<br>`locationId` int64,默认 0<br>`requestLimitJSON` string:`{isPrior,isOn,maxConns,maxConsPerIP,outBandwidthPerConn{count,unit},maxBodySize{count,unit}}`<br>`maxConns/maxConsPerIP` 0–1000000(`maxConsPerIP` 拼写按源码);`outBandwidthPerConn` ≤1TiB;`maxBodySize` ≤64GiB;单位 `byte/kb/mb/gb`,空默认 byte | 成功 data 为空 | 错误:"最大并发连接数必须在 0 至 1000000 之间"/"单 IP 并发连接数…"/"单连接下行带宽单位无效/超出允许范围"/"请求体上限…"(后三者完整文案待确认) | 更新并发连接、带宽与请求体上限。不直接调用 `validateMutationID`,ID 由 `loadSiteScopedHTTPWeb` 校验。 | Server 模块 |
| admin-02-sites | 更新访客 IP | POST | `/api/admin/sites/settings/client-ip` | `serverId` int64,必填<br>`locationId` int64,默认 0<br>`clientIPJSON` string:`{isPrior,isOn,type,requestHeaderName,value,trustedProxyRanges[]}`<br>`type`:`default/proxy/requestHeader/variable`;非 default 必须配置 `trustedProxyRanges`;Header 名最多 8 个,每个 ≤64;`value`(变量)1–256 字符无换行 | 成功 data 为空 | 错误:"访客 IP 获取方式无效"/"自定义访客 IP 变量必须为 1 至 256 个无换行字符"/"可信代理范围无效：…"/"从代理、Header 或变量读取访客 IP 时必须配置可信代理范围"/"请输入访客 IP Header 名称"/"访客 IP Header 最多支持 8 个" | 更新客户端真实 IP 的获取方式。 | Server 模块 |
| admin-02-sites | 更新防盗链 | POST | `/api/admin/sites/settings/referer` | `serverId` int64,必填<br>`locationId` int64,默认 0<br>`refererJSON` string:`{isPrior,isOn,allowEmpty,allowSameDomain,allowDomains[],denyDomains[],checkOrigin}`<br>域名规则:去空白,单条 ≤512,`~` 开头为正则,否则小写并校验;去重;最多 128 条 | 成功 data 为空 | 错误:"允许来源域名无效：…"/"禁止来源域名无效：…"/"防盗链配置无效：…"<br>域名规则底层错误为英文:"domain rule is too long"/"invalid domain regular expression"/"too many domain rules" | 更新 Referer 防盗链。 | Server 模块 |
| admin-02-sites | 更新 User-Agent 过滤 | POST | `/api/admin/sites/settings/user-agent` | `serverId` int64,必填<br>`locationId` int64,默认 0<br>`userAgentJSON` string:`{isPrior,isOn,filters[{action(allow/deny),keywords[]}]}`<br>`filters` ≤32 组;关键词总数 ≤512;单个 ≤256 且无换行;每组至少 1 个 | 成功 data 为空 | 错误:"第 N 组 User-Agent 动作无效/关键词过长或包含换行/规则至少需要一个关键词"/"User-Agent 配置无效：…" | 更新 User-Agent 允许/禁止规则。 | Server 模块 |
| admin-02-sites | 更新字符集 | POST | `/api/admin/sites/settings/charset` | `serverId` int64,必填<br>`locationId` int64,默认 0<br>`charsetJSON` string:`{isPrior,isOn,charset,isUpper,force}`<br>`charset` 空默认 `utf-8`,≤64,`^[A-Za-z0-9._:-]+$` | 成功 data 为空 | 错误:"字符集名称无效" | 更新响应字符集。 | Server 模块 |
| admin-02-sites | 更新 HTTPS 跳转 | POST | `/api/admin/sites/settings/https-redirect` | `serverId` int64,必填<br>`locationId` int64,默认 0<br>`redirectToHTPSJSON` string(参数名拼写 HTPS,按源码):`{isPrior,isOn,status,host,port,onlyDomains[],exceptDomains[]}`<br>`status`:0/301/302/303/307/308;`port` 0–65535;`host` 不得含端口或通配 | 成功 data 为空<br>错误(英文):"HTTPS redirect configuration is invalid: …" | 见模块约定 | 更新 HTTP 自动跳转 HTTPS。不直接调用 `validateMutationID`。 | Server 模块 |
| admin-02-sites | 更新 WebP | POST | `/api/admin/sites/settings/webp` | `serverId` int64,必填<br>`locationId` int64,默认 0<br>`webpJSON` string(alias):`{isPrior,isOn,minLength,maxLength,mimeTypes[],extensions[]}`<br>容量对象 `{count,unit}` ≤512MiB;`mimeTypes`/`extensions` 各 ≤32;扩展名 `^\.[a-z0-9][a-z0-9._+-]{0,31}$` | 成功 data 为空<br>错误(英文):"WebP configuration is invalid: …" | 见模块约定 | 更新 WebP 转换设置。 | Server 模块 |
| admin-02-sites | 更新 HLS | POST | `/api/admin/sites/settings/hls` | `serverId` int64,必填<br>`locationId` int64,默认 0<br>`hlsJSON` string:`{isPrior,isOn}`(`advancedURLPaternCount` 只读) | 成功 data 为空<br>错误(英文):"HLS configuration is invalid: …" | 见模块约定 | 更新 HLS 设置。 | Server 模块 |
| admin-02-sites | 更新统计 | POST | `/api/admin/sites/settings/stat` | `serverId` int64,必填<br>`locationId` int64,默认 0<br>`statJSON` string:`{isPrior,isOn}` | 成功 data 为空<br>错误(英文):"statistics configuration is invalid: …" | 见模块约定 | 更新 Web 统计开关。 | Server 模块 |
| admin-02-sites | 更新静态根目录 | POST | `/api/admin/sites/settings/root` | `serverId` int64,必填<br>`locationId` int64,默认 0<br>`rootJSON` string:`{isPrior,isOn,dir,indexes[],stripPrefix,decodePath,isBreak,exceptHiddenFiles}`<br>`dir` ≤2048;`indexes` ≤64;`stripPrefix` 须以 `/` 开头(`dir` 的字符限制待确认) | 成功 data 为空<br>错误前缀:"静态资源根目录配置无效：…" | 见模块约定 | 更新静态资源根目录。 | Server 模块 |
| admin-02-sites | 更新全局页面开关 | POST | `/api/admin/sites/settings/pages/global` | `serverId` int64,必填<br>`locationId` int64,默认 0<br>`enableGlobalPages` bool | 成功 data 为空 | 见模块约定 | 开关是否启用全局自定义页面。 | Server 模块 |
| admin-02-sites | 新增自定义页面 | POST | `/api/admin/sites/settings/pages/create` | `serverId` int64,必填<br>`locationId` int64,默认 0<br>`pageJSON` string:`{id,isOn,status[],newStatus,bodyType(html/url/redirectURL),url,body}`<br>`status` 元素匹配 `^[1-5][0-9x]{2}$`,每页 ≤16 个;`url` ≤512;html `body` ≤32KiB;每作用域 ≤64 页 | 成功 data 为空(**不返回新页面 id**)<br>错误前缀:"自定义页面配置无效：…" | 见模块约定 | 新增自定义状态码页面。 | Server 模块 |
| admin-02-sites | 更新自定义页面 | POST | `/api/admin/sites/settings/pages/update` | 同"新增自定义页面",另加<br>`pageId` int64,必填(≤0 报错) | 成功 data 为空(**不返回新 id**) | 错误:同上 | 修改页面。实现为新建页面再替换引用,**页面 id 会变**。 | Server 模块 |
| admin-02-sites | 删除自定义页面 | POST | `/api/admin/sites/settings/pages/delete` | `serverId` int64,必填<br>`locationId` int64,默认 0<br>`pageId` int64,必填 | 成功 data 为空 | 见模块约定 | 从作用域移除自定义页面引用。 | Server 模块 |
| admin-02-sites | 更新临时关闭页面 | POST | `/api/admin/sites/settings/shutdown` | `serverId` int64,必填<br>`locationId` int64,默认 0<br>`shutdownJSON` string:`{isPrior,isOn,bodyType,url,body,status}`<br>`status`:0 或 100–599 | 成功 data 为空<br>错误前缀:"临时关闭页面配置无效：…" | 见模块约定 | 更新网站临时关闭(维护)页面。 | Server 模块 |
| admin-02-sites | 更新认证引用 | POST | `/api/admin/sites/settings/auth` | `serverId` int64,必填<br>`locationId` int64,默认 0<br>`authJSON` string:`{isPrior,isOn,policies[{id,isOn}]}`<br>策略 ≤32;不可重复;不可遗漏当前已存在的策略 | 成功 data 为空 | 错误:"认证策略 ID 无效"/"认证策略不能重复"/"认证配置包含不属于当前作用域的策略"/前缀"认证配置无效：…" | 仅调整策略引用与启停,不修改策略内容。 | Server 模块 |
| admin-02-sites | 新增认证策略 | POST | `/api/admin/sites/settings/auth/policies/create` | `serverId` int64,必填<br>`locationId` int64,默认 0<br>`policyJSON` string:<br>通用:`isOn,name`(1–128,无控制字符)`,type`(`basicAuth/subRequest/signURL`)`,exts[]`(归一化为小写带点,≤64,不含 `/` `\`)`,domains[]`<br>basicAuth:`users[{username(≤128,不含 :),password(≤512)}]`(1–64 个)`,realm,charset`<br>subRequest:`url`(≤2048)`,method`(默认 GET)`,headers[{name,value}],timeout{count,unit}`(默认 10 秒)`,followRedirects`<br>signURL:`secret,secondarySecret,clearSecondarySecret,algorithm,signParam,timestampParam,life,keepParams,mode,bindClientIP,bindQuery`(signURL 各字段取值待确认)<br>每作用域 ≤32 个 | 成功 data 为空(**不返回新策略 id**) | 错误:"认证策略无效：…"/"认证策略名称无效"/"不支持的认证类型"/"BasicAuth 用户数必须为 1 到 64"/"BasicAuth 用户名无效/不能重复"/`BasicAuth 用户 "x" 必须设置密码`/"子请求认证 URL 无效"/`子请求认证 Header "x" 重复`/"子请求认证配置无效：…"/"URL 签名配置无效：…"/"每个作用域最多支持 32 个认证策略" | 新增认证策略并引用。请求体含密码/密钥,请勿写日志。 | Server 模块 |
| admin-02-sites | 更新认证策略 | POST | `/api/admin/sites/settings/auth/policies/update` | 同"新增认证策略",另加<br>`policyId` int64,必填(≤0 报"认证策略 ID 无效") | 成功 data 为空(**不返回新 id**) | 错误:同上,另有"认证策略不属于当前作用域" | 修改策略。实现为新建策略并替换引用,**策略 id 会变**。敏感值留空表示保留原值:basicAuth 空密码保留旧密码;subRequest 敏感 Header(authorization/proxy-authorization/cookie/set-cookie 及名称含 api-key/apikey/api-token/token/secret/password)空值保留;signURL 的 `secret`、`secondarySecret` 空保留(`clearSecondarySecret` 可清除备用密钥)。 | Server 模块 |
| admin-02-sites | 删除认证策略 | POST | `/api/admin/sites/settings/auth/policies/delete` | `serverId` int64,必填<br>`locationId` int64,默认 0<br>`policyId` int64,必填 | 成功 data 为空 | 错误:"认证策略 ID 无效"/"认证策略不属于当前作用域" | 移除认证策略。 | Server 模块 |
| admin-02-sites | 更新 FastCGI 引用 | POST | `/api/admin/sites/settings/fastcgi` | `serverId` int64,必填<br>`locationId` int64,默认 0<br>`fastcgiJSON` string:`{isPrior,isOn,configs[{id}]}`,`configs` ≤32 | 成功 data 为空 | 错误:"FastCGI ID 无效"/"FastCGI 不属于当前作用域"/"FastCGI 引用不能重复" | 仅调整 FastCGI 引用与启停。 | Server 模块 |
| admin-02-sites | 新增 FastCGI | POST | `/api/admin/sites/settings/fastcgi/create` | `serverId` int64,必填<br>`locationId` int64,默认 0<br>`fastcgiJSON` string:`{isOn,address,params[{name,value}],readTimeout,connTimeout,poolSize,pathInfoPattern}`<br>`poolSize` 0–1000000;每作用域 ≤32(`readTimeout/connTimeout` 结构待确认) | 成功 data 为空(**不返回新 id**) | 错误:"FastCGI 配置无效：…"/"FastCGI connection pool size is out of range" | 新增 FastCGI 配置并引用。 | Server 模块 |
| admin-02-sites | 更新 FastCGI 配置 | POST | `/api/admin/sites/settings/fastcgi/update` | 同"新增 FastCGI",另加<br>`fastcgiId` int64,必填 | 成功 data 为空(**不返回新 id**) | 错误:同上 | 修改配置。实现为新建并替换引用,**id 会变**。 | Server 模块 |
| admin-02-sites | 删除 FastCGI | POST | `/api/admin/sites/settings/fastcgi/delete` | `serverId` int64,必填<br>`locationId` int64,默认 0<br>`fastcgiId` int64,必填 | 成功 data 为空 | 错误:"FastCGI ID 无效"/"FastCGI 不属于当前作用域" | 移除 FastCGI 配置。 | Server 模块 |
| admin-02-sites | 新增路由规则 | POST | `/api/admin/sites/settings/locations/create` | `serverId` int64,必填(无 `locationId`)<br>`locationJSON` string:`{parentId,isOn,name,description,pattern,patternType,isCaseInsensitive,isReverse,isBreak,domains[]}`<br>`name` ≤100;`description` ≤1000;`pattern` 1–2048 无换行<br>`patternType` int:1=前缀,2=精确,3=正则,4=后缀<br>前缀/精确:无空白 `?` `#`,自动补前导 `/`;后缀:无空白 `?` `#` `/`;正则须可编译<br>站点内 ≤256 条,嵌套 <8 层 | 成功 `locationId` int64 | 错误:"路由规则格式无效：…"/"路由名称不能超过 100 个字符"/"路由说明不能超过 1000 个字符"/"路由匹配规则必须为 1 至 2048 个无换行字符"/"前缀和精确路径不能包含空白、查询参数或片段"/"后缀规则不能包含空白、斜杠、查询参数或片段"/"路由正则表达式无效：…"/"路由匹配类型无效"/"路由域名范围无效：…"/"每个网站最多支持 256 条路由规则"/"父级路由无效"/"路由规则最多支持 8 层嵌套"/"同一父级下已经存在相同的路由匹配规则"/"父级路由不存在或已被修改" | 新增路由规则(Location)。 | Server 模块 |
| admin-02-sites | 更新路由规则 | POST | `/api/admin/sites/settings/locations/update` | `serverId` int64,必填<br>`locationJSON` string:同上,`id` 必填(否则 `invalid location id`) | 成功 data 为空 | 错误:同上,另有"编辑时不能更改父级路由"/"路由规则不存在或不属于当前网站" | 修改路由规则,不可更换父级。 | Server 模块 |
| admin-02-sites | 删除路由规则 | POST | `/api/admin/sites/settings/locations/delete` | `serverId` int64,必填<br>`locationId` int64,必填 | 成功 data 为空 | 错误:"请先删除当前规则的子规则"/"路由规则不存在或不属于当前网站" | 删除路由规则;有子规则时拒绝。 | Server 模块 |
| admin-02-sites | 路由规则排序 | POST | `/api/admin/sites/settings/locations/sort` | `serverId` int64,必填<br>`parentId` int64,默认 0(顶层)<br>`locationIdsJSON` string:`[]int64`,须包含该层全部路由 | 成功 data 为空 | 错误:"排序必须包含当前网站的全部路由规则"/"排序中存在重复路由"/"排序包含不属于当前网站的路由"/"父级路由不存在"/"路由排序格式无效：…" | 调整同一父级下路由规则的顺序。 | Server 模块 |
| admin-02-sites | 新增重写规则 | POST | `/api/admin/sites/settings/rewrites/create` | `serverId` int64,必填<br>`locationId` int64,默认 0<br>`rewriteJSON` string:`{isOn,pattern,replace,mode,redirectStatus,proxyHost,withQuery,isBreak}`<br>`pattern`(正则)1–2048;`replace` 1–4096;`mode`:`redirect/proxy`;`redirectStatus`:301/302/303/307/308;当前作用域 ≤128 条 | 成功 `rewriteRuleId` int64 | 错误:"匹配正则必须为 1 至 2048 个无换行字符"/"匹配正则无效：…"/"目标地址必须为 1 至 4096 个无换行字符"/"跳转状态码无效"/"重写模式无效"/"代理 Host 必须是有效的主机名、IP 或主机端口"/"当前配置最多支持 128 条重写规则" | 新增 URL 重写规则。 | Server 模块 |
| admin-02-sites | 更新重写规则 | POST | `/api/admin/sites/settings/rewrites/update` | 同"新增重写规则",`rewriteJSON` 内另含<br>`id` 必填(否则 `invalid rewrite rule id`)<br>`sourceKey` string(乐观并发键,取自读取结果) | 成功 data 为空 | 错误:同上,另有"重写规则已被其他操作修改，请刷新后重试" | 修改重写规则;`sourceKey` 不符则拒绝。 | Server 模块 |
| admin-02-sites | 删除重写规则 | POST | `/api/admin/sites/settings/rewrites/delete` | `serverId` int64,必填<br>`locationId` int64,默认 0<br>`rewriteRuleId` int64,必填<br>`sourceKey` string,必填 | 成功 data 为空 | 错误:"重写规则已被其他操作修改，请刷新后重试" | 删除重写规则。 | Server 模块 |
| admin-02-sites | 重写规则排序 | POST | `/api/admin/sites/settings/rewrites/sort` | `serverId` int64,必填<br>`locationId` int64,默认 0<br>`rewriteRuleIdsJSON` string:`[]int64`,须包含全部规则 | 成功 data 为空 | 错误:排序不完整/重复/不属于当前作用域(具体文案待确认) | 调整重写规则顺序。 | Server 模块 |
| admin-02-sites | 更新请求脚本 | POST | `/api/admin/sites/settings/request-scripts` | `serverId` int64,必填<br>`locationId` int64,默认 0<br>`scriptsJSON` string:`{sourceKey,initGroup{isOn,scripts[{sourceIndex,sourceKey,isOn,code}]},requestGroup{同上}}`<br>`sourceIndex=-1` 表示新增;每阶段脚本 ≤8 个 | 成功 data 为空 | 错误:"请求脚本已被其他操作修改，请刷新后重试"/"第 N 个脚本来源无效/已被删除，请刷新后重试/已被其他操作修改"/"脚本列表包含重复来源"/"请求初始化脚本：…"/"发送请求前脚本：…"/"当前网站分组已接管请求脚本，请在分组配置中修改"(仅网站级)/`site web configuration not found` | 更新请求初始化与发送前脚本。脚本 `code` 在读取时原样回显。 | Server 模块 |
| admin-02-sites | 更新跳转规则 | POST | `/api/admin/sites/settings/redirects` | `serverId` int64,必填(**无 `locationId`**)<br>`redirectsJSON` string:数组,≤128 条;元素:`sourceIndex(-1=新增),sourceKey,isOn,type(url/domain/port),status(301/302/303/307/308),matchMode(equal/prefix/regexp),beforeURL,afterURL,keepRequestURI,keepArgs,domainsAll,domainsBefore[],domainBeforeIgnorePorts,domainAfter,domainAfterScheme(空/http/https),portsAll,portsBefore[](≤32),portAfter(1–65535),portAfterScheme,onlyDomains[],exceptDomains[]` | 成功 data 为空 | 错误:"跳转规则格式无效：…"/"第 N 条跳转规则：…"/"每个网站最多支持 128 条跳转规则"/"跳转状态码无效"/"跳转类型无效"/"URL 匹配方式无效"/"请输入跳转前域名"/"请输入跳转前端口"/"跳转后端口必须在 1 至 65535 之间"/"第 N 条跳转规则已被其他操作修改，请刷新页面"/"跳转协议仅支持保持原协议、HTTP 或 HTTPS"/"URL 必须为 1 至 2048 个无换行字符"/"请输入完整的 HTP 或 HTTPS URL"/"正则目标必须以 http:// 或 https:// 开头" | 整体更新网站级跳转规则(域名/端口/URL 三类)。 | Server 模块 |
| admin-02-sites | 启停网站 | POST | `/api/admin/sites/toggle` | `serverId` int64,必填<br>`isOn` bool | 成功 data 为空 | 错误:`invalid site id` | 开启或关闭网站。 | Server 模块 |
| admin-02-sites | 删除网站 | POST | `/api/admin/sites/delete` | `serverId` int64,必填 | 成功 data 为空 | 错误:`invalid site id`;其余 `actionutils.Fail` | 调用 RPC `DeleteServers` 删除网站。是否软删除、是否可恢复由 RPC 决定(待确认),前端应二次确认。 | Server 模块 |
| admin-02-sites | 分组列表 | GET | `/api/admin/server-groups` | 无 | `items[]`:`id,name,isOn,siteCount,policies[]`<br>`policies` 为中文标签数组,可能值:HTTP 回源/TCP 回源/UDP 回源/WAF/缓存/访问日志/压缩/Header/请求限制/请求脚本/页面/WebP/HLS | 见模块约定 | 返回全部网站分组及其已配置的策略标签。 | Server 模块 |
| admin-02-sites | 分组详情 | GET | `/api/admin/server-groups/detail` | `groupId` int64,必填<br>`page` int64,默认 1<br>`size` int64,默认 20,上限 100<br>`keyword` string,可选 | `group{id,name,isOn,policies[]}`<br>`sites[]`:`id,name,isOn,type,firstDomain,cluster{id,name}`(可为 null)`,owner{id,name,username}`(可为 null)<br>`total,page,size` | 错误:`invalid server group id`/`server group not found` | 分页返回分组内的网站。 | Server 模块 |
| admin-02-sites | 分组请求脚本读取 | GET | `/api/admin/server-groups/request-scripts` | `groupId` int64,必填 | `group{id,name}`<br>`scripts`:`sourceKey,initGroup{isOn,scripts[{sourceIndex,sourceKey,isOn,code,hasAuditingCode}]},requestGroup{同上}`(不含 `groupOverride`) | 错误:`invalid server group id`/`server group not found`/`server group web configuration not found` | 读取分组级请求脚本配置。 | Server 模块 |
| admin-02-sites | 分组可添加网站 | GET | `/api/admin/server-groups/site-options` | `groupId` int64,必填<br>`keyword` string,可选 | `items[]`:`id,name,firstDomain`<br>最多取前 100 个站点,并排除已在该分组的站点 | 错误:`invalid server group id` | 搜索可加入该分组的网站。 | Server 模块 |
| admin-02-sites | 创建分组 | POST | `/api/admin/server-groups/create` | `name` string,必填(本地无校验,交 RPC) | 成功 `groupId` int64 | 错误:`server group service returned an invalid id` | 创建网站分组。 | Server 模块 |
| admin-02-sites | 更新分组 | POST | `/api/admin/server-groups/update` | `groupId` int64,必填<br>`name` string(本地无校验) | 成功 data 为空 | 错误:`invalid server group id` | 修改分组名称。 | Server 模块 |
| admin-02-sites | 删除分组 | POST | `/api/admin/server-groups/delete` | `groupId` int64,必填 | 成功 data 为空 | 错误:`invalid server group id` | 删除分组。分组内网站的处理由 RPC 决定(待确认)。 | Server 模块 |
| admin-02-sites | 分组排序 | POST | `/api/admin/server-groups/sort` | `groupIds` []int64,重复 key,按目标顺序(无本地校验) | 成功 data 为空 | 见模块约定 | 调整分组显示顺序。 | Server 模块 |
| admin-02-sites | 分组添加网站 | POST | `/api/admin/server-groups/sites/add` | `groupId` int64,必填<br>`serverId` int64,必填 | 成功 data 为空(已在分组内则幂等成功) | 错误:`invalid site id`/`site not found`/`invalid server group id` | 把网站加入分组。 | Server 模块 |
| admin-02-sites | 分组移除网站 | POST | `/api/admin/server-groups/sites/remove` | `groupId` int64,必填<br>`serverId` int64,必填 | 成功 data 为空 | 错误:同"分组添加网站" | 把网站移出分组。 | Server 模块 |
| admin-02-sites | 更新分组请求脚本 | POST | `/api/admin/server-groups/request-scripts` | `groupId` int64,必填<br>`scriptsJSON` string:结构同"更新请求脚本" | 成功 data 为空<br>错误(英文前缀):"invalid request script configuration: …";其余沿用请求脚本合并的中文错误 | 见模块约定 | 更新分组级请求脚本。 | Server 模块 |


### 公共结构

- `SizeCapacity`:`{"count":int,"unit":"byte|kb|mb|gb"}`;`TimeDuration`:`{"count":int,"unit":"ms|second|minute|hour"}`(缓存有效期另支持 `day/week`,各接口上限见对应行)。
- `reverseProxy`(`SiteSettings` / `SiteLocationPolicies` 返回):字段与"更新反向代理"入参同名(含 `schedulingCode`、`requestHostType`、超时、`originShield*` 等),另含主/备源站列表。精确键名见 `site_settings_resource.go`,此处未逐项核对,待确认。
- 对象 id 稳定性:`CreateSiteOrigin`、`CreateSiteLocation`、`CreateSiteRewrite`、`ServerGroupCreate`、`SiteCreate` 返回新 id;`CreateSitePage`、`CreateSiteAuthPolicy`、`CreateSiteFastcgi` 不返回;`UpdateSitePage`、`UpdateSiteAuthPolicy`、`UpdateSiteFastcgiConfig` 会生成新对象并替换引用,调用后须重新读取设置获取新 id。

<a id="m3"></a>

## 3. 管理端 · 节点与集群(管理端)

**模块约定**

> 路由总数 53,来源 EdgeAdmin/internal/web/actions/default/api/admin/init.go
说明:
- 路径前缀 `/api/admin`(`server.Prefix("/api/admin")`)。处理器均在同目录,TeaGo Action。
- 认证:全部经 `helpers.NewUserMustAuth(<模块>)`。`/api/` 路径下未登录返回 HTTP 401 `{"code":401,"message":"authentication required","data":null}`;无模块权限返回 403 `{"code":403,"message":"permission denied","data":null}`;系统未初始化返回 409;WAF/IP 拦截返回 403(文本或空 body)。本模块涉及的权限模块:`Setting`(用户节点、数据库节点)、`Server`(`/nodes/*` 详情与操作)、`Node`(概览、监控设置、集群、`/nodes/form`),下表"认证"列已逐行标注。
- 请求参数:TeaGo 从 URL 查询串合并 POST 表单(urlencoded 或 multipart)取值,**不解析 JSON 请求体**。参数名为结构体字段名首字母小写,`alias` 标注的按别名(如 `ip`、`ipAddressId`、`dnsResolverJSON`)。缺省参数为零值(整数 0、字符串空、布尔 false);字符串与数值取值前会去首尾空白。布尔值 `on/true/yes/enabled` 为 true,其余按常规布尔解析,所以**漏传布尔参数等同于 false**。数组参数用重复键(`certificateIds=1&certificateIds=2`)。`[]byte` 类型取第一个值。本模块没有 `default:"x"` 标签,下文"默认"均指处理器代码里的默认。
- 成功响应:HTTP 200,`Content-Type: application/json; charset=utf-8`,`{"code":200,"message":"","data":{...}}`。下表"返回"列只写 `data` 内的字段;写操作无业务字段时 `data` 为空对象(待确认 `Data` 初始值序列化形态)。
- 失败响应有两种格式,同模块混用:
  - **F 类**(`this.Fail(msg)`):`{"code":400,"message":"<原始英文或中文消息>","data":...,"errors":null}`,代码未设置 HTTP 状态,默认 HTTP 200,业务码在 body 的 `code`(是否有外层中间件改写状态:待确认)。"未找到"类错误(如 `node not found`)在 F 类中也是 code 400,不是 404。
  - **A 类**(`actionutils.Fail(this, err)`):同时设置 HTTP 状态与 `code`:InvalidArgument 400、Unauthenticated 401、PermissionDenied 403、NotFound 404、AlreadyExists 409、ResourceExhausted 429、Unavailable 503、DeadlineExceeded 504,其他 500;Unknown 错误按消息关键字映射(not found 404、already exists 409、invalid/required 400 等)。`message` 形如 `{AdminCommon_ServerError}（<净化后的原因>）`。净化规则:含 password=/secret/token 等敏感字样的错误统一替换为"服务处理失败，请检查相关配置后重试。";权限类替换为"没有权限执行此操作，请检查当前账号权限。";不可用替换为"服务暂时不可用，请稍后重试。";未找到替换为"请求的资源不存在或已被删除。";已存在替换为"资源已存在或已被占用，请更换后重试。"。本地 API 节点异常时消息末尾附"；最近一次错误提示:…"。下表中"A:"表示该错误走 A 类并给出其内部原因,"F:"表示 F 类原文。
- CSRF:所有 POST 都声明了 `CSRF *actionutils.CSRF` 字段,所有 GET 都没有。令牌作为表单或查询参数 `csrfToken` 提交(不看请求头),与会话 sid 绑定,有效期 1800 秒,服务端最多缓存 50000 个令牌。取令牌:`GET /api/admin/session` 的 `data.csrfToken`,或 `default/csrf/token.go` 的 `data.token`。校验失败返回 HTTP 403,**纯文本**(非 JSON)`表单已失效，请刷新页面后重试(001)`。下表 POST 行的"认证"列均含"+CSRF"。
- 分页(`normalizePagination`,`page`/`size` 列表):`page<1` 取 1;`size<1` 取 20;`size>100` 取 100。响应 `items/total/page/size`。日志类接口用 `offset/size` 分页,规则见"公共结构 NodeLogsResult"。
- 操作日志:大部分写操作通过 `CreateLogInfo` 记录,例外在各行说明。
- 敏感信息:数据库节点密码、源站保护密钥只写不读,响应中只给 `hasPassword`/`...Configured` 之类布尔值。例外:`GET /user-nodes/install` 的 `configYAML` 含用户节点凭据,见该行。

| 模块 | 接口名称 | 方法 | 路径 | 请求参数 | 响应数据 | 主要错误 | 功能说明 | 认证/权限 |
|---|---|---|---|---|---|---|---|---|
| admin-03a-nodes-clusters | 用户节点列表 | GET | `/api/admin/user-nodes` | 查询参数:<br>`page` int64,可选,默认 1<br>`size` int64,可选,默认 20,最大 100 | `{"items":[{"id":int64,"name":string,"description":string,"isOn":bool,"accessAddrs":[string],"status":UserNodeStatus}],"page":int,"size":int,"total":int64}`(`UserNodeStatus` 见公共结构) | 错误:A:RPC 失败;F:"user node service returned an empty response" | 分页列出已启用的用户节点,附带心跳状态。`total` 为全部已启用用户节点数。 | 需登录+Setting 权限 |
| admin-03a-nodes-clusters | 用户节点表单 | GET | `/api/admin/user-nodes/form` | 查询参数:`nodeId` int64,可选;>0 时返回该节点,≤0 时只返回证书选项 | `{"certificates":[{"id":int64,"name":string,"dnsNames":[string],"expiresTime":string,"isACME":bool}],"node?":UserNodeView}`<br>`certificates` 为可用的非 CA 证书,最多取前 2000 条,`expiresTime` 由 `formatUnixDate` 格式化(具体格式待确认)。`UserNodeView` 见公共结构。 | 错误:F:"invalid user node id"/"user node not found";A:RPC 失败 | 编辑用户节点前加载表单所需数据。仅当 `nodeId>0` 才有 `node` 字段。 | 需登录+Setting 权限 |
| admin-03a-nodes-clusters | 用户节点详情 | GET | `/api/admin/user-nodes/detail` | 查询参数:`nodeId` int64,必填,须 >0 | `{"node":UserNodeView}` | 错误:F:"invalid user node id"/"user node not found";A:RPC 失败 | 读取单个用户节点配置与状态。与表单接口的区别:不返回证书选项列表。 | 需登录+Setting 权限 |
| admin-03a-nodes-clusters | 用户节点安装配置 | GET | `/api/admin/user-nodes/install` | 查询参数:`nodeId` int64,必填,须 >0 | `{"node":{"id":int64,"name":string},"configYAML":string,"version":string}`,响应头 `Cache-Control: no-store` | 错误:F:"invalid user node id"/"user node not found";A:RPC 失败,或 "user node credentials are incomplete"(缺 UniqueId/Secret)等 | 用已启用 API 节点的访问地址生成用户节点的安装配置 YAML。**`configYAML` 含该节点的 UniqueId 与 Secret 凭据**,调用方不得缓存或记录。YAML 内部字段:待确认(未逐项核对)。`version` 来自 `productconfigs.Version`。 | 需登录+Setting 权限 |
| admin-03a-nodes-clusters | 用户节点日志 | GET | `/api/admin/user-nodes/logs` | 查询参数:`nodeId` int64,必填,>0<br>`offset` int64,默认 0,≥0<br>`size` int64,默认 20(传 0),范围 1-100<br>`dayFrom`/`dayTo` string,可选,`YYYY-MM-DD`<br>`keyword` string,可选,0-200 字符<br>`level` string,可选,`error/warning/info/success`<br>`tag` string,可选,`^[A-Z0-9_:-]+$`,最长 64 | `NodeLogsResult` + `tags`(常用标签列表,元素结构待确认) | 错误:F:节点不存在;A:参数校验失败(如 "invalid log offset"、"log page size must be between 1 and 100"、"invalid node log date")或 RPC 失败 | 查询用户节点运行日志。校验失败走 A 类,消息经净化后含原因。 | 需登录+Setting 权限 |
| admin-03a-nodes-clusters | 创建用户节点 | POST | `/api/admin/user-nodes/create` | 表单参数:<br>`name` string,必填,1-100 字符,无控制字符<br>`description` string,可选,0-500 字符<br>`httpListens` string,换行分隔的 `host:port`<br>`httpsListens` string,同上<br>`accessAddrs` string,换行分隔的公共访问地址,可写 `host:port` 或 `http(s)://host[:port]`<br>`certificateIds` []int64,可选,去重后最多 32 个,须 >0<br>`isOn` bool,默认 false<br>`csrfToken` string 必填 | `{"nodeId":int64}`<br>错误(均为 F):"at least one HTTP or HTTPS listener is required"、"at least one public access URL is required"、"HTTPS listeners require at least one certificate"、"certificates require an HTTPS listener"、"invalid listener address/host/port …"、"duplicate listener address …"、"invalid public access URL/host/port …"、"too many user node addresses"、"user node address is too long"、"user node service returned an invalid id";A:RPC 失败 | 见模块约定 | 创建用户节点。规则:每条地址最长 512 字符且不含 `\x00`,总数不超过 16 条(常量 `maximumAdminUserNodeAddresses`,取值待在源码二次确认);host 可为 IP 或主机名,`*`/空表示全部;端口 1-65535;访问地址未写协议时按监听端口推断(HTTP 与 HTTPS 都含该端口则报 "ambiguous public access protocol";443 默认 HTTPS)。有 HTTPS 监听时自动创建/复用 SSL 策略。记录操作日志。 | 需登录+Setting 权限+CSRF |
| admin-03a-nodes-clusters | 更新用户节点 | POST | `/api/admin/user-nodes/update` | 同"创建用户节点",另加:<br>`nodeId` int64,必填,>0 | 成功无业务字段 | 错误:F:"invalid user node id"/"user node not found" 及创建接口的全部校验消息;A:RPC 失败 | 整体覆盖用户节点配置。`isOn` 是普通布尔,漏传即置为关闭;`certificateIds` 漏传在有 HTTPS 监听时触发校验失败。记录操作日志。 | 需登录+Setting 权限+CSRF |
| admin-03a-nodes-clusters | 启停用户节点 | POST | `/api/admin/user-nodes/toggle` | 表单参数:`nodeId` int64 必填,>0<br>`isOn` bool,必填(漏传视为 false 即关闭)<br>`csrfToken` string 必填 | 成功无业务字段 | 错误:F:"invalid user node id"/"user node not found";A:RPC 失败 | 读取原配置后仅改 `isOn` 再调用更新 RPC。疑似缺陷:操作日志复用"更新用户节点"文案,无单独的启停日志(待确认意图)。 | 需登录+Setting 权限+CSRF |
| admin-03a-nodes-clusters | 删除用户节点 | POST | `/api/admin/user-nodes/delete` | 表单参数:`nodeId` int64 必填,>0<br>`csrfToken` string 必填 | 成功无业务字段 | 错误:F:"invalid user node id"/"user node not found";A:RPC 失败 | 删除用户节点,记录操作日志。 | 需登录+Setting 权限+CSRF |
| admin-03a-nodes-clusters | 数据库节点列表 | GET | `/api/admin/database-nodes` | 查询参数:<br>`page` int64,默认 1<br>`size` int64,默认 20,最大 100<br>`keyword` string,可选,按名称/主机/库名不区分大小写子串匹配 | `{"items":[DatabaseNode],"total":int64,"page":int,"size":int}`(`DatabaseNode` 见公共结构) | 错误:A:RPC 失败 | 分页列出数据库节点。疑似缺陷:`keyword` 在 RPC 分页之后只过滤当前页,`total` 仍是未过滤的总数,翻页与关键字组合会漏数据。 | 需登录+Setting 权限 |
| admin-03a-nodes-clusters | 数据库节点表单 | GET | `/api/admin/database-nodes/form` | 查询参数:`nodeId` int64,可选 | `{"node":DatabaseNode}`;`nodeId≤0` 时返回默认值 `{"id":0,"isOn":true,"dbEngine":"mysql","port":3306,"hasPassword":false}`(仅这些键) | 错误:A:"invalid database node id"/"database node not found"(映射为 400/404) | 加载数据库节点编辑表单,密码不返回。 | 需登录+Setting 权限 |
| admin-03a-nodes-clusters | 数据库节点日志 | GET | `/api/admin/database-nodes/logs` | 同"用户节点日志"全部查询参数 | `NodeLogsResult`(**不含 `tags`**) | 错误:A:节点不存在(404)、日志参数校验失败、RPC 失败 | 查询数据库节点日志。与用户节点日志的差异:节点不存在走 A 类,且响应无 `tags`。 | 需登录+Setting 权限 |
| admin-03a-nodes-clusters | 保存数据库节点 | POST | `/api/admin/database-nodes/save` | 表单参数:<br>`nodeId` int64,可选;>0 为更新,否则创建<br>`name` string,必填,≤100 字节<br>`dbEngine` string,默认 `mysql`,≤32 字节且不含空白<br>`description` string,可选,≤500<br>`isOn` bool,默认 false<br>`host` string,必填,≤253<br>`port` int32,必填,1-65535<br>`database` string,必填,≤128<br>`username` string,必填,≤128<br>`password` string,≤4096 字节且不含换行;更新时留空表示保持原密码<br>`csrfToken` string 必填 | 创建时 `{"nodeId":int64}`,更新时无业务字段 | 错误:F:"name is required"/"invalid database name"(host、database、username、description 同理)、"database port must be between 1 and 65535"、"invalid database engine"、"invalid database password";A:更新时节点不存在、RPC 失败 | 创建或更新数据库节点,同一接口按 `nodeId` 分支。字符串字段不得含 `\r \n \x00`。记录操作日志。长度按字节计,不是字符数。 | 需登录+Setting 权限+CSRF |
| admin-03a-nodes-clusters | 启停数据库节点 | POST | `/api/admin/database-nodes/toggle` | 表单参数:`nodeId` int64 必填,>0<br>`isOn` bool(漏传即关闭)<br>`csrfToken` string 必填 | 成功无业务字段 | 错误:A:"invalid database node id"/"database node not found"、RPC 失败 | 读取原配置后仅改 `isOn`。未写操作日志(与保存/删除不同,疑似遗漏)。 | 需登录+Setting 权限+CSRF |
| admin-03a-nodes-clusters | 删除数据库节点 | POST | `/api/admin/database-nodes/delete` | 表单参数:`nodeId` int64 必填,>0<br>`csrfToken` string 必填 | 成功无业务字段 | 错误:A:同上 | 删除数据库节点,记录操作日志。 | 需登录+Setting 权限+CSRF |
| admin-03a-nodes-clusters | 检测数据库节点状态 | POST | `/api/admin/database-nodes/status` | 表单参数:`nodeId` int64 必填,>0<br>`csrfToken` string 必填 | `{"status":{"isOk":bool,"error":string,"size":int64,"version":string}}`(`size`、`version` 的类型与单位:待确认;RPC 未返回状态对象时 `status` 缺省的行为待确认) | 错误:A:同上 | 对数据库节点发起一次连通性与状态检测(RPC `CheckDBNodeStatus`)。用 POST 但不改动数据,不写日志。 | 需登录+Setting 权限+CSRF |
| admin-03a-nodes-clusters | 边缘节点详情 | GET | `/api/admin/nodes/detail` | 查询参数:`nodeId` int64,必填,>0<br>`clusterId` int64,可选;>0 时校验节点属于该集群(主或从集群均可) | `{"node":NodeDetail,"clusters":[{"id","name","isOn"}],"groups":[{"id","name"}],"regions":[{"id","name"}],"grants":[{"id","name","method","username"}],"ssh":{"id":int64,"grantId":int64,"host":string,"port":int,"configured":bool},"dnsDomains":[{"id","name","clusterId","clusterName","routes":[{"name","code","selected"}]}],"advanced":{…},"actions":[{"id","isOn","condsJSON","actionJSON","durationJSON"}],"actionParams":[…],"actionDefinitions":[…],"actionOperators":[…],"thresholdItems":[…],"thresholdActions":[…]}`<br>`NodeDetail`、`advanced` 见公共结构。`actionParams/actionDefinitions/actionOperators/thresholdItems/thresholdActions` 来自 `nodeconfigs.FindAll…` 定义列表,元素字段:待确认。`regions` 仅含已启用地区,`groups` 为主集群下的分组。 | 错误:F:"invalid node id"/"node not found"/"node does not belong to the selected cluster"/"node IP service returned an empty response"/"node grant service returned an empty response"/"node SSH configuration is invalid";A:任一 RPC 失败 | 加载节点编辑页所需的全部数据,串行调用十余个 RPC,其中每个 IP 地址各查一次阈值(N+1)。`ssh.host` 为空时回填第一个 IP。`advanced.dnsFailoverJSON` 无数据时为 `"[]"`,`apiNodeAddrsJSON` 无配置时为 `""`。检测新版本失败会被忽略,不影响响应。 | 需登录+Server 权限 |
| admin-03a-nodes-clusters | 节点缓存诊断 | POST | `/api/admin/nodes/cache-diagnostics` | 表单参数:`nodeId` int64 必填,>0<br>`clusterId` int64,可选<br>`csrfToken` string 必填 | `{"result":{"isOk":bool,"message":string,"diagnostics":any}}`<br>无响应时 `{"isOk":false,"message":"未收到节点响应","diagnostics":null}`;诊断数据解析失败时 `isOk=false`、`message="节点诊断数据格式无效：…"` | 错误:F:节点校验错误;A:RPC 失败 | 向节点下发缓存诊断消息并等待最多 15 秒。`diagnostics` 是节点回传的任意 JSON,结构待确认。 | 需登录+Server 权限+CSRF |
| admin-03a-nodes-clusters | 节点回源健康 | POST | `/api/admin/nodes/origin-health` | 同"节点缓存诊断" | `{"result":{"isOk":bool,"message":string,"health":any}}`<br>无响应时 `message="未收到节点响应"`;解析失败 `message="节点健康数据格式无效：…"` | 见模块约定 | 向节点下发读取回源健康的消息,超时 15 秒,逻辑与缓存诊断完全一致,仅消息类型与返回键(`health`)不同。`health` 结构待确认。 | 需登录+Server 权限+CSRF |
| admin-03a-nodes-clusters | 创建节点动作 | POST | `/api/admin/nodes/actions/create` | 表单参数:`nodeId` int64 必填,>0<br>`clusterId` int64,可选<br>`condsJSON` string,默认 `{}`<br>`actionJSON` string,默认 `{}`<br>`durationJSON` string,默认 `{"count":0,"unit":"minute"}`<br>三个 JSON 各自 ≤256 KB 且须是合法 JSON<br>`csrfToken` string 必填 | `{"id":int64}` | 错误:F:节点校验错误及 JSON 校验消息(具体文案待确认,出自 `validateNodeActionJSON`);A:RPC 失败 | 为节点新增一条条件动作(阈值触发的自动动作)。 | 需登录+Server 权限+CSRF |
| admin-03a-nodes-clusters | 更新节点动作 | POST | `/api/admin/nodes/actions/update` | 表单参数:`nodeId` int64 必填<br>`clusterId` int64,可选<br>`actionId` int64 必填,>0 且属于该节点<br>`condsJSON`/`actionJSON`/`durationJSON` string,留空时沿用原值<br>`isOn` bool,漏传即置为 false<br>`csrfToken` string 必填 | 成功无业务字段 | 错误:F:"node action not found"及 JSON 校验消息;A:RPC 失败 | 更新节点动作。陷阱:三个 JSON 留空会合并为已有值,但 `isOn` 没有这种保护,忘记传会把动作关掉。 | 需登录+Server 权限+CSRF |
| admin-03a-nodes-clusters | 删除节点动作 | POST | `/api/admin/nodes/actions/delete` | 表单参数:`nodeId`、`clusterId`(可选)、`actionId` int64 必填<br>`csrfToken` string 必填 | 成功无业务字段 | 错误:F:"node action not found";A:RPC 失败 | 删除节点上的一条动作,先校验动作归属该节点。 | 需登录+Server 权限+CSRF |
| admin-03a-nodes-clusters | 排序节点动作 | POST | `/api/admin/nodes/actions/sort` | 表单参数:`nodeId`、`clusterId`(可选)<br>`actionIds` []int64,必填,非空、无重复、每项 >0<br>`csrfToken` string 必填 | 成功无业务字段 | 错误:F:"action order cannot be empty"/"invalid action order";A:RPC 失败 | 按 `actionIds` 顺序重排节点动作。是否要求列出节点全部动作:待确认。 | 需登录+Server 权限+CSRF |
| admin-03a-nodes-clusters | 复制节点动作 | POST | `/api/admin/nodes/actions/copy` | 表单参数:`nodeId`、`clusterId`(可选)<br>`target` string,必填,`group` 或 `cluster`(不区分大小写)<br>`csrfToken` string 必填 | 成功无业务字段 | 错误:F:"copy target must be group or cluster";A:RPC 失败 | 把该节点的动作复制到所在节点分组(`group`)或集群(`cluster`)内的其他节点。 | 需登录+Server 权限+CSRF |
| admin-03a-nodes-clusters | 重置节点动作状态 | POST | `/api/admin/nodes/actions/reset` | 表单参数:`nodeId` 必填、`clusterId` 可选<br>`csrfToken` string 必填 | 成功无业务字段 | 错误:F:节点校验错误;A:RPC 失败 | 重置节点动作的触发状态(RPC `ResetNodeActionStatus`)。 | 需登录+Server 权限+CSRF |
| admin-03a-nodes-clusters | 更新节点 IP 阈值 | POST | `/api/admin/nodes/ip/thresholds` | 表单参数:`nodeId` 必填、`clusterId` 可选<br>`ipAddressId` int64 必填<br>`thresholdsJSON` string,留空视为 `[]`,必须是 JSON 数组<br>`csrfToken` string 必填 | 成功无业务字段 | 错误:F:"thresholds JSON is invalid"/"thresholds JSON must be an array"/"IP address does not belong to node"及节点校验错误;A:RPC 失败 | 整体替换某个节点 IP 的阈值规则。注意:此处字段 `ipAddressId` 无别名问题,但与 `ip/*` 系列接口同名同义。 | 需登录+Server 权限+CSRF |
| admin-03a-nodes-clusters | 节点日志 | GET | `/api/admin/nodes/logs` | 查询参数:`nodeId` 必填,>0<br>`clusterId` 可选<br>`offset`、`size`、`dayFrom`、`dayTo`、`keyword`、`level`、`tag` 同"用户节点日志" | `NodeLogsResult` + `tags` | 错误:F:节点校验错误;A:日志参数校验失败、RPC 失败 | 查询边缘节点日志。 | 需登录+Server 权限 |
| admin-03a-nodes-clusters | 启停节点 | POST | `/api/admin/nodes/is-on` | 表单参数:`nodeId` 必填、`clusterId` 可选<br>`isOn` bool(漏传即关闭)<br>`csrfToken` string 必填 | 成功无业务字段 | 错误:F:节点校验错误;A:RPC 失败 | 启用或停用节点。开启与关闭使用不同的操作日志文案。 | 需登录+Server 权限+CSRF |
| admin-03a-nodes-clusters | 远程安装节点 | POST | `/api/admin/nodes/install` | 表单参数:`nodeId` 必填、`clusterId` 可选<br>`csrfToken` string 必填 | 成功无业务字段 | 错误:F:"请先为节点配置 SSH 授权"(节点无 SSH 登录配置);A:RPC 失败 | 触发通过 SSH 的远程安装(RPC `InstallNode`),只发起任务,不等待安装完成,进度见节点详情的 `installStatus`。 | 需登录+Server 权限+CSRF |
| admin-03a-nodes-clusters | 更新节点 SSH 登录 | POST | `/api/admin/nodes/login` | 表单参数:`nodeId` 必填、`clusterId` 可选<br>`grantId` int64,必填,须存在<br>`sshHost` string,必填,IP 或主机名<br>`sshPort` int,必填,1-65535<br>`csrfToken` string 必填 | 成功无业务字段 | 错误:F:"invalid SSH host"/"invalid SSH port"及授权不存在消息(来自 `findAdminNodeGrant`,文案待确认);A:RPC 失败 | 保存节点 SSH 登录:授权、主机、端口。登录名固定为 `SSH`,类型 `ssh`。记录操作日志。 | 需登录+Server 权限+CSRF |
| admin-03a-nodes-clusters | 更新节点拓扑 | POST | `/api/admin/nodes/topology` | 表单参数:`nodeId` 必填、`clusterId` 可选(当前所在集群)<br>`primaryClusterId` int64,必填,须是已启用集群<br>`secondaryNodeClusterIds` []int64,可选,须为已启用集群,与主集群重复的项会被剔除<br>`groupId` int64,可选,>0 时须属于主集群<br>`regionId` int64,可选,>0 时须存在且已启用<br>`level` int32,只能 1(L1)或 2(L2)<br>`lnAddrs` []string,`level=2` 时必填,1-64 个<br>`enableIPLists` bool<br>`csrfToken` string 必填 | 成功无业务字段 | 错误:F:"请选择主集群"、"主集群不存在或已停用"、"从集群不存在或已停用"、"节点级别只能是 L1 或 L2"、"L2 节点至少需要一个访问地址"、"node region does not exist or is disabled"及分组校验消息;A:RPC 失败 | 修改节点所属集群、分组、地区、级别与 L2 地址。保留节点名与启用状态。分组/地区传 0 即清空。 | 需登录+Server 权限+CSRF |
| admin-03a-nodes-clusters | 更新节点 DNS 线路 | POST | `/api/admin/nodes/dns` | 表单参数:`nodeId` 必填、`clusterId` 可选<br>`routes` []string,可选(元素格式待确认,与 DNS 修复接口的 `code@domainId` 不同,此处原样透传)<br>`csrfToken` string 必填 | 成功无业务字段 | 错误:F:节点校验错误;A:RPC 失败 | 更新节点的 DNS 线路。日志在函数返回时写入(`defer`),RPC 失败也会记录。 | 需登录+Server 权限+CSRF |
| admin-03a-nodes-clusters | 新增节点 IP | POST | `/api/admin/nodes/ip/create` | 表单参数:`nodeId` 必填、`clusterId` 可选<br>`name` string,可选<br>`ip` string,必填,合法 IP<br>`canAccess` bool<br>`isUp` bool<br>`csrfToken` string 必填 | 成功无业务字段(**不返回新 IP 的 id**) | 错误:F:"invalid IP address";A:RPC 失败 | 为节点新增一个 IP 地址,角色固定为边缘节点。 | 需登录+Server 权限+CSRF |
| admin-03a-nodes-clusters | 更新节点 IP | POST | `/api/admin/nodes/ip/update` | 表单参数:`nodeId` 必填、`clusterId` 可选<br>`ipAddressId` int64,必填,>0<br>`name`、`ip`(必填合法 IP)<br>`canAccess`、`isOn`、`isUp` bool(漏传即 false)<br>`csrfToken` string 必填 | 成功无业务字段 | 错误:F:"invalid IP address"(ID≤0 或 IP 非法);A:RPC 失败 | 修改 IP 地址。是否校验该 IP 属于此节点:待确认,代码只校验节点存在。 | 需登录+Server 权限+CSRF |
| admin-03a-nodes-clusters | 停用节点 IP | POST | `/api/admin/nodes/ip/delete` | 表单参数:`nodeId` 必填、`clusterId` 可选<br>`ipAddressId` int64,必填,>0<br>`csrfToken` string 必填 | 成功无业务字段 | 错误:F:"invalid IP address id";A:RPC 失败 | 逻辑删除(RPC `DisableNodeIPAddress`)。同样未校验该 IP 属于此节点(待确认)。 | 需登录+Server 权限+CSRF |
| admin-03a-nodes-clusters | 更新节点高级设置 | POST | `/api/admin/nodes/advanced` | 表单参数:`nodeId` 必填、`clusterId` 可选<br>`name` string,必填<br>`maxCPU` int32,≥0<br>`bypassMobile` int32(注意是整数,不是布尔,取值含义待确认)<br>`maxCacheDiskCount` int64、`maxCacheDiskUnit` string、`maxCacheMemoryCount` int64、`maxCacheMemoryUnit` string<br>`cacheDiskDir` string<br>`cacheDiskSubDirsJSON` string<br>`dnsResolverJSON` string,非空才更新<br>`ddosProtectionJSON` string,非空才更新<br>`offlineDay` string,`-` 与 `/` 会被剔除(如 `2026-12-31` 变 `20261231`)<br>`isBackupForCluster`、`isBackupForGroup` bool<br>`backupIPs` []string<br>`dnsFailoverJSON` string<br>`apiNodeAddrsConfigured` bool<br>`apiNodeAddrsJSON` string<br>`csrfToken` string 必填 | 成功无业务字段 | 错误:F:"node name is required"、"CPU limit cannot be negative"、"API node addresses JSON is invalid"、"too many API node addresses"、"API node address is incomplete"、"API node address is invalid: …"、"API node address port is invalid"及节点校验错误;A:RPC 失败 | 保存节点高级设置。`apiNodeAddrsConfigured=true` 时才处理 `apiNodeAddrsJSON`:必须是 `NetworkAddressConfig` 数组(拒绝未知字段),最多 64 个,每个需有 host 与 portRange;传 `[]` 表示恢复平台默认。风险:依次调用多个 RPC(基础信息、系统、移动绕过、缓存、DNS 解析、DDoS、API 地址、调度信息),标量在第一次写入前已校验,但中途 RPC 失败会留下部分已生效的修改,不是事务。缓存容量单位与 `…SubDirsJSON` 的校验在处理器内没有,依赖后端 RPC(待确认)。 | 需登录+Server 权限+CSRF |
| admin-03a-nodes-clusters | 远程启动节点 | POST | `/api/admin/nodes/start` | 表单参数:`nodeId` 必填、`clusterId` 可选<br>`csrfToken` string 必填 | 成功无业务字段 | 错误:F:"node start failed"(或 RPC 返回的失败原因);A:RPC 失败 | 通过 SSH 远程启动节点进程,同步等待结果。 | 需登录+Server 权限+CSRF |
| admin-03a-nodes-clusters | 远程停止节点 | POST | `/api/admin/nodes/stop` | 同"远程启动节点" | 成功无业务字段 | 错误:F:"node stop failed"(或 RPC 失败原因);A:RPC 失败 | 通过 SSH 远程停止节点进程。 | 需登录+Server 权限+CSRF |
| admin-03a-nodes-clusters | 删除节点 | POST | `/api/admin/nodes/delete` | 表单参数:`nodeId` 必填、`clusterId` 可选<br>`uninstall` bool,可选;为 true 且节点已安装时先远程卸载<br>`csrfToken` string 必填 | 成功无业务字段 | 错误:F:卸载失败时 "node uninstall failed"(或 RPC 返回原因);A:RPC 失败 | 删除节点;`uninstall=true` 时卸载失败会中止删除。日志在返回时写入(`defer`)。 | 需登录+Server 权限+CSRF |
| admin-03a-nodes-clusters | 节点概览 | GET | `/api/admin/nodes/overview` | 无参数 | `{"serverTime":int64,"nodes":[NodeOverviewItem],"clusters":[{"id","name"}],"regions":[{"id","name"}],"refreshSeconds":15,"todayTraffic":{"available":bool,"day":string,"bytes":int64,"requests":int64},"accounting":{…}}`<br>`NodeOverviewItem` 见公共结构。`accounting` 为 `trafficstats.Describe` 的输出,字段:待确认。 | 错误:A:RPC 失败 | 全部已启用节点的概览与心跳状态,分批(每批 500)拉取全部节点,无上限,节点很多时响应较大。前端建议按 `refreshSeconds` 轮询。 | 需登录+Node 权限 |
| admin-03a-nodes-clusters | 节点监控设置 | GET | `/api/admin/nodes/monitor-settings` | 无参数 | `{"origin":{"minSamples":int,"failureRateWarning":float,"failureRateCritical":float,"headerP99WarningMS":int64},"defaults":{同 origin},"dnsOverload":{"isOn":bool,"threshold":float},"dnsOverloadDefaults":{同 dnsOverload},"source":"default"或"setting"}`<br>默认值:`minSamples=50`、`failureRateWarning=0.05`、`failureRateCritical=0.20`、`headerP99WarningMS=2000`、`dnsOverload={isOn:false,threshold:0.95}` | 错误:A:RPC 失败 | 读取回源质量告警阈值与 DNS 过载摘除设置。无设置、JSON 无法解析或校验不通过时都按默认值返回且 `source="default"`。 | 需登录+Node 权限 |
| admin-03a-nodes-clusters | 更新回源监控阈值 | POST | `/api/admin/nodes/monitor-settings/update` | 表单参数:`minSamples` int,范围 1-1000000<br>`failureRateWarning` float,(0,1]<br>`failureRateCritical` float,(0,1],不小于 warning<br>`headerP99WarningMS` int64(别名 `headerP99WarningMS`),100-600000<br>`reset` bool,true 时忽略前四项并恢复默认<br>`csrfToken` string 必填 | 成功无业务字段<br>错误(F):"回源最少样本数应在 1 到 1000000 之间"、"回源失败率阈值应在 0 到 1 之间（不含 0）"、"回源失败率警告阈值不能大于严重阈值"、"回源 P99 延迟阈值应在 100 到 600000 毫秒之间";A:RPC 失败 | 见模块约定 | 先读后改,只替换回源阈值,保留同一份设置里的 DNS 过载摘除。同一份配置的 `dnsOverload` 也会被一并校验,若库中已有非法值会让本次保存失败(待确认,属于代码推断)。 | 需登录+Node 权限+CSRF |
| admin-03a-nodes-clusters | 更新 DNS 过载摘除 | POST | `/api/admin/nodes/monitor-settings/dns-overload` | 表单参数:`isOn` bool<br>`threshold` float,范围 0.5-1;≤0 时取默认 0.95<br>`reset` bool,true 时恢复默认(关闭、0.95)<br>`csrfToken` string 必填 | 成功无业务字段 | 错误:F:"DNS 过载摘除阈值应在 50% 到 100% 之间"或回源阈值校验消息;A:RPC 失败 | 开关过载节点临时摘出 DNS 解析。源码注释:CPU 或内存连续 3 次(约 3 分钟)不低于阈值被摘除,连续 3 次低于阈值减 0.10 后恢复;每个集群最多摘一半且至少保留一个,全部过载时不摘。漏传 `isOn` 等于关闭。 | 需登录+Node 权限+CSRF |
| admin-03a-nodes-clusters | 集群列表 | GET | `/api/admin/clusters` | 查询参数:`page`、`size`(同分页规则)<br>`keyword` string,可选 | `{"items":[{"id":int64,"name":string,"isOn":bool,"isPinned":bool,"installDir":string,"countNodes":int64,"countActiveNodes":int64,"countUpgradeNodes":int64,"countServers":int64,"dnsName":string,"dnsDomainName":string,"timeZone":string}],"total":int64,"page":int,"size":int,"totalNodes":int64,"globalCluster?":{"id":int64,"name":string,"isGlobal":true}}` | 错误:A:RPC 失败 | 分页列出集群,按 id 倒序。`timeZone` 等于默认时区时返回空字符串。`globalCluster` 仅在存在全局集群时出现。 | 需登录+Node 权限 |
| admin-03a-nodes-clusters | 集群表单 | GET | `/api/admin/clusters/form` | 查询参数:`clusterId` int64,可选 | `{"timeZones":[{"name":string,"offset":string}],"threadLimits":{"min":int,"max":int},"cluster":ClusterForm或null}`<br>`ClusterForm` 见公共结构;`clusterId≤0` 时 `cluster` 为 null<br>`timeZones` 只含规范时区、默认时区与 UTC。`offset` 类型:待确认 | 错误:F:"cluster not found";A:RPC 失败 | 加载集群创建/编辑表单。 | 需登录+Node 权限 |
| admin-03a-nodes-clusters | 集群详情 | GET | `/api/admin/clusters/detail` | 查询参数:`clusterId` int64,必填,>0 | `{"cluster":{"id","name","isOn","isGlobal"(待确认),"installDir","timeZone"(待确认),"dns":{"domainName","nodesAutoSync","serversAutoSync","ttl"},"countNodes","countActiveNodes","countUpgradeNodes","countSites","cacheLayers":{"l1":CacheLayer,"l2":CacheLayer,"cloudflare":{"enabled","mode","modeName","host","protocol","providerId","configuration"}},"nodes":[ClusterNode],"nodesTruncated":bool}}`<br>`CacheLayer`:`nodeCount,activeNodeCount,usedBytes,capacityBytes,memoryUsedBytes,memoryCapacityBytes,l2EndpointCount,l2DegradedEndpointCount`<br>`ClusterNode`:`id,name,isOn,isUp,isInstalled,level,active,heartbeatActive,synced,version,latestVersion,group,region,installState{running,finished,ok,error},status{…}` | 错误:F/A:参数无效或集群不存在(消息文案待确认) | 集群详情与节点列表。`nodes` 最多返回前 100 个,超出时 `nodesTruncated=true`。`cluster` 上除上述键外是否还有其他键:待确认(只核对了主干键)。 | 需登录+Node 权限 |
| admin-03a-nodes-clusters | 集群策略 | GET | `/api/admin/clusters/policies` | 查询参数:`clusterId` int64,必填,>0 | `{"cluster":{"id","name","isGlobal"},"sync":{"state":string,"total":int,"synced":int,"pending":int,"offline":int,"checkedAt":int64},"health":Health,"http3":{"isOn","port","supportMobileBrowsers"},"systemService":{"systemdIsOn","useGlobal"},"webp":{"isOn","quality","requireCache","minLengthMB","maxLengthMB","useGlobal"},"uam":{"isOn","useGlobal"},"networkSecurity":{…},"cc":{…},"ddos":{…},"serverBaseline":{…},"assignments":{…}}`<br>各子对象见公共结构"ClusterPolicies"。 | 错误:F:"invalid cluster id"、"cluster not found";A:RPC 失败 | 一次返回集群的全部策略。`useGlobal=true` 表示该项继承全局集群配置,写入该项策略时不会落到本集群(规则见 `policyutils.Resolve`,细节待确认)。`sync.state` 取值:`unknown`(读取节点失败)、`empty`(无节点)、`syncing`(有活跃但未同步节点)、`partial`(有离线节点)、`synced`。 | 需登录+Node 权限 |
| admin-03a-nodes-clusters | 集群运维设置 | GET | `/api/admin/clusters/operations` | 查询参数:`clusterId` int64,必填,>0 | `{"cluster":{"id","name","isGlobal"},"thresholds":{"useGlobal":bool,"items":[{"id","itemName","param","paramName","operator","operatorName","value","message","sumMethod","duration","durationUnit","notifyDuration","isOn"}],"definitions":[…],"operators":[…]},"notifications":{"useGlobal":bool,"selected":[{"type","id"}],"recipients":[{"id","name","instanceName"}],"groups":[{"id","name"}]},"firewall":{"useGlobal":bool,"items":[{"id","name","eventLevel","eventLevelName","type","typeName","params","hasSecret"}],"types":[…],"levels":[…]}}`<br>防火墙动作的 `params.secret` 已被移除并以 `hasSecret` 布尔值代替。`definitions/operators/types/levels` 元素字段:待确认。 | 错误:参数无效或集群不存在(文案待确认) | 集群的监控阈值、通知接收人、防火墙联动动作。 | 需登录+Node 权限 |
| admin-03a-nodes-clusters | 创建节点表单 | GET | `/api/admin/nodes/form` | 查询参数:`clusterId` int64,必填,>0 | `{"cluster":{"id","name"},"groups":[{"id","name"}],"regions":[{"id","name"}],"grants":[{"id","name","method","username"}],"dnsDomainId":int64,"dnsRoutes":[{"name","code"}]}` | 错误:F:"invalid node cluster id"、"node cluster not found";A:RPC 失败 | 加载"在某集群下新增节点"所需的选项。同目录有 `NodeCreateAction`(POST `/nodes/create`,`init.go:356`)不在本模块路由清单中,此处不做说明。 | 需登录+Node 权限 |
| admin-03a-nodes-clusters | 创建集群 | POST | `/api/admin/clusters/create` | 表单参数:`name` string<br>`installDir` string<br>`autoInstallNftables`、`autoSystemTuning`、`autoTrimDisks` bool<br>`maxConcurrentReads`、`maxConcurrentWrites` int32<br>`csrfToken` string 必填 | `{"clusterId":int64}` | 错误:A:RPC 失败(包括名称为空等业务校验) | 创建集群,默认带全局服务配置与 systemd 服务。处理器自身不校验任何字段,必填性与取值范围全由 RPC 层决定(具体规则待确认)。注意:没有 `timeZone`、`nodeMaxThreads`,需创建后再更新。日志在返回时写入,失败时 `clusterId` 为 0。 | 需登录+Node 权限+CSRF |
| admin-03a-nodes-clusters | 更新集群 | POST | `/api/admin/clusters/update` | 表单参数:`clusterId` int64,必填,>0<br>`name`、`installDir`、`timeZone` string<br>`nodeMaxThreads` int32<br>`autoOpenPorts`、`autoRemoteStart`、`autoInstallNftables`、`autoSystemTuning`、`autoTrimDisks` bool(漏传即 false)<br>`maxConcurrentReads`、`maxConcurrentWrites` int32<br>`csrfToken` string 必填 | 成功无业务字段 | 错误:F:"invalid cluster id"、"cluster not found";A:RPC 失败 | 整体覆盖集群基本设置,SSH 授权、时钟配置沿用原值。布尔和数值字段漏传会被清零。字段范围由 RPC 决定(`threadLimits` 在表单接口给出提示范围)。 | 需登录+Node 权限+CSRF |
| admin-03a-nodes-clusters | 更新集群健康检查 | POST | `/api/admin/clusters/policies/health` | 表单参数:`clusterId` int64(**处理器不校验 >0**)<br>`isOn` bool<br>`url` string,首尾空白会被去除,漏传为空串(处理器不套默认值)<br>`intervalSeconds` int64、`timeoutSeconds` int64<br>`statusCodes` string,以 `, ; 空格 换行 制表` 分隔,每项 100-599,留空取 `[200]`<br>`countTries` int64、`tryDelayMillis` int64<br>`autoDown` bool、`countUp` int、`countDown` int<br>`userAgent` string<br>`onlyBasicRequest`、`accessLogIsOn` bool<br>`csrfToken` string 必填 | 成功无业务字段 | 错误:F:"health check status codes must be between 100 and 599"及 `HealthCheckConfig.Init()` 的校验消息(文案待确认);A:RPC 失败 | 保存集群健康检查配置。该接口不会套用默认值:漏传的数值按 0 提交,由 `Init()` 决定是否补默认或报错(待确认)。读取接口 `GET /clusters/policies` 在无配置时给出的默认:`url=http://${host}/`、间隔 30 秒、超时 5 秒、重试 3 次、重试间隔 100 毫秒、`autoDown=true`、`countUp=1`、`countDown=3`、UA `EdgeRoute Health Check`、`onlyBasicRequest=true`。 | 需登录+Node 权限+CSRF |
| admin-03a-nodes-clusters | 更新集群 HTTP/3 | POST | `/api/admin/clusters/policies/http3` | 表单参数:`clusterId` int64(**不校验 >0**)<br>`isOn` bool<br>`port` int,必填,1-65535<br>`supportMobileBrowsers` bool<br>`csrfToken` string 必填 | 成功无业务字段 | 错误:F:"HTTP/3 port must be between 1 and 65535"及 `HTTP3Policy.Init()` 的消息(待确认);A:RPC 失败 | 保存集群 HTTP/3 策略。端口无论是否开启都必须合法,所以关闭时也要带上合法端口。 | 需登录+Node 权限+CSRF |


### 公共结构

### NodeLogsResult
`{"nodeId":int64,"total":int64,"offset":int64,"size":int64,"items":[{"id":int64,"tag":string,"description":string,"level":string,"createdAt":int64(待确认单位,推测为 Unix 秒),"count":int64,"isFixed":bool,"isRead":bool}]}`

校验:`offset≥0`,否则 "invalid log offset";`size` 传 0 取 20,否则须 1-100,否则 "log page size must be between 1 and 100";`dayFrom/dayTo` 须为 `YYYY-MM-DD`,否则 "invalid node log date",两者都给时 from 不得晚于 to;`keyword` 0-200 字符且无控制字符;`level` 小写后须为 `error/warning/info/success`;`tag` 转大写后须匹配 `^[A-Z0-9_:-]+$` 且 ≤64。上述各项具体错误文案除已列出的以外:待确认。

### UserNodeStatus(用户节点状态)
`state`(`ClassifyNodeHeartbeat` 的结果字符串,可选值集合待确认)、`freshWindowSeconds`、`heartbeatWindowSeconds`、`active`、`heartbeatActive`、`updatedAt`、`serverTime`、`buildVersion`、`os`、`arch`、`hostname`、`hostIP`、`cpuUsage`、`memoryUsage`、`memoryTotal`、`diskUsage`、`diskTotal`、`load1m`、`connectionCount`、`openFileCount`、`openFileSoftLimit`、`openFileHardLimit`、`openFileUsage`、`apiSuccessPercent`、`apiAvgCostSeconds`、`trafficInBytes`、`trafficOutBytes`、`systemTrafficInBytes`、`systemTrafficOutBytes`、`systemTrafficSampleSec`、`systemTrafficInDrops`、`systemTrafficOutDrops`、`error`、`statusReasons`。各数值的单位(使用率是 0-1 还是百分比、字节单位等)来自 `nodeconfigs.NodeStatus`,本文未核对,待确认。

### UserNodeView
`{"id","name","description","isOn","httpListens":[string],"httpsListens":[string],"accessAddrs":[string],"sslPolicyId":int64,"certificateIds":[int64],"certificates":[{"id","name","dnsNames"}],"status":UserNodeStatus}`。`httpListens` 元素为 `host:port`,host 为空时显示 `0.0.0.0`。

### DatabaseNode
`{"id":int64,"isOn":bool,"name":string,"dbEngine":string,"description":string,"host":string,"port":int32,"database":string,"username":string,"hasPassword":bool,"status":{"isOk","error","size","version"}}`。**不含密码**。`status` 在列表与表单中的取值来源:待确认。

### NodeDetail(`GET /nodes/detail` 的 `node`)
`id,name,isOn,isUp,isInstalled,active,heartbeatActive,installDir,uniqueId,version,latestVersion,newVersion,maxCPU,level,synced,enableIPLists,offlineDay,isBackupForCluster,isBackupForGroup,lnAddrs[],apiNodeCount,cluster{id,name}或null,secondaryClusters[{id,name,isOn}],group{id,name}或null,region{id,name}或null,ips[{id,name,ip,backupIP,canAccess,isOn,isUp,isHealthy,role,thresholdsJSON(JSON 字符串)}],installStatus{isRunning,isFinished,isOk,error,updatedAt},status{…}`。

`status` 键:`state,freshWindowSeconds,heartbeatWindowSeconds,isActive,heartbeatActive,hostname,hostIP,os,arch,buildVersion,cpuUsage,cpuPhysicalCount,cpuLogicalCount,memoryUsage,memoryTotal,diskUsage,diskTotal,diskPartition,diskWritingSpeedMB,updatedAt,serverTime,load1m,load5m,load15m,connectionCount,trafficInBytes,trafficOutBytes,openFileCount,openFileSoftLimit,openFileHardLimit,openFileUsage,cacheDiskBytes,cacheMemoryBytes,systemTrafficInBytes,systemTrafficOutBytes,systemTrafficSampleSec,systemTrafficInDrops,systemTrafficOutDrops,apiSuccessPercent,apiAvgCostSeconds,runtime,trafficUpload,l2EndpointCount,l2DegradedEndpointCount,error,licenseStatus,licenseMessage,licenseAllowed,licenseExpiresAt,licenseSource,licenseGraceUntil,listenerConfigured,listenerActive,listenerFailed,listenerFailures,statusReasons`。其中 `runtime`、`trafficUpload`、`listenerFailures`、`statusReasons` 的内部结构,以及各字段单位:待确认。

`advanced`:`maxCPU,bypassMobile,maxCacheDiskCapacity,maxCacheMemoryCapacity`(两个容量为 pb `SizeCapacity` 对象,键应为 count/unit,待确认)`,cacheDiskDir,cacheDiskSubDirsJSON(字符串),dnsResolverJSON(字符串),ddosProtectionJSON(字符串),offlineDay,isBackupForCluster,isBackupForGroup,backupIPs[],dnsFailoverJSON(字符串,默认 "[]"),apiNodeAddrsJSON(字符串,无配置为 "")`。存在调度信息时 `offlineDay`、`isBackup*`、`backupIPs`、`dnsFailoverJSON` 以调度信息为准。

### NodeOverviewItem(`GET /nodes/overview` 的 `nodes[]`)
`id,name,clusterId,clusters,regionId,regionName,level,isOn,isInstalled,heartbeatState,heartbeatActive,fresh,telemetryAvailable,addresses,status{freshWindowSeconds,heartbeatWindowSeconds,updatedAt,hostname,os,arch,buildVersion,cpuUsage,cpuLogicalCount,memoryUsage,memoryTotal,diskUsage,diskTotal,load1m,load5m,load15m,connections,inBytesPerSecond,outBytesPerSecond,uptimeSeconds,statusReasons}`。`clusters` 与 `addresses` 的元素结构:待确认。

### ClusterForm(`cluster`)
`id,name,installDir,timeZone,nodeMaxThreads,autoOpenPorts,autoRemoteStart,autoInstallNftables,autoSystemTuning,autoTrimDisks,maxConcurrentReads,maxConcurrentWrites,isGlobal`。

### ClusterPolicies 子对象
- `health`:`isOn,url,intervalSeconds,statusCodes(逗号拼接的字符串),timeoutSeconds,countTries,tryDelayMillis,autoDown,countUp,countDown,userAgent,onlyBasicRequest,accessLogIsOn`。
- `networkSecurity`:`status,useGlobal,tcpSyncookies,tcpRFC1337,tcpMaxSynBacklog,ipRPFilter,ipAcceptRedirects,ipSendRedirects,ipAcceptSourceRoute,icmpIgnoreBroadcasts,icmpIgnoreBogusResponses,icmpRateLimitMilliseconds`(各项取值含义待确认)。
- `cc`:`isOn,useGlobal,thresholds[{path,conditions,groupBy,priority,isOn,name,periodSeconds,maxRequests,blockSeconds,statusCode,perURI,methods(逗号拼接字符串)}]`。
- `ddos`:`isOn,useGlobal,maxConnections,maxConnectionsPerIP,newConnectionsMinutelyRate,newConnectionsMinutelyRateBlockTimeout,newConnectionsSecondlyRate,newConnectionsSecondlyRateBlockTimeout,synProxy,xdp,allowIPs(换行拼接字符串),ports(换行拼接字符串)`。
- `serverBaseline`:`useGlobal,matchDomainStrictly,domainMismatchAction,domainMismatchStatusCode,domainMismatchContentHTML,domainMismatchRedirectURL,allowMismatchDomains(换行拼接),allowNodeIP,nodeIPShowPage,nodeIPPageHTML,defaultDomain,domainAuditingIsOn,domainAuditingPrompt,serverName,supportsLowVersionHTTP,forceLnRequest,lnRequestSchedulingMethod,enableServerAddrVariable,requestOriginsWithEncodings,xffMaxAddresses,accessLogIsOn,accessLogEnableRequestHeaders,accessLogCommonRequestHeadersOnly,accessLogEnableResponseHeaders,accessLogEnableCookies,accessLogEnableServerNotFound,recordServerError,performanceAutoReadTimeout,performanceAutoWriteTimeout,performanceDebug,tcpPortRangeMin,tcpPortRangeMax,tcpDenyPorts(换行拼接),originShieldIsOn,originShieldMode,originShieldHost,originShieldProtocol,originShieldPort,originShieldCloudflareHost,originShieldCloudflareProtocol,originShieldCloudflareDNSProviderId,originShieldTokenConfigured,originProtectIsOn,originProtectSecretConfigured,originRateLimitIsOn,originRateLimitMaxQPS,originRateLimitBurst,cloudflareProviders[{id,name,type}]`。密钥与回环令牌只以 `…Configured` 布尔值给出。
- `assignments`:`cachePolicyId,cacheUseGlobal,wafPolicyId,wafUseGlobal,cachePolicies[{id,name,isOn}],wafPolicies[{id,name,isOn}]`。两个策略列表各最多取前 100 条。

本模块路由不包含 `/clusters/policies/` 下的其他保存接口(webp、uam、cc、ddos、server-baseline、network-security 等,见 `init.go:339-347`),它们不在本文范围。

<a id="m4"></a>

## 4. 管理端 · 节点与集群后半部分(管理端)

**模块约定**

> 路由总数 53,来源 EdgeAdmin/internal/web/actions/default/api/admin/init.go(路由注册,第 339-391 行,前缀 `/api/admin`;处理器见 `cluster_policies_resource.go`/`cluster_operations_resource.go`/`cluster_mutations.go`/`origin_protect_export_resource.go`/`node_create_resource.go`/`node_lifecycle_resource.go`/`node_enrollment_resource.go`/`node_rollouts_resource.go`/`node_groups_resource.go`/`node_regions_resource.go`/`global_node_logs_resource.go`/`node_grants_resource.go`)
说明:
- 权限:全部接口经 `helpers.NewUserMustAuth(AdminModuleCodeNode)`,须已登录且拥有 `node`(节点)模块权限。未登录 HTTP 401 `{"code":401,"message":"authentication required","data":null}`;无模块权限 HTTP 403 `{"code":403,"message":"permission denied","data":null}`;系统未初始化 409 `system setup is required`。
- 响应包络:成功 HTTP 200 `{"code":200,"message":"","data":{...}}`,下表"返回数据结构"只写 `data` 内字段,写"data 为空"表示 `data` 无业务字段。失败为 `{"code":N,"message":"...","data":...}`。
  - 经 `actionutils.Fail` 的错误:HTTP 状态按 gRPC 码映射(InvalidArgument 400/Unauthenticated 401/PermissionDenied 403/NotFound 404/AlreadyExists 409/ResourceExhausted 429/Unavailable 503/DeadlineExceeded 504/其他 500),`message` 为"服务处理失败"前缀加脱敏后的原因,含 password/secret/token/private key 字样的错误会被替换为通用文案。
  - 直接 `this.Fail("...")` 的校验错误:`message` 为下表所列英文/中文原文,`code` 默认 400(HTTP 状态头是否同步为 400:待确认)。下表"错误"一栏均指这类原文。
  - `node-rollouts` 系列写接口的 RPC 错误直接 `this.Fail(err.Error())`,不经脱敏和状态码映射。
- 请求参数:参数名为 params 结构字段名首字母小写(或 `alias` 指定名),从 query/表单取值,值先去首尾空白。整型非法值按 0 处理;bool 接受 `on`/`true`/`yes`/`enabled`(其他值按 Go 布尔解析规则,非法为 false);切片字段用重复参数(如 `groupIds=1&groupIds=2`)。是否支持 JSON 请求体:待确认。
- CSRF:全部 POST 须带参数 `csrfToken`(令牌由 `GET /api/admin/session` 的 `data.csrfToken` 获得,绑定当前会话,有效期 30 分钟)。校验失败返回 HTTP 403,纯文本体 `表单已失效,请刷新页面后重试(001)`,不是 JSON 包络;CSRF 校验先于所有业务校验。下表各 POST 行不再重复列出 `csrfToken`。全部 GET 不校验 CSRF。
- 分页:`page` 从 1 起,<1 按 1;`size` <1 按 20,>100 按 100。节点日志用 `offset`/`size`,规则见对应行。
- 集群策略类写接口(第 1、3-7、9、10、11-15 行):集群 ID 不存在报 `node cluster not found`;参数 `useGlobal`=true 且目标不是全局集群时,只把"沿用全局"开关写回集群,不写策略内容,也不做字段校验;`useGlobal`=false 时写入集群自身策略。
- 审计日志:未写操作日志的接口有第 11-15 行(阈值/通知/防火墙动作)、第 24-28 行(node-rollouts 写接口)、第 42-43 行(node-logs read/read-all)、第 8 行(GET 导出,含 reveal)。其余写接口均有。
- 密钥类字段:文档只记录字段,不记录值。

| 模块 | 接口名称 | 方法 | 路径 | 请求参数 | 响应数据 | 主要错误 | 功能说明 | 认证/权限 |
|---|---|---|---|---|---|---|---|---|
| admin-03b-nodes-clusters | 更新集群系统服务策略 | POST | `/api/admin/clusters/policies/system-service` | `clusterId` int64 必填<br>`useGlobal` bool 默认 false<br>`systemdIsOn` bool 默认 false | data 为空 | 错误:`node cluster not found` | 开关集群节点的 systemd 服务托管;写入后记录日志。 | 需登录(node 模块)+CSRF |
| admin-03b-nodes-clusters | 检查集群 systemd 服务 | POST | `/api/admin/clusters/policies/system-service/check` | `clusterId` int64 必填,>0 | `results`:向集群节点下发检查消息(超时 10 秒)的聚合结果,元素结构待确认 | 错误:`invalid cluster id` | 向集群下所有节点发送 systemd 服务检查消息并返回结果。 | 需登录(node 模块)+CSRF |
| admin-03b-nodes-clusters | 更新集群 WebP 策略 | POST | `/api/admin/clusters/policies/webp` | `clusterId` int64 必填<br>`useGlobal` bool<br>`isOn` bool<br>`quality` int,0..100<br>`requireCache` bool<br>`minLengthMB` int64,≥0<br>`maxLengthMB` int64,≥ minLengthMB | data 为空 | 错误:`invalid WebP policy values`;`policy.Init()` 的错误原文 | 配置集群 WebP 转换策略(质量、是否要求命中缓存、体积范围 MB)。 | 需登录(node 模块)+CSRF |
| admin-03b-nodes-clusters | 更新集群 UAM 策略 | POST | `/api/admin/clusters/policies/uam` | `clusterId` int64 必填<br>`useGlobal` bool<br>`isOn` bool | data 为空 | 错误:`node cluster not found` | 开关集群 UAM(5 秒盾)策略。 | 需登录(node 模块)+CSRF |
| admin-03b-nodes-clusters | 更新集群 CC 防护策略 | POST | `/api/admin/clusters/policies/cc` | `clusterId` int64 必填<br>`useGlobal` bool<br>`thresholdsJSON` string,`HTTPCCThreshold` 数组的 JSON,最多 20 个,至少 1 个已启用 | data 为空 | 错误:`invalid CC thresholds`;`CC thresholds cannot exceed 20`;`at least one CC threshold is required when protection is enabled` | 保存集群 CC 阈值列表;策略 `IsOn` 强制为 true。阈值元素字段:待确认(见 `serverconfigs.HTTPCCThreshold`)。 | 需登录(node 模块)+CSRF |
| admin-03b-nodes-clusters | 更新集群 DDoS 防护策略 | POST | `/api/admin/clusters/policies/ddos` | `clusterId` int64 必填<br>`useGlobal` bool<br>`isOn` bool<br>`maxConnections` int32,≥0<br>`maxConnectionsPerIP` int32,0 或 ≥5<br>`newConnectionsMinutelyRate` int32,0 或 ≥3<br>`newConnectionsMinutelyRateBlockTimeout` int32,≥0<br>`newConnectionsSecondlyRate` int32,0 或 ≥3<br>`newConnectionsSecondlyRateBlockTimeout` int32,≥0<br>`synProxy` bool<br>`xdp` bool<br>`allowIPs` string,以逗号/分号/换行/空白分隔,≤200 个合法 IP<br>`ports` string,同样分隔,≤100 个,1..65535 | data 为空 | 错误:`DDoS limits cannot be negative`;`per-IP connection limit is below the supported minimum`;`per-minute connection rate is below the supported minimum`;`per-second connection rate is below the supported minimum`;`DoS allow list cannot exceed 200 addresses`;`invalid DDoS allow-list IP: x`;`DDoS protected ports cannot exceed 100` | 保存集群 DDoS 防护设置(连接数/新建连接速率/SYN 代理/XDP/白名单/受保护端口)。 | 需登录(node 模块)+CSRF |
| admin-03b-nodes-clusters | 更新集群服务基线 | POST | `/api/admin/clusters/policies/server-baseline` | `clusterId` int64 必填<br>`useGlobal` bool<br>域名匹配:`matchDomainStrictly` bool;`domainMismatchAction` string,`page`/`redirect`/`close`;`domainMismatchStatusCode` int,400..599;`domainMismatchContentHTML` string,≤64KiB;`domainMismatchRedirectURL` string,action=redirect 时须为绝对 http/https 地址且无 userinfo;`allowMismatchDomains` string,≤100 个合法域名;`allowNodeIP` bool;`nodeIPShowPage` bool;`nodeIPPageHTML` string,≤64KiB;`defaultDomain` string,合法域名或 IP;`domainAuditingIsOn` bool;`domainAuditingPrompt` string,≤200<br>通用:`serverName` string,≤60 且不含 CR/LF/NUL;`supportsLowVersionHTTP`/`forceLnRequest`/`enableServerAddrVariable`/`requestOriginsWithEncodings` bool;`lnRequestSchedulingMethod` string,`random`/`urlMapping`;`xffMaxAddresses` int,0..100<br>访问日志:`accessLogIsOn`/`accessLogEnableRequestHeaders`/`accessLogCommonRequestHeadersOnly`/`accessLogEnableResponseHeaders`/`accessLogEnableCookies`/`accessLogEnableServerNotFound` bool;`recordServerError` bool<br>性能:`performanceAutoReadTimeout`/`performanceAutoWriteTimeout`/`performanceDebug` bool<br>TCP:`tcpPortRangeMin` int,≥1024;`tcpPortRangeMax` int,≤65534 且 ≥ min;`tcpDenyPorts` string,≤200 个,1..65535<br>全局 L2:`originShieldIsOn` bool;`originShieldMode` string,`direct`/`cloudflare`;`originShieldHost`/`originShieldProtocol`(http/https)/`originShieldPort`(1..65535);`originShieldCloudflareHost`(须为域名)/`originShieldCloudflareProtocol`(http/https)/`originShieldCloudflareDNSProviderId` int64,>0 且类型为 cloudFlare<br>源站保护:`originProtectIsOn` bool;`originProtectRotateSecret` bool<br>回源限速:`originRateLimitIsOn` bool;`originRateLimitMaxQPS` int,1..1000;`originRateLimitBurst` int,0..1000000 | data 为空<br>错误(原文):`invalid unmatched-domain action`;`invalid default domain`;`X-Forwarded-For address limit must be between 0 and 100`;`TCP port range must be ordered and within 1024-65534`;回源限速以"回源限速配置无效:"开头;其余域名/端口/URL/HTML 长度类错误原文待确认 | 见模块约定 | 一次性保存集群服务基线(未匹配域名处理/默认域名/访问日志/性能/TCP 端口段/L2 回源/源站保护/回源限速)。仅在需要写入时(非沿用全局)校验。源站保护开启时沿用旧密钥或生成新密钥,`originProtectRotateSecret`=true 时轮换;密钥只写不读,响应中不返回。L2 的 `LoopToken` 自动生成并保留。 | 需登录(node 模块)+CSRF |
| admin-03b-nodes-clusters | 导出源站保护规则 | GET | `/api/admin/clusters/origin-protect/export` | `clusterId` int64 必填,>0<br>`ports` string,可空,分隔字符串,≤20 个,1..65535<br>`reveal` bool 默认 false | `enabled`,`nodeCount`,`ipv4`,`ipv6`,`skipped`,`nginx`,`iptables`,`header`,`ipCount`,`note`,`secretConfigured`;`reveal`=true 且 `enabled` 时多一个 `secret`(回源签名密钥,文档不列值)<br>响应头 `Cache-Control: no-store`<br>`ipv4`/`ipv6`/`skipped`/`nginx`/`iptables` 元素结构来自 `BuildOriginProtectRules`,待确认 | 错误:`invalid cluster id`,及端口数量/范围类错误原文待确认 | 按集群内已启用节点的登记 IP 与状态上报的 HostIP 生成源站防火墙放行规则。GET 请求可通过 `reveal` 取得密钥,无 CSRF;er-v45 起返回密钥时会写一条审计日志(`NodeCluster_LogRevealOriginProtectSecret`)。 | 需登录(node 模块) |
| admin-03b-nodes-clusters | 更新集群网络安全策略 | POST | `/api/admin/clusters/policies/network-security` | `clusterId` int64 必填<br>`useGlobal` bool<br>`status` string,`auto`/`on`/`off`<br>`tcpSyncookies` bool<br>`tcpRFC1337` bool<br>`tcpMaxSynBacklog` int,0 取默认,否则 128..1000000<br>`ipRPFilter` int,0..2<br>`ipAcceptRedirects`/`ipSendRedirects`/`ipAcceptSourceRoute` bool<br>`icmpIgnoreBroadcasts`/`icmpIgnoreBogusResponses` bool<br>`icmpRateLimitMilliseconds` int,0..60000 | data 为空 | 错误:`invalid network security status "x"`;`maxSynBacklog should be between 128 and 1000000`;`rpFilter should be between 0 and 2`;`ICMP rateLimitMilliseconds should be between 0 and 60000` | 保存节点内核网络安全策略。`auto` 模式会强制 syncookies/RFC1337 开启、重定向与源路由关闭、ICMP 忽略广播与错误响应;`off` 模式不校验其余数值。 | 需登录(node 模块)+CSRF |
| admin-03b-nodes-clusters | 更新集群缓存/WAF 策略指派 | POST | `/api/admin/clusters/policies/assignments` | `clusterId` int64 必填<br>`cachePolicyId` int64,需写入时 >0<br>`cacheUseGlobal` bool<br>`wafPolicyId` int64,需写入时 >0<br>`wafUseGlobal` bool | data 为空 | 错误:`cache policy is required`;`WAF policy is required`;`node cluster not found` | 为集群指派缓存策略与 WAF 策略,两者的"沿用全局"开关各自独立;各写一条日志。 | 需登录(node 模块)+CSRF |
| admin-03b-nodes-clusters | 保存集群阈值 | POST | `/api/admin/clusters/operations/thresholds/save` | `clusterId` int64 必填<br>`thresholdId` int64,≤0 新建,>0 更新<br>`useGlobal` bool<br>`item`/`param` string,须在节点监控项定义中<br>`operator` string,须有效<br>`value` string,须为数字<br>`message` string,≤500<br>`sumMethod` string,`sum`/`avg`<br>`duration` int32,1..10080<br>`durationUnit` string,固定 `minute`<br>`notifyDuration` int32,0..10080<br>`isOn` bool,仅更新时生效 | data 为空 | 错误:`invalid threshold metric`;`threshold value must be numeric`;`invalid threshold aggregation`;`invalid threshold duration`;`threshold message is too long`;`cluster threshold limit reached`(每集群 100 条);`threshold not found in cluster`;`invalid threshold id` | 新建或更新集群监控阈值。沿用全局且非全局集群时只写开关并直接成功。无审计日志。 | 需登录(node 模块)+CSRF |
| admin-03b-nodes-clusters | 删除集群阈值 | POST | `/api/admin/clusters/operations/thresholds/delete` | `clusterId` int64 必填<br>`thresholdId` int64 必填,>0 | data 为空 | 错误:`inherited thresholds cannot be deleted from this cluster`;`threshold not found in cluster`;`invalid threshold id` | 删除集群自有阈值;沿用全局的集群不可删。无审计日志。 | 需登录(node 模块)+CSRF |
| admin-03b-nodes-clusters | 保存集群通知接收人 | POST | `/api/admin/clusters/operations/notifications` | `clusterId` int64 必填<br>`useGlobal` bool<br>`receiversJSON` string,≤64KiB,格式 `[{"type":"recipient"或"group","id":int64}]`,按 type+id 去重,id≤0 或未知 type 跳过,最多 200 个 | data 为空 | 错误:`notification selection is too large`;`invalid notification selection`;`too many notification receivers` | 设置集群告警通知的接收人/接收分组;沿用全局时不改接收人。无审计日志。 | 需登录(node 模块)+CSRF |
| admin-03b-nodes-clusters | 保存集群防火墙动作 | POST | `/api/admin/clusters/operations/firewall/save` | `clusterId` int64 必填<br>`actionId` int64,0 新建,>0 更新<br>`useGlobal` bool<br>`name` string 必填,≤100<br>`eventLevel` string,`debug`/`notice`/`warning`/`error`/`critical`/`fatal`<br>`type` string,`ipset`/`firewalld`/`iptables`/`script`/`httpAPI`/`html`<br>ipset:`ipSetWhiteName`/`ipSetBlackName`/`ipSetWhiteNameIPv6`/`ipSetBlackNameIPv6` string,仅字母数字下划线;`ipSetMaxElements` int,0..10000000;`ipSetAutoAddToIPTables`/`ipSetAutoAddToFirewalld` bool<br>script:`path` string 必填;`scriptCwd` string;`scriptArgsJSON` string,JSON 字符串数组,≤100 个,每个 ≤500<br>httpAPI:`httpAPIURL` string,http/https 且无 userinfo;`httpAPITimeoutSeconds` int,1..60;`httpAPISecret` string,≤512 字节,留空沿用旧值<br>html:`htmlContent` string,1 字节..64KiB | data 为空 | 错误:`invalid firewall action metadata`;`firewall action limit reached`(每集群 100 条);`unsupported firewall action type`;`IPSet names may only contain letters, numbers and underscores`;`HTML response must be between 1 byte and 64 KiB`;其余字段错误原文待确认 | 新建或更新集群防火墙响应动作。字段随 `type` 取舍。无审计日志;密钥不在响应中返回。 | 需登录(node 模块)+CSRF |
| admin-03b-nodes-clusters | 删除集群防火墙动作 | POST | `/api/admin/clusters/operations/firewall/delete` | `clusterId` int64 必填<br>`actionId` int64 必填 | data 为空 | 错误:`inherited firewall actions cannot be deleted from this cluster`;`firewall action not found in cluster` | 删除集群自有防火墙动作。无审计日志。 | 需登录(node 模块)+CSRF |
| admin-03b-nodes-clusters | 置顶/取消置顶集群 | POST | `/api/admin/clusters/pin` | `clusterId` int64 必填,>0<br>`isPinned` bool | data 为空 | 错误:`invalid cluster id` | 设置集群是否置顶;记录日志。 | 需登录(node 模块)+CSRF |
| admin-03b-nodes-clusters | 删除集群 | POST | `/api/admin/clusters/delete` | `clusterId` int64 必填 | data 为空 | 错误:`cluster still has active sites` | 集群下仍有未删除站点时拒绝,否则调用 `DeleteNodeCluster`;记录日志。 | 需登录(node 模块)+CSRF |
| admin-03b-nodes-clusters | 创建节点 | POST | `/api/admin/nodes/create` | `clusterId` int64 必填<br>`name` string 必填,≤100 字符<br>`addresses` string 必填,以换行/逗号/分号分隔,支持 IP 段展开,去重后 1..256 个<br>`groupId` int64,可选<br>`regionId` int64,可选<br>`grantId` int64,可选(SSH 授权)<br>`sshHost` string,可选<br>`sshPort` int,0 取 22,范围 1..65535<br>`sshPassword` string,可选,≤255 且不含 NUL<br>`dnsRoutes` string 数组,可选<br>`dnsDomainId` int64:接收但未使用,实际按集群 DNS 配置推导 | `nodeId` int64 | 错误:`selected node group is unavailable`;`selected node region is unavailable`;`selected SSH grant is unavailable`;`已选择 SSH 授权,请清空密码,或将授权改为'稍后配置'`;`SSH 登录失败:用户名或密码错误,请检查后重试` | 创建节点并登记 IP 地址。填写 `sshPassword` 时自动创建 SSH 授权(用户名固定 `root`,方式 `user`),先测试连接:认证失败则禁用该授权并报错,主机不可达不阻塞。`dnsRoutes` 仅保留属于集群绑定域名的有效线路,无效项被静默丢弃。IP 创建失败会回滚节点与新建授权。记录日志。 | 需登录(node 模块)+CSRF |
| admin-03b-nodes-clusters | 批量创建节点 | POST | `/api/admin/nodes/create-batch` | `clusterId` int64 必填<br>`namePrefix` string,可选<br>`addresses` string 必填,≤100 个<br>`groupId` int64,可选<br>`regionId` int64,可选<br>`grantId` int64,可选 | `nodeIds` int64 数组 | 错误:分组/区域/授权不可用类提示原文待确认 | 每个地址创建一个节点,节点名为地址;有前缀时为 `<前缀> 001` 递增。选了授权则 SSH 配置 host=节点 IP、port=22。任一失败会回滚。记录日志。 | 需登录(node 模块)+CSRF |
| admin-03b-nodes-clusters | 集群生命周期总览 | GET | `/api/admin/clusters/lifecycle` | `clusterId` int64 必填 | `cluster{id,name}`;`groups[{id,name}]`;`regions[{id,name}]`;`grants[{id,name,method,username}]`;`notInstalled[Item]`;`upgrades[Item]`;`registration{endpoints,clusterId,secret,configJSON}`(`Item` 见公共结构)<br>响应头 `Cache-Control: no-store` | 错误:`node cluster not found` | 返回集群待安装节点、待升级节点、可选分组/区域/SSH 授权与节点注册信息。`registration.secret` 及 `configJSON` 内的 secret 是集群密钥明文(疑似缺陷,见文末),文档不列值。 | 需登录(node 模块) |
| admin-03b-nodes-clusters | 查询节点安装令牌(列表) | GET | `/api/admin/nodes/enrollment` | `clusterId` int64 必填 | `items`:`Record` 数组(见公共结构)<br>响应头 `Cache-Control: no-store` | 见模块约定 | 同一 action(`NodeEnrollmentAction`)的 GET 分支:固定以 `action=list` 调用 RPC 列出集群的节点安装记录。不写日志。 | 需登录(node 模块) |
| admin-03b-nodes-clusters | 管理节点安装令牌 | POST | `/api/admin/nodes/enrollment` | `action` string 必填,`create`/`approve`/`revoke`<br>`clusterId` int64<br>`groupId` int64<br>`regionId` int64<br>`level` int<br>`count` int<br>`id` string,64 位小写十六进制(approve/revoke 用)<br>`address` string<br>`baseURL` string,action=create 时必须为 https、无 userinfo/query/fragment、路径为空或 `/admin` | `items` Record 数组;`tokens` string 数组(create 时一次性返回安装令牌,文档不列值);`record` Record;均为 omitempty,可能缺省<br>响应头 `Cache-Control: no-store` | 错误:`不支持的安装管理操作`;`请填写可从节点访问的 HTTPS 管理端地址`;`管理端地址路径只能为空或 /admin` | 同一 action 的 POST 分支:创建/批准/撤销节点安装记录。令牌有效期 15 分钟,重试期 60 分钟。`baseURL` 仅校验,未传入 RPC 请求。`count`/`level` 范围及 `state`/`phase` 枚举由 RPC 侧决定,待确认。写操作记录日志(不含令牌)。 | 需登录(node 模块)+CSRF |
| admin-03b-nodes-clusters | 节点发布批次列表 | GET | `/api/admin/node-rollouts` | `clusterId` int64 | `items[Rollout]`(见公共结构)<br>响应头 `Cache-Control: no-store` | 见模块约定 | 列出集群的节点滚动发布任务。`clusterId` 必填性待确认。 | 需登录(node 模块) |
| admin-03b-nodes-clusters | 创建节点发布批次 | POST | `/api/admin/node-rollouts/create` | `clusterId` int64<br>`role` string,空则 `node`<br>`taskType` string<br>`nodeIds` int64 数组<br>`batchSize` int,1..100<br>`userId` int64<br>`serverId` int64 | `rollout`:`Rollout` | 错误:`每批节点数必须在 1 到 100 之间`;RPC 错误原文 | 创建分批滚动发布任务。`taskType` 取值待确认。无审计日志;错误用 `this.Fail(err.Error())` 原样返回。 | 需登录(node 模块)+CSRF |
| admin-03b-nodes-clusters | 暂停节点发布批次 | POST | `/api/admin/node-rollouts/pause` | `rolloutId` int64 | data 为空 | 错误:RPC 错误原文 | 暂停指定发布任务。无审计日志。 | 需登录(node 模块)+CSRF |
| admin-03b-nodes-clusters | 恢复节点发布批次 | POST | `/api/admin/node-rollouts/resume` | `rolloutId` int64 | data 为空 | 错误:RPC 错误原文 | 恢复已暂停的发布任务。无审计日志。 | 需登录(node 模块)+CSRF |
| admin-03b-nodes-clusters | 取消节点发布批次 | POST | `/api/admin/node-rollouts/cancel` | `rolloutId` int64<br>`reason` string,可选 | data 为空 | 错误:RPC 错误原文 | 取消发布任务并记录原因。无审计日志。 | 需登录(node 模块)+CSRF |
| admin-03b-nodes-clusters | 重试节点发布批次 | POST | `/api/admin/node-rollouts/retry` | `rolloutId` int64,>0 | data 为空 | 错误:`rollout id is required`;RPC 错误原文 | 重试失败的发布任务。无审计日志。 | 需登录(node 模块)+CSRF |
| admin-03b-nodes-clusters | 升级节点 | POST | `/api/admin/nodes/upgrade` | `nodeId` int64 必填,>0<br>`clusterId` int64,可选;>0 时校验节点属于该集群(含次要集群) | data 为空 | 错误:节点不属于集群/无效 ID 类提示原文待确认 | 触发单个节点升级;记录日志。 | 需登录(node 模块)+CSRF |
| admin-03b-nodes-clusters | 节点分组列表 | GET | `/api/admin/node-groups` | `clusterId` int64 必填,>0 | `items[{id,name,countNodes}]`;`clusterId` | 错误:集群不存在/无效 ID 原文待确认 | 列出集群下的节点分组及节点数。 | 需登录(node 模块) |
| admin-03b-nodes-clusters | 创建节点分组 | POST | `/api/admin/node-groups/create` | `clusterId` int64 必填<br>`name` string 必填,去空白后 1..100 字符 | `groupId` int64 | 错误:`node group name is required`;`node group name is too long` | 在集群下创建节点分组;记录日志。 | 需登录(node 模块)+CSRF |
| admin-03b-nodes-clusters | 更新节点分组 | POST | `/api/admin/node-groups/update` | `clusterId` int64 必填<br>`groupId` int64 必填<br>`name` string 必填,1..100 字符 | data 为空 | 错误:`node group name is required`;`node group name is too long`;`node group does not belong to the selected cluster`;`invalid node group` | 修改分组名称;记录日志。 | 需登录(node 模块)+CSRF |
| admin-03b-nodes-clusters | 删除节点分组 | POST | `/api/admin/node-groups/delete` | `clusterId` int64 必填<br>`groupId` int64 必填 | data 为空 | 错误:`node group still has active nodes`;`node group does not belong to the selected cluster`;`invalid node group` | 分组下仍有节点时拒绝删除;记录日志。 | 需登录(node 模块)+CSRF |
| admin-03b-nodes-clusters | 排序节点分组 | POST | `/api/admin/node-groups/sort` | `clusterId` int64 必填<br>`groupIds` int64 数组,须恰好是该集群现有全部分组的一个排列 | data 为空 | 错误:提交 ID 与现有集合不一致的提示原文待确认 | 按提交顺序重排集群的全部节点分组;记录日志。 | 需登录(node 模块)+CSRF |
| admin-03b-nodes-clusters | 节点区域列表 | GET | `/api/admin/node-regions` | `page` int 默认 1<br>`size` int 默认 20,最大 100<br>`regionId` int64,≥0,0 表示全部 | `regions[{id,name,description,isOn,countNodes,displayName,countryCode,hasLocation,latitude,longitude}]`;`nodes[{id,name,cluster{id,name}或null,region{id,name}或null}]`;`page`;`size`;`total`;`selectedRegionId` | 见模块约定 | 分页列出节点区域,并列出所选区域(或全部)下的节点。 | 需登录(node 模块) |
| admin-03b-nodes-clusters | 创建节点区域 | POST | `/api/admin/node-regions/create` | `name` string 必填,≤100 字符<br>`description` string,≤500<br>`displayName` string,≤100<br>`countryCode` string,2 位字母,转大写<br>`hasLocation` bool<br>`latitude` float,-90..90<br>`longitude` float,-180..180 | `regionId` int64 | 错误:名称必填/超长/国家码/经纬度越界/含控制字符(`\n`、`\t` 例外)类提示,原文待确认 | 创建区域。经纬度仅在 `hasLocation`=true 时校验,否则置 0。日志用 defer 写入,创建失败也会记录一条 regionId=0。 | 需登录(node 模块)+CSRF |
| admin-03b-nodes-clusters | 更新节点区域 | POST | `/api/admin/node-regions/update` | 同创建,另加:<br>`regionId` int64 必填<br>`isOn` bool | data 为空<br>错误同创建 | 见模块约定 | 修改区域信息和启停状态;记录日志。 | 需登录(node 模块)+CSRF |
| admin-03b-nodes-clusters | 删除节点区域 | POST | `/api/admin/node-regions/delete` | `regionId` int64 必填 | data 为空 | 错误:`node region still has active nodes` | 区域下仍有节点时拒绝删除;记录日志。 | 需登录(node 模块)+CSRF |
| admin-03b-nodes-clusters | 排序节点区域 | POST | `/api/admin/node-regions/sort` | `regionIds` int64 数组,须恰好是全部区域的一个排列 | data 为空 | 错误:提交 ID 与现有集合不一致的提示原文待确认 | 按提交顺序重排全部区域;记录日志。 | 需登录(node 模块)+CSRF |
| admin-03b-nodes-clusters | 移动节点到区域 | POST | `/api/admin/node-regions/move-node` | `nodeId` int64 必填,>0<br>`regionId` int64,0 表示取消区域,<0 非法 | data 为空 | 错误:`node region is disabled`;节点/区域 ID 无效提示原文待确认 | 把节点划入某区域或移出区域;目标区域须启用;记录日志。 | 需登录(node 模块)+CSRF |
| admin-03b-nodes-clusters | 节点日志列表 | GET | `/api/admin/node-logs` | `offset` int,≥0,默认 0<br>`size` int,0 取 20,有效 1..100<br>`dayFrom`/`dayTo` string,日期,dayFrom 不得晚于 dayTo(格式待确认,疑为 YYYY-MM-DD)<br>`keyword` string,≤200<br>`level` string,`error`/`warning`/`info`/`success`,转小写<br>`type` string,空/`unread`/`needFix`<br>`tag` string,转大写,≤64,匹配 `^[A-Z0-9_:-]+$`<br>`clusterId` int64,≥0<br>`nodeId` int64,≥0 | `items[{id,tag,description,level,createdAt,count,isFixed,isRead,canFix,node{id,name,cluster{id,name}},server{id,name}或null}]`;`total`;`offset`;`size`;`countUnread`;`countNeedFix`;`tags`(`FindNodeCommonTags` 的结构待确认);`clusters[{id,name}]`;`nodes[{id,name}]`(仅 `clusterId`>0 时返回,最多 500 个) | 错误:`invalid node log type`;分页/日期/级别/标签/关键词越界提示原文待确认 | 查询全局节点日志。`nodeId`>0 时校验节点存在(且属于所给 `clusterId`);否则 `clusterId`>0 时校验集群存在。`canFix = 未修复 且 serverId>0 且 level 为 success/warning/error`。`createdAt` 为 pb 原值,格式待确认。 | 需登录(node 模块) |
| admin-03b-nodes-clusters | 标记节点日志已读 | POST | `/api/admin/node-logs/read` | `logIds` int64 数组,1..100 个 | data 为空 | 错误:`select between 1 and 100 node logs` | 把所选日志标为已读,并通知未读数刷新。无审计日志。 | 需登录(node 模块)+CSRF |
| admin-03b-nodes-clusters | 全部标记节点日志已读 | POST | `/api/admin/node-logs/read-all` | 无业务参数 | data 为空 | 见模块约定 | 把全部节点日志标为已读,并通知未读数刷新。无审计日志。 | 需登录(node 模块)+CSRF |
| admin-03b-nodes-clusters | 修复节点日志 | POST | `/api/admin/node-logs/fix` | `logIds` int64 数组,1..100 个 | data 为空 | 错误:`select between 1 and 100 node logs` | 把所选日志标记为已修复;记录日志。 | 需登录(node 模块)+CSRF |
| admin-03b-nodes-clusters | 全部修复节点日志 | POST | `/api/admin/node-logs/fix-all` | 无业务参数 | data 为空 | 见模块约定 | 把全部可修复日志标记为已修复;记录日志。 | 需登录(node 模块)+CSRF |
| admin-03b-nodes-clusters | 删除节点日志 | POST | `/api/admin/node-logs/delete` | `dayFrom`/`dayTo` string<br>`keyword` string,必填<br>`level` string<br>`type` string<br>`tag` string<br>`clusterId` int64,不校验,允许负值<br>`nodeId` int64,不校验,允许负值 | data 为空 | 错误:`deleting node logs requires a keyword` | 按条件批量删除节点日志,必须带关键词;记录日志。 | 需登录(node 模块)+CSRF |
| admin-03b-nodes-clusters | SSH 授权列表 | GET | `/api/admin/node-grants` | `page` int 默认 1<br>`size` int 默认 20,最大 100<br>`keyword` string,可选 | `items[{id,name,method,methodName,username,description,sudo,countClusters,countNodes}]`;`total`;`page`;`size`<br>`methodName`:`SH 私钥`/`用户名与密码` | 见模块约定 | 分页列出节点 SSH 授权,不返回任何密码或私钥。 | 需登录(node 模块) |
| admin-03b-nodes-clusters | SSH 授权表单 | GET | `/api/admin/node-grants/form` | `grantId` int64,≤0 表示新建 | `grant`:新建时为 null,否则含列表项各键,另加 `hasPassword`/`hasPrivateKey`/`hasPassphrase`(bool,不返回明文)<br>响应头 `Cache-Control: no-store` | 见模块约定 | 取得编辑表单所需的授权信息。 | 需登录(node 模块) |
| admin-03b-nodes-clusters | SSH 授权详情 | GET | `/api/admin/node-grants/detail` | `grantId` int64 必填 | `grant`(同 form,`countClusters`/`countNodes` 有值);`clusters[{id,name}]`;`nodes[{id,name,isOn,cluster{id,name}或null}]`<br>响应头 `Cache-Control: no-store` | 错误:授权不存在提示原文待确认 | 查看授权及其被哪些集群、节点使用。 | 需登录(node 模块) |
| admin-03b-nodes-clusters | 创建 SSH 授权 | POST | `/api/admin/node-grants/create` | `name` string<br>`method` string,已见 `privateKey`/`user`<br>`username` string<br>`password` string<br>`privateKey` string<br>`passphrase` string<br>`description` string<br>`sudo` bool | `grantId` int64 | 错误:校验全部交给 RPC,取值范围待确认 | 创建 SSH 授权;记录日志。 | 需登录(node 模块)+CSRF |
| admin-03b-nodes-clusters | 更新 SSH 授权 | POST | `/api/admin/node-grants/update` | 同创建,另加 `grantId` int64 必填 | data 为空 | 错误:同创建,待确认 | 更新 SSH 授权;密码/私钥留空时是否保留旧值取决于 RPC,待确认;记录日志。 | 需登录(node 模块)+CSRF |
| admin-03b-nodes-clusters | 删除 SSH 授权 | POST | `/api/admin/node-grants/delete` | `grantId` int64 必填 | data 为空 | 见模块约定 | 软删除授权(`DisableNodeGrant`);记录日志。 | 需登录(node 模块)+CSRF |
| admin-03b-nodes-clusters | 测试 SSH 授权 | POST | `/api/admin/node-grants/test` | `grantId` int64 必填<br>`host` string<br>`port` int32 | `isOk` bool;`error` string | 见模块约定 | 用授权连接指定主机端口,测试结果通过 `isOk`/`error` 返回,连接失败不走错误包络。 | 需登录(node 模块)+CSRF |


### 公共结构

### Record(节点安装记录)
- `id` string
- `clusterId` int64
- `groupId` int64
- `regionId` int64
- `level` int
- `createdBy`
- `createdAt`
- `expiresAt`
- `updatedAt`
- `claimedAt?`
- `approvedBy?`
- `nodeId?`
- `name?`
- `state`
- `phase`
- `arch?`
- `version`
- `sha256?`
- `clientHash?`
- `online` bool
- `nodeVersion?`
- `failureReason?`

字段类型(除 `id`/`clusterId`/`groupId`/`regionId`/`level`/`online` 外)及 `state`/`phase` 枚举:待确认,见 `nodeenrollment/types.go`。

### Rollout(发布批次)
- `id` int64
- `clusterId` int64
- `role` string
- `taskType` string
- `nodeIds` int64 数组
- `batchSize` int
- `currentBatch` int
- `batchCount` int
- `status` string
- `failureReason` string
- `createdAt`
- `updatedAt`

`status`/`taskType` 枚举及时间字段格式:待确认。

### Item(生命周期节点项)
- `id` int64
- `name` string
- `addreses[{ip,canAccess}]`(键名为源码原拼写 `addreses`)
- `ssh{configured,host,port}`
- `installStatus{isRunning,isFinished,isOk,error}`
- `os` string
- `arch` string
- `oldVersion` string
- `newVersion` string

### 异常与疑似缺陷
- `clusters/lifecycle` 的 `registration.secret`/`configJSON` 明文返回集群密钥。
- `origin-protect/export?reveal=true` 以 GET 返回回源签名密钥,无 CSRF;已补审计日志(er-v45),仍建议后续改为 POST。
- 阈值/通知/防火墙动作、node-rollouts 写接口、node-logs read/read-all 无审计日志。
- node-rollouts 写接口错误绕过脱敏与状态码映射。
- `nodes/create` 的 `dnsDomainId` 未使用;`nodes/enrollment` 的 `baseURL` 仅校验未传 RPC。
- `node-regions/create` 日志用 defer,失败也记录;`node-logs/delete` 的 `clusterId`/`nodeId` 不校验。
- CSRF 失败返回纯文本 403,与其余 JSON 错误格式不一致。
- 多数集群策略接口无 `clusterId>0` 前置校验,不存在时才报 `node cluster not found`。

<a id="m5"></a>

## 5. 管理端 · DNS 管理(管理端)

**模块约定**

> 路由总数 22,来源 EdgeAdmin/internal/web/actions/default/api/admin/init.go
说明:
- 路径前缀 `/api/admin`。处理器全部在 `EdgeAdmin/internal/web/actions/default/api/admin/dns_resource.go`,TeaGo Action。22 条路由全部经 `helpers.NewUserMustAuth(AdminModuleCodeDNS)`(init.go 397-418 行),下表"认证"列统一写"需登录+DNS权限"。
- 认证失败:`/api/` 路径下未登录返回 HTTP 401 `{"code":401,"message":"authentication required","data":null}`;无 DNS 模块权限返回 403 `{"code":403,"message":"permission denied","data":null}`;系统未初始化返回 409;WAF/IP 拦截返回 403(文本或空 body)。
- 请求参数:TeaGo 从 URL 查询串合并 POST 表单取值,**不解析 JSON 请求体**。参数名为结构体字段名首字母小写(`DnsName` 对应 `dnsName`,`IpAddrId` 对应 `ipAddrId`,`Ttl` 对应 `ttl`)。缺省参数为零值(整数 0、字符串空、布尔 false);字符串与数值取值前会去首尾空白。布尔值 `on/true/yes/enabled` 为 true,其余按常规布尔解析,所以**漏传布尔参数等同于 false**。数组参数用重复键(`cnameRecords=a&cnameRecords=b`)。`[]byte` 类型(`paramsJSON`)取第一个值。本模块没有 `default:"x"` 标签,下文"默认"均指处理器代码里的默认。
- 成功响应:HTTP 200,`{"code":200,"message":"","data":{...}}`。下表"返回"列只写 `data` 内的字段;写操作无业务字段时 `data` 为空对象(`Data` 初始值序列化形态:待确认)。
- 失败响应有两种格式,同模块混用:
  - **F 类**(`this.Fail(msg)`):`{"code":400,"message":"<原始英文消息>","data":...,"errors":null}`。代码未设置 HTTP 状态,默认 HTTP 200,业务码在 body 的 `code`(外层是否改写状态:待确认)。"未找到"类错误(如 `DNS domain not found`)在 F 类中也是 code 400,不是 404。
  - **A 类**(`actionutils.Fail(this, err)`):同时设置 HTTP 状态与 `code`:InvalidArgument 400、Unauthenticated 401、PermissionDenied 403、NotFound 404、AlreadyExists 409、ResourceExhausted 429、Unavailable 503、DeadlineExceeded 504,其他 500;Unknown 错误按消息关键字映射。`message` 形如 `{AdminCommon_ServerError}（<净化后的原因>）`;含 password=/secret/token 等字样的错误统一替换为"服务处理失败，请检查相关配置后重试。",权限类、不可用、未找到、已存在各有固定替换文案。下表"A:"表示走 A 类(原因由后端 RPC 决定,具体文案待确认),"F:"表示 F 类原文。
- ID 校验:大部分接口用 `validateMutationID(资源, id)`,`id<=0` 时 F 类返回 `invalid <资源> id`,资源名随接口不同(`cluster` / `DNS provider` / `DNS domain` / `node cluster` / `node`),下表逐行写出。
- CSRF:22 条路由中 POST 共 12 条,全部声明 `CSRF *actionutils.CSRF`;GET 共 10 条,均无 CSRF。令牌作为表单或查询参数 `csrfToken` 提交(不看请求头),与会话 sid 绑定,有效期 1800 秒。取令牌:`GET /api/admin/session` 的 `data.csrfToken`。校验失败返回 HTTP 403,**纯文本**(非 JSON)`表单已失效，请刷新页面后重试(001)`。
- 分页(`normalizePagination`):`page<1` 取 1;`size<1` 取 20;`size>100` 取 100。响应含 `items/total/page/size`(`DNSProviderDetail` 的列表键为 `domains`)。
- 时间格式:`dataUpdatedTime` 为 `Y-m-d H:i:s` 本地时间字符串,无数据时为空字符串。
- 操作日志:除 `DNSProviderSync` 外,所有 POST 都通过 `CreateLogInfo` 记录操作日志(创建类记录的是创建后的 ID,创建失败时记录 ID 为 0)。
- 敏感信息:服务商 API 参数(`params`)读取时以 `MaskParams:true` 向后端请求。脱敏规则在后端 RPC 内,**本次未核实**(待确认),文档不展示任何参数值。

| 模块 | 接口名称 | 方法 | 路径 | 请求参数 | 响应数据 | 主要错误 | 功能说明 | 认证/权限 |
|---|---|---|---|---|---|---|---|---|
| admin-04-dns | DNS 集群列表 | GET | `/api/admin/dns/clusters` | 查询参数:<br>`page` int64,默认 1<br>`size` int64,默认 20,上限 100<br>`keyword` string,可选,原样传给后端(不去空白以外的处理) | 200 `{"items":[DNSClusterItem],"total":int64,"page":int64,"size":int64}`(`DNSClusterItem` 见公共结构) | 错误:A:统计/列表 RPC 失败 | 分页列出启用的节点集群及其 DNS 配置概况。每个集群最多 4 路并发各查一次 DNS 配置;已配置域名时再查一次是否有未同步变更。单个集群的 DNS 查询失败不会让整个请求失败,而是写入该项的 `error` 字段(见异常说明)。 | 需登录+DNS权限 |
| admin-04-dns | DNS 集群表单 | GET | `/api/admin/dns/clusters/form` | 查询参数:<br>`clusterId` int64 必填,须 >0 | 200 `{"cluster":{"id":int64,"name":string,"dnsName":string,"domainId":int64,"providerId":int64,"providerType":string,"nodesAutoSync":bool,"serversAutoSync":bool,"cnameRecords":[string],"ttl":int32,"cnameAsDomain":bool,"includingLnNodes":bool},"providers":[{"id","name","type","typeName"}],"providerTypes":[{"code","name"}]}`<br>未配置 DNS 时 `domainId/providerId` 为 0,`providerType` 为空串 | 错误:F:`invalid cluster id`;F:`node cluster not found`;A | 读取某集群的 DNS 设置,附带全部启用的服务商和服务商类型,供表单下拉使用。`providers` 与 `providerTypes` 不分页。 | 需登录+DNS权限 |
| admin-04-dns | 服务商域名下拉 | GET | `/api/admin/dns/provider-domains` | 查询参数:<br>`providerId` int64 必填,须 >0 | 200 `{"domains":[{"id":int64,"name":string}]}` | 错误:F:`invalid DNS provider id`;A | 列出某服务商下可选的域名。只保留 `isOn` 为真、`isUp` 为真、未删除的域名,按 `name` 字符串升序。不分页。 | 需登录+DNS权限 |
| admin-04-dns | 更新集群 DNS 设置 | POST | `/api/admin/dns/clusters/update` | 表单参数:<br>`csrfToken` string 必填<br>`clusterId` int64 必填,须 >0<br>`dnsName` string,可选,集群在域名下的子域前缀<br>`domainId` int64,可选,0 表示不绑定<br>`nodesAutoSync` bool,默认 false<br>`serversAutoSync` bool,默认 false<br>`cnameRecords` string[],可选<br>`ttl` int32,可选,0 表示默认(后端取值规则待确认)<br>`cnameAsDomain` bool,默认 false<br>`includingLnNodes` bool,默认 false | 200 `{}` | 错误:F:`invalid cluster id`;A:RPC 失败(名称/域名合法性由后端校验,具体文案待确认);403 CSRF 失败(纯文本) | 整体覆盖集群的 DNS 设置。处理器**不校验** `dnsName`、`domainId`、`ttl`、`cnameRecords` 的格式与范围,全部交给后端。所有字段整体写入,漏传布尔即被改为 false。 | 需登录+DNS权限+CSRF |
| admin-04-dns | 同步集群 DNS | POST | `/api/admin/dns/clusters/sync` | 表单参数:<br>`csrfToken` string 必填<br>`clusterId` int64 必填,须 >0 | 200 `{}` | 错误:F:`invalid cluster id`;F:`DNS is not configured for this cluster`(未绑定域名);F:`DNS configuration contains issues that must be fixed first`(后端返回 shouldFix);F:后端返回的 `Error` 原文(同步失败且无 shouldFix 时);A | 先读集群 DNS 配置,再对其绑定域名执行 DNS 数据同步。F 类的 `Error` 原文来自后端,未经过 A 类的敏感信息净化(见异常说明)。 | 需登录+DNS权限+CSRF |
| admin-04-dns | DNS 服务商列表 | GET | `/api/admin/dns/providers` | 查询参数:<br>`page` int64,默认 1<br>`size` int64,默认 20,上限 100<br>`keyword` string,可选<br>`domain` string,可选,先转小写、去空白、去前缀 `www.` 后再查询<br>`providerType` string,可选,服务商类型代码 | 200 `{"items":[{"id":int64,"name":string,"type":string,"typeName":string,"minTTL":int32,"dataUpdatedTime":string,"countDomains":int64}],"providerTypes":[{"code","name","description"}],"total":int64,"page":int64,"size":int64}` | 错误:A | 分页列出启用的 DNS 服务商,并带每个服务商的域名数量与全部服务商类型。查询带上当前管理员 ID(`AdminId`),是否按管理员做可见性过滤由后端决定(待确认)。 | 需登录+DNS权限 |
| admin-04-dns | DNS 服务商表单 | GET | `/api/admin/dns/providers/form` | 查询参数:<br>`providerId` int64,可选;`<=0` 时视为新建表单 | 新建:200 `{"providerTypes":[{"code","name","description"}]}`<br>编辑:再加 `"provider":{"id":int64,"name":string,"type":string,"typeName":string,"minTTL":int32,"params":object}`,`params` 为后端脱敏后的 API 参数 JSON 对象(键集因服务商类型而异,脱敏规则待确认) | 错误:F:`DNS provider not found`;A:参数 JSON 解析失败或 RPC 失败 | 返回服务商类型列表;给了有效 `providerId` 时同时返回该服务商详情和脱敏参数。这里**不**调用 `validateMutationID`,`providerId<=0` 不报错。 | 需登录+DNS权限 |
| admin-04-dns | DNS 服务商详情 | GET | `/api/admin/dns/providers/detail` | 查询参数:<br>`providerId` int64 必填,须 >0<br>`page` int64,默认 1<br>`size` int64,默认 20,上限 100<br>`filter` string,可选:`deleted` 只看已删除域名;`down` 只看不可用域名;其他任何值(含空)表示不过滤。严格区分大小写,不做 Trim 以外的处理 | 200 `{"provider":{"id","name","type","typeName","minTTL","dataUpdatedTime"},"domains":[{"id":int64,"name":string,"isOn":bool,"isUp":bool,"isDeleted":bool,"dataUpdatedTime":string,"countRoutes":int,"countServerRecords":int64,"serversChanged":bool,"countNodeRecords":int64,"nodesChanged":bool,"countClusters":int64,"countAllNodes":int64,"countAllServers":int64}],"total":int64,"page":int64,"size":int64}` | 错误:F:`invalid DNS provider id`;F:`DNS provider not found`;A | 分页列出某服务商下的域名及解析统计。`countRoutes` 为线路条数。此接口不返回服务商 API 参数。 | 需登录+DNS权限 |
| admin-04-dns | DNS 域名表单 | GET | `/api/admin/dns/domains/form` | 查询参数:<br>`providerId` int64,新建时必填(须 >0)<br>`domainId` int64,可选;>0 时为编辑,此时 `providerId` 被忽略,改取域名所属的服务商 | 200 `{"provider":{"id","name","type","typeName"}}`;`domainId>0` 时再加 `"domain":{"id":int64,"name":string,"isOn":bool,"isDeleted":bool}` | 错误:F:`DNS domain not found`;F:`invalid DNS provider id`;F:`DNS provider not found`;A | 返回域名新建/编辑表单所需的服务商信息,编辑时附带域名本身。服务商查询带 `MaskParams`,但返回里不含参数。 | 需登录+DNS权限 |
| admin-04-dns | DNS 域名详情 | GET | `/api/admin/dns/domains/detail` | 查询参数:<br>`domainId` int64 必填,须 >0 | 200 `{"domain":{"id","name","isOn","isUp","isDeleted","dataUpdatedTime","countNodeRecords","nodesChanged","countServerRecords","serversChanged","countClusters","countAllNodes","countAllServers"},"provider":{"id","name","type","typeName"},"routes":[{"name","code"}],"clusters":[{"id":int64,"name":string,"dnsName":string,"defaultRoute":string,"isResolved":bool}],"nodes":[{"id","name","ipAddr","ipAddrId","clusterId","clusterName","dnsName","route":{"name","code"},"isResolved":bool}],"servers":[{"id","name","dnsName","clusterId","clusterName","isResolved":bool}]}`<br>服务商查不到时 `provider` 为 `{"id":<域名所属服务商ID>,"name":"-","type":"","typeName":"-"}` | 错误:F:`invalid DNS domain id`;F:`DNS domain not found`;A | 返回域名及使用该域名的集群、节点、站点,并逐项检查解析记录是否已存在(`isResolved`)。每个集群检查 A 与 AAAA 记录(命中即停),每个节点的每条线路检查一次 A 或 AAAA,每个站点检查一次 CNAME。未设置线路的节点按集群默认线路补一条名为"默认线路"的记录。检查次数随集群、节点、站点规模线性增长且串行执行(性能风险,见异常说明),任何一次 RPC 失败整个请求失败。节点只含已安装节点。 | 需登录+DNS权限 |
| admin-04-dns | 创建 DNS 服务商 | POST | `/api/admin/dns/providers/create` | 表单参数:<br>`csrfToken` string 必填<br>`name` string<br>`type` string,服务商类型代码<br>`minTTL` int32,可选<br>`paramsJSON` string(JSON 文本,按字节原样传给后端),各类型所需键不同 | 200 `{"providerId":int64}` | 错误:A:RPC 失败(名称/类型/参数合法性全由后端校验,具体文案待确认);403 CSRF 失败 | 新建一个 DNS 服务商。处理器**不做任何字段校验**,也不检查 `paramsJSON` 是不是合法 JSON。 | 需登录+DNS权限+CSRF |
| admin-04-dns | 更新 DNS 服务商 | POST | `/api/admin/dns/providers/update` | 表单参数:<br>`csrfToken` string 必填<br>`providerId` int64 必填,须 >0<br>`name` string<br>`minTTL` int32,可选<br>`paramsJSON` string(JSON 文本) | 200 `{}` | 错误:F:`invalid DNS provider id`;A | 更新服务商名称、最小 TTL 和 API 参数。不带 `type`,类型不可改。`paramsJSON` 为空时后端是保留旧值还是清空:待确认,处理器直接透传。 | 需登录+DNS权限+CSRF |
| admin-04-dns | 同步服务商域名 | POST | `/api/admin/dns/providers/sync` | 表单参数:<br>`csrfToken` string 必填<br>`providerId` int64 必填,须 >0 | 200 `{"hasChanges":bool}` | 错误:F:`invalid DNS provider id`;A:远端 DNS 服务商调用失败等 | 从服务商拉取最新域名列表并同步到本地,`hasChanges` 表示是否有变化。**不写操作日志**(处理器中没有 `CreateLogInfo`)。 | 需登录+DNS权限+CSRF |
| admin-04-dns | 删除 DNS 服务商 | POST | `/api/admin/dns/providers/delete` | 表单参数:<br>`csrfToken` string 必填<br>`providerId` int64 必填,须 >0 | 200 `{}` | 错误:F:`invalid DNS provider id`;A(有集群仍引用时后端是否拒绝:待确认) | 删除(停用)服务商。是否有引用检查由后端决定,处理器无前置检查。 | 需登录+DNS权限+CSRF |
| admin-04-dns | 创建 DNS 域名 | POST | `/api/admin/dns/domains/create` | 表单参数:<br>`csrfToken` string 必填<br>`providerId` int64<br>`name` string,域名 | 200 `{"domainId":int64}` | 错误:A:RPC 失败(格式/重复等校验在后端,文案待确认) | 在某服务商下新建域名记录。处理器**不校验** `providerId` 是否 >0,也不校验域名格式。 | 需登录+DNS权限+CSRF |
| admin-04-dns | 更新 DNS 域名 | POST | `/api/admin/dns/domains/update` | 表单参数:<br>`csrfToken` string 必填<br>`domainId` int64 必填,须 >0<br>`name` string<br>`isOn` bool,默认 false | 200 `{}` | 错误:F:`invalid DNS domain id`;A | 修改域名名称和启停。`isOn` 漏传即被置为停用。 | 需登录+DNS权限+CSRF |
| admin-04-dns | 同步域名 DNS 数据 | POST | `/api/admin/dns/domains/sync` | 表单参数:<br>`csrfToken` string 必填<br>`domainId` int64 必填,须 >0 | 200 `{}`<br>失败(后端返回 `IsOk=false`)时 F 类:`message` 为后端 `Error` 原文,`data` 为 `{"shouldFix":bool}`,`shouldFix=true` 表示需先修复节点 DNS 问题(见 `GET /dns/issues`)<br>其他 | 错误:F:`invalid DNS domain id`;A | 把该域名下集群/节点/站点的解析记录同步到服务商。请求固定带 `CheckNodeIssues=true`,会先检查节点问题。此接口的失败是 HTTP 200 加 body `code:400`(F 类默认),调用方必须看 `code` 而不是 HTTP 状态。 | 需登录+DNS权限+CSRF |
| admin-04-dns | 删除 DNS 域名 | POST | `/api/admin/dns/domains/delete` | 表单参数:<br>`csrfToken` string 必填<br>`domainId` int64 必填,须 >0 | 200 `{}` | 错误:F:`invalid DNS domain id`;A | 删除域名,软删除(`isDeleted`),可由"恢复"接口还原。 | 需登录+DNS权限+CSRF |
| admin-04-dns | 恢复 DNS 域名 | POST | `/api/admin/dns/domains/recover` | 表单参数:<br>`csrfToken` string 必填<br>`domainId` int64 必填,须 >0 | 200 `{}` | 错误:F:`invalid DNS domain id`;A | 恢复已删除的域名。 | 需登录+DNS权限+CSRF |
| admin-04-dns | 节点 DNS 修复表单 | GET | `/api/admin/dns/issues/node-form` | 查询参数:<br>`clusterId` int64 必填,须 >0<br>`nodeId` int64 必填,须 >0<br>`ipAddrId` int64,可选,不校验;用于定位节点的具体 IP | 200 `{"node":{"id":int64,"name":string,"ipAddr":string,"ipAddrId":int64,"clusterId":int64,"clusterDNSName":string,"domainId":int64,"domainName":string},"routes":[{"name":string,"code":string,"selected":bool}]}`<br>节点未绑定域名(`domainId<=0`)时 `routes` 为空数组 | 错误:F:`invalid node cluster id`;F:`invalid node id`;F:`node not found`;A | 返回修复节点 DNS 所需的节点信息与该域名的全部线路,`selected` 标记节点当前已选线路。 | 需登录+DNS权限 |
| admin-04-dns | 修复节点 DNS | POST | `/api/admin/dns/issues/node-update` | 表单参数:<br>`csrfToken` string 必填<br>`nodeId` int64 必填,须 >0<br>`ipAddr` string 必填,须是合法 IP(先去空白)<br>`ipAddrId` int64,可选<br>`domainId` int64,可选,线路只有在 `domainId>0` 时才生效<br>`routes` string[],可选,线路 code,空项忽略 | 200 `{}` | 错误:F:`invalid node id`;F:`invalid node IP address`;A | 更新节点的 DNS 解析 IP、绑定域名和线路。线路提交给后端时拼成 `code@domainId`。`domainId<=0` 时线路被全部丢弃,等于清空线路。 | 需登录+DNS权限+CSRF |
| admin-04-dns | DNS 问题列表 | GET | `/api/admin/dns/issues` | 无参数 | 200 `{"issues":[{"target":string,"targetId":int64,"type":string,"description":string,"params":<后端原样>,"mustFix":bool}]}`<br>`params` 的具体类型(pb 字段原样输出):待确认 | 错误:A | 返回当前所有 DNS 配置问题(例如节点缺少 IP 或线路),`mustFix` 为真时会阻止同步。不分页。 | 需登录+DNS权限 |


### 公共结构

### DNSClusterItem
`GET /dns/clusters` 的 `items` 元素:
- `id` int64
- `name` string
- `isOn` bool
- `dnsName` string:先取集群自身的 `DnsName`,读到 DNS 配置后改为 DNS 配置里的 `name`
- `configured` bool:有绑定域名且 `dnsName` 非空时为 true,否则 false
- `hasChanges` bool:仅在已绑定域名且 `dnsName` 非空时才会检查,否则恒为 false
- `nodesAutoSync` bool
- `serversAutoSync` bool
- `ttl` int:没有 DNS 配置时为 0
- `cnameRecords` string[],无时为 `[]`
- `domain?` `{"id":int64,"name":string}`,仅在绑定了域名时出现
- `provider?` `{"id":int64,"name":string,"type":string,"typeName":string}`,仅在存在服务商时出现
- `error?` string:该集群 DNS 查询或变更检查失败时出现,值为后端错误的 `err.Error()` 原文

### 异常与待确认

- **失败格式不统一**:大部分参数错误走 F 类(HTTP 状态不变、`code:400`),RPC 错误走 A 类(HTTP 状态随错误类型变化)。同一资源的"未找到"在 F 类是 400、在 A 类是 404。调用方应同时判断 HTTP 状态与 body 的 `code`。
- **原始错误未净化**:`GET /dns/clusters` 的 `items[].error`、`POST /dns/clusters/sync` 与 `POST /dns/domains/sync` 的 `message`(后端 `Error` 字段)都是原样输出的后端错误文本,没有经过 `actionutils.Fail` 的敏感信息替换。若后端错误里带有服务商返回的详情,可能出现在响应中(具体内容待确认)。
- **列表局部失败仍返回成功**:`GET /dns/clusters` 单个集群出错只在该项写 `error`,整体仍是 `code:200`。
- **创建类接口缺少前置校验**:`dns/providers/create`、`dns/domains/create` 和 `dns/clusters/update` 的字段、`providerId` 等几乎全部交给后端;`dns/providers/create` 与 `dns/providers/update` 不检查 `paramsJSON` 是否为合法 JSON。
- **布尔参数漏传即为 false**:`dns/clusters/update`(4 个布尔)和 `dns/domains/update`(`isOn`)是整体覆盖,调用方必须回传全部当前值。
- **`DNSDomainDetail` 查询量大**:每个集群、节点线路、站点各发起一次(集群最多两次)解析记录检查 RPC,全部串行,大规模集群下响应会很慢。
- **服务商参数脱敏**:`GET /dns/providers/form` 的 `provider.params` 依赖后端 `MaskParams` 脱敏,脱敏规则未核实,标记待确认。
- **无操作日志**:`POST /dns/providers/sync` 不写操作日志。
- **CSRF**:本模块没有发现缺 CSRF 的 POST。
- **待确认汇总**:HTTP 状态(F 类)是否被外层改写;空 `Data` 的序列化形态;`ttl=0`、`paramsJSON` 为空、删除有引用服务商/域名时的后端行为;`issues[].params` 类型;`ListEnabledDNSProviders` 是否按 `AdminId` 过滤;后端各类校验的错误文案。

<a id="m6"></a>

## 6. 管理端 · 证书与 ACME(管理端)

**模块约定**

> 路由总数 27,来源 EdgeAdmin/internal/web/actions/default/api/admin/init.go(路由注册,第 231-257 行,前缀 `/api/admin`;处理器见 `certificates_resource.go`/`acme_resource.go`)
说明:
- 权限:全部接口经 `helpers.NewUserMustAuth(AdminModuleCodeServer)`,须已登录且拥有 `server`(站点)模块权限。未登录 HTTP 401 `{"code":401,"message":"authentication required","data":null}`;无模块权限 HTTP 403 `{"code":403,"message":"permission denied","data":null}`;系统未初始化 409 `system setup is required`。
- 响应包络:成功 HTTP 200 `{"code":200,"message":"","data":{...}}`,下表"返回数据结构"只写 `data` 内字段,写"data 为空"表示无业务字段。失败为 `{"code":N,"message":"...","data":...}`。
  - 经 `actionutils.Fail` 的错误:HTTP 状态按 gRPC 码映射(InvalidArgument 400/Unauthenticated 401/PermissionDenied 403/NotFound 404/AlreadyExists 409/ResourceExhausted 429/Unavailable 503/DeadlineExceeded 504/其他 500),`message` 为"服务处理失败"前缀加脱敏后的原因。
  - 直接 `this.Fail("...")` 的校验错误:`message` 为下表所列原文,`code` 默认 400(HTTP 状态头是否同步:待确认)。
- 请求参数:参数名为 params 结构字段名首字母小写,值先去首尾空白。整型非法值按 0 处理;bool 接受 `on`/`true`/`yes`/`enabled`;切片字段用重复参数(如 `certIds=1&certIds=2`、`domains=a.com&domains=b.com`)。是否支持 JSON 请求体:待确认。
- CSRF:全部 POST 须带参数 `csrfToken`(令牌由 `GET /api/admin/session` 的 `data.csrfToken` 获得,绑定当前会话,有效期 30 分钟)。校验失败返回 HTTP 403,纯文本体 `表单已失效,请刷新页面后重试(001)`,不是 JSON 包络,且先于业务校验。下表各 POST 行不再重复列出 `csrfToken`。GET 不校验 CSRF。
- 分页:`page` 从 1 起,<1 按 1;`size` <1 按 20,>100 按 100。
- 文件下载:`certificates/download` 与 `certificates/private-key` 成功时直接返回 PEM 原文(不是 JSON),响应头 `Content-Type: application/x-pem-file`、`Content-Disposition: attachment`、`Cache-Control: no-store`、`X-Content-Type-Options: nosniff`;失败仍为 JSON 错误。
- 范围参数差异:证书列表用 `scope=owner`(属主用户),ACME 任务列表用 `scope=user`;两个接口的筛选值集合也不同(证书有 `ca`,ACME 无)。
- 审计日志:除各 GET 列表/表单/详情外,所有变更接口及证书下载、私钥下载均记录日志。
- 密钥类字段:ACME 账户的 EAB 密钥、证书私钥只写不读(除超管私钥下载外),文档不列值。

| 模块 | 接口名称 | 方法 | 路径 | 请求参数 | 响应数据 | 主要错误 | 功能说明 | 认证/权限 |
|---|---|---|---|---|---|---|---|---|
| admin-05-certificates-acme | 证书列表 | GET | `/api/admin/certificates` | `page` int 默认 1<br>`size` int 默认 20,最大 100<br>`keyword` string,可选,无长度限制<br>`filter` string,`all`(默认)/`available`/`expired`/`expiring7`/`expiring30`/`ca`<br>`scope` string,`platform`(默认)/`users`/`owner`<br>`userId` int64,`scope=owner` 时须 >0 | `items[Certificate]`(见公共结构);`total`;`page`;`size`;`filter`;`scope`;`userId`;`summary{all,available,expired,expiring7,expiring30,ca}`;`canDownloadPrivateKey` bool | 错误:`invalid certificate filter`;`invalid certificate scope`;`invalid certificate owner` | 分页查询 SSL 证书,附各筛选分类的计数。列表中 `hasPrivateKey` 恒为 false。 | 需登录(server 模块) |
| admin-05-certificates-acme | 证书表单 | GET | `/api/admin/certificates/form` | `certId` int64,≤0 表示新建 | `certificate`:新建时为 null,否则为 `Certificate`;`canDownloadPrivateKey` bool<br>响应头 `Cache-Control: no-store` | 见模块约定 | 取得编辑表单数据。`hasPrivateKey` 取自 `len(KeyData)>0`,但查询未带私钥数据,可能恒为 false:待确认。 | 需登录(server 模块) |
| admin-05-certificates-acme | 证书详情 | GET | `/api/admin/certificates/detail` | `certId` int64 必填,>0 | `certificate`:`Certificate`;`canDownloadPrivateKey` bool<br>响应头 `Cache-Control: no-store` | 错误:`invalid certificate id` | 查看证书详情,不返回证书原文与私钥。 | 需登录(server 模块) |
| admin-05-certificates-acme | 下载证书 | GET | `/api/admin/certificates/download` | `certId` int64 必填 | 成功:直接返回证书 PEM 原文,文件名 `certificate-<id>.pem`<br>错误(JSON):`certificate data is empty`;证书不存在/无效 ID 提示原文待确认 | 见模块约定 | 下载证书公钥链(不含私钥);记录日志。 | 需登录(server 模块) |
| admin-05-certificates-acme | 创建证书 | POST | `/api/admin/certificates/create` | `name` string<br>`description` string<br>`serverName` string<br>`isCA` bool<br>`isOn` bool<br>`certData` string(PEM)<br>`keyData` string(PEM)<br>均已去首尾空白,校验交给 RPC | `certificateId` int64 | 错误:RPC 校验原文,取值范围待确认 | 上传并创建 SSL 证书;记录日志。 | 需登录(server 模块)+CSRF |
| admin-05-certificates-acme | 更新证书 | POST | `/api/admin/certificates/update` | `certId` int64 必填<br>`name`/`description`/`serverName` string<br>`isCA`/`isOn` bool<br>`certData`/`keyData` string(PEM) | data 为空 | 错误:RPC 校验原文;证书/密钥留空时是否保留旧值取决于 RPC,待确认 | 修改证书信息,可替换证书内容;记录日志。 | 需登录(server 模块)+CSRF |
| admin-05-certificates-acme | 启停证书 | POST | `/api/admin/certificates/toggle` | `certId` int64 必填<br>`isOn` bool | data 为空 | 见模块约定 | 仅切换启用状态:沿用旧的 name/description/serverName/isCA 调用更新,不传证书数据;记录日志。 | 需登录(server 模块)+CSRF |
| admin-05-certificates-acme | 删除证书 | POST | `/api/admin/certificates/delete` | `certId` int64 必填 | data 为空 | 错误:被引用时的限制由 RPC 决定,待确认 | 删除证书;记录日志。 | 需登录(server 模块)+CSRF |
| admin-05-certificates-acme | 下载证书私钥 | POST | `/api/admin/certificates/private-key` | `certId` int64 必填 | 成功:直接返回私钥 PEM 原文,文件名 `private-key-<id>.pem`(响应头同公共约定)<br>错误(JSON):`仅超级管理员可以下载证书私钥`;`证书私钥为空` | 见模块约定 | 仅超级管理员可用。因含私钥,使用 POST 并强制 CSRF;记录日志。文档不示例私钥内容。 | 需登录(server 模块)+CSRF+超级管理员 |
| admin-05-certificates-acme | 重置证书 OCSP | POST | `/api/admin/certificates/ocsp/reset` | `certIds` int64 数组;处理器不校验是否为空,交给 RPC | data 为空 | 错误:RPC 错误原文 | 重置所选证书的 OCSP 状态,使其重新获取;记录日志。 | 需登录(server 模块)+CSRF |
| admin-05-certificates-acme | 忽略证书 OCSP 错误 | POST | `/api/admin/certificates/ocsp/ignore` | `certIds` int64 数组;不校验是否为空 | data 为空 | 错误:RPC 错误原文 | 忽略所选证书的 OCSP 错误;记录日志。 | 需登录(server 模块)+CSRF |
| admin-05-certificates-acme | ACME 任务列表 | GET | `/api/admin/acme/tasks` | `page` int 默认 1<br>`size` int 默认 20,最大 100<br>`keyword` string,≤200 且不含 CR/LF/NUL<br>`filter` string,空/`all`/`available`/`expired`/`expiring7`/`expiring30`(无 `ca`)<br>`scope` string,空/`platform`/`users`/`user`<br>`userId` int64,`scope=user` 时须 >0 | `items[ACMETask]`(见公共结构);`total`;`page`;`size`;`filter`;`scope`(空值回显 `platform`);`userId`;`summary{all,available,expired,expiring7,expiring30}` | 错误:`invalid ACME task keyword`;`invalid ACME task filter`;`platform user is required`;`invalid ACME task scope` | 分页查询证书申请任务。 | 需登录(server 模块) |
| admin-05-certificates-acme | ACME 任务表单 | GET | `/api/admin/acme/tasks/form` | `taskId` int64,≤0 表示新建<br>`userId` int64 | `task`(新建时为 null);`providers`;`acmeUsers`;`dnsProviders[{id,name,type,typeName}]`;`userId`<br>响应头 `Cache-Control: no-store` | 错误:`ACME task not found`<br>`providers`/`acmeUsers` 元素结构:待确认 | 取得任务编辑所需的任务、证书提供商、ACME 用户与 DNS 服务商选项。 | 需登录(server 模块) |
| admin-05-certificates-acme | 创建 ACME 任务 | POST | `/api/admin/acme/tasks/create` | `userId` int64<br>`authType` string,取值待确认<br>`acmeUserId` int64<br>`dnsProviderId` int64<br>`dnsDomain` string<br>`domains` string 数组<br>`autoRenew` bool<br>`authURL` string | `taskId` int64 | 错误:校验交给 RPC,原文待确认 | 创建证书申请任务;记录日志。 | 需登录(server 模块)+CSRF |
| admin-05-certificates-acme | 更新 ACME 任务 | POST | `/api/admin/acme/tasks/update` | `taskId` int64 必填<br>`acmeUserId` int64<br>`dnsProviderId` int64<br>`dnsDomain` string<br>`domains` string 数组<br>`autoRenew` bool<br>`authURL` string | data 为空 | 错误:RPC 校验原文,待确认 | 修改任务配置(不含 `authType`、`userId`);记录日志。 | 需登录(server 模块)+CSRF |
| admin-05-certificates-acme | 删除 ACME 任务 | POST | `/api/admin/acme/tasks/delete` | `taskId` int64 必填 | data 为空 | 见模块约定 | 删除任务;记录日志。 | 需登录(server 模块)+CSRF |
| admin-05-certificates-acme | 执行 ACME 任务 | POST | `/api/admin/acme/tasks/run` | `taskId` int64 必填 | `certificateId` int64 | 错误:ACME 返回的错误文本(`response.Error`)或 `ACME task failed` | 同步执行申请,耗时取决于 ACME 与 DNS 提供商。失败信息未经脱敏,会原样返回提供商错误;记录日志。 | 需登录(server 模块)+CSRF |
| admin-05-certificates-acme | ACME 账户列表 | GET | `/api/admin/acme/accounts` | `page` int 默认 1<br>`size` int 默认 20,最大 100 | `items[ACMEAccount]`(见公共结构);`total`;`page`;`size` | 见模块约定 | 分页列出证书提供商账户,不返回 EAB 密钥。 | 需登录(server 模块) |
| admin-05-certificates-acme | ACME 账户表单 | GET | `/api/admin/acme/accounts/form` | `accountId` int64,≤0 表示新建 | `account`(新建时为 null,否则 `ACMEAccount`);`providers[Provider]`<br>响应头 `Cache-Control: no-store` | 错误:`ACME provider account not found` | 取得账户编辑表单与可选提供商。 | 需登录(server 模块) |
| admin-05-certificates-acme | 创建 ACME 账户 | POST | `/api/admin/acme/accounts/create` | `name` string<br>`providerCode` string<br>`eabKid` string<br>`eabKey` string(只写) | `accountId` int64 | 错误:校验交给 RPC,取值范围待确认 | 创建提供商账户(含 EAB 凭据);记录日志。 | 需登录(server 模块)+CSRF |
| admin-05-certificates-acme | 更新 ACME 账户 | POST | `/api/admin/acme/accounts/update` | `accountId` int64 必填<br>`name` string<br>`eabKid` string<br>`eabKey` string(只写) | data 为空 | 错误:RPC 校验原文,待确认 | 修改账户;`eabKey` 留空是否保留旧值取决于 RPC,待确认;记录日志。 | 需登录(server 模块)+CSRF |
| admin-05-certificates-acme | 删除 ACME 账户 | POST | `/api/admin/acme/accounts/delete` | `accountId` int64 必填 | data 为空 | 见模块约定 | 删除账户;记录日志。 | 需登录(server 模块)+CSRF |
| admin-05-certificates-acme | ACME 用户列表 | GET | `/api/admin/acme/users` | `page` int 默认 1<br>`size` int 默认 20,最大 100<br>`userId` int64,可选;0 的含义(全部或仅平台)取决于 RPC,待确认 | `items[ACMEUser]`(见公共结构);`total`;`page`;`size`;`userId` | 见模块约定 | 分页列出 ACME 用户(邮箱账号)。 | 需登录(server 模块) |
| admin-05-certificates-acme | ACME 用户表单 | GET | `/api/admin/acme/users/form` | `acmeUserId` int64,≤0 表示新建<br>`userId` int64 | `acmeUser`(新建时为 null);`providers`;`accounts`(最多 500 个);`userId`<br>响应头 `Cache-Control: no-store` | 错误:`ACME user not found` | 取得 ACME 用户编辑所需的提供商与账户选项。 | 需登录(server 模块) |
| admin-05-certificates-acme | 创建 ACME 用户 | POST | `/api/admin/acme/users/create` | `userId` int64<br>`accountId` int64<br>`email` string<br>`providerCode` string<br>`description` string | `acmeUserId` int64 | 错误:校验交给 RPC,待确认 | 创建 ACME 用户;记录日志。 | 需登录(server 模块)+CSRF |
| admin-05-certificates-acme | 更新 ACME 用户 | POST | `/api/admin/acme/users/update` | `acmeUserId` int64 必填<br>`description` string | data 为空 | 见模块约定 | 仅可修改说明;记录日志。 | 需登录(server 模块)+CSRF |
| admin-05-certificates-acme | 删除 ACME 用户 | POST | `/api/admin/acme/users/delete` | `acmeUserId` int64 必填 | data 为空 | 见模块约定 | 删除 ACME 用户;记录日志。 | 需登录(server 模块)+CSRF |


### 公共结构

### Certificate
- `id` int64
- `name` string
- `description` string
- `serverName` string
- `isOn` bool
- `isCA` bool
- `isACME` bool
- `timeBeginAt` int64(Unix 秒)
- `timeEndAt` int64(Unix 秒)
- `timeBegin` string(`2006-01-02 15:04:05`,时间 ≤0 时为空串)
- `timeEnd` string(同上)
- `dnsNames` string 数组
- `commonNames` string 数组
- `ocspError` string
- `status` string:`disabled`/`notYetValid`/`expired`/`expiring`(30 天内到期)/`valid`
- `owner` `{id,name,username}` 或 null
- `serverCount` int
- `apiNodeCount` int
- `hasPrivateKey` bool(列表恒为 false;表单/详情见上文待确认)

### ACMETask
- `id` int64
- `isOn` bool
- `authType` string
- `dnsDomain` string
- `domains` string 数组
- `autoRenew` bool
- `authURL` string
- `createdTime` string(`2006-01-02 15:04:05`)
- `owner` `{id,username,name}`(无属主时 `{id:0,username:"",name:"平台"}`)
- `acmeUser?` 对象,键待确认
- `dnsProvider?` `{id,name,typeName}`
- `certificate?` `{id,name,isOn,timeBegin,timeEnd}`(日期格式 `2006-01-02`)
- `latestLog?` `{id,isOk,error,createdTime}`

### ACMEAccount
- `id` int64
- `name` string
- `isOn` bool
- `providerCode` string
- `eabKid` string
- `hasEABKey` bool(不返回密钥本身)
- `error` string
- `provider?` Provider

### Provider
- `name`、`code`、`description`、`apiURL`、`requireEAB`、`eabDescription`

### ACMEUser
- `id` int64
- `email` string
- `description` string
- `createdTime` string
- `providerCode` string
- `taskCount` int
- `provider?` Provider
- `account?` 对象,键待确认

### 异常与疑似缺陷
- 证书下载与私钥下载成功时返回 PEM 原文,不是 JSON 包络。
- `acme/tasks/run` 失败时原样返回 ACME/提供商错误文本,未经脱敏。
- 证书列表用 `scope=owner`,ACME 列表用 `scope=user`;`keyword` 长度限制仅 ACME 有。
- 证书表单/详情的 `hasPrivateKey` 可能因查询未带私钥数据而恒为 false,待确认。
- CSRF 失败返回纯文本 403,与其余 JSON 错误格式不一致。

<a id="m7"></a>

## 7. 管理端 · WAF/IP名单/缓存(管理端)

**模块约定**

> 路由总数 46,来源 EdgeAdmin/internal/web/actions/default/api/admin/init.go(258-303 行)
说明:
- 本模块全部路由权限模块为 `server`(`AdminModuleCodeServer`),经 `helpers.NewUserMustAuth` 校验。未登录且路径以 `/api/` 开头:HTTP 401 `{"code":401,"message":"authentication required","data":null}`;无模块权限:HTTP 403 `{"code":403,"message":"permission denied","data":null}`;系统未配置:409 `system setup is required`;IP 不被允许或恶意请求:403 空体;注入检测命中:403 `Denied By WAF`。登录态为 session 中的 `adminId`。
- 响应包络 `{"code":200,"message":"","data":{...}}`。表中"200 `{...}`"仅写 `data` 内的字段。无返回数据的写接口,`data` 为空对象/null(待确认具体形态)。
- 错误有两种来源,HTTP 状态不一致:`this.Fail(msg)` 为 HTTP 200 且 body `{"code":400,"message":msg}`(客户端必须看 body 的 `code`);`actionutils.Fail(err)`(RPC 错误)为真实 HTTP 状态,与 body `code` 同步(InvalidArgument 400 / Unauthenticated 401 / PermissionDenied 403 / NotFound 404 / AlreadyExists 409 / ResourceExhausted 429 / Unavailable 503 / DeadlineExceeded 504 / 其它 500),message 形如"服务器出了点小问题,请联系技术人员处理。(脱敏后的原因)"(前缀随语言变化)。表中"错误:"后只列处理器自身 `this.Fail` 的文案及主要的 RPC 校验文案;大多数文案为英文,少数为中文。
- CSRF:所有 POST 路由须携带参数 `csrfToken`(form 字段或 query),与 session 绑定,有效期 1800 秒,校验通过后不销毁可复用。失败:HTTP 403,纯文本(非 JSON)`表单已失效,请刷新页面后重试(001)`。获取:`GET /api/admin/session` 的 `data.csrfToken`,或 `GET /csrf/token` 的 `data.token`。GET 路由不校验 CSRF。
- 请求体仅支持 `application/x-www-form-urlencoded` 与 `multipart/form-data`,JSON 请求体不会被解析为参数。表中 `xxxJSON` 参数是"值为 JSON 字符串的表单字段"。数组参数用重复参数名(`ids=1&ids=2`)。bool 参数 `on/true/yes/enabled` 为真,其余为假。string 参数自动去首尾空白。
- 分页(列表接口):`page` 小于 1 取 1;`size` 小于 1 取 20,大于 100 取 100;响应回带 `page`/`size`/`total`。
- JSON 字符串参数(`policyJSON`/`rulesJSON`/`configJSON`/`captchaOptionsJSON` 等)统一用严格解码:只允许一个 JSON 对象/数组,不允许未知字段,超长报"请求内容为空或过大"/"请求只能包含一个 JSON 对象"。
- "EdgeAPI 校验"指 Admin 层仅透传、由 EdgeAPI RPC 层校验的规则,失败走 `actionutils.Fail`。
- 导出类接口(`ip-lists/export`、`waf-policies/export`)为 POST 且响应为文件流,不是 JSON 包络。
- 部分写接口缺少 id>0 或归属校验(见各行),已如实标注。

| 模块 | 接口名称 | 方法 | 路径 | 请求参数 | 响应数据 | 主要错误 | 功能说明 | 认证/权限 |
|---|---|---|---|---|---|---|---|---|
| admin-06-waf-ip-cache | 缓存任务列表 | GET | `/api/admin/cache-tasks` | 查询参数:<br>`page` int,默认 1<br>`size` int,默认 20,1-100<br>`type` string,可选,`purge`/`fetch`<br>`status` string,可选,`all`/`doing`/`success`/`failed`<br>`scope` string,可选,`all`/`platform`/`users`/`user`<br>`userId` int64,`scope=user` 时必须 >0,其它 scope 必须为 0 | 200 `{"items":[CacheTask],"total":int,"page":int,"size":int,"summary":{"all":int,"doing":int,"success":int,"failed":int}}`(`CacheTask` 见公共结构) | 错误:EdgeAPI 校验 scope/type/status/userId 不合法时返回 400 | 分页查询平台与用户的缓存刷新/预热任务记录,可按范围、类型、状态筛选,并返回各状态计数。 | 需登录,模块 server |
| admin-06-waf-ip-cache | 缓存任务详情 | GET | `/api/admin/cache-tasks/detail` | 查询参数:`taskId` int64 必填,须 >0 | 200 `{"task":CacheTask + "keys":[{"id":int64,"key":string,"keyType":string,"isDone":bool,"isDoing":bool,"status":string(failed/success/doing/pending),"errors":[{"nodeId":int64,"error":string}],"cluster":{"id":int64,"name":string}}]}` | 错误:"invalid cache task id";"cache task not found" | 读取单个任务及其每个 key 的执行状态与各节点错误。 | 需登录,模块 server |
| admin-06-waf-ip-cache | 校验缓存任务 | POST | `/api/admin/cache-tasks/validate` | 表单参数:<br>`type` string 必填,`purge`/`fetch`<br>`keyType` string 必填,`key`/`prefix`(`fetch` 仅支持 `key`)<br>`keys` string[] 必填,重复参数,1-1000 个<br>`csrfToken` 必填 | 200 `{"valid":bool,"failures":[{"key":string,"reason":string}]}`<br>`reason` 取值:requireKey/requireDomain/invalidPrefix/invalidKey/requireServer/requireUser/requireClusterId | 错误:EdgeAPI 校验(type/keyType 不合法,keys 数量超限,单 key 超 4096 字节或含控制字符/非 UTF-8) | 仅校验 key 是否可创建任务,不落库。 | 需登录+CSRF,模块 server |
| admin-06-waf-ip-cache | 创建缓存任务 | POST | `/api/admin/cache-tasks/create` | 表单参数:同"校验缓存任务"(`type`/`keyType`/`keys`/`csrfToken`) | 200 `{"taskId":int64,"countKeys":int}` | 错误:EdgeAPI 校验同上,调用者需 server 模块权限 | 创建缓存清理(purge)或预热(fetch)任务。 | 需登录+CSRF,模块 server |
| admin-06-waf-ip-cache | 重置缓存任务 | POST | `/api/admin/cache-tasks/reset` | 表单参数:`taskId` int64 必填,须 >0;`csrfToken` 必填 | 200 `{}`(无业务字段,待确认) | 错误:"invalid cache task id" | 将任务重置为待重新执行状态(具体重置语义由 EdgeAPI `ResetHTTPCacheTask` 决定,待确认)。 | 需登录+CSRF,模块 server |
| admin-06-waf-ip-cache | 删除缓存任务 | POST | `/api/admin/cache-tasks/delete` | 表单参数:`taskId` int64 必填,须 >0;`csrfToken` 必填 | 200 `{}` | 错误:"invalid cache task id" | 删除缓存任务记录。 | 需登录+CSRF,模块 server |
| admin-06-waf-ip-cache | 缓存策略列表 | GET | `/api/admin/cache-policies` | 查询参数:<br>`page` int,默认 1<br>`size` int,默认 20,1-100<br>`keyword` string,可选<br>`clusterId` int64,可选<br>`storageType` string,可选(合法取值待确认,推测 `file`/`memory`) | 200 `{"items":[{"id":int64,"name":string,"description":string,"isOn":bool,"type":string,"typeName":string,"capacity":{"count":int64,"unit":string},"maxSize":{"count":int64,"unit":string},"clusterCount":int,"ruleCount":int,"fileDir":string}],"total":int,"page":int,"size":int,"types":[{"code":string,"name":string,"description":string}],"clusters":[{"id":int64,"name":string}]}` | 见模块约定 | 分页查询缓存策略,同时返回存储类型和集群下拉数据。 | 需登录,模块 server |
| admin-06-waf-ip-cache | 缓存策略详情 | GET | `/api/admin/cache-policies/detail` | 查询参数:`policyId` int64 必填 | 200 `{"policy":{"id","name","description","isOn","type","typeName","capacity":{count,unit},"maxSize":{count,unit},"fetchTimeout":{count,unit},"syncCompressionCache","fileDir","fileMemoryCapacity","fileMinFreeSize","fileOpenFileCacheMax","fileEnableSendfile","fileEnableMMAP","enableIncompletePartialContent","fileSubDirCount"},"clusters":[{"id","name"}],"rules":[siteWebCacheRule],"advancedRuleCount":int,"rawRulesJSON":string,"types":[{code,name,description}]}`<br>`siteWebCacheRule` 字段:待确认 | 错误:策略不存在时由 RPC/处理器报错(文案待确认) | 读取缓存策略完整配置、使用它的集群、结构化规则以及原始规则 JSON。 | 需登录,模块 server |
| admin-06-waf-ip-cache | 创建缓存策略 | POST | `/api/admin/cache-policies/create` | 表单参数:`policyJSON` string(JSON 对象)必填;`csrfToken`<br>`policyJSON` 字段:`name` string,`description` string,`isOn` bool,`type` string(`file`/`memory`,否则报"缓存存储类型无效"),`capacity`/`maxSize`/`fetchTimeout` `{count:int,unit:string}`,`syncCompressionCache` bool,`fileDir` string,`fileMemoryCapacity` 对象,`fileMinFreeSize` 对象,`fileOpenFileCacheMax` int,`fileEnableSendfile` bool,`fileEnableMMAP` bool,`enableIncompletePartialContent` bool<br>各字段数值范围由 EdgeAPI 校验(待确认) | 200 `{"policyId":int64}` | 错误:"缓存存储类型无效";JSON 解码错误 | 创建缓存策略。 | 需登录+CSRF,模块 server |
| admin-06-waf-ip-cache | 更新缓存策略 | POST | `/api/admin/cache-policies/update` | 表单参数:`policyId` int64 必填;`policyJSON` 同创建;`csrfToken` | 200 `{}` | 错误:同创建 | 整体覆盖更新(漏传字段按零值写入);子目录配置沿用旧值。 | 需登录+CSRF,模块 server |
| admin-06-waf-ip-cache | 保存缓存规则(结构化) | POST | `/api/admin/cache-policies/rules` | 表单参数:`policyId` int64 必填;`rulesJSON` string(JSON 数组)必填;`csrfToken`<br>数组元素为规则对象,可含 `sourceIndex` int(≥0 表示引用旧规则的下标) | 200 `{}` | 错误:"缓存规则索引无效";"不能通过结构化编辑器修改高级缓存规则";"缓存规则不能超过 N 条"(N 待确认) | 用结构化编辑器保存缓存规则;高级规则只能保留不可改。 | 需登录+CSRF,模块 server |
| admin-06-waf-ip-cache | 保存缓存规则(原始 JSON) | POST | `/api/admin/cache-policies/rules/raw` | 表单参数:`policyId` int64 必填;`rawRulesJSON` string 必填,非空且 ≤512KB;`csrfToken` | 200 `{}` | 错误:"缓存规则 JSON 为空或过大";内容合法性仅由 EdgeAPI 校验 | 直接用原始 JSON 覆盖缓存规则,Admin 层不校验内容。 | 需登录+CSRF,模块 server |
| admin-06-waf-ip-cache | 删除缓存策略 | POST | `/api/admin/cache-policies/delete` | 表单参数:`policyId` int64 必填;`csrfToken` | 200 `{}` | 错误:由 RPC 返回(例如策略仍被引用,文案待确认) | 删除缓存策略。 | 需登录+CSRF,模块 server |
| admin-06-waf-ip-cache | 缓存策略节点操作 | POST | `/api/admin/cache-policies/operate` | 表单参数:<br>`policyId` int64 必填<br>`clusterId` int64 必填,须是使用该策略的集群<br>`operation` string 必填,`stat`/`clean`/`read`/`write`<br>`key` string,`read`/`write` 用,≤2048,不含 `\x00` `\r` `\n`<br>`value` string,`write` 用,≤64KiB<br>`csrfToken` | 200 `{"isAllOk":bool,"results":[{"nodeId":int64,"nodeName":string,"isOk":bool,"message":string}]}` | 错误:"请选择使用当前策略的节点集群";"节点集群未使用当前缓存策略";"缓存 Key 格式无效";"测试缓存内容不能超过 64 KiB";"不支持的缓存操作" | 向集群各节点下发缓存统计/清理/读/写。超时 10 秒,`clean` 60 秒,`write` 缓存存活 3600 秒。`clean` 会清空该策略在集群所有节点的缓存,影响大。节点全部失败时仍返回 code 200,须看 `isAllOk`。 | 需登录+CSRF,模块 server |
| admin-06-waf-ip-cache | IP名单列表 | GET | `/api/admin/ip-lists` | 查询参数:<br>`page` int,默认 1<br>`size` int,默认 20,1-100<br>`type` string,可选,`black`/`white`/`grey`<br>`keyword` string,可选<br>固定仅查询公共名单(`IsPublic=true`) | 200 `{"items":[{"id":int64,"isOn":bool,"type":string,"name":string,"code":string,"description":string,"isPublic":bool,"isGlobal":bool,"itemCount":int64}],"total":int,"page":int,"size":int}` | 见模块约定 | 分页查询公共 IP 名单。 | 需登录,模块 server |
| admin-06-waf-ip-cache | IP名单详情 | GET | `/api/admin/ip-lists/detail` | 查询参数:<br>`listId` int64 必填<br>`page` int,默认 1<br>`size` int,默认 20,1-100<br>`keyword` string,可选<br>`eventLevel` string,可选,`debug`/`notice`/`warning`/`error`/`critical`/`fatal` | 200 `{"list":{名单字段同列表项},"items":[IPItem],"total":int,"page":int,"size":int,"eventLevels":[{"code":string,"name":string}]}`(`IPItem` 见公共结构) | 错误:名单不存在/无权限(文案待确认) | 读取名单及其分页的 IP 项。 | 需登录,模块 server |
| admin-06-waf-ip-cache | IP检查 | GET | `/api/admin/ip-lists/check` | 查询参数:`listId` int64 必填;`ip` string 必填 | 200 `{"isOk":bool,"error":string,"isFound":bool,"isAllowed":bool,"item":IPItem(可选)}` | 见模块约定 | 检查某 IP 是否命中该名单。 | 需登录,模块 server |
| admin-06-waf-ip-cache | 创建IP名单 | POST | `/api/admin/ip-lists/create` | 表单参数:<br>`name` string 必填,≤100<br>`code` string,可选,≤100,格式合法;type 为 white/black/grey 时自动追加随机后缀<br>`type` string 必填,`black`/`white`/`grey`<br>`description` string,可选,≤200<br>`isGlobal` bool,可选<br>`csrfToken` | 200 `{"listId":int64}` | 错误:EdgeAPI 校验(名称为空/超长、类型无效、编码格式无效、重复等) | 创建公共 IP 名单。 | 需登录+CSRF,模块 server |
| admin-06-waf-ip-cache | 更新IP名单 | POST | `/api/admin/ip-lists/update` | 表单参数:`listId` int64 必填;`name` string;`code` string;`description` string;`csrfToken`<br>校验规则同创建 | 200 `{}` | 见模块约定 | 更新名单名称、编码、描述。 | 需登录+CSRF,模块 server |
| admin-06-waf-ip-cache | 删除IP名单 | POST | `/api/admin/ip-lists/delete` | 表单参数:`listId` int64 必填;`csrfToken` | 200 `{}` | 错误:由 RPC 返回(仍被引用时的文案待确认) | 删除 IP 名单。 | 需登录+CSRF,模块 server |
| admin-06-waf-ip-cache | 导入IP项 | POST | `/api/admin/ip-lists/import` | multipart 表单:<br>`listId` int64 必填<br>`file` 文件必填,`.csv`/`.txt`,≤5MiB,≤1000 条<br>`csrfToken`<br>列顺序:`value,expiredAt,type,eventLevel,reason`;`type` 取 `ipv4`/`ipv6`/`all`/`ip`;`eventLevel` 默认 `critical`;`expiredAt` 可为 Unix 时间戳/RFC3339/`2006-01-02 15:04:05`/`2006-01-02` | 200 `{"itemIds":[int64],"count":int}` | 错误:文件类型/大小/条数超限;记录格式错误 | 批量导入 IP 项。注意:代码对表头 `IP/IP段` 的跳过判断因先转小写而永远不成立,该表头行会被当数据解析(疑似缺陷)。 | 需登录+CSRF,模块 server |
| admin-06-waf-ip-cache | 导出IP项 | POST | `/api/admin/ip-lists/export` | 表单参数:`listId` int64 必填;`csrfToken` | 成功:CSV 文件流,不是 JSON 包络。UTF-8 BOM;表头 `value,expiredAt,type,eventLevel,reason`;文件名 `ip-list-{id}.csv`;`Cache-Control: no-store`;以 `=` `+` `-` `@` 开头的单元格前缀 `'` 防公式注入 | 错误:JSON 包络 | 导出名单全部 IP 项为 CSV。 | 需登录+CSRF,模块 server |
| admin-06-waf-ip-cache | 批量添加IP项 | POST | `/api/admin/ip-lists/items/create` | 表单参数:<br>`listId` int64 必填<br>`values` string[] 必填,重复参数,去重后 ≤1000<br>`type` string,`ipv4`/`ipv6`/`all`;`all` 时值固定为 0.0.0.0(待确认完整形态)<br>`expiredAt` int64,可选,Unix 秒,0 表示永久(待确认)<br>`reason` string,可选<br>`eventLevel` string,可选,取值同上<br>`csrfToken` | 200 `{"itemIds":[int64],"count":int}` | 错误:EdgeAPI 校验(IP 格式、数量超限) | 向名单批量添加 IP/IP 段。不校验名单是否属于公共名单,仅依赖 RPC 层权限(待确认)。 | 需登录+CSRF,模块 server |
| admin-06-waf-ip-cache | 更新IP项 | POST | `/api/admin/ip-lists/items/update` | 表单参数:`itemId` int64 必填;`value` string;`type` string;`expiredAt` int64;`reason` string;`eventLevel` string;`csrfToken` | 200 `{}` | 见模块约定 | 更新单个 IP 项。不校验归属,仅依赖 RPC 层(待确认)。 | 需登录+CSRF,模块 server |
| admin-06-waf-ip-cache | 删除IP项 | POST | `/api/admin/ip-lists/items/delete` | 表单参数:`itemIds` int64[] 必填,重复参数,1-1000 个;`csrfToken` | 200 `{}` | 错误:数量为 0 或超限 | 批量删除 IP 项。不校验归属,仅依赖 RPC 层(待确认)。 | 需登录+CSRF,模块 server |
| admin-06-waf-ip-cache | WAF策略列表 | GET | `/api/admin/waf-policies` | 查询参数:<br>`page` int,默认 1<br>`size` int,默认 20,1-100<br>`keyword` string,可选<br>`clusterId` int64,可选 | 200 `{"items":[{"id":int64,"name":string,"description":string,"isOn":bool,"mode":string,"modeName":string,"useLocalFirewall":bool,"inboundCount":int,"outboundCount":int,"clusterCount":int}],"total":int,"page":int,"size":int,"modes":[{"code":string,"name":string,"description":string}]}`<br>`mode` 取值 `defend`/`observe`/`bypass` | 见模块约定 | 分页查询共享 WAF 策略。 | 需登录,模块 server |
| admin-06-waf-ip-cache | WAF策略详情 | GET | `/api/admin/waf-policies/detail` | 查询参数:`policyId` int64 必填 | 200 `{"policy":HTTPFirewallPolicy,"clusters":[{"id","name"}],"upgradeCount":int,"modes":[{code,name,description}],"connectors":[{code,name}],"actions":[{"code","name","description","category"}],"operators":[{"code","name","description","dataType","caseInsensitive"}],"checkpoints":[{"prefix","name","description","isRequest","hasParams","dataType"}]}`<br>`policy` 键:id,isOn,serverId,name,description,inbound,outbound,blockOptions,captchaOptions,pageOptions,get302Options,post307Options,jsCookieOptions,mode,useLocalFirewall,synFlood,log,maxRequestBodySize,denyCountryHTML,denyProvinceHTML;`captchaOptions.secretKey` 被置空并附 `secretKeyConfigured`(bool)<br>inbound/outbound 等嵌套对象的完整字段:待确认 | 错误:"shared WAF policy not found";"too many shared WAF policies to verify ownership" | 读取策略完整配置与规则编辑器所需的枚举数据。 | 需登录,模块 server |
| admin-06-waf-ip-cache | WAF策略IP检查 | GET | `/api/admin/waf-policies/check` | 查询参数:`policyId` int64 必填;`ip` string 必填 | 200 `{"isOk":bool,"error":string,"isFound":bool,"isAllowed":bool,"ipList":{"id","name","type"},"ipItem":{"id","value","reason","eventLevel"},"country":{"id","name"},"province":{"id","name"}}`(后四项命中时才有) | 见模块约定 | 检查 IP 在该策略下是否被名单/地区规则命中。不做归属校验(疑似缺陷)。 | 需登录,模块 server |
| admin-06-waf-ip-cache | 创建WAF策略 | POST | `/api/admin/waf-policies/create` | 表单参数:`name` string 必填;`description` string,可选;`isOn` bool;`useTemplate` bool;`csrfToken` | 200 `{"policyId":int64}` | 错误:名称为空等由 EdgeAPI 校验 | 创建 WAF 策略,可选基于默认模板。 | 需登录+CSRF,模块 server |
| admin-06-waf-ip-cache | 更新WAF策略 | POST | `/api/admin/waf-policies/update` | 表单参数:<br>`policyId` int64 必填<br>`name` string<br>`description` string<br>`isOn` bool<br>`mode` string,`defend`/`observe`/`bypass`<br>`useLocalFirewall` bool<br>`maxRequestBodySize` int64<br>`captchaOptionsJSON` string,≤64KB,空则保留原值;Cap 类型传空 `secretKey` 时保留旧密钥;非 Cap 类型归一为默认类型<br>`csrfToken` | 200 `{}` | 错误:"shared WAF policy not found" 等 | 整体覆盖基础项,漏传 isOn/useLocalFirewall/maxRequestBodySize/mode 会被写成零值或空值;其余选项沿用旧值。 | 需登录+CSRF,模块 server |
| admin-06-waf-ip-cache | 删除WAF策略 | POST | `/api/admin/waf-policies/delete` | 表单参数:`policyId` int64 必填;`csrfToken` | 200 `{}` | 错误:由 RPC 返回 | 删除 WAF 策略。未经过 `findAdminWAFPolicy`,无 id>0 校验与共享策略归属校验(疑似缺陷)。 | 需登录+CSRF,模块 server |
| admin-06-waf-ip-cache | 创建规则分组 | POST | `/api/admin/waf-policies/groups/create` | 表单参数:`policyId` int64 必填;`direction` string 必填,`inbound`/`outbound`;`name` string 必填;`code` string;`description` string;`isOn` bool;`csrfToken` | 200 `{"groupId":int64}` | 错误:"WAF rule group name or code already exists" | 在策略的入站/出站方向新增规则分组。 | 需登录+CSRF,模块 server |
| admin-06-waf-ip-cache | 更新规则分组 | POST | `/api/admin/waf-policies/groups/update` | 表单参数:`policyId` int64;`groupId` int64;`name` string;`code` string;`description` string;`isOn` bool;`csrfToken` | 200 `{}` | 错误:分组不存在、重名 | 更新规则分组基础信息。 | 需登录+CSRF,模块 server |
| admin-06-waf-ip-cache | 删除规则分组 | POST | `/api/admin/waf-policies/groups/delete` | 表单参数:`policyId` int64;`groupId` int64;`csrfToken` | 200 `{}` | 见模块约定 | 删除规则分组。 | 需登录+CSRF,模块 server |
| admin-06-waf-ip-cache | 规则分组排序 | POST | `/api/admin/waf-policies/groups/sort` | 表单参数:`policyId` int64;`direction` string,`inbound`/`outbound`;`groupIds` int64[],必须覆盖该方向全部分组且不重复;`csrfToken` | 200 `{}` | 错误:分组 ID 不完整/重复 | 重排分组顺序。重建引用时会把所有分组引用强制设为启用(isOn=true)。 | 需登录+CSRF,模块 server |
| admin-06-waf-ip-cache | 规则分组灰度 | POST | `/api/admin/waf-policies/groups/rollout` | 表单参数:`policyId` int64;`direction` string;`groupId` int64;`mode` string(`defend` 视为空);`enforcePercent` int;`csrfToken`<br>`mode`/`enforcePercent` 的合法取值由 `ValidateGroupRefRollout` 决定(待确认) | 200 `{}` | 错误:"invalid WAF rule group rollout settings" | 设置分组的灰度发布模式与生效百分比。 | 需登录+CSRF,模块 server |
| admin-06-waf-ip-cache | 创建规则集 | POST | `/api/admin/waf-policies/sets/create` | 表单参数:`policyId` int64;`groupId` int64;`configJSON` string(JSON 对象),≤4MiB,不允许未知字段;`csrfToken`<br>规则集对象字段:待确认 | 200 `{}`(不返回新规则集 ID) | 错误:JSON 解码错误 | 在分组下新增规则集。 | 需登录+CSRF,模块 server |
| admin-06-waf-ip-cache | 更新规则集 | POST | `/api/admin/waf-policies/sets/update` | 表单参数:`policyId` int64;`groupId` int64;`setId` int64;`configJSON` 同上;`csrfToken` | 200 `{}` | 见模块约定 | 更新规则集。 | 需登录+CSRF,模块 server |
| admin-06-waf-ip-cache | 启停规则集 | POST | `/api/admin/waf-policies/sets/toggle` | 表单参数:`policyId` int64;`groupId` int64;`setId` int64;`isOn` bool;`csrfToken` | 200 `{}` | 见模块约定 | 启用或停用规则集。 | 需登录+CSRF,模块 server |
| admin-06-waf-ip-cache | 删除规则集 | POST | `/api/admin/waf-policies/sets/delete` | 表单参数:`policyId` int64;`groupId` int64;`setId` int64;`csrfToken` | 200 `{}` | 见模块约定 | 删除规则集。 | 需登录+CSRF,模块 server |
| admin-06-waf-ip-cache | 规则集排序 | POST | `/api/admin/waf-policies/sets/sort` | 表单参数:`policyId` int64;`groupId` int64;`setIds` int64[];`csrfToken` | 200 `{}` | 见模块约定 | 重排分组内的规则集顺序。 | 需登录+CSRF,模块 server |
| admin-06-waf-ip-cache | 导入WAF策略 | POST | `/api/admin/waf-policies/import` | multipart 表单:`policyId` int64;`file` 文件必填,`.json`,≤4MiB;`csrfToken` | 200 `{}` | 错误:文件类型/大小、JSON 无效 | 导入策略配置到指定策略。是覆盖还是合并:待确认。 | 需登录+CSRF,模块 server |
| admin-06-waf-ip-cache | 导入规则包 | POST | `/api/admin/waf-policies/rule-pack` | multipart 表单:`policyId` int64;`file` 文件必填,`.json`,≤8MiB;`csrfToken` | 200 `{"result":{"added":...,"updated":...,"unchanged":...,"disabled":...}}`(各项元素类型待确认,`disabled` 为数量) | 错误:"no trusted WAF rule signing key is installed in ..."(无信任公钥);验签失败 | 验签后将规则包合并进策略;无变化时不导入、不记操作日志。 | 需登录+CSRF,模块 server |
| admin-06-waf-ip-cache | 禁用过期补丁 | POST | `/api/admin/waf-policies/expire-patches` | 表单参数:`policyId` int64;`csrfToken` | 200 `{"disabled":int}` | 见模块约定 | 禁用已过期的规则分组(只禁用不删除)。 | 需登录+CSRF,模块 server |
| admin-06-waf-ip-cache | 导出WAF策略 | POST | `/api/admin/waf-policies/export` | 表单参数:`policyId` int64;`csrfToken` | 成功:JSON 文件流(文件名 `waf-policy-{id}.json`),不是 JSON 包络 | 错误:包络 | 导出策略配置。 | 需登录+CSRF,模块 server |
| admin-06-waf-ip-cache | 升级WAF策略 | POST | `/api/admin/waf-policies/upgrade` | 表单参数:`policyId` int64;`csrfToken` | 200 `{}` | 见模块约定 | 将模板中新增的分组/规则集合并到策略。 | 需登录+CSRF,模块 server |


### 公共结构

### CacheTask
- `id` int64
- `type` string:`purge`/`fetch`
- `keyType` string:`key`/`prefix`
- `createdAt` / `doneAt` 数值,时间戳,精确类型与单位:待确认
- `isDone` bool,`isOk` bool
- `description` string
- `status` string:`doing`/`success`/`failed`
- `owner` `{"id":int64,"username":string,"fullname":string}`

### IPItem
`id`,`value`,`ipFrom`,`ipTo`,`createdAt`,`expiredAt`,`reason`,`type`,`eventLevel`,`eventLevelName`,`sourceNodeId`,`sourceServerId`,`sourcePolicyId`,`sourceGroupId`,`sourceSetId`,`region`,`provider`,`isExpired`;可选 `sourceServer`/`sourcePolicy`/`sourceGroup`/`sourceSet`,各为 `{"id","name"}`。时间字段的精确类型待确认。

<a id="m8"></a>

## 8. 管理端 · 日志与指标(管理端)

**模块约定**

> 路由总数 26,来源 EdgeAdmin/internal/web/actions/default/api/api/init.go(18-20 行)、EdgeAdmin/internal/web/actions/default/api/admin/init.go(138-139、221-230、315-322、392-394 行)
说明:
- 权限模块按路由分四种:`/api/metrics/*` 为 `common`(只需登录);审计日志为 `log`;访问日志、访问日志策略/设置、指标项/图表为 `server`;指标集群设置为 `node`。认证拦截 `helpers.NewUserMustAuth`:未登录且路径以 `/api/` 开头 HTTP 401 `{"code":401,"message":"authentication required","data":null}`;无模块权限 HTTP 403 `{"code":403,"message":"permission denied","data":null}`;系统未配置 409 `system setup is required`。登录态为 session 中的 `adminId`。
- `/api/admin/*` 响应包络 `{"code":200,"message":"","data":{...}}`,表中"200 `{...}`"仅写 `data` 内字段。错误有两种来源,HTTP 状态不一致:`this.Fail(msg)` 为 HTTP 200 且 body `{"code":400,"message":msg}`(须看 body 的 `code`);`actionutils.Fail(err)`(RPC 错误)为真实 HTTP 状态并与 body `code` 同步(InvalidArgument 400 / Unauthenticated 401 / PermissionDenied 403 / NotFound 404 / AlreadyExists 409 / ResourceExhausted 429 / Unavailable 503 / DeadlineExceeded 504 / 其它 500),message 形如"服务器出了点小问题,请联系技术人员处理。(脱敏后的原因)"。
- CSRF:`/api/admin/*` 的 POST 路由须携带 `csrfToken`(form 字段或 query),与 session 绑定,有效期 1800 秒,校验后可复用。失败:HTTP 403,纯文本(非 JSON)`表单已失效,请刷新页面后重试(001)`。获取:`GET /api/admin/session` 的 `data.csrfToken`,或 `GET /csrf/token` 的 `data.token`。GET 不校验。
- `/api/metrics/*` 的特殊约定:不校验 CSRF(代码注释:application/json 跨站需要 CORS 预检,故有意不校验);两个 POST 请求体为 JSON(`Content-Type: application/json`),上限 64KB,拒绝未知字段与尾随数据,否则 400 `请求格式无效`;每管理员每分钟最多 60 次,超限 HTTP 429 加 `Retry-After: 60`,message `上报过于频繁,请稍后再试`;成功响应 HTTP 204 无响应体;错误格式为 `{"code":status,"message":msg}`,没有 `data` 字段,与 TeaGo 包络不同。
- `/api/admin/*` 请求体仅支持 form-urlencoded 与 multipart,JSON 请求体不会被解析为参数。`xxxJSON` 参数是"值为 JSON 字符串的表单字段",用严格解码(只允许单个 JSON 对象,不允许未知字段,≤256KB),错误文案"请求内容为空或过大"/"请求只能包含一个 JSON 对象"。数组参数用重复参数名。bool 参数 `on/true/yes/enabled` 为真。string 参数自动去首尾空白。
- 分页(列表接口 `page`/`size`):`page` 小于 1 取 1;`size` 小于 1 取 20,大于 100 取 100;响应回带 `page`/`size`/`total`。访问日志是游标分页,不用此规则。
- 访问日志策略的创建/修改/删除/测试及队列设置的修改均要求超级管理员,否则报"only a super administrator can change access log storage"(设置接口文案待确认是否相同)。
- 部分指标写接口在 Admin 层没有 id>0 校验,只依赖 RPC 层,已在对应行标注。

| 模块 | 接口名称 | 方法 | 路径 | 请求参数 | 响应数据 | 主要错误 | 功能说明 | 认证/权限 |
|---|---|---|---|---|---|---|---|---|
| admin-07-logs-metrics | 上报Web Vitals | POST | `/api/metrics/web-vitals` | JSON 请求体(≤64KB):<br>`url` string 必填<br>`userAgent` string,可选<br>`timestamp` 数值,可选<br>`metrics` 数组必填,1-50 个,元素 `{name,value,id,delta,rating,navigationType,url,userAgent,timestamp}`;`name` 仅接受 LCP/FCP/CLS/INP/TTFB/FID,其它名称静默丢弃 | 204 无响应体(持久化失败只记日志,仍返回 204) | 错误:`{"code":400,"message":"metrics array is empty"/"too many metrics"/"url is required"/"请求格式无效"}`;429 限流 | 上报前端 Web Vitals 指标。入库时 `UserId` 字段实际保存的是 adminId。不校验 CSRF。 | 需登录,模块 common,限流 60 次/分钟 |
| admin-07-logs-metrics | 上报性能时间线 | POST | `/api/metrics/performance-timeline` | JSON 请求体(≤64KB):<br>`navigation` 对象必填:`url` 必填,`userAgent`,`timestamp`,`dnsLookup`,`tcpConnect`,`sslHandshake`,`requestTime`,`responseTime`,`ttfb`,`domProcessing`,`domInteractive`,`domComplete`,`loadComplete`,`totalTime`,`navigationType`<br>`resources` 数组,可选,≤500 个,元素 `{url,pageUrl,userAgent,timestamp,name,initiatorType,duration,transferSize,encodedBodySize,decodedBodySize,cacheStatus}`,仅 `duration` >100ms 的资源入库 | 204 无响应体 | 错误:`{"code":400,"message":"navigation.url is required"/"too many resources"/"请求格式无效"}`;429 限流 | 上报页面导航与慢资源时间线。不校验 CSRF。 | 需登录,模块 common,限流 60 次/分钟 |
| admin-07-logs-metrics | 查询前端性能指标 | GET | `/api/metrics/query` | 查询参数:<br>`metricType` string 必填,`web_vitals`/`performance_timeline`,否则 400 `invalid metricType`<br>`range` string,可选,`24h`/`7d`/`30d`,未知值按 `24h`<br>`page` string,可选,≤512 字符,超长报 `page is too long`<br>`limit` int32,≤0 取 1000,>5000 取 5000 | 200 `{"records":[{"page":string,"metricType":string,"keys":待确认,"value":number,"timestamp":待确认}]}`(此处是否带 `{code,data}` 包络待确认,错误是 `{code,message}`) | 错误:400 `invalid metricType`/`page is too long` | 查询已上报的前端性能记录。 | 需登录,模块 common |
| admin-07-logs-metrics | 审计日志列表 | GET | `/api/admin/audit-logs` | 查询参数:<br>`page` int,默认 1<br>`size` int,默认 20,1-100<br>`dayFrom` string,可选(格式待确认)<br>`dayTo` string,可选(格式待确认)<br>`keyword` string,可选<br>`userType` string,可选,`admin`/`user`/`provider`/`system`<br>`level` string,可选 | 200 `{"items":[{"id":int64,"level":string,"action":string,"adminId":int64,"userId":int64,"providerId":int64,"userType":string,"userName":string,"description":string,"ip":string,"region":string,"createdTime":"2006-01-02 15:04:05"}],"total":int,"page":int,"size":int}` | 见模块约定 | 分页查询操作审计日志。 | 需登录,模块 log |
| admin-07-logs-metrics | 删除审计日志 | POST | `/api/admin/audit-logs/delete` | 表单参数:`logId` int64 必填,须 >0;`csrfToken` | 200 `{}` | 错误:参数无效时 `this.Fail` | 调用 `DeleteLogPermanently` 永久删除单条审计日志,不可恢复,且本操作自身不写操作日志。 | 需登录+CSRF,模块 log |
| admin-07-logs-metrics | 访问日志列表 | GET | `/api/admin/access-logs` | 查询参数:<br>`day` string,默认当天,`YYYY-MM-DD`,错误 "invalid access log day"<br>`hourFrom` / `hourTo` string,可选(Admin 层无格式校验,待确认)<br>`keyword` string,`ip` string,`domain` string,可选<br>`requestId` string,可选,游标(取上一页的 `nextRequestId`)<br>`clusterId` / `nodeId` / `serverId` int64,可选,负数报 "invalid access log target";`nodeId` 须属于 `clusterId`("node does not belong to the selected cluster");`serverId` 须是 HTTP 族站点("advanced website settings require an HTTP-family site")<br>`hasError` bool,`hasWAF` bool,可选<br>`partition` int32,默认 -1<br>`size` int,默认 50(传 0),范围 1-100,否则 "access log page size should be between 1 and 100" | 200 `{"items":[{"requestId","timestamp","time","method","scheme","host","uri","protocol","status","bytesSent","requestTimeMilliseconds","remoteAddr","region","node":{"id","name","cluster":{"id","name"}},"server":{"id","name"},"hasWAF","errorCount","tags"}],"day":string,"size":int,"nextRequestId":string,"hasMore":bool,"partitions":[int32],"clusters":[{id,name}],"nodes":[{id,name}](仅 clusterId>0,最多 500),"sites":[{"id","name","domain"}](最多 500),"queryMilliseconds":int}`<br>各字段精确类型待确认 | 见模块约定 | 按日期和多条件游标分页查询访问日志,同时返回筛选器下拉数据。 | 需登录,模块 server |
| admin-07-logs-metrics | 访问日志详情 | GET | `/api/admin/access-logs/detail` | 查询参数:`requestId` string 必填 | 200 `{"item":{...}}`,响应头 `Cache-Control: no-store`<br>`item` 键:requestId,serverId,nodeId,locationId,rewriteId,originId,remoteAddr,rawRemoteAddr,remotePort,remoteUser,region,requestURI,requestPath,requestLength,requestTimeMilliseconds,requestMethod,requestFilename,requestBody,requestBodyEncoding(utf8/base64),requestBodyTruncated(请求体超 64KB 截断),scheme,proto,bytesSent,bodyBytesSent,status,statusMessage,timeISO8601,timeLocal,timestamp,host,referer,userAgent,request,contentType,cookies,args,queryString,headers,sentHeaders,serverName,serverPort,serverProtocol,hostname,originAddress,originStatus,errors,attrs,firewallPolicyId,firewallRuleGroupId,firewallRuleSetId,firewallRuleId,firewallActions,tags,node,server<br>键名含 authorization/cookie/token/secret/password/api-key/apikey/session 的 header/cookie/attrs 值被替换为 `[REDACTED]` | 错误:日志不存在时的文案待确认 | 读取单条访问日志详情,敏感值已脱敏。 | 需登录,模块 server |
| admin-07-logs-metrics | 访问日志策略列表 | GET | `/api/admin/access-log-policies` | 查询参数:`page` int,默认 1;`size` int,默认 20,1-100 | 200 `{"items":[{"id","name","typeName","isOn","isPublic","firewallOnly","disableDefaultDB","options","hasPassword","summary"}],"total":int,"page":int,"size":int,"canManage":bool}`<br>es/clickhouse 类型 `options.password` 已移除,以 `hasPassword` 表示是否已配置 | 见模块约定 | 分页查询访问日志存储策略。 | 需登录,模块 server |
| admin-07-logs-metrics | 访问日志策略表单数据 | GET | `/api/admin/access-log-policies/form` | 查询参数:`policyId` int64,可选;小于等于 0 表示新建 | 200 `{"canManage":bool,"types":[{"name","code","description","icon"}],"syslogPriorities":[{"name","value"}],"policy":对象或 null}`,响应头 `Cache-Control: no-store`<br>存储类型 `code`:file/es/tcp/syslog/command/clickhouse;`policy` 同列表项键 | 见模块约定 | 返回新建/编辑策略表单所需的类型与现有值(密码已脱敏)。 | 需登录,模块 server |
| admin-07-logs-metrics | 创建访问日志策略 | POST | `/api/admin/access-log-policies/create` | 表单参数:<br>`name` string 必填<br>`type` string 必填,file/es/tcp/syslog/command/clickhouse(Admin 层不校验)<br>`optionsJSON` string(JSON 字节)必填,内容依 type 而定(Admin 层不校验)<br>`isPublic` bool,`firewallOnly` bool,`disableDefaultDB` bool<br>`csrfToken` | 200 `{"policyId":int64}` | 错误:"only a super administrator can change access log storage";EdgeAPI 校验错误 | 创建访问日志存储策略,要求超级管理员。 | 需登录+CSRF,模块 server,超级管理员 |
| admin-07-logs-metrics | 更新访问日志策略 | POST | `/api/admin/access-log-policies/update` | 表单参数:`policyId` int64 必填;`name` string;`optionsJSON` string;`isOn` bool;`isPublic` bool;`firewallOnly` bool;`disableDefaultDB` bool;`csrfToken`(不能修改 type) | 200 `{}` | 错误:同创建 | 更新访问日志存储策略,要求超级管理员。 | 需登录+CSRF,模块 server,超级管理员 |
| admin-07-logs-metrics | 删除访问日志策略 | POST | `/api/admin/access-log-policies/delete` | 表单参数:`policyId` int64 必填;`csrfToken` | 200 `{}` | 错误:同创建 | 删除访问日志存储策略,要求超级管理员。 | 需登录+CSRF,模块 server,超级管理员 |
| admin-07-logs-metrics | 测试访问日志策略 | POST | `/api/admin/access-log-policies/test` | 表单参数:`policyId` int64 必填;`csrfToken` | 200 `{}`(结果形态待确认) | 错误:同创建 | 向该策略写入一条固定测试访问日志(RequestURI 为 `/edgeroute-access-log-policy-test`),有写入副作用,要求超级管理员。 | 需登录+CSRF,模块 server,超级管理员 |
| admin-07-logs-metrics | 访问日志队列设置 | GET | `/api/admin/access-log-settings` | 无参数 | 200 `{"config":{"maxLength":int,"countPerSecond":int,"percent":int,"enableAutoPartial":bool,"rowsPerTable":int64},"queue":{"pending","current","retry","old","oldQueues","capacity","dropped","usagePercent"},"canManage":bool}`<br>`config` 默认值 1000/10000/100/true/500000;`queue` 各字段类型待确认 | 见模块约定 | 读取访问日志写入队列配置与实时队列状态。 | 需登录,模块 server |
| admin-07-logs-metrics | 更新访问日志队列设置 | POST | `/api/admin/access-log-settings` | 表单参数:`maxLength` int;`countPerSecond` int;`percent` int;`enableAutoPartial` bool;`rowsPerTable` int64;`csrfToken`<br>Admin 层无范围校验,范围由 EdgeAPI 决定(待确认) | 200 `{}` | 错误:无超级管理员权限(文案待确认) | 更新队列配置,要求超级管理员。 | 需登录+CSRF,模块 server,超级管理员 |
| admin-07-logs-metrics | 指标项列表 | GET | `/api/admin/metrics` | 查询参数:`page` int,默认 1;`size` int,默认 20,1-100;`category` string,默认 `http`,`http`/`tcp`/`udp` | 200 `{"items":[{"id","name","code","category","keys","period","periodUnit","periodUnitName","expiresPeriod","value","valueName","valueType","isOn","isPublic","isBuiltIn","clusterCount"}],"total":int,"page":int,"size":int,"category":string,"categories":[{"code","name"}],"periodUnits":[...],"keyDefinitions":[...],"valueDefinitions":[...],"chartTypes":[...]}`<br>`periodUnit` 取值 minute/hour/day/week/month;各定义数组元素字段待确认 | 见模块约定 | 分页查询指标项及其下拉定义。 | 需登录,模块 server |
| admin-07-logs-metrics | 指标项详情 | GET | `/api/admin/metrics/detail` | 查询参数:`itemId` int64 必填;`statPage` int,默认 1;`statSize` int,默认 20(同分页规范) | 200 `{"item":{同列表项},"charts":[{"id","name","type","typeName","widthDiv","maxItems","paramsJSON","isOn","ignoreEmptyKeys","ignoredKeys"}],"chartTotal":int,"stats":[{"id","keys","value","ratio","time","cluster":{id,name},"node":{id,name},"server":{id,name}}],"statTotal":int,"statPage":int,"statSize":int,"clusters":[...],"keyDefinitions":[...],"valueDefinitions":[...],"periodUnits":[...],"chartTypes":[...]}` | 错误:"指标不存在" | 读取指标项、图表和统计数据。无 id>0 校验。 | 需登录,模块 server |
| admin-07-logs-metrics | 创建指标项 | POST | `/api/admin/metrics/create` | 表单参数:`itemJSON` string(JSON 对象)必填;`csrfToken`<br>字段:`name` string,`category` string,`keys` string[],`period` int,`periodUnit` string,`expiresPeriod` int,`value` string,`isPublic` bool(`isOn` 在创建时不使用) | 200 `{"itemId":int64}` | 错误:JSON 解码错误;EdgeAPI 校验 | 创建指标项。 | 需登录+CSRF,模块 server |
| admin-07-logs-metrics | 更新指标项 | POST | `/api/admin/metrics/update` | 表单参数:`itemId` int64 必填;`itemJSON` string 必填;`csrfToken`<br>字段同创建,另用 `isOn` bool;不使用 `category` | 200 `{}` | 见模块约定 | 更新指标项。无 id>0 校验。 | 需登录+CSRF,模块 server |
| admin-07-logs-metrics | 删除指标项 | POST | `/api/admin/metrics/delete` | 表单参数:`itemId` int64 必填;`csrfToken` | 200 `{}` | 见模块约定 | 删除指标项。无 id>0 校验。 | 需登录+CSRF,模块 server |
| admin-07-logs-metrics | 创建指标图表 | POST | `/api/admin/metrics/charts/create` | 表单参数:`itemId` int64 必填;`chartJSON` string 必填;`csrfToken`<br>字段:`name` string,`type` string,`widthDiv` int,`maxItems` int,`paramsJSON` string(字符串化的 JSON 对象,默认 `"{}"`),`ignoreEmptyKeys` bool,`ignoredKeys` string[],`isOn` bool | 200 `{"chartId":int64}` | 错误:"图表参数不是有效 JSON 对象" | 为指标项新增图表。无 id>0 校验。 | 需登录+CSRF,模块 server |
| admin-07-logs-metrics | 更新指标图表 | POST | `/api/admin/metrics/charts/update` | 表单参数:`chartId` int64 必填;`chartJSON` string 必填(字段同创建);`csrfToken` | 200 `{}` | 错误:同创建 | 更新图表。无 id>0 校验。 | 需登录+CSRF,模块 server |
| admin-07-logs-metrics | 删除指标图表 | POST | `/api/admin/metrics/charts/delete` | 表单参数:`chartId` int64 必填;`csrfToken` | 200 `{}` | 见模块约定 | 删除图表。无 id>0 校验。 | 需登录+CSRF,模块 server |
| admin-07-logs-metrics | 指标集群设置列表 | GET | `/api/admin/metrics/clusters` | 查询参数:`clusterId` int64,可选,未匹配到集群时静默回退到第一个集群;`category` string,可选(默认值待确认) | 200 `{"clusters":[{"id","name","isGlobal","useGlobal"}],"cluster":{...},"items":[{指标项字段,"isAssigned":bool,"isEffective":bool,"clusterCount":0}],"category":string,"categories":[{"code","name"}]}` | 错误:"没有可用的节点集群";"too many metric items: N"(指标项超过 1000) | 查看某节点集群启用了哪些指标项(含全局设置)。`items[].clusterCount` 固定为 0(疑似未赋值)。 | 需登录,模块 node |
| admin-07-logs-metrics | 设置集群指标项 | POST | `/api/admin/metrics/clusters/assignment` | 表单参数:`clusterId` int64;`itemId` int64;`enabled` bool;`csrfToken` | 200 `{}` | 见模块约定 | 为集群启用/停用某指标项。无 id>0 校验。 | 需登录+CSRF,模块 node |
| admin-07-logs-metrics | 设置集群全局指标 | POST | `/api/admin/metrics/clusters/global` | 表单参数:`clusterId` int64;`useGlobal` bool;`csrfToken` | 200 `{}` | 见模块约定 | 设置集群是否使用全局指标配置(写入集群全局设置 `metrics`),不写操作日志。无 id>0 校验。 | 需登录+CSRF,模块 node |

<a id="m9"></a>

## 9. 管理端 · 财务(管理端)

**模块约定**

> 路由总数 47,来源 EdgeAdmin/internal/web/actions/default/api/admin/init.go(处理器:finance_operations_resource.go、finance_catalog_resource.go、finance_coupons_resource.go、finance_settings_resource.go、finance_plan_sort.go)
说明:
- 路径前缀 `/api/admin`,本模块全部路由依次经 `helpers.NewUserMustAuth(AdminModuleCodeFinance)` 与 `helpers.NewLicenseFeature("finance")`。认证失败包络为 `{"code":N,"message":"...","data":null}`:未登录 401 `authentication required`;无 finance 模块权限 403 `permission denied`;系统未初始化 409 `system setup is required`;恶意请求/IP 不允许 403 空体;注入检测 403 文本 `Denied By WAF`。授权许可不含 finance 功能:HTTP 403 + 纯文本 `当前 EdgeRoute 授权未包含该功能`(非 JSON);许可状态读取失败:HTTP 503 + 纯文本 `EdgeRoute license status is temporarily unavailable`(非 JSON)。
- 成功响应:`{"code":200,"message":"","data":{...}}`,下表"返回"只写 `data` 内字段;"data 为 {}"指成功但无业务字段。失败有两种来源:
  - 校验错误(处理器内 `this.Fail(msg)`):`{"code":400,"message":msg,"data":{...},"errors":null}`。HTTP 状态行未被显式设置(推断为 200,待确认)。
  - RPC 错误(`actionutils.Fail`,下表简写为"RPC错误"):HTTP 状态码与 JSON `code` 一致,按 gRPC 状态映射(InvalidArgument 400 / Unauthenticated 401 / PermissionDenied 403 / NotFound 404 / AlreadyExists 409 / ResourceExhausted 429 / Unavailable 503 / DeadlineExceeded 504 / 其他 500;Unknown 错误按文本关键字推断)。`message` 为 `AdminCommon_ServerError` 中文文案 + `（原因）`;原因含 password=/token=/secret= 等敏感词时替换为"服务处理失败，请检查相关配置后重试。"。RPC 层校验文案多为英文(如 `invalid plan id`),下表只在已读到时列出。
  - RPC 客户端不可用:`RPC服务暂时不可用，请稍后重试：<原因>`(`this.Fail`)。
- 参数传递:TeaGo 只读 URL 查询串和 POST 表单(`application/x-www-form-urlencoded` 或 multipart),**不解析 JSON 请求体**。参数名取处理器 params struct 字段名,先按原样精确匹配,再按首字母小写匹配;有 `alias` tag 时以 alias 为准(同样再试首字母小写),大小写敏感。因此 `AppID` 对应 `AppID`/`appID`,`APIV3Key` 对应 `APIV3Key`/`aPIV3Key`。缺省参数取零值(数字 0、字符串空、布尔 false),`[]T` 参数用重复键传多个值。布尔值 `on/true/yes/enabled` 为 true,其余按 `types.Bool` 转换(具体规则待确认)。字符串参数自动 trim。
- **布尔字段缺省即 false**:凡"整体覆盖"类更新接口(plans/update、coupons/save、methods/update、settings/update 等)漏传布尔字段会把它置为 false(如停用),调用方须带全量字段。
- CSRF:所有 POST 处理器参数里都声明了 `CSRF`,须携带表单(或查询串)参数 `csrfToken`,与会话 `Sid` 配对,有效期 1800 秒,令牌来源 `GET /api/admin/session` 的 `data.csrfToken`(或 `GET /csrf/token` 的 `data.token`)。失败返回 HTTP 403 + 纯文本 `表单已失效，请刷新页面后重试(001)`(非 JSON)。下表 POST 行不再重复列出 `csrfToken`(均必填)。CSRF 校验在参数绑定阶段、业务逻辑之前执行。
- 分页(`page`/`size` 型):`page` int64,<1 取 1;`size` int64,<1 取 20,>100 取 100(截断,不报错)。响应回显实际生效的 `page`/`size`,并带 `total`。其余列表无分页或见各行。
- 时间字段:订单/退款/账户日志/用户流量包为 `YYYY-MM-DD HH:mm:ss`(服务器本地时区,0 值为空串);账单日期为 `YYYY-MM-DD`,月份为 `YYYY-MM`;优惠券 `startsAt`/`endsAt` 及兑换记录时间为 RFC3339 字符串(0 值为空串),而优惠券 `createdAt`/`updatedAt` 为原始 int64(单位待确认,疑似 Unix 秒)。
- 金额字段单位:订单/退款 `amount` 等为元(`amountCents>0` 时取 `amountCents/100`);账单、账户余额、套餐价格的币种/精度待确认。
- 操作日志:本模块处理器内仅 `plans/sort` 调用 `CreateLogInfo`;其余写接口是否在 EdgeAPI 侧记日志待确认。

| 模块 | 接口名称 | 方法 | 路径 | 请求参数 | 响应数据 | 主要错误 | 功能说明 | 认证/权限 |
|---|---|---|---|---|---|---|---|---|
| admin-08-finance | 订单列表 | GET | `/api/admin/finance/orders` | 查询参数:<br>`page` int64,默认 1<br>`size` int64,默认 20,最大 100<br>`keyword` string,可选,RPC 限 ≤200 字符<br>`status` string,可选,空/`pending`/`cancelled`/`finished`/`expired`,其余 RPC 报 `invalid order status`<br>`userId` int64,可选,0 不过滤 | `{"items":[Order],"total":int64,"page":int64,"size":int64}` | 错误:RPC错误 | 分页查询用户订单(先 Count 再 List)。 | 登录+finance 模块+finance 授权 |
| admin-08-finance | 订单详情 | GET | `/api/admin/finance/orders/detail` | 查询参数:`code` string,必填(trim 后为空返回校验错误) | `{"order":Order}` | 错误:400 `order code is required`;400 `order not found`;RPC错误 | 按订单号读取单个订单。 | 登录+finance 模块+finance 授权 |
| admin-08-finance | 完成订单 | POST | `/api/admin/finance/orders/finish` | 表单:`code` string,订单号(处理器不校验,空值由 RPC 报错) | data 为 `{}` | 错误:RPC错误 | 人工将订单置为已完成(`FinishUserOrder`)。状态流转规则在 EdgeAPI,待确认。 | 登录+finance 模块+finance 授权+CSRF |
| admin-08-finance | 取消订单 | POST | `/api/admin/finance/orders/cancel` | 表单:`code` string,订单号(处理器不校验) | data 为 `{}` | 错误:RPC错误(已读到 RPC 文案 `finished order cannot be cancelled`) | 取消订单(`CancelUserOrder`)。已完成订单不可取消;其余状态约束待确认。 | 登录+finance 模块+finance 授权+CSRF |
| admin-08-finance | 退款列表 | GET | `/api/admin/finance/refunds` | 查询参数:`page`、`size` 同订单列表<br>`keyword` string,可选<br>`status` string,可选(DAO 状态常量为 `pending`/`finished`/`cancelled`/`rejected`,过滤校验规则待确认)<br>`userId` int64,可选 | `{"items":[Refund],"total":int64,"page":int64,"size":int64}` | 错误:RPC错误 | 分页查询退款申请。 | 登录+finance 模块+finance 授权 |
| admin-08-finance | 完成退款 | POST | `/api/admin/finance/refunds/finish` | 表单:<br>`code` string,退款单号<br>`providerReference` string,可选,支付渠道退款凭证<br>`adminNote` string,可选 | data 为 `{}` | 错误:RPC错误 | 标记退款已完成(`FinishUserRefund`),本接口仅记录结果,是否实际向支付渠道发起退款待确认。处理器不校验参数。 | 登录+finance 模块+finance 授权+CSRF |
| admin-08-finance | 驳回退款 | POST | `/api/admin/finance/refunds/reject` | 表单:<br>`code` string,退款单号<br>`adminNote` string,可选 | data 为 `{}` | 错误:RPC错误 | 驳回退款申请(`RejectUserRefund`)。 | 登录+finance 模块+finance 授权+CSRF |
| admin-08-finance | 账单列表 | GET | `/api/admin/finance/bills` | 查询参数:<br>`page`、`size` 同订单列表<br>`status` string,可选,`paid`/`unpaid`(忽略大小写),其他值不过滤<br>`month` string,可选,`YYYY-MM` 或 `YYYYMM`(去掉 `-` 后传给 RPC)<br>`userId` int64,可选 | `{"items":[Bill],"total":int64,"page":int64,"size":int64,"suggestedMonth":"YYYY-MM","suggestedDay":"YYYY-MM-DD"}`<br>`suggestedMonth` 为上月,`suggestedDay` 为昨天(服务器本地时间) | 错误:RPC错误 | 分页查询用户账单。Count 固定带 `trafficRelated=true`,List 未带该条件,二者口径是否一致待确认(疑似 total 与 items 不一致)。 | 登录+finance 模块+finance 授权 |
| admin-08-finance | 账单详情 | GET | `/api/admin/finance/bills/detail` | 查询参数:`billId` int64,必填,>0 | `{"bill":Bill,"details":[{"id":int64,"serverId":int64,"amount":number,"priceType":string,"trafficGB":number,"bandwidthMbps":number,"bandwidthPercentile":int,"server?":{"id","name"},"plan?":{"id","name"}}]}` | 错误:400 `invalid bill id`;400 `bill not found`(账单或其用户为空);RPC错误 | 读取账单及按网站拆分的明细。明细最多取 5000 条;`pricePeriod=daily` 时只取账单起始日当天明细。`trafficGB` 按 1024^3 换算;`bandwidthMbps` 按 `字节*8/1024/1024` 换算(与区域价格用的 1e6 不同,换算口径待确认)。 | 登录+finance 模块+finance 授权 |
| admin-08-finance | 生成账单 | POST | `/api/admin/finance/bills/generate` | 表单:<br>`month` string,`YYYY-MM`/`YYYYMM`(去掉 `-`)<br>`day` string,`YYYY-MM-DD`/`YYYYMMDD`(去掉 `-`) | data 为 `{}` | 错误:RPC错误 | 触发全部用户账单生成。RPC 侧 `day` 非空时按日结算,否则按月结算;两者都为空时的行为待确认。 | 登录+finance 模块+finance 授权+CSRF |
| admin-08-finance | 标记账单已付 | POST | `/api/admin/finance/bills/pay` | 表单:`billId` int64,账单 ID(处理器不校验) | data 为 `{}` | 错误:RPC错误(RPC 对非法 ID 报 `invalid user bill id`) | 人工将账单标记为已支付(`PayUserBill`),是否扣减账户余额待确认。 | 登录+finance 模块+finance 授权+CSRF |
| admin-08-finance | 账户列表 | GET | `/api/admin/finance/accounts` | 查询参数:`page`、`size` 同订单列表<br>`keyword` string,可选(trim) | `{"items":[Account],"total":int64,"page":int64,"size":int64}` | 错误:RPC错误 | 分页查询用户账户余额。 | 登录+finance 模块+finance 授权 |
| admin-08-finance | 账户详情 | GET | `/api/admin/finance/accounts/detail` | 查询参数:<br>`accountId` int64,必填,>0<br>`page`、`size` 同订单列表(用于流水分页) | `{"account":Account,"logs":[{"id":int64,"delta":number,"total":number,"eventType":string,"description":string,"createdTime":string}],"total":int64,"page":int64,"size":int64}`(`total` 为流水总数) | 错误:400 `invalid account id`;400 `account not found`;RPC错误 | 读取账户及余额变动流水。`eventType` 取值含 `adminCharge`/`adminDebit`,其余待确认。 | 登录+finance 模块+finance 授权 |
| admin-08-finance | 调整账户余额 | POST | `/api/admin/finance/accounts/adjust` | 表单:<br>`accountId` int64,可选;≤0 且 `userId`>0 时按用户查找(必要时创建)账户<br>`userId` int64,可选<br>`delta` float64,必填,非 0,绝对值 ≤1000000,不可 NaN/Inf(由 RPC 校验)<br>`description` string,可选,默认 `Administrator balance adjustment`<br>`requestKey` string,可选,幂等键,相同键对同账户只生效一次 | `{"accountId":int64}` | 错误:400 `account could not be created`;RPC错误(`accountId` 与 `userId` 均未给时 RPC 报 `invalid account update`) | 手动充值(`delta>0`,事件 `adminCharge`)或扣减(`delta<0`,事件 `adminDebit`)。涉及资金,处理器未调用 `CreateLog`,审计依赖 EdgeAPI 侧(待确认)。 | 登录+finance 模块+finance 授权+CSRF |
| admin-08-finance | 套餐列表 | GET | `/api/admin/finance/plans` | 查询参数:`page`、`size` 同订单列表 | `{"items":[Plan],"total":int64,"page":int64,"size":int64}` | 错误:RPC错误 | 分页查询套餐。 | 登录+finance 模块+finance 授权 |
| admin-08-finance | 套餐表单数据 | GET | `/api/admin/finance/plans/form` | 查询参数:`planId` int64,可选,>0 时返回该套餐 | `{"clusters":[{"id":int64,"name":string,"isOn":bool}],"features":[{"value":string,"label":string,"description":string}],"plan?":Plan}`<br>`clusters` 取前 5000 个集群(含停用);`features` 仅含支持套餐的用户功能 | 错误:400 `plan not found`;RPC错误 | 为新建/编辑套餐表单提供集群与功能选项,`planId>0` 时附带套餐详情。 | 登录+finance 模块+finance 授权 |
| admin-08-finance | 创建套餐 | POST | `/api/admin/finance/plans/create` | 表单(与更新接口共用字段,不含 `planId`、`isOn`):<br>`name` string,必填,≤200 字符<br>`description` string,≤500 字符<br>`clusterId` int64,必填,>0 且集群存在<br>`priceType` string,`period`/`traffic`/`bandwidth`<br>`trafficPriceBase` float64;`bandwidthPriceBase` float64;`bandwidthPercentile` int<br>`bandwidthRangesJSON` string,可选,JSON 数组,元素为 数值字典(键名待确认),非法返回 `invalid bandwidth price ranges`<br>`monthlyPrice`/`seasonallyPrice`/`yearlyPrice` float32,≥0<br>`totalServers`/`totalServerNames`/`totalServerNamesPerServer` int32,`monthlyRequests`/`monthlyWebsocketConnections` int64,均 ≥0<br>`trafficLimitIsOn` bool;`monthlyTrafficSize`/`totalTrafficSize` int64 ≥0(0 表示不限);`monthlyTrafficUnit`/`totalTrafficUnit` string,`byte`/`kb`/`mb`/`gb`/`tb`/`pb`/`eb`;`trafficLimitNoticePageBody` string,≤256KB<br>`bandwidthLimitPerNode` int64 ≥0;`bandwidthLimitPerNodeUnit` string,`b`/`kb`/`mb`/`gb`/`tb`/`pb`/`eb`<br>`maxUploadSize` int64 ≥0;`maxUploadSizeUnit` string,同流量单位<br>`maxUserPurchases` int32,0–100000<br>`hasFullFeatures` bool;`featuresJSON` string,JSON 字符串数组(功能代码),`hasFullFeatures=true` 时忽略,否则去重、必须是支持套餐的功能、最多 64 个 | `{"planId":int64}` | 错误:400 校验文案(`invalid monthly traffic limit: ...`/`invalid total traffic limit: ...`/`invalid per-node bandwidth limit: ...`/`invalid maximum upload size: ...`/`traffic limit notice page is too large`/`invalid plan features`/`feature is not available for plans: <code>`/`too many plan features`);RPC错误(`invalid plan name`/`invalid cluster id`/`invalid plan price type`/`node cluster not found` 等) | 创建套餐。任一月/总流量上限大于 0 时 `trafficLimitIsOn` 自动视为 true。 | 登录+finance 模块+finance 授权+CSRF |
| admin-08-finance | 更新套餐 | POST | `/api/admin/finance/plans/update` | 表单:同创建套餐的全部字段,另加:<br>`planId` int64,必填(处理器不校验,RPC 报 `invalid plan id`)<br>`isOn` bool,是否启用(漏传即停用) | data 为 `{}` | 错误:同创建套餐 | 整体覆盖更新套餐。 | 登录+finance 模块+finance 授权+CSRF |
| admin-08-finance | 删除套餐 | POST | `/api/admin/finance/plans/delete` | 表单:`planId` int64,必填(RPC 校验) | data 为 `{}` | 错误:RPC错误(`invalid plan id`) | 删除套餐;是否允许删除已被购买的套餐待确认。 | 登录+finance 模块+finance 授权+CSRF |
| admin-08-finance | 套餐排序 | POST | `/api/admin/finance/plans/sort` | 表单:`planIds` []int64,重复键传多个,必须恰好包含当前全部套餐各一次 | data 为 `{}` | 错误:400 `plan order must contain every plan`/`current plan order is invalid`/`plan order is invalid`;RPC错误 | 按给定顺序重排套餐。先拉取至多 10000 个套餐校验,成功后记操作日志(`Plan_LogSortPlans`)。 | 登录+finance 模块+finance 授权+CSRF |
| admin-08-finance | 优惠券列表 | GET | `/api/admin/finance/coupons` | 查询参数:`page`、`size` 同订单列表 | `{"items":[Coupon],"total":int64,"page":int64,"size":int64,"completedCount":int64,"totalDiscountCents":int64}` | 错误:RPC错误 | 分页查询优惠券及汇总统计。与 `POST .../coupons/save` 为同一处理器 `FinanceCouponsAction`,GET 分支为列表。 | 登录+finance 模块+finance 授权 |
| admin-08-finance | 优惠券兑换记录 | GET | `/api/admin/finance/coupons/redemptions` | 查询参数:`page`、`size` 同订单列表 | `{"items":[{"id":int64,"couponId":int64,"couponCode":string,"orderId":int64,"userId":int64,"status":string,"discountCents":int64,"createdAt":string,"finishedAt":string}],"total":int64,"page":int64,"size":int64}`(时间为 RFC3339,0 值为空串) | 错误:RPC错误 | 分页查询优惠券使用记录。`status` 取值待确认。 | 登录+finance 模块+finance 授权 |
| admin-08-finance | 保存优惠券 | POST | `/api/admin/finance/coupons/save` | 表单:<br>`id` int64,可选;>0 为更新,否则为创建<br>`code` string,转大写,RPC 限 3–64 字符<br>`name` string<br>`isOn` bool(漏传即停用)<br>`discountType` string,`fixed`(默认)/`percent`,其他 RPC 报 `优惠码折扣类型无效`<br>`discountValue` float64,`percent` 时 (0,100];`fixed` 时 (0,1e8]<br>`planId` int64,≥0,0 表示不限套餐<br>`startsAt`/`endsAt` int64,Unix 秒,≥0,均>0 时 `endsAt` 须大于 `startsAt`<br>`maxUses`/`perUserLimit` int32,≥0(0 的含义待确认)<br>`minAmountCents` int64,0–10000000000<br>`newUserOnly` bool | data 为 `{}` | 错误:400 `有效期参数无效`(`startsAt`/`endsAt`<0);RPC错误(`优惠码长度必须为 3-64 个字符`/`优惠码折扣值无效`/`百分比折扣必须大于 0 且不超过 100`/`固定折扣金额无效`/`优惠码限制参数无效`;更新时 `invalid coupon id`/`coupon not found`) | 创建或更新优惠券(同一处理器的 POST 分支)。处理器内对 `discountType` 的默认值归一化作用于值拷贝,不会传给 RPC,实际默认由 RPC 处理。 | 登录+finance 模块+finance 授权+CSRF |
| admin-08-finance | 批量生成优惠券 | POST | `/api/admin/finance/coupons/bulk` | 表单:<br>`codePrefix` string,可选,转大写后 ≤24 位,仅 `A-Z0-9-`<br>`count` int32,必填,1–500<br>`name`、`isOn`、`discountType`、`discountValue`、`planId`、`startsAt`、`endsAt`、`maxUses`、`perUserLimit`、`minAmountCents`、`newUserOnly` 同保存优惠券 | `{"codes":[string]}`(生成的优惠码列表) | 错误:RPC错误(`批量生成数量必须为 1-500`/`前缀仅支持 1-24 位字母、数字或短横线` 及保存优惠券同类校验) | 批量生成优惠码。 | 登录+finance 模块+finance 授权+CSRF |
| admin-08-finance | 删除优惠券 | POST | `/api/admin/finance/coupons/delete` | 表单:`id` int64,优惠券 ID | data 为 `{}` | 错误:RPC错误(`invalid coupon id`) | 删除优惠券。 | 登录+finance 模块+finance 授权+CSRF |
| admin-08-finance | 流量包概览 | GET | `/api/admin/finance/packages` | 查询参数:<br>`page`、`size` 同订单列表(仅作用于 `grants`)<br>`userId` int64,可选,过滤用户已持有流量包<br>`availableOnly` bool,可选,仅返回可用 | `{"packages":[{"id","size","unit","bytes","isOn"}],"periods":[{"id","count","unit","months","isOn","name"}],"regions":[{"id","name","isOn"}],"prices":[{"packageId","regionId","periodId","price"}],"grants":[UserPackage],"grantTotal":int64,"page":int64,"size":int64}`<br>`unit` 大写(GB/TB/PB);`period.name` 如 `3 months`/`1 years` | 错误:RPC错误 | 一次返回流量包规格、有效期规格、区域、价格矩阵(均不分页)与用户已授予流量包(分页)。 | 登录+finance 模块+finance 授权 |
| admin-08-finance | 创建流量包 | POST | `/api/admin/finance/packages/create` | 表单:<br>`size` int32,>0<br>`unit` string,`gb`/`tb`/`pb`(忽略大小写) | `{"packageId":int64}` | 错误:RPC错误(`invalid traffic package size`/`invalid traffic package unit`/`traffic package size is too large`) | 创建流量包规格。 | 登录+finance 模块+finance 授权+CSRF |
| admin-08-finance | 更新流量包 | POST | `/api/admin/finance/packages/update` | 表单:<br>`packageId` int64<br>`isOn` bool(漏传即停用) | data 为 `{}` | 错误:RPC错误(`invalid traffic package id`) | 启停流量包规格。 | 登录+finance 模块+finance 授权+CSRF |
| admin-08-finance | 删除流量包 | POST | `/api/admin/finance/packages/delete` | 表单:`packageId` int64 | data 为 `{}` | 错误:RPC错误(`invalid traffic package id`) | 删除流量包规格。 | 登录+finance 模块+finance 授权+CSRF |
| admin-08-finance | 创建流量包有效期 | POST | `/api/admin/finance/packages/periods/create` | 表单:<br>`count` int32,1–1200<br>`unit` string,`month`/`year` | `{"periodId":int64}` | 错误:RPC错误(`invalid traffic package period`/`invalid traffic package period unit`/`traffic package period is too long`) | 创建流量包有效期规格(`year` 折算为 12 个月)。 | 登录+finance 模块+finance 授权+CSRF |
| admin-08-finance | 更新流量包有效期 | POST | `/api/admin/finance/packages/periods/update` | 表单:<br>`periodId` int64<br>`isOn` bool(漏传即停用) | data 为 `{}` | 错误:RPC错误(`invalid traffic package period id`) | 启停有效期规格。 | 登录+finance 模块+finance 授权+CSRF |
| admin-08-finance | 删除流量包有效期 | POST | `/api/admin/finance/packages/periods/delete` | 表单:`periodId` int64 | data 为 `{}` | 错误:RPC错误(`invalid traffic package period id`) | 删除有效期规格。 | 登录+finance 模块+finance 授权+CSRF |
| admin-08-finance | 更新流量包价格 | POST | `/api/admin/finance/packages/prices/update` | 表单:<br>`packageId` int64<br>`regionId` int64<br>`periodId` int64<br>`price` float64,价格(取值范围由 RPC 校验,`invalid traffic package price`,具体范围待确认) | data 为 `{}` | 错误:RPC错误 | 设置某流量包在某区域、某有效期下的价格。 | 登录+finance 模块+finance 授权+CSRF |
| admin-08-finance | 授予用户流量包 | POST | `/api/admin/finance/packages/grants/create` | 表单:<br>`userId` int64<br>`packageId` int64<br>`regionId` int64<br>`periodId` int64<br>`count` int32,授予数量(范围由 RPC 校验,待确认) | `{"createdCount":int}` | 错误:RPC错误(`traffic package is not available`/`traffic package period is not available`/`node region is not available`/`user not found`) | 管理员直接给用户发放流量包(不经支付订单)。 | 登录+finance 模块+finance 授权+CSRF |
| admin-08-finance | 删除用户流量包 | POST | `/api/admin/finance/packages/grants/delete` | 表单:`grantId` int64,用户流量包 ID | data 为 `{}` | 错误:RPC错误(`invalid user traffic package id`/`user traffic package not found`/`used traffic package cannot be deleted`) | 删除已授予的用户流量包,已被使用的不可删。 | 登录+finance 模块+finance 授权+CSRF |
| admin-08-finance | 区域价格列表 | GET | `/api/admin/finance/regional-prices` | 查询参数:`type` string,可选,`bandwidth` 则为带宽,其余任意值(含空)均按 `traffic` | `{"items":[{"id","name","isOn","type","from":number,"to":number,"fromLabel":string,"toLabel":string}],"regions":[{"id","name","isOn","prices":{"<key>":number}}],"priceType":"traffic/bandwidth","rangeUnit":"GB/Mbps","priceUnit":"CNY/Gbit 或 CNY/Mbps"}`<br>`from`/`to`:0 表示不限;label 为 `<数值><单位>` 或 `unlimited`;`prices` 键含义(推测为价格区间 ID)待确认 | 错误:RPC错误(区域 `pricesJSON` 解析失败也走此路径) | 读取按流量/带宽区间定价的价格区间与各区域单价。区间换算:traffic 为 `1GB=8e9 bit`,bandwidth 为 `1Mbps=1e6 bit`。 | 登录+finance 模块+finance 授权 |
| admin-08-finance | 创建区域价格区间 | POST | `/api/admin/finance/regional-prices/items/create` | 表单:<br>`name` string,1–100 字符<br>`type` string,`bandwidth` 或 其他(按 traffic)<br>`from` float64,≥0<br>`to` float64,≥0,0 不限;>0 时须大于 `from` | `{"itemId":int64}` | 错误:400 `invalid price range`/`price range is too large`;RPC错误(`invalid node price item name`/`node price item range overlaps an existing item`) | 创建价格区间(按单位换算为 bit 存储),区间不可与已有区间重叠。 | 登录+finance 模块+finance 授权+CSRF |
| admin-08-finance | 更新区域价格区间 | POST | `/api/admin/finance/regional-prices/items/update` | 表单:<br>`itemId` int64<br>`name` string<br>`isOn` bool(漏传即停用)<br>`type` string,仅用于单位换算,不会修改区间类型<br>`from`/`to` float64,规则同创建 | data 为 `{}` | 错误:同创建;RPC错误(`invalid node price item id`/`node price item not found`) | 更新价格区间。**疑似缺陷**:`type` 漏传时按 traffic 换算,更新带宽区间会把 Mbps 值按 GB 倍率写入,调用方必须与区间原类型一致。 | 登录+finance 模块+finance 授权+CSRF |
| admin-08-finance | 删除区域价格区间 | POST | `/api/admin/finance/regional-prices/items/delete` | 表单:`itemId` int64 | data 为 `{}` | 错误:RPC错误(`invalid node price item id`) | 删除价格区间。 | 登录+finance 模块+finance 授权+CSRF |
| admin-08-finance | 设置区域单价 | POST | `/api/admin/finance/regional-prices/update` | 表单:<br>`regionId` int64,区域 ID<br>`itemId` int64,价格区间 ID<br>`price` float32,单价(取值范围由 RPC 校验,待确认) | data 为 `{}` | 错误:RPC错误 | 设置某区域在某价格区间的单价。 | 登录+finance 模块+finance 授权+CSRF |
| admin-08-finance | 支付方式列表 | GET | `/api/admin/finance/methods` | 无 | `{"items":[Method]}`(不分页) | 错误:RPC错误 | 列出所有支付方式;密钥类字段只返回是否已配置。 | 登录+finance 模块+finance 授权 |
| admin-08-finance | 支付方式表单数据 | GET | `/api/admin/finance/methods/form` | 查询参数:`methodId` int64,可选,>0 时返回该支付方式 | `{"parentTypes":[{"code":"manual","name":"人工收款"},{"code":"alipay","name":"支付宝"},{"code":"wechat","name":"微信支付"},{"code":"epay","name":"彩虹易支付 V2"},{"code":"bepusdt","name":"USDT（TRC20）"}],"method?":Method}` | 错误:400 `payment method not found`;RPC错误 | 提供支付类型选项与待编辑的支付方式详情(同样不含密钥原文)。 | 登录+finance 模块+finance 授权 |
| admin-08-finance | 创建支付方式 | POST | `/api/admin/finance/methods/create` | 表单:<br>通用:`name` string 1–100 字符;`code` string,小写,格式 `orderMethodCodePattern`(待确认),且不可重复;`description` ≤255;`url` string ≤1024;`parentCode` string,`manual`/`alipay`/`wechat`/`epay`/`bepusdt`(`custom` 被 RPC 拒绝);`clientType` string,空/`desktop`/`mobile`;`qrcodeTitle` string ≤200;`isOn` 创建时不使用<br>alipay:`appID`、`privateKey`、`alipayMode`(`publicKey`/其他视为证书模式)、`appPublicCert`、`alipayRootCert`、`alipayPublicCert`、`alipayPublicKey`、`productCode`(空/`FACE_TO_FACE_PAYMENT`/`FAST_INSTANT_TRADE_PAY`/`QUICK_WAP_WAY`)、`isProduction` bool;`url` 被强制清空,`qrcodeTitle` 空时默认 `请扫描二维码完成支付`<br>wechat:`appID`、`merchantID`、`merchantCertificateSerialNumber`、`merchantPrivateKey`、`APIV3Key`、`platformCertificate`;`paymentType` 强制为 `wxpay`<br>epay:`merchantID`、`merchantPrivateKey`、`platformPublicKey`、`paymentType`(小写)<br>bepusdt:`apiToken`、`exchangeRate` string、`callbackAck` string<br>(上述凭证类参数为敏感字段,仅请求时提交,响应不回显) | `{"methodId":int64}`;`parentCode=custom` 时还会读回并返回 `secret`(该分支被 RPC 拒绝,实际不可达) | 错误:400 `unsupported Alipay payment flow`;RPC错误(`invalid payment method name`/`invalid payment method code`/`payment method field is too long`/`invalid payment client type`/`custom payment methods are no longer supported`/`payment method code already exists`,及按类型的 `payments.ValidateMethod` 校验,文案待确认) | 创建支付方式。非 alipay/wechat/epay/bepusdt 类型(含 `manual`)参数 JSON 固定为 `{}`。 | 登录+finance 模块+finance 授权+CSRF |
| admin-08-finance | 更新支付方式 | POST | `/api/admin/finance/methods/update` | 表单:`methodId` int64,必填;`name`、`code`、`description`、`url`、`clientType`、`qrcodeTitle`、`isOn`(漏传即停用)及各渠道凭证字段同创建支付方式。`parentCode` 参数被忽略,以已有记录为准。凭证类字段留空表示保留旧值(无法通过本接口清空)。alipay 的 `isProduction` 为 bool,漏传即视为沙箱;`alipayMode` 漏传会按证书模式处理并清空已存的 `alipayPublicKey` | data 为 `{}` | 错误:400 `payment method not found`/`unsupported Alipay payment flow`;RPC错误(含 `invalid payment method id`/`payment method code already exists`/`payment method code cannot be changed after orders have been created`) | 整体覆盖更新支付方式;已产生订单后不可修改 `code`。 | 登录+finance 模块+finance 授权+CSRF |
| admin-08-finance | 删除支付方式 | POST | `/api/admin/finance/methods/delete` | 表单:`methodId` int64 | data 为 `{}` | 错误:RPC错误(`invalid payment method id`) | 删除支付方式。 | 登录+finance 模块+finance 授权+CSRF |
| admin-08-finance | 财务设置读取 | GET | `/api/admin/finance/settings` | 无 | `{"finance":FinanceConfig,"unpaidBillPolicy":{"isOn":bool,"minDailyBillDays":int32,"minMonthlyBillDays":int32,"disableServers":bool},"serverGroups":[{"id","name"}],"serverPolicy":{"defaultGroupId":int64,"requirePlan":bool,"domainPolicy":object,"enableStat":bool,"purge":{"maxKeysPerTask":int32,"maxKeysPerDay":int32},"fetch":{"maxKeysPerTask":int32,"maxKeysPerDay":int32}}} | 错误:RPC错误 | 读取财务与用户侧网站策略(两项系统设置合并返回)。`domainPolicy` 为 `domainpolicy.Config`,内部字段待确认。 | 登录+finance 模块+finance 授权 |
| admin-08-finance | 财务设置更新 | POST | `/api/admin/finance/settings/update` | 表单(全量覆盖,布尔漏传即 false):<br>`financeIsOn` bool<br>`priceType` string,`period`/`traffic`/`bandwidth`(仅转小写,未校验)<br>`pricePeriod` string(仅转小写,未校验,取值待确认)<br>`trafficPriceBase` float64<br>`bandwidthPriceBase` float64<br>`bandwidthPercentile` int(未校验)<br>`bandwidthRangesJSON` string,JSON 数组 `[{"minMB","maxMB","pricePerMB","totalPrice"}]`,空表示清空区间<br>`enablePlans`/`showPlansInUserSystem`/`enableTrafficPackages`/`showTrafficPackages`/`userCanChangePriceType`/`userCanChangePricePeriod`/`showPrices` bool<br>`unpaidBillPolicyIsOn` bool;`unpaidBillPolicyMinDailyBillDays`/`unpaidBillPolicyMinMonthlyBillDays` int32;`unpaidBillPolicyDisableServers` bool<br>`defaultGroupId` int64,>0 时须存在<br>`requirePlan` bool,为 true 时要求 `financeIsOn` 与 `enablePlans` 均为 true<br>`domainPolicyJSON` string,可选,≤256KB 的 JSON,空则保持原策略<br>`enableStat` bool<br>`purgeMaxKeysPerTask`/`fetchMaxKeysPerTask` int32,0–1000<br>`purgeMaxKeysPerDay`/`fetchMaxKeysPerDay` int32,0–10000;单次与每日均>0 时单次不得大于每日 | data 为 `{}` | 错误:400 `invalid bandwidth price ranges`/`invalid default site group`/`requiring a plan needs the finance system and plans to be enabled`/`invalid purge limits: ...`/`invalid prefetch limits: ...`/`域名接入策略格式无效`/域名策略 `Validate` 文案;RPC错误 | 写入两项系统设置:`UserFinanceConfig` 与 `UserServerConfig`(先写前者再写后者,后者失败时尽力回滚两项,非原子)。缓存任务额度 0 或非正数由读取方回落到默认值(依据 `userconfigs` 注释,细节待确认)。 | 登录+finance 模块+finance 授权+CSRF |


### 公共结构

### Order(订单)
`code` string、`userId` int64、`type` string(`charge`/`plan_purchase`/`plan_renewal`/`traffic_package_purchase`)、`amount` number、`status` string(`pending`/`cancelled`/`finished`)、`isExpired` bool、`canPay` bool、`payURL` string、`createdTime`/`cancelledTime`/`finishedTime` string、`refundedAmount`/`pendingRefundAmount`/`refundableAmount` number(元)、`canRefund` bool;`user?` `{id,username,name,email,mobile}`;`method?` `{id,name,code,parentCode}`。

### Refund(退款)
`id` int64、`code` string、`userId`、`orderId`、`orderMethodId` int64、`status` string、`amount` number、`reason`、`providerReference`、`adminNote` string、`requestedTime`/`processedTime` string;`user?` 同订单;`order?` 为 Order。

### Bill(账单)
`id` int64、`code` string、`type` string、`typeName` string、`description` string、`amount` number、`month` string(`YYYY-MM`)、`rawMonth` string(`YYYYMM`)、`dayFrom`/`dayTo` string(`YYYY-MM-DD`)、`pricePeriod` string(含 `daily`)、`isPaid`/`isOverdue`/`canPay` bool、`paidTime` string;`user?` 同订单。

### Account(账户)
`id` int64、`userId` int64、`total` number(余额)、`totalFrozen` number(冻结);`user?` `{id,username,name,email,mobile}`。

### Plan(套餐)
`id`、`name`、`description`、`isOn`、`clusterId`、`priceType`、`trafficPrice`(`{"base":number}`)、`bandwidthPrice`(`{"base","percentile","ranges"}`,默认 `{"base":0,"percentile":95,"ranges":[]}`)、`monthlyPrice`/`seasonallyPrice`/`yearlyPrice`、`totalServers`、`totalServerNames`、`totalServerNamesPerServer`、`monthlyRequests`、`monthlyWebsocketConnections`、`trafficLimitIsOn`(无月/总上限时恒为 false)、`monthlyTrafficSize`+`monthlyTrafficUnit`(默认 `gb`)、`totalTrafficSize`+`totalTrafficUnit`(默认 `tb`)、`trafficLimitNoticePageBody`、`bandwidthLimitPerNode`+`bandwidthLimitPerNodeUnit`(默认 `mb`)、`maxUploadSize`+`maxUploadSizeUnit`(默认 `mb`)、`maxUserPurchases`、`hasFullFeatures`、`features`(功能代码数组)。

### Coupon(优惠券)
`id`、`code`、`name`、`isOn`、`discountType`、`discountValue`、`planId`、`startsAt`(RFC3339 字符串)、`endsAt`(RFC3339 字符串)、`maxUses`、`perUserLimit`、`minAmountCents`、`newUserOnly`、`usedCount`、`createdAt`(原始 int64)、`updatedAt`(原始 int64)。

### UserPackage(用户流量包授予记录)
`id`、`userId`、`packageId`、`regionId`、`periodId`、`totalBytes`、`usedBytes`、`availableBytes`(`max(total-used,0)`)、`periodName`、`dayFrom`/`dayTo`(`YYYY-MM-DD`)、`createdTime`、`canDelete` bool、`isExpired` bool(`dayTo` 早于今天);`user?` `{id,username,name}`;`package?` `{id,size,unit}`;`region?` `{id,name}`。

### Method(支付方式,不含密钥)
`id`、`name`、`code`、`description`、`url`、`parentCode`、`clientType`、`qrcodeTitle`、`isOn`、`hasSecret` bool;按 `parentCode` 附加一个子对象:
- `alipay`:`appId`、`productCode`、`mode`(`publicKey`/`cert`)、`isProduction`(缺省 true)、`hasPrivateKey`、`hasAppPublicCert`、`hasAlipayRootCert`、`hasAlipayPublicCert`、`hasAlipayPublicKey`。
- `epay`:`merchantId`、`paymentType`、`hasMerchantPrivateKey`、`hasPlatformPublicKey`。
- `wechat`:`appId`、`merchantId`、`merchantCertificateSerialNumber`、`hasMerchantPrivateKey`、`apiV3KeyConfigured`、`hasPlatformCertificate`。
- `bepusdt`:`exchangeRate`、`callbackAck`(`ok`/`success`,非 `ok` 一律显示 `success`)、`hasApiToken`。

注意:读取字段名为 `appId`/`merchantId`(小写 d),提交参数名按字段名为 `appID`/`merchantID`。

### FinanceConfig(`finance` 字段)
`isOn`、`priceType`、`pricePeriod`、`trafficPrice`(`{"base"}`)、`bandwidthPrice`(`{"base","percentile","ranges":[{"minMB","maxMB","pricePerMB","totalPrice"}]}`)、`enablePlans`、`showPlansInUserSystem`、`enableTrafficPackages`、`showTrafficPackages`、`userCanChangePriceType`、`userCanChangePricePeriod`、`userUI`(`{"showPrices"}`)。

### 已发现的疑似缺陷与待确认项
- 账单列表 Count 带 `trafficRelated=true` 而 List 不带,`total` 与 `items` 口径可能不一致。
- `regional-prices/items/update` 的 `type` 仅用于换算,漏传会按 GB 倍率写入带宽区间。
- `methods/create` 的 `secret` 返回分支只对 `parentCode=custom` 生效,而 RPC 已拒绝 custom,属不可达代码。
- `methods/update` 中 alipay 的 `isProduction`、`alipayMode` 漏传会静默改变配置(沙箱/清空公钥)。
- 优惠券时间字段输入为 Unix 秒、输出为 RFC3339(`startsAt`/`endsAt`)或原始数值(`createdAt`/`updatedAt`),格式不对称。
- `settings/update` 对 `priceType`、`pricePeriod`、`bandwidthPercentile` 未做取值校验;两项设置写入非原子。
- 资金类写接口(余额调整、账单标记已付、订单完成/取消、退款完成)在处理器内无 `CreateLog`,审计情况待确认。
- `this.Fail` 的 HTTP 状态行与 JSON `code` 是否一致待确认。

<a id="m10"></a>

## 10. 管理端 · 用户与管理员(管理端)

**模块约定**

> 路由总数 21,来源 EdgeAdmin/internal/web/actions/default/api/admin/init.go(处理器:admin_management_resource.go、users.go、user_form.go、user_detail.go、user_mutations.go;管理员 10 条 init.go 111-120,用户 11 条 init.go 471-481)
说明:
- 路径前缀 `/api/admin`。管理员类 10 条经 `helpers.NewUserMustAuth(AdminModuleCodeAdmin)`,用户类 11 条经 `helpers.NewUserMustAuth(AdminModuleCodeUser)`;本模块没有许可证功能闸门。认证失败包络为 `{"code":N,"message":"...","data":null}`:未登录 401 `authentication required`;无对应模块权限 403 `permission denied`;系统未初始化 409 `system setup is required`;恶意请求/IP 不允许 403 空体;注入检测 403 文本 `Denied By WAF`(后两者的具体响应格式待确认)。
- 成功响应:`{"code":200,"message":"","data":{...}}`,下表"返回"只写 `data` 内字段;"data 为 {}"指成功但无业务字段。失败有两种来源:
  - 校验错误(处理器内 `this.Fail(msg)`):`{"code":400,"message":msg,"data":{...},"errors":null}`。HTTP 状态行未被显式设置(推断为 200,待确认)。
  - RPC 错误(`actionutils.Fail`,下表简写为"RPC错误"):HTTP 状态码与 JSON `code` 一致,按 gRPC 状态映射(InvalidArgument 400 / Unauthenticated 401 / PermissionDenied 403 / NotFound 404 / AlreadyExists 409 / ResourceExhausted 429 / Unavailable 503 / DeadlineExceeded 504 / 其他 500;Unknown 错误按文本关键字推断)。`message` 为 `AdminCommon_ServerError` 中文文案(最终渲染文本待确认)加 `（原因）`;原因含 password=/token=/secret= 等敏感词时替换为"服务处理失败，请检查相关配置后重试。"。
- 参数传递:TeaGo 只读 URL 查询串和 POST 表单(`application/x-www-form-urlencoded` 或 multipart),**不解析 JSON 请求体**。参数名取处理器 params struct 字段名,先按原样精确匹配,再按首字母小写匹配(如 `AdminId` 对应 `AdminId`/`adminId`),大小写敏感。缺省参数取零值(数字 0、字符串空、布尔 false),`[]string` 参数用重复键传多个值(如 `moduleCodes=a&moduleCodes=b`)。布尔值 `on/true/yes/enabled` 为 true,其余按 `types.Bool` 转换(具体规则待确认)。
- **布尔字段缺省即 false**:`admins/update`、`users/update` 是整体覆盖语义,漏传 `isOn` 会停用账号,漏传 `otpOn` 会关闭 OTP,漏传 `canLogin` 会禁止登录,调用方须带全量字段。
- CSRF:所有 POST 处理器参数里都声明了 `CSRF`,须携带表单(或查询串)参数 `csrfToken`,与会话 `Sid` 配对,有效期 1800 秒,令牌来源 `GET /api/admin/session` 的 `data.csrfToken`(或 `GET /csrf/token` 的 `data.token`)。失败返回 HTTP 403 + 纯文本 `表单已失效，请刷新页面后重试(001)`(非 JSON)。下表 POST 行不再重复列出 `csrfToken`(均必填)。GET 接口不校验 CSRF。
- 分页(`admins`、`users`):`page` int64,<1 取 1;`size` int64,<1 取 20,>100 取 100(截断,不报错)。响应回显实际生效的 `page`/`size`,并带 `total`。
- 时间字段:`createdTime`、`accessedTime` 为 `YYYY-MM-DD HH:mm:ss`(服务器本地时区,0 值为空串);实名信息 `identities[]` 中的 `createdAt` 等为原始 int64(单位待确认,疑似 Unix 秒)。
- 操作日志:管理员/用户的写接口均调用 `CreateLogInfo`。其中 `users/create`、`users/update`、`users/toggle`、`users/delete`、`users/features` 使用 `defer` 记录,即使操作失败也会写日志(`users/create` 失败时对象 ID 为 0)。
- 敏感信息:`admins/access-keys/create`、`users/access-keys/create` 的 `secret` 明文只在创建响应返回一次,其余接口只返回 `uniqueId`;`admins/otp-qrcode` 返回含 MFA 密钥的二维码图片;`users/detail` 返回实名证件号 `number`。下表不出现任何真实值。
- 取值范围中标注"RPC"的校验位于 EdgeAPI 服务层(`admin` 服务、`users/service_user.go`),处理器本身只做 `id>0` 校验;`id<=0` 的错误文案为 `invalid administrator id` / `invalid user id` / `invalid access key id`。

| 模块 | 接口名称 | 方法 | 路径 | 请求参数 | 响应数据 | 主要错误 | 功能说明 | 认证/权限 |
|---|---|---|---|---|---|---|---|---|
| admin-09-users-admins | 管理员列表 | GET | `/api/admin/admins` | 查询参数:<br>`page` int64,可选,默认 1<br>`size` int64,可选,默认 20,最大 100<br>`keyword` string,可选,匹配字段待确认<br>`hasWeakPassword` bool,可选,默认 false,为 true 时只列弱密码管理员(过滤语义依据 RPC 字段名推断,待确认) | `{"items":[Admin],"total":int64,"page":int64,"size":int64}`<br>Admin 键:`id`、`username`、`fullname`、`isOn`、`isSuper`、`canLogin`、`otpEnabled`、`hasWeakPassword`、`createdTime` | 错误:RPC错误 | 分页查询已启用(未删除)的管理员。 | 登录+admin 模块 |
| admin-09-users-admins | 管理员表单数据 | GET | `/api/admin/admins/form` | 查询参数:<br>`adminId` int64,可选;<=0 时返回空 `admin`(新建表单) | `{"modules":[{"code":string,"name":string}],"admin":Admin或null}`<br>`admin` 键同列表项,另加 `moduleCodes`([]string,已授权模块码) | 错误:400 `administrator not found`;RPC错误 | 返回可授权模块清单(名称按当前语言)与指定管理员详情,供新建/编辑页使用。可授权模块码范围:dashboard/server/node/dns/user/admin/finance/plan/ticket/log/setting(以 `IsSupportedAdminModule` 为准,common 是否包含待确认)。 | 登录+admin 模块 |
| admin-09-users-admins | 管理员详情 | GET | `/api/admin/admins/detail` | 查询参数:<br>`adminId` int64,必填,>0 | `{"admin":Admin(含 moduleCodes),"accessKeys":[{"id":int64,"uniqueId":string,"description":string,"isOn":bool,"accessedTime":string}]}` | 错误:400 `invalid administrator id`/`administrator not found`;RPC错误 | 管理员信息及其访问密钥列表(不含 secret)。 | 登录+admin 模块 |
| admin-09-users-admins | 管理员 MFA 二维码 | GET | `/api/admin/admins/otp-qrcode` | 查询参数:<br>`adminId` int64,必填,>0<br>`download` bool,可选,默认 false,为 true 时加 `Content-Disposition: attachment; filename="EdgeRoute-MFA-<username>.png"` | 成功:**非 JSON**,响应体为 `image/png` 二进制(256px,Medium 纠错),头 `Cache-Control: no-store`<br>失败走 JSON 包络:400 `invalid administrator id`/`administrator not found`/`administrator MFA is not enabled`/`administrator MFA secret is missing`;RPC错误 | 见模块约定 | 以该管理员 OTP 密钥生成 TOTP 配置二维码(issuer 为后台系统名称)。二维码内含 MFA 密钥,仅 admin 模块权限即可通过 GET 读取,且无 CSRF/二次确认,注意泄露面。 | 登录+admin 模块 |
| admin-09-users-admins | 创建管理员 | POST | `/api/admin/admins/create` | 表单:<br>`username` string,必填(RPC),`^[a-zA-Z0-9_]+$`,最长 100<br>`password` string,必填(RPC),8-128 字符,首尾无空白,无控制字符<br>`fullname` string,必填(RPC),1-100 字符<br>`moduleCodes` []string,可选,须为受支持模块码,否则 `不支持的管理员模块：<code>`;空串忽略,重复去重;`isSuper=true` 时忽略<br>`isSuper` bool,默认 false,仅超管可设为 true(RPC)<br>`canLogin` bool,默认 false<br>`otpOn` bool,默认 false<br>`isOn` bool:**被忽略**,创建默认启用 | `{"adminId":int64}` | 错误:400 `不支持的管理员模块：<code>`;RPC错误(用户名/密码/昵称校验、非超管授予超管等) | 创建管理员。顺序:创建 → 按 `otpOn` 写 OTP 登录配置(失败则回滚删除该管理员并返回错误)→ 刷新模块映射 → 写日志。`canLogin` 漏传即不能登录。 | 登录+admin 模块+CSRF |
| admin-09-users-admins | 更新管理员 | POST | `/api/admin/admins/update` | 表单(整体覆盖):<br>`adminId` int64,必填,>0<br>`username` string,同创建(空值是否被接受待确认)<br>`password` string,可选,留空表示不修改;非空时 8-128 字符规则同创建<br>`fullname` string,1-100 字符(空值是否被接受待确认)<br>`moduleCodes` []string,同创建<br>`isSuper` bool,默认 false<br>`isOn` bool,默认 false(漏传即停用)<br>`canLogin` bool,默认 false<br>`otpOn` bool,默认 false(漏传即关闭 OTP) | data 为 `{}` | 错误:400 `invalid administrator id`/`不支持的管理员模块：<code>`;RPC错误(含 `at least one active super administrator is required`:降级/停用/禁登最后一个启用的超管) | 更新管理员并同步 OTP、刷新模块映射、写日志。OTP 开启时若无登录记录会生成新密钥;`otpOn=false` 且无记录则不动作。 | 登录+admin 模块+CSRF |
| admin-09-users-admins | 删除管理员 | POST | `/api/admin/admins/delete` | 表单:<br>`adminId` int64,必填,>0 | data 为 `{}` | 错误:400 `invalid administrator id`;RPC错误(含 `can not delete current administrator`;删除最后一个超管是否被拒待确认) | 删除(停用标记)管理员,刷新模块映射并写日志。 | 登录+admin 模块+CSRF |
| admin-09-users-admins | 创建管理员访问密钥 | POST | `/api/admin/admins/access-keys/create` | 表单:<br>`adminId` int64,必填,>0<br>`description` string,可选,自动 trim | `{"accessKey":{"id":int64,"uniqueId":string,"secret":string}}` | 错误:400 `invalid administrator id`;RPC错误 | 为管理员创建 API 访问密钥。`secret` 明文仅此一次返回,请调用方即时保存。不校验 `adminId` 对应管理员是否存在(由 RPC 决定,待确认)。 | 登录+admin 模块+CSRF |
| admin-09-users-admins | 启停管理员访问密钥 | POST | `/api/admin/admins/access-keys/toggle` | 表单:<br>`adminId` int64,必填,>0<br>`accessKeyId` int64,必填,>0<br>`isOn` bool,默认 false(漏传即停用) | data 为 `{}` | 错误:`invalid administrator id`;`invalid access key id`;`access key does not belong to this administrator`;RPC错误 | 启用/停用访问密钥。先用 `adminId` 拉取该管理员的密钥列表,`accessKeyId` 不在其中则拒绝(er-v45 起,与用户侧 `validateCustomerAccessKeyOwnership` 一致)。 | 登录+admin 模块+CSRF |
| admin-09-users-admins | 删除管理员访问密钥 | POST | `/api/admin/admins/access-keys/delete` | 表单:<br>`adminId` int64,必填,>0<br>`accessKeyId` int64,必填,>0 | data 为 `{}` | 错误:同 toggle;RPC错误 | 删除访问密钥。同样先校验归属(er-v45 起)。 | 登录+admin 模块+CSRF |
| admin-09-users-admins | 用户列表 | GET | `/api/admin/users` | 查询参数:<br>`page` int64,可选,默认 1<br>`size` int64,可选,默认 20,最大 100<br>`keyword` string,可选,匹配字段待确认<br>`verifying` bool,可选,默认 false,为 true 时只列待实名审核用户<br>`mobileIsVerified` int32,可选,默认 -1(推断为不过滤),其他取值含义待确认 | `{"items":[User],"total":int64,"page":int64,"size":int64,"countVerifying":int64}`<br>User 键:`id`、`username`、`fullname`、`company`、`email`、`mobile`、`verifiedMobile`、`isOn`、`identityState`(`unsubmitted`/`verified`/`rejected`)、`registeredIP`、`createdTime`、`cluster`(`{"id","name"}`或 null) | 错误:RPC错误 | 分页查询已启用用户;`countVerifying` 为全局待审核用户数(不受过滤条件影响)。`identityState`:个人/企业任一已通过为 verified,否则有驳回为 rejected,否则 unsubmitted。每个用户额外 2 次实名 RPC(并发 8),存在 N+1 调用,`size` 较大时较慢。 | 登录+user 模块 |
| admin-09-users-admins | 用户表单数据 | GET | `/api/admin/users/form` | 查询参数:<br>`userId` int64,可选;<=0 时 `user` 为 null | `{"globalRequirePlan":bool,"clusters":[{"id":int64,"name":string}],"bandwidthAlgorithms":[{"value":string,"label":string}],"user":User或null}`<br>`bandwidthAlgorithms` 取值:空串(使用系统默认)/`secondly`(按秒峰值)/`avg`(平均带宽)<br>`user` 键:`id`、`username`、`fullname`、`company`、`mobile`、`tel`、`email`、`remark`、`isOn`、`clusterId`、`bandwidthAlgo`、`otpEnabled`、`requirePlan` | 错误:400 `user not found`;RPC错误 | 返回新建/编辑用户所需选项:全局"必须套餐"开关(来自财务服务设置)、已启用集群(最多取前 1000 个,按 ID 升序)、带宽算法选项,以及可选的用户详情。 | 登录+user 模块 |
| admin-09-users-admins | 用户详情 | GET | `/api/admin/users/detail` | 查询参数:<br>`userId` int64,必填,>0 | `{"user":{...},"accessKeys":[{"id","uniqueId","description","isOn","accessedTime"}],"features":[{"name","code","description","supportPlan":bool,"enabled":bool}],"identities":[{"id","orgType","type","realName","number","fileIds","status","createdAt","updatedAt","submittedAt","rejectedAt","verifiedAt","rejectReason"}]}`<br>`user` 键:`id`、`username`、`fullname`、`company`、`email`、`verifiedEmail`、`mobile`、`verifiedMobile`、`tel`、`remark`、`isOn`、`cluster`、`createdTime`、`registeredIP`、`registeredRegion`、`identityState`、`otpEnabled`、`bandwidthAlgo`、`countSites`、`countAccessKeys`、`countPlans`(仅统计可用套餐)、`priceType`、`pricePeriod`、`serversEnabled`、`requirePlan` | 错误:400 `invalid user id`/`user not found`;RPC错误 | 并发聚合多项 RPC(站点数、密钥、套餐数、计价信息、服务启用状态、个人与企业实名记录、功能定义与已授权功能)。`features` 以 EdgeCommon 内置功能目录为准(而非 RPC 返回),`enabled` 表示该用户是否已授权。`identities` 含实名证件号 `number` 与证件文件 ID `fileIds`,属敏感信息,仅 user 模块权限即可读取。`identities` 字段取值枚举(`orgType`/`type`/`status`)待确认。 | 登录+user 模块 |
| admin-09-users-admins | 创建用户访问密钥 | POST | `/api/admin/users/access-keys/create` | 表单:<br>`userId` int64,必填,>0<br>`description` string,可选,自动 trim | `{"accessKey":{"id":int64,"uniqueId":string,"secret":string}}` | 错误:400 `invalid user id`;RPC错误 | 为用户创建 API 访问密钥。`secret` 明文仅此一次返回。 | 登录+user 模块+CSRF |
| admin-09-users-admins | 启停用户访问密钥 | POST | `/api/admin/users/access-keys/toggle` | 表单:<br>`userId` int64,必填,>0<br>`accessKeyId` int64,必填,>0<br>`isOn` bool,默认 false(漏传即停用) | data 为 `{}` | 错误:400 `invalid user id`/`invalid access key id`/`access key does not belong to this user`(归属校验失败,依据校验函数推断文案);RPC错误 | 先列出该用户全部启用密钥,确认 `accessKeyId` 属于该用户再更新,避免越权(与管理员侧同类接口不同)。 | 登录+user 模块+CSRF |
| admin-09-users-admins | 删除用户访问密钥 | POST | `/api/admin/users/access-keys/delete` | 表单:<br>`userId` int64,必填,>0<br>`accessKeyId` int64,必填,>0 | data 为 `{}` | 错误:同上;RPC错误 | 同样先校验归属后删除。 | 登录+user 模块+CSRF |
| admin-09-users-admins | 更新用户功能授权 | POST | `/api/admin/users/features` | 表单:<br>`userId` int64,必填,>0<br>`featureCodes` []string,可选,重复键传多个;为空则清空授权(整体覆盖),合法功能码范围待确认 | data 为 `{}` | 错误:400 `invalid user id`;RPC错误 | 用 `featureCodes` 整体覆盖该用户的功能授权。使用 `defer` 记日志,失败时也会记录。 | 登录+user 模块+CSRF |
| admin-09-users-admins | 创建用户 | POST | `/api/admin/users/create` | 表单:<br>`username` string,必填(RPC),`^[a-zA-Z0-9_]+$`,3-64 字符,重复报 `username is already in use`<br>`password` string,必填(RPC),8-128 字符,首尾无空白,无控制字符,须含大写/小写/数字/符号(比管理员更严)<br>`fullname` string,必填(RPC),1-200 字符<br>`company` string,可选<br>`mobile`/`tel` string,可选,匹配 `^[0-9+(). -]+$`,最长 64<br>`email` string,可选,格式规则待确认<br>`remark` string,可选,最长 2000<br>`clusterId` int64,必填(RPC),否则 `node cluster is required`<br>`otpOn` bool,默认 false<br>`requirePlan` bool,默认 false | `{"userId":int64}` | 错误:400 `user service returned an invalid user id`;RPC错误 | 创建用户,来源标记为 `admin:<当前管理员ID>`。使用 `defer` 记日志,失败时也会写入(用户 ID 为 0)。 | 登录+user 模块+CSRF |
| admin-09-users-admins | 更新用户 | POST | `/api/admin/users/update` | 表单(整体覆盖):<br>`userId` int64,必填,>0<br>`username`/`fullname`/`company`/`mobile`/`tel`/`email`/`remark` 同创建规则<br>`password` string,留空是否保持原密码待确认;非空须符合创建规则<br>`isOn` bool,默认 false(漏传即停用用户)<br>`clusterId` int64,必填(RPC)<br>`bandwidthAlgo` string,空/`secondly`/`avg`<br>`otpOn` bool,默认 false(固定同步 OTP,漏传即关闭)<br>`requirePlan` bool,默认 false | data 为 `{}` | 错误:400 `invalid user id`;RPC错误(`user not found`、`username is already in use` 等) | 更新用户资料与启停状态,固定管理 OTP(`ManageOTP=true`)。使用 `defer` 记日志。 | 登录+user 模块+CSRF |
| admin-09-users-admins | 启停用户 | POST | `/api/admin/users/toggle` | 表单:<br>`userId` int64,必填,>0<br>`isOn` bool,默认 false(漏传即停用) | data 为 `{}` | 错误:400 `invalid user id`;RPC错误 | 仅切换用户启用状态。日志码复用 `User_LogUpdateUser`,与"更新用户"无法区分。 | 登录+user 模块+CSRF |
| admin-09-users-admins | 删除用户 | POST | `/api/admin/users/delete` | 表单:<br>`userId` int64,必填,>0 | data 为 `{}` | 错误:400 `invalid user id`;RPC错误 | 删除用户(软删除语义待确认);关联订单/账户/站点的处理待确认。使用 `defer` 记日志。 | 登录+user 模块+CSRF |


### 公共结构

### Admin
`id`、`username`、`fullname`、`isOn`、`isSuper`、`canLogin`、`otpEnabled`(OTP 登录配置存在且启用)、`hasWeakPassword`、`createdTime`;表单与详情接口另有 `moduleCodes`([]string)。

### AccessKey(列表项)
`id`、`uniqueId`、`description`、`isOn`、`accessedTime`(最近使用时间);仅创建响应含 `secret`。

### 已发现的疑似缺陷与待确认项
- 已修复(er-v45):`admins/access-keys/toggle`、`admins/access-keys/delete` 现需传 `adminId` 并校验密钥归属(`validateAdminAccessKeyOwnership`)。
- `admins/create` 的 `isOn` 参数被忽略;`admins/update`、`users/update` 漏传布尔字段会停用账号/关闭 OTP/禁止登录。
- `admins/otp-qrcode` 成功响应为 PNG 二进制,非 JSON 包络;二维码含 MFA 密钥,GET 即可获取。
- `users/detail` 返回实名证件号 `number` 与证件文件 ID;`users/create` 失败也写操作日志。
- 用户密码强度(须含四类字符)严于管理员密码(仅长度与空白规则)。
- `users` 列表存在 N+1 实名查询。
- 待确认:`this.Fail` 的 HTTP 状态行;`AdminCommon_ServerError` 渲染文本;`keyword` 匹配字段;`mobileIsVerified` 取值含义;`hasWeakPassword` 过滤语义;`featureCodes` 合法范围;`admins/update` 对空 `username`/`fullname` 的处理;`users/update` 空密码语义;删除最后一个超管/删除用户的后续影响;`common` 模块是否可授权。

<a id="m11"></a>

## 11. 管理端 · 工单/消息/通知(管理端)

**模块约定**

> 路由总数 26,来源 EdgeAdmin/internal/web/actions/default/api/admin/init.go
说明:
- 路径前缀 `/api/admin`,均经 `helpers.NewUserMustAuth(<模块>)`。消息 4 条模块为 `AdminModuleCodeSetting`,通知 15 条为 `AdminModuleCodeAdmin`,工单 7 条为 `AdminModuleCodeTicket`。认证失败包络为 `{"code":N,"message":"...","data":null}`:未登录 401 `authentication required`;无模块权限 403 `permission denied`;系统未初始化 409 `system setup is required`;恶意请求/IP 不允许 403 空体;注入检测 403 文本 `Denied By WAF`。
- 成功响应:`{"code":200,"message":"","data":{...}}`(TeaGo `Success`),下表"返回"只写 `data` 内字段。`this.Fail(msg)` 失败为 `{"code":400,"message":msg,"data":...,"errors":[...]}`(JSON `code` 已确认;HTTP 状态行是否同为 400 待确认)。`actionutils.Fail(this, err)` 失败会设置 HTTP 状态码(gRPC InvalidArgument→400,Unauthenticated→401,PermissionDenied→403,NotFound→404,AlreadyExists→409,ResourceExhausted→429,Unavailable→503,DeadlineExceeded→504,其他→500),message 为 `AdminCommon_ServerError` 文本 + `（原因）`(服务端错误文本具体内容待确认;含 password=/token=/secret 等敏感词的原因会被替换为固定提示)。本模块多数校验错误为英文短句。
- CSRF:所有 POST 处理器的参数里都声明了 `CSRF`,需要携带表单参数 `csrfToken`(与会话 `Sid` 配对,有效期 1800 秒)。令牌来源 `GET /api/admin/session` 的 `data.csrfToken`。校验失败返回 HTTP 403 + 纯文本 `表单已失效，请刷新页面后重试(001)`,不是 JSON 包络。是否支持 JSON 请求体携带 `csrfToken` 待确认。下表 POST 行的"主要请求参数"默认不再重复列出 `csrfToken`(均必填)。
- 参数名:取自处理器 params struct 的字段名(首字母小写);大小写匹配规则未在 TeaGo 源码中验证,待确认。
- 分页(`normalizePagination`):`page` 默认 1(<1 取 1),`size` 默认 20(<1 取 20,>100 取 100);响应回显 `page`/`size`/`total`。
- 变更类接口一般通过 `CreateLogInfo` 记录操作日志;`POST /messages/alert-state` 不记日志。

| 模块 | 接口名称 | 方法 | 路径 | 请求参数 | 响应数据 | 主要错误 | 功能说明 | 认证/权限 |
|---|---|---|---|---|---|---|---|---|
| admin-10-tickets-messages-notifications | 未读消息列表 | GET | `/api/admin/messages` | 查询参数:<br>`page` int,可选,默认 1<br>`size` int,可选,默认 20,最大 100 | `data`:`{"items":[Message],"total":int,"page":int,"size":int}`<br>`Message`:`id,role,isRead,body,level,type,createdAt,createdTime`(格式 Y-m-d H:i:s)`,params`(字符串形式的 ParamsJSON)`,alertState,silencedUntil,acknowledgedAt`,可选 `cluster{id,name}`、`node{id,name}` | 错误:RPC 失败走 `actionutils.Fail` | 只列出当前管理员的未读消息(`ListUnreadMessages`),`total` 为未读总数(`CountUnreadMessages`),已读消息不会返回。`alertState` 为有效告警状态:为空视为 `open`;`silenced` 且已过期视为 `open`。 | 需登录,模块 Setting |
| admin-10-tickets-messages-notifications | 标记消息已读 | POST | `/api/admin/messages/read` | `messageIds` int64 数组,必填,1~500 个<br>`csrfToken` string 必填 | `data` 为空对象 | 错误:`message ids are required`(400,数量为 0 或 >500) | 将指定消息置为已读(`UpdateMessagesRead`),日志 `Message_LogReadMessages`。 | 需登录,模块 Setting,CSRF |
| admin-10-tickets-messages-notifications | 全部标记已读 | POST | `/api/admin/messages/read-all` | 仅 `csrfToken` string 必填 | `data` 为空对象 | 见模块约定 | 将当前管理员全部消息置为已读(`UpdateAllMessagesRead`),日志 `Message_LogReadAll`。 | 需登录,模块 Setting,CSRF |
| admin-10-tickets-messages-notifications | 更新告警状态 | POST | `/api/admin/messages/alert-state` | `messageIds` int64 数组,必填,1~500 个<br>`state` string,管理端不校验;EdgeAPI 要求 `open`/`acknowledged`/`silenced`<br>`silencedUntil` int64,Unix 秒,`state=silenced` 时必须大于当前时间<br>`csrfToken` 必填 | `data` 为空对象 | 错误:`message ids are required`;EdgeAPI 返回 `invalid alert state`、`silence expiry must be in the future`(经 `actionutils.Fail`,按文本映射为 400) | 设置消息的告警状态:`acknowledged` 会记录确认时间与确认管理员;`silenced` 静默到指定时间。**不记操作日志**(与 read/read-all 不一致)。 | 需登录,模块 Setting,CSRF |
| admin-10-tickets-messages-notifications | 通知选项 | GET | `/api/admin/notifications/options` | 无 | `data`:<br>`admins[{id,username,fullname}]`(仅启用,最多 2000,分批 200)<br>`groups[{id,name}]`(仅启用)<br>`instances[{id,name,mediaType,mediaName,userDescription}]`(仅启用,最多 1000)<br>`medias[{type,name,description,userDescription}]`(仅启用)<br>`canManageSecrets` bool(当前是否超管)<br>响应头 `Cache-Control: no-store` | 见模块约定 | 返回收件人/分组/通道/媒介类型的下拉候选项。 | 需登录,模块 Admin |
| admin-10-tickets-messages-notifications | 通知收件人列表 | GET | `/api/admin/notifications/recipients` | 查询参数:`page` 默认 1,`size` 默认 20 最大 100,`keyword` string 可选 | `data`:`{"items":[Recipient],"total":int,"page":int,"size":int}`<br>`Recipient`:`id,admin{id,username,fullname},instance{id,name},groups[{id,name}],groupIds[],user,description,timeFrom,timeTo,isOn` | 见模块约定 | 分页查询通知收件人。**疑似缺陷**:管理员或通道对象为 nil 的记录会被跳过,导致 `items` 数量可能少于 `size`,而 `total` 仍包含它们。 | 需登录,模块 Admin |
| admin-10-tickets-messages-notifications | 保存通知收件人 | POST | `/api/admin/notifications/recipients/save` | `recipientId` int64,≤0 新建,>0 更新<br>`adminId` int64,必填,>0<br>`instanceId` int64,必填,>0<br>`groupIds` int64 数组,可选,≤100 个,每项 >0,去重排序<br>`user` string,可选,≤300 字符<br>`description` string,可选,≤500 字符<br>`timeFrom`/`timeTo` string,须同时为空或同时为 `HH:MM:SS`(8 位)<br>`isOn` bool,**仅更新时生效,创建时被忽略**<br>`csrfToken` 必填 | 创建时 `data.recipientId` int64;更新时 `data` 为空 | 错误:`invalid notification recipient`、`too many notification groups`、`invalid notification group id`、`notification time window requires both start and end`、`notification time must use HH:MM:SS` | 新建或更新通知收件人(管理员 + 通道 + 分组 + 生效时间窗)。 | 需登录,模块 Admin,CSRF |
| admin-10-tickets-messages-notifications | 删除通知收件人 | POST | `/api/admin/notifications/recipients/delete` | `recipientId` int64,必填,>0<br>`csrfToken` 必填 | `data` 为空对象 | 错误:`invalid notification recipient id` | 删除指定通知收件人。 | 需登录,模块 Admin,CSRF |
| admin-10-tickets-messages-notifications | 通知分组列表 | GET | `/api/admin/notifications/groups` | 无 | `data`:`{"items":[{"id":int64,"name":string,"isOn":bool}]}` | 见模块约定 | 返回全部通知分组,不分页。 | 需登录,模块 Admin |
| admin-10-tickets-messages-notifications | 保存通知分组 | POST | `/api/admin/notifications/groups/save` | `groupId` int64,≤0 新建,>0 更新<br>`name` string,必填,trim 后 1~50 字符<br>`isOn` bool,**仅更新时生效**<br>`csrfToken` 必填 | 创建时 `data.groupId` int64;更新时 `data` 为空 | 错误:`notification group name is required and must not exceed 50 characters` | 新建或更新通知分组。 | 需登录,模块 Admin,CSRF |
| admin-10-tickets-messages-notifications | 删除通知分组 | POST | `/api/admin/notifications/groups/delete` | `groupId` int64,必填,>0<br>`csrfToken` 必填 | `data` 为空对象 | 错误:`invalid notification group id` | 删除指定通知分组。 | 需登录,模块 Admin,CSRF |
| admin-10-tickets-messages-notifications | 通知通道列表 | GET | `/api/admin/notifications/instances` | 查询参数:`page` 默认 1,`size` 默认 20 最大 100,`keyword` string 可选 | `data`:`{"items":[Instance],"total":int,"page":int,"size":int,"canManageSecrets":bool}`<br>`Instance`:`id,name,isOn,media{type,name},description,hashLife`;**仅超管**额外含 `paramsJSON`、`rateJSON`(规范化后缩进的 JSON 字符串)<br>响应头 `Cache-Control: no-store` | 见模块约定 | 分页查询通知通道实例。**疑似问题**:向超管返回未脱敏的 `paramsJSON`,其中可能包含通道凭据。 | 需登录,模块 Admin |
| admin-10-tickets-messages-notifications | 保存通知通道 | POST | `/api/admin/notifications/instances/save` | **仅超管**<br>`instanceId` int64,≤0 新建,>0 更新<br>`name` string,必填,1~100 字符<br>`mediaType` string,必填,1~64 字符<br>`paramsJSON` string,JSON 对象,空视为 `{}`,≤64KiB<br>`rateJSON` string,JSON 对象,空视为 `{}`,≤64KiB<br>`hashLife` int32,0~31536000<br>`description` string,可选,≤500 字符<br>`isOn` bool,**仅更新时生效**<br>`csrfToken` 必填 | 创建时 `data.instanceId` int64;更新时 `data` 为空 | 错误:`only a super administrator can manage notification channel credentials`、`invalid notification channel`、`invalid channel parameters: ...`、`invalid channel rate limit: ...` | 新建或更新通知通道(邮件/短信/Webhook 等媒介实例及其凭据、限流)。 | 需登录,模块 Admin,仅超管,CSRF |
| admin-10-tickets-messages-notifications | 测试通知通道 | POST | `/api/admin/notifications/instances/test` | **仅超管**<br>`instanceId` int64,可选<br>`mediaType` string,必填<br>`paramsJSON` string,通道参数 JSON<br>`recipient` string,必填:`mediaType=email` 时须为纯邮箱地址,否则非空且 ≤512 字符<br>`csrfToken` 必填 | 成功 `data` 为空对象 | 错误:`invalid test email recipient`、`invalid test recipient`、非超管同上;发送失败走 `actionutils.Fail` | 调用 `SendMediaMessage` 向 `recipient` 发送一条主题为 `EdgeRoute test message` 的测试消息。 | 需登录,模块 Admin,仅超管,CSRF |
| admin-10-tickets-messages-notifications | 删除通知通道 | POST | `/api/admin/notifications/instances/delete` | **仅超管**<br>`instanceId` int64,必填,>0<br>`csrfToken` 必填 | `data` 为空对象 | 错误:`invalid notification channel id`、非超管同上 | 删除指定通知通道。 | 需登录,模块 Admin,仅超管,CSRF |
| admin-10-tickets-messages-notifications | 通知任务列表 | GET | `/api/admin/notifications/tasks` | 查询参数:`page` 默认 1,`size` 默认 20 最大 100<br>`status` int32,默认 0,取值 0~3(<0 或 >3 静默改为 0);RPC 枚举 0=None,1=Sending,2=Success,3=Failed | `data`:`{"items":[Task],"counts":[int,int,int,int],"status":int,"total":int,"page":int,"size":int}`<br>`counts` 下标 0~3 对应各状态数量;`total` 等于 `counts[status]`<br>`Task`:`id,user,instance{id,name}`(可能为 nil)`,subject,body,status,result{isOk,error,response},createdTime,sentTime`(格式 `yyy-MM-d HH:mm:ss`,时间 ≤0 为空串) | 见模块约定 | 分页查询通知发送任务及各状态计数。`status=0` 在列表中是否表示"全部"待确认。`result.error`/`response` 来自第三方通道,可能含敏感内容(是否在 API 侧脱敏待确认)。 | 需登录,模块 Admin |
| admin-10-tickets-messages-notifications | 重试通知任务 | POST | `/api/admin/notifications/tasks/retry` | `taskId` int64,必填,>0<br>`csrfToken` 必填 | `data` 为空对象 | 错误:`invalid notification task id` | 通过 `UpdateMessageTaskStatus` 把任务状态置为 None(重新排队),由后台重新发送。 | 需登录,模块 Admin,CSRF |
| admin-10-tickets-messages-notifications | 删除通知任务 | POST | `/api/admin/notifications/tasks/delete` | `taskId` int64,必填,>0<br>`csrfToken` 必填 | `data` 为空对象 | 错误:`invalid notification task id` | 删除指定通知任务。 | 需登录,模块 Admin,CSRF |
| admin-10-tickets-messages-notifications | 通知发送日志 | GET | `/api/admin/notifications/logs` | 查询参数:`page` 默认 1,`size` 默认 20 最大 100 | `data`:`{"items":[Log],"total":int,"page":int,"size":int}`<br>`Log`:`id,isOk,error,response,createdTime,task{同通知任务列表中的 Task}` | 见模块约定 | 分页查询通知发送日志;`MessageTask` 为 nil 的日志会被跳过(`items` 可能少于 `size`)。`error`/`response` 为第三方原文。 | 需登录,模块 Admin |
| admin-10-tickets-messages-notifications | 工单列表 | GET | `/api/admin/tickets` | 查询参数:`page` 默认 1,`size` 默认 20 最大 100<br>`status` string,可选(trim),空或 `none`/`solved`/`closed`<br>`userId` int64,可选<br>`categoryId` int64,可选 | `data`:`{"items":[Ticket],"categories":[Category],"total":int,"page":int,"size":int}`<br>`Ticket`:`id,categoryId,userId,subject,status,createdAt,lastLogAt`,可选 `category{id,name,isOn}`、`user{id,username,fullname,company}`、`latestReply{...}`(列表不含 `body`)<br>`Category`:`id,name,isOn` | 错误:`invalid ticket status` | 分页查询用户工单,可按状态、用户、分类筛选。`latestReply` 的具体键与工单日志相同(待确认列表内是否完全一致)。 | 需登录,模块 Ticket |
| admin-10-tickets-messages-notifications | 工单详情 | GET | `/api/admin/tickets/detail` | 查询参数:`ticketId` int64,必填,>0 | `data`:`{"ticket":Ticket(含 body),"logs":[Log],"categories":[Category],"logsTruncated":bool}`<br>`Log`:`id,adminId,userId,ticketId,status,coment`(注意字段拼写为 `coment`)`,createdAt,isReadonly`,可选 `admin{id,username,fullname}`、`user{id,username,fullname}` | 错误:`invalid ticket id`;工单不存在为 `ticket not found`(`this.Fail`,code 400,**非 404**) | 读取工单及回复记录,最多返回 1000 条日志(每批 100),超出时 `logsTruncated=true`。 | 需登录,模块 Ticket |
| admin-10-tickets-messages-notifications | 回复工单 | POST | `/api/admin/tickets/reply` | `ticketId` int64,必填,>0<br>`status` string,必填,`none`/`solved`/`closed`(不可为空)<br>`comment` string,必填,trim 后 1~10000 字符<br>`csrfToken` 必填 | `data.ticketLogId` int64 | 错误:参数不合法统一为 `invalid ticket reply` | 管理员回复工单并同时设置工单状态。 | 需登录,模块 Ticket,CSRF |
| admin-10-tickets-messages-notifications | 删除工单 | POST | `/api/admin/tickets/delete` | `ticketId` int64,必填,>0<br>`csrfToken` 必填 | `data` 为空对象 | 错误:`invalid ticket id` | 删除指定工单。 | 需登录,模块 Ticket,CSRF |
| admin-10-tickets-messages-notifications | 工单分类列表 | GET | `/api/admin/tickets/categories` | 无 | `data`:`{"items":[{"id":int64,"name":string,"isOn":bool}]}` | 见模块约定 | 返回全部工单分类,不分页。 | 需登录,模块 Ticket |
| admin-10-tickets-messages-notifications | 保存工单分类 | POST | `/api/admin/tickets/categories/save` | `categoryId` int64,≤0 新建,>0 更新<br>`name` string,必填,2~64 字符<br>`isOn` bool,**仅更新时生效**<br>`csrfToken` 必填 | 创建时 `data.categoryId` int64;更新时 `data` 为空 | 错误:`ticket category name must contain 2 to 64 characters` | 新建或更新工单分类。 | 需登录,模块 Ticket,CSRF |
| admin-10-tickets-messages-notifications | 删除工单分类 | POST | `/api/admin/tickets/categories/delete` | `categoryId` int64,必填,>0<br>`csrfToken` 必填 | `data` 为空对象 | 错误:`invalid ticket category id` | 删除指定工单分类。 | 需登录,模块 Ticket,CSRF |


### 公共结构

- 消息 `Message.alertState`:`open`/`acknowledged`/`silenced`;`silencedUntil`、`acknowledgedAt` 为 Unix 秒(0 表示未设置,待确认)。
- 工单状态:`none`(处理中)/`solved`/`closed`,其中 `none` 的中文语义由处理器未说明,待确认。
- 通知任务状态:0=None(待发送/重新排队),1=Sending,2=Success,3=Failed。

<a id="m12"></a>

## 12. 管理端 · 系统设置(管理端)

**模块约定**

> 路由总数 32,来源 EdgeAdmin/internal/web/actions/default/api/admin/init.go
说明:
- 路径前缀 `/api/admin`,全部经 `helpers.NewUserMustAuth(AdminModuleCodeSetting)`。认证失败包络为 `{"code":N,"message":"...","data":null}`:未登录 401 `authentication required`;无模块权限 403 `permission denied`;系统未初始化 409 `system setup is required`;恶意请求/IP 不允许 403 空体;注入检测 403 文本 `Denied By WAF`;请求安全检查失败的响应格式(`writeSecurityDenied`)待确认。
- 成功响应:`{"code":200,"message":"","data":{...}}`(TeaGo `Success`),下表"返回"只写 `data` 内字段;"data 为空"指成功但无业务字段。`this.Fail(msg)` 失败为 `{"code":400,"message":msg,"data":...,"errors":[...]}`(JSON `code` 已确认,HTTP 状态行是否同为 400 待确认)。`actionutils.Fail(this, err)` 会设置 HTTP 状态码(gRPC InvalidArgument→400,Unauthenticated→401,PermissionDenied→403,NotFound→404,AlreadyExists→409,ResourceExhausted→429,Unavailable→503,DeadlineExceeded→504,其他→500;Unknown 错误按文本关键字推断),message 为 `AdminCommon_ServerError` 文本 + `（原因）`(具体文本待确认;含 password=/token=/secret 等敏感词的原因被替换为"服务处理失败,请检查相关配置后重试。";SMTP 535 认证失败为固定中文提示)。校验错误文案中英文混用。
- CSRF:本模块所有 POST 处理器的参数里都声明了 `CSRF`,需携带表单参数 `csrfToken`(与会话 `Sid` 配对,有效期 1800 秒),来源 `GET /api/admin/session` 的 `data.csrfToken`。失败返回 HTTP 403 + 纯文本 `表单已失效，请刷新页面后重试(001)`,不是 JSON 包络。是否支持 JSON 请求体携带 `csrfToken` 待确认。下表 POST 行不再重复列出 `csrfToken`(均必填),`POST /settings/general/asset` 为 multipart。
- 参数名取自处理器 params struct 字段名(首字母小写,部分字段有 `alias` tag);大小写匹配规则未在 TeaGo 源码中验证,待确认。
- 各 `POST /settings/<分组>` 为"整组覆盖提交":传入各字段后整体写入该分组配置,变更后一般记录操作日志(下表个别接口注明不记)。
- 密钥类字段(API 密钥、客户端密钥、AccessKey Secret 等)响应中只返回是否已配置(`xxxConfigured`)或掩码,请求中留空表示保留旧值。
- 分页(仅 `/settings/history`):用 `offset`/`size`,不是 `page`。

| 模块 | 接口名称 | 方法 | 路径 | 请求参数 | 响应数据 | 主要错误 | 功能说明 | 认证/权限 |
|---|---|---|---|---|---|---|---|---|
| admin-11-settings | 系统设置总览 | GET | `/api/admin/settings` | 无 | `data` 聚合对象:<br>`general{productName,adminSystemName,showFinance,showVersion,version,defaultPageSize,timeZone,dnsResolverType,faviconFileId,logoFileId,faviconURL,logoURL}`<br>`portal{landingPageEnabled,productName,userSystemName,title,description,footerText,privacyPolicyURL,termsURL,faviconFileId,logoFileId,faviconURL,logoURL}`<br>`users{registrationEnabled,requireApproval,requireIdentity,identityProvider,identityRestrictedFeatures,identityAliyunMarket{appCodeConfigured,endpoint},identityAliyun{accessKeyId,accessKeySecretConfigured,region,endpoint},identityTencent{secretId,secretKeyConfigured,region,endpoint},requireMobile,clusterId,features,emailSenderConfigured,emailVerification{enabled,subject,body},passwordReset{enabled,lifeMinutes,subject,body}}`<br>`authentication{publicBaseURL,publicBaseURLSource,oauthProviders[{isOn,code,name,clientId,clientSecretConfigured,isBuiltin,kind,authorizationURL,tokenURL,userInfoURL,scopes,subjectField,emailField,emailVerifiedField,allowEmailLink}],captcha{isOn,provider,siteKey,secretKeyConfigured,endpoint}}`<br>`runtime{showFinance,showVersion,version,checkCNAME}`<br>`operations{logs{canDelete,canClean,retentionDays,capacityCount,capacityUnit},updates{autoCheck,ignoredVersion,updatesURL,fallbackURL,fallbackSource}}`<br>`security{frame,allowCountryIds,allowProvinceIds,allowLocal,allowIPs,allowRememberLogin,denySearchEngines,denySpiders,allowDomains,checkClientFingerprint,checkClientRegion,clientIPHeaderNames,clientIPHeaderOnly}`<br>`server{http{isOn,listens,redirectToHTTPS},https{isOn,listens,certificateId,certificateConfigured},restartRequired}`<br>`profile{username,fullname,passwordLocked}`<br>`options{timeZones[{name,offset,group}],dnsResolvers[{value,label}],userClusters[{id,name}],userFeatures[{value,label,description}],capacityUnits[{value,label}],countries[{id,name,codes}],provinces[{id,name,countryId,codes}],certificates[{id,name,dnsNames,expiresTime,isACME}]}` | 见模块约定 | 一次性返回管理端所有系统设置分组与下拉选项,供设置页初始化。各 `POST /settings/<分组>` 的字段与此处同名。 | 需登录,模块 Setting |
| admin-11-settings | 授权状态 | GET | `/api/admin/settings/license` | 无 | 响应头 `Cache-Control: no-store`<br>`data.license`:`allowed,status,message,customerCode,customerName,edition,licenseKeyHint,installationId,serial,kind,lastCheckedAt,lastRefreshAt,issuedAt,refreshAfter,expiresAt,licenseExpiresAt,hasLicenseKey,online,lastError,endpoint,serviceURL,features[{code,title,granted}],limits[{code,title,value,used,tracked}]`(`limits.value` 为 -1 表示不限) | 见模块约定 | 查询当前系统授权(License)状态。是否仅超管可调用由 EdgeAPI 判定(待确认)。 | 需登录,模块 Setting |
| admin-11-settings | 激活授权 | POST | `/api/admin/settings/license/activate` | `licenseKey` string,必填,trim 后 1~256 字符 | `data.license` 同"授权状态" | 错误:`请输入有效的授权码`;激活失败走 `actionutils.Fail` | 使用授权码在线激活。 | 需登录,模块 Setting,CSRF |
| admin-11-settings | 刷新授权 | POST | `/api/admin/settings/license/refresh` | 无(仅 `csrfToken`) | `data.license` 同"授权状态" | 见模块约定 | 向授权服务重新拉取授权状态。 | 需登录,模块 Setting,CSRF |
| admin-11-settings | 释放授权 | POST | `/api/admin/settings/license/release` | 无(仅 `csrfToken`) | `data.license` 同"授权状态" | 见模块约定 | 释放当前安装占用的授权席位。 | 需登录,模块 Setting,CSRF |
| admin-11-settings | 生成离线授权请求 | POST | `/api/admin/settings/license/offline-request` | `licenseKey` string,可选,≤256 字符 | `data.requestCode` string | 错误:`授权码过长` | 生成离线激活所需的请求码。 | 需登录,模块 Setting,CSRF |
| admin-11-settings | 导入离线授权 | POST | `/api/admin/settings/license/offline-import` | `bundleJSON` string,必填,trim 后非空,≤262144 字节 | `data.license` 同"授权状态" | 错误:`请粘贴或上传离线授权文件内容`、`离线授权文件过大` | 导入离线授权文件内容完成激活。 | 需登录,模块 Setting,CSRF |
| admin-11-settings | 生效配置列表 | GET | `/api/admin/settings/effective` | 无 | `data`:`{"revision":...,"settings":[Item]}`(`revision` 类型待确认)<br>`Item`:`code,title,group,applyMode,isDefault,secretPaths,value`(嵌套 JSON,未保存为 null)`,lastHistoryId,updatedAt,updatedAdminId,updatedAdminName`<br>凭据已由 API 侧脱敏 | 见模块约定 | 返回各系统配置项的当前生效值与最近修改信息。 | 需登录,模块 Setting |
| admin-11-settings | 配置变更历史 | GET | `/api/admin/settings/history` | 查询参数:<br>`code` string,可选,按配置项编码过滤<br>`offset` int,默认 0(<0 取 0)<br>`size` int,默认 20(≤0 或 >100 取 20) | `data`:`{"count":int,"histories":[H]}`<br>`H`:`id,code,title,action,rollbackFromId,adminId,adminName,createdAt,oldValue,newValue,changedSecretPaths` | 见模块约定 | 分页查询配置变更历史;用 `offset`/`size`,不同于其他接口的 `page`。 | 需登录,模块 Setting |
| admin-11-settings | 回滚配置历史 | POST | `/api/admin/settings/history/rollback` | `historyId` int64,必填,>0 | `data.revision` | 错误:`请选择要恢复的历史版本` | 将配置恢复到指定历史版本,本进程立即失效缓存。 | 需登录,模块 Setting,CSRF |
| admin-11-settings | 读取 AI 设置 | GET | `/api/admin/settings/ai` | 无 | `data`:<br>`ai`:`isOn,secretInvalid?,provider,baseURL,apiKey?,model,defenseIntervalMinutes,optimizationIntervalMinutes,requestTimeoutSeconds,maxRequestsPerDay,maxRequestsPerSitePerDay,defaultMode,maxRetries,retentionDays,autoAplyRiskMax,fullAutoEnabled,autoActionMaxDurationSeconds,autoActionCooldownSeconds,autoMaxActionsPerSitePerHour,autoMaxActionsPerRun,emergencyStop,maxConcurentTasks,analysisJitterSeconds,taskLeaseSeconds`(`apiKey` 已配置时显示 `********`;部分字段名为源码原拼写 `autoAplyRiskMax`/`maxConcurentTasks`)<br>`usage`:`requestsToday,activeUsers,runningTasks,completedRunsToday,failedRunsToday,failureRate,pendingRecommendations,costTracked,estimatedCostUSD` | 见模块约定 | 同一 action `AISettingsAction` 同时注册 GET 与 POST。GET 返回当前 AI 配置(密钥掩码)与今日用量统计。 | 需登录,模块 Setting |
| admin-11-settings | 保存 AI 设置 | POST | `/api/admin/settings/ai` | 参数与上行 `ai` 字段同名(含 `isOn,provider,baseURL,apiKey,model,defenseIntervalMinutes,optimizationIntervalMinutes,requestTimeoutSeconds,maxRequestsPerDay,maxRequestsPerSitePerDay,defaultMode,maxRetries,retentionDays,autoAplyRiskMax,fullAutoEnabled,autoActionMaxDurationSeconds,autoActionCooldownSeconds,autoMaxActionsPerSitePerHour,autoMaxActionsPerRun,emergencyStop,maxConcurentTasks,analysisJitterSeconds,taskLeaseSeconds`;`baseURL`/`apiKey` 带 `alias`,实际参数名待确认)<br>`apiKey` 为空或等于 `********` 时保留已存密钥<br>参考范围(`aiconfigs.Config.Validate`):provider 为 openai/deepseek/claude/custom;开启时必须有密钥;model 1~128;baseURL 须 HTTPS;defenseIntervalMinutes 5~1440;optimizationIntervalMinutes 15~1080;requestTimeoutSeconds 5~120;maxRequestsPerDay 1~1000;maxRequestsPerSitePerDay 0~10000;defaultMode 为 analyze/review/auto;maxRetries 0~10;retentionDays 1~365;autoAplyRiskMax 只能 low;autoActionMaxDurationSeconds 60~86400;autoActionCooldownSeconds 60~3600;autoMaxActionsPerSitePerHour 1~60;autoMaxActionsPerRun 1~20;maxConcurentTasks 1~32;analysisJitterSeconds 0~300;taskLeaseSeconds 30~3600 | 成功 `data` 为空(返回内容待确认) | 错误:校验失败文案待确认 | 保存 AI 配置。**管理端处理器未调用 `Validate()`,也不记操作日志**;范围校验是否在 EdgeAPI `UpdateSysSetting` 执行待确认。 | 需登录,模块 Setting,CSRF |
| admin-11-settings | 测试 AI 连接 | POST | `/api/admin/settings/ai/test` | 无(仅 `csrfToken`) | 成功 `data` 为空 | 错误:走 `actionutils.Fail` | 通过 `ReadSysSetting(code=aiConfigTest)` 让 EdgeAPI 用**已保存**配置(强制 IsOn=true)发起一次真实 AI 请求,超时取 `requestTimeoutSeconds`。需要管理员校验和 AI 功能授权。**注意**:测试的是已保存配置而非表单当前值;读类 RPC 带外部调用副作用。 | 需登录,模块 Setting,CSRF |
| admin-11-settings | 保存基础设置 | POST | `/api/admin/settings/general` | `productName` string,必填,1~80<br>`adminSystemName` string,必填,1~100<br>`showFinance` bool<br>`showVersion` bool<br>`version` string,0~40<br>`defaultPageSize` int,1~200<br>`timeZone` string,须能被 `time.LoadLocation` 识别<br>`dnsResolverType` string,须在支持集合内(选项含 default/go-native/cgo,完整集合待确认)<br>所有字符串 trim,且不允许控制字符 | 成功 `data` 为空 | 错误:各字段校验失败(文案见源码,待确认逐项原文) | 保存产品名称、管理端系统名、分页大小、时区、DNS 解析器等。日志 `AdminUI_LogUpdateUISettings`。 | 需登录,模块 Setting,CSRF |
| admin-11-settings | 保存用户端门户设置 | POST | `/api/admin/settings/portal` | `landingPageEnabled` string,`true`/`false`/空(空=保持原值),其他值报 `用户端落地页开关值无效`<br>`productName` string,1~80<br>`userSystemName` string,1~100<br>`title` string,1~120<br>`description` string,0~500<br>`footerText` string,0~200<br>`privacyPolicyURL`/`termsURL` string,空或 http/https URL | 成功 `data` 为空 | 错误:`用户端落地页开关值无效`、`invalid privacy policy URL`、`invalid terms URL` 及其他字段校验错误 | 保存用户端门户的落地页开关、名称、标题、页脚与协议链接。 | 需登录,模块 Setting,CSRF |
| admin-11-settings | 保存用户设置 | POST | `/api/admin/settings/users` | `registrationEnabled`/`requireApproval`/`requireIdentity`/`requireMobile` bool<br>`identityProvider` string,`aliyun_appcode`/`aliyun`/`tencent`,空=`aliyun_appcode`<br>`identityRestrictedFeatures` string 数组(未校验)<br>`identityAliyunMarketAppCode`、`identityAliyunMarketEndpoint`<br>`identityAliyunAccessKeyID`、`identityAliyunAccessKeySecret`、`identityAliyunRegion`、`identityAliyunEndpoint`<br>`identityTencentSecretID`、`identityTencentSecretKey`、`identityTencentRegion`、`identityTencentEndpoint`<br>`clusterId` int64<br>`features` string 数组,≤64 项,须为合法用户功能<br>`passwordResetEnabled` bool,`passwordResetLifeMinutes` int32 5~1440,`passwordResetSubject` 1~200,`passwordResetBody` 1~10000<br>`emailVerificationEnabled` bool,`emailVerificationSubject` 1~200,`emailVerificationBody` 1~1000 | 成功 `data` 为空 | 错误:依赖校验与字段范围错误(原文待确认) | 保存用户注册/实名/邮件验证/找回密码设置。依赖:开启注册需 `clusterId>0` 且集群可用;开启邮件功能需有启用的 email 通道;邮箱验证需先配置 `publicBaseURL`;开启实名时对应 provider 的凭据必填(可沿用已存值)。密钥字段留空保留旧值;可选字段用 `HasParam` 区分"未传"与"清空"。日志 `User_LogUpdateUserGlobalSettings`。 | 需登录,模块 Setting,CSRF |
| admin-11-settings | 保存认证设置 | POST | `/api/admin/settings/authentication` | `publicBaseURL` string<br>`providersJSON` string,JSON 数组,≤256KiB,≤20 个,元素键 `isOn,code,name,clientId,clientSecret,authorizationURL,tokenURL,userInfoURL,scopes,subjectField,emailField,emailVerifiedField,allowEmailLink`<br>`captchaIsOn` bool<br>`captchaProvider`、`captchaSiteKey`、`captchaSecretKey`、`captchaEndpoint` string | 成功 `data` 为空 | 错误:`config.Validate()` 的具体规则未读,待确认 | 保存公开访问基址、OAuth 登录提供方与验证码配置。内置 provider 的端点由预设覆盖;`clientSecret`/`captchaSecretKey` 留空沿用已存值。 | 需登录,模块 Setting,CSRF |
| admin-11-settings | 保存运行设置 | POST | `/api/admin/settings/runtime` | `showFinance` bool<br>`showVersion` bool<br>`version` string,0~40<br>`checkCNAME` bool | 成功 `data` 为空 | 见模块约定 | 保存用户端运行显示选项;会重置流量统计相关默认值。日志 `AdminUserUI_LogUpdateUISettings`。 | 需登录,模块 Setting,CSRF |
| admin-11-settings | 保存运维设置 | POST | `/api/admin/settings/operations` | `section` string,必填,`logs`/`updates`<br>`section=logs`:`logsCanDelete`、`logsCanClean` bool;`logsRetentionDays` int 1~3650;`logsCapacityCount` int 1~1024;`logsCapacityUnit` string,MB/GB(具体常量值待确认)<br>`section=updates`:`updatesAutoCheck` bool;`updatesClearIgnored` bool;`updatesURL` string(经 `Normalize`+`Validate`,规则待确认) | 成功 `data` 为空 | 错误:`invalid operations section`(`actionutils.Fail`,按文本映射 400)及各字段校验错误 | 按 `section` 分别保存日志留存策略或更新检查设置。日志:logs 为 `Log_LogUpdateSettings`,updates 为 `AdminUpdate_LogUpdateCheckSettings`。 | 需登录,模块 Setting,CSRF |
| admin-11-settings | 读取运行参数 | GET | `/api/admin/settings/runtime-parameters` | 无 | 非超管:`data.allowed=false`(仍为成功响应)<br>超管:`allowed=true,config,defaults,metricsTokenConfigured,nodeMetricsTokenConfigured,source,sync`<br>`source` 为 `default`/`setting`;`sync{source,lastSyncAt,lastErrorAt,failures}`<br>`config`/`defaults` 为 `RuntimeConfig`:`backup,transfer,userPortal,security,integrate,rollout,installer,node`;令牌 `security.metricsToken`、`node.metrics.token` 已清空<br>已确认 `backup{enabled,intervalHours(1~168),retention(1~90),dir,offsiteDir,offsiteKeyFile,offsiteRetention}`,其余子结构字段待确认 | 见模块约定 | 同一路径 GET 读取、POST 保存(action 不同:`RuntimeSettingsAction`/`UpdateRuntimeSettingsAction`)。GET 仅超管可见内容。 | 需登录,模块 Setting |
| admin-11-settings | 保存运行参数 | POST | `/api/admin/settings/runtime-parameters` | **仅超管**<br>`configJSON` string,`RuntimeConfig` 的 JSON,≤64KiB<br>`clearMetricsToken` bool<br>`clearNodeMetricsToken` bool<br>`reset` bool,恢复默认<br>令牌留空保留旧值 | 成功 `data` 为空(待确认是否回显配置) | 错误:`只有超管可以查看和修改运行参数`、`运行参数过大`、`运行参数格式无效：...`、`Validate()` 返回的校验信息 | 保存或重置运行参数。日志 `AdminUpdate_LogUpdateRuntimeConfig`(摘要为变更的分组名,不含值)或 `AdminUpdate_LogResetRuntimeConfig`。 | 需登录,模块 Setting,仅超管,CSRF |
| admin-11-settings | 上传品牌图片 | POST | `/api/admin/settings/general/asset` | multipart/form-data:<br>`kind` string,必填,`logo`/`favicon`/`user-logo`/`user-favicon`<br>`file` 文件,必填,≤2MiB;PNG/JPEG/GIF/WebP,`favicon` 另允许 ICO<br>`csrfToken` | `data.fileId` int64,`data.url` string(形如 `/ui/image/{id}`) | 错误:`invalid asset kind` 及文件类型/大小校验错误(原文待确认) | 上传管理端或用户端的 Logo/Favicon 并更新对应 FileId。 | 需登录,模块 Setting,CSRF |
| admin-11-settings | 删除品牌图片 | POST | `/api/admin/settings/general/asset/delete` | `kind` string,必填,取值同上 | 成功 `data` 为空 | 错误:`invalid asset kind` | 清除对应品牌图片的 FileId(恢复默认)。 | 需登录,模块 Setting,CSRF |
| admin-11-settings | 保存安全设置 | POST | `/api/admin/settings/security` | `frame` string,转大写,空/`DENY`/`SAMEORIGIN`<br>`allowCountryIds`/`allowProvinceIds` int64 数组,正数,≤256,须存在<br>`allowLocal` bool<br>`allowIPs` string 数组,≤128 条,每条 ≤128 字符且为合法 IP/范围<br>`allowDomains` string 数组,≤128 条,每条 ≤512,支持 `~正则`<br>`allowRememberLogin`/`denySearchEngines`/`denySpiders`/`checkClientFingerprint`/`checkClientRegion` bool<br>`clientIPHeaderNames` string 数组,≤16 个,每个 ≤100<br>`clientIPHeaderOnly` bool,开启时须至少一个报头 | 成功 `data` 为空 | 错误:`the proposed IP policy would block the current administrator`、`the proposed domain policy would block the current administrator`,其余字段校验错误原文待确认 | 保存管理端访问安全策略。含防自锁:新策略会拦截当前管理员 IP/域名时拒绝。日志 `AdminSecurity_LogUpdateSecuritySettings`。 | 需登录,模块 Setting,CSRF |
| admin-11-settings | 保存管理端服务设置 | POST | `/api/admin/settings/server` | `httpIsOn` bool<br>`httpListens` string 数组<br>`httpRedirectToHTTPS` bool<br>`httpsIsOn` bool<br>`httpsListens` string 数组,每组 ≤16 条,默认端口 80/443,不支持 `unix:`<br>`certificateId` int64,≥0;>0 时把证书/私钥写入 `configs/`,证书须当前有效 | `data.restartRequired=true` | 错误:HTTP 与 HTTPS 不可同时关闭、监听冲突、需要证书、证书无效等(原文待确认) | 保存管理端 HTTP/HTTPS 监听与证书,需重启生效;会写 `https.cert.id` 文件。日志 `AdminServer_LogUpdateServerHTTPSettings` 与 `AdminServer_LogUpdateServerHTTPSSettings`。 | 需登录,模块 Setting,CSRF |
| admin-11-settings | 预览管理端服务设置 | POST | `/api/admin/settings/server/preview` | 同"保存管理端服务设置" | `data`:`{"valid":true,"restartRequired":bool,"diff":[{"field":string,"before":any,"after":any}]}`<br>`field` 取值:HTTP 服务、HTTP 监听、HTTP 跳转 HTTPS、HTTPS 服务、HTTPS 监听、TLS 证书<br>错误同保存接口的校验错误 | 见模块约定 | 只校验并计算差异,不落盘、不写日志。 | 需登录,模块 Setting,CSRF |
| admin-11-settings | 读取页面托管设置 | GET | `/api/admin/settings/pages` | 无 | `data`:<br>`config{isOn,storageId,projectsPerLimit,storageSizePerLimit,defaultPage,default404Page}`<br>`storages[{id,type,name,storagePath,endpoint,bucket,region,createdAt,isDefault}]`(不含 accessKey/secretKey) | 见模块约定 | 读取静态页面托管(Pages)的全局配置与存储列表。 | 需登录,模块 Setting |
| admin-11-settings | 保存页面托管设置 | POST | `/api/admin/settings/pages` | `isOn` bool<br>`storageId` int64,`isOn` 时须 >0<br>`projectsPerLimit` int,≥0<br>`storageSizePerLimit` int,≥0<br>`defaultPage` string,空取 `index.html`,≤255 字符<br>`default404Page` string,空取 `404.html`,≤255 字符<br>页面文件名经 `commonpages.NormalizePath` 规范化 | 成功 `data` 为空 | 错误:数值为负、`isOn` 缺存储、文件名非法等(原文待确认) | 保存 Pages 开关、存储、配额与默认页面。**不记操作日志**。 | 需登录,模块 Setting,CSRF |
| admin-11-settings | 创建页面存储 | POST | `/api/admin/settings/pages/storages/create` | `name` string,1~100<br>`storageType` string,`local`/`nfs`/`s3`<br>`storagePath` string,local/nfs 必填,绝对路径,1~2048<br>`bucket` string,1~255(s3)<br>`region` string,1~128(s3)<br>`accessKey`/`secretKey` string,各 0~512,须同时提供或同时为空<br>`endpoint` string,可选;无协议补 `https`;`http` 仅允许 loopback;不得含凭据/query/fragment | `data.storageId` int64 | 错误:各字段校验错误(原文待确认) | 创建页面托管存储。**不记操作日志**。 | 需登录,模块 Setting,CSRF |
| admin-11-settings | 删除页面存储 | POST | `/api/admin/settings/pages/storages/delete` | `storageId` int64,必填,>0 | 成功 `data` 为空 | 错误:`invalid pages storage id` | 删除页面托管存储。**不记操作日志**;被引用时的行为待确认。 | 需登录,模块 Setting,CSRF |
| admin-11-settings | 保存个人资料 | POST | `/api/admin/settings/profile` | `fullname` string,必填,1~100 | 成功 `data` 为空 | 见模块约定 | 修改当前管理员姓名。日志 `AdminProfile_LogUpdateProfile`。 | 需登录,模块 Setting,CSRF |
| admin-11-settings | 保存登录账号 | POST | `/api/admin/settings/login` | `username` string,必填,1~100,仅字母/数字/下划线<br>`password` string,可选;非空时 8~128 字符,不得首尾空白,不得含控制字符 | 成功 `data` 为空 | 错误:`username is already in use`;账号被密码锁定时 HTTP 403 + `PasswordLockedMessage`;其他校验错误原文待确认 | 修改当前管理员的登录用户名与密码(密码留空则不改)。日志 `AdminLogin_LogUpdateLogin`。 | 需登录,模块 Setting,CSRF |


### 公共结构

- 密钥脱敏:`GET /settings` 与 `GET /settings/ai` 不返回密钥明文;`xxxConfigured` 表示已配置,AI `apiKey` 以 `********` 掩码。提交时留空或回传掩码表示保留旧值。
- `GET /settings/runtime-parameters` 对非超管返回 `allowed=false` 的成功响应,而 `POST` 对非超管返回错误,两者处理不一致。

<a id="m13"></a>

## 13. 管理端 · 运维维护(管理端)

**模块约定**

> 路由总数 31,来源 EdgeAdmin/internal/web/actions/default/api/admin/init.go
说明:
- 路径前缀均为 `/api/admin/maintenance`,处理器在同目录 `maintenance_*_resource.go`、`api_nodes_resource.go`、`api_node_mutations.go`。
- 参数传法:TeaGo 只解析查询串与表单(`application/x-www-form-urlencoded` 等),不解析 JSON 请求体。参数名为 params 结构体字段名首字母小写。下表参数均按此理解。
- 成功包络:`{"code":200,"message":"","data":{...}}`,`Content-Type: application/json; charset=utf-8`。`Success()` 内部终止处理,所以文档中的"返回"均指 `data` 内容。
- 失败包络有两种:(1) `this.Fail(msg)` 输出 `{"code":400,"message":msg,"data":...,"errors":[...]}`,HTTP 状态码是否为 200 待确认(未读到框架 `failWithoutPanic` 的末尾);(2) `actionutils.Fail(this, err)` 把 gRPC 错误映射为真实 HTTP 状态并写入同值 `code`:InvalidArgument 400 / Unauthenticated 401 / PermissionDenied 403 / NotFound 404 / AlreadyExists 409 / ResourceExhausted 429 / Unavailable 503 / DeadlineExceeded 504 / 其他 500。message 为本地化的服务错误文案加括号内的客户端消息(含密码/token 等敏感词的错误会被脱敏)。
- 认证(`helpers.NewUserMustAuth`),`/api/` 路径下失败返回 JSON `{"code":状态码,"message":...,"data":null}`:未完成安装 409 `system setup is required`;未登录 401 `authentication required`;无模块权限 403 `permission denied`;另有 WAF 拦截 403 `Denied By WAF`、IP 限制 403、恢复模式重定向 `/admin/recover.html`。
- 权限模块:`health`、`health/history`、`doctor` 为公共模块(`AdminModuleCodeCommon`,登录即可);其余 28 条为系统设置模块(`AdminModuleCodeSetting`)。其中备份(backups 全部)、迁移(transfers 全部)、更新(updates 全部)、`api-nodes/install` 额外要求**超级管理员**,否则 `this.Fail("only a super administrator can manage ...")`。`database/*`、`api-nodes` 的 list/form/logs/create/update/toggle/delete 不要求超级管理员。
- CSRF:所有 19 条 POST 都声明了 `CSRF *actionutils.CSRF`,需随表单/查询参数提交 `csrfToken`(来自 `GET /csrf/token` 的 `data.token`,绑定 session,有效期 1800 秒)。校验失败为 **HTTP 403 纯文本** `表单已失效，请刷新页面后重试(001)`,不是 JSON。GET 接口不校验 CSRF。
- GET 请求经鉴权 helper 时会向 `action.Data` 注入若干 `tea*` 页面键(如 `teaTitle`、`teaMenu`、`teaLang`),它们是否出现在下表各 GET 响应的 `data` 中:待确认。下表只列处理器显式赋值的键。
- 分页:`api-nodes` 用 `page`/`size`,`normalizePagination`:page 默认 1;size 默认 20,最大 100,超出截断为 100。`api-nodes/logs` 用 `offset`/`size`,见该行。
- 错误文案多为英文(如 `invalid database host`),与其他模块的中文提示不统一。

| 模块 | 接口名称 | 方法 | 路径 | 请求参数 | 响应数据 | 主要错误 | 功能说明 | 认证/权限 |
|---|---|---|---|---|---|---|---|---|
| admin-12-maintenance | 服务健康检查 | GET | `/api/admin/maintenance/health` | 无 | 响应头 `Cache-Control: no-store`<br>`checkedAt` string<br>`services[]`:`name`,`code`(admin/api/user),`status`(ok/critical/warning/unknown/inactive),`message`,`ports`;`user` 项仅在 API 配置可读时追加<br>`disk`:`status`,`path`,`usedBytes`,`totalBytes`,`usedPercent`,`message`(≥90% warning,≥95% critical)<br>`license`:`status`,`allowed`,`expiresAt`,`message`,可选 `customerCode`,`licenseKeyHint`<br>`apiEndpoints[]`:`endpoint`,`state`<br>`apiEndpointSummary`:`total`,`ready`<br>`expectedPorts`:[7788,8001,7800]<br>`capabilities`:`readOnly`,`scheduledChecks`,`history`,`databaseReadWriteProbe`,`notifications`<br>`schedule`:`intervalSeconds`(300),`lastScheduledCheckAt`,`lastAlertStatus`,`pendingAlertStatus`,`nextScheduledCheckAt` | 错误:RPC 不可用时 `RPC服务暂时不可用...`(全文待确认) | 汇总 admin/api/user 服务、磁盘、许可证、API 端点状态。**副作用**:每次调用都会追加一条 manual 历史并持久化到 `data/maintenance-health/history.json`;请求若带 `x-edgeroute-license-refresh` 元数据则触发许可证刷新。`licenseKeyHint` 为提示串,不含完整授权码。 | 需登录(公共模块) |
| admin-12-maintenance | 健康检查历史 | GET | `/api/admin/maintenance/health/history` | 查询参数:`limit` int,可选;≤0 或 >96 时取 24 | 响应头 `no-store`<br>`items[]`:`checkedAt`,`critical`,`warnings`,`unknown`,`services`,`license`,`disk`(后三者结构细节待确认)<br>`retention` int,固定 96 | 见模块约定 | 读取最近的健康检查历史,最多保留 96 条。 | 需登录(公共模块) |
| admin-12-maintenance | 系统自检(Doctor) | GET | `/api/admin/maintenance/doctor` | 查询参数:`refresh` bool,可选,默认 false;结果缓存 15 秒,为 true 时强制重跑 | `component` string,`version` string,`checkedAt` string<br>`items[]`:`id`,`title`,`durationMs`,`status`(ok/warn/fail/skip),可选 `detail`,`hint`<br>`summary`:`ok`,`warn`,`fail`,`skip`<br>`exitCode` int(0/1/2)<br>报告中的 `root` 字段不输出;检查项清单待确认(未读 `admindoctor.Checks()`) | 见模块约定 | 运行内置自检并返回结果。 | 需登录(公共模块) |
| admin-12-maintenance | 数据库概览 | GET | `/api/admin/maintenance/database` | 无 | `connection`:`host`,`port`,`username`,`database`,`passwordConfigured` bool(不返回密码),`restartRequired` 固定 false<br>`retention`:`serverAccessLogDays`,`serverBandwidthStatDays`,`userBandwidthStatDays`,`userPlanBandwidthStatDays`,`serverDailyStatDays`,`serverDomainHourlyStatDays`,`traficDailyStatDays`(原样拼写),`trafficHourlyStatDays`,`nodeClusterTrafficDailyStatDays`,`nodeTrafficDailyStatDays`,`nodeTrafficHourlyStatDays`,`httpCacheTaskDays`<br>`tables[]`:`name`,`engine`,`rows`,`dataLength`,`indexLength`,`size`,`collation`,`comment`,`canClean`,`canDelete`,按 `size` 降序<br>`summary`:`tableCount`,`totalBytes`,`cleanableBytes` | 见模块约定 | 查看数据库连接信息(脱敏)、各表保留天数、表占用。 | 需登录(系统设置模块) |
| admin-12-maintenance | 更新数据库连接 | POST | `/api/admin/maintenance/database/update` | `host` string 必填,≤253,IP 或主机名(主机名正则 `adminUserNodeHostnamePattern` 待确认)<br>`port` int 必填,1..65535<br>`database` string 必填,≤128,`^[A-Za-z0-9_.-]+$`<br>`username` string 必填,同上<br>`password` string,可空(空则沿用现有),≤4096 且无控制字符<br>`csrfToken` 必填 | `data.restartRequired` = true<br>错误(Fail,英文):如 `invalid database host`;连接校验失败的具体文案待确认 | 见模块约定 | 先用新参数真实连接数据库校验,通过后写入私有配置文件;需重启管理端才生效。 | 需登录(系统设置模块)+CSRF |
| admin-12-maintenance | 更新数据保留天数 | POST | `/api/admin/maintenance/database/retention` | 12 个 int 字段,均范围 0..3650:`serverAccessLogDays`,`serverBandwidthStatDays`,`userBandwidthStatDays`,`userPlanBandwidthStatDays`,`serverDailyStatDays`,`serverDomainHourlyStatDays`,`traficDailyStatDays`,`trafficHourlyStatDays`,`nodeClusterTrafficDailyStatDays`,`nodeTrafficDailyStatDays`,`nodeTrafficHourlyStatDays`,`httpCacheTaskDays`<br>`csrfToken` 必填 | 成功无业务字段(`data` 内容待确认) | 错误:`retention days must be between 0 and 3650` | 保存各类统计/日志的保留天数。**疑似缺陷**:未提交的字段按 0 处理,没有"缺省保持原值"逻辑,漏传会把对应保留期清为 0;0 的具体含义(不保留或不清理)未核实,待确认。 | 需登录(系统设置模块)+CSRF |
| admin-12-maintenance | 清理或删除数据表 | POST | `/api/admin/maintenance/database/table` | `table` string 必填<br>`operation` string 必填,`truncate`/`delete`(大小写不敏感)<br>`confirmation` string 必填,须与 `table` 完全一致<br>`csrfToken` 必填 | 成功无业务字段(`data` 内容待确认) | 错误:`table confirmation does not match`;`invalid database table operation`;表是否允许清理/删除(`canClean`/`canDelete`)由 RPC 层判断,具体文案待确认 | 对指定表执行清空或删除。**破坏性操作,不要求超级管理员**,仅需系统设置模块权限;本层不校验表名合法性。 | 需登录(系统设置模块)+CSRF |
| admin-12-maintenance | 备份列表与状态 | GET | `/api/admin/maintenance/backups` | 无 | `items[]`(Manifest):`version`,`fileName`,`database`,`createdAt`,`size`,`sha256`,`uncompressedBytes`,`tableCount`,`excludedAccessLogTables`,`includeAccessLogs`,可选 `source`,`tool`,`tolVersion`(原样拼写),可选 `verifiedAt`<br>`job`:当前/最近任务或 null,见公共结构 BackupJob<br>`schedule`:`enabled`,`intervalSeconds`,`retention`,可选 `configurationError`<br>`offsite`:`enabled`,可选 `configurationError`<br>`restoreDatabasePrefix` 固定 `edgeroute_restore_`<br>非超管:Fail `only a super administrator can manage ...`(全文待确认) | 见模块约定 | 列出本地备份、当前任务及定时/异地备份配置状态。 | 需登录(系统设置模块)+超级管理员 |
| admin-12-maintenance | 下载备份 | GET | `/api/admin/maintenance/backups/download` | 查询参数:`fileName` string 必填(命名规则校验函数 `validateBackupFileName` 未读,待确认) | 成功:**二进制流**,`Content-Type: application/gzip`,`Content-Disposition: attachment`,`Content-Length`,`Cache-Control: no-store`,`X-Content-Type-Options: nosniff`,写超时 2 小时<br>找不到:**HTTP 404 纯文本** `backup not found`<br>非超管:JSON Fail | 见模块约定 | 下载指定备份文件。 | 需登录(系统设置模块)+超级管理员 |
| admin-12-maintenance | 创建备份 | POST | `/api/admin/maintenance/backups/create` | `includeAccessLogs` bool,可选,默认 false<br>`csrfToken` 必填 | `data.job`(BackupJob,kind 为备份) | 错误:`another backup operation is already running` | 异步创建数据库备份,通过 `backups` 接口轮询 `job`。 | 需登录(系统设置模块)+超级管理员+CSRF |
| admin-12-maintenance | 校验备份 | POST | `/api/admin/maintenance/backups/verify` | `fileName` string 必填<br>`csrfToken` 必填 | `data.job`(kind=`verify`)<br>错误同上(任务冲突) | 见模块约定 | 异步校验备份文件完整性(大小、sha256 等),结果在 `job.verification`。 | 需登录(系统设置模块)+超级管理员+CSRF |
| admin-12-maintenance | 恢复校验 | POST | `/api/admin/maintenance/backups/restore` | `fileName` string 必填<br>`targetDatabase` string,临时库名,约束由 `validateRestoreDatabase` 决定(未读,待确认;前缀应为 `edgeroute_restore_`)<br>`dropAfter` bool,可选,默认 false<br>`csrfToken` 必填 | `data.job`(kind=`restore-verify`),结果在 `job.restore` | 错误:任务冲突、文件不存在等(全文待确认) | **名称易误解**:实际是把备份恢复到临时库做"恢复校验",不覆盖线上库;`dropAfter` 控制校验后是否删除临时库。 | 需登录(系统设置模块)+超级管理员+CSRF |
| admin-12-maintenance | 删除备份 | POST | `/api/admin/maintenance/backups/delete` | `fileName` string 必填<br>`confirmation` string 必填,trim 后须等于 `fileName`<br>`csrfToken` 必填 | 成功无业务字段 | 错误:确认文本不匹配(英文文案待确认);任务冲突 | 删除本地备份文件。 | 需登录(系统设置模块)+超级管理员+CSRF |
| admin-12-maintenance | 迁移计划列表 | GET | `/api/admin/maintenance/transfers` | 查询参数:`planId` string,可选 | `currentEndpoints[]`:当前 API 端点(元素结构待确认)<br>`plans[]`:`id`,`status`,`sourceEndpoint`,`targetEndpoint`,`targetAPINodeName`,`adminSwitched`,`createdAt`,`updatedAt`,`summary`<br>`plan`:指定 `planId` 的详情,未指定时取第一个,无则 null,见公共结构 TransferPlan | 见模块约定 | 查看 API 节点迁移计划,可指定查看某一计划。 | 需登录(系统设置模块)+超级管理员 |
| admin-12-maintenance | 创建迁移计划 | POST | `/api/admin/maintenance/transfers/create` | `targetEndpoint` string 必填(格式校验待确认)<br>`nodeIds` int64 数组,参与迁移的边缘节点(多值传法与上限待确认)<br>`csrfToken` 必填 | `data.plan`(TransferPlan)<br>错误文案待确认 | 见模块约定 | 创建把边缘节点指向新 API 端点的迁移计划。RPC 超时 60 秒。 | 需登录(系统设置模块)+超级管理员+CSRF |
| admin-12-maintenance | 开始迁移 | POST | `/api/admin/maintenance/transfers/start` | `planId` string 必填<br>`csrfToken` 必填 | `data.plan`(TransferPlan) | 见模块约定 | 启动迁移计划。 | 需登录(系统设置模块)+超级管理员+CSRF |
| admin-12-maintenance | 回滚迁移 | POST | `/api/admin/maintenance/transfers/rollback` | `planId` string 必填<br>`csrfToken` 必填 | `data.plan`(TransferPlan) | 见模块约定 | 回滚迁移计划,节点恢复原端点。 | 需登录(系统设置模块)+超级管理员+CSRF |
| admin-12-maintenance | 切换管理端到目标端点 | POST | `/api/admin/maintenance/transfers/switch-admin` | `planId` string 必填<br>`confirmation` string 必填,须匹配目标端点<br>`allowOffline` bool,可选,默认 false<br>`csrfToken` 必填 | `data.plan`(TransferPlan) | 错误:`target endpoint confirmation does not match the transfer plan` | 把管理端自身切换到迁移目标 API 端点;`allowOffline` 允许存在离线节点时继续。 | 需登录(系统设置模块)+超级管理员+CSRF |
| admin-12-maintenance | 更新状态 | GET | `/api/admin/maintenance/updates` | 无 | `currentVersion`<br>`official`:`adminManifestURL`,`adminManifestSource`,`userManifestURL`<br>`operation`:`running`,`stage`,`lastError`,`manifestURL`,`progress`,`startedAt`,`completedAt`,`restartScheduled`,`rollbackAttempted`,`rollbackCompleted`,`available`,`preflight`<br>`trust`:`directory`,`keyIds`,`ready`,可选 `error`<br>`journal`:`state`,`version`,`failureReason`,`startedAt`,`updatedAt`,`rollbackAvailable`;无记录时 `state` 为 `none`/`error`<br>`history[]`:`version`,`stage`,可选 `error`,`startedAt`,`completedAt`,`rollbackAttempted`,`rollbackCompleted`<br>`userPlatform`:`configured`,`executable`,`service`,`running`,`lastError`,`lastOutput`,`startedAt`,`completedAt` | 见模块约定 | 查看管理端更新状态、签名信任、历史,以及用户端更新器状态。`trust.directory`、`userPlatform.executable` 等会暴露服务器路径。 | 需登录(系统设置模块)+超级管理员 |
| admin-12-maintenance | 检查更新 | POST | `/api/admin/maintenance/updates/check` | `manifestURL` string,可空(空则用官方清单,解析逻辑 `ResolveAdminUpdatesURL` 未读)<br>`csrfToken` 必填 | `manifest`:`version`,`component`,`os`,`arch`,`edition`,`channel`,`fileName`,`size`,`sha256`,`publishedAt`,`expiresAt`,`releaseNotesURL`,`rollbackSupported`<br>`preflight`:`ready`,`currentVersion`,`targetVersion`,`checks[]`(`code`,`name`,`status`,`blocking`,`message`),`checkedAt`<br>错误文案待确认 | 见模块约定 | 拉取并验签更新清单,执行更新前检查。`manifestURL` 由调用者指定,服务端会主动去抓取(仅超管可用)。 | 需登录(系统设置模块)+超级管理员+CSRF |
| admin-12-maintenance | 应用管理端更新 | POST | `/api/admin/maintenance/updates/apply` | `manifestURL` string,可空<br>`confirmation` string **必填**,须等于 `UPDATE <版本号>`(留空或不匹配一律拒绝)<br>`csrfToken` 必填 | 成功:更新异步执行,`data` 内容待确认<br>前置检查 | 失败:`data.preflight` 加 Fail `更新前检查未通过，请先处理阻断项`<br>其他错误:`the bundled local API node is required for a controlled update`;`another admin update is already running` | 下载、验签并替换管理端程序,完成后重启。 | 需登录(系统设置模块)+超级管理员+CSRF |
| admin-12-maintenance | 回滚管理端更新 | POST | `/api/admin/maintenance/updates/rollback` | `confirmation` string 必填,必须为 `ROLLBACK`<br>`csrfToken` 必填 | 成功后 2 秒延迟重启管理端 | 错误:`type ROLLBACK to confirm`;更新进行中报错;`ExternalAPISynced` 时拒绝(文案待确认) | 回滚到更新前版本。 | 需登录(系统设置模块)+超级管理员+CSRF |
| admin-12-maintenance | 应用用户端更新 | POST | `/api/admin/maintenance/updates/user/apply` | `manifestURL` string,可空<br>`confirmation` string **必填**,须为 `UPDATE USER`(留空或不匹配一律拒绝)<br>`csrfToken` 必填 | 成功 `data` 内容待确认<br>错误文案待确认 | 见模块约定 | 触发用户端平台(EdgeUserPortal)的更新,状态在 `updates` 的 `userPlatform` 中。 | 需登录(系统设置模块)+超级管理员+CSRF |
| admin-12-maintenance | API 节点列表 | GET | `/api/admin/maintenance/api-nodes` | 查询参数:`page` int,默认 1<br>`size` int,默认 20,最大 100 | `items[]`:`id`,`name`,`description`,`isOn`,`isPrimary`,`restIsOn`,`accessAddrs`,`rpcHTTPListens`,`rpcHTTPSListens`,`restHTTPListens`,`restHTTPSListens`,`status`<br>`status`:`state`,`freshWindowSeconds`,`heartbeatWindowSeconds`,`active`,`heartbeatActive`,`updatedAt`,`serverTime`,`buildVersion`,`os`,`arch`,`hostname`,`hostIP`,`cpuUsage`,`memoryUsage`,`memoryTotal`,`diskUsage`,`diskTotal`,`load1m`,`connectionCount`,`openFile*`,`apiSuccessPercent`,`apiAvgCostSeconds`,`trafficInBytes`,`trafficOutBytes`,`systemTraffic*`,`error`,`statusReasons`<br>`page`,`size`,`total` | 见模块约定 | 分页列出 API 节点及运行状态。`openFile*`、`systemTraffic*` 为前缀写法,具体键名待确认。 | 需登录(系统设置模块) |
| admin-12-maintenance | API 节点表单数据 | GET | `/api/admin/maintenance/api-nodes/form` | 查询参数:`nodeId` int64,可选(有则返回节点详情) | `certificates[]`:`id`,`name`,`dnsNames`,`expiresTime`,`isACME`<br>有 `nodeId` 时另有 `node`:列表项字段加 `sslPolicyId`,`certificateIds`,`certificates[]`(`id`,`name`,`dnsNames`),`status` | 见模块约定 | 提供新建/编辑 API 节点所需的可选证书与节点详情。 | 需登录(系统设置模块) |
| admin-12-maintenance | API 节点安装配置 | GET | `/api/admin/maintenance/api-nodes/install` | 查询参数:`nodeId` int64 必填 | 响应头 `no-store`<br>`node`:`id`,`name`<br>`apiConfigYAML`,`databaseConfigYAML`,`apiConfigFile`,`databaseConfigFile`,`version`<br>**敏感**:`apiConfigYAML` 含节点 secret,`databaseConfigYAML` 含数据库凭据(此处不列真实值) | 见模块约定 | 生成在新机器上部署该 API 节点所需的配置文件内容。 | 需登录(系统设置模块)+超级管理员 |
| admin-12-maintenance | API 节点日志 | GET | `/api/admin/maintenance/api-nodes/logs` | 查询参数:`nodeId` int64 必填<br>`offset` int,≥0,默认 0<br>`size` int,0 时取 20,范围 1..100,否则报错<br>`dayFrom`/`dayTo` string,`YYYY-MM-DD`,须 from≤to<br>`keyword` string,≤200<br>`level` string,`error`/`warning`/`info`/`success`<br>`tag` string,转大写后匹配 `^[A-Z0-9_:-]+$`,≤64 | `nodeId`,`total`,`offset`,`size`<br>`items[]`:`id`,`tag`,`description`,`level`,`createdAt`,`count`,`isFixed`,`isRead`<br>`tags`(元素结构待确认,来自 `FindNodeCommonTags`) | 错误:参数不合法的具体文案待确认 | 按条件分页查询 API 节点日志。 | 需登录(系统设置模块) |
| admin-12-maintenance | 创建 API 节点 | POST | `/api/admin/maintenance/api-nodes/create` | `name` string 必填,1..100<br>`description` string,0..500<br>`rpcHttpListens`/`rpcHttpsListens` string 多行监听地址,至少填一个 RPC 监听<br>`restIsOn` bool<br>`restHttpListens`/`restHttpsListens` string,REST 开启时至少一个,关闭时不得填<br>`accessAddrs` string,必须含公网访问地址<br>`certificateIds` int64 数组,正数、去重、≤32;有 HTTPS 监听必须有证书,无 HTTPS 则不得有证书<br>`isOn` bool<br>`csrfToken` 必填 | `data.nodeId` int64 | 错误:上述各校验失败的英文文案待确认;多行文本分行规则(`normalizeAdminUserNodeLines`)及主机名规范待确认 | 创建 API 节点。 | 需登录(系统设置模块)+CSRF |
| admin-12-maintenance | 更新 API 节点 | POST | `/api/admin/maintenance/api-nodes/update` | 同创建,另加:`nodeId` int64 必填<br>`isPrimary` bool,**缺省 false 会被写回 false**(疑似缺陷,漏传会取消主节点)<br>`csrfToken` 必填 | 成功无业务字段(待确认)<br>错误同创建 | 见模块约定 | 更新 API 节点配置。 | 需登录(系统设置模块)+CSRF |
| admin-12-maintenance | 启停 API 节点 | POST | `/api/admin/maintenance/api-nodes/toggle` | `nodeId` int64 必填<br>`isOn` bool,**缺省 false 即停用**<br>`csrfToken` 必填 | 成功无业务字段(待确认) | 见模块约定 | 启用或停用节点,其余字段保持不变。不要求超级管理员。 | 需登录(系统设置模块)+CSRF |
| admin-12-maintenance | 删除 API 节点 | POST | `/api/admin/maintenance/api-nodes/delete` | `nodeId` int64 必填<br>`csrfToken` 必填 | 成功无业务字段(待确认)<br>错误文案待确认(如删除主节点是否被拒,未核实) | 见模块约定 | 删除 API 节点。不要求超级管理员,而同组 `install` 要求超管,权限不一致。 | 需登录(系统设置模块)+CSRF |


### 公共结构

### BackupJob(备份任务,`backups` 的 `job`、各备份 POST 的 `data.job`)
- `id` string,`kind` string(`backup`/`verify`/`restore-verify` 等,完整取值待确认),`status` string(取值待确认)
- 可选:`fileName`,`target`,`message`,`error`
- `startedAt`;可选 `finishedAt`
- 可选 `manifest`:同 `backups.items[]`
- 可选 `verification`:`fileName`,`verifiedAt`,`size`,`sha256`,`uncompressedBytes`
- 可选 `restore`:`fileName`,`targetDatabase`,`tableCount`,`checkedAt`,`dropped`,`cleanup`

### TransferPlan(迁移计划)
- `id`,`status`,`sourceEndpoint`,`targetEndpoint`,`targetAPINodeId`,`targetAPINodeName`,`adminSwitched`,`createdAt`,`updatedAt`,`startedAt`,`finishedAt`,`error`
- `summary`:`total`,`pending`,`completed`,`alreadyTarget`,`offline`,`failed`,`blocked`,`rolledBack`
- `canStart`,`canSwitchAdmin`,`canRollback`
- `nodes[]`:`id`,`name`,`status`,`error`,`isActive`,`originalEndpoint`,`originalAPINodeIds`,`updatedAt`
- `plans[]` 列表项只含上面的头部字段和 `summary`,不含 `nodes` 与 `can*`(待确认)。

<a id="m14"></a>

## 14. 管理端 · 安装向导、找回、确认与 CSRF 令牌(管理端)

**模块约定**

> 路由总数 24,来源 EdgeAdmin/internal/web/actions/default/{csrf,recover,setup,setup/confirm,ui}/init.go(`/api/metrics` 3 条另见文末补充小节,不计入 24 条)
说明:
- **与任务描述的差异**:路由清单 JSON 实际为 24 条,其中没有 `/api/metrics`,却包含 `GET /ui/image/:fileId`。`/api/metrics` 的 3 条注册在 `EdgeAdmin/internal/web/actions/default/api/init.go`,不在本 JSON 内,已读源码后写在文末补充小节。
- 参数传法:TeaGo 只解析查询串与表单,不解析 JSON 请求体。参数名为 params 结构体字段名首字母小写。`apiNodeJSON`、`dbJSON`、`adminJSON` 是"值为 JSON 字符串的表单字段"。
- 成功包络:`{"code":200,"message":"","data":{...}}`,`Content-Type: application/json; charset=utf-8`。下表"返回"均指 `data` 内容。
- 失败包络:`this.Fail(msg)` 输出 `{"code":400,"message":msg,"data":...,"errors":[...]}`,HTTP 状态码是否为 200 待确认(未读到框架 `failWithoutPanic` 末尾)。`this.FailField(field,msg)` 的输出格式待确认。`actionutils.Fail` 才会设置真实 HTTP 状态。本模块的失败大多是 `this.Fail`。
- 认证:本模块 24 条均不走登录鉴权(`NewUserMustAuth`),但各自受不同的 Helper 或口令保护:
  - **安装口令 `setupToken`**:进程启动时生成 16 字节 hex,打印在服务启动日志的 `[SETUP]` 行,常量时间比较。除 `/setup`、`/setup/`、`/setup/bootstrap`、`/setup/status` 与 `/admin/setup.html` 外,`/setup/*` 的写操作都要求表单字段 `setupToken`。失败为 **403 JSON** `安装口令无效，请查看服务启动日志中的 [SETUP] 口令`。
  - **setup.Helper**:系统已配置(`setup.IsConfigured()`)时,除 `/setup/status` 外都重定向 `/`。
  - **recover.Helper**:非恢复模式(`IsRecoverMode`)时重定向 `/`。
  - **confirm.Helper**:非"新安装待确认"(`setup.IsNewInstalled()`)时重定向 `/`。
  - 三个 HTML 页面路由(`/admin/setup.html`、`/admin/recover.html`、`/admin/confirm.html`)注册在 Helper 之外,不受上述限制,任何时候都可访问。
- CSRF:本模块 **所有 POST 均未声明 CSRF 校验**(无 `csrfToken` 参数)。`/csrf/token` 只是为其他需 CSRF 的接口发令牌。通用的 CSRF 失败形态(其他模块)为 HTTP 403 纯文本 `表单已失效，请刷新页面后重试(001)`。
- 页面类 GET 在 `Content-Type` 上返回 HTML 文件内容,文件缺失时为 **503 纯文本**。
- 注意:`setup/*`、`recover/*`、`setup/confirm` 的 POST 在响应里会回显或返回敏感数据,见各行"风险"。本文不列任何真实值。

| 模块 | 接口名称 | 方法 | 路径 | 请求参数 | 响应数据 | 主要错误 | 功能说明 | 认证/权限 |
|---|---|---|---|---|---|---|---|---|
| admin-13-setup-recover-csrf | 获取 CSRF 令牌 | GET | `/csrf/token` | 无 | `data.token` string,32 字节 base64 RawURL,绑定当前 session sid,有效期 1800 秒<br>错误(Fail):`请求速度过快，请稍后刷新后重试`<br>`Init()` 注入的 `tea*` 页面键是否出现在响应中:待确认 | 见模块约定 | 为后续需 CSRF 的表单/POST 签发令牌,作为 `csrfToken` 提交。**未登录时存在全局限速**:同一秒内第二次调用即被拒,限速按进程全局而非按 IP,可被他人占用。session 创建时机未核实,待确认。 | 无需登录(登录态可选) |
| admin-13-setup-recover-csrf | 找回页面(静态文件) | GET | `/admin/recover.html` | 无 | HTML 文件 `public/admin/recover.html`,`Cache-Control: no-store`<br>文件缺失:503 纯文本 | 见模块约定 | 返回找回(恢复模式)前端页面。不受 recover.Helper 保护。 | 无需登录 |
| admin-13-setup-recover-csrf | 找回入口 | GET | `/recover` | 无 | 同上:返回 `public/admin/recover.html`,`no-store`;缺失 503 纯文本 | 见模块约定 | 与 `/admin/recover.html` 内容相同,但受 recover.Helper 保护:非恢复模式时重定向 `/`。 | 无需登录(仅恢复模式可用) |
| admin-13-setup-recover-csrf | 找回:校验 API 节点 | POST | `/recover/validateApi` | `protocol` string 必填(未校验取值)<br>`host` string 必填<br>`port` string 必填,纯数字<br>`nodeId` string 必填<br>`nodeSecret` string 必填<br>各值首尾的引号和空格会被剔除 | `data.apiNode`:`protocol`,`host`,`port`,`nodeId`,`nodeSecret`(回显调用者自己提交的值),`hosts[]`(该 API 节点下已启用节点的监听 host) | 错误:`FailField("host", ...)` 等,具体文案待确认 | 用提交的凭据连接指定 API 节点,取出其监听地址列表,供下一步替换。**风险**:无 CSRF、无登录,仅靠 recover.Helper 与提交的节点凭据。 | 无需登录(仅恢复模式可用) |
| admin-13-setup-recover-csrf | 找回:更新 hosts | POST | `/recover/updateHosts` | `protocol`,`host`,`port`,`nodeId`,`nodeSecret`(同上,但**无 Must 校验**,缺失时的行为待确认)<br>`oldHosts` string 数组<br>`newHosts` string 数组,长度须与 `oldHosts` 相等 | 成功无业务字段(`data` 内容待确认) | 错误:`参数配置错误，请刷新页面后重试`(长度不等)等,其余待确认 | 在 API 节点 JSON 上做字符串 `ReplaceAll` 把旧 host 换成新 host,写入 `configs/<ConfigFileName>`,重载 RPC,并把 `teaconst.IsRecoverMode` 置 false(退出恢复模式)。**风险**:无 CSRF、无登录;替换为纯字符串替换,可能误改非 host 内容。 | 无需登录(仅恢复模式可用) |
| admin-13-setup-recover-csrf | 安装向导页面(静态文件) | GET | `/admin/setup.html` | 无 | HTML 文件 `public/admin/setup.html`,`no-store`<br>缺失:503 纯文本 | 见模块约定 | 返回安装向导前端页面。不受 setup.Helper 保护,系统安装后依旧可访问。 | 无需登录 |
| admin-13-setup-recover-csrf | 安装向导入口 | GET | `/setup` | 无 | 重定向到 `/admin/setup.html` | 见模块约定 | 跳转到安装向导页面。受 setup.Helper:已配置时重定向 `/`。 | 无需登录 |
| admin-13-setup-recover-csrf | 安装向导入口(带斜杠) | GET | `/setup/` | 无 | 同 `/setup`(同一处理器 `IndexAction`) | 见模块约定 | 同上,仅路径多一个斜杠。 | 无需登录 |
| admin-13-setup-recover-csrf | 安装向导初始数据 | GET | `/setup/bootstrap` | 无 | `data.bootstrap`:`currentHost`(仅当请求 Host 是非回环 IP 字面量时有值,否则 `""`),`licenseURL`,`version` | 见模块约定 | 为前端预填当前访问地址、授权获取地址与版本。无需口令。 | 无需登录 |
| admin-13-setup-recover-csrf | 校验安装口令 | POST | `/setup/verifyToken` | `setupToken` string 必填 | 成功:`data` 为空对象 `{}`(`{"code":200,"message":"","data":{}}`) | 失败:**HTTP 403 JSON** `安装口令无效，请查看服务启动日志中的 [SETUP] 口令` | 校验用户输入的安装口令,不产生副作用。 | 无需登录(受 setup.Helper) |
| admin-13-setup-recover-csrf | 校验 API 节点安装参数 | POST | `/setup/validateApi` | `setupToken` string 必填<br>`mode` string 必填,只能为 `new`,否则 `仅支持新 API 节点安装`<br>`newPort` string 必填,纯数字,长度 4..5,值 1024..65534<br>`newHost` string 必填,IP 或主机名;纯数字点分但非合法 IP 的被拒绝<br>`licenseKey` string 必填,以 `ERK2-` 开头,≤256,无空白 | `data.apiNode`:`mode`,`newPort`,`newHost`,`licenseKey`,`licenseKeyHint`(前 9 个字符加 `****`) | 错误:口令无效 403;各字段格式错误(文案待确认,仅确认 `仅支持新 API 节点安装`) | 校验即将安装的 API 节点参数并回显给前端,供最后一步 `install` 提交。**风险**:原样回显完整 `licenseKey`;`apiNode` 在校验前就已赋值,失败时 `data` 是否仍带 `apiNode` 待确认。 | 无需登录(口令 + setup.Helper) |
| admin-13-setup-recover-csrf | 校验数据库连接 | POST | `/setup/validateDb` | `setupToken` string 必填<br>`host` string 必填,IP 或 `^[\w.-]+$`<br>`port` string 必填,纯数字<br>`database` string 必填<br>`username` string 必填,`^[\w\.-]+$`<br>`password` string<br>`accessLogKeepDays` int(取值范围待确认) | `data.db`:`host`,`port`,`database`,`username`,`passwordMask`(与密码等长的 `*`,会暴露密码长度),`accessLogKeepDays` | 错误:连接失败、MySQL 版本低于 5.7.8 等,文案待确认 | 真实连接 MySQL(30 秒超时):库不存在(Error 1049)时自动 `CREATE DATABASE`;创建并删除测试表 `edgeTest1`;检查版本 ≥5.7.8。**副作用**:会创建数据库。**缺陷**:1049 分支内 `db, err :=` 遮蔽外层变量,内层连接未关闭(连接泄漏)。 | 无需登录(口令 + setup.Helper) |
| admin-13-setup-recover-csrf | 校验管理员账号 | POST | `/setup/validateAdmin` | `setupToken` string 必填<br>`adminUsername` string 必填,`^[a-zA-Z0-9_]+$`,≤100<br>`adminPassword` string 必填,`^[a-zA-Z0-9_]+$`,trim 后长度 8..100<br>`adminPassword2` string 必填,须与 `adminPassword` 一致 | `data.admin`:`username`,`passwordMask`(等长 `*`) | 错误:格式、长度、两次不一致(文案待确认) | 校验首个管理员的用户名与密码。仅允许字母数字下划线,不允许特殊字符的密码是已知限制。 | 无需登录(口令 + setup.Helper) |
| admin-13-setup-recover-csrf | 执行安装 | POST | `/setup/install` | `setupToken` string 必填<br>`apiNodeJSON` string 必填(JSON 字符串)<br>`dbJSON` string 必填(JSON 字符串,含**明文** `password`)<br>`adminJSON` string 必填(JSON 字符串)<br>三者的具体键沿用前三个校验接口回显的对象(`apiNode`/`db`/`admin`) | 成功:`data.started` = true(安装异步执行,用 `/setup/status` 轮询) | 错误:`已有安装任务正在执行，请等待当前安装完成`;JSON 解析错误;目录检查错误;失败文案会拼接命令 stdout/stderr 与日志路径 | 同步校验后异步安装,调用 `edgeroute-api setup`,授权码通过环境变量 `EDGEROUTE_LICENSE_KEY` 传递。**缺陷**:对 `newHost`/`newPort` 仅检查非空且不含 `\r\n"`,**不复核** `validateApi` 的格式与端口范围,`dbJSON` 各字段同样未再校验,可绕过前面几步直接提交。 | 无需登录(口令 + setup.Helper) |
| admin-13-setup-recover-csrf | 安装状态 | POST | `/setup/status` | 无(**不要求口令**) | `data`:`statusText`,`running`,`installed`,`failed`,失败时可选 `error`(可能含命令输出与服务器路径)<br>失败路径外无其他错误 | 见模块约定 | 轮询安装进度。在 Helper 中被特判:无需口令,系统安装完成后依旧可调。**风险**:`error` 任何人可读,可能泄露路径与命令输出。 | 无需登录、无需口令 |
| admin-13-setup-recover-csrf | 检测本机数据库 | POST | `/setup/detectDB` | `setupToken` string 必填 | `data.localDB`:`host`,`port`,`username`(固定 `""`),`password`(固定 `""`),`canInstall` bool | 见模块约定 | 依次尝试 127.0.0.1(原文 `127.0.1` 待核对)、localhost、172.20.0.2 的 3306 端口,每次 3 秒超时。`canInstall = linux && amd64 && os.Getgid()==0`,用的是**组 ID 而非 UID**,疑似缺陷。 | 无需登录(口令 + setup.Helper) |
| admin-13-setup-recover-csrf | 判断是否本机 IP | POST | `/setup/checkLocalIP` | `host` string(**无需口令**,是否必填待确认) | `data.isLocal` bool<br>IP 无法解析时默认 true;私网判断函数 `utils.IsLocalIP` 未读,具体判定规则待确认 | 见模块约定 | 判断填写的数据库地址是否本机/内网,用于决定是否展示本机 MySQL 安装入口。 | 无需登录(受 setup.Helper) |
| admin-13-setup-recover-csrf | 一键安装 MySQL | POST | `/setup/mysql/installPopup` | `setupToken` string 必填 | 成功:`data`:`isOk` = true,`user` = `root`,`password`(**生成的 root 明文密码**),`dir` | 失败:仍返回 `code:200`,`data`:`isOk` = false,`err` string | **同步长耗时**:下载并安装 MySQL 到 `/usr/local/mysql`。调用时重置全局 `SharedLoger`(原样拼写)。**风险**:响应携带 root 明文密码;失败也返回 `code:200`,调用方必须看 `data.isOk`。 | 无需登录(口令 + setup.Helper) |
| admin-13-setup-recover-csrf | MySQL 安装日志 | POST | `/setup/mysql/installLogs` | `setupToken` string 必填 | `data.logs` = `SharedLogger.ReadAll()` 的结果,元素结构待确认(未读 `mysqlinstallers/utils`) | 见模块约定 | 读取一键安装 MySQL 的进度日志,供前端轮询。 | 无需登录(口令 + setup.Helper) |
| admin-13-setup-recover-csrf | 确认页面(静态文件) | GET | `/admin/confirm.html` | 无 | HTML 文件 `public/admin/confirm.html`<br>缺失:503 纯文本 | 见模块约定 | 返回"确认 API 节点"页面。不受 confirm.Helper 保护。 | 无需登录 |
| admin-13-setup-recover-csrf | 确认页初始数据 | GET | `/setup/confirm/bootstrap` | 无 | `data.bootstrap`:`endpoints[]`(为空时为 `[""]`),`nodeId`,`canInstall`(读取配置失败时为 true) | 见模块约定 | 返回已有 API 配置,供确认页预填。受 confirm.Helper:非新安装时重定向 `/`。 | 无需登录(仅新安装待确认时可用) |
| admin-13-setup-recover-csrf | 确认页面入口 | GET | `/setup/confirm` | 无 | 返回 `public/admin/confirm.html`(与 POST 同一处理器 `IndexAction`) | 见模块约定 | 同 `/admin/confirm.html`,受 confirm.Helper 保护。 | 无需登录(仅新安装待确认时可用) |
| admin-13-setup-recover-csrf | 提交确认 | POST | `/setup/confirm` | `endpoints` string 数组,必填,每项为 URL,仅允许 http/https,缺端口补 80/443<br>`nodeId` string 必填<br>`secret` string 必填<br>`setupToken` string 必填(安装口令,见管理端启动日志 `[SETUP]` 行;er-v45 起)<br>(`Must` 声明了规则但**代码未使用**,实际校验由手写判断完成) | 成功无业务字段 | 错误:`请输入至少一个API节点地址`;`请输入NodeId`;`请输入Secret`;协议错误;连接失败;角色不是 `admin`(各文案待确认) | 用提交的凭据逐个连接 API 节点并要求角色为 `admin`,通过后写入配置(`RPCDisableUpdate=true`)并重载 RPC。er-v45 起 POST 需带安装口令,口令无效返回 403 `安装口令无效,请查看服务启动日志中的 [SETUP] 口令`;仍无登录、无 CSRF,并受 `IsNewInstalled` 限制;角色不符分支未调用 `client.Close()`(连接泄漏)。 | 无需登录(仅新安装待确认时可用) |
| admin-13-setup-recover-csrf | 公开图片 | GET | `/ui/image/:fileId` | 路径参数:`fileId` int64 | 成功:图片字节流,`Content-Type` 按文件扩展名确定(缺省 `image/png`),`Last-Modified: Fri, 06 Sep 2019 08:29:50 GMT`(固定值),逐块输出<br>不存在或非公开:`NotFound` 走 `FailPage`,返回 HTML 错误页,HTTP 状态按错误映射(not found 为 404);若带 `X-Requested-With` 头则返回纯文本错误 | 见模块约定 | 输出被标记为公开(`IsPublic`)的文件,用于登录页 Logo 等。**风险**:不限定图片类型,任何公开文件都按扩展名输出,且未设 `X-Content-Type-Options: nosniff`;`Last-Modified` 为固定值会使更新后的图片被缓存。 | 无需登录 |


### 补充:前端性能上报 `/api/metrics`(3 条,不在路由清单 JSON 内)

来源 `EdgeAdmin/internal/web/actions/default/api/init.go`、`web_vitals_action.go`、`performance_timeline_action.go`、`performance_query_action.go`、`json.go`、`metrics_limiter.go`、`EdgeCommon/pkg/frontendmetrics/frontendmetrics.go`。

通用约定:
- 认证:`helpers.NewUserMustAuth(AdminModuleCodeCommon)`,必须已登录管理员,公共模块。代码注释称**不要求 CSRF**,理由是 sendBeacon 携带同源 cookie、`application/json` 跨站提交需 CORS 预检。
- 请求体:**JSON**(不同于 TeaGo 表单),上限 64KB(`65536` 字节),严格解码:**拒绝未知字段**、拒绝尾随数据(须只有一个 JSON 对象)。
- **独立错误格式**(上报两条):`{"code":状态码,"message":"..."}`,`Content-Type: application/json; charset=utf-8`,HTTP 状态与 `code` 一致,**无 `data` 字段**。
- 限流:按管理员 ID 固定窗口,每分钟 60 次,超限 **429** 加响应头 `Retry-After: 60`,`message` 为 `上报过于频繁，请稍后再试`。上报是 best-effort,前端会忽略 429。
- 成功:**HTTP 204 无响应体**。持久化 RPC 失败只记服务端日志,不向前端返回错误(所以 204 不代表已入库)。
- 数据归属:写入的 `UserId` 取当前 `AdminId`。

| 接口名称 | 方法 | 路径 | 主要请求参数 | 返回数据结构 | 功能说明 | 认证 |
|---|---|---|---|---|---|---|
| 上报 Web Vitals | POST | `/api/metrics/web-vitals` | JSON 请求体(≤64KB,不允许未知字段):<br>`url` string 必填<br>`userAgent` string<br>`timestamp` int64,客户端毫秒时间<br>`metrics[]` 必填,1..50 个,元素 `{name,value,id,delta,rating,navigationType,url,userAgent,timestamp}`(`name` string,`value`/`delta` float,`timestamp` int64,其余 string;所有键都必须在结构里声明,多传未知键会 400) | 成功 204 无体<br>错误(独立格式):400 `请求格式无效`(JSON 非法、超 64KB、未知字段、尾随数据);400 `metrics array is empty`;400 `too many metrics`;400 `url is required`;429 `上报过于频繁，请稍后再试` | 每个已知指标写一行:`name` 仅接受 `LCP`/`FCP`/`CLS`/`INP`/`TTFB`/`FID`,**未知名称被静默丢弃**;`Page` 取外层 `url`,`Value` 为保留 2 位小数的字符串,`Timestamp` 取外层 `timestamp`(元素自身的 url/timestamp 不入库)。 | 需登录(公共模块) |
| 上报 Performance Timeline | POST | `/api/metrics/performance-timeline` | JSON 请求体(≤64KB,不允许未知字段):<br>`navigation` 对象:`url` 必填,`userAgent`,`timestamp` int64,以及整型 `dnsLookup`,`tcpConnect`,`sslHandshake`,`requestTime`,`responseTime`,`ttfb`,`domProcessing`,`domInteractive`,`domComplete`,`loadComplete`,`totalTime`,string `navigationType`<br>`resources[]` 可选,≤500 个,元素 `{url,pageUrl,userAgent,timestamp,name,initiatorType,duration,transferSize,encodedBodySize,decodedBodySize,cacheStatus}`(`cacheStatus` 注释取值 hit/miss/validated) | 成功 204 无体<br>错误(独立格式):400 `请求格式无效`;400 `navigation.url is required`;400 `too many resources`;429 `上报过于频繁，请稍后再试` | 写 1 行 `keys=navigation`(Value 为各耗时的 JSON 串)加每个 `duration>100` 毫秒的资源各 1 行 `keys=resource`(Value 为 `name`/`initiatorType`/`duration`/`transferSize`/`encodedBodySize`/`decodedBodySize`/`cacheStatus` 的 JSON 串);资源无 `pageUrl` 时沿用 `navigation.url`;慢资源阈值外的资源不入库。 | 需登录(公共模块) |
| 查询前端性能指标 | GET | `/api/metrics/query` | 查询参数:<br>`metricType` string 必填,`web_vitals`/`performance_timeline`<br>`range` string,`24h`/`7d`/`30d`,其他值按 `24h`<br>`page` string,可选,trim 后 ≤512<br>`limit` int32,≤0 取 1000,>5000 取 5000 | 成功(标准包络):`data.records[]`:`page`,`metricType`,`keys`,`value`(字符串;web_vitals 为数值串,performance_timeline 为 JSON 串),`timestamp`<br>错误:`this.Fail` 标准格式 `invalid metricType` / `page is too long`;RPC 错误经 `actionutils.Fail` 映射(含 EdgeAPI 侧 `ValidateAdmin` 的权限拒绝)<br>**注意**:此条错误格式是标准包络,不是上面两条的独立格式 | 按类型、时间窗口(`now-range` 到 `now`,格式 `200601021504`)与页面查询已入库的指标记录。`timestamp` 的单位及排序待确认。 | 需登录(公共模块) |

<a id="m15"></a>

## 15. 用户端 · 认证与会话(用户端)

**模块约定**

> 路由总数 17,来源 EdgeUserPortal/internal/web/server.go
通用约定:错误统一为 `{"error":"消息"}`;所有 JSON 请求体使用 `DisallowUnknownFields`(出现未知字段即 400「请求格式无效」),大小上限 64KB。本模块带 `csrfToken` 的 POST 接口需先 `GET /api/session` 获取会话 Cookie 与 `csrfToken`,再把 `csrfToken` 放入 JSON 请求体(不是请求头)。

| 模块 | 接口名称 | 方法 | 路径 | 请求参数 | 响应数据 | 主要错误 | 功能说明 | 认证/权限 |
|---|---|---|---|---|---|---|---|---|
| user-01-auth | 门户配置 | GET | `/api/portal` | 无 | 200:<br>`landingPageEnabled` bool 落地页开关<br>`productName` string<br>`siteName` string<br>`title` / `description` / `footerText` string<br>`privacyPolicyURL` / `termsURL` string<br>`logoURL` string(有 Logo 时为 `/api/portal/logo`,否则空)<br>`faviconURL` string(同理 `/api/portal/favicon`)<br>`showFinance` / `showVersion` bool<br>`version` string<br>`showTrafficCharts` / `showBandwidthCharts` bool<br>`defaultTrafficDays` int(7 或 31)<br>`bandwidthUnit` string<br>`requireIdentity` bool | 错误:503 用户门户配置暂时不可用 | 返回登录页/门户展示配置。实名认证配置读取失败或未启用时 `requireIdentity` 为 false(失败即关闭)。 | 公开 |
| user-01-auth | 门户 Logo | GET | `/api/portal/logo` | 请求头 `If-None-Match`(可选,ETag 命中返回 304) | 200:图片二进制(`Content-Type` 为 image/*,带 `ETag`、`Cache-Control: public, max-age=300`、`X-Content-Type-Options: nosniff`)<br>304:ETag 命中无正文<br>404:未配置/非图片(纯文本 404,非 JSON) | 见模块约定 | 输出后台配置的门户 Logo 文件;MIME 非图片则尝试按扩展名、内容嗅探,仍非图片则 404。 | 公开 |
| user-01-auth | 门户 Favicon | GET | `/api/portal/favicon` | 同 Logo | 同 Logo(与 Logo 共用处理函数 `apiPortalAsset`,按路径后缀 `/favicon` 取 Favicon 文件) | 见模块约定 | 输出后台配置的站点图标。 | 公开 |
| user-01-auth | 获取会话 | GET | `/api/session` | 无(依赖会话 Cookie) | 200:见「公共结构 SessionResponse」 | 错误:500 创建会话失败 | 返回当前会话状态;无有效会话时自动创建匿名会话(`theme` 默认 `light`)并写入会话 Cookie。前端启动时先调用以取得 `csrfToken`。 | 公开 |
| user-01-auth | 注册配置 | GET | `/api/registration` | 无 | 200:<br>`enabled` bool 是否开放注册<br>`requireVerification` bool 注册需后台审核<br>`requireEmail` bool(恒为 true)<br>`requireMobile` bool<br>`passwordResetEnabled` bool<br>`captcha` 见「公共结构 Captcha」 | 错误:502 读取注册配置失败/读取安全验证配置失败 | 提供注册与找回密码页所需的开关和人机验证配置。 | 公开 |
| user-01-auth | 用户注册 | POST | `/api/register` | JSON 体:<br>`username` string 必填,正则 `^[A-Za-z0-9_]{3,64}$`<br>`password` string 必填,8-128 位且含大小写字母、数字、符号<br>`email` string 必填,≤254,合法邮箱(会转小写)<br>`mobile` string 可选(后台强制时必填),≤32,仅 `0-9+(). -`<br>`captchaToken` string 人机验证令牌(启用验证码时必填)<br>`csrfToken` string 必填 | 201:<br>`registered` bool<br>`requireEmailVerification` bool<br>`requiresApproval` bool(仅需验证/审核时出现)<br>`autoLoginFailed` bool(未自动登录时出现)<br>`session` 见 SessionResponse<br>自动登录成功时仅含 `registered`/`requireEmailVerification`/`session`,并下发新会话 Cookie | 错误:400 参数格式无效;403 CSRF 校验失败(含已登录)/未开放注册/需人机验证;409 邮箱或用户名已被使用;429 注册尝试过多;502 | 创建用户账号。无需邮箱验证且无需审核时自动登录(轮换会话);否则返回 201 并要求前端引导验证/等待审核。按客户端 IP 限流。 | 公开(需会话 CSRF,且当前会话未登录) |
| user-01-auth | 校验邮箱验证码 | POST | `/api/email-verification/verify` | JSON 体:<br>`code` string 必填,1-128(验证链接中的 code)<br>`csrfToken` string 必填 | 200:`verified` bool(true)、`email` string | 错误:400 验证链接无效或已过期;403 CSRF 校验失败;502 邮箱验证服务暂时不可用 | 用邮件中的验证码/链接完成邮箱验证。 | 公开(需会话 CSRF) |
| user-01-auth | 重发验证邮件 | POST | `/api/email-verification/resend` | JSON 体:<br>`email` string 必填,合法邮箱,≤254<br>`csrfToken` string 必填 | 200:`sent` bool(true) | 错误:400 电子邮箱格式无效;403;429 发送请求过多(本地限流或上游 temporarily limited);502 验证邮件发送失败 | 重新发送邮箱验证邮件,按客户端 IP 限流。 | 公开(需会话 CSRF) |
| user-01-auth | 发送密码重置验证码 | POST | `/api/password-reset/send` | JSON 体:<br>`email` string 必填,合法邮箱<br>`captchaToken` string 人机验证令牌<br>`csrfToken` string 必填 | 200:`sent` bool、`codeLength` int(验证码位数) | 错误:400 电子邮箱格式无效;403 CSRF 校验失败(含已登录)/请先完成人机验证;429 请求过于频繁;502 读取安全验证配置失败;503 密码重置邮件服务暂不可用 | 向邮箱发送找回密码验证码,限流键 `password-reset-send:<IP>`。 | 公开(需会话 CSRF,且当前会话未登录) |
| user-01-auth | 确认重置密码 | POST | `/api/password-reset/confirm` | JSON 体:<br>`email` string 必填<br>`code` string 必填,4-32<br>`newPassword` string 必填,8-128 位且含大小写字母、数字、符号<br>`captchaToken` string<br>`csrfToken` string 必填 | 200:`ok` bool(true) | 错误:400 邮箱格式无效/验证码或新密码格式无效/验证码无效或已过期;403 CSRF/人机验证/密码被锁定(PASSWORD_LOCKED);429 验证尝试过多;502 读取安全验证配置失败/密码重置服务暂不可用 | 校验验证码并设置新密码;成功后踢出该用户所有会话并重置限流计数。 | 公开(需会话 CSRF,且当前会话未登录) |
| user-01-auth | 用户登录 | POST | `/api/login` | JSON 体:<br>`username` string 必填,≤200<br>`password` string 必填,≤512<br>`otpCode` string 可选,MFA 动态码或恢复码<br>`captchaToken` string 人机验证令牌<br>`csrfToken` string 必填 | 200:SessionResponse,并下发新会话 Cookie(会话轮换) | 错误:400 请输入有效的用户名和密码;401 用户名或密码错误,若需 MFA 则返回 `{"error":"...","code":"mfa_required"}`;403 CSRF 校验失败/请先完成人机验证;429 登录尝试过多;502 暂时无法连接 API 服务/读取安全验证配置失败 | 账号密码登录(可带 MFA)。按客户端 IP 限流,成功后重置计数;登录 IP 透传给后端。 | 公开(需会话 CSRF) |
| user-01-auth | OAuth 登录方式列表 | GET | `/api/oauth/providers` | 无 | 200:`providers` array,元素:<br>`code` string 提供方代码<br>`name` string 显示名<br>`startURL` string(`/oauth/<code>/start`)<br>按 `name` 排序 | 错误:502 读取登录配置失败 | 返回已启用并配置有效的第三方登录提供方(含 Telegram)。 | 公开 |
| user-01-auth | OAuth 发起授权 | GET | `/oauth/{provider}/start` | 路径参数:`provider` string 提供方代码(不区分配置外的值,未配置则 404)<br>查询参数:`action` string 可选,`bind` 表示已登录用户绑定账号 | 非 JSON 接口:<br>普通 OAuth2:302 重定向到提供方授权地址(带 PKCE `code_challenge`、`state`)<br>Telegram:200 HTML 页(内嵌 Telegram Login Widget) | 失败:303 重定向 `/login?oauth-error=<configuration/session/internal/provider>`<br>404:提供方未配置 | 浏览器跳转型接口,不是 AJAX。`action=bind` 时要求当前为已登录的普通用户会话(管理员协助会话不可绑定),`state` 与会话绑定防 CSRF。 | 公开(`action=bind` 需已登录) |
| user-01-auth | OAuth 回调 | GET | `/oauth/{provider}/callback` | 路径参数:`provider`<br>查询参数:`state` string 必填<br>`code` string 授权码(非 Telegram 必填,≤4096)<br>`error` string 提供方返回的错误<br>Telegram 回调另含 `id`、`first_name`、`last_name`、`username`、`photo_url`、`auth_date`、`hash`(签名校验,`auth_date` 有效期 24 小时) | 非 JSON 接口,全部为重定向:<br>登录成功:303 → `/`(下发新会话 Cookie)<br>绑定成功:303 → `/security?oauth-bound=<provider>`<br>登录 | 失败:303 → `/login?oauth-error=<denied/state/session/token/profile/identity/unlinked/internal/configuration>`<br>绑定失败:303 → `/security?oauth-error=<session/token/profile/identity/already-linked/state...>`<br>404:提供方未配置 | 处理第三方授权回调:校验 `state`(一次性)与会话一致后换取令牌、拉取用户信息,按 `subject` 登录已绑定账号或执行绑定。 | 公开(绑定流程要求原会话为同一登录用户) |
| user-01-auth | 管理员协助登录 | POST | `/api/admin-login` | JSON 体:<br>`ticket` string 必填,长度必须为 32(管理端签发的一次性票据)<br>`csrfToken` string 必填 | 200:SessionResponse(`impersonated` 为 true,`sensitiveWritesLocked` 为 true),下发新会话 Cookie | 错误:400 管理员登录票据格式无效;401 票据无效或已使用/与当前 IP 不匹配/目标用户不存在;403 CSRF 校验失败;429 登录尝试过多;502 暂时无法验证管理员登录票据 | 管理员通过一次性票据以协助会话进入用户端;票据绑定 IP,协助会话下所有敏感写操作被禁止。 | 公开(需会话 CSRF + 有效管理员票据) |
| user-01-auth | 退出登录 | POST | `/api/logout` | JSON 体:`csrfToken` string 必填 | 200:`ok` bool(true),并清除会话 Cookie | 错误:400;403 CSRF 校验失败 | 销毁当前会话并清除 Cookie。 | 需会话 + CSRF(未登录会话也可调用,处理函数未调用 `requireAPIUser`) |
| user-01-auth | 设置主题 | POST | `/api/theme` | JSON 体:<br>`theme` string 必填,枚举 `system`/`light`/`dark`<br>`csrfToken` string 必填 | 200:`ok` bool(true) | 错误:400 主题设置无效/请求格式无效;401 请先登录;403 CSRF 校验失败 | 保存当前会话的界面主题。 | 需登录 + CSRF(处理函数内校验,未套用 `sensitiveUserWrite`) |


### 公共结构

### SessionResponse
`GET /api/session`、登录、协助登录、注册、修改密码等接口返回:
- `authenticated` bool 是否已登录
- `impersonated` bool 是否管理员协助会话
- `sensitiveWritesLocked` bool 敏感写操作是否被锁定(协助会话为 true)
- `csrfToken` string 当前会话 CSRF 令牌
- `theme` string 主题(`system`/`light`/`dark`)
- `licenseAllowed` bool 平台授权是否有效
- `licenseStatusAvailable` bool 授权状态是否读取成功
- `licenseReadOnly` bool 授权失效只读模式(= `!licenseAllowed`,仅状态可读时有意义)
- `licenseFeatures` object<string,bool> 已授权功能码,仅用于导航,各 API 仍会单独校验授权

### Captcha
`enabled` bool、`provider` string、`siteKey` string、`endpoint` string、`registration` bool、`login` bool、`passwordReset` bool。启用后三个场景开关同时为 true。

### 限流说明
`s.limiter.Allow(...)` 的具体阈值与窗口未在本模块处理函数内定义,待确认:限流器配置位置。

<a id="m16"></a>

## 16. 用户端 · 安全设置(用户端)

**模块约定**

> 路由总数 11,来源 EdgeUserPortal/internal/web/server.go
通用约定:错误统一为 `{"error":"消息"}`;JSON 请求体不允许未知字段,上限 64KB;凡写操作(含 DELETE)均需在 JSON 请求体中携带 `csrfToken`(取自 `GET /api/session`)。`sensitiveUserWrite` 中间件额外拒绝管理员协助会话(403「管理员协助会话不能修改用户资源或安全配置」),授权确认失效时返回 403 只读提示,并记录一条通用操作审计日志。

| 模块 | 接口名称 | 方法 | 路径 | 请求参数 | 响应数据 | 主要错误 | 功能说明 | 认证/权限 |
|---|---|---|---|---|---|---|---|---|
| user-02-security | 安全设置概览 | GET | `/api/security` | 无 | 200:<br>`mfa.enabled` bool 是否已启用 MFA<br>`mfa.recoveryCodeCount` int 剩余恢复码数<br>`oauthProviders` array,元素:`code` string、`name` string、`bound` bool 是否已绑定 | 错误:401 请先登录;502 读取安全设置失败/OAuth2 配置不可用/账号状态不可用 | 返回当前用户的 MFA 状态与各第三方登录绑定状态。 | 需登录 |
| user-02-security | 解绑第三方登录 | DELETE | `/api/security/oauth/{provider}` | 路径参数:`provider` string 提供方代码(转小写)<br>JSON 体:`csrfToken` string 必填 | 200:`ok` bool(true) | 错误:400 invalid request body;403 CSRF token is invalid(此处消息为英文);404 OAuth2 provider is not configured;502 配置/解绑失败(英文消息) | 解除当前账号与指定第三方登录的绑定。 | 需登录 + CSRF 写保护 |
| user-02-security | 开始设置 MFA | POST | `/api/security/mfa/setup` | JSON 体:`csrfToken` string 必填 | 200:<br>`secret` string TOTP 密钥(32 位 base32)<br>`provisioningURI` string otpauth URI<br>`recoveryCodes` string[8],格式 `XXXX-XXXX-XXXXX` | 错误:403 CSRF 校验失败;409 多因素认证已经启用;500 生成密钥/恢复码失败;502 | 生成 TOTP 密钥和 8 个恢复码返回给前端。服务端不保存,需调用 confirm 时原样回传。 | 需登录 + CSRF 写保护 |
| user-02-security | 确认启用 MFA | POST | `/api/security/mfa/confirm` | JSON 体:<br>`secret` string 必填,长度 32(转大写)<br>`currentPassword` string 必填,当前登录密码<br>`code` string 必填,当前 TOTP 动态码<br>`recoveryCodes` string[] 必填,必须恰好 8 个,去除 `-`/空格后每个 13 位且不重复<br>`csrfToken` string 必填 | 200:`ok` bool(true) | 错误:400 动态验证码无效/恢复码无效,请重新开始设置;403 CSRF 校验失败;409 多因素认证已经启用;502 读取/保存安全设置失败 | 校验动态码后保存密钥并启用 MFA,恢复码以 SHA-256 哈希保存。 | 需登录 + CSRF 写保护 |
| user-02-security | 关闭 MFA | DELETE | `/api/security/mfa` | JSON 体:<br>`code` string 必填,当前 TOTP 动态码或恢复码<br>`csrfToken` string 必填 | 200:`ok` bool(true)(未启用 MFA 时也直接返回 ok) | 错误:401 动态验证码或恢复码错误;403 CSRF 校验失败;502 | 风险操作:需提供 MFA 动态码或恢复码方可关闭 MFA。 | 需登录 + CSRF 写保护 |
| user-02-security | API 访问密钥列表 | GET | `/api/access-keys` | 无 | 200:`accessKeys` array,元素:<br>`id` int64<br>`isOn` bool 是否启用<br>`uniqueId` string 密钥 ID(AccessKey ID)<br>`description` string 用途说明<br>`accessedAt` 数值,最近使用时间(单位待确认:取自 RPC 字段 `AccessedAt`)<br>不含 secret | 错误:401;502 读取 API 访问密钥失败 | 列出当前用户的 API 访问密钥。 | 需登录 |
| user-02-security | 创建 API 访问密钥 | POST | `/api/access-keys` | JSON 体:<br>`currentPassword` string 必填,当前登录密码<br>`otpCode` string,已启用 MFA 时必填<br>`description` string 必填,去空格后 1-100 个字符<br>`csrfToken` string 必填 | 201:`id` int64、`uniqueId` string、`secret` string(密钥仅在创建时返回) | 错误:400 请输入不超过 100 个字符的用途说明;403 CSRF 校验失败/`{"error":"请先完成实名认证","code":"identity_required"}`;409 API 访问密钥数量已达到上限;502 | 创建密钥,每个用户最多 20 个(`maxUserAccessKeys`)。后台启用实名且该功能(`api_access_keys`)被限制时,未实名返回 403。 | 需登录 + 需实名 + CSRF 写保护 |
| user-02-security | 启停 API 访问密钥 | PATCH | `/api/access-keys/{accessKeyID}` | 路径参数:`accessKeyID` int64 必填,>0<br>JSON 体:<br>`isOn` bool 是否启用(未传按 false)<br>`csrfToken` string 必填 | 200:`ok` bool(true) | 错误:400 API 访问密钥 ID 无效;403 CSRF 校验失败;404 API 访问密钥不存在或无权操作;502 更新 API 访问密钥失败 | 启用或停用指定密钥。 | 需登录 + CSRF 写保护 |
| user-02-security | 删除 API 访问密钥 | DELETE | `/api/access-keys/{accessKeyID}` | 路径参数:`accessKeyID` int64 必填,>0<br>JSON 体:`csrfToken` string 必填 | 200:`ok` bool(true) | 错误:400 API 访问密钥 ID 无效;403;404 API 访问密钥不存在或无权操作;502 删除 API 访问密钥失败 | 删除指定密钥。 | 需登录 + CSRF 写保护 |
| user-02-security | 域名接入策略 | GET | `/api/domain-ownership/policy` | 无 | 200:`requireOwnership` bool 平台是否要求域名所有权验证(响应头 `Cache-Control: no-store`) | 错误:401;503 暂时无法读取域名接入策略 | 供新建站点表单决定是否展示 TXT 验证步骤。创建站点的 API 仍会自行强制校验。 | 需登录 |
| user-02-security | 域名所有权验证 | POST | `/api/domain-ownership` | JSON 体:<br>`domain` string 必填,经 `domainpolicy.Normalize` 规范化并校验<br>`verify` bool,false 仅获取验证挑战,true 执行 DNS TXT 校验<br>`csrfToken` string 必填 | 200:Challenge 对象(`Cache-Control: no-store`):<br>`domain` string<br>`recordName` string TXT 记录名<br>`recordType` string<br>`value` string 记录值<br>`expiresAt` int64 挑战过期时间<br>`verifiedUntil` int64 验证有效期至 | 错误:400 域名不合法/不允许接入/域名验证未通过,请检查 TXT 记录、有效期或平台接入限制后重试;403 CSRF 校验失败;503 当前 API 尚不支持域名验证 | 生成或校验域名归属 TXT 挑战。时间戳单位待确认(源码未标注)。 | 需登录 + CSRF 写保护 |


### 说明
- 本模块仅 `POST /api/access-keys` 套用了 `requireIdentity`,其余接口无实名要求。
- 无任何接口使用 `licenseFeature`(财务授权)。

<a id="m17"></a>

## 17. 用户端 · 仪表盘与个人资料(用户端)

**模块约定**

> 路由总数 12,来源 EdgeUserPortal/internal/web/server.go
通用约定:错误统一为 `{"error":"消息"}`;JSON 请求体不允许未知字段,上限 64KB;写操作在 JSON 请求体中携带 `csrfToken`(身份材料上传例外,使用请求头 `X-CSRF-Token`)。`sensitiveUserWrite` 拒绝管理员协助会话、授权失效只读模式,并记录通用操作审计。实名认证相关接口统一受 `identityEnabled` 守卫:后台未开启实名返回 404「实名认证功能未开启」,配置读取失败返回 503「实名认证配置暂时不可用」。

| 模块 | 接口名称 | 方法 | 路径 | 请求参数 | 响应数据 | 主要错误 | 功能说明 | 认证/权限 |
|---|---|---|---|---|---|---|---|---|
| user-03-dashboard-profile | 仪表盘 | GET | `/api/dashboard` | 无 | 200:<br>`user` 见「公共结构 apiUser」<br>`dashboard` 见「公共结构 Dashboard」<br>`currentDay` string,当天 `YYYYMMDD`<br>`accounting` 流量口径说明对象:`version`、`source`(`customer_connection`)、`scope`(`account`)、`timezone`、`utcOffsetSeconds`、`originTrafficMeasured`、`internalTrafficMeasured`、`managementTrafficMeasured` | 错误:401;502 读取用户信息失败/读取统计数据失败 | 返回账户总览与流量/带宽统计(连接字节口径,含传输开销)。 | 需登录 |
| user-03-dashboard-profile | 个人资料 | GET | `/api/profile` | 无 | 200:<br>`user` 见 apiUser<br>`emailVerificationRequired` bool 后台开启邮箱验证且当前邮箱未验证<br>`passwordLocked` bool 密码是否被锁定(不可改) | 错误:401;502 读取个人资料失败/读取邮箱验证设置失败 | 返回当前用户资料与邮箱验证、密码锁定状态。 | 需登录 |
| user-03-dashboard-profile | 实名认证详情 | GET | `/api/identity` | 查询参数:`orgType` string 可选,默认 `individual`(取值 `individual`/`enterprise`,处理函数未校验) | 200:`identity` 为 null 或 UserIdentity 对象:<br>`id` int64<br>`orgType` string<br>`type` string(`idCard`/`enterpriseLicense`)<br>`realName` string<br>`number` string 证件号<br>`fileIds` int64[] 材料文件 ID<br>`status` string(常量:`none`/`rejected`/`verified`)<br>`createdAt`/`updatedAt`/`submittedAt`/`rejectedAt`/`verifiedAt` int64<br>`rejectReason` string<br>(字段均带 omitempty,零值不返回) | 错误:401;404 实名认证功能未开启;501 实名认证服务不可用;502 读取实名认证失败;503 | 查询指定主体类型下的实名认证记录。 | 需登录(且后台启用实名) |
| user-03-dashboard-profile | 上传实名材料 | POST | `/api/identity/materials` | 请求头:`X-CSRF-Token` 必填(取自会话 `csrfToken`)<br>`Content-Type: multipart/form-data`,仅一个表单字段 `file`(图片文件):JPG 或 PNG,≤5MB,像素≤4000 万,必须可完整解码 | 201:`fileId` int64 材料文件 ID(`Cache-Control: no-store`) | 错误:400 请选择材料图片/图片大小必须在 5 MB 以内/每次只能上传一张图片/请上传有效的 JPG 或 PNG 图片/图片内容不完整;403 请求验证失败,请刷新页面后重试;404;429 材料上传过于频繁(按用户限流);502 材料上传失败;503 材料上传暂不可用 | 上传单张身份证明材料,返回的 `fileId` 用于创建/更新实名认证的 `fileIds`。 | 需登录 + CSRF 写保护(CSRF 走请求头) |
| user-03-dashboard-profile | 创建实名认证 | POST | `/api/identity` | JSON 体:<br>`orgType` string 主体类型 `individual`/`enterprise`(去空格后传给后端)<br>`type` string 证件类型 `idCard`/`enterpriseLicense`<br>`realName` string 真实姓名/企业名称<br>`number` string 证件号<br>`fileIds` int64[] 材料文件 ID(来自上传接口)<br>`csrfToken` string 必填<br>字段必填性与格式校验在后端 RPC,待确认 | 201:`id` int64 新建记录 ID | 错误:400 请求格式无效/后端校验错误(直接透出错误文本);403 CSRF 校验失败;404;501 | 创建实名认证草稿,之后需调用 submit 提交审核。 | 需登录 + CSRF 写保护 |
| user-03-dashboard-profile | 更新实名认证 | PATCH | `/api/identity/{identityID}` | 路径参数:`identityID` int64<br>JSON 体:`type`、`realName`、`number`、`fileIds`、`csrfToken` 同创建(此时忽略 `orgType`) | 200:`ok` bool(true) | 错误:400/403/404/501 同创建 | 更新已有认证记录。`identityID` 非正整数时返回 400 `实名记录编号无效`,不会再落入创建分支(提交 eaa88f1 前会误新建记录,已修复)。 | 需登录 + CSRF 写保护 |
| user-03-dashboard-profile | 提交实名认证审核 | POST | `/api/identity/{identityID}/submit` | 路径参数:`identityID` int64,必须 >0<br>JSON 体:`csrfToken` string 必填 | 200:`ok` bool(true) | 错误:400 `实名记录编号无效` 或后端业务错误(透出文本);403 CSRF 校验失败(请求体格式错误同样返回 403);404;501 | 将认证记录提交审核。 | 需登录 + CSRF 写保护 |
| user-03-dashboard-profile | 更新个人资料 | PUT | `/api/profile` | JSON 体:<br>`fullname` string,去空格后 ≤200 字节<br>`company` string,≤200 个字符<br>`mobile` string,≤50 且仅含 `0-9+(). -`,可为空<br>`csrfToken` string 必填<br>邮箱不可通过本接口修改,沿用账户现有邮箱 | 200:同 `GET /api/profile`(`user`、`emailVerificationRequired`、`passwordLocked`) | 错误:400 名称/公司名称不能超过 200 个字符、手机号码格式无效、电子邮箱为必填项/邮箱格式无效;403;409 该电子邮箱已被其他账号使用;502 | 修改姓名、公司、手机号,返回更新后的资料。 | 需登录 + CSRF 写保护 |
| user-03-dashboard-profile | 修改密码 | PUT | `/api/profile/password` | JSON 体:<br>`currentPassword` string 必填<br>`newPassword` string 必填,8-128 位且含大小写字母、数字、符号,且不得与当前密码相同<br>`csrfToken` string 必填 | 200:SessionResponse(同 `GET /api/session`,下发新会话 Cookie) | 错误:400 当前密码不能为空,新密码长度必须为 8-128 个字符/新密码不能与当前密码相同/当前密码错误或新密码不符合要求;403 CSRF 校验失败/密码被锁定;500 更新登录会话失败;502 读取账号信息失败 | 风险操作:需验证当前密码。成功后轮换当前会话,并使该用户其他会话全部失效。 | 需登录 + CSRF 写保护 |
| user-03-dashboard-profile | 安全防护概览 | GET | `/api/security-overview` | 无 | 200:<br>`sites` array,元素:`id` string 站点引用、`name`、`domain` string、`wafEnabled` bool、`status` string(`available`/`unavailable`)、`todayBlocks` int64 今日拦截、`todayCaptcha` int64 今日验证码挑战、`todayObserved` int64 今日观察/日志数<br>`total` int 站点总数<br>`truncated` bool 超过 50 个站点被截断<br>按 `todayBlocks` 降序、同数按名称排序 | 错误:401;502 读取网站列表失败 | 汇总用户 HTTP 站点(不含四层)的今日 WAF 数据,最多统计前 50 个站点(按名称排序后截取),单站失败时该站 `status` 为 `unavailable`。 | 需登录 |
| user-03-dashboard-profile | 安全防护态势 | GET | `/api/security-posture` | 查询参数:<br>`range` string 可选,`24h`(默认)或 `7d`<br>`site` string 可选,站点引用(公开站点 ID),用于只看单个站点 | 200:<br>`from` / `to` string RFC3339(UTC)<br>`range` string<br>`sampled` bool 数据是否被采样/截断<br>`totalSites` int<br>`sites` array:`id`、`name`、`domain`、`wafEnabled`、`status`、`blocked`、`challenges`、`observed`(int64)<br>`blocked` / `challenges` / `observed` int64 总计<br>`categories` / `regions` / `countries` / `sources` array,元素 `name` string、`count` int、`countryCode` string(可选)<br>`events` array(最多 100 条,按时间倒序),元素:`id`、`siteId`、`siteName`、`timestamp` int64、`host`、`path`、`method`、`status` int32、`action`、`clientIP`、`category`、`region`、`countryCode`、`edgeTarget`(`regionId`、`regionName`、`countryCode`、`countryName`、`latitude`、`longitude` 等,完整字段待确认)<br>`timeline` array:`label` string、`blocked`、`challenges`、`observed` int64 | 错误:400 时间范围仅支持 24h 或 7d;401;403 当前套餐未开通网站防护态势功能(需套餐含查看访问日志功能 `UserFeatureCodeServerViewAccessLog`);404 网站不存在;502 读取网站列表失败 | 汇总用户站点近 24 小时/7 天的安全事件态势;最多统计前 50 个站点,超出或单站失败时 `sampled=true`。 | 需登录(另需套餐功能:查看访问日志) |
| user-03-dashboard-profile | 聚合统计 | GET | `/api/statistics` | 查询参数(二选一):<br>`from` + `to` string 自定义日期,格式 `YYYY-MM-DD` 或 `YYYYMMDD`,必须同时提供,from≤to,跨度≤100 天<br>`days` int 预设天数,默认 30,范围 1-100 | 200:<br>`currentDay` string `YYYYMMDD`<br>`accounting` 口径对象(`source=business_response`,`scope=websites`,其余字段同仪表盘 accounting)<br>`traffic`:`days` int,`stats` array 按天合并,元素:`day`、`bytes`、`cachedBytes`、`uncachedBytes`、`countRequests`、`countCachedRequests`、`countAttackRequests`、`attackBytes`、`countIPs`(int64;`timeFrom`/`timeTo`/`timeAt` 在日汇总行中被清空)<br>`bandwidth`:`days` int,`stats` array(`day`、`timeAt`、`bytes`、`bits`),`aggregation` 固定 `coincident_window_peak_upper_bound`<br>`waf`:`countDailyLog`、`countDailyBlock`、`countDailyCaptcha`、`countWeeklyBlock`、`countMonthlyBlock`(int64),`logDailyStats`/`blockDailyStats`/`captchaDailyStats` array(`day` string、`count` int64,覆盖区间内每一天) | 错误:400 统计日期范围无效;401;502 读取站点失败/读取站点统计失败(任一站点失败即整体失败) | 汇总用户全部 HTTP 站点(不含四层转发)的流量、带宽、WAF 统计。最多 8 个站点并发拉取;带宽峰值为各站点同窗口峰值之和的上界。 | 需登录 |


### 公共结构

### apiUser
`id` int64、`username` string、`fullname` string(可省略)、`company` string(可省略)、`mobile` string(可省略)、`email` string(可省略)、`verifiedEmail` string(可省略)、`emailVerified` bool(邮箱非空且等于已验证邮箱)、`nodeCluster` 对象 `{id int64, name string}`(可省略)。

### Dashboard
`GET /api/dashboard` 中的 `dashboard` 直接序列化后端 protobuf `ComposeUserDashboardResponse`,字段带 omitempty(零值不返回):
- `countServers` int64 站点数
- `monthlyTrafficBytes` / `monthlyPeekBandwidthBytes` int64 本月流量/峰值带宽
- `dailyTrafficBytes` / `dailyPeekBandwidthBytes` int64 今日流量/峰值带宽
- `dailyTrafficStats` array:`day`、`bytes`、`cachedBytes`、`attackBytes`、`countRequests`、`countCachedRequests`、`countAttackRequests`、`countIPs`
- `dailyPeekBandwidthStats` array:`day`、`bytes`
- `bandwidthPercentile` int32 带宽百分位、`bandwidthPercentileBits` int64
- `dailyCountIPs` int64

<a id="m18"></a>

## 18. 用户端 · 消息与工单(用户端)

**模块约定**

> 路由总数 12,来源 EdgeUserPortal/internal/web/server.go
通用约定:错误统一为 `{"error":"消息"}`;JSON 请求体不允许未知字段,上限 64KB;写操作(含 DELETE)均需在 JSON 请求体中携带 `csrfToken`(取自 `GET /api/session`)。`sensitiveUserWrite` 拒绝管理员协助会话、授权失效只读模式,并记录通用操作审计。
分页约定(`parseBoundedQueryInt`):`offset` 默认 0、有效范围 0-1000000;`size` 默认 20、有效范围 0-100。值非法或超出上限时**回落为默认值**(不是截断到上限),例如 `size=500` 实际按 20 处理。

| 模块 | 接口名称 | 方法 | 路径 | 请求参数 | 响应数据 | 主要错误 | 功能说明 | 认证/权限 |
|---|---|---|---|---|---|---|---|---|
| user-04-messages-support | 工单分类列表 | GET | `/api/support/categories` | 无 | 200:`categories` array,元素:`id` int64、`name` string | 错误:401;502 读取工单分类失败 | 提交工单时可选的分类。 | 需登录 |
| user-04-messages-support | 工单列表 | GET | `/api/support/tickets` | 查询参数:<br>`status` string 可选,枚举 `none`/`solved`/`closed`,空为全部<br>`offset` int 默认 0<br>`size` int 默认 20(见分页约定) | 200:<br>`tickets` array,元素见「公共结构 SupportTicket」(不含 `body`)<br>`count` int64 符合条件的总数 | 错误:400 工单状态无效;401;502 读取工单失败 | 分页查询当前用户的工单。 | 需登录 |
| user-04-messages-support | 创建工单 | POST | `/api/support/tickets` | JSON 体:<br>`categoryId` int64 必填,>0<br>`subject` string 必填,去空格后 3-200 个字符<br>`body` string 必填,去空格后 1-10000 个字符<br>`csrfToken` string 必填 | 201:`id` int64 新工单 ID | 错误:400 请填写有效的工单分类、主题和内容;403 CSRF 校验失败;502 创建工单失败 | 提交新工单。 | 需登录 + CSRF 写保护 |
| user-04-messages-support | 工单详情 | GET | `/api/support/tickets/{ticketID}` | 路径参数:`ticketID` int64,>0 | 200:<br>`ticket` 见 SupportTicket(含 `body`)<br>`replies` array,元素见「公共结构 SupportReply」 | 错误:400 工单 ID 无效;401;404 工单不存在;502 读取工单详情失败 | 查看工单内容及全部回复记录。 | 需登录 |
| user-04-messages-support | 回复工单 | POST | `/api/support/tickets/{ticketID}/replies` | 路径参数:`ticketID` int64,>0<br>JSON 体:<br>`comment` string 必填,去空格后 1-10000 个字符<br>`csrfToken` string 必填 | 200:`ok` bool(true) | 错误:400 回复内容无效/工单 ID 无效;403 CSRF 校验失败;502 回复工单失败 | 用户追加回复。 | 需登录 + CSRF 写保护 |
| user-04-messages-support | 删除工单 | DELETE | `/api/support/tickets/{ticketID}` | 路径参数:`ticketID` int64,>0<br>JSON 体:`csrfToken` string 必填 | 200:`ok` bool(true) | 错误:400 工单 ID 无效;403 CSRF 校验失败;404 工单不存在或无权操作;409 工单已有回复,无法删除;502 删除工单失败 | 删除工单,已有回复的工单不允许删除。 | 需登录 + CSRF 写保护 |
| user-04-messages-support | 未读消息列表 | GET | `/api/messages` | 查询参数:`offset` int 默认 0;`size` int 默认 20(见分页约定) | 200:<br>`messages` array,元素见「公共结构 Message」<br>`count` int64 未读总数 | 错误:401;502 读取消息失败 | 分页获取未读站内消息。 | 需登录 |
| user-04-messages-support | 历史消息列表 | GET | `/api/messages/history` | 查询参数:`offset` int 默认 0;`size` int 默认 20(见分页约定) | 200:`messages` array(元素同 Message,为已读消息)、`count` int64 已读总数 | 错误:401;502 读取历史消息失败 | 分页获取已读历史消息。 | 需登录 |
| user-04-messages-support | 操作日志 | GET | `/api/audit-logs` | 查询参数:<br>`offset` int 默认 0;`size` int 默认 20(见分页约定)<br>`level` string 可选,枚举 `info`/`warn`/`error`<br>`from` / `to` string 起止日期(原样透传后端 RPC,格式待确认)<br>`q` string 关键字(原样透传) | 200:<br>`logs` array,元素:`id` int64、`level` string(≤16 字符)、`action` string(≤128)、`description` string(≤400)、`createdAt` int64(时间戳,单位待确认)<br>`count` int64 总数 | 错误:400 日志级别无效;401;502 读取操作记录失败 | 查询当前用户自己的操作历史,不接受用户/管理员选择参数,范围由后端限定;文本字段按字符数截断。 | 需登录 |
| user-04-messages-support | 未读消息数 | GET | `/api/messages/unread-count` | 无 | 200:`count` int64 | 错误:401;502 读取未读消息数量失败 | 返回未读消息数量,用于角标。 | 需登录 |
| user-04-messages-support | 标记消息已读 | POST | `/api/messages/read` | JSON 体:<br>`messageIds` int64[] 必填,去除非正数和重复后数量 1-100<br>`csrfToken` string 必填 | 200:`ok` bool(true) | 错误:400 消息列表无效;403 CSRF 校验失败;502 更新消息状态失败 | 将指定消息标记为已读。 | 需登录 + CSRF 写保护 |
| user-04-messages-support | 全部标记已读 | POST | `/api/messages/read-all` | JSON 体:`csrfToken` string 必填 | 200:`ok` bool(true) | 错误:403 CSRF 校验失败;502 更新消息状态失败 | 将当前用户全部未读消息标记为已读。 | 需登录 + CSRF 写保护 |


### 公共结构

### SupportTicket
- `id` int64
- `categoryId` int64
- `categoryName` string
- `subject` string
- `body` string(仅详情接口返回,省略于列表)
- `status` string(`none`/`solved`/`closed`)
- `createdAt` int64 创建时间(时间戳,单位待确认)
- `lastLogAt` int64 最后回复时间
- `latestReply` string 最近一条回复内容(可省略)

### SupportReply
- `id` int64
- `actorType` string:`admin`(管理员)/`user`(本人)/`system`(系统)
- `actorName` string:管理员为其姓名,无姓名时为「技术支持」;本人为「我」;系统为「系统」
- `comment` string
- `status` string 该条记录对应的工单状态
- `createdAt` int64

### Message
- `id` int64
- `type` string 消息类型
- `body` string
- `level` string 级别
- `createdAt` int64
- `role` string
- `isRead` bool
- `nodeCluster` `{id int64, name string}`(可省略)
- `node` `{id int64, name string}`(可省略)

<a id="m19"></a>

## 19. 用户端 · 计费与订单(用户端)

**模块约定**

> 路由总数 17,来源 EdgeUserPortal/internal/web/server.go
通用约定(适用于本文件全部接口,除特别说明):
- 全部路由外层都有 `licenseFeature(Finance)`:授权服务读取失败返回 503 `EdgeRoute 授权状态暂时无法读取，请稍后重试`;授权未包含财务功能返回 403 `当前 EdgeRoute 授权未包含该功能`。
- 带 `sensitiveUserWrite` 的写接口:未登录 401 `请先登录`;管理员协助会话 403 `管理员协助会话不能修改用户资源或安全配置`;授权已确认失效 403 `平台授权已失效，当前为只读模式…`。
- 写接口请求体均为 JSON(上限 64KB,`DisallowUnknownFields`,出现未知字段即 400 `请求格式无效`),且必须携带 `csrfToken`(string,必填,须与会话 CSRF 一致,否则 403 `CSRF 校验失败`)。
- 错误统一为 `{"error":"消息"}`。

| 模块 | 接口名称 | 方法 | 路径 | 请求参数 | 响应数据 | 主要错误 | 功能说明 | 认证/权限 |
|---|---|---|---|---|---|---|---|---|
| user-05-billing | 计费总览 | GET | `/api/billing` | 无 | 200:<br>`balanceReminderEligible` bool 是否需要余额提醒<br>`balanceReminderLimit` 余额提醒阈值(来自 userconfigs.BalanceReminderLimit,类型待确认)<br>`account` {`id`,`total`,`totalFrozen`}<br>`financeEnabled` / `plansEnabled` / `plansPurchasable` / `requirePlan` / `showPrices` bool<br>`plans` []Plan<br>`purchases` []UserPlan<br>`logs` []AccountLog<br>`methods` []OrderMethod<br>`orders` []UserOrder(最多 100)<br>`refunds` []UserRefund(最多 100)<br>`bills` []UserBill(最多 100)<br>`trafficPackagesEnabled` / `trafficPackagesPurchasable` bool<br>`trafficPackages` []TrafficPackage<br>`trafficPackagePeriods` []{`id`,`count`,`unit`,`months`}<br>`trafficPackagePrices` []{`trafficPackageId`,`nodeRegionId`,`trafficPackagePeriodId`,`price`}(仅 price>0)<br>`nodeRegions` []{`id`,`name`}<br>`userTrafficPackages` []UserTrafficPackage | 错误:502(`读取计费设置失败`、`读取已购套餐失败`、`读取支付订单失败` 等分项消息) | 并发聚合读取用户账户、套餐、订单、退款、账单、流量包等;各 RPC 最多重试 3 次,整体超时 15s。`showPrices=false` 时套餐不返回价格,且不返回流量包价格;套餐/支付方式/流量包列表按计费开关决定是否加载。 | 需登录;需财务授权(GET 无 CSRF) |
| user-05-billing | 直接购买套餐(余额) | POST | `/api/plans/{planID}/purchase` | 路径:`planID` int64 >0<br>JSON:`csrfToken`<br>`period` string 必填,`monthly`/`seasonally`/`yearly`(忽略大小写与首尾空格)<br>`countPeriod` int32,≤0 时按 1,最大 120 | 201:`{"id": 已购套餐ID(int64)}` | 错误:400(套餐 ID 无效/周期无效/购买周期不能超过 120)、403、409/404/502(见公共结构“计费 RPC 错误映射”,如 `账户余额不足`、`套餐当前不可购买`) | 使用账户余额直接购买套餐,无需支付网关。 | 需登录+CSRF写保护;需财务授权 |
| user-05-billing | 创建套餐支付订单 | POST | `/api/plans/{planID}/order` | 路径:`planID` int64 >0<br>JSON:`csrfToken`<br>`period` string 必填,`monthly`/`seasonally`/`yearly`<br>`countPeriod` int32 必填,1~120(此处不会默认为 1)<br>`methodCode` string,可空;非空时转小写,≤32 字符<br>`couponCode` string,可空,会 trim<br>请求头:`Idempotency-Key` 可选,16~128 字符,仅 `a-zA-Z0-9-_.:` | 201:CreatedBillingOrder | 错误:400(`请选择有效的套餐和周期`、`订单请求标识无效`)、409(`支付方式不可用`、`在线支付暂不可用，请联系管理员`、优惠码类错误、`同一订单请求标识不能用于不同购买内容` 等)、其余见映射 | 创建套餐购买订单(可带优惠码),支持幂等。`methodCode` 为空时不生成回调地址,是否走余额由后端 RPC 决定(待确认:空 methodCode 的具体语义)。 | 需登录+CSRF写保护;需财务授权 |
| user-05-billing | 校验套餐优惠码 | POST | `/api/plans/{planID}/coupon` | 路径:`planID` int64 >0<br>JSON:`csrfToken`<br>`code` string,优惠码(trim)<br>`period` string 必填 `monthly`/`seasonally`/`yearly`<br>`countPeriod` int32,≤0 按 1,最大 120 | 200:`{"discountCents": int64 优惠金额(分), "amountCents": int64 实付金额(分)}` | 错误:400(`套餐 ID 无效`、`当前套餐不支持优惠码`——套餐不存在或该周期价格≤0)、409(`请输入优惠码`、`优惠码无效或已停用`、`优惠码已过期`、`优惠码校验次数过多，请 15 分钟后重试` 等)、其余见映射 | 按套餐周期价×周期数计算基础金额并校验优惠码,返回优惠结果。优惠码校验有频率限制(15 分钟)。 | 需登录+CSRF写保护;需财务授权 |
| user-05-billing | 创建流量包订单 | POST | `/api/traffic-packages/{packageID}/order` | 路径:`packageID` int64 >0<br>JSON:`csrfToken`<br>`methodCode` string 必填,≤32,转小写<br>`regionId` int64 必填 >0(节点区域)<br>`periodId` int64 必填 >0(流量包周期)<br>`count` int32 必填,1~20 | 201:CreatedBillingOrder | 错误:400(`请选择有效的流量包、区域、有效期和支付方式`、`流量包订单参数无效`)、409(`流量包当前不可购买`、`流量包自助购买当前未开放`、`支付方式不可用`、`在线支付暂不可用，请联系管理员`)、其余见映射 | 为指定区域、周期、数量的流量包创建支付订单。 | 需登录+CSRF写保护;需财务授权 |
| user-05-billing | 续费已购套餐(余额) | POST | `/api/user-plans/{userPlanID}/renew` | 路径:`userPlanID` int64 >0<br>JSON:`csrfToken`<br>`period` string 必填 `monthly`/`seasonally`/`yearly`<br>`countPeriod` int32,≤0 按 1,最大 120 | 200:`{"ok": true}` | 错误:400(`已购套餐 ID 无效`、周期无效)、403、409(`账户余额不足` 等)、其余见映射 | 使用账户余额续费已购套餐。 | 需登录+CSRF写保护;需财务授权 |
| user-05-billing | 设置套餐自动续费 | PUT | `/api/user-plans/{userPlanID}/auto-renew` | 路径:`userPlanID` int64 >0<br>JSON:`csrfToken`<br>`enabled` bool<br>`maxAmount` float64,单次自动续费金额上限,经 `PlanAutoRenew.Validate()` 校验(具体范围待确认:校验在 EdgeCommon userconfigs 中) | 200:`{"ok": true}` | 错误:400(`套餐无效`、校验错误文本、`无法保存自动续费设置，请检查套餐归属、状态和金额上限`)、403、503(`当前 API 不支持套餐自动续费`) | 开启/关闭已购套餐的自动续费并设置金额上限。注意:该处理函数先校验登录再解码 JSON。 | 需登录+CSRF写保护;需财务授权 |
| user-05-billing | 创建套餐续费订单 | POST | `/api/user-plans/{userPlanID}/order` | 路径:`userPlanID` int64 >0<br>JSON/请求头同“创建套餐支付订单”:`csrfToken`、`period`、`countPeriod`(1~120)、`methodCode`、`couponCode`、`Idempotency-Key` | 201:CreatedBillingOrder<br>错误同“创建套餐支付订单” | 见模块约定 | 为已购套餐创建续费支付订单(订单类型 `plan_renewal`),支持优惠码与幂等。 | 需登录+CSRF写保护;需财务授权 |
| user-05-billing | 校验续费优惠码 | POST | `/api/user-plans/{userPlanID}/coupon` | 路径:`userPlanID` int64 >0<br>JSON:`csrfToken`、`code`、`period`、`countPeriod`(同“校验套餐优惠码”) | 200:`{"discountCents","amountCents"}` | 错误:400(`套餐 ID 无效`、`当前套餐不支持优惠码`)、404(`已购套餐不存在`)、409(优惠码类错误)、其余见映射 | 先由已购套餐 ID 反查所属套餐,再按其周期价校验优惠码。 | 需登录+CSRF写保护;需财务授权 |
| user-05-billing | 创建充值订单 | POST | `/api/orders` | JSON:`csrfToken`<br>`methodCode` string 必填,≤32,转小写<br>`amount` float64 必填,0.01~1,000,000,最多 2 位小数 | 201:CreatedBillingOrder | 错误:400(`请选择支付方式并输入有效充值金额`、`充值金额无效`)、409(`支付方式不可用`、`读取支付方式失败`、`在线支付暂不可用，请联系管理员`、`支付方式当前不可用`)、其余见映射 | 创建账户余额充值订单,返回支付跳转信息;对 alipay/wechat/custom/epay/bepusdt 会自动拼接回调地址 `/api/pay/{provider}/{methodCode}/notify`。 | 需登录+CSRF写保护;需财务授权 |
| user-05-billing | 查询订单详情 | GET | `/api/orders/{orderCode}` | 路径:`orderCode` string,trim 后非空且 ≤64 | 200:`{"order": UserOrder}` | 错误:400(`订单号无效`)、404(`订单不存在`)、502(`读取订单失败`) | 按订单号查询当前用户订单。 | 需登录;需财务授权 |
| user-05-billing | 取消订单 | POST | `/api/orders/{orderCode}/cancel` | 路径:`orderCode` string ≤64<br>JSON:`csrfToken` | 200:`{"ok": true}` | 错误:400(`订单号无效`)、404(`订单不存在`)、409(`已完成订单不能取消`)、其余见映射 | 取消未完成的订单。 | 需登录+CSRF写保护;需财务授权 |
| user-05-billing | 申请订单退款 | POST | `/api/orders/{orderCode}/refunds` | 路径:`orderCode` string ≤64<br>JSON:`csrfToken`<br>`amount` float64 必填,0.01~1,000,000,最多 2 位小数<br>`reason` string 必填,trim 后 3~500 字符(按 rune) | 201:`{"code": 退款单号}` | 错误:400(`请输入有效的退款金额和退款原因`)、其余见映射(可退金额等业务限制由 RPC 判定,不在本层暴露原文,多为 502 `计费操作失败`) | 对已支付订单发起退款申请。 | 需登录+CSRF写保护;需财务授权 |
| user-05-billing | 取消退款申请 | POST | `/api/refunds/{refundCode}/cancel` | 路径:`refundCode` string ≤64<br>JSON:`csrfToken` | 200:`{"ok": true}` | 错误:400(`退款单号无效`)、其余见映射 | 撤销尚未处理的退款申请。 | 需登录+CSRF写保护;需财务授权 |
| user-05-billing | 支付用量账单 | POST | `/api/bills/{billID}/pay` | 路径:`billID` int64 >0<br>JSON:`csrfToken` | 200:`{"ok": true}` | 错误:400(`账单 ID 无效`)、409(`账单当前无法支付`、`账户余额不足`)、其余见映射 | 使用账户余额支付一笔用量账单。 | 需登录+CSRF写保护;需财务授权 |
| user-05-billing | 支付回调通知(POST) | POST | `/api/pay/{provider}/{methodCode}/notify` | 路径:`provider` 取值 `alipay`/`custom`/`epay`/`wechat`/`bepusdt`;`methodCode` ≤32,转小写<br>查询:`ack`(仅 bepusdt)`success`/`ok`,缺省 `success`<br>请求体:支付网关原始表单/报文,非空,≤64KB;仅转发 `Wechatpay-*`、`Content-Type`、`Request-Id` 请求头用于验签 | 成功 200,纯文本 `success` 或 `ok`(bepusdt 按 `ack`) | 失败:纯文本 `failure`,状态码 400(参数/验签失败/报文为空)、405(bepusdt 非 POST)、429(按 IP+methodCode 限流)<br>另:授权层错误为 JSON 503/403 | 接收第三方支付异步通知并交由 API 验签入账。响应头 `Cache-Control: no-store`;不使用统一 JSON 错误格式。 | 公共回调(不要求登录/CSRF,依赖网关签名验证);仍需财务授权 |
| user-05-billing | 支付回调通知(GET) | GET | `/api/pay/{provider}/{methodCode}/notify` | 路径同上<br>查询:整串 `RawQuery` 作为通知报文(非空,≤64KB,超限 413);`provider=bepusdt` 时不允许 GET | 成功 200,纯文本 `success`/`ok` | 失败:纯文本 `failure`,400、405(bepusdt)、413、429 | 部分网关(如易支付、支付宝同步/异步)以 GET 回调,处理逻辑与 POST 相同(同一 handler `apiPaymentNotify`)。 | 公共回调(不要求登录/CSRF);仍需财务授权 |


### 公共结构

CreatedBillingOrder(创建订单类接口的 201 响应):
- `code` string 订单号
- `payURL` string 支付跳转地址
- `paymentAmount` string,可选,实付金额(含币种场景)
- `paymentCurrency` string,可选
- `paymentNetwork` string,可选

计费 RPC 错误映射(`writeBillingRPCError`,下游多数接口共用):
- 409:已知用户可见规则文案(`请输入优惠码`、`优惠码无效或已停用`、`优惠码尚未生效`、`优惠码已过期`、`优惠码不适用于当前套餐`、`订单金额未达到优惠码使用门槛`、`该优惠码仅限首次购买套餐的用户使用`、`优惠码使用次数已用完`、`你已达到该优惠码的使用次数上限`、`优惠码状态已变化，请重新提交订单`、`原订单已关闭，请重新发起购买`、`同一订单请求标识不能用于不同购买内容`、`原订单支付方式暂不可用，请到订单列表查看`、`订单请求标识无效`、`优惠码校验次数过多，请 15 分钟后重试`)
- 409:`账户余额不足`、`账单当前无法支付`、`套餐当前不可购买`、`该套餐已达到你的购买数量上限`、`计费服务当前已停用`、`套餐自助购买当前未开放`、`流量包自助购买当前未开放`、`流量包当前不可购买`、`支付方式当前不可用`、`已完成订单不能取消`
- 400:`流量包订单参数无效`、`充值金额无效`
- 404:`订单不存在`
- 403:`没有权限操作此套餐`
- 502:其他一律 `计费操作失败`

UserOrder:`code`,`type`,`status`,`amount`(元,float),`createdAt`(int64 时间戳),`cancelledAt`,`finishedAt`(可选),`expired` bool,`canPay` bool,`payURL`,`paymentAmount`,`paymentCurrency`,`paymentNetwork`(可选),`paymentMethod`(OrderMethod,可选),`refundedAmount`,`pendingRefundAmount`,`refundableAmount`(元),`canRefund` bool。

OrderMethod:`id`,`name`,`code`,`description?`,`url?`,`parentCode`,`clientType?`,`qrcodeTitle?`,`callbackAck?`。

UserRefund:`code`,`orderCode`,`status`,`amount`,`reason`,`providerReference?`,`adminNote?`,`requestedAt`,`processedAt?`,`paymentMethod?`(OrderMethod)。

UserBill:`id`,`code`,`type`,`typeName`,`description`,`amount`,`month`,`isPaid`,`isOverdue`,`paidAt?`,`canPay`,`dayFrom`,`dayTo`。

AccountLog:`id`,`delta`,`total`,`eventType`,`description`,`createdAt`。

Plan:`id`,`name`,`description?`,`priceType`,`monthlyPrice`,`seasonallyPrice`,`yearlyPrice`,`trafficPrice?`,`bandwidthPrice?`(价格字段仅 `showPrices=true` 返回),`totalServers`,`totalServerNamesPerServer`,`totalServerNames`,`monthlyRequests`,`monthlyWebsocketConnections`,`trafficLimit?`{`isOn`,`monthlySize?`,`totalSize?`(均为 {`count`,`unit`})},`bandwidthLimitPerNode?`,`maxUploadSize?`({`count`,`unit`}),`maxUserPurchases`。

UserPlan:`id`,`name`,`dayTo`,`isOn`,`expired`,`autoRenew?`(PlanAutoRenew,含 `maxAmount` 等,完整字段待确认),`dailyTrafficBytes`,`monthlyTrafficBytes`,`monthlyProcessedTrafficBytes`,`monthlyPendingTrafficBytes`,`plan?`(Plan),`servers`[]{`id`,`name`}。

TrafficPackage:`id`,`size`,`unit`,`bytes`。

UserTrafficPackage:`id`,`totalBytes`,`usedBytes`,`dayFrom`,`dayTo`,`createdAt`,`expired`,`depleted`,`package?`(TrafficPackage),`nodeRegion?`{`id`,`name`},`periodCount`,`periodUnit`,`availableBytes`。

<a id="m20"></a>

## 20. 用户端 · 站点管理(用户端)

**模块约定**

> 路由总数 27,来源 EdgeUserPortal/internal/web/server.go
说明:认证列中"需登录+CSRF写保护"= 经 `sensitiveUserWrite`(未登录 401"请先登录";管理员协助会话 403"管理员协助会话不能修改用户资源或安全配置";授权只读 403;并记录操作日志),且 JSON 请求体必须携带 `csrfToken`(校验失败 403"CSRF 校验失败")。"需登录"= 处理函数内调用 `requireAPIUser`。错误统一为 `{"error":"消息"}`。请求体统一经 `decodeAPIJSON`(≤64KB、禁止未知字段、仅一个 JSON 对象,失败 400"请求格式无效"),另有说明除外。

| 模块 | 接口名称 | 方法 | 路径 | 请求参数 | 响应数据 | 主要错误 | 功能说明 | 认证/权限 |
|---|---|---|---|---|---|---|---|---|
| user-06-sites | 站点列表 | GET | `/api/sites` | 无 | 200 `{"sites":[ApiSite]}`(见公共结构) | 错误:502"读取站点失败" | 列出当前用户名下的 HTTP 类站点(`httpProxy`/`httpWeb`,不含 TCP/UDP 与 Pages)。 | 需登录 |
| user-06-sites | 站点接入预检 | POST | `/api/sites/preflight` | JSON 请求体:<br>`csrfToken` string 必填<br>`domains` string[] 必填,1-20 个,小写化,仅允许 `*.` 前缀通配符,平台保留域名 400<br>`origins` object[] 必填,1-20 个:`protocol` string,`host` string,`port` int | 200 `{"ok":bool,"checks":[{"target":string,"status":"ok/warning/critical","message":string,"addresses":string[]?,"statusCode":int?,"latencyMs":int?}],"failed":int}` | 错误:400 | 对源站做连通性检查(HEAD 请求,超时 12s)。源站为内网/本机 IP 返回 `critical`,`s3` 协议仅给 `warning`。不创建任何资源。 | 需登录+CSRF写保护 |
| user-06-sites | 创建站点 | POST | `/api/sites` | 请求头 `Idempotency-Key`(可选,≤128 字符,字符集 `[A-Za-z0-9-_.:]`)<br>JSON 请求体:<br>`csrfToken` string 必填<br>`domains` string[] 必填,1-20<br>`origins` object[] 必填,1-20:`protocol` `http`/`https`/`s3`,`name` string,`host` string,`port` int(默认 80/443,1-65535),`weight` int(默认 10,1-1000),`s3` object(`endpoint`,`region`,`bucket`,`accessKeyId`,`accessKeySecret`,`addressStyle`,仅 `s3` 协议使用)<br>`enableWebsocket` bool<br>`userPlanId` int ≥0(0=不指定套餐) | 201 `{"id":string}`(站点公共引用)<br>幂等重放:200 + 响应头 `Idempotent-Replay: true` + 同样 `{"id"}` | 错误:400 参数无效;403(域名 TXT 未验证/不允许接入/需先开通套餐/套餐不可用/未实名,未实名 body 为 `{"error":"请先完成实名认证","code":"identity_required"}`);409(站点数或域名数超额/域名被占用/无可用集群/同一 Idempotency-Key 参数不一致);502 | 创建 HTTP 站点并绑定域名与源站。带 `Idempotency-Key` 时结果在内存中缓存 24 小时,相同参数重复提交返回原 `id`。 | 需登录+CSRF写保护+需实名(`requireIdentity("sites")`,仅当系统启用 `RequireIdentity` 时生效) |
| user-06-sites | 站点详情 | GET | `/api/sites/{siteID}` | 路径参数:`siteID` string,站点公共引用(HMAC 派生字符串,非数字 ID) | 200 `{"site":ApiSite}` | 错误:400"资源编号无效";404"站点不存在";502 | 读取单个站点的基础信息。 | 需登录 |
| user-06-sites | 站点同步状态 | GET | `/api/sites/{siteID}/sync-status` | 路径参数:`siteID` | 200 `{"status":?,"pendingTasks":?,"failedTasks":?,"updatedAt":?,"error":?}`,各字段类型待确认:状态对象来自 RPC pb 对象,未逐字段核实 | 错误:404;502 | 查询站点配置下发到边缘节点的同步状态(待处理/失败任务数)。 | 需登录 |
| user-06-sites | 修改站点 | PATCH | `/api/sites/{siteID}` | 路径参数:`siteID`<br>JSON 请求体:`csrfToken` 必填,另外以下三个字段必须且只能提供一个:<br>`name` string,非空,≤100 字符<br>`isOn` bool<br>`serverGroupIds` int64[],≤100 个,每项 >0,不可重复 | 200 `{"ok":true}` | 错误:400"必须且只能修改一个站点字段"/校验失败;404;502 | 单次只修改一个站点字段:名称、启停或所属站点分组。 | 需登录+CSRF写保护 |
| user-06-sites | 删除站点 | DELETE | `/api/sites/{siteID}` | 路径参数:`siteID`<br>JSON 请求体:`csrfToken` 必填(无其他字段) | 200 `{"ok":true}` | 错误:404;502 | 删除站点。DELETE 请求同样需要在请求体中携带 CSRF。 | 需登录+CSRF写保护 |
| user-06-sites | 站点协议与证书设置 | GET | `/api/sites/{siteID}/protocols` | 路径参数:`siteID` | 200 `{"httpEnabled":bool,"httpsEnabled":bool,"http2Enabled":bool,"http3Enabled":bool,"alwaysUseHTTPS":bool,"certificateIds":int64[],"serverNames":string[],"filterable":bool,"certificates":[ApiSiteCertificate]}`(见公共结构,按 `timeEndAt` 升序) | 错误:404;502 | 读取站点的 HTTP/HTTPS/HTTP2/HTTP3 开关、已绑证书及用户可选证书列表。 | 需登录 |
| user-06-sites | 修改站点协议与证书 | PUT | `/api/sites/{siteID}/protocols` | 路径参数:`siteID`<br>JSON 请求体:`csrfToken` 必填<br>`httpEnabled` bool<br>`httpsEnabled` bool<br>`http2Enabled` bool<br>`http3Enabled` bool<br>`alwaysUseHTTPS` bool<br>`certificateIds` int64[],≤20 | 200(成功体待确认:未核实是否为 `{"ok":true}`) | 错误:400;403"当前套餐未开通…功能";404;502 | 整体保存协议设置。规则:`http3Enabled` 需同时开启 `httpsEnabled`+`http2Enabled`,且套餐含 `ServerHTTP3`;`alwaysUseHTTPS` 需同时开启 `httpEnabled`+`httpsEnabled`,且套餐含 `ServerHTTPSRedirect`;证书须启用、非 CA、未过期且属于当前用户;开启 HTTPS 至少 1 张证书。 | 需登录+CSRF写保护 |
| user-06-sites | 站点流量统计 | GET | `/api/sites/{siteID}/statistics` | 路径参数:`siteID`<br>查询参数:`from`+`to`(`YYYY-MM-DD` 或 `YYYYMMDD`,必须同时出现)或 `days`(1-100,默认 30);范围 ≤100 天 | 200 `{"currentDay":string,"accounting":object,"traffic":{"days":int,"stats":[{"day","bytes","cachedBytes","countRequests","countCachedRequests","countAttackRequests","attackBytes","uncachedBytes","countIPs"}]},"bandwidth":{"days":int,"stats":[{"day","timeAt"?,"bytes","bits"}],"nthStat"?:object,"aggregation"?:string},"waf":{"countDailyLog","countDailyBlock","countDailyCaptcha","countWeklyBlock","countMonthlyBlock":int,"logDailyStats","blockDailyStats","captchaDailyStats":[{"day","count"}]}}`<br>`accounting` 来自 `trafficstats.Describe`,内部字段待确认 | 错误:400"统计日期范围无效";分项读取失败 502 | 返回站点流量、带宽与 WAF 的按日统计。注意 WAF 周统计字段名拼写为 `countWeklyBlock`(源码原样)。 | 需登录 |
| user-06-sites | 安全态势分钟统计 | GET | `/api/sites/{siteID}/waf-minute-stats` | 路径参数:`siteID`<br>查询参数:`minutes` int,5-120,默认 60 | 200 `{"minutes":int,"stats":[{"minute","bytes","cachedBytes","countRequests","countCachedRequests"}]}`,响应头 `Cache-Control: no-store` | 错误:400;403"当前套餐未开通安全态势统计功能"(功能 `ServerViewAccessLog`);502 | 返回最近若干分钟的请求/流量分钟级统计。 | 需登录;套餐需含 `ServerViewAccessLog` |
| user-06-sites | 站点访问分析 | GET | `/api/sites/{siteID}/analysis` | 路径参数:`siteID`<br>查询参数:`range`(`10m`/`30m`/`1h`(默认)/`7d`/`14d`/`30d`/`31d`)或 `from`+`to`(`YYYY-MM-DD`,≤100 天,`to` 不得晚于当前时间) | 200 `{"from","to":RFC3339 UTC,"generatedAt":string,"sampled":bool,"totalRequests":int,"totalBytes":int,"wafRequests":int,"topDomains","topURLs","topClientIPs","topReferers","topPorts","topProtocols","attackDomains","attackURLs":[{"label":string,"requests":int,"bytes":int,"percentage":number}]}`(各 Top 10) | 错误:400"统计时间范围无效";403 套餐功能;502 | 基于访问日志采样(每天最多 1000 条)做 Top 维度分析。 | 需登录;套餐需含 `ServerViewAccessLog` |
| user-06-sites | 安全事件 | GET | `/api/sites/{siteID}/security-events` | 路径参数:`siteID`<br>查询参数:同 `analysis`(`range` 或 `from`+`to`) | 200 `{"from","to","sampled","blocked":int,"challenges":int,"observed":int,"events":[SecurityEvent],"categories","regions","countries","sources":[{"name":string,"count":int,"countryCode"?:string}]}`<br>`SecurityEvent`:`id`,`siteId?`,`siteName?`,`timestamp`,`host`,`path`,`method`,`status`,`action`,`clientIP`(已掩码),`category`,`region?`,`countryCode?`,`edgeTarget?`(结构 `securityEventEdgeTarget` 未读全,待确认),`events` 最多 100 条 | 错误:400;403 套餐功能;502 | 返回采样窗口内的 WAF 拦截/验证/观察事件及分类聚合。 | 需登录;套餐需含 `ServerViewAccessLog` |
| user-06-sites | 请求记录 | GET | `/api/sites/{siteID}/access-logs` | 路径参数:`siteID`<br>查询参数:窗口 `range` 或 `from`+`to`(同 `analysis`),或 `fromAt`+`toAt`(RFC3339,≤100 天,否则 400"请求记录时间范围无效")<br>`filter` string,可空,取值 `url`/`domain`/`ip`/`referer`/`port`/`protocol`<br>`value` string,≤2048,`filter` 非空时必填<br>`waf` string,仅 `1` 有效(仅看 WAF 命中) | 200 `{"from","to","sampled":bool,"items":[{"timestamp","host","path","clientIP","method","status","bytes","durationMS","protocol","port","action"}]}`,最多 200 条,响应头 `Cache-Control: no-store` | 错误:400"请求记录筛选条件无效";403 套餐功能;502 | 查询站点近期访问日志(采样),支持按维度筛选。 | 需登录;套餐需含 `ServerViewAccessLog` |
| user-06-sites | 站点 WAF 配置读取 | GET | `/api/sites/{siteID}/waf` | 路径参数:`siteID` | 200 `{"site":ApiSite,"waf":{"isPrior":bool,"isOn":bool,"ignoreGlobalRules":bool,"defaultCaptcha":string,"forceCaptcha":bool,"mode":string,"maxRequestBodySize":int,"groups":[{"code","name","description","direction":string,"isOn":bool}]}}` | 错误:404;经 `writeWAFRPCError` 映射的错误码待确认(该函数未读全);502 | 读取站点 Web 防火墙配置及可选规则组开关。 | 需登录 |
| user-06-sites | 站点 WAF 配置修改 | PUT | `/api/sites/{siteID}/waf` | 路径参数:`siteID`<br>JSON 请求体:`csrfToken` 必填,及上面 `waf` 对象各字段平铺:`isPrior`,`isOn`,`ignoreGlobalRules`,`defaultCaptcha`(`none`/`default`),`forceCaptcha`(需 `isOn`),`mode`(`defend`/`observe`/`bypass`),`maxRequestBodySize`(0=默认值,范围 64KiB-16MiB),`groups`[{`code`,`isOn`}] | 200(成功体待确认,未核实) | 错误:400(含未知规则组);404;502 | 保存站点 WAF 配置。 | 需登录+CSRF写保护 |
| user-06-sites | 站点访问控制读取 | GET | `/api/sites/{siteID}/access` | 路径参数:`siteID` | 200 `{"site":ApiSite,"access":{"allowIPs":string[],"denyIPs":string[],"allowCountryIds":int64[],"denyCountryIds":int64[]},"countries":[{"id":int64,"name":string,"code":string}]}` | 错误:404;502 | 读取 IP 与地区黑白名单及可选国家列表。 | 需登录 |
| user-06-sites | 站点访问控制修改 | PUT | `/api/sites/{siteID}/access` | 路径参数:`siteID`<br>JSON 请求体:`csrfToken` 必填<br>`allowIPs` string[],≤500,每项为单 IP 或 CIDR<br>`denyIPs` string[],≤500,同上<br>`allowCountryIds` int64[],≤300,须在可选国家列表内<br>`denyCountryIds` int64[],≤300,同上 | 200(成功体待确认,未核实) | 错误:400(允许与禁止名单重叠/格式无效);404;502 | 保存 IP/地区黑白名单。 | 需登录+CSRF写保护 |
| user-06-sites | 站点访问验证(UAM)读取 | GET | `/api/sites/{siteID}/uam` | 路径参数:`siteID` | 200 `{"site":ApiSite,"uam":{"isPrior":bool,"isOn":bool,"addToWhiteList":bool,"minQPSPerIP":int,"keyLife":int,"challengeDifficulty":int,"challengeTimeout":int,"onlyURLPatterns":[UrlPattern],"exceptURLPatterns":[UrlPattern]}}`(`UrlPattern` 见公共结构) | 错误:403"当前套餐未开通访问验证功能";404;502 | 读取 5 秒盾/访问验证配置。 | 需登录;套餐需含 `ServerUAM` |
| user-06-sites | 站点访问验证(UAM)修改 | PUT | `/api/sites/{siteID}/uam` | 路径参数:`siteID`<br>JSON 请求体:`csrfToken` 必填,及 `uam` 各字段平铺:`isPrior`,`isOn`,`addToWhiteList`,`minQPSPerIP`(0-999999),`keyLife`(0-3600),`challengeDifficulty`(14-22,0=默认 18),`challengeTimeout`(60-180,0=默认 120),`onlyURLPatterns`/`exceptURLPatterns`(各 ≤100 条) | 200(成功体待确认,未核实) | 错误:400;403 套餐功能;404;502 | 保存访问验证配置。 | 需登录+CSRF写保护;套餐需含 `ServerUAM` |
| user-06-sites | 站点 CC 防护读取 | GET | `/api/sites/{siteID}/cc` | 路径参数:`siteID` | 200 `{"site":ApiSite,"cc":{"isPrior":bool,"isOn":bool,"thresholds":[Threshold],"onlyURLPatterns":[UrlPattern],"exceptURLPatterns":[UrlPattern]}}`<br>`Threshold`:`path?`,`conditions?`,`groupBy?`,`priority?`,`isOn`,`name`,`periodSeconds`,`maxRequests`,`blockSeconds`,`statusCode`,`perURI`,`methods` | 错误:403"当前套餐未开通CC 防护功能";404;502 | 读取 CC 防护阈值规则。 | 需登录;套餐需含 `ServerCC` |
| user-06-sites | 站点 CC 防护修改 | PUT | `/api/sites/{siteID}/cc` | 路径参数:`siteID`<br>JSON 请求体:`csrfToken` 必填,及 `cc` 各字段平铺:`isPrior`,`isOn`,`thresholds`(1-10 条;`name` ≤100,`methods` ≤16 且合法),`onlyURLPatterns`/`exceptURLPatterns`(各 ≤100)<br>`isOn` 为真时至少需一条启用的阈值 | 200(成功体待确认,未核实) | 错误:400;403 套餐功能;404;502 | 保存 CC 防护配置。阈值各字段的数值范围校验待确认(未全部核实)。 | 需登录+CSRF写保护;套餐需含 `ServerCC` |
| user-06-sites | 站点缓存设置读取 | GET | `/api/sites/{siteID}/cache` | 路径参数:`siteID` | 200 `{"site":ApiSite,"cache":{"isPrior":bool,"isOn":bool,"addStatusHeader":bool,"addAgeHeader":bool,"enableCacheControlMaxAge":bool,"useDefaultPolicy":bool,"purgeIsOn":bool,"purgeKeyConfigured":bool,"rules":[CacheRule]}}`(`CacheRule` 见公共结构) | 错误:404;502 | 读取站点缓存策略及规则。 | 需登录 |
| user-06-sites | 站点缓存设置修改 | PUT | `/api/sites/{siteID}/cache` | 路径参数:`siteID`<br>JSON 请求体:`csrfToken` 必填,及 `cache` 各字段平铺(`purgeKeyConfigured` 虽在结构体中,但写入时被忽略)<br>`rules` ≤50 条:`sourceIndex` ≥0 表示修改已有规则、<0 表示新增,不能修改高级规则<br>`matchType` 取值 `extension`/`all`/`index`/`prefix`/`exact`/`wildcard`/`regexp`,`prefix`/`exact`/`wildcard`/`regexp` 的 `values` 须恰好 1 个且 ≤1000 字符<br>`life.unit` 取 `minute`/`hour`/`day`/`week`,`count` 1-525600(`isReverse` 时无需)<br>`minSize`/`maxSize` 单位 `byte`/`kb`/`mb`/`gb`,`count` 0-1,000,000,min 不得大于 max<br>`status` 状态码 100-599 | 200(成功体待确认,未核实) | 错误:400;404;502 | 保存站点缓存规则。`applyAPICacheRule` 后半段校验未读全,其余校验待确认。 | 需登录+CSRF写保护 |
| user-06-sites | 创建缓存刷新/预热任务 | POST | `/api/sites/{siteID}/cache/tasks` | 路径参数:`siteID`<br>JSON 请求体(上限 256KB):<br>`csrfToken` 必填<br>`type` `purge`/`fetch`<br>`keyType` `key`/`prefix`(`fetch` 只能 `key`)<br>`keys` string[],1-1000 个,每项 ≤2048,须为 http/https 完整 URL 且不含 userinfo | 201 `{"id":int64,"count":int}` | 错误:400;404"站点不存在";422"没有匹配到当前账户的有效站点 URL"(`count==0`);502 | 为站点提交缓存清除(`purge`)或预热(`fetch`)任务,查询见缓存任务模块。 | 需登录+CSRF写保护 |
| user-06-sites | HTTP 反向代理读取 | GET | `/api/sites/{siteID}/http-reverse-proxy` | 路径参数:`siteID` | 200 `{"site":ApiSite,"reverseProxy":{"isPrior","isOn":bool,"schedulingCode":string,"hashKey":string,"stickyType":string,"stickyParam":string,"primaryOrigins":[Origin],"backupOrigins":[Origin],"requestHostType":int,"requestHost":string,"requestHostExcludingPort":bool,"requestURI":string,"stripPrefix":string,"autoFlush":bool,"followRedirects":bool,"retry50X":bool,"retry40X":bool,"addHeaders":string[],"connTimeout","readTimeout","idleTimeout":{"count":int,"unit":string},"maxConns":int,"maxIdleConns":int}}`(`Origin` 见公共结构) | 错误:404;502 | 读取源站、调度、回源请求头与超时配置。 | 需登录 |
| user-06-sites | HTTP 反向代理修改 | PUT | `/api/sites/{siteID}/http-reverse-proxy` | 路径参数:`siteID`<br>JSON 请求体:`csrfToken` 必填,及 `reverseProxy` 各字段平铺<br>`primaryOrigins`/`backupOrigins` 各 ≤32,`isOn` 时至少 1 个主源站<br>`requestHostType` 0=代理服务器/1=源站/2=自定义,自定义时 `requestHost` 非空、≤253、不含空白<br>`requestURI`/`stripPrefix` ≤500,须以 `/` 开头或含 `${`<br>`maxConns`/`maxIdleConns` 0-1,000,000<br>超时单位 `ms`/`second`/`minute`/`hour`,不超过 7 天<br>`addHeaders` 仅允许 `X-Real-IP`/`X-Forwarded-For`/`X-Forwarded-By`/`X-Forwarded-Host`/`X-Forwarded-Proto`,为空时存哨兵 `__none__`<br>`schedulingCode` `random`/`roundRobin`/`leastConn`/`hash`(`hashKey` ≤500)/`sticky`(`stickyType` `cookie`/`header`/`argument`,`stickyParam` 匹配 `^[A-Za-z0-9_-]{1,50}$`)<br>源站 `protocol` 仅 `http`/`http4`/`http6`/`https`/`https4`/`https6`,端口支持范围,跨度 ≤1000 | 200 `{"ok":true}` | 错误:400;404;502(含配置未初始化) | 保存回源配置。 | 需登录+CSRF写保护 |


### 公共结构

### 站点引用 `{siteID}`
非数字 ID,是 HMAC 派生的公共引用字符串(`publicSiteReference`),经 `requireAPISite` → `resolveOwnedServerReference` 解析。只匹配当前用户的 HTTP 类站点(`httpProxy`/`httpWeb`)。空值 400"资源编号无效";无匹配 404"资源不存在"(部分接口为"站点不存在");读取失败 502。

### ApiSite
`id` string(公共引用);`name` string;`domain` string;`isOn` bool;`type` string;`description?` string;`dnsName?` string;`createdAt?` string;`nodeCluster?` `{id,name}`;`serverGroups?` `[{id,name}]`;`issues?` `[{label,target}]`。

### ApiSiteCertificate(协议接口的 `certificates[]`)
`id`,`isOn`,`name`,`description?`,`serverName?`,`isCA`,`isACME`,`timeBeginAt`,`timeEndAt`,`dnsNames`,`commonNames`,`ocspExpiresAt?`,`ocspError?`,`matchesSite`,`matchedNames`。

### UrlPattern(UAM/CC 的 `onlyURLPatterns`/`exceptURLPatterns`)
类型 `wildcard`/`regexp`/`images`/`audios`/`videos`;`wildcard` 与 `regexp` 的 `pattern` 非空且 ≤2000 字符。具体 JSON 字段名(如 `type`/`pattern`)待确认:未核对该结构的 json tag。

### CacheRule
`sourceIndex`,`isOn`,`isReverse`,`matchType`,`values`,`isCaseInsensitive`,`life{count,unit}`,`ignoreQuery`,`status[]`,`minSize{count,unit}`,`maxSize{count,unit}`,`skipCacheControlValues`,`skipSetCookie`,`enableRequestCachePragma`,`allowChunkedEncoding`,`allowPartialContent`,`forcePartialContent`,`alwaysForwardRangeRequest`,`enableIfNoneMatch`,`enableIfModifiedSince`,`enableReadingOriginAsync`。

### Origin(反向代理源站)
`id`,`isOn`,`name`,`description`,`protocol`,`host`,`portRange`,`weight`,`domains`,`requestHost`,`followPort`,`http2Enabled`,`readOnly`,`hasCertificate`。

<a id="m21"></a>

## 21. 用户端 · 缓存任务(用户端)

**模块约定**

> 路由总数 4,来源 EdgeUserPortal/internal/web/server.go
说明:错误统一为 `{"error":"消息"}`。"需登录+CSRF写保护"= 经 `sensitiveUserWrite`(未登录 401"请先登录";管理员协助会话 403;授权只读 403;记录操作日志),且 JSON 请求体须携带 `csrfToken`(失败 403"CSRF 校验失败")。创建缓存任务的接口位于站点模块(`POST /api/sites/{siteID}/cache/tasks`)。

| 模块 | 接口名称 | 方法 | 路径 | 请求参数 | 响应数据 | 主要错误 | 功能说明 | 认证/权限 |
|---|---|---|---|---|---|---|---|---|
| user-07-cache-tasks | 缓存任务列表 | GET | `/api/cache-tasks` | 查询参数:<br>`offset` int,≥0,默认 0<br>`type` string,可选,`purge`/`fetch`<br>`status` string,可选,`doing`/`success`/`failed`<br>`days` int,可选,取值 0/7/30/90<br>每页固定 20 条 | 200 `{"tasks":[CacheTask],"count":int}`(`CacheTask` 见公共结构) | 错误:400"任务分页参数无效"/"任务筛选参数无效";503(RPC 不支持);502"读取任务记录失败" | 分页查询当前用户的缓存刷新/预热任务记录。 | 需登录 |
| user-07-cache-tasks | 缓存任务额度 | GET | `/api/cache-task-quota` | 查询参数:`type` string 必填,`purge`/`fetch` | 200 `{"day":string,"maxKeysPerTask":int,"maxKeysPerDay":int,"usedKeys":int,"remainingKeys":int}`,响应头 `Cache-Control: no-store` | 错误:400;503"额度暂不可用,请联系管理员升级服务";502 | 查询当日该类型任务的单次与每日 URL 额度及已用量。 | 需登录 |
| user-07-cache-tasks | 缓存任务详情 | GET | `/api/cache-tasks/{taskID}` | 路径参数:`taskID` int64,须 >0 | 200 `{"task":CacheTask}` | 错误:400"任务编号无效";404"任务不存在";503"任务记录暂不可用" | 读取单个任务及其逐 URL 状态。与重试接口共用处理函数 `apiCacheTask`,按请求方法分支。 | 需登录 |
| user-07-cache-tasks | 重试缓存任务 | POST | `/api/cache-tasks/{taskID}/retry` | 路径参数:`taskID` int64,须 >0<br>JSON 请求体(上限 4KB):`csrfToken` string 必填 | 201 `{"id":int64,"count":int}`(新任务编号与 URL 数) | 错误:400"任务编号无效";404"任务不存在";409"仅已结束的失败任务可以重试"/无可重试 URL;502;503 | 仅已结束且失败的任务可重试。优先重提失败的 key,无失败 key 则重提全部;会新建任务,不修改原任务。执行顺序:先查任务(404),再解码 body 和校验 CSRF。 | 需登录+CSRF写保护 |


### 公共结构

### CacheTask
- `id` int64
- `sequence?` int(可选)
- `type` string:`purge`/`fetch`
- `state` string:`waiting`/`running`/`success`/`failed`
- `createdAt` string
- `doneAt` string
- `keys` 数组,元素 `{"url":string,"state":string(waiting/running/success/failed),"error?":string}`

`createdAt`/`doneAt` 的具体时间格式及未完成时的取值:待确认,未核实格式化函数。

<a id="m22"></a>

## 22. 用户端 · 证书 / DNS 服务商 / ACME 自动签发(用户端)

**模块约定**

> 路由总数 27,来源 EdgeUserPortal/internal/web/server.go
说明:
- 所有错误统一返回 `{"error":"消息"}`;未登录统一 401 `请先登录`。
- 标注"需登录+CSRF写保护"的接口 = 路由包了 `sensitiveUserWrite`(管理员协助会话返回 403 `管理员协助会话不能修改用户资源或安全配置`;平台授权失效返回 403 只读提示;未登录 401),且处理函数内再用 `validCSRF` 校验请求体 `csrfToken`(不匹配 403 `CSRF 校验失败`)。
- 请求体统一使用 `decodeAPIJSON`/`decodeAPIJSONLimit`:禁止未知字段、只允许单个 JSON 对象,默认体积上限 64KiB;格式错误/超限返回 400。
- "CSRF体" 指仅含 `csrfToken` 的 JSON 请求体,见"公共结构"。

| 模块 | 接口名称 | 方法 | 路径 | 请求参数 | 响应数据 | 主要错误 | 功能说明 | 认证/权限 |
|---|---|---|---|---|---|---|---|---|
| user-08-certificates-dns-acme | 证书列表 | GET | `/api/certificates` | 查询参数(均可选,任一出现即启用分页):<br>`offset` int64 ≥0,默认0<br>`size` int64 1~100,默认20<br>`q` string 关键字,≤200字符 | 200:<br>`certificates` Certificate[]<br>分页时额外:`count` int64 总数、`offset`、`size` | 错误:400 `证书列表分页或搜索参数无效`;502 `读取证书失败` | 不带分页参数时返回全部证书;带分页/搜索参数时走分页接口。 | 需登录 |
| user-08-certificates-dns-acme | 上传证书 | POST | `/api/certificates` | JSON 请求体(上限512KiB):<br>`csrfToken` string 必填<br>`certData` string 必填 PEM 证书(≤384KiB)<br>`keyData` string 必填 PEM 私钥(≤128KiB)<br>`name` string 可选,≤100字符,缺省取首个域名(多域名追加" 等 N 个域名"),再缺省为"手动证书"<br>`isOn` bool 可选,默认 true | 201:`{"id": int64}`(新证书ID) | 错误:400 `请输入证书和私钥`/`证书或私钥文件过大`/`证书或私钥无效: …`/`证书已经过期`/名称超长;403 CSRF;502 `上传证书失败` | 上传自有证书和私钥,服务端解析校验有效期,已过期证书拒绝。首个 DNS 名写入 `serverName`。 | 需实名(`requireIdentity`)+需登录+CSRF写保护 |
| user-08-certificates-dns-acme | 证书详情 | GET | `/api/certificates/{certificateID}` | 路径参数:`certificateID` int64 >0 | 200:`{"certificate": Certificate}` | 错误:400 `证书 ID 无效`;404 `证书不存在`;502 `读取证书失败` | 查询单个证书的元信息,不含私钥内容。 | 需登录 |
| user-08-certificates-dns-acme | 下载证书(公钥链) | GET | `/api/certificates/{certificateID}/download` | 路径参数:`certificateID` int64 >0 | 200:二进制附件 `certificate-{id}.pem`,`Content-Type: application/x-pem-file`,仅含 `CERTIFICATE` 类型 PEM 块(不含私钥) | 错误:400;404 `证书不存在`;422 `证书内容无效`;502 `读取证书失败` | 下载证书公钥链 PEM;响应带 `Cache-Control: no-store`。成功时不是 JSON。 | 需登录(无 CSRF,GET) |
| user-08-certificates-dns-acme | 导出证书私钥 | POST | `/api/certificates/{certificateID}/private-key/download` | 路径参数:`certificateID` int64 >0<br>JSON 请求体(上限4KiB):<br>`csrfToken` string 必填<br>`password` string 必填,≤512<br>`otpCode` string 必填(实际要求非空),≤32 | 200:二进制附件 `private-key-{id}.pem`(`application/x-pem-file`,`no-store`) | 错误:400 `请输入当前密码和有效的验证码`;403 管理员协助会话/CSRF/`请先启用多因素认证再导出私钥`/`请输入多因素认证验证码`/`密码或验证码错误`;404;422 `私钥内容无效`/`证书与私钥不匹配`;429 `验证尝试过多,请稍后再试`;502;503 `审计服务不可用,已取消私钥导出` | 高风险操作:必须启用 MFA,并重新校验密码+动态码;按用户限流(`certificate-export:{userID}`);管理员协助会话禁止;写入审计日志失败则取消导出。路由未包 `sensitiveUserWrite`,CSRF 在处理函数内校验。 | 需登录+CSRF(体内)+密码+MFA;管理员协助会话禁止 |
| user-08-certificates-dns-acme | 修改证书 | PATCH | `/api/certificates/{certificateID}` | 路径参数:`certificateID`<br>JSON 请求体:<br>`csrfToken` string 必填<br>`name` string 可选,≤100字符<br>`isOn` bool 可选<br>(`name` 与 `isOn` 必须且只能传一个) | 200:`{"ok": true}` | 错误:400 `必须且只能修改一个证书字段`/名称超长;403;404 `证书不存在`;502 `更新证书失败` | 重命名或启用/停用证书,一次只能改一项。 | 需登录+CSRF写保护 |
| user-08-certificates-dns-acme | 删除证书 | DELETE | `/api/certificates/{certificateID}` | 路径参数:`certificateID`<br>CSRF体 | 200:`{"ok": true}` | 错误:400;403;502 `删除证书失败` | 删除证书。 | 需登录+CSRF写保护 |
| user-08-certificates-dns-acme | DNS服务商列表 | GET | `/api/dns-providers` | 无 | 200:`{"providers": DNSProvider[]}`(仅返回受支持类型) | 错误:502 `读取 DNS 服务商失败`(可能附 RPC 细节) | 列出当前用户的 DNS 服务商配置,过滤掉不支持的类型。 | 需登录 |
| user-08-certificates-dns-acme | 创建DNS服务商 | POST | `/api/dns-providers` | JSON 请求体(上限16KiB),结构 DNSProviderWrite:<br>`csrfToken` string 必填<br>`name` string 必填,≤100字符<br>`type` string 可选,缺省 `cloudFlare`;枚举 `cloudFlare`/`dnspod`/`alidns`/`huaweiDNS`/`dnsla`/`volcengine`<br>`authMode` string 仅 cloudFlare 有效:空或 `token`<br>`apiParams` object 凭据,见公共结构 DNSProviderParams<br>`minTTL` int32,0 视为60,范围60~86400 | 201:`{"id": int64}` | 错误:400(`不支持的 DNS 服务商类型`/`请输入配置名称`/`最小 TTL 必须在 60 到 86400 秒之间`/缺少凭据提示等);403;502 `创建 DNS 服务商失败` | 创建时按类型强校验凭据必填:cloudFlare 需 `apiToken`(≤512);alidns/huaweiDNS/volcengine 需 `accessKeyId`+`accessKeySecret`;dnspod 同(`apiType` 固定 `tencentDNS`);dnsla 需 `apiId`+`secret`。 | 需实名(`requireIdentity`)+需登录+CSRF写保护 |
| user-08-certificates-dns-acme | DNS服务商详情 | GET | `/api/dns-providers/{providerID}` | 路径参数:`providerID` int64 >0 | 200:`{"provider": DNSProvider + "apiParams": DNSProviderParams}` | 错误:400 `DNS 服务商 ID 无效`;404 `DNS 服务商不存在`;502 `DNS 服务商配置无效`/读取失败 | 返回服务商基础信息及 `apiParams`。待确认:`apiParams` 中密钥字段是否已由 RPC 层脱敏,处理函数仅做 JSON 反序列化直接返回。 | 需登录 |
| user-08-certificates-dns-acme | 修改DNS服务商 | PATCH | `/api/dns-providers/{providerID}` | 路径参数:`providerID`<br>JSON 请求体(上限16KiB)同 DNSProviderWrite;更新时凭据可省略(由后端 RPC 层恢复已存密钥);`type` 缺省取现有类型 | 200:`{"ok": true}` | 错误:400(校验同创建、`不能修改 DNS 服务商类型,请新建其他类型的配置`);403;404 `DNS 服务商不存在`;502 `更新 DNS 服务商失败` | 修改名称/凭据/最小TTL;类型不可变;`name` 仍为必填。 | 需登录+CSRF写保护 |
| user-08-certificates-dns-acme | 删除DNS服务商 | DELETE | `/api/dns-providers/{providerID}` | 路径参数:`providerID`<br>CSRF体 | 200:`{"ok": true}` | 错误:400;403;502 `删除 DNS 服务商失败`(可能附 RPC 细节) | 删除 DNS 服务商配置。 | 需登录+CSRF写保护 |
| user-08-certificates-dns-acme | ACME选项汇总 | GET | `/api/acme/options` | 无 | 200:<br>`providers` ACMEProvider[] 可用证书服务商<br>`users` ACMEIdentity[] ACME 身份<br>`accounts` ACMEProviderAccount[] EAB 账户<br>`dnsProviders` DNSProvider[](仅受支持类型) | 错误:502(各子查询分别提示) | 一次性返回创建自动签发任务所需的下拉数据。 | 需登录 |
| user-08-certificates-dns-acme | ACME身份列表 | GET | `/api/acme/identities` | 无 | 200:`{"identities": ACMEIdentity[]}` | 错误:502 `读取 ACME 身份失败` | 列出 ACME 账号(身份)。 | 需登录 |
| user-08-certificates-dns-acme | 创建ACME身份 | POST | `/api/acme/identities` | JSON 请求体(上限64KiB):<br>`csrfToken` string 必填<br>`email` string 必填,≤254,合法邮箱<br>`description` string 可选,≤2000字符,允许换行/制表,不含其他控制字符<br>`providerCode` string 可选,缺省 `letsencrypt`(小写化)<br>`accountId` int64 当服务商需要 EAB 时必填,须属于同一服务商;否则忽略 | 201:`{"id": int64}` | 错误:400 `请输入有效的联系邮箱`/`ACME 身份说明无效`/`证书服务商不存在`/`请选择当前服务商的 EAB 账户`;403;502 `创建 ACME 身份失败` | 注册 ACME 账号;服务商 `requireEAB=true` 时必须绑定其 EAB 账户。 | 需登录+CSRF写保护 |
| user-08-certificates-dns-acme | 修改ACME身份说明 | PATCH | `/api/acme/identities/{identityID}` | 路径参数:`identityID` int64 >0<br>JSON 请求体(上限16KiB):<br>`csrfToken` string 必填<br>`description` string ≤2000字符(同上规则;可为空字符串) | 200:`{"ok": true}` | 错误:400 `ACME 身份 ID 无效`/`ACME 身份说明无效`;403;404 `ACME 身份不存在`;502 | 仅允许修改身份备注。 | 需登录+CSRF写保护 |
| user-08-certificates-dns-acme | 删除ACME身份 | DELETE | `/api/acme/identities/{identityID}` | 路径参数:`identityID`<br>CSRF体(上限8KiB) | 200:`{"ok": true}` | 错误:400;403;404 `ACME 身份不存在`;409 `ACME 身份仍被签发任务使用` | 删除前确认归属;RPC 删除失败一律按 409 返回(代码未区分失败原因)。 | 需登录+CSRF写保护 |
| user-08-certificates-dns-acme | EAB账户列表 | GET | `/api/acme/provider-accounts` | 无 | 200:`{"accounts": ACMEProviderAccount[]}` | 错误:502 `读取 ACME EAB 账户失败` | 列出 EAB(外部账户绑定)账户;不回显 EAB Key,仅 `hasEABKey`。 | 需登录 |
| user-08-certificates-dns-acme | 创建EAB账户 | POST | `/api/acme/provider-accounts` | JSON 请求体(上限16KiB):<br>`csrfToken` string 必填<br>`name` string 必填,≤120字符,无控制字符<br>`providerCode` string 可选,缺省 `letsencrypt`;必须是 `requireEAB` 的服务商<br>`eabKid` string 必填,≤1024<br>`eabKey` string 必填,≤4096 | 201:`{"id": int64}` | 错误:400 `ACME EAB 账户名称无效`/`EAB KID 无效`/`EAB HMAC Key 无效`/`请选择需要 EAB 的证书服务商`/`请填写 EAB KID 和 HMAC Key`;403;502 | 保存 EAB KID/HMAC Key,供需要 EAB 的服务商注册身份。 | 需登录+CSRF写保护 |
| user-08-certificates-dns-acme | 修改EAB账户 | PATCH | `/api/acme/provider-accounts/{accountID}` | 路径参数:`accountID` int64 >0<br>JSON 请求体(上限16KiB):<br>`csrfToken` string 必填<br>`name` string 必填,≤120<br>`eabKid` string 必填,≤1024<br>`eabKey` string 可选(更新时可空,空则不改),≤4096 | 200:`{"ok": true}` | 错误:400(同创建校验,服务商取自现有账户,不可改);403;404 `ACME EAB 账户不存在`;502 | 修改名称/KID/Key;`providerCode` 不在请求体中,不可变更。 | 需登录+CSRF写保护 |
| user-08-certificates-dns-acme | 删除EAB账户 | DELETE | `/api/acme/provider-accounts/{accountID}` | 路径参数:`accountID`<br>CSRF体(上限8KiB) | 200:`{"ok": true}` | 错误:400;403;404 `ACME EAB 账户不存在`;409 `ACME EAB 账户仍被签发身份使用` | 删除 EAB 账户;RPC 失败一律按 409 返回。 | 需登录+CSRF写保护 |
| user-08-certificates-dns-acme | 自动签发任务列表 | GET | `/api/acme/tasks` | 无 | 200:`{"tasks": ACMETask[]}` | 错误:502 `读取自动签发任务失败` | 列出所有自动签发(ACME)任务,不分页。 | 需登录 |
| user-08-certificates-dns-acme | 创建自动签发任务 | POST | `/api/acme/tasks` | JSON 请求体(上限64KiB):<br>`csrfToken` string 必填<br>`domains` string[] 必填,1~100个,小写去重;通配符 `*.x` 仅限 DNS 验证<br>`authType` string 可选,`http`(缺省)/`dns`<br>`dnsDomain` string DNS 验证必填(根域名),`domains` 必须属于它;http 时忽略<br>`dnsProviderId` int64 DNS 验证必填且须属于当前用户<br>`autoRenew` bool<br>`acmeUserId` int64 >0 时复用该身份;否则按下列字段查找或新建身份:<br>`providerCode`(缺省 `letsencrypt`)、`email`(必填、合法)、`description`、`accountId`(服务商需 EAB 时必填) | **202**:`{"id": int64}`(任务ID,异步签发) | 错误:400 `验证方式无效`/`域名数量必须在 1 到 100 个之间`/`请输入有效的 DNS 根域名`/`通配符证书必须使用 DNS 验证`/`域名 "…" 格式无效`/`域名 "…" 不属于 DNS 根域名 …`/`ACME 账号不存在`/`证书服务商不存在`/`DNS 服务商不存在`;403;502 `创建自动签发任务失败`/`创建 ACME 账号失败` | 创建任务并异步(`Async:true`)签发;未指定 `acmeUserId` 时会按"服务商+邮箱+EAB账户"复用或自动创建 ACME 身份。 | 需登录+CSRF写保护 |
| user-08-certificates-dns-acme | 自动签发任务详情 | GET | `/api/acme/tasks/{taskID}` | 路径参数:`taskID` int64 >0 | 200:`{"task": ACMETask}` | 错误:400 `自动签发任务 ID 无效`;404 `自动签发任务不存在`;502 | 通过遍历任务列表按 ID 查找。 | 需登录 |
| user-08-certificates-dns-acme | 修改自动续签 | PATCH | `/api/acme/tasks/{taskID}` | 路径参数:`taskID`<br>JSON 请求体:<br>`csrfToken` string 必填<br>`autoRenew` bool 必填(缺失 400 `缺少自动续签状态`) | 200:`{"ok": true}` | 错误:400;403;404 `自动签发任务不存在`;502 `更新自动续签状态失败` | 仅允许切换自动续签开关,其他字段沿用原任务值回写。 | 需登录+CSRF写保护 |
| user-08-certificates-dns-acme | 立即执行签发任务 | POST | `/api/acme/tasks/{taskID}/run` | 路径参数:`taskID`<br>CSRF体 | 200:<br>`ok` bool 本次签发是否成功<br>`error` string 失败原因<br>`certificateId` int64 成功后的证书ID | 错误:400;403;502 `执行自动签发任务失败` | 同步执行,超时 3 分钟;签发失败仍返回 200,须看 `ok`/`error`。 | 需登录+CSRF写保护 |
| user-08-certificates-dns-acme | 删除自动签发任务 | DELETE | `/api/acme/tasks/{taskID}` | 路径参数:`taskID`<br>CSRF体 | 200:`{"ok": true}` | 错误:400;403;502 `删除自动签发任务失败` | 删除任务。 | 需登录+CSRF写保护 |


### 公共结构

CSRF体:`{"csrfToken": string}`(必填,与会话 CSRF 令牌一致)。

Certificate(`apiCertificateResponse`):
- `id` int64、`isOn` bool、`name` string、`description` string(空则省略)、`serverName` string(空则省略)
- `isCA` bool、`isACME` bool(是否 ACME 签发)
- `timeBeginAt` int64、`timeEndAt` int64(Unix 秒)
- `dnsNames` string[]、`commonNames` string[]
- `ocspExpiresAt` int64(0 省略)、`ocspError` string(空省略)

DNSProvider(`apiDNSProviderResponse`):`id` int64、`name`、`type`、`typeName`、`dataUpdatedAt` int64(0 省略)、`minTTL` int32。

DNSProviderParams(`apiDNSProviderParams`,字段均 omitempty):`apiToken`、`accessKeyId`、`accessKeySecret`、`regionId`(alidns)、`region`(volcengine)、`apiType`、`endpoint`(huaweiDNS/volcengine)、`apiId`、`secret`(dnsla)、`url`、`host`、`role`。

ACMEProvider:`code`、`name`、`description`(省略空)、`requireEAB` bool、`eabDescription`(省略空)。

ACMEProviderAccount:`id`、`name`、`providerCode`、`eabKid`(省略空)、`hasEABKey` bool、`error`(省略空)、`provider` ACMEProvider(可选)。

ACMEIdentity:`id`、`email`、`description`(省略空)、`providerCode`、`createdAt` int64(省略0)、`account` ACMEProviderAccount(可选)。

ACMETask:
- `id`、`isOn`、`domains` string[]、`createdAt`、`autoRenew`、`authType`(`http`/`dns`)、`dnsDomain`(省略空)
- `acmeUser` ACMEIdentity(可选)、`dnsProvider` DNSProvider(可选)
- `certificate` {`id`,`name`,`expiresAt`}(可选,已签发证书)
- `latestTaskLog` {`isOK` bool,`error` string,`createdAt` int64}(可选)

<a id="m23"></a>

## 23. 用户端 · 静态网站 Pages(用户端)

**模块约定**

> 路由总数 22,来源 EdgeUserPortal/internal/web/server.go
说明:
- 所有错误统一返回 `{"error":"消息"}`;未登录统一 401 `请先登录`。
- 路径参数 `{siteID}` 不是数据库自增ID,而是服务端生成的公开引用串(前缀 + HMAC-SHA256 后 base32 小写,由 `POST /api/pages` 与列表/详情的 `id` 返回)。引用串无效或不属于当前用户的静态站点,统一 404 `资源不存在`;服务端先拉取用户全部站点再比对,上游失败 502 `读取资源失败`。
- "需登录+CSRF写保护" = 路由包了 `sensitiveUserWrite`(管理员协助会话 403 `管理员协助会话不能修改用户资源或安全配置`;平台授权失效 403 只读提示),且处理函数内再用 `validCSRF` 校验 `csrfToken`(不匹配 403 `CSRF 校验失败`)。文件上传例外:CSRF 令牌放在请求头 `X-CSRF-Token`。
- 请求体使用 `decodeAPIJSON`/`decodeAPIJSONLimit`:禁止未知字段、只允许单个 JSON 对象,默认上限 64KiB;格式错误/超限 400 `请求格式无效`。
- "CSRF体" 见"公共结构"。

| 模块 | 接口名称 | 方法 | 路径 | 请求参数 | 响应数据 | 主要错误 | 功能说明 | 认证/权限 |
|---|---|---|---|---|---|---|---|---|
| user-09-pages | 静态网站列表 | GET | `/api/pages` | 无 | 200:<br>`config` PagesConfig<br>`sites` PageSite[]<br>`requirePlan` bool(用户站点策略是否要求套餐) | 错误:502 `读取静态站点配置失败`/`读取用户站点策略失败`/`读取静态站点失败` | 返回静态站点服务配置与用户全部 Pages 站点(仅 `Type=pages` 的站点)。 | 需登录 |
| user-09-pages | 创建静态网站 | POST | `/api/pages` | JSON 请求体(上限32KiB):<br>`csrfToken` string 必填<br>`domains` string[] 必填,1~20个,小写去重,支持 `*.` 前缀通配,不能是平台保留域名<br>`userPlanId` int64 ≥0,0 表示不绑定套餐 | 201:`{"id": string}`(站点公开引用) | 错误:400 `套餐 ID 无效`/`至少需要一个域名`/`域名不能超过 20 个`/`域名 "…" 格式无效`/`域名 "…" 是平台保留域名,不能绑定到客户站点`;403;409 `静态站点服务尚未由管理员启用`;502 `创建静态站点失败` | 新建 Pages 站点并绑定访问域名;管理员未启用 Pages 或未配置存储时返回 409。 | 需实名(`requireIdentity`)+需登录+CSRF写保护 |
| user-09-pages | 静态网站详情 | GET | `/api/pages/{siteID}` | 路径参数:`siteID` string 公开引用 | 200:<br>`site` PageSite<br>`pages` ServerPages {`isOn` bool,`createdAt` int64,`storageLimitMB` int64}<br>`config` PagesConfig<br>`dns` PagesDNS(DNS 解析自检结果) | 错误:404 `资源不存在`/`静态站点不存在`;502 `读取静态站点存储失败`/`读取静态站点配置失败`/`读取静态站点失败` | 并发读取存储、配置并做域名 DNS 自检(单次最长 4 秒);`storageLimitMB` 取站点桶容量,为 0 时回落到全局每站点存储上限。 | 需登录 |
| user-09-pages | 修改静态网站 | PATCH | `/api/pages/{siteID}` | 路径参数:`siteID`<br>JSON 请求体(上限32KiB):<br>`csrfToken` string 必填<br>`name` string 可选,去空白后 1~128 字节<br>`domains` string[] 可选,规则同创建<br>`isOn` bool 可选<br>(三者必须且只能传一个) | 200:`{"ok": true}` | 错误:400 `每次只能修改一个静态网站配置项`/`静态网站名称必须为 1 至 128 个字符`;403;404;502 `更新静态网站失败: …` | 重命名、改域名、启停,一次一项。注意:`domains` 的格式/保留域名校验失败也走 502 `更新静态网站失败: <原因>`,而非 400。 | 需登录+CSRF写保护 |
| user-09-pages | 删除静态网站 | DELETE | `/api/pages/{siteID}` | 路径参数:`siteID`<br>CSRF体(上限8KiB) | 200:`{"ok": true}` | 错误:403;404;502 `删除静态网站失败` | 删除 Pages 站点。 | 需登录+CSRF写保护 |
| user-09-pages | 获取协议配置 | GET | `/api/pages/{siteID}/protocols` | 路径参数:`siteID` | 200:SiteProtocols<br>`httpEnabled`/`httpsEnabled`/`http2Enabled`/`http3Enabled`/`alwaysUseHTTPS` bool<br>`certificateIds` int64[] 已启用证书ID<br>`serverNames` string[] 站点精确域名<br>`filterable` bool<br>`certificates` SiteCertificate[](非 CA 证书,按 `timeEndAt` 升序) | 错误:404;502 `读取站点协议配置失败`/`读取 HTTPS 跳转配置失败`/`读取证书列表失败` | 返回协议开关、已绑定证书及全部可选证书,并带与站点域名的匹配结果。`filterable=false`(无域名或含通配/正则)时前端应展示全部证书。 | 需登录 |
| user-09-pages | 更新协议配置 | PUT | `/api/pages/{siteID}/protocols` | 路径参数:`siteID`<br>JSON 请求体(上限64KiB):<br>`csrfToken` string 必填<br>`httpEnabled` bool<br>`httpsEnabled` bool<br>`http2Enabled` bool<br>`http3Enabled` bool<br>`alwaysUseHTTPS` bool<br>`certificateIds` int64[] ≤20,去重 | 200:`{"ok": true}` | 错误:400 `HTTP/3 需要同时开启 HTTPS 和 HTTP/2`/`始终使用 HTTPS 需要同时开启 HTTP 和 HTTPS`/`单个站点最多选择 20 张证书`/`包含不可用、已过期或不属于当前用户的证书`/`开启 HTTPS 前请至少选择一张有效证书`;403 `当前套餐未开通HTTP/3功能`、`当前套餐未开通始终使用 HTTPS功能`(格式:`当前套餐未开通<名称>功能`);404;502 `更新站点协议配置失败` | 全量覆盖协议配置。启用 HTTP/3 需套餐含 `UserFeatureCodeServerHTTP3` 功能,启用 `alwaysUseHTTPS` 需含 HTTPS 跳转功能;所选证书必须启用、非 CA、未过期且属于当前用户。 | 需登录+CSRF写保护 |
| user-09-pages | 获取WAF配置 | GET | `/api/pages/{siteID}/waf` | 路径参数:`siteID` | 200:`{"site": PageSite, "waf": WAFConfig}` | 错误:404 `站点不存在`;403 `当前套餐未开通 WAF 功能`;502 `读取站点失败`/`读取 WAF 配置失败` | 返回站点独立 WAF 配置及可选规则组(入站/出站)。 | 需登录 |
| user-09-pages | 更新WAF配置 | PUT | `/api/pages/{siteID}/waf` | 路径参数:`siteID`<br>JSON 请求体(上限64KiB)= `csrfToken` + WAFConfig 全部字段(见公共结构):<br>`isPrior`、`isOn`、`ignoreGlobalRules`、`defaultCaptcha`、`forceCaptcha`、`mode`、`maxRequestBodySize`、`groups`<br>校验:`mode` 须为有效模式;`defaultCaptcha` 须为有效类型;`forceCaptcha=true` 需 `isOn=true`;`maxRequestBodySize` 0 取默认值,否则 64KiB~16MiB;`groups[].code` 须为模板内已知规则组 | 200:`{"ok": true}` | 错误:400 `WAF 模式无效`/`默认人机验证方式无效`/`启用强制 CAPTCHA 前请先启用 WAF`/`WAF 请求内容检查上限必须在 64 KiB 至 16 MiB 之间`/`未知的 WAF 规则组: <code>`;403 `当前套餐未开通 WAF 功能`;404;502 `读取 WAF 配置失败`/`更新 WAF 配置失败` | 全量覆盖站点 WAF;首次开启且无策略时按模板创建"Pages WAF"策略;`groups` 中未传入或 `isOn=false` 的模板规则组会被关闭。 | 需登录+CSRF写保护 |
| user-09-pages | 获取访问控制 | GET | `/api/pages/{siteID}/access` | 路径参数:`siteID` | 200:<br>`site` PageSite<br>`access` {`allowIPs` string[],`denyIPs` string[],`allowCountryIds` int64[],`denyCountryIds` int64[]}<br>`countries` [{`id` int64,`name` string,`code` string 大写国家码}] | 错误:404 `站点不存在`;502 `读取站点失败`/`读取访问控制配置失败` | 返回 IP 黑白名单和国家/地区黑白名单,以及可选国家列表。 | 需登录 |
| user-09-pages | 更新访问控制 | PUT | `/api/pages/{siteID}/access` | 路径参数:`siteID`<br>JSON 请求体(上限64KiB):<br>`csrfToken` string 必填<br>`allowIPs` string[] 每项单个 IP 或 CIDR,≤500,自动规范化去重<br>`denyIPs` string[] 同上<br>`allowCountryIds` int64[] ≤300,>0,须为可选国家<br>`denyCountryIds` int64[] 同上 | 200:`{"ok": true}` | 错误:400 `允许名单无效: …`/`禁止名单无效: …`(`每个名单最多 500 条`/`请输入单个 IP 或 CIDR`)/`同一地址范围不能同时出现在允许和禁止名单中`/`国家或地区选项过多`/`国家或地区选项无效`/`同一国家或地区不能同时允许和禁止`;403;404;502 `读取国家和地区列表失败`/`更新访问控制配置失败` | 全量覆盖访问控制;允许与禁止名单的 IP 范围、国家均不能重叠(同一IP版本内的包含关系即视为重叠)。 | 需登录+CSRF写保护 |
| user-09-pages | 获取访问验证(UAM) | GET | `/api/pages/{siteID}/uam` | 路径参数:`siteID` | 200:`{"site": PageSite, "uam": UAMConfig}` | 错误:403 `当前套餐未开通访问验证功能`;404 `站点不存在`;502 `读取站点访问验证配置失败` | 需套餐含 `UserFeatureCodeServerUAM` 功能;无配置时返回默认值(`challengeDifficulty` 18、`challengeTimeout` 120),`isPrior` 恒为 true。 | 需登录;需套餐功能 |
| user-09-pages | 更新访问验证(UAM) | PUT | `/api/pages/{siteID}/uam` | 路径参数:`siteID`<br>JSON 请求体(上限64KiB)= `csrfToken` + UAMConfig 字段:<br>`isOn` bool、`addToWhiteList` bool<br>`minQPSPerIP` int 0~999999<br>`keyLife` int 0~3600<br>`challengeDifficulty` int 14~22(0→18)<br>`challengeTimeout` int 60~180(0→120)<br>`onlyURLPatterns`/`exceptURLPatterns` URLPattern[] ≤100<br>(`isPrior` 字段可传但被忽略,服务端固定 true) | 200:`{"ok": true}` | 错误:400 `访问验证高级参数无效`/`仅限 URL 规则无效: …`/`排除 URL 规则无效: …`/`访问验证配置无效: …`;403 `当前套餐未开通访问验证功能`;404;502 `更新站点访问验证配置失败` | 全量覆盖 5 秒盾/访问验证配置。 | 需登录+CSRF写保护;需套餐功能 |
| user-09-pages | 获取CC防护 | GET | `/api/pages/{siteID}/cc` | 路径参数:`siteID` | 200:`{"site": PageSite, "cc": CCConfig}` | 错误:403 `当前套餐未开通CC 防护功能`;404 `站点不存在`;502 `读取站点 CC 防护配置失败` | 需套餐含 `UserFeatureCodeServerCC` 功能;无配置时返回默认配置,`isPrior` 恒为 true。 | 需登录;需套餐功能 |
| user-09-pages | 更新CC防护 | PUT | `/api/pages/{siteID}/cc` | 路径参数:`siteID`<br>JSON 请求体(上限64KiB)= `csrfToken` + CCConfig 字段:<br>`isOn` bool<br>`thresholds` CCThreshold[] 必填 1~10 条<br>`onlyURLPatterns`/`exceptURLPatterns` URLPattern[] ≤100 | 200:`{"ok": true}` | 错误:400 `CC 阈值数量必须为 1 到 10 条`/`CC 阈值名称不能超过 100 个字符`/`HTTP 方法数量过多`/`HTTP 方法无效: X`/`CC 阈值无效: …`/`启用 CC 防护时至少需要一条已启用阈值`/URL 规则错误/`CC 防护配置无效: …`;403 `当前套餐未开通CC 防护功能`;404;502 `更新站点 CC 防护配置失败` | 全量覆盖 CC 防护;仅 `isOn=true` 的阈值会被初始化校验;整体启用时至少需一条已启用阈值。`methods` 仅允许标准 HTTP 方法(≤16,大写去重)。 | 需登录+CSRF写保护;需套餐功能 |
| user-09-pages | 获取缓存配置 | GET | `/api/pages/{siteID}/cache` | 路径参数:`siteID` | 200:`{"site": PageSite, "cache": CacheConfig}` | 错误:404 `站点不存在`;502 `读取站点失败`/`读取缓存配置失败` | 返回站点缓存开关与规则;高级缓存规则(无法映射为简单规则的)不在 `rules` 中展示。 | 需登录 |
| user-09-pages | 更新缓存配置 | PUT | `/api/pages/{siteID}/cache` | 路径参数:`siteID`<br>JSON 请求体(上限64KiB)= `csrfToken` + CacheConfig 字段:<br>`isPrior`、`isOn`、`addStatusHeader`、`addAgeHeader`、`enableCacheControlMaxAge`、`useDefaultPolicy`、`purgeIsOn` bool<br>`rules` CacheRule[] ≤50<br>(`purgeKeyConfigured` 可传但被忽略) | 200:`{"ok": true}` | 错误:400 `缓存规则不能超过 50 条`/`缓存规则索引无效`/`不能修改高级缓存规则`/`缓存最小内容大小不能大于最大内容大小`/`缓存配置无效: …`等规则校验错误;403;404;502 `读取缓存配置失败`/`更新缓存配置失败` | 全量覆盖缓存配置:`sourceIndex≥0` 的规则按原下标修改,`-1`(<0) 视为新增;未在请求中出现的简单规则被删除,高级规则保留不动。待确认:`matchType`、`life`、`status` 等字段的具体枚举/范围(由 `buildCacheCondition`、`validateCacheLife` 等函数校验,本次未逐项读取)。 | 需登录+CSRF写保护 |
| user-09-pages | 创建缓存任务 | POST | `/api/pages/{siteID}/cache/tasks` | 路径参数:`siteID`<br>JSON 请求体(上限256KiB):<br>`csrfToken` string 必填<br>`type` string 必填:`purge`(刷新)/`fetch`(预热)<br>`keyType` string 必填:`key`(完整URL)/`prefix`(前缀);`fetch` 只能用 `key`<br>`keys` string[] 必填,1~1000个,每个 ≤2048 字符,必须是 http/https 完整 URL(不含用户信息),去重 | **201**:`{"id": int64, "count": int64}`(任务ID,有效URL数) | 错误:400 `缓存任务类型无效`/`缓存任务匹配方式无效`/`缓存预热只能使用完整 URL`/`请至少输入一个 URL`/`单次最多提交 1000 个 URL`/`URL 长度不能超过 2048 个字符`/`URL 无效: …`;403;404 `站点不存在`;422 `没有匹配到当前账户的有效站点 URL`;502 `创建缓存任务失败` | 提交缓存刷新/预热任务;URL 是否属于当前账户由后端 RPC 判定,全不匹配则 422。 | 需登录+CSRF写保护 |
| user-09-pages | 静态文件列表 | GET | `/api/pages/{siteID}/files` | 路径参数:`siteID`<br>查询参数:`directory` string 可选,站点内相对目录,空=根;禁止 `..` 越界与 NUL | 200:<br>`directory` string 规范化后目录<br>`parent` string 上级目录("" 为根)<br>`files` PagesFile[] | 错误:400 `目录路径无效`;404 `资源不存在`/`静态站点不存在`;502 `读取静态文件失败` | 列出目录内容;文件名不合法的条目被静默跳过。 | 需登录 |
| user-09-pages | 上传静态文件 | POST | `/api/pages/{siteID}/files` | 路径参数:`siteID`<br>请求头:`X-CSRF-Token` 必填<br>查询参数:`directory` string 可选(同上)<br>请求体:`multipart/form-data`,表单字段 `file`(文件);文件名 ≤255,不得含 `/`、`\`、NUL,且不能被路径规范化改变;只处理第一个 `file` 字段 | 201:`{"ok": true, "path": string}`(文件完整相对路径) | 错误:400 `上传请求格式无效`/`请选择上传文件`/`文件名无效`/`目录路径无效`;403 CSRF;404;413 `上传文件不能超过 512 MiB`;502 `上传静态文件失败` | 流式上传单个文件到站点存储,超时 10 分钟,体积上限 512MiB(`maximumUploadBytes`)。同名文件的覆盖行为待确认:由 RPC 层决定。 | 需登录+CSRF写保护(CSRF 在请求头) |
| user-09-pages | 删除静态文件 | DELETE | `/api/pages/{siteID}/files` | 路径参数:`siteID`<br>JSON 请求体(上限16KiB):<br>`csrfToken` string 必填<br>`filename` string 必填,站点内相对路径,不能为空 | 200:`{"ok": true, "purgeQueued": bool}`(缓存清理任务是否已提交) | 错误:400 `文件路径无效`;403;404;502 `删除静态文件失败` | 删除文件后自动对其所在目录发起缓存清理;清理失败不影响删除结果,仅 `purgeQueued=false`。 | 需登录+CSRF写保护 |
| user-09-pages | 清理缓存(按目录) | POST | `/api/pages/{siteID}/purge` | 路径参数:`siteID`<br>JSON 请求体(上限16KiB):<br>`csrfToken` string 必填<br>`directory` string 可选,空=整站 | **202**:`{"ok": true}` | 错误:400 `目录路径无效`;403;404;502 `创建缓存清理任务失败` | 对站点全部域名的指定目录创建缓存清理任务(异步)。与"创建缓存任务"不同,这里按目录而非 URL 列表。 | 需登录+CSRF写保护 |


### 公共结构

CSRF体:`{"csrfToken": string}`。

PagesConfig:`available` bool(服务已启用且配置了存储)、`projectsPerLimit` int64(每用户站点数上限)、`storageLimitMB` int64(每站点存储上限)、`defaultPage` string、`default404Page` string、`maximumUploadBytes` int64(固定 536870912)。配置读取为空时仅返回 `maximumUploadBytes`。

PageSite:`id` string(公开引用)、`name`、`domain`(首个绑定域名,依次回落到 ServerNames 首项、站点名)、`isOn` bool、`dnsName` string(集群 CNAME 接入域名,空省略)、`createdAt` int64(0 省略)。不含源站、集群、站点组信息。

PagesDNS:`recordType`(`CNAME` 或 `A/AAAA`)、`host`、`target`(空省略)、`resolved` bool、`addresses` string[]、`message` string(解析状态提示,如"域名尚未解析,请添加 CNAME 记录后等待 DNS 生效")。含通配符域名时不做检查,`message` 提示先绑定非通配域名。

SiteCertificate:Certificate 全部字段(`id`、`isOn`、`name`、`description`、`serverName`、`isCA`、`isACME`、`timeBeginAt`、`timeEndAt`、`dnsNames`、`commonNames`、`ocspExpiresAt`、`ocspError`,字段含义同 user-08 的 Certificate)+ `matchesSite` bool、`matchedNames` string[]。

WAFConfig:`isPrior` bool、`isOn` bool、`ignoreGlobalRules` bool、`defaultCaptcha` string、`forceCaptcha` bool、`mode` string、`maxRequestBodySize` int64(字节)、`groups` WAFRuleGroup[]。待确认:`mode`(防御/观察/绕过三值)与 `defaultCaptcha`(none/default 两值)的字面量由 `firewallconfigs` 常量定义,本次未读取。
WAFRuleGroup:`code` string、`name`、`description`、`direction`(`inbound`/`outbound`)、`isOn` bool。

URLPattern:`type` string(wildcard/regexp/images/audios/videos 五类,字面量由 `shared.URLPatternType*` 常量定义,待确认)、`pattern` string(wildcard/regexp 必填且 ≤2000 字符,其余类型忽略)。

UAMConfig:`isPrior`、`isOn`、`addToWhiteList` bool,`minQPSPerIP`、`keyLife`、`challengeDifficulty`、`challengeTimeout` int,`onlyURLPatterns`、`exceptURLPatterns` URLPattern[]。

CCConfig:`isPrior`、`isOn` bool,`thresholds` CCThreshold[],`onlyURLPatterns`、`exceptURLPatterns` URLPattern[]。
CCThreshold:`name` string(≤100)、`isOn` bool、`path` string(可选)、`conditions` object(可选,HTTPCCConditions)、`groupBy` string(可选)、`priority` int(可选)、`periodSeconds` int、`maxRequests` int、`blockSeconds` int、`statusCode` int、`perURI` bool、`methods` string[]。待确认:各数值字段的范围由 `threshold.Init()` 校验,本次未读取。

CacheConfig:`isPrior`、`isOn`、`addStatusHeader`、`addAgeHeader`、`enableCacheControlMaxAge`、`useDefaultPolicy`、`purgeIsOn`、`purgeKeyConfigured`(仅响应有意义)bool,`rules` CacheRule[]。
CacheRule:`sourceIndex` int(原规则下标,新增传负数)、`isOn`、`isReverse`、`matchType` string(含 `extension` 等)、`values` string[]、`isCaseInsensitive` bool、`life` {`count` int64,`unit` string}、`ignoreQuery` bool、`status` int[]、`minSize`/`maxSize` {`count` int64,`unit` string}、`skipCacheControlValues` string[]、`skipSetCookie`、`enableRequestCachePragma`、`allowChunkedEncoding`、`allowPartialContent`、`forcePartialContent`、`alwaysForwardRangeRequest`、`enableIfNoneMatch`、`enableIfModifiedSince`、`enableReadingOriginAsync` bool。

PagesFile:`name`、`path`(含目录的完整相对路径)、`size` int64、`isDirectory` bool、`createdAt` int64(Unix秒,0 省略)。

<a id="m24"></a>

## 24. 用户端 · 站点分组配置(用户端)

**模块约定**

> 路由总数 40,来源 EdgeUserPortal/internal/web/server.go
说明:
- 所有错误统一返回 `{"error":"消息"}`;未登录统一 `401` `请先登录`。
- 路径参数 `{groupID}` 为站点组 ID(int64),`<=0` 或非数字返回 `400` `站点组 ID 无效`(由 `requireAPIGroup` 处理)。
- "需登录+CSRF写保护" = 路由包了 `sensitiveUserWrite`(未登录 401;管理员协助会话 403 `管理员协助会话不能修改用户资源或安全配置`;平台授权失效 403 只读提示),并且处理函数内用 `validCSRF` 比对请求体字段 `csrfToken`(不匹配 403 `CSRF 校验失败`)。CSRF 令牌放在 JSON 请求体,不是请求头。
- 请求体使用 `decodeAPIJSON`:上限 64KiB、禁止未知字段、只允许单个 JSON 对象;失败 400 `请求格式无效`(缓存任务接口上限为 256KiB)。
- 配置类接口统一模式:
  - GET:`200` `{"group":ServerGroup,"<配置键>":{...}}`。通用错误:`400` `站点组 ID 无效`、`404` `站点组不存在`、`502` `读取站点组失败`/`读取...配置失败`。
  - PUT:请求体 = `csrfToken` + 该配置的字段(平铺);成功 `200` `{"ok":true}`。通用错误:`400`(`请求格式无效` 或字段校验文案)、`403` `CSRF 校验失败`、`404` `站点组不存在`、`502`。
  - 表中"通用 GET 返回"指上述 GET 模式,只写出特有的配置键与字段;"通用 PUT 返回"指 `200 {"ok":true}` 及上述通用错误。
- 套餐功能门控:需要的功能未开通 `403` `当前套餐未开通X功能`;读取套餐功能失败 `502` `读取套餐功能失败`。
- 配置字段 `isPrior` 表示是否覆盖(优先于)默认配置,`isOn` 表示是否启用。
- "公共结构"见表后。

| 模块 | 接口名称 | 方法 | 路径 | 请求参数 | 响应数据 | 主要错误 | 功能说明 | 认证/权限 |
|---|---|---|---|---|---|---|---|---|
| user-10-groups | 站点组列表 | GET | `/api/groups` | 无 | `200` `{"groups":[ServerGroup...]}`(无数据时为 `[]`) | 错误:`502` `读取站点组失败` | 列出当前用户的站点组。RPC 超时 15s。 | 需登录 |
| user-10-groups | 创建站点组 | POST | `/api/groups` | JSON 请求体:<br>`csrfToken` string 必填<br>`name` string 必填,trim 后 1-100 字符 | 成功 `201` `{"id":groupID}` | 错误:`400` 名称无效/`请求格式无效`;`403` CSRF;`502` `创建站点组失败` | 创建站点组。 | 需登录+CSRF写保护 |
| user-10-groups | 站点组详情 | GET | `/api/groups/{groupID}` | 路径参数:`groupID` int 必填 | `200`:<br>`group` ServerGroup<br>`config` 配置概要(`pb.FindEnabledServerGroupConfigInfoResponse`,字段为 `hasHTTPReverseProxy`、`hasTCPReverseProxy`、`hasUDPReverseProxy`、`serverGroupId`、`hasRootConfig`、`hasWAFConfig`、`hasCacheConfig`、`hasCharsetConfig`、`hasStatConfig`、`hasCompressionConfig`、`hasOptimizationConfig`、`hasRequestHeadersConfig`、`hasResponseHeadersConfig`、`hasWebsocketConfig`、`hasWebPConfig`、`hasAccessLogConfig`、`hasRemoteAddrConfig`、`hasPagesConfig`、`hasRequestLimitConfig`、`hasRequestScriptsConfig`、`hasHLSConfig`,均 bool;待确认:Go 序列化时的 json tag 大小写以 pb 生成代码为准,本模块未核对)<br>`features` []string(当前套餐功能码) | 错误:`404` `站点组不存在`;`502` `读取站点组配置失败` / `unable to read plan features` | 读取站点组及其各类配置是否已设置、套餐功能列表。 | 需登录 |
| user-10-groups | 修改站点组名称 | PATCH | `/api/groups/{groupID}` | 路径参数:`groupID`<br>JSON 请求体:`csrfToken` 必填;`name` string 必填,1-100 | `200` `{"ok":true}` | 错误:`404` `站点组不存在或无权操作`;`409` `站点组名称已存在，请使用其他名称`;`502`;`400`;`403` | 重命名站点组,名称在用户下唯一。 | 需登录+CSRF写保护 |
| user-10-groups | 删除站点组 | DELETE | `/api/groups/{groupID}` | 路径参数:`groupID`<br>JSON 请求体:`csrfToken` 必填 | `200` `{"ok":true}` | 错误:`404` `站点组不存在或无权操作`;`409` `站点组中仍有站点，请先移出站点后再删除`;`502`;`403` | 删除站点组。风险操作,组内仍有站点时拒绝删除。 | 需登录+CSRF写保护 |
| user-10-groups | 读取 5 秒盾配置 | GET | `/api/groups/{groupID}/uam` | 路径参数:`groupID` | 通用 GET 返回,配置键 `uam`:<br>`isPrior`、`isOn`、`addToWhiteList` bool<br>`minQPSPerIP` int<br>`keyLife` int(秒)<br>`challengeDifficulty` int<br>`challengeTimeout` int(秒)<br>`onlyURLPatterns`、`exceptURLPatterns` URLPattern[]<br>额外 | 错误:`403` `当前套餐未开通5秒盾功能`(功能码 `server.uam`) | 读取站点组 5 秒盾(UAM)配置。 | 需登录 |
| user-10-groups | 更新 5 秒盾配置 | PUT | `/api/groups/{groupID}/uam` | 路径参数:`groupID`<br>JSON 请求体:`csrfToken` 必填;`isPrior`、`isOn`、`addToWhiteList` bool;`minQPSPerIP` int 0-999999;`keyLife` int 0-3600;`challengeDifficulty` int,0 取默认 18,范围 14-22;`challengeTimeout` int,0 取默认 120,范围 60-180;`onlyURLPatterns`、`exceptURLPatterns` URLPattern[] | 通用 PUT 返回。<br>特有 | 错误:`400` `5 秒盾高级参数无效`、`仅限/排除 URL 规则无效: …`;`403` 套餐未开通 | 更新 5 秒盾。 | 需登录+CSRF写保护 |
| user-10-groups | 读取 CC 防护配置 | GET | `/api/groups/{groupID}/cc` | 路径参数:`groupID` | 通用 GET 返回,配置键 `cc`:<br>`isPrior`、`isOn` bool<br>`thresholds` CCThreshold[]<br>`onlyURLPatterns`、`exceptURLPatterns` URLPattern[]<br>额外 | 错误:`403` 套餐未开通 CC 功能(`server.cc`) | 读取 CC 防护配置。 | 需登录 |
| user-10-groups | 更新 CC 防护配置 | PUT | `/api/groups/{groupID}/cc` | 路径参数:`groupID`<br>JSON 请求体:`csrfToken` 必填;`isPrior`、`isOn`;`thresholds` CCThreshold[],1-10 条;`onlyURLPatterns`、`exceptURLPatterns` | 通用 PUT 返回。<br>特有 | 错误:`400` 阈值数量/字段无效、`HTTP 方法无效: X`、`HTTP 方法数量过多`;启用 `isOn` 时至少需一条已启用阈值;`403` 套餐未开通 | 更新 CC 防护阈值,启用的阈值会做配置初始化校验。 | 需登录+CSRF写保护 |
| user-10-groups | 读取 WebP 配置 | GET | `/api/groups/{groupID}/webp` | 路径参数:`groupID` | 通用 GET 返回,配置键 `webp`(直接输出 `serverconfigs.WebPImageConfig`):<br>`isPrior`、`isOn` bool<br>`minLength`、`maxLength` SizeCapacity(`{count,unit}`)<br>`mimeTypes` []string<br>`extensions` []string<br>`conds` 匹配条件对象(用户端不可编辑) | 见模块约定 | 读取 WebP 图片转换配置。 | 需登录 |
| user-10-groups | 更新 WebP 配置 | PUT | `/api/groups/{groupID}/webp` | 路径参数:`groupID`<br>JSON 请求体:`csrfToken` 必填;`isPrior`、`isOn` bool;`mimeTypes` []string;`extensions` []string | 通用 PUT 返回。<br>特有 | 错误:`400` `启用 WebP 时必须配置 MIME 类型或扩展名` | 更新 WebP 转换配置。启用时 MIME 与扩展名至少填一项。 | 需登录+CSRF写保护 |
| user-10-groups | 读取 HLS 加密配置 | GET | `/api/groups/{groupID}/hls` | 路径参数:`groupID` | 通用 GET 返回,配置键 `hls`:<br>`isPrior`、`isOn` bool(对应 `Encrypting.IsOn`)<br>`onlyURLPatterns`、`exceptURLPatterns` URLPattern[] | 见模块约定 | 读取 HLS 加密配置。 | 需登录 |
| user-10-groups | 更新 HLS 加密配置 | PUT | `/api/groups/{groupID}/hls` | 路径参数:`groupID`<br>JSON 请求体:`csrfToken` 必填;`isPrior`、`isOn`;`onlyURLPatterns`、`exceptURLPatterns` URLPattern[] | 通用 PUT 返回。<br>特有 | 错误:`400` 仅限/排除 URL 规则无效 | 更新 HLS 加密配置。 | 需登录+CSRF写保护 |
| user-10-groups | 读取内容压缩配置 | GET | `/api/groups/{groupID}/compression` | 路径参数:`groupID` | 通用 GET 返回,配置键 `compression`:<br>`isPrior`、`isOn` bool<br>`useDefaultTypes` bool<br>`types` []string(`brotli`\|`gzip`\|`zstd`\|`deflate`)<br>`decompressData` bool<br>`minLength`、`maxLength` SizeCapacity<br>`mimeTypes`、`extensions`、`exceptExtensions` []string<br>`enablePartialContent` bool | 见模块约定 | 读取内容压缩配置。 | 需登录 |
| user-10-groups | 更新内容压缩配置 | PUT | `/api/groups/{groupID}/compression` | 路径参数:`groupID`<br>JSON 请求体:`csrfToken` 必填;字段同 GET 的 `compression`。校验:`isPrior&&isOn&&!useDefaultTypes` 时 `types` 不能为空;`types` 取值见左;容量 `unit` 为 `byte`\|`kb`\|`mb`\|`gb`(空视为 byte),`count` 0-1000000;`minLength` 不得大于 `maxLength`;列表项 ≤128 字符,扩展名自动补 `.` | 通用 PUT 返回。<br>特有 | 错误:`400` 压缩类型/容量/列表校验文案 | 更新内容压缩配置。 | 需登录+CSRF写保护 |
| user-10-groups | 读取页面优化配置 | GET | `/api/groups/{groupID}/optimization` | 路径参数:`groupID` | 通用 GET 返回,配置键 `optimization`:<br>`isPrior` bool<br>`html` `{isOn,keepComments,keepConditionalComments,keepDefaultAttrVals,keepDocumentTags,keepEndTags,keepQuotes,keepWhitespace}`<br>`javascript` `{isOn,precision,version,keepVarNames}`<br>`css` `{isOn,precision,keepCSS2}` | 见模块约定 | 读取 HTML/JS/CSS 压缩优化配置。 | 需登录 |
| user-10-groups | 更新页面优化配置 | PUT | `/api/groups/{groupID}/optimization` | 路径参数:`groupID`<br>JSON 请求体:`csrfToken` 必填;字段同 GET。校验:`precision` 0-20;`javascript.version` 为 0(自动)或 2015-2024 | 通用 PUT 返回。<br>特有 | 错误:`400` 精度/版本无效 | 更新页面优化配置。 | 需登录+CSRF写保护 |
| user-10-groups | 读取 WAF 配置 | GET | `/api/groups/{groupID}/waf` | 路径参数:`groupID` | 通用 GET 返回,配置键 `waf`:<br>`isPrior`、`isOn` bool<br>`ignoreGlobalRules` bool<br>`defaultCaptcha` string(`none`\|`default`)<br>`forceCaptcha` bool<br>`mode` string(`defend`\|`observe`\|`bypass`)<br>`maxRequestBodySize` int64(字节)<br>`groups` `[{code,name,description,direction,isOn}]`,`direction` 为 `inbound`\|`outbound` | 错误:`403` `当前套餐未开通 WAF 功能`;`502` | 读取 WAF 配置及规则组开关。RPC 超时 20s。 | 需登录 |
| user-10-groups | 更新 WAF 配置 | PUT | `/api/groups/{groupID}/waf` | 路径参数:`groupID`<br>JSON 请求体:`csrfToken` 必填;`isPrior`、`isOn`、`ignoreGlobalRules`、`defaultCaptcha`、`forceCaptcha`(需 `isOn=true`)、`mode`、`maxRequestBodySize`(0 取默认 1MiB,范围 64KiB-16MiB)、`groups[{code,isOn,...}]`(`code` 必须在模板内) | 通用 PUT 返回。<br>特有 | 错误:`400` `未知的 WAF 规则组` 等;`403` `当前套餐未开通 WAF 功能`(权限不足)。 | 更新 WAF 模式、验证码与规则组开关。 | 需登录+CSRF写保护 |
| user-10-groups | 读取缓存配置 | GET | `/api/groups/{groupID}/cache` | 路径参数:`groupID` | 通用 GET 返回,配置键 `cache`:<br>`isPrior`、`isOn`<br>`addStatusHeader`、`addAgeHeader`、`enableCacheControlMaxAge`、`useDefaultPolicy`、`purgeIsOn` bool<br>`purgeKeyConfigured` bool(只读)<br>`rules` CacheRule[](高级复杂条件规则不返回) | 见模块约定 | 读取缓存配置与规则。 | 需登录 |
| user-10-groups | 更新缓存配置 | PUT | `/api/groups/{groupID}/cache` | 路径参数:`groupID`<br>JSON 请求体:`csrfToken` 必填;`isPrior`、`isOn`、`addStatusHeader`、`addAgeHeader`、`enableCacheControlMaxAge`、`useDefaultPolicy`、`purgeIsOn`;`rules` CacheRule[],最多 50 条 | 通用 PUT 返回。<br>特有 | 错误:`400` `缓存规则索引无效`(`sourceIndex` 无效)、`不能修改高级缓存规则`、规则字段校验文案 | 更新缓存规则。`sourceIndex=-1` 表示新增;高级规则 PUT 时保留不动。 | 需登录+CSRF写保护 |
| user-10-groups | 创建缓存任务(刷新/预热) | POST | `/api/groups/{groupID}/cache/tasks` | 路径参数:`groupID`<br>JSON 请求体(上限 256KiB):<br>`csrfToken` 必填<br>`type` string 必填,`purge`\|`fetch`<br>`keyType` string 必填,`key`\|`prefix`(`fetch` 只能用 `key`)<br>`keys` []string 必填,1-1000 条,每条 ≤2048,必须 http/https、有 host、无 userinfo,自动去重 | 成功 `201` `{"id":taskID,"count":有效URL数}` | 错误:`400` `请至少输入一个 URL` 等校验文案;`422` `没有匹配到当前账户的有效站点 URL`;`404`;`403`;`502` `创建缓存任务失败` | 对站点组下的 URL 发起缓存清除或预热任务,仅匹配当前账户站点的 URL。RPC 超时 20s。 | 需登录+CSRF写保护 |
| user-10-groups | 读取请求限制配置 | GET | `/api/groups/{groupID}/request-limit` | 路径参数:`groupID` | 通用 GET 返回,配置键 `requestLimit`:<br>`isPrior`、`isOn`<br>`maxConns`、`maxConnsPerIP` int<br>`outBandwidthPerConn` SizeCapacity<br>`maxBodySize` SizeCapacity | 见模块约定 | 读取并发连接、单连接带宽、请求体大小限制。 | 需登录 |
| user-10-groups | 更新请求限制配置 | PUT | `/api/groups/{groupID}/request-limit` | 路径参数:`groupID`<br>JSON 请求体:`csrfToken` 必填;`isPrior`、`isOn`;`maxConns`、`maxConnsPerIP` int 0-1000000;`outBandwidthPerConn`、`maxBodySize` SizeCapacity(`unit` 为 `byte`\|`kb`\|`mb`\|`gb`,默认 kb;`count` -1 到 1000000,-1 表示不限) | 通用 PUT 返回。<br>特有 | 错误:`400` 数值/单位无效 | 更新请求限制。 | 需登录+CSRF写保护 |
| user-10-groups | 读取 Web 策略配置 | GET | `/api/groups/{groupID}/web-policy` | 路径参数:`groupID` | 通用 GET 返回,配置键 `policy`:<br>`charset` `{isPrior,isOn,charset,isUper,force}`<br>`remoteAddr` `{isPrior,isOn,type,requestHeaderName,value,trustedProxyRanges}`<br>`stat` `{isPrior,isOn}` | 见模块约定 | 读取字符集、客户端真实 IP 获取方式、统计开关。 | 需登录 |
| user-10-groups | 更新 Web 策略配置 | PUT | `/api/groups/{groupID}/web-policy` | 路径参数:`groupID`<br>JSON 请求体:`csrfToken` 必填;`charset`、`remoteAddr`、`stat`(结构同 GET)。校验:`charset.isOn` 时 `charset` 匹配 `^[a-z0-9._-]{1,40}$`;`remoteAddr.type` 为 `default`\|`proxy`\|`requestHeader`\|`variable`;`requestHeader` 要求 `requestHeaderName` 匹配 `^[A-Za-z0-9_-]+(?:,[A-Za-z0-9_-]+)*$`;`variable` 的 `value` 长度 1-200 且含 `${` | 通用 PUT 返回。<br>特有 | 错误:`400` 字符集/来源类型/变量无效 | 更新 Web 策略。`trustedProxyRanges` 的校验规则待确认:未逐项核对。 | 需登录+CSRF写保护 |
| user-10-groups | 读取访问日志配置 | GET | `/api/groups/{groupID}/access-log` | 路径参数:`groupID` | 通用 GET 返回,配置键 `accessLog`:<br>`isPrior`、`isOn`<br>`fields` []int(仅返回用户可编辑项)<br>`status1`..`status5` bool(2xx..5xx 状态码记录开关,待确认:各自对应关系按字段名推断)<br>`enableClientClosed` bool<br>`firewallOnly` bool | 见模块约定 | 读取访问日志记录设置。 | 需登录 |
| user-10-groups | 更新访问日志配置 | PUT | `/api/groups/{groupID}/access-log` | 路径参数:`groupID`<br>JSON 请求体:`csrfToken` 必填;`isPrior`、`isOn`;`fields` []int,取值 1 请求头、2 响应头、3 参数、4 Cookie、5 扩展、6 Referer、7 UserAgent、8 请求体;`status1`..`status5`、`enableClientClosed`、`firewallOnly` bool | 通用 PUT 返回。<br>特有 | 错误:`400` `不支持的访问日志字段: N`(9 响应体等不可选) | 更新访问日志设置。PUT 时保留已有的、用户端不可编辑的字段码。 | 需登录+CSRF写保护 |
| user-10-groups | 读取 WebSocket 配置 | GET | `/api/groups/{groupID}/websocket` | 路径参数:`groupID` | 通用 GET 返回,配置键 `websocket`:<br>`isPrior`、`isOn`<br>`handshakeTimeout` `{count,unit}`<br>`allowAllOrigins` bool<br>`allowedOrigins` []string<br>`requestSameOrigin` bool<br>`requestOrigin` string | 见模块约定 | 读取 WebSocket 设置。 | 需登录 |
| user-10-groups | 更新 WebSocket 配置 | PUT | `/api/groups/{groupID}/websocket` | 路径参数:`groupID`<br>JSON 请求体:`csrfToken` 必填;字段同 GET。校验:`handshakeTimeout.unit` 为 `ms`\|`second`\|`minute`\|`hour` 且不超过 24h;`allowedOrigins` 小写、≤253 字符、匹配 `^[a-z0-9*._:-]+$`,通配符仅允许 `*`、`*.x` 或以 `:*` 结尾,最多 64 个;`isPrior&&isOn&&!allowAllOrigins` 时不能为空;`requestOrigin` ≤200 且无换行 | 通用 PUT 返回。<br>特有 | 错误:`400` 超时/来源无效 | 更新 WebSocket 配置。 | 需登录+CSRF写保护 |
| user-10-groups | 读取 HTTP 反向代理配置 | GET | `/api/groups/{groupID}/http-reverse-proxy` | 路径参数:`groupID` | 通用 GET 返回,配置键 `reverseProxy`:<br>`isPrior`、`isOn`<br>`schedulingCode`、`hashKey`、`stickyType`、`stickyParam` string<br>`primaryOrigins`、`backupOrigins` OriginConfig[]<br>`requestHostType` int、`requestHost` string、`requestHostExcludingPort` bool<br>`requestURI`、`stripPrefix` string<br>`autoFlush`、`followRedirects`、`retry50X`、`retry40X` bool<br>`addHeaders` []string<br>`connTimeout`、`readTimeout`、`idleTimeout` Duration<br>`maxConns`、`maxIdleConns` int | 见模块约定 | 读取 HTTP/HTTPS 回源配置。 | 需登录 |
| user-10-groups | 更新 HTTP 反向代理配置 | PUT | `/api/groups/{groupID}/http-reverse-proxy` | 路径参数:`groupID`<br>JSON 请求体:`csrfToken` 必填;字段同 GET。校验:`schedulingCode` 为 `random`\|`roundRobin`\|`leastConn`\|`hash`\|`sticky`;`hash` 要求 `hashKey`(≤500);`sticky` 要求 `stickyType`(`cookie`\|`header`\|`argument`)和 `stickyParam`(`^[A-Za-z0-9_-]{1,50}$`);`primaryOrigins`/`backupOrigins` 各 ≤32;`requestHostType` 0 代理服务器 Host、1 源站 Host、2 自定义;`stripPrefix` ≤500 且以 `/` 开头或含 `${`;`addHeaders` 仅允许 `X-Real-IP`、`X-Forwarded-For`、`X-Forwarded-By`、`X-Forwarded-Host`、`X-Forwarded-Proto`(空数组表示禁用);三个超时 `unit` 为 `ms`\|`second`\|`minute`\|`hour` 且 ≤7 天;`maxConns`、`maxIdleConns` 0-1000000 | 通用 PUT 返回。<br>特有 | 错误:`400` 源站/调度/超时校验文案;`502` `HTTP 回源配置未初始化` | 更新 HTTP 回源配置。RPC 超时 30s。源站校验见公共结构 OriginConfig。 | 需登录+CSRF写保护 |
| user-10-groups | 读取 TCP 反向代理配置 | GET | `/api/groups/{groupID}/tcp-reverse-proxy` | 路径参数:`groupID` | `200` `{"group":ServerGroup,"network":"tcp","reverseProxy":Layer4Proxy}`<br>额外 | 错误:`403` 套餐未开通 TCP(`server.tcp`) | 读取站点组 TCP 四层反代配置。 | 需登录 |
| user-10-groups | 更新 TCP 反向代理配置 | PUT | `/api/groups/{groupID}/tcp-reverse-proxy` | 路径参数:`groupID`<br>JSON 请求体:`csrfToken` 必填;Layer4Proxy 字段(见公共结构),`proxyProtocol` 仅 TCP 生效 | 通用 PUT 返回。<br>特有 | 错误:`400` 源站/调度校验;`403` 套餐未开通 TCP | 更新 TCP 四层反代。源站协议允许 `tcp`\|`tcp4`\|`tcp6`\|`tls`\|`tls4`\|`tls6`。 | 需登录+CSRF写保护 |
| user-10-groups | 读取 UDP 反向代理配置 | GET | `/api/groups/{groupID}/udp-reverse-proxy` | 路径参数:`groupID` | `200` `{"group":ServerGroup,"network":"udp","reverseProxy":Layer4Proxy}`<br>额外 | 错误:`403` 套餐未开通 UDP(`server.udp`) | 读取站点组 UDP 四层反代配置。 | 需登录 |
| user-10-groups | 更新 UDP 反向代理配置 | PUT | `/api/groups/{groupID}/udp-reverse-proxy` | 路径参数:`groupID`<br>JSON 请求体:`csrfToken` 必填;Layer4Proxy 字段(见公共结构);`proxyProtocol` 对 UDP 不生效 | 通用 PUT 返回。<br>特有 | 错误:`400`;`403` 套餐未开通 UDP | 更新 UDP 四层反代。源站协议仅允许 `udp`。 | 需登录+CSRF写保护 |
| user-10-groups | 读取请求/响应头配置 | GET | `/api/groups/{groupID}/headers` | 路径参数:`groupID` | 通用 GET 返回,配置键 `headers`:`{"request":HeaderPolicy,"response":HeaderPolicy}`,HeaderPolicy 见公共结构 | 见模块约定 | 读取请求头与响应头策略(含 CORS)。 | 需登录 |
| user-10-groups | 更新请求/响应头配置 | PUT | `/api/groups/{groupID}/headers` | 路径参数:`groupID`<br>JSON 请求体:`csrfToken` 必填;`request`、`response`(HeaderPolicy)。校验:`setHeaders`、`deleteHeaders` 各 ≤32;名称 ≤128 且为合法 token;禁用头 `connection`、`proxy-connection`、`keep-alive`、`transfer-encoding`、`content-length`、`upgrade`、`trailer`、`te`、`host`;`value` ≤2000 且不含 `\r`、`\n`、`\x00`;仅 `response` 处理 `cors` | 通用 PUT 返回。<br>特有 | 错误:`400` 头名称/值无效、CORS 校验文案(含 `allowOrigin=="*"` 且 `allowCredentials` 为 true) | 更新请求/响应头设置与 CORS。RPC 超时 30s。 | 需登录+CSRF写保护 |
| user-10-groups | 读取自定义页面配置 | GET | `/api/groups/{groupID}/custom-pages` | 路径参数:`groupID` | 通用 GET 返回,配置键 `customPages`:<br>`enableGlobalPages` bool<br>`pages` `[{id,status[],newStatus,bodyType,url,body}]` | 见模块约定 | 读取自定义错误页面配置。 | 需登录 |
| user-10-groups | 更新自定义页面配置 | PUT | `/api/groups/{groupID}/custom-pages` | 路径参数:`groupID`<br>JSON 请求体:`csrfToken` 必填;`enableGlobalPages` bool;`pages` 最多 20 个。每页:`id`(须属于当前站点组)、`status` []string(每项匹配 `^[1-5][0-9x]{2}$`,1-20 个)、`newStatus` int(0 或 100-599;`redirectURL` 时只能 301/302/307/308)、`bodyType` `html`\|`url`\|`redirectURL`、`url`(http/https,≤512)、`body`(`html` 时 ≤32KiB) | 通用 PUT 返回。<br>特有 | 错误:`400` 页面数量/状态码/类型/URL 无效 | 更新自定义错误页面。 | 需登录+CSRF写保护 |


### 公共结构

### ServerGroup(`group`)
- `id` int64、`name` string、`userId` int64、`isOn` bool(均 omitempty)

### URLPattern(`onlyURLPatterns`/`exceptURLPatterns` 元素)
- `type` string:`wildcard`\|`regexp`\|`images`\|`audios`\|`videos`
- `pattern` string:`wildcard`/`regexp` 必填且 ≤2000;其他类型会被清空
- 每个列表最多 100 条,按 type+pattern 去重

### SizeCapacity / Duration
- 容量:`{count int64, unit string}`,`unit` 为 `byte`\|`kb`\|`mb`\|`gb`
- 时长:`{count int64, unit string}`,`unit` 为 `ms`\|`second`\|`minute`\|`hour`(缓存规则 `life` 单位为 `minute`\|`hour`\|`day`\|`week`)

### CCThreshold
- `name` string ≤100、`isOn` bool、`priority` int
- `path` string、`conditions` `{ips,domains,fingerprints}`
- `groupBy` `""`\|`ip`\|`subnet`\|`domain`\|`uri`\|`fingerprint`
- `periodSeconds` int、`maxRequests` int、`blockSeconds` int、`statusCode` int、`perURI` bool
- `methods` []string,≤16,取值 `GET`、`HEAD`、`POST`、`PUT`、`PATCH`、`DELETE`、`OPTIONS`、`CONNECT`、`TRACE`(自动转大写去重)
- 待确认:`periodSeconds`/`maxRequests`/`blockSeconds`/`statusCode` 的数值范围未逐项核对。

### CacheRule
- `sourceIndex` int(-1 新增)、`isOn`、`isReverse`
- `matchType`:`extension`\|`all`\|`index`\|`prefix`\|`exact`\|`wildcard`\|`regexp`
- `values` []string、`isCaseInsensitive` bool
- `life` `{count,unit}`:`count` 1-525600,`isReverse` 时忽略
- `ignoreQuery` bool、`status` []int(100-599)
- `minSize`、`maxSize` SizeCapacity
- `skipCacheControlValues` []string、`skipSetCookie`、`enableRequestCachePragma`、`allowChunkedEncoding`、`allowPartialContent`、`forcePartialContent`、`alwaysForwardRangeRequest`、`enableIfNoneMatch`、`enableIfModifiedSince`、`enableReadingOriginAsync` bool

### OriginConfig(`primaryOrigins`/`backupOrigins` 元素)
- `id`、`isOn`、`name`、`description`(≤200 字符)、`protocol`、`host`(≤253)、`portRange`、`weight`(夹到 0-1000)、`domains`(≤32)
- `requestHost`、`followPort`、`http2Enabled`、`readOnly`、`hasCertificate`
- HTTP 回源 `protocol`:`http`\|`http4`\|`http6`\|`https`\|`https4`\|`https6`
- `portRange` 为单个端口或范围,范围跨度不超过 1000
- 源站 `id` 必须属于当前配置且不能重复

### Layer4Proxy(`reverseProxy`,TCP/UDP)
- `isPrior`、`isOn`
- `schedulingCode`:`random`\|`roundRobin`\|`leastCon`(源码取值拼写如此)
- `primaryOrigins`、`backupOrigins` OriginConfig[]
- `connTimeout`、`readTimeout`、`idleTimeout` Duration
- `maxConns`、`maxIdleConns` int
- `proxyProtocol` `{isOn,version}`,`version` 为 1 或 2,仅 TCP 生效

### HeaderPolicy(`headers.request` / `headers.response`)
- `isPrior`、`isOn`
- `setHeaders` `[{id,name,value}]`、`deleteHeaders` []string
- `cors`(仅 response):`isOn`、`allowMethods`([]string,取值 `GET`、`HEAD`、`POST`、`PUT`、`PATCH`、`DELETE`、`OPTIONS`)、`allowOrigin`(≤500,`isOn` 时必填)、`allowCredentials` bool、`exposeHeaders`(≤32)、`maxAge`(0-86400)、`requestHeaders`(≤32)、`optionsMethodOnly` bool

<a id="m25"></a>

## 25. 用户端 · 四层转发与 AI(用户端)

**模块约定**

> 路由总数 16,来源 EdgeUserPortal/internal/web/server.go
说明:
- 所有错误统一返回 `{"error":"消息"}`;未登录统一 `401` `请先登录`。
- "需登录+CSRF写保护" = 路由包了 `sensitiveUserWrite`(未登录 401;管理员协助会话 403 `管理员协助会话不能修改用户资源或安全配置`;平台授权失效 403 只读提示),并且处理函数内用 `validCSRF` 比对请求体字段 `csrfToken`(不匹配 403 `CSRF 校验失败`)。CSRF 令牌放在 JSON 请求体,不是请求头。
- 请求体使用 `decodeAPIJSON`:上限 64KiB、禁止未知字段、只允许单个 JSON 对象。
- 四层功能受套餐门控:`/api/layer4` 系列 GET 与 export 要求套餐含 `server.tcp` 或 `server.udp` 任一功能,否则 `403` `当前套餐未开通四层转发功能`;创建/更新项目按 `network` 要求对应的 `server.tcp` / `server.udp`,否则 `403` `当前套餐未开通X功能`;套餐功能读取失败 `502` `读取套餐功能失败`。
- `POST/PATCH/DELETE /api/layer4/projects*` 与 `POST /api/layer4/projects/{id}/status` 在请求体解码失败时返回 `400` `请求格式无效`(提交 eaa88f1 前曾直接 return,客户端收到空 200,已修复)。
- "公共结构"见表后。

| 模块 | 接口名称 | 方法 | 路径 | 请求参数 | 响应数据 | 主要错误 | 功能说明 | 认证/权限 |
|---|---|---|---|---|---|---|---|---|
| user-11-layer4-ai | 四层转发概览 | GET | `/api/layer4` | 查询参数:<br>`protocol` string 可选,`tcp`\|`udp`<br>`port` int 可选,1000-40000<br>`range` string 可选,`24h`\|`7d`\|`30d`,默认 `24h` | `200` `apiLayer4Response`:<br>`range` string<br>`rules` array(见公共结构 Layer4Rule)<br>`trends` array(见 Layer4Trend)<br>`metricsAvailable` bool<br>`metricsMessage` string<br>`totalConnections` int64<br>`totalTrafficInBytes` int64<br>`totalTraficOutBytes` int64(源码字段名拼写即为 `Trafic`)<br>`healthyOrigins` int<br>`unhealthyOrigins` int | 错误:`403` 套餐未开通四层;`502` `读取四层转发数据失败`(所有收集错误统一映射为该文案) | 汇总站点分组的 TCP/UDP 反代与独立四层项目,返回规则列表、连接/流量统计和趋势。独立项目 `groupId=0`、`groupName="独立转发"`。 | 需登录 |
| user-11-layer4-ai | 四层项目列表 | GET | `/api/layer4/projects` | 无 | `200` `{"projects":[Layer4Project...]}`(结构见公共结构) | 错误:`403` 套餐未开通四层;`502` | 列出当前用户的独立四层转发项目。 | 需登录 |
| user-11-layer4-ai | 四层项目详情 | GET | `/api/layer4/projects/{id}` | 路径参数:`id` int 必填(项目编号) | `200` Layer4Project 对象 | 错误:`400` `项目编号无效`;`404` `四层项目不存在`;`502` | 读取单个独立四层项目。 | 需登录 |
| user-11-layer4-ai | 四层项目诊断 | GET | `/api/layer4/projects/{id}/diagnostics` | 路径参数:`id` int 必填 | `200` 原样输出 RPC 返回的 JSON(`Cache-Control: no-store`)。参考 `serverconfigs.Layer4Diagnostics`:`unavailable`、`collectedAt`、`startedAt`、`active`、`accepted`、`trafficIn`、`trafficOut`、`reasons`(map[string]int64);待确认:RPC 是否直接返回该结构,源码此处按 `json.RawMessage` 透传 | 错误:`400` `项目编号无效`;`503` `当前版本不支持四层诊断`;`403` `无法读取该项目诊断，请确认权限和节点状态`(RPC 任意错误均映射为 403) | 读取项目的运行时诊断信息(活动连接、接入数、流量、拒绝原因分布)。 | 需登录 |
| user-11-layer4-ai | 创建四层项目 | POST | `/api/layer4/projects` | JSON 请求体:<br>`csrfToken` string 必填<br>`name` string 必填,1-200<br>`network` string 必填,`tcp`\|`udp`<br>`port` int 必填,10000-40000<br>`userPlanId` int<br>`origins` array 必填,1-32 项,每项 `id`、`name`、`host` 必填、`port` 1-65535、`protocol` `tcp`\|`udp`\|`tls`、`enabled` bool、`backup` bool<br>`limits` object,`maxConnectionsPerIP` 0-100000、`newConnectionsPerSecond` 0-100000、`packetsPerSecond` 0-1000000<br>`schedulingCode` string<br>`connTimeout`/`readTimeout`/`idleTimeout` apiTimeDuration<br>`maxConns`/`maxIdleConns` int<br>`proxyProtocol` object | 成功 `201` 返回完整 Layer4Project;创建后查询不到时降级为 `{"id":id}`。 | 错误:`400`(经 `localizeLayer4Error` 本地化的校验文案);`403` CSRF/套餐未开通对应 tcp/udp 功能;`503` `当前 API 不支持项目防护设置`(`limits` 非空但 RPC 不支持)。<br>请求体解码失败时无响应(见说明)。 | 创建独立四层转发项目,端口由用户在 10000-40000 范围内指定。需要套餐含对应协议功能。 | 需登录+CSRF写保护 |
| user-11-layer4-ai | 更新四层项目 | PATCH | `/api/layer4/projects/{id}` | 路径参数:`id` int 必填<br>JSON 请求体:同创建项目(`csrfToken`、`name`、`network`、`port`、`userPlanId`、`origins`、`limits`、`schedulingCode`、`connTimeout`、`readTimeout`、`idleTimeout`、`maxConns`、`maxIdleConns`、`proxyProtocol`) | 成功 `200` `{"ok":true}`。 | 错误:`400` `项目编号无效` / `至少需要一个主源站` / 其他校验文案;`403`;`404`;`503` `当前 API 不支持项目防护设置`。<br>请求体解码失败时无响应。 | 全量更新项目配置。`origins` 中至少要有一个非备用(`backup=false`)的主源站。 | 需登录+CSRF写保护 |
| user-11-layer4-ai | 删除四层项目 | DELETE | `/api/layer4/projects/{id}` | 路径参数:`id` int 必填<br>JSON 请求体:`csrfToken` string 必填 | 成功 `200` `{"ok":true,"auditLogged":bool}`。 | 错误:`400` `项目编号无效`;`403`;`404`。<br>请求体解码失败时无响应。 | 删除项目。风险操作,不可恢复;`auditLogged` 表示审计日志是否写入成功。 | 需登录+CSRF写保护 |
| user-11-layer4-ai | 启停四层项目 | POST | `/api/layer4/projects/{id}/status` | 路径参数:`id` int 必填<br>JSON 请求体:`csrfToken` string 必填;`isOn` bool(true 启用,false 停用) | 成功 `200` `{"ok":true}`。 | 错误:`400` `项目编号无效`;`403`。<br>请求体解码失败时无响应。 | 切换项目启用状态。 | 需登录+CSRF写保护 |
| user-11-layer4-ai | 导出四层规则 CSV | GET | `/api/layer4/export` | 查询参数同 `/api/layer4`:`protocol`、`port`、`range` | `200` `text/csv`,下载文件名 `edgeroute-layer4-rules.csv`,列:分组、协议、源站地址、端口、状态、健康状态。 | 错误:`403` 套餐未开通四层;`502` `读取四层转发数据失败` | 按当前筛选条件导出四层规则列表。 | 需登录 |
| user-11-layer4-ai | 批量启停四层规则 | POST | `/api/layer4/batch` | JSON 请求体:<br>`csrfToken` string 必填<br>`action` string 必填,`enable`\|`disable`<br>`rules` array 必填,1-50 项,每项 `groupId` int、`network` `tcp`\|`udp`、`originId` int | `200` `{"ok":bool,"failed":[{"groupId","network","originId","error"}]}`。部分失败也返回 200,以 `ok=false` 且 `failed` 非空表示。 | 错误:`400`;`403`。 | 批量启用/停用站点分组下的 TCP/UDP 源站规则,逐条执行并收集失败原因。 | 需登录+CSRF写保护 |
| user-11-layer4-ai | 读取 AI 配置与状态 | GET | `/api/ai` | 无 | `200`:<br>`config` `{defenseMode,optimizationMode,updatedAt}`<br>`state` UserState(摘要:`status`、`lastDefenseAnalysisAt`、`lastOptimizationAt`、`lastError`、`metrics`、`siteMetrics`、`recommendations`、`changes`、`usageDay`、`requestsToday`、`siteRequestsToday`、`nextDefenseAnalysisAt`、`nextOptimizationAt`、`analysisRuns`、`manualTask`、`siteHealth`、`leases`、`autoWafRecoveries`)<br>`defenseAvailable` bool<br>`optimizationAvailable` bool<br>`autoAplyAvailable` bool(源码字段名拼写即为 `Aply`)<br>`autoExecutionEnabled` bool<br>`siteDomains` map[siteId]域名 | 错误:`403` `AI 功能未授权或暂不可用` / `AI 功能授权校验失败`;`502` | 读取当前用户的 AI 防御/优化模式、分析状态、建议和变更记录。 | 需登录 |
| user-11-layer4-ai | 更新 AI 模式 | PUT | `/api/ai` | JSON 请求体:<br>`csrfToken` string 必填<br>`defenseMode` string,`off`\|`analyze`\|`review`\|`auto`,空视为 `off`<br>`optimizationMode` string,取值同上 | 成功 `200` `{"ok":true}`。 | 错误:`403`;`422`(直接输出 RPC 返回的错误信息);`400` 模式取值无效(待确认:具体文案)。 | 设置 AI 防御与优化的工作模式。 | 需登录+CSRF写保护 |
| user-11-layer4-ai | 触发 AI 分析 | POST | `/api/ai/analyze` | JSON 请求体:<br>`csrfToken` string 必填<br>`category` string 必填,`defense`\|`optimization`<br>`serverId` int,≥0(0 表示全部站点)<br>`windowDays` int,`0`\|`1`\|`7`\|`30` | 成功 `202` `{"ok":true}`(异步执行)。 | 错误:`400` 参数无效;`403`;`422`(RPC 错误信息)。 | 手动发起一次 AI 分析任务,结果通过 `GET /api/ai` 的 `state.manualTask` / `recommendations` 获取。 | 需登录+CSRF写保护 |
| user-11-layer4-ai | 应用 AI 建议 | POST | `/api/ai/recommendations/{recommendationID}/apply` | 路径参数:`recommendationID` string 必填,匹配 `[A-Za-z0-9-]` 且 ≤128<br>JSON 请求体:`csrfToken` string 必填 | 成功 `200` `{"ok":true}`。 | 错误:`400` `AI 记录标识无效`;`403`;`422`(RPC 失败文案)。 | 采纳并执行一条 AI 建议,生成对应变更记录。与 reject、rollback 共用 `handleAIAction`。 | 需登录+CSRF写保护 |
| user-11-layer4-ai | 拒绝 AI 建议 | POST | `/api/ai/recommendations/{recommendationID}/reject` | 路径参数同上<br>JSON 请求体:`csrfToken` string 必填 | 成功 `200` `{"ok":true}`。<br>错误同上。 | 见模块约定 | 忽略一条 AI 建议。 | 需登录+CSRF写保护 |
| user-11-layer4-ai | 回滚 AI 变更 | POST | `/api/ai/changes/{changeID}/rollback` | 路径参数:`changeID` string 必填,匹配 `[A-Za-z0-9-]` 且 ≤128<br>JSON 请求体:`csrfToken` string 必填 | 成功 `200` `{"ok":true}`。 | 错误:`400` `AI 记录标识无效`;`403`;`422`。 | 回滚一条已执行的 AI 变更。 | 需登录+CSRF写保护 |


### 公共结构

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

<a id="m26"></a>

## 26. 用户端 · 系统与运维(用户端)

**模块约定**

> 路由总数 7,来源 EdgeUserPortal/internal/web/server.go
说明:
- 错误统一返回 `{"error":"消息"}`(`writeAPIError`),但 `/health`、`/healthz`、`/readyz`、`/metrics` 为运维探针,返回纯文本。
- 本模块 7 条路由均不要求登录,也不经过 `sensitiveUserWrite`。
- 路由清单(JSON)中 `/readyz`、`/metrics` 的 `handler` 为空,实际分别由 `observability.ReadinessHandler`、`s.metrics.Handler()` 处理(见 `server.go` 路由注册)。

| 模块 | 接口名称 | 方法 | 路径 | 请求参数 | 响应数据 | 主要错误 | 功能说明 | 认证/权限 |
|---|---|---|---|---|---|---|---|---|
| user-12-system | 存活探针 | GET | `/health` | 无 | 成功 `200`,`text/plain`,正文 `ok`。<br>失败 `503`,正文 `unavailable`,响应头 `Retry-After: 5`。<br>响应头固定带 `Cache-Control: no-store`、`X-EdgeRoute-Version`、`X-EdgeRoute-API-Version`。 | 见模块约定 | 调用 `s.rpc.Health` 检查上游 API RPC 是否可用,超时 3 秒(`healthCheckTimeout`)。与 `/healthz` 共用 `health` 处理函数。 | 公开 |
| user-12-system | 存活探针(别名) | GET | `/healthz` | 无 | 同 `/health`。 | 见模块约定 | 与 `/health` 完全相同,供 Kubernetes 等探针习惯路径使用。 | 公开 |
| user-12-system | 就绪探针 | GET | `/readyz` | 无 | 全部检查通过 `200`,`text/plain`,正文 `ok\n`。<br>有失败项 `503`,正文 `unavailable: <失败项名,逗号分隔>\n`,带 `Retry-After: 5`。<br>响应头 `Cache-Control: no-store`、`X-Content-Type-Options: nosniff`。 | 见模块约定 | 并发执行就绪检查,目前只有一项 `api_rpc`(即 `s.rpc.Health`),超时 3 秒。 | 公开 |
| user-12-system | Prometheus 指标 | GET | `/metrics` | 请求头 `Authorization: Bearer <token>`(仅当服务端配置了 metrics token 时需要) | Prometheus/OpenMetrics 文本格式。<br>配置了 token 但缺失或 | 错误:`401`,响应头 `WWW-Authenticate: Bearer realm="metrics"`。<br>未配置 token:非回环地址直连,或带有 `X-Forwarded-For`/`X-Forwarded-Host`/`X-Real-Ip`/`Forwarded` 头时返回 `404`。 | 暴露进程指标。未配置 token 时仅允许本机回环地址直接访问,避免经反向代理泄露。 | 公开路由,但受 Bearer token 或"仅回环直连"保护 |
| user-12-system | 用户端 OpenAPI 文档 | GET | `/api/openapi.json` | 无 | 成功 `200`,`application/json`,响应头 `Cache-Control: public, max-age=300`。<br>正文为 OpenAPI `3.0.3` 文档:`info.title="EdgeRoute User API"`;`servers[0].url` 取自 `RefreshPublicBaseURL`,为空时为 `/`;安全方案 `cookieAuth`(Cookie 名 `edgeroute_user_session`);`paths` 为用户侧接口的静态子集(约 14 个路径示例,不是全部接口)。<br>失败 `500`,纯文本 `生成 API 文档失败`。 | 见模块约定 | 返回用户侧 API 的精简 OpenAPI 描述,供开发者查阅。 | 公开 |
| user-12-system | 上报 Web Vitals | POST | `/api/metrics/web-vitals` | JSON 请求体:<br>`metrics` array 必填,1-50 条,每项 `name` string、`value` number、`id` string、`delta` number、`rating` string、`navigationType` string、`url` string、`userAgent` string、`timestamp` number<br>`url` string 必填<br>`userAgent` string<br>`timestamp` number<br>请求体上限 64KiB,禁止未知字段。 | 成功 `204`,无正文。 | 错误:`400` `请求格式无效` / 参数校验失败;`429` `{"error":"上报过于频繁，请稍后再试"}` + `Retry-After: 60`。 | 前端上报核心 Web 指标。仅 `LCP`、`FCP`、`CLS`、`INP`、`TTFB`、`FID` 会入库,其余忽略;RPC 写入失败只记日志,仍返回 `204`。按客户端 IP 限流,每分钟 60 次,限流在解析请求体之前执行。 | 公开(匿名可访问;已登录时记录用户 ID,未登录记为 0),按 IP 限流 |
| user-12-system | 上报页面性能时间线 | POST | `/api/metrics/performance-timeline` | JSON 请求体:<br>`navigation` object 必填,其中 `url` string 必填,其余 `userAgent`、`timestamp`、`dnsLookup`、`tcpConnect`、`sslHandshake`、`requestTime`、`responseTime`、`ttfb`、`domProcessing`、`domInteractive`、`domComplete`、`loadComplete`、`totalTime`、`navigationType`<br>`resources` array,最多 500 条,每项 `url`、`pageUrl`、`userAgent`、`timestamp`、`name`、`initiatorType`、`duration`、`transferSize`、`encodedBodySize`、`decodedBodySize`、`cacheStatus`<br>请求体上限 64KiB,禁止未知字段。 | 成功 `204`,无正文。 | 错误:`400` `请求格式无效` / 参数校验失败;`429` 同上。 | 前端上报页面导航耗时与资源加载明细。资源只有耗时超过 100ms(`SlowResourceThresholdMs`)才会入库;RPC 写入失败只记日志,仍返回 `204`。按客户端 IP 限流,每分钟 60 次。 | 公开(匿名可访问;已登录时记录用户 ID),按 IP 限流 |


### 公共结构与补充说明

- 限流(`rateLimitMetrics`):按客户端 IP 固定窗口,每分钟 60 次,最多跟踪 100000 个 IP(`MaxKeys`);超限返回 `429`、`Retry-After: 60`、`{"error":"上报过于频繁，请稍后再试"}`。
- 探针超时:`healthCheckTimeout = 3s`。
- 上报限额常量(`EdgeCommon/pkg/frontendmetrics`):`MaxMetricsPerRequest=50`、`MaxResourcesPerRequest=500`、`SlowResourceThresholdMs=100`。
- 待确认:`metrics` 与 `navigation` 内各数值字段的具体单位(毫秒/字节),本模块未逐一核对,请以前端上报代码为准。

### 未计入 7 条清单的相关路由(仅供参考)

以下路由在 `server.go` 中注册,但不在本模块路由清单 JSON 中,不计入"路由总数":
- `/assets/`、`/landing/`:静态资源,`Cache-Control: public, max-age=31536000, immutable`,`Vary: Accept-Encoding`,支持 gzip,公开。
- `GET /`:SPA 兜底,返回前端 `index.html`(`200 text/html`,`Cache-Control: no-store`);命中敏感 SPA 路径返回 `404`;`/` 在门户关闭时 `302` 到 `/login`。公开。

<a id="pending"></a>

## 附录 A 待确认事项（涉及 212 条路由）

先列共性问题，实现前应优先拿到答案：

1. 管理端 `this.Fail` 返回时 HTTP 状态码是 200 还是 400，未实测；目前新前端按 `body.code !== 200` 判定失败。
2. 多处错误文案标注“待确认”，不要依赖具体文案做分支判断。
3. 若干响应里的数组元素结构、枚举取值（如 `status`、`issues[].code`）源码未读全，下表逐条列出。
4. 用户端订单、退款等状态枚举的真实取值需对照后端常量确认。

| 模块 | 方法 | 路径 | 待确认内容 |
|---|---|---|---|
| 认证与会话(管理端) | GET | `/api/admin/node-join/installer.py` | 脚本具体内容由 `nodeinstall.RenderScript` 渲染,待确认(未读到该函数源码) |
| 认证与会话(管理端) | POST | `/api/admin/node-join` | `PayloadJSON` 的具体字段由控制面决定,可参考 `nodeenrollment.Result`:items/tokens/record/config/manifest/publicKey/size/sha256,实际返回哪些待确认 |
| 认证与会话(管理端) | GET | `/api/admin/session` | `data`:`authenticated`(bool)、`setupRequired`(bool,系统是否待初始化)、`csrfToken`(string,供后续 POST 使用)、`loginToken`(string,供 `login` 使用,有效期 30 分钟)、`systemName`、`productNam |
| 认证与会话(管理端) | GET | `/api/admin/dashboard` | 流量:`todayTrafficBytes`、`todayCountIPs`、`todayRequests`、`accounting`(计费统计,结构待确认)<br>`diskUsageWarning`(磁盘告警,结构待确认) |
| 认证与会话(管理端) | GET | `/api/admin/security-events/notifications` | `data`:`status`(`available`/`unavailable`)、`enabled`(bool)、`minSeverity`(low/medium/high)、`minCount`、`cooldownMinutes`、`updatedAt`、`recipientScope`(取值含义待确认) |
| 认证与会话(管理端) | GET | `/api/admin/security/tls-baseline` | `issues[].code` 的取值枚举待确认 |
| 网站与网站分组(管理端) | GET | `/api/admin/sites/detail` | `cluster/owner/groups/domains` 的元素字段待确认(未逐项核对) |
| 网站与网站分组(管理端) | GET | `/api/admin/sites/settings` | `web`:`compression,cache,websocket,accessLog`,另有 `waf,headers,requests,delivery,content,auth,fastcgi,locations,rewrites,scripts,redirects`(各子结构键以源码为准,部分待确认) |
| 网站与网站分组(管理端) | POST | `/api/admin/sites/create` | 域名/源站的具体校验由 RPC 负责(待确认) |
| 网站与网站分组(管理端) | POST | `/api/admin/sites/settings/websocket` | 错误:握手超时单位无效/时间无效;来源格式无效(具体文案待确认) |
| 网站与网站分组(管理端) | POST | `/api/admin/sites/settings/request-limit` | 错误:"最大并发连接数必须在 0 至 1000000 之间"/"单 IP 并发连接数…"/"单连接下行带宽单位无效/超出允许范围"/"请求体上限…"(后三者完整文案待确认) |
| 网站与网站分组(管理端) | POST | `/api/admin/sites/settings/root` | `dir` ≤2048;`indexes` ≤64;`stripPrefix` 须以 `/` 开头(`dir` 的字符限制待确认) |
| 网站与网站分组(管理端) | POST | `/api/admin/sites/settings/auth/policies/create` | signURL:`secret,secondarySecret,clearSecondarySecret,algorithm,signParam,timestampParam,life,keepParams,mode,bindClientIP,bindQuery`(signURL 各字段取值待确认) |
| 网站与网站分组(管理端) | POST | `/api/admin/sites/settings/fastcgi/create` | `poolSize` 0–1000000;每作用域 ≤32(`readTimeout/connTimeout` 结构待确认) |
| 网站与网站分组(管理端) | POST | `/api/admin/sites/settings/rewrites/sort` | 错误:排序不完整/重复/不属于当前作用域(具体文案待确认) |
| 网站与网站分组(管理端) | POST | `/api/admin/sites/delete` | 是否软删除、是否可恢复由 RPC 决定(待确认),前端应二次确认 |
| 网站与网站分组(管理端) | POST | `/api/admin/server-groups/delete` | 分组内网站的处理由 RPC 决定(待确认) |
| 节点与集群(管理端) | GET | `/api/admin/user-nodes/form` | `certificates` 为可用的非 CA 证书,最多取前 2000 条,`expiresTime` 由 `formatUnixDate` 格式化(具体格式待确认) |
| 节点与集群(管理端) | GET | `/api/admin/user-nodes/install` | YAML 内部字段:待确认(未逐项核对) |
| 节点与集群(管理端) | GET | `/api/admin/user-nodes/logs` | `NodeLogsResult` + `tags`(常用标签列表,元素结构待确认) |
| 节点与集群(管理端) | POST | `/api/admin/user-nodes/toggle` | 疑似缺陷:操作日志复用"更新用户节点"文案,无单独的启停日志(待确认意图) |
| 节点与集群(管理端) | POST | `/api/admin/database-nodes/status` | `{"status":{"isOk":bool,"error":string,"size":int64,"version":string}}`(`size`、`version` 的类型与单位:待确认;RPC 未返回状态对象时 `status` 缺省的行为待确认) |
| 节点与集群(管理端) | GET | `/api/admin/nodes/detail` | `actionParams/actionDefinitions/actionOperators/thresholdItems/thresholdActions` 来自 `nodeconfigs.FindAll…` 定义列表,元素字段:待确认 |
| 节点与集群(管理端) | POST | `/api/admin/nodes/cache-diagnostics` | `diagnostics` 是节点回传的任意 JSON,结构待确认 |
| 节点与集群(管理端) | POST | `/api/admin/nodes/origin-health` | `health` 结构待确认 |
| 节点与集群(管理端) | POST | `/api/admin/nodes/actions/create` | 错误:F:节点校验错误及 JSON 校验消息(具体文案待确认,出自 `validateNodeActionJSON`);A:RPC 失败 |
| 节点与集群(管理端) | POST | `/api/admin/nodes/actions/sort` | 是否要求列出节点全部动作:待确认 |
| 节点与集群(管理端) | POST | `/api/admin/nodes/login` | 错误:F:"invalid SSH host"/"invalid SSH port"及授权不存在消息(来自 `findAdminNodeGrant`,文案待确认);A:RPC 失败 |
| 节点与集群(管理端) | POST | `/api/admin/nodes/dns` | `routes` []string,可选(元素格式待确认,与 DNS 修复接口的 `code@domainId` 不同,此处原样透传) |
| 节点与集群(管理端) | POST | `/api/admin/nodes/ip/update` | 是否校验该 IP 属于此节点:待确认,代码只校验节点存在 |
| 节点与集群(管理端) | POST | `/api/admin/nodes/ip/delete` | 同样未校验该 IP 属于此节点(待确认) |
| 节点与集群(管理端) | POST | `/api/admin/nodes/advanced` | `bypassMobile` int32(注意是整数,不是布尔,取值含义待确认)<br>缓存容量单位与 `…SubDirsJSON` 的校验在处理器内没有,依赖后端 RPC(待确认) |
| 节点与集群(管理端) | GET | `/api/admin/nodes/overview` | `accounting` 为 `trafficstats.Describe` 的输出,字段:待确认 |
| 节点与集群(管理端) | POST | `/api/admin/nodes/monitor-settings/update` | 同一份配置的 `dnsOverload` 也会被一并校验,若库中已有非法值会让本次保存失败(待确认,属于代码推断) |
| 节点与集群(管理端) | GET | `/api/admin/clusters/form` | `offset` 类型:待确认 |
| 节点与集群(管理端) | GET | `/api/admin/clusters/detail` | `{"cluster":{"id","name","isOn","isGlobal"(待确认),"installDir","timeZone"(待确认),"dns":{"domainName","nodesAutoSync","serversAutoSync","ttl"},"countNodes","countAct<br>错误:F/A:参数无效或集群不存在(消息文案待确认)<br>`cluster` 上除上述键外是否还有其他键:待确认(只核对了主干键) |
| 节点与集群(管理端) | GET | `/api/admin/clusters/policies` | `useGlobal=true` 表示该项继承全局集群配置,写入该项策略时不会落到本集群(规则见 `policyutils.Resolve`,细节待确认) |
| 节点与集群(管理端) | GET | `/api/admin/clusters/operations` | `definitions/operators/types/levels` 元素字段:待确认<br>错误:参数无效或集群不存在(文案待确认) |
| 节点与集群(管理端) | POST | `/api/admin/clusters/create` | 处理器自身不校验任何字段,必填性与取值范围全由 RPC 层决定(具体规则待确认) |
| 节点与集群(管理端) | POST | `/api/admin/clusters/policies/health` | 错误:F:"health check status codes must be between 100 and 599"及 `HealthCheckConfig.Init()` 的校验消息(文案待确认);A:RPC 失败<br>该接口不会套用默认值:漏传的数值按 0 提交,由 `Init()` 决定是否补默认或报错(待确认) |
| 节点与集群(管理端) | POST | `/api/admin/clusters/policies/http3` | 错误:F:"HTTP/3 port must be between 1 and 65535"及 `HTTP3Policy.Init()` 的消息(待确认);A:RPC 失败 |
| 节点与集群后半部分(管理端) | POST | `/api/admin/clusters/policies/system-service/check` | `results`:向集群节点下发检查消息(超时 10 秒)的聚合结果,元素结构待确认 |
| 节点与集群后半部分(管理端) | POST | `/api/admin/clusters/policies/cc` | 阈值元素字段:待确认(见 `serverconfigs.HTTPCCThreshold`) |
| 节点与集群后半部分(管理端) | POST | `/api/admin/clusters/policies/server-baseline` | 错误(原文):`invalid unmatched-domain action`;`invalid default domain`;`X-Forwarded-For address limit must be between 0 and 100`;`TCP port range must be ordered and  |
| 节点与集群后半部分(管理端) | GET | `/api/admin/clusters/origin-protect/export` | `ipv4`/`ipv6`/`skipped`/`nginx`/`iptables` 元素结构来自 `BuildOriginProtectRules`,待确认<br>错误:`invalid cluster id`,及端口数量/范围类错误原文待确认 |
| 节点与集群后半部分(管理端) | POST | `/api/admin/clusters/operations/firewall/save` | 错误:`invalid firewall action metadata`;`firewall action limit reached`(每集群 100 条);`unsupported firewall action type`;`IPSet names may only contain letters, numbe |
| 节点与集群后半部分(管理端) | POST | `/api/admin/nodes/create-batch` | 错误:分组/区域/授权不可用类提示原文待确认 |
| 节点与集群后半部分(管理端) | POST | `/api/admin/nodes/enrollment` | `count`/`level` 范围及 `state`/`phase` 枚举由 RPC 侧决定,待确认 |
| 节点与集群后半部分(管理端) | GET | `/api/admin/node-rollouts` | `clusterId` 必填性待确认 |
| 节点与集群后半部分(管理端) | POST | `/api/admin/node-rollouts/create` | `taskType` 取值待确认 |
| 节点与集群后半部分(管理端) | POST | `/api/admin/nodes/upgrade` | 错误:节点不属于集群/无效 ID 类提示原文待确认 |
| 节点与集群后半部分(管理端) | GET | `/api/admin/node-groups` | 错误:集群不存在/无效 ID 原文待确认 |
| 节点与集群后半部分(管理端) | POST | `/api/admin/node-groups/sort` | 错误:提交 ID 与现有集合不一致的提示原文待确认 |
| 节点与集群后半部分(管理端) | POST | `/api/admin/node-regions/create` | 错误:名称必填/超长/国家码/经纬度越界/含控制字符(`\n`、`\t` 例外)类提示,原文待确认 |
| 节点与集群后半部分(管理端) | POST | `/api/admin/node-regions/sort` | 错误:提交 ID 与现有集合不一致的提示原文待确认 |
| 节点与集群后半部分(管理端) | POST | `/api/admin/node-regions/move-node` | 错误:`node region is disabled`;节点/区域 ID 无效提示原文待确认 |
| 节点与集群后半部分(管理端) | GET | `/api/admin/node-logs` | `items[{id,tag,description,level,createdAt,count,isFixed,isRead,canFix,node{id,name,cluster{id,name}},server{id,name}或null}]`;`total`;`offset`;`size`;`countUnre<br>错误:`invalid node log type`;分页/日期/级别/标签/关键词越界提示原文待确认<br>`dayFrom`/`dayTo` string,日期,dayFrom 不得晚于 dayTo(格式待确认,疑为 YYYY-MM-DD)<br>`createdAt` 为 pb 原值,格式待确认 |
| 节点与集群后半部分(管理端) | GET | `/api/admin/node-grants/detail` | 错误:授权不存在提示原文待确认 |
| 节点与集群后半部分(管理端) | POST | `/api/admin/node-grants/create` | 错误:校验全部交给 RPC,取值范围待确认 |
| 节点与集群后半部分(管理端) | POST | `/api/admin/node-grants/update` | 错误:同创建,待确认<br>更新 SSH 授权;密码/私钥留空时是否保留旧值取决于 RPC,待确认;记录日志 |
| DNS 管理(管理端) | POST | `/api/admin/dns/clusters/update` | 错误:F:`invalid cluster id`;A:RPC 失败(名称/域名合法性由后端校验,具体文案待确认);403 CSRF 失败(纯文本)<br>`ttl` int32,可选,0 表示默认(后端取值规则待确认) |
| DNS 管理(管理端) | GET | `/api/admin/dns/providers` | 查询带上当前管理员 ID(`AdminId`),是否按管理员做可见性过滤由后端决定(待确认) |
| DNS 管理(管理端) | GET | `/api/admin/dns/providers/form` | 编辑:再加 `"provider":{"id":int64,"name":string,"type":string,"typeName":string,"minTTL":int32,"params":object}`,`params` 为后端脱敏后的 API 参数 JSON 对象(键集因服务商类型而异,脱敏规则待确认) |
| DNS 管理(管理端) | POST | `/api/admin/dns/providers/create` | 错误:A:RPC 失败(名称/类型/参数合法性全由后端校验,具体文案待确认);403 CSRF 失败 |
| DNS 管理(管理端) | POST | `/api/admin/dns/providers/update` | `paramsJSON` 为空时后端是保留旧值还是清空:待确认,处理器直接透传 |
| DNS 管理(管理端) | POST | `/api/admin/dns/providers/delete` | 错误:F:`invalid DNS provider id`;A(有集群仍引用时后端是否拒绝:待确认) |
| DNS 管理(管理端) | POST | `/api/admin/dns/domains/create` | 错误:A:RPC 失败(格式/重复等校验在后端,文案待确认) |
| DNS 管理(管理端) | GET | `/api/admin/dns/issues` | `params` 的具体类型(pb 字段原样输出):待确认 |
| 证书与 ACME(管理端) | GET | `/api/admin/certificates/form` | `hasPrivateKey` 取自 `len(KeyData)>0`,但查询未带私钥数据,可能恒为 false:待确认 |
| 证书与 ACME(管理端) | GET | `/api/admin/certificates/download` | 错误(JSON):`certificate data is empty`;证书不存在/无效 ID 提示原文待确认 |
| 证书与 ACME(管理端) | POST | `/api/admin/certificates/create` | 错误:RPC 校验原文,取值范围待确认 |
| 证书与 ACME(管理端) | POST | `/api/admin/certificates/update` | 错误:RPC 校验原文;证书/密钥留空时是否保留旧值取决于 RPC,待确认 |
| 证书与 ACME(管理端) | POST | `/api/admin/certificates/delete` | 错误:被引用时的限制由 RPC 决定,待确认 |
| 证书与 ACME(管理端) | GET | `/api/admin/acme/tasks/form` | `providers`/`acmeUsers` 元素结构:待确认 |
| 证书与 ACME(管理端) | POST | `/api/admin/acme/tasks/create` | 错误:校验交给 RPC,原文待确认<br>`authType` string,取值待确认 |
| 证书与 ACME(管理端) | POST | `/api/admin/acme/tasks/update` | 错误:RPC 校验原文,待确认 |
| 证书与 ACME(管理端) | POST | `/api/admin/acme/accounts/create` | 错误:校验交给 RPC,取值范围待确认 |
| 证书与 ACME(管理端) | POST | `/api/admin/acme/accounts/update` | 错误:RPC 校验原文,待确认<br>修改账户;`eabKey` 留空是否保留旧值取决于 RPC,待确认;记录日志 |
| 证书与 ACME(管理端) | GET | `/api/admin/acme/users` | `userId` int64,可选;0 的含义(全部或仅平台)取决于 RPC,待确认 |
| 证书与 ACME(管理端) | POST | `/api/admin/acme/users/create` | 错误:校验交给 RPC,待确认 |
| WAF/IP名单/缓存(管理端) | POST | `/api/admin/cache-tasks/reset` | 200 `{}`(无业务字段,待确认)<br>将任务重置为待重新执行状态(具体重置语义由 EdgeAPI `ResetHTTPCacheTask` 决定,待确认) |
| WAF/IP名单/缓存(管理端) | GET | `/api/admin/cache-policies` | `storageType` string,可选(合法取值待确认,推测 `file`/`memory`) |
| WAF/IP名单/缓存(管理端) | GET | `/api/admin/cache-policies/detail` | `siteWebCacheRule` 字段:待确认<br>错误:策略不存在时由 RPC/处理器报错(文案待确认) |
| WAF/IP名单/缓存(管理端) | POST | `/api/admin/cache-policies/create` | 各字段数值范围由 EdgeAPI 校验(待确认) |
| WAF/IP名单/缓存(管理端) | POST | `/api/admin/cache-policies/rules` | 错误:"缓存规则索引无效";"不能通过结构化编辑器修改高级缓存规则";"缓存规则不能超过 N 条"(N 待确认) |
| WAF/IP名单/缓存(管理端) | POST | `/api/admin/cache-policies/delete` | 错误:由 RPC 返回(例如策略仍被引用,文案待确认) |
| WAF/IP名单/缓存(管理端) | GET | `/api/admin/ip-lists/detail` | 错误:名单不存在/无权限(文案待确认) |
| WAF/IP名单/缓存(管理端) | POST | `/api/admin/ip-lists/delete` | 错误:由 RPC 返回(仍被引用时的文案待确认) |
| WAF/IP名单/缓存(管理端) | POST | `/api/admin/ip-lists/items/create` | `type` string,`ipv4`/`ipv6`/`all`;`all` 时值固定为 0.0.0.0(待确认完整形态)<br>`expiredAt` int64,可选,Unix 秒,0 表示永久(待确认)<br>不校验名单是否属于公共名单,仅依赖 RPC 层权限(待确认) |
| WAF/IP名单/缓存(管理端) | POST | `/api/admin/ip-lists/items/update` | 不校验归属,仅依赖 RPC 层(待确认) |
| WAF/IP名单/缓存(管理端) | POST | `/api/admin/ip-lists/items/delete` | 不校验归属,仅依赖 RPC 层(待确认) |
| WAF/IP名单/缓存(管理端) | GET | `/api/admin/waf-policies/detail` | inbound/outbound 等嵌套对象的完整字段:待确认 |
| WAF/IP名单/缓存(管理端) | POST | `/api/admin/waf-policies/groups/rollout` | `mode`/`enforcePercent` 的合法取值由 `ValidateGroupRefRollout` 决定(待确认) |
| WAF/IP名单/缓存(管理端) | POST | `/api/admin/waf-policies/sets/create` | 规则集对象字段:待确认 |
| WAF/IP名单/缓存(管理端) | POST | `/api/admin/waf-policies/import` | 是覆盖还是合并:待确认 |
| WAF/IP名单/缓存(管理端) | POST | `/api/admin/waf-policies/rule-pack` | 200 `{"result":{"added":...,"updated":...,"unchanged":...,"disabled":...}}`(各项元素类型待确认,`disabled` 为数量) |
| 日志与指标(管理端) | GET | `/api/metrics/query` | 200 `{"records":[{"page":string,"metricType":string,"keys":待确认,"value":number,"timestamp":待确认}]}`(此处是否带 `{code,data}` 包络待确认,错误是 `{code,message}`) |
| 日志与指标(管理端) | GET | `/api/admin/audit-logs` | `dayFrom` string,可选(格式待确认)<br>`dayTo` string,可选(格式待确认) |
| 日志与指标(管理端) | GET | `/api/admin/access-logs` | 各字段精确类型待确认<br>`hourFrom` / `hourTo` string,可选(Admin 层无格式校验,待确认) |
| 日志与指标(管理端) | GET | `/api/admin/access-logs/detail` | 错误:日志不存在时的文案待确认 |
| 日志与指标(管理端) | POST | `/api/admin/access-log-policies/test` | 200 `{}`(结果形态待确认) |
| 日志与指标(管理端) | GET | `/api/admin/access-log-settings` | `config` 默认值 1000/10000/100/true/500000;`queue` 各字段类型待确认 |
| 日志与指标(管理端) | POST | `/api/admin/access-log-settings` | 错误:无超级管理员权限(文案待确认)<br>Admin 层无范围校验,范围由 EdgeAPI 决定(待确认) |
| 日志与指标(管理端) | GET | `/api/admin/metrics` | `periodUnit` 取值 minute/hour/day/week/month;各定义数组元素字段待确认 |
| 日志与指标(管理端) | GET | `/api/admin/metrics/clusters` | 查询参数:`clusterId` int64,可选,未匹配到集群时静默回退到第一个集群;`category` string,可选(默认值待确认) |
| 财务(管理端) | POST | `/api/admin/finance/orders/finish` | 状态流转规则在 EdgeAPI,待确认 |
| 财务(管理端) | POST | `/api/admin/finance/orders/cancel` | 已完成订单不可取消;其余状态约束待确认 |
| 财务(管理端) | GET | `/api/admin/finance/refunds` | `status` string,可选(DAO 状态常量为 `pending`/`finished`/`cancelled`/`rejected`,过滤校验规则待确认) |
| 财务(管理端) | POST | `/api/admin/finance/refunds/finish` | 标记退款已完成(`FinishUserRefund`),本接口仅记录结果,是否实际向支付渠道发起退款待确认 |
| 财务(管理端) | GET | `/api/admin/finance/bills` | Count 固定带 `trafficRelated=true`,List 未带该条件,二者口径是否一致待确认(疑似 total 与 items 不一致) |
| 财务(管理端) | GET | `/api/admin/finance/bills/detail` | `trafficGB` 按 1024^3 换算;`bandwidthMbps` 按 `字节*8/1024/1024` 换算(与区域价格用的 1e6 不同,换算口径待确认) |
| 财务(管理端) | POST | `/api/admin/finance/bills/generate` | RPC 侧 `day` 非空时按日结算,否则按月结算;两者都为空时的行为待确认 |
| 财务(管理端) | POST | `/api/admin/finance/bills/pay` | 人工将账单标记为已支付(`PayUserBill`),是否扣减账户余额待确认 |
| 财务(管理端) | GET | `/api/admin/finance/accounts/detail` | `eventType` 取值含 `adminCharge`/`adminDebit`,其余待确认 |
| 财务(管理端) | POST | `/api/admin/finance/accounts/adjust` | 涉及资金,处理器未调用 `CreateLog`,审计依赖 EdgeAPI 侧(待确认) |
| 财务(管理端) | POST | `/api/admin/finance/plans/create` | `bandwidthRangesJSON` string,可选,JSON 数组,元素为 数值字典(键名待确认),非法返回 `invalid bandwidth price ranges` |
| 财务(管理端) | POST | `/api/admin/finance/plans/delete` | 删除套餐;是否允许删除已被购买的套餐待确认 |
| 财务(管理端) | GET | `/api/admin/finance/coupons/redemptions` | `status` 取值待确认 |
| 财务(管理端) | POST | `/api/admin/finance/coupons/save` | `maxUses`/`perUserLimit` int32,≥0(0 的含义待确认) |
| 财务(管理端) | POST | `/api/admin/finance/packages/prices/update` | `price` float64,价格(取值范围由 RPC 校验,`invalid traffic package price`,具体范围待确认) |
| 财务(管理端) | POST | `/api/admin/finance/packages/grants/create` | `count` int32,授予数量(范围由 RPC 校验,待确认) |
| 财务(管理端) | GET | `/api/admin/finance/regional-prices` | `from`/`to`:0 表示不限;label 为 `<数值><单位>` 或 `unlimited`;`prices` 键含义(推测为价格区间 ID)待确认 |
| 财务(管理端) | POST | `/api/admin/finance/regional-prices/update` | `price` float32,单价(取值范围由 RPC 校验,待确认) |
| 财务(管理端) | POST | `/api/admin/finance/methods/create` | 错误:400 `unsupported Alipay payment flow`;RPC错误(`invalid payment method name`/`invalid payment method code`/`payment method field is too long`/`invalid payment c<br>通用:`name` string 1–100 字符;`code` string,小写,格式 `orderMethodCodePattern`(待确认),且不可重复;`description` ≤255;`url` string ≤1024;`parentCode` string,`manual`/`alipay`/`w |
| 财务(管理端) | GET | `/api/admin/finance/settings` | `domainPolicy` 为 `domainpolicy.Config`,内部字段待确认 |
| 财务(管理端) | POST | `/api/admin/finance/settings/update` | `pricePeriod` string(仅转小写,未校验,取值待确认)<br>缓存任务额度 0 或非正数由读取方回落到默认值(依据 `userconfigs` 注释,细节待确认) |
| 用户与管理员(管理端) | GET | `/api/admin/admins` | `keyword` string,可选,匹配字段待确认<br>`hasWeakPassword` bool,可选,默认 false,为 true 时只列弱密码管理员(过滤语义依据 RPC 字段名推断,待确认) |
| 用户与管理员(管理端) | GET | `/api/admin/admins/form` | 可授权模块码范围:dashboard/server/node/dns/user/admin/finance/plan/ticket/log/setting(以 `IsSupportedAdminModule` 为准,common 是否包含待确认) |
| 用户与管理员(管理端) | POST | `/api/admin/admins/update` | `username` string,同创建(空值是否被接受待确认)<br>`fullname` string,1-100 字符(空值是否被接受待确认) |
| 用户与管理员(管理端) | POST | `/api/admin/admins/delete` | 错误:400 `invalid administrator id`;RPC错误(含 `can not delete current administrator`;删除最后一个超管是否被拒待确认) |
| 用户与管理员(管理端) | POST | `/api/admin/admins/access-keys/create` | 不校验 `adminId` 对应管理员是否存在(由 RPC 决定,待确认) |
| 用户与管理员(管理端) | GET | `/api/admin/users` | `keyword` string,可选,匹配字段待确认<br>`mobileIsVerified` int32,可选,默认 -1(推断为不过滤),其他取值含义待确认 |
| 用户与管理员(管理端) | GET | `/api/admin/users/detail` | `identities` 字段取值枚举(`orgType`/`type`/`status`)待确认 |
| 用户与管理员(管理端) | POST | `/api/admin/users/features` | `featureCodes` []string,可选,重复键传多个;为空则清空授权(整体覆盖),合法功能码范围待确认 |
| 用户与管理员(管理端) | POST | `/api/admin/users/create` | `email` string,可选,格式规则待确认 |
| 用户与管理员(管理端) | POST | `/api/admin/users/update` | `password` string,留空是否保持原密码待确认;非空须符合创建规则 |
| 用户与管理员(管理端) | POST | `/api/admin/users/delete` | 删除用户(软删除语义待确认);关联订单/账户/站点的处理待确认 |
| 工单/消息/通知(管理端) | GET | `/api/admin/notifications/tasks` | `status=0` 在列表中是否表示"全部"待确认<br>`result.error`/`response` 来自第三方通道,可能含敏感内容(是否在 API 侧脱敏待确认) |
| 工单/消息/通知(管理端) | GET | `/api/admin/tickets` | `latestReply` 的具体键与工单日志相同(待确认列表内是否完全一致) |
| 系统设置(管理端) | GET | `/api/admin/settings/license` | 是否仅超管可调用由 EdgeAPI 判定(待确认) |
| 系统设置(管理端) | GET | `/api/admin/settings/effective` | `data`:`{"revision":...,"settings":[Item]}`(`revision` 类型待确认) |
| 系统设置(管理端) | POST | `/api/admin/settings/ai` | 成功 `data` 为空(返回内容待确认)<br>错误:校验失败文案待确认<br>参数与上行 `ai` 字段同名(含 `isOn,provider,baseURL,apiKey,model,defenseIntervalMinutes,optimizationIntervalMinutes,requestTimeoutSeconds,maxRequestsPerDay,maxRequestsPerS<br>**管理端处理器未调用 `Validate()`,也不记操作日志**;范围校验是否在 EdgeAPI `UpdateSysSetting` 执行待确认 |
| 系统设置(管理端) | POST | `/api/admin/settings/general` | 错误:各字段校验失败(文案见源码,待确认逐项原文)<br>`dnsResolverType` string,须在支持集合内(选项含 default/go-native/cgo,完整集合待确认) |
| 系统设置(管理端) | POST | `/api/admin/settings/users` | 错误:依赖校验与字段范围错误(原文待确认) |
| 系统设置(管理端) | POST | `/api/admin/settings/authentication` | 错误:`config.Validate()` 的具体规则未读,待确认 |
| 系统设置(管理端) | POST | `/api/admin/settings/operations` | `section=logs`:`logsCanDelete`、`logsCanClean` bool;`logsRetentionDays` int 1~3650;`logsCapacityCount` int 1~1024;`logsCapacityUnit` string,MB/GB(具体常量值待确认)<br>`section=updates`:`updatesAutoCheck` bool;`updatesClearIgnored` bool;`updatesURL` string(经 `Normalize`+`Validate`,规则待确认) |
| 系统设置(管理端) | GET | `/api/admin/settings/runtime-parameters` | 已确认 `backup{enabled,intervalHours(1~168),retention(1~90),dir,offsiteDir,offsiteKeyFile,offsiteRetention}`,其余子结构字段待确认 |
| 系统设置(管理端) | POST | `/api/admin/settings/runtime-parameters` | 成功 `data` 为空(待确认是否回显配置) |
| 系统设置(管理端) | POST | `/api/admin/settings/general/asset` | 错误:`invalid asset kind` 及文件类型/大小校验错误(原文待确认) |
| 系统设置(管理端) | POST | `/api/admin/settings/security` | 错误:`the proposed IP policy would block the current administrator`、`the proposed domain policy would block the current administrator`,其余字段校验错误原文待确认 |
| 系统设置(管理端) | POST | `/api/admin/settings/server` | 错误:HTTP 与 HTTPS 不可同时关闭、监听冲突、需要证书、证书无效等(原文待确认) |
| 系统设置(管理端) | POST | `/api/admin/settings/pages` | 错误:数值为负、`isOn` 缺存储、文件名非法等(原文待确认) |
| 系统设置(管理端) | POST | `/api/admin/settings/pages/storages/create` | 错误:各字段校验错误(原文待确认) |
| 系统设置(管理端) | POST | `/api/admin/settings/pages/storages/delete` | **不记操作日志**;被引用时的行为待确认 |
| 系统设置(管理端) | POST | `/api/admin/settings/login` | 错误:`username is already in use`;账号被密码锁定时 HTTP 403 + `PasswordLockedMessage`;其他校验错误原文待确认 |
| 运维维护(管理端) | GET | `/api/admin/maintenance/health` | 错误:RPC 不可用时 `RPC服务暂时不可用...`(全文待确认) |
| 运维维护(管理端) | GET | `/api/admin/maintenance/health/history` | `items[]`:`checkedAt`,`critical`,`warnings`,`unknown`,`services`,`license`,`disk`(后三者结构细节待确认) |
| 运维维护(管理端) | GET | `/api/admin/maintenance/doctor` | 报告中的 `root` 字段不输出;检查项清单待确认(未读 `admindoctor.Checks()`) |
| 运维维护(管理端) | POST | `/api/admin/maintenance/database/update` | 错误(Fail,英文):如 `invalid database host`;连接校验失败的具体文案待确认<br>`host` string 必填,≤253,IP 或主机名(主机名正则 `adminUserNodeHostnamePattern` 待确认) |
| 运维维护(管理端) | POST | `/api/admin/maintenance/database/retention` | 成功无业务字段(`data` 内容待确认)<br>**疑似缺陷**:未提交的字段按 0 处理,没有"缺省保持原值"逻辑,漏传会把对应保留期清为 0;0 的具体含义(不保留或不清理)未核实,待确认 |
| 运维维护(管理端) | POST | `/api/admin/maintenance/database/table` | 成功无业务字段(`data` 内容待确认)<br>错误:`table confirmation does not match`;`invalid database table operation`;表是否允许清理/删除(`canClean`/`canDelete`)由 RPC 层判断,具体文案待确认 |
| 运维维护(管理端) | GET | `/api/admin/maintenance/backups` | 非超管:Fail `only a super administrator can manage ...`(全文待确认) |
| 运维维护(管理端) | GET | `/api/admin/maintenance/backups/download` | 查询参数:`fileName` string 必填(命名规则校验函数 `validateBackupFileName` 未读,待确认) |
| 运维维护(管理端) | POST | `/api/admin/maintenance/backups/restore` | 错误:任务冲突、文件不存在等(全文待确认)<br>`targetDatabase` string,临时库名,约束由 `validateRestoreDatabase` 决定(未读,待确认;前缀应为 `edgeroute_restore_`) |
| 运维维护(管理端) | POST | `/api/admin/maintenance/backups/delete` | 错误:确认文本不匹配(英文文案待确认);任务冲突 |
| 运维维护(管理端) | GET | `/api/admin/maintenance/transfers` | `currentEndpoints[]`:当前 API 端点(元素结构待确认) |
| 运维维护(管理端) | POST | `/api/admin/maintenance/transfers/create` | 错误文案待确认<br>`targetEndpoint` string 必填(格式校验待确认)<br>`nodeIds` int64 数组,参与迁移的边缘节点(多值传法与上限待确认) |
| 运维维护(管理端) | POST | `/api/admin/maintenance/updates/check` | 错误文案待确认 |
| 运维维护(管理端) | POST | `/api/admin/maintenance/updates/apply` | 成功:更新异步执行,`data` 内容待确认 |
| 运维维护(管理端) | POST | `/api/admin/maintenance/updates/rollback` | 错误:`type ROLLBACK to confirm`;更新进行中报错;`ExternalAPISynced` 时拒绝(文案待确认) |
| 运维维护(管理端) | POST | `/api/admin/maintenance/updates/user/apply` | 成功 `data` 内容待确认<br>错误文案待确认 |
| 运维维护(管理端) | GET | `/api/admin/maintenance/api-nodes` | `openFile*`、`systemTraffic*` 为前缀写法,具体键名待确认 |
| 运维维护(管理端) | GET | `/api/admin/maintenance/api-nodes/logs` | `tags`(元素结构待确认,来自 `FindNodeCommonTags`)<br>错误:参数不合法的具体文案待确认 |
| 运维维护(管理端) | POST | `/api/admin/maintenance/api-nodes/create` | 错误:上述各校验失败的英文文案待确认;多行文本分行规则(`normalizeAdminUserNodeLines`)及主机名规范待确认 |
| 运维维护(管理端) | POST | `/api/admin/maintenance/api-nodes/update` | 成功无业务字段(待确认) |
| 运维维护(管理端) | POST | `/api/admin/maintenance/api-nodes/toggle` | 成功无业务字段(待确认) |
| 运维维护(管理端) | POST | `/api/admin/maintenance/api-nodes/delete` | 成功无业务字段(待确认)<br>错误文案待确认(如删除主节点是否被拒,未核实) |
| 安装向导、找回、确认与 CSRF 令牌(管理端) | GET | `/csrf/token` | `Init()` 注入的 `tea*` 页面键是否出现在响应中:待确认<br>session 创建时机未核实,待确认 |
| 安装向导、找回、确认与 CSRF 令牌(管理端) | POST | `/recover/validateApi` | 错误:`FailField("host", ...)` 等,具体文案待确认 |
| 安装向导、找回、确认与 CSRF 令牌(管理端) | POST | `/recover/updateHosts` | 成功无业务字段(`data` 内容待确认)<br>错误:`参数配置错误，请刷新页面后重试`(长度不等)等,其余待确认<br>`protocol`,`host`,`port`,`nodeId`,`nodeSecret`(同上,但**无 Must 校验**,缺失时的行为待确认) |
| 安装向导、找回、确认与 CSRF 令牌(管理端) | POST | `/setup/validateApi` | 错误:口令无效 403;各字段格式错误(文案待确认,仅确认 `仅支持新 API 节点安装`)<br>**风险**:原样回显完整 `licenseKey`;`apiNode` 在校验前就已赋值,失败时 `data` 是否仍带 `apiNode` 待确认 |
| 安装向导、找回、确认与 CSRF 令牌(管理端) | POST | `/setup/validateDb` | 错误:连接失败、MySQL 版本低于 5.7.8 等,文案待确认<br>`accessLogKeepDays` int(取值范围待确认) |
| 安装向导、找回、确认与 CSRF 令牌(管理端) | POST | `/setup/validateAdmin` | 错误:格式、长度、两次不一致(文案待确认) |
| 安装向导、找回、确认与 CSRF 令牌(管理端) | POST | `/setup/checkLocalIP` | IP 无法解析时默认 true;私网判断函数 `utils.IsLocalIP` 未读,具体判定规则待确认<br>`host` string(**无需口令**,是否必填待确认) |
| 安装向导、找回、确认与 CSRF 令牌(管理端) | POST | `/setup/mysql/installLogs` | `data.logs` = `SharedLogger.ReadAll()` 的结果,元素结构待确认(未读 `mysqlinstallers/utils`) |
| 安装向导、找回、确认与 CSRF 令牌(管理端) | POST | `/setup/confirm` | 错误:`请输入至少一个API节点地址`;`请输入NodeId`;`请输入Secret`;协议错误;连接失败;角色不是 `admin`(各文案待确认) |
| 安全设置(用户端) | GET | `/api/access-keys` | `accessedAt` 数值,最近使用时间(单位待确认:取自 RPC 字段 `AccessedAt`) |
| 安全设置(用户端) | POST | `/api/domain-ownership` | 时间戳单位待确认(源码未标注) |
| 仪表盘与个人资料(用户端) | POST | `/api/identity` | 字段必填性与格式校验在后端 RPC,待确认 |
| 仪表盘与个人资料(用户端) | GET | `/api/security-posture` | `events` array(最多 100 条,按时间倒序),元素:`id`、`siteId`、`siteName`、`timestamp` int64、`host`、`path`、`method`、`status` int32、`action`、`clientIP`、`category`、`region`、`coun |
| 消息与工单(用户端) | GET | `/api/audit-logs` | `logs` array,元素:`id` int64、`level` string(≤16 字符)、`action` string(≤128)、`description` string(≤400)、`createdAt` int64(时间戳,单位待确认)<br>`from` / `to` string 起止日期(原样透传后端 RPC,格式待确认) |
| 计费与订单(用户端) | GET | `/api/billing` | `balanceReminderLimit` 余额提醒阈值(来自 userconfigs.BalanceReminderLimit,类型待确认) |
| 计费与订单(用户端) | POST | `/api/plans/{planID}/order` | `methodCode` 为空时不生成回调地址,是否走余额由后端 RPC 决定(待确认:空 methodCode 的具体语义) |
| 计费与订单(用户端) | PUT | `/api/user-plans/{userPlanID}/auto-renew` | `maxAmount` float64,单次自动续费金额上限,经 `PlanAutoRenew.Validate()` 校验(具体范围待确认:校验在 EdgeCommon userconfigs 中) |
| 站点管理(用户端) | GET | `/api/sites/{siteID}/sync-status` | 200 `{"status":?,"pendingTasks":?,"failedTasks":?,"updatedAt":?,"error":?}`,各字段类型待确认:状态对象来自 RPC pb 对象,未逐字段核实 |
| 站点管理(用户端) | PUT | `/api/sites/{siteID}/protocols` | 200(成功体待确认:未核实是否为 `{"ok":true}`) |
| 站点管理(用户端) | GET | `/api/sites/{siteID}/statistics` | `accounting` 来自 `trafficstats.Describe`,内部字段待确认 |
| 站点管理(用户端) | GET | `/api/sites/{siteID}/security-events` | `SecurityEvent`:`id`,`siteId?`,`siteName?`,`timestamp`,`host`,`path`,`method`,`status`,`action`,`clientIP`(已掩码),`category`,`region?`,`countryCode?`,`edgeTarget? |
| 站点管理(用户端) | GET | `/api/sites/{siteID}/waf` | 错误:404;经 `writeWAFRPCError` 映射的错误码待确认(该函数未读全);502 |
| 站点管理(用户端) | PUT | `/api/sites/{siteID}/waf` | 200(成功体待确认,未核实) |
| 站点管理(用户端) | PUT | `/api/sites/{siteID}/access` | 200(成功体待确认,未核实) |
| 站点管理(用户端) | PUT | `/api/sites/{siteID}/uam` | 200(成功体待确认,未核实) |
| 站点管理(用户端) | PUT | `/api/sites/{siteID}/cc` | 200(成功体待确认,未核实)<br>阈值各字段的数值范围校验待确认(未全部核实) |
| 站点管理(用户端) | PUT | `/api/sites/{siteID}/cache` | 200(成功体待确认,未核实)<br>`applyAPICacheRule` 后半段校验未读全,其余校验待确认 |
| 证书 / DNS 服务商 / ACME 自动签发(用户端) | GET | `/api/dns-providers/{providerID}` | 待确认:`apiParams` 中密钥字段是否已由 RPC 层脱敏,处理函数仅做 JSON 反序列化直接返回 |
| 静态网站 Pages(用户端) | PUT | `/api/pages/{siteID}/cache` | 待确认:`matchType`、`life`、`status` 等字段的具体枚举/范围(由 `buildCacheCondition`、`validateCacheLife` 等函数校验,本次未逐项读取) |
| 静态网站 Pages(用户端) | POST | `/api/pages/{siteID}/files` | 同名文件的覆盖行为待确认:由 RPC 层决定 |
| 站点分组配置(用户端) | GET | `/api/groups/{groupID}` | `config` 配置概要(`pb.FindEnabledServerGroupConfigInfoResponse`,字段为 `hasHTTPReverseProxy`、`hasTCPReverseProxy`、`hasUDPReverseProxy`、`serverGroupId`、`hasRootConfig`、 |
| 站点分组配置(用户端) | PUT | `/api/groups/{groupID}/web-policy` | `trustedProxyRanges` 的校验规则待确认:未逐项核对 |
| 站点分组配置(用户端) | GET | `/api/groups/{groupID}/access-log` | `status1`..`status5` bool(2xx..5xx 状态码记录开关,待确认:各自对应关系按字段名推断) |
| 四层转发与 AI(用户端) | GET | `/api/layer4/projects/{id}/diagnostics` | 参考 `serverconfigs.Layer4Diagnostics`:`unavailable`、`collectedAt`、`startedAt`、`active`、`accepted`、`trafficIn`、`trafficOut`、`reasons`(map[string]int64);待确认:RPC 是否 |
| 四层转发与 AI(用户端) | PUT | `/api/ai` | 错误:`403`;`422`(直接输出 RPC 返回的错误信息);`400` 模式取值无效(待确认:具体文案) |

<a id="defects"></a>

## 附录 B 疑似缺陷与字段偏差（涉及 13 条路由）

以下是后端现状，不是要实现的需求。其他 AI 对接时应知悉，避免依赖缺陷行为。

| 模块 | 方法 | 路径 | 说明 |
|---|---|---|---|
| 网站与网站分组(管理端) | POST | `/api/admin/sites/create` | 疑似缺陷:操作日志 `defer` 在失败时 serverId 为 0 |
| 节点与集群(管理端) | POST | `/api/admin/user-nodes/toggle` | 疑似缺陷:操作日志复用"更新用户节点"文案,无单独的启停日志(待确认意图) |
| 节点与集群(管理端) | GET | `/api/admin/database-nodes` | 疑似缺陷:`keyword` 在 RPC 分页之后只过滤当前页,`total` 仍是未过滤的总数,翻页与关键字组合会漏数据 |
| 节点与集群后半部分(管理端) | GET | `/api/admin/clusters/lifecycle` | `registration.secret` 及 `configJSON` 内的 secret 是集群密钥明文(疑似缺陷,见文末),文档不列值 |
| WAF/IP名单/缓存(管理端) | POST | `/api/admin/ip-lists/import` | 注意:代码对表头 `IP/IP段` 的跳过判断因先转小写而永远不成立,该表头行会被当数据解析(疑似缺陷) |
| WAF/IP名单/缓存(管理端) | GET | `/api/admin/waf-policies/check` | 不做归属校验(疑似缺陷) |
| WAF/IP名单/缓存(管理端) | POST | `/api/admin/waf-policies/delete` | 未经过 `findAdminWAFPolicy`,无 id>0 校验与共享策略归属校验(疑似缺陷) |
| 财务(管理端) | POST | `/api/admin/finance/regional-prices/items/update` | **疑似缺陷**:`type` 漏传时按 traffic 换算,更新带宽区间会把 Mbps 值按 GB 倍率写入,调用方必须与区间原类型一致 |
| 工单/消息/通知(管理端) | GET | `/api/admin/notifications/recipients` | **疑似缺陷**:管理员或通道对象为 nil 的记录会被跳过,导致 `items` 数量可能少于 `size`,而 `total` 仍包含它们 |
| 运维维护(管理端) | POST | `/api/admin/maintenance/database/retention` | **疑似缺陷**:未提交的字段按 0 处理,没有"缺省保持原值"逻辑,漏传会把对应保留期清为 0;0 的具体含义(不保留或不清理)未核实,待确认 |
| 安装向导、找回、确认与 CSRF 令牌(管理端) | POST | `/setup/validateDb` | **缺陷**:1049 分支内 `db, err :=` 遮蔽外层变量,内层连接未关闭(连接泄漏) |
| 安装向导、找回、确认与 CSRF 令牌(管理端) | POST | `/setup/install` | **缺陷**:对 `newHost`/`newPort` 仅检查非空且不含 `\r\n"`,**不复核** `validateApi` 的格式与端口范围,`dbJSON` 各字段同样未再校验,可绕过前面几步直接提交 |
| 安装向导、找回、确认与 CSRF 令牌(管理端) | POST | `/setup/detectDB` | `canInstall = linux && amd64 && os.Getgid()==0`,用的是**组 ID 而非 UID**,疑似缺陷 |

已核对源码后与原文档不一致的字段，以源码为准：

- 管理端 AI 设置：真实字段为 `autoApplyRiskMax`、`maxConcurrentTasks`（原文档拼成 `autoAplyRiskMax`、`maxConcurentTasks`）。
- 管理端安全设置：`clientIPHeaderNames` 在 GET 与 POST 都是**单个字符串**（逗号或空白分隔），原文档写成数组。
- 管理端数据库保留天数：字段名原样含拼写 `traficDailyStatDays`；接口漏传的字段按 0 处理，须全部回传。
- 管理端备份与 API 节点：Manifest 里有原样拼写 `tolVersion`；API 节点列表的监听字段为 `rpcHTTPListens` 等，提交字段为 `rpcHttpListens` 等（大小写不同）。
- 用户端四层（Layer4）与 AI 模块的部分字段拼写与类型，以对应模块说明为准，实现前请再对照源码。
- “漏传布尔即置 false”的整体覆盖语义出现在集群、DNS、节点 IP、管理员/用户 update、API 节点 update/toggle 等接口：调用方必须把读取到的全部字段原样回传。
