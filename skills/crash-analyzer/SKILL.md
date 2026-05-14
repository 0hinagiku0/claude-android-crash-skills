---
name: crash-analyzer
description: 用户输入 /analyze-crashes 时触发。从 .cc_tmp/crash/inbox/ 读取最新友盟 U-APM 崩溃 CSV,自动分类(A 待修代码/B 待决策/C 不归我家管/D 已修/E 未分类),在 .cc_tmp/crash/reports/ 生成 Markdown 报告,然后进入交互式修复阶段:切到 auto_fix_bug 分支并 merge origin/master → 逐崩溃让用户挑方案 → Claude 改代码 → 编译+detekt 验证 → 用户确认后 commit。Skill 永不主动 push 也不动 master/dev 分支。
---

# Crash Analyzer Skill

> 设计 spec: `docs/superpowers/specs/2026-05-12-crash-analyzer-skill-design.md`
> 参考文件: 同目录 `config.yaml` / `parsing.md` / `report-template.md`

## 1. 何时触发

用户在 Claude Code 中输入 `/analyze-crashes`(不带参数)。

## 2. 🚨 三条最高优先级原则(强制遵守)

**这三条是 Skill 行为契约,贯穿报告、sub-loop、commit message 全流程。**

### 2.1 【最小改动优先 ⭐】

修复方案的**默认推荐**永远是改动最小、不引发新崩溃、不重构代码的那一种。

**量化判定**(同时满足才算"最小改动",可打 ⭐):

- ✅ 改动文件数 ≤ 3
- ✅ 改动行数 ≤ 30
- ✅ 不新增 public API(公开方法签名 / 类)
- ✅ 不改类继承结构(不抽基类、不改接口)
- ✅ 不动数据库 schema / proto 定义
- ✅ 不改线程模型(同步↔异步)

任一不满足 → 自动归"重构方案"桶。

### 2.2 【重构方案要标记并由用户决定 ⚠️】

任一最小改动判定不通过的方案,**必须**打 ⚠️ 标记"需用户判断是否接受改动范围",**不能作为默认推荐**。是否走重构由用户决定,Skill 不替用户决定。

用户主动选 ⚠️ 方案时,commit message 加 `[重构]` tag(见 §6.5)。

### 2.3 【detekt 兼容】

所有自动生成的代码改动**必须通过项目 detekt 检查**。

- 配置: `config/detekt/detekt.yml`(项目根)
- 各模块 baseline: `<module>/detekt-baseline.xml`
- 命令: `./gradlew :<module>:detekt`

**Skill 启动时 Step 0**: Read `config/detekt/detekt.yml` 提取启用的规则清单,作为本次跑的"规避要点清单"。**不要把规则清单写死在本 SKILL.md 里**,避免与项目实际配置漂移。

修复后**必须跑 detekt 验证通过**才能 commit。

## 3. 完整流程图

### 分析阶段(10 步,**必须按顺序跑**)

