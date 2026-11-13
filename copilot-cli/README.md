# Copilot CLI 定制包

给 GitHub Copilot CLI 用的 4 个 agent + 3 个 skill。
无第三方依赖，纯 markdown，公司环境可审。

## 装法

项目级（团队共享，进 git）：

```bash
mkdir -p .github/agents .github/skills
cp copilot-kit/agents/*.agent.md .github/agents/
cp -r copilot-kit/skills/* .github/skills/
```

个人级（所有项目可用，不进 git）：

```bash
mkdir -p ~/.copilot/agents ~/.copilot/skills
cp copilot-kit/agents/*.agent.md ~/.copilot/agents/
cp -r copilot-kit/skills/* ~/.copilot/skills/
```

同 ID 时个人级覆盖项目级。装完 `/agent` 能看到，`/skills list` 能看到 skill。
改完不用重启：`/skills reload`。

## 内容

### agents

- **spec-miner** — 从代码反推行为规格。只读，输出契约/边界/隐含假设/与约定不符处/可评审断言。工具只有 read + search，物理上改不了文件。
- **ws-scout** — 多仓库 workspace 里定位某个行为属于哪个 repo。只读，输出命中表 + 真正的归属 + 改动影响面。
- **reviewer** — 评审变更，只报会导致故障的问题。不带 edit/execute，纯静态评审，零副作用。
- **fixer** — 把失败的测试或构建修到绿。唯一带 edit + execute 的，协议里写死了不许夹带重构、不许改断言、三次不成就停。

### skills

- **spec-review-rubric** — 评审规格文档的六项清单和固定输出结构。配 spec-miner 用。
- **session-handoff** — 收尾写 `.copilot/handoff/latest.md`，开头读它恢复上下文。这就是跨会话经验沉淀。
- **minimal-diff** — 改他人代码时的最小改动纪律。

## 顺带解决权限弹窗

agent 文件里的 `tools` 字段已经在源头砍掉了大部分弹窗：reviewer 没有 execute，spec-miner 连 edit 都没有，自然不会被问"允许写入吗"。

再按这个顺序往上调，从保守到激进：

```bash
# 1. 中间档：模型审查后自动放行安全操作，危险的才问你
/permissions assisted

# 2. 设为每次新会话的默认（写进 ~/.copilot/settings.json）
/permissions assisted   # 然后 /settings 里把 defaultPermissionMode 改 assisted
# 或者直接编辑 ~/.copilot/settings.json：
# { "defaultPermissionMode": "assisted" }

# 3. 精确白名单：只放行具体命令，别的一律还问
copilot --allow-tool='shell(git:*),shell(npm test),shell(npm run build)'

# 4. 彻底放行，但用沙箱兜底（先开沙箱再放行，顺序不能反）
#    进会话后先 /sandbox enable，再
copilot --allow-all-tools
```

检查当前是什么档：`/permissions show`。
误点错了想退回：`/permissions reset` 清掉本会话的所有批准，`/reset-allowed-tools` 连模式一起回 manual。

企业管理员可能禁掉 `allow-all`（`permissions.disableBypassPermissionsMode`），但 `assisted` 一般还能用。

## 少问"要不要继续"

三类停问，三种解法：

1. **工具权限弹窗** — 上面那套。
2. **agent 主动停下来问你** — 用 `--no-ask-user`，或者在 prompt 里写明"遇到分歧自己按最保守方案决定，做完再一起说"。
3. **任务没结束它以为结束了** — 用 autopilot：`/goal 把 X 改到测试全绿 --max-ai-credits 5`，或者 `Shift+Tab` 切到 autopilot 模式。它会自己往下推，到信用上限才停。

注意 autopilot 下需要人工审批的请求会被**直接拒绝**而不是等你，并把原因回传给 agent 让它换路子。所以 autopilot + assisted 是搭配用的，不是替代。
