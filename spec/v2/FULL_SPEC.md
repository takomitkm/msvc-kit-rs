# msvc-kit Fork 工程规格（合并版）

> 本文件由分文件规格合并生成，便于整体阅读。分文件是维护源；若修改规格，必须同步重建本文件。


---

<!-- SOURCE: 00_AGENT_MASTER_PROMPT.md -->

# Agent 主指令：自主实施 msvc-kit Fork 工程改进

你是该仓库的实现工程师、测试工程师和代码审查者。用户不会逐项指导你。你的任务是**阅读规格、检查真实代码、按阶段实施、自己设计并运行测试、修复失败、提交可审查的代码和报告**。

## A. 项目背景与归因

用户 fork 了 `loonghao/msvc-kit`，计划借鉴 `tgbender/portablemsvc` 的安装器设计，把两者有价值的做法结合起来：

- `portablemsvc`：纯 Python `python-msi/pymsi` MSI 解包路径、可切换 MSI backend、SHA256 内容缓存、详细 Lockfile（源、包、解包顺序、移除记录、环境）、文件裁剪、安装 ID、输出目录保护、临时目录提取。
- `msvc-kit`：Rust CLI/Library、Visual Studio Manifest 解析、MSVC/Windows SDK 组件与架构选择、redb 下载索引、并发下载、哈希验证、Manifest 缓存、doctor/query/run/CMake/Ninja/Bundle/自更新等。

**必须如实注明设计参考：** 本项目的安装器设计明确参考 `tgbender/portablemsvc`。不要隐藏来源，也不要声称这些设计全部由本 fork 首创。若复制任何代码，先检查其 LICENSE，并保留要求的版权/许可证声明。本工作默认只借鉴设计，不直接复制 Python 源码。

参考仓库：
- https://github.com/tgbender/portablemsvc
- https://github.com/loonghao/msvc-kit
- https://github.com/mmozeiko/portable-msvc （portablemsvc 的早期灵感来源；只作历史参考）

## B. 执行时不可违反的规则

### B1. 仓库保护

开始前必须执行并记录：

```text
git status --short --branch
git rev-parse HEAD
git remote -v
git branch --show-current
```

要求：

- 不修改 Git remote，不添加上游 remote，不改变默认分支。
- 不 push，不 force-push，不创建 GitHub PR/Issue，不发布 release。
- 默认不 commit；只有用户明确要求时才 commit。
- 不执行 `git reset --hard`、`git clean -fdx`、强制 checkout 或覆盖用户文件。
- 如果工作区有未提交修改，先识别并保护；不得覆盖或“顺手格式化”用户改动。
- 不自动切换分支。只在当前用户 fork 工作分支工作。
- 需要生成文件时先确认目标路径不存在，或按安全的编辑方式更新。

### B2. 不要求用户作技术决定

本工作包已经规定默认行为。遇到普通设计选择时，按本规格选择并记录到 `DECISIONS.md`，不要反复问用户。只有遇到无法安全继续、涉及数据损失、需要真实管理员操作或需要大量外网下载时才停下来报告。

### B3. 不得假定规格中的基线仍然最新

在本地 checkout 中重新搜索代码、测试和文档。若发现某项已实现：

1. 不要重复实现。
2. 找到实际实现和对应测试。
3. 检查是否满足本规格的行为和验收标准。
4. 满足则标记 `EXISTING_AND_VERIFIED`。
5. 部分满足则标记 `PARTIAL`，只补缺口。
6. 不满足或存在 bug 才标记 `IMPLEMENT` / `FIX`。

报告中必须列出证据：文件路径、符号名、测试名、实际命令结果。不能只写“看起来已经有”。

### B4. 每个阶段先测试后改代码

每阶段执行：

1. 阅读本阶段规格和涉及源文件。
2. 写阶段计划（文件、函数、测试、风险）。
3. 运行基线相关测试并记录结果。
4. 实现最小改动。
5. 新增测试，先验证测试能检测错误，再实现/修复。
6. 运行新增测试。
7. 运行受影响模块回归。
8. 运行项目规定的完整验证命令。
9. 检查 `git diff --check`、`git status`、diff 内容。
10. 更新 `PROGRESS.md`、`TEST-RESULTS.md`、阶段报告。

如果测试失败：先判断是新回归、旧基线失败、环境限制还是测试本身错误。不得简单删除/弱化测试来得到绿色结果。若因环境无法运行，标记 `BLOCKED_ENVIRONMENT`，列出缺失工具和替代证据；不得标记通过。

**重要：单进程中注入 `Err` / panic 只能测试错误回滚，不足以证明 crash recovery。** 对事务关键提交点，必须至少增加子进程测试：子进程执行到指定 journal/fsync 边界后由父进程强制终止，再启动新进程执行 recovery 并验证文件 hash。若当前平台不能可靠做进程终止测试，明确标记 `BLOCKED_ENVIRONMENT`，不得声称 crash-safe。

### B5. 禁止危险实机操作

默认只允许 fixture、mock、临时目录和普通用户权限测试。

严禁未经用户明确批准执行：
- 将 MSVC 写入 `Program Files` 或 Windows 系统目录。
- `msiexec /i` 安装产品、修改系统级 MSI 状态或注册产品。
- 修改 HKLM、系统 PATH、机器级环境变量、Windows SDK/VS 原有目录。
- 管理员提升、修改 UAC、安全策略、系统注册表。
- 删除、覆盖用户现有的 VS/MSVC/SDK/LLVM/MinGW。
- 自动下载数 GB 的完整真实工具链。

允许在显式 opt-in、专用测试机、用户可写临时路径下运行经过审查的 `msiexec /a` administrative extraction 测试；不得调用 `/i`。

### B6. 网络测试规则

- 常规单元测试必须离线、确定性、快速。
- CI 不得默认下载数 GB 真实 MSVC/SDK。
- 真实微软 Manifest/包下载测试必须单独标记为 `network_heavy` / `manual` / `scheduled`，并要求显式环境变量确认。
- 测试只可从官方 Manifest 声明 URL 下载。不得从不明镜像替代。
- 若没有真实 SDK/MSI fixture，不得伪造一个空 MSI 当作真实 MSI 测试。

### B7. API 与行为兼容

- 优先 additive change，保持现有 CLI 和公开 Rust API 源码兼容。
- 不随意改变默认目录、现有环境变量、JSON schema、退出码、CLI 参数含义。
- 必须修改时提供迁移说明和向后兼容测试。
- 不允许新增破坏性 `DownloadOptions` / `ToolchainRequest` 字段导致外部 struct literal 用户无声编译失败；评估 builder、新 options type 或 non_exhaustive 策略。
- 保留用户级安装默认值；不要把 `install-into-vs` 这种特权流程混进普通 download。

## C. 第 0 阶段：必须先完成本地审计

阅读：
- `README.md`、`README_zh.md`
- `Cargo.toml`、`Cargo.lock`
- `justfile`、`vx.toml`、CI workflow
- `docs/guide/toolchain-contracts.md`
- `src/downloader/**`
- `src/installer/**`
- `src/toolchain_lock.rs`
- `src/storage/**`
- `src/toolchain/**`
- `src/install_into_vs/**`
- 当前相关测试目录

用 `rg` / `git grep` 搜索以下概念：

```text
msiexec, extract_msi, extract_vsix, extract_cab, TARGETDIR
DownloadIndex, IndexEntry, FileSystemCacheManager, CacheManager
ToolchainLock, InstallationReceipt, LockedPayload, receipt
extraction_marker, .msvc-kit-extracted
find_msvc_packages, dependencies, MsvcComponent
find_vs_instances, vswhere, Registry
DownloadPreview, dry_run, doctor, query, run, cmake, clean
```

交付 `agent-work/BASELINE.md`：
- 当前 HEAD、分支、dirty state。
- 当前 Rust/toolchain 版本、CI 命令。
- 模块树和关键入口。
- 每项功能 `EXISTING_AND_VERIFIED / PARTIAL / MISSING / BROKEN / NOT_TESTED` 状态。
- 每项证据路径和测试结果。
- 不能核实的地方写 `UNKNOWN`，不要猜。
- 明确指出实际下载目录和最终安装目录的布局。

Phase 0 未通过前，不开始 Phase 1。

## D. 实施顺序

严格遵守 `09_IMPLEMENTATION_ROADMAP.md`。不得跳过依赖前置阶段：

- P00：本地审计、基线测试和风险清单。
- P01：数据模型、路径安全、归档安全测试。
- P02：统一安装计划与 CLI Dry Run。
- P03：Staging、文件所有权、事务提交与恢复。
- P04：MSI backend 抽象与 native MSI parser 可行性；不成熟则保留 msiexec。
- P05：PruningEngine。
- P06：内容寻址 Blob Cache 与安全 GC。
- P07：Lockfile/Receipt/Installed Manifest 扩展。
- P08：Build Profile 和 Manifest 依赖解析。
- P09：环境探测和安全复用。
- P10：Provider API（先 MSVC/SDK，再扩展 LLVM/MinGW/Build Tools）。
- P11：端到端回归、文档和发布准备。

每阶段必须达到 `09_IMPLEMENTATION_ROADMAP.md` 的退出条件才能进入下一阶段。

## E. 代码质量底线