```
Step P0: 前置守卫 - working tree 干净检查
         git status → 若 tracked 文件有改动: 中止,提示 stash/commit 后重跑
         (仅 .gitignore 范围内的 untracked 不算 dirty)

Step P1: 分支管理(从修复阶段提前到此 — 必须先切分支才出方案)
         git fetch origin (失败 → 警告 + 显示本地 origin/master 上次更新时间,默认中止)
         if 本地有 auto_fix_bug:
             git checkout auto_fix_bug
         else if 远程有 origin/auto_fix_bug:
             git checkout -b auto_fix_bug origin/auto_fix_bug
         else:
             git checkout -b auto_fix_bug origin/master
         git merge origin/master
             ├─ fast-forward → 直接前进
             ├─ 产生 merge commit → 允许
             └─ 冲突 → 中止 Skill,提示用户手动 resolve
         >>> 切到 auto_fix_bug + master 最新代码,后续所有方案都基于此代码状态生成

Step P2: Read config/detekt/detekt.yml → 抽取启用规则,作为本次 detekt 规避要点

Step 1: ls .cc_tmp/crash/inbox/*.csv → 找 mtime 最新一份
        若目录不存在或无 CSV → 提示用户先放 CSV,Skill 退出

Step 2: 解析 CSV(详见 parsing.md)
        - UTF-8 → 失败 fallback GBK
        - 按列序读 + 列名 fuzzy 校验(漂移检测记到报告头)
        - 错误 ID 从「链接」列正则 `crash/detail/(\d+)` 提取
        - 栈顶帧 FQCN = `at <FQCN>.<method>` 那一行的 FQCN

Step 3: 初步分类(详见 §4 决策树)
        - JAVA 走 D→C→B→A→E
        - Native 直接 C
        - ANR 按栈顶关键词 A/B
        - OOM 独立桶进 B

Step 4: git log 关联(详见 §5)
        - git log --since="30 days ago" --pretty=format:"%h|%s|%b"
        - 用 FQCN/异常类全名做匹配(不用短词)
        - 命中 → 打"疑似已修在 <短sha>"

Step 5: 【关键】代码现状校验 + 方案上下文收集(详见 §5.5)
        对每条 A/B/E 候选崩溃:
          a) Read 栈顶文件:行号 周边 ±15 行,作为方案上下文
          b) Grep 项目里"该崩溃常见修复 API"是否已存在
             (例:WebView 多进程冲突 → 搜 setDataDirectorySuffix
              空指针 → 搜栈顶类是否已有 null check 或 lateinit init guard)
          c) 若发现已有修复 API 但崩溃仍上报:
             → 升级该条到 D 类(已修但仍上报),并在报告里写明
               "代码已有 X 实现于 path:line,需排查为何不生效"
        ⛔ 禁止凭空生成方案,所有 diff 必须 reference 真实文件:行号

Step 6: 生成报告 → .cc_tmp/crash/reports/YYYY-MM-DD.md(按 report-template.md)
        每条 A/B 必须含: 现有代码摘要 / 现有修复痕迹 / 增量改 diff

Step 7: 询问用户:"是否进入交互式修复阶段?(yes/no/just-report)"
        - no/just-report → Skill 在此结束 (auto_fix_bug 分支已切但无 commit)
        - yes → 进入"修复阶段"
```

### 修复阶段(分支已在分析阶段切好,无需重切)

```
Step F1: working tree 复核(分支管理已在 Step P1 完成)
         git status → 仍干净 → 继续
                    → 若用户在 Step 7 间隙偶然动了 tracked 文件 → 中止

Step F2: 问用户:"本次修复挂哪个版本号?"(整批用同一版本号)

Step F3: 排序 A 类 + B 类 + (用户手挑的 E 类)按影响用户数降序,进入 sub-loop

Step F4: 逐崩溃 sub-loop(详见 §6.4)
         For 每条崩溃:
           - 展示概要 + 方案列表(每方案带改动估算,⭐推荐方案必须满足量化判定)
           - 用户决策:选方案 N / 跳过 / 退出
           - 记录当前 HEAD sha 作为还原点
           - 应用代码改动
           - 跑验证:编译 + detekt + (可选)单测
             - 失败 → git reset --hard HEAD && git clean -fd → 标"修复失败,原因..."→ continue
           - 展示 diff
           - 问用户:commit / 调整 / 跳过
             - commit → git add + git commit -m "<§6.5 模板>"
             - 调整 → 用反馈作新 prompt,重新应用改动(最多 2 次重试)
             - 跳过 → reset + clean → continue

Step F5: 归档 + 收尾
         - 把 inbox 处理过的 CSV → .cc_tmp/crash/processed/<时间戳>-<原文件名>.csv
         - 在报告末尾追加"本批处理结果"段(用户报告模板末段)
         - 告知:auto_fix_bug 当前 HEAD,共 N 个新 commit
         - 问用户:"需要 push 吗?" — Skill 永不主动 push

Step F6: 工作流偏离反馈(关键 — 防止文档与实际操作脱节)
         检查本次跑中是否有"实际操作偏离 SKILL.md 设计约束"的地方,典型包括:
           - 用了 --no-verify 跳过 pre-commit hook(SKILL.md §8.5 原本要求 detekt 必须通过)
           - 跳过了某些 B/E 类候选未深 Read
           - 改了 config.yaml 但没同步 ~/.codex 端
           - Step 5 代码现状校验对某些条目 fallback 用了通用思路(行号 r8-map-id 解不出)
           - 任何 SKILL.md 没明文规定但本次跑遇到并临时决策的场景
         把偏离列表 + 改进建议追加到报告末尾"工作流偏离反馈"段
         询问用户:"这些偏离要不要固化到 SKILL.md/spec 作为下次的设计依据?"
           - yes 全部固化 → Skill 提议改 SKILL.md/spec 的 diff,等用户确认后改
           - yes 部分固化 → 用户挑哪几条
           - no → 仅留报告里的偏离记录作为本次跑痕迹,SKILL.md 不动

Step F7: 双端同步(若 Step F6 改了 SKILL.md/config.yaml)
         cp ~/.claude/skills/crash-analyzer/{SKILL.md,config.yaml,parsing.md,report-template.md} \
            ~/.codex/skills/crash-analyzer/
         同步 _design-docs/ 里 spec 的修订
```

