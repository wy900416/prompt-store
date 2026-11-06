# Super-agent 实现规格：Grok Build + Web GUI 多任务工作台

> 状态：草案，待人 review。目标读者：实现者（Amp / Grok Build）、reviewer（人）。
> 相关：`super-agent-design-exploration.md`（DeepSeek Harness 原设计文章存档）、`ws-setup-amp-prompt.md`（ws 工作区约定）。

## 1. 目标与非目标

### 目标

一个 web 界面，人在这里下指令；系统把每条指令变成一个**隔离、可并行、可恢复、可验收**的后台任务。

必须成立的四件事：

1. 任务之间不互相踩文件（同一 repo 的多个任务可以同时跑）。
2. 一个任务可以横跨多个 repo；也可以完全不涉及 repo。
3. 关掉浏览器、重启服务，任务历史与状态都不丢。
4. worker 不能 push，push 与开 PR 由控制面在人验收后执行。

### 非目标（第一版明确不做）

| 不做 | 原因 |
|---|---|
| 接 Teams / 飞书 / CI / 邮件事件源 | 当前唯一事件源是人在 GUI 里发消息，Router 无路由可做 |
| LLM 推断该 clone 哪些 repo | GUI 多选，一次点击比一次往返便宜且不会错 |
| ACP（`grok agent stdio`）长连接与实时插话 | 恢复复杂；见 §12 决策记录 |
| 前端点按权限提示 | 用预授权白名单 + pre-push hook 替代 |
| 自动 merge / 自动发布 | 人负责验收是这套设计的前提 |
| 跨机器调度、多租户、鉴权 | 单机自用，绑定 127.0.0.1 |

## 2. 形态

- 单机、单进程 asyncio、SQLite 持久化、FastAPI + 一个单页前端。
- 每个任务 = 一个目录 + 一个 `grok` headless 子进程 + 一个 session。
- 并发上限 3（可配），超出进队列。上限由内存决定，不是模型。
- 无守护进程、无消息队列、无 Redis。SQLite 就是账本。

## 3. 目录布局

```
~/sa/                        # 控制面代码（本规格的实现物）
  app.py                     # FastAPI：REST + WebSocket
  worker.py                  # 起进程、解析事件、落库、推送
  prepare.py                 # 建任务目录、clone repo、装 hook
  db.py                      # schema 与查询
  watchdog.py                # 超时与进程对账
  static/index.html          # 单页前端，无构建步骤
  data/sa.db
  tasks/<task_id>/           # 任务目录（worker 的 cwd）
    AGENTS.md                # 任务约束，grok 每次启动自动读
    task.json                # 元数据快照：prompt、repo 列表、base sha
    .git/                    # 任务目录自身的 repo（含产出物）
    repo-a/                  # git clone --depth 1
    repo-b/
```

关键约束：**worker 的 cwd 永远是 `tasks/<task_id>/`，不是任何 repo 根目录。** Grok 在非 git 目录可正常启动（已实测），且这样它在一个会话里能同时看到 `repo-a/` 和 `repo-b/`，跨 repo 的改动才是一个任务而不是两个。

## 4. 数据模型

```sql
CREATE TABLE work_item (
  id            TEXT PRIMARY KEY,        -- uuid4 hex
  title         TEXT NOT NULL,           -- 从 prompt 首行截取，可改
  prompt        TEXT NOT NULL,           -- 本轮指令
  state         TEXT NOT NULL,           -- 见 §5
  dir           TEXT NOT NULL,           -- tasks/<id> 绝对路径
  session_id    TEXT,                    -- grok 返回的 sessionId，续跑用
  repos_json    TEXT,                    -- [{name,url,branch,base_sha}]
  pid           INTEGER,                 -- 当前 worker 进程 pid
  last_event_at TEXT,                    -- watchdog 的唯一输入
  created_at    TEXT NOT NULL,
  started_at    TEXT,
  finished_at   TEXT,
  exit_code     INTEGER,
  report        TEXT                     -- 最后一轮的结构化汇报
);

CREATE TABLE event (
  id        INTEGER PRIMARY KEY AUTOINCREMENT,
  item_id   TEXT NOT NULL,
  ts        TEXT NOT NULL,
  kind      TEXT NOT NULL,   -- text|thought|tool|status|error|permission
  payload   TEXT NOT NULL    -- 原始 JSON 行，不加工
);

CREATE INDEX idx_event_item ON event(item_id, id);
CREATE INDEX idx_item_state ON work_item(state);

CREATE TABLE repo_cache (          -- 可选：mirror 缓存，见 §6.3
  name TEXT PRIMARY KEY,
  url  TEXT NOT NULL,
  path TEXT NOT NULL,
  fetched_at TEXT
);
```

