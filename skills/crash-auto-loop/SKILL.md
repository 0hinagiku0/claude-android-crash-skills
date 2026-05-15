---
name: crash-auto-loop
description: 用户输入 /auto-fix-crashes 时触发。全自动崩溃修复闭环:chrome-devtools MCP 抓友盟崩溃日志 → CC 出方案 → codex review CLI 评审 → CC 改代码或写 rebuttal 最多 3 轮 → 9 维度无 ❌ 自动 commit + push auto_fix_bug 分支。Skill 永不动 master/dev,push 前三道防呆,永不 --force。
---

# Crash Auto-Fix Loop Skill

> 设计 spec: `docs/superpowers/specs/2026-05-15-crash-auto-fix-loop-design.md`
> 实施 plan: `docs/superpowers/plans/2026-05-15-crash-auto-fix-loop.md` (路径相对 repo 根)
> v1 参考: `~/.claude/skills/crash-analyzer/SKILL.md` (复用 §3 分类决策树、§6.5 commit message 模板、§7 detekt 规避)
> Reviewer 角色: `codex review` CLI (`/opt/homebrew/bin/codex` v0.130.0)

## 1. 何时触发

用户在 Claude Code 中输入 `/auto-fix-crashes`(不带参数)。

## 2. 最高优先级原则

### 2.1-2.3 沿用 v1 crash-analyzer §2 三条
> 三条同进同出,统一引用 v1 取全文规则,不单独 override 任一条。

完全沿用 v1 `~/.claude/skills/crash-analyzer/SKILL.md` 的三条规则:
- 最小改动优先 ⭐
- 重构方案需打 ⚠️ 标记并由用户决定
- detekt 兼容

**Skill 执行时 Read 该文件取规则全文,不以节号定位**(防 v1 重编号后引用失联)。

### 2.4 【push 安全红线 — v2 新增】

- 只 push `auto_fix_bug`,push 前三道防呆校验:
  1. 当前 branch 是 `auto_fix_bug`(`git rev-parse --abbrev-ref HEAD`)
  2. branch 的 remote 是 `origin`(`git config --get branch.auto_fix_bug.remote`)
  3. 永不 `--force` / `--force-with-lease`,push 命令不含 `+refs`
- push 失败后**本次 Skill 运行内永不自动重试**,直接 escalate 显示 stderr;用户可重新跑 `/auto-fix-crashes` 触发新一次尝试
- 永不动 `master` / `dev`

## 3. 前置校验 (Step P-1)

任一 abort 项失败 → Skill 退出并提示用户修。codex CLI 不可用时允许降级(见 §3.1)。

### Onboarding (首次使用前一次性设置,实测固化)

> ⚠️ **这些步骤必须在 Claude Code session 启动前完成**(MCP 工具是 session 启动时 attach,session 中途装的 MCP 不会自动 attach 到当前 session)。

1. **装 chrome-devtools MCP,带 `--browserUrl` 连真实 Chrome**(默认 puppeteer 空 profile 没登录态):
   ```bash
   claude mcp add --scope user chrome-devtools -- \
     npx -y chrome-devtools-mcp@latest --browserUrl http://127.0.0.1:9222
   ```

2. **启 Chrome 带 debug port + 独立 profile**(Chrome 136+ 安全策略,默认 profile 用 `--remote-debugging-port` 会被静默忽略):
   ```bash
   open -na "Google Chrome" --args \
     --remote-debugging-port=9222 \
     '--remote-allow-origins=*' \
     --user-data-dir="$HOME/.cc_tmp/chrome-debug-profile" \
     --new-window \
     "https://apm.umeng.com/platform/<appkey>/error_analysis/crash"
   ```
   - `*` 必须用引号包(zsh 会把 `*` 当 glob)
   - 验证: `curl -s http://127.0.0.1:9222/json/version` 应拿到 Chrome 版本 JSON
   - 在该窗口手动登录友盟一次(2FA 等),登录态固化到独立 profile,后续跑不需要重登

3. **启动 Claude Code session,在 repo 根目录加载**,然后 `/auto-fix-crashes`。

### P-1 校验逻辑

```bash
# (a) codex CLI 可用(用 --version 真实调,不只是 which)
if ! codex --version >/dev/null 2>&1; then
  # 不直接 abort,允许降级到 fallback 半自动 (mode=fallback_handoff,§3.1 详细规则)
  mode="fallback_handoff"
  # mode 变量 (M5 + M3/M4 接力 implementer 注意):
  #   传递到 §7 sub-loop L2 codex review 调用,值为:
  #     "auto"             → 主路径 codex CLI 调用 (M3 实现)
  #     "fallback_handoff" → handoff 文件 + 轮询 reviews/ (M5 §3.1 实现)
  warn "codex CLI 不可用,切到半自动 handoff 模式 (sub-loop L2 写 handoff/<id>.md 轮询)"
else
  # 版本号校验: ≥ 0.130 (用数字比较,glob 模式有 false-positive 风险)
  codex_version=$(codex --version 2>/dev/null | awk '{print $NF}')
  if [ -z "$codex_version" ]; then
    warn "codex --version 输出无法解析版本号"
  else
    major=$(echo "$codex_version" | cut -d. -f1)
    minor=$(echo "$codex_version" | cut -d. -f2)
    if [ "$major" -eq 0 ] && [ "$minor" -lt 130 ]; then
      warn "codex 版本 $codex_version 低于 0.130,review 子命令可能不可用"
    fi
  fi
  mode="auto"
  # mode 见上方说明
fi

# (b) chrome-devtools MCP 已装
if ! claude mcp list 2>&1 | grep -q "chrome-devtools"; then
  abort "chrome-devtools MCP 未装,跑: claude mcp add --scope user chrome-devtools -- npx -y chrome-devtools-mcp@latest"
fi

# (c) detekt config 加载(供 v1 §2.3 detekt 规避 + L7 验证用)
if [ ! -f config/detekt/detekt.yml ]; then
  abort "找不到 config/detekt/detekt.yml(必须在 repo 根目录运行 Skill)"
fi
```

