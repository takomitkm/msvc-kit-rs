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

- Mark：所有 active transactions、未完成 plans、有效 Toolchain Lock、安装 receipts、旧安装仍引用的 payload，以及活跃 partial downloads。
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
