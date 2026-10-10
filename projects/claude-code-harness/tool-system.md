# 工具系统

记录日期：2026-10-10
标签：Claude Code/工具/安全

## 背景

模型不能直接操作文件系统、Shell 或网络。所有副作用能力都通过统一工具系统暴露，工具系统是模型智能与真实世界之间的唯一桥梁。

## 要点

### 三层架构

- **设计层**：统一 `Tool` 接口，定义执行、Schema、权限、安全语义和 UI 渲染。
- **组装层**：编译时裁剪、运行时过滤、合并内置工具和 MCP 工具。
- **执行层**：解析 `tool_use`，执行校验、权限、Hook、调用和结果映射。

### Tool 契约

每个工具需要提供：

- `name` 与输入 Schema。
- `call()` 执行逻辑。
- `description()` 与 `prompt()`。
- `isConcurrencySafe(input)`。
- `isReadOnly(input)`。
- `isDestructive(input)`。
- `validateInput()` 与 `checkPermissions()`。
- UI 渲染方法。

安全语义绑定在工具实现上，而不是靠外部配置猜测，能随代码一起演进。

### Fail-closed 默认值

- 默认不可并发，防止竞态。
- 默认非只读，必须进入权限链。
- 默认不参与自动分类器，防止意外自动审批。

这样新工具遗漏配置时会走保守路径：多问用户，而不是静默执行危险操作。

### 执行生命周期

```text
找到工具
  -> Schema 校验
  -> 工具输入校验
  -> PreToolUse Hook
  -> 权限检查
  -> call()
  -> 大结果处理
  -> 映射 tool_result
  -> PostToolUse Hook
  -> 回传模型
```

### MCP 集成

MCP 工具通过桥接层转换成统一 `Tool`，进入同一个执行、权限和结果处理框架。连接支持多种传输、状态机和 OAuth。

## 示例

模型调用 `Read(src/tools.ts)` 后，系统先找到 FileReadTool，校验路径和输入，确认只读语义，执行读取，处理超大文件，最后生成带原 `tool_use_id` 的 `tool_result`，作为 user message 塞回下一轮上下文。

## 结论

统一接口是工具系统可扩展的关键：新增工具不需要修改主循环；权限、并发、Hook、结果截断和 UI 渲染都有稳定接入点。

## 后续行动

- 阅读 `buildTool()` 默认值与工具目录结构。
- 对比一个内置工具和一个 MCP 工具的组装路径。

## 来源

- [工具系统](https://poizon.feishu.cn/wiki/FcFpwx642ih2qokApsIciK8hnPe)
