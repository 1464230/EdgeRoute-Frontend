# 系统与运维(用户端)
> 路由总数 7,来源 EdgeUserPortal/internal/web/server.go

说明:
- 错误统一返回 `{"error":"消息"}`(`writeAPIError`),但 `/health`、`/healthz`、`/readyz`、`/metrics` 为运维探针,返回纯文本。
- 本模块 7 条路由均不要求登录,也不经过 `sensitiveUserWrite`。
- 路由清单(JSON)中 `/readyz`、`/metrics` 的 `handler` 为空,实际分别由 `observability.ReadinessHandler`、`s.metrics.Handler()` 处理(见 `server.go` 路由注册)。

| 接口名称 | 方法 | 路径 | 主要请求参数 | 返回数据结构 | 功能说明 | 认证 |
|---|---|---|---|---|---|---|
| 存活探针 | GET | `/health` | 无 | 成功 `200`,`text/plain`,正文 `ok`。<br>失败 `503`,正文 `unavailable`,响应头 `Retry-After: 5`。<br>响应头固定带 `Cache-Control: no-store`、`X-EdgeRoute-Version`、`X-EdgeRoute-API-Version`。 | 调用 `s.rpc.Health` 检查上游 API RPC 是否可用,超时 3 秒(`healthCheckTimeout`)。与 `/healthz` 共用 `health` 处理函数。 | 公开 |
| 存活探针(别名) | GET | `/healthz` | 无 | 同 `/health`。 | 与 `/health` 完全相同,供 Kubernetes 等探针习惯路径使用。 | 公开 |
| 就绪探针 | GET | `/readyz` | 无 | 全部检查通过 `200`,`text/plain`,正文 `ok\n`。<br>有失败项 `503`,正文 `unavailable: <失败项名,逗号分隔>\n`,带 `Retry-After: 5`。<br>响应头 `Cache-Control: no-store`、`X-Content-Type-Options: nosniff`。 | 并发执行就绪检查,目前只有一项 `api_rpc`(即 `s.rpc.Health`),超时 3 秒。 | 公开 |
| Prometheus 指标 | GET | `/metrics` | 请求头 `Authorization: Bearer <token>`(仅当服务端配置了 metrics token 时需要) | Prometheus/OpenMetrics 文本格式。<br>配置了 token 但缺失或错误:`401`,响应头 `WWW-Authenticate: Bearer realm="metrics"`。<br>未配置 token:非回环地址直连,或带有 `X-Forwarded-For`/`X-Forwarded-Host`/`X-Real-Ip`/`Forwarded` 头时返回 `404`。 | 暴露进程指标。未配置 token 时仅允许本机回环地址直接访问,避免经反向代理泄露。 | 公开路由,但受 Bearer token 或"仅回环直连"保护 |
| 用户端 OpenAPI 文档 | GET | `/api/openapi.json` | 无 | 成功 `200`,`application/json`,响应头 `Cache-Control: public, max-age=300`。<br>正文为 OpenAPI `3.0.3` 文档:`info.title="EdgeRoute User API"`;`servers[0].url` 取自 `RefreshPublicBaseURL`,为空时为 `/`;安全方案 `cookieAuth`(Cookie 名 `edgeroute_user_session`);`paths` 为用户侧接口的静态子集(约 14 个路径示例,不是全部接口)。<br>失败 `500`,纯文本 `生成 API 文档失败`。 | 返回用户侧 API 的精简 OpenAPI 描述,供开发者查阅。 | 公开 |
| 上报 Web Vitals | POST | `/api/metrics/web-vitals` | JSON 请求体:<br>`metrics` array 必填,1-50 条,每项 `name` string、`value` number、`id` string、`delta` number、`rating` string、`navigationType` string、`url` string、`userAgent` string、`timestamp` number<br>`url` string 必填<br>`userAgent` string<br>`timestamp` number<br>请求体上限 64KiB,禁止未知字段。 | 成功 `204`,无正文。<br>错误:`400` `请求格式无效` / 参数校验失败;`429` `{"error":"上报过于频繁，请稍后再试"}` + `Retry-After: 60`。 | 前端上报核心 Web 指标。仅 `LCP`、`FCP`、`CLS`、`INP`、`TTFB`、`FID` 会入库,其余忽略;RPC 写入失败只记日志,仍返回 `204`。按客户端 IP 限流,每分钟 60 次,限流在解析请求体之前执行。 | 公开(匿名可访问;已登录时记录用户 ID,未登录记为 0),按 IP 限流 |
| 上报页面性能时间线 | POST | `/api/metrics/performance-timeline` | JSON 请求体:<br>`navigation` object 必填,其中 `url` string 必填,其余 `userAgent`、`timestamp`、`dnsLookup`、`tcpConnect`、`sslHandshake`、`requestTime`、`responseTime`、`ttfb`、`domProcessing`、`domInteractive`、`domComplete`、`loadComplete`、`totalTime`、`navigationType`<br>`resources` array,最多 500 条,每项 `url`、`pageUrl`、`userAgent`、`timestamp`、`name`、`initiatorType`、`duration`、`transferSize`、`encodedBodySize`、`decodedBodySize`、`cacheStatus`<br>请求体上限 64KiB,禁止未知字段。 | 成功 `204`,无正文。<br>错误:`400` `请求格式无效` / 参数校验失败;`429` 同上。 | 前端上报页面导航耗时与资源加载明细。资源只有耗时超过 100ms(`SlowResourceThresholdMs`)才会入库;RPC 写入失败只记日志,仍返回 `204`。按客户端 IP 限流,每分钟 60 次。 | 公开(匿名可访问;已登录时记录用户 ID),按 IP 限流 |

## 公共结构与补充说明

- 限流(`rateLimitMetrics`):按客户端 IP 固定窗口,每分钟 60 次,最多跟踪 100000 个 IP(`MaxKeys`);超限返回 `429`、`Retry-After: 60`、`{"error":"上报过于频繁，请稍后再试"}`。
- 探针超时:`healthCheckTimeout = 3s`。
- 上报限额常量(`EdgeCommon/pkg/frontendmetrics`):`MaxMetricsPerRequest=50`、`MaxResourcesPerRequest=500`、`SlowResourceThresholdMs=100`。
- 待确认:`metrics` 与 `navigation` 内各数值字段的具体单位(毫秒/字节),本模块未逐一核对,请以前端上报代码为准。

### 未计入 7 条清单的相关路由(仅供参考)

以下路由在 `server.go` 中注册,但不在本模块路由清单 JSON 中,不计入"路由总数":
- `/assets/`、`/landing/`:静态资源,`Cache-Control: public, max-age=31536000, immutable`,`Vary: Accept-Encoding`,支持 gzip,公开。
- `GET /`:SPA 兜底,返回前端 `index.html`(`200 text/html`,`Cache-Control: no-store`);命中敏感 SPA 路径返回 `404`;`/` 在门户关闭时 `302` 到 `/login`。公开。