**注意**: `abort` / `warn` 是给 Skill 执行者 CC 看的伪代码指令,意为"输出错误信息并退出" / "输出警告后继续"。

### 3.1 Fallback: codex CLI 不可用

当 §3 P-1 校验 `codex --version` 失败 → `mode="fallback_handoff"`(已在 §3 设),Skill **不 abort**,改走半自动 handoff 流程。

```
sub-loop §7.2 L2 调用改写为 (mode==fallback_handoff 分支):

a) 写 handoff 文件
   HANDOFF=.cc_tmp/crash/handoff/<c.id>-r$N.md
   cp $REVIEW_INPUT $HANDOFF   # review-prompt.md + 本崩溃上下文已构造好

b) 提示用户去 Codex 端跑
   echo "✋ codex CLI 不可用,请在 Codex 端打开 repo,跑:"
   echo "   /review-crash-fixes <c.id>"
   echo "把结果保存到: .cc_tmp/crash/reviews/<c.id>-r$N.md"

c) 轮询 reviews/ (沿用 §5.2 P2.fallback.2 分批模式)
   REVIEW=.cc_tmp/crash/reviews/<c.id>-r$N.md
   每批 CC Bash 调用内 sleep 10 * 6 = 60 秒后:
     [ -f "$REVIEW" ] && mtime > $START_TS → 退出轮询,继续 §7.3 parse
   一批未检测到 → CC 主循环判断 (date +%s) - START_TS >= 300:
     未到 5 分钟 → 再发起一批
     到 5 分钟 → AskUserQuestion "继续等 5 分钟 / 跳过本崩溃 / 取消 Skill"
   用户跳过 → continue 下一条;取消 → Skill 退出
```

**为什么不直接报错退出**: codex 可能临时不可用(更新中 / 网络问题 / 没 login),但用户可以在另一台机/另一个工具里跑 review 把结果文件丢回来,Skill 继续接力。设计上不让一个工具的临时故障阻塞整个修复流程。

## 4. 分支管理 (P1 + P1.5)

### 4.1 P1 切分支 + merge (沿用 v1 crash-analyzer §3 Step P1)

```bash
# 前置: working tree 干净(.gitignore 范围外无 tracked 改动)
# 检测 staged + unstaged 全部 tracked 改动 (排除 ?? untracked 和 !! ignored)
if [ -n "$(git status --porcelain | grep -vE '^(\?\?|\!\!)')" ]; then
  abort "working tree 不干净(含 tracked 改动),先 stash/commit"
fi

git fetch origin || warn "git fetch 失败,使用本地 origin/master"

if git show-ref --verify --quiet refs/heads/auto_fix_bug; then
  git checkout auto_fix_bug
elif git show-ref --verify --quiet refs/remotes/origin/auto_fix_bug; then
  git checkout -b auto_fix_bug origin/auto_fix_bug
else
  git checkout -b auto_fix_bug origin/master
fi

# merge origin/master,冲突 → abort 让用户手动 resolve
# --no-edit 防 non-fast-forward 时 git 打开 $EDITOR 在 CC Bash 工具下 hang
git merge --no-edit origin/master || abort "merge 冲突,手动 resolve 后重跑"
```

### 4.2 P1.5 upstream 绑定 (v2 新增,解决"新分支 push 错"问题)

```bash
# Note: 在 detached-HEAD 时此命令也 exit 0 (输出 "HEAD"),会跳过 push -u 绑定。
# §4.1 已保证我们在 named branch (auto_fix_bug) 上,所以这里不会 detached;
# 若未来 §4.1 变更,需要在此处加 named-branch 守卫。
if ! git rev-parse --abbrev-ref --symbolic-full-name @{upstream} >/dev/null 2>&1; then
  # 首次创建分支,无 upstream → push -u 绑定
  git push -u origin auto_fix_bug
fi
# 已绑则跳过
```

**为什么必须先绑定**: 后续每条 commit 完成后跑 `git push origin auto_fix_bug` 才不会出现"推到 dev / 推不出去 / a 分支推到 b"问题。绑定是一次性动作,后续跑跳过。

## 5. 友盟自动抓取 (P2)

### 5.1 主路径: chrome-devtools evaluate_script 主动 fetch `/hsf/analysis/errorList`

**前提**: 用户已在系统 Chrome 打开友盟后台并选好"车来了-android"项目,2FA 一次性完成。
**已知 URL 形态**: `https://apm.umeng.com/platform/<24位 hex appkey>/error_analysis/crash` (appkey 在 path 段,不在 query)

