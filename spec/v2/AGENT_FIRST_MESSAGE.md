# 直接发送给 Coding Agent 的第一条消息

你正在用户自己的 `msvc-kit` fork 仓库中工作。请先完整阅读本目录中的：

1. `00_AGENT_MASTER_PROMPT.md`
2. `01_BASELINE_AND_GAP_AUDIT.md`
3. `02_REQUIREMENTS_AND_INVARIANTS.md`
4. `03_TARGET_ARCHITECTURE_AND_SCHEMAS.md`
5. `04_PLAN_AND_DEPENDENCY_RESOLVER.md`
6. `05_SAFE_EXTRACTORS_AND_TRANSACTIONS.md`
7. `06_PRUNING_CACHE_LOCKFILE.md`
8. `07_DETECTION_AND_PROVIDER_API.md`
9. `08_TEST_STRATEGY_AND_CASES.md`
10. `09_IMPLEMENTATION_ROADMAP.md`
11. `10_ATTRIBUTION_SECURITY_AND_RELEASE.md`
12. `11_AGENT_PROGRESS_REPORT_TEMPLATE.md`
13. `12_PREFLIGHT_REVIEW.md`

严格遵守 `00_AGENT_MASTER_PROMPT.md`。先阅读 `12_PREFLIGHT_REVIEW.md` 中的修订点。**从 P00 开始：先审计仓库、确认 git 状态、运行现有基线测试并生成报告；不要立即修改核心代码。** P00 完成后，按 `09_IMPLEMENTATION_ROADMAP.md` 从 P01 到 P11 顺序实施。每个阶段必须先增加能暴露问题的测试，再实现代码，运行测试、修复失败、做回归并更新报告。涉及事务恢复时，单进程错误注入不等于进程崩溃测试。任何阶段 Exit Gate 未通过，不得进入下一阶段。

不要问用户普通技术选择题；按规格中的默认设计执行并把决定记录在 `agent-work/DECISIONS.md`。不要 push、不要改 remote、不要 commit、不要进行管理员操作、不要安装真实 VS/MSVC、不要默认下载大型 payload。遇到危险数据操作或必须由用户批准的真实 Windows/MSI 集成测试时，停下并说明原因。

用户要求的是可运行、可验证的实现，不是设计建议。最终必须交付代码、自动化测试、更新后的文档和 `agent-work/FINAL_REPORT.md`；不得把未执行测试标记为通过。
