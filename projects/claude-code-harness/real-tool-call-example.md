# 模拟一次真实工具调用

记录日期：2026-10-10
标签：Claude Code/工具调用/链路

## 背景

这篇文章用一次 `Read(src/tools.ts)` 调用解释工具系统从模型输出到结果回传的完整链路。它是理解 Agent Loop 的最短路径。

## 要点

### 1. 模型产生 tool_use

```json
{
  "type": "tool_use",
  "id": "toolu_abc123",
  "name": "Read",
  "input": { "file_path": "src/tools.ts" }
}
```

`id` 用于把后续结果精确关联到这次调用。

### 2. 流式识别与排队

系统在流式接收 assistant 消息时识别 `tool_use` block。如果启用 StreamingToolExecutor，解析完整的调用会提前排队；Read 是只读并发安全工具，通常可以并行执行。

### 3. 执行前检查

执行层依次完成：

1. 在工具池中查找 `Read`。
2. 检查请求是否已取消。
3. Schema 校验。
4. 工具自身路径与读取语义校验。
5. PreToolUse Hook。
6. 权限判断。

### 4. 真正执行

系统调用 FileReadTool 的 `call()`，读取文件并生成内部结果，包括文件内容、行号、路径、是否截断等元信息。

### 5. 映射结果

内部结果被包装成模型协议可理解的：

```json
{
  "type": "tool_result",
  "tool_use_id": "toolu_abc123",
  "content": "src/tools.ts 的内容"
}
```

如果结果太大，会截断或落盘，只把预览交给模型。`tool_result` 最终作为 user message 返回，因为协议由工具方向 assistant 消息回应结果。

## 示例

```text
tool_use(Read)
  -> 流式识别
  -> 并发排队
  -> 找到 FileReadTool
  -> schema / path / hook / permission
  -> call()
  -> tool_result(user message)
  -> 模型继续思考
```

## 结论

一次工具调用不是简单函数执行，而是一条包含身份关联、校验、权限、并发、结果预算和协议映射的受控流水线。

## 后续行动

- 对照 Bash 或 MCP 工具复走这条链路。
- 观察超大 Read 结果落盘后的上下文表现。

## 来源

- [模拟一次真实例子](https://poizon.feishu.cn/wiki/WgnHwapSxikLqZkhjRqcLG6SnBf)
