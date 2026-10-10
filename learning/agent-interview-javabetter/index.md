# Agent 面试目录来源

- 来源：<https://github.com/itwanger/toBeBetterJavaer>
- 站点目录：<https://javabetter.cn/ai/video/>
- 说明：以下条目对应 `learning/agent-interview-javabetter/` 中的同名 Markdown 原文，禁止合并或摘要替代。

## START / 题库入口

- [readme](readme.md)


## 01 / Agent 基础

- [用户中途取消，Agent 任务怎么停下来？](agent-task-cancellation.md)

- [Agent 调用工具失败或者超时了怎么办？](agent-tool-failure-timeout.md)

- [Agent 的意图路由怎么设计？](agent-intent-routing.md)

- [多 Agent 之间怎么传递上下文？](multi-agent-context-passing.md)

- [Plan 模式已经死了吗？](is-plan-mode-dead.md)

- [Pi Agent 里有哪些反复出现的设计模式？](pi-agent-design-patterns.md)

- [构建有效 Agent 有哪三个核心原则？](building-effective-agents-principles.md)

- [订机票客服该用工作流还是自主 Agent？](workflow-vs-agent-flight-booking.md)

- [模型越来越强，Harness 会不会被模型「吃掉」？](will-model-eat-harness.md)

- [Agent 的沙箱（Sandbox）你了解吗？](what-is-agent-sandbox.md)

- [了解 Agent 的 Checkpoint 机制吗？](what-is-agent-checkpoint.md)

- [如果只能给 Agent 增加一项能力，更强的 LLM，更长的 Context，还是更多的 Tool？](llm-context-or-tool.md)

- [Harness 的五要素是什么？](harness-five-elements.md)

- [Multi-Agent 协作时意见冲突怎么办？](multi-agent-conflict-resolution.md)

- [Agent 怎么做经验积累和自我学习？](how-agent-accumulates-experience.md)

- [为什么选择手搓 Agent 而不用 LangChain 这种框架？](why-handcraft-agent-not-framework.md)

- [Agent 的反思机制是什么？Self-Refine 和 Reflexion 有什么区别？](what-is-agent-reflection.md)

- [Agent 的 Plan 模式有哪些实现方式？](agent-plan-modes.md)

- [AI Agent 是如何工作的？完整执行过程拆解](how-agent-works.md)

- [一个 AI Agent 有哪些核心组件？](agent-core-components.md)

- [什么是 ReAct？它和 CoT 有什么区别？](what-is-react.md)

- [Workflow 和 Agent 有什么区别？面试时该怎么回答？](workflow-vs-agent.md)

- [Agent 和 ChatBot 有什么区别？如何让聊天机器人进化成 Agent？](agent-chatbot-difference.md)

- [什么是 AI Agent？与直接调用大模型 API 有什么区别？](what-is-agent.md)

- [Multi-Agent 协作是怎么实现的？](multi-agent-collaboration.md)

- [什么是 Plan-and-Execute？它和 ReAct 怎么选？](plan-and-execute.md)

- [ReAct 会死循环吗？如何设计可靠的终止机制？](react-death-loop.md)

- [Agent 怎么知道该调用哪个工具？Function Calling 原理拆解](agent-hnow-tool-call.md)


## 02 / 上下文与记忆

- [Agent 的长期记忆是怎么实现的？](agent-long-term-memory.md)

- [为什么聊着聊着 Agent 就变笨了？](why-agent-gets-dumber.md)

- [Agent 的短期记忆怎么实现？](agent-short-term-memory.md)

- [怎么让 Agent 拥有记忆？短期与长期记忆设计](how-to-give-agent-memory.md)

- [什么是上下文窗口？它不是聊天窗口](what-is-context-window.md)

- [为什么说 LLM 本身没有记忆？](why-llm-has-no-memory.md)

- [什么是提示词工程（Prompt Engineering）？](what-is-prompt-engineering.md)

- [什么是上下文工程（Context Engineering）？](what-is-context-engineering.md)

- [为什么 LLM 的上下文窗口不能无限大？](context-window-limit.md)

- [Agent 怎么避免上下文爆炸？四种常用方案](agent-context-explosion.md)


## 03 / Harness 与 Skills

- [什么是 Loop Engineering？](what-is-loop-engineering.md)

