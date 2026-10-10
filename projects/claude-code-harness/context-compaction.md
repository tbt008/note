# 上下文压缩

来源：<https://poizon.feishu.cn/wiki/Vju3wTVPAidy2WkocwSckbUPnmc>

# 上下文压缩

## 概览

在一次真正的 API 请求发出前，消息上下文会依次经过多层“减肥/压缩”处理。它们目标相近，都是为了节省 token、延长会话寿命，但处理对象和破坏程度不同。

主链路位于 src/query.ts：

1. `Tool Result Budget`
2. `History Snip`
3. `Microcompact`
4. `Context Collapse`
5. `Autocompact`

这 5 层可以理解为从“最轻量、最局部”一路走到“最重、最全局”的上下文治理策略。



<readonly-block type="isv"></readonly-block>

## 当前仓库实现状态

这份 reverse-engineered 仓库里，布线是完整的，但并不是所有层都真正实现了：

| 层 | 当前状态 | 说明 |
| --- | --- | --- |
| Tool Result Budget | 已实现 | 真实生效 |
| History Snip | Stub | 接口保留，当前基本 no-op |
| Microcompact | 已实现 | 真实生效 |
| Context Collapse | Stub | 接口、日志类型、调用点保留，主逻辑缺失 |
| Autocompact | 已实现 | 真实生效 |

因此，当前仓库里真正稳定工作的主要是：

- 大工具结果落盘替换
- 旧工具结果微压缩
- 自动全量 compact

而 `History Snip` 和 `Context Collapse` 目前主要还能看到接口、调用位置和设计意图。

## Tool Result Budget

`Tool Result Budget` 的目标是处理“太大的单条工具结果”。

典型场景：

- `Bash` 输出太长
- `Read` 读出的文件太大
- `Git log`、`grep`、MCP 返回了大段文本

处理方式：

- 将完整工具结果落盘到 `tool-results/`
- 在上下文里只保留一个较短的预览和文件引用

效果：

- 消息还在原位置
- 不是摘要
- 只是把“大正文”搬到磁盘，给模型看一个瘦身版

关键实现：

- src/utils/toolResultStorage.ts
- src/utils/toolResultStorage.ts

一句话概括：

`Budget` 解决的是“太胖”。

## History Snip

`History Snip` 的目标是处理“太旧的历史区间”。

它不是针对单条工具结果，而是针对较早的一段消息历史做剪裁，尽量把远离当前工作焦点的上下文移出活跃窗口。

从调用链和注释看，它的设计意图大致是：

- 优先剪更老的消息段
- 尽量保护最近对话尾部
- 裁掉的是消息区间，不是只清某一条内容
- 裁完后会留下边界或标记，方便后续恢复或投影

更可能被裁掉的内容：

- 很久以前的 user/assistant 往返
- 早已脱离当前任务焦点的旧分析过程
- 更早期的工具调用历史段

更可能被保留的内容：

- 最近几轮消息
- 当前正在依赖的对话尾部
- 与当前编辑或操作直接相关的上下文

相关调用和投影点：

- src/query.ts
- src/utils/messages.ts
- src/QueryEngine.ts

补充说明：

- SDK schema 里还专门考虑了“先前的 Read 因 snip 被移出上下文”后的续接问题，说明 snip 原本确实会剪走较早的读取消息，而不是仅做显示层隐藏：  
src/entrypoints/sdk/controlSchemas.ts

当前仓库状态：

- src/services/compact/snipCompact.ts 目前是 stub
- src/services/compact/snipProjection.ts 目前也是 stub

一句话概括：

`Snip` 解决的是“太老的历史段”。

## Microcompact

`Microcompact` 的目标是处理“太旧的工具结果”。

它和 `Tool Result Budget` 有相似点，但不是同一个问题。

入口位于：

- src/services/compact/microCompact.ts

当前实现里主要有两条路径：

### 3.1 时间型 Microcompact

当距离上次主线程 assistant 回复已经太久，系统会认为服务端 prompt cache 很可能已经冷掉了。这时继续保留较老的工具结果正文意义不大，于是会把旧 `tool_result` 的内容直接清成：

```LaTeX
[Old tool result content cleared]
```

关键逻辑：

