# Token 缓存检查内网拦截方案

> 版本：2026-10-22  
 > 范围：`CACHE_COMPARE` 供应商缓存对比任务的用户输入 `baseUrl` 安全校验，以及真实模型请求阶段的跳转限制。  
 > 结论：当前实现通过「URL 归一化 + 全量 DNS 解析校验 + 禁止用户直填 IP + 禁止 HTTP 重定向」降低 SSRF 风险；主要剩余风险是任务创建与真实请求之间的 DNS rebinding 时间窗。

## 1. 背景与目标

Token 缓存检查允许用户传入第三方供应商的 API `baseUrl`，后端会代替用户调用该地址完成模型列表预检和 5 轮缓存对比请求。因为请求发生在服务端，如果直接信任用户 URL，攻击者可能构造：

- `http://10.117.100.165:7070/internal/v1` 这类内网地址；
- `http://127.0.0.1:8080` 这类本机地址；
- 一个公网域名，再通过 302 跳转到内网地址；
- 一个同时解析出公网 IP 和内网 IP 的域名，利用多次解析或多 A 记录绕过校验。

内网拦截的目标是：**用户提供的缓存检查目标必须是公网域名，且 DNS 解析出的所有地址都必须是公网地址；请求阶段不允许隐藏跨主机跳转**。

## 2. 总体流程

```mermaid
flowchart TD
    A[POST /api/vendor/detection/tasks] --> B[CreateVendorDetectionRequest.baseUrl]
    B --> C[EndpointUtil.sanitizeBaseUrl]
    C -->|非法 URL / userinfo / path 注入| R1[400: absolute http(s) URL]
    C -->|归一化成功| D[EndpointUtil.isPrivateNetworkEndpoint]
    D --> E[URI.getHost 提取 host]
    E --> F["InetAddress.getAllByName(host)"]
    F -->|解析失败或无结果| R2[400: public network address]
    F --> G[逐个检查全部 A/AAAA 解析结果]
    G -->|任一内网/非公网地址| R2
    G -->|全部为公网地址| H[EndpointUtil.isBlockedUserEndpoint]
    H -->|字面量 IPv4/IPv6| R3[400: domain name]
    H -->|域名| I[模型列表预检]
    I --> J[创建缓存对比任务]
    J --> K[CacheCompareExecutor 执行 5 轮请求]
    K --> L["OkHttp: followRedirects(false)"]
    L --> M[目标响应 3xx 时不跟随跳转]
```

## 3. 入口与调用链

### 3.1 HTTP 入口

控制器：

- `ai-platform-relay-check-server/src/main/java/com/ai/relay/check/trigger/http/VendorDetectionController.java`

入口接口：

```http
POST /api/vendor/detection/tasks
```

核心代码：

```java
@PostMapping("/tasks")
public ApiResponse<VendorDetectionCreatedResponse> createTask(
        @RequestBody @Valid CreateVendorDetectionRequest request,
        @RequestHeader(value = "UserId", required = false) String deviceId,
        @RequestHeader(value = "User-Agent", required = false) String userAgent) {
    String extractedDeviceId = DeviceIdExtractor.extract(deviceId, userAgent);
    return ApiResponse.ok(createVendorTask(normalizeBaseUrl(request), extractedDeviceId));
}
```

### 3.2 安全校验入口

`normalizeBaseUrl()` 在进入业务创建逻辑前完成三层校验：

```java
private CreateVendorDetectionRequest normalizeBaseUrl(CreateVendorDetectionRequest request) {
    String sanitized = EndpointUtil.sanitizeBaseUrl(request.getBaseUrl());
    if (sanitized == null) {
        throw new BizException(400, "baseUrl must be an absolute http(s) URL");
    }
    if (EndpointUtil.isPrivateNetworkEndpoint(sanitized)) {
        throw new BizException(400, "baseUrl must be a public network address");
    }
    if (EndpointUtil.isBlockedUserEndpoint(sanitized)) {
        throw new BizException(400, "baseUrl must be a domain name");
    }
    request.setBaseUrl(sanitized);
    return request;
}
```

执行顺序固定为：

