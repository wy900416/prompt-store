# SpecGate 实施提示词（Amp / 编码 agent 执行）

你是这个仓库的实现 agent。请在**本机**把 SpecGate 从零实现出来：一个本地 web 应用，让人只审 spec 的变更，批准后自动产出 design / plan / tests 并触发编码 agent 实现。

## 0. 权威与边界

1. **`specgate-impl-spec.md` 是唯一真值源。** 先完整读完它，再动手。规格里写了的东西按规格做；规格没写的，优先选更简单、更少依赖的实现，并在最终报告里单列"规格未覆盖，我做了这三个决定"。
2. **先确认现实路径再写代码。** 用 `terminal` + `search_files (target='files')` 核实：
   - 目标目录是否存在（默认 `~/specgate`）、是否已有文件、是否有同名项目
   - Python 版本、是否已有 venv、`fastapi`/`uvicorn` 是否可用
   - `8790` 端口是否被占用
   - 机器上是否存在 `grok` / `amp` CLI，以及它们**真实的** headless 参数（跑 `--help`，不许凭记忆猜参数）
3. **不要覆盖已有文件。** 目标目录若已存在非空内容，先停下来把现状贴出来，等我确认。
4. **不允许改名、不允许自作主张删规格里的字段。** 数据结构、`check_id`、`data-testid`、错误码 `409` 这些是下游验收依赖的，名字必须逐字一致。
5. 遇到规格自相矛盾或无法实现的地方：**停下来问**，不要一边猜一边写。

## 1. 硬性工程约束

- 依赖只有 `fastapi`、`uvicorn[standard]`、`pydantic`。不要引 ORM、不要引前端框架、不要引任务队列、不要引 Redis。
- 前端是 `static/index.html` + `app.js` + `style.css` 三个文件，**无构建步骤、无 CDN**，离线可用。
- `AGENT_PROVIDER=stub` 必须让整条流水线在**无网络、无任何 LLM** 的情况下跑通全部验收项。真实 CLI 接入是可选路径。
- 所有 agent 调用走子进程；会话 id 落库；worker 只在 `workspaces/<spec>/<run>/` 里跑，并装 `pre-push` hook（exit 1）。
- 状态以 SQLite 为准（WAL 模式）；进程内状态只是缓存；启动时做一次对账。
- 每完成一个阶段，跑该阶段的验收命令，**把真实输出贴进你的汇报**。

## 2. 执行阶段（严格按序，不要跳）

### 阶段 0 — 硬化事实（先做，不写业务代码）

要做：
- 核实 §0.2 列出的四项现实
- 建 venv 并装依赖，记录实际版本
- 跑一次 `grok --help` / `amp --help`（若存在），把 **headless 调用、会话 id 位置、续跑方式** 的原文片段抄进 `docs/facts.md`；不存在则 `docs/facts.md` 里写明"本机无 CLI，仅实现 stub"
- 跑一次真实 CLI 的最小调用（若存在），把原始输出贴回

要求：`docs/facts.md` 只写**跑出来的事实**，不写推测。规格与事实冲突时，先改实现方案，并在汇报里指出冲突点。

### 阶段 1 — 骨架、DB、模型

产出：`config.py`、`db.py`（含 §4 全部 DDL）、`model.py`、`app.py`（只挂 `/healthz` 与静态目录）、`cli.py` 空壳。

验收：
```
python -c "import specgate, sqlite3; ..."     # 或等价的 DB 初始化调用
sqlite3 data/specgate.db ".tables"
sqlite3 data/specgate.db "PRAGMA table_info(rule);"
```
贴回：建表后的真实表名列表 + `rule` 表字段列表。

### 阶段 2 — Lint 引擎 + fixtures（先做这个，它是闸门的地基）

产出：`lint/checks.py`（§8 全部 check）、`lint/words.py`、`cli.py lint`、`fixtures/spec-seed.json`。

`fixtures/spec-seed.json` 必须**故意**包含：1 条缺 `source_url` 的 external、1 条未验证的 assumed、1 条 AND 了两个行为的复合 rule、1 条无 scenario 的 rule、1 条措辞模糊的 rule、以及若干条干净的 rule，使每条 check 至少被触发一次。

验收：
```
python -m specgate.cli lint --spec order-reservation --json
```
贴回：完整 JSON 输出 + 你期望的 level 分布，并说明每条 check 由哪条 fixture rule 触发。

### 阶段 3 — spec / rule / change 的核心闭环

