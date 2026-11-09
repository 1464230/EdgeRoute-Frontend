# 认证与会话(用户端)

> 路由总数 17,来源 EdgeUserPortal/internal/web/server.go

通用约定:错误统一为 `{"error":"消息"}`;所有 JSON 请求体使用 `DisallowUnknownFields`(出现未知字段即 400「请求格式无效」),大小上限 64KB。本模块带 `csrfToken` 的 POST 接口需先 `GET /api/session` 获取会话 Cookie 与 `csrfToken`,再把 `csrfToken` 放入 JSON 请求体(不是请求头)。

| 接口名称 | 方法 | 路径 | 主要请求参数 | 返回数据结构 | 功能说明 | 认证 |
|---|---|---|---|---|---|---|
| 门户配置 | GET | `/api/portal` | 无 | 200:<br>`landingPageEnabled` bool 落地页开关<br>`productName` string<br>`siteName` string<br>`title` / `description` / `footerText` string<br>`privacyPolicyURL` / `termsURL` string<br>`logoURL` string(有 Logo 时为 `/api/portal/logo`,否则空)<br>`faviconURL` string(同理 `/api/portal/favicon`)<br>`showFinance` / `showVersion` bool<br>`version` string<br>`showTrafficCharts` / `showBandwidthCharts` bool<br>`defaultTrafficDays` int(7 或 31)<br>`bandwidthUnit` string<br>`requireIdentity` bool<br>错误:503 用户门户配置暂时不可用 | 返回登录页/门户展示配置。实名认证配置读取失败或未启用时 `requireIdentity` 为 false(失败即关闭)。 | 公开 |
| 门户 Logo | GET | `/api/portal/logo` | 请求头 `If-None-Match`(可选,ETag 命中返回 304) | 200:图片二进制(`Content-Type` 为 image/*,带 `ETag`、`Cache-Control: public, max-age=300`、`X-Content-Type-Options: nosniff`)<br>304:ETag 命中无正文<br>404:未配置/非图片(纯文本 404,非 JSON) | 输出后台配置的门户 Logo 文件;MIME 非图片则尝试按扩展名、内容嗅探,仍非图片则 404。 | 公开 |
| 门户 Favicon | GET | `/api/portal/favicon` | 同 Logo | 同 Logo(与 Logo 共用处理函数 `apiPortalAsset`,按路径后缀 `/favicon` 取 Favicon 文件) | 输出后台配置的站点图标。 | 公开 |
| 获取会话 | GET | `/api/session` | 无(依赖会话 Cookie) | 200:见「公共结构 SessionResponse」<br>错误:500 创建会话失败 | 返回当前会话状态;无有效会话时自动创建匿名会话(`theme` 默认 `light`)并写入会话 Cookie。前端启动时先调用以取得 `csrfToken`。 | 公开 |
| 注册配置 | GET | `/api/registration` | 无 | 200:<br>`enabled` bool 是否开放注册<br>`requireVerification` bool 注册需后台审核<br>`requireEmail` bool(恒为 true)<br>`requireMobile` bool<br>`passwordResetEnabled` bool<br>`captcha` 见「公共结构 Captcha」<br>错误:502 读取注册配置失败/读取安全验证配置失败 | 提供注册与找回密码页所需的开关和人机验证配置。 | 公开 |
| 用户注册 | POST | `/api/register` | JSON 体:<br>`username` string 必填,正则 `^[A-Za-z0-9_]{3,64}$`<br>`password` string 必填,8-128 位且含大小写字母、数字、符号<br>`email` string 必填,≤254,合法邮箱(会转小写)<br>`mobile` string 可选(后台强制时必填),≤32,仅 `0-9+(). -`<br>`captchaToken` string 人机验证令牌(启用验证码时必填)<br>`csrfToken` string 必填 | 201:<br>`registered` bool<br>`requireEmailVerification` bool<br>`requiresApproval` bool(仅需验证/审核时出现)<br>`autoLoginFailed` bool(未自动登录时出现)<br>`session` 见 SessionResponse<br>自动登录成功时仅含 `registered`/`requireEmailVerification`/`session`,并下发新会话 Cookie<br>错误:400 参数格式无效;403 CSRF 校验失败(含已登录)/未开放注册/需人机验证;409 邮箱或用户名已被使用;429 注册尝试过多;502 | 创建用户账号。无需邮箱验证且无需审核时自动登录(轮换会话);否则返回 201 并要求前端引导验证/等待审核。按客户端 IP 限流。 | 公开(需会话 CSRF,且当前会话未登录) |
| 校验邮箱验证码 | POST | `/api/email-verification/verify` | JSON 体:<br>`code` string 必填,1-128(验证链接中的 code)<br>`csrfToken` string 必填 | 200:`verified` bool(true)、`email` string<br>错误:400 验证链接无效或已过期;403 CSRF 校验失败;502 邮箱验证服务暂时不可用 | 用邮件中的验证码/链接完成邮箱验证。 | 公开(需会话 CSRF) |
| 重发验证邮件 | POST | `/api/email-verification/resend` | JSON 体:<br>`email` string 必填,合法邮箱,≤254<br>`csrfToken` string 必填 | 200:`sent` bool(true)<br>错误:400 电子邮箱格式无效;403;429 发送请求过多(本地限流或上游 temporarily limited);502 验证邮件发送失败 | 重新发送邮箱验证邮件,按客户端 IP 限流。 | 公开(需会话 CSRF) |
| 发送密码重置验证码 | POST | `/api/password-reset/send` | JSON 体:<br>`email` string 必填,合法邮箱<br>`captchaToken` string 人机验证令牌<br>`csrfToken` string 必填 | 200:`sent` bool、`codeLength` int(验证码位数)<br>错误:400 电子邮箱格式无效;403 CSRF 校验失败(含已登录)/请先完成人机验证;429 请求过于频繁;502 读取安全验证配置失败;503 密码重置邮件服务暂不可用 | 向邮箱发送找回密码验证码,限流键 `password-reset-send:<IP>`。 | 公开(需会话 CSRF,且当前会话未登录) |
| 确认重置密码 | POST | `/api/password-reset/confirm` | JSON 体:<br>`email` string 必填<br>`code` string 必填,4-32<br>`newPassword` string 必填,8-128 位且含大小写字母、数字、符号<br>`captchaToken` string<br>`csrfToken` string 必填 | 200:`ok` bool(true)<br>错误:400 邮箱格式无效/验证码或新密码格式无效/验证码无效或已过期;403 CSRF/人机验证/密码被锁定(PASSWORD_LOCKED);429 验证尝试过多;502 读取安全验证配置失败/密码重置服务暂不可用 | 校验验证码并设置新密码;成功后踢出该用户所有会话并重置限流计数。 | 公开(需会话 CSRF,且当前会话未登录) |
| 用户登录 | POST | `/api/login` | JSON 体:<br>`username` string 必填,≤200<br>`password` string 必填,≤512<br>`otpCode` string 可选,MFA 动态码或恢复码<br>`captchaToken` string 人机验证令牌<br>`csrfToken` string 必填 | 200:SessionResponse,并下发新会话 Cookie(会话轮换)<br>错误:400 请输入有效的用户名和密码;401 用户名或密码错误,若需 MFA 则返回 `{"error":"...","code":"mfa_required"}`;403 CSRF 校验失败/请先完成人机验证;429 登录尝试过多;502 暂时无法连接 API 服务/读取安全验证配置失败 | 账号密码登录(可带 MFA)。按客户端 IP 限流,成功后重置计数;登录 IP 透传给后端。 | 公开(需会话 CSRF) |
| OAuth 登录方式列表 | GET | `/api/oauth/providers` | 无 | 200:`providers` array,元素:<br>`code` string 提供方代码<br>`name` string 显示名<br>`startURL` string(`/oauth/<code>/start`)<br>按 `name` 排序<br>错误:502 读取登录配置失败 | 返回已启用并配置有效的第三方登录提供方(含 Telegram)。 | 公开 |
| OAuth 发起授权 | GET | `/oauth/{provider}/start` | 路径参数:`provider` string 提供方代码(不区分配置外的值,未配置则 404)<br>查询参数:`action` string 可选,`bind` 表示已登录用户绑定账号 | 非 JSON 接口:<br>普通 OAuth2:302 重定向到提供方授权地址(带 PKCE `code_challenge`、`state`)<br>Telegram:200 HTML 页(内嵌 Telegram Login Widget)<br>失败:303 重定向 `/login?oauth-error=<configuration/session/internal/provider>`<br>404:提供方未配置 | 浏览器跳转型接口,不是 AJAX。`action=bind` 时要求当前为已登录的普通用户会话(管理员协助会话不可绑定),`state` 与会话绑定防 CSRF。 | 公开(`action=bind` 需已登录) |
| OAuth 回调 | GET | `/oauth/{provider}/callback` | 路径参数:`provider`<br>查询参数:`state` string 必填<br>`code` string 授权码(非 Telegram 必填,≤4096)<br>`error` string 提供方返回的错误<br>Telegram 回调另含 `id`、`first_name`、`last_name`、`username`、`photo_url`、`auth_date`、`hash`(签名校验,`auth_date` 有效期 24 小时) | 非 JSON 接口,全部为重定向:<br>登录成功:303 → `/`(下发新会话 Cookie)<br>绑定成功:303 → `/security?oauth-bound=<provider>`<br>登录失败:303 → `/login?oauth-error=<denied/state/session/token/profile/identity/unlinked/internal/configuration>`<br>绑定失败:303 → `/security?oauth-error=<session/token/profile/identity/already-linked/state...>`<br>404:提供方未配置 | 处理第三方授权回调:校验 `state`(一次性)与会话一致后换取令牌、拉取用户信息,按 `subject` 登录已绑定账号或执行绑定。 | 公开(绑定流程要求原会话为同一登录用户) |
| 管理员协助登录 | POST | `/api/admin-login` | JSON 体:<br>`ticket` string 必填,长度必须为 32(管理端签发的一次性票据)<br>`csrfToken` string 必填 | 200:SessionResponse(`impersonated` 为 true,`sensitiveWritesLocked` 为 true),下发新会话 Cookie<br>错误:400 管理员登录票据格式无效;401 票据无效或已使用/与当前 IP 不匹配/目标用户不存在;403 CSRF 校验失败;429 登录尝试过多;502 暂时无法验证管理员登录票据 | 管理员通过一次性票据以协助会话进入用户端;票据绑定 IP,协助会话下所有敏感写操作被禁止。 | 公开(需会话 CSRF + 有效管理员票据) |
| 退出登录 | POST | `/api/logout` | JSON 体:`csrfToken` string 必填 | 200:`ok` bool(true),并清除会话 Cookie<br>错误:400;403 CSRF 校验失败 | 销毁当前会话并清除 Cookie。 | 需会话 + CSRF(未登录会话也可调用,处理函数未调用 `requireAPIUser`) |
| 设置主题 | POST | `/api/theme` | JSON 体:<br>`theme` string 必填,枚举 `system`/`light`/`dark`<br>`csrfToken` string 必填 | 200:`ok` bool(true)<br>错误:400 主题设置无效/请求格式无效;401 请先登录;403 CSRF 校验失败 | 保存当前会话的界面主题。 | 需登录 + CSRF(处理函数内校验,未套用 `sensitiveUserWrite`) |

## 公共结构

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
