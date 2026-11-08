# SpecGate 实施提示词（自主一条龙：跑完为止，只在 blocker 停）

你是这个仓库的实现 agent。任务：在**本机**把 SpecGate 从零实现出来并跑完全部验收——一个本地 web 应用，让人只审 spec 的变更，批准后自动产出 design / plan / tests 并触发编码 agent 实现。

## 0. 执行模式（最重要，先读这一节）

1. **一条龙连续执行。** 阶段 0→6 一次跑完，**每个阶段结束不停下来等人确认**，自检通过就继续下一阶段。
2. **不要向我提问。** 我不在中途。你本来想问的问题，按 §3 默认选择表自己决定，写进 `docs/decisions.md`，继续往前走。
3. **只有 §2 的 blocker 清单命中才停。** 停时把状态、卡点、可选项写进 `docs/blocker.md` 和最终汇报，然后结束本次运行——不要用破坏性操作硬试。
4. **失败不算 blocker。** 某个验收项跑不通，按 §6 自修复预算处理，记「未通过 + 原因 + 复现命令」后继续做剩下的，别因为一项挂了就停。
5. **边跑边落盘。** 每完成一个阶段更新 `docs/progress.md`（阶段、已完成验收项、剩余项、下一步），保证被打断后能续跑。

## 1. 权威与边界

1. **`specgate-impl-spec.md` 是唯一真值源。** 先完整读完再动手。规格没写的地方，选更简单、依赖更少的实现，并在汇报里单列「规格未覆盖，我做了这些决定」。
2. **先核实现实，再写代码。** 用 `terminal` + `search_files (target='files')` 确认：目标目录是否存在及内容、Python 版本、venv、`fastapi`/`uvicorn` 是否可用、`8790` 端口占用、机器上是否有 `grok` / `amp` CLI 及它们**真实的** headless 参数（跑 `--help`，不许凭记忆猜）。
3. **不许覆盖别人的东西。** 目标目录已有非本项目内容 → 按 §3 换目录，不删、不清空。
4. **名字逐字一致。** 表名、字段名、`check_id`、`data-testid`、HTTP 码 `409`、`AGENT_PROVIDER=stub`、验收项编号都是下游依赖，不许改名、不许删字段。
5. 机器上已有 `~/specgate` 且能启动 → **增量补齐**缺失的验收项，不要推倒重来。

## 2. Blocker 清单（封闭集合，只有这些才停）

| # | 情形 | 为什么必须停 |
|---|---|---|
| B1 | 需要**删除或覆盖**已有文件/目录/数据库，且无法判断它是否本项目的产出 | 破坏性、不可逆 |
| B2 | 需要动系统级配置（改全局 python、装系统包、改防火墙、改已有 systemd 服务） | 影响本机其它东西 |
| B3 | 规格内部互斥到两个实现方向二选一，且规格里找不到判断依据 | 猜错整层返工 |
| B4 | 缺无法替代的外部资源：网络不可用且 fixtures 也替代不了、必须有真实凭据、磁盘不足 | 环境问题，你解决不了 |
| B5 | 需要 `git push` 到不属于本项目的远程仓库 | 越权 |
| B6 | 某验收项按 §6 用满 3 轮仍失败，且判定原因在环境而非代码 | 需要人来看环境 |

不在此列的一律按 §3 自己决定。

## 3. 默认选择表（遇到歧义照此处理，不问我）

