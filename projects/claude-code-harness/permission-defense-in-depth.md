# 权限系统纵深防御

记录日期：2026-10-10
标签：Claude Code/权限/安全

## 背景

Claude Code 能读写文件、执行 Shell、访问网络和调用外部工具，因此每次工具调用都必须在 `allow`、`ask`、`deny` 之间做出可靠判断。权限系统的难点是既不能盲目放行，也不能让用户被无限弹窗淹没。

## 要点

### 核心组件

- **PermissionMode**：default、acceptEdits、bypassPermissions、dontAsk、plan、auto 等全局策略。
- **ToolPermissionContext**：当前会话的运行时权限状态。
- **permissionSetup**：启动时合并 CLI、用户、项目、本地、托管策略和临时规则。
- **permissions 决策总线**：串联 deny、ask、工具自检、模式和 allow 规则。
- **审批 UI**：把剩余不确定性交给用户。

### 主决策链

```text
整工具 deny
  -> 整工具 ask
  -> 工具自检
  -> 处理工具返回的 deny / ask
  -> 当前权限模式
  -> allow 规则
  -> 仍然不确定则 ask
```

前面的层给出结论后，后面的层不再执行。

### Bash 为什么最复杂

Shell 语法灵活，复合命令、wrapper、引号、相对路径和环境变量都可能绕过简单字符串匹配。Bash 权限检查包含命令拆分、模式判断、路径验证、安全语义、破坏性命令警告和沙箱选择。

### 权限模式取舍

- `default`：均衡，但需要人工确认。
- `acceptEdits`：减少编辑确认，仍保留高风险拦截。
- `bypassPermissions`：效率最高，风险也最高。
- `dontAsk`：适合脚本，但必须提前配置好规则。
- `plan`：先调查和规划，不直接执行修改。
- `auto`：结合智能判断，同时剥离过宽危险规则。

`ask` 不是失败，而是系统主动把控制权交还给用户。

## 示例

用户第一次批准 `npm test -- --runInBand` 并选择记住后，系统沉淀精确规则，例如 `Bash(npm test:*)`。下次执行类似命令可以直接通过，而不是开放整个 Bash 工具。

危险命令即使用户曾配置过过宽的 `Bash(python:*)`，auto 模式也可能剥离该规则，让命令继续经过 Bash 自检、安全检查、分类器和人工确认。

## 结论

权限系统把用户意图、静态规则、工具自检、硬性安全边界、智能判断和人工审核串成逐层收敛的决策链。它不追求某一层完美，而是接受单层失效后仍有后续防线。

## 后续行动

- 梳理权限规则表达式的精确匹配语义。
- 阅读 Bash AST 与路径验证逻辑。

## 来源

- [权限系统-纵深防御](https://poizon.feishu.cn/wiki/G5cQwek32iKm9bkJv20cqb3znZg)
