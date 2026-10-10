# 多 Agent 架构

记录日期：2026-10-10
标签：Claude Code/多Agent/并发

## 背景

单个 Agent 适合线性任务，但复杂工程常包含多个可并行子任务、需要中央编排的任务，以及 Agent 之间持续交换信息的工作。Claude Code 在同一套基础设施上提供子 Agent、Coordinator 和 Swarm 三种模式。

## 要点

### 三种模式

| 模式 | 适用场景 | 特点 |
|-|-|-|
| 子 Agent | 独立子任务 | 父 Agent 等待结果后继续 |
| Coordinator | 多 Worker 并行任务 | 协调器只编排和综合，不直接执行 |
| Swarm | 对等协作 | Agent 通过命名信箱通信 |

简单独立任务优先用子 Agent；需要并行且统一综合时用 Coordinator；Agent 之间要持续对等协作时才升级到 Swarm。

### 子 Agent 生命周期

```text
解析 Agent 类型
  -> 组装子 Agent 工具池
  -> 构建专用系统提示词
  -> 创建隔离上下文
  -> 执行子任务
  -> 返回摘要和结果
```

子 Agent 类型包括 Explore、Plan、General-purpose 等。Explore 偏只读调查，Plan 偏方案设计，General-purpose 可执行完整任务。

### 上下文隔离

默认最大隔离：文件读取状态克隆、AbortController 新建、可变状态默认 no-op，只有显式 opt-in 才共享。这样单个 Worker 的故障或状态污染不会自动扩散。

### 工具过滤

子 Agent 工具池经过多级过滤：全局禁止、类型限制、自定义限制、异步白名单，最后合并 MCP 工具。权限边界不依赖单层判断。

### Fork 的真实动机

Fork 子 Agent 的关键价值是复用父级 Prompt Cache。字节级相同的继承前缀能让多个子任务共享缓存，上下文继承是缓存共享的副产品。

### Coordinator 约束

协调器不直接执行工程操作，只负责拆解、派发、跟踪、终止和综合。提示词中的“Never write based on your findings”要求协调器必须理解并总结 Worker 结果，而不是把理解责任继续外包。

### Swarm

Swarm 使用命名信箱支持 Agent 对等通信，并有 Scratchpad 作为跨 Worker 知识共享空间，减少所有信息都必须经过中央协调器的延迟和信息损耗。

## 示例

一个复杂重构可以由 Coordinator 拆成三个子任务：调查调用链、设计兼容层、编写回归测试。调查和测试 Worker 并行执行，Coordinator 汇总结果后给出统一重构方案。

## 结论

多 Agent 架构的核心不是“Agent 越多越好”，而是根据任务耦合度选择通信模式，同时通过上下文隔离、工具过滤和独立取消保证故障边界。

## 后续行动

- 阅读 `createSubagentContext()` 的默认隔离策略。
- 对比 Fork、Coordinator 和 Swarm 的缓存成本。

## 来源

- [多agent架构](https://poizon.feishu.cn/wiki/LIw0wTvdCiSFqdkqpZXceBchnuZ)