`event.payload` 存原始 JSON 行而不是解析后的字段，理由：`streaming-json` 的事件结构还没完全固化（§11），存原文可以后置解析，不用改历史数据。

## 5. 状态机

```
queued ──► preparing ──► running ──► done
  │            │            │  │
  │            │            │  └──► needs_input（保留，第一版不会进入）
  │            │            └──► failed / cancelled
  └────────────┴──────────────► cancelled
```

| 转换 | 触发条件 | 副作用 |
|---|---|---|
| queued → preparing | 并发槽有空位 | 建目录、写 AGENTS.md、clone |
| preparing → running | clone 全部成功 | 起 grok 进程，记 pid |
| preparing → failed | clone 返回非 0 | report 写失败 repo 与 stderr 尾部 |
| running → done | 进程 exit 0 且收到 `end` 事件 | 记 exit_code、抽 report、`git -C dir status` |
| running → failed | 进程 exit≠0 / 超时被杀 | 保留目录供排查 |
| running → cancelled | 人点中止 | SIGTERM，5s 后 SIGKILL |
| 任意 → failed | watchdog 判定无进展 | 见 §9 |

`needs_input` 在第一版用不上（headless 无法中途提问），但状态位保留，避免以后加 ACP 时改表。

## 6. 仓库准备

### 6.1 clone 规则

```bash
git clone --depth 1 --branch "<base_branch>" "<url>" "tasks/<id>/<name>"
git -C "tasks/<id>/<name>" rev-parse HEAD     # 存入 repos_json.base_sha
```

- 目录名用 `<org>-<repo>`，避免两个组织同名 repo 撞目录。
- 默认 `base_branch = main`，不存在则回退 `master`，仍失败则报错停下（不要猜）。
- 私有 repo 依赖 `gh auth setup-git` 已配置；prepare 前先检查，未配置则直接报错并给出命令，不要静默失败。

### 6.2 任务目录自身建 repo

无论是否有子 repo，任务目录都执行：

```bash
git -C tasks/<id> init -q
git -C tasks/<id> config user.name  "grok-worker"
git -C tasks/<id> config user.email "worker@local"
git -C tasks/<id> add -A && git -C tasks/<id> commit -qm "task init"
```

两条理由：

1. `git status --porcelain` 与 `git diff` 成为「这个任务产出了什么」的通用答案，无 repo 任务和有 repo 任务一致。
2. 无 repo 任务（纯调研、写文档、跑脚本）也有回滚点。

注意：clone 出来的子 repo 会继承全局 git 身份，若目标机器未配置 `user.name`，grok 的 `git commit` 会失败。prepare 时对每个子 repo 也设 local 身份。

### 6.3 大 repo 加速（可选，第二版）

```bash
git clone --mirror <url> /var/cache/sa/repos/<name>.git        # 建一次
git clone --depth 1 --reference /var/cache/sa/repos/<name>.git <url> tasks/<id>/<name>
```

先不做，但 `prepare.py` 里留 `clone_repo(task, repo)` 单点接缝，届时只改这一个函数。

### 6.4 pre-push 硬阻断

对每个子 repo（以及任务目录自身）安装：

```bash
cat > .git/hooks/pre-push <<'EOF'
#!/bin/sh
echo "pre-push blocked: worker must not push; control plane pushes after review" >&2
exit 1
EOF
chmod +x .git/hooks/pre-push
```

这是不可绕过的兜底，不依赖 `--deny` 的 glob 语法是否生效（§7）。

