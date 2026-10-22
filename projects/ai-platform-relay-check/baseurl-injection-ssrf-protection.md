# AI Relay Check 用户 baseUrl 注入与 SSRF 防护技术方案

# 背景与问题

2026-10-20，用户缓存测试链路出现一个可复现的安全注入样例：用户提交的 baseUrl 指向公网 redirect 服务，并将内网地址编码在 query 参数中。服务端原样拼装模型接口后，OkHttp 默认跟随 302，导致请求先访问公网 httpbin.org，再从应用网络访问内网 Consul。

```text
https://httpbin.org/redirect-to?url=http%3A%2F%2F10.117.100.165%3A7070%2Finternal%2Fv1%2Fsupplier%3Fmanufacturer_code%3Dxcode
```

解码后的跳转目标为 `http://10.117.100.165:7070/internal/v1/supplier?manufacturer_code=xcode`。该行为形成 SSRF：公网 redirect 服务获取了完整内网地址、端口、路径和参数；应用代表外部服务访问了内网 HTTP API；LLM HTTP debug 日志又将 Consul 服务目录响应体完整记录，进一步扩大内网拓扑泄露面。

| 风险路径 | 原行为 | 影响 |
|-|-|-|
| 用户 baseUrl → endpoint 拼装 | query / fragment / 完整路径被带入 | 注入 redirect 参数和内部 API 路径 |
| OkHttp 默认 redirect | 自动访问 30x Location | 外部跳转服务可将调用导向内网 |
| Consul catalog 响应 | 作为 LLM HTTP 响应体进入日志和错误信息 | 泄露服务名、节点、内网 IP、端口、tags |
| 任务失败信息 | 直接返回原始上游 body / 数据库 error_message | 非流式和异常响应可透出给用户 |

# 目标与非目标

## 目标

1. 用户 baseUrl 只保留安全的 base URL，query、fragment、userinfo 和路径注入被丢弃。
2. 禁止用户 endpoint 解析到内网、回环、链路本地、CGNAT、IPv6 ULA 等非公网地址。
3. 用户 baseUrl 的 host 必须是域名，禁止 IPv4 和 IPv6 字面量。
4. 所有 HTTP 客户端禁止跟随 3xx，业务层收到 300\~399 时直接丢弃请求，不读取响应体。
5. 模型列表预检、缓存测试、FULL 模型一致性测试均复用上述防护。
6. 调用失败和非流式异常响应不再把原始 body 返回给用户。
7. 限制异常大响应和 SSE 原始缓存的内存占用，降低 OOM 风险。

## 非目标

- 不修改数据库表结构，不执行启动期 schema migration。
- 不改变模型调用参数、token 统计口径、CACHE_COMPARE 判定算法和报告评分逻辑。
- 不改变 OneToken Kafka 常驻链路，不新增功能开关。
- 不在本方案内收敛 SkyWalking / JVM 堆配置问题；相关 OOM 仍需运维提供 heap dump 与实际 JVM 参数。

# 总体技术方案

方案采用四层纵深防御：入口规范化、网络地址策略、重定向策略、响应体最小化。前两层在任务创建前阻断；第三层保证即使配置被绕过也不访问 Location；第四层保证异常响应即使到达应用也不会外泄或无限占用内存。

| 层级 | 策略 | 作用位置 | 失败行为 |
|-|-|-|-|
| 入口规范化 | sanitizeBaseUrl | VendorDetectionController | 400，任务不创建 |
| 网络地址策略 | isPrivateNetworkEndpoint + isBlockedUserEndpoint | VendorDetectionController | 400，任务不创建 |
| 重定向策略 | followRedirects=false；3xx 早返回 | 所有 OkHttp 客户端与 LLM adaptor | 请求丢弃 / parseOk=false / 异常 |
| 响应体最小化 | 不返回原始错误体；读取上限 | CacheCompare、报告、SSE、代理 | 通用错误文案 / null / 400 |

# 入口 URL 规范化

## 实现位置

`ai-platform-relay-check-server/src/main/java/com/ai/relay/check/common/util/EndpointUtil.java`

`ai-platform-relay-check-server/src/main/java/com/ai/relay/check/trigger/http/VendorDetectionController.java`

## normalizeBaseUrl 流程