- src/services/compact/microCompact.ts
- src/services/compact/microCompact.ts

### 3.2 Cached Microcompact

这一路径更巧。它尽量不改本地消息内容，而是在 API 层通过 `cache_edits` 告诉服务端：

- 某些旧工具结果可以从缓存前缀中删除

因此它的目标是：

- 省 token
- 尽量不破坏本地 transcript 结构
- 尽量维持 prompt cache 稳定

关键逻辑：

- src/services/compact/microCompact.ts

一句话概括：

`Microcompact` 解决的是“太老的工具结果”。

## Tool Result Budget 与 Microcompact 的区别

两者都在减少 `tool_result` 的 token，但关注点不同。

`Tool Result Budget`：

- 关注“太大”
- 解决单条结果过胖的问题
- 方法是落盘 + 预览替换

`Microcompact`：

- 关注“太旧”
- 解决旧工具结果还占着上下文的问题
- 方法是清正文或做缓存编辑

一句话区分：

- `Budget` 处理“尺寸问题”
- `Microcompact` 处理“时效问题”

## Context Collapse

`Context Collapse` 是最容易误解的一层。

它不是删历史，也不是像 `Autocompact` 那样把整段会话重写成 summary。它更像是对“旧消息区间”做折叠显示。

基本思想：

1. 选中一段已经完成、短期内不需要逐字保留的旧消息区间
2. 为这段区间生成摘要
3. 在“发送给模型的视图”中，用一个 `<collapsed ...>` 占位摘要替换这段原始消息
4. 原始 transcript 仍然保留，不是物理删除

这就是代码中提到的 `read-time projection`：

- src/query.ts

也就是说：

- REPL 或 transcript 层：原始消息还在
- API 上下文层：模型看到的是“折叠后的投影视图”

### 5.1 它和 Autocompact 的区别

`Autocompact` 通常会把大量旧历史整体总结，然后重建一套新上下文。

`Context Collapse` 更细粒度：

- 不是整场会话一起压
- 而是一小段、一小段地折叠
- 保留更多局部结构
- 在 token 压力不特别极端时，比 full compact 更温和

### 5.2 为什么说它是“投影式只读视图”

从注释和类型定义可以看出，collapse 的核心不是删除消息，而是记录一条“替换指令”，让 `projectView()` 在构造 API 输入时进行重放。

相关线索：

- src/commands/context/context.tsx
- src/types/logs.ts
- src/types/logs.ts

它会记录类似这些信息：

- `collapseId`
- `summaryUuid`
- `summaryContent`
- `summary`
- `firstArchivedUuid`
- `lastArchivedUuid`

这说明它持久化的是：

- 哪一段消息区间被折叠了
- 这段区间对应的摘要占位符是什么
- 以后恢复或重放时，如何把这段区间再次替换成 collapsed placeholder

### 5.3 staged 与 committed

从类型和 UI 统计字段可以看出，collapse 不一定是立即提交的，它有“排队”和“已提交”的区别。

相关线索：

- src/types/logs.ts
- src/components/ContextVisualization.tsx

这意味着它大概率遵循这样的过程：

1. 发现一段适合折叠的区间
2. 放进 staged 队列
3. 后台 agent 生成摘要
4. 成功后 commit 成正式 collapse

### 5.4 为什么它会参与 413 恢复

在 prompt-too-long 场景下，系统会先尝试把 staged collapse 全部“冲洗提交”，再重试请求。

原因是：

- collapse 比 full compact 更便宜
- 能保留更多细节
- 所以应优先于 reactive compact 或 full compact 尝试

相关逻辑：

- src/query.ts

当前仓库状态：

- src/services/contextCollapse/index.ts 是 stub
- src/services/contextCollapse/operations.ts 是 stub

一句话概括：

`Context Collapse` 解决的是“旧历史太长，但还不想做全量 compact”。

## Autocompact

`Autocompact` 是最后手段，也是最重的一层。

当前面几层都没把上下文压下来时，系统会触发 `compactConversation()`，把旧历史整体总结成 compact summary，然后重建一套新的、更短的上下文。

相关入口：

- src/services/compact/autoCompact.ts
- src/services/compact/compact.ts

处理效果通常包括：

