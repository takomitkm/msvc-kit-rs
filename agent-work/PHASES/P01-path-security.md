# Phase P01 Report — Data contracts + Path Security

## Scope
Per roadmap P01: unified safe path resolution for all extractors, Windows path-semantics test suite, archive entry collision handling, minimal internal schemas for InstallPlan/InstallReceipt/OwnershipIndex/TransactionJournal with golden tests, extraction marker hardened to content identity (v2) with legacy handling, public API compatibility review.

## Baseline code findings (pre-P01, from P00)
- `src/installer/extractor.rs:223` (VSIX) and `:473` (CAB): `target_dir.join(relative)` with no guards — R-001/R-002.
- `src/installer/mod.rs:121-143,180-226`: markers keyed on path+size+mtime, write failures ignored, marker hits never verified — R-004.

## Changes
- `src/installer/path_safety.rs` (new, pub(crate)): `SafeRelativePath::resolve` — Windows-semantics validation of every untrusted entry name (traversal, POSIX absolute, drive-absolute/relative, UNC, verbatim/device prefixes, ADS, `.`/`..`, empty components, reserved device names incl. `NUL.txt` form, trailing dot/space); `collision_key()` = NFC + case-fold; `PathOccupancy` for file/directory conflict detection; `MAX_ENTRY_BYTES` (8 GiB) / `MAX_ARCHIVE_OUTPUT_BYTES` (64 GiB) caps. Unit tests included.
- `src/installer/extractor.rs`: VSIX/ZIP and CAB extraction rewritten as **validate-all-then-write**. Pass 1 resolves every name, rejects symlink entries, enforces size caps, detects file/dir conflicts, hashes every entry (ZIP) and resolves duplicate collision keys (equal bytes → deterministic skip; different bytes → hard reject) before any write. Pass 2 writes only surviving entries. CAB additionally rejects duplicate entry names (the reader resolves by name). Zero-write guarantee on any rejection.
- `src/installer/mod.rs`: extraction markers upgraded to `msvc-kit.extraction-marker.v2` — JSON marker carrying the source archive SHA256; skip requires schema + digest match; legacy v1 markers (`<name>-<fp16>.done`, body `ok`) never validate and are left untouched while re-extraction writes the v2 marker. Marker writes now go through `storage::atomic_write` (failure = re-extract next run, never a false skip).
- `src/installer/contracts.rs` (new, pub(crate), `#![allow(dead_code)]` until P02 consumers land): `InstallPlanDocument` (`msvc-kit.install-plan.v1`), `InstallReceiptDocument` (`msvc-kit.install-receipt.v1`), `OwnershipIndexDocument` (`msvc-kit.ownership.v1`), `TransactionJournalDocument` (`msvc-kit.transaction-journal.v1`) with `PlanActionKind`, `TransactionState`, `JournalEventKind` enums per spec 02 §6 / 03 §3. All `deny_unknown_fields`.
- `Cargo.toml`/`Cargo.lock`: added `unicode-normalization = "0.1"` (see DEC-006).
- `tests/installer_safety_tests.rs` (new): 24 integration tests implementing the spec 08 §3 matrix for both extractors.

## Tests added
- `tests/installer_safety_tests.rs`: traversal (4 shapes), POSIX/drive/UNC/verbatim absolute, ADS, reserved names, trailing dot/space, dot component + double separator, case-fold duplicate conflict, NFC/NFD duplicate conflict, file/dir conflicts (both orders), symlink rejection, benign Unicode/spaces/deep paths, duplicate-name rejection (CAB), empty-dir policy, `Contents/` prefix preservation, marker v2 content identity, legacy marker upgrade. Each rejection asserts error AND byte-identical target tree AND absent escape files.
- Bug-detection proof (spec B4 step 5): round 1 ran this suite against the unfixed extractor — 19 of 23 failed for exactly the intended reasons (traversal escaped, ADS created, legacy marker skipped extraction, collisions overwrote). Log: D:\rust\p01a-test.log.
- `src/installer/path_safety.rs` unit tests: benign/absolutely-every-rejection-shape matrix, collision key folding (case + NFC/NFD), occupancy conflict ordering, relative path construction.
- `src/installer/contracts.rs` golden tests: golden JSON round-trip for all four documents, unknown-field rejection, unknown action kind rejection, wrong schema rejection, full `TransactionState` round-trip.