1. 调用 `EndpointUtil.sanitizeBaseUrl(baseUrl)`。
2. 输入必须能解析为绝对 http/https URI，且 host 非空、无 userinfo。
3. 路径包含 `//` 或 `..` 时拒绝。
4. 仅保留 scheme、host、显式端口和正常 base path。
5. 丢弃全部 query 和 fragment。
6. 再执行 stripKnownPath，移除已知协议路径。
7. 规范化失败返回 400：`baseUrl must be an absolute http(s) URL`。

## 地址拦截规则

`isPrivateNetworkEndpoint` 解析最终 host 的全部地址。只要任意一个地址属于非公网范围即拦截，用于防御 DNS 混合记录绕过：同一域名同时返回公网 IP 和内网 IP 时仍拒绝。

| 范围 | 处理 |
|-|-|
| 10.0.0.0/8、172.16.0.0/12、192.168.0.0/16 | 拦截 |
| 127.0.0.0/8、0.0.0.0、169.254.0.0/16 | 拦截 |
| 100.64.0.0/10 Carrier-grade NAT | 拦截 |
| IPv6 fc00::/7、fe80::/10、multicast | 拦截 |
| IPv4-mapped IPv6 | 按映射后的 IPv4 判断 |
| 无法解析、无 DNS 记录、非法 URL | 拦截 |

`isBlockedUserEndpoint` 实施用户端点 host 白名单形态：host 必须是域名，IPv4 与 IPv6 字面量全部拒绝，即使该 IP 是公网地址。规则已按最新要求移除 token / oken 关键字拦截。

```text
http://10.117.100.165:7070/internal/v1        拦截：内网地址 + IP host
http://8.8.8.8/v1                              拦截：IP host
http://[2606:4700::1]/v1                       拦截：IPv6 host
https://gateway.oken.ai/v1                     允许：关键字拦截已移除
https://api.vendor.com/v1                      允许：公网域名
```

## 模型一致性覆盖

FULL 模型一致性与 CACHE_COMPARE 使用同一个创建入口 `POST /api/vendor/tasks`，均先经过 normalizeBaseUrl，因此两类任务的 baseUrl 校验完全一致。FULL 检查点执行链路最终走 LlmGateway.chat / probeStream，同样受到 3xx 拦截和错误脱敏约束。

# 3xx 重定向拦截

## HTTP Client 层

所有自建 OkHttp 客户端均显式设置：

```java
.followRedirects(false)
.followSslRedirects(false)
```

覆盖范围包括 LLM 主客户端、模型列表预检、OpenRouter 代理、SKU 价格客户端、Token Detection 客户端。OkHttp 不再自动访问 Location。

## 业务响应码层

即使 3xx 响应已经到达应用，也在读取 body 前丢弃。拦截范围是 300 ≤ status < 400，包含 301、302、303、307 和 308。

| 链路 | 300\~399 行为 | 是否读取 body |
|-|-|-|
| LLM 非流式 chat | 抛 LlmException | 否 |
| LLM 流式 chatStream | 抛 LlmException | 否 |
| Abstract probeStream | parseOk=false，bodyText 为空 | 否 |
| Claude probeStream | 返回失败信号 | 否 |
| OpenAI Chat probeStream | 返回失败信号 | 否 |
| OpenAI Responses probeStream | 返回失败信号 | 否 |
| Gemini probeStream | 返回失败信号 | 否 |
| 模型列表预检 | 抛 BizException | 否 |
| SKU 价格查询 | 返回 null | 否 |
| Token Detection model group / sku groups | 返回失败结果 | 否 |
| OpenRouter 代理 | 对外返回 400 | 否 |

# 失败结果与响应体脱敏

## CACHE_COMPARE 单轮结果

当 `parseOk=false`、HTTP 状态非 200、非流式响应或空流时，不再把上游 body 拼入 errorMessage。错误信息只保留 HTTP 状态等安全摘要；诊断中最多输出 `responseBody=redacted`。构造原因前会复制 StreamSignals 并清空 `unexpectedResponseBody` 与 `streamErrorBody`。

## 用户可见接口