- 生成 `compact_boundary`
- 生成 summary 消息
- 保留必要附件、计划、文件状态等
- 后续对话在 compact 后的新上下文上继续进行

此外，生成 compact 摘要时优先走 forked agent 路径，以复用 prompt cache：

- src/services/compact/compact.ts

一句话概括：

`Autocompact` 解决的是“整场会话太大了”。

## 用同一段消息举例

假设原始上下文如下：

```LaTeX
M1  user: 帮我排查登录失败
M2  assistant: 我先看认证链路
M3  assistant: tool_use(Read auth.ts)
M4  user: tool_result(200 行源码)
M5  assistant: tool_use(Read session.ts)
M6  user: tool_result(300 行源码)
M7  assistant: tool_use(Bash grep ...)
M8  user: tool_result(超长终端输出 50KB)
M9  assistant: 问题可能在 token 刷新
M10 user: 顺便看看最近一次提交
M11 assistant: tool_use(Git log)
M12 user: tool_result(很长 git 输出)
M13 assistant: 我准备给出修复方案
```

### 7.1 Tool Result Budget 后

如果 `M8` 太大，会变成：

```LaTeX
M8  user: tool_result(
      <persisted-output>
      Output too large. Full output saved to: .../tool-results/<id>.txt
      Preview:
      ...
      </persisted-output>
    )
```

特点：

- 位置不变
- 不是摘要
- 只是把大正文搬走

### 7.2 Microcompact 后

如果 `M4/M6/M8` 已经很旧，可能变成：

```LaTeX
M4  user: tool_result([Old tool result content cleared])
M6  user: tool_result([Old tool result content cleared])
M8  user: tool_result([Old tool result content cleared])
```

特点：

- 只清工具结果内容
- 消息骨架仍在

### 7.3 Context Collapse 后

如果 `M2 ~ M9` 是一段已完成分析，API 视图可能投影成：

```LaTeX
M1  user: 帮我排查登录失败
C1  meta: <collapsed id="0001">
    已检查 auth.ts、session.ts 与登录链路，
    发现 token 刷新逻辑可疑，并执行了多次代码读取和命令检查。
    </collapsed>
M10 user: 顺便看看最近一次提交
M11 assistant: tool_use(Git log)
M12 user: tool_result(...)
M13 assistant: 我准备给出修复方案
```

特点：

- 不是删掉 `M2~M9`
- 而是在 API 视图里折叠成一个摘要占位
- transcript 原始历史仍可保留

### 7.4 Autocompact 后

会更像这样：

```LaTeX
B1  system: compact_boundary
S1  user: 以下是之前对话摘要：
    - 用户要排查登录失败
    - 已检查 auth.ts / session.ts
    - 初步怀疑 token 刷新逻辑
A1  attachment: 若干保留附件/计划/文件状态
N1  recent: 最近几条保留消息
```

特点：

- 整体重建上下文
- 原先细粒度消息结构不再直接保留
- 属于最重的压缩方式

## 为什么顺序是这样

这条链路是从轻到重排列的：

1. 先处理大工具结果，因为最常见、成本最低
2. 再剪老历史，因为比做摘要便宜
3. 再清旧工具结果，继续降 token
4. 再做区间折叠，尽量保留结构
5. 最后才做全量 compact，因为破坏性最大

目标是：

- 尽量在不丢失太多细节的前提下，把上下文控制在阈值以内
- 只有前面手段都不够时，才触发真正的会话重写

## 四层机制的直觉类比

如果把上下文比作桌面资料：

- `Tool Result Budget`：大文件移进柜子，桌上留索引卡
- `Microcompact`：旧便签把内容擦掉，只留标签
- `Context Collapse`：把做完的一摞材料夹起来，贴摘要条
- `Autocompact`：整张桌面归档，只留一页总摘要和当前材料

## 总结

这套系统可以概括成四句话：

- 大结果先外存化：`Tool Result Budget`
- 老工具结果再清理：`Microcompact`
- 老历史区间可折叠：`Context Collapse`
- 实在太大才全量摘要：`Autocompact`

再用一句话收束：

`Budget` 控制大小，`Microcompact` 控制陈旧工具结果，`Collapse` 控制旧历史区间，`Autocompact` 兜底重写整场会话。


---
