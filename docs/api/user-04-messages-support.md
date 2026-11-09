# 消息与工单(用户端)

> 路由总数 12,来源 EdgeUserPortal/internal/web/server.go

通用约定:错误统一为 `{"error":"消息"}`;JSON 请求体不允许未知字段,上限 64KB;写操作(含 DELETE)均需在 JSON 请求体中携带 `csrfToken`(取自 `GET /api/session`)。`sensitiveUserWrite` 拒绝管理员协助会话、授权失效只读模式,并记录通用操作审计。

分页约定(`parseBoundedQueryInt`):`offset` 默认 0、有效范围 0-1000000;`size` 默认 20、有效范围 0-100。值非法或超出上限时**回落为默认值**(不是截断到上限),例如 `size=500` 实际按 20 处理。

| 接口名称 | 方法 | 路径 | 主要请求参数 | 返回数据结构 | 功能说明 | 认证 |
|---|---|---|---|---|---|---|
| 工单分类列表 | GET | `/api/support/categories` | 无 | 200:`categories` array,元素:`id` int64、`name` string<br>错误:401;502 读取工单分类失败 | 提交工单时可选的分类。 | 需登录 |
| 工单列表 | GET | `/api/support/tickets` | 查询参数:<br>`status` string 可选,枚举 `none`/`solved`/`closed`,空为全部<br>`offset` int 默认 0<br>`size` int 默认 20(见分页约定) | 200:<br>`tickets` array,元素见「公共结构 SupportTicket」(不含 `body`)<br>`count` int64 符合条件的总数<br>错误:400 工单状态无效;401;502 读取工单失败 | 分页查询当前用户的工单。 | 需登录 |
| 创建工单 | POST | `/api/support/tickets` | JSON 体:<br>`categoryId` int64 必填,>0<br>`subject` string 必填,去空格后 3-200 个字符<br>`body` string 必填,去空格后 1-10000 个字符<br>`csrfToken` string 必填 | 201:`id` int64 新工单 ID<br>错误:400 请填写有效的工单分类、主题和内容;403 CSRF 校验失败;502 创建工单失败 | 提交新工单。 | 需登录 + CSRF 写保护 |
| 工单详情 | GET | `/api/support/tickets/{ticketID}` | 路径参数:`ticketID` int64,>0 | 200:<br>`ticket` 见 SupportTicket(含 `body`)<br>`replies` array,元素见「公共结构 SupportReply」<br>错误:400 工单 ID 无效;401;404 工单不存在;502 读取工单详情失败 | 查看工单内容及全部回复记录。 | 需登录 |
| 回复工单 | POST | `/api/support/tickets/{ticketID}/replies` | 路径参数:`ticketID` int64,>0<br>JSON 体:<br>`comment` string 必填,去空格后 1-10000 个字符<br>`csrfToken` string 必填 | 200:`ok` bool(true)<br>错误:400 回复内容无效/工单 ID 无效;403 CSRF 校验失败;502 回复工单失败 | 用户追加回复。 | 需登录 + CSRF 写保护 |
| 删除工单 | DELETE | `/api/support/tickets/{ticketID}` | 路径参数:`ticketID` int64,>0<br>JSON 体:`csrfToken` string 必填 | 200:`ok` bool(true)<br>错误:400 工单 ID 无效;403 CSRF 校验失败;404 工单不存在或无权操作;409 工单已有回复,无法删除;502 删除工单失败 | 删除工单,已有回复的工单不允许删除。 | 需登录 + CSRF 写保护 |
| 未读消息列表 | GET | `/api/messages` | 查询参数:`offset` int 默认 0;`size` int 默认 20(见分页约定) | 200:<br>`messages` array,元素见「公共结构 Message」<br>`count` int64 未读总数<br>错误:401;502 读取消息失败 | 分页获取未读站内消息。 | 需登录 |
| 历史消息列表 | GET | `/api/messages/history` | 查询参数:`offset` int 默认 0;`size` int 默认 20(见分页约定) | 200:`messages` array(元素同 Message,为已读消息)、`count` int64 已读总数<br>错误:401;502 读取历史消息失败 | 分页获取已读历史消息。 | 需登录 |
| 操作日志 | GET | `/api/audit-logs` | 查询参数:<br>`offset` int 默认 0;`size` int 默认 20(见分页约定)<br>`level` string 可选,枚举 `info`/`warn`/`error`<br>`from` / `to` string 起止日期(原样透传后端 RPC,格式待确认)<br>`q` string 关键字(原样透传) | 200:<br>`logs` array,元素:`id` int64、`level` string(≤16 字符)、`action` string(≤128)、`description` string(≤400)、`createdAt` int64(时间戳,单位待确认)<br>`count` int64 总数<br>错误:400 日志级别无效;401;502 读取操作记录失败 | 查询当前用户自己的操作历史,不接受用户/管理员选择参数,范围由后端限定;文本字段按字符数截断。 | 需登录 |
| 未读消息数 | GET | `/api/messages/unread-count` | 无 | 200:`count` int64<br>错误:401;502 读取未读消息数量失败 | 返回未读消息数量,用于角标。 | 需登录 |
| 标记消息已读 | POST | `/api/messages/read` | JSON 体:<br>`messageIds` int64[] 必填,去除非正数和重复后数量 1-100<br>`csrfToken` string 必填 | 200:`ok` bool(true)<br>错误:400 消息列表无效;403 CSRF 校验失败;502 更新消息状态失败 | 将指定消息标记为已读。 | 需登录 + CSRF 写保护 |
| 全部标记已读 | POST | `/api/messages/read-all` | JSON 体:`csrfToken` string 必填 | 200:`ok` bool(true)<br>错误:403 CSRF 校验失败;502 更新消息状态失败 | 将当前用户全部未读消息标记为已读。 | 需登录 + CSRF 写保护 |

## 公共结构

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