| 接口 / 服务 | 修改后行为 |
|-|-|
| VendorDetectionProgressService | 失败时返回“调用失败，请检查服务配置后重试” / Request failed. Check the service configuration and retry. |
| CacheCompareStreamService | SSE task_error 只返回通用错误文案 |
| DetectionReportQueryService | 失败报告 errorMessage 只返回通用文案，不读取任务表原始错误 |

平台诊断内部接口仍保留较详细错误，便于运维排障；如果该类接口后续开放给普通用户，需要继续收敛权限或统一脱敏。

# 内存占用上限

部署过程中出现过 Java heap OOM。3xx 拦截提交只增加状态码早返回，不增加对象分配、集合、循环或并发，代码路径上不是直接 OOM 根因。为降低异常响应造成的堆压力，本方案同步增加三个读取上限。

| 位置 | 上限 | 目的 |
|-|-|-|
| SseResponseCapture.rawBody | 512 KB / 每次流式请求 | 超限后继续解析但不缓存原始行 |
| AbstractLlmAdaptor 非流式响应 | 8 MB / 每次响应 | 避免超大 provider body 一次性入堆 |
| OpenRouterProxyController 响应 | 4 MB / 每次代理响应 | 避免代理响应全量读入堆 |

> 💡 **OOM 待办：**本次防线不能替代 JVM 与容器资源排查。需提供 Pod memory limit、实际 JVM 参数、-Xmx / MaxRAMPercentage、/mnt 下的 heap dump。当前 start.sh 仍包含 CMS 参数，而部署镜像为 Java 21，建议改用 G1 并按容器内存设置 MaxRAMPercentage。

# 兼容性与影响范围

| 场景 | 影响 | 判断 |
|-|-|-|
| 正常公网域名 baseUrl | 继续可用 | 不受影响 |
| 标准 OpenAI / Anthropic 兼容网关 | 2xx 响应继续可用 | 不受影响 |
| baseUrl 携带 query 鉴权 / 路由参数 | query 被丢弃，可能失败 | 预期取舍 |
| 依赖 30x 跳转的中转 | 请求丢弃并失败 | 安全要求，必须阻断 |
| IP 直连用户 | 400 拒绝 | 必须使用域名 |
| 内网 baseUrl | 400 拒绝 | 必须使用公网域名 |
| FULL 模型一致性 | 同样受入口与 3xx 防护 | 正常公网域名不受影响 |
| 失败详情可见性 | 仅通用错误文案 | 降低排障细节，换取数据安全 |

# 关键实现变更

| 模块 / 文件 | 变更 |
|-|-|
| EndpointUtil.java | 新增 sanitizeBaseUrl、isPrivateNetworkEndpoint、isBlockedUserEndpoint |
| VendorDetectionController.java | 用户任务创建前执行 URL 清洗、内网拦截、IP host 拦截 |
| DefaultLlmHttpClientFactory.java | LLM OkHttp 禁跟随 redirect |
| AbstractLlmAdaptor.java | chat / stream / probe 的 3xx 早返回；非流式响应 8MB 上限 |
| Claude / OpenAI / OpenAI Responses / Gemini adaptor | 自定义 probeStream 收到 3xx 返回失败信号，不读 body |
| ModelListVerificationService.java | 预检客户端禁 redirect；3xx 抛业务异常 |
| TokenDetectionClient.java | 禁 redirect；两个查询 3xx 返回失败结果 |
| SkuPriceRatioClient.java | 禁 redirect；3xx 返回 null |
| OpenRouterProxyController.java | 禁 redirect；3xx 返回 400；响应读取 4MB 上限 |
| CacheCompareExecutor.java | 失败与非流式响应不再返回 body；诊断 redacted |
| VendorDetectionProgressService / CacheCompareStreamService / DetectionReportQueryService | 用户可见错误统一通用文案 |
| SseResponseCapture.java | rawBody 捕获 512KB 上限 |

# 验证结果

本地在 macOS 开发环境执行以下 Maven 验证，均通过：

```bash
mvn -pl ai-platform-relay-check-server -Dtest=EndpointUtilTest test -q
mvn -pl ai-platform-llm-client,ai-platform-relay-check-server test -q
git diff --check
```

