# DNS 管理(管理端)

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

| 接口名称 | 方法 | 路径 | 主要请求参数 | 返回数据结构 | 功能说明 | 认证 |
|---|---|---|---|---|---|---|
| DNS 集群列表 | GET | `/api/admin/dns/clusters` | 查询参数:<br>`page` int64,默认 1<br>`size` int64,默认 20,上限 100<br>`keyword` string,可选,原样传给后端(不去空白以外的处理) | 200 `{"items":[DNSClusterItem],"total":int64,"page":int64,"size":int64}`(`DNSClusterItem` 见公共结构)<br>错误:A:统计/列表 RPC 失败 | 分页列出启用的节点集群及其 DNS 配置概况。每个集群最多 4 路并发各查一次 DNS 配置;已配置域名时再查一次是否有未同步变更。单个集群的 DNS 查询失败不会让整个请求失败,而是写入该项的 `error` 字段(见异常说明)。 | 需登录+DNS权限 |
| DNS 集群表单 | GET | `/api/admin/dns/clusters/form` | 查询参数:<br>`clusterId` int64 必填,须 >0 | 200 `{"cluster":{"id":int64,"name":string,"dnsName":string,"domainId":int64,"providerId":int64,"providerType":string,"nodesAutoSync":bool,"serversAutoSync":bool,"cnameRecords":[string],"ttl":int32,"cnameAsDomain":bool,"includingLnNodes":bool},"providers":[{"id","name","type","typeName"}],"providerTypes":[{"code","name"}]}`<br>未配置 DNS 时 `domainId/providerId` 为 0,`providerType` 为空串<br>错误:F:`invalid cluster id`;F:`node cluster not found`;A | 读取某集群的 DNS 设置,附带全部启用的服务商和服务商类型,供表单下拉使用。`providers` 与 `providerTypes` 不分页。 | 需登录+DNS权限 |
| 服务商域名下拉 | GET | `/api/admin/dns/provider-domains` | 查询参数:<br>`providerId` int64 必填,须 >0 | 200 `{"domains":[{"id":int64,"name":string}]}`<br>错误:F:`invalid DNS provider id`;A | 列出某服务商下可选的域名。只保留 `isOn` 为真、`isUp` 为真、未删除的域名,按 `name` 字符串升序。不分页。 | 需登录+DNS权限 |
| 更新集群 DNS 设置 | POST | `/api/admin/dns/clusters/update` | 表单参数:<br>`csrfToken` string 必填<br>`clusterId` int64 必填,须 >0<br>`dnsName` string,可选,集群在域名下的子域前缀<br>`domainId` int64,可选,0 表示不绑定<br>`nodesAutoSync` bool,默认 false<br>`serversAutoSync` bool,默认 false<br>`cnameRecords` string[],可选<br>`ttl` int32,可选,0 表示默认(后端取值规则待确认)<br>`cnameAsDomain` bool,默认 false<br>`includingLnNodes` bool,默认 false | 200 `{}`<br>错误:F:`invalid cluster id`;A:RPC 失败(名称/域名合法性由后端校验,具体文案待确认);403 CSRF 失败(纯文本) | 整体覆盖集群的 DNS 设置。处理器**不校验** `dnsName`、`domainId`、`ttl`、`cnameRecords` 的格式与范围,全部交给后端。所有字段整体写入,漏传布尔即被改为 false。 | 需登录+DNS权限+CSRF |
| 同步集群 DNS | POST | `/api/admin/dns/clusters/sync` | 表单参数:<br>`csrfToken` string 必填<br>`clusterId` int64 必填,须 >0 | 200 `{}`<br>错误:F:`invalid cluster id`;F:`DNS is not configured for this cluster`(未绑定域名);F:`DNS configuration contains issues that must be fixed first`(后端返回 shouldFix);F:后端返回的 `Error` 原文(同步失败且无 shouldFix 时);A | 先读集群 DNS 配置,再对其绑定域名执行 DNS 数据同步。F 类的 `Error` 原文来自后端,未经过 A 类的敏感信息净化(见异常说明)。 | 需登录+DNS权限+CSRF |
| DNS 服务商列表 | GET | `/api/admin/dns/providers` | 查询参数:<br>`page` int64,默认 1<br>`size` int64,默认 20,上限 100<br>`keyword` string,可选<br>`domain` string,可选,先转小写、去空白、去前缀 `www.` 后再查询<br>`providerType` string,可选,服务商类型代码 | 200 `{"items":[{"id":int64,"name":string,"type":string,"typeName":string,"minTTL":int32,"dataUpdatedTime":string,"countDomains":int64}],"providerTypes":[{"code","name","description"}],"total":int64,"page":int64,"size":int64}`<br>错误:A | 分页列出启用的 DNS 服务商,并带每个服务商的域名数量与全部服务商类型。查询带上当前管理员 ID(`AdminId`),是否按管理员做可见性过滤由后端决定(待确认)。 | 需登录+DNS权限 |
| DNS 服务商表单 | GET | `/api/admin/dns/providers/form` | 查询参数:<br>`providerId` int64,可选;`<=0` 时视为新建表单 | 新建:200 `{"providerTypes":[{"code","name","description"}]}`<br>编辑:再加 `"provider":{"id":int64,"name":string,"type":string,"typeName":string,"minTTL":int32,"params":object}`,`params` 为后端脱敏后的 API 参数 JSON 对象(键集因服务商类型而异,脱敏规则待确认)<br>错误:F:`DNS provider not found`;A:参数 JSON 解析失败或 RPC 失败 | 返回服务商类型列表;给了有效 `providerId` 时同时返回该服务商详情和脱敏参数。这里**不**调用 `validateMutationID`,`providerId<=0` 不报错。 | 需登录+DNS权限 |
| DNS 服务商详情 | GET | `/api/admin/dns/providers/detail` | 查询参数:<br>`providerId` int64 必填,须 >0<br>`page` int64,默认 1<br>`size` int64,默认 20,上限 100<br>`filter` string,可选:`deleted` 只看已删除域名;`down` 只看不可用域名;其他任何值(含空)表示不过滤。严格区分大小写,不做 Trim 以外的处理 | 200 `{"provider":{"id","name","type","typeName","minTTL","dataUpdatedTime"},"domains":[{"id":int64,"name":string,"isOn":bool,"isUp":bool,"isDeleted":bool,"dataUpdatedTime":string,"countRoutes":int,"countServerRecords":int64,"serversChanged":bool,"countNodeRecords":int64,"nodesChanged":bool,"countClusters":int64,"countAllNodes":int64,"countAllServers":int64}],"total":int64,"page":int64,"size":int64}`<br>错误:F:`invalid DNS provider id`;F:`DNS provider not found`;A | 分页列出某服务商下的域名及解析统计。`countRoutes` 为线路条数。此接口不返回服务商 API 参数。 | 需登录+DNS权限 |
| DNS 域名表单 | GET | `/api/admin/dns/domains/form` | 查询参数:<br>`providerId` int64,新建时必填(须 >0)<br>`domainId` int64,可选;>0 时为编辑,此时 `providerId` 被忽略,改取域名所属的服务商 | 200 `{"provider":{"id","name","type","typeName"}}`;`domainId>0` 时再加 `"domain":{"id":int64,"name":string,"isOn":bool,"isDeleted":bool}`<br>错误:F:`DNS domain not found`;F:`invalid DNS provider id`;F:`DNS provider not found`;A | 返回域名新建/编辑表单所需的服务商信息,编辑时附带域名本身。服务商查询带 `MaskParams`,但返回里不含参数。 | 需登录+DNS权限 |
| DNS 域名详情 | GET | `/api/admin/dns/domains/detail` | 查询参数:<br>`domainId` int64 必填,须 >0 | 200 `{"domain":{"id","name","isOn","isUp","isDeleted","dataUpdatedTime","countNodeRecords","nodesChanged","countServerRecords","serversChanged","countClusters","countAllNodes","countAllServers"},"provider":{"id","name","type","typeName"},"routes":[{"name","code"}],"clusters":[{"id":int64,"name":string,"dnsName":string,"defaultRoute":string,"isResolved":bool}],"nodes":[{"id","name","ipAddr","ipAddrId","clusterId","clusterName","dnsName","route":{"name","code"},"isResolved":bool}],"servers":[{"id","name","dnsName","clusterId","clusterName","isResolved":bool}]}`<br>服务商查不到时 `provider` 为 `{"id":<域名所属服务商ID>,"name":"-","type":"","typeName":"-"}`<br>错误:F:`invalid DNS domain id`;F:`DNS domain not found`;A | 返回域名及使用该域名的集群、节点、站点,并逐项检查解析记录是否已存在(`isResolved`)。每个集群检查 A 与 AAAA 记录(命中即停),每个节点的每条线路检查一次 A 或 AAAA,每个站点检查一次 CNAME。未设置线路的节点按集群默认线路补一条名为"默认线路"的记录。检查次数随集群、节点、站点规模线性增长且串行执行(性能风险,见异常说明),任何一次 RPC 失败整个请求失败。节点只含已安装节点。 | 需登录+DNS权限 |
| 创建 DNS 服务商 | POST | `/api/admin/dns/providers/create` | 表单参数:<br>`csrfToken` string 必填<br>`name` string<br>`type` string,服务商类型代码<br>`minTTL` int32,可选<br>`paramsJSON` string(JSON 文本,按字节原样传给后端),各类型所需键不同 | 200 `{"providerId":int64}`<br>错误:A:RPC 失败(名称/类型/参数合法性全由后端校验,具体文案待确认);403 CSRF 失败 | 新建一个 DNS 服务商。处理器**不做任何字段校验**,也不检查 `paramsJSON` 是不是合法 JSON。 | 需登录+DNS权限+CSRF |
| 更新 DNS 服务商 | POST | `/api/admin/dns/providers/update` | 表单参数:<br>`csrfToken` string 必填<br>`providerId` int64 必填,须 >0<br>`name` string<br>`minTTL` int32,可选<br>`paramsJSON` string(JSON 文本) | 200 `{}`<br>错误:F:`invalid DNS provider id`;A | 更新服务商名称、最小 TTL 和 API 参数。不带 `type`,类型不可改。`paramsJSON` 为空时后端是保留旧值还是清空:待确认,处理器直接透传。 | 需登录+DNS权限+CSRF |
| 同步服务商域名 | POST | `/api/admin/dns/providers/sync` | 表单参数:<br>`csrfToken` string 必填<br>`providerId` int64 必填,须 >0 | 200 `{"hasChanges":bool}`<br>错误:F:`invalid DNS provider id`;A:远端 DNS 服务商调用失败等 | 从服务商拉取最新域名列表并同步到本地,`hasChanges` 表示是否有变化。**不写操作日志**(处理器中没有 `CreateLogInfo`)。 | 需登录+DNS权限+CSRF |
| 删除 DNS 服务商 | POST | `/api/admin/dns/providers/delete` | 表单参数:<br>`csrfToken` string 必填<br>`providerId` int64 必填,须 >0 | 200 `{}`<br>错误:F:`invalid DNS provider id`;A(有集群仍引用时后端是否拒绝:待确认) | 删除(停用)服务商。是否有引用检查由后端决定,处理器无前置检查。 | 需登录+DNS权限+CSRF |
| 创建 DNS 域名 | POST | `/api/admin/dns/domains/create` | 表单参数:<br>`csrfToken` string 必填<br>`providerId` int64<br>`name` string,域名 | 200 `{"domainId":int64}`<br>错误:A:RPC 失败(格式/重复等校验在后端,文案待确认) | 在某服务商下新建域名记录。处理器**不校验** `providerId` 是否 >0,也不校验域名格式。 | 需登录+DNS权限+CSRF |
| 更新 DNS 域名 | POST | `/api/admin/dns/domains/update` | 表单参数:<br>`csrfToken` string 必填<br>`domainId` int64 必填,须 >0<br>`name` string<br>`isOn` bool,默认 false | 200 `{}`<br>错误:F:`invalid DNS domain id`;A | 修改域名名称和启停。`isOn` 漏传即被置为停用。 | 需登录+DNS权限+CSRF |
| 同步域名 DNS 数据 | POST | `/api/admin/dns/domains/sync` | 表单参数:<br>`csrfToken` string 必填<br>`domainId` int64 必填,须 >0 | 200 `{}`<br>失败(后端返回 `IsOk=false`)时 F 类:`message` 为后端 `Error` 原文,`data` 为 `{"shouldFix":bool}`,`shouldFix=true` 表示需先修复节点 DNS 问题(见 `GET /dns/issues`)<br>其他错误:F:`invalid DNS domain id`;A | 把该域名下集群/节点/站点的解析记录同步到服务商。请求固定带 `CheckNodeIssues=true`,会先检查节点问题。此接口的失败是 HTTP 200 加 body `code:400`(F 类默认),调用方必须看 `code` 而不是 HTTP 状态。 | 需登录+DNS权限+CSRF |
| 删除 DNS 域名 | POST | `/api/admin/dns/domains/delete` | 表单参数:<br>`csrfToken` string 必填<br>`domainId` int64 必填,须 >0 | 200 `{}`<br>错误:F:`invalid DNS domain id`;A | 删除域名,软删除(`isDeleted`),可由"恢复"接口还原。 | 需登录+DNS权限+CSRF |
| 恢复 DNS 域名 | POST | `/api/admin/dns/domains/recover` | 表单参数:<br>`csrfToken` string 必填<br>`domainId` int64 必填,须 >0 | 200 `{}`<br>错误:F:`invalid DNS domain id`;A | 恢复已删除的域名。 | 需登录+DNS权限+CSRF |
| 节点 DNS 修复表单 | GET | `/api/admin/dns/issues/node-form` | 查询参数:<br>`clusterId` int64 必填,须 >0<br>`nodeId` int64 必填,须 >0<br>`ipAddrId` int64,可选,不校验;用于定位节点的具体 IP | 200 `{"node":{"id":int64,"name":string,"ipAddr":string,"ipAddrId":int64,"clusterId":int64,"clusterDNSName":string,"domainId":int64,"domainName":string},"routes":[{"name":string,"code":string,"selected":bool}]}`<br>节点未绑定域名(`domainId<=0`)时 `routes` 为空数组<br>错误:F:`invalid node cluster id`;F:`invalid node id`;F:`node not found`;A | 返回修复节点 DNS 所需的节点信息与该域名的全部线路,`selected` 标记节点当前已选线路。 | 需登录+DNS权限 |
| 修复节点 DNS | POST | `/api/admin/dns/issues/node-update` | 表单参数:<br>`csrfToken` string 必填<br>`nodeId` int64 必填,须 >0<br>`ipAddr` string 必填,须是合法 IP(先去空白)<br>`ipAddrId` int64,可选<br>`domainId` int64,可选,线路只有在 `domainId>0` 时才生效<br>`routes` string[],可选,线路 code,空项忽略 | 200 `{}`<br>错误:F:`invalid node id`;F:`invalid node IP address`;A | 更新节点的 DNS 解析 IP、绑定域名和线路。线路提交给后端时拼成 `code@domainId`。`domainId<=0` 时线路被全部丢弃,等于清空线路。 | 需登录+DNS权限+CSRF |
| DNS 问题列表 | GET | `/api/admin/dns/issues` | 无参数 | 200 `{"issues":[{"target":string,"targetId":int64,"type":string,"description":string,"params":<后端原样>,"mustFix":bool}]}`<br>`params` 的具体类型(pb 字段原样输出):待确认<br>错误:A | 返回当前所有 DNS 配置问题(例如节点缺少 IP 或线路),`mustFix` 为真时会阻止同步。不分页。 | 需登录+DNS权限 |

## 公共结构

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

## 异常与待确认

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
