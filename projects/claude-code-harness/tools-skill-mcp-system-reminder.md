# Tools、Skill、MCP 与 System-Reminder

记录日期：2026-10-10
标签：Claude Code/工具/技能/MCP

## 背景

Tools、Skill、MCP 和 `<system-reminder>` 经常被混淆。它们分别对应真实可调用能力、可加载工作流、外部工具来源和上下文提示外壳，语义完全不同。

## 要点

### tools[]

`tools[]` 是模型真正可调用的按钮。内置工具和 MCP 工具最终都会出现在这里，模型通过 `tool_use` 调用。

### Skill

Skill 是一个真实工具，也是统一入口。技能名称和说明可以在 reminder 中列出，但执行时模型调用：

```json
{
  "type": "tool_use",
  "name": "Skill",
  "input": { "skill": "commit" }
}
```

系统随后把技能正文加载进上下文，模型按工作流继续执行。

### MCP

MCP server 是一组外部能力的提供方。它暴露的具体工具直接进入 `tools[]`，例如 `mcp__cook__wash_vegetables`。模型不需要先调用统一的 MCP 工具，而是直接调用具体工具。

### System-Reminder

`<system-reminder>` 只是消息中的文本包装，可以承载技能列表、MCP server 使用说明、项目上下文、日期或其他元信息。同一个外壳不代表同一类型，必须看正文语义。

## 示例

```text
tools[]:
  Read
  Edit
  Skill
  mcp__cook__wash_vegetables

messages[]:
  <system-reminder>
    可用技能：commit
    cook MCP：先洗菜，再切菜，最后炒菜
  </system-reminder>
```

模型读 reminder 获得使用说明，但真正执行仍通过 `tools[]` 里的具体工具。

## 结论

最短心智模型：`tools[]` 是真正可调用按钮，Skill 是加载工作流的统一工具，MCP 是具体工具来源，`<system-reminder>` 只是提示模型的信息外壳。

## 后续行动

- 抓取一次真实 API 请求，标注 tools、Skill 调用和 MCP reminder。

## 来源

- [Tools、Skill、MCP 与 System-Reminder 说明](https://poizon.feishu.cn/wiki/O32mwsSACizyJ5kjfojciW4ln0c)
