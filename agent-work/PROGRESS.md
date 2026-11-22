# Progress

## Current phase
P04 — MSI Backend abstraction + Native feasibility (COMPLETE, pending push + CI verification)

## Status
P00 COMPLETE (b784452). P01 COMPLETE (448f877). P02 COMPLETE (21e3603). P03 COMPLETE (67d87dc) — all CI success. P04 pending push.

## Completed (P03)
- [x] Transaction workspace (.msvc-kit/transactions/<tx>) with journal.jsonl, per-package staging, backup dir
- [x] Merge with cross-package collision detection (deterministic order)
- [x] Commit pre-pass: Create/Skip/ReplaceOwned/UnownedConflict (refuse before mutation)
- [x] Write-ahead journal + hash-gated per-path rollback + ownership/manifest revert
- [x] recover_pending (idempotent) wired into both extract-and-finalize paths
- [x] InstalledManifest schema (installed-manifest.v1) with golden tests
- [x] 9 transaction tests incl. 11-point fault-injection matrix; found and fixed a real backup-restore data-loss bug
- [x] Full verification green: fmt/clippy/681 tests/0 failed + doc

## Completed (P04)
- [x] MsiBackendPreference (auto/msiexec/native/fallback) — native+fallback report the parity gate
- [x] MsiProcessRunner seam + SystemRunner; MsiexecBackend with /a /qn TARGETDIR /L*V, 1618 retry budget, code-specific guidance
- [x] extract_msi_sync delegates to the backend (duplicate retry loop removed)
- [x] 6 fake-runner tests (arg shape, no /i, retry counts/delays, diagnostics, spawn failure)
- [x] Native MSI feasibility report: parity gate unmet (no legal fixtures) -> native stays disabled (agent-work/PHASES/P04-msi-feasibility.md)

## Remaining
- [ ] push P04 commits; verify CI workflow conclusion=success on the pushed sha
- [ ] P05 (next): Pruning Engine (rule schema, profiles, staging-only execution, removed-file receipts)

## Files changed (P04)
- `src/installer/msi.rs`: new (preference enum, runner seam, msiexec backend + tests)
- `src/installer/extractor.rs`: extract_msi_sync delegates to the backend; format_msiexec_failure folded into backend diagnostics
- `src/installer/mod.rs`, `src/lib.rs`: module registration + re-exports
- `agent-work/PHASES/P04-msi-feasibility.md`: feasibility report

## Files changed (P06)
- `src/installer/cas.rs`: new public module (BlobStore, pins, GC)
- `src/bin/msvc-kit.rs`: cache subcommand
- `src/installer/mod.rs`, `src/lib.rs`: registration + re-exports
- `agent-work/PHASES/P06-cas-cache.md`: phase report

## Files changed (P05)
- `src/installer/pruning.rs`: new public module (profiles, rules schema, planner, executor)
- `src/installer/transaction.rs`: prune_staged (journal integration)
- `src/installer/mod.rs`: *_with_profile variants
- `src/bin/msvc-kit.rs`: --prune-profile flag
- `src/lib.rs`: pruning re-exports
- `agent-work/PHASES/P05-pruning.md`: phase report

## Files changed (P03)
- `src/installer/transaction.rs`: new public module (workspace, journal, merge, commit, rollback, recovery, fault hook)
- `src/installer/mod.rs`: extract_and_finalize_* wired through transactions; probe visibility
- `src/installer/contracts.rs`: InstalledManifest schema + golden test
- `tests/transaction_tests.rs`: fault matrix + exit-gate scenarios (9 tests)
- `agent-work/*`: P03 report

## Tests
- PASS: `cargo fmt --all -- --check` (D:\rust\p02c-fmt.log, final P06 round)
- PASS: `cargo clippy --all-targets --all-features -- -D warnings` (D:\rust\p02c-clippy.log)
- PASS: `cargo test --all-features --no-fail-fast` — 697 passed / 0 failed / 7 ignored (D:\rust\p02c-test.log)
- NOT_RUN: minimal-profile compile/link acceptance (opt-in real-payload test; minimal profile is rule-free by construction)
- PASS: `cargo test --doc` — 23 passed; `cargo doc` -D warnings (D:\rust\p02c-test-doc.log, p02c-doc.log)
- P02 records (p02c-*.log) retained in TEST-RESULTS.md

## Risks / regressions
- Transaction staging + commit adds journal fsyncs per file event (I/O cost accepted).
- Legacy roots are adopted (backup + journal) rather than refused — one-time migration path.
- Recovery cannot restore bytes if the backup was lost in the pre-fix journal format; the format bug was caught by the fault matrix and fixed before shipping.

## Next exact action
Commit P04 as `feat(installer)` (MSI backend abstraction + hardened wrapper) and `docs(agent-work)` (feasibility report); push; poll CI for conclusion=success. Then P05: pruning rule schema (msvc-kit.pruning-rules.v1), full/portable/minimal profiles, staging-only execution wired into the transaction journal, removed-file receipt records, and positive/negative/version fixtures per rule.
