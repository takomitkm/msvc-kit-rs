# 安全提取、MSI Backend、Staging、事务提交与恢复

本文件规定最高优先级的安全工作。提取器与事务层必须独立测试。

## 1. 当前设计风险

- msvc-kit 当前提取函数会把 VSIX/MSI/CAB 内容写到正式 target root。
- 多 package 提取并发运行。
- `.msvc-kit-extracted` marker 仅用 source path/size/mtime fingerprint，且 marker 写入失败被忽略；marker 命中也不验证实际输出。
- 归档路径如果直接 `target.join(name)`，可能发生 Zip Slip 或路径逃逸。
- `installation.lock` 防止并发，不提供失败回滚。
- portablemsvc 虽然先提取临时目录，但替换时先删除旧目录再 rename，不能完整恢复。

Agent 必须首先为当前 extractor 增加安全测试；不能在没测试的情况下直接切换 backend。

## 2. 统一 Extractor 接口

建议概念：

```rust
#[async_trait]
pub trait PackageExtractor: Send + Sync {
    fn kind(&self) -> PackageKind;
    fn backend_id(&self) -> &'static str;
    async fn inspect(&self, source: &Path) -> Result<ArchiveInspection>;
    async fn extract(&self, source: &Path, destination: &Path, options: &ExtractOptions)
        -> Result<ExtractReceipt>;
}
```

MSI 特殊：

```rust
enum MsiBackendPreference {
    Auto,
    Msiexec,
    Native,
    Fallback,
}
```

- Windows 保留 `msiexec /a` backend。
- 非 Windows 保留当前已支持的 `msiextract` path，或明确限制该平台。
- Native backend 只有在真实 MSI fixture parity tests 达到标准后才加入可用选项。
- `Auto` 选择经过验证的稳定 backend；不能因为 crate 编译成功就宣布 native backend 稳定。
- `Fallback` 必须只对可恢复的 backend errors fallback；损坏文件、hash mismatch、path traversal、安全拒绝不能 fallback 后忽略。

## 3. MSI Native Parser 调研要求

Rust 可读取/处理 MSI 数据库；但“有 msi crate”不等于完整 MSI extractor。

Agent 先做技术 spike，评估：
- `msi` crate（Rust MSI relational database read/write；确认版本、MSRV、license、API、维护活跃度）。
- `cab` crate（CAB cabinet extraction；确认 MSZIP/多卷/压缩方式/文件名行为）。
- 其他候选（必须评估成熟度，不因 crate 名字像就采用）。

必须支持/解释的 MSI 内容：

1. Compound File Binary (CFB) 容器与 MSI database tables。
2. `Directory`：目录树、DefaultDir 短名|长名规则、目标路径属性。
3. `Component` / `FeatureComponents`：文件所属组件与 feature 关联。
4. `File`：FileName、FileSize、Sequence、Version、Attributes。
5. `Media`：LastSequence、Cabinet、DiskPrompt、Source。
6. Embedded CAB streams（常见 `#cabinetname`）及外置 CAB source media。
7. 按 `File.Sequence` 与 `Media.LastSequence` 关联正确 cabinet。
8. Cabinet 内部压缩方式和分卷处理。
9. MSI 文件表与 cabinet 文件名短/长名映射。
10. MSI 中可能存在非普通文件、重复 sequence、缺失 media、压缩/未压缩 payload。
11. Directory properties：不能把 Windows SystemFolder/ProgramFiles 等系统属性直接映射到真实系统目录；管理提取必须映射到指定 staging root。
12. 只提取目标 feature/components 所需的文件时，要正确处理 feature selection 和 condition；初期可先提取整个安全的 administrative image，不得假装支持精确组件裁剪。
13. CAB 文件路径必须限制在 staging/source roots，防绝对路径、UNC、`..` 和 symlink escape。
14. MSI 文件可能引用相邻 CAB；MSVC/SDK 的 CAB source media 必须由 Plan 精确列出并哈希验证。

### Native MSI 通过标准

