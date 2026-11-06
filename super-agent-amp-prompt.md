# 任务：实现 super-agent 最小版（Web GUI + Grok headless 多任务工作台）

## 目标

在 `~/sa` 实现一个可用的多任务工作台：我在浏览器里下指令，每条指令变成一个隔离、可并行、可恢复的后台任务，跑完我验收，再由控制面 push。

规格以 **`super-agent-impl-spec.md`（同目录）为唯一真源**。先完整读它，再动手。本文档只规定执行顺序、验收方式与禁令；两者冲突时以规格为准，并告诉我冲突点。

## 前置（不满足就停下来问，不要猜）

1. `pwd`、`ls`，确认真实路径与已有内容。若 `~/sa` 已存在，先读现有文件，**只补缺失项，不覆盖**，冲突列出来问我。
2. `grok --version`、`gh auth status`、`python3 --version` 全部记录真实输出。
3. `gh auth setup-git` 是否已配置（决定私有 repo 能否 clone）。未配置就报告，不要自己跑 `gh auth login`。
4. 确认 `~/sa` 之外不要创建任何文件；不要碰 `~/ws/repos/` 下任何 repo。

## 阶段 0（必须先做，做完停下来等我确认）

固化 Grok 的真实行为到 `~/sa/docs/events.md`，逐条执行并把**原样输出**贴进该文件：

```bash
cd ~/sa
grok --no-auto-update -p "say ok" --output-format streaming-json | head -20
grok --no-auto-update -p "say ok" --output-format json
```

从输出里确定并写进文档：

- 每一类事件的 `type` 取值，以及 text / thought / tool 调用的字段路径。
- `end` 事件的完整结构（`sessionId`、`stopReason`、结果字段各叫什么）。
- `json` 格式下 `sessionId` 的取法（`jq -r '.sessionId'` 是否成立）。

再验证三条，同样贴原始输出：

```bash
# 续接是否保留上下文
SID=$(grok --no-auto-update -p "remember the number 41" --output-format json | jq -r '.sessionId')
grok --no-auto-update -r "$SID" -p "what number did I ask you to remember?" --output-format json

# deny 语法是否生效（不生效不算失败，规范里有 pre-push hook 兜底）
grok --no-auto-update --deny 'Bash(git push*)' -p "run: git push --dry-run" --output-format json

# sandbox 的 profile 名
grok --no-auto-update -p "say ok" --sandbox read-only --output-format json
```

⚠️ 这一步的任何结论**不要凭文档或记忆填写**，只写真跑出来的东西。跑不通就写「未验证 + 报错原文」，然后继续用 hook 兜底方案。

## 阶段 1：数据层 + prepare

按规格 §3、§4、§6 实现 `db.py`、`prepare.py`。

要求：

- `prepare.py` 可独立 CLI 调用：`python prepare.py --task-id <id> --prompt "..." --repo <org/repo>[,<org/repo>]`
- 幂等：同 task_id 重复调用不重复 clone。
- 建 `tasks/<id>/`、写 `AGENTS.md`、clone 每个 repo（`--depth 1`）、记录 base_sha、任务目录 `git init` + 初始 commit、**给每个子 repo 和任务目录装 pre-push hook 并 chmod +x**。
- 给每个子 repo 设 local `user.name`/`user.email`（防 grok 的 commit 失败）。
- repo 目录名用 `<org>-<repo>`。
- 失败时打印哪个 repo、哪一步、stderr 尾部 20 行，退出码非 0。

验收（贴真实输出）：

```bash
python prepare.py --task-id t1 --prompt "add a README section" --repo <org/repo>
ls -la ~/sa/tasks/t1
git -C ~/sa/tasks/t1/<org>-<repo> rev-parse HEAD
cd ~/sa/tasks/t1/<org>-<repo> && git push --dry-run; echo "exit=$?"   # 期望被 hook 拒绝
cd ~/sa/tasks/t1 && git log --oneline
```

`AGENTS.md` 必须包含（照抄，不要改写语义）：

```
# 任务约束
- 你只能修改本目录内的文件。
- 允许 git commit，禁止 git push / git remote add / gh pr create。
- commit message 用英文，一行祈使句，不超过 72 字符。
- 完成任务后，在最后一条消息里给出：改了哪些文件、每个文件改了什么、还有哪些没做、需要人决定什么。
- 不要执行 rm -rf、不要动 .git/hooks、不要修改本文件。
```

## 阶段 2：worker

按规格 §7 实现 `worker.py`：`run_task(item_id)` 协程。

要求：

