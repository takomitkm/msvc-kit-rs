# Phase P09 Report — Detection / Reuse

## Scope
Per roadmap P09: read-only detection of MSVC/Windows SDK roots with capability validation, reuse planner (ReusableExact/ReusableWithAdditions/Incompatible/Unknown), all tests fixture-based (no PATH scanning), and read-only guarantee.

## Baseline code findings
- No detection module existed (P00 audit: PARTIAL). The existing `find_vs_instances` is VS/UBT-specific and is left unchanged.
- The P03 transaction provides exactly the managed-root ownership model that `detect` checks against.

## Changes
- `src/installer/detection.rs` (new public module):
  - `DetectedToolchain` (root/kind/origin/version/target_arches/validation) with `ToolchainKind` (Msvc/WindowsSdk/Llvm/MinGw/BuildTool) and `DetectionOrigin`.
  - `detect_msvc(root, arch)` — validates `VC/Tools/MSVC/{version}/bin/Host{host}/{target}/cl.exe|link.exe|lib.exe`, non-empty `include/`, non-empty `lib/{target}/`. Discover the version directory lexically last.
  - `detect_windows_sdk(root, arch)` — validates `Include/{version}/um/`, `Lib/{version}/um/{arch}/`, `Lib/{version}/ucrt/{arch}/`.
  - `detect_build_tool(exe, tool)` — CMake/Ninja/NASM existence check.
  - `plan_reuse(detected, required_capabilities)` — compares required capability prefixes against the detected kind's namespace (mismatch → Incompatible); then checks validation results (all present → ReusableExact, some missing → ReusableWithAdditions, no requirements → Unknown).
  - All detection functions are strictly read-only (only `is_file()`/`is_dir()`/`read_dir()`).

## Tests added (in-module, 9)
- `msvc_detector_finds_complete_installation` (positive fixture + read-only check)
- `msvc_detector_rejects_cl_exe_only_false_positive` (link.exe/lib.exe missing, headers present — no false positive)
- `msvc_detector_handles_empty_root` (empty temp dir → no version, not all present)
- `sdk_detector_finds_complete_installation`
- `sdk_detector_rejects_incomplete_installation` (Include only, no Lib)
- `build_tool_detector_checks_exe_existence`
- `reuse_planner_exact_when_all_present`
- `reuse_planner_with_additions_when_partial` (cl.exe-only fixture)
- `reuse_planner_incompatible_for_wrong_kind` (SDK root asked for MSVC capabilities)
- `reuse_planner_unknown_when_no_requirements`

## Commands executed
| Command | Exit | Result | Output/log |
|---|---:|---|---|
| `cargo test --all-features --no-fail-fast` (final) | 0 | PASS — 724 passed / 0 failed / 7 ignored | D:\rust\p02c-test.log |
| `cargo fmt --all -- --check` (final) | 0 | PASS | D:\rust\p02c-fmt.log |
| `cargo clippy --all-targets --all-features -- -D warnings` (final) | 0 | PASS | D:\rust\p02c-clippy.log |
| `cargo test --doc` / `cargo doc` -D warnings (final) | 0 | PASS | D:\rust\p02c-test-doc.log, p02c-doc.log |
| Intermediate rounds | — | fixes: Architecture Ord (HashSet instead of BTreeSet), needless borrows, plan_reuse prefix-mismatch logic (compare required capability prefix vs detected kind, not detected checks) | D:\rust\p02c-*.log |

## Compatibility
- CLI: additive — no new CLI flag (detect subcommand lands with the plan/apply integration).
- Rust API: additive — `installer::detection` module public; nothing removed or renamed.

## Security review
- Read-only: detection functions only call `is_file()`, `is_dir()`, `read_dir()` on the user-specified root. No writes, no registry, no PATH modification.
- False positive: the `cl_exe_only` test proves that having cl.exe alone does not yield all_present.

## Unresolved items
- LLVM/MinGW detectors are typed (ToolchainKind::Llvm/MinGw) but not yet implemented (no capability checks yet — the spec 07 §2 "第二阶段 LLVM/MinGW" is future work).
- `--detect` CLI flag and the JSON detection output land with the plan/apply integration.
- Reuse planner does not yet integrate with the plan (planner integration is P10+).

## Exit Gate
- [x] Fixtures don't depend on local PATH (all tests use temp dirs)
- [x] Detection doesn't modify external state (read-only check in test)
- [x] Only compiler exe doesn't give false positive (cl_exe_only test)
- [x] Exact reuse / partial / incompatible state tests

## Decision
COMPLETE
