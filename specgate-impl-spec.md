# SpecGate 实现规格：规格驱动的评审闸门（Spec → Design → Plan → Code → Tests）

> 状态：草案，待人 review。目标读者：实现者（Amp / Grok Build 等编码 agent）、reviewer（人）。
> 相关：`super-agent-impl-spec.md`（任务隔离 / clone / 状态机 / 权限边界，本规格复用其结论）、`ws-setup-amp-prompt.md`（ws 多 repo 约定）。

## 0. 一句话

一个本地 web 应用：**人只审 spec 的变更**，spec 批准后自动产出 design、实施计划、测试，并触发编码 agent 去实现；之后的每一次改动也只改 spec，下游按最小受影响集合增量重算。

评审界面的唯一目标：让人在 10 分钟内判断"agent 分析出来的 spec 改动是否符合预期"，且**不可能因为界面折叠而漏看真实变更**。

## 1. 目标与非目标

### 目标（必须同时成立）

1. **rule 级评审**：评审单位是一条规则，不是一个文件、不是一个 PR。
2. **变更单，不是文本 diff**：人看到的是 `+ADDED / ~MODIFIED / −REMOVED` 的规则条目，带"为什么"和影响面。
3. **机械闸门**：blocking 的 spec 缺陷（不可测、无出处、未验证假设等）不清理就不能批准——这是代码约束，不是流程约定。
4. **下游可增量**：改一条 rule 只重算受影响的 design 节、plan 步骤、测试、实现点。
5. **可追溯**：任意一条实现或测试都能反查到它服务的 `rule_id`；任意一条 rule 都能列出它的测试与实现点。
6. **agent 起草、人做编辑**：spec 由 agent 从自由文本意图起草成结构化 rule，人只改错标、清红点、按批准。
7. **每次改动只改 spec**：人不会直接编辑 design / plan / tests / 代码；这些是 spec 的投影。

### 非目标（第一版明确不做）

| 不做 | 原因 |
|---|---|
| 多用户、鉴权、权限体系 | 单机自用，绑定 `127.0.0.1` |
| 多 repo 原子提交 / 跨仓事务 | 第一版一个 spec 对应一个目标 repo 集合，逐个开分支即可 |
| 自动合并到 `main` | 闸门只给"可以合并"的判定，合并动作留给人或后续版本 |
| 存量代码反向挖掘（brownfield baseline mining） | 独立的一大块，见 §16；第一版只支持新 spec 与手工录入的 baseline |
| 前端框架 / 构建步骤 / CDN | 离线可用、单文件前端，零构建 |
| LLM 当仲裁者决定能不能合并 | 闸门必须可复现：只有确定性 lint + 确定性校验能 block |
| 富文本 / 图片 / 流程图编辑器 | 纯文本 spec，够用 |

## 2. 形态与技术栈

- 单机、单进程、`FastAPI` + `uvicorn` + `SQLite`（WAL）+ 一个单页前端（原生 JS，无构建）。
- 默认监听 `127.0.0.1:8790`。
- 所有 agent 调用走**子进程 + headless**（`-p` 一次性提示），会话 id 落库以便续跑；不接 ACP。
- **必须内置 stub agent provider**：`AGENT_PROVIDER=stub` 时用 fixtures 返回固定 JSON。整条流水线在没有任何 LLM、没有网络的情况下必须可跑通、可验收。这是硬要求——否则实现者无法自测。
- Python 3.11+，依赖仅 `fastapi`、`uvicorn[standard]`、`pydantic`。不引 ORM、不引任务队列、不引 Redis。

## 3. 目录布局

```
~/specgate/
  app.py                     # FastAPI：REST + 静态文件
  db.py                      # schema、迁移、查询
  model.py                   # rule/change/scenario 的 pydantic 模型
  lint/
    __init__.py              # run_all()，返回 findings
    checks.py                # 每条检查一个函数，见 §8
    words.py                 # 禁用词表、不可观察动词表
  agent.py                   # provider 抽象：stub | grok | amp
  pipeline.py                # design / plan / implement / tests 四段，增量传播
  impact.py                  # rule → artifact_ref 的索引与失效计算
  export.py                  # spec → markdown 快照（审计用）
  config.py                  # 端口、路径、AGENT_PROVIDER、并发上限
  cli.py                     # `python -m specgate ...`：lint / export / smoke
  static/index.html          # 单页前端（评审台 + 其余视图，全部在这一页里切视图）
  static/app.js
  static/style.css
  fixtures/                  # stub provider 的输入输出，验收用
    spec-seed.json
    draft-order-reservation.json
    design-frag.md
    tests-frag.py
  data/specgate.db
  workspaces/<spec>/<run>/   # 每个实现 run 的隔离 clone（worker 的 cwd）
    AGENTS.md                # 每次启动重读的约束
    repo-a/                  # git clone --depth 1
```

