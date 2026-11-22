# msvc-kit Fork 工程工作包（给 Coding Agent 执行）

**目标仓库：** 用户自己的 `msvc-kit` fork  
**设计参考：** [`tgbender/portablemsvc`](https://github.com/tgbender/portablemsvc)  
**上游基线：** [`loonghao/msvc-kit`](https://github.com/loonghao/msvc-kit)  
**工作包日期：** 2026-11-22（v2 preflight 修订：2026-11-23）  
**语言：** 规格与执行说明为中文；代码、CLI、结构体和测试名称按 Rust 项目惯例使用英文。

## 这是什么

本目录是一套可以直接交给 coding agent 的工程规格。它不只是功能想法，而是包括：当前基线、目标行为、数据模型、安装安全约束、模块边界、逐阶段任务、测试矩阵、故障注入和交付报告格式。

**核心要求：用自动化测试弥补模型判断不稳定。** Agent 必须先读本地仓库、运行基线测试，再按阶段实现；每个阶段都要运行该阶段测试和回归测试，生成报告。禁止一次性大改或把“编译通过”当作完成。

## 文件阅读顺序

1. `00_AGENT_MASTER_PROMPT.md`：主指令。先把它完整交给 Agent。
2. `01_BASELINE_AND_GAP_AUDIT.md`：两个项目的当前实现基线与已知差距。
3. `02_REQUIREMENTS_AND_INVARIANTS.md`：必须始终成立的行为、安全边界和非目标。
4. `03_TARGET_ARCHITECTURE_AND_SCHEMAS.md`：模块架构、数据模型和数据契约。
5. `04_PLAN_AND_DEPENDENCY_RESOLVER.md`：安装计划、Dry Run、Build Profile 和依赖解析。
6. `05_SAFE_EXTRACTORS_AND_TRANSACTIONS.md`：MSI/VSIX/CAB、路径安全、事务式安装和恢复。
7. `06_PRUNING_CACHE_LOCKFILE.md`：裁剪、内容寻址缓存、Lockfile、Receipt 和 GC。
8. `07_DETECTION_AND_PROVIDER_API.md`：环境探测、外部复用和 Provider 扩展。
9. `08_TEST_STRATEGY_AND_CASES.md`：测试总纲、具体用例、故障注入和 CI 分层。
10. `09_IMPLEMENTATION_ROADMAP.md`：严格实施顺序、每阶段完成门槛。
11. `10_ATTRIBUTION_SECURITY_AND_RELEASE.md`：归因、许可证、供应链和发布要求。
12. `11_AGENT_PROGRESS_REPORT_TEMPLATE.md`：Agent 每阶段必须更新的报告模板。
13. `12_PREFLIGHT_REVIEW.md`：本规格自身的预检结果、已修正问题和仍需 Agent 在本地验证的事项。

## 推荐交付结构

Agent 应在 fork 仓库内新增或维护：

```text
agent-work/
├── BASELINE.md
├── DECISIONS.md
├── PROGRESS.md
├── TEST-RESULTS.md
├── RISKS.md
└── PHASES/
    ├── P00-baseline.md
    ├── P01-plan-model.md
    ├── P02-safe-extraction.md
    └── ...
```

如果项目已有类似文档目录，可复用，不要重复创建同义文件。

## v2 preflight 修订

v2 增补了 pinned source snapshots、MSI 自定义动作信任边界、ZIP 目录 entry 规则、reparse/TOCTOU 要求、真正的子进程 crash-recovery 测试，以及外部 Lockfile 的 cache pin 机制。详见 `12_PREFLIGHT_REVIEW.md`。

## 最重要的三条

1. **先检查本地 fork 的真实代码。** 本工作包的代码路径是审计入口，不是永久真理；main 可能变化，fork 也可能不同。Agent 必须在 `BASELINE.md` 中记录实际 commit、dirty state、真实实现位置和证据。
2. **不破坏当前布局。** msvc-kit 的 `target_dir` 同时承载 `downloads/`、`VC/`、`Windows Kits/`、`.msvc-kit/` 等内容。严禁把整个 `target_dir` 当成单个安装目录整体替换。事务必须只操作本次安装明确拥有的文件，保留其他版本、缓存和未知内容。
3. **失败不能伪装成功。** 不允许跳过失败测试、不允许用空实现/假数据让测试绿、不允许未经验证宣称 MSI native extractor 或回滚完成。

## 一键复制主指令

见 `00_AGENT_MASTER_PROMPT.md`。建议先让 Agent 只执行 Phase 0（审计与基线），阅读报告后再继续；也可以允许它按 `09_IMPLEMENTATION_ROADMAP.md` 自动逐阶段执行，但每阶段必须停在验收门槛，不得跨过失败门槛。