- 错误使用项目现有 `MsvcKitError` / `Result` 风格；错误必须包含可诊断上下文。
- 不用 `unwrap()` / `expect()` 处理外部输入、文件系统、下载、MSI 或归档内容；测试代码允许。
- 异步代码不要把长时间同步文件 IO 无保护放在 Tokio runtime 核心线程；使用现有 `spawn_blocking` 约定或明确选择。
- 文件路径使用 `Path` / `PathBuf`，不要用字符串拼接做安全判断。
- Windows 路径比较必须考虑不区分大小写、盘符、UNC、扩展路径、junction/reparse point。
- 归档解包必须防止 Zip Slip、路径穿越、绝对路径、UNC、盘符路径、ADS、reparse point 跳出 staging。
- 所有缓存和收据写入采用项目统一 atomic-write 方案；不能在正式路径直接截断写入。
- 所有可并发状态修改必须使用已有锁设施或新增明确锁协议。
- 没有用到的 dependency 不得添加；新增 crate 必须记录用途、许可证、维护状态、MSRV、feature 体积和替代品。
- 不要把 `msi` crate 的 MSI 数据库解析能力误说成完整 MSI 安装/提取能力；CAB、Media、Sequence、Directory、Component、File 等表需要完整处理。
- 不得无测试地修改 pruning 规则；每条规则需要 fixture 和构建验收证据。

## F. 统一完成标准

最终完成时，必须满足：

1. `cargo fmt --check` 通过。
2. 项目规定的 lint/clippy 通过。
3. 项目现有单元测试通过；基线失败另列，不可掩盖。
4. 所有新增测试通过。
5. Windows 专用测试在 Windows CI 或本机 Windows 运行；未运行则不得写“全平台通过”。
6. 故障注入测试证明事务失败时恢复旧状态，或者明确进入可恢复 `needs_recovery` 状态。
7. 非空未知目录测试证明不会覆盖。
8. 默认测试不进行真实大体积下载。
9. 现有 CLI/API/JSON/lock v1 兼容测试通过。
10. 文档、示例、帮助文本更新。
11. `git diff --check` 无空白错误。
12. `git status` 中没有临时包、真实下载、密钥、日志或不相关改动。

最终输出：
- `agent-work/FINAL_REPORT.md`
- 已完成阶段和未完成阶段。
- 每个测试命令、通过/失败/跳过和原因。
- 重要 diff 摘要。
- 安全与兼容性风险。
- 已知限制。
- 用户需要手工进行的唯一动作（若有）。

**现在从 P00 开始。先审计，不要一上来改代码。**


---

<!-- SOURCE: 01_BASELINE_AND_GAP_AUDIT.md -->

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


---

<!-- SOURCE: 02_REQUIREMENTS_AND_INVARIANTS.md -->

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


---

<!-- SOURCE: 03_TARGET_ARCHITECTURE_AND_SCHEMAS.md -->

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


---

<!-- SOURCE: 04_PLAN_AND_DEPENDENCY_RESOLVER.md -->

# Install Plan、Dry Run、Build Profile 与 Dependency Resolver

## 1. 目标

用户描述“我要编译什么”，不是背诵 Visual Studio package ID。系统必须将声明的构建需求转换成 capability 集，再转换为精确的 MSVC/SDK Manifest package/payload 集合。

`plan` 是下载、复用、提取、裁剪和提交的唯一事实来源。Dry Run 不是另写一套展示逻辑。

## 2. CLI 目标（最终形态）

现有 CLI 必须兼容。以下是目标命令；按阶段增加，不要一次性实现所有 flag：

```powershell
# 默认标准 C++ 桌面工具链计划
msvc-kit plan --profile cpp-desktop

# 指定版本、host/target、feature
msvc-kit plan --profile cpp-desktop `
  --msvc-version 14.44 `
  --sdk-version 10.0.26100.0 `
  --host-arch x64 `
  --arch arm64

# 机器 JSON 输出
msvc-kit plan --profile cpp-desktop --format json --output plan.json

# 显示 plan 内容，不写安装目录
msvc-kit plan --show

# 应用已经生成的精确计划
msvc-kit apply plan.json

# 检查计划是否仍然有效
msvc-kit plan verify plan.json
```

如果当前 Clap 命令组织不能支持嵌套 `plan verify`，可采用 `msvc-kit verify-plan plan.json`；选择一种并在 CLI 文档中保持一致。

## 3. Profile 模型

首批 Profile 仅用于用户友好选择，不应内置大量未经验证的 package ID。

| Profile ID | 表达的需求 | 备注 |
|---|---|---|
| `c-native` | 基础 Windows C 编译 | 最小可运行验证目标 |
| `cpp-native` | 常规 C++ 原生构建 | 默认 MSVC + SDK |
| `cpp-desktop` | 桌面应用 C++ | Windows SDK 常规 headers/libs/tools |
| `cpp-mfc` | MFC/ATL 项目 | 必须明确启用 MFC/ATL |
| `cpp-spectre` | Spectre mitigation libraries | 与编译 `/Qspectre` 使用相关 |
| `cpp-asan` | AddressSanitizer | 依据实际 MSVC 支持版本 |
| `cpp-modules` | C++20/23 modules | 标记版本/实验限制，不保证所有 project 自动可用 |
| `cross-arm64` | x64 Host → ARM64 Target | Host 工具 + ARM64 libs/SDK |
| `custom` | 用户指定 feature/capability | 不隐藏 package ID 作为高级 escape hatch |

Profile 可以通过 YAML/TOML 文件声明：

```toml
schema = "msvc-kit.build-profile.v1"
id = "cpp-desktop"
provider = "msvc"

[compiler]
language = ["c", "cpp"]
standard = ["c17", "c++20"]

[target]
architectures = ["x64"]

[features]
required = ["msvc.compiler", "msvc.crt", "windows-sdk.headers", "windows-sdk.um-libraries", "windows-sdk.ucrt"]
optional = []

[pruning]
profile = "portable"
```

Profile 声明 capability，不直接 hardcode package file name。Manifest resolver 映射 capability ↔ package/component 时，必须给出证据或官方定义，不得猜名称。

## 4. Dependency Graph

### 4.1 图模型

Node 可表示：
- `Capability`（例如 `msvc.compiler`, `windows-sdk.headers`）
- `ManifestPackage`（Visual Studio package ID）
- `Payload`（VSIX/MSI/CAB/ZIP）

Edge 类型：
- `Requires`：必须依赖。
- `Optional`：仅在 feature 选择时依赖。
- `ConflictsWith`：不能同时选择。
- `Provides`：package 提供 capability。
- `HostConstraint` / `TargetConstraint`：架构适用约束。
- `VersionConstraint`：依赖版本约束。

最好把 capability graph 和 Manifest Package graph 分开，避免 Manifest 字段表达能力不足时把 feature 规则塞进 parser。

### 4.2 解析算法

1. Normalize request（规范化架构、版本约束、feature 名称）。
2. 将 profile + CLI overrides 转成 required/optional capabilities。
3. 对每个 capability 找 provider package candidates。
4. 按版本、channel、Host/Target、language、manifest metadata 过滤。
5. 选取满足所有必需能力的 package 集合。
6. 递归展开 `dependencies`；按真实 Manifest schema 解释 dependency JSON。
7. 检查依赖版本与 package conflict。
8. 检查 cycle。
9. 做稳定拓扑排序（同等节点按 package ID 字典序）。
10. 从 package 集合汇总 payload；相同 payload identity 只保留一次。
11. 将结果写入 `ResolvedToolchain`。

**禁止：** 将 `dependencies: HashMap<String, Value>` 的存在当成“解析器已实现”。先检查实际 JSON schema、真实 Manifest fixture、当前 find_*_packages 实现和微软数据样本。

### 4.3 Manifest Dependency 的安全规则

- 解析未知 dependency kind 时返回明确错误或保守保留，不能静默忽略 required dependency。
- 不能通过 package ID substring 模糊匹配依赖对象。
- 不允许从未验证的 package 元数据执行命令或脚本。
- 必须验证依赖节点版本、architecture 和 channel。
- 依赖 package 不可用时 plan 为 blocking conflict，不可自动找“差不多”的版本。

### 4.4 图测试

必须有 fixture：
- 无依赖节点。
- A requires B。
- A → B → C chain。
- A → B 和 A → C diamond。
- 两个 package 共用 payload。
- 依赖缺失。
- 版本冲突。
- Host arch 不匹配。
- Target arch 不匹配。
- A ↔ B cycle。
- optional dependency 未启用/启用。
- unknown dependency schema。
- 同一输入多次解析输出完全一致。

使用 property-based test（若现有依赖允许）随机生成 DAG 验证：所有 required dependencies 出现在 resolver 输出中，输出顺序满足边关系，无重复节点；cycle 一定被发现。

## 5. InstallPlan 输出

Plan 至少包含：

```json
{
  "schema": "msvc-kit.install-plan.v1",
  "plan_id": "sha256:...",
  "request": {
    "profile": "cpp-desktop",
    "provider": "msvc",
    "host_arch": "x64",
    "target_arches": ["x64"]
  },
  "resolved": {
    "msvc_version": "...",
    "sdk_version": "...",
    "channel": "...",
    "manifest_digest": "...",
    "packages": [],
    "payloads": []
  },
  "actions": [],
  "disk_usage": {
    "download_bytes": 0,
    "reused_cache_bytes": 0,
    "staging_bytes_estimate": null,
    "backup_bytes": null,
    "peak_extra_bytes_estimate": null,
    "final_install_bytes_estimate": null,
    "confidence": "partial"
  },
  "privilege": {"requires_elevation": false, "reason": "user-writable target"},
  "warnings": [],
  "blocking_conflicts": []
}
```

