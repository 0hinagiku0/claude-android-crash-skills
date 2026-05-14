---
name: crash-fix-review
description: 用户输入 /review-crash-fixes 时触发。独立 review crash-analyzer Skill 在 auto_fix_bug 分支生成的崩溃修复 commit,按 9 维度评估:A 方案根因 / B 新崩溃风险(NPE/资源泄漏/死循环/线程安全) / C 最小改动 / D 验证状态(detekt + --no-verify) / E commit message / F 业务影响(调用方+业务流+多进程) / G 向后兼容 / H 合规与隐私(隐私同意守卫+敏感API时机+SDK合规) / I 错误处理+可观测性(@Suppress注释+失败可监控)。读 git diff + 当次报告,输出 review 报告到 .cc_tmp/crash/reviews/YYYY-MM-DD-auto_fix_bug.md,含跨commit全局/测试覆盖/回滚成本"整批观察"。Skill 仅审不改,所有修改建议交还用户决定。
---

# Crash Fix Review Skill

> 配套 skill: `crash-analyzer` (生成被 review 的 commit)
> 设计目标: 用独立 context 扮演 reviewer,避开 crash-analyzer 跑时的"自我辩护"偏见

## 1. 何时触发

用户在 Claude Code 中输入 `/review-crash-fixes` (不带参数,默认 review `auto_fix_bug` 分支)
或 `/review-crash-fixes <branch>` 指定其他分支

## 2. 🚨 review 行为契约

### 2.1 【独立 reviewer 立场】

本 skill **不** 应该:
- 复用 crash-analyzer 跑时给出的方案理由 — 那些理由是"作者立场",reviewer 要 challenge
- 假设 commit 一定是对的 — 永远从"会不会引入新问题"角度看

本 skill **应该**:
- Read 真实 git diff 全部内容
- Read 被改的文件**改动周边代码**(±20 行),理解上下文
- Grep 被改函数/字段的其他调用方,看改动是否影响别处
- 对每条都过下面 §3 的 9 个维度

### 2.2 【只审不改】

skill 不**主动 Edit / Write 任何代码**。所有改进建议**只写在 review 报告里**,由用户决定是否回头让 crash-analyzer 或人工动手。

### 2.3 【失败安全】

若 git/Read 命令失败 → 在报告里标"⚠️ 无法验证",**不要绕过**或假设。

## 3. Review 流程

### 分析阶段(9 步)

```
Step R1: 解析参数 + 验证分支存在
         默认 branch = auto_fix_bug
         git rev-parse --verify <branch>
           └─ 不存在 → 提示用户先跑 /analyze-crashes 后再来

Step R2: 列出本批待 review 的 commit
         git log origin/master..<branch> --oneline
         若 0 commit → 提示用户"无待 review 修复",skill 退出
         若 >20 commit → 警告"批次过大,review 可能耗时",问用户是否继续

Step R3: 找最新崩溃报告
         ls -t .cc_tmp/crash/reports/*.md | head -1
         若无 → 提示用户"找不到对应 crash-analyzer 报告,只能盲审 commit"

Step R4: 提取 commit ↔ 崩溃条目映射
         对每个 commit:
           - 读 commit message subject 找异常类全名 (如 SQLiteBlobTooBigException)
           - 在崩溃报告里 grep 该异常 + 栈顶 FQCN,找到对应崩溃记录
           - 抽取:错误 ID / 影响用户数 / 友盟链接 / 报告里给的方案描述

Step R5: 对每个 commit 逐条 review (§4 九维度)
         For 每个 commit:
           a) Read commit message + git show <sha> 完整 diff
           b) Read 被改文件 diff 周边 ±20 行
           c) Grep 被改函数/字段的其他调用方
           d) 过 9 个 review 维度(A-I),每个给评级(✅/⚠️/❌)+ 一句话理由
              新增维度 H 合规 / I 错误处理可观测,见 §4
           e) 给出 ✋ 强制问题清单(必须解决才能进主干)和 💭 建议清单(可选)

Step R5.5: 整批观察(跨 commit 视角,详见 §4 后"整批观察"段)
         - 跨 commit 全局影响: 本批 N 个 commit 合起来是否打破某个抽象/互相干扰?
         - 测试覆盖: 被改文件是否有现成单测?如有,跑一遍看是否覆盖 fix 场景
         - 回滚成本: 改动是否容易 git revert,还是已经传播到调用方需要连锁回滚?
         (这些不强制每 commit 评级,作为整批观察写在报告末尾)

Step R6: 生成 review 报告 → .cc_tmp/crash/reviews/YYYY-MM-DD-<branch>.md
         按 §5 模板

Step R7: 计算总评 + 决策建议
         - 任何 commit 触发 ❌ → 总评 "BLOCK,需修改后才能合主干"
         - 仅 ⚠️ → "可合主干但有改进点"
         - 全 ✅ → "可直接合主干"

Step R8: 询问用户是否要立刻回头让 crash-analyzer 处理 ❌/⚠️
         - yes → 用户决定哪些处理,自己跑 /analyze-crashes 重新走 sub-loop
         - no → review 报告留作记录,用户线下决策

Step R9: skill 结束,不动任何代码,不动 git
```

