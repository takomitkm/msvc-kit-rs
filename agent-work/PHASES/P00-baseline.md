# Phase P00 Report — Baseline Audit

## Scope
Audit only per master prompt §C / roadmap P00: git state, module tree, feature status with evidence, baseline test execution, risk register, report scaffolding. No core code changes. Two baseline-repair fixes were required to restore a green CI baseline (DEC-001/DEC-002, DEC-005).

## Baseline code findings
- `src/installer/extractor.rs:223` (`extract_vsix_sync`) and `:473` (`extract_cab_sync`): output path is `target_dir.join(relative)` with no traversal/absolute/ADS/device-name/collision guards → P01 target.
- `src/installer/mod.rs:121-143,180-226`: extraction markers use path+size+mtime fingerprints, writes are best-effort, marker hits never verify output → P01 legacy handling, P03 replacement.
- `src/installer/mod.rs:146-258`: parallel extraction writes straight into the shared install root; no staging/journal/rollback → P03 target.
- `src/toolchain_lock.rs`: Lock v1 + Receipt v2 exist and are strict (deny_unknown_fields, rehash on record); receipt covers acquisition only.
- `src/downloader/manifest.rs:87`: `VsPackage.dependencies` parsed but never consumed; no dependency resolution anywhere → P08 target.
- `src/downloader/traits.rs`: `ComponentDownloader`/`ComponentType::{Msvc,Sdk}` is download-centric → P10 target.
- `src/storage.rs`: OS file locks + atomic write primitives exist and are reused (good foundation for P03/P06).
- CLI `Commands` enum has no `plan`/`apply`/`detect`/`recover` → P02/P09 targets.
- Existing behavior is compatible with the workpack's "known facts" and with spec 01's gap list; no spec item was found already implemented beyond what spec 01 credits (no duplicate-implementation risk for the planned work).

## Changes
- `tests/cli_exit_code_tests.rs`: `get_binary_path()` — upward search for the CLI binary across cargo 1.100's test layout, classic fallback preserved.
- `tests/update_tests.rs`: identical repair (duplicated helper).
- `src/install_into_vs/mod.rs:252`: doc-comment placeholder backticked.
- `agent-work/`: BASELINE.md, RISKS.md, TEST-RESULTS.md, DECISIONS.md, PROGRESS.md, PHASES/P00-baseline.md.
- No runtime code paths changed; no public API, CLI, JSON schema or lock format changed.

## Tests added
- None in P00 (audit phase). The existing 24 test suites + doc tests constitute the baseline; all recorded. P01 will add the traversal suite that exposes R-001/R-002.

## Commands executed
| Command | Exit | Result | Output/log |
|---|---:|---|---|
| `cargo fmt --all -- --check` (after component install) | 0 | PASS | D:\rust\p00b-fmt.log, p00c-fmt.log |
| `cargo clippy --all-targets --all-features -- -D warnings` | 0 | PASS | D:\rust\p00b-clippy.log, p00d-clippy.log |
| `cargo test --all-features` (baseline, pre-fix) | 101 | FAIL (17) | D:\rust\p00-test.log |
| `cargo test --all-features --no-fail-fast` (baseline, pre-fix) | 101 | FAIL (21 across 2 suites) | D:\rust\p00b-test.log |
| OUT_DIR experiment (`copy msvc-kit.exe` → suite rerun) | 0 | PASS 20/20 | ssh transcript |
| `cargo test --all-features --no-fail-fast` (post-fix) | 0 | PASS 621/0/7 ign | D:\rust\p00c-test.log |
| `cargo test --doc` (post-fix) | 0 | PASS 23 | D:\rust\p00c-test-doc.log |
| `cargo doc --no-deps --document-private-items` + `-D warnings` (post-fix) | 0 | PASS | D:\rust\p00c-doc.log |
| gh api: CI run 44968093273 jobs + logs (pre-existing main red) | — | FAIL reproduced | .tmp/msvc-kit/ci-rust-tests.log, ci-docs.log |

All cargo commands ran on the school machine (Win11, cargo 1.100.0, standard user) via schtasks; the local machine never compiled (DEC-003).

## Compatibility
- CLI: unchanged.
- Rust API: unchanged.
- JSON/schema: unchanged.
- Windows behavior: unchanged (two test files + one doc comment only).

## Security review
- target path: no new target-path handling introduced.
- path traversal: audit confirmed the gap (R-001/R-002, `agent-work/RISKS.md`); fix scheduled P01.
- ownership: audit confirmed absence (R-005); scheduled P03.
- rollback: audit confirmed absence (R-003); scheduled P03.
- privilege: no elevation paths touched; msiexec usage confirmed `/a`-only (no `/i` anywhere in `src/` — verified by search).

## Unresolved items
- Windows signed-catalog trust chain and real-MSI `msiexec /a` behavior remain NOT_TESTED (network/privilege-gated per spec B5/B6; opt-in only).
- Coverage workflow's codecov upload step fails on the fork for lack of token (known fork noise; its test step is the signal).

## Exit Gate
- [x] Every user goal has real code/test evidence and a status (BASELINE.md feature table)
- [x] Current shared-root layout documented (BASELINE.md "Target root layout")
- [x] Every baseline test failure recorded and dispositioned (TEST-RESULTS.md)
- [x] No user files modified, no remote/branch changes (git state re-checked before commit)

## Decision
COMPLETE
