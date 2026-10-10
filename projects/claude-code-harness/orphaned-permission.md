# 孤儿权限

来源：<https://poizon.feishu.cn/wiki/RINLw1v9Biv6uuk88v2cfpMSnzh>

# 孤儿权限

`submitMessage` 里的“孤儿权限”(`orphanedPermission`) 指的是：



> 某个 `tool_use` 原本发起了权限请求，但权限结果回来的时候，原来的执行现场已经不在当前 loop 里了，于是这次权限结果成了“没人接”的状态。



定义在：

\[types/textInputTypes.ts\](/Users/admin/Desktop/claude-code-main/src/types/textInputTypes.ts#L384)



```TypeScript
export type OrphanedPermission = {
  permissionResult: PermissionResult
  assistantMessage: AssistantMessage
}
```



也就是两样东西：



- `permissionResult`：用户最终是允许还是拒绝
- `assistantMessage`：当初发起这个 `tool_use` 的那条 assistant 消息

**为什么叫“孤儿”**

正常流程应该是：



1. 模型发出 `tool_use`
2. 系统弹权限确认
3. 用户点允许/拒绝
4. 当前 loop 立刻接住这个结果，继续执行工具或返回拒绝结果

但有时候第 4 步发生不了，比如：



- 原来的 loop 已经结束了
- 进程切换 / 恢复了
- 控制响应晚到了
- 当前执行上下文已经不在原位置了

这时权限结果虽然来了，但“原来等它的那段执行链”没了，所以它变成了孤儿权限。



**在 `submitMessage()` 里怎么处理**

`submitMessage()` 开头会专门检查这个状态：



\[QueryEngine.ts\](/Users/admin/Desktop/claude-code-main/src/QueryEngine.ts#L425)



如果有 `orphanedPermission`，就先调用：



\[queryHelpers.ts\](/Users/admin/Desktop/claude-code-main/src/utils/queryHelpers.ts#L234)



也就是 `handleOrphanedPermission(...)`



意思是：



> 在正式处理这次新用户输入之前，先把上次遗留的权限结果补处理掉。



**补处理具体做什么**

`handleOrphanedPermission()` 主要做这几步：



1. 从 `assistantMessage` 里找到对应的 `tool_use block`  

见 \[queryHelpers.ts\](/Users/admin/Desktop/claude-code-main/src/utils/queryHelpers.ts#L243)



1. 根据 `permissionResult` 判断是 allow 还是 deny  

如果允许，还可能带 `updatedInput`  

见 \[queryHelpers.ts\](/Users/admin/Desktop/claude-code-main/src/utils/queryHelpers.ts#L267)



1. 确保那条 assistant `tool_use` 消息已经在消息历史里  

避免出现只有 `tool_result` 没有 `tool_use` 的断链  

见 \[queryHelpers.ts\](/Users/admin/Desktop/claude-code-main/src/utils/queryHelpers.ts#L295)



1. 重新执行这一个工具，或者生成对应拒绝结果  

它内部调用 `runTools(...)`  

见 \[queryHelpers.ts\](/Users/admin/Desktop/claude-code-main/src/utils/queryHelpers.ts#L337)



1. 把产出的消息补写回 transcript 和 SDK 输出流

**这个状态是从哪来的**

在 `print.ts` 里，如果收到一个晚到的 control response，并发现它对应的 `tool_use` 还没真正解决，就会把它入队成 `orphaned-permission`：



\[print.ts\](/Users/admin/Desktop/claude-code-main/src/cli/print.ts#L5242)



然后塞入：



- `permissionResult`
- `assistantMessage`

最后交给下一次 `submitMessage()` 处理。



**最简单的例子**

比如模型想执行 Bash：



1. assistant 发出 `tool_use(Bash)`
2. 系统向用户请求权限
3. 用户过了几秒才点“允许”
4. 但这时原来的 loop 已经不在等待这个结果了
5. 那这个“允许”就成了 orphaned permission
6. 下一次 `submitMessage()` 开始时，系统先把这个 Bash 工具补执行掉，再继续新问题

**一句话总结**

`orphanedPermission` 就是：



> 上一轮遗留下来的、已经拿到用户权限结论，但没被原执行链正常消费的工具权限结果。



它存在的目的，是为了把这类“迟到的权限结果”安全地补回消息链和工具执行链里，避免对话历史断掉。


---