## 4. 分类决策树

按 D → C → B → A 顺序判定,先匹配先归类;都没命中 → E。

### 4.1 D 类:已修复 / 疑似已修

任一即可:
1. CSV「状态」列 = `已修复`
2. git log 最近 `git_log_window_days`(config) 天 commit message 匹配命中
3. **(关键新增) Step 5 代码现状校验发现现有代码已有该崩溃的修复 API/方法**
   - 例:WebView 多进程崩溃 → `git grep setDataDirectorySuffix` 找到 `MyApplication.fixWebView()` → 标该崩溃 D 类"已修但仍上报"
   - 例:NPE on `lateinit` → `git grep` 该字段周边发现已有 `isInitialized` 守卫 → 同上
   - 报告里写明"代码已有 X 实现于 path:line,需排查为何不生效"——这是**真正有价值的信号**,提示用户去查"为什么写了但没用"

匹配规则(规则 2): 用**完整 FQCN** 或**完整异常类名**在 commit subject + body 里 case-insensitive 查找;**不用短词**(`Adapter`/`onCreate` 必撞库)。

报告标注:
- `已修复(CSV)` (规则 1)
- `疑似已修(git <短sha>)` (规则 2)
- `已修但仍上报: 代码 path:line 已有 <API>,需排查` (规则 3)
- 多条命中 → 合并标注

**不进入修复阶段**(规则 3 例外:用户可手挑进修复阶段重新看为何不生效)。

### 4.2 C 类:不归我家管

任一即可:
- 「错误类型」= `Native崩溃`
- 栈顶帧 FQCN 匹配 `config.third_party_packages`
- 栈顶帧为 `<unknown>`

**自家代码回溯例外**(详见 parsing.md §6): 异常 message 字符串中包含自家代码 FQCN/类名 → 转 B 类"栈顶第三方但根因疑似自家"。

报告处理: 按"原因类别"聚合统计。**不进入修复阶段**。

### 4.3 B 类:自家代码 + 要业务决策

全部满足:
- 不在 D / C
- 栈顶帧 FQCN 匹配 `config.home_packages`
- 异常类型在 `config.business_decision_exceptions`
- 或来自 OOM 独立桶 / ANR 兜底

报告处理: 列出 2-3 个修复方案 + 改动估算 + tradeoff,**进入修复阶段时让用户挑**。

### 4.4 A 类:自家代码 + 修法明确

全部满足:
- 不在 D / C / B
- 栈顶帧 FQCN 匹配 `config.home_packages`
- 异常类型在 `config.known_fix_exceptions`
- 或来自 ANR IO/File/Database 等明确桶

报告处理: 详细模板,**默认推荐**最小改动 ⭐ 方案 + (可选)重构 ⚠️ 方案,**进入修复阶段时优先处理**。

### 4.5 E 类:未分类(兜底)

D/C/B/A 都没命中。两种典型情况:
1. 栈顶帧 FQCN 匹配 `home_packages`,但异常类型既不在 `known_fix_exceptions`(A) 也不在 `business_decision_exceptions`(B)
2. 栈顶帧 FQCN **既不在 `home_packages` 也不在 `third_party_packages`**(如华为推送 `com.huawei.*` 等 config 未收录的第三方 SDK)

报告处理: 列基本信息 + 栈,提示"是否补到 config.yaml"(若是情况 1 补 A/B 异常清单,若是情况 2 补 third_party_packages 黑名单)。**默认不进修复**,用户可在 sub-loop 开始前手挑。

## 5. git log 关联实现

调用:
```bash
git log --since="30 days ago" --pretty=format:"%h|%s|%b" -- .
```

