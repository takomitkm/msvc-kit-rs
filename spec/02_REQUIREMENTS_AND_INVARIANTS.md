# 产品需求、范围与不可违反的不变量

## 1. 产品目标

将 msvc-kit 从“MSVC + Windows SDK 下载/激活工具”演进为一个安全、可复现、可诊断、可扩展的 Windows 用户态开发工具链 provisioning manager。

重点不是堆叠 CLI 参数，而是统一生命周期：

```text
Discover → Resolve → Plan → Acquire → Verify → Extract
         → Prune → Validate → Commit → Activate → Maintain/Recover
```

首要目标是继续可靠管理 MSVC 和 Windows SDK；LLVM、MinGW、CMake、Ninja、NASM 的通用接口属于后续扩展目标，不应为了架构抽象破坏当前主路径。

## 2. 功能范围

### 必须交付

1. **安全提取**：MSI/VSIX/ZIP/CAB 提取到受控 staging；防归档路径穿越；记录输出文件；不污染正式目录。
2. **路径与所有权保护**：拒绝危险目录、未知非空目标、外部安装目录写入和未经登记的文件覆盖。
3. **统一 InstallPlan**：下载、缓存命中、已有工具链复用、提取、裁剪、磁盘、冲突、权限需求均进入一个可序列化计划。
4. **事务安装**：可中断恢复、rollback 或安全进入 needs_recovery；不删除未知文件。
5. **MSI backend abstraction**：保留 `msiexec /a`，评估独立 Rust MSI database + CAB extraction。native backend 没有足够 parity evidence 时不可默认启用。
6. **裁剪引擎**：可审计、可 Dry Run、规则版本化、可恢复，默认保留调试能力。
7. **CAS 缓存**：以 SHA256 作为 payload 内容身份；保留 redb 做来源和引用索引；安全 GC。
8. **Lockfile / Receipt**：不仅锁选择和下载源，还能验证最终安装树、pruning 和环境。
9. **依赖解析**：Build Profile/capability → Manifest packages 的 dependency closure。
10. **环境探测**：VS/MSVC/SDK 起步，逐步扩展 LLVM/MinGW/build tools；区分发现、验证、可复用。
11. **Provider 抽象**：管理生命周期统一，但允许不同 Provider 有不同下载/安装后端。
12. **自动测试**：离线单元、Windows integration、fault injection、兼容测试和明确 opt-in 的真实下载测试。

### 不在第一版交付范围

- 不开发通用 Windows 软件包仓库或替代 winget/choco/scoop。
- 不默认安装 WDK/EWDK、驱动签名工具、调试工具或 Visual Studio IDE。
- 不自动修改机器级 PATH、HKLM 或系统注册表。
- 不自动卸载/修复用户已有 Visual Studio/SDK/LLVM/MinGW。
- 不自动推断任意源码项目完整的所有编译 feature；输入应是 Profile、CMake metadata 或用户参数。
- 不承诺任意 MSI 都无 UAC、任意 MSI 都可被 native backend 完整提取。
- 不默认删除全部 PDB、DIA、Debug CRT 或开发者调试工具。
- 不把所有 payload 安装目录做 hardlink/reflink 去重。
- 不更改 Microsoft 官方包内容、签名或许可条款。

## 3. 行为不变量（任何阶段都不得破坏）

### I-001：目标目录不可被盲目覆盖

- 不存在：可以建立。
- 存在且为空：经规范化与安全检查后可以使用。
- 存在且包含当前 msvc-kit 管理的文件：只允许按 ownership receipt 精确增删改。
- 存在未知内容：默认拒绝冲突文件，不整体删除。
- 其他版本/缓存/日志/用户文件必须保留。
- 不能仅凭目标内存在一个名字叫 `.msvc-kit` 的文件就信任其所有权。

### I-002：共享根目录不能整体替换

当前 msvc-kit root 同时包含缓存、多个版本/组件、receipts、索引和配置。不得 `rename(staging_root, target_root)` 或先 `remove_dir_all(target_root)`。事务只能对明确拥有的相对文件路径操作，或引入单独的 immutable generation 目录并经过兼容迁移。

### I-003：外部工具链只读

系统中探测到的 VS/SDK/LLVM/MinGW 默认只读。没有明确用户选择，不得复制、清理、升级、修改注册表或替换其文件。

### I-004：完整性证据分层

必须分别记录：

1. Manifest identity/integrity。
2. 下载 archive 官方 SHA256 验证。
3. 解包器 backend 和版本。
4. extracted file tree hash/metadata。
5. pruning rules version 和 removed files。
6. 最终 installed manifest hash。

