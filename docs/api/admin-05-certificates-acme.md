# 证书与 ACME(管理端)

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

| 接口名称 | 方法 | 路径 | 主要请求参数 | 返回数据结构 | 功能说明 | 认证 |
|---|---|---|---|---|---|---|
| 证书列表 | GET | `/api/admin/certificates` | `page` int 默认 1<br>`size` int 默认 20,最大 100<br>`keyword` string,可选,无长度限制<br>`filter` string,`all`(默认)/`available`/`expired`/`expiring7`/`expiring30`/`ca`<br>`scope` string,`platform`(默认)/`users`/`owner`<br>`userId` int64,`scope=owner` 时须 >0 | `items[Certificate]`(见公共结构);`total`;`page`;`size`;`filter`;`scope`;`userId`;`summary{all,available,expired,expiring7,expiring30,ca}`;`canDownloadPrivateKey` bool<br>错误:`invalid certificate filter`;`invalid certificate scope`;`invalid certificate owner` | 分页查询 SSL 证书,附各筛选分类的计数。列表中 `hasPrivateKey` 恒为 false。 | 需登录(server 模块) |
| 证书表单 | GET | `/api/admin/certificates/form` | `certId` int64,≤0 表示新建 | `certificate`:新建时为 null,否则为 `Certificate`;`canDownloadPrivateKey` bool<br>响应头 `Cache-Control: no-store` | 取得编辑表单数据。`hasPrivateKey` 取自 `len(KeyData)>0`,但查询未带私钥数据,可能恒为 false:待确认。 | 需登录(server 模块) |
| 证书详情 | GET | `/api/admin/certificates/detail` | `certId` int64 必填,>0 | `certificate`:`Certificate`;`canDownloadPrivateKey` bool<br>响应头 `Cache-Control: no-store`<br>错误:`invalid certificate id` | 查看证书详情,不返回证书原文与私钥。 | 需登录(server 模块) |
| 下载证书 | GET | `/api/admin/certificates/download` | `certId` int64 必填 | 成功:直接返回证书 PEM 原文,文件名 `certificate-<id>.pem`<br>错误(JSON):`certificate data is empty`;证书不存在/无效 ID 提示原文待确认 | 下载证书公钥链(不含私钥);记录日志。 | 需登录(server 模块) |
| 创建证书 | POST | `/api/admin/certificates/create` | `name` string<br>`description` string<br>`serverName` string<br>`isCA` bool<br>`isOn` bool<br>`certData` string(PEM)<br>`keyData` string(PEM)<br>均已去首尾空白,校验交给 RPC | `certificateId` int64<br>错误:RPC 校验原文,取值范围待确认 | 上传并创建 SSL 证书;记录日志。 | 需登录(server 模块)+CSRF |
| 更新证书 | POST | `/api/admin/certificates/update` | `certId` int64 必填<br>`name`/`description`/`serverName` string<br>`isCA`/`isOn` bool<br>`certData`/`keyData` string(PEM) | data 为空<br>错误:RPC 校验原文;证书/密钥留空时是否保留旧值取决于 RPC,待确认 | 修改证书信息,可替换证书内容;记录日志。 | 需登录(server 模块)+CSRF |
| 启停证书 | POST | `/api/admin/certificates/toggle` | `certId` int64 必填<br>`isOn` bool | data 为空 | 仅切换启用状态:沿用旧的 name/description/serverName/isCA 调用更新,不传证书数据;记录日志。 | 需登录(server 模块)+CSRF |
| 删除证书 | POST | `/api/admin/certificates/delete` | `certId` int64 必填 | data 为空<br>错误:被引用时的限制由 RPC 决定,待确认 | 删除证书;记录日志。 | 需登录(server 模块)+CSRF |
| 下载证书私钥 | POST | `/api/admin/certificates/private-key` | `certId` int64 必填 | 成功:直接返回私钥 PEM 原文,文件名 `private-key-<id>.pem`(响应头同公共约定)<br>错误(JSON):`仅超级管理员可以下载证书私钥`;`证书私钥为空` | 仅超级管理员可用。因含私钥,使用 POST 并强制 CSRF;记录日志。文档不示例私钥内容。 | 需登录(server 模块)+CSRF+超级管理员 |
| 重置证书 OCSP | POST | `/api/admin/certificates/ocsp/reset` | `certIds` int64 数组;处理器不校验是否为空,交给 RPC | data 为空<br>错误:RPC 错误原文 | 重置所选证书的 OCSP 状态,使其重新获取;记录日志。 | 需登录(server 模块)+CSRF |
| 忽略证书 OCSP 错误 | POST | `/api/admin/certificates/ocsp/ignore` | `certIds` int64 数组;不校验是否为空 | data 为空<br>错误:RPC 错误原文 | 忽略所选证书的 OCSP 错误;记录日志。 | 需登录(server 模块)+CSRF |
| ACME 任务列表 | GET | `/api/admin/acme/tasks` | `page` int 默认 1<br>`size` int 默认 20,最大 100<br>`keyword` string,≤200 且不含 CR/LF/NUL<br>`filter` string,空/`all`/`available`/`expired`/`expiring7`/`expiring30`(无 `ca`)<br>`scope` string,空/`platform`/`users`/`user`<br>`userId` int64,`scope=user` 时须 >0 | `items[ACMETask]`(见公共结构);`total`;`page`;`size`;`filter`;`scope`(空值回显 `platform`);`userId`;`summary{all,available,expired,expiring7,expiring30}`<br>错误:`invalid ACME task keyword`;`invalid ACME task filter`;`platform user is required`;`invalid ACME task scope` | 分页查询证书申请任务。 | 需登录(server 模块) |
| ACME 任务表单 | GET | `/api/admin/acme/tasks/form` | `taskId` int64,≤0 表示新建<br>`userId` int64 | `task`(新建时为 null);`providers`;`acmeUsers`;`dnsProviders[{id,name,type,typeName}]`;`userId`<br>响应头 `Cache-Control: no-store`<br>错误:`ACME task not found`<br>`providers`/`acmeUsers` 元素结构:待确认 | 取得任务编辑所需的任务、证书提供商、ACME 用户与 DNS 服务商选项。 | 需登录(server 模块) |
| 创建 ACME 任务 | POST | `/api/admin/acme/tasks/create` | `userId` int64<br>`authType` string,取值待确认<br>`acmeUserId` int64<br>`dnsProviderId` int64<br>`dnsDomain` string<br>`domains` string 数组<br>`autoRenew` bool<br>`authURL` string | `taskId` int64<br>错误:校验交给 RPC,原文待确认 | 创建证书申请任务;记录日志。 | 需登录(server 模块)+CSRF |
| 更新 ACME 任务 | POST | `/api/admin/acme/tasks/update` | `taskId` int64 必填<br>`acmeUserId` int64<br>`dnsProviderId` int64<br>`dnsDomain` string<br>`domains` string 数组<br>`autoRenew` bool<br>`authURL` string | data 为空<br>错误:RPC 校验原文,待确认 | 修改任务配置(不含 `authType`、`userId`);记录日志。 | 需登录(server 模块)+CSRF |
| 删除 ACME 任务 | POST | `/api/admin/acme/tasks/delete` | `taskId` int64 必填 | data 为空 | 删除任务;记录日志。 | 需登录(server 模块)+CSRF |
| 执行 ACME 任务 | POST | `/api/admin/acme/tasks/run` | `taskId` int64 必填 | `certificateId` int64<br>错误:ACME 返回的错误文本(`response.Error`)或 `ACME task failed` | 同步执行申请,耗时取决于 ACME 与 DNS 提供商。失败信息未经脱敏,会原样返回提供商错误;记录日志。 | 需登录(server 模块)+CSRF |
| ACME 账户列表 | GET | `/api/admin/acme/accounts` | `page` int 默认 1<br>`size` int 默认 20,最大 100 | `items[ACMEAccount]`(见公共结构);`total`;`page`;`size` | 分页列出证书提供商账户,不返回 EAB 密钥。 | 需登录(server 模块) |
| ACME 账户表单 | GET | `/api/admin/acme/accounts/form` | `accountId` int64,≤0 表示新建 | `account`(新建时为 null,否则 `ACMEAccount`);`providers[Provider]`<br>响应头 `Cache-Control: no-store`<br>错误:`ACME provider account not found` | 取得账户编辑表单与可选提供商。 | 需登录(server 模块) |
| 创建 ACME 账户 | POST | `/api/admin/acme/accounts/create` | `name` string<br>`providerCode` string<br>`eabKid` string<br>`eabKey` string(只写) | `accountId` int64<br>错误:校验交给 RPC,取值范围待确认 | 创建提供商账户(含 EAB 凭据);记录日志。 | 需登录(server 模块)+CSRF |
| 更新 ACME 账户 | POST | `/api/admin/acme/accounts/update` | `accountId` int64 必填<br>`name` string<br>`eabKid` string<br>`eabKey` string(只写) | data 为空<br>错误:RPC 校验原文,待确认 | 修改账户;`eabKey` 留空是否保留旧值取决于 RPC,待确认;记录日志。 | 需登录(server 模块)+CSRF |
| 删除 ACME 账户 | POST | `/api/admin/acme/accounts/delete` | `accountId` int64 必填 | data 为空 | 删除账户;记录日志。 | 需登录(server 模块)+CSRF |
| ACME 用户列表 | GET | `/api/admin/acme/users` | `page` int 默认 1<br>`size` int 默认 20,最大 100<br>`userId` int64,可选;0 的含义(全部或仅平台)取决于 RPC,待确认 | `items[ACMEUser]`(见公共结构);`total`;`page`;`size`;`userId` | 分页列出 ACME 用户(邮箱账号)。 | 需登录(server 模块) |
| ACME 用户表单 | GET | `/api/admin/acme/users/form` | `acmeUserId` int64,≤0 表示新建<br>`userId` int64 | `acmeUser`(新建时为 null);`providers`;`accounts`(最多 500 个);`userId`<br>响应头 `Cache-Control: no-store`<br>错误:`ACME user not found` | 取得 ACME 用户编辑所需的提供商与账户选项。 | 需登录(server 模块) |
| 创建 ACME 用户 | POST | `/api/admin/acme/users/create` | `userId` int64<br>`accountId` int64<br>`email` string<br>`providerCode` string<br>`description` string | `acmeUserId` int64<br>错误:校验交给 RPC,待确认 | 创建 ACME 用户;记录日志。 | 需登录(server 模块)+CSRF |
| 更新 ACME 用户 | POST | `/api/admin/acme/users/update` | `acmeUserId` int64 必填<br>`description` string | data 为空 | 仅可修改说明;记录日志。 | 需登录(server 模块)+CSRF |
| 删除 ACME 用户 | POST | `/api/admin/acme/users/delete` | `acmeUserId` int64 必填 | data 为空 | 删除 ACME 用户;记录日志。 | 需登录(server 模块)+CSRF |

## 公共结构

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

## 异常与疑似缺陷
- 证书下载与私钥下载成功时返回 PEM 原文,不是 JSON 包络。
- `acme/tasks/run` 失败时原样返回 ACME/提供商错误文本,未经脱敏。
- 证书列表用 `scope=owner`,ACME 列表用 `scope=user`;`keyword` 长度限制仅 ACME 有。
- 证书表单/详情的 `hasPrivateKey` 可能因查询未带私钥数据而恒为 false,待确认。
- CSRF 失败返回纯文本 403,与其余 JSON 错误格式不一致。
