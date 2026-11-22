# Phase P03 Report — File-level Staging / Transaction / Recovery

## Scope
Per roadmap P03: per-transaction workspace, per-package staging, package output manifests, merge-time collision detection, root lock protocol, ownership index, write-ahead journal, file-level backup/commit/rollback, recovery, marker migration, and the guarantee that the shared root is never replaced.

## Baseline code findings
- Extraction wrote straight into the shared root with parallel workers (R-003); `.msvc-kit/installation.lock` covered concurrency but not failure recovery.
- Receipt v2 proved acquisition only; no ownership model existed (R-005/R-006).

## Changes
- `src/installer/transaction.rs` (new public module):
  - `Transaction::begin` creates `.msvc-kit/transactions/<tx-id>/` with `journal.jsonl`, per-package `packages/` staging and a `backup/` directory; refuses to start while any transaction carries a `needs-recovery` marker; callers hold the existing install-root lock (documented lock protocol — one lock covers plan revalidation, staging, commit, rollback and receipt updates).
  - `stage_package` extracts each package into its own staging directory through the P01 safe extractors (marker v2 skips still honored: skipped packages stage nothing and the root is untouched).
  - `merge` builds a deterministic, case-insensitively ordered file list; cross-package collisions (same path, different bytes) are rejected before any commit; identical bytes merge owners.
  - `commit` runs a pre-pass that classifies every file as Create / Skip (identical bytes) / ReplaceOwned (previous owner recorded, or legacy msvc-kit root — receipts/markers — with backup) / UnownedConflict (refused before any mutation, I-001), then executes with journal-protected mutations: `will_backup` → backup → `backup_done` → `will_install` → install + re-hash → `install_done`, then an ownership-index backup + atomic write, the installed manifest (`msvc-kit.installed-manifest.v1`), the `committed` journal event (point of no return), and backup cleanup.
  - `rollback` is per-path from the journal: created files are removed only when they still match the installed digest; replaced files are restored from backup (hash-gated — a third-party write after the failure becomes a rollback conflict → needs_recovery); the ownership index and the transaction's installed manifest are reverted.
  - `recover_pending` (idempotent, called at the start of both extract-and-finalize paths): no-journal or no-executed-action transactions are staging-cleaned; executed transactions are rolled back through the journal; committed transactions only have backups cleaned; needs-recovery transactions are reported and never touched.
- `src/installer/mod.rs`: `extract_and_finalize_msvc` and `extract_and_finalize_sdk` now run through transactions (recovery → stage → merge → commit; on failure they roll back explicitly, and a fault after the Committed marker is treated as a durable install). The public signatures are unchanged.
- `src/installer/contracts.rs`: added `InstalledManifest`/`InstalledFileEntry` (`msvc-kit.installed-manifest.v1`) with golden tests.
- `tests/transaction_tests.rs` (new, 9 tests): the spec 08 §10 fault-injection matrix plus the P03 exit-gate scenarios.

## Tests added
- `commit_fresh_root_writes_files_ownership_and_manifest` — files, ownership index (lowercase-canonical paths), installed manifest, Committed event, backup cleanup.
- `reinstall_identical_files_skips_writes_and_adds_owner` — zero writes on identical re-install; owners appended.
- `replace_owned_file_backs_up_then_installs` — legacy adoption path with backup; journal shows will_backup/backup_done/install_done/committed.
- `unowned_existing_files_are_never_overwritten` — foreign file at the target path → refusal, root byte-identical.
- `two_versions_coexist_in_one_root` — disjoint version trees stay intact.
- `fault_injection_matrix_recovers_to_safe_states` — 9 pre-commit fault points (workspace, journal header, staged, merge, before/after install, ownership write, manifest write, committed marker) each abort the commit; recovery leaves the root without the installed file and is idempotent; 2 post-commit points (after committed marker, before cleanup) keep the durable install and only acknowledge the journal.
- `backup_fault_points_roll_back_to_original_bytes` — BeforeBackup/AfterBackup in a replace scenario restore the original bytes exactly.
- `needs_recovery_blocks_new_transactions_until_resolved` — begin refuses, recovery reports without touching, explicit marker removal unblocks.
- `explicit_rollback_restores_original_bytes_and_marks_journal` — the wiring path: commit failure → rollback → original bytes + rolled_back journal record.
- Bug-detection proof: the fault matrix caught a real data-loss bug before it shipped — the journal originally recorded lowercase paths while backup names used original case, so `find_backup` never matched and recovery deleted the backup (the only copy of the original bytes) along with the transaction directory. The journal now records original-case paths and backup names join with a double underscore so the suffix search always matches.

