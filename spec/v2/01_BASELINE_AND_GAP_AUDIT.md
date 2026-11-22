# 当前基线与差距审计（参考快照）

**审计快照日期：2026-11-22。** 本工作包参考的上游代码快照：
- `loonghao/msvc-kit`: `87dc08ce510a11a874f8e088f17c4be37ee01672`
- `tgbender/portablemsvc`: `92554c094a1d00abef1cfb610877edcb1eb325f3`

后续 `main` 可能变化；用户 fork 的 HEAD 也可能不同。Agent 必须重做本地审计。本文件提供检查入口，不是让 Agent 直接照抄的“最终事实”。源码引用优先使用上述 commit permalink；不要把本地 fork 与参考快照混为一谈。

## 1. msvc-kit 已经具备的能力

参考：
- [README](https://github.com/loonghao/msvc-kit/blob/87dc08ce510a11a874f8e088f17c4be37ee01672/README.md)
- [Toolchain Contracts](https://github.com/loonghao/msvc-kit/blob/87dc08ce510a11a874f8e088f17c4be37ee01672/docs/guide/toolchain-contracts.md)
- [downloader/mod.rs](https://github.com/loonghao/msvc-kit/blob/87dc08ce510a11a874f8e088f17c4be37ee01672/src/downloader/mod.rs)
- [downloader/common.rs](https://github.com/loonghao/msvc-kit/blob/87dc08ce510a11a874f8e088f17c4be37ee01672/src/downloader/common.rs)
- [downloader/index.rs](https://github.com/loonghao/msvc-kit/blob/87dc08ce510a11a874f8e088f17c4be37ee01672/src/downloader/index.rs)
- [installer/mod.rs](https://github.com/loonghao/msvc-kit/blob/87dc08ce510a11a874f8e088f17c4be37ee01672/src/installer/mod.rs)
- [installer/extractor.rs](https://github.com/loonghao/msvc-kit/blob/87dc08ce510a11a874f8e088f17c4be37ee01672/src/installer/extractor.rs)
- [toolchain_lock.rs](https://github.com/loonghao/msvc-kit/blob/87dc08ce510a11a874f8e088f17c4be37ee01672/src/toolchain_lock.rs)

已知存在的基础：

| 能力 | 快照中的实现 | Agent 要确认什么 |
|---|---|---|
| VS Manifest 获取/解析 | `downloader/manifest.rs` 等 | 当前 channel、digest 和包选择行为 |
| MSVC/SDK 下载器 | `MsvcDownloader`、`SdkDownloader` | 下载和提取分离点、目录布局 |
| 组件类别 | `MsvcComponent`（Spectre/MFC/ATL/ASAN/UWP/CLI/Modules/Redist/Custom） | 实际 package matching 是否完整 |
| Host/Target 架构 | `Architecture` 与 manifest filter | 所有合法组合、cross-target 配置 |
| 下载并发 | `CommonDownloader`，含并发调节 | 取消、错误和索引一致性 |
| 下载索引 | redb `DownloadIndex` / `IndexEntry` | key、路径、迁移和锁范围 |
| 下载哈希 | 预期/实际 SHA256 | 无官方 SHA 时的策略是否安全 |
| Manifest 缓存 | ETag / Last-Modified、哈希校验 | 304、签名 fallback、失效策略 |
| MSI 提取 | Windows `msiexec /a`；非 Windows `msiextract` | 具体参数、日志、权限和错误传播 |
| VSIX/ZIP、CAB | Rust `zip` / `cab` | Zip Slip 和路径冲突安全 |
| SDK MSI 处理 | CAB 保留作 source media；部分可选 MSI 跳过 | CAB 关联和具体包策略 |
| 提取标记 | `.msvc-kit-extracted` | 标记是否可证明输出内容完整 |
| 安装锁 | `.msvc-kit/installation.lock` | 锁的粒度、恢复与死锁行为 |
| Lockfile | `msvc-kit.toolchain-lock.v1` | v1 兼容、receipts 完整性 |
| Receipt | `msvc-kit.installation-receipt.v2` | 目前证明下载包来源，不证明最终树未改 |
| Dry Run | `DownloadOptions.dry_run`、`DownloadPreview` | CLI 是否暴露完整计划；下载预览有何缺项 |
| VS 探测 | `find_vs_instances()` | 这是 VS/UBT 相关探测，不是全工具链探测 |
| 诊断 | `doctor` / `doctor --compile` | 覆盖平台、fixture、实际验收边界 |
| 环境查询/运行 | `query`、`env`、`run`、`cmake` | API 兼容、JSON schema、环境变量行为 |
| Bundle | 便携 bundle | 是否与新 InstallPlan/receipt 接合 |

### 关键实现缺口/风险入口

以下在参考快照中能看到，需要本地复核：

1. **下载目录与安装根目录共存。** `DownloadOptions.target_dir` 同时用作安装根，下载器在其下创建 `downloads/msvc/...`、`downloads/sdk/...`，工具链提取写入 `VC/...`、`Windows Kits/...`，还存在 `.msvc-kit/` 收据和 `.msvc-kit-extracted/` 标记。不得整体替换 target root。
2. **提取目前直接写正式 target root。** `extract_and_finalize_msvc`、`extract_and_finalize_sdk` 调用并行包提取，安装包文件在共享目录内写入；失败可能留下部分文件。
3. **提取 marker 指纹较弱。** 快照中的 marker 基于 source path、size、mtime 计算，不等于源归档内容 SHA256，也不验证输出文件是否仍存在/未被修改。
4. **MSI Windows backend 是系统 `msiexec /a`。** 不是 Rust 原生 MSI 数据库解析器。`/qn` 只抑制 UI，不保证任何 MSI 在所有情形都不要求提升。
5. **Archive path safety 必须实测。** 快照的 VSIX ZIP 代码使用 `target_dir.join(relative_path)`；Agent 必须检验是否有完整防穿越保护和归档文件名冲突检查。不能仅凭依赖库或测试名认为安全。
6. **目录保护与事务回滚没有等价完整机制的证据。** installation lock 是进程并发保护，不是 rollback。
7. **缓存不是全局 CAS Blob Store。** 有 redb 索引和 hash verify，但下载文件仍处于组件版本/架构目录和 manifest payload 文件名路径；不能把它说成跨所有包全局按内容散列去重。
8. **Lockfile receipt 与 installed tree 是两个概念。** 当前 receipt 证明归档获取来源/哈希；不证明后续安装目录没有修改。
9. **组件依赖字段存在不等于递归依赖图解析。** `VsPackage.dependencies` 有数据字段，但需实测选择逻辑是否计算依赖 closure、冲突、循环和排序。
10. **现有 VS instance discovery 范围有限。** `vswhere`/Registry/常见路径探测是 VS 安装发现能力，不是 LLVM、MinGW、CMake、Ninja、NASM 的统一探测器。
11. **`ComponentDownloader` 不是完整 Provider 生命周期。** 当前 enum 只有 MSVC/SDK；trait 以 download 为中心，而非 Detect/Resolve/Plan/Apply/Verify/Activate/Clean 全生命周期。

## 2. portablemsvc 可借鉴能力

参考：
- [README](https://github.com/tgbender/portablemsvc/blob/92554c094a1d00abef1cfb610877edcb1eb325f3/README.md)
- [extract.py](https://github.com/tgbender/portablemsvc/blob/92554c094a1d00abef1cfb610877edcb1eb325f3/src/portablemsvc/extract.py)
- [install.py](https://github.com/tgbender/portablemsvc/blob/92554c094a1d00abef1cfb610877edcb1eb325f3/src/portablemsvc/install.py)
- [download.py](https://github.com/tgbender/portablemsvc/blob/92554c094a1d00abef1cfb610877edcb1eb325f3/src/portablemsvc/download.py)
- [lockfile.py](https://github.com/tgbender/portablemsvc/blob/92554c094a1d00abef1cfb610877edcb1eb325f3/src/portablemsvc/lockfile.py)
- [controller.py](https://github.com/tgbender/portablemsvc/blob/92554c094a1d00abef1cfb610877edcb1eb325f3/src/portablemsvc/controller.py)
- [parse_msi.py](https://github.com/tgbender/portablemsvc/blob/92554c094a1d00abef1cfb610877edcb1eb325f3/src/portablemsvc/parse_msi.py)
- [install_status.py](https://github.com/tgbender/portablemsvc/blob/92554c094a1d00abef1cfb610877edcb1eb325f3/src/portablemsvc/install_status.py)

### 2.1 具体值得借鉴的内容

| 设计 | portablemsvc 快照 | 移植要求 |
|---|---|---|
| MSI backend 抽象 | `MsiExtractor` Protocol | Rust trait/backend enum |
| MSI 默认路径 | `python-msi/pymsi` 纯 Python提取 | 研究 native Rust 后端，不照搬 Python 依赖 |
| msiexec backend | `/a` administrative extraction | 保留为可用 backend |
| fallback | native/primary 失败后 msiexec | 失败类型分层，避免错误 fallback 隐藏损坏 |
| 安全归档路径 | `_safe_destination_path` | Rust 所有 extractor 一致实现 |
| 临时输出 | 先提取到同级临时目录 | 扩展为真正 rollback-safe transaction |
| 输出目录保护 | 拒绝根目录、保护 cwd/home/cache/config/data，拒绝未知非空目录 | 使用 msvc-kit ownership receipt；加强 reparse/symlink 检查 |
| 下载 CAS 思路 | `{expected_sha256}{extension}` 缓存文件，hash-to-names map | 统一 SHA256 Blob Store + redb 元数据 |
| 下载重试/Range | 1 MiB chunks，失败时 Range 尝试续传 | 只实现协议正确、持久、可验证的 resume；不可照抄内存 BytesIO 方案 |
| Lockfile | source manifests、URL/SHA、文件类型、CAB parent、解包路径/顺序、removed files、env spec、install ID、工具版本 | 形成版本化 source lock + installed receipt |
| pruning | 按架构/目录/文件规则裁剪，记录删除内容 | 规则化、可预览、可测试、默认保守 |
| install registry | installed.json / UUID side-by-side | msvc-kit installation registry 或现有索引扩展 |
| 激活 | env spec 与 CMD/Powershell scripts | 与已有 msvc-kit env API 合并，不重复造脚本 |

### 2.2 必须不要照搬的弱点

portablemsvc 是设计参考，不是完美实现：

- 它的输出替换流程会先 `shutil.rmtree(output_dir)`，再 rename staging；rename 如果失败，旧输出已经删除。因此它不是完整事务回滚。msvc-kit 必须做比它更强的 journal/backup/rollback。
- 它用 `portablemsvc.lock` 中 `lockfile_version == 1.0` 作为已管理目录的初步识别，容易被伪造。msvc-kit 应校验安装 ID、root identity、receipt digest、路径清单和 ownership registry。
- 下载缓存路径带扩展名：hash + extension；hash-to-names JSON 不是成熟的多进程数据库。msvc-kit 保留 redb，做全局 hash blob index。
- `download.py` 把下载内容累计在 `BytesIO`，它的 Range 重试不等于磁盘持久断点续传；新实现不要把大 MSI/CAB 全部放 RAM。
- `parse_msi.py` 通过扫描 `.cab` 字节片段猜 CAB 名字，不应作为 MSI parser 设计。必须解析 MSI 数据库 Media/Sequence/Streams 结构。
- `install.py` 的 pruning 路径和包结构属于特定版本假设。必须由 Manifest/安装树/构建探针验证后才能删除。
- README 提到 debug symbols cleanup，但代码不能证明所有 PDB 都被删除；不能自行扩大到 `*.pdb` 全删。
- `install_status.py` stale lock TTL 删除锁文件的做法可能有并发风险，不应照搬。

## 3. 六项用户目标的准确范围

| 目标 | 定义 | 不能误认为 |
|---|---|---|
| 依赖图解析 | 从 Profile/capability 计算 package dependency closure、架构和版本约束 | 组件字符串匹配 |
| 环境探测 | 发现并验证机器已有工具链，判断能否 read-only reuse | PATH 找到一个 exe 就算完整 |
| 空间优化 | 下载内容去重、现有组件复用、裁剪和 peak space estimate | 单纯减少 zip 大小 |
| 安装计划预览 | Plan 同时展示 download/reuse/extract/prune/conflict/space/privilege | 仅显示包数量/总下载大小 |
| 事务式安装 | 可持久恢复的 staging + journal + ownership-aware commit/rollback | 有一个 lock file 或临时目录 |
| 统一工具链接口 | Provider lifecycle 可扩展编译器、SDK 和 build tools | 把所有类型塞进一个 enum |

## 4. UAC/权限基线

- `msiexec /a` 表示 administrative installation / administrative image extraction，不是 `/i` 产品安装。
- `/qn` 意味无界面，不是“禁止 elevation”。
- `/a` 本身不自动等于管理员权限；目标目录 ACL、具体 MSI AdminExecuteSequence/custom action 仍可能导致需要提升或失败。
- 默认只安装/提取到当前用户可写目录，例如 `%LOCALAPPDATA%` 下的 msvc-kit root。
- 默认不得自动触发 UAC 或用 `runas` 重启自身。
- `install-into-vs`、机器级环境变量等特权功能继续单独明确标注。
- 需要在标准用户账户、管理员但未提升账户以及显式 elevation 测试场景分别测试；绝不通过普通 unit test 宣称“任何 MSI 均无 UAC”。

Microsoft docs:
- https://learn.microsoft.com/en-us/windows/win32/msi/command-line-options
- https://learn.microsoft.com/en-us/windows/win32/msi/administrative-installation
- https://learn.microsoft.com/en-us/windows/win32/msi/using-windows-installer-with-uac

## 5. 最终审计结论的写法

Agent 必须避免“这个项目没有 X”这种未经核验的绝对说法。使用以下状态：

- `EXISTING_AND_VERIFIED`：代码与测试均证实满足要求。
- `PARTIAL`：有实现基础，但覆盖不了需求。
- `MISSING`：检查源代码、入口和测试后，没有找到实现。
- `BROKEN`：有实现但存在可复现失败/安全问题。
- `NOT_TESTED`：找到实现但没有可执行证据。
- `UNKNOWN`：因代码未获取/平台不能运行等原因无法确定。

所有状态必须给出路径、符号和测试证据。