产出：spec & rule & scenario 的 CRUD API、`POST /draft`（走 stub）、`POST /changes/{id}/assemble`（含 100% 覆盖校验）、单条与批量批准、`submit`（`409` + `reasons`）。

验收（贴回每条的真实响应体与 HTTP 码）：
- assemble 后 `set(change_entry.rule_id) == set(rule_rev.rule_id)` 为真
- 手动删掉一条 entry 再 assemble → 被拒绝，并指出缺哪条
- 缺来源的 external 批准 → `409`；补 `source_url` 后 → `200`
- assumed 未处置时 `submit` → `409`；`demote` 后 → `200`
- 措辞改动 → `semantic=wording`，`approve-all-wording` 生效

### 阶段 4 — 影响分析 + 流水线 + 闸门

产出：`impact.py`、`pipeline.py`（design/plan/implement/tests/gate 五段，全部可 stub 跑通）、`POST /gate/evaluate`。

要点（规格 §10/§11 已定义，照做）：
- design 阶段对每个设计决策打 `impl-only` / `spec-required` tag；出现 `spec-required` 必须中止并输出诊断
- tests 阶段强制 RED 验证：在未实现的基线上跑，必须失败且失败原因与该 rule 语义匹配，结果落 `test_red`
- gate 五条件全部满足才 `allow_merge=true`

验收（贴回响应）：
- 改一条 rule → 该 rule 的 ref 全 `stale`，其它 rule 的 ref 不变，数量与 fixture 预期一致
- 一个"未实现前就通过"的测试 → `allow_merge=false` 并指出该测试
- 实现 diff 触碰 `touch_paths` 外路径 → `allow_merge=false`

### 阶段 5 — Web GUI（优先级最高的界面部分，做完才算能用）

按规格 §12 实现五个视图，**评审台 `/c/<change_id>` 必须最先完成且最完整**。

硬要求：
- 卡片字段齐全（规格 §12.1 的 9 项），`data-testid="rule-card-<local_id>"` 逐字一致
- 语义等价折叠成一行 + 批量批准
- "需你判断 N 张"计数必须真实计算并显示；`N > 8` 出现黄色提示条；>15 条规范性 rule 出现 `L-TOO-MANY` 提示
- blocking 未清时提交按钮 `disabled` 且说明原因
- 键盘 `j/k/a/x/u/e/f/?` 可用
- 写视图与审视图分离（`/spec/<id>` 卡片模式 / 表单模式）

验收：
```
python -m specgate.cli smoke
```
该命令必须自实现：依次请求 5 个视图，断言关键 `data-testid` 存在、计数正确、闸门条状态正确、并打印每个断言的真实结果。贴回完整输出。

### 阶段 6 — 对账、watchdog、导出、全量验收

产出：启动对账、15/30 分钟 watchdog、`export.py` + import、`docs/facts.md` 补完。

最后：**逐条执行规格 §17 的 15 项验收**，每项给出命令与真实输出。任何一项没跑通，就明确写"未通过 + 原因 + 复现命令"，不要绕过、不要改小验收标准。

## 3. 禁止事项

- 禁止用"已实现 / should work / 应该没问题"作为完成凭证。每个结论后面必须跟一条真实执行的输出。
- 禁止为了让验收通过而修改验收标准、放宽 lint level、或把 blocking 降级为 warning。
- 禁止把 `AGENT_PROVIDER=stub` 做成"跳过流程"的空实现——stub 必须产出**结构合法的真实数据**（fixtures），让流水线真正走过每一段的代码路径。
- 禁止在 worker 目录里放任何凭据；token 只在控制面自己的环境里。
- 禁止在真实 CLI 接入时凭记忆猜参数——先跑 `--help`，把原文抄进 `docs/facts.md`。

## 4. 交付物清单

1. `~/specgate/` 可运行服务（`127.0.0.1:8790`），`AGENT_PROVIDER=stub` 下五个视图 + 全流程可跑
2. `fixtures/` 全套 + `docs/facts.md`（跑出来的事实，不是推测）
3. 规格 §17 的 15 项验收：每项的命令 + 真实输出 + 通过/未通过
4. 一份最终汇报：做了哪些规格未覆盖的决定、哪些地方与规格有出入及原因、遗留风险、下一步建议

## 5. 汇报格式

```
## 阶段 N 完成
命令: <实际执行的命令>
输出: <真实输出，不要摘要>
结论: 通过 / 未通过（附原因）

## 规格未覆盖的决定
- ...

## 与规格的出入
- ...（没有就写"无"）

## 遗留风险
- ...
```
