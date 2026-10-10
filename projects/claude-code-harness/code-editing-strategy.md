# 代码编辑策略

记录日期：2026-10-10
标签：Claude Code/代码编辑/安全

## 背景

代码编辑是 Coding Agent 最核心也最危险的能力。一次错误编辑可能破坏整个代码库，因此 Claude Code 的编辑策略围绕最小破坏性、可验证性和抗幻觉设计。

## 要点

### 两种基础工具

- **FileEditTool**：search-and-replace，默认优先，用于局部修改。
- **FileWriteTool**：全文件写入，用于创建新文件或完整重写。

### Search-and-replace 的优势

1. **低破坏性**：只替换匹配片段，不重写无关内容。
2. **可验证性**：old/new 提供明确的 before/after。
3. **抗幻觉**：必须匹配现有代码，不能凭空写入。
4. **Token 效率**：不需要整文件重写。
5. **Git 友好**：产生更小、更可审查的 diff。

### 编辑前读取要求

修改文件前必须先读取相关内容。系统维护文件状态缓存，用于检测模型是否基于过期内容编辑；如果文件并发变化，编辑会被阻止，避免覆盖外部修改。

### 验证管线

编辑输入经过预处理和多层验证：匹配内容唯一性、路径合法性、权限、缩进保持、删除语义、diff 生成和编辑去重。匹配不唯一时要求提供更多上下文，而不是随机选择第一个匹配。

### 多文件编辑

- 默认串行执行，避免竞态。
- 遇到依赖关系时做级联保护。
- Worktree 隔离用于并行实验，避免污染主工作区。
- 原子写入降低半成品状态概率。
- LSP 集成帮助发现引用和诊断错误。

### 换行与编码

FileWriteTool 使用 LF 保持一致性，同时尽可能检测原文件编码，避免中文或特殊字符被破坏。

## 示例

```text
old_string:
function getUser(id) {
  return db.user.find(id)
}

new_string:
function getUser(id) {
  return db.user.findRequired(id)
}
```

系统验证 old_string 唯一匹配后执行替换；如果多个文件或多个位置匹配，则要求补充上下文。

## 结论

好的编辑策略不是让模型“更会写代码”，而是让它以最小、可验证、可回滚的方式修改代码。局部替换优先、全量写入兜底、编辑前必读、并发状态检测是四个关键约束。

## 后续行动

- 阅读 FileEditTool 的匹配唯一性和文件状态缓存实现。
- 梳理 NotebookEditTool 与普通文件编辑的差异。

## 来源

- [代码编辑策略](https://poizon.feishu.cn/wiki/DPRxwYwOtiK9ctkGjYdc9XNqnYf)
