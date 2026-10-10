# 模拟一次真实例子

来源：<https://poizon.feishu.cn/wiki/WgnHwapSxikLqZkhjRqcLG6SnBf>

# 模拟一次真实例子

我们就用一个最容易理解、也最典型的例子来讲：



模型想读一个文件，于是在流式输出里产生了一个 `Read` 工具调用。



假设它吐出的 `tool_use` 长这样：



```JSON
{
  "type": "tool_use",
  "id": "toolu_abc123",
  "name": "Read",
  "input": {
    "file_path": "src/tools.ts"
  }
}
```



下面我按真实代码，把这一次调用从头走到尾。



---



**第 0 步：这块东西是什么意思**



这不是模型“说给用户听的话”，而是模型在对运行时说：



- 我想调用名字叫 `Read` 的工具
- 参数是 `file_path = src/tools.ts`
- 这次调用的编号是 `toolu_abc123`

这个编号特别重要，因为后面系统回结果时，必须说：



“这是对 `toolu_abc123` 的回应。”



---



**第 1 步：流式接收时，系统先把它识别出来**

在 \[src/query.ts:859\](/Users/admin/Desktop/claude-code-main/src/query.ts#L859) 往后，系统一边收模型流式消息，一边检查 assistant message 里有没有 `tool_use` block。



当它看到这块：



```TypeScript
const msgToolUseBlocks = ...filter(content => content.type === 'tool_use')
```



见 \[src/query.ts:863\](/Users/admin/Desktop/claude-code-main/src/query.ts#L863)



就会做两件事：



- 把这次调用推进 `toolUseBlocks`
- 把 `needsFollowUp = true`

见：



- \[src/query.ts:867\](/Users/admin/Desktop/claude-code-main/src/query.ts#L867)
- \[src/query.ts:868\](/Users/admin/Desktop/claude-code-main/src/query.ts#L868)

白话一点就是：



系统说，“我知道了，这轮不是普通聊天，后面还得先去跑个工具。”



---



**第 2 步：如果开了流式工具执行，会立刻排队**

如果当前启用了 `StreamingToolExecutor`，这块 `Read` 调用还会立刻被塞进执行器：



```TypeScript
streamingToolExecutor.addTool(toolBlock, assistantMessage)
```



见 \[src/query.ts:875\](/Users/admin/Desktop/claude-code-main/src/query.ts#L875)



执行器定义在 \[src/services/tools/StreamingToolExecutor.ts\](/Users/admin/Desktop/claude-code-main/src/services/tools/StreamingToolExecutor.ts)。



对于我们的例子，它会先判断：



- `Read` 工具在不在工具池里
- 输入能不能 parse
- 这次调用是不是并发安全

`Read` 通常是只读、并发安全的，所以它大概率会被标记成可并发。



你可以把这一步理解成：



“前台已经把工单交给后厨排队了。”



---



**第 3 步：正式进入单次工具执行入口 `runToolUse()`**

真正执行一条工具调用的入口是 \[src/services/tools/toolExecution.ts:338\](/Users/admin/Desktop/claude-code-main/src/services/tools/toolExecution.ts#L338)。



这里收到的参数就是：



- 那个 `tool_use` block
- 它所在的 assistant message
- 权限判断器 `canUseTool`
- 当前会话上下文 `toolUseContext`

进来之后第一件事：



```TypeScript
let tool = findToolByName(toolUseContext.options.tools, toolName)
```



见 \[src/services/tools/toolExecution.ts:355\](/Users/admin/Desktop/claude-code-main/src/services/tools/toolExecution.ts#L355)



对于我们的例子：



- `toolName = "Read"`
- 系统在当前工具池里找到 `FileReadTool`

如果找不到，会直接报错并生成错误 `tool_result`。  

但这里能找到，所以继续。



---



**第 4 步：检查是不是已经被取消**

在真正做任何事情前，它会先看当前调用有没有已经被 abort：



- \[src/services/tools/toolExecution.ts:426\](/Users/admin/Desktop/claude-code-main/src/services/tools/toolExecution.ts#L426)

比如：



- 用户中途打断
- 上层请求取消
- 并发兄弟工具出错导致取消

如果取消了，会生成一个错误 `tool_result` 然后结束。



我们的例子先假设没有取消。



---



**第 5 步：进入“校验 + 权限 + 执行”的总流程**

接着它调用：



```TypeScript
streamedCheckPermissionsAndCallTool(...)
```



见 \[src/services/tools/toolExecution.ts:466\](/Users/admin/Desktop/claude-code-main/src/services/tools/toolExecution.ts#L466)



这个函数的意思不是“马上返回最终结果”，而是：



- 可能先吐 progress
- 最后再吐真正的 tool result

所以它是个小流。



内部真正干活的是 \[src/services/tools/toolExecution.ts:623\](/Users/admin/Desktop/claude-code-main/src/services/tools/toolExecution.ts#L623) 的 `checkPermissionsAndCallTool(...)`。



---



**第 6 步：先做 schema 校验**

第一关：



```TypeScript
const parsedInput = tool.inputSchema.safeParse(input)
```



见 \[src/services/tools/toolExecution.ts:648\](/Users/admin/Desktop/claude-code-main/src/services/tools/toolExecution.ts#L648)



对我们的例子来说，就是检查：



- `file_path` 这个字段有没有
- 类型是不是字符串
- 有没有别的结构错误

如果模型误写成：



```JSON
{ "file_path": 123 }
```



那这一步就挂了，系统会直接返回一个错误 `tool_result`，根本不会真的去读文件。



但我们现在这个输入是：



```JSON
{ "file_path": "src/tools.ts" }
```



结构上没问题，通过。



---



**第 7 步：再做 `Read` 工具自己的语义校验**

接着会调用工具自己的 `validateInput`：



```TypeScript
const isValidCall = await tool.validateInput?.(parsedInput.data, toolUseContext)
```



见 \[src/services/tools/toolExecution.ts:716\](/Users/admin/Desktop/claude-code-main/src/services/tools/toolExecution.ts#L716)



对 `Read` 这种工具，这一层一般会检查：



- 路径是不是合法
- 是不是危险设备文件
- 范围参数是否合法
- 是否越过某些读取限制

比如如果模型想读 `/dev/zero` 这种无限输出设备文件，`Read` 工具会直接拦掉，不真的去读。



我们的例子里是 `src/tools.ts`，这是普通文件，所以继续。



---



**第 8 步：PreToolUse hooks 先过一遍**

然后会先跑预执行 hooks：



- \[src/services/tools/toolExecution.ts:834\](/Users/admin/Desktop/claude-code-main/src/services/tools/toolExecution.ts#L834)

对于 `Read` 来说，hook 可能做的事情包括：



- 记日志
- 给输入补观察字段
- 额外安全检查
- 甚至阻止继续执行

大多数情况下，这一步只是让系统里的扩展机制先插手一下，不改变主流程。



如果某个 hook 说“停止”，这里就会直接返回，不读文件。



我们的例子假设没有 hook 阻止。



---



**第 9 步：权限判断**

这是新手最容易误解的地方。



“读文件”虽然通常没“删文件”敏感，但依然可能受权限模式控制。  

代码在 \[src/services/tools/toolExecution.ts:962\](/Users/admin/Desktop/claude-code-main/src/services/tools/toolExecution.ts#L962)。



它会综合考虑：



- 当前 permission mode
- 工具自己的 `checkPermissions`
- 用户配置的 allow / deny 规则
- hook 给的权限结论
- `canUseTool` 的结果

最后拿到 `permissionDecision`。



如果结果不是 `allow`，比如是 `deny` 或 `ask` 但未批准，那么不会读文件，而是直接构造：



```JSON
{
  "type": "tool_result",
  "tool_use_id": "toolu_abc123",
  "is_error": true,
  "content": "Permission denied ..."
}
```



见 \[src/services/tools/toolExecution.ts:1084\](/Users/admin/Desktop/claude-code-main/src/services/tools/toolExecution.ts#L1084) 之后。



我们的例子假设读文件被允许，所以继续。



---



**第 10 步：终于真正调用 `Read` 工具**

现在才走到：



```TypeScript
const result = await tool.call(...)
```



见 \[src/services/tools/toolExecution.ts:1272\](/Users/admin/Desktop/claude-code-main/src/services/tools/toolExecution.ts#L1272)



如果当前工具是 `Read`，这个 `tool.call` 实际就是 \[src/tools/FileReadTool/FileReadTool.ts\](/Users/admin/Desktop/claude-code-main/src/tools/FileReadTool/FileReadTool.ts) 里的实现。



这时它真的会去做文件读取相关工作，例如：



- 解析路径
- 检查文件是否存在
- 检查是不是目录、二进制、图片、PDF
- 读取内容
- 必要时加行号、截断、分页
- 返回标准化结果

你可以把这一层理解成：



前面所有流程都只是“审批和准备”，  

到这里才是真正拿起工具开工。



---



**第 11 步：`Read` 工具产出自己的结果对象**

`Read` 工具不会直接返回一个协议层的 `tool_result` block。  

它先返回项目内部的 `ToolResult`，形态大概是：



```TypeScript
{
  data: {
    ...某种读取结果...
  }
}
```



这个 `data` 可能包含：



- 文件内容
- 路径
- 是否截断
- 行号后的文本
- 其他读取元信息

也就是说，工具作者只负责“业务结果”，  

协议格式还要由执行层统一包装。



---



**第 12 步：执行层把内部结果映射成 API 认识的 `tool_result`**

执行层接到 `result.data` 后，会做这一句：



```TypeScript
const mappedToolResultBlock = tool.mapToolResultToToolResultBlockParam(
  result.data,
  toolUseID,
)
```



见 \[src/services/tools/toolExecution.ts:1369\](/Users/admin/Desktop/claude-code-main/src/services/tools/toolExecution.ts#L1369)



这一步的意思是：



把工具内部结果翻译成“Anthropic / 模型协议看得懂的 `tool_result` 结构”。



比如最终会变成类似：



```JSON
{
  "type": "tool_result",
  "tool_use_id": "toolu_abc123",
  "content": "这里是 src/tools.ts 的内容（可能带行号）"
}
```



注意这里有两个非常关键的点：



- `tool_use_id` 必须是原来那次调用的 `id`
- `content` 是工具结果，不一定是原始对象，可能已经格式化过

---



**第 13 步：如果结果太大，可能不会整段直接塞回模型**

然后会经过 `processToolResultBlock(...)` 一类逻辑，见 \[src/services/tools/toolExecution.ts:1480\](/Users/admin/Desktop/claude-code-main/src/services/tools/toolExecution.ts#L1480)。



这一步会处理现实问题，比如：



- 文本太大要不要截断
- 要不要落盘存文件，只给模型一个预览
- 是否附加图像块等内容

对 `Read` 来说，这一步尤其重要，因为文件可能很大。



所以并不是“读出来多少就原样回多少”，而是：



- 尽量让模型拿到有用的结果
- 同时避免上下文爆炸

---



**第 14 步：包装成一条 user message 返回给模型**

最后执行层会构造一条用户消息：



```TypeScript
createUserMessage({
  content: [toolResultBlock, ...],
  toolUseResult: ...,
  sourceToolAssistantUUID: assistantMessage.uuid,
})
```



见 \[src/services/tools/toolExecution.ts:1533\](/Users/admin/Desktop/claude-code-main/src/services/tools/toolExecution.ts#L1533)



这一步很容易让人惊讶：



**工具结果在会话历史里，是作为一条“user message”塞回去的。**



原因是对模型协议来说，顺序通常是：



- assistant 发起 `tool_use`
- user 返回对应的 `tool_result`

不是 assistant 自己给自己回结果。



所以这一条消息，本质上是系统代替“工具世界”对模型说：



“你刚才申请的 `Read(src/tools.ts)`，结果在这里。”



---



**第 15 步：PostToolUse hooks 再处理一下**

工具执行成功之后，还会跑后处理 hooks：



- \[src/services/tools/toolExecution.ts:1567\](/Users/admin/Desktop/claude-code-main/src/services/tools/toolExecution.ts#L1567)

对于 `Read`，这可能只是附加日志或审计信息。  

一般不会改变你对主流程的理解，但系统设计上给了扩展点。



---



**第 16 步：结果回到 query 主循环，准备下一轮模型思考**

无论是流式执行还是统一调度执行，最终这条 `tool_result` 消息都会被 `query.ts` 收集进 `toolResults`。



在流式路径里，你可以看到：



- \[src/query.ts:885\](/Users/admin/Desktop/claude-code-main/src/query.ts#L885)
- \[src/query.ts:888\](/Users/admin/Desktop/claude-code-main/src/query.ts#L888)

这样下一次调用模型时，消息历史里就会同时存在：



1. 模型刚才发出的 `tool_use`
2. 系统回给它的 `tool_result`

于是模型可以继续思考：



- 我已经读到 `src/tools.ts` 了
- 下一步要不要再读别的文件？
- 还是现在回答用户？

这就是 agent loop 的核心闭环。



---



**把这次真实例子浓缩成一句流程图**



用我们的 `Read(src/tools.ts)` 例子，真实链路就是：



1. 模型流式吐出 `tool_use(Read, file_path=src/tools.ts)`
2. `query.ts` 识别到它，登记到 `toolUseBlocks`
3. 调度器决定它可以执行
4. `runToolUse()` 找到 `FileReadTool`
5. 校验输入结构
6. 校验路径和读取语义
7. 跑 pre hooks
8. 做权限检查
9. 调 `FileReadTool.call(...)` 真正读文件
10. 把读取结果映射成 `tool_result`
11. 组装成 user message
12. 塞回下一轮上下文
13. 模型看着结果继续思考

---



**你可以把它想成一次快递流程**



- `tool_use`：模型下单
- 调度器：分拣中心
- 校验/权限：安检 + 审核
- `tool.call()`：仓库真正拣货
- `tool_result`：物流回执
- 下一轮模型：客户收到回执后决定下一步

---



如果你愿意，我下一步可以继续用第二个更“有戏剧性”的例子讲透：



- `Bash` 工具调用一次命令时，怎么处理进度、权限、失败、输出截断
- 或者 `MCP` 工具调用一次外部服务时，怎么从远端结果变成本地 `tool_result`

这两个会比 `Read` 更能看出这套系统设计的威力。


---
