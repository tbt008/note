# Claude Code 概述

记录日期：2026-10-10
标签：Claude Code/Agent/架构

## 背景

Claude Code 的定位不是简单把 CLI 接到大模型 API，而是 Anthropic 官方的受控工具循环 Agent。它要让模型在真实软件工程中连续执行读文件、改代码、跑测试和修复问题等多步操作，同时保证操作可观察、可恢复、可控权。

AI 辅助编程可以粗分为三级：代码补全、IDE 聊天助手、自主 Agent。前两级主要生成建议，用户负责执行；Claude Code 属于第三级，模型在两次人工交互之间自主决定下一步工具调用。

## 要点

### Agent 的四个基础需求

- **循环**：不是一次生成，而是持续“思考 → 执行 → 观察”。
- **工具**：模型不直接操作机器，而是通过工具读写文件和执行命令。
- **记忆**：跨会话保留用户偏好、项目事实和当前任务状态。
- **安全控制**：真实机器上的副作用必须经过权限和验证体系。

### 核心设计原则

1. **Generator 流式架构**：API、主循环和 UI 通过异步生成器连接，token、工具结果和错误实时流出。
2. **分层安全**：权限规则、Bash AST、静态验证、分类器和用户确认共同防御。
3. **编译时 Feature Gate**：内部功能在外部构建中被物理裁剪，而不是运行时隐藏。
4. **集中状态与不可变更新**：降低子系统耦合，方便追踪状态变化。
5. **渐进式压缩**：按破坏程度处理上下文，最后才全量摘要。
6. **工具即扩展点**：新增能力优先实现统一 Tool 协议，而不是修改主循环。

### 主干结构

```text
src/
  main.tsx                 CLI 入口
  QueryEngine.ts           会话生命周期
  query.ts                 核心循环
  Tool.ts                  统一工具契约
  tools/                   内置工具
  services/compact/        上下文压缩
  services/mcp/            MCP 集成
  hooks/toolPermission/    权限处理
  memdir/                  长期记忆
  skills/                  技能系统
```

## 示例

给项目添加一个 REST endpoint 时，Claude Code 可以先搜索现有路由，读取中间件配置，再创建 handler、注册路由、编写测试、运行测试、根据失败修复代码，最后提交 commit。这个过程由模型和工具循环共同完成，而不是一次性输出代码片段。

## 结论

Claude Code 的工程价值在于把“模型生成”升级为“可信执行系统”。模型是决策者，Harness 提供循环、工具、上下文、权限和恢复机制。

## 后续行动

- 阅读 `query.ts` 时先定位主循环，再分析每个 continue site。
- 对比本概述与实际源码目录，确认模块边界的版本变化。

## 来源

- [Claude Code 概述](https://poizon.feishu.cn/wiki/UkDtwNeAriNIHkkxUnVcVnIen0f)
