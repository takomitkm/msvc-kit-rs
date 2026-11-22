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