## 4. Review 九维度

每个 commit 必过 9 个维度,每维度给 ✅/⚠️/❌ + 一句话理由 + 必要时贴证据(grep 输出/file:line):

### A. 方案对应根因

- ✅ diff 改的代码确实是崩溃栈顶/调用链上层
- ⚠️ diff 是 catch 兜底/降级,缓解症状但没消除根因(对部分崩溃 OK,需明示)
- ❌ diff 跟崩溃栈无关,改了别处

### B. 新引入崩溃风险(regression — 4 类必查)

读 diff + 周边代码,**逐项明文检查**:

1. **空指针/Null 安全**:
   - 改动里新建的链式调用每一段是否有 null 可能?
   - 用了 `Application.getProcessName()` 这类返回 nullable 的 API,后续是否有 null 守卫?
   - 改动如果在 Kotlin 中用了 `!!`,**自动 ❌**

2. **资源泄漏**:
   - 改动里 open 的资源(Cursor / Stream / Reader / FileDescriptor / Bitmap.recycle / WebView.destroy)是否在 finally 块 close?
   - 改动里 BroadcastReceiver / Observer / Listener 是否注册了但没注销?
   - 改动里 Handler 持有 outer this(可能内存泄漏)?

3. **死循环/递归爆栈**:
   - 改动里新加的 catch + retry 是否有终止条件?
   - 递归调用是否有 base case?
   - while 循环条件是否能保证退出?

4. **线程安全**:
   - 改动里读写的字段是否在多线程访问?(看其他调用方在什么线程)
   - 加了同步还是没加?
   - volatile / AtomicXxx 用得对吗?
   - 主线程做了 IO/阻塞?

评级: ✅ 4 项全无问题 / ⚠️ 1-2 项需注意 / ❌ 任一项明显有问题

### C. 改动是否真"最小"

按 crash-analyzer SKILL.md §2.1 量化判定:
- 改动文件数 ≤ 3 ? 改动行数 ≤ 30 ?
- 不新增 public API / 不改类继承 / 不动 schema / 不改线程模型?
- crash-analyzer 报告里方案标了 ⭐ "最小改动",diff 实际是否满足?
- 评级: ✅ 满足 ⭐ / ⚠️ 接近但有一项超 / ❌ 不该打 ⭐

### D. 验证状态

- commit 是否用了 `--no-verify`? 若是,该 commit 必标 ⚠️ 起步
- 看 commit 时间能否对上 detekt 命中清单(本地跑一次 detekt 看改动本身是否撞)
- 评级: ✅ detekt+编译都过 / ⚠️ 用了 --no-verify 但失败属预存且改动本身干净 / ❌ 用了 --no-verify 且改动本身命中 detekt

### E. commit message 准确性

参考 crash-analyzer SKILL.md §6.5 模板,看 commit message 是否包含:
- 版本号 tag (如 `[7.4.0]`)
- `[fix bug]` 或 `[重构]` tag
- 异常类全名
- 友盟上报数据(N 次/M 用户)
- 简述修复机制
- 评级: ✅ 全齐 / ⚠️ 缺 1-2 项 / ❌ message 与改动不符 或 漏关键信息

### F. 对其他业务的影响(扩展版)

**三层检查**,任一层有问题 → 整体降级:

1. **调用方影响**(grep 改动函数/字段的所有引用):
   - 改动的 method 签名变了吗?返回值/参数语义变了吗?
   - 跨 module 调用方是否受影响?
   - 行为变化对调用方是隐式的(无编译错但运行时崩)?

2. **业务流影响**(看本批改动覆盖的业务模块):
   - 改动所在模块的核心业务流(查看历史/支付/登录/启动等)是否受影响?
   - 用户路径上某个分支是否被改动绕过/废弃?
   - 是否影响埋点上报、A/B 实验路径、灰度配置读取等横切流?

3. **多进程行为**(本项目有多进程,关键):
   - 改动在所有进程都执行吗?还是只主进程?
   - 子进程(推送/服务/WebView 进程)的代码路径是否受影响?
   - 改动依赖的字段在子进程是否已初始化?

