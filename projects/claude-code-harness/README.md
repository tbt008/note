# Claude Code Harness 工程笔记索引

记录日期：2026-10-10

## 背景

这里整理飞书 Wiki「harness工程」目录下的文章。每篇源文档对应一篇独立笔记，便于单独阅读、检索和后续补充；本页只做导航和整体关系说明。

## 阅读顺序

1. [Claude Code 概述](claude-code-overview.md)
2. [系统主循环](system-main-loop.md)
3. [孤儿权限](orphaned-permission.md)
4. [记忆系统](memory-system.md)
5. [上下文工程](context-engineering.md)
6. [上下文压缩](context-compaction.md)
7. [工具系统](tool-system.md)
8. [模拟一次真实例子](real-tool-call-example.md)
9. [权限系统纵深防御](permission-defense-in-depth.md)
10. [多 Agent 架构](multi-agent-architecture.md)
11. [代码编辑策略](code-editing-strategy.md)
12. [技能系统](skill-system.md)
13. [Tools、Skill、MCP 与 System-Reminder](tools-skill-mcp-system-reminder.md)
14. [系统提示词](system-prompt.md)
15. [Hooks 与可扩展性](hooks-extensibility.md)

## 整体关系

```text
主循环负责决策闭环
  -> 上下文工程控制模型看到什么
  -> 工具系统负责真实世界动作
  -> 权限系统决定动作能否发生
  -> 记忆/技能/提示词/Hook 扩展能力边界
  -> 多 Agent 与编辑策略组织复杂工程任务
```

## 来源

- [harness工程目录](https://poizon.feishu.cn/wiki/W8J1wBui1iamGsk82kPcGQiQnjg?fromScene=spaceOverview)
