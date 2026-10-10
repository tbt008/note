# 系统提示词

记录日期：2026-10-10
标签：Claude Code/提示词/行为设计

## 背景

系统提示词是 Claude Code 行为稳定性的基础。它不仅声明身份，还把任务执行、风险评估、工具使用、输出格式和运行环境规则写成明确约束。

## 要点

### 静态部分

- **Intro**：身份定义。
- **System**：运行环境和平台规则。
- **Doing Tasks**：编码原则、调查流程、验证要求。
- **Actions**：风险评估框架和审批语义。
- **Using Your Tools**：工具选择和使用规则。
- **Tone and Style**：语气、格式和沟通方式。
- **Output Efficiency**：控制冗余输出。

### 动态部分

输出风格、语言偏好、MCP server 指令、动态 Agent、当前环境等信息按需注入。静态与动态边界清晰，有助于前缀缓存命中。

### 内置 Agent 提示词

- **Explore Agent**：只读调查，快速收集结构和证据。
- **Plan Agent**：产出可执行方案，避免直接修改。
- **General-purpose Agent**：执行完整任务并验证结果。
- **Verification Agent**：给出可复现的验证步骤和结果。
- **Statusline / Guide 等专用 Agent**：面向特定 UI 或说明任务。

### Coordinator 提示词

Coordinator 提示词规定角色、工具、Worker 管理、任务阶段、并发策略、失败处理和结果综合。它强调协调器不能只做消息转发，必须理解并总结 Worker 的工作。

## 示例

一个 Verification Agent 的输出应该类似：

```text
Check: POST /api/register validation
Command: npm test -- register.test.ts
Result: passed
Evidence: 18 tests passed, 0 failed
```

而不是只写“验证通过”。

## 结论

系统提示词是运行时行为的规格说明。Claude Code 的重点不是“写一段人设”，而是按职责拆分静态规则、动态上下文和专用 Agent 行为边界。

## 后续行动

- 对比不同内置 Agent 的工具池与提示词差异。
- 整理自己项目 CLAUDE.md 与系统提示词的职责边界。

## 来源

- [系统提示词](https://poizon.feishu.cn/wiki/KVVXw0rUtiVetskJSXzcXe43nae)