- `cwd` 用 `create_subprocess_exec(..., cwd=item.dir)`，命令行参数按 §7.1 组装，**不要用 `--cwd` 也不要 `-w`**。
- 逐行读 stdout；`json.loads` 失败则丢弃该行，不得让整任务崩掉。
- 每行落 `event` 表 + 更新 `last_event_at` + 回调（阶段 3 接 WS）。
- `end` 事件到达时保存 `session_id`。
- 进程 exit 后按 exit_code 与是否收到 end 决定 done / failed，并把最后一条助手消息写入 `report`，附 `git -C dir status --porcelain` 摘要。
- 中止函数：SIGTERM → 等 5s → SIGKILL。

验收（先用 CLI 触发，不接前端）：

```bash
python -c "import asyncio,worker; asyncio.run(worker.run_task('t1'))"
sqlite3 ~/sa/data/sa.db "select state, session_id, exit_code, length(report) from work_item where id='t1'"
sqlite3 ~/sa/data/sa.db "select kind, count(*) from event where item_id='t1' group by kind"
```

再验证续接：

```bash
python worker.py --task-id t1 --prompt "also update the CHANGELOG"
# 检查 event 表新增的会话里，第一轮上下文是否还在（让它复述上一轮改了什么）
```

## 阶段 3：API + WebSocket

按规格 §8 的协议实现 `app.py`（FastAPI + uvicorn，绑 127.0.0.1:8788）。

端点最小集：

```
GET  /api/items                 列表（含 state、耗时、repo）
GET  /api/items/{id}            详情 + report
GET  /api/items/{id}/events?after=<event_id>   历史回放（分页）
POST /api/items                 派活 {prompt, repos:[...], base_branch}
POST /api/items/{id}/message    追加指令（带 -r 续跑）
POST /api/items/{id}/cancel     中止
POST /api/items/{id}/push       控制面推送（人点过才允许）
GET  /api/repos                 仓库清单（缓存，来自 gh repo list 或本地扫描）
WS   /ws                        订阅全部任务的增量
```

要求：

- 服务启动时先跑一次 §9 的对账（纠正遗留 running/preparing）。
- 任何状态变更都先写库再广播，不允许只改内存。
- 并发上限 3：`POST /api/items` 在满槽时仍返回 201，状态为 queued，不许阻塞请求。

## 阶段 4：前端

`static/index.html`，单文件，原生 JS + WebSocket，**不引入任何构建步骤、CDN 或框架**。布局按规格 §8 四个区域。

最低要求：

- text 与 thought 分开显示（thought 默认折叠），tool 调用折叠成一行可展开。
- 任务卡片有状态色；running 显示已跑时长。
- 详情底部有「中止」；任务 done/failed 后出现「导出 patch」「推送分支」，推送前弹一次 `diff --stat` 确认。
- 刷新页面后靠 `GET /api/items` 全量重建，不依赖前端内存。

验收：浏览器实际操作，截图或贴 DOM 文本，证明：派活、看流、中止、刷新恢复四项可用。

## 阶段 5：watchdog

按规格 §9 实现 `watchdog.py`，60s 周期，三条规则。用 `pgrep -f "tasks/<id>"` 或直接记录并检查 pid 是否存活做进程对账。

验收：

```bash
SQL 手动把某任务 last_event_at 改到 31 分钟前
# 等 60s，确认状态变 failed 且 report 有「30 分钟无事件」
# kill -9 一个 worker 进程，等 60s，确认状态变 failed 且 session_id 保留
# 重启 app.py，确认遗留 running 状态被纠正
```

## 验收（全部按规格 §13 的 9 条走，逐条贴真实输出）

不接受「已实现」「应该可以」。每条给出：执行的命令、真实输出、结论。

第 2 条（跨 repo 单会话）和第 5 条（服务重启后 `-r` 续接）最容易出问题，必须真实跑通；跑不通就停在报告里说明卡在哪，不要改需求绕过去。

## 约束

- 只新增两个依赖：`fastapi`、`uvicorn`，装在 `~/sa/.venv`，不装全局包。
- **不要用** `grok agent stdio`（ACP）、不要用 `grok -w`（worktree）、不要引入消息队列/Redis/Docker。
- worker 环境里不得出现任何 git token；不得把 token 写进 clone URL、task.json、AGENTS.md、prompt。
- 不得删除任务目录，不得自动 merge、自动发布、自动改 base 分支。
- 不做规格里的「非目标」任何一项，即使你觉得有用。
- 需要我决策的点（真实路径不一致、repo 清单有歧义、sandbox 不可用、事件字段与规格不符）——停下来问，不要自行决定。

## 回报格式

1. 阶段 0 的实测结论（事件字段、续接是否成功、deny 与 sandbox 是否生效）。
2. 每个阶段：改了哪些文件（绝对路径）、验收命令与真实输出摘要。
3. 规格 §13 九条验收逐条结论，通过/未通过。
4. 遗留 TODO 与你建议我决策的点。

如果规格 §11 的任何实测结果与我写的不同，先改 `super-agent-impl-spec.md` 再改代码，并在回报里单列一节「规格修订」。
