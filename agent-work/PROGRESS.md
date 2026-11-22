# Progress

## Current phase
P01 — Data contracts + Path Security (COMPLETE, pending push + CI verification)

## Status
P00 COMPLETE (b784452, CI success). P01 COMPLETE pending push.

## Completed (P01)
- [x] Unified `SafeRelativePath` (Windows semantics, host-independent) + `PathOccupancy` conflict detection + size caps (`src/installer/path_safety.rs`)
- [x] VSIX/ZIP and CAB extractors wired to the safety layer: validate-all-then-write, symlink rejection, content-hash duplicate resolution (DEC-007)
- [x] Spec 08 §3 test matrix as `tests/installer_safety_tests.rs` (24 integration tests; round-1 run against unfixed code exposed 19 real failures)
- [x] Marker v2 content identity + legacy v1 ignored-and-upgraded (DEC-008)
- [x] Minimal internal schemas + golden tests: InstallPlan / InstallReceipt / OwnershipIndex / TransactionJournal (`src/installer/contracts.rs`, DEC-009)
- [x] Dependency review + addition of `unicode-normalization 0.1` (DEC-006)
- [x] Public API compatibility review: zero public surface change
- [x] Full verification green: fmt, clippy `-D warnings`, 660 tests / 0 failed (+7 network-ignored), 23 doc tests, rustdoc `-D warnings`

## Remaining
- [ ] push P01 commits; verify CI workflow conclusion=success on the pushed sha
- [ ] P02 (next): canonical InstallPlan + `plan` CLI + stable plan_id + stale detection

## Files changed (P01)
- `src/installer/path_safety.rs`: new — resolver, occupancy, caps, unit tests
- `src/installer/extractor.rs`: two-pass guarded VSIX/CAB extraction
- `src/installer/mod.rs`: marker v2 + module registration
- `src/installer/contracts.rs`: new — four schema documents + golden tests
- `tests/installer_safety_tests.rs`: new — spec 08 §3 matrix for both extractors
- `Cargo.toml`/`Cargo.lock`: unicode-normalization 0.1
- `agent-work/*`: reports updated

## Tests
- PASS: `cargo fmt --all -- --check` (D:\rust\p01c-fmt.log)
- PASS: `cargo clippy --all-targets --all-features -- -D warnings` (D:\rust\p01c-clippy.log)
- PASS: `cargo test --all-features --no-fail-fast` — 29 suites, 660 passed / 0 failed / 7 ignored (D:\rust\p01c-test.log)
- PASS: `cargo test --doc` — 23 passed (D:\rust\p01c-test-doc.log)
- PASS: `cargo doc --no-deps --document-private-items` + `RUSTDOCFLAGS=-D warnings` (D:\rust\p01c-doc.log)
- FAIL (intentional, pre-implementation): 19/23 safety tests vs unfixed extractor (D:\rust\p01a-test.log) — proof the suite detects the bugs

## Risks / regressions
- Extraction I/O doubles for ZIP (validation hashing) — accepted per DEC-007, revisited in P03.
- One-time re-extraction after marker v2 rollout — idempotent.
- Legacy v1 markers remain on disk untouched (harmless orphans).

## Next exact action
Commit P01 as `feat(installer)` (safety layer + extractors + markers + contracts) and `test(installer)` (safety suite) with `agent-work` docs; push; poll CI for conclusion=success. Then P02: read `DownloadPreview` reuse path, define canonical `InstallPlanDocument` fill-in (actions from resolved packages + cache state), stable `plan_id`, `plan` CLI subcommand with JSON output, and apply-side schema/hash verification.
