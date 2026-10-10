# 权限系统-纵深防御

来源：<https://poizon.feishu.cn/wiki/G5cQwek32iKm9bkJv20cqb3znZg>

# 权限系统-纵深防御

如果只用一句话概括这套权限系统，它的核心是：

每一次工具调用，系统都要在 `allow`、`ask`、`deny` 三者之间做决定；为了避免误判，它不是只靠一层判断，而是使用“多层防线”的纵深防御架构。

核心文件：

- `src/types/permissions.ts`
- `src/Tool.ts`
- `src/utils/permissions/permissionSetup.ts`
- `src/utils/permissions/permissions.ts`
- `src/tools/BashTool/bashPermissions.ts`
- `src/hooks/useCanUseTool.tsx`
- `src/components/permissions/PermissionRequest.tsx`

---

## 一、先建立整体图景

### 权限系统到底在解决什么问题

这个 CLI 不是只会回答问题，它还会：

- 读文件
- 写文件
- 编辑文件
- 执行 Bash / PowerShell
- 发起 Web 请求
- 调用 MCP 工具
- 启动子 agent

这意味着模型不只是“说”，还会“做”。

一旦“做”的权限没有控制好，风险就会非常大，例如：

- 误删文件
- 改坏配置
- 读取敏感信息
- 执行高风险命令
- 在错误目录里改代码
- 在无交互场景下卡死等待审批

所以权限系统需要同时做到两件事：

1. 尽量安全
2. 尽量别烦死用户

如果每次都弹窗，用户会很痛苦。

如果什么都不问，系统又太危险。

这就是为什么它必须是“分层决策”的，而不是一个简单的 if/else。

### 最终只有三种结论

在 `src/types/permissions.ts` 里，系统最终只会给每次工具调用一个裁决：

- `allow`：直接执行
- `ask`：先问用户
- `deny`：直接拒绝

你可以把整套权限系统理解成一台非常复杂的“三选一机器”。

### 为什么是纵深防御

纵深防御的意思是：不依赖单点正确。

如果只靠一层：

- 只靠规则，规则可能漏
- 只靠字符串检查，容易被绕过
- 只靠分类器，分类器可能误判
- 只靠用户确认，用户会被大量弹窗疲劳

所以它把防线拆成多层，每层关注不同问题：

1. 权限上下文层：当前会话到底处于什么权限状态
2. 静态规则层：有没有已经声明过的允许/拒绝/必问规则
3. 工具自检层：这个工具对这次输入怎么看
4. 硬性安全层：有没有必须拦住的高风险情况
5. 智能判断层：静态规则不好覆盖时，交给分类器
6. 人工兜底层：还不确定，就让用户决定

这就像实验室的多道门：

- 大楼门禁
- 楼层刷卡
- 实验室密码
- 危险柜审批
- 最后还有人工值守

前一层漏了，后一层还能补。

### 一张图先看懂整体架构

<readonly-block type="isv"></readonly-block>

读这张图时，最重要的是抓住两点：

- 不是每一层都会参与每一次请求，前面已经给出结论时，后面就不会再跑。
- `ask` 不是失败，而是“自动系统在这里主动把控制权交还给用户”。

---

## 二、架构设计：系统由哪些部件组成

## 1. `PermissionMode`：权限系统的总开关

定义在 `src/types/permissions.ts`，展示逻辑在 `src/utils/permissions/PermissionMode.ts`。

主要模式：

- `default`
- `acceptEdits`
- `bypassPermissions`
- `dontAsk`
- `plan`
- `auto`（内部/特性开关控制）

这些模式不是单条规则，而是“系统整体怎么工作”的总策略。

可以把它理解为汽车的驾驶模式：

- `default` 像普通模式
- `acceptEdits` 像舒适模式
- `bypassPermissions` 像赛道模式
- `dontAsk` 像无人驾驶脚本模式
- `plan` 像勘探模式
- `auto` 像系统先帮你判断风险的智能模式

模式的存在，让同一套权限系统既能适应谨慎用户，也能适应高效率用户。

## 2. `ToolPermissionContext`：运行时权限地图

定义在 `src/Tool.ts`。

这是最核心的运行时对象之一。它包含：

