# 安装向导、找回、确认与 CSRF 令牌(管理端)

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

| 接口名称 | 方法 | 路径 | 主要请求参数 | 返回数据结构 | 功能说明 | 认证 |
|---|---|---|---|---|---|---|
| 获取 CSRF 令牌 | GET | `/csrf/token` | 无 | `data.token` string,32 字节 base64 RawURL,绑定当前 session sid,有效期 1800 秒<br>错误(Fail):`请求速度过快，请稍后刷新后重试`<br>`Init()` 注入的 `tea*` 页面键是否出现在响应中:待确认 | 为后续需 CSRF 的表单/POST 签发令牌,作为 `csrfToken` 提交。**未登录时存在全局限速**:同一秒内第二次调用即被拒,限速按进程全局而非按 IP,可被他人占用。session 创建时机未核实,待确认。 | 无需登录(登录态可选) |
| 找回页面(静态文件) | GET | `/admin/recover.html` | 无 | HTML 文件 `public/admin/recover.html`,`Cache-Control: no-store`<br>文件缺失:503 纯文本 | 返回找回(恢复模式)前端页面。不受 recover.Helper 保护。 | 无需登录 |
| 找回入口 | GET | `/recover` | 无 | 同上:返回 `public/admin/recover.html`,`no-store`;缺失 503 纯文本 | 与 `/admin/recover.html` 内容相同,但受 recover.Helper 保护:非恢复模式时重定向 `/`。 | 无需登录(仅恢复模式可用) |
| 找回:校验 API 节点 | POST | `/recover/validateApi` | `protocol` string 必填(未校验取值)<br>`host` string 必填<br>`port` string 必填,纯数字<br>`nodeId` string 必填<br>`nodeSecret` string 必填<br>各值首尾的引号和空格会被剔除 | `data.apiNode`:`protocol`,`host`,`port`,`nodeId`,`nodeSecret`(回显调用者自己提交的值),`hosts[]`(该 API 节点下已启用节点的监听 host)<br>错误:`FailField("host", ...)` 等,具体文案待确认 | 用提交的凭据连接指定 API 节点,取出其监听地址列表,供下一步替换。**风险**:无 CSRF、无登录,仅靠 recover.Helper 与提交的节点凭据。 | 无需登录(仅恢复模式可用) |
| 找回:更新 hosts | POST | `/recover/updateHosts` | `protocol`,`host`,`port`,`nodeId`,`nodeSecret`(同上,但**无 Must 校验**,缺失时的行为待确认)<br>`oldHosts` string 数组<br>`newHosts` string 数组,长度须与 `oldHosts` 相等 | 成功无业务字段(`data` 内容待确认)<br>错误:`参数配置错误，请刷新页面后重试`(长度不等)等,其余待确认 | 在 API 节点 JSON 上做字符串 `ReplaceAll` 把旧 host 换成新 host,写入 `configs/<ConfigFileName>`,重载 RPC,并把 `teaconst.IsRecoverMode` 置 false(退出恢复模式)。**风险**:无 CSRF、无登录;替换为纯字符串替换,可能误改非 host 内容。 | 无需登录(仅恢复模式可用) |
| 安装向导页面(静态文件) | GET | `/admin/setup.html` | 无 | HTML 文件 `public/admin/setup.html`,`no-store`<br>缺失:503 纯文本 | 返回安装向导前端页面。不受 setup.Helper 保护,系统安装后依旧可访问。 | 无需登录 |
| 安装向导入口 | GET | `/setup` | 无 | 重定向到 `/admin/setup.html` | 跳转到安装向导页面。受 setup.Helper:已配置时重定向 `/`。 | 无需登录 |
| 安装向导入口(带斜杠) | GET | `/setup/` | 无 | 同 `/setup`(同一处理器 `IndexAction`) | 同上,仅路径多一个斜杠。 | 无需登录 |
| 安装向导初始数据 | GET | `/setup/bootstrap` | 无 | `data.bootstrap`:`currentHost`(仅当请求 Host 是非回环 IP 字面量时有值,否则 `""`),`licenseURL`,`version` | 为前端预填当前访问地址、授权获取地址与版本。无需口令。 | 无需登录 |
| 校验安装口令 | POST | `/setup/verifyToken` | `setupToken` string 必填 | 成功:`data` 为空对象 `{}`(`{"code":200,"message":"","data":{}}`)<br>失败:**HTTP 403 JSON** `安装口令无效，请查看服务启动日志中的 [SETUP] 口令` | 校验用户输入的安装口令,不产生副作用。 | 无需登录(受 setup.Helper) |
| 校验 API 节点安装参数 | POST | `/setup/validateApi` | `setupToken` string 必填<br>`mode` string 必填,只能为 `new`,否则 `仅支持新 API 节点安装`<br>`newPort` string 必填,纯数字,长度 4..5,值 1024..65534<br>`newHost` string 必填,IP 或主机名;纯数字点分但非合法 IP 的被拒绝<br>`licenseKey` string 必填,以 `ERK2-` 开头,≤256,无空白 | `data.apiNode`:`mode`,`newPort`,`newHost`,`licenseKey`,`licenseKeyHint`(前 9 个字符加 `****`)<br>错误:口令无效 403;各字段格式错误(文案待确认,仅确认 `仅支持新 API 节点安装`) | 校验即将安装的 API 节点参数并回显给前端,供最后一步 `install` 提交。**风险**:原样回显完整 `licenseKey`;`apiNode` 在校验前就已赋值,失败时 `data` 是否仍带 `apiNode` 待确认。 | 无需登录(口令 + setup.Helper) |
| 校验数据库连接 | POST | `/setup/validateDb` | `setupToken` string 必填<br>`host` string 必填,IP 或 `^[\w.-]+$`<br>`port` string 必填,纯数字<br>`database` string 必填<br>`username` string 必填,`^[\w\.-]+$`<br>`password` string<br>`accessLogKeepDays` int(取值范围待确认) | `data.db`:`host`,`port`,`database`,`username`,`passwordMask`(与密码等长的 `*`,会暴露密码长度),`accessLogKeepDays`<br>错误:连接失败、MySQL 版本低于 5.7.8 等,文案待确认 | 真实连接 MySQL(30 秒超时):库不存在(Error 1049)时自动 `CREATE DATABASE`;创建并删除测试表 `edgeTest1`;检查版本 ≥5.7.8。**副作用**:会创建数据库。**缺陷**:1049 分支内 `db, err :=` 遮蔽外层变量,内层连接未关闭(连接泄漏)。 | 无需登录(口令 + setup.Helper) |
| 校验管理员账号 | POST | `/setup/validateAdmin` | `setupToken` string 必填<br>`adminUsername` string 必填,`^[a-zA-Z0-9_]+$`,≤100<br>`adminPassword` string 必填,`^[a-zA-Z0-9_]+$`,trim 后长度 8..100<br>`adminPassword2` string 必填,须与 `adminPassword` 一致 | `data.admin`:`username`,`passwordMask`(等长 `*`)<br>错误:格式、长度、两次不一致(文案待确认) | 校验首个管理员的用户名与密码。仅允许字母数字下划线,不允许特殊字符的密码是已知限制。 | 无需登录(口令 + setup.Helper) |
| 执行安装 | POST | `/setup/install` | `setupToken` string 必填<br>`apiNodeJSON` string 必填(JSON 字符串)<br>`dbJSON` string 必填(JSON 字符串,含**明文** `password`)<br>`adminJSON` string 必填(JSON 字符串)<br>三者的具体键沿用前三个校验接口回显的对象(`apiNode`/`db`/`admin`) | 成功:`data.started` = true(安装异步执行,用 `/setup/status` 轮询)<br>错误:`已有安装任务正在执行，请等待当前安装完成`;JSON 解析错误;目录检查错误;失败文案会拼接命令 stdout/stderr 与日志路径 | 同步校验后异步安装,调用 `edgeroute-api setup`,授权码通过环境变量 `EDGEROUTE_LICENSE_KEY` 传递。**缺陷**:对 `newHost`/`newPort` 仅检查非空且不含 `\r\n"`,**不复核** `validateApi` 的格式与端口范围,`dbJSON` 各字段同样未再校验,可绕过前面几步直接提交。 | 无需登录(口令 + setup.Helper) |
| 安装状态 | POST | `/setup/status` | 无(**不要求口令**) | `data`:`statusText`,`running`,`installed`,`failed`,失败时可选 `error`(可能含命令输出与服务器路径)<br>失败路径外无其他错误 | 轮询安装进度。在 Helper 中被特判:无需口令,系统安装完成后依旧可调。**风险**:`error` 任何人可读,可能泄露路径与命令输出。 | 无需登录、无需口令 |
| 检测本机数据库 | POST | `/setup/detectDB` | `setupToken` string 必填 | `data.localDB`:`host`,`port`,`username`(固定 `""`),`password`(固定 `""`),`canInstall` bool | 依次尝试 127.0.0.1(原文 `127.0.1` 待核对)、localhost、172.20.0.2 的 3306 端口,每次 3 秒超时。`canInstall = linux && amd64 && os.Getgid()==0`,用的是**组 ID 而非 UID**,疑似缺陷。 | 无需登录(口令 + setup.Helper) |
| 判断是否本机 IP | POST | `/setup/checkLocalIP` | `host` string(**无需口令**,是否必填待确认) | `data.isLocal` bool<br>IP 无法解析时默认 true;私网判断函数 `utils.IsLocalIP` 未读,具体判定规则待确认 | 判断填写的数据库地址是否本机/内网,用于决定是否展示本机 MySQL 安装入口。 | 无需登录(受 setup.Helper) |
| 一键安装 MySQL | POST | `/setup/mysql/installPopup` | `setupToken` string 必填 | 成功:`data`:`isOk` = true,`user` = `root`,`password`(**生成的 root 明文密码**),`dir`<br>失败:仍返回 `code:200`,`data`:`isOk` = false,`err` string | **同步长耗时**:下载并安装 MySQL 到 `/usr/local/mysql`。调用时重置全局 `SharedLoger`(原样拼写)。**风险**:响应携带 root 明文密码;失败也返回 `code:200`,调用方必须看 `data.isOk`。 | 无需登录(口令 + setup.Helper) |
| MySQL 安装日志 | POST | `/setup/mysql/installLogs` | `setupToken` string 必填 | `data.logs` = `SharedLogger.ReadAll()` 的结果,元素结构待确认(未读 `mysqlinstallers/utils`) | 读取一键安装 MySQL 的进度日志,供前端轮询。 | 无需登录(口令 + setup.Helper) |
| 确认页面(静态文件) | GET | `/admin/confirm.html` | 无 | HTML 文件 `public/admin/confirm.html`<br>缺失:503 纯文本 | 返回"确认 API 节点"页面。不受 confirm.Helper 保护。 | 无需登录 |
| 确认页初始数据 | GET | `/setup/confirm/bootstrap` | 无 | `data.bootstrap`:`endpoints[]`(为空时为 `[""]`),`nodeId`,`canInstall`(读取配置失败时为 true) | 返回已有 API 配置,供确认页预填。受 confirm.Helper:非新安装时重定向 `/`。 | 无需登录(仅新安装待确认时可用) |
| 确认页面入口 | GET | `/setup/confirm` | 无 | 返回 `public/admin/confirm.html`(与 POST 同一处理器 `IndexAction`) | 同 `/admin/confirm.html`,受 confirm.Helper 保护。 | 无需登录(仅新安装待确认时可用) |
| 提交确认 | POST | `/setup/confirm` | `endpoints` string 数组,必填,每项为 URL,仅允许 http/https,缺端口补 80/443<br>`nodeId` string 必填<br>`secret` string 必填<br>`setupToken` string 必填(安装口令,见管理端启动日志 `[SETUP]` 行;er-v45 起)<br>(`Must` 声明了规则但**代码未使用**,实际校验由手写判断完成) | 成功无业务字段<br>错误:`请输入至少一个API节点地址`;`请输入NodeId`;`请输入Secret`;协议错误;连接失败;角色不是 `admin`(各文案待确认) | 用提交的凭据逐个连接 API 节点并要求角色为 `admin`,通过后写入配置(`RPCDisableUpdate=true`)并重载 RPC。er-v45 起 POST 需带安装口令,口令无效返回 403 `安装口令无效,请查看服务启动日志中的 [SETUP] 口令`;仍无登录、无 CSRF,并受 `IsNewInstalled` 限制;角色不符分支未调用 `client.Close()`(连接泄漏)。 | 无需登录(仅新安装待确认时可用) |
| 公开图片 | GET | `/ui/image/:fileId` | 路径参数:`fileId` int64 | 成功:图片字节流,`Content-Type` 按文件扩展名确定(缺省 `image/png`),`Last-Modified: Fri, 06 Sep 2019 08:29:50 GMT`(固定值),逐块输出<br>不存在或非公开:`NotFound` 走 `FailPage`,返回 HTML 错误页,HTTP 状态按错误映射(not found 为 404);若带 `X-Requested-With` 头则返回纯文本错误 | 输出被标记为公开(`IsPublic`)的文件,用于登录页 Logo 等。**风险**:不限定图片类型,任何公开文件都按扩展名输出,且未设 `X-Content-Type-Options: nosniff`;`Last-Modified` 为固定值会使更新后的图片被缓存。 | 无需登录 |

## 补充:前端性能上报 `/api/metrics`(3 条,不在路由清单 JSON 内)

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
