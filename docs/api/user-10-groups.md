# 站点分组配置(用户端)
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

| 接口名称 | 方法 | 路径 | 主要请求参数 | 返回数据结构 | 功能说明 | 认证 |
|---|---|---|---|---|---|---|
| 站点组列表 | GET | `/api/groups` | 无 | `200` `{"groups":[ServerGroup...]}`(无数据时为 `[]`)<br>错误:`502` `读取站点组失败` | 列出当前用户的站点组。RPC 超时 15s。 | 需登录 |
| 创建站点组 | POST | `/api/groups` | JSON 请求体:<br>`csrfToken` string 必填<br>`name` string 必填,trim 后 1-100 字符 | 成功 `201` `{"id":groupID}`<br>错误:`400` 名称无效/`请求格式无效`;`403` CSRF;`502` `创建站点组失败` | 创建站点组。 | 需登录+CSRF写保护 |
| 站点组详情 | GET | `/api/groups/{groupID}` | 路径参数:`groupID` int 必填 | `200`:<br>`group` ServerGroup<br>`config` 配置概要(`pb.FindEnabledServerGroupConfigInfoResponse`,字段为 `hasHTTPReverseProxy`、`hasTCPReverseProxy`、`hasUDPReverseProxy`、`serverGroupId`、`hasRootConfig`、`hasWAFConfig`、`hasCacheConfig`、`hasCharsetConfig`、`hasStatConfig`、`hasCompressionConfig`、`hasOptimizationConfig`、`hasRequestHeadersConfig`、`hasResponseHeadersConfig`、`hasWebsocketConfig`、`hasWebPConfig`、`hasAccessLogConfig`、`hasRemoteAddrConfig`、`hasPagesConfig`、`hasRequestLimitConfig`、`hasRequestScriptsConfig`、`hasHLSConfig`,均 bool;待确认:Go 序列化时的 json tag 大小写以 pb 生成代码为准,本模块未核对)<br>`features` []string(当前套餐功能码)<br>错误:`404` `站点组不存在`;`502` `读取站点组配置失败` / `unable to read plan features` | 读取站点组及其各类配置是否已设置、套餐功能列表。 | 需登录 |
| 修改站点组名称 | PATCH | `/api/groups/{groupID}` | 路径参数:`groupID`<br>JSON 请求体:`csrfToken` 必填;`name` string 必填,1-100 | `200` `{"ok":true}`<br>错误:`404` `站点组不存在或无权操作`;`409` `站点组名称已存在，请使用其他名称`;`502`;`400`;`403` | 重命名站点组,名称在用户下唯一。 | 需登录+CSRF写保护 |
| 删除站点组 | DELETE | `/api/groups/{groupID}` | 路径参数:`groupID`<br>JSON 请求体:`csrfToken` 必填 | `200` `{"ok":true}`<br>错误:`404` `站点组不存在或无权操作`;`409` `站点组中仍有站点，请先移出站点后再删除`;`502`;`403` | 删除站点组。风险操作,组内仍有站点时拒绝删除。 | 需登录+CSRF写保护 |
| 读取 5 秒盾配置 | GET | `/api/groups/{groupID}/uam` | 路径参数:`groupID` | 通用 GET 返回,配置键 `uam`:<br>`isPrior`、`isOn`、`addToWhiteList` bool<br>`minQPSPerIP` int<br>`keyLife` int(秒)<br>`challengeDifficulty` int<br>`challengeTimeout` int(秒)<br>`onlyURLPatterns`、`exceptURLPatterns` URLPattern[]<br>额外错误:`403` `当前套餐未开通5秒盾功能`(功能码 `server.uam`) | 读取站点组 5 秒盾(UAM)配置。 | 需登录 |
| 更新 5 秒盾配置 | PUT | `/api/groups/{groupID}/uam` | 路径参数:`groupID`<br>JSON 请求体:`csrfToken` 必填;`isPrior`、`isOn`、`addToWhiteList` bool;`minQPSPerIP` int 0-999999;`keyLife` int 0-3600;`challengeDifficulty` int,0 取默认 18,范围 14-22;`challengeTimeout` int,0 取默认 120,范围 60-180;`onlyURLPatterns`、`exceptURLPatterns` URLPattern[] | 通用 PUT 返回。<br>特有错误:`400` `5 秒盾高级参数无效`、`仅限/排除 URL 规则无效: …`;`403` 套餐未开通 | 更新 5 秒盾。 | 需登录+CSRF写保护 |
| 读取 CC 防护配置 | GET | `/api/groups/{groupID}/cc` | 路径参数:`groupID` | 通用 GET 返回,配置键 `cc`:<br>`isPrior`、`isOn` bool<br>`thresholds` CCThreshold[]<br>`onlyURLPatterns`、`exceptURLPatterns` URLPattern[]<br>额外错误:`403` 套餐未开通 CC 功能(`server.cc`) | 读取 CC 防护配置。 | 需登录 |
| 更新 CC 防护配置 | PUT | `/api/groups/{groupID}/cc` | 路径参数:`groupID`<br>JSON 请求体:`csrfToken` 必填;`isPrior`、`isOn`;`thresholds` CCThreshold[],1-10 条;`onlyURLPatterns`、`exceptURLPatterns` | 通用 PUT 返回。<br>特有错误:`400` 阈值数量/字段无效、`HTTP 方法无效: X`、`HTTP 方法数量过多`;启用 `isOn` 时至少需一条已启用阈值;`403` 套餐未开通 | 更新 CC 防护阈值,启用的阈值会做配置初始化校验。 | 需登录+CSRF写保护 |
| 读取 WebP 配置 | GET | `/api/groups/{groupID}/webp` | 路径参数:`groupID` | 通用 GET 返回,配置键 `webp`(直接输出 `serverconfigs.WebPImageConfig`):<br>`isPrior`、`isOn` bool<br>`minLength`、`maxLength` SizeCapacity(`{count,unit}`)<br>`mimeTypes` []string<br>`extensions` []string<br>`conds` 匹配条件对象(用户端不可编辑) | 读取 WebP 图片转换配置。 | 需登录 |
| 更新 WebP 配置 | PUT | `/api/groups/{groupID}/webp` | 路径参数:`groupID`<br>JSON 请求体:`csrfToken` 必填;`isPrior`、`isOn` bool;`mimeTypes` []string;`extensions` []string | 通用 PUT 返回。<br>特有错误:`400` `启用 WebP 时必须配置 MIME 类型或扩展名` | 更新 WebP 转换配置。启用时 MIME 与扩展名至少填一项。 | 需登录+CSRF写保护 |
| 读取 HLS 加密配置 | GET | `/api/groups/{groupID}/hls` | 路径参数:`groupID` | 通用 GET 返回,配置键 `hls`:<br>`isPrior`、`isOn` bool(对应 `Encrypting.IsOn`)<br>`onlyURLPatterns`、`exceptURLPatterns` URLPattern[] | 读取 HLS 加密配置。 | 需登录 |
| 更新 HLS 加密配置 | PUT | `/api/groups/{groupID}/hls` | 路径参数:`groupID`<br>JSON 请求体:`csrfToken` 必填;`isPrior`、`isOn`;`onlyURLPatterns`、`exceptURLPatterns` URLPattern[] | 通用 PUT 返回。<br>特有错误:`400` 仅限/排除 URL 规则无效 | 更新 HLS 加密配置。 | 需登录+CSRF写保护 |
| 读取内容压缩配置 | GET | `/api/groups/{groupID}/compression` | 路径参数:`groupID` | 通用 GET 返回,配置键 `compression`:<br>`isPrior`、`isOn` bool<br>`useDefaultTypes` bool<br>`types` []string(`brotli`\|`gzip`\|`zstd`\|`deflate`)<br>`decompressData` bool<br>`minLength`、`maxLength` SizeCapacity<br>`mimeTypes`、`extensions`、`exceptExtensions` []string<br>`enablePartialContent` bool | 读取内容压缩配置。 | 需登录 |
| 更新内容压缩配置 | PUT | `/api/groups/{groupID}/compression` | 路径参数:`groupID`<br>JSON 请求体:`csrfToken` 必填;字段同 GET 的 `compression`。校验:`isPrior&&isOn&&!useDefaultTypes` 时 `types` 不能为空;`types` 取值见左;容量 `unit` 为 `byte`\|`kb`\|`mb`\|`gb`(空视为 byte),`count` 0-1000000;`minLength` 不得大于 `maxLength`;列表项 ≤128 字符,扩展名自动补 `.` | 通用 PUT 返回。<br>特有错误:`400` 压缩类型/容量/列表校验文案 | 更新内容压缩配置。 | 需登录+CSRF写保护 |
| 读取页面优化配置 | GET | `/api/groups/{groupID}/optimization` | 路径参数:`groupID` | 通用 GET 返回,配置键 `optimization`:<br>`isPrior` bool<br>`html` `{isOn,keepComments,keepConditionalComments,keepDefaultAttrVals,keepDocumentTags,keepEndTags,keepQuotes,keepWhitespace}`<br>`javascript` `{isOn,precision,version,keepVarNames}`<br>`css` `{isOn,precision,keepCSS2}` | 读取 HTML/JS/CSS 压缩优化配置。 | 需登录 |
| 更新页面优化配置 | PUT | `/api/groups/{groupID}/optimization` | 路径参数:`groupID`<br>JSON 请求体:`csrfToken` 必填;字段同 GET。校验:`precision` 0-20;`javascript.version` 为 0(自动)或 2015-2024 | 通用 PUT 返回。<br>特有错误:`400` 精度/版本无效 | 更新页面优化配置。 | 需登录+CSRF写保护 |
| 读取 WAF 配置 | GET | `/api/groups/{groupID}/waf` | 路径参数:`groupID` | 通用 GET 返回,配置键 `waf`:<br>`isPrior`、`isOn` bool<br>`ignoreGlobalRules` bool<br>`defaultCaptcha` string(`none`\|`default`)<br>`forceCaptcha` bool<br>`mode` string(`defend`\|`observe`\|`bypass`)<br>`maxRequestBodySize` int64(字节)<br>`groups` `[{code,name,description,direction,isOn}]`,`direction` 为 `inbound`\|`outbound`<br>错误:`403` `当前套餐未开通 WAF 功能`;`502` | 读取 WAF 配置及规则组开关。RPC 超时 20s。 | 需登录 |
| 更新 WAF 配置 | PUT | `/api/groups/{groupID}/waf` | 路径参数:`groupID`<br>JSON 请求体:`csrfToken` 必填;`isPrior`、`isOn`、`ignoreGlobalRules`、`defaultCaptcha`、`forceCaptcha`(需 `isOn=true`)、`mode`、`maxRequestBodySize`(0 取默认 1MiB,范围 64KiB-16MiB)、`groups[{code,isOn,...}]`(`code` 必须在模板内) | 通用 PUT 返回。<br>特有错误:`400` `未知的 WAF 规则组` 等;`403` `当前套餐未开通 WAF 功能`(权限不足)。 | 更新 WAF 模式、验证码与规则组开关。 | 需登录+CSRF写保护 |
| 读取缓存配置 | GET | `/api/groups/{groupID}/cache` | 路径参数:`groupID` | 通用 GET 返回,配置键 `cache`:<br>`isPrior`、`isOn`<br>`addStatusHeader`、`addAgeHeader`、`enableCacheControlMaxAge`、`useDefaultPolicy`、`purgeIsOn` bool<br>`purgeKeyConfigured` bool(只读)<br>`rules` CacheRule[](高级复杂条件规则不返回) | 读取缓存配置与规则。 | 需登录 |
| 更新缓存配置 | PUT | `/api/groups/{groupID}/cache` | 路径参数:`groupID`<br>JSON 请求体:`csrfToken` 必填;`isPrior`、`isOn`、`addStatusHeader`、`addAgeHeader`、`enableCacheControlMaxAge`、`useDefaultPolicy`、`purgeIsOn`;`rules` CacheRule[],最多 50 条 | 通用 PUT 返回。<br>特有错误:`400` `缓存规则索引无效`(`sourceIndex` 无效)、`不能修改高级缓存规则`、规则字段校验文案 | 更新缓存规则。`sourceIndex=-1` 表示新增;高级规则 PUT 时保留不动。 | 需登录+CSRF写保护 |
| 创建缓存任务(刷新/预热) | POST | `/api/groups/{groupID}/cache/tasks` | 路径参数:`groupID`<br>JSON 请求体(上限 256KiB):<br>`csrfToken` 必填<br>`type` string 必填,`purge`\|`fetch`<br>`keyType` string 必填,`key`\|`prefix`(`fetch` 只能用 `key`)<br>`keys` []string 必填,1-1000 条,每条 ≤2048,必须 http/https、有 host、无 userinfo,自动去重 | 成功 `201` `{"id":taskID,"count":有效URL数}`<br>错误:`400` `请至少输入一个 URL` 等校验文案;`422` `没有匹配到当前账户的有效站点 URL`;`404`;`403`;`502` `创建缓存任务失败` | 对站点组下的 URL 发起缓存清除或预热任务,仅匹配当前账户站点的 URL。RPC 超时 20s。 | 需登录+CSRF写保护 |
| 读取请求限制配置 | GET | `/api/groups/{groupID}/request-limit` | 路径参数:`groupID` | 通用 GET 返回,配置键 `requestLimit`:<br>`isPrior`、`isOn`<br>`maxConns`、`maxConnsPerIP` int<br>`outBandwidthPerConn` SizeCapacity<br>`maxBodySize` SizeCapacity | 读取并发连接、单连接带宽、请求体大小限制。 | 需登录 |
| 更新请求限制配置 | PUT | `/api/groups/{groupID}/request-limit` | 路径参数:`groupID`<br>JSON 请求体:`csrfToken` 必填;`isPrior`、`isOn`;`maxConns`、`maxConnsPerIP` int 0-1000000;`outBandwidthPerConn`、`maxBodySize` SizeCapacity(`unit` 为 `byte`\|`kb`\|`mb`\|`gb`,默认 kb;`count` -1 到 1000000,-1 表示不限) | 通用 PUT 返回。<br>特有错误:`400` 数值/单位无效 | 更新请求限制。 | 需登录+CSRF写保护 |
| 读取 Web 策略配置 | GET | `/api/groups/{groupID}/web-policy` | 路径参数:`groupID` | 通用 GET 返回,配置键 `policy`:<br>`charset` `{isPrior,isOn,charset,isUper,force}`<br>`remoteAddr` `{isPrior,isOn,type,requestHeaderName,value,trustedProxyRanges}`<br>`stat` `{isPrior,isOn}` | 读取字符集、客户端真实 IP 获取方式、统计开关。 | 需登录 |
| 更新 Web 策略配置 | PUT | `/api/groups/{groupID}/web-policy` | 路径参数:`groupID`<br>JSON 请求体:`csrfToken` 必填;`charset`、`remoteAddr`、`stat`(结构同 GET)。校验:`charset.isOn` 时 `charset` 匹配 `^[a-z0-9._-]{1,40}$`;`remoteAddr.type` 为 `default`\|`proxy`\|`requestHeader`\|`variable`;`requestHeader` 要求 `requestHeaderName` 匹配 `^[A-Za-z0-9_-]+(?:,[A-Za-z0-9_-]+)*$`;`variable` 的 `value` 长度 1-200 且含 `${` | 通用 PUT 返回。<br>特有错误:`400` 字符集/来源类型/变量无效 | 更新 Web 策略。`trustedProxyRanges` 的校验规则待确认:未逐项核对。 | 需登录+CSRF写保护 |
| 读取访问日志配置 | GET | `/api/groups/{groupID}/access-log` | 路径参数:`groupID` | 通用 GET 返回,配置键 `accessLog`:<br>`isPrior`、`isOn`<br>`fields` []int(仅返回用户可编辑项)<br>`status1`..`status5` bool(2xx..5xx 状态码记录开关,待确认:各自对应关系按字段名推断)<br>`enableClientClosed` bool<br>`firewallOnly` bool | 读取访问日志记录设置。 | 需登录 |
| 更新访问日志配置 | PUT | `/api/groups/{groupID}/access-log` | 路径参数:`groupID`<br>JSON 请求体:`csrfToken` 必填;`isPrior`、`isOn`;`fields` []int,取值 1 请求头、2 响应头、3 参数、4 Cookie、5 扩展、6 Referer、7 UserAgent、8 请求体;`status1`..`status5`、`enableClientClosed`、`firewallOnly` bool | 通用 PUT 返回。<br>特有错误:`400` `不支持的访问日志字段: N`(9 响应体等不可选) | 更新访问日志设置。PUT 时保留已有的、用户端不可编辑的字段码。 | 需登录+CSRF写保护 |
| 读取 WebSocket 配置 | GET | `/api/groups/{groupID}/websocket` | 路径参数:`groupID` | 通用 GET 返回,配置键 `websocket`:<br>`isPrior`、`isOn`<br>`handshakeTimeout` `{count,unit}`<br>`allowAllOrigins` bool<br>`allowedOrigins` []string<br>`requestSameOrigin` bool<br>`requestOrigin` string | 读取 WebSocket 设置。 | 需登录 |
| 更新 WebSocket 配置 | PUT | `/api/groups/{groupID}/websocket` | 路径参数:`groupID`<br>JSON 请求体:`csrfToken` 必填;字段同 GET。校验:`handshakeTimeout.unit` 为 `ms`\|`second`\|`minute`\|`hour` 且不超过 24h;`allowedOrigins` 小写、≤253 字符、匹配 `^[a-z0-9*._:-]+$`,通配符仅允许 `*`、`*.x` 或以 `:*` 结尾,最多 64 个;`isPrior&&isOn&&!allowAllOrigins` 时不能为空;`requestOrigin` ≤200 且无换行 | 通用 PUT 返回。<br>特有错误:`400` 超时/来源无效 | 更新 WebSocket 配置。 | 需登录+CSRF写保护 |
| 读取 HTTP 反向代理配置 | GET | `/api/groups/{groupID}/http-reverse-proxy` | 路径参数:`groupID` | 通用 GET 返回,配置键 `reverseProxy`:<br>`isPrior`、`isOn`<br>`schedulingCode`、`hashKey`、`stickyType`、`stickyParam` string<br>`primaryOrigins`、`backupOrigins` OriginConfig[]<br>`requestHostType` int、`requestHost` string、`requestHostExcludingPort` bool<br>`requestURI`、`stripPrefix` string<br>`autoFlush`、`followRedirects`、`retry50X`、`retry40X` bool<br>`addHeaders` []string<br>`connTimeout`、`readTimeout`、`idleTimeout` Duration<br>`maxConns`、`maxIdleConns` int | 读取 HTTP/HTTPS 回源配置。 | 需登录 |
| 更新 HTTP 反向代理配置 | PUT | `/api/groups/{groupID}/http-reverse-proxy` | 路径参数:`groupID`<br>JSON 请求体:`csrfToken` 必填;字段同 GET。校验:`schedulingCode` 为 `random`\|`roundRobin`\|`leastConn`\|`hash`\|`sticky`;`hash` 要求 `hashKey`(≤500);`sticky` 要求 `stickyType`(`cookie`\|`header`\|`argument`)和 `stickyParam`(`^[A-Za-z0-9_-]{1,50}$`);`primaryOrigins`/`backupOrigins` 各 ≤32;`requestHostType` 0 代理服务器 Host、1 源站 Host、2 自定义;`stripPrefix` ≤500 且以 `/` 开头或含 `${`;`addHeaders` 仅允许 `X-Real-IP`、`X-Forwarded-For`、`X-Forwarded-By`、`X-Forwarded-Host`、`X-Forwarded-Proto`(空数组表示禁用);三个超时 `unit` 为 `ms`\|`second`\|`minute`\|`hour` 且 ≤7 天;`maxConns`、`maxIdleConns` 0-1000000 | 通用 PUT 返回。<br>特有错误:`400` 源站/调度/超时校验文案;`502` `HTTP 回源配置未初始化` | 更新 HTTP 回源配置。RPC 超时 30s。源站校验见公共结构 OriginConfig。 | 需登录+CSRF写保护 |
| 读取 TCP 反向代理配置 | GET | `/api/groups/{groupID}/tcp-reverse-proxy` | 路径参数:`groupID` | `200` `{"group":ServerGroup,"network":"tcp","reverseProxy":Layer4Proxy}`<br>额外错误:`403` 套餐未开通 TCP(`server.tcp`) | 读取站点组 TCP 四层反代配置。 | 需登录 |
| 更新 TCP 反向代理配置 | PUT | `/api/groups/{groupID}/tcp-reverse-proxy` | 路径参数:`groupID`<br>JSON 请求体:`csrfToken` 必填;Layer4Proxy 字段(见公共结构),`proxyProtocol` 仅 TCP 生效 | 通用 PUT 返回。<br>特有错误:`400` 源站/调度校验;`403` 套餐未开通 TCP | 更新 TCP 四层反代。源站协议允许 `tcp`\|`tcp4`\|`tcp6`\|`tls`\|`tls4`\|`tls6`。 | 需登录+CSRF写保护 |
| 读取 UDP 反向代理配置 | GET | `/api/groups/{groupID}/udp-reverse-proxy` | 路径参数:`groupID` | `200` `{"group":ServerGroup,"network":"udp","reverseProxy":Layer4Proxy}`<br>额外错误:`403` 套餐未开通 UDP(`server.udp`) | 读取站点组 UDP 四层反代配置。 | 需登录 |
| 更新 UDP 反向代理配置 | PUT | `/api/groups/{groupID}/udp-reverse-proxy` | 路径参数:`groupID`<br>JSON 请求体:`csrfToken` 必填;Layer4Proxy 字段(见公共结构);`proxyProtocol` 对 UDP 不生效 | 通用 PUT 返回。<br>特有错误:`400`;`403` 套餐未开通 UDP | 更新 UDP 四层反代。源站协议仅允许 `udp`。 | 需登录+CSRF写保护 |
| 读取请求/响应头配置 | GET | `/api/groups/{groupID}/headers` | 路径参数:`groupID` | 通用 GET 返回,配置键 `headers`:`{"request":HeaderPolicy,"response":HeaderPolicy}`,HeaderPolicy 见公共结构 | 读取请求头与响应头策略(含 CORS)。 | 需登录 |
| 更新请求/响应头配置 | PUT | `/api/groups/{groupID}/headers` | 路径参数:`groupID`<br>JSON 请求体:`csrfToken` 必填;`request`、`response`(HeaderPolicy)。校验:`setHeaders`、`deleteHeaders` 各 ≤32;名称 ≤128 且为合法 token;禁用头 `connection`、`proxy-connection`、`keep-alive`、`transfer-encoding`、`content-length`、`upgrade`、`trailer`、`te`、`host`;`value` ≤2000 且不含 `\r`、`\n`、`\x00`;仅 `response` 处理 `cors` | 通用 PUT 返回。<br>特有错误:`400` 头名称/值无效、CORS 校验文案(含 `allowOrigin=="*"` 且 `allowCredentials` 为 true) | 更新请求/响应头设置与 CORS。RPC 超时 30s。 | 需登录+CSRF写保护 |
| 读取自定义页面配置 | GET | `/api/groups/{groupID}/custom-pages` | 路径参数:`groupID` | 通用 GET 返回,配置键 `customPages`:<br>`enableGlobalPages` bool<br>`pages` `[{id,status[],newStatus,bodyType,url,body}]` | 读取自定义错误页面配置。 | 需登录 |
| 更新自定义页面配置 | PUT | `/api/groups/{groupID}/custom-pages` | 路径参数:`groupID`<br>JSON 请求体:`csrfToken` 必填;`enableGlobalPages` bool;`pages` 最多 20 个。每页:`id`(须属于当前站点组)、`status` []string(每项匹配 `^[1-5][0-9x]{2}$`,1-20 个)、`newStatus` int(0 或 100-599;`redirectURL` 时只能 301/302/307/308)、`bodyType` `html`\|`url`\|`redirectURL`、`url`(http/https,≤512)、`body`(`html` 时 ≤32KiB) | 通用 PUT 返回。<br>特有错误:`400` 页面数量/状态码/类型/URL 无效 | 更新自定义错误页面。 | 需登录+CSRF写保护 |

## 公共结构

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