- `mode`
- `alwaysAllowRules`
- `alwaysDenyRules`
- `alwaysAskRules`
- `additionalWorkingDirectories`
- `isBypassPermissionsModeAvailable`
- `isAutoModeAvailable`
- `shouldAvoidPermissionPrompts`
- `awaitAutomatedChecksBeforeDialog`
- `prePlanMode`

这个对象可以理解为：

当前会话在“权限视角”下的完整状态快照。

后面的所有判断，都不是直接去翻 settings 文件，而是读这个上下文。

好处：

- 判断逻辑集中
- 不用到处重复读配置
- 模式切换、会话变更、临时授权都可以汇总到一个地方

## 3. `permissionSetup.ts`：系统启动时的权限装配厂

主要在 `src/utils/permissions/permissionSetup.ts`。

它负责把多来源信息合并成 `ToolPermissionContext`。来源包括：

- CLI 参数，比如 `--allowedTools`
- 用户级配置
- 项目级配置
- 本地配置
- 托管策略配置
- 会话临时规则
- 额外开放目录

这里的设计思路很像“配置编译”：

- 原始材料：来自各个配置来源
- 编译输出：当前会话可直接使用的权限上下文

为什么这么做有价值？

因为真实项目里，权限从来不是一个来源说了算：

- 用户想临时允许一类操作
- 项目希望团队共享一批规则
- 企业管理员可能强制禁止某些能力
- 某次会话又会产生“以后别再问”的临时授权

如果不先统一汇总，后面的权限判断会非常混乱。

## 4. `permissionsLoader.ts`：从磁盘加载规则

主要在 `src/utils/permissions/permissionsLoader.ts`。

它负责从设置源读取权限规则，并支持：

- 多来源加载
- 托管策略优先
- 只允许 managed rules 的模式
- 对编辑场景的宽松读取

这说明设计者考虑的不只是个人使用，也考虑企业治理：

- 用户自己的规则
- 项目规则
- 管理员下发规则

它们可以共存，而且来源可追踪。

## 5. `permissions.ts`：权限决策总线

主要在 `src/utils/permissions/permissions.ts`。

这是整个系统的“大脑”。

核心函数包括：

- `hasPermissionsToUseTool()`
- `hasPermissionsToUseToolInner()`
- `checkRuleBasedPermissions()`
- `createPermissionRequestMessage()`

这里负责把所有信息汇总成最终的 `allow / ask / deny`。

## 6. `useCanUseTool.tsx`：把决策接进交互流程

主要在 `src/hooks/useCanUseTool.tsx`。

它做两件关键的事：

1. 调用权限判断函数拿到结果
2. 如果结果是 `ask`，把请求放进审批队列，由 UI 处理

所以它是“权限判断”和“审批交互”之间的桥。

## 7. `PermissionRequest.tsx`：用户审批入口

主要在 `src/components/permissions/PermissionRequest.tsx`。

它根据工具类型选择不同的审批组件：

- Bash → `BashPermissionRequest`
- FileEdit → `FileEditPermissionRequest`
- FileWrite → `FileWritePermissionRequest`
- WebFetch → `WebFetchPermissionRequest`
- 其他 → `FallbackPermissionRequest`

好处是：

- 用户看到的是和工具类型匹配的审批 UI
- Bash 可以展示命令级信息
- 文件编辑可以展示 diff
- 不同工具的审批体验更自然

## 一张表看懂“谁负责什么”

| 层 | 主要文件 | 负责回答的问题 | 典型产出 |
| --- | --- | --- | --- |
| 权限上下文层 | src/utils/permissions/permissionSetup.ts | 当前会话整体处于什么权限状态？ | ToolPermissionContext |
| 规则层 | src/utils/permissions/permissions.ts src/utils/permissions/permissionRuleParser.ts | 有没有现成规则能直接决定 allow/ask/deny？ | 命中规则后的初步裁决 |
| 工具自检层 | 各 Tool.checkPermissions() | 这个工具对本次具体输入怎么看？ | 输入级别的 allow/ask/deny |
| 硬性安全层 | pathValidation.ts dangerousPatterns.ts | 有没有即使高权限模式也不能轻易跳过的风险？ | safetyCheck、强制 ask/deny |
| 智能判断层 | permissions.ts yoloClassifier.ts | 静态规则不够时，是否能根据上下文做动态判断？ | classifier allow/deny |
| 人工兜底层 | useCanUseTool.tsx PermissionRequest.tsx | 自动系统不确定时，用户是否批准？ | 最终用户决策、权限更新 |