## 7. Worker 执行

### 7.1 命令模板

```bash
grok --no-auto-update \
     --output-format streaming-json \
     --max-turns 80 \
     --deny 'Bash(git push*)' \
     -p "<prompt>" \
     [ -r <session_id> ]          # 仅当本任务已有 session_id
```

- `cwd = tasks/<id>`，通过 `subprocess(cwd=...)` 设置，不用 `--cwd`，避免两处真源。
- 首次运行不带 `-r`；从 `end` 事件取 `sessionId` 写入 `work_item.session_id`。
- 同一任务的后续指令带 `-r <session_id>`，prompt 前缀固定为：`继续本任务。上一轮停在：<上次 report 摘要>。新指令：<用户消息>`。
- `--deny` 的 glob 语法需实测（§11）。若无效，不影响安全，因为 §6.4 的 hook 是硬阻断。

### 7.2 事件处理

逐行读 stdout，`json.loads` 失败即丢弃该行（grok 可能混入非 JSON 输出，例如更新提示）。

每行：

1. 原样写入 `event` 表。
2. 更新 `work_item.last_event_at`。
3. 通过 WebSocket 推给所有订阅该任务的客户端。
4. 若 `type == "end"`：尝试取 `sessionId`、`stopReason`、`result`。

> `streaming-json` 的确切字段名以实测为准。实现前先跑一次并把结果固化到 `~/sa/docs/events.md`，之后所有解析以此为准，**不要在代码里散落猜测的字段名**。

### 7.3 中止

`SIGTERM` → 等待 5s → `SIGKILL` → 状态置 `cancelled`，report 写「人工中止，最后 20 条事件已保留」。不要删除任务目录。

## 8. Web GUI

单页，四个区域，不做更多：

1. **输入区**：textarea + repo 多选（来自缓存的仓库清单）+ 可选 base branch + 「派活」按钮。
2. **任务列表**：按状态分组（运行中 / 待处理 / 完成 / 失败），显示标题、耗时、所属 repo。
3. **详情区**：选中任务后流式显示事件（text 与 thought 分区显示，tool 折叠）；顶部有「中止」。
4. **验收区**：任务完成后显示 `git status` 摘要、`git diff --stat`、最后一轮 report；两个按钮：**导出 patch**、**推送分支**（推送由控制面执行，见 §10）。

WebSocket 协议最小化：

```json
{"type":"event","item_id":"...","kind":"text","payload":{...}}
{"type":"state","item_id":"...","state":"running","report":null}
```

前端不做本地状态真相，每次连接先 `GET /api/items` 全量拉一次，之后只接受增量。

## 9. Watchdog

每 60s 扫一次。三条规则，都只用 `last_event_at` 与 pid，不猜模型行为：

| 情况 | 判定 | 动作 |
|---|---|---|
| state=running 且 `now - last_event_at > 15min` | 疑似卡住 | 记录 `stalled` 事件，UI 标黄，不杀 |
| state=running 且 `now - last_event_at > 30min` | 判定无进展 | SIGKILL，state=failed，report 写「30 分钟无事件」 |
| state=running 或 preparing 且 pid 不存在 / 非本进程子进程 | 进程已消失（含服务重启） | state=failed，report 写「进程丢失」；保留 session_id，UI 提供「继续」按钮用 `-r` 重跑 |

服务启动时先跑一次全量对账，把上一轮遗留的 running/preparing 状态纠正掉。**不要相信数据库里的 running 就等于真的在跑。**

为什么不做「模型自我纠偏」：Grok 有 `--max-turns` 兜底，外层只需要回答「它还活着吗、还在产出吗」这两个问题。DeepSeek 那套 Watchdog 会话是给长期在线服务用的，这个形态用不上。

## 10. 权限与凭据边界

| 主体 | 能做什么 | 不能做什么 |
|---|---|---|
| worker（grok 进程） | 读写任务目录、执行命令、`git commit` | `git push`（hook 硬阻断 + deny）、访问控制面代码目录、读取 git token |
| 控制面 | clone、push、开 PR、改数据库 | 不代替 worker 写业务代码 |

