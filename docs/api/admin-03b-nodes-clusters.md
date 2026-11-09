# 节点与集群后半部分(管理端)

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

| 接口名称 | 方法 | 路径 | 主要请求参数 | 返回数据结构 | 功能说明 | 认证 |
|---|---|---|---|---|---|---|
| 更新集群系统服务策略 | POST | `/api/admin/clusters/policies/system-service` | `clusterId` int64 必填<br>`useGlobal` bool 默认 false<br>`systemdIsOn` bool 默认 false | data 为空<br>错误:`node cluster not found` | 开关集群节点的 systemd 服务托管;写入后记录日志。 | 需登录(node 模块)+CSRF |
| 检查集群 systemd 服务 | POST | `/api/admin/clusters/policies/system-service/check` | `clusterId` int64 必填,>0 | `results`:向集群节点下发检查消息(超时 10 秒)的聚合结果,元素结构待确认<br>错误:`invalid cluster id` | 向集群下所有节点发送 systemd 服务检查消息并返回结果。 | 需登录(node 模块)+CSRF |
| 更新集群 WebP 策略 | POST | `/api/admin/clusters/policies/webp` | `clusterId` int64 必填<br>`useGlobal` bool<br>`isOn` bool<br>`quality` int,0..100<br>`requireCache` bool<br>`minLengthMB` int64,≥0<br>`maxLengthMB` int64,≥ minLengthMB | data 为空<br>错误:`invalid WebP policy values`;`policy.Init()` 的错误原文 | 配置集群 WebP 转换策略(质量、是否要求命中缓存、体积范围 MB)。 | 需登录(node 模块)+CSRF |
| 更新集群 UAM 策略 | POST | `/api/admin/clusters/policies/uam` | `clusterId` int64 必填<br>`useGlobal` bool<br>`isOn` bool | data 为空<br>错误:`node cluster not found` | 开关集群 UAM(5 秒盾)策略。 | 需登录(node 模块)+CSRF |
| 更新集群 CC 防护策略 | POST | `/api/admin/clusters/policies/cc` | `clusterId` int64 必填<br>`useGlobal` bool<br>`thresholdsJSON` string,`HTTPCCThreshold` 数组的 JSON,最多 20 个,至少 1 个已启用 | data 为空<br>错误:`invalid CC thresholds`;`CC thresholds cannot exceed 20`;`at least one CC threshold is required when protection is enabled` | 保存集群 CC 阈值列表;策略 `IsOn` 强制为 true。阈值元素字段:待确认(见 `serverconfigs.HTTPCCThreshold`)。 | 需登录(node 模块)+CSRF |
| 更新集群 DDoS 防护策略 | POST | `/api/admin/clusters/policies/ddos` | `clusterId` int64 必填<br>`useGlobal` bool<br>`isOn` bool<br>`maxConnections` int32,≥0<br>`maxConnectionsPerIP` int32,0 或 ≥5<br>`newConnectionsMinutelyRate` int32,0 或 ≥3<br>`newConnectionsMinutelyRateBlockTimeout` int32,≥0<br>`newConnectionsSecondlyRate` int32,0 或 ≥3<br>`newConnectionsSecondlyRateBlockTimeout` int32,≥0<br>`synProxy` bool<br>`xdp` bool<br>`allowIPs` string,以逗号/分号/换行/空白分隔,≤200 个合法 IP<br>`ports` string,同样分隔,≤100 个,1..65535 | data 为空<br>错误:`DDoS limits cannot be negative`;`per-IP connection limit is below the supported minimum`;`per-minute connection rate is below the supported minimum`;`per-second connection rate is below the supported minimum`;`DoS allow list cannot exceed 200 addresses`;`invalid DDoS allow-list IP: x`;`DDoS protected ports cannot exceed 100` | 保存集群 DDoS 防护设置(连接数/新建连接速率/SYN 代理/XDP/白名单/受保护端口)。 | 需登录(node 模块)+CSRF |
| 更新集群服务基线 | POST | `/api/admin/clusters/policies/server-baseline` | `clusterId` int64 必填<br>`useGlobal` bool<br>域名匹配:`matchDomainStrictly` bool;`domainMismatchAction` string,`page`/`redirect`/`close`;`domainMismatchStatusCode` int,400..599;`domainMismatchContentHTML` string,≤64KiB;`domainMismatchRedirectURL` string,action=redirect 时须为绝对 http/https 地址且无 userinfo;`allowMismatchDomains` string,≤100 个合法域名;`allowNodeIP` bool;`nodeIPShowPage` bool;`nodeIPPageHTML` string,≤64KiB;`defaultDomain` string,合法域名或 IP;`domainAuditingIsOn` bool;`domainAuditingPrompt` string,≤200<br>通用:`serverName` string,≤60 且不含 CR/LF/NUL;`supportsLowVersionHTTP`/`forceLnRequest`/`enableServerAddrVariable`/`requestOriginsWithEncodings` bool;`lnRequestSchedulingMethod` string,`random`/`urlMapping`;`xffMaxAddresses` int,0..100<br>访问日志:`accessLogIsOn`/`accessLogEnableRequestHeaders`/`accessLogCommonRequestHeadersOnly`/`accessLogEnableResponseHeaders`/`accessLogEnableCookies`/`accessLogEnableServerNotFound` bool;`recordServerError` bool<br>性能:`performanceAutoReadTimeout`/`performanceAutoWriteTimeout`/`performanceDebug` bool<br>TCP:`tcpPortRangeMin` int,≥1024;`tcpPortRangeMax` int,≤65534 且 ≥ min;`tcpDenyPorts` string,≤200 个,1..65535<br>全局 L2:`originShieldIsOn` bool;`originShieldMode` string,`direct`/`cloudflare`;`originShieldHost`/`originShieldProtocol`(http/https)/`originShieldPort`(1..65535);`originShieldCloudflareHost`(须为域名)/`originShieldCloudflareProtocol`(http/https)/`originShieldCloudflareDNSProviderId` int64,>0 且类型为 cloudFlare<br>源站保护:`originProtectIsOn` bool;`originProtectRotateSecret` bool<br>回源限速:`originRateLimitIsOn` bool;`originRateLimitMaxQPS` int,1..1000;`originRateLimitBurst` int,0..1000000 | data 为空<br>错误(原文):`invalid unmatched-domain action`;`invalid default domain`;`X-Forwarded-For address limit must be between 0 and 100`;`TCP port range must be ordered and within 1024-65534`;回源限速以"回源限速配置无效:"开头;其余域名/端口/URL/HTML 长度类错误原文待确认 | 一次性保存集群服务基线(未匹配域名处理/默认域名/访问日志/性能/TCP 端口段/L2 回源/源站保护/回源限速)。仅在需要写入时(非沿用全局)校验。源站保护开启时沿用旧密钥或生成新密钥,`originProtectRotateSecret`=true 时轮换;密钥只写不读,响应中不返回。L2 的 `LoopToken` 自动生成并保留。 | 需登录(node 模块)+CSRF |
| 导出源站保护规则 | GET | `/api/admin/clusters/origin-protect/export` | `clusterId` int64 必填,>0<br>`ports` string,可空,分隔字符串,≤20 个,1..65535<br>`reveal` bool 默认 false | `enabled`,`nodeCount`,`ipv4`,`ipv6`,`skipped`,`nginx`,`iptables`,`header`,`ipCount`,`note`,`secretConfigured`;`reveal`=true 且 `enabled` 时多一个 `secret`(回源签名密钥,文档不列值)<br>响应头 `Cache-Control: no-store`<br>`ipv4`/`ipv6`/`skipped`/`nginx`/`iptables` 元素结构来自 `BuildOriginProtectRules`,待确认<br>错误:`invalid cluster id`,及端口数量/范围类错误原文待确认 | 按集群内已启用节点的登记 IP 与状态上报的 HostIP 生成源站防火墙放行规则。GET 请求可通过 `reveal` 取得密钥,无 CSRF;er-v45 起返回密钥时会写一条审计日志(`NodeCluster_LogRevealOriginProtectSecret`)。 | 需登录(node 模块) |
| 更新集群网络安全策略 | POST | `/api/admin/clusters/policies/network-security` | `clusterId` int64 必填<br>`useGlobal` bool<br>`status` string,`auto`/`on`/`off`<br>`tcpSyncookies` bool<br>`tcpRFC1337` bool<br>`tcpMaxSynBacklog` int,0 取默认,否则 128..1000000<br>`ipRPFilter` int,0..2<br>`ipAcceptRedirects`/`ipSendRedirects`/`ipAcceptSourceRoute` bool<br>`icmpIgnoreBroadcasts`/`icmpIgnoreBogusResponses` bool<br>`icmpRateLimitMilliseconds` int,0..60000 | data 为空<br>错误:`invalid network security status "x"`;`maxSynBacklog should be between 128 and 1000000`;`rpFilter should be between 0 and 2`;`ICMP rateLimitMilliseconds should be between 0 and 60000` | 保存节点内核网络安全策略。`auto` 模式会强制 syncookies/RFC1337 开启、重定向与源路由关闭、ICMP 忽略广播与错误响应;`off` 模式不校验其余数值。 | 需登录(node 模块)+CSRF |
| 更新集群缓存/WAF 策略指派 | POST | `/api/admin/clusters/policies/assignments` | `clusterId` int64 必填<br>`cachePolicyId` int64,需写入时 >0<br>`cacheUseGlobal` bool<br>`wafPolicyId` int64,需写入时 >0<br>`wafUseGlobal` bool | data 为空<br>错误:`cache policy is required`;`WAF policy is required`;`node cluster not found` | 为集群指派缓存策略与 WAF 策略,两者的"沿用全局"开关各自独立;各写一条日志。 | 需登录(node 模块)+CSRF |
| 保存集群阈值 | POST | `/api/admin/clusters/operations/thresholds/save` | `clusterId` int64 必填<br>`thresholdId` int64,≤0 新建,>0 更新<br>`useGlobal` bool<br>`item`/`param` string,须在节点监控项定义中<br>`operator` string,须有效<br>`value` string,须为数字<br>`message` string,≤500<br>`sumMethod` string,`sum`/`avg`<br>`duration` int32,1..10080<br>`durationUnit` string,固定 `minute`<br>`notifyDuration` int32,0..10080<br>`isOn` bool,仅更新时生效 | data 为空<br>错误:`invalid threshold metric`;`threshold value must be numeric`;`invalid threshold aggregation`;`invalid threshold duration`;`threshold message is too long`;`cluster threshold limit reached`(每集群 100 条);`threshold not found in cluster`;`invalid threshold id` | 新建或更新集群监控阈值。沿用全局且非全局集群时只写开关并直接成功。无审计日志。 | 需登录(node 模块)+CSRF |
| 删除集群阈值 | POST | `/api/admin/clusters/operations/thresholds/delete` | `clusterId` int64 必填<br>`thresholdId` int64 必填,>0 | data 为空<br>错误:`inherited thresholds cannot be deleted from this cluster`;`threshold not found in cluster`;`invalid threshold id` | 删除集群自有阈值;沿用全局的集群不可删。无审计日志。 | 需登录(node 模块)+CSRF |
| 保存集群通知接收人 | POST | `/api/admin/clusters/operations/notifications` | `clusterId` int64 必填<br>`useGlobal` bool<br>`receiversJSON` string,≤64KiB,格式 `[{"type":"recipient"或"group","id":int64}]`,按 type+id 去重,id≤0 或未知 type 跳过,最多 200 个 | data 为空<br>错误:`notification selection is too large`;`invalid notification selection`;`too many notification receivers` | 设置集群告警通知的接收人/接收分组;沿用全局时不改接收人。无审计日志。 | 需登录(node 模块)+CSRF |
| 保存集群防火墙动作 | POST | `/api/admin/clusters/operations/firewall/save` | `clusterId` int64 必填<br>`actionId` int64,0 新建,>0 更新<br>`useGlobal` bool<br>`name` string 必填,≤100<br>`eventLevel` string,`debug`/`notice`/`warning`/`error`/`critical`/`fatal`<br>`type` string,`ipset`/`firewalld`/`iptables`/`script`/`httpAPI`/`html`<br>ipset:`ipSetWhiteName`/`ipSetBlackName`/`ipSetWhiteNameIPv6`/`ipSetBlackNameIPv6` string,仅字母数字下划线;`ipSetMaxElements` int,0..10000000;`ipSetAutoAddToIPTables`/`ipSetAutoAddToFirewalld` bool<br>script:`path` string 必填;`scriptCwd` string;`scriptArgsJSON` string,JSON 字符串数组,≤100 个,每个 ≤500<br>httpAPI:`httpAPIURL` string,http/https 且无 userinfo;`httpAPITimeoutSeconds` int,1..60;`httpAPISecret` string,≤512 字节,留空沿用旧值<br>html:`htmlContent` string,1 字节..64KiB | data 为空<br>错误:`invalid firewall action metadata`;`firewall action limit reached`(每集群 100 条);`unsupported firewall action type`;`IPSet names may only contain letters, numbers and underscores`;`HTML response must be between 1 byte and 64 KiB`;其余字段错误原文待确认 | 新建或更新集群防火墙响应动作。字段随 `type` 取舍。无审计日志;密钥不在响应中返回。 | 需登录(node 模块)+CSRF |
| 删除集群防火墙动作 | POST | `/api/admin/clusters/operations/firewall/delete` | `clusterId` int64 必填<br>`actionId` int64 必填 | data 为空<br>错误:`inherited firewall actions cannot be deleted from this cluster`;`firewall action not found in cluster` | 删除集群自有防火墙动作。无审计日志。 | 需登录(node 模块)+CSRF |
| 置顶/取消置顶集群 | POST | `/api/admin/clusters/pin` | `clusterId` int64 必填,>0<br>`isPinned` bool | data 为空<br>错误:`invalid cluster id` | 设置集群是否置顶;记录日志。 | 需登录(node 模块)+CSRF |
| 删除集群 | POST | `/api/admin/clusters/delete` | `clusterId` int64 必填 | data 为空<br>错误:`cluster still has active sites` | 集群下仍有未删除站点时拒绝,否则调用 `DeleteNodeCluster`;记录日志。 | 需登录(node 模块)+CSRF |
| 创建节点 | POST | `/api/admin/nodes/create` | `clusterId` int64 必填<br>`name` string 必填,≤100 字符<br>`addresses` string 必填,以换行/逗号/分号分隔,支持 IP 段展开,去重后 1..256 个<br>`groupId` int64,可选<br>`regionId` int64,可选<br>`grantId` int64,可选(SSH 授权)<br>`sshHost` string,可选<br>`sshPort` int,0 取 22,范围 1..65535<br>`sshPassword` string,可选,≤255 且不含 NUL<br>`dnsRoutes` string 数组,可选<br>`dnsDomainId` int64:接收但未使用,实际按集群 DNS 配置推导 | `nodeId` int64<br>错误:`selected node group is unavailable`;`selected node region is unavailable`;`selected SSH grant is unavailable`;`已选择 SSH 授权,请清空密码,或将授权改为'稍后配置'`;`SSH 登录失败:用户名或密码错误,请检查后重试` | 创建节点并登记 IP 地址。填写 `sshPassword` 时自动创建 SSH 授权(用户名固定 `root`,方式 `user`),先测试连接:认证失败则禁用该授权并报错,主机不可达不阻塞。`dnsRoutes` 仅保留属于集群绑定域名的有效线路,无效项被静默丢弃。IP 创建失败会回滚节点与新建授权。记录日志。 | 需登录(node 模块)+CSRF |
| 批量创建节点 | POST | `/api/admin/nodes/create-batch` | `clusterId` int64 必填<br>`namePrefix` string,可选<br>`addresses` string 必填,≤100 个<br>`groupId` int64,可选<br>`regionId` int64,可选<br>`grantId` int64,可选 | `nodeIds` int64 数组<br>错误:分组/区域/授权不可用类提示原文待确认 | 每个地址创建一个节点,节点名为地址;有前缀时为 `<前缀> 001` 递增。选了授权则 SSH 配置 host=节点 IP、port=22。任一失败会回滚。记录日志。 | 需登录(node 模块)+CSRF |
| 集群生命周期总览 | GET | `/api/admin/clusters/lifecycle` | `clusterId` int64 必填 | `cluster{id,name}`;`groups[{id,name}]`;`regions[{id,name}]`;`grants[{id,name,method,username}]`;`notInstalled[Item]`;`upgrades[Item]`;`registration{endpoints,clusterId,secret,configJSON}`(`Item` 见公共结构)<br>响应头 `Cache-Control: no-store`<br>错误:`node cluster not found` | 返回集群待安装节点、待升级节点、可选分组/区域/SSH 授权与节点注册信息。`registration.secret` 及 `configJSON` 内的 secret 是集群密钥明文(疑似缺陷,见文末),文档不列值。 | 需登录(node 模块) |
| 查询节点安装令牌(列表) | GET | `/api/admin/nodes/enrollment` | `clusterId` int64 必填 | `items`:`Record` 数组(见公共结构)<br>响应头 `Cache-Control: no-store` | 同一 action(`NodeEnrollmentAction`)的 GET 分支:固定以 `action=list` 调用 RPC 列出集群的节点安装记录。不写日志。 | 需登录(node 模块) |
| 管理节点安装令牌 | POST | `/api/admin/nodes/enrollment` | `action` string 必填,`create`/`approve`/`revoke`<br>`clusterId` int64<br>`groupId` int64<br>`regionId` int64<br>`level` int<br>`count` int<br>`id` string,64 位小写十六进制(approve/revoke 用)<br>`address` string<br>`baseURL` string,action=create 时必须为 https、无 userinfo/query/fragment、路径为空或 `/admin` | `items` Record 数组;`tokens` string 数组(create 时一次性返回安装令牌,文档不列值);`record` Record;均为 omitempty,可能缺省<br>响应头 `Cache-Control: no-store`<br>错误:`不支持的安装管理操作`;`请填写可从节点访问的 HTTPS 管理端地址`;`管理端地址路径只能为空或 /admin` | 同一 action 的 POST 分支:创建/批准/撤销节点安装记录。令牌有效期 15 分钟,重试期 60 分钟。`baseURL` 仅校验,未传入 RPC 请求。`count`/`level` 范围及 `state`/`phase` 枚举由 RPC 侧决定,待确认。写操作记录日志(不含令牌)。 | 需登录(node 模块)+CSRF |
| 节点发布批次列表 | GET | `/api/admin/node-rollouts` | `clusterId` int64 | `items[Rollout]`(见公共结构)<br>响应头 `Cache-Control: no-store` | 列出集群的节点滚动发布任务。`clusterId` 必填性待确认。 | 需登录(node 模块) |
| 创建节点发布批次 | POST | `/api/admin/node-rollouts/create` | `clusterId` int64<br>`role` string,空则 `node`<br>`taskType` string<br>`nodeIds` int64 数组<br>`batchSize` int,1..100<br>`userId` int64<br>`serverId` int64 | `rollout`:`Rollout`<br>错误:`每批节点数必须在 1 到 100 之间`;RPC 错误原文 | 创建分批滚动发布任务。`taskType` 取值待确认。无审计日志;错误用 `this.Fail(err.Error())` 原样返回。 | 需登录(node 模块)+CSRF |
| 暂停节点发布批次 | POST | `/api/admin/node-rollouts/pause` | `rolloutId` int64 | data 为空<br>错误:RPC 错误原文 | 暂停指定发布任务。无审计日志。 | 需登录(node 模块)+CSRF |
| 恢复节点发布批次 | POST | `/api/admin/node-rollouts/resume` | `rolloutId` int64 | data 为空<br>错误:RPC 错误原文 | 恢复已暂停的发布任务。无审计日志。 | 需登录(node 模块)+CSRF |
| 取消节点发布批次 | POST | `/api/admin/node-rollouts/cancel` | `rolloutId` int64<br>`reason` string,可选 | data 为空<br>错误:RPC 错误原文 | 取消发布任务并记录原因。无审计日志。 | 需登录(node 模块)+CSRF |
| 重试节点发布批次 | POST | `/api/admin/node-rollouts/retry` | `rolloutId` int64,>0 | data 为空<br>错误:`rollout id is required`;RPC 错误原文 | 重试失败的发布任务。无审计日志。 | 需登录(node 模块)+CSRF |
| 升级节点 | POST | `/api/admin/nodes/upgrade` | `nodeId` int64 必填,>0<br>`clusterId` int64,可选;>0 时校验节点属于该集群(含次要集群) | data 为空<br>错误:节点不属于集群/无效 ID 类提示原文待确认 | 触发单个节点升级;记录日志。 | 需登录(node 模块)+CSRF |
| 节点分组列表 | GET | `/api/admin/node-groups` | `clusterId` int64 必填,>0 | `items[{id,name,countNodes}]`;`clusterId`<br>错误:集群不存在/无效 ID 原文待确认 | 列出集群下的节点分组及节点数。 | 需登录(node 模块) |
| 创建节点分组 | POST | `/api/admin/node-groups/create` | `clusterId` int64 必填<br>`name` string 必填,去空白后 1..100 字符 | `groupId` int64<br>错误:`node group name is required`;`node group name is too long` | 在集群下创建节点分组;记录日志。 | 需登录(node 模块)+CSRF |
| 更新节点分组 | POST | `/api/admin/node-groups/update` | `clusterId` int64 必填<br>`groupId` int64 必填<br>`name` string 必填,1..100 字符 | data 为空<br>错误:`node group name is required`;`node group name is too long`;`node group does not belong to the selected cluster`;`invalid node group` | 修改分组名称;记录日志。 | 需登录(node 模块)+CSRF |
| 删除节点分组 | POST | `/api/admin/node-groups/delete` | `clusterId` int64 必填<br>`groupId` int64 必填 | data 为空<br>错误:`node group still has active nodes`;`node group does not belong to the selected cluster`;`invalid node group` | 分组下仍有节点时拒绝删除;记录日志。 | 需登录(node 模块)+CSRF |
| 排序节点分组 | POST | `/api/admin/node-groups/sort` | `clusterId` int64 必填<br>`groupIds` int64 数组,须恰好是该集群现有全部分组的一个排列 | data 为空<br>错误:提交 ID 与现有集合不一致的提示原文待确认 | 按提交顺序重排集群的全部节点分组;记录日志。 | 需登录(node 模块)+CSRF |
| 节点区域列表 | GET | `/api/admin/node-regions` | `page` int 默认 1<br>`size` int 默认 20,最大 100<br>`regionId` int64,≥0,0 表示全部 | `regions[{id,name,description,isOn,countNodes,displayName,countryCode,hasLocation,latitude,longitude}]`;`nodes[{id,name,cluster{id,name}或null,region{id,name}或null}]`;`page`;`size`;`total`;`selectedRegionId` | 分页列出节点区域,并列出所选区域(或全部)下的节点。 | 需登录(node 模块) |
| 创建节点区域 | POST | `/api/admin/node-regions/create` | `name` string 必填,≤100 字符<br>`description` string,≤500<br>`displayName` string,≤100<br>`countryCode` string,2 位字母,转大写<br>`hasLocation` bool<br>`latitude` float,-90..90<br>`longitude` float,-180..180 | `regionId` int64<br>错误:名称必填/超长/国家码/经纬度越界/含控制字符(`\n`、`\t` 例外)类提示,原文待确认 | 创建区域。经纬度仅在 `hasLocation`=true 时校验,否则置 0。日志用 defer 写入,创建失败也会记录一条 regionId=0。 | 需登录(node 模块)+CSRF |
| 更新节点区域 | POST | `/api/admin/node-regions/update` | 同创建,另加:<br>`regionId` int64 必填<br>`isOn` bool | data 为空<br>错误同创建 | 修改区域信息和启停状态;记录日志。 | 需登录(node 模块)+CSRF |
| 删除节点区域 | POST | `/api/admin/node-regions/delete` | `regionId` int64 必填 | data 为空<br>错误:`node region still has active nodes` | 区域下仍有节点时拒绝删除;记录日志。 | 需登录(node 模块)+CSRF |
| 排序节点区域 | POST | `/api/admin/node-regions/sort` | `regionIds` int64 数组,须恰好是全部区域的一个排列 | data 为空<br>错误:提交 ID 与现有集合不一致的提示原文待确认 | 按提交顺序重排全部区域;记录日志。 | 需登录(node 模块)+CSRF |
| 移动节点到区域 | POST | `/api/admin/node-regions/move-node` | `nodeId` int64 必填,>0<br>`regionId` int64,0 表示取消区域,<0 非法 | data 为空<br>错误:`node region is disabled`;节点/区域 ID 无效提示原文待确认 | 把节点划入某区域或移出区域;目标区域须启用;记录日志。 | 需登录(node 模块)+CSRF |
| 节点日志列表 | GET | `/api/admin/node-logs` | `offset` int,≥0,默认 0<br>`size` int,0 取 20,有效 1..100<br>`dayFrom`/`dayTo` string,日期,dayFrom 不得晚于 dayTo(格式待确认,疑为 YYYY-MM-DD)<br>`keyword` string,≤200<br>`level` string,`error`/`warning`/`info`/`success`,转小写<br>`type` string,空/`unread`/`needFix`<br>`tag` string,转大写,≤64,匹配 `^[A-Z0-9_:-]+$`<br>`clusterId` int64,≥0<br>`nodeId` int64,≥0 | `items[{id,tag,description,level,createdAt,count,isFixed,isRead,canFix,node{id,name,cluster{id,name}},server{id,name}或null}]`;`total`;`offset`;`size`;`countUnread`;`countNeedFix`;`tags`(`FindNodeCommonTags` 的结构待确认);`clusters[{id,name}]`;`nodes[{id,name}]`(仅 `clusterId`>0 时返回,最多 500 个)<br>错误:`invalid node log type`;分页/日期/级别/标签/关键词越界提示原文待确认 | 查询全局节点日志。`nodeId`>0 时校验节点存在(且属于所给 `clusterId`);否则 `clusterId`>0 时校验集群存在。`canFix = 未修复 且 serverId>0 且 level 为 success/warning/error`。`createdAt` 为 pb 原值,格式待确认。 | 需登录(node 模块) |
| 标记节点日志已读 | POST | `/api/admin/node-logs/read` | `logIds` int64 数组,1..100 个 | data 为空<br>错误:`select between 1 and 100 node logs` | 把所选日志标为已读,并通知未读数刷新。无审计日志。 | 需登录(node 模块)+CSRF |
| 全部标记节点日志已读 | POST | `/api/admin/node-logs/read-all` | 无业务参数 | data 为空 | 把全部节点日志标为已读,并通知未读数刷新。无审计日志。 | 需登录(node 模块)+CSRF |
| 修复节点日志 | POST | `/api/admin/node-logs/fix` | `logIds` int64 数组,1..100 个 | data 为空<br>错误:`select between 1 and 100 node logs` | 把所选日志标记为已修复;记录日志。 | 需登录(node 模块)+CSRF |
| 全部修复节点日志 | POST | `/api/admin/node-logs/fix-all` | 无业务参数 | data 为空 | 把全部可修复日志标记为已修复;记录日志。 | 需登录(node 模块)+CSRF |
| 删除节点日志 | POST | `/api/admin/node-logs/delete` | `dayFrom`/`dayTo` string<br>`keyword` string,必填<br>`level` string<br>`type` string<br>`tag` string<br>`clusterId` int64,不校验,允许负值<br>`nodeId` int64,不校验,允许负值 | data 为空<br>错误:`deleting node logs requires a keyword` | 按条件批量删除节点日志,必须带关键词;记录日志。 | 需登录(node 模块)+CSRF |
| SSH 授权列表 | GET | `/api/admin/node-grants` | `page` int 默认 1<br>`size` int 默认 20,最大 100<br>`keyword` string,可选 | `items[{id,name,method,methodName,username,description,sudo,countClusters,countNodes}]`;`total`;`page`;`size`<br>`methodName`:`SH 私钥`/`用户名与密码` | 分页列出节点 SSH 授权,不返回任何密码或私钥。 | 需登录(node 模块) |
| SSH 授权表单 | GET | `/api/admin/node-grants/form` | `grantId` int64,≤0 表示新建 | `grant`:新建时为 null,否则含列表项各键,另加 `hasPassword`/`hasPrivateKey`/`hasPassphrase`(bool,不返回明文)<br>响应头 `Cache-Control: no-store` | 取得编辑表单所需的授权信息。 | 需登录(node 模块) |
| SSH 授权详情 | GET | `/api/admin/node-grants/detail` | `grantId` int64 必填 | `grant`(同 form,`countClusters`/`countNodes` 有值);`clusters[{id,name}]`;`nodes[{id,name,isOn,cluster{id,name}或null}]`<br>响应头 `Cache-Control: no-store`<br>错误:授权不存在提示原文待确认 | 查看授权及其被哪些集群、节点使用。 | 需登录(node 模块) |
| 创建 SSH 授权 | POST | `/api/admin/node-grants/create` | `name` string<br>`method` string,已见 `privateKey`/`user`<br>`username` string<br>`password` string<br>`privateKey` string<br>`passphrase` string<br>`description` string<br>`sudo` bool | `grantId` int64<br>错误:校验全部交给 RPC,取值范围待确认 | 创建 SSH 授权;记录日志。 | 需登录(node 模块)+CSRF |
| 更新 SSH 授权 | POST | `/api/admin/node-grants/update` | 同创建,另加 `grantId` int64 必填 | data 为空<br>错误:同创建,待确认 | 更新 SSH 授权;密码/私钥留空时是否保留旧值取决于 RPC,待确认;记录日志。 | 需登录(node 模块)+CSRF |
| 删除 SSH 授权 | POST | `/api/admin/node-grants/delete` | `grantId` int64 必填 | data 为空 | 软删除授权(`DisableNodeGrant`);记录日志。 | 需登录(node 模块)+CSRF |
| 测试 SSH 授权 | POST | `/api/admin/node-grants/test` | `grantId` int64 必填<br>`host` string<br>`port` int32 | `isOk` bool;`error` string | 用授权连接指定主机端口,测试结果通过 `isOk`/`error` 返回,连接失败不走错误包络。 | 需登录(node 模块)+CSRF |

## 公共结构

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

## 异常与疑似缺陷
- `clusters/lifecycle` 的 `registration.secret`/`configJSON` 明文返回集群密钥。
- `origin-protect/export?reveal=true` 以 GET 返回回源签名密钥,无 CSRF;已补审计日志(er-v45),仍建议后续改为 POST。
- 阈值/通知/防火墙动作、node-rollouts 写接口、node-logs read/read-all 无审计日志。
- node-rollouts 写接口错误绕过脱敏与状态码映射。
- `nodes/create` 的 `dnsDomainId` 未使用;`nodes/enrollment` 的 `baseURL` 仅校验未传 RPC。
- `node-regions/create` 日志用 defer,失败也记录;`node-logs/delete` 的 `clusterId`/`nodeId` 不校验。
- CSRF 失败返回纯文本 403,与其余 JSON 错误格式不一致。
- 多数集群策略接口无 `clusterId>0` 前置校验,不存在时才报 `node cluster not found`。
