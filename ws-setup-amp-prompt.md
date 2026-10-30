# 任务：搭建多 repo agent workspace（ws）+ 专做逆向规格的 spec-miner agent

## 目标
在 `~/ws` 建立 Amp 与 Grok Build 共用的工作区：跨 repo 上下文、跨会话记忆、一个只做逆向规格的 spec-miner agent。

## 前置
1. 先 `pwd` 与 `ls`，确认真实路径、确认真实子目录名。**不要假设**目录名。
2. 已存在的文件：先读，再按缺失项补齐，**不要覆盖**我已有的内容。有冲突就列出来问我。

## 1. 目录结构
`ws/` 保持**普通目录**：不要 `git init`，不要建 `.gitignore`，不要把 `repos/` 写进任何忽略规则（否则 agent 可能搜不到代码）。
需要的结构：
```
ws/
  AGENTS.md
  LEARNINGS.md
  specs/
  archive/
  .agents/skills/spec-miner/
  .amp/plugins/            # 若走插件方案
  .grok/agents/            # Grok 侧 agent 定义
  repos/                   # 已有，不要动里面的任何 repo
```

## 2. 写 ws/AGENTS.md（这是唯一两个工具都必定加载的规则文件）
必须包含这几节：

**记忆协议**
- 会话开始：读 `LEARNINGS.md` 全文；任务相关时读 `specs/` 下对应文件。
- 学到可复用结论（踩坑、有效命令、架构约束、被否决的方案）→ 立刻追加一行：
  `- YYYY-MM-DD [repo名] 结论 | 证据(file:line 或命令输出) | 适用范围`
- 只记下次还会用到的；一次性任务进度不记。
- `LEARNINGS.md` 超过 200 行时压缩：合并同类项、删除已废弃项，压缩行为本身也留一条记录。旧内容移到 `archive/`。

**会话约定**
- cwd 是 ws 而不是某个 repo：任何 git 命令必须先 `cd repos/<repo>` 再执行。
- 跨 repo 的任务先说清楚动哪个 repo，不明确就问。

**各 repo 分节**
先用 `ls repos/` 得到真实列表，为每个 repo 建一个 `## repos/<名字>` 小节，写它的构建/测试命令；拿不到就先留 `TODO`，别编。

**agent 分工**
- 逆向规格（从代码出 spec）→ 使用下面第 4 步暴露的 `spec_miner` 工具（或降级方案，见第 4 步）；主会话不自己现编规格。
- 写码/改码 → 正常主会话或 Task subagent。
- 规格产出统一落 `specs/<repo>_reverse_spec.md`，我 review 这份 md，之后代码改动要对着它来。

## 3. 装 spec-miner skill（两个工具共用）
从 `https://raw.githubusercontent.com/jeffallan/claude-skills/main/skills/spec-miner/SKILL.md` 取正文，连同它的 4 个参考文件（`references/ears-format.md`、`specification-template.md`、`analysis-process.md`、`analysis-checklist.md`），落到 `ws/.agents/skills/spec-miner/`。
网络失败就停下来报告，**不要自己发明一份规格方法论**。

## 4. Amp 侧：把 spec-miner 做成真正的独立 agent
首选方案：写插件 `ws/.amp/plugins/spec-miner.ts`，用 `amp.createAgent(...)` + `amp.registerTool(...)` 暴露一个工具 `spec_miner`：
- `name: 'spec-miner'`
- `instructions`：在插件加载时用 fs 读 `.agents/skills/spec-miner/SKILL.md` 的正文作为指令（保持单一真源）；读不到才内联
- `reasoningEffort: 'high'`
- 工具集允许读文件与搜索，**禁止改任何源码**；只允许写 `specs/` 下的文件
- `execute` 里调用 `agent.run(request, { parentThreadID: ctx.thread.id, timeoutMs: 30*60*1000 })`
- 工具描述写清楚：输入是"要逆向的 repo 路径 + 关注范围（整仓/某模块/某切面）+ 深度"

若插件因类型解析、沙箱限制等原因跑不通：**降级方案**（明确告诉我你降级了），改在 `AGENTS.md` 里写死流程——逆向规格时派一个 Task subagent，把 `.agents/skills/spec-miner/SKILL.md` 全文作为它的指令，只读、产出写 `specs/`。

## 5. Grok 侧
- `~/.grok/config.toml` 加：
  ```toml
  [skills]
  paths = ["<ws 的绝对路径>/.agents/skills"]
  ```
  （若已有 `[skills]` 段，追加到 paths 数组，别覆盖）
- 在 `ws/.grok/agents/` 下建一个 spec-miner 的 subagent 定义：只读、不写代码、产出写 `specs/`。
- `.grok/agents/` 的确切文件格式你若不确定，**先查 `https://docs.x.ai/build/features/subagents` 并用 `grok inspect` 核对**，不要凭空猜格式；这条拿不准就留 TODO 并告诉我。

## 6. 验收（必须实际执行并回报真实输出，不要复述意图）
- `amp skills list --json`：确认 spec-miner 在列表里，且来源是 `ws/.agents/skills`
- `grok inspect`：确认它发现了 `ws/AGENTS.md`、`.agents/skills`、`.grok/agents`
- 插件是否成功加载（或是否降级）
- 回报一份清单：实际创建/修改了哪些文件（绝对路径）、两个命令的真实输出摘要、遗留 TODO

## 约束
- 不动 `repos/` 里任何代码，不 `git init`，不改已有 repo 的分支/配置
- 不装全局包、不新增 MCP、不引入除 `@ampcode/plugin` 之外的依赖
- 需要我决策的点（目录名不一致、repo 构建命令、插件跑不通要不要降级）——停下来问，不要自行决定