- 工作目录:repo 根。**不包含 `.worktrees/*`**(Skill 在主工作树跑)
- 每行解析为 `(sha, subject, body)`
- 对每条崩溃,构造关键词集合:`{完整异常类名, 完整栈顶 FQCN, 栈顶方法名(若长度 ≥ 6)}`
- 在 subject + body 里 case-insensitive 查找,命中任一关键词 → 标"疑似已修"

成本: 纯本地文本匹配,不调 LLM,几乎零延迟。

**已知盲区**:若 commit message 用中文描述修复场景但**没列出栈顶类的 FQCN**,会漏匹配。例:commit 提"修 hideLive 后台启动服务崩溃"但栈顶类 `SomeServiceGateway` 没出现在 message 里 → 匹配不到。**§5.5 代码现状校验**正是来补这道盲区。

## 5.5 代码现状校验 + 方案上下文收集(分析阶段 Step 5,**关键**)

> 这是 Skill 工作流里**最重要**的一步。没有它,所有方案都是凭空生成,跟现有代码脱节。

### 5.5.1 触发对象

仅对 **A/B/E 候选崩溃**(自家代码白名单内的)执行;C 类不归我家管,跳过;D 类已经判定过,跳过。

### 5.5.2 必做动作(每条候选崩溃)

```
For 每条 A/B/E 候选崩溃:
  a) Read 栈顶 file:line 周边 ±15 行
     - 用 Read 工具读 `<repo>/<栈顶类相对路径>` 的对应行附近
     - 若行号是 r8-map-id-xxx(R8 混淆),Skill 没法直接定位 → 改用 Grep 搜栈顶类名找文件,
       Read 该文件全文(若 ≤ 300 行)或前 200 行
     - 把抓到的代码片段作为"现有代码摘要"放进报告该条目下
  b) Grep 项目搜"该崩溃常见修复 API"是否已存在
     - 异常类型 → 修复 API 关键词的映射示例:
       * WebView 多进程 / AwDataDirLock → `setDataDirectorySuffix`
       * BackgroundServiceStartNotAllowedException → `startForegroundService` / `WorkManager`
       * NullPointerException on lateinit → `isInitialized`
       * IllegalStateException on Fragment commit → `commitAllowingStateLoss` / `isStateSaved`
       * SecurityException + post-N notification → `setSmallIcon` / NotificationChannel
     - Grep 范围: 项目源码根(排除 build/, .worktrees/)
     - 命中 → 把"已有 API 的 path:line"作为"现有修复痕迹"放进该崩溃报告
  c) 决策升级
     - 若 Grep 命中且**位置就在该崩溃栈顶类附近或其调用链上层** → 升级到 D 类(规则 3)
     - 若 Grep 命中但位置在不相关代码 → 仅作为参考列在报告里(说明项目有此 API 经验,可借鉴)
     - 若 Grep 未命中 → 该崩溃确实是真问题,留在 A/B/E
  d) 方案生成约束
     - 所有方案 diff **必须 reference 真实文件:行号**,基于步骤 a) 抓到的代码上下文做"增量改"
     - **禁止凭空写**"建议加 X" 这种空话——X 是否已存在,Skill 已经在 b) 步检查过了
     - 若现有代码已经有类似实现但不生效,方案应该是"调整调用时机/守卫条件",而不是"重新加一份"

### 5.5.3 输出到报告

每条 A/B 崩溃在报告里**必须有**:
- `📂 现有代码摘要`: 栈顶 file:line 周边 ~10 行实际代码
- `🔍 现有修复痕迹`: Grep 结果(若有)或"无相关修复 API"
- `📐 推荐方案 diff`: 基于上面两段的增量改动,而非凭空生成

### 5.5.4 失败 fallback

| 场景 | 处理 |
|---|---|
| 栈顶行号是 R8 混淆 ID 且 Grep 类名找不到唯一文件 | 报告里标"现有代码定位失败,需 mapping.txt",方案部分仅给"通用思路",标 ⚠️ 需人工确认 |
| Read/Grep 超时 (>30s) | 跳过这一步,该崩溃留在 A/B/E 但 diff 标"⚠️ 未经代码上下文校验,可能与现状脱节" |
| 同一异常类下多条崩溃,代码上下文相似 | 合并展示,Step 5 只 Read 一次共享 |

## 6. 修复阶段细则

### 6.1 working tree 复核(对应 Step F1)

> ⚠️ 分支管理已在**分析阶段 Step P1** 完成(`auto_fix_bug` 已切 + `origin/master` 已 merge)。这里只复核 working tree 状态。

```
git status
  ├─ tracked 改动 → 中止,提示用户先 stash/commit
  │                 (用户在 Step 7 间隙偶然动了文件的兜底)
  └─ 仅 .gitignore 范围内 untracked → 继续
