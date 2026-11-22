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
