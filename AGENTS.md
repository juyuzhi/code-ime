# CodeIME Agents 约定

本文件约束 AI 代理在本仓库的一切代码修改。改动前先读完本页。

## 平台 API 稳定性约束（最高优先级）

插件 sinceBuild=261（2026.1），编译目标 263（2026.3）。**只允许使用在 261/262/263
三版本上都存在、且未被弃用/未标 experimental 的平台 API。**

- **禁止弃用 API**：验证器报告 deprecated usages 必须为 0。弃用信息在 class 文件的
  `Deprecated` 属性与 Kotlin 元数据里（`tools/dump_class.py` 可查）。
- **禁止 experimental API**：`@ApiStatus.Experimental` / 平台 api-info 标记 experimental
  的类与方法一律不用。历史教训：`WriteIntentReadAction` 是 experimental，
  `ReadAction.compute(ThrowableComputable)` 在 261/262/263 三版本均已弃用——
  从弃用 API 迁移时必须先核实替代 API 本身稳定，不要跳进另一个坑。
- **Companion 陷阱（曾造成 Marketplace Critical）**：平台类会在版本间被 Kotlin 化。
  2026.3 把 `FileEditorManager` Kotlin 化后，从 Kotlin 源码调用其静态方法会编译成
  `Companion` 引用，261/262 上该内部类不存在 → 缺失引用 Critical + 旧版运行时崩溃。
  规则：跨版本调用的平台静态 API 若在某个版本被 Kotlin 化，须经
  `src/main/java/com/codeime/compat/` 下的 Java 包装类调用（javac 生成 invokestatic，
  三版本通用）。新增此类包装时放同一目录并在类注释说明原因。
- **Kotlin jvm-default**：`build.gradle.kts` 已设 `-Xjvm-default=all`，勿删。
  默认的 all-compatibility 模式会给实现 Kotlin 接口的类生成 ACC_BRIDGE 兼容方法
  （如 `getPresentation(PlatformType)`），验证器会误判为"覆写弃用方法"。
- **状态栏组件**：只 override 无参的 `getPresentation()`（稳定）；
  `getPresentation(PlatformType)` 是弃用方法，不要实现它。
- **读锁**：EDT 上读 PSI 用 `ReadAction.runBlocking`（三版本稳定）。
  不用 compute 系列（全弃用）与 WriteIntentReadAction（experimental）。
- **不要凭注释/记忆判断 API 状态**：平台注释会过时。用 `tools/dump_class.py` 直接看
  目标版本字节码里的方法、`Deprecated` 属性与注解。

## 验证流程（改动平台相关代码后必做）

1. **快速比对（秒级，先跑）**：
   ```
   python tools/check_refs_262.py build/libs/<版本>.jar <IDE目录>
   ```
   IDE 目录用两个极端版本（本地缓存通常已有）：
   - IU-2026.1.5.1：`~/.gradle/caches/9.8.1/transforms/*/transformed/idea-2026.1.5.1-win`
   - IU-2026.2.5：同上 `idea-2026.2.5-win`
   期望输出"未发现缺失引用"。
2. **核查单个平台类形态**：`python tools/dump_class.py <IDE目录> <class内部名>`
3. **正式验证（约 10 分钟，上传前必跑）**：
   ```
   ./gradlew verifyPlugin -PverifyTarget=market
   ```
   三目标 IU-261.27258.136 / IU-262.12573.28 / IU-263.7307.200 必须全部 Compatible，
   且无 deprecated、无 experimental。本地 IDE 都是 263，**不带 -PverifyTarget=market
   的本地验证查不出 261/262 问题，不能作为发布依据**。

## 构建注意

- 平台类是 Java 25 字节码（v69），JDK 21 的 javac 读不了。`build.gradle.kts` 已配置
  JavaCompile 自动 fork 到发现的 IDE 自带 JBR javac 25 并 `--release 21` 输出——
  不要改动这套逻辑，也不要在本机乱装 JDK 期望它被用到。
- IDE 路径一律自动发现（`build.gradle.kts` 的 discoverIdeInstalls），**禁止写死路径**。
- 依赖 IDE 自带的库（如 JNA）用 `compileOnly`，不打进插件包。

## 版本与发布

- 每次产出可上传的构建必须升版本号（build.gradle.kts 的 `version`），Marketplace
  拒绝重复版本号。上传包：`build/distributions/code-ime-<版本>.zip`。
- 发布前跑一遍完整验证流程（见上），并把验证结果写入交接笔记
  （桌面 `CodeIME-交接笔记.md`）。
- 提交信息沿用现状：`<版本>：<要点>`，推送 gitee 与 origin（github）双仓库；
  github 网络不通时在交接笔记里记录，勿反复重试。

## 其他

- VSCode 扩展（`vscode/`）与 IDEA 插件版本号各自独立，改动一侧时确认另一侧是否需要同步。
- 诊断日志走 `Logger.getInstance("CodeIME.Diag")`，正式版保持现有级别，不要随意增删。