评级: ✅ 三层都无影响 / ⚠️ 有 1 层需 follow up / ❌ 明显破坏现有调用方或业务流

### G. 向后兼容

- **API 守卫**: `if (SDK_INT >= XX)` 是否对所有低版本设备无害?用了 minSdk 之下不存在的 API?
- **数据/Schema 兼容**: 改了 schema / Parcelable 字段 / 序列化格式吗?老版本数据/缓存能读吗?
- **配置兼容**: 改了 config / SharedPreferences key 吗?老 key 仍能读吗?
- 评级: ✅ 完全兼容 / ⚠️ 边界 case 需 follow up / ❌ 破坏老版本

### H. 合规与隐私(新增)

参考本项目对隐私同意的严格策略(任何**可能涉及用户数据/设备状态/反射读取**的 API 都等 `splashPrivacyAgreed` 后再调):

1. **隐私同意守卫**:
   - 改动是否**绕过**了 `if (confirmPrivacy)` / `splashPrivacyAgreed()` 守卫?
   - 若绕过,被调 API 是否真的**不涉及任何用户数据**?(如 `setDataDirectorySuffix` 是纯本地配置 OK,反射读 `/proc/self/cmdline` 不 OK)
   - **强制写注释说明绕过理由**——commit/代码注释里必须有合规论证

2. **敏感 API 调用时机**:
   - 改动里有没有早于隐私同意就调读设备 ID/MAC/IMEI/GPS/Contacts/Phone state 的代码?
   - 改动里有没有早于隐私同意就发起网络请求 / 写日志到外部 / 上报数据?

3. **SDK 合规清单**:
   - 改动是否新引入第三方 SDK?该 SDK 的合规清单是否已审过?
   - 改动是否触发现有 SDK 的初始化时机变化?

4. **数据收集最小化**:
   - 新写入 SharedPreferences / DB 的字段是否真有必要持久化?
   - 日志是否打了敏感信息(user_id, token, location)?

评级: ✅ 全合规 / ⚠️ 1 项需法务/合规顾问确认 / ❌ 明显违规(应 BLOCK)

### I. 错误处理一致性 + 可观测性(新增)

参考项目惯例(`@Suppress + 中文注释说明意图`,见 git log `b0d3c529b0` / `f28da392ac`):

1. **catch 处理一致性**:
   - 改动里 catch 的异常类型是否合适?(catch `Exception` 而不是具体子类要有理由)
   - catch 块里**至少有 log** 吗?完全空 catch 必须配 `@Suppress("SwallowedException")` + 中文注释
   - 异常吞掉后是否有降级/兜底行为?

2. **新失败路径可监控**:
   - 改动新增的失败分支(if-return / return early / catch-skip)是否有日志或埋点?
   - 失败时用户感知是什么?是静默 still-ok / 报错提示 / 自动重试?
   - 失败率能从现有日志/友盟/埋点抓到吗?

3. **日志规范**:
   - 用了 `println` / `e.printStackTrace()` 吗?(应该用项目里的 Timber / Logger)
   - 日志包含定位信息(类名/方法名/关键参数)吗?
   - 日志是否打了敏感字段?(交叉 H.4)

4. **@Suppress 注释完整性**:
   - 用了 `@Suppress(...)` 吗?是否附中文注释说明**为什么需要 Suppress**(参照项目成熟做法)?

评级: ✅ 全遵守惯例 + 失败可监控 / ⚠️ 1 项有瑕疵 / ❌ 完全吞异常无监控 或 用了 println 等违规日志

### G. 向后兼容

- API 守卫: `if (SDK_INT >= XX)` 是否对所有低版本设备无害?
- 数据兼容: schema / 序列化格式变了吗?老版本数据能读吗?
- 配置兼容: 改了 config.yaml? 老 config 仍能解吗?
- 评级: ✅ 完全兼容 / ⚠️ 边界 case 需 follow up / ❌ 破坏老版本

## 5. Review 报告模板

文件路径: `.cc_tmp/crash/reviews/YYYY-MM-DD-<branch>.md`

