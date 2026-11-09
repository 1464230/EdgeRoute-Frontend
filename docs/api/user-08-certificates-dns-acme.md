# 证书 / DNS 服务商 / ACME 自动签发(用户端)
> 路由总数 27,来源 EdgeUserPortal/internal/web/server.go

说明:
- 所有错误统一返回 `{"error":"消息"}`;未登录统一 401 `请先登录`。
- 标注"需登录+CSRF写保护"的接口 = 路由包了 `sensitiveUserWrite`(管理员协助会话返回 403 `管理员协助会话不能修改用户资源或安全配置`;平台授权失效返回 403 只读提示;未登录 401),且处理函数内再用 `validCSRF` 校验请求体 `csrfToken`(不匹配 403 `CSRF 校验失败`)。
- 请求体统一使用 `decodeAPIJSON`/`decodeAPIJSONLimit`:禁止未知字段、只允许单个 JSON 对象,默认体积上限 64KiB;格式错误/超限返回 400。
- "CSRF体" 指仅含 `csrfToken` 的 JSON 请求体,见"公共结构"。

| 接口名称 | 方法 | 路径 | 主要请求参数 | 返回数据结构 | 功能说明 | 认证 |
|---|---|---|---|---|---|---|
| 证书列表 | GET | `/api/certificates` | 查询参数(均可选,任一出现即启用分页):<br>`offset` int64 ≥0,默认0<br>`size` int64 1~100,默认20<br>`q` string 关键字,≤200字符 | 200:<br>`certificates` Certificate[]<br>分页时额外:`count` int64 总数、`offset`、`size`<br>错误:400 `证书列表分页或搜索参数无效`;502 `读取证书失败` | 不带分页参数时返回全部证书;带分页/搜索参数时走分页接口。 | 需登录 |
| 上传证书 | POST | `/api/certificates` | JSON 请求体(上限512KiB):<br>`csrfToken` string 必填<br>`certData` string 必填 PEM 证书(≤384KiB)<br>`keyData` string 必填 PEM 私钥(≤128KiB)<br>`name` string 可选,≤100字符,缺省取首个域名(多域名追加" 等 N 个域名"),再缺省为"手动证书"<br>`isOn` bool 可选,默认 true | 201:`{"id": int64}`(新证书ID)<br>错误:400 `请输入证书和私钥`/`证书或私钥文件过大`/`证书或私钥无效: …`/`证书已经过期`/名称超长;403 CSRF;502 `上传证书失败` | 上传自有证书和私钥,服务端解析校验有效期,已过期证书拒绝。首个 DNS 名写入 `serverName`。 | 需实名(`requireIdentity`)+需登录+CSRF写保护 |
| 证书详情 | GET | `/api/certificates/{certificateID}` | 路径参数:`certificateID` int64 >0 | 200:`{"certificate": Certificate}`<br>错误:400 `证书 ID 无效`;404 `证书不存在`;502 `读取证书失败` | 查询单个证书的元信息,不含私钥内容。 | 需登录 |
| 下载证书(公钥链) | GET | `/api/certificates/{certificateID}/download` | 路径参数:`certificateID` int64 >0 | 200:二进制附件 `certificate-{id}.pem`,`Content-Type: application/x-pem-file`,仅含 `CERTIFICATE` 类型 PEM 块(不含私钥)<br>错误:400;404 `证书不存在`;422 `证书内容无效`;502 `读取证书失败` | 下载证书公钥链 PEM;响应带 `Cache-Control: no-store`。成功时不是 JSON。 | 需登录(无 CSRF,GET) |
| 导出证书私钥 | POST | `/api/certificates/{certificateID}/private-key/download` | 路径参数:`certificateID` int64 >0<br>JSON 请求体(上限4KiB):<br>`csrfToken` string 必填<br>`password` string 必填,≤512<br>`otpCode` string 必填(实际要求非空),≤32 | 200:二进制附件 `private-key-{id}.pem`(`application/x-pem-file`,`no-store`)<br>错误:400 `请输入当前密码和有效的验证码`;403 管理员协助会话/CSRF/`请先启用多因素认证再导出私钥`/`请输入多因素认证验证码`/`密码或验证码错误`;404;422 `私钥内容无效`/`证书与私钥不匹配`;429 `验证尝试过多,请稍后再试`;502;503 `审计服务不可用,已取消私钥导出` | 高风险操作:必须启用 MFA,并重新校验密码+动态码;按用户限流(`certificate-export:{userID}`);管理员协助会话禁止;写入审计日志失败则取消导出。路由未包 `sensitiveUserWrite`,CSRF 在处理函数内校验。 | 需登录+CSRF(体内)+密码+MFA;管理员协助会话禁止 |
| 修改证书 | PATCH | `/api/certificates/{certificateID}` | 路径参数:`certificateID`<br>JSON 请求体:<br>`csrfToken` string 必填<br>`name` string 可选,≤100字符<br>`isOn` bool 可选<br>(`name` 与 `isOn` 必须且只能传一个) | 200:`{"ok": true}`<br>错误:400 `必须且只能修改一个证书字段`/名称超长;403;404 `证书不存在`;502 `更新证书失败` | 重命名或启用/停用证书,一次只能改一项。 | 需登录+CSRF写保护 |
| 删除证书 | DELETE | `/api/certificates/{certificateID}` | 路径参数:`certificateID`<br>CSRF体 | 200:`{"ok": true}`<br>错误:400;403;502 `删除证书失败` | 删除证书。 | 需登录+CSRF写保护 |
| DNS服务商列表 | GET | `/api/dns-providers` | 无 | 200:`{"providers": DNSProvider[]}`(仅返回受支持类型)<br>错误:502 `读取 DNS 服务商失败`(可能附 RPC 细节) | 列出当前用户的 DNS 服务商配置,过滤掉不支持的类型。 | 需登录 |
| 创建DNS服务商 | POST | `/api/dns-providers` | JSON 请求体(上限16KiB),结构 DNSProviderWrite:<br>`csrfToken` string 必填<br>`name` string 必填,≤100字符<br>`type` string 可选,缺省 `cloudFlare`;枚举 `cloudFlare`/`dnspod`/`alidns`/`huaweiDNS`/`dnsla`/`volcengine`<br>`authMode` string 仅 cloudFlare 有效:空或 `token`<br>`apiParams` object 凭据,见公共结构 DNSProviderParams<br>`minTTL` int32,0 视为60,范围60~86400 | 201:`{"id": int64}`<br>错误:400(`不支持的 DNS 服务商类型`/`请输入配置名称`/`最小 TTL 必须在 60 到 86400 秒之间`/缺少凭据提示等);403;502 `创建 DNS 服务商失败` | 创建时按类型强校验凭据必填:cloudFlare 需 `apiToken`(≤512);alidns/huaweiDNS/volcengine 需 `accessKeyId`+`accessKeySecret`;dnspod 同(`apiType` 固定 `tencentDNS`);dnsla 需 `apiId`+`secret`。 | 需实名(`requireIdentity`)+需登录+CSRF写保护 |
| DNS服务商详情 | GET | `/api/dns-providers/{providerID}` | 路径参数:`providerID` int64 >0 | 200:`{"provider": DNSProvider + "apiParams": DNSProviderParams}`<br>错误:400 `DNS 服务商 ID 无效`;404 `DNS 服务商不存在`;502 `DNS 服务商配置无效`/读取失败 | 返回服务商基础信息及 `apiParams`。待确认:`apiParams` 中密钥字段是否已由 RPC 层脱敏,处理函数仅做 JSON 反序列化直接返回。 | 需登录 |
| 修改DNS服务商 | PATCH | `/api/dns-providers/{providerID}` | 路径参数:`providerID`<br>JSON 请求体(上限16KiB)同 DNSProviderWrite;更新时凭据可省略(由后端 RPC 层恢复已存密钥);`type` 缺省取现有类型 | 200:`{"ok": true}`<br>错误:400(校验同创建、`不能修改 DNS 服务商类型,请新建其他类型的配置`);403;404 `DNS 服务商不存在`;502 `更新 DNS 服务商失败` | 修改名称/凭据/最小TTL;类型不可变;`name` 仍为必填。 | 需登录+CSRF写保护 |
| 删除DNS服务商 | DELETE | `/api/dns-providers/{providerID}` | 路径参数:`providerID`<br>CSRF体 | 200:`{"ok": true}`<br>错误:400;403;502 `删除 DNS 服务商失败`(可能附 RPC 细节) | 删除 DNS 服务商配置。 | 需登录+CSRF写保护 |
| ACME选项汇总 | GET | `/api/acme/options` | 无 | 200:<br>`providers` ACMEProvider[] 可用证书服务商<br>`users` ACMEIdentity[] ACME 身份<br>`accounts` ACMEProviderAccount[] EAB 账户<br>`dnsProviders` DNSProvider[](仅受支持类型)<br>错误:502(各子查询分别提示) | 一次性返回创建自动签发任务所需的下拉数据。 | 需登录 |
| ACME身份列表 | GET | `/api/acme/identities` | 无 | 200:`{"identities": ACMEIdentity[]}`<br>错误:502 `读取 ACME 身份失败` | 列出 ACME 账号(身份)。 | 需登录 |
| 创建ACME身份 | POST | `/api/acme/identities` | JSON 请求体(上限64KiB):<br>`csrfToken` string 必填<br>`email` string 必填,≤254,合法邮箱<br>`description` string 可选,≤2000字符,允许换行/制表,不含其他控制字符<br>`providerCode` string 可选,缺省 `letsencrypt`(小写化)<br>`accountId` int64 当服务商需要 EAB 时必填,须属于同一服务商;否则忽略 | 201:`{"id": int64}`<br>错误:400 `请输入有效的联系邮箱`/`ACME 身份说明无效`/`证书服务商不存在`/`请选择当前服务商的 EAB 账户`;403;502 `创建 ACME 身份失败` | 注册 ACME 账号;服务商 `requireEAB=true` 时必须绑定其 EAB 账户。 | 需登录+CSRF写保护 |
| 修改ACME身份说明 | PATCH | `/api/acme/identities/{identityID}` | 路径参数:`identityID` int64 >0<br>JSON 请求体(上限16KiB):<br>`csrfToken` string 必填<br>`description` string ≤2000字符(同上规则;可为空字符串) | 200:`{"ok": true}`<br>错误:400 `ACME 身份 ID 无效`/`ACME 身份说明无效`;403;404 `ACME 身份不存在`;502 | 仅允许修改身份备注。 | 需登录+CSRF写保护 |
| 删除ACME身份 | DELETE | `/api/acme/identities/{identityID}` | 路径参数:`identityID`<br>CSRF体(上限8KiB) | 200:`{"ok": true}`<br>错误:400;403;404 `ACME 身份不存在`;409 `ACME 身份仍被签发任务使用` | 删除前确认归属;RPC 删除失败一律按 409 返回(代码未区分失败原因)。 | 需登录+CSRF写保护 |
| EAB账户列表 | GET | `/api/acme/provider-accounts` | 无 | 200:`{"accounts": ACMEProviderAccount[]}`<br>错误:502 `读取 ACME EAB 账户失败` | 列出 EAB(外部账户绑定)账户;不回显 EAB Key,仅 `hasEABKey`。 | 需登录 |
| 创建EAB账户 | POST | `/api/acme/provider-accounts` | JSON 请求体(上限16KiB):<br>`csrfToken` string 必填<br>`name` string 必填,≤120字符,无控制字符<br>`providerCode` string 可选,缺省 `letsencrypt`;必须是 `requireEAB` 的服务商<br>`eabKid` string 必填,≤1024<br>`eabKey` string 必填,≤4096 | 201:`{"id": int64}`<br>错误:400 `ACME EAB 账户名称无效`/`EAB KID 无效`/`EAB HMAC Key 无效`/`请选择需要 EAB 的证书服务商`/`请填写 EAB KID 和 HMAC Key`;403;502 | 保存 EAB KID/HMAC Key,供需要 EAB 的服务商注册身份。 | 需登录+CSRF写保护 |
| 修改EAB账户 | PATCH | `/api/acme/provider-accounts/{accountID}` | 路径参数:`accountID` int64 >0<br>JSON 请求体(上限16KiB):<br>`csrfToken` string 必填<br>`name` string 必填,≤120<br>`eabKid` string 必填,≤1024<br>`eabKey` string 可选(更新时可空,空则不改),≤4096 | 200:`{"ok": true}`<br>错误:400(同创建校验,服务商取自现有账户,不可改);403;404 `ACME EAB 账户不存在`;502 | 修改名称/KID/Key;`providerCode` 不在请求体中,不可变更。 | 需登录+CSRF写保护 |
| 删除EAB账户 | DELETE | `/api/acme/provider-accounts/{accountID}` | 路径参数:`accountID`<br>CSRF体(上限8KiB) | 200:`{"ok": true}`<br>错误:400;403;404 `ACME EAB 账户不存在`;409 `ACME EAB 账户仍被签发身份使用` | 删除 EAB 账户;RPC 失败一律按 409 返回。 | 需登录+CSRF写保护 |
| 自动签发任务列表 | GET | `/api/acme/tasks` | 无 | 200:`{"tasks": ACMETask[]}`<br>错误:502 `读取自动签发任务失败` | 列出所有自动签发(ACME)任务,不分页。 | 需登录 |
| 创建自动签发任务 | POST | `/api/acme/tasks` | JSON 请求体(上限64KiB):<br>`csrfToken` string 必填<br>`domains` string[] 必填,1~100个,小写去重;通配符 `*.x` 仅限 DNS 验证<br>`authType` string 可选,`http`(缺省)/`dns`<br>`dnsDomain` string DNS 验证必填(根域名),`domains` 必须属于它;http 时忽略<br>`dnsProviderId` int64 DNS 验证必填且须属于当前用户<br>`autoRenew` bool<br>`acmeUserId` int64 >0 时复用该身份;否则按下列字段查找或新建身份:<br>`providerCode`(缺省 `letsencrypt`)、`email`(必填、合法)、`description`、`accountId`(服务商需 EAB 时必填) | **202**:`{"id": int64}`(任务ID,异步签发)<br>错误:400 `验证方式无效`/`域名数量必须在 1 到 100 个之间`/`请输入有效的 DNS 根域名`/`通配符证书必须使用 DNS 验证`/`域名 "…" 格式无效`/`域名 "…" 不属于 DNS 根域名 …`/`ACME 账号不存在`/`证书服务商不存在`/`DNS 服务商不存在`;403;502 `创建自动签发任务失败`/`创建 ACME 账号失败` | 创建任务并异步(`Async:true`)签发;未指定 `acmeUserId` 时会按"服务商+邮箱+EAB账户"复用或自动创建 ACME 身份。 | 需登录+CSRF写保护 |
| 自动签发任务详情 | GET | `/api/acme/tasks/{taskID}` | 路径参数:`taskID` int64 >0 | 200:`{"task": ACMETask}`<br>错误:400 `自动签发任务 ID 无效`;404 `自动签发任务不存在`;502 | 通过遍历任务列表按 ID 查找。 | 需登录 |
| 修改自动续签 | PATCH | `/api/acme/tasks/{taskID}` | 路径参数:`taskID`<br>JSON 请求体:<br>`csrfToken` string 必填<br>`autoRenew` bool 必填(缺失 400 `缺少自动续签状态`) | 200:`{"ok": true}`<br>错误:400;403;404 `自动签发任务不存在`;502 `更新自动续签状态失败` | 仅允许切换自动续签开关,其他字段沿用原任务值回写。 | 需登录+CSRF写保护 |
| 立即执行签发任务 | POST | `/api/acme/tasks/{taskID}/run` | 路径参数:`taskID`<br>CSRF体 | 200:<br>`ok` bool 本次签发是否成功<br>`error` string 失败原因<br>`certificateId` int64 成功后的证书ID<br>错误:400;403;502 `执行自动签发任务失败` | 同步执行,超时 3 分钟;签发失败仍返回 200,须看 `ok`/`error`。 | 需登录+CSRF写保护 |
| 删除自动签发任务 | DELETE | `/api/acme/tasks/{taskID}` | 路径参数:`taskID`<br>CSRF体 | 200:`{"ok": true}`<br>错误:400;403;502 `删除自动签发任务失败` | 删除任务。 | 需登录+CSRF写保护 |

## 公共结构

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