金额/size 使用整数 bytes，不存格式化文本。无法估算要用 null + reason，而不是 0 假装不占空间。

## 6. Plan 与 Apply 规则

- `plan` 可以获取必要 Manifest 和少量元数据，但不得下载实际大型 payload，除非用户明确指定 `--prefetch`。
- `plan` 可能更新独立的 Manifest HTTP cache（例如 ETag/Last-Modified 缓存）；它不得创建/修改安装 target root、下载 payload cache、staging、receipt 或 installed files。若提供 `--no-cache`，必须真正只读并在网络失败时明确失败。测试必须区分 Manifest cache 与 payload cache，不能用“完全零文件写入”的错误断言否定合法 Manifest cache 更新。
- Plan 必须 pin exact Manifest digest + payload URL + official SHA256 + size + CAB relations。
- Plan 保存当前缓存 blob verification status 和 install ownership state。
- Apply 读取 plan，不得默认解析 latest 重新做选择。
- Apply 前重新校验：Manifest snapshot/plan data integrity、目标 root identity、目标目录内容/ownership digest、缓存 blob hash、free space、已有 toolchain state。
- 如果状态发生变化导致动作不安全：失败 `StaleInstallPlan`，输出重做 plan 的命令。
- Apply 不得在 plan 中自动添加未记录的新 payload。
- 用户如要更新 latest，应显式重新 plan。
- JSON output 必须 schema versioning，并有 golden tests。

## 7. 磁盘估算算法

分开统计：

- `download_bytes`: 缓存中不可复用的 archive bytes。
- `cache_reuse_bytes`: 已有 hash-verified Blob bytes。
- `extract_bytes`: 已知归档解压总文件大小（有 manifest size 才准确）。
- `staging_bytes`: 本次新文件在 staging 的大小。
- `backup_bytes`: 需保留旧 owned files 的大小。
- `final_delta_bytes`: 最终目标树增加或减少量。
- `peak_extra_bytes`: download + staging + backup 的峰值估算，考虑不同阶段是否可释放。

不能简单用压缩包 size 估计 extract size。已知 ZIP entry uncompressed size 可用；MSI 可由 File table + CAB sequence metadata 计算；否则报告 unknown/range。Plan 要说明是 lower bound / upper bound / exact。

## 8. CLI/API 兼容

- 保持当前 `download` 行为和参数兼容。
- 可将 `download` 内部改造成 `plan -> apply`，但旧命令最终行为相同。
- 原有 Library `DownloadOptions.dry_run` 保留；可转接到新 Plan API。
- 新 API 建议：`create_install_plan(request, state) -> InstallPlan` 和 `apply_install_plan(plan, options) -> InstallReceipt`。
- 不要因添加新 field 破坏外部 public struct literal。
- `--format json` stdout 只打印 JSON；日志进 stderr。


---

<!-- SOURCE: 05_SAFE_EXTRACTORS_AND_TRANSACTIONS.md -->

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
- 只允许对来源可信且已按官方 Manifest SHA256 验证的 Microsoft payload 使用 Windows `msiexec` backend；不要把 `msiexec /a` 当作对任意不可信 MSI 的安全沙箱。
- MSI 的 AdminExecuteSequence / CustomAction 可能产生 `TARGETDIR` 以外的副作用。普通用户权限和用户可写 staging 可降低风险，但无法证明任意 MSI 的自定义动作只写 staging。不得 elevation；实机测试必须使用来源可信的测试 MSI，并记录权限上下文、日志和测试前后系统状态。
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

**目录 entry 的特别规则：** ZIP/VSIX 中 `folder/` 这种带一个结尾分隔符的目录 entry 是合法常见格式，不能因为 basename 为空而误拒。解析器必须先依据 archive entry 的 directory metadata/末尾分隔符识别目录 entry，再仅移除一个合法的末尾分隔符进行组件校验。对文件 entry，空 basename 一律拒绝。统一把 `/` 和 `\` 视为 archive separator；拒绝空的内部 path component、重复 separator、`.`、`..`、绝对路径和盘符/UNC 前缀。若真实 Microsoft fixture 证明某种非规范路径必须兼容，必须单独记录例外规则并增加正反测试，不能静默放宽通用 resolver。

**链接与 TOCTOU：** 默认拒绝 ZIP/CAB 中的 symlink、hardlink、reparse-point 类型 entry；只创建普通目录和普通文件。不能只做一次 `canonicalize()` 后就认为安全：创建每个文件前都要确保目标父目录仍在 staging root 内且不是 reparse point。Windows 实现应优先采用 handle-based/no-follow 检查；若现有 Rust API 做不到，使用 staging 内不可信内容隔离、逐级 revalidation，并把残余 TOCTOU 风险写入报告。路径比较应采用 Windows 默认不区分大小写语义，并检测大小写/尾随点空格等规范化碰撞。

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


---

<!-- SOURCE: 06_PRUNING_CACHE_LOCKFILE.md -->

# Pruning、Content-addressed Cache、Lockfile 与 Installed Manifest

## 1. Pruning Engine

### 1.1 目标

参考 portablemsvc 的文件裁剪流程与 `removed_files` Lockfile 记录，将 msvc-kit 当前跳过少量可选 SDK MSI 的机制扩展为**安装树级、可预览、可审计、可验证**的裁剪规则。

### 1.2 初始 profiles

| Profile | 默认行为 |
|---|---|
| `full` | 不裁剪任何已选 package 文件 |
| `portable` | 删除经验证的非必要 telemetry/非目标架构/特定冗余目录；保留开发/调试能力所需文件 |
| `minimal` | 更激进，但仅能启用有独立测试的规则 |
| `custom` | 用户通过配置文件启用/禁用规则 |

建议规则结构：

```toml
schema = "msvc-kit.pruning-rules.v1"
profile = "portable"

[[rules]]
id = "msvc.telemetry.vctip"
version = 1
action = "remove"
match = [
  "VC/Tools/MSVC/{vctools}/bin/Host{host}/{target}/vctip.exe",
  "VC/Tools/MSVC/{vctools}/bin/Host{host}/{target}/Microsoft.VisualStudio.Telemetry.dll"
]
reason = "Telemetry-only executable/library not required for tested compilation workflows"
requires_package = "..."

