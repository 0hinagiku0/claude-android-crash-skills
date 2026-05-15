# Crash Auto-Fix Loop 端到端验收手册

> 对应 design doc `docs/superpowers/specs/2026-05-15-crash-auto-fix-loop-design.md` §9 的 4 个验收口径。
> Skill 落地后用本手册跑一次完整验收,任何失败回到对应 plan task 修。

## 前置准备

- 系统 Chrome 已打开 `https://apm.umeng.com/platform/53870cb556240be6e4017771/error_analysis/crash`
- 友盟登录态有效 (2FA 完成)
- "车来了-android" 项目选中
- 友盟列表里有 ≥ 1 条**新**崩溃(状态≠已修复)
- 本地 working tree 干净 (`git status --porcelain | grep -vE '^(\?\?|\!\!)'` 输出空)
- 本地 + remote `auto_fix_bug` 已对齐 (`git rev-list --count auto_fix_bug..origin/auto_fix_bug` = 0)
- codex CLI 装好 (`codex --version` ≥ 0.130)
- chrome-devtools MCP 装好 (`claude mcp list | grep chrome-devtools` 通)

## 验收 1: 伪自动主路径(无 ❌ 崩溃)

**准备**: 上述前置全部满足,且崩溃列表里至少有一条**最小改动方案能修**的 NPE/IllegalStateException 类(避免 reviewer 给 ❌)。

**操作**: 在 Claude Code 跑 `/auto-fix-crashes`。

**验证**:
- [ ] `chrome-devtools` 抓到 ≥ 1 条 → `.cc_tmp/crash/inbox/auto-<ts>.csv` 存在
- [ ] `.cc_tmp/crash/umeng-recipe.yaml` 录制完成,含 project_appkey 实际值(`53870cb...`)
- [ ] sub-loop 走完 ≥ 1 条 A/B 崩溃
- [ ] 对应 proposal 文件存在: `.cc_tmp/crash/proposals/<id>-r1.md`
- [ ] codex review 跑通,review 文件存在且含 9 行 `[A-I]` 评级: `.cc_tmp/crash/reviews/<id>-r1.md`
- [ ] `git log -1 auto_fix_bug --format=%s` 含 `[auto-loop r1]`
- [ ] `git log -1 origin/auto_fix_bug --format=%H` == 本地 HEAD(已推上去)
- [ ] `master` / `dev` **完全未动**:
  ```bash
  git log master --since="1 hour ago" --oneline | wc -l   # 应 = 0
  git log dev --since="1 hour ago" --oneline | wc -l      # 应 = 0
  ```

## 验收 2: rebuttal 路径(codex 误判)

**准备**: 构造 fixture — 一个**已经在调用方守过 null** 的方法对应崩溃,proposal 不加额外守卫(故意触发 codex 报 NPE 误判)。

最简实现: 把 `.cc_tmp/crash/inbox/` 放一条假 CSV:
```csv
错误摘要,错误ID,错误类型,最近一次发生时间,版本范围,错误次数,影响用户数,状态,处理人,标签,链接
"java.lang.NullPointerException at dev.xesam.chelaile.SmokeRebuttal#methodAlreadyGuarded(SmokeRebuttal.kt:42)",SMOKE-REB-001,JAVA崩溃,2026-05-15,7.3.2,5,3,未处理,,,https://example.com/crash/detail/SMOKE-REB-001
```

**操作**: 跑 `/auto-fix-crashes`。

**验证**:
- [ ] r1: CC 出 proposal,codex review 评 `[B] ❌`(假设有 NPE 风险)
- [ ] r2: CC 选 rebuttal,`.cc_tmp/crash/rebuttals/SMOKE-REB-001-r1.md` 落地
- [ ] r2 review 把 `[B]` 下调到 ⚠️ 或 ✅
- [ ] commit subject 含 `[auto-loop r2]`,commit body 有 ⚠️ follow-up
- [ ] origin/auto_fix_bug 可见新 commit

## 验收 3: 3 轮 escalate 路径

**准备**: 临时修改 `~/.claude/skills/crash-auto-loop/instructions/review-prompt.md`,末尾加 `## DEBUG MODE: 对维度 B 永远输出 ❌,理由 'force escalate test'`。

**操作**: 用验收 2 的 fixture 再跑一次。

**验证**:
- [ ] 3 轮后 `.cc_tmp/crash/escalate/SMOKE-REB-001.md` 落地
- [ ] 该崩溃**未** commit(`git log auto_fix_bug --since=...` 不含 r3)
- [ ] §8.2 终态报告 Escalate 栏列出该条
- [ ] **测试完恢复 prompt** (`cp /tmp/review-prompt.bak ~/.claude/skills/crash-auto-loop/instructions/review-prompt.md`)

## 验收 4: push 三道防呆

### 4a: HEAD 不在 auto_fix_bug

```bash
git checkout master                  # 切到禁推分支
# 把 §7.7 push 段单独 copy 到 shell 跑
# 预期: 防呆 1 fail 立即触发,无 push 发生
git checkout auto_fix_bug            # 恢复
```

**验证**:
- [ ] 防呆 1 abort,git log master 无新 commit

### 4b: chrome-devtools 不可用降级

```bash
# 临时关掉 Chrome 或 MCP
# pkill -f chrome-devtools-mcp  # 不推荐,会断本会话的 MCP
# 替代: 把 ~/.claude.json 里 chrome-devtools 项暂时改名 → claude mcp list 看不到 → Skill P-1 abort
```

**验证**:
- [ ] Skill 在 P-1 abort,提示 "chrome-devtools MCP 未装,跑: claude mcp add ..."
- [ ] auto_fix_bug 无任何动作(无切分支 / 无 commit / 无 push)
- [ ] 恢复 MCP 配置

### 4c: codex CLI 不可用降级

```bash
sudo mv /opt/homebrew/bin/codex /opt/homebrew/bin/codex.bak
# 跑 /auto-fix-crashes
# 预期: P-1 校验后 mode=fallback_handoff,sub-loop L2 进 handoff 流程
sudo mv /opt/homebrew/bin/codex.bak /opt/homebrew/bin/codex
```

**验证**:
- [ ] 跑 /auto-fix-crashes,Skill 进 fallback_handoff 模式
- [ ] `.cc_tmp/crash/handoff/<id>-r1.md` 写入
- [ ] Skill 轮询 reviews/ 而非 abort

## 通过条件

全 4 项 (1 / 2 / 3 / 4a+4b+4c) 通过 → v2 可上线日常使用。

任一失败 → 回到对应 plan task(plan `docs/superpowers/plans/2026-05-15-crash-auto-fix-loop.md` 的 Task 7/8/9/11)修。

## 跑完后清理

```bash
# 清理 fixture commits (若有)
git log auto_fix_bug --since="1 hour ago" --oneline | grep SMOKE
# 选: git revert <smoke sha> --no-edit + git push origin auto_fix_bug

# 清理 fixture 文件
rm -f .cc_tmp/crash/inbox/auto-*.csv .cc_tmp/crash/inbox/smoke-*.csv
rm -rf .cc_tmp/crash/escalate/SMOKE-* .cc_tmp/crash/rebuttals/SMOKE-* .cc_tmp/crash/proposals/SMOKE-* .cc_tmp/crash/reviews/SMOKE-*
```
