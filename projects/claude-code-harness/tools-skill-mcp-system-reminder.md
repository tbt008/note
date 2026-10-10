# Tools、Skill、MCP 与 System-Reminder 说明

来源：<https://poizon.feishu.cn/wiki/O32mwsSACizyJ5kjfojciW4ln0c>

# Tools、Skill、MCP 与 System-Reminder 说明

## 一句话总结

- `tools[]` 里放的是“模型真的可以调用的工具”。
- `messages[]` 里放的是“模型当前能看到的上下文”。
- `<system-reminder>` 只是 `messages[]` 里的文本包装，不是单独的一种 API 字段。
- `Skill` 是一个真实存在于 `tools[]` 中的工具。
- `commit`、`simplify`、`update-config` 这些是 skill 名，不是一个个独立 tool。
- skill 名会先通过 `<system-reminder>` 告诉模型，然后模型通过 `Skill` 这个工具去加载并执行 skill。
- MCP 一般会把很多具体工具直接暴露进 `tools[]`。
- MCP 相关的 `<system-reminder>` 更多是“使用说明”，不是执行入口。

## 整体结构

一次请求发给模型时，最重要的两块内容是：

1. `tools[]`
2. `messages[]`

可以先用最直观的话来理解：

- `tools[]`：模型面前有哪些“按钮”可以按
- `messages[]`：模型此刻能看到哪些“文字说明”和上下文

## 一个简化后的 Payload 长相

下面这个 JSON 不是完整原始请求，而是根据我们刚才分析过的代码和实际示例，抽出来的最关键结构：

```JSON
{
  "model": "claude-sonnet-4-6",
  "tools": [
    { "name": "Skill", "input_schema": { "...": "..." } },
    { "name": "Bash", "input_schema": { "...": "..." } },
    { "name": "Read", "input_schema": { "...": "..." } },
    { "name": "mcp__cook__wash_vegetables", "input_schema": { "...": "..." } },
    { "name": "mcp__cook__cut_vegetables", "input_schema": { "...": "..." } },
    { "name": "mcp__cook__stir_fry", "input_schema": { "...": "..." } }
  ],
  "messages": [
    {
      "role": "user",
      "content": [
        {
          "type": "text",
          "text": "<system-reminder>\nThe following skills are available for use with the Skill tool:\n\n- commit: Create a git commit with a descriptive message\n- simplify: Review changed code for reuse, quality, and efficiency\n</system-reminder>"
        },
        {
          "type": "text",
          "text": "<system-reminder>\n# MCP Server Instructions\n\nThe following MCP servers have provided instructions for how to use their tools and resources:\n\n## cook\nUse wash_vegetables before cut_vegetables. Use stir_fry only after prep.\n</system-reminder>"
        },
        {
          "type": "text",
          "text": "请帮我完成任务"
        }
      ]
    }
  ]
}
```

从这里你可以直接看出：

- `tools[]` 里有真正的 tool
- `messages[]` 里有 `<system-reminder>`
- `<system-reminder>` 只是 `messages[]` 里的 `text`

## `tools[]` 到底是什么

`tools[]` 是模型可以通过 `tool_use` 真正调用的工具列表。

比如：

- `Bash`
- `Read`
- `Edit`
- `Skill`
- `mcp__cook__wash_vegetables`
- `mcp__cook__cut_vegetables`
- `mcp__cook__stir_fry`

如果某个能力不在 `tools[]` 里，模型就不能直接把它当作工具调用。

所以 `tools[]` 是“真实可执行能力清单”。

## `messages[]` 到底是什么

`messages[]` 是模型看到的上下文消息。

它可能包含：

- 用户输入
- 助手历史消息
- 工具执行结果
- 元消息
- `<system-reminder>` 文本

一个非常关键的点是：

`<system-reminder>` 不是 API 的顶层字段，而是 `messages[].content[]` 里的普通文本块。

也就是说，它只是“包装成系统提醒风格的一段文本”。

## 什么是 Skill

Skill 可以理解成“一个可复用的高层工作流提示词”。

例如：

- `commit`
- `simplify`
- `update-config`
- `batch`

一个 skill 通常会包含：

- 名字
- 描述
- 什么时候该用
- 参数提示
- 真正被加载时的正文内容

你可以把它理解成：

- tool 更像“动作按钮”
- skill 更像“工作流模板”或者“操作说明书”

## `Skill`（tool) 和 `skill` 的区别

这是最容易混淆的一点。

### `Skill`

`Skill` 是一个真实存在于 `tools[]` 里的工具名。

模型调用它时，形状大概是：