1. URL 结构与路径安全校验；
2. DNS 解析结果与网段校验；
3. 用户 URL 必须使用域名的策略校验；
4. 将归一化后的 base URL 写回请求对象；
5. 再执行模型列表预检和任务创建。

## 4. 方案分层

### 4.1 第一层：URL 归一化

实现：

- `EndpointUtil.sanitizeBaseUrl(String baseUrl)`

处理规则：

| 校验项 | 行为 |
| --- | --- |
| 空 URL | 返回 `null`，拒绝 |
| 非绝对 URL | 返回 `null`，拒绝 |
| 非 `http` / `https` scheme | 返回 `null`，拒绝 |
| URL 缺少 host | 返回 `null`，拒绝 |
| 带 userinfo，如 `https://user:pass@example.com` | 返回 `null`，拒绝 |
| path 包含 `//` | 返回 `null`，拒绝 |
| path 包含 `..` | 返回 `null`，拒绝 |
| query string | 丢弃 |
| fragment | 丢弃 |
| 常见 API 路径后缀 | 剥离 |
| 归一化结果为空 | 返回 `null`，拒绝 |

可剥离路径包括：

```text
/v1/chat/completions
/v1/responses
/responses
/v1/messages
/v1
```

同时会识别并剥离 Gemini / Vertex AI 模型路径前缀：

```text
/v1beta/models/
/v1beta2/models/
/v1/publishers/google/models/
```

典型例子：

```text
输入：
https://httpbin.org/redirect-to?url=http%3A%2F%2F10.117.100.165%3A7070%2Finternal%2Fv1

输出：
https://httpbin.org/redirect-to
```

作用：防止把用户构造的 redirect query、内部路径、认证信息等继续带入后续 endpoint 拼接和请求。

### 4.2 第二层：DNS 全量解析校验

实现：

- `EndpointUtil.isPrivateNetworkEndpoint(String endpoint)`

核心流程：

```java
URI uri = URI.create(value);
String host = uri.getHost();

InetAddress[] addresses = InetAddress.getAllByName(host);
if (addresses == null || addresses.length == 0) {
    return true;
}

for (InetAddress address : addresses) {
    if (isPrivateAddress(address)) {
        return true;
    }
}
return false;
```

解析方式：

- 不自研 DNS 协议；
- 不指定自定义 DNS server；
- 使用 JDK 默认解析入口 `InetAddress.getAllByName(host)`；
- host 是字面量 IP 时直接得到 `InetAddress`；
- host 是域名时走 JVM 与操作系统 resolver，通常会读取 `/etc/resolv.conf` 指定的 DNS 服务。

关键点：使用 `getAllByName()` 而不是 `getByName()`，并检查 **全部返回地址**。因此下面这种混合解析会被拒绝：

```text
api.example.com
  → 203.0.113.10   公网
  → 10.117.100.165 内网
```

策略是只要任意一个解析结果属于内网或非公网地址，即拒绝。

解析失败采用失败关闭策略：

| 异常或异常状态 | 返回值 |
| --- | --- |
| 空 URL | `true`，拒绝 |
| 非法 URL | `true`，拒绝 |
| 非 `http(s)` | `true`，拒绝 |
| 无 host | `true`，拒绝 |
| DNS 解析失败 `UnknownHostException` | `true`，拒绝 |
| DNS 返回空结果 | `true`，拒绝 |

### 4.3 第三层：禁止用户直填 IP

实现：

- `EndpointUtil.isBlockedUserEndpoint(String endpoint)`

规则：

```java
return host != null && (host.matches("^[0-9.]+$") || host.indexOf(':') >= 0);
```

用户 `baseUrl` 必须使用域名：

| 输入 | 结果 |
| --- | --- |
| `http://8.8.8.8/v1` | 拒绝 |
| `https://10.117.100.165:7070/v1` | 拒绝 |
| `http://[2606:4700::1]/v1` | 拒绝 |
| `http://token.example.com/v1` | 允许进入后续流程 |
| `https://api.example.com/v1` | 允许进入后续流程 |

注意：这里拒绝的是「用户输入」必须是域名的策略；对域名解析出的公网 IP 没有额外惩罚。

### 4.4 第四层：请求阶段禁止重定向

模型列表预检 HTTP client：

- `ModelListVerificationService`

