# AIBrix KV Cache 路由机制

记录日期：2026-10-10
标签：后端/缓存/架构/大模型推理

## 背景和应用场景

大模型推理时，请求常包含重复的长前缀，例如系统提示词、项目背景和固定规范。如果每个请求都让模型从头读取相同前缀，会浪费预填充时间、GPU 算力和成本。AIBrix 的目标是复用推理引擎已经算好的前缀中间结果，并把相似请求路由到最可能命中这些结果的机器。

## 核心概念

- **KV Cache**：模型读取 prompt 后产生的中间计算结果，不是最终答案缓存。K 可理解为每个 token 的索引线索，V 可理解为每个 token 携带的信息内容；生成新 token 时可复用已算好的 K/V，避免重复读取相同前缀。
- **AIBrix Gateway**：负责请求调度，不直接运行大模型。它在普通负载均衡之外，还会判断哪台 vLLM Pod 最可能复用已有 KV Cache。
- **缓存索引**：AIBrix 不保存巨大的 KV 张量，只维护“前缀 hash → Pod”的索引表。真正数据仍保存在 vLLM Pod 或分布式缓存后端。
- **事件同步**：vLLM 通过 `BlockStored`、`BlockRemoved`、`AllBlocksCleared` 等事件通知 AIBrix 更新索引。
- **两层设计**：KVCache CRD / Controller 负责部署和管理分布式缓存后端；Gateway Prefix Cache Router 负责根据前缀索引选择更合适的 vLLM Pod。

## 示例

请求路由链路如下：

```text
用户 prompt
  -> Gateway 读取模型名和 prompt
  -> Tokenizer 切成 token
  -> 根据 token 计算前缀 hash
  -> 查询前缀索引表
  -> 有命中：选择命中率高的 Pod
     无命中：按负载策略兜底
  -> vLLM 推理
  -> vLLM 上报 KV Cache 事件
  -> AIBrix 更新索引表
```

索引示意：

```text
模型：Qwen-72B

前缀 hash 111 -> Pod A
前缀 hash 222 -> Pod A
前缀 hash 333 -> Pod B
前缀 hash 444 -> Pod C
```

## 实践结论

1. **控制面和数据面分离**：网关只保存轻量索引，不搬运或存储大 KV 张量。
2. **事件驱动**：缓存变化由 vLLM 主动上报，避免网关反复轮询所有推理实例。
3. **智能路由**：路由同时考虑缓存命中率和机器负载，而不只看机器空闲程度。
4. **后端可插拔**：HPKV、InfiniStore、Vineyard 等通过统一 reconciler 接口接入，降低替换实现成本。
5. **可降级**：KV 同步不可用时，系统可退回普通负载均衡，避免影响整体可用性。

一句话总结：AIBrix 的 KV Cache 方案核心是让网关知道每台推理机器已经算过哪些 prompt 前缀，再把相似请求发送到最省计算的那台机器。

## 参考资料

- [AIBrix 里的 KV Cache：零基础版](https://poizon.feishu.cn/wiki/ZZv0wV59jits8RkV6R4ccz2tn6e)