## Commands executed
| Command | Exit | Result | Output/log |
|---|---:|---|---|
| `cargo test --all-features --test transaction_tests` (final round) | 0 | PASS 9/9 | D:\rust\p03a-test.log |
| `cargo fmt --all -- --check` (final round) | 0 | PASS | D:\rust\p02c-fmt.log |
| `cargo clippy --all-targets --all-features -- -D warnings` (final round) | 0 | PASS | D:\rust\p02c-clippy.log |
| `cargo test --all-features --no-fail-fast` (final round) | 0 | PASS — 681 passed / 0 failed / 7 ignored | D:\rust\p02c-test.log |
| `cargo test --doc` (final round) | 0 | PASS 23 | D:\rust\p02c-test-doc.log |
| `cargo doc` + `-D warnings` (final round) | 0 | PASS | D:\rust\p02c-doc.log |
| Intermediate rounds | — | compile/behavior fixes: async rollback chain, journal-path construction, `old_hash` journaling, backup-name separator, per-path rollback rewrite, clippy lints | D:\rust\p03a-test.log, p02c-clippy.log |

## Compatibility
- CLI: `download` and `apply` behavior unchanged from the caller's perspective (same outputs; extraction now transactional internally). New `.msvc-kit/transactions/`, `.msvc-kit/installed-manifests/`, `.msvc-kit/ownership.json` artifacts appear in managed roots.
- Rust API: additive — `installer::transaction` module is public; `extract_and_finalize_*` signatures unchanged.
- JSON/schema: new internal schemas (`installed-manifest.v1` golden-tested); journal is JSONL with `type: header|event` lines.
- Windows behavior: legacy roots (receipts/markers present, no ownership.json) are adopted with backup instead of refused — one-time migration recorded in the journal; unknown files are never deleted.

## Security review
- target path: staging + commit operate on P01-validated relative paths only; package IDs are sanitized into safe staging directory names.
- path traversal: no new paths accepted from archives beyond P01's resolver.
- ownership: pre-pass refuses unowned overwrites before any mutation; ownership index is hash-gated (owner added only when bytes match).
- rollback: per-path hash-gated restores; third-party modifications surface as rollback conflicts (needs_recovery) instead of being overwritten; post-Committed faults never roll back a durable install.
- privilege: unchanged; no elevation paths.

## Unresolved items
- Merge happens in memory per transaction; `merged-tree/` on disk (spec 05 §6 layout) is not materialized separately — staging directories already provide isolation for the current single-package-per-component flow (MSVC, SDK). Revisit if multi-package merges need a unified staging tree.
- Pruning integration (P05) will add `prune` events to the same journal.
- CAS/GC interplay: transaction staging directories must be pinned by the P06 GC — recorded as a P06 requirement.
- The old `install_msvc`/`install_sdk` legacy functions and `extract_packages_with_progress` remain for API compatibility and are no longer used by the CLI path.

## Exit Gate
- [x] Unknown non-empty root never cleared (only planned files are touched; unowned conflicts refuse before mutation)
- [x] Multiple versions coexist in one root (test)
- [x] Every fault point: recovery succeeds or the transaction lands in needs_recovery (11-point matrix, idempotent)
- [x] Failed transactions do not change non-ownership files (unowned refusal + hash-gated restores)
- [x] Owner state correct after success (ownership index + installed manifest + skip/adopt/replace semantics)

## Decision
COMPLETE