## Commands executed
| Command | Exit | Result | Output/log |
|---|---:|---|---|
| `cargo test --all-features --test installer_safety_tests` (round 1, pre-implementation) | 101 | FAIL 4/19 — exposed the bugs | D:\rust\p01a-test.log |
| `cargo test --all-features --test installer_safety_tests` (round 2, post-implementation) | 0 | PASS 24/24 | D:\rust\p01b-safety.log (23 then 24 after empty-dir test) |
| `cargo fmt --all -- --check` (round 3) | 0 | PASS | D:\rust\p01c-fmt.log |
| `cargo clippy --all-targets --all-features -- -D warnings` (round 3) | 0 | PASS | D:\rust\p01c-clippy.log |
| `cargo test --all-features --no-fail-fast` (round 3) | 0 | PASS — 29 suites, 660 passed / 7 ignored / 0 failed | D:\rust\p01c-test.log |
| `cargo test --doc` (round 3) | 0 | PASS 23 | D:\rust\p01c-test-doc.log |
| `cargo doc --no-deps --document-private-items` + `-D warnings` | 0 | PASS | D:\rust\p01c-doc.log |

Intermediate rounds caught and fixed: `unicode-normalization` version (0.2 does not exist → 0.1), `Copy` on a String-bearing struct, clippy dead_code on the contracts module (expected pre-P02, silenced with documented allow), clippy `cloned_ref_to_slice_refs`, missing empty-dir coverage, zip fixture limitation (`unix_permissions` drops file-type bits → symlink fixture uses `add_symlink`).

## Compatibility
- CLI: unchanged.
- Rust API: unchanged — all new items are `pub(crate)`; `extract_vsix`/`extract_cab`/`extract_msi`/`extract_packages_with_progress` signatures unchanged. `MsvcKitError` untouched (safety errors are surfaced as contextual `Other` messages; public error model lands with P02/P03 per spec 03 §6).
- JSON/schema: four new internal schemas introduced (no existing schema changed). Lock v1 / Receipt v2 untouched.
- Windows behavior: extraction markers now re-extract once after upgrade (v2 rollout) — one-time idempotent cost, recorded as a behavioral note; extraction now hashes each ZIP entry twice (validate pass + write pass) and re-opens CABs per entry during validation — accepted I/O cost for the zero-write guarantee.

## Security review
- target path: all archive output paths now pass `SafeRelativePath` (Windows semantics on every host).
- path traversal: covered by 9 dedicated rejection tests + resolver unit matrix; zero-write assertions prove no outside mutation.
- ownership: `PathOccupancy` enforces in-archive file/dir conflict rules; cross-install ownership stays P03.
- rollback: N/A this phase (staging/transaction is P03); note that extraction still writes into the live root for surviving benign archives.
- privilege: unchanged; no elevation paths touched; no `/i` introduced.

## Unresolved items
- CAB pass 1 re-opens the cabinet per entry for content hashing of duplicates... (not implemented: CAB rejects duplicate names outright instead; per-entry hashing for CAB is deferred with the MSI-native work).
- Public error model for path-safety failures still `Other`-wrapped (spec 03 §6 variants arrive with the plan/transaction work).
- Unicode normalization: collision keys use NFC + `to_lowercase`; Windows-specific case tables (e.g. Turkish İ) are approximated — documented limitation.

## Exit Gate
- [x] Path traversal tests all pass (23 + 24 integration cases, spec 08 §3 matrix)
- [x] Zero changes outside staging on safety rejection (asserted per test)
- [x] Existing benign fixture extractor tests all pass (full suite 660/0 green)
- [x] Schema golden tests (4 documents, round-trip + rejection cases)

## Decision
COMPLETE