---

## 三、主决策链：一次 tool call 是怎么被判断的

这一部分是整套系统最重要的主线。

核心入口在 `src/utils/permissions/permissions.ts` 的 `hasPermissionsToUseTool()` 和 `hasPermissionsToUseToolInner()`。

可以把它想象成机场安检通道：

每个工具调用来了以后，先过前面的固定检查；前面已经明确放行或拦截了，后面就不用再问。

## 第一步：看会不会被直接 deny

系统首先检查：

- 这个工具是否命中了整工具 deny 规则

例如：

- `Bash` 整体被禁用
- 某个 MCP server 的工具整体被禁用

如果命中，直接 `deny`。

为什么第一步先 deny？

因为 deny 规则是最强表达。

它代表“无论别的东西怎么说，这个都不该做”。

安全系统里通常都会优先处理显式禁止。

## 第二步：看会不会被要求总是 ask

系统接着检查：

- 这个工具是否命中了整工具 ask 规则

例如：

- `Bash` 总是需要审批
- `WebFetch` 总是先问

这一步的意义是：

有些能力不是要完全禁掉，而是要“保留人工确认”。

## 第三步：让 tool 自己检查输入

这是最关键的层之一。

系统会调用：

- `tool.checkPermissions(parsedInput, context)`

为什么这一层一定要有？

因为“允许某个工具”不等于“允许这个工具的所有输入”。

例子：

- 允许文件编辑，不代表允许编辑 `.ssh/config`
- 允许 Bash，不代表允许运行 `rm -rf /`
- 允许读文件，不代表允许读私钥

所以每个工具有权对自身输入做内容级检查。

这一步可能返回：

- `allow`
- `ask`
- `deny`
- `passthrough`

`passthrough` 的意思是：

tool 自己没有做出最终结论，把问题交还给主流程继续判断。

## 第四步：处理 tool 自己给出的 deny / ask

如果工具自检明确返回 `deny`，就直接拒绝。  
如果工具自检返回某类重要的 `ask`，也必须尊重。

这里有一类特别重要的检查：

- `safetyCheck`

仓库里对某些安全检查是非常强硬的。  
它们甚至对 `bypassPermissions` 这种高权限模式也具有“免疫性”。

这意味着：

某些风险太高了，不能因为用户开了更激进的模式就直接略过。

这类设计非常像操作系统里的“内核保护区”：

- 普通权限能绕过很多东西
- 但不能绕过最核心的保护边界

## 第五步：模式是否允许直接通过

经过前面的规则和 tool 自检后，系统才会看模式层。

例如：

- 当前是不是 `bypassPermissions`
- 当前是不是 `dontAsk`
- 当前是不是 `auto`

注意这一点很重要：

模式不是从一开始就统治一切，而是在规则和安全检查之后才介入。

这说明设计者不是把模式当“万能开关”，而是把它当“后续策略调节器”。

### `bypassPermissions`

如果当前是 `bypassPermissions`，并且前面的硬性拦截没有触发，系统会倾向直接 `allow`。

但这不是“所有东西都能过”。

前面如果命中了：

- deny 规则
- ask 规则中的硬性要求
- safetyCheck

仍然可能不会无条件放行。

这就是它和“完全裸奔模式”的区别。

## 第六步：看 allow 规则能不能直接放行

在完成前面的安全和模式处理后，系统会检查：

- 是否命中了 allow 规则

例如：

- `Bash(npm test:*)`
- `FileRead(src/**)`
- `mcp__myserver__*`

如果命中，就可以直接 `allow`。

allow 规则的价值在于减少噪音：

- 高频低风险操作没必要每次都问
- 用户已经表达过意图，不该总让系统重新猜

## 第七步：都没决定，就转成 ask

如果前面都没有给出明确结论，系统就会把这次请求转成 `ask`。