[[rules]]
id = "sdk.non_target_arch"
version = 1
action = "remove"
match = ["Windows Kits/10/Lib/{sdk}/um/{non_target_arch}/**"]
reason = "Architecture not selected by this install plan"
```

实际规则只能在 package/file manifest 证明路径含义后启用。示例 path pattern 不代表这些路径在所有 SDK 版本必然存在。

### 1.3 规则类型

- `RemoveFile`
- `RemoveDirectoryIfEmpty`
- `Keep`
- `MoveFile`（例如经过验证的 DIA/Debug CRT relocation；必须有特殊语义测试）
- `ArchitectureFilter`
- `PackageOptOut`

不要允许任意 shell command pruning。规则应声明式、可审计、可重放。

### 1.4 默认禁止项

- 不默认删除 `*.pdb`。
- 不默认删 DIA SDK 全部内容；若移动 `msdia140.dll`，需保证目标路径和多架构调试工具可用。
- 不默认删除所有 Debug CRT；portablemsvc 的逻辑会搬移部分 debug_nonredist DLL，再删除 Redist，不能只学删除动作。
- 不默认删 `Auxiliary` 整目录，除非兼容 `vcvars`/nvcc/UBT/其他工具所需路径有替代文件，并有测试。
- 不根据单一旧版本硬编码目录；规则要限定版本范围或 feature flag。

### 1.5 Preview 与执行

Pruning Plan 必须列出：
- relative path。
- 当前存在的类型和 size/hash。
- 规则 ID / 版本。
- reason。
- package owner。
- 预计释放空间。
- 是否影响已选 capability。

执行仅在 staging tree。任何规则失败必须终止 transaction，不允许忽略删除失败后仍标记成功。

每个 removed path 写入 Receipt：path、former SHA256（如有）、size、owner package、rule ID/reason、timestamp。需要支持从源 archive + Lockfile 重建，而不承诺从被裁剪后的目录恢复删除文件。

### 1.6 Pruning 测试

- `full` 无删除。
- `portable` 仅删白名单明确文件。
- 目标/非目标 architecture 规则。
- 同名但不同 package path 不误删。
- 目录非空不删。
- PDB 保留默认。
- 保留项 rule 优先级。
- rule overlap deterministic。
- 未识别版本不启用规则。
- 规则执行中失败不提交。
- `removed_files` 与真实删除集合完全一致。
- 最小 profile 的 cl/link/rc/mt/SDK headers/libs smoke tests。
- DIA/debug CRT relocation 的编译/调试依赖验证。

## 2. SHA256 Content-addressed Blob Cache

### 2.1 为什么改

msvc-kit 当前已有 redb 下载索引、URL/size/expected SHA/computed hash/path/status 等信息、verified cache reuse 和 atomic download publish；这很好，应保留。当前下载文件位于 version/architecture 目录中，不等同于全局 CAS。

portablemsvc 把 payload 缓存为 `{sha256}{extension}` 并用 hash-to-names map 追踪名称，可跨版本复用；新实现保留这种内容身份思想，但使用现有 redb 索引并解决路径/并发/原子发布问题。

### 2.2 目录

```text
<cache-root>/msvc-kit/
├── blobs/sha256/ab/<full-sha256>       # Blob key 不依赖文件名/扩展名
├── partial/<download-id>.part
├── manifests/by-digest/<sha256>.json
├── indexes/cache.redb
└── locks/
```

可以在 blob metadata 中记录原始 filename/extension/MIME；物理文件名不必有 extension。若现有 extractor 依赖后缀，可在 staging 中创建带预期扩展名的临时链接/副本，但不能把源文件复制到 CAS 之外后失去 hash identity。

### 2.3 Blob identity

唯一内容 key：`sha256:<lowercase-64-hex>`。

Source identity 独立记录：
- exact source URL。
- channel/manifest digest。
- package ID/component ID。
- payload filename。
- declared size。
- declared SHA256（若存在）。
- actual SHA256。
- downloaded timestamp/status。
- source trust level。

相同 hash，不同 URL/name/package：共用一个 Blob，多条 source reference。

### 2.4 下载流程

1. Validate URL scheme（HTTPS；若微软 manifest 允许特定 URL，按既有 trust policy）。
2. Validate payload relative filename（不能 path escape cache）。
3. 获取 source identity 和 expected hash。
4. 如果 Blob 存在：先读 Blob file metadata，再计算/复核 SHA256；必要时 verify source metadata。正确才复用。
5. 下载到 `.part`，1MiB 或合理固定 buffer 流式写入，不能把完整 payload 存 RAM。
6. Range resume 只有在 server `206`, Content-Range 起始 offset 和 ETag/Last-Modified validators 都匹配时继续；若 server 返回 200、错误 offset、416 或 validator 变化，重置并从零开始。
7. 完成后验证大小（若官方给定）和 SHA256。
8. `sync_all` 临时文件。
9. 在 blob lock 下原子 rename 到 CAS 路径；若 blob 已由其他进程发布，再校验已存在内容。
10. 更新 redb 索引事务。
11. 更新 source map/reference。

如果官方未提供 hash：
- 不得把只有大小的文件标为 `Verified`。
- 可按显式 opt-in 策略下载并计算 local hash，标为 `LocallyHashedUnpinnedSource`，但不可伪称官方来源验证。
- `strict` 默认拒绝对 source-unpinned lock 进行可复现部署，或按项目现有配置给出明显 warning/状态。

### 2.5 兼容迁移

- 保留旧下载目录读取能力。
- 迁移前检查旧文件 hash；hash 符合 payload expected hash 后才导入 CAS。
- 导入成功后更新 redb source index。
- 不立即删除旧缓存；先做只读迁移/重复使用，再通过 GC 明确清理。
- 对正在下载或被旧版本 CLI 使用的目录不移动。
- 迁移必须支持中断重试。

### 2.6 安全 GC

GC 采用 Mark-and-Sweep：

- Mark：所有 active transactions、未完成 plans、已注册/导入到 cache registry 的 Toolchain Lock、安装 receipts、旧安装仍引用的 payload，以及活跃 partial downloads。
- **外部任意路径的 Lockfile 无法被 GC 自动发现。** 必须提供显式 pin/unpin 或将 lockfile 导入 cache registry 的机制（例如 `cache pin-lock <path>` / `cache unpin-lock <id>`），并将 pin 关系持久保存在 redb。没有登记的外部 lockfile 不得被默认为 GC 已知引用；文档必须说明这一点。用户也可以选择不启用 cache GC 或保留相关 Blob。
- Sweep：只清除未被任何 active reference 引用、没有活动锁、超过最小保留期的 Blob。
- 执行前支持 `clean --cache --dry-run` 或 `cache gc --dry-run`。
- GC 执行时获取 GC/cache lock；避免和下载/安装竞争。
- 不跟随 symlinks/reparse points 离开 cache root。
- 对 redb 中记录不存在的文件只清索引；不要误删路径外文件。
- CAS 文件 hash 不符时移到 quarantine/删除并重新获取，不按有效 Blob 使用。
- 崩溃在删除文件和删索引之间必须可恢复。

## 3. Lockfile 与 Receipt

### 3.1 三种文件各司其职

| 文件 | 用途 | 方向 |
|---|---|---|
| `toolchain.lock.json` | 指定“要安装什么、来源是什么” | 输入/重建依据 |
| `installation-receipt.json` | 证明本次实际获取和安装了什么 | 安装结果 |
| `installed-manifest.json` | 证明当前磁盘最终文件树是什么 | 完整性检测/ownership |

### 3.2 Lockfile v2 必备字段

- schema：`msvc-kit.toolchain-lock.v2`。
- provider ID。
- channel + exact channel manifest URL/digest。
- VS manifest URL/digest。
- requested selector + resolved exact versions。
- host/target arches。
- Build Profile ID/version/digest。
- capability/features。
- exact selected package IDs/versions/dependencies。
- exact payloads：file name、URL、expected size、SHA256、package ID、type、CAB parent/media relation。
- extractor backend preference（重建时允许策略不同，但最终 receipt 要记录实际 backend）。
- pruning profile/rules version。
- environment profile reference。
- lock provenance status：`source-pinned` / `selection-only` / `local-unpinned`。

避免绝对本机路径成为跨机器 lock identity。可在 `local_install_hint` 提供非权威建议，但不参与重建。

### 3.3 Receipt

记录执行结果，而不是请求结果：
- actual source downloads/reused blobs。
- expected and actual hashes。
- extractor backend/version。
- extracted package file list。
- collision resolution result。
- pruned paths and reasons。
- final installed file manifest digest。
- transaction ID and journal result。
- validation/doctor result。
- privilege/elevation status（只记录观察事实）。
- license acceptance state/source/time（如项目现有要求）。

### 3.4 v1 migration

- Toolchain Lock v1 必须仍可读取。
- 没有 receipts 的 lock 是 `selection-only`，不得声称 source-pinned。
- v1 receipts 可读但需标记旧 schema 的证明范围。
- 新 v2 生成后不要悄悄改写原 v1；显式转换/复制。
- 旧 lock v1 的 `fingerprint` 只能证明 selection identity，不能当 installed hash。
- JSON fixtures 覆盖 valid v1、valid v2、unknown schema、wrong hash、missing field、unknown field、wrong package order、duplicate payload、CAB parent missing。

### 3.5 Installed Manifest 完整性

- 每个文件记录 path、size、SHA256、kind、owners、source package。
- Directory existence 可以记录，但最终完整性以文件为主。
- `doctor --lockfile` 或 `verify` 可以做快速 metadata check 和 full hash check 两种模式。
- 文件被修改/删除/新增，必须具体报告相对路径。
- 不要自动把外部新文件加入 msvc-kit ownership。

## 4. 验收门槛

- 同 hash 不同 URL/name/package 只存一个 Blob。
- hash 错误缓存绝不命中。
- partial download resume 协议错误时不会拼接损坏文件。
- 并发同一 Blob 只产生一个有效发布结果。
- GC 不删除 active plan/install/lock 引用 Blob。
- Lock v1 compatibility 通过。
- Lock v2 重建得到同 package/payload set。
- Receipt + InstalledManifest 能发现被修改的 installed file。
- pruning 所有删除有规则 ID 和 reason。


---

<!-- SOURCE: 07_DETECTION_AND_PROVIDER_API.md -->

# 环境探测、复用和 Provider API

## 1. 区分 Detect、Validate、Reuse

这三个词不可混用：

- **Detect**：找到可能的安装路径和可执行文件。
- **Validate**：检查版本、架构、headers/libs/linker/runtime 和实际 build capability。
- **Reuse**：确认某个既有安装满足当前 plan，且允许只读使用。

仅 PATH 中有 `clang.exe` 不证明完整 LLVM toolchain 可用；仅存在 `cl.exe` 也不证明正确 Host/Target libs 和 SDK 齐全。

## 2. Detector 范围

### 第一阶段：MSVC / Visual Studio

- `vswhere`：VS 2017+ 安装实例；先定位 VS installation，再验证 MSVC component 实际路径。
- Registry：只读查询；不能假设所有版本都注册同样键。
- 常见安装路径：只作 fallback。
- 用户指定路径：优先级高，但仍进行完整结构验证。
- `find_vs_instances()` 当前用于 VS instance / UBT 安装目标，不等于通用 VS/MSVC/SDK 探测完整实现。

### 第二阶段：Windows SDK

- Registry/known installation paths。
- 用户指定 `--sdk-dir`。
- 检查 Include/Lib/bin 版本布局。
- 验证 UCRT、UM/WinRT headers/libs（按 profile）。
- 支持独立 SDK root，但不能混用另一 root 的 receipt/source provenance。

### 第三阶段：LLVM / MinGW

LLVM：
- `clang --version` / `clang-cl --version`。
- `clang -print-resource-dir`。
- `llvm-ar` / `llvm-lib` / `lld-link` / `llvm-rc` / `llvm-nm`（按 capability）。
- include/resource dirs、target triple、sysroot、CRT/Windows SDK 关系。

MinGW：
- `gcc -dumpmachine`、`gcc --version`。
- `g++`、`ld`、`ar`、`windres`。
- binutils 和 target triple。
- UCRT/MSVCRT runtime family、headers/libs。
- 不把 MSYS2 shell runtime 与 MinGW compiler 混淆。

### Build Tools

- CMake：`cmake --version`，解析 CMake executable path。
- Ninja：`ninja --version`。
- NASM：`nasm -v`。
- 不将它们误分类为完整编译器 toolchain；单独提供 BuildToolProvider。

## 3. 探测结果模型

```rust
struct DetectedToolchain {
    id: String,
    provider: String,
    display_name: String,
    root: PathBuf,
    version: VersionObservation,
    host_arches: BTreeSet<Architecture>,
    target_arches: BTreeSet<Architecture>,
    capabilities: BTreeSet<CapabilityId>,
    binaries: BTreeMap<String, PathBuf>,
    include_roots: Vec<PathBuf>,
    library_roots: Vec<PathBuf>,
    runtime_family: Option<String>,
    origin: DetectionOrigin,
    validation: ValidationSummary,
    writable: bool,
    managed_by_msvc_kit: bool,
}
```

版本 observation 可以是 `Exact`、`FamilyOnly`、`Unknown`；不能把 `14.0` 这种旧版本 family-only 伪装成精确 patch version。外部安装状态默认 `managed_by_msvc_kit=false`。

## 4. 可复用能力判定

给当前 Plan 的需求集合 `required_capabilities`，候选 installation 必须同时满足：

1. provider 兼容。
2. 版本约束。
3. Host 和所有 Target architecture。
4. required compiler/linker tools。
5. required headers/libs/runtime。
6. ABI/CRT 要求（存在时）。
7. profile 特有能力（MFC/Spectre/ASAN/UWP 等）。
8. 未被损坏/删除。
9. 许可证/来源策略允许使用。

结果：
- `ReusableExact`：完整满足。
- `ReusableWithAdditions`：可在独立 msvc-kit root 补缺，不修改外部 root；实际能否组合由兼容性检测决定。
- `Incompatible`：不能复用。
- `Unknown`：证据不足，不自动复用。

**默认只读**：探测过程不修改环境变量、注册表、文件 ACL、原有目录、PATH 或安装状态。

## 5. CLI 目标

```powershell
msvc-kit detect
msvc-kit detect --compiler msvc --format json
msvc-kit detect --sdk --format json
msvc-kit detect --compiler llvm --format json
msvc-kit doctor --dir <root> --format json
msvc-kit plan --profile cpp-desktop --reuse-detected
```

不做模糊 `--best` 自动挑选而不给依据。若有多个满足项，输出按确定规则排序的 candidates 和 reasons；自动复用可选，但 plan 必须明确写出使用哪个 instance ID/path 以及原因。

## 6. Provider API 设计

当前 `ComponentDownloader` / `ComponentType::{Msvc,Sdk}` 是下载器抽象，不是完整通用 provider 生命周期。不要仅把 enum 加上 LLVM/MinGW 就宣称完成。

建议分层：

### Compiler/SDK Provider

```rust
trait ToolchainProvider {
    fn provider_id(&self) -> ProviderId;
    async fn detect(&self, context: &DetectionContext) -> Result<Vec<DetectedToolchain>>;
    async fn resolve(&self, request: &ToolchainRequestV2) -> Result<ResolvedToolchain>;
    async fn acquire(&self, plan: &InstallPlan, cache: &dyn BlobStore) -> Result<AcquisitionReceipt>;
    async fn validate(&self, root: &Path, request: &ToolchainRequestV2) -> Result<ValidationSummary>;
    fn environment(&self, detected: &DetectedToolchain) -> Result<EnvironmentSpec>;
}
```

### Build Tool Provider

```rust
trait BuildToolProvider {
    fn tool_id(&self) -> BuildToolId;
    async fn detect(&self, context: &DetectionContext) -> Result<Vec<DetectedBuildTool>>;
    async fn resolve(&self, request: &BuildToolRequest) -> Result<ResolvedBuildTool>;
    async fn acquire(&self, plan: &InstallPlan, cache: &dyn BlobStore) -> Result<AcquisitionReceipt>;
}
```

### 共用核心负责什么

- Plan serialization/validation。
- Blob cache。
- Transaction/journal。
- Ownership/InstalledManifest。
- Receipt/Lockfile。
- Progress/cancellation。
- diagnostics/JSON/CLI.

Provider 自己负责：
- source catalog/manifest format。
- version resolution。
- platform package selection。
- provider-specific extractor/install action。
- capability/architecture semantics。

不要要求所有 Provider 都一定通过 MSI/VSIX。

## 7. Provider 扩展顺序

1. MSVC Provider：用现有 MsvcDownloader/manifest code 适配新接口。
2. Windows SDK Provider：处理 MSI/CAB source media，独立 SDK root。
3. LLVM Provider：官方 archive，验证 release checksum/signature；先只 detect + validate，再 download provider。
4. MinGW Provider：明确发行版源（例如用户配置的 MSYS2/其他 distro），不私自猜一个默认第三方源。
5. BuildTool Providers：CMake/Ninja/NASM，支持多个安装源但不强行塞进 compiler Toolchain enum。

## 8. Detector 测试

- Unit tests 只使用固定目录 fixtures/mock process output，不扫描开发机 PATH。
- PATH 缺失/多个同名可执行文件。
- exe 版本 output 格式差异。
- 只有 cl.exe 无 lib/linker/header。
- wrong target lib。
- SDK root 与 MSVC root 分开。
- VS2015 family-only version。
- x64 Host→arm64 target。
- LLVM 缺 resource dir。
- MinGW target triple 与实际 lib mismatch。
- CMake/Ninja/NASM 只是 build tools。
- 外部 root 不可写仍可 read-only detect/reuse。
- invalid/modified install receipt 导致不复用。
- 多 candidate 选择稳定。
- detector 不调用修改系统的命令。


---

<!-- SOURCE: 08_TEST_STRATEGY_AND_CASES.md -->

# 测试总纲与详细测试用例

**本项目把测试当作核心功能，不是开发完之后的检查项。** 每个新行为必须先有能失败的测试；测试应由真实行为和安全不变量驱动。

## 1. 测试分层

| 层级 | 名称 | 默认 CI | 外网/管理员 | 目标 |
|---|---|---:|---:|---|
| T0 | Unit + pure logic | 是 | 否 | 快速覆盖数据逻辑 |
| T1 | Filesystem fixture integration | 是 | 否 | 临时目录、真实 archive fixture |
| T2 | Windows OS integration | Windows CI | 否 | 路径、rename、reparse、msiexec wrapper |
| T3 | Fault injection | 是 | 否 | 事务 crash/recovery |
| T4 | Official manifest smoke | 可按需 | 有网络、低流量 | 真实 Manifest 解析 |
| T5 | Full real MSI/SDK install | 不默认 | 可能数 GB；仅手动/专用 runner | 端到端最终验收 |

不要让 T5 变成每次 PR 的 CI；也不要把 T0/T1 全部跳过后只跑一次真实 download。

## 2. 基线测试流程

Agent 必须先检查 `justfile`、`vx.toml`、`.github/workflows`，再确定正确命令。参考 README 可能有 `vx setup`、`vx just ci`，但以当前 fork 实际脚本为准。

最低验证（按仓库真实支持执行）：

```powershell
cargo fmt --check
cargo check --all-targets
cargo test --all-targets
cargo clippy --all-targets --all-features -- -D warnings
```

如果 CI 使用 `vx just ci` 或 `just ci`，还要执行对应 canonical CI command。不要同时执行多个会互相污染 target root 的测试。

Linux/macOS 宿主无法验证 Windows-specific MSI/registry/reparse behavior；必须在 Windows CI 运行对应测试，或明确标记 `NOT_RUN_PLATFORM`。

## 3. 安全路径测试矩阵

每种 archive extractor（VSIX/ZIP/CAB/MSI native）都必须走同一安全路径测试集。

### 应拒绝

- `../escape.txt`
- `a/../../escape.txt`
- `..\\escape.txt`
- `/absolute/file`
- `C:\\absolute\\file`
- `C:/absolute/file`
- `\\server\\share\\file`
- `\\?\\C:\\outside`
- `a\\..\\..\\outside`
- `file.txt:payload`（ADS）
- `CON`, `NUL.txt`, `AUX`, `COM1.log`, `LPT9`（Windows reserved names）
- `foo.`、`foo ` 与规范化碰撞。
- 空字符串、`.`、重复 separator。
- Unicode normalization/case-fold 后重复的目标路径。
- entry `a` 是文件，同时有 `a/b.txt`。
- symlink/junction 在 staging 中指向 staging 外路径。
- 不可信压缩归档巨大展开 size / 极端 compression ratio。

### 应正确处理

- 中文文件名。
- 路径含空格。
- path length 接近 Windows MAX_PATH 和长路径启用场景。
- 多级目录。
- 目录 entry 与文件 entry 顺序变化。
- duplicate same path + same bytes（拒绝或确定性去重，须明确定义）。
- duplicate same path + different bytes（必须冲突）。
- 空目录（无文件）处理策略明确。

每个拒绝测试断言：staging 外没有任何文件/目录发生创建、覆盖或删除。

## 4. Target root 安全测试

构造临时目录 fixture，检查：

- filesystem root 被拒绝。
- Windows drive root/UNC share root 被拒绝。
- `%WINDIR%`/Program Files 等保护路径拒绝普通 install。
- home/cwd/temp/cache/config 与 target 相同被拒绝。
- target 在 cache/config 的父/子关系有明确规则。
- 不存在 target 可以创建。
- 空目录可以安装。
- 未知非空目录拒绝。
- 有伪造 `.msvc-kit/receipt.json` 但 hash/ID 不匹配，拒绝。
- 合法 managed root 能精确更新。
- managed root 中存在 receipt 未拥有的用户文件，更新不得删除它。
- target 内 symlink/junction 连接到 root 外部，拒绝修改路径。
- 不同版本共存：安装 A 后安装 B，A 文件保持不变。
- 两个安装共用同一文件且 SHA 相同：owner reference 正确增加。
- 两个安装同路径但 SHA 不同：Plan conflict。

所有拒绝情况保存原文件 hash，操作后比对完全一致。

## 5. 下载/CAS 测试

### 单文件

- 官方 expected SHA 正确：Blob 发布。
- expected SHA 错误：拒绝发布；partial quarantined/removed。
- 服务器中断：partial 保留且有安全 metadata，重试可继续。
- HTTP server 不支持 Range 返回 200：从零重新开始，不拼接。
- 206 Content-Range 起始 offset 错：拒绝 resume。
- ETag 变化：丢弃旧 partial。
- HTTP 416：按服务器响应决定重启，不误判成功。
- declared size 不同：记录异常并按 policy fail。
- expected SHA missing：状态不是 official-verified。
- cache hit 文件被篡改：重新下载或 fail。
- 文件存在但 index 缺记录：严格模式不得仅按 size 复用。

### 并发

- 20 个任务下载同 SHA：只发布一个有效 Blob。
- 同 SHA、不同 URL：同 Blob，多 source references。
- 不同 SHA、相同 filename：不同 Blob。
- 下载中另一个进程查询 Blob：不读到 half-written。
- 下载和 GC 并发：活跃 Blob 不被删。
- 崩溃在 Blob publish 与 index commit 之间：下次 reconciliation 可恢复。
- redb corruption：备份/重建策略不丢失 blobs；报告可重建 index。

### GC

- 活跃 transaction 引用的 Blob 不删。
- Lockfile 引用的 Blob 不删（依 retention policy）。
- 已安装 Receipt 引用 source Blob 的保留策略明确。
- 无引用 Blob 仅在 grace period 之后删。
- dry-run 不改变任何文件。
- symlink 不可让 GC 跳出 cache root。
- 删除前 hash/metadata mismatch 进入 quarantine，不顺手删除其他路径。

## 6. Manifest/Dependency Graph 测试

详见 `04_PLAN_AND_DEPENDENCY_RESOLVER.md`。至少覆盖：
- manifest golden fixture 多版本。
- dependency closure、diamond、duplicate payload。
- cycle、missing dep、version conflict。
- host/target filters。
- unknown dependency fields。
- deterministic resolution。
- package order stable。
- no network test required。

## 7. Plan/Dry Run 测试

- `plan` 不下载实际 packages。
- `plan` 不创建/更改正式 target root。
- 同一状态多次 Plan 业务字段相同，plan_id 相同。
- created_at 改变不改变 plan_id。
- cache hit / miss 展示正确。
- unknown hash 不标记 verified。
- reuse existing toolchain 需有验证证据。
- package dependency action 排序正确。
- pruning list/estimated reclaimed bytes正确。
- disk usage unknown 不显示成 0。
- conflict blocking 使 `apply` 拒绝。
- plan file 被篡改时 hash/signature/plan digest 校验失败。
- stale target root 检测。
- Manifest 发布变化不会让 apply 静默切换 latest。
- JSON stdout 可被 serde_json 解析，日志不混入 stdout。
- existing DownloadPreview API/CLI 行为兼容。

## 8. MSI 测试

### Backend wrapper tests（所有平台）

用 fake process runner 注入返回码：
- 0 成功。
- 1618 retry 次数准确、延迟可 mock。
- 1603 提示 generic fatal 并保留 log path。
- 1619 提示 MSI/CAB source media 关联检查。
- 1619 不错误地认定 MSI 一定损坏。
- 1620 invalid package。
- spawn failure / msiexec missing。
- timeout/cancellation。
- 输出目录不可写。
- command args 不通过 shell 拼接。
- 不存在 `/i` 参数。

### Windows 实机集成（opt-in）

- 标准用户账户对 `%LOCALAPPDATA%` staging 运行 `msiexec /a`。
- 管理员账户但未提升运行相同任务，确认工具本身未主动触发 `runas`。
- 使用已审核的合法测试 MSI；检查 Windows Installer 日志、目标文件和系统注册状态不发生 MSI 产品安装式注册。
- 若 MSI 自身的 AdminExecuteSequence/custom action 请求权限，明确记录，不声称所有 MSI 无 UAC。
- 不在 CI 中弹出不可交互 UAC prompt；测试要有 timeout 和用户交互保护。
- 不执行 `/i`。

### Native parser

必须满足 `05_SAFE_EXTRACTORS_AND_TRANSACTIONS.md` 的 parity gate。没有真实可合法测试的 MSI/CAB fixture，则 native parser 不可宣布稳定或默认启用。

## 9. Pruning 测试

完整参照 `06_PRUNING_CACHE_LOCKFILE.md`。所有规则必须有：
- positive fixture。
- negative fixture（相似但必须保留的路径）。
- version applicability。
- before/after manifest assertions。
- smoke compile/link 证明。

## 10. Transaction fault injection

为文件系统操作抽象 `FsOps` 或等效测试 seam。必须能在指定 operation sequence 抛错/模拟 crash。

注入位置至少：
1. 创建 transaction dir 后。
2. plan journal 写入后。
3. payload 下载写到一半。
4. 下载完成但未更新 index。
5. 每个 archive extraction 写到一半。
6. merged tree 构建中。
7. pruning 每条规则之前/之后。
8. validation 之后。
9. 旧文件 backup 前。
10. 旧文件 backup 后、新文件移动前。
11. 新文件移动后、journal 完成事件之前。
12. OwnershipIndex 写入中。
13. Receipt 写入前后。
14. Committed marker 写入前后。
15. backup cleanup 期间。

对每个 fault point 断言：
- 下一次 `recover` 可重复执行。
- 旧文件要么完整恢复，要么保留安全冲突状态。
- 不丢失未由本 transaction 管理的文件。
- 没有半文件被误认为完整文件。
- journal 状态和实际文件 hash 可以解释。
- transaction 不会报 committed 除非 receipt/manifest 和全部目标文件一致。

**Crash durability 子集（必须真正终止进程）：** 至少覆盖 `BackupMoved` 后、`NewFileInstalled` 后、OwnershipIndex/Receipt 原子替换前后、`Committed` 写入前后。使用 helper child process 写 journal 并在测试 hook 到达指定边界时等待父进程杀死；父进程重新启动 recovery。单纯在同一进程返回错误或 panic 不算 crash durability 证据。Windows CI 必须运行这些测试；不可运行时标记 BLOCKED。

## 11. 环境探测与 Provider 测试

- 单元测试使用 fake registry/vswhere/path fixtures。
- Detector 不修改外部 root。
- Candidate 结果有稳定排序。
- 缺失某个重要工具时 capability 不应假阳性。
- 同名多个版本不混用 PATH 顺序。
- build tool provider 不被识别为 compiler provider。
- 未经配置不从第三方随机下载 LLVM/MinGW。

## 12. End-to-end 验收

最小 MSVC + SDK：

1. 使用固定的官方 manifest snapshot 和 fixture package / 或专用 Windows runner 的官方版本。
2. 生成 plan。
3. 输出 JSON 可解析。
4. 下载包并验证 SHA256。
5. 提取到 staging。
6. 生成 InstallReceipt 和 InstalledManifest。
7. `doctor` 通过。
8. 使用 `run` 调用 `cl.exe` / `link.exe` 编译链接小型 C/C++ 程序。
9. 运行结果（native host/target 时）。
10. 第二次 plan 显示可复用，不重复下载/提取。
11. 修改一个 installed file，verify/doctor 报出 hash mismatch。
12. 再执行 clean/upgrade，确认只删除自己拥有的文件。

## 13. 测试报告格式

每条测试记录：
- Test ID。
- 测试层级。
- 命令。
- 环境（OS、版本、权限等级）。
- 结果：PASS/FAIL/BLOCKED/NOT_RUN。
- 输出日志路径。
- 失败堆栈/错误码。
- 是否为已有基线失败。
- 是否影响阶段验收。


---

<!-- SOURCE: 09_IMPLEMENTATION_ROADMAP.md -->

# 实施路线图：按门槛逐阶段开发

不得把全部功能混入一个 giant PR。每阶段独立、可 review、可回退。Agent 可以自主实施，但不能跳过 Exit Gate。

## P00 — Baseline Audit（不改核心代码）

### 任务
- [ ] 确认 git HEAD、branch、dirty state、remotes。
- [ ] 阅读 README/README_zh/Cargo/CI/justfile/vx。
- [ ] 画当前模块和 download→extract→install 调用流程。
- [ ] 核实当前 cache/index/lockfile/receipt/marker。
- [ ] 检查所有 archive write path。
- [ ] 检查 current tests 和 test command。
- [ ] 建立 `agent-work/BASELINE.md`、`RISKS.md`、`TEST-RESULTS.md`。
- [ ] 运行基线测试。

### Exit Gate
- [ ] 每个用户目标有真实代码/测试证据和状态。
- [ ] 识别当前共享 root 目录布局。
- [ ] 记录基线测试的每个失败。
- [ ] 没有修改用户文件或仓库 remote。

## P01 — Data contracts + Path Security（P0 安全）

### 任务
- [ ] 设计统一 `SafePathResolver`。
- [ ] 给 VSIX/ZIP/CAB/MSI 输出路径接入该安全层。
- [ ] 加入 Windows path semantics tests。
- [ ] 检查 Zip Slip 以及 archive entry collisions。
- [ ] 设计 `InstallPlan`、`InstallReceipt`、`OwnershipIndex`、`TransactionJournal` 的最小 schema（可先内部）。
- [ ] 现有 marker 不再作为唯一“提取完成”凭据；旧 marker 要安全地 invalid/legacy handle。
- [ ] 完成 public API compatibility review。

### Exit Gate
- [ ] path traversal 测试全过。
- [ ] 安全拒绝时 staging 外零变更。
- [ ] 现有正常 fixture extractor 测试全过。
- [ ] schema golden tests。

## P02 — Canonical InstallPlan / CLI Dry Run

### 任务
- [ ] 读取当前 DownloadPreview 并重用逻辑，不重复请求 Manifest。
- [ ] 添加统一 InstallPlan。
- [ ] 把 package list、payload list、cache state、target state、pruning actions、disk estimate、privilege reason 统一为 PlanAction。
- [ ] 添加 `plan` CLI + JSON output。
- [ ] Plan 不写 target root、不下载完整包。
- [ ] stable plan_id。
- [ ] apply plan file 时验证 schema/hash。
- [ ] stale plan detection。

### Exit Gate
- [ ] Dry Run 不修改任何目标目录/缓存 payload。
- [ ] Plan JSON 可 round-trip。
- [ ] 同一状态 plan_id 稳定。
- [ ] CLI 旧 download preview/API 兼容。
- [ ] 所有 blocking conflict 都阻止 Apply。

## P03 — File-level Staging / Transaction / Recovery

### 任务
- [ ] 创建 per-transaction workspace。
- [ ] 每 package 独立 staging。
- [ ] 形成 package output manifest。
- [ ] merge tree 时检测 collisions。
- [ ] 建立 root lock protocol。
- [ ] 建立 ownership index。
- [ ] 建立 write-ahead journal。
- [ ] 文件级 backup/commit/rollback。
- [ ] recover command 或 apply 前自动 recovery。
- [ ] 旧 marker 与新 Receipt/InstalledManifest 的迁移策略。
- [ ] 在 transaction 中不替换整个 shared root。

### Exit Gate
- [ ] 未知非空 root 永不被清除。
- [ ] 同 root 多个版本共存。
- [ ] 每个 fault point crash injection 后恢复成功或安全进入 NeedsRecovery。
- [ ] 失败 transaction 不更改非 ownership 文件。
- [ ] 成功后旧/新 owner state 正确。

## P04 — MSI Backend abstraction / Native MSI Spike

### 任务
- [ ] 保留当前 msiexec backend。
- [ ] 抽象 backend preference `auto/msiexec/native/fallback`（如果 native 未交付，native 显示 unsupported/experimental）。
- [ ] Fake process runner 测试参数、代码、retry、timeout。
- [ ] 增加 MSI verbose log。
- [ ] UAC/标准用户安全测试方案。
- [ ] 调研候选 Rust MSI DB crate + CAB crate。
- [ ] 用真实合法 fixture 做 native feasibility report。
- [ ] 若 parity 达标，实现 native；若未达标，保留 spike 报告，不进入默认流程。

### Exit Gate
- [ ] 不存在 `/i` 调用。
- [ ] 1618/1603/1619/1620 测试完整。
- [ ] msiexec backend 在测试目标路径可运行。
- [ ] native backend 只有在真实 parity gate 达标后标记 stable。

## P05 — Pruning Engine

### 任务
- [ ] 定义 rule schema 和 profiles。
- [ ] 将 portablemsvc 有证据的规则逐条录入候选清单。
- [ ] 规则默认先 Dry Run。
- [ ] 为每条规则添加 positive/negative/version fixture。
- [ ] 实现 staging-only pruning。
- [ ] 记录 removed paths/old hash/size/owner/rule reason。
- [ ] 保留 PDB/DIA/Debug CRT 默认策略。
- [ ] 增加 profile minimal compile/link acceptance。

### Exit Gate
- [ ] full 无删除。
- [ ] portable 每条删除可解释。
- [ ] 未知版本不裁剪。
- [ ] 失败不提交。
- [ ] 最小工具链编译测试通过。

## P06 — CAS Blob Cache

### 任务
- [ ] 保留现有 redb index 功能。
- [ ] 设计 Blob Store keyed by SHA256。
- [ ] 实现 streaming `.part` 下载。
- [ ] 实现 validator-aware HTTP Range resume（若决定保留此功能）。
- [ ] Blob atomically publish + verify。
- [ ] 同 hash 多 source refs。
- [ ] 旧目录向 CAS 可重入导入。
- [ ] Cache GC mark/sweep + dry-run。
- [ ] active install/lock/plan pinning。

### Exit Gate
- [ ] 同 hash 只一个 Blob。
- [ ] corrupted cache 不复用。
- [ ] 并发安全。
- [ ] GC 不删活跃内容。
- [ ] 老 cache migration 可反复运行、不丢数据。

## P07 — Lockfile v2 / Receipt / Installed Manifest

### 任务
- [ ] 定义锁文件 v2。
- [ ] 保留 v1 reader 和 compatibility fixtures。
- [ ] 记录 Manifest/payload source pinning。
- [ ] 记录 CAB/MSI relationships。
- [ ] 记录 pruning rules/removed files。
- [ ] 定义 final installed file manifest。
- [ ] 安装提交最后更新 Receipt。
- [ ] `verify`/`doctor` 能检测 modified/missing/extra files。
- [ ] 外部安装输出 selection-only lock 明确标记。

### Exit Gate
- [ ] v1 lock 可读。
- [ ] v2 rebuild 下载集合精确。
- [ ] source-pinned 与 selection-only 不混淆。
- [ ] installed mutation 被检测。
- [ ] Receipt 与 transaction 状态一致。

## P08 — Build Profile / Dependency Resolver

### 任务
- [ ] 实际审查 Manifest dependency JSON schema。
- [ ] 定义 capability mapping。
- [ ] 定义初始 `c-native`、`cpp-native`、`cpp-desktop`、`cpp-mfc`、`cpp-spectre`、`cross-arm64` profiles。
- [ ] 依赖 graph closure/cycle/conflict/version/architecture。
- [ ] stable topological sort。
- [ ] Plan 由 resolved graph 产生。
- [ ] 保留旧 `--include-component`、`--exclude-pattern` 作为高级覆盖。

### Exit Gate
- [ ] fixture tests 覆盖 DAG/cycle/conflict。
- [ ] 真实 manifest fixture 解析结果符合 package selection 预期。
- [ ] 未知依赖 schema 不被静默忽略。
- [ ] 同输入结果确定。

## P09 — Detection / Reuse

### 任务
- [ ] 当前 VS detector 抽成通用只读探测接口（保持旧行为）。
- [ ] MSVC detector。
- [ ] Windows SDK detector。
- [ ] 能力验证。
- [ ] LLVM/Clang detector。
- [ ] MinGW/GCC detector。
- [ ] CMake/Ninja/NASM build tool detectors。
- [ ] capability-based reuse planner。
- [ ] 外部安装 read-only。

### Exit Gate
- [ ] fixtures 不依赖本机 PATH。
- [ ] 检测不修改外部状态。
- [ ] 仅有 compiler exe 不能假阳性。
- [ ] exact reuse / partial / incompatible 状态测试。

## P10 — Provider API

### 任务
- [ ] 定义 ToolchainProvider 和 BuildToolProvider 分层。
- [ ] 将现有 MSVC downloader/provider 适配。
- [ ] 将 Windows SDK adapter 适配。
- [ ] 共用 Plan/Cache/Transaction/Receipt/Diagnostics。
- [ ] 添加 mock provider integration tests。
- [ ] 确定 external provider package sources policy。
- [ ] 再评估 LLVM/MinGW/CMake/Ninja/NASM provider。

### Exit Gate
- [ ] 新增 mock provider 不需修改 transaction core。
- [ ] MSVC/SDK 当前 CLI/API 不回归。
- [ ] provider-specific behaviors 不被过度抽象丢失。

## P11 — Full regression / Documentation / Release readiness

- [ ] 跑完整 offline suite。
- [ ] 跑 Windows integration suite。
- [ ] 跑故障注入 suite。
- [ ] 按 opt-in 跑真实 MSI/SDK acceptance（无用户批准则不做）。
- [ ] 更新 README/README_zh/docs/API/CLI help。
- [ ] 更新 architecture diagrams。
- [ ] 更新 migration notes。
- [ ] 更新 lock schema examples。
- [ ] 更新 tests matrix。
- [ ] 检查 license attribution。
- [ ] 检查 Cargo dependency/license/MSRV。
- [ ] 检查 `git diff`、secrets、binary/large test assets。

### 最终 Gate

- 没有未知的 data-loss risk。
- 所有未测试平台/后端明确标记。
- 真实下载测试不被误说为 CI 覆盖。
- 兼容性变更有迁移说明。
- `FINAL_REPORT.md` 完整。


---

<!-- SOURCE: 10_ATTRIBUTION_SECURITY_AND_RELEASE.md -->

# 归因、许可证、供应链与发布检查

## 1. 对 portablemsvc 的明确归因

项目文档必须明确说明本 fork 的安装器改进参考了 `tgbender/portablemsvc`。可在 README 的 Acknowledgements/Design references 或 architecture doc 中写：

> Installer design in this fork is inspired by [tgbender/portablemsvc](https://github.com/tgbender/portablemsvc), particularly its MSI extraction backend abstraction, content-hash-based payload caching, reproducible installation lockfile, post-extraction pruning, and output-directory safety checks. This fork reimplements and adapts these concepts in Rust and integrates them with msvc-kit’s existing Visual Studio Manifest, component selection, diagnostics, and toolchain APIs.

并补一句：

> The implementation is an independent Rust implementation unless a source file explicitly notes copied or adapted code.

不要写“portablemsvc 有 rollback-safe transaction”，因为其当前 output replacement 会先删除旧目录再 rename staging。可以写“staging extraction and output validation”，并说明本项目在此基础上加强 rollback。

## 2. 代码复用与许可证

当前两个项目都是 MIT 项目（提交代码前必须再次检查对应 commit 的 LICENSE 文件）。

- 借鉴思想：注明设计来源是良好做法。
- 直接复制/改写代码：保留原 copyright notice、MIT license text 和必要的源文件 attribution。
- 从第三方 crate 引入代码/依赖：审核 crate license 和 transitive licenses。
- 不复制微软 SDK/MSVC payload 内容到仓库测试 fixtures，除非许可证明确允许；优先在用户 opt-in 的本地外部 fixture 运行集成测试。
- 不提交真实 SDK MSI、CAB、VSIX 或商业/专有 payload。

## 3. Microsoft payload 和许可

msvc-kit 自身 MIT 许可证不等于 Microsoft Visual Studio Build Tools/Windows SDK payload 可按 MIT 分发。

- 保留微软官方 source URLs 和原始 hash。
- 保留项目当前 license acceptance 逻辑（若现有 workflow 要求用户明确接受则不能绕过）。
- 不把官方 payload 镜像进自身 GitHub release。
- 不把 Microsoft binaries 作为 Rust crate 的默认测试资源提交。
- bundle/redistribution 流程必须遵守 Microsoft License Terms。
- 记录 license acceptance 状态但不要伪造用户同意。

参考：
- https://visualstudio.microsoft.com/license-terms/
- https://learn.microsoft.com/en-us/visualstudio/install/workload-component-id-vs-build-tools

## 4. UAC/安全边界

- 默认普通用户权限。
- 普通安装只写用户可写的 msvc-kit root 和其 staging/cache。
- 不调用 `/i`。
- `msiexec /a` 是 administrative extraction，不保证具体所有 MSI 一定不触发 elevation。
- 不自动 `runas`。
- `install-into-vs` / persistent system env 明确标注管理员需要。
- MSI 自定义 action 如要求 elevation，则失败并报告，不尝试规避 Windows security boundary。

## 5. 供应链安全

- 官方 Manifest digest、签名 trust validation 和 payload SHA256 保留。
- cache hit 必须重新验证 Blob hash 或使用经证明的可信验证元数据；严格模式需定期 rehash。
- Lockfile exact URL/hash；不自动替换为其他镜像。
- 下载 URL 必须校验 scheme/host policy（兼容微软重定向时记录最终 URL/证书信息，不以简单 host allowlist 误拒必要 redirect）。
- 不接受 Manifest 内未校验的相对路径写入 cache。
- 日志不打印 secret headers。
- GC 不跟随符号链接。
- Archive extractor 在 staging sandbox 内处理；不执行包内脚本。
- MSIs 仅 administrative extract，除非另有明确产品功能，不做 MSI install。

## 6. Cargo dependency 审查

新增依赖前 Agent 必须在 `DECISIONS.md` 记录：
- crate + version。
- 解决的问题。
- 是否可用现有依赖实现。
- license。
- MSRV。
- maintenance/activity。
- default features 和二进制体积影响。
- Windows/Linux compile compatibility。
- 安全/解析器风险。
- 是否可用真实 fixture 测试。

特别 MSI：`msi` crate 的数据库读取不等于完整文件提取。必须确认 CAB parser、MSI table schemas 和 sequence mapping。

## 7. 发布前检查

- [ ] README/README_zh 功能表一致。
- [ ] CLI help 示例可运行。
- [ ] `plan`/`apply` 的 stdout/stderr 规范明确。
- [ ] Lock v1/v2 migration notes。
- [ ] 默认无网络大型测试。
- [ ] release archive 不含 Microsoft payload。
- [ ] Cargo.lock 同步。
- [ ] license notice/attribution 更新。
- [ ] Git diff 不包含临时 cache、真实 payload、user data。
- [ ] 版本号/CHANGELOG 由用户或 release 流程决定，Agent 不擅自发版。


---

<!-- SOURCE: 11_AGENT_PROGRESS_REPORT_TEMPLATE.md -->

# Agent 进度、决策与测试报告模板

Agent 必须持续更新，不要只在最终阶段写报告。

## 1. BASELINE.md 模板

```markdown
# Baseline Audit