## 4. 数据模型

**SQLite 是唯一真值源**；markdown 是导出物（`specgate export` 生成快照，可提交进 git 做审计）。理由见 §15 决策记录。

```sql
CREATE TABLE spec (
  id          TEXT PRIMARY KEY,      -- 'order-reservation'
  title       TEXT NOT NULL,
  intent      TEXT NOT NULL,         -- L0：为什么做、为谁、成功长什么样（≤5 行）
  non_goals   TEXT NOT NULL,         -- L0：一行一条，禁止为空
  glossary    TEXT,                  -- JSON [{term,def}]：spec 内所有名词的定义
  status      TEXT NOT NULL,         -- active | archived
  created_at  TEXT, updated_at TEXT
);

CREATE TABLE rule (
  id              TEXT PRIMARY KEY,  -- '<spec_id>#<local_id>'，稳定，永不因分类改变
  spec_id         TEXT NOT NULL,
  local_id        TEXT NOT NULL,     -- 'R7'，spec 内单调递增，不复用
  kind            TEXT NOT NULL,     -- event|state|unwanted|invariant|optional（EARS 句式，见 §6）
  authority       TEXT NOT NULL,     -- external|internal|assumed（见 §7）
  class           TEXT NOT NULL,     -- behavior|slo|security|ops
  statement       TEXT NOT NULL,     -- 单行规范句
  obligation      TEXT,              -- external 专用：我方如何履行（可自由改）
  source_url      TEXT,              -- external 必填
  source_hash     TEXT,              -- 抓取快照的 sha256，用于检测"来源已变"
  source_checked_at TEXT,
  tradeoff        TEXT,              -- internal 建议填：取舍 + 代价
  verified_at     TEXT,              -- assumed 验证时间
  verified_by     TEXT,
  verify_hint     TEXT,              -- assumed 专用：怎么验证（具体命令/文件/URL），界面的「去验证」显示它
  status          TEXT NOT NULL,     -- draft|approved|deprecated
  rev             INTEGER NOT NULL,  -- 每次内容变更 +1
  approved_hash   TEXT,              -- 批准时 statement+scenarios 的 sha256
  approved_at     TEXT, approved_by TEXT,
  created_at      TEXT, updated_at TEXT,
  UNIQUE(spec_id, local_id)
);

CREATE TABLE scenario (               -- L2：直接编译成测试，没有 scenario 的 rule 不许批准
  id TEXT PRIMARY KEY, rule_id TEXT NOT NULL,
  kind TEXT NOT NULL,                 -- happy|edge|failure
  given_ TEXT NOT NULL, when_ TEXT NOT NULL, then_ TEXT NOT NULL,
  seq INTEGER
);

CREATE TABLE rule_rev (               -- append-only；一处编辑一行，不可改不可删
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  rule_id TEXT NOT NULL, rev INTEGER NOT NULL,
  op TEXT NOT NULL,                   -- create|modify|remove
  before_json TEXT, after_json TEXT,
  by_ TEXT NOT NULL,                  -- human|agent
  why TEXT NOT NULL,                  -- 必填非空，空则拒绝写入
  ts TEXT NOT NULL
);

CREATE TABLE change (                 -- 一个评审单元 = 一次 spec 变更
  id TEXT PRIMARY KEY,                -- 'order-reservation-v3'
  spec_id TEXT NOT NULL,
  title TEXT NOT NULL,
  why TEXT NOT NULL,                  -- 变更意图，人写或 agent 起草后确认
  state TEXT NOT NULL,                -- drafting|open|approved|rejected|applied
  agent_summary TEXT,                 -- changeset 的语义叙述（agent 产出）
  created_at TEXT, decided_at TEXT, decided_by TEXT
);

CREATE TABLE change_entry (           -- 一行 = 一个 (change, rule)，100% 覆盖校验的对象
  change_id TEXT NOT NULL, rule_id TEXT NOT NULL,
  op TEXT NOT NULL,                   -- added|modified|removed
  semantic TEXT NOT NULL,             -- material|wording（自动判定，见 §9）
  rationale TEXT,                     -- agent 的一行说明
  decision TEXT NOT NULL,             -- pending|approved|rejected
  decided_by TEXT, decided_at TEXT,
  PRIMARY KEY(change_id, rule_id)
);

CREATE TABLE lint_finding (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  scope TEXT NOT NULL,                -- rule:<rule_id> | change:<change_id> | spec:<spec_id>
  check_id TEXT NOT NULL,             -- 见 §8，如 L-UNCERTAIN
  level TEXT NOT NULL,                -- blocking|warning|info
  message TEXT NOT NULL,
  detail_json TEXT,
  resolved_at TEXT,                   -- 置位即视为已处理（仅 warning/info 可手动置）
  ts TEXT NOT NULL
);

CREATE TABLE artifact_ref (           -- rule ↔ 下游产物的双向索引，影响分析的唯一依据
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  rule_id TEXT NOT NULL,
  kind TEXT NOT NULL,                 -- design|plan|test|impl
  target TEXT NOT NULL,               -- 'design.md#D4.2' | 'plan.md#step-3.2' |
                                      -- 'tests/test_order.py::test_out_of_stock' | 'orders/reserve.py:88'
  state TEXT NOT NULL,                -- live|stale|obsolete
  updated_at TEXT
);

CREATE TABLE test_red (               -- RED 验证记录，缺此记录的 test 不算合格
  artifact_ref_id INTEGER PRIMARY KEY,
  verified_at TEXT NOT NULL,
  failed_as_expected INTEGER NOT NULL, -- 1/0
  fail_reason TEXT NOT NULL            -- 断言失败信息，用于人眼复核"红得对不对"
);

CREATE TABLE pipeline_run (           -- 每个阶段一次运行
  id TEXT PRIMARY KEY, spec_id TEXT NOT NULL, change_id TEXT,
  stage TEXT NOT NULL,                -- design|plan|implement|tests|gate
  state TEXT NOT NULL,                -- queued|running|done|failed|blocked
  started_at TEXT, finished_at TEXT, exit_code INTEGER,
  session_id TEXT, log_path TEXT, report TEXT
);

CREATE TABLE event (                  -- agent 原始事件行，不解析地存
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  run_id TEXT NOT NULL, ts TEXT NOT NULL, kind TEXT NOT NULL, payload TEXT NOT NULL
);
CREATE INDEX idx_event_run ON event(run_id, id);
```