> **实测固化** (2026-05-15 端到端跑通发现):
> - 友盟 SPA 的版本/日期筛选**不响应 URL query** — 必须 POST `/hsf/analysis/errorList` 显式传 body
> - 主路径用 `evaluate_script` 在页面内**主动 fetch** 取数,**比被动拦截 XHR 稳**(无须等触发,无须 navigate 改变 URL)
> - API endpoint 固定: `POST /hsf/analysis/errorList`(车来了-android 实测确认)

```
P2.0a START_TS=$(date +%s)   # fallback.2 轮询窗口计时基准

P2.0b 读 master 当前版本号 (筛选友盟"当前最新版本"用)
      CURRENT_VERSION=$(git show master:build.gradle | awk -F'"' '/defaultVersionName/ {print $2}')
      [ -z "$CURRENT_VERSION" ] && abort "找不到 master 上的 defaultVersionName"
      # 例: CURRENT_VERSION="7.3.2"

P2.0c 算 last 7 days 日期窗口 (POST body 用):
      START_DAY="$(date -v-6d +%Y%m%d) 000000"   # macOS;Linux 用 date -d "6 days ago"
      END_DAY="$(date +%Y%m%d) 235959"

P2.1 MCP list_pages → 找匹配 "apm\\.umeng\\.com/platform/[a-f0-9]{24}" 的 page
     若无 → 提示"请按 §3 onboarding 启 Chrome (--remote-debugging-port=9222 + 独立 user-data-dir) 并登录友盟"
     暂停 60 秒等用户,超时 abort
     从匹配的 URL path 段提取 appkey (正则 `platform/([a-f0-9]{24})/`),记为 PAGE_APPKEY

P2.2 读 .cc_tmp/crash/umeng-recipe.yaml
     若文件不存在 → 进 P2.2.1 录制 (轻量,只录 appkey)
     若文件存在:
       校验 recipe.project_appkey 不含字面量 "${" (示例文件未录制的标志)
       含 "${" → 进 P2.2.1 录制
       不含 "${" + 与 PAGE_APPKEY 一致 → 进 P2.3 复用
       不一致 → 提示"当前页 appkey 与 recipe 不匹配,请切回正确项目"

P2.2.1 录制 (轻量版,API 已固化):
     a) 把 recipe-example.yaml 复制到 .cc_tmp/crash/umeng-recipe.yaml
     b) sed 替换 ${appkey} → PAGE_APPKEY (实际值)
     c) AskUserQuestion: "记录的 appkey=$PAGE_APPKEY 对应 '车来了-android' 项目?(yes/no)"
     d) yes → 保存;no → abort 让用户确认浏览器选中的项目

P2.3 复用模式 (主动 fetch,不依赖被动 XHR 拦截):
     a) MCP evaluate_script 跑下面的 JS,fetch /hsf/analysis/errorList:

       ```js
       async ({appkey, version, startDay, endDay}) => {
         const body = {
           dataSourceId: appkey, appType: "app", type: "app",
           errorAbstract: "", version: [version], crashType: "JAVA",
           pageSize: 100, page: 1,
           status: "", order: "desc", orderBy: "happenTimes",
           startDay, endDay,
           dateType: "last7days", errorType: "crash",
           summaryMd5: "",
           onlyNew: false, onlyOom: false,
           hasLogcat: false, noSystemError: false,
           onlyHarmonyOS: false, hasNativePage: false,
           nativePages: [], sdkTypes: [], thirdSdks: [],
           appStatus: [], customTags: []
         };
         const resp = await fetch("/hsf/analysis/errorList", {
           method: "POST", credentials: "include",
           headers: {"content-type": "application/json"},
           body: JSON.stringify(body)
         });
         return await resp.json();
       }
       ```
       args = {appkey: PAGE_APPKEY, version: CURRENT_VERSION, startDay: START_DAY, endDay: END_DAY}

     b) 校验响应:
       - resp.code == 200 → 继续
       - resp.code != 200 (例如 401/403 → 登录过期, 400 → 参数错) → 落 P2.fallback
       - 不是合法 JSON (拿到登录页 HTML) → 提示"登录过期,请在 Chrome 重新登录" + 暂停等用户

     c) parse resp.data.list (默认 pageSize=100,total > 100 时翻页):
       - 第 1 页 list_len 满 100 → fetch page=2/3... 直到 list_len < 100
     d) 按 field_map 转 CSV (11 列对齐 v1)
     e) 写 .cc_tmp/crash/inbox/auto-$(date +%Y%m%d-%H%M%S).csv

P2.3.csv_schema (与 v1 parsing.md §1 对齐,11 列):
     1. 错误摘要 ← it.summary (第一行=异常类+message;第二行起=单帧栈,完整保留)
        ⚠️ v1 §3/§4 决策依赖此列
     2. 错误ID ← it.id (13 位长整型)
     3. 错误类型 ← {JAVA: "JAVA崩溃", Native: "Native崩溃", ANR: "ANR"}[it.crashType]
     4. 最近一次发生时间 ← it.lastHappenTime
     5. 版本范围 ← it.appVersion (例 "4.72.10 - 7.3.4" 或单版本 "7.3.2")
     6. 错误次数 ← it.happenTimes
     7. 影响用户数 ← it.affectUsers
     8. 状态 ← {0:"未处理", 1:"已修复", 2:"处理中", 3:"已忽略"}[it.status]
     9. 处理人 ← it.processors[].name 拼接 (无则空)
     10. 标签 ← it.tags 拼接 (无则空)
     11. 链接 ← "https://apm.umeng.com/platform/${appkey}/crash/detail/${it.id}"

P2.4 Skill 后续流程 (§6 起) 把这个 CSV 当成 v1 inbox 同等输入处理
```

