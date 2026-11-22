# 目标架构与数据契约

本文件规定模块职责和概念数据模型。Agent 可按现有 crate/module 结构调整物理文件，但不得改变语义，也不得为了匹配目录树做无意义重构。

## 1. 架构分层

```text
CLI / Rust Library API
        │
        ▼
Application Service
  detect / resolve / plan / apply / verify / list / clean / recover
        │
        ├──────────────┐
        ▼              ▼
Build Profile      Provider Registry
        │              │
        └──────┬───────┘
               ▼
        Dependency Resolver
               │
               ▼
          Install Planner ◄──── Installed State / Cache Index
               │
               ▼
           InstallPlan
               │
               ▼
        Transaction Manager
          ├── Downloader / CAS
          ├── Extractor Backends
          ├── Pruning Engine
          ├── Validator / Doctor
          ├── Journal / Recovery
          └── Receipt / Lockfile
               │
               ▼
        Environment / Activation
```

## 2. 模块职责

### `catalog` / 现有 `downloader::manifest`

- 获取并验证官方 channel Manifest。
- 解析 package、payload、dependencies、host/target metadata、安装条件。
- 维护可重复的 Manifest snapshot/digest。
- 不负责决定用户最终需要什么 profile。

### `resolver`

- 将 BuildProfile/显式请求解析成 capability set。
- 计算 package dependency closure。
- 检测版本冲突、架构不支持、循环依赖、缺少 payload。
- 输出稳定排序的 `ResolvedToolchain`。

### `detection`

- 只读查找外部安装。
- 输出 `DetectedToolchain` 与能力、版本、完整性状态。
- 不修改任何外部目录。

### `planner`

- 读取 Resolve 结果和本地 Cache/Installed inventory。
- 决定每个 payload 的 Download/Reuse、每个文件的 Keep/Extract/Prune/Conflict。
- 计算磁盘空间和 privilege requirement。
- 生成可序列化、可验证、可 apply 的 InstallPlan。

### `storage`

- CAS blobs、redb 索引、Manifest cache、atomic writes、file locks。
- 不决定工具链 feature 和目录结构。

### `installer`

- 事务生命周期、staging、extractor、pruning、validation、commit、rollback/recovery。
- 所有写操作必须受 ownership manifest 限制。

### `lockfile` / receipt

- Lockfile 是输入重建意图和获取来源的锁。
- Receipt 是一次安装的输出证明。
- Installed Manifest 是最终安装树摘要。
- 三者不能合并成一个含糊文件。

### `providers`

- 每个 provider 提供发现/解析/获取/验证/激活所需的后端实现。
- 共用 Planner、Transaction、Cache、Receipt、Diagnostics。

### `environment`

- 把已验证 Toolchain 描述转换成进程环境变量/激活脚本。
- 不负责安装/提取。

## 3. 核心结构体（概念规范）

以下为字段语义，不要求照抄全部 Rust 字段名。若已有类型满足要求，应扩展而非创建重复模型。

### 3.1 `ToolchainRequest`

```rust
struct ToolchainRequest {
    profile: Option<String>,
    provider: String,                 // msvc initially
    channel: Option<String>,
    version_constraints: VersionConstraints,
    host_arch: Architecture,
    target_arches: Vec<Architecture>,
    features: BTreeSet<String>,
    excluded_features: BTreeSet<String>,
    install_root: PathBuf,
    sdk_root: Option<PathBuf>,
    pruning_profile: PruningProfile,
}
```

注意：不要破坏现有公开 `ToolchainRequest` 的 struct literal API。可以设计 `ToolchainRequestV2`/builder 或 additive wrapper。

### 3.2 `ResolvedToolchain`

包含：
- provider ID。
- resolved toolchain/compiler version。
- SDK version。
- channel 与 channel Manifest digest。
- Manifest URL/identity。
- host 和所有 target architectures。
- selected capabilities/features。
- package graph nodes/edges。
- stable topological package order。
- exact payload source list。
- unsupported/conflicting capability errors。
- resolver version/schema。

Resolved 结果不含临时目录，也不依赖当前系统 PATH 顺序。

### 3.3 `InstallPlan`

```rust
struct InstallPlan {
    schema: String,
    plan_id: String,
    created_at: DateTime<Utc>,       // 不参与 plan_id
    request: NormalizedRequest,
    resolved: ResolvedToolchain,
    observations: PlanObservations,
    actions: Vec<PlanAction>,
    disk_usage: DiskUsageEstimate,
    privilege: PrivilegeRequirement,
    warnings: Vec<PlanWarning>,
    blocking_conflicts: Vec<PlanConflict>,
}
```

`plan_id` 由规范化后的稳定序列化数据计算（例如 canonical JSON + SHA256），不包含 created_at、随机 ID、临时路径和进度数据。

`PlanObservations` 必须记录 apply 所依赖的事实：
- manifest digest/channel identity。
- cache blob digest + verified state。
- existing installation ID/receipt digest。
- target root identity（规范化路径、volume/file identity 等可行信息）。
- relevant directory listing/ownership manifest digest。
- required free space snapshot（apply 仍要重新检查）。

`PlanAction` 至少支持：
- `DownloadPayload`
- `ReuseCachedPayload`
- `ReuseInstalledComponent`
- `ExtractPackage`
- `RetainSourceMedia`
- `PrunePath`
- `CreateDirectory`
- `WriteMetadata`
- `BackupOwnedFile`
- `CommitFile`
- `SkipOptionalPackage`
- `Conflict`

每项 action 有稳定 ID、dependencies/action references、source package、path、expected hash/size（适用时）、estimated size、reason、risk。