## 5. Spec 的四层内容（display 反推出的字段）

| 层 | 内容 | 是否逐条批准 | 在界面上 |
|---|---|---|---|
| L0 意图 | intent（≤5 行）+ non_goals + glossary | 否，随 change 一起过 | 评审台顶部折叠区 |
| **L1 规则** | 编号的规范句，带 kind / authority / class | **是，逐条** | 卡片主列表，唯一需要人肉眼过的东西 |
| L2 例证 | 每条 rule 1~3 个 GIVEN/WHEN/THEN | 否，但缺则 rule 不能批准 | 卡片内可展开 |
| L3 未决 | open questions（见 `change.why` 与 finding `L-OPEN`） | 否，但不许空着 | 评审台侧栏 |

**不许进 spec 的内容**：文件路径、类名、库选型、步骤顺序、重构策略、性能优化手段。这些属于 design / plan。

判断线（同时也是"design 不得新增可观察行为"的操作化定义）：**不看代码就能一句话判断合规 → 属于 spec；需要判断"怎么做到" → 属于 design。**

## 6. 受控自然语言（lint 与测试生成的前提）

每条 `statement` 必须匹配以下五种 EARS 句式之一，`kind` 与句式一致：

| kind | 句式 | 例 |
|---|---|---|
| `event` | 当 `<触发>` 时，系统应 `<可观察响应>` | 当库存不足时，系统应拒绝创建订单 |
| `state` | 在 `<状态>` 下，当 `<触发>` 时，系统应 ... | 在订单已支付状态下，当收到重复支付回调时，系统应返回 200 且不重复扣款 |
| `unwanted` | 若 `<不希望的情况>`，则系统应 ... | 若下游超时，则系统应回滚已扣库存并返回 504 |
| `invariant` | 系统应始终 `<不变量>` | 系统应始终保持 `reserved_qty ≤ available_qty` |
| `optional` | 仅在 `<前置条件>` 时，系统应 ... | 仅在 `FEATURE_PARTIAL_REFUND=on` 时，系统应支持部分退款 |

每条还要满足四要素：**可观察的响应 + 触发条件 + 量化指标（数字、状态码、字段名）+ 失败路径**。

## 7. 权威分类（agent 起草必填，人只改错标）

三值，`changeable` 是 `authority` 的推论，不做独立字段。