```java
new OkHttpClient.Builder()
        .followRedirects(false)
        .followSslRedirects(false)
        .build();
```

真实模型请求 HTTP client：

- `ai-platform-llm-client/src/main/java/com/ai/relay/llm/infrastructure/http/DefaultLlmHttpClientFactory.java`

```java
// 用户/渠道 endpoint 不应触发隐藏的跨主机跳转。禁止跟随 302/307 可阻断
// “外部 redirect 服务 -> 内网地址”的 SSRF 链路。
new OkHttpClient.Builder()
        .followRedirects(false)
        .followSslRedirects(false)
        .build();
```

作用：

- 阻断「公网域名 → 302 → 内网地址」的 SSRF 链路；
- 避免 HTTP client 自动向另一个 host 发送用户 API Key；
- 3xx 会保留为响应结果，由上层按请求失败或非目标响应处理。

## 5. 网段拦截规则

网段判断集中在：

- `EndpointUtil.isPrivateAddress(InetAddress address)`

| 地址类型 | 判断方式 | 示例 |
| --- | --- | --- |
| Any-local | `isAnyLocalAddress()` | `0.0.0.0` |
 | Loopback | `isLoopbackAddress()` | `127.0.0.1`、`::1` |
| Link-local | `isLinkLocalAddress()` | `169.254.x.x`、IPv6 link-local |
| Site-local / RFC1918 | `isSiteLocalAddress()` | `10/8`、`172.16/12`、`192.168/16` |
| Multicast | `isMulticastAddress()` | IPv4/IPv6 multicast |
| IPv6 unique-local | 首字节匹配 `fc00::/7` | `fc00::1`、`fd00::1` |
| IPv4-mapped IPv6 | 解析映射 IPv4 后递归校验 | `::ffff:10.1.2.3` |
| Carrier-grade NAT | `100.64.0.0/10` 手写判断 | `100.64.1.1` |

当前测试覆盖：

```java
assertTrue(EndpointUtil.isPrivateNetworkEndpoint("http://10.117.100.165:7070/internal/v1"));
assertTrue(EndpointUtil.isPrivateNetworkEndpoint("http://127.0.0.1:8080"));
assertTrue(EndpointUtil.isPrivateNetworkEndpoint("http://192.168.1.1/v1"));
assertTrue(EndpointUtil.isPrivateNetworkEndpoint("http://[fc00::1]/v1"));
assertTrue(EndpointUtil.isPrivateNetworkEndpoint("http://100.64.1.1/v1"));
assertTrue(EndpointUtil.isPrivateNetworkEndpoint("not-a-url"));

assertFalse(EndpointUtil.isPrivateNetworkEndpoint("https://api.openai.com/v1"));
assertFalse(EndpointUtil.isPrivateNetworkEndpoint("http://8.8.8.8/v1"));
```

## 6. 与缓存检查任务的关系

创建任务时，校验通过的 `baseUrl` 会根据模型和协议生成真实任务 endpoint：

- `VendorDetectionService.userTaskEndpoint(...)`

```java
return EndpointUtil.protocolEndpoint(request.getBaseUrl(), protocol, endpointModel, true);
```

然后任务记录保存 endpoint，`CacheCompareExecutor` 后续执行目标线路缓存对比：

```text
用户 baseUrl 校验通过
    ↓
按 provider / model / path 推断协议
    ↓
生成完整任务 endpoint
    ↓
保存 relay_detection_task
    ↓
执行模型请求
    ↓
解析 usage 中 cache_write / cache_read / input / output token
    ↓
计算缓存结论
```

需要注意：`CacheCompareExecutor` 自身不再重复执行 `isPrivateNetworkEndpoint()`，它使用的是任务创建阶段已校验、已归一化的 endpoint。

## 7. 防护效果

### 7.1 已覆盖

