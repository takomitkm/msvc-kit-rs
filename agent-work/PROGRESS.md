# Progress

## Current phase
P00 — Baseline audit (audit + baseline repair + reports; no core code changes)

## Status
COMPLETE (pending push + CI verification)

## Completed
- [x] Git state recorded: main @ 1e24a75, clean, origin=takomitkm/msvc-kit-rs, no remote changes
- [x] README/README_zh/Cargo.toml/justfile/vx.toml/CI workflows read; canonical CI commands identified
- [x] Module tree and download→extract→install flow mapped (`agent-work/BASELINE.md`)
- [x] Spec §C concept searches (`msiexec`, `DownloadIndex`, `ToolchainLock`, markers, `find_vs_instances`, `DownloadPreview`, `dry_run`, ...) — all concepts located, states assigned
- [x] Extractor write paths audited (`extractor.rs` VSIX:223 / CAB:473 unguarded `join`)
- [x] Baseline tests executed on school machine (initial fail → root cause proven → green after repair)
- [x] GitHub CI at HEAD diagnosed: Rust Tests + Documentation red from toolchain drift (pre-existing, not machine-specific)
- [x] Baseline repairs: robust CLI-binary lookup in tests (2 files), doc-comment escape (1 file); verified fmt/clippy/test/doc all green (621+23 passed, 0 failed)
- [x] `agent-work/BASELINE.md`, `RISKS.md`, `TEST-RESULTS.md`, `DECISIONS.md`, `PHASES/P00-baseline.md` written

## Remaining
- [ ] push P00 commits; verify CI workflow conclusion=success on the pushed sha
- [ ] P01 (next): SafePathResolver + data contract schemas + traversal test suite

## Files changed (P00)
- `tests/cli_exit_code_tests.rs`: binary lookup searches upward from the test exe (cargo ≥1.100 OUT_DIR layout) — baseline repair DEC-001
- `tests/update_tests.rs`: same repair, duplicated helper
- `src/install_into_vs/mod.rs`: doc comment placeholder escaped (rustdoc `-D warnings`) — DEC-002
- `agent-work/*`: P00 reports (new)

## Tests
- PASS: `cargo fmt --all -- --check` (D:\rust\p00c-fmt.log)
- PASS: `cargo clippy --all-targets --all-features -- -D warnings` (D:\rust\p00d-clippy.log)
- PASS: `cargo test --all-features --no-fail-fast` — 621 passed / 0 failed / 7 ignored (D:\rust\p00c-test.log)
- PASS: `cargo test --doc` — 23 passed (D:\rust\p00c-test-doc.log)
- PASS: `cargo doc --no-deps --document-private-items` + `RUSTDOCFLAGS=-D warnings` (D:\rust\p00c-doc.log)
- FAIL (baseline, repaired in P00): cli_exit_code/update binary lookup — see TEST-RESULTS P00-T03..T09
- NOT_RUN: network-gated `#[ignore]` tests (spec B6 — opt-in only)

## Risks / regressions
- None introduced: fixes are test-infra + doc-comment only; full suite green before and after on the same tree apart from the fixes.
- Security/design risks catalogued in `agent-work/RISKS.md` (P0: extractor path escape, direct-to-root parallel extraction, weak markers, no ownership model).

## Next exact action
Commit the three repair files as `fix(tests)`/`docs:` commits, commit `agent-work/` reports, push, then poll `gh api repos/takomitkm/msvc-kit-rs/actions/runs?branch=main` until the CI workflow run for the new sha concludes; require conclusion=success. Then start P01 with the traversal test suite against `extract_vsix_sync`/`extract_cab_sync`.