| 情形 | 默认选择 |
|---|---|
| `8790` 被占用 | 依次试 `8791`→`8792`，实际端口写进 `config.py` 与最终汇报 |
| `~/specgate` 非空且不是本项目 | 改用 `~/specgate-build`，原目录原样不动并在汇报里点明 |
| `~/specgate` 是本项目半成品 | 读 `docs/progress.md` 从中断处续跑 |
| 机器上没有 `grok` / `amp` CLI | 只实现 `AGENT_PROVIDER=stub`，真实接入写成 `agent.py` 的可插拔函数 + TODO，记录 |
| 规格未定义的小细节（字段顺序、函数名、错误文案、日志格式、CSS 数值） | 选最简单、依赖最少的；CSS 用规格给的字面值，没给的用黑白灰 + 规格里的语义色（蓝/绿/琥珀/红） |
| 测试框架 | `pytest`（装进 venv） |
| 某个 check 的真值判断有歧义 | 按 §8 字面实现，不许自行放宽 |
| 需要示例数据 | 用你自己造的合法 fixture，别去网上抓 |
| 不确定要不要写文档 | 写，放 `docs/` |

每个默认选择都进 `docs/decisions.md`：`决定 / 理由 / 如果错了怎么回退`。

## 4. 落盘三件套

- `docs/facts.md` — **跑出来的事实**：版本号、`--help` 原文片段、端口、路径、真实输出。不写推测。规格与事实冲突时按事实改方案，并在汇报里指出冲突点。
- `docs/decisions.md` — 你替人做的每个决定（§3 全部条目 + 规格未覆盖处）。
- `docs/progress.md` — 阶段进度与验收项打勾表，用于续跑。

## 5. 硬性工程约束

- 依赖只有 `fastapi`、`uvicorn[standard]`、`pydantic`（+ 测试用 `pytest`）。不引 ORM、前端框架、任务队列、Redis。
- 前端 `static/index.html` + `app.js` + `style.css`，**零构建、无 CDN**，离线可用。
- `AGENT_PROVIDER=stub` 必须让整条流水线在**无网络、无任何 LLM** 下跑通全部验收；stub 要产出结构合法的真实数据（fixtures），不许做成「跳过流程」的空壳。
- agent 调用走子进程，会话 id 落库；worker 只在 `workspaces/<spec>/<run>/` 里跑，并装 `pre-push` hook（exit 1）。
- SQLite WAL 为唯一真值源；进程内状态只是缓存；启动做一次对账。
- 服务最后要**留在运行状态**（后台 `uvicorn`），最终汇报给出访问地址与重启命令。

## 6. 自修复预算

每个验收项：`跑 → 读真实报错 → 定位 → 修 → 重跑`，最多 3 轮。

- 3 轮内通过 → 记录通过 + 输出
- 3 轮未通过 → **记录 `未通过：<原因> + 复现命令`，继续下一项**，不要为它停下整个构建
- 同一根因导致多项一起挂 → 当一个修，修好后相关项全部重跑

**唯一不可协商**：禁止为了让某项变绿而缩小验收标准、把 blocking 降级成 warning、或删断言。

## 7. 阶段（按序，每阶段自检后直接进入下一阶段）

### 阶段 0 — 硬化事实
核实 §1.2 全部项；建 venv 装依赖；存在 CLI 就跑 `--help` 与一次最小调用，原文抄进 `docs/facts.md`；不存在就明确写「本机无 CLI，仅 stub」。

### 阶段 1 — 骨架、DB、模型
`config.py`、`db.py`（规格 §4 全部 DDL）、`model.py`、`app.py`（`/healthz` + 静态目录）、`cli.py` 空壳。
自检：建库后 `sqlite3 data/specgate.db ".tables"` 与 `PRAGMA table_info(rule);` 的真实输出。

### 阶段 2 — Lint 引擎 + fixtures（闸门地基，先做这个）
`lint/checks.py`（§8 全部 check）、`lint/words.py`、`cli lint`、`fixtures/spec-seed.json`。
fixture 必须**故意**包含：缺 `source_url` 的 external、未验证的 assumed、AND 了两个行为的复合 rule、无 scenario 的 rule、措辞模糊的 rule，使每条 check 至少触发一次。
自检：`python -m specgate.cli lint --spec order-reservation --json` 的输出与期望 level 分布逐条对照。

