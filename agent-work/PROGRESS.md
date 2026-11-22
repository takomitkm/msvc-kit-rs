# Progress

## Current phase
P02 — Canonical InstallPlan / CLI Dry Run (COMPLETE, pending push + CI verification)

## Status
P00 COMPLETE (b784452, CI success). P01 COMPLETE (448f877, CI success). P02 COMPLETE pending push.

## Completed (P02)
- [x] `ManifestIdentity` + `VsManifest::fetch_with_identity` (manifest digest exposure)
- [x] Public `InstallPlanDocument` schema + `create_install_plan` planner (cache probing, per-payload actions, blocking conflicts, disk estimates)
- [x] Stable `plan_id` (SHA256 over canonical form without created_at) + tamper detection
- [x] `plan` CLI (text/json, --output) + `apply` CLI (verify -> conflicts -> revalidation -> pinned execution)
- [x] Stale detection: manifest digest change, cached payload digest drift, free-space drop
- [x] Channel selector accepts explicit http(s) URLs (custom channels, URL-derived cache slug)
- [x] 7 mockito plan tests + 3 CLI smoke tests; full suite 671 passed / 0 failed

## Remaining
- [ ] push P02 commits; verify CI workflow conclusion=success on the pushed sha
- [ ] P03 (next): file-level staging / transaction journal / recovery

## Files changed (P02)
- `src/downloader/manifest.rs`: ManifestIdentity + fetch_with_identity (+ test call sites)
- `src/installer/plan.rs`: new public plan module (document, planner, plan_id, revalidation)
- `src/installer/contracts.rs`: plan types moved out; internal receipt/ownership/journal schemas remain
- `src/bin/msvc-kit.rs`: plan/apply subcommands
- `src/vs_channel.rs`: URL channel selector + URL-derived cache slug
- `src/lib.rs`, `src/installer/mod.rs`: re-exports
- `Cargo.toml`: windows-sys Win32_Storage_FileSystem feature
- `tests/plan_tests.rs`, `tests/cli_exit_code_tests.rs`: new tests
- `agent-work/*`: P02 report

## Files changed (P01, retained for reference)
- `src/installer/path_safety.rs`: resolver, occupancy, caps, unit tests
- `src/installer/extractor.rs`: two-pass guarded VSIX/CAB extraction
- `src/installer/mod.rs`: marker v2
- `tests/installer_safety_tests.rs`: spec 08 §3 matrix

## Tests
- PASS: `cargo fmt --all -- --check` (D:\rust\p02c-fmt.log)
- PASS: `cargo clippy --all-targets --all-features -- -D warnings` (D:\rust\p02c-clippy.log)
- PASS: `cargo test --all-features --no-fail-fast` — 671 passed / 0 failed / 7 ignored (D:\rust\p02c-test.log)
- PASS: `cargo test --doc` — 23 passed (D:\rust\p02c-test-doc.log)
- PASS: `cargo doc --no-deps --document-private-items` + `RUSTDOCFLAGS=-D warnings` (D:\rust\p02c-doc.log)
- FLAKE (pre-existing, load-dependent): `downloader::common_tests::a_corrupted_first_transfer_is_refetched_and_succeeds` failed once under full-suite parallel load; passes in isolation and in the two following full runs (upstream test from c0b9e0e; no download-path changes in P02)
- P01 records (D:\rust\p01c-*.log) retained in TEST-RESULTS.md

## Risks / regressions
- Apply is not yet transactional — half-install risk (R-003) unchanged until P03; the plan pin check closes the silent-package-swap gap.
- Disk estimates leave staging/backup/final as null (bounded confidence) until P03/P06 data exists.
- Plan identity includes the normalized target root string; path-independent identity is deferred to P03 (root identity) / P08.

## Next exact action
Commit P02 as `feat(plan)` (planner + CLI + manifest identity), `test(plan)` (plan tests), `docs(agent-work)`; push; poll CI for conclusion=success. Then P03: per-transaction workspace (.msvc-kit/transactions/<tx>), write-ahead journal using the P01 schemas, file-level backup/commit/rollback, recovery command, ownership index write-through, and fault-injection tests at the spec 08 §10 points.
