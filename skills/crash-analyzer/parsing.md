# CSV 解析规则

> Skill 在分析阶段 Step 1 读取此文档。

## 1. 字段映射(11 列,基于 2026-05-12 实采样本)

| 列序 | 列名 | Skill 用法 |
|---|---|---|
| 1 | 错误摘要 | 第一行=异常类名+message,第二行起=单帧栈 `at <FQCN>.<方法>(<file>:<line>)` |
| 2 | 错误ID | ⚠️ **不读此列**,见 §2 |
| 3 | 错误类型 | `JAVA崩溃` / `Native崩溃` / `ANR`,影响分类策略 |
| 4 | 最近一次发生时间 | 报告头部"时间窗口"显示 |
| 5 | 版本范围 | 如 `7.3.0` 或 `7.3.0 - 7.3.2` |
| 6 | 错误次数 | 排序信号 |
| 7 | 影响用户数 | 排序信号(A 类主排序) |
| 8 | 状态 | `已修复` → 直接归 D 类 |
| 9 | 处理人 | 报告里展示,不参与决策 |
| 10 | 标签 | 报告里展示,不参与决策 |
| 11 | 链接 | **真实错误 ID 唯一来源**,见 §2 |

## 2. 错误 ID 处理(必读)

友盟原始 CSV 里"错误ID"列**本应是 13 位长整型**(如 `11390243063113`),但只要 CSV 被 Excel 打开过并保存,Excel 会把它显示并存成科学计数法 `1.13902E+13`,**精度永久丢失**(只保留 6 位有效数字)。此时 `11389749661113` 和 `11388899021113` 在 CSV 里都会变成 `1.13889E+13`,无法区分。

**规则:Skill 永远从「链接」列提取真实 ID**:

```
正则: crash/detail/(\d+)
示例: https://apm.umeng.com/.../crash/detail/11390243063113
                                             ^^^^^^^^^^^^^^ 这串才是真长整型 ID
```

**Fallback**:若某行「链接」列缺失或解不出 ID,标记该行"ID 缺失",在 E 类报告中用行号当临时标识列出。

## 3. 错误类型分支(决定走哪条决策树)

- `JAVA崩溃`: 走完整 D → C → B → A 决策树
- `Native崩溃`: 直接归 C 类(应用层不可修符号),不做深度分析
- `ANR`: **按栈顶方法名/类名细分**
  - 匹配 `*IO*`、`*File*`、`*Database*`、`*SQLite*`、`*SharedPreferences*`、`Thread.sleep` 等"主线程做了重活"关键词 → A 类(修法明确:异步化)
  - 匹配 `Object.wait`、`Looper.loop`、锁等待等 → B 类(要业务决策)
  - 其他无明显特征 → 兜底 B 类
- **OOM 特例**: `java.lang.OutOfMemoryError` 不参与栈顶包名匹配(栈顶随机性极大),统一独立桶归 B 类,报告中提示"栈顶不可信,需 Memory Profiler / hprof dump"

## 4. "栈顶帧"严格定义

CSV「错误摘要」列里 `at <FQCN>.<method>(<file>:<line>)` 那一行的 `<FQCN>`——是**调用方类的全限定名**,**不是异常类名**。

示例:
```
java.lang.UnsatisfiedLinkError
... couldn't find "libwebviewchromium.so"
    at org.chromium.base.library_loader.LibraryLoader.loadAlreadyLocked(LibraryLoader.java:56)
```
- 栈顶帧 FQCN = `org.chromium.base.library_loader.LibraryLoader`
- 异常类名 = `java.lang.UnsatisfiedLinkError`
- 包名匹配只用栈顶帧 FQCN → 匹配 `org.chromium.` → 归 C
- 异常类名 `java.lang.*` 不参与黑名单匹配

## 5. 解析约束

- CSV 以 UTF-8 读入;失败回退 GBK。
- 字段值含换行(主要是"错误摘要"列)按 RFC 4180(双引号包裹)。
- **表头双保险**:
  - 先按列序读(友盟当前 11 列序固定)
  - 同时对表头做列名模糊匹配校验:`错误摘要 / 摘要`、`错误ID / ID`、`错误类型 / 类型`、`链接 / URL / 详情` 等
  - 列名校验失败 → 警告但不中止(友盟改列名时仍能按列序跑),在报告头部记"列名漂移检测:xxx",提醒用户更新 Skill 映射

## 6. 自家代码回溯例外(归到 C 但实际是自家)

栈顶虽在第三方/系统层,但**异常 message 中的字符串包含自家代码 FQCN/类名** → 转 B 类"栈顶第三方但根因疑似自家"桶。

示例:
```
android.app.BackgroundServiceStartNotAllowedException
Not allowed to start service Intent { act=com.example.app.android.cac.reminderlive.action.UPDATE
cmp=com.example.app.standard/com.example.app.android.cac.reminderlive.service.SomeForegroundService ...
    at com.example.app.android.cac.reminderlive.SomeServiceGateway.dispatchService(...:47)
```
- 栈顶 FQCN 已经是 `com.example.app.*` → 直接归 A/B,不触发此规则
- 但若有崩溃栈顶在 `android.app.ActivityThread` 但 message 含 `com.example.app.*` → 触发回溯,归 B

> 注:CSV「错误摘要」列可能含多帧栈,但 Skill 决策只用第一帧(`at` 开头的第一行)做 FQCN 包名匹配;深层栈帧不参与自动归类。若需基于深层栈做归因,用户在交互修复阶段点链接看完整栈后,可手动把该崩溃加入修复列表。
