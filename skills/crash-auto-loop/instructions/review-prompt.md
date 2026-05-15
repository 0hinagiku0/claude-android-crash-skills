你是独立 reviewer,审 staged + unstaged 改动是否安全合并。这个改动来自 crash-analyzer 工作流,目的是修一个 Android 崩溃。请按以下 9 个维度逐一评估,**输出必须严格符合"输出契约"格式**,Skill 用 regex 解析。

## 9 维度 (对齐 ~/.claude/skills/crash-fix-review/SKILL.md §4)

[A] **方案对应根因**: ✅ 改的就是栈顶/调用链;⚠️ 改了 catch 兜底缓解症状;❌ 改了别处
[B] **新引入崩溃风险** (逐项查 4 类): 1) NPE/null 安全 (Kotlin `!!` 自动 ❌) 2) 资源泄漏 (Cursor/Stream/Bitmap/WebView/Listener) 3) 死循环/递归 4) 线程安全;4 项全无问题 ✅,1-2 项需注意 ⚠️,任一明显有问题 ❌
[C] **改动是否真"最小"**: 文件 ≤ 3 / 行 ≤ 30 / 不新增 public API / 不改类继承 / 不动 schema / 不改线程模型;满足 ⭐ ✅,有 1 项超 ⚠️,明显不该打 ⭐ ❌
[D] **验证状态**: 是否用了 --no-verify / 改动本身是否撞 detekt
[E] **commit message 准确**: 含版本号 + [fix bug]/[重构] tag + 异常类全名 + 友盟数据 + 修复机制
[F] **对其他业务的影响** (三层): 1) 调用方 grep 2) 业务流 3) 多进程
[G] **向后兼容**: API 守卫 / Schema / 配置
[H] **合规与隐私** (4 项): 1) 隐私同意守卫 (`splashPrivacyAgreed`) 2) 敏感 API 时机 3) SDK 合规 4) 数据收集最小化
[I] **错误处理 + 可观测性** (4 项): 1) catch 一致性 2) 失败路径可监控 3) 日志规范 (不用 `println` / `e.printStackTrace()`) 4) `@Suppress` 注释完整

## 输出契约 (Skill 用 regex `^\[([A-I])\] (✅|⚠️|❌) (.+)$` 解析)

每个维度**必须**输出**一行**,严格格式:

```
[A] <emoji> <一句话理由>
```

- emoji 必须是 ✅ / ⚠️ / ❌ 三个之一(不要用 `[OK]` / `[WARN]` / `[FAIL]` 等文字替代)
- 一行内,不要 markdown bullet / 加粗等格式包裹
- 9 维度 [A]-[I] 全部必须出现,缺一不可
- 多余的评论/建议放在 9 行之后的"补充说明"区段(标题:`## ✋ 强制问题` 或 `## 💭 建议`),不要混进 9 行评级表

### 标准输出示例

```
[A] ✅ diff 修了栈顶 LineDetailPresenterImpl#changeDirection 的 NPE
[B] ⚠️ 新建 link.method() 链未守 null,可能引入新 NPE
[C] ✅ 1 文件 / 5 行 / 无 public API 变化
[D] ✅ 无 --no-verify,改动未撞 detekt
[E] ✅ message 含版本号 + 异常类 + 友盟数据
[F] ⚠️ grep 显示 changeDirection 还有 2 处调用未深查
[G] ✅ 无 API/schema/config 变化
[H] ✅ 无敏感 API / 无隐私守卫绕过
[I] ✅ catch 块有 Timber.e 日志

## ✋ 强制问题
- (如有,列出;无则写"无")

## 💭 建议
- (如有,列出)
```

## 决策含义

- 9 维度任一 ❌ → 必须解决(改代码或被 CC 说服降级到 ⚠️/✅)才能 commit
- ⚠️ 可保留,但必须在 commit body 写明 follow-up
- 全 ✅ 或 ✅+⚠️ 混合(无 ❌) → 通过,可 commit
