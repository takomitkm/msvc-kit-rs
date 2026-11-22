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