然后通过 `useCanUseTool.tsx` 进入审批队列，最终由 UI 呈现给用户。

这个设计非常关键，因为它说明系统是“默认保守”的：

- 能证明安全，就 allow
- 能证明危险，就 deny
- 证明不了，就 ask

这是一种典型的工程化安全思路。

## 一张时序图看懂一次真实判定

下面这张图对应的是“模型发起一次 tool call，系统一路判断，最后可能进入审批”的完整过程。

<readonly-block type="isv"></readonly-block>

这张图最值得注意的点有三个：

- 规则层和 tool 自检层都发生在审批 UI 之前。
- classifier 并不是所有请求都走，主要是 `auto` 模式下的额外判断层。
- 用户批准后，不只是“这次通过”，还可能顺手更新规则，让后续相似操作更顺滑。

## 主决策链速查版

如果你想快速记住 `hasPermissionsToUseTool()` 的主逻辑，可以先记这个缩略版：

1. 先看有没有明确 `deny`
2. 再看有没有明确 `ask`
3. 再让工具自己判断这次输入
4. 再处理敏感路径和硬性安全检查
5. 再看当前模式是否改变默认行为
6. 再看 allow 规则能不能直接放行
7. 实在不确定，就 `ask`

它背后的工程思想是：

- 先处理最硬的禁止
- 再处理最敏感的风险
- 再做便利化放行
- 最后才把剩余不确定性交给用户

---

## 四、规则系统：规则是怎么表达和匹配的

## 规则的数据结构

规则相关定义在 `src/types/permissions.ts`。

一条规则大致可以理解为：

- 从哪里来的
- 针对哪个工具
- 是 allow / ask / deny
- 是否带内容模式

规则来源包括：

- `userSettings`
- `projectSettings`
- `localSettings`
- `flagSettings`
- `policySettings`
- `cliArg`
- `command`
- `session`

好处是：

- 规则可追踪
- 规则可解释
- 规则可以按来源读写
- 托管规则能和用户规则并存

## 规则字符串解析

在 `src/utils/permissions/permissionRuleParser.ts` 中，规则支持这种形式：

- `Bash`
- `Bash(npm test:*)`
- `FileRead(src/**)`

解析器还处理：

- 转义括号
- 兼容旧工具名
- 空括号归一化
- `(*)` 归一化成整工具规则

为什么要认真做解析器？

因为规则是要持久化到配置文件里的。

如果没有稳定的解析和序列化规则：

- 用户存进去的规则可能读不回来
- 老版本规则可能失效
- 带特殊字符的内容会损坏

这看起来是小细节，其实是配置系统能否长期稳定的重要基础。

## 规则为什么要有 allow / ask / deny 三种

不是只有白名单和黑名单。

`allow`

- 表示这类操作通常安全，没必要打扰用户

`deny`

- 表示这类操作明确不允许

`ask`

- 表示这类操作不一定错误，但必须经过人工确认

这三者都需要，是因为现实需求不是非黑即白。

例如：

- `npm test` 可能适合 allow
- `rm -rf /` 适合 deny
- `git push` 可能适合 ask

---

## 五、Bash 为什么最复杂：它到底难在哪

这部分是权限系统里最值得单独讲的地方。

对应核心文件：

- `src/tools/BashTool/bashPermissions.ts`
- `src/tools/BashTool/bashSecurity.ts`
- `src/tools/BashTool/pathValidation.ts`
- `src/tools/BashTool/modeValidation.ts`
- `src/utils/bash/*`

## Bash 难的不是“危险”，而是“太灵活”

Bash 的问题不是只有危险命令多，而是表达方式极其自由。

同一个意图，可以写成很多变体：

- `rm -rf /tmp/foo`
- `r""m -rf /tmp/foo`
- `$(echo rm) -rf /tmp/foo`
- `bash -c 'rm -rf /tmp/foo'`
- `eval "rm -rf /tmp/foo"`

如果你只靠简单字符串匹配，很容易被绕过。

所以 Bash 权限系统不能只问：

这行文本长得像不像危险命令？

而要问：

这行命令真正会执行什么？

## Bash 权限检查的大致组成

可以把 Bash 的权限判断拆成几块：

