# 上下文压缩

记录日期：2026-10-10
标签：Claude Code/上下文/压缩

## 背景

上下文压缩的各层目标相近，都是为了节省 token 和延长会话寿命，但处理对象与破坏程度不同。Claude Code 将压缩设计成多层流水线，从“单条结果太大”到“整场会话太大”逐级升级。

## 要点

### 1. Tool Result Budget

处理“太胖”的单条工具结果：

- 完整结果写入 `tool-results/`。
- 上下文保留短预览和文件引用。
- 消息位置不变，不做摘要。

适用场景：超长 Bash 输出、大文件读取、大量 grep 或 MCP 返回。

### 2. History Snip

处理“太老”的历史区间：

- 优先裁剪远离当前焦点的旧消息段。
- 保护最近对话尾部。
- 保留边界或投影标记。

### 3. Microcompact

处理“太旧”的工具结果：

- **时间型**：服务端缓存大概率已冷时，将旧工具结果正文清为占位符。
- **Cached 型**：通过 `cache_edits` 让服务端从缓存中删除旧结果，尽量不改本地 transcript。

Tool Result Budget 关注大小，Microcompact 关注时效。

### 4. Context Collapse

将一段已经完成的旧历史折叠成摘要占位：

- 原始 transcript 保留。
- API 视图替换旧区间。
- 比 Autocompact 便宜，能保留更多结构。
- Prompt Too Long 时可先提交 staged collapse 再重试。

### 5. Autocompact

最后兜底：

- 用子 Agent 生成结构化摘要。
- 重建压缩边界后的新上下文。
- 恢复最近文件、已激活技能、计划状态和延迟工具通告。
- 连续三次失败触发熔断。

## 示例

```text
原始：M8 = 50KB Bash 输出
Budget 后：M8 = 预览 + 落盘文件引用
Microcompact 后：M8 = [Old tool result content cleared]
Collapse 后：M2~M9 = 一段摘要占位
Autocompact 后：整场旧会话 = summary + 最近关键状态
```

## 结论

压缩顺序体现了风险控制：先外存化大结果，再清理旧结果，其次折叠历史区间，最后才重写整场会话。

## 后续行动

- 对比各层压缩前后的 API 请求结构。
- 验证压缩后最近文件和技能恢复机制。

## 来源

- [上下文压缩](https://poizon.feishu.cn/wiki/Vju3wTVPAidy2WkocwSckbUPnmc)
