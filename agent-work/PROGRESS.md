# Progress

## Current phase
P03 — File-level Staging / Transaction / Recovery (COMPLETE, pending push + CI verification)

## Status
P00 COMPLETE (b784452). P01 COMPLETE (448f877). P02 COMPLETE (21e3603) — all CI success. P03 pending push.

## Completed (P03)
- [x] Transaction workspace (.msvc-kit/transactions/<tx>) with journal.jsonl, per-package staging, backup dir
- [x] Merge with cross-package collision detection (deterministic order)
- [x] Commit pre-pass: Create/Skip/ReplaceOwned/UnownedConflict (refuse before mutation)
- [x] Write-ahead journal + hash-gated per-path rollback + ownership/manifest revert
- [x] recover_pending (idempotent) wired into both extract-and-finalize paths
- [x] InstalledManifest schema (installed-manifest.v1) with golden tests
- [x] 9 transaction tests incl. 11-point fault-injection matrix; found and fixed a real backup-restore data-loss bug
- [x] Full verification green: fmt/clippy/681 tests/0 failed + doc

## Remaining
- [ ] push P03 commits; verify CI workflow conclusion=success on the pushed sha
- [ ] P04 (next): MSI backend abstraction + native MSI spike

## Files changed (P03)
- `src/installer/transaction.rs`: new public module (workspace, journal, merge, commit, rollback, recovery, fault hook)
- `src/installer/mod.rs`: extract_and_finalize_* wired through transactions; probe visibility
- `src/installer/contracts.rs`: InstalledManifest schema + golden test
- `tests/transaction_tests.rs`: fault matrix + exit-gate scenarios (9 tests)
- `agent-work/*`: P03 report

## Tests
- PASS: `cargo fmt --all -- --check` (D:\rust\p02c-fmt.log, final P03 round)
- PASS: `cargo clippy --all-targets --all-features -- -D warnings` (D:\rust\p02c-clippy.log)
- PASS: `cargo test --all-features --no-fail-fast` — 681 passed / 0 failed / 7 ignored (D:\rust\p02c-test.log)
- PASS: `cargo test --all-features --test transaction_tests` — 9/9 (D:\rust\p03a-test.log)
- PASS: `cargo test --doc` — 23 passed; `cargo doc` -D warnings (D:\rust\p02c-test-doc.log, p02c-doc.log)
- P02 records (p02c-*.log) retained in TEST-RESULTS.md

## Risks / regressions
- Transaction staging + commit adds journal fsyncs per file event (I/O cost accepted).
- Legacy roots are adopted (backup + journal) rather than refused — one-time migration path.
- Recovery cannot restore bytes if the backup was lost in the pre-fix journal format; the format bug was caught by the fault matrix and fixed before shipping.

## Next exact action
Commit P03 as `feat(installer)` (transaction core + wiring + schema) and `test(installer)` (fault matrix), `docs(agent-work)`; push; poll CI for conclusion=success. Then P04: keep msiexec backend, add MsiBackendPreference abstraction + fake process-runner tests (1618/1603/1619/1620, no /i), verbose log, and the native MSI feasibility report (msi + cab crates).