| authority | 含义 | 批准语义 | 界面动作 | blocking 条件 |
|---|---|---|---|---|
| `external` | 法规、上游 API 契约、平台限制、硬件现实、客户书面要求 | **"我对它的理解是否正确"**，不是"我同意" | `[引用无误] [引用有误] [来源已变更]` + 可点出处 | 无 `source_url`；agent 自己写了 `external` 却没出处 → **直接判 `found` 为伪造，blocking** |
| `internal` | 我方决策：选的行为、取舍、容错策略 | "我批准这个决策" | `[approve] [修改] [驳回]` | 无（缺 `tradeoff` 是 warning） |
| `assumed` | agent 对现状或外部世界的推断（如"现有对账任务每日 02:00 运行"） | 不允许批准，只能处置 | `[去验证] [降级为 known-drift] [升级为 reference]`；「去验证」必须显示 `verify_hint`（具体命令 / 文件 / URL） | 任何 `assumed` 处于未处置状态 → 该 rule 不能批准，其所在 change 不能放行 |

**external 必须拆成 claim + obligation**：

- `statement` 承载外部事实（只引用，不可改）
- `obligation` 承载"我们因此要做什么"（可自由改，属于我方决策）

外部条款更新只动前半句，我们的实现方式与测试保持不动；我们改实现节奏只动后半句，不触碰外部事实。**这是"变更不穿透"的机制来源。**

## 8. Lint 引擎

确定性检查，纯 Python，不调 LLM，全量跑完 < 3 秒。CLI：`python -m specgate lint --spec <id> --json`。

| check_id | level | 判定 |
|---|---|---|
| `L-SHAPE` | blocking | `statement` 不匹配 §6 五句式，或 `kind` 与句式不一致 |
| `L-VAGUE` | blocking | 命中禁用词/不可观察动词表（快、友好、稳定、适当、支持、处理、优化、尽量、合理、足够…） |
| `L-NO-QUANT` | blocking | 含时间/数量语但无数字或单位（"迅速返回"） |
| `L-COMPOUND` | blocking | 一条 rule 里 AND 了两个及以上独立可观察行为 → 要求拆条 |
| `L-NO-FAILURE` | blocking | `class=behavior` 且无任何 `kind=failure` 的 scenario |
| `L-NO-SCENARIO` | blocking | rule 无 scenario |
| `L-NO-SOURCE` | blocking | `authority=external` 且无 `source_url` |
| `L-ASSUMED` | blocking | `authority=assumed` 且 `verified_at` 为空且未降级 |
| `L-DANGLING` | blocking | 引用不存在的 `rule_id` 或 glossary 外实体 |
| `L-OPEN` | blocking | `change.why` 为空，或存在未决问题条目超过 0 条而 change 声明可放行 |
| `L-CONFLICT` | warning | 两条 rule 在同一条件下给出不同响应（确定性检测靠结构化 `kind+条件指纹`；agent 辅助检测结果只进 info） |
| `L-NEAR-DUP` | warning | 与已有 rule 语义近似（agent 判定，仅 info/警告，附相似度与并排展示） |
| `L-EDGE-UNDECLARED` | warning | 未声明空集合 / 并发 / 超时 / 幂等边界 |
| `L-SCENARIO-DRIFT` | warning | rule 改了但 scenario 未跟着改 |
| `L-LONG` | warning | `statement` 词数 > 25 |
| `L-NO-NONGOALS` | warning | spec 的 `non_goals` 为空 |
| `L-NO-OBSERVABILITY` | warning | 整个 spec 无任何 rule 描述日志 / 错误码 / 指标 |
| `L-TOO-MANY` | warning | change 内规范性 rule > 15 → 提示拆 change |

规则：**blocking 一律不可手动置为 resolved，必须改内容或改分类才消失**；warning/info 可 `resolved_at` 置位但保留记录。

## 9. 变更单生成与"语义等价折叠"

1. 人（或 agent 起草后经人确认）编辑 rule → 每次写入 `rule.rev+1` 且追加一行 `rule_rev`。
2. `POST /api/changes/{id}/assemble` 把自 `change.created_at` 以来的 `rule_rev` 组装成 `change_entry`：
   - `op`：create→`added`，modify→`modified`，remove→`removed`
   - `semantic`：`material` 或 `wording`，判定方式——对 `before/after` 的 `statement` 去空白、去标点、同义数字归一后相等，且 scenario 集合未变 → `wording`；否则 `material`
3. **100% 覆盖校验（机械断言）**：`set(change_entry.rule_id)` 必须等于 `set(rule_rev.rule_id WHERE change_id 范围)`。不等则拒绝进入 `open` 状态，并在界面显示缺失条目。这条替代"人工保证没漏"。
4. agent 只负责写 `rationale` 与 `agent_summary`（自然语言），**不参与决定有哪些条目**——条目集合是机械算出来的，agent 漏述不会导致漏审。

## 10. 影响分析与增量传播