### 3.4 `InstallReceipt`

每次成功安装/更新产生独立 receipt：
- schema version。
- install ID / transaction ID。
- provider + versions + channel + profile。
- Host/Targets/selected features。
- exact source packages/payload digests。
- extractor backend + backend version。
- pruning profile/rule version/actions。
- installed file manifest digest。
- installed file list（relative path、size、SHA256、source package、ownership IDs）。
- removed file list（relative path、former owner、reason、rule ID、size）。
- environment spec reference/hash。
- completed timestamp。
- transaction commit sequence/status。

不写缓存机密、不把本机绝对临时路径作为跨机器重建凭据。

### 3.5 `InstalledManifest`

建议 JSON 或可压缩格式，包含最终实际文件树：

```json
{
  "schema": "msvc-kit.installed-manifest.v1",
  "install_id": "...",
  "root_relative": ".",
  "files": [
    {
      "path": "VC/Tools/MSVC/.../bin/Hostx64/x64/cl.exe",
      "size": 123456,
      "sha256": "...",
      "owners": ["package-id"],
      "kind": "file"
    }
  ]
}
```

路径全部相对安装 root。不要把目录时间戳当作内容完整性凭据。大量文件 hashing 可以分层缓存，但验证模式必须重新 hash。

### 3.6 `OwnershipIndex`

必须描述每个受管理相对路径由哪些 package/install ID 拥有。支持共享文件：
- 同一目标路径 + 相同 SHA256：允许多 owner 引用。
- 同一目标路径 + 不同 SHA256：默认冲突；不可静默覆盖。
- 卸载/升级时只在最后一个 owner 不再引用且路径 hash 与 receipt 相符时删除。
- 若目标文件已被用户改写，标记 modified/unowned conflict，不删除。

### 3.7 `TransactionJournal`

必须持久记录：
- schema / transaction ID / plan ID。
- target root identity。
- staging path、backup path（均须在受控同卷位置）。
- current state。
- action sequence。
- 每个文件变更前后的预期 hash。
- `BackupMoved`、`NewFileInstalled`、`FileRemoved`、`ReceiptCommitted` 等事件。
- event sequence number 和可校验记录。
- recovery result。

Journal 采用 write-ahead 语义：关键文件移动/替换之前，先持久写入将要执行的操作；操作完成后写完成记录。恢复必须能区分“日志写了但动作没执行”和“动作完成但日志还没写”。可使用文件存在/Hash 状态进行判定。

## 4. 统一目录建议

必须先尊重当前 msvc-kit 实际目录布局。建议保持向后兼容：

```text
<install-root>/
├── downloads/                    # 当前 cache/payload 区域；未来可迁往 cache root
│   ├── msvc/...                  # 可保留旧目录作为兼容映射
│   └── sdk/...
├── VC/                           # 现有 MSVC tree
├── Windows Kits/                 # 现有 SDK tree
├── .msvc-kit/
│   ├── installation.lock
│   ├── transactions/<tx-id>/
│   │   ├── journal.jsonl
│   │   ├── staging/
│   │   └── backup/
│   ├── receipts/<install-id>.json
│   ├── installed-manifests/<install-id>.json
│   ├── ownership.json
│   └── registry.json
└── .msvc-kit-extracted/           # 旧 marker；迁移后只读/兼容清理
```

CAS 可以在默认 cache root 的 `.msvc-kit-cache/blobs/sha256/`，不要强行放进每个安装根；配置已有 cache path 时应尊重其既有约定。

## 5. 并发与锁的建议粒度

- Manifest cache lock：保护一个 manifest URL/hash metadata 更新。
- Blob lock：保护一个 SHA256 blob 的创建/验证/删除。
- Download index lock：保护 redb 事务外的协调操作。
- Install root lock：保护一次 target root 的 transaction，覆盖 Plan revalidation、commit、rollback、receipt update。
- Installation registry lock：保护全局列表变更。

避免用同一把全局锁阻塞所有独立缓存下载。锁顺序必须固定并文档化，避免死锁。禁止基于 lockfile mtime 超时直接删除仍被活跃 PID 持有的锁。

## 6. 错误模型建议

至少区分：
- `UnsafeTargetPath`
- `TargetDirectoryNotEmpty`
- `UnownedPathConflict`
- `ArchivePathTraversal`
- `ArchiveEntryCollision`
- `ManifestResolution`
- `DependencyCycle`
- `UnsatisfiedDependency`
- `PayloadHashMismatch`
- `CacheCorrupt`
- `MsiExtractionFailed { backend, code, log_path }`
- `InstallerBusy`（1618）
- `InsufficientDiskSpace`
- `StaleInstallPlan`
- `TransactionFailed`
- `RollbackFailed`
- `NeedsRecovery`
- `InstalledTreeModified`

按项目既有错误风格实现；公开 API 不得无理由暴露难以稳定的内部 enum 结构。

## 7. 序列化和版本

- 所有 machine-readable schema 使用显式 `schema`/`schema_version`。
- 新 reader 对未来未知字段采取明确策略；当前锁文件 `deny_unknown_fields` 需考虑 v1 compatibility。
- v1 ToolchainLock 必须可读；不支持的 schema 明确报错。
- 新锁建议 `msvc-kit.toolchain-lock.v2`，receipt 使用新版本，InstalledManifest 独立。
- 原子保存：同目录临时文件 → flush/sync → replace；Windows replace 行为要测试。
- 文件 hash 使用 SHA256 lowercase hex。
- JSON key order 需 canonicalize 才能计算稳定 digest；普通 `serde_json::to_vec` map 排序特性需明确，不可依赖随机迭代。
