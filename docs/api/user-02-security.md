# 安全设置(用户端)

> 路由总数 11,来源 EdgeUserPortal/internal/web/server.go

通用约定:错误统一为 `{"error":"消息"}`;JSON 请求体不允许未知字段,上限 64KB;凡写操作(含 DELETE)均需在 JSON 请求体中携带 `csrfToken`(取自 `GET /api/session`)。`sensitiveUserWrite` 中间件额外拒绝管理员协助会话(403「管理员协助会话不能修改用户资源或安全配置」),授权确认失效时返回 403 只读提示,并记录一条通用操作审计日志。

| 接口名称 | 方法 | 路径 | 主要请求参数 | 返回数据结构 | 功能说明 | 认证 |
|---|---|---|---|---|---|---|
| 安全设置概览 | GET | `/api/security` | 无 | 200:<br>`mfa.enabled` bool 是否已启用 MFA<br>`mfa.recoveryCodeCount` int 剩余恢复码数<br>`oauthProviders` array,元素:`code` string、`name` string、`bound` bool 是否已绑定<br>错误:401 请先登录;502 读取安全设置失败/OAuth2 配置不可用/账号状态不可用 | 返回当前用户的 MFA 状态与各第三方登录绑定状态。 | 需登录 |
| 解绑第三方登录 | DELETE | `/api/security/oauth/{provider}` | 路径参数:`provider` string 提供方代码(转小写)<br>JSON 体:`csrfToken` string 必填 | 200:`ok` bool(true)<br>错误:400 invalid request body;403 CSRF token is invalid(此处消息为英文);404 OAuth2 provider is not configured;502 配置/解绑失败(英文消息) | 解除当前账号与指定第三方登录的绑定。 | 需登录 + CSRF 写保护 |
| 开始设置 MFA | POST | `/api/security/mfa/setup` | JSON 体:`csrfToken` string 必填 | 200:<br>`secret` string TOTP 密钥(32 位 base32)<br>`provisioningURI` string otpauth URI<br>`recoveryCodes` string[8],格式 `XXXX-XXXX-XXXXX`<br>错误:403 CSRF 校验失败;409 多因素认证已经启用;500 生成密钥/恢复码失败;502 | 生成 TOTP 密钥和 8 个恢复码返回给前端。服务端不保存,需调用 confirm 时原样回传。 | 需登录 + CSRF 写保护 |
| 确认启用 MFA | POST | `/api/security/mfa/confirm` | JSON 体:<br>`secret` string 必填,长度 32(转大写)<br>`currentPassword` string 必填,当前登录密码<br>`code` string 必填,当前 TOTP 动态码<br>`recoveryCodes` string[] 必填,必须恰好 8 个,去除 `-`/空格后每个 13 位且不重复<br>`csrfToken` string 必填 | 200:`ok` bool(true)<br>错误:400 动态验证码无效/恢复码无效,请重新开始设置;403 CSRF 校验失败;409 多因素认证已经启用;502 读取/保存安全设置失败 | 校验动态码后保存密钥并启用 MFA,恢复码以 SHA-256 哈希保存。 | 需登录 + CSRF 写保护 |
| 关闭 MFA | DELETE | `/api/security/mfa` | JSON 体:<br>`code` string 必填,当前 TOTP 动态码或恢复码<br>`csrfToken` string 必填 | 200:`ok` bool(true)(未启用 MFA 时也直接返回 ok)<br>错误:401 动态验证码或恢复码错误;403 CSRF 校验失败;502 | 风险操作:需提供 MFA 动态码或恢复码方可关闭 MFA。 | 需登录 + CSRF 写保护 |
| API 访问密钥列表 | GET | `/api/access-keys` | 无 | 200:`accessKeys` array,元素:<br>`id` int64<br>`isOn` bool 是否启用<br>`uniqueId` string 密钥 ID(AccessKey ID)<br>`description` string 用途说明<br>`accessedAt` 数值,最近使用时间(单位待确认:取自 RPC 字段 `AccessedAt`)<br>不含 secret<br>错误:401;502 读取 API 访问密钥失败 | 列出当前用户的 API 访问密钥。 | 需登录 |
| 创建 API 访问密钥 | POST | `/api/access-keys` | JSON 体:<br>`currentPassword` string 必填,当前登录密码<br>`otpCode` string,已启用 MFA 时必填<br>`description` string 必填,去空格后 1-100 个字符<br>`csrfToken` string 必填 | 201:`id` int64、`uniqueId` string、`secret` string(密钥仅在创建时返回)<br>错误:400 请输入不超过 100 个字符的用途说明;403 CSRF 校验失败/`{"error":"请先完成实名认证","code":"identity_required"}`;409 API 访问密钥数量已达到上限;502 | 创建密钥,每个用户最多 20 个(`maxUserAccessKeys`)。后台启用实名且该功能(`api_access_keys`)被限制时,未实名返回 403。 | 需登录 + 需实名 + CSRF 写保护 |
| 启停 API 访问密钥 | PATCH | `/api/access-keys/{accessKeyID}` | 路径参数:`accessKeyID` int64 必填,>0<br>JSON 体:<br>`isOn` bool 是否启用(未传按 false)<br>`csrfToken` string 必填 | 200:`ok` bool(true)<br>错误:400 API 访问密钥 ID 无效;403 CSRF 校验失败;404 API 访问密钥不存在或无权操作;502 更新 API 访问密钥失败 | 启用或停用指定密钥。 | 需登录 + CSRF 写保护 |
| 删除 API 访问密钥 | DELETE | `/api/access-keys/{accessKeyID}` | 路径参数:`accessKeyID` int64 必填,>0<br>JSON 体:`csrfToken` string 必填 | 200:`ok` bool(true)<br>错误:400 API 访问密钥 ID 无效;403;404 API 访问密钥不存在或无权操作;502 删除 API 访问密钥失败 | 删除指定密钥。 | 需登录 + CSRF 写保护 |
| 域名接入策略 | GET | `/api/domain-ownership/policy` | 无 | 200:`requireOwnership` bool 平台是否要求域名所有权验证(响应头 `Cache-Control: no-store`)<br>错误:401;503 暂时无法读取域名接入策略 | 供新建站点表单决定是否展示 TXT 验证步骤。创建站点的 API 仍会自行强制校验。 | 需登录 |
| 域名所有权验证 | POST | `/api/domain-ownership` | JSON 体:<br>`domain` string 必填,经 `domainpolicy.Normalize` 规范化并校验<br>`verify` bool,false 仅获取验证挑战,true 执行 DNS TXT 校验<br>`csrfToken` string 必填 | 200:Challenge 对象(`Cache-Control: no-store`):<br>`domain` string<br>`recordName` string TXT 记录名<br>`recordType` string<br>`value` string 记录值<br>`expiresAt` int64 挑战过期时间<br>`verifiedUntil` int64 验证有效期至<br>错误:400 域名不合法/不允许接入/域名验证未通过,请检查 TXT 记录、有效期或平台接入限制后重试;403 CSRF 校验失败;503 当前 API 尚不支持域名验证 | 生成或校验域名归属 TXT 挑战。时间戳单位待确认(源码未标注)。 | 需登录 + CSRF 写保护 |

## 说明
- 本模块仅 `POST /api/access-keys` 套用了 `requireIdentity`,其余接口无实名要求。
- 无任何接口使用 `licenseFeature`(财务授权)。