`impact.py` 的职责：给定一组变更 rule_id，返回受影响的 design 节、plan 步骤、测试、实现点，并把这些 `artifact_ref` 标 `stale`。

- 新增 rule：无既有 `artifact_ref` → 产出"需新增 design / plan / test"清单。
- 修改 rule：其全部 `artifact_ref` 置 `stale`。
- 删除 rule：置 `obsolete`，其测试必须删除或改标 `obsolete`，**不允许静默保留**。
- 外部条款 hash 变化（`source_hash` 变了）：只把该 rule 置 `source_changed` 并开一个新 change，不动下游。

下游**只重算 stale 的部分**，不整份重写。design / plan 文件按 `<!-- rule:R7 -->` 段落锚点组织，重算时按锚点做段落级替换（这是"增量"可实现的机械前提）。

## 11. 下游流水线与闸门

| stage | 输入 | 输出 | 硬约束 |
|---|---|---|---|
| `design` | 已批准的 rule | `design.md`（按 rule 锚点分段，每段注明服务的 rule_id） | 每个设计决策必须打 tag：`impl-only` 或 `spec-required`。出现任何 `spec-required` → 中止，输出诊断清单，要求先提 spec change。**design 不得新增可观察行为。** |
| `plan` | design | `plan.md`（步骤编号、目标文件路径），并声明 `touch_paths` | 每个 step 必须挂至少一个 rule_id |
| `implement` | plan | 隔离 clone 上的分支 | worker 的 cwd 是 `workspaces/<spec>/<run>/`；禁 push（pre-push hook exit 1）；禁改 spec/design/plan 文件（写入被 API 层面拒绝，hook 兜底） |
| `tests` | scenario | 测试文件 + `artifact_ref(kind=test)` + `test_red` | 每个场景一个用例；用例头带 `# spec: <spec>#<local_id>`；**必须先跑 RED 验证**（在未实现的基线上运行，必须失败且失败原因与该 rule 语义匹配），红得不对 → 回炉 |
| `gate` | 以上全部 | 放行/拒绝判定 | 见下 |

**gate 全部条件同时满足才给"可合并"判定**（缺一即转人工）：

1. 变更涉及的所有 rule `status=approved`，且 `approved_hash` 与当前内容一致
2. 无 blocking finding
3. 每条变更 rule 至少有一个 `live` 的 test ref，且都有 `test_red.failed_as_expected=1`
4. 实现 diff 的路径集合 ⊆ `plan.touch_paths`（越界即转人工）
5. 变更涉及的所有 rule 无未处置的 `assumed`

**spec rule 覆盖率**是核心指标：`有 live 且 RED 验证过的测试的已批准 rule 数 / 已批准 rule 数`。不用代码行覆盖率。

## 12. Web GUI（本项目的核心，规格到组件字段）

单页 `static/index.html`，左侧导航 + 主区，无框架无构建。所有关键节点带 `data-testid`，以便验收脚本做 DOM 断言。

### 12.1 视图一：评审台 `/c/<change_id>`（最重要，优先做）

布局：单列卡片流 + 右侧固定摘要栏 + 底部固定闸门条。

```
┌ SpecGate ─ order-reservation · change v3 ──────────────────────────┐
│ ⚠ 2 blocking · 1 warning · 需你判断 3 张 · 已折叠 5 张(等同措辞)    │
├─────────────────────────────────────────────────────────────────────┤
│ ┌ R7  +ADDED          ●EXTERNAL   event ──────────────────┐        │
│ │ 当报告提交失败时,系统应保留原始报文并告警。               │        │
│ │ 出处在 条款4.3(2024-06) ✓verified                        │        │
│ │ scenarios 2 · 影响 1 test / 2 impl sites / design D4.2    │        │
│ │ [引用无误] [引用有误] [来源已变更]  [展开 scenario]        │        │
│ └──────────────────────────────────────────────────────────┘        │
│ ┌ R2  ~MODIFIED       ○INTERNAL   invariant ⚠ L-CONFLICT ┐        │
│ │ 旧:订单创建后库存立即扣减                                │        │
│ │ 新:订单创建后库存预占,支付成功后扣减                      │        │
│ │ why:支持部分退款  ·  取舍:需新增对账任务 / 代价:多一个定时任务 │    │
│ │ [approve] [修改] [驳回] [对比 R9]                          │        │
│ └──────────────────────────────────────────────────────────┘        │
│ ▸ 4 rules unchanged (collapsed)                                     │
│ ▸ 5 rules: 措辞变化,语义等价 — [全部批准]                           │
├─────────────────────────────────────────────────────────────────────┤
│ 2 blocking 未清 → 提交被禁用      [全绿则提交: 批准本次 change]      │
└─────────────────────────────────────────────────────────────────────┘
```