- [什么是 Harness Engineering？为什么 Agent 需要运行时？](what-is-harness-engineering.md)

- [Agent 挂了几十个 Skill，怎么保证命中率？](agent-skill-hit-rate.md)


## 04 / RAG知识库

- [Agent 的 RAG 遇到 PDF 怎么办？](agent-rag-pdf.md)


## 05 / LLM 基础与 API

- [DeepSeek 的 Prompt Caching 为什么能全自动生效，Claude 为什么要手动加 cache_control？](what-is-prompt-caching.md)

- [Prefix Caching（前缀缓存）是什么？和 KV Cache 是什么关系？](what-is-prefix-caching.md)

- [KV Cache 是什么？为什么只缓存 K 和 V，不缓存 Q？](what-is-kv-cache.md)

- [什么是线性注意力？GLM-5.3-Flash 为什么采用线性注意力加稀疏注意力的混合架构？](glm-flash-linear-sparse-attention.md)

- [Responses API 和 Chat Completions API 有什么区别？](responses-api-vs-chat-completions.md)

- [大模型 API 缓存命中和未命中为什么差价巨大？](api-cache-hit-miss.md)

- [MoE 是什么？DeepSeek 模型为什么采用混合专家架构？](what-is-moe.md)

- [到底什么是语义缓存？](what-is-semantic-caching.md)

- [SWE-bench、Terminal-Bench、GPQA Diamond 这些大模型评测集到底在测什么？](what-benchmarks-test.md)


## 06 / Claude Code与Codex

- [Codex 的长期记忆是怎么实现的？](codex-long-term-memory.md)

- [Claude Code 如何检索长期记忆？](claude-code-memory-retrieval.md)

- [CLAUDE.md 怎么写才真正有用？](how-to-write-claudemd.md)

- [Claude Code 的长期记忆是怎么实现的？](claude-code-long-term-memory.md)

- [Codex 的短期记忆是怎么实现的？](codex-short-term-memory.md)

- [Claude Code 的短期记忆是怎么实现的？](claude-code-short-term-memory.md)


## 07 / DeepSeek专题

- [多模态模型和纯文本模型有什么区别？以 DeepSeek V4-Flash-Vision-Exp 为例](deepseek-multimodal.md)

- [CSA、HCA 与 KV Cache：DeepSeek V4 缓存为什么更便宜？](v4-csa-hca-kv-cache.md)

- [DeepSeek V4 为什么用 CSA 和 HCA 替换 MLA？](v4-why-replace-mla.md)

- [Cordis 是什么？DeepSeek Harness 的插件系统如何理解？](what-is-cordis.md)

- [DeepSeek V4 Flash 正式版的激活参数比 V4 Pro 小得多，Agent 能力为什么超过了 Pro 预览版？](why-post-training-beats-params.md)


## HOT / 模型与行业热点

- [CPU、GPU、NPU 到底有什么区别？](cpu-gpu-npu-difference.md)

- [Kolibri 如何蒸馏 GLM-5.3 和 Qwen3.8？](aleph-alpha-kolibri.md)

- [Gemini 4 Argon 发布，100 万 Token 输出意味着什么？](gemini-4-argon.md)

- [MiMo-V3 的 HySparse2 和混合滑动窗口注意力有什么区别？](mimo-v3-hysparse2.md)

- [推荐 5 个复刻 Jev 的开源项目](open-source-jev-replicas.md)

- [不生成文本的 Jev，为什么让 14 万开发者排队？](what-is-jev.md)

- [GPT Images 2.5 一手实测，OpenAI 越来越有水平啦](chatgpt-images-2-5.md)

- [Claude Haiku 5.5 怎么在成本和智能之间做选择？](claude-haiku-5-5.md)

- [GPT-6 Astra 全量上线，为什么 OpenAI 敢说 AGI 时代来了？](gpt-6-astra.md)

- [Claude、ChatGPT、Grok 全崩，你的中转为什么还能用？](why-relay-still-works.md)


## 08 / Prompt 工程

- [9个让Agent提高输出质量的提示词技巧](nine-prompting-techniques-for-better-llm.md)


## 11 / 模型训练与微调

- [大模型预训练完了，为什么还要做强化学习？](what-is-reinforcement-learning.md)


## 12 / Agent 产品与框架演进

- [什么是 Manus？说说你对它的了解](what-is-manus.md)