```markdown
# Crash Fix Review 报告 YYYY-MM-DD

**分支**: `<branch>` (HEAD `<short sha>`)
**基底**: `origin/master` (`<master short sha>`)
**待 review commit 数**: N
**对应 crash-analyzer 报告**: `.cc_tmp/crash/reports/YYYY-MM-DD.md`

## 总评

- 🟢 可直接合主干 / 🟡 可合但有改进点 / 🔴 BLOCK 需修改

| commit | 异常类 | 影响用户 | 总评 |
|---|---|---|---|
| `<short sha>` | `<exc>` | <N> | 🟢/🟡/🔴 |
| ... | | | |

---

## Commit 1: `<short sha>` — `<异常类>`

**对应崩溃**: 错误 ID `<id>`, <N 次/M 用户>, [友盟详情](<链接>)
**commit message** (摘要): `<subject>`
**改动**: <X 文件 / Y 行>

### 九维度评估

| 维度 | 评级 | 理由 |
|---|---|---|
| A 方案对应根因 | ✅/⚠️/❌ | <一句话> |
| B 新引入崩溃风险(NPE/资源泄漏/死循环/线程安全) | ✅/⚠️/❌ | <一句话,4 项中哪几项命中> |
| C 改动是否真最小 | ✅/⚠️/❌ | <一句话> |
| D 验证状态 | ✅/⚠️/❌ | <一句话,若 --no-verify 必写出> |
| E commit message 准确 | ✅/⚠️/❌ | <一句话> |
| F 对其他业务的影响(调用方+业务流+多进程) | ✅/⚠️/❌ | <一句话,带 grep 结果 + 业务流判断> |
| G 向后兼容 | ✅/⚠️/❌ | <一句话> |
| H 合规与隐私 | ✅/⚠️/❌ | <一句话,是否绕开守卫/敏感 API 时机/SDK 合规> |
| I 错误处理一致性 + 可观测性 | ✅/⚠️/❌ | <一句话,@Suppress 注释/失败监控/日志规范> |

### ✋ 强制问题 (BLOCK)
- <如有,列出;无则写"无">

### 💭 建议 (非强制)
- <如有,列出>

### 📂 关键 diff 节选
```diff
<最关键的几行 diff>
```

---

## Commit 2: ...

...

---

## 整批观察(跨 commit 视角)

### 跨 commit 全局影响
- 本批 N 个 commit 合起来,是否打破某个抽象/互相干扰?
- 本批是否多个 commit 同时改动同一文件? diff 是否相容?
- 本批是否有"前一个 commit 引入的新代码,后一个 commit 又改了"的链式依赖?

### 测试覆盖
- 被改文件的现有单测路径(`<module>/src/test/...`)是否存在?
- 现有单测是否覆盖本次 fix 的场景? (跑一遍 `./gradlew :<module>:testReleaseUnitTest <被改类相关>`)
- 若无现成单测,建议补哪几条? (针对崩溃场景 + 修复后的预期行为)

### 回滚成本
- 本批 commit 能否独立 `git revert`,还是已经互相依赖(后续 commit 引用了前面的新代码)?
- 改动是否传播到调用方(F 维度的反向看): 若 revert 是否会让调用方编译失败?
- 评估"应急回滚"的难度: 🟢 单个 revert OK / 🟡 需顺序 revert 多个 / 🔴 必须人工 cherry-pick

## 整体改进建议(基于本批 review)

- <如本批反复出现某类问题,可建议 crash-analyzer SKILL.md 增加约束>

## 下一步行动

- 🔴 BLOCK 的 commit: 用户决定是否撤回(`git reset`)或修订
- 🟡 改进点: 可在本批继续修,或留作下次跑时关注
- 🟢 通过的 commit: 可直接合 dev / 拉 MR 合 master
```

## 6. 失败 fallback

| 场景 | 处理 |
|---|---|
| auto_fix_bug 分支不存在 | 提示用户先跑 /analyze-crashes,skill 退出 |
| 分支存在但 origin/master..<branch> 为 0 commit | 提示用户"无待 review",skill 退出 |
| 找不到对应 crash-analyzer 报告 | 报告里标"⚠️ 盲审(无报告对照)",维度 A "方案对应根因" 标 ⚠️ "无法验证报告里描述的根因" |
| Grep 调用方超时 (>30s) | 报告里标"⚠️ 调用方未深查",维度 F 自动降级 ⚠️ |
| 单个 commit diff 过大 (>500 行) | 重点 review,但维度 C "最小改动" 自动 ❌ |
| 待 review commit > 20 个 | 提示用户分批 review,默认只 review 前 10 个 |

## 7. 数据安全

- review 报告落在 `.cc_tmp/crash/reviews/`,不进 git
- skill 不动代码、不动 git、不主动 push
- 不调用任何外部 API

## 8. 触发命令登记

- Slash command 名: `/review-crash-fixes` (默认 review `auto_fix_bug`)
  或 `/review-crash-fixes <branch>` 指定其他分支
- 触发后 Claude 读本 SKILL.md 全文,按 §3 流程执行
