# Hooks 与可扩展性

记录日期：2026-10-10
标签：Claude Code/Hook/扩展

## 背景

Hook 系统让外部基础设施介入 Agent Loop 的关键节点，实现审计、安全策略、CI、上下文注入和自动审批，而不要求修改 Claude Code 核心循环。

## 要点

### 27 种事件

事件覆盖完整生命周期：用户输入、模型推理、工具前后、权限请求、通知、会话起止、子 Agent、压缩前后、停止、配置变化、文件变化、Worktree 变化等。

### Hook 类型

- **Command Hook**：执行本地命令。
- **Prompt Hook**：调用模型处理提示。
- **Agent Hook**：与 Agent 生命周期联动。
- **HTTP Hook**：调用外部服务。
- **Callback Hook**：SDK / 插件编程式回调。
- **Function Hook**：会话内函数钩子。

### 执行引擎

```text
快速存在性检查
  -> 信任检查
  -> matcher 匹配
  -> Hook 去重
  -> 输入构建与并行执行
  -> 输出解析
  -> 结果聚合与事件发射
```

### 退出码协议

- `0`：静默成功。
- `1`：向用户展示提示，不改变模型行为。
- `2`：把 stdout 解析为结构化信息，交回模型或运行时处理。

### 常用能力

Hook 可以阻止工具调用、修改输入、追加上下文、自动审批、异步运行 CI、失败时唤醒模型、上报审计日志或执行安全评估。

### 安全模型

Hook 配置必须整体可信，并做配置快照、来源信任检查、环境变量白名单和注入防护。不能假设只有部分 Hook 需要信任。

## 示例

一个 PostToolUse CI Hook 可以异步运行构建；成功时保持安静，失败时以退出码 2 返回错误日志，中断模型并要求其修复。

## 结论

Hook 是 Agent Loop 的横切关注点：核心循环保持简洁，企业审计、安全策略、CI/CD 和本地工作流通过事件驱动方式接入。

## 后续行动

- 为常用项目配置最小化 CI 或审计 Hook。
- 阅读 PermissionRequest Hook 与权限系统的竞速处理。

## 来源

- [Hooks 与可扩展性](https://poizon.feishu.cn/wiki/TyRVwW0yJi1Vj0kYJLxcWKzfnVf)