1. 命令拆分
2. 模式相关的快速判断
3. 路径相关检查
4. 安全语义检查
5. 规则匹配
6. 分类器/推测性审批
7. 最后人工确认

### 命令拆分

复杂 Bash 命令可能包含：

- `&&`
- `||`
- `;`
- pipe
- 重定向

系统会先尝试把复合命令拆成子命令，再分别分析。

这很重要，因为：

- 一整串命令里，可能只有一个子命令危险
- 用户也更容易理解“到底哪一部分需要审批”

### 模式相关判断

在 `src/tools/BashTool/modeValidation.ts` 中，`acceptEdits` 模式会对部分文件系统命令更宽松。

例如：

- `mkdir`
- `touch`
- `rm`
- `mv`
- `cp`
- `sed`

这说明 `acceptEdits` 并不是“允许所有 Bash”，而是只对一小类预期中的编辑命令开绿灯。

### 路径验证

在 `src/tools/BashTool/pathValidation.ts` 中，Bash 会对涉及文件路径的命令提取路径并检查：

- 是否在允许的工作目录里
- 是否涉及危险删除路径
- 是否涉及敏感配置路径

例如它会特别防：

- `rm -rf /`
- `rm -rf ~`
- `rm -- -/../.claude/settings.local.json`

这说明系统不是只看“rm”这个命令，而是看“rm 到底删哪儿”。

### 安全语义检查

这部分用来拦截一些结构上可疑的 Bash 写法。

它的目标不是穷举所有危险命令，而是识别：

- 会执行任意代码的结构
- 会隐藏真实意图的结构
- 会让简单规则失效的结构

## 为什么不能只用正则

这是小白特别容易问的问题。

答案很简单：

因为 Bash 不是普通字符串，它是一种编程语言。

你不能用“搜关键词”的方式可靠理解一门编程语言。

例如：

- 拼接
- 变量展开
- 命令替换
- shell wrapper
- 转义
- 子进程

这些都会让“表面文本”和“真实行为”分离。

所以系统必须尽量做结构化理解，而不是停留在文本表面。

## Bash 权限系统的核心好处

这套 Bash 权限设计带来几个明显好处：

1. 更难被表面变形绕过
2. 能对复合命令做更细粒度分析
3. 能把“命令本身”和“目标路径”一起考虑
4. 能为审批 UI 提供更好的解释
5. 能让 allow 规则更精确，而不是把整个 Bash 一把放开

## 再补一张 Bash 专属小图

<readonly-block type="isv"></readonly-block>

这张图说明 Bash 的本质难点不是“命令多”，而是“需要同时理解结构、路径、模式、规则和上下文”。

---

## 六、权限模式逐个讲清楚

模式控制的是“默认策略”，不是直接取代全部安全检查。

## 1. `default`

这是最普通的模式。

它的思路是：

- 规则明确允许的，allow
- 规则明确拒绝的，deny
- tool 自检明确危险的，deny 或 ask
- 其余很多不确定操作，ask

这是一个比较平衡的模式，适合日常交互。

### 好处

- 安全感较强
- 用户保留最终控制权
- 误操作成本较低

### 缺点

- 高频操作会比较容易打断用户

## 2. `acceptEdits`

这个模式不是“接受一切修改”，而是“对一部分预期内编辑操作更宽松”。

在 Bash 里，它主要对一些文件系统命令做自动允许，如：

- `mkdir`
- `touch`
- `rm`
- `mv`
- `cp`
- `sed`

这个模式的设计意图是：

用户已经接受“模型帮我做本地编辑”这件事，但还不等于接受模型任意执行脚本和命令。

### 好处

- 降低常规改代码任务的打断频率
- 比 `bypassPermissions` 更保守

### 风险

- 仍需确保路径和敏感操作有额外防线

## 3. `bypassPermissions`

这是高权限模式。

它的思路是：

只要没有命中前面的硬性阻断，就尽量别弹确认。

但注意，它不是绝对无边界。

一些强安全检查仍然可能保留，例如：

- 敏感路径
- safetyCheck
- tool 自检给出的关键 ask/deny

### 好处

- 速度最快
- 高信任场景下效率极高

### 风险