- Date/time:
- Repository path:
- HEAD SHA:
- Branch:
- Git status:
- Rust version:
- Cargo version:
- OS:
- Canonical CI command:

## Relevant files and symbols
| Feature | Path | Symbol | Existing tests | Status | Evidence |
|---|---|---|---|---|---|

## Current install flow
[从 CLI/API → Manifest → Resolve → Download → Cache → Extract → receipt → env 的真实调用路径]

## Target root layout
[列出实际目录，并区分 download cache / installed files / receipts]

## Baseline test results
| Command | Exit | PASS/FAIL/BLOCKED | Log | Notes |
|---|---:|---|---|---|

## Risks
[按 P0/P1/P2]

## Unknowns
[没有证据的事项]
```

## 2. DECISIONS.md 模板

```markdown
# Engineering Decisions

## DEC-001: <title>
- Date:
- Phase:
- Context:
- Options considered:
- Decision:
- Why:
- Trade-offs:
- Evidence/tests:
- Revisit condition:
```

所有偏离规格的设计必须记录决策和理由。

## 3. PROGRESS.md 模板

```markdown
# Progress

## Current phase
Pxx — title

## Status
NOT_STARTED / IN_PROGRESS / BLOCKED / COMPLETE

## Completed
- [x] ...

## Remaining
- [ ] ...