**卡片字段（缺一不可）**：`data-testid="rule-card-<local_id>"`

- `authority` 徽章（EXTERNAL 蓝 / INTERNAL 绿 / ASSUMED 琥珀虚框）
- `op` 徽章（`+ADDED` / `~MODIFIED` / `−REMOVED`）
- `local_id` + `kind`
- `statement`（单行；`modified` 支持并排旧/新与词级高亮，默认并排）
- `rationale` + `tradeoff`（`internal`）/ `source`（`external`，可点）/ 推断原文（`assumed`）
- scenario 折叠区，展开显示 GIVEN/WHEN/THEN
- **影响面一行**：`N tests · N design § · N impl sites`，可点开
- 该 rule 的 lint 红点，就地显示
- 按 `authority` 变化的动作按钮（见 §7）
- 决定状态

**折叠纪律（防止 overload 的机制，不是审美）**：

- `unchanged` 卡片默认收起，只显示计数
- `semantic=wording` 的条目**合并为一行**批量批准，不占注意力
- `ASSUMED` 且未处置的，即使未变更也置顶到"待验证"抽屉
- 界面必须显式统计并显示 **"需你判断 N 张"**；`N > 8` 时显示黄色提示条，建议拆分 change

**键盘**：`j/k` 上下、`a` 批准、`x` 驳回、`u` 撤销上一动作、`e` 就地编辑、`f` 折叠等同项、`?` 帮助。

**闸门条**：只要存在 blocking 或有 `pending` 条目，"提交"按钮 `disabled`，并说明原因。

### 12.2 视图二：Dashboard `/`

- 三个数字：待审 change 数、blocking 总数、spec rule 覆盖率
- spec 列表（每条显示已批准 rule 数 / 覆盖率 / stale artifact 数）
- 待验证 `assumed` 清单（跨 spec）
- 外部来源已变更清单（`source_hash` 漂移）

### 12.3 视图三：Spec 浏览 / 编辑 `/spec/<id>`

双模式切换：**卡片模式**（只读渲染，同评审台卡片）与**填写模式**（每条 rule 一个表单：句式下拉、statement 单行输入、authority、source、scenario 三列表）。写视图与审视图必须分离——不许让人在卡片里创作。

### 12.4 视图四：追溯 `/trace?rule=<id>`

左：rule 列表；右：该 rule 的 design 锚点、plan 步骤、测试（含 RED 状态）、实现位置、批准人/时间/哈希。任何一条实现点必须能点回它在哪个 rule 底下。

### 12.5 视图五：运行 `/runs`

pipeline_run 列表 + 实时事件流（WebSocket）+ 中止按钮 + 每个 run 的 report。沿用 `super-agent-impl-spec.md` 的 watchdog 规则：15 分钟无事件→标 stalled，30 分钟→杀进程并标 failed，pid 消失→标 failed 并允许 Resume。

## 13. API（最小面）

```
GET  /api/specs                            列表 + 覆盖率
POST /api/specs                            新建（body: id,title,intent,non_goals,glossary）
GET  /api/specs/{id}                       spec + rules + findings + 覆盖率
POST /api/specs/{id}/draft                 {intent} → 调 agent 起草 rules（全部 draft）
GET  /api/rules/{rule_id}                  单条 + revisions + refs
PATCH /api/rules/{rule_id}                 编辑（必须带 why、by）；rev+1
POST /api/rules/{rule_id}/decision         {action: approve|reject|verify|demote|promote}
POST /api/changes                          新建 change
POST /api/changes/{id}/assemble            由 rule_rev 组装 entries（含 100% 覆盖校验）
POST /api/changes/{id}/lint                跑 lint，落 findings
GET  /api/changes/{id}                     组装后的评审台数据（含计数、折叠分组）
POST /api/changes/{id}/entries/{rule_id}   单条批准/驳回
POST /api/changes/{id}/approve-all-wording 批量批准同义措辞
POST /api/changes/{id}/submit              整体放行（blocking 未清 → 409 并附原因）
POST /api/pipeline/{stage}/run             {spec,change} → 起子进程，返回 run_id
GET  /api/runs                             列表；GET /api/runs/{id}/events 事件流
POST /api/runs/{id}/abort
POST /api/gate/evaluate                    {spec,change} → {allow_merge, reasons[]}
GET  /api/export/{spec}                    markdown 快照
POST /api/import                         批量导入 spec（见 §18 导入契约），幂等
```

错误约定：闸门类拒绝统一返回 `409` + `{"reasons":[...]}`，前端直接渲染在闸门条上。

