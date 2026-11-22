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