### 阶段 3 — spec / rule / change 核心闭环
spec & rule & scenario CRUD、`POST /draft`（stub）、`POST /changes/{id}/assemble`（含 100% 覆盖校验）、单条与批量批准、`submit`（`409` + `reasons`）、`POST /api/import` + `cli import`（规格 §18，含幂等三情形与 draft 覆盖语义）。
自检：覆盖校验为真；删 entry 后 assemble 被拒；缺来源 external 批准 `409`、补来源后 `200`；assumed 未处置 `submit` `409`、demote 后 `200`；措辞改动 `semantic=wording` 且批量批准生效；重复导入第二次 0 变更。

### 阶段 4 — 影响分析 + 流水线 + 闸门
`impact.py`、`pipeline.py`（design/plan/implement/tests/gate，全部可 stub 跑通）、`POST /gate/evaluate`。
要点：design 决策打 `impl-only`/`spec-required` tag，出现后者即中止并输出诊断；tests 强制 RED 验证并落 `test_red`；gate 五条件全满足才 `allow_merge=true`。
自检：改一条 rule → 该 rule 的 ref 全 stale 且其它 rule 不受影响；假测试 → `allow_merge=false`；diff 越界 → `allow_merge=false`。

### 阶段 5 — Web GUI（评审台优先）
五个视图，`/c/<change_id>` 最先做完且最完整。
硬要求：卡片 9 项字段齐全、`data-testid="rule-card-<local_id>"` 逐字一致、语义等价折叠 + 批量批准、「需你判断 N 张」真实计数、`N>8` 黄色提示条、>15 条规范性 rule 出 `L-TOO-MANY`、blocking 未清时提交按钮 `disabled` 且说明原因、键盘 `j/k/a/x/u/e/f/?`、写视图与审视图分离。
自检：`python -m specgate.cli smoke`（自己实现）：请求 5 个视图，断言关键 `data-testid`、计数、闸门条状态，打印每个断言的真实结果。

### 阶段 6 — 对账、watchdog、导出、全量验收
启动对账、15/30 分钟 watchdog、`export.py` + import 往返、补完 `docs/facts.md`。
然后**逐条执行规格 §17 的 16 项验收**，每项给命令 + 真实输出。完成后把服务留在运行状态。

## 8. 禁止

- 禁止用「已实现 / should work / 应该没问题」当完成凭证；每个结论后必须跟一条真实执行的输出。
- 禁止缩小验收标准、放宽 lint level、把 blocking 降级为 warning、删断言。
- 禁止把 `AGENT_PROVIDER=stub` 做成跳过流程的空壳——stub 必须走完每一段真实代码路径。
- 禁止把凭据写进 worker 目录；token 只在控制面自己的环境。
- 禁止凭记忆猜 CLI 参数——先 `--help`，把原文抄进 `docs/facts.md`。
- 禁止中途向我提问（除非 §2 blocker），也禁止在汇报里留开放式提问——要给默认决定 + 回退方案。

## 9. 交付物

1. 可运行服务（`127.0.0.1:<port>`，端口以实际为准），`AGENT_PROVIDER=stub` 下五视图 + 全流程可跑，**服务留在运行状态**
2. `fixtures/` 全套、`docs/facts.md`、`docs/decisions.md`、`docs/progress.md`
3. 规格 §17 的 16 项验收：命令 + 真实输出 + 通过/未通过
4. 最终汇报（见 §10）

## 10. 最终汇报格式（一次给全，别分多条发）

```
## 交付状态
服务地址 / 启动命令 / 实际端口 / 运行方式（前台 or 后台 + pid）
验收：16 项中通过 n，未通过 m（逐条列未通过项 + 原因 + 复现命令）

## 逐项验收证据
1. <验收项> — 命令: ... / 输出: <真实输出>

## 我替你做过的决定（不用你回，但要让你知情）
- <决定> / 理由 / 回退方式

## 与规格的出入
- ...（没有就写"无"）

## Blocker（没有就写"无"）
- B<n>: <卡点> / 我试过什么 / 可选项

## 遗留风险与下一步建议
- ...
```
