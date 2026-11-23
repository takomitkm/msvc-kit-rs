# P00 Re-Audit — v2 Revision Points Gap List (2026-11-23)

HEAD: ae7dc84 (v2 workpack). Audit focus: v2 preflight revisions R1–R7.
Each gap is verified by reading the actual code at the specified line numbers.

## Verified gaps

| ID | Rev | Priority | Location | Gap | Fix |
|---|---|---|---|---|---|
| G-1 | R7+R5 | **P0** | `cas.rs` `list_blobs_at()` (line 439): `shard.path().is_dir()` follows symlinks/junctions. A symlink in `blobs/sha256/` pointing outside the cache root would be traversed by GC. | GC can escape cache root via symlink shard | Use `symlink_metadata()` to detect reparse points; skip them |
| G-2 | R7+R5 | **P0** | `cas.rs` `pins()` (line 289): `std::fs::read(...).ok().and_then(from_slice.ok).unwrap_or_default()` — corrupted/unreadable pins.json silently returns empty pins → GC sweeps pinned blobs | GC fail-open on pins corruption | Fail-closed: if pins.json exists but is corrupt, return an error |
| G-3 | 05 rev | **P1** | `transaction.rs` `stage_package()` (lines 403–418): `skipped_via_marker` is a package-level boolean. If ANY payload has a valid marker, ALL staged files are dropped (`if !skipped_via_marker { collect_files(...) }`). In mixed cache state (some payloads marker-skipped, some newly downloaded), the newly extracted files are lost. | Mixed marker/download state loses files | Per-payload skip: collect files only from newly extracted payloads; marker-skipped payloads leave root untouched |
| G-4 | R3 | **P1** | `msi.rs` `MsiexecBackend::extract()` (line 165): no SHA256 verification of the MSI before invoking msiexec. Per R3: "只对来源可信且 SHA256 已验证的官方 payload 使用" | Untrusted MSI can be passed to msiexec /a | Add `expected_sha256` parameter; re-hash before extraction; error on mismatch |
| G-5 | R4 | **P1** | `transaction_tests.rs` (line 71): same-process fault injection does not prove cross-process crash recovery. Process kill has different persistence semantics (open handles, OS-level cleanup). | Same-process test does not prove crash-safety | Helper child process: spawn a helper that begins a transaction to a specified point and is killed; parent verifies recovery |
| G-6 | R7 | **P1** | `extractor.rs` pass-2 write (lines 262/274/277 for VSIX; 498 for CAB): `create_dir_all(parent)` and `File::create(out_path)` do not check for reparse points at intermediate directory levels. A symlink placed in the staging tree redirects writes outside the boundary. | TOCTOU: intermediate symlink redirects write | Before `create_dir_all`/`File::create`, walk each component from the staging root and reject reparse points. Reject archive symlink entries (already done in pass 1 for zip). |
| G-7 | R2 | P2 | `path_safety.rs` `resolve()`: trailing separator (`dir/`) correctly produces `EmptyComponent` error, but callers (VSIX extractor) strip trailing `/` before calling. Need explicit positive/negative tests for directory entries with trailing separators. | Need explicit tests proving the stripping is correct | Add tests: `dir/` stripped by caller → resolves fine; `dir//` (double) → rejected; `dir/./` → rejected |
| G-8 | R6 | P2 | `plan.rs` `create_install_plan` fetches manifest with cache (ETag writes). Per R6: "Plan 可更新独立 Manifest cache；禁止写入 target root、payload Blob、staging、Receipt 和 installed files". Need `--no-cache` that truly bypasses cache. | Plan writes to manifest cache; no opt-out | Add `--no-cache` flag on plan; when set, use a temp dir for manifest fetch |
| G-9 | R1 | P2 | v2 spec pins reference snapshot: msvc-kit `87dc08c`, portablemsvc `92554c09`. Document links to `main` may drift. | Already fixed in v2 spec itself | No code change needed; note in BASELINE.md |

## Already correct (verified, no change needed)

- `extractor.rs` pass 1 already rejects symlink entries via `entry.is_symlink()` (VSIX) — R7 link-entry rejection is done
- `path_safety.rs` already rejects absolute paths, UNC, drive letters, ADS, `..`, `.` and reserved names
- `transaction.rs` already creates per-package staging dirs (spec 05 §6)
- `cas.rs` GC already has pins + retention window + dry-run

## Priority order for fixes

1. G-1 + G-2 (CAS GC symlink + fail-closed pins) — P0
2. G-3 (stage_package mixed marker state) — P1
3. G-4 (MSI SHA256 trust boundary) — P1
4. G-5 (real crash recovery test) — P1
5. G-6 (extraction reparse-point TOCTOU) — P1
6. G-7 (trailing separator tests) — P2
7. G-8 (--no-cache for plan) — P2