- 一旦前面的规则或工具判断失误，人工兜底减少
- 不适合陌生项目或高风险目录

## 4. `dontAsk`

这个模式主要面向不能交互的场景。

它的逻辑是：

- 如果一个操作本来需要 ask
- 但当前上下文又不能弹窗
- 那就直接 deny

这类场景包括：

- headless 运行
- 后台 agent
- 无终端交互环境

### 好处

- 不会卡死在等待用户确认
- 适合自动化

### 风险

- 很多原本“用户本可同意”的操作，会直接失败

## 5. `plan`

这个模式更偏“先想清楚，再执行”。

它和普通执行模式的差别，不只是名字变了，而是代表一种工作方式：

- 模型优先产出计划
- 需要退出 plan 才进入更激进执行
- 会记录 `prePlanMode` 以便恢复之前模式

### 好处

- 适合高风险任务前的勘察
- 让用户在真正执行前先看到行动方案

### 风险

- 多一步切换，效率略低

## 6. `auto`

这是最有特点的模式。

它的思路不是“全部自动批准”，而是：

对原本要 ask 的操作，尽量先交给分类器判断。

流程上通常会先：

1. 检查是不是 tool 的快速安全路径
2. 检查 `acceptEdits` 快速路径
3. 检查 allowlisted tool
4. 再调用 classifier

如果 classifier 认为危险，就 deny；

如果 classifier 认为安全，就 allow；

如果 classifier 不可用或上下文过大，可能回退到人工确认。

### 好处

- 减少大量机械确认
- 能捕捉静态规则没覆盖到的风险
- 比单纯 allowlist 更有弹性

### 风险

- 分类器不是完美的
- 所以系统才需要 denial tracking 和回退机制

## auto 模式为什么要剥离危险规则

在 `src/utils/permissions/permissionSetup.ts` 和 `src/utils/permissions/dangerousPatterns.ts` 中，进入 auto 模式时会剥离危险 allow 规则，例如：

- `Bash(*)`
- `Bash(python:*)`
- `Bash(node:*)`
- `PowerShell(iex:*)`

为什么？

因为如果这些规则还保留，那么：

- 命令会在分类器介入前被直接 allow
- auto 模式的智能防线就被架空了

这是一种非常成熟的安全设计：

不只防当前请求，还防“旧授权规则把新安全机制绕过去”。

## 模式对照速查表

| 模式 | 核心目标 | 默认倾向 | 适合什么场景 | 最大风险 |
| --- | --- | --- | --- | --- |
| default | 平衡安全与效率 | 不确定就 ask | 日常交互式开发 | 高频任务打断较多 |
| acceptEdits | 放宽常规编辑操作 | 对部分编辑类操作更容易 allow | 本地改代码、修测试 | 用户容易误以为“所有命令都更宽松” |
| bypassPermissions | 尽量别弹确认 | 尽量 allow | 高信任、熟悉项目、追求速度 | 人工兜底大幅减少 |
| dontAsk | 不能交互时不阻塞 | ask 转 deny | headless、后台、自动化 | 原本可确认通过的操作会直接失败 |
| plan | 先规划再执行 | 收紧执行，强调计划 | 高风险任务前的勘察阶段 | 效率略低，需要额外切换 |
| auto | 用 classifier 取代一部分人工确认 | 先自动评估，再 allow/deny | 需要减少机械弹窗但又不想完全放开 | 依赖 classifier，需要回退机制 |

---

## 七、人工确认层：为什么不是“最后没办法才问人”

很多人会误以为人工确认说明系统不够高级。

其实正相反。

人工确认之所以必要，是因为权限最终服务的是用户意图，而用户意图不能被完全形式化。

例如：

- 删除目录，在一个任务里可能是错误，在另一个任务里就是目标
- 修改配置，在一个场景里是危险，在另一个场景里是修复动作

所以系统做得再聪明，也很难完全代替用户。

审批层的价值是：

1. 把最终控制权留给人
2. 给自动系统兜底
3. 让用户能带着说明批准/拒绝
4. 帮助系统形成新的持久化授权

这也是为什么审批层不是只有 yes/no。

---

## 八、一个完整实际例子

下面用一个真实感较强的例子，把整条链串起来。