在默认启用前必须：
- 至少使用许可允许的真实 MSI fixtures，覆盖 MSVC VSIX 不是 MSI，Windows SDK MSI + 外置 CAB，embedded CAB MSI。
- 与 `msiexec /a` 输出比较：目标文件集合、文件长度、文件 hash。允许并解释明确的语义差异。
- 对每个文件证明正确的 package/sequence mapping。
- 测试长路径、Unicode、空格、短名|长名、重复目录、缺失 CAB、错误 CAB hash、恶意路径。
- 在 Windows x64 上跑。
- 至少在两个独立版本的 SDK/MSI 上验证，而非单一 fixture。
- 完成 parity tests 前，native backend 只能实验性 opt-in，默认仍用 msiexec。

如果这些条件无法满足，停止 native extractor 实现，仅交付 backend abstraction + 可行性报告；不要制造一个不完整 parser。

## 4. msiexec `/a` 与 UAC

调用形式必须通过 `Command::new("msiexec")` 参数数组，不能 shell interpolate：

```text
msiexec /a <msi-path> /qn TARGETDIR=<staging-path> /L*V <log-path>
```

具体属性参数按 MSI 测试修正。日志路径在独立 transaction diagnostics 中。

行为规则：
- `/a` 是 administrative image extraction；不是 `/i` 产品安装。
- `/qn` 只控制 UI，不禁用 UAC 或权限校验。
- `/a` 不因参数本身必然要求管理员，但具体 MSI action/ACL 可改变结果。
- 不主动请求 UAC、不执行 `runas`。
- 目标 staging 必须当前用户可写。
- 如果 Windows Installer/custom action 需要系统权限，返回明确权限错误，不擅自提升。
- 1618 进行有限重试（重试间隔可配置/指数退避）；其他错误不重复盲试。
- 1603 是 generic fatal，不要直接断言一定因为路径问题。
- 1619 常见 source media/路径问题；诊断提示核对 MSI 和 CAB，不要盲目清缓存。
- `/i` 只允许在静态代码/命令构造测试中确认未调用；不执行产品安装。

## 5. 统一安全路径验证

所有 ZIP/VSIX/CAB/MSI extractor 写入前必须调用统一 `SafePathResolver`。

必须拒绝：
- 绝对 POSIX path `/...`。
- Windows drive absolute `C:\...`。
- UNC `\\server\share`。
- `..` 穿越（包含 mixed slash、重复编码后检查）。
- ADS `file.txt:stream`（除非明确允许且有测试；默认拒绝）。
- `.`/空 path/空 basename。
- 设备名 `CON`, `NUL`, `AUX`, `COM1..9`, `LPT1..9`（含扩展名情况）。
- 尾随空格和点导致 Win32 规范化碰撞。
- 目标根内的 symlink/junction/reparse point 跳出 staging。
- 两个不同 archive entry 规范化后映射到同一 Windows path。
- 文件与目录路径冲突（例如 `a` 同时作为文件和 `a/b` 目录）。
- archive header 声明的大小超过限制或压缩比构成资源耗尽风险。

Path validation 必须基于 Windows 语义，即使测试宿主是 Linux，也要用 Windows path parser/fixture 验证。

## 6. Package Staging

每个 package 单独解包，避免并行包写入互相覆盖：

```text
<install-root>/.msvc-kit/transactions/<tx-id>/
├── journal.jsonl
├── plan.json
├── downloads/                 # 可指向已验证 Blob，不复制大文件
├── packages/
│   ├── <safe-package-id-1>/    # 每个包独立提取
│   └── <safe-package-id-2>/
├── merged-tree/               # 由合并器按 ownership/collision 规则产生
└── backup/                    # 仅本 transaction 修改的旧 owned files
```

- Staging 必须与最终目标在同一 volume，确保单文件 rename/move 可以在同卷操作。
- 包 ID 作为目录名之前必须 safe encode/hash；不允许原始 Manifest ID 注入路径。
- package extractor 输出 `ExtractReceipt`，列出相对路径、size、hash、kind。
- package 间相同 path：hash 一致可以共享多个 owner；hash 不一致则 Plan conflict，禁止“最后写入者胜出”。
- 不允许并发向同一个合并目录直接写。
- CAB source media 在 MSI extraction 完成前不得被 GC。

## 7. msvc-kit 共享安装根的事务策略（关键）

**不得替换整个 `<install-root>`。** 当前 root 还存下载、多个版本、receipt、marker 和其他工具链文件。

