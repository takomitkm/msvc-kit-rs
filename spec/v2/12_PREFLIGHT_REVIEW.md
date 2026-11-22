# 工作包 v2 预检报告

**检查日期：2026-11-23**  
**检查对象：** `msvc-kit-agent-workpack` 原始 Markdown 包及 ZIP  
**结果：** 文件结构与压缩包完整性通过；发现若干会影响实现正确性的规格歧义，已在 v2 修订。

## 1. 文件/打包检查

- 原包包含 15 个 Markdown 文件。
- ZIP 内文件数与目录内容一致。
- `unzip -t` 检查无损坏。
- 所有本地 Markdown 文件引用均能在工作包内找到。
- `FULL_SPEC.md` 是合并阅读副本；Agent 应以分文件规格为维护源，修改规格时同步重建 FULL_SPEC，避免两份内容漂移。

## 2. 发现并修订的问题

### R1 — 源码快照未固定

**风险：** 文档链接到 `main`，Agent 在稍后运行时可能看到不同实现，导致“当前代码”证据与规格快照不一致。  
**修订：** 固定参考快照：
- msvc-kit: `87dc08ce510a11a874f8e088f17c4be37ee01672`
- portablemsvc: `92554c094a1d00abef1cfb610877edcb1eb325f3`

Agent 仍必须以本地 fork HEAD 为最终事实；固定快照只用于审计历史上下文。

### R2 — ZIP/VSIX 目录 entry 的尾斜杠

**风险：** 一般 ZIP/VSIX 会用 `folder/` 表示目录。若按文件路径规则一律拒绝空 basename，会误拒合法包。  
**修订：** 先识别目录 entry，再剥离一个合法尾分隔符；内部空组件、重复 separator、`.`、`..`、绝对路径和盘符/UNC 仍拒绝。真实包 fixture 可以支持明确例外，但必须有正反测试。

### R3 — MSI `msiexec /a` 不是安全沙箱

**风险：** `/a` 是 administrative image extraction，但 MSI 的 AdminExecuteSequence/CustomAction 可能存在 `TARGETDIR` 以外的副作用。普通用户运行、staging 路径和 `/qn` 并不能证明任意 MSI 只写 staging，也不能保证绝不触发 UAC。  
**修订：** 只对来源可信且 SHA256 已验证的官方 Microsoft payload 使用；不提升权限、不执行 `/i`；记录 MSI 日志和权限上下文。明确说明无法把 `msiexec` 当作对不可信 MSI 的 sandbox。

### R4 — 单进程错误注入不证明 crash recovery

**风险：** 同一进程返回错误/panic 与进程在文件系统操作中途被杀死的持久化语义不同。  
**修订：** 对关键 commit 窗口增加 helper child process，由父进程在指定 journal 边界终止，再启动新的 recovery 进程验证文件 hash 和状态。不能运行则标记 BLOCKED，不宣称 crash-safe。

### R5 — GC 无法自动发现任意外部 Lockfile

**风险：** 一个放在任意工作区/离线介质的 Lockfile 不会自动出现在 cache root 的数据库中。只扫描 cache registry 无法知道它仍引用某些 Blob。  
**修订：** 明确区分已登记 lockfile 与外部 lockfile；提供 pin/unpin 或导入注册机制，GC 只对已知引用做保证。文档必须说明外部 lockfile 若未 pin，不受 GC 保护。

### R6 — Manifest cache 与 payload cache 被统称为“Dry Run 不写入”

**风险：** Plan 为获取 Manifest 可能更新 ETag/Last-Modified 缓存，但不应下载大型 payload 或改写目标安装树。  
**修订：** 明确 Plan 可更新独立 Manifest cache；禁止写入 target root、payload Blob、staging、Receipt 和 installed files。`--no-cache` 若提供则必须真正只读。

### R7 — Archive path race / link entry

**风险：** 一次性检查 lexical/canonical path 无法完整抵御目录被换成 symlink/junction 的 TOCTOU；归档中的 symlink/hardlink 也可能突破 staging 边界。  
**修订：** 默认拒绝链接 entry；逐级检查 staging root/reparse points；优先 handle-based/no-follow API，并明确无法消除的剩余风险。

## 3. 仍需本地 Agent 验证、不能由规格替代的事项

- 用户 fork 的 HEAD、dirty state、已有本地修改和 CI 真实命令。
- 用户 fork 是否已经在 2026-11-23 之后增加某些功能。
- 当前 `msiexec` 调用构造和日志参数是否与本文一致。
- 各 MSI/VSIX/CAB 解析 crate 当前可用版本、MSRV、许可证和测试覆盖。
- 官方 Windows SDK MSI/CAB 真实文件的 CAB/Media/Sequence 行为。
- Windows 用户权限、UAC、reparse point、rename/replace、进程崩溃恢复行为。
- 现有 `DownloadIndex` 的数据库 schema、索引 key 和 migration 行为。
- 所有当前 CLI/API/JSON schema 的兼容性。

## 4. 当前不应对 Agent 作出的承诺

- 不保证任意 MSI 的 `/a` 都不提示 UAC。
- 不保证纯 Rust MSI extractor 一定能达到 `msiexec` parity；可以得出不实现的结论。
- 不保证 portable/minimal pruning 在所有 MSVC/SDK 版本都安全；未知版本默认不裁剪。
- 不保证外部未注册的 Lockfile 自动阻止 CAS GC。
- 不保证文件级事务对所有文件系统/杀毒软件/打开中的文件都能原子完成；无法 rollback 时必须进入 `NeedsRecovery`。
- 不把 fixture tests 的通过写成真实 SDK/MSI 实机安装通过。

## 5. 建议执行方式

1. 把 `AGENT_FIRST_MESSAGE.md` 发给 Agent。
2. 允许 Agent 自主完成 P00，先查看 `agent-work/BASELINE.md` 和 `TEST-RESULTS.md`。
3. P00 后继续 P01-P11；每个 Exit Gate 不通过就停在该阶段，先修复或记录阻塞。
4. Agent 即将耗尽上下文/额度时，要求它先更新 `PROGRESS.md` 并写出下一条精确命令，再结束本轮。
5. Windows-only 与真实 MSI 集成测试必须在 Windows 环境执行；没有运行就保持 BLOCKED/NOT_RUN。