| 测试 / 验证点 | 结果 |
|-|-|
| 注入 query 被移除 | 通过 |
| 非法 URL、userinfo URL 被拒绝 | 通过 |
| 10.x、127.x、192.168.x、fc00::1、100.64.x 被拦截 | 通过 |
| 公网域名允许 | 通过 |
| IPv4 / IPv6 字面量 host 拒绝 | 通过 |
| token / oken URL 不再因关键字拒绝 | 通过 |
| 失败错误信息不包含原始响应体 | 通过 |
| 完整模块测试 | 通过 |

# 上线与回滚

## 上线步骤

1. 合并安全防护提交并重新构建部署包。
2. 先在 STG 创建 CACHE_COMPARE 与 FULL 任务，各覆盖一个正常公网域名 baseUrl。
3. 分别提交内网 IP、公网 IP、带 query 注入的 baseUrl，确认均返回 400。
4. 用可控 30x endpoint 验证请求不会跳转，任务失败且错误信息不包含响应体。
5. 确认生产部署后 Pod 稳定，观察内存、GC、线程池和任务成功率。
6. 保持 `relay-check.llm-http-log-enabled=false`，仅在安全网络内短期排障时开启。

## 回滚方案

代码回滚采用 Git revert 安全提交；配置层保持模型目录与任务数据不变。若上线后出现异常，可先回滚到 `prd-java21-2026-1020-1739`，再排查 JVM / SkyWalking / heap dump。安全缺陷修复不应长期以关闭 URL 校验的方式回退。

# 遗留事项

- 生产 OOM 根因尚未确认，需要运维提供 heap dump、Pod memory limit 和实际 JVM 启动参数。
- start.sh 中的 CMS 参数与 Java 21 部署镜像不匹配，建议改为 G1 + MaxRAMPercentage。
- LLM HTTP debug 日志仍可能记录完整 URL 和 body，生产必须保持关闭。
- 平台诊断类接口保留详细错误，若开放给普通用户需继续脱敏或加权限。
- 监控建议增加：入口 400 分类计数、3xx 拦截计数、单任务 SSE 捕获超限计数、非流式响应超限计数。

# 附录：内部路径分号绕过问题记录

除用户 baseUrl SSRF 外，另确认一类典型的路径规范化不一致绕过，机制为边缘访问控制与后端 Servlet 路由对同一 URI 的解析结果不同。

## 复现机制

1. 网关 / WAF / 鉴权过滤器拿到原始 URI，例如 `/api/internal;/platform-scheduler/config`。
2. 边缘层用字符串或正则比对拦截规则，未命中 `/api/internal/**`，于是放行。
3. 后端 Servlet 容器在路由前规范化路径，剥掉分号参数，URI 变成 `/api/internal/platform-scheduler/config`。
4. 规范化后的路径精准命中后端 Controller，请求成功打到内部接口。

这是 Parsing Differential：同一请求在安全组件和业务路由组件中的语义不一致。Sam Curry 曾用 `/admin;/` 绕过 Apple 的 IP 限制，机制与此相同。

## 常见畸形变体

```text
/api/internal;/platform-scheduler/config
/api/;/xxx
/api;;/xxx
/api/./xxx
//api/xxx
/API/internal/xxx
```

## 风险判断

`ai-platform-relay-check-server` 的 `/api/internal` 是内部服务前缀。该绕过说明边缘访问控制大概率基于原始路径字符串匹配，分号变形后即可绕过，导致内部配置、调度或诊断接口暴露。

## 修复建议

1. 边缘网关 / WAF / 鉴权过滤器先按后端同等规则规范化 URI，再执行路径匹配；至少应移除路径参数、`./`、`../`、重复斜杠并统一大小写后再比较。
2. 拒绝包含无法安全归一化的畸形路径，例如路径段中的分号、双斜杠、`..` 与非预期编码。
3. 内部接口不要依赖单一网关路径前缀作为唯一授权边界，后端增加身份认证、来源网络白名单或内部服务 token 校验。
4. 在网关和应用层分别记录原始 URI 与规范化后 URI，便于检测二者不一致的请求。
5. 补充回归用例：分号、空路径段、连续分号、点段、双斜杠、大小写和 URL 编码变体均应被拒绝或按同一规范化结果授权。

> 💡 **安全原则：**访问控制必须发生在与业务路由相同的规范化语义之后；否则任何语法变形都会成为绕过面。