```JSON
{
  "type": "tool_use",
  "name": "Skill",
  "input": {
    "skill": "commit",
    "args": "-m \"Fix bug\""
  }
}
```

### `skill`

`commit`、`simplify`、`update-config` 这些是 skill 名字。

它们不是一个个单独出现在 `tools[]` 里的工具，而是作为参数传给 `Skill` 这个统一入口工具。

所以要牢牢记住：

- `Skill` 是 tool 名
- `commit` 是 skill 名

不是“每个 skill 都是一个 tool”。

## 模型为什么知道 skill 可以调用

模型之所以知道 skill 是可以调用的，不是只靠猜，而是因为它同时看到了两类信息：

1. `tools[]` 里真的有一个叫 `Skill` 的工具
2. `<system-reminder>` 里明确写着：

```LaTeX
The following skills are available for use with the Skill tool:
```

这句话的意思非常明确：

- 下面这些名字是 skill
- 这些 skill 应该通过 `Skill tool` 去使用

所以模型脑子里的组合信息其实是：

- 我有一个工具叫 `Skill`
- 它要收一个 `skill` 参数
- 当前可用的 skill 名就在 reminder 里列着

这样它就能把 skill 名映射到 `Skill` 工具的参数里。

## skill 是怎么执行的

skill 的执行流程不是“在 reminder 里直接执行”，而是下面这样：

1. 模型先在 reminder 里看到可用 skill 列表
2. 模型决定某个 skill 与用户需求匹配
3. 模型发出一个 `tool_use`，调用 `Skill`
4. 系统校验这个 skill 名是否存在
5. 系统把 skill 的正文内容加载出来
6. skill 正文作为新的 meta message 注入回会话
7. 模型继续执行，此时它已经真正读到了 skill 的完整说明

因此：

- reminder 负责“发现有哪些 skill”
- `Skill` tool 负责“加载 skill 正文”
- skill 正文加载后，模型才真正按那套工作流去做事

## skill 不只是 `<system-reminder>`

这个点也很重要。

不能把 skill 理解成“只存在于 `<system-reminder>` 中的一串名字”。

skill 至少涉及三层：

1. 发现层  
通过 `<system-reminder>` 告诉模型有哪些 skill 可用
2. 调用层  
通过 `Skill` 这个真实 tool 发起调用
3. 加载层  
skill 正文被注入到消息流里，成为后续推理依据

所以 skill 不是只有 reminder，那只是第一步。

## 什么是 MCP Server

MCP server 可以理解成“外部能力提供方”或者“工具命名空间”。

一个 MCP server 往往会暴露很多具体工具。

例如：

- MCP server 名字：`cook`
- 它提供的工具：

  - `wash_vegetables`
  - `cut_vegetables`
  - `stir_fry`

在系统真正注册到 `tools[]` 后，通常会变成：

- `mcp__cook__wash_vegetables`
- `mcp__cook__cut_vegetables`
- `mcp__cook__stir_fry`

所以一个 MCP server 不是一个 tool，而是一组 tool 的来源。

## 用“做饭”例子理解 MCP

我们用一个最简单的类比：

- MCP server：`cook`
- 工具 1：`wash_vegetables`
- 工具 2：`cut_vegetables`
- 工具 3：`stir_fry`

你可以先粗略把它理解成：

- 一个服务叫“做饭”
- 它暴露了多个接口
- 每个接口对应一个具体动作

不一定底层真的是 HTTP，但为了理解，它很像：

- `/wash_vegetables`
- `/cut_vegetables`
- `/stir_fry`

所以“一个 MCP 包含多个工具”这件事，本质上就像“一个服务提供多个接口”。

## MCP 是怎么执行的

MCP 的执行方式和 skill 不一样。

### 对 skill 来说

- `tools[]` 里有一个统一入口 `Skill`
- 具体 skill 名在 reminder 里列出
- 模型调用的是 `Skill(skill: "commit")`

### 对 MCP 来说

- 一般没有一个统一的单一工具叫 `MCP`
- 而是这个 MCP server 暴露的具体工具直接进入 `tools[]`
- 模型直接调用这些具体工具

例如：

```JSON
{
  "type": "tool_use",
  "name": "mcp__cook__wash_vegetables",
  "input": {
    "vegetable": "bok choy"
  }
}
```

然后再调用：

```JSON
{
  "type": "tool_use",
  "name": "mcp__cook__cut_vegetables",
  "input": {
    "style": "sliced"
  }
}
```

最后再调用：

```JSON
{
  "type": "tool_use",
  "name": "mcp__cook__stir_fry",
  "input": {
    "heat": "medium"
  }
}
```