初始安全实现建议文件级 transaction：

1. Plan 计算本 transaction 需要新增/更新/删除的 owned relative paths。
2. 检查每个目标 path 是否已有文件。
3. 如果不存在：标记 `Create`。
4. 如果已有且 receipt/ownership 指明由 msvc-kit 管理，并且当前 hash 与 receipt 相符：可备份后替换。
5. 如果已有且 hash 相同：可以复用，增加 owner 引用，不写文件。
6. 如果已有但无 ownership 记录：`UnownedPathConflict`，默认拒绝。
7. 如果已有且属于另一个 package/install owner：相同内容可共享；不同内容冲突。
8. 对被新版本淘汰的旧文件：只在没有其他 owner 且实际 hash 仍与 receipt 相符时计划删除；否则留存并警告/冲突。
9. Commit 只改以上 plan 列出的文件和元数据。
10. 下载缓存和其他版本不在动作清单内，绝不触碰。

另一种长期方案是将每个安装完整放入 `.msvc-kit/instances/<install-id>/`，再通过环境脚本选择实例；但迁移需要评估现有路径/消费者兼容性。不要在第一阶段悄悄改布局。

## 8. Transaction Journal / Commit / Rollback

### 8.1 写前日志顺序

每项文件变更：

1. 写 `WillBackup(path, old_hash, backup_path)` 并 flush/sync。
2. 将原文件移动到 transaction backup（如果有）。
3. 写 `BackupDone` 并 flush/sync。
4. 写 `WillInstall(path, new_hash, staged_path)` 并 flush/sync。
5. 将 staged 文件移动到目标（同卷 rename 优先；不能 rename 时 copy-to-temp+verify+replace）。
6. 重新 hash 目标文件。
7. 写 `InstallDone` 并 flush/sync。
8. 完成全部文件后更新 OwnershipIndex / InstalledManifest / Receipt，metadata 必须 atomic write。
9. 写 `Committed`，它是 transaction 完成标记。
10. 后续才清理 backup/staging。

具体顺序由 Agent 设计成真正 crash-safe protocol。关键是恢复能识别动作完成与日志状态不一致的窗口。

### 8.2 Rollback

按逆序处理 journal：
- 新文件仍与本次 expected hash 相同：删除。
- 原文件备份存在且目标不存在：恢复备份。
- 目标文件内容与本次 hash 不同（有人同时改过）：不要删除/覆盖，标记 conflict/needs_recovery。
- 备份缺失、journal 不完整或 hash 不匹配：停止自动恢复，保留所有证据并生成诊断报告。
- Rollback 完成后恢复旧 Receipt/OwnershipIndex，并写 `RolledBack`。

### 8.3 Recovery

每次新的 Apply / clean 前先检查未完成 transactions：
- 未进入 commit：安全清理 staging（需确认其 ownership）。
- commit 已开始但未完成：根据 journal 自动继续 rollback；默认 rollback 而不是继续 apply。
- 已 `Committed`：仅清理已确认的 backup。
- `NeedsRecovery`：拒绝新写入，并提供 `msvc-kit recover --dir ... --transaction <id> --report`。

Recovery 必须幂等：重复执行不会二次破坏。

## 9. 并发 MSI 行为

- 多 MSI 并行是否安全必须先实测；Windows Installer 有系统级 busy/1618 限制。
- 即使 VSIX/CAB 可以并行，MSI 默认串行或使用受限 semaphore。
- 1618 retry 应检测到状态并等待；不要持有其他不相关全局锁造成死锁。
- transaction/root lock 在 Plan revalidation、提交、rollback 关键部分保持；下载锁与 root lock 的持有范围要避免长期阻塞。

## 10. 验收门槛

- 归档 traversal suite 全部通过。
- 未知非空目录从未改变。
- 两个版本同 root 共存测试通过。
- transaction 在每个 fault point 被杀死后下一次启动可恢复。
- 原文件内容 hash 在失败回滚后与开始前完全一致。
- 其他非本 transaction 文件在成功和失败时均保持 hash 不变。
- 无 MSI `/i` 调用。
- 标准用户写入用户目录的 integration test 不要求工具主动 elevation；若具体 MSI 自身要求，则报告而非提权。