## Files changed
- `path`: reason

## Tests
- PASS: command
- FAIL: command + reason
- BLOCKED: command + missing environment

## Risks / regressions
...

## Next exact action
[写出下一步要改哪个文件/添加哪个测试，不写模糊愿望]
```

## 4. TEST-RESULTS.md 模板

| Test ID | Phase | Command/Test | OS/Privilege | Result | Evidence/log | Notes |
|---|---|---|---|---|---|---|

结果只能为：
- `PASS`
- `FAIL`
- `BLOCKED_ENVIRONMENT`
- `NOT_RUN`

不得把 `NOT_RUN` 或 `BLOCKED_ENVIRONMENT` 统计为 PASS。

## 5. 每阶段报告模板

```markdown
# Phase Pxx Report

## Scope

## Baseline code findings
- path/symbol:
- existing behavior:
- test evidence:

## Changes
- path:
- changed symbol:
- behavior:

## Tests added
- test name:
- bug it detects:
- why it would fail before change:

## Commands executed
| Command | Exit | Result | Output/log |
|---|---:|---|---|

## Compatibility
- CLI:
- Rust API:
- JSON/schema:
- Windows behavior:

## Security review
- target path:
- path traversal:
- ownership:
- rollback:
- privilege:

## Unresolved items

## Exit Gate
- [ ] all criteria checked

## Decision
COMPLETE / BLOCKED / NEEDS_REVISION
```

## 6. FINAL_REPORT.md 模板

```markdown
# Final Engineering Report

## Summary

## Delivered phases

## Not delivered

## Architecture changes

## Public CLI/API/schema changes

## Attribution / license changes

## Test matrix
| Layer | OS | Command | Passed | Failed | Blocked | Not run |
|---|---|---|---:|---:|---:|---:|

## Fault injection summary
- total points:
- passed:
- failed:
- blocked:

## Real Windows/MSI acceptance
- MSI used:
- source/license:
- Windows version:
- privilege context:
- msiexec args:
- exit code:
- log path:
- output validation:
- UAC observed:

## Known risks

## Manual actions needed from user

## Exact git diff summary
```


---

<!-- SOURCE: 12_PREFLIGHT_REVIEW.md -->

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
