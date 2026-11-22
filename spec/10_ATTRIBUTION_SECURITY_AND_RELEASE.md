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