### 场景

用户让系统修复前端测试，并且模型决定执行下面这条命令：

```Bash
npm test -- --runInBand
```

假设当前环境如下：

- 当前模式：`default`
- 工作目录：项目根目录
- 用户之前配置过 allow 规则：`FileRead(src/**)`
- 没有配置 `Bash(npm test:*)`
- 没有 deny Bash

### 第 1 步：工具调用产生

模型在推理后生成一个 Bash tool call，请求执行：

```Bash
npm test -- --runInBand
```

### 第 2 步：进入主权限判断

`useCanUseTool.tsx` 调用 `hasPermissionsToUseTool()`。

### 第 3 步：检查整工具 deny 规则

系统看：

- 有没有 `Bash` 整体 deny

结果：没有。

继续。

### 第 4 步：检查整工具 ask 规则

系统看：

- 有没有 `Bash` 整体 ask

结果：假设没有。

继续。

### 第 5 步：进入 Bash 自检

`BashTool.checkPermissions()` 开始分析这条命令。

它会做的事情包括：

- 拆分命令
- 识别主命令是 `npm`
- 看有没有特殊 wrapper
- 检查是否涉及危险路径
- 检查命令模式是否命中规则

这条命令看起来不像明显危险命令，也没有删除敏感目录，不涉及文件写入目标路径问题。

于是 Bash 自检大概率不会直接 deny。

但因为没有 allow 规则，它通常也不会直接 allow。

所以结果很可能是：

- `ask`

### 第 6 步：模式层介入

当前模式是 `default`。

`default` 不会自动把这类 ask 转成 allow。

所以这次请求仍然是 `ask`。

### 第 7 步：进入审批 UI

`useCanUseTool.tsx` 发现结果是 `ask`，于是把它塞进审批队列。  
`PermissionRequest.tsx` 根据工具类型选择 Bash 专属审批组件。

用户这时会看到一条 Bash 权限确认对话框，内容大致是：

- 工具：Bash
- 描述：执行 `npm test -- --runInBand`
- 问题：是否允许继续

### 第 8 步：用户点击“允许并记住”

如果用户选择“允许，并且以后别再问类似命令”，系统会生成一条权限更新。

例如可能变成：

- `Bash(npm test:*)`

然后：

1. 写入对应 settings
2. 更新内存中的 `ToolPermissionContext`

### 第 9 步：下一次再执行类似命令

之后模型再调用：

```Bash
npm test -- src/foo.test.ts
```

这次在更前面的规则层就可能直接命中 allow 规则：

- `Bash(npm test:*)`

于是无需再弹窗。

### 这个例子体现了什么

这个例子体现了权限系统的几个核心目标：

1. 第一次谨慎处理，避免自动误放
2. 用户一旦表达稳定意图，就减少重复打扰
3. 规则不是一开始就放太宽，而是在真实使用中逐步收敛

这就是“安全”和“效率”之间很典型的工程折中。

---

## 九、一个从头到尾的故事版例子

如果你想把这套系统讲给完全不懂代码的人听，最适合用“故事版”理解。

### 场景

用户说：

帮我修一下测试，先跑一遍单元测试看看。

模型决定调用 Bash 执行：

```Bash
npm test -- --runInBand
```

### 系统脑子里发生了什么

#### 第一反应：这是不是明显不允许的

系统先看：

- Bash 有没有被整体 deny
- 当前会话有没有明确禁止这类工具

如果这里就被禁了，后面根本不会继续。

#### 第二反应：这类工具是不是规定必须问

如果项目规定：

Bash 总要先确认

那系统也会在这一步停下来，直接准备弹窗。

#### 第三反应：让 Bash 自己说说看

系统把命令交给 Bash 的权限检查逻辑。

Bash 会分析：

- 这是一个什么命令
- 是不是复合命令
- 有没有奇怪 wrapper
- 有没有危险路径
- 有没有命中已有 Bash allow 规则

`npm test -- --runInBand` 通常不像明显危险命令，但它也不一定在默认规则里被认作“完全安全”。

所以很可能得到：

- 不是 deny
- 但也不能直接 allow
- 最终结果偏向 ask

#### 第四反应：看当前模式