## 14. 非功能约束（可验收）

- 单次审阅：**需人判断的卡片 ≤ 8 张**（界面提示 + §1 目标）；change 内规范性 rule **≤ 15**（超出给 `L-TOO-MANY`）
- lint 全量 **< 3s**（100 条 rule 的 spec）
- 页面首屏 **< 500ms**（本地）
- 所有写操作幂等、落 `rule_rev` / 审计，服务重启不丢状态
- 状态以 DB 为准，进程内状态只是缓存；启动时对账（有 `running` 的 run 但无 pid → 置 failed）
- 无网络可用：所有前端资源本地，agent provider 为 `stub` 时全流程可跑
- 所有 agent 调用可被 `AGENT_PROVIDER=stub` 替换；不允许出现只有真实 LLM 才能跑的验收路径

## 15. 决策记录（为什么否掉其它方案）

| 决策 | 被否掉的替代 | 为什么否 |
|---|---|---|
| SQLite 为真值源，markdown 为导出快照 | markdown+git 为真值源，靠解析 diff | 需要解析人类自由格式、无法机械保证引用完整性与 100% 覆盖；界面必须每帧渲染结构化卡片。git 审计需求由 `export` 快照满足 |
| 评审单位 = rule 卡片 | 整份文档批准 / 按文件 diff 批准 | 整份批准是橡皮章；文本 diff 读不出语义变化 |
| 变更单机制生成，agent 只写叙述 | 让 agent 总结 diff | agent 漏述会导致人漏审。条目集合必须机械可算，叙述可以错但不会致盲 |
| 确定性 lint 才可 block | LLM 自评是否可合并 | 不可复现、不可解释；同类判定（语义近似/冲突）降级为 warning+agent 建议 |
| 三值权威分类 | 二元"可改/不可改" | `changeable` 是 `authority` 的推论，独立字段会出现"外部且可改"的矛盾组合；且缺 `assumed` 这一档，agent 的推断会伪装成事实 |
| external 拆 claim + obligation | 一条规则混合外部事实与我方实现 | 外部条款微调会穿透整条下游链，违背"只改 spec 下游增量" |
| EARS 五句式受控语言 | 自由自然语言 | 没有可机检形状，lint 与测试生成都无从下手 |
| 单页原生 JS、零构建 | React/Vue + 构建链 | 离线、单机、单人；构建链是纯粹的额外失败面 |
| 内置 stub provider | 直接接 CLI | 否则实现者无法自测、验收无法复现 |
| 借鉴 OpenSpec 的 delta 概念与 Kiro 的 EARS 需求层 | 直接采用 OpenSpec / Kiro / Spec Kit | 三者都没有 rule 级批准、语义等价折叠、影响面展示与机械闸门；本项目的差异化正在这四点 |

## 16. 后续版本（第一版不做，但接口要留出来）

- **brownfield baseline mining**：从存量代码反挖"应然 spec"，与现状不符的点必须显式记 `known-drift` 挂账，禁止假装一致。
- **drift 巡检**：定时从代码反挖实际行为，与已批准 spec 比对，产出违规清单。
- 多 repo 单 change 的原子分支与合并策略。
- 外部来源自动抓取复校（`source_hash` 漂移自动开 change）。
- **delta 导入**：带 `op=added/modified/removed` 的增量导入格式，让 agent 能直接起草变更提案而不是全量重导入（第一版靠"人在评审台改"过渡）。

## 17. 验收清单（必须真跑，输出贴回，不允许"已实现"式claim）