| 攻击或误用方式 | 防护机制 |
| --- | --- |
| 直接填 `10.x.x.x`、`192.168.x.x`、`172.16.x.x` | DNS/IP 网段校验拒绝 |
| 直接填 `127.0.0.1`、`localhost` 解析 loopback | loopback 校验拒绝 |
| 填 `169.254.x.x`、IPv6 link-local | link-local 校验拒绝 |
| 填 IPv6 ULA 地址 | `fc00::/7` 校验拒绝 |
| 填 IPv4-mapped IPv6 内网地址 | 映射地址递归校验拒绝 |
| 填 `100.64.x.x` CGNAT | 手写网段校验拒绝 |
| 直接填公网 IP | 用户 endpoint 必须是域名策略拒绝 |
| query 中注入 redirect 到内网 | `sanitizeBaseUrl()` 丢弃 query |
| 公网域名 302/307 到内网 | OkHttp 禁止跟随重定向 |
| 域名多记录中混合公网与内网 IP | `getAllByName()` 检查全部地址 |
| URL userinfo 泄漏或混淆 | sanitize 阶段拒绝 |
| path `..` / `//` 注入 | sanitize 阶段拒绝 |
| DNS 解析失败 | 失败关闭，拒绝 |

### 7.2 当前限制

1. **DNS rebinding 时间窗**

   校验发生在任务创建时，真实请求阶段 OkHttp 使用默认系统 DNS 再次解析。如果攻击者控制域名 TTL 和解析结果，可能出现：

   ```text
   创建任务时：api.example.com → 8.8.8.8，通过校验
   真实请求时：api.example.com → 10.1.2.3，连接内网
   ```

2. **未自定义连接期 DNS 校验**

   当前没有为 OkHttp 配置自定义 `okhttp3.Dns`，因此连接阶段不会重复执行同一套网段校验。

3. **未显式控制 DNS 缓存**

   项目未显式设置：

   ```text
   networkaddress.cache.ttl
   networkaddress.cache.negative.ttl
   ```

   实际缓存行为受 JVM、操作系统 resolver 和部署环境影响。

4. **非标准共享地址依赖运行时环境**

   主要依赖 JDK `InetAddress` 的地址分类方法。不同 JDK 版本对特殊地址类型的分类总体稳定，但如需严格安全边界，仍建议部署网络层兜底。

## 8. 加固建议

优先级从高到低：

1. **连接期二次校验**

   为用户供应商请求创建独立 OkHttp client，并实现自定义 `Dns`：

   ```text
   OkHttp 发起连接
       ↓
   自定义 Dns.lookup(hostname)
       ↓
   InetAddress.getAllByName(hostname)
       ↓
   逐个执行 isPrivateAddress()
       ↓
   任一内网地址则抛 UnknownHostException
   ```

   这样可以把校验从「任务创建时」下沉到「每次建连时」，直接压缩 DNS rebinding 时间窗。

2. **部署网络层兜底**

   通过安全组、NetworkPolicy、egress proxy 或防火墙禁止应用实例访问内网网段。应用层校验负责业务语义，网络层负责硬边界。

3. **强制用户请求经出网代理**

   将用户供应商流量统一导向 egress proxy，由代理只允许公网目标，并禁用内网地址和重定向。

4. **限制 DNS TTL 与缓存**

   明确 JVM DNS 缓存参数，避免长缓存掩盖地址变化；同时避免过长正向缓存降低连接期校验价值。

5. **可选：校验与建连结果一致性**

   在业务日志中记录创建时解析地址和连接时实际地址，发现漂移时告警。该方案适合审计和定位，不建议作为唯一防护。

## 9. 关键代码索引

| 职责 | 文件 |
| --- | --- |
| 任务入口与三层校验 | `ai-platform-relay-check-server/src/main/java/com/ai/relay/check/trigger/http/VendorDetectionController.java` |
| URL 归一化、DNS 校验、IP 策略、网段判断 | `ai-platform-relay-check-server/src/main/java/com/ai/relay/check/common/util/EndpointUtil.java` |
 | 模型列表预检 client，禁用 redirect | `ai-platform-relay-check-server/src/main/java/com/ai/relay/check/service/ModelListVerificationService.java` |
| 真实模型请求 client，禁用 redirect | `ai-platform-llm-client/src/main/java/com/ai/relay/llm/infrastructure/http/DefaultLlmHttpClientFactory.java` |
| 缓存对比执行器 | `ai-platform-relay-check-server/src/main/java/com/ai/relay/check/service/CacheCompareExecutor.java` |
| 校验单元测试 | `ai-platform-relay-check-server/src/test/java/com/ai/relay/check/common/util/EndpointUtilTest.java` |

