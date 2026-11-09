# 计费与订单(用户端)

> 路由总数 17,来源 EdgeUserPortal/internal/web/server.go

通用约定(适用于本文件全部接口,除特别说明):
- 全部路由外层都有 `licenseFeature(Finance)`:授权服务读取失败返回 503 `EdgeRoute 授权状态暂时无法读取，请稍后重试`;授权未包含财务功能返回 403 `当前 EdgeRoute 授权未包含该功能`。
- 带 `sensitiveUserWrite` 的写接口:未登录 401 `请先登录`;管理员协助会话 403 `管理员协助会话不能修改用户资源或安全配置`;授权已确认失效 403 `平台授权已失效，当前为只读模式…`。
- 写接口请求体均为 JSON(上限 64KB,`DisallowUnknownFields`,出现未知字段即 400 `请求格式无效`),且必须携带 `csrfToken`(string,必填,须与会话 CSRF 一致,否则 403 `CSRF 校验失败`)。
- 错误统一为 `{"error":"消息"}`。

| 接口名称 | 方法 | 路径 | 主要请求参数 | 返回数据结构 | 功能说明 | 认证 |
|---|---|---|---|---|---|---|
| 计费总览 | GET | `/api/billing` | 无 | 200:<br>`balanceReminderEligible` bool 是否需要余额提醒<br>`balanceReminderLimit` 余额提醒阈值(来自 userconfigs.BalanceReminderLimit,类型待确认)<br>`account` {`id`,`total`,`totalFrozen`}<br>`financeEnabled` / `plansEnabled` / `plansPurchasable` / `requirePlan` / `showPrices` bool<br>`plans` []Plan<br>`purchases` []UserPlan<br>`logs` []AccountLog<br>`methods` []OrderMethod<br>`orders` []UserOrder(最多 100)<br>`refunds` []UserRefund(最多 100)<br>`bills` []UserBill(最多 100)<br>`trafficPackagesEnabled` / `trafficPackagesPurchasable` bool<br>`trafficPackages` []TrafficPackage<br>`trafficPackagePeriods` []{`id`,`count`,`unit`,`months`}<br>`trafficPackagePrices` []{`trafficPackageId`,`nodeRegionId`,`trafficPackagePeriodId`,`price`}(仅 price>0)<br>`nodeRegions` []{`id`,`name`}<br>`userTrafficPackages` []UserTrafficPackage<br>错误:502(`读取计费设置失败`、`读取已购套餐失败`、`读取支付订单失败` 等分项消息) | 并发聚合读取用户账户、套餐、订单、退款、账单、流量包等;各 RPC 最多重试 3 次,整体超时 15s。`showPrices=false` 时套餐不返回价格,且不返回流量包价格;套餐/支付方式/流量包列表按计费开关决定是否加载。 | 需登录;需财务授权(GET 无 CSRF) |
| 直接购买套餐(余额) | POST | `/api/plans/{planID}/purchase` | 路径:`planID` int64 >0<br>JSON:`csrfToken`<br>`period` string 必填,`monthly`/`seasonally`/`yearly`(忽略大小写与首尾空格)<br>`countPeriod` int32,≤0 时按 1,最大 120 | 201:`{"id": 已购套餐ID(int64)}`<br>错误:400(套餐 ID 无效/周期无效/购买周期不能超过 120)、403、409/404/502(见公共结构“计费 RPC 错误映射”,如 `账户余额不足`、`套餐当前不可购买`) | 使用账户余额直接购买套餐,无需支付网关。 | 需登录+CSRF写保护;需财务授权 |
| 创建套餐支付订单 | POST | `/api/plans/{planID}/order` | 路径:`planID` int64 >0<br>JSON:`csrfToken`<br>`period` string 必填,`monthly`/`seasonally`/`yearly`<br>`countPeriod` int32 必填,1~120(此处不会默认为 1)<br>`methodCode` string,可空;非空时转小写,≤32 字符<br>`couponCode` string,可空,会 trim<br>请求头:`Idempotency-Key` 可选,16~128 字符,仅 `a-zA-Z0-9-_.:` | 201:CreatedBillingOrder<br>错误:400(`请选择有效的套餐和周期`、`订单请求标识无效`)、409(`支付方式不可用`、`在线支付暂不可用，请联系管理员`、优惠码类错误、`同一订单请求标识不能用于不同购买内容` 等)、其余见映射 | 创建套餐购买订单(可带优惠码),支持幂等。`methodCode` 为空时不生成回调地址,是否走余额由后端 RPC 决定(待确认:空 methodCode 的具体语义)。 | 需登录+CSRF写保护;需财务授权 |
| 校验套餐优惠码 | POST | `/api/plans/{planID}/coupon` | 路径:`planID` int64 >0<br>JSON:`csrfToken`<br>`code` string,优惠码(trim)<br>`period` string 必填 `monthly`/`seasonally`/`yearly`<br>`countPeriod` int32,≤0 按 1,最大 120 | 200:`{"discountCents": int64 优惠金额(分), "amountCents": int64 实付金额(分)}`<br>错误:400(`套餐 ID 无效`、`当前套餐不支持优惠码`——套餐不存在或该周期价格≤0)、409(`请输入优惠码`、`优惠码无效或已停用`、`优惠码已过期`、`优惠码校验次数过多，请 15 分钟后重试` 等)、其余见映射 | 按套餐周期价×周期数计算基础金额并校验优惠码,返回优惠结果。优惠码校验有频率限制(15 分钟)。 | 需登录+CSRF写保护;需财务授权 |
| 创建流量包订单 | POST | `/api/traffic-packages/{packageID}/order` | 路径:`packageID` int64 >0<br>JSON:`csrfToken`<br>`methodCode` string 必填,≤32,转小写<br>`regionId` int64 必填 >0(节点区域)<br>`periodId` int64 必填 >0(流量包周期)<br>`count` int32 必填,1~20 | 201:CreatedBillingOrder<br>错误:400(`请选择有效的流量包、区域、有效期和支付方式`、`流量包订单参数无效`)、409(`流量包当前不可购买`、`流量包自助购买当前未开放`、`支付方式不可用`、`在线支付暂不可用，请联系管理员`)、其余见映射 | 为指定区域、周期、数量的流量包创建支付订单。 | 需登录+CSRF写保护;需财务授权 |
| 续费已购套餐(余额) | POST | `/api/user-plans/{userPlanID}/renew` | 路径:`userPlanID` int64 >0<br>JSON:`csrfToken`<br>`period` string 必填 `monthly`/`seasonally`/`yearly`<br>`countPeriod` int32,≤0 按 1,最大 120 | 200:`{"ok": true}`<br>错误:400(`已购套餐 ID 无效`、周期无效)、403、409(`账户余额不足` 等)、其余见映射 | 使用账户余额续费已购套餐。 | 需登录+CSRF写保护;需财务授权 |
| 设置套餐自动续费 | PUT | `/api/user-plans/{userPlanID}/auto-renew` | 路径:`userPlanID` int64 >0<br>JSON:`csrfToken`<br>`enabled` bool<br>`maxAmount` float64,单次自动续费金额上限,经 `PlanAutoRenew.Validate()` 校验(具体范围待确认:校验在 EdgeCommon userconfigs 中) | 200:`{"ok": true}`<br>错误:400(`套餐无效`、校验错误文本、`无法保存自动续费设置，请检查套餐归属、状态和金额上限`)、403、503(`当前 API 不支持套餐自动续费`) | 开启/关闭已购套餐的自动续费并设置金额上限。注意:该处理函数先校验登录再解码 JSON。 | 需登录+CSRF写保护;需财务授权 |
| 创建套餐续费订单 | POST | `/api/user-plans/{userPlanID}/order` | 路径:`userPlanID` int64 >0<br>JSON/请求头同“创建套餐支付订单”:`csrfToken`、`period`、`countPeriod`(1~120)、`methodCode`、`couponCode`、`Idempotency-Key` | 201:CreatedBillingOrder<br>错误同“创建套餐支付订单” | 为已购套餐创建续费支付订单(订单类型 `plan_renewal`),支持优惠码与幂等。 | 需登录+CSRF写保护;需财务授权 |
| 校验续费优惠码 | POST | `/api/user-plans/{userPlanID}/coupon` | 路径:`userPlanID` int64 >0<br>JSON:`csrfToken`、`code`、`period`、`countPeriod`(同“校验套餐优惠码”) | 200:`{"discountCents","amountCents"}`<br>错误:400(`套餐 ID 无效`、`当前套餐不支持优惠码`)、404(`已购套餐不存在`)、409(优惠码类错误)、其余见映射 | 先由已购套餐 ID 反查所属套餐,再按其周期价校验优惠码。 | 需登录+CSRF写保护;需财务授权 |
| 创建充值订单 | POST | `/api/orders` | JSON:`csrfToken`<br>`methodCode` string 必填,≤32,转小写<br>`amount` float64 必填,0.01~1,000,000,最多 2 位小数 | 201:CreatedBillingOrder<br>错误:400(`请选择支付方式并输入有效充值金额`、`充值金额无效`)、409(`支付方式不可用`、`读取支付方式失败`、`在线支付暂不可用，请联系管理员`、`支付方式当前不可用`)、其余见映射 | 创建账户余额充值订单,返回支付跳转信息;对 alipay/wechat/custom/epay/bepusdt 会自动拼接回调地址 `/api/pay/{provider}/{methodCode}/notify`。 | 需登录+CSRF写保护;需财务授权 |
| 查询订单详情 | GET | `/api/orders/{orderCode}` | 路径:`orderCode` string,trim 后非空且 ≤64 | 200:`{"order": UserOrder}`<br>错误:400(`订单号无效`)、404(`订单不存在`)、502(`读取订单失败`) | 按订单号查询当前用户订单。 | 需登录;需财务授权 |
| 取消订单 | POST | `/api/orders/{orderCode}/cancel` | 路径:`orderCode` string ≤64<br>JSON:`csrfToken` | 200:`{"ok": true}`<br>错误:400(`订单号无效`)、404(`订单不存在`)、409(`已完成订单不能取消`)、其余见映射 | 取消未完成的订单。 | 需登录+CSRF写保护;需财务授权 |
| 申请订单退款 | POST | `/api/orders/{orderCode}/refunds` | 路径:`orderCode` string ≤64<br>JSON:`csrfToken`<br>`amount` float64 必填,0.01~1,000,000,最多 2 位小数<br>`reason` string 必填,trim 后 3~500 字符(按 rune) | 201:`{"code": 退款单号}`<br>错误:400(`请输入有效的退款金额和退款原因`)、其余见映射(可退金额等业务限制由 RPC 判定,不在本层暴露原文,多为 502 `计费操作失败`) | 对已支付订单发起退款申请。 | 需登录+CSRF写保护;需财务授权 |
| 取消退款申请 | POST | `/api/refunds/{refundCode}/cancel` | 路径:`refundCode` string ≤64<br>JSON:`csrfToken` | 200:`{"ok": true}`<br>错误:400(`退款单号无效`)、其余见映射 | 撤销尚未处理的退款申请。 | 需登录+CSRF写保护;需财务授权 |
| 支付用量账单 | POST | `/api/bills/{billID}/pay` | 路径:`billID` int64 >0<br>JSON:`csrfToken` | 200:`{"ok": true}`<br>错误:400(`账单 ID 无效`)、409(`账单当前无法支付`、`账户余额不足`)、其余见映射 | 使用账户余额支付一笔用量账单。 | 需登录+CSRF写保护;需财务授权 |
| 支付回调通知(POST) | POST | `/api/pay/{provider}/{methodCode}/notify` | 路径:`provider` 取值 `alipay`/`custom`/`epay`/`wechat`/`bepusdt`;`methodCode` ≤32,转小写<br>查询:`ack`(仅 bepusdt)`success`/`ok`,缺省 `success`<br>请求体:支付网关原始表单/报文,非空,≤64KB;仅转发 `Wechatpay-*`、`Content-Type`、`Request-Id` 请求头用于验签 | 成功 200,纯文本 `success` 或 `ok`(bepusdt 按 `ack`)<br>失败:纯文本 `failure`,状态码 400(参数/验签失败/报文为空)、405(bepusdt 非 POST)、429(按 IP+methodCode 限流)<br>另:授权层错误为 JSON 503/403 | 接收第三方支付异步通知并交由 API 验签入账。响应头 `Cache-Control: no-store`;不使用统一 JSON 错误格式。 | 公共回调(不要求登录/CSRF,依赖网关签名验证);仍需财务授权 |
| 支付回调通知(GET) | GET | `/api/pay/{provider}/{methodCode}/notify` | 路径同上<br>查询:整串 `RawQuery` 作为通知报文(非空,≤64KB,超限 413);`provider=bepusdt` 时不允许 GET | 成功 200,纯文本 `success`/`ok`<br>失败:纯文本 `failure`,400、405(bepusdt)、413、429 | 部分网关(如易支付、支付宝同步/异步)以 GET 回调,处理逻辑与 POST 相同(同一 handler `apiPaymentNotify`)。 | 公共回调(不要求登录/CSRF);仍需财务授权 |

## 公共结构

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