1. `python -m specgate lint --spec order-reservation --json` 对 `fixtures/spec-seed.json`（含 1 条缺来源的 external、1 条 assumed、1 条复合规则、1 条无 scenario）返回预期 findings 数，且 level 分布与预期一致。
2. `AGENT_PROVIDER=stub` 起草一个 change：`change_entry` 的 rule_id 集合与 `rule_rev` 集合完全相等（100% 覆盖校验通过）；故意删掉一条 entry 再 assemble → 返回拒绝并指出缺哪条。
3. 仅改措辞的编辑 → `semantic=wording`；`approve-all-wording` 一次批准；界面"需你判断 N 张"计数正确。
4. 对缺 `source_url` 的 external rule 调 `POST /entries/{id}` approve → `409`；补上出处后 → `200`。
5. `assumed` 未处置时 `submit` → `409`；`demote` 为 known-drift 后 → `200`。
6. 冲突检测：两条 rule 同条件不同响应 → `L-CONFLICT` 出现，并排展示。
7. 影响分析：改一条 rule → 其 `artifact_ref` 全部 `stale`，数量与 fixture 预期一致；其它 rule 的 ref 不受影响。
8. RED 闸门：给一条改动配一个"未实现前就通过"的测试 → `POST /gate/evaluate` 返回 `allow_merge=false` 并指出该测试。
9. scope 闸门：实现 diff 触碰 `touch_paths` 之外的路径 → `allow_merge=false`。
10. 重启服务：中途 kill 后重启，`running` 的 run 被对账为 `failed`，Resume 可用，无重复 run。
11. `git push` 在 worker cwd 内执行被拒绝（`pre-push` hook exit 1）。
12. 追溯：`GET /trace?rule=R7` 返回该 rule 的 design 锚点 / plan step / test（含 RED 状态）/ 实现位置 / 批准哈希。
13. change 含 16 条规范性 rule → `L-TOO-MANY` 出现；"需你判断"超过 8 张 → 黄色提示条出现。
14. 前端：`python -m specgate smoke` 抓取 5 个视图并断言关键 `data-testid` 存在且 blocking 未清时提交按钮为 `disabled`。
15. `specgate export` 产出的 markdown 能完整重建回 DB（export → 新库 import → rule 集合与 scenario 一致）。
16. 导入契约（§18）：`specgate import fixtures/spec-import-sample.json` 成功导入并返回 lint 计数；**同一文件重复导入第二次 = 0 条变更**（幂等）；把文件里一条**仍是 draft** 的 rule 内容改掉再导入 → 成功且该 rule `rev+1`；把一条**已批准**的 rule 改掉再导入 → `409` 并列出冲突的 `local_id`，不静默覆盖。

## 18. 导入契约（agent 起草 spec 的唯一入口）

外部（agent 起草器、人写、CI）向系统灌 spec，只有这一种格式。实现者必须让 `POST /api/import` 与 `specgate import <file.json>` 行为完全一致。

### 18.1 文件格式

```json
{
  "spec": {
    "id": "order-reservation",
    "title": "订单库存预占",
    "intent": "为防止超卖，下单时先预占库存，支付成功再扣减。成功标准：超卖为 0。",
    "non_goals": ["不做跨仓调拨", "不处理预售商品"],
    "glossary": [
      {"term": "预占", "def": "在库存上占用数量但尚未扣减，可回滚"}
    ]
  },
  "change": {
    "id": "order-reservation-v1",
    "title": "初始 spec",
    "why": "第一版库存预占规则"
  },
  "rules": [
    {
      "local_id": "R1",
      "kind": "unwanted",
      "authority": "internal",
      "class": "behavior",
      "statement": "若可用库存不足，则系统应拒绝创建订单并返回 409、code=OUT_OF_STOCK。",
      "obligation": null,
      "source_url": null,
      "tradeoff": "预占需要额外一张表与清理任务，换取不超卖",
      "verify_hint": null,
      "scenarios": [
        {"kind": "failure", "given": "商品可用库存为 0", "when": "提交下单请求", "then": "返回 409 且不创建订单记录"}
      ]
    }
  ]
}
```

### 18.2 语义

- 全部 rule 以 `status=draft` 落库，`rev=1`，写一行 `rule_rev(op=create, by_=agent|human, why=change.why)`。
- `change` 存在时：创建 `state=open` 的 change，`assemble` 出全部 `op=added` / `semantic=material` 的 entry；`change.why` 为空则整份导入被拒（400）。
- `change` 省略时：只建 spec 与 rules，不建 change（用于纯录入）。
- **导入即跑 lint**，findings 落库。blocking 存在**不阻止导入**（数据要落得下来），但阻止该 rule 被批准、阻止 change 放行。
- 幂等键：`(spec.id, rule.local_id)`，三种情形：
  1. 不存在 → 创建（`rev=1`, `op=create`）
  2. 存在，且 `status=draft` 且未被任何 `change_entry` 决策过 → **允许覆盖**（`rev+1`, `op=modify`, `by_=agent`, `why` 写明"重导入"）。这条让 agent 的"起草 → 导入 → 读 lint → 修 → 再导入"能闭环。
  3. 存在且已批准（有 `approved_hash`）或已有人工决策 → 整份导入**失败** `409`，列出冲突的 `local_id` 与差异字段，绝不静默覆盖。此时变更必须走 change 流程（人在评审台改），而不是重导入。
- 返回体：`{"spec_id": "...", "imported_rules": n, "skipped_rules": n, "change_id": "...|null", "lint": {"blocking": n, "warning": n, "findings": [...]}}`。
- 该导入路径同样受 §7 的 authority 约束：`authority=external` 而无 `source_url` → `L-NO-SOURCE` blocking。