如果当前是 `default`，那就维持 ask。  
如果当前是 `auto`，系统可能会再让 classifier 看一眼。  
如果当前是 `bypassPermissions`，且没命中硬性安全边界，就可能直接放行。

#### 第五反应：如果还是 ask，就让用户决定

用户看到一个审批框：

- 即将执行什么命令
- 为什么需要确认
- 是否允许

如果用户选择：

允许，并记住类似命令

系统就会把这次意图沉淀成规则，例如：

```LaTeX
Bash(npm test:*)
```

下一次再跑类似命令，就不需要重复打扰了。

### 这个例子体现了整套系统的价值

它不是一上来就信任模型，也不是一上来就拒绝模型，而是：

- 第一次先谨慎
- 用户确认后逐步放宽
- 放宽也是精确放宽，不是把整个 Bash 全开放

这就是一套成熟权限系统最关键的体验特点：

安全不是靠一直打断用户实现的，效率也不是靠直接跳过所有防线实现的。

---

## 十、再举一个危险例子：为什么多层防御有必要

现在看一个更危险的命令：

```Bash
python -c "import shutil; shutil.rmtree('/')"
```

假设当前模式是 `auto`，并且用户曾经误配置过：

```LaTeX
Bash(python:*)
```

如果系统设计不好，会发生什么？

- 因为命中了 allow 规则
- 命令在分类器介入前就直接放行
- auto 模式的智能审核完全失效

但这套系统会在进入 auto 模式时识别：

- `Bash(python:*)` 属于危险 allow 规则

然后把它剥掉。

这样这条命令就不会在前面被粗暴放行，而会继续走：

- Bash 自检
- 安全检查
- classifier
- 必要时人工确认

这就是纵深防御最有价值的地方：

它不仅防“危险请求”，还防“错误授权把后续安全机制架空”。

---

## 十一、为什么这套设计对小白也重要

即使你不打算修改权限系统代码，理解它也很重要，因为它会直接影响你使用这个 CLI 的体验。

你会明白：

- 为什么有时它直接执行，有时要问你
- 为什么同一个 Bash 工具，有些命令秒过，有些命令必须确认
- 为什么切换模式后行为明显不同
- 为什么 auto 模式并不是“完全自动放行”
- 为什么某些敏感操作即使在高权限模式下仍然会被拦

理解这些后，你在使用时会更容易判断：

- 该不该加 allow 规则
- 什么时候该用 `acceptEdits`
- 什么时候该坚持 `default`
- 什么时候应该先进 `plan`
- 什么时候绝不该开 `bypassPermissions`

---

## 十二、总结

如果把这套权限系统压缩成一句工程化总结：

它把“用户意图、静态规则、工具自检、硬性安全边界、智能上下文判断、人工最终审核”串成了一条逐层收敛的决策链。

它的优点不是“某一层特别神”，而是：

- 每层解决不同问题
- 每层都可以单独拦截风险
- 某一层失效时，后面还有机会补救
- 能在安全和效率之间逐步收敛，而不是二选一

如果把它用最直白的话讲给小白听：

这不是一个“要么全信 AI，要么完全不让 AI 动手”的系统，而是一套让 AI 在可控边界里逐步获得行动权限的系统。

---

## 十三、推荐阅读顺序

如果你想继续顺着源码深入，建议按这个顺序看：

1. `src/types/permissions.ts`
2. `src/Tool.ts`
3. `src/utils/permissions/permissionSetup.ts`
4. `src/utils/permissions/permissions.ts`
5. `src/tools/BashTool/bashPermissions.ts`
6. `src/hooks/useCanUseTool.tsx`
7. `src/components/permissions/PermissionRequest.tsx`

这样最容易把“配置、判断、交互”三部分串起来。

## 十四、如何使用这份文档

如果你是第一次学习，建议顺序是：

1. 先看“整体架构图”
2. 再看“主决策链”
3. 再看“模式对照表”
4. 最后看两个实际例子

如果你是准备讲给别人听，建议顺序是：

1. 先讲“为什么需要纵深防御”
2. 再讲“一次 tool call 的时序图”
3. 再讲“Bash 为什么最复杂”
4. 最后用故事版例子收尾


---