所以：

- skill 是通过一个统一 dispatcher `Skill` 来执行
- MCP 是把很多具体工具直接暴露给模型执行

## MCP 的 `<system-reminder>` 是干什么的

MCP 相关 reminder 更多是“说明书”，不是执行入口。

例如：

```XML
<system-reminder>
# MCP Server Instructions

The following MCP servers have provided instructions for how to use their tools and resources:

## cook
Use wash_vegetables before cut_vegetables.
Use stir_fry only after ingredients are prepared.
</system-reminder>
```

这段话的作用是告诉模型：

- 哪些 MCP server 已连接
- 这些 server 的工具/资源怎么用
- 使用顺序、限制、建议是什么

但真正执行时，模型调用的还是 `tools[]` 中的具体 MCP tool。

## skill 和 MCP 的关系

最简单的对照方式如下：

- skill：高层工作流、食谱、操作方案
- `Skill` tool：加载 skill 的统一入口
- MCP server：一组外部能力的提供方
- MCP tools：由 MCP server 提供的具体动作按钮

继续用做饭类比：

- skill = 菜谱
- `Skill` tool = 把菜谱展开并读进上下文
- MCP server = 厨房工作台
- MCP tools = 工作台上的按钮：洗菜、切菜、炒菜

## `<system-reminder>` 只是外壳，不代表类型

这一点特别关键。

`<system-reminder>` 只是消息里的文本包装方式，不等于“这是 skill”或者“这是 MCP”。

它可以承载很多不同内容：

- skill 列表
- MCP server 使用说明
- 项目上下文
- 日期提醒
- 其他元说明

所以：

- 两段内容都包在 `<system-reminder>` 里
- 不代表它们是同一类东西
- 还是要看里面实际写了什么

例如：

### skill reminder

```XML
<system-reminder>
The following skills are available for use with the Skill tool:

- commit: Create a git commit with a descriptive message
- simplify: Review changed code for reuse, quality, and efficiency
</system-reminder>
```

### MCP reminder

```XML
<system-reminder>
# MCP Server Instructions

The following MCP servers have provided instructions for how to use their tools and resources:

## cook
Use wash_vegetables before cut_vegetables.
</system-reminder>
```

这两者共用了同一个外壳，但语义完全不同。

## `tool_use` 和 `tool_result` 是怎么工作的

整条执行协议可以概括为：

1. 客户端把 `tools[]` 和 `messages[]` 发给模型
2. 模型如果要调用工具，会产出一个 `tool_use`
3. 客户端执行这个工具
4. 客户端把结果封装成 `tool_result`
5. 模型继续下一轮生成

一个最小例子如下：

助手输出：

```JSON
{
  "role": "assistant",
  "content": [
    {
      "type": "tool_use",
      "id": "toolu_123",
      "name": "Skill",
      "input": {
        "skill": "commit"
      }
    }
  ]
}
```

客户端执行后回传：

```JSON
{
  "role": "user",
  "content": [
    {
      "type": "tool_result",
      "tool_use_id": "toolu_123",
      "content": "Launching skill: commit"
    }
  ]
}
```

然后系统还可能继续注入 skill 正文，模型再接着往下推理。

## skill 路径与 MCP 路径对比

### skill 路径

1. `tools[]` 里有 `Skill`
2. `messages[]` 里有 skill listing reminder
3. 模型从 reminder 里挑一个 skill 名
4. 模型调用 `Skill(skill: "...")`
5. skill 正文被加载进上下文
6. 模型按 skill 正文继续执行

### MCP 路径

1. `tools[]` 里有很多具体 MCP tools
2. `messages[]` 里可能有 MCP server instructions
3. 模型直接调用这些 MCP tools
4. 工具结果返回
5. 模型继续执行

## 最容易记住的最终版本

如果要压缩成最短的心智模型，可以记成下面这几句：

- `tools[]` = 真正可调用的按钮
- `messages[]` = 模型当前看到的文字上下文
- `<system-reminder>` = `messages[]` 中的一种文本包装
- `Skill` = 一个真实 tool，用来加载 skill
- `skill` 名 = reminder 里列出来，作为 `Skill` 的参数传入
- `MCP server` = 一组具体工具的来源
- `MCP tools` = 通常直接出现在 `tools[]` 中供模型调用

## 最后一段超短总结

如果只保留一句话，那就是：

`Skill` 是“通过统一入口加载工作流”，MCP 是“把一组具体工具直接暴露给模型”，而 `<system-reminder>` 只是把这些信息以文字形式提示给模型看。


---
