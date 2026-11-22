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
