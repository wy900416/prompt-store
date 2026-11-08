# prompt-store

Repository for storing prompts and prompt templates.

## 提示词 / Prompts

- [super-agent-amp-prompt.md](super-agent-amp-prompt.md) — 用 Amp 实现 super-agent 最小版（Grok headless 多任务 + Web GUI）的执行提示词
- [specgate-amp-prompt.md](specgate-amp-prompt.md) — 用编码 agent 实现 SpecGate 的分阶段执行提示词（先硬化事实，再 DB/lint/闭环/流水线/GUI，附 15 项真跑验收）
- [specgate-spec-authoring-prompt.md](specgate-spec-authoring-prompt.md) — 用 agent 起草功能 spec 并导入 SpecGate 的提示词（EARS 句式、三值权威判定、自检循环、导入与冲突处理）
- [ws-setup-amp-prompt.md](ws-setup-amp-prompt.md) — 搭建多 repo agent workspace（ws）+ 专做逆向规格的 spec-miner agent
- [second-brain-setup-prompt.md](second-brain-setup-prompt.md) — second brain 初始化提示词

## 规格 / Specs

- [super-agent-impl-spec.md](super-agent-impl-spec.md) — Super-agent 最小版实现规格：任务隔离、clone 策略、状态机、Watchdog、权限边界与验收清单
- [specgate-impl-spec.md](specgate-impl-spec.md) — SpecGate 实现规格：规格驱动的评审闸门（rule 级批准、三值权威分类、EARS 受控语言、lint 闸门、影响分析与增量传播、Web 评审台）

## 用例 / Use Cases

- [copilot-spec-driven-agent-workflow.md](copilot-spec-driven-agent-workflow.md) — 规格驱动、角色隔离的 Copilot Agent 工作流（任务编排 + 持久记忆），含给 Copilot 团队的问题

## 文章存档 / Articles

- [super-agent-design-exploration.md](super-agent-design-exploration.md) — 《Super-agent 设计探索：迈向更自主的 Agent 系统》（DeepSeek Harness 团队，2026-10-31）｜[原文](https://mp.weixin.qq.com/s/zoS7GlReWPhfCjENGxXCdg)