下载 SHA256 不能替代 installed tree 验证。

### I-005：Plan 与 Apply 一致

Dry Run 和实际 Apply 必须使用同一 resolved package set、manifest digest、source URLs、hashes、actions。Apply 前重新验证计划依赖的外部状态；若目标/缓存/manifest/锁有变化且计划不再成立，返回 stale plan 并重新计划，不允许静默换包。

### I-006：失败无假成功

任何下载、hash、提取、pruning、验证、提交错误都必须导致明确失败状态。若无法完成 rollback，交易状态必须持久写成 `needs_recovery`，下次命令不得继续覆盖。

### I-007：缓存不是可信来源本身

缓存命中必须验证 source identity 和文件内容。文件名相同、文件大小相同、mtime 未变都不够。预期 digest 缺失时按明确的 unverified policy 处理，默认严格模式不能把未知文件当作 verified。

### I-008：CAS Blob 不等于安装目录硬链接

下载归档可以按 SHA256 去重；最终可变工具链文件默认 copy/extract 到独立目录。不得用会互相影响写入的硬链接连接多个工具链安装实例。

### I-009：并发保护正确

锁必须围绕实际共享状态和事务生命周期，不允许 stale-lock TTL 机制删除另一个仍活跃进程的锁。优先 OS file locks / kernel locks；若锁是 lockfile，使用独立 owner/token + process liveness + crash recovery 协议并测试竞争条件。

### I-010：Windows path semantics

路径逻辑须考虑：大小写不敏感、盘符、UNC、`\\?\`、短路径、ADS（冒号）、保留设备名、尾随点/空格、Unicode normalization、junction/reparse points、符号链接、相对路径、父目录穿越。

### I-011：默认无管理员

普通用户安装到用户可写目录，不主动提权。不调用 `msiexec /i`。`msiexec /a` 仍须按特定 MSI 测试，不保证任何 MSI 永不提示 UAC。

### I-012：确定性

给定相同 Manifest snapshot、BuildProfile、当前目标状态、缓存验证状态和配置，Dependency Resolution 与 InstallPlan 的业务结果必须确定。时间戳、临时路径等非决定因素不应影响 Plan identity。

### I-013：现有契约向后兼容

CLI selector、环境变量、JSON schema、exit code、Rust public API、Toolchain Lock v1、Receipt v2 要有兼容策略和测试。

### I-014：可观测

每个 transaction 有 ID；每个下载/提取/裁剪/提交 action 有状态、耗时和关联 ID。日志不得泄露 token/敏感 header；路径可展示但注意隐私。

## 4. 性能和容量要求

- 下载仍支持并发和流式文件写入，不把数 GB payload 全部载入内存。
- 解包的内存占用由 buffer 和并发限制控制。
- Plan 要估算 download size、cache reuse、staging size、backup size、peak required space、final installed size。
- Disk estimate 必须说明不确定性，例如压缩包未知展开比例时报告下界/上界或 unknown，不伪造精确值。
- CAS GC 必须可 Dry Run；绝不在未确认引用关系时删 Blob。

## 5. 易用性要求

- 默认配置保持适用于普通 MSVC C/C++ 用户，不要求用户理解 VS Package ID。
- 高级功能提供清晰的子命令和 JSON API。
- `plan` 默认不写目标目录，不下载大 payload；只允许读取必要的 Manifest/缓存元数据。
- `apply` 不可在 Plan stale 时自动重新选择 latest。
- 错误消息说明原因、受影响路径、已执行动作、可执行恢复命令。
- 进度输出到 stderr；机器 JSON 到 stdout，避免混流。

## 6. 明确状态机

Transaction 状态：

```text
Created
  → Planned
  → Acquiring
  → Acquired
  → Extracting
  → Extracted
  → Pruning
  → Validating
  → ReadyToCommit
  → Committing
  → Committed

任意提交前阶段失败：Failed / Aborted（staging 可安全清理）
提交过程中失败：RollingBack → RolledBack
回滚不能完成：NeedsRecovery（禁止自动开始新 transaction）
```

Action 状态：`Pending | Running | Reused | Succeeded | Failed | Skipped | RolledBack`。

## 7. 安全优先级

P0：归档路径穿越、未知文件覆盖、共享根目录误删、失败后半安装、缓存哈希绕过。  
P1：回滚中断、重入/并发、Lockfile/Receipt 伪造、junction/reparse 绕过。  
P2：资源耗尽、压缩炸弹、超长路径、日志/诊断信息不足。  
P3：CLI ergonomics、更多 provider、可视化依赖图。