- git 凭据只存在于控制面进程的 `gh` 会话中。**不得**写入 clone URL、`repos_json`、AGENTS.md、prompt。
- 推送实现：`git -C tasks/<id>/<repo> push origin HEAD:refs/heads/task/<task_id>`，成功后才可开 PR。
- 推送前必须人已点击「推送分支」，且 UI 显示过一次 diff 摘要。
- 建议给 worker 加 `--sandbox` 一次实测（§11）；跑通则默认开启，作为第三层防线。

## 11. 待实测固化（实现前必须先做）

按顺序执行，把真实输出记进 `docs/events.md`：

1. `grok -p "say ok" --output-format streaming-json | head -20` → 固化事件类型与字段名。
2. `grok -p "..." -r <上一步的 sessionId> --output-format json` → 确认续接成功、上下文保留。
3. `grok -p "try git push" --deny 'Bash(git push*)'` → 确认 deny 语法是否生效（不生效则只依赖 hook）。
4. `grok -p "..." --sandbox <profile> --output-format json` → 确认 sandbox 配置名与是否影响 clone 目录读取。
5. 在**非 git 目录**里跑一次带 `-r` 的续接 → 确认 session 按 cwd 索引的行为符合预期（服务重启后 cwd 不变，应当能续）。

任何一条与本文档描述不符，改文档再改代码，不要反过来。

## 12. 决策记录

**为什么不用 `git worktree`（`grok -w`）**
worktree 一任务只能隔离一个 repo，无法承载跨 repo 任务；且共享 `.git`，同一 repo 的并行任务仍需自己处理分支冲突。clone 让每个任务拿到完整独立工作副本，同一 repo 可无限并行，代价是磁盘与 clone 时间。

**为什么第一版不用 ACP**
ACP 的优势是实时插话与权限交互，代价是长连接生命周期（服务重启、进程被 OOM、机器休眠）都要自己处理重连与 `session/load`（Grok 是否实现待确认）。headless 的恢复路径简单到只有一个 uuid：进程死掉就是死掉，`-r` 续即可，控制面完全无状态。「随时给指令」由多任务并行满足，不依赖单任务插话。

**为什么 Router 退化成一个函数**
不接事件源时，Router 唯一剩余价值是「一句话拆多任务」与「跨任务找上下文」，两个都还没出现。写一个不许干活的 LLM Router 是为对称而对称。

**为什么任务目录也 `git init`**
让「产出物」有统一的答案（`git status` / `git diff`），且无 repo 任务同样可回滚。

## 13. 验收清单

每条都必须实际执行并留存真实输出，不接受「应该可以」。

1. 单 repo 任务：派活 → 看到 clone 日志 → 流式文本 → done，report 含改动摘要。
2. 跨 repo 任务：选两个 repo，worker 在一个会话内改到两个目录，`git -C tasks/<id>/repo-a status` 与 `repo-b` 均有改动。
3. 无 repo 任务：不选 repo，能创建文件并执行命令，`git -C tasks/<id> status` 显示产出。
4. 断线恢复：任务运行中关闭浏览器，重开仍能看到完整历史与实时流。
5. 服务重启：运行中 kill 控制面并重启，重启后状态被纠正为 failed 且「继续」可用，`-r` 能接上上下文。
6. 并发：同一 repo 同时跑 3 个任务，改动互不污染，各自 `git status` 独立。
7. push 阻断：在任务目录里 `git push` 被 hook 拒绝，`--deny` 若不生效也不影响结论。
8. 中止：点中止后 5s 内进程消失，状态 cancelled，目录保留。
9. Watchdog：人为把 `last_event_at` 改到 31 分钟前，60s 内被标 failed。

## 14. 已知未定项

- `streaming-json` 字段名与 `end` 事件的完整结构（§11）。
- `--sandbox` 的 profile 名与对文件系统访问的实际影响。
- 是否需要 mirror 缓存（取决于 repo 体积与任务频率）。
- 是否把控制面本身放进某个 git repo 并推 GitHub（当前假设本地 `~/sa` 即可）。