### 5.2 Fallback 链 (XHR 抓不到 → DOM scrape → v1 CSV inbox)

**P2.fallback.1** fetch 失败 (resp.code != 200 / 非 JSON 响应 / 网络错):

- DOM scrape: MCP evaluate_script 跑选择器:

  ```js
  Array.from(document.querySelectorAll('<recipe.dom.row_selector>'))
    .map(row => ({
      错误摘要: row.querySelector('<recipe.dom.summary_selector>').textContent.trim(),
      链接: row.querySelector('a').href,
      // 其他 9 列字段按需补 selector,无值列留空字符串
    }))
  ```

- 转 CSV 同主路径(11 列对齐 v1 schema),写 inbox/auto-<ts>.csv
- 若 recipe.dom 未配 → AskUserQuestion 让用户提供 CSS selector,或跳 P2.fallback.2

**P2.fallback.2** DOM scrape 也失败:

- 提示"自动抓取失败,请手动从友盟导出 CSV 放到 .cc_tmp/crash/inbox/"
- Skill 进入等待模式 (CC Bash 默认 timeout 120s,所以分批轮询):
  * 每批 CC Bash 调用内 sleep 10 * 6 = 60 秒,然后 ls .cc_tmp/crash/inbox/*.csv
    筛 mtime > $START_TS 的新文件
  * 检测到新 CSV → 退出轮询,继续 §6 解析
  * 一批 (60s) 未检测到 → CC 主循环判断 `(date +%s) - START_TS >= 300`
    未到 5 分钟 → 再发起一批
    到 5 分钟 → AskUserQuestion "继续等 5 分钟 / 取消跑"
- 用户取消 → Skill 退出,不动 git

**P2.fallback.3** chrome-devtools MCP 整体不可用:

(P-1 abort 保护;若 §3 改为允许降级,此处需补兜底逻辑)
- 同 P2.fallback.2,直接退 v1 CSV inbox 模式

**Fallback 决策原则**: 任何 fallback 都**不静默降级**,必须在 §8.2 汇总报告里写"本次跑用了 fallback X,主路径失败原因 Y"。

## 6. 解析/分类/git log 关联/代码现状校验 (P3-P5)

> ⚠️ **此节直接复用 v1 `crash-analyzer` Skill 的相应章节,Skill 执行时按引用顺序跑,不复述。**

执行时 Read `~/.claude/skills/crash-analyzer/SKILL.md` 并按以下顺序跑(节号以章节内容/职责定位,不以编号定位 — 防 v1 重编号失联):

| v2 Step | v1 对应章节(按职责) | 说明 |
|---|---|---|
| P3 解析 CSV | v1 §3 Step 2 + `parsing.md` 11 列 schema | UTF-8/GBK 回退 + 列名 fuzzy + 链接 ID 提取 |
| P3.1 初步分类 | v1 §3 Step 3 + §4 决策树 (D→C→B→A→E) | 按异常类型 + 栈顶 FQCN 分桶 |
| P4 git log 关联 | v1 §3 Step 4 + git log 关联节 | 30 天窗口 + FQCN 匹配 |
| P5 代码现状校验 ⭐ | v1 §3 Step 5 + 代码现状校验节 | Read 栈顶 ±15 行 + Grep 已有修复 API + 升级 D 规则 3 |

**为什么不复述**:
1. DRY,v1 已产品验证过
2. v1 SKILL.md 是单一事实源,v2 改动只新增上下游(P-1/P1.5/P2/P7-L)
3. v1 章节标题(中文长名)清晰 + Skill 执行时 Read 全文,不以节号定位

**v2 不同点(关键)**:
- P5 完成后,**不生成"用户读"的汇总报告**(v1 Step 6),直接进 §7 自动闭环 sub-loop
- v2 接收的 inbox CSV 既可能来自 v1 手工放,也可能来自 v2 §5 自动抓取写入的 `auto-<ts>.csv` — 两者列 schema 完全一致(都对齐 parsing.md 11 列)

## 7. 自动闭环 sub-loop (L1-L10)

> 对每条 A/B 崩溃跑一次完整 sub-loop。E 类用户手挑后才进。

### 7.0 启动前: codex 输出格式探测 (首次跑录制)

```bash
RECIPE_PATH=.cc_tmp/crash/codex-output-recipe.yaml

if [ ! -f "$RECIPE_PATH" ]; then
  # 制造最小 diff (subshell 隔离 CWD,防 cd 漏出);count-only smoke test,不自动派生 regex
  PROBE_DIR=$(mktemp -d)
  (
    cd "$PROBE_DIR"
    git init -q probe && cd probe
    echo "# probe $(date)" > README.md && git add . && git commit -q -m "init"
    echo "# probe 2" >> README.md   # 制造 unstaged 改动

    # 跑 codex review,喂 review-prompt.md 作为 PROMPT
    PROMPT_FILE="$HOME/.claude/skills/crash-auto-loop/instructions/review-prompt.md"
    cat "$PROMPT_FILE" | codex review --uncommitted - > /tmp/codex-probe-output.md 2>/tmp/codex-probe-stderr.log
  )
  PROBE_RC=$?

  # 验证输出含 9 行 [A]-[I] 评级 (count-only,regex 权威定义在 recipe yaml + review-prompt.md 输出契约)
  vlines=$(grep -cE '^\[([A-I])\] (✅|⚠️|❌) (.+)$' /tmp/codex-probe-output.md)
  rm -rf "$PROBE_DIR"

  if [ "$vlines" -ge 9 ]; then
    # 格式符合契约 → 复制 example 作为 runtime recipe (recipe 是 static schema 不自动派生)
    cp "$HOME/.claude/skills/crash-auto-loop/recipes/codex-output-recipe-example.yaml" "$RECIPE_PATH"
  else
    abort "codex 输出格式不符合契约 ($vlines 行 < 9),需调整 review-prompt.md。probe 输出见 /tmp/codex-probe-output.md / stderr /tmp/codex-probe-stderr.log"
  fi
fi

# 加载 recipe 里的权威 regex (用 python yaml 解,比 sed 稳),后续 §7.3 用它解析评级表
VERDICT_REGEX=$(python3 -c "import yaml,sys; print(yaml.safe_load(open(sys.argv[1]))['verdict_line_regex'])" "$RECIPE_PATH")
```

### 7.1 L1: CC 写 proposal + 应用 diff (不 commit)

```
For 每条 A/B 崩溃 c (按 v1 §6.3 排序: 影响用户数降序):
  N=1
  PROPOSAL=.cc_tmp/crash/proposals/<c.id>-r$N.md
  REVIEW=.cc_tmp/crash/reviews/<c.id>-r$N.md

  Skill 内嵌 LLM (CC 自己) 按 v1 §6.3 模板生成 $PROPOSAL,内容含:
    - 现有代码摘要 (§6/P5 抓的 ±15 行)
    - 现有修复痕迹 (§6/P5 Grep 结果)
    - 推荐方案 diff (满足 v1 §2.1 最小改动量化判定)
    - 备选方案 (有时含 ⚠️ 重构方案)

  应用 diff 到 working tree (Edit / Write 工具按 $PROPOSAL 落地);
  不 git add / 不 git commit。
  git diff > /tmp/proposal-diff-<c.id>-r$N.patch  (留作备份)
```

### 7.2 L2: codex review (主路径) / handoff (fallback)

```bash
# 构造 review 输入: review-prompt.md + 本崩溃上下文
REVIEW_INPUT=/tmp/review-input-<c.id>-r$N.md
PROMPT_FILE="$HOME/.claude/skills/crash-auto-loop/instructions/review-prompt.md"
{
  cat "$PROMPT_FILE"
  echo ""
  echo "## 本次崩溃上下文"
  echo "异常类: <c.exception>"
  echo "栈顶 FQCN: <c.top_frame>"
  echo "影响用户数: <c.affected_users>"
  echo "友盟链接: <c.umeng_link>"
  echo "改动文件: <c.changed_files>"
  echo ""
  echo "## 待审 diff (unstaged 改动)"
  echo ""
  echo '```diff'
  git diff
  echo '```'
  echo ""
  echo "## 你要做的"
  echo "按上面 9 维度评估这个 diff,严格用输出契约 (\`[X] emoji 一句话\`) 输出 9 行 + ✋强制问题 + 💭建议。不要其他对话/确认。"
} > "$REVIEW_INPUT"

if [ "$mode" = "auto" ]; then
  # 主路径: codex exec (实测固化 — codex review --uncommitted 与 PROMPT 互斥,不能传自定义指令)
  # REVIEW_INPUT 必须含: review-prompt.md + 崩溃上下文 + git diff (直接 cat 进 prompt)
  # 因为 exec 不像 review 子命令会自动读 git diff,所以 diff 要显式塞进 prompt body
  codex exec --sandbox read-only --skip-git-repo-check - \
    < "$REVIEW_INPUT" \
    > "$REVIEW" 2>/tmp/codex-stderr-<c.id>-r$N.log
else
  # mode = "fallback_handoff": 走 §3.1 描述的 handoff 半自动流程
  HANDOFF=.cc_tmp/crash/handoff/<c.id>-r$N.md
  mkdir -p .cc_tmp/crash/handoff
  cp "$REVIEW_INPUT" "$HANDOFF"

  echo "✋ codex CLI 不可用,请在 Codex 端打开 repo,跑:"
  echo "   /review-crash-fixes <c.id>"
  echo "把结果保存到: $REVIEW"

  # 轮询 reviews/ (沿用 §3.1 + §5.2 P2.fallback.2 模式: 60s 一批,5min 超时)
  POLL_DEADLINE=$((START_TS + 300))
  while [ ! -f "$REVIEW" ] || [ ! "$(stat -f %m "$REVIEW" 2>/dev/null)" -gt "$START_TS" ]; do
    for i in 1 2 3 4 5 6; do
      [ -f "$REVIEW" ] && break 2
      sleep 10
    done
    if [ $(date +%s) -ge $POLL_DEADLINE ]; then
      # AskUserQuestion: 继续等 / 跳过 / 取消 Skill
      decision=$(AskUserQuestion "$HANDOFF 等待 5 分钟无 review,继续等 / 跳过本崩溃 / 取消 Skill?")
      [ "$decision" = "skip" ] && continue 2   # 跳到外层 for 循环下一条崩溃
      [ "$decision" = "abort" ] && abort "用户取消 Skill"
      POLL_DEADLINE=$((POLL_DEADLINE + 300))   # 续 5 分钟
    fi
  done
  # 检测到 $REVIEW 文件,继续 §7.3 parse
fi
```

### 7.3 L3: parse 9 维度评级

```bash
# 用 §7.0 加载的 VERDICT_REGEX (来自 recipe 权威定义),保持 prompt/recipe/grep 三处一致
VERDICTS=/tmp/verdicts-<c.id>-r$N.txt
grep -E "$VERDICT_REGEX" "$REVIEW" > "$VERDICTS"

vcount=$(wc -l < "$VERDICTS" | tr -d ' ')
nack_count=$(grep -c '❌' "$VERDICTS")
warn_count=$(grep -c '⚠️' "$VERDICTS")   # ⚠️ 不能叫 warn,会撞 warn 伪指令
pass_count=$(grep -c '✅' "$VERDICTS")

# 校验输出契约: 必须 9 行
if [ "$vcount" -lt 9 ]; then
  warn "codex 输出格式漂移: 只匹配到 $vcount 行 [A]-[I] 评级 (期望 9)"
  # 标记本次 review 不可靠,可由 §7.4 L4 决策时降级处理 (M4 接力)
fi
```

### 7.4 L4: 完整决策树 (CC 自主拍板,不再问用户)

```
if [ "$nack_count" -eq 0 ] && [ "$vcount" -ge 9 ]; then
  goto L7   # 全绿,进 §7.5-§7.7 (编译 + commit + push)
fi

if [ "$N" -ge 3 ]; then
  git reset --hard HEAD
  goto escalate "<c.id>" "3 轮后仍未全绿"
  # escalate 流程见 §8.1
fi

# 还有轮次 (N < 3),撤回当前 diff 准备 N+1 轮
git reset --hard HEAD

# CC 决策: 改代码 or 写 rebuttal
# 判定标准 (CC 自己想清楚,不再问用户):
#   - reviewer 指出"可验证的事实" → 改代码:
#       * NPE 风险 (Kotlin !! / 漏 null check)
#       * 资源未关 (Cursor/Stream/Bitmap/WebView/Listener)
#       * SDK_INT 未守
#       * detekt 撞规则
#       * commit message 缺关键信息
#   - reviewer 指出"业务上下文盲点" → rebuttal:
#       * 不知道现有代码已 wrap (调用方已守 null)
#       * 不知道项目特定行为 (例: 所有 BroadcastReceiver 都在主进程)
#       * 异常类全名 + 栈顶能证明 reviewer 误判

decision=$(CC analyze: 看 review 报告的 ❌ 维度,基于上述判定标准产 "fix_code" 或 "rebuttal")

if [ "$decision" = "rebuttal" ]; then
  # 写 rebuttal-r<N>.md (按 instructions/rebuttal-template.md)
  REBUTTAL=.cc_tmp/crash/rebuttals/<c.id>-r$N.md
  CC 按模板填充 $REBUTTAL,含:
    - 被驳回的维度 (A-I)
    - Codex 的原文理由 (逐字摘抄)
    - CC 的辩论依据 (类型 1: 现有代码守 / 类型 2: 调用方分析 / 类型 3: 业务上下文)

  # 重跑 codex review,把 rebuttal 附加到本崩溃上下文后面再喂
  REVIEW_INPUT_NEXT=/tmp/review-input-<c.id>-r$((N+1)).md
  cat /tmp/review-input-<c.id>-r$N.md $REBUTTAL > $REVIEW_INPUT_NEXT

  # 注意: rebuttal 阶段 working tree 已 reset,需重新应用上一轮的 diff 让 codex 看见同样 staged
  apply_diff_from_proposal "<c.id>" "$N"   # 重放上一轮 proposal diff

  N=$((N+1))
  REVIEW_INPUT=$REVIEW_INPUT_NEXT
  goto L2 with N
fi

if [ "$decision" = "fix_code" ]; then
  # CC 基于 reviewer 反馈重新出 proposal
  N=$((N+1))
  PROPOSAL=.cc_tmp/crash/proposals/<c.id>-r$N.md
  CC 生成新 diff 写入 $PROPOSAL,基于:
    - 上一轮 review 的 ❌ 维度反馈
    - 现有代码摘要 (§6/P5 已抓)
    - 上一轮 proposal-r$((N-1)).md 作为基线

  apply_diff $PROPOSAL
  goto L2 with N
fi
```

**关键约束**:
- 每次 L4 决策(改代码或 rebuttal)都消耗 1 轮,N+=1
- rebuttal 后如果仍 ❌ 且 N < 3,**不再 rebuttal 二次**,强制走 fix_code 路径(防 CC 反复狡辩)
- 实现细节: CC 记录本轮 decision 类型,下一轮 L4 如果上轮是 rebuttal 且仍 ❌ → 强制 fix_code

### 7.5 L7: 编译 + detekt 验证 (沿用 v1 sub-loop)

```bash
# module 推断: 改动文件路径首段;改 .java → JavaWithJavac;改 .kt → Kotlin;跨 module/root → fallback :cllAppRelease:compileReleaseKotlin
changed_files=$(git diff --name-only)
module=$(echo "$changed_files" | head -1 | cut -d/ -f1)
[ -z "$module" ] || [ ! -d "$module" ] && module="cllAppRelease"  # fallback

# 判断 compile task: 全是 .java → JavaWithJavac;含 .kt → Kotlin
if echo "$changed_files" | grep -qE '\.kt$'; then
  compile_task="compileReleaseKotlin"
else
  compile_task="compileReleaseJavaWithJavac"
fi

./gradlew ":$module:$compile_task" 2>/tmp/compile-<c.id>-r$N.log
compile_rc=$?

if [ $compile_rc -ne 0 ]; then
  git reset --hard HEAD
  mark_failed "<c.id>" "compile failed: $(tail -3 /tmp/compile-<c.id>-r$N.log | tr '\n' ' ')"
  continue   # 跳下一条崩溃
fi

# detekt 验证
./gradlew ":$module:detekt" 2>/tmp/detekt-<c.id>-r$N.log
detekt_rc=$?

if [ $detekt_rc -ne 0 ]; then
  git reset --hard HEAD
  mark_failed "<c.id>" "detekt failed: $(grep -E 'rule|MaxLineLength|LongMethod' /tmp/detekt-<c.id>-r$N.log | head -3 | tr '\n' ' ')"
  continue
fi
```

### 7.6 L8: commit (沿用 v1 §6.5 模板 + [auto-loop r<N>] tag)

```bash
# commit message 由 CC 按 v1 §6.5 模板生成,加 [auto-loop r<N>] 标识本次迭代轮数
# ⚠️ 维度进 commit body 作 known follow-up

VERSION="<c.version>"   # 用户在 F2 (沿用 v1 §6.2) 指定的整批版本号
# 收集本轮所有 ⚠️ 维度的 follow-up
warn_followups=$(grep '⚠️' "$VERDICTS" | sed 's/^/- /')

commit_msg=$(cat <<EOF
[${VERSION}][fix bug][auto-loop r${N}] <c.exception 一句话>:<具体描述>

崩溃数据: <c.affected_users> 用户 / 友盟 <c.umeng_link>

[补充] codex r${N} review 9 维度通过,以下维度有 ⚠️ 作 follow-up:
${warn_followups}

修复机制: <简述>

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>
EOF
)

git add ${changed_files}
git commit -m "$commit_msg"
commit_sha=$(git rev-parse HEAD)
```

### 7.7 L9: push 三道防呆 (沿用 §2.4 push 安全红线)

```bash
# 防呆 1: 当前 branch == auto_fix_bug
current_branch=$(git rev-parse --abbrev-ref HEAD)
if [ "$current_branch" != "auto_fix_bug" ]; then
  abort "L9 防呆 1 fail: HEAD 不在 auto_fix_bug (当前: $current_branch)"
fi

# 防呆 2: branch.auto_fix_bug.remote == origin
branch_remote=$(git config --get branch.auto_fix_bug.remote 2>/dev/null)
if [ "$branch_remote" != "origin" ]; then
  abort "L9 防呆 2 fail: auto_fix_bug 未绑 origin (当前: $branch_remote);需 §4.2 重新绑 push -u"
fi

# 防呆 3: 直接显式 push 命令,**不要用变量**(shell 把含空格的变量当单命令名,实测会 exit 127)
# 命令本身硬编码,自证不含 --force / +refs

# 执行 push,失败永不重试,直接 escalate
git push origin auto_fix_bug 2>/tmp/push-<c.id>-r$N.stderr
push_rc=$?

if [ $push_rc -ne 0 ]; then
  push_err=$(cat /tmp/push-<c.id>-r$N.stderr)
  # 不 git reset HEAD~1 (commit 已存在保留),只是没推上去
  escalate "<c.id>" "push 失败 (commit $commit_sha 已落地但未推):\n$push_err"
  # 不 continue,但要让用户决策;M5 escalate 流程接力 (是否继续下一条由用户/Skill 配置决定)
fi

mark_success "<c.id>" "commit $commit_sha pushed (r$N)"
```

## 8. 终态汇总 + 收尾 (F5-F7)

### 8.1 Escalate 文件格式

3 轮后仍未全绿 / push 失败 / 编译失败等场景触发 escalate。Skill 把上下文写到 `.cc_tmp/crash/escalate/<c.id>.md`,**不 commit 该条**,跳下一条崩溃。

文件格式:

```markdown
# Escalate: <c.id> <c.exception>

**触发原因**: <"3 轮未全绿" / "push 失败" / "compile failed" / "detekt failed">
**轮数**: N (3 上限 / 中途失败时为实际轮数)
**最后一轮 ❌ 维度** (若适用): <A-I 列表 + 一句话原因>
**对应崩溃**: <c.affected_users> 用户 / [友盟链接](<c.umeng_link>)

## 完整轮次回放

### Round 1
- Proposal: `.cc_tmp/crash/proposals/<c.id>-r1.md`
- Review: `.cc_tmp/crash/reviews/<c.id>-r1.md`
- (无 rebuttal,N=2 起才有)

### Round 2 (若到此)
- Proposal: ...
- Review: ...
- Rebuttal (若 L4 选 rebuttal): `.cc_tmp/crash/rebuttals/<c.id>-r1.md`

### Round 3 (若到此)
- 同上

## 失败上下文 (若 push/compile/detekt 触发)

- compile log: `/tmp/compile-<c.id>-r<N>.log` 摘要
- detekt log: `/tmp/detekt-<c.id>-r<N>.log` 摘要
- push stderr: `/tmp/push-<c.id>-r<N>.stderr` 摘要

## CC 最后的分析

<CC 写一段:
- 为什么我认为应该过但 codex 不松口 (3 轮场景)
- 为什么编译/detekt 失败(改动撞了什么规则)
- 为什么 push 被拒(可能上游 diverged,需 user rebase)>

## 用户决策选项

- (a) 放弃这条修复,跳过
- (b) 接受当前 proposal r<N>,**用户手动**:
  - 跑 ./gradlew :module:compileReleaseKotlin 自己解决 compile/detekt
  - 跑 git push origin auto_fix_bug 自己 push
  - Skill 不动手
- (c) 转人工修复:Skill 退出该崩溃,user 自己处理
```

### 8.2 终态汇总报告 (F5)

跑完所有 A/B 崩溃后(含 escalate / 跳过),Skill 把以下内容写到 `.cc_tmp/crash/reports/auto-<YYYY-MM-DD-HHMM>.md`:

```markdown
# Auto-Fix 跑结果 <date>

**Skill**: crash-auto-loop v2
**分支**: auto_fix_bug
**HEAD**: <short sha>
**远端**: origin/auto_fix_bug (push 状态: success / partial / failed)
**版本号**: <CURRENT_VERSION> (从 master:build.gradle 读)
**项目**: 车来了-android (appkey: <PAGE_APPKEY 前 8 位>...)

## 总览

| 类型 | 计数 |
|---|---|
| 自动抓崩溃总数 | N |
| 进入 sub-loop (A 类 + B 类 + 手挑 E) | M |
| 全绿 commit + push | K |
| 编译/detekt 失败跳过 | F1 |
| 3 轮 escalate | E1 |
| push 失败 escalate | E2 |
| 用户跳过 / 取消 | S |

## 成功 commit (K 条)

| crash_id | 异常类 | 影响用户 | commit sha | 迭代轮数 | review 主路径 |
|---|---|---|---|---|---|
| ... | ... | ... | ... | r1/r2/r3 | codex CLI / handoff |

## 失败跳过 (F1 条)

| crash_id | 异常类 | 失败原因 |
|---|---|---|
| ... | ... | compile / detekt + 一句话摘要 |

## Escalate 待人工 (E1 + E2 条)

| crash_id | 异常类 | escalate 文件 | 触发原因 |
|---|---|---|---|
| ... | ... | .cc_tmp/crash/escalate/<id>.md | 3轮未全绿 / push fail / ... |

## 数据来源

- 友盟抓取主路径: <chrome-devtools XHR / DOM scrape fallback / 用户手放 CSV>
- inbox 文件: <.cc_tmp/crash/inbox/auto-<ts>.csv>
- 总数据条数: N
- 数据时间窗: 最近 7 天 + 版本=<CURRENT_VERSION>

## codex 调用统计

- 模式: <auto / fallback_handoff>
- 总调用次数: <X>
- 平均每条崩溃迭代轮数: <Y> (理论 1-3)
- codex 输出格式 parse 失败次数: <Z> (期望 0)
```

### 8.3 工作流偏离反馈 (F6)

> 沿用 v1 `crash-analyzer` Skill §F6 流程,本节不复述,Skill 执行时按 v1 §F6 章节内容跑。

v2 新增检查项(在 v1 §F6 列表基础上追加):
- 是否走过 §5.2 任一 fallback(XHR 失败 / DOM scrape / 手放 CSV)
- 是否走过 §3.1 codex 不可用 handoff 模式
- 是否有 codex 输出格式 parse 失败(§7.3 vcount < 9)
- 是否有 3 轮 escalate 数 ≥ 2(可能 review-prompt 需调强)
- 是否有 push 失败需要人工 rebase

把偏离列表追加到 §8.2 报告末尾,询问用户是否固化到 SKILL.md / spec(同 v1 §F6 三选项: 全部固化 / 部分固化 / 仅留记录)。

### 8.4 双端 SKILL 同步 (F7)

> 沿用 v1 `crash-analyzer` Skill §F7,只在 §8.3 改了 SKILL.md / instructions / recipes 时触发。

```bash
# v1 §F7 sync 命令对 crash-auto-loop 的实例化
rsync -a --delete ~/.claude/skills/crash-auto-loop/ ~/.codex/skills/crash-auto-loop/
```

同步范围: SKILL.md + instructions/{review-prompt.md, rebuttal-template.md} + recipes/{umeng-recipe-example.yaml, codex-output-recipe-example.yaml}

**为什么用 rsync --delete**: 避免 codex 端有过时文件残留(例如 instructions/ 加新文件后,旧机器同步时清理)。