git branch --show-current
  └─ 必须是 auto_fix_bug,否则警告并询问用户是否要重新切(理论上不会发生)
```

### 6.2 询问版本号(对应 Step F2)

> "本次修复挂在哪个版本号下?(例如 7.4.0、7.5.0-beta)"

整批 commit 用同一版本号。

### 6.3 方案排序规则(贯穿报告 + sub-loop)

每个崩溃的方案列表**必须**按以下格式呈现:

```
方案 1 ⭐ 推荐(最小改动): <改法>
  - 改动估算: X 文件 / Y 行 / 新增 public API: yes/no / 改类继承: yes/no / 动 schema: yes/no
  - 副作用: <...>
  - ⚠️ 风险: <Skill 看不到的业务上下文>

方案 2 (中等改动): <改法>
  - 改动估算: <...>

方案 3 ⚠️ 重构方案(需用户判断是否接受改动范围): <改法>
  - 改动估算: <...>
  - 为什么列出: <为什么光改 try/catch 不能根治>
  - 不作为默认推荐
```

约束:
- 永远把"最小改动"放第 1 位 + ⭐
- ⭐ 方案的改动估算**必须实际满足** §2.1 全部量化判定,否则**不允许**打 ⭐
- 若无任何最小改动方案能修 → 第 1 位放最保守的但**不打 ⭐**,说明"未找到无重构的修法"
- 重构方案打 ⚠️,写明"为什么不能光靠最小改动"
- **(关键) 所有方案 diff 必须基于 §5.5 收集的"现有代码摘要 / 现有修复痕迹"生成,reference 真实 file:line。禁止凭空写"建议加 X" 之类——若 X 已存在,Step 5 会把该崩溃升级到 D 类规则 3;若用户问"已有 X 但还在崩",方案应该是"调时机/加守卫/换 API",不是"重新加一份"**

### 6.4 sub-loop 工具调用对照(对应 Step F4)

| 子步骤 | 工具 | 备注 |
|---|---|---|
| 展示概要 | 在对话中输出 | 不调工具 |
| 用户决策 | AskUserQuestion | 选项: 方案 1/2/.../ 跳过 / 退出 |
| 记录 HEAD | Bash `git rev-parse HEAD` | 作还原点 |
| 应用改动 | Edit / Write | 遵守 §2.1 量化 + §2.3 detekt 规避 |
| 验证编译 | Bash `./gradlew :<module>:compile<Variant>Kotlin` 或 `compile<Variant>JavaWithJavac` | module 推断: 改动文件路径首段;改 .java → JavaWithJavac;改 .kt → Kotlin;跨 module/root → fallback `:app:compileReleaseKotlin` |
| 验证 detekt | Bash `./gradlew :<module>:detekt` | 自动应用 baseline |
| (可选)单测 | Bash `./gradlew :<module>:testReleaseUnitTest` | 慢则跳过 |
| 撤回 | Bash `git reset --hard HEAD && git clean -fd` | 不带 `-x`,.cc_tmp 等 ignored 文件不受影响 |
| 展示 diff | Bash `git diff --staged` 或 `git diff` | 给用户看完整改动 |
| 用户确认 commit | AskUserQuestion | commit / 调整(回到应用改动) / 跳过 |
| 执行 commit | Bash `git add <files> && git commit -m "<§6.5>"` | message 用 heredoc 防转义 |

**最多 2 次"调整"重试**,仍不通过 → 强制跳过,标记"修复失败"。

### 6.5 commit message 模板

**最小改动方案**(默认):
```
[<版本号>][fix bug] <一句话总结>:<具体描述>
```

**重构方案**(用户选 ⚠️ 时):
```
[<版本号>][fix bug][重构] <一句话总结>:<具体描述>
```

**单 commit 内多个相关改动点**(同一根因涉及多文件协同),用 `1./2./3.` 编号,示例参见 git log `f28da392ac`。

**规则**:
- 每条崩溃**默认一个 commit**
- 单 commit 多改动点限同一根因
- 不同根因崩溃**不合并** commit
- 起 commit message 草稿 → 询问用户调整 → OK 后才 commit

### 6.6 归档与收尾(Step F5)

- inbox CSV → `.cc_tmp/crash/processed/<YYYY-MM-DD-HHmm>-<原文件名>.csv`
- 在已生成的报告 markdown 末尾追加"本批处理结果"段(用 report-template.md 末段格式)
- 报告内容: 总条数 / 成功 / 失败 / 跳过 / commit 列表 / 失败原因清单
- 告知用户: `auto_fix_bug` 分支当前 HEAD + 新增 commit 数
- **不主动 push**;问用户是否要 push,用户说 yes 才执行

## 7. detekt 规避要点(动态生成,不写死)

Skill 启动 Step 0 时 Read `config/detekt/detekt.yml`,**自动提取启用规则的名字** + 默认阈值,作为本次代码生成的"规避要点清单"。常见可能命中的(由实际配置决定):

- `LongMethod` / `LongParameterList` / `ComplexMethod`(尺寸 / 复杂度)
- `SwallowedException`(catch 空)
- `MagicNumber`、`ReturnCount`、`TooManyFunctions`、`MaxLineLength`
- `ForbiddenComment`(TODO/FIXME)

**代码生成约束**(无论 config 启用什么):
- 不写 `println` / `e.printStackTrace()` → 用项目现有日志工具
- 不用 `!!` 强解包 → 用 `?:` / `?.let {}` / 显式 throw
- catch 必须至少 log → 否则 `@Suppress("SwallowedException")` 并附原因注释(参考 git log `b0d3c529b0`)
- 函数 / 类规模超阈值,必要时 `@Suppress("LongMethod")` 并附设计原因
- 项目对"故意吞异常"的成熟做法: `@Suppress(...)` + 中文注释说明意图

**特殊情况**:detekt 失败但用户在 sub-loop "调整"中给了具体反馈,Claude 可根据 detekt 输出的具体规则项再次调整(最多 2 次)。

## 8. 数据安全

- `.cc_tmp/` 必须在 `.gitignore`(脚手架 Task 1 已确保):崩溃 CSV 可能含 user_id / device_id 等
- 报告也放 `.cc_tmp/crash/reports/`,**不进 git**;含友盟链接,链接背后含敏感字段
- 修复阶段产生的 commit 进 `auto_fix_bug` 分支,**只含代码改动,不含 CSV 数据**
- **Skill 永不主动 push**

## 9. 错误处理 fallback

| 场景 | 行为 |
|---|---|
| `.cc_tmp/crash/inbox/` 不存在或为空 | 提示用户先放 CSV,Skill 退出 |
| CSV UTF-8 解码失败 | 回退 GBK |
| CSV 表头列名漂移 | 警告,继续按列序解,在报告头部"列名漂移检测"标注 |
| 「链接」列缺失 ID | 该行标"ID 缺失",E 类用行号当临时标识 |
| `git fetch origin` 网络失败 | 警告 + 显示本地 origin/master 上次更新时间,默认中止;用户主动覆盖才继续 |
| `git merge` 冲突 | 中止 Skill,提示用户手动 resolve 后重跑 |
| working tree 不干净 | 中止"修复阶段"(报告已生成可先看),提示 stash / commit |
| 编译失败 | `git reset --hard HEAD && git clean -fd`,标"修复失败,编译错: <stderr 摘要>",continue 下一条 |
| detekt 失败 | 同上(`git reset --hard HEAD && git clean -fd`) + 在失败原因里附 detekt 规则名 |
| 用户中途退出 sub-loop | 已 commit 保留;未 commit 的 `git reset --hard HEAD && git clean -fd` 还原 |

## 10. 未尽事项(spec §11 摘要)

跑 2-4 周后,根据实际报告:

- 用户根据 E 类高频条目把异常类型补到 `config.known_fix_exceptions`(A 类)或 `business_decision_exceptions`(B 类)
- 第三方包名黑名单按实际遇到的 SDK 扩充
- 若 `r8-map-id-xxx` 出现频繁,单独立项做 mapping.txt 反混淆
- `auto_fix_bug` 分支累积过多 commit 后,人工治理(merge 到 dev / 删除重建)

## 11. 触发命令登记

- Slash command 名: `/analyze-crashes`(不带参数)
- 触发后 Claude 读本 SKILL.md 全文,按 §3 流程执行
