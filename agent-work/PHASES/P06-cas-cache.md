# Phase P06 Report — Content-addressed Blob Cache

## Scope
Per roadmap P06: SHA256-keyed blob store, atomic publish, verification + quarantine, multi-source references, re-runnable migration import, GC mark/sweep with dry-run and pinning. The downloader keeps writing the per-version layout for compatibility; the CAS is the deduplication/integrity layer on top (spec 06 §2.5 migration).

## Changes
- `src/installer/cas.rs` (new public module):
  - `BlobStore::open(root)` — layout `blobs/sha256/<xx>/<digest>`, `partial/`, `quarantine/`, `cache.redb` (source-reference index), `pins.json`. A corrupted redb index is renamed aside and recreated — blobs on disk are never lost to index corruption.
  - `import_file` — hash → publish atomically (copy to `partial/<digest>.part`, fsync via a write-mode handle, rename into place); concurrent publications converge (same digest ⇒ identical bytes); re-runnable (second import reports `AlreadyPresent`, never rewrites). Every import records a `BlobSourceRef` (path/size/timestamp) in redb; same digest from different files accumulates references.
  - `verify_blob` — re-hash; corrupted blobs are renamed to `quarantine/<digest-prefix>-<ts>` and never served (I-007).
  - `BlobPins` + `pin` — active installs/plans/locks pin blobs against GC.
  - `gc(dry_run)` — mark-and-sweep on the blocking pool: blobs with at least one live source reference or an explicit pin are retained; unreferenced blobs younger than `min_age_seconds` are retained; the rest are swept (file removed, then index entry dropped — a crash between the two leaves a tolerable missing-blob entry). Dry run reports without deleting.
- `src/bin/msvc-kit.rs`: new `cache` subcommand — `cache import <files...>`, `cache verify [digests...]`, `cache gc [--dry-run]`; blob root derived from the configured cache directory.
- `src/lib.rs` / `src/installer/mod.rs`: re-exports.

## Tests added (in-module, 6)
- `same_content_imported_twice_yields_one_blob` — one blob per content, both source refs recorded (exit gate: same hash ⇒ one blob).
- `verify_quarantines_corrupted_blobs` — corrupt blob removed + quarantined (exit gate: corrupted cache is never reused).
- `gc_dry_run_changes_nothing_and_respects_pins` — dry run touches nothing; pinned blob never swept; real sweep removes only the orphan; re-run idempotent (exit gates: GC safety + idempotence).
- `gc_never_sweeps_younger_than_retention` — retention window honored.
- `migration_is_re_runnable_without_data_loss` — re-import keeps blob bytes intact.
- `invalid_digests_are_rejected` — digest validation.

## Commands executed
| Command | Exit | Result | Output/log |
|---|---:|---|---|
| `cargo test --all-features --no-fail-fast` (final) | 0 | PASS — 697 passed / 0 failed / 7 ignored | D:\rust\p02c-test.log |
| `cargo fmt --all -- --check` (final) | 0 | PASS | D:\rust\p02c-fmt.log |
| `cargo clippy --all-targets --all-features -- -D warnings` (final) | 0 | PASS | D:\rust\p02c-clippy.log |
| `cargo test --doc` / `cargo doc` -D warnings | 0 | PASS | D:\rust\p02c-test-doc.log, p02c-doc.log |
| Intermediate rounds | — | fixes: redb 4 `create` vs `open` (new stores), `ReadableDatabase` trait import, BTreeSet→HashSet conversion, **Windows `sync_all` on a read-only handle fails with Access Denied** (fixed by opening the partial file in write mode for the fsync), gc test fixtures (orphan source removal models real orphaning), clippy lints | D:\rust\p02c-*.log |

## Compatibility
- CLI: additive — new `cache` subcommand; nothing removed.
- Rust API: additive — `installer::cas` module public; no existing signature changed.
- Layout: the CAS lives beside the existing caches; downloads keep the per-version layout (I-013), so bundle/install/apply flows are untouched.

## Security review
- Digest validation rejects non-hex/wrong-length keys; blob paths are always derived from validated digests (no traversal).
- GC never follows symlinks out of the cache root (walks only `blobs/sha256/` two-level shards); pinned and live-referenced blobs are never swept; dry run deletes nothing.
- Quarantine preserves corrupted bytes for diagnostics instead of deleting them.
- fsync-before-rename + rename publication: no torn blobs; a crash leaves either the partial file (cleaned on next import) or a complete blob.

## Unresolved items
- The downloader still writes the per-version layout; CAS--native downloads (spec 06 §2.4 full flow incl. validator-aware Range resume) are future work — the import path already provides dedup + integrity for existing downloads.
- GC pinning is manual (`pin()`/`pins.json`); wiring automatic pinning into transaction begin/end lands with plan/apply integration (P08+).
- Range resume remains not implemented (same as baseline; documented in RISKS R-104).

## Exit Gate
- [x] 同 hash 只一个 Blob (dedup test)
- [x] corrupted cache 不复用 (quarantine test)
- [x] 并发安全 (atomic publish + convergence; source refs via redb transactions)
- [x] GC 不删活跃内容 (pins + live-source marking)
- [x] 老 cache migration 可反复运行不丢数据 (re-runnable import test)

## Decision
COMPLETE
