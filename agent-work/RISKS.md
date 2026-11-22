# Risks (P00 audit)

Priorities follow spec 02 §7 (P0 = archive traversal / unknown overwrite / shared-root deletion / half-install / cache bypass; P1; P2).

## P0 — must address first (P01/P03 scope)

| ID | Risk | Location | Spec invariant violated |
|---|---|---|---|
| R-001 | ~~Zip Slip / path escape in VSIX/ZIP extraction~~ **ADDRESSED P01**: all entries resolve through `SafeRelativePath` (Windows semantics on every host); validate-all-then-write with zero-write rejections; symlink entries rejected; size caps enforced. | `src/installer/path_safety.rs`, `src/installer/extractor.rs` | I-001, I-010 — now enforced; residual: NFC/case-fold approximation documented in DEC-006/DEC-007 |
| R-002 | ~~Same gaps in CAB extraction~~ **ADDRESSED P01**: same resolver + occupancy; duplicate CAB entry names rejected outright (reader resolves by name). | `src/installer/extractor.rs` (`extract_cab_sync`) | I-001, I-010 — enforced |
| R-003 | **Parallel extraction writes directly into the live shared root.** Any failure (msiexec 1603, disk full, lock timeout) leaves a partially extracted tree; there is no staging, journal, backup or rollback; a second install can interleave files with a running one beyond the lock's coverage. | `src/installer/mod.rs:146-258` (`extract_packages_with_progress`), called from `extract_and_finalize_msvc/sdk` | I-002, I-006; spec 05 §1 |
| R-004 | ~~Extraction marker is the only skip credential and is weak~~ **MITIGATED P01**: markers are now v2 JSON tied to the source archive SHA256 (schema + digest must match); legacy v1 markers never validate and are upgraded on first re-extraction; marker write failures cost a re-extract, never a false skip. Output-tree verification still pending (P03 InstalledManifest). | `src/installer/mod.rs` (`probe_extraction_marker`, `persist_extraction_marker`) | I-004, I-007 — source-identity half enforced |
| R-005 | **No target-root safety model.** Nothing refuses dangerous targets (drive roots, Program Files, protected dirs), unknown non-empty directories, or unowned overwrites; extraction overwrites existing files via `File::create` without ownership checks. | extractor writers + CLI download flow (no target validation anywhere in `src/bin/msvc-kit.rs` download arm) | I-001, I-002, I-003 |
| R-006 | **Receipt ≠ installed tree.** `record_installation` proves archives only; later mutation of the installed tree is undetectable today (no InstalledManifest). Foundation must arrive with P03/P07. | `src/toolchain_lock.rs:309-384` | I-004 |

## P1

| ID | Risk | Location | Notes |
|---|---|---|---|
| R-101 | MSI wrapper has no verbose log path (`/L*V`), no timeout/cancellation, no fake process-runner coverage of 1603/1619/1620 retry semantics; 1618 retry delay is fixed 2 s. | `src/installer/extractor.rs:14-17,308-363` | spec 08 §8 requires injected-runner tests |
| R-102 | Lock files have no owner token / liveness / recovery protocol (basic OS locks are crash-safe but give no diagnostics or fencing beyond exclusion). | `src/storage.rs:10-22`, `toolchain_lock.rs:45-47` | I-009 partially met (no TTL deletion — good); document and extend in P03 |
| R-103 | Manifest cache meta write failures ignored (`let _ = write_meta(...)` in both success paths). Consequence is bounded (next run re-fetches), but failures are invisible. | `src/downloader/cache.rs:334,370` | observability |
| R-104 | No HTTP Range resume; interrupted large payloads restart from zero (correct, but a P06 requirement input). | `src/downloader/common.rs` | spec 06 §2.4 |
| R-105 | `download_all` runs MSVC and SDK concurrently but each takes its own `.download.lock`; the `msi-extract.lock` serializes msiexec at extraction, which happens **after both downloads** — OK today, but the lock-ordering rules must be written down in P03. | `src/downloader/mod.rs:595-602`, `extractor.rs:304` | deadlock documentation |
| R-106 | Lock v1 `deny_unknown_fields` blocks forward-compat; v2 reader strategy (spec 03 §7) needed when lock grows fields. | `src/toolchain_lock.rs:51-89` | I-013 |
| R-107 | `DownloadPreview` has no disk/privilege/conflict/prune sections and no plan identity — cannot serve as the P02 InstallPlan without extension (additive). | `src/downloader/mod.rs:502-541` | I-005 |
| R-108 | `payload_path` normalization replaces `/` with `\` but does not reject Windows reserved names or trailing dot/space (download-side filenames come from Microsoft manifests; extractor-side must still handle untrusted names). | `src/downloader/common.rs:274-289` | I-010 defense-in-depth |

## P2

| ID | Risk | Location |
|---|---|---|
| R-201 | `get_extractor` uses `block_on` inside a sync fn (runtime hazard if called on a runtime thread). | `src/installer/extractor.rs:516-527` |
| R-202 | `extract_cab_sync` re-opens the CAB per file (O(n) I/O) — performance only. | `src/installer/extractor.rs:467-486` |
| R-203 | Compression-bomb / total-size caps absent (zip entry sizes trusted). | extractor paths |
| R-204 | Progress bars on stderr + println! on stdout are mixed in CLI download path; plan/apply must formalize the stream split. | `src/bin/msvc-kit.rs` |
| R-205 | Documentation drift risk: two duplicated `get_binary_path` helpers (tests) — consolidated behavior must be kept in sync (P00 fix applied to both). | `tests/cli_exit_code_tests.rs`, `tests/update_tests.rs` |

## Environment / infrastructure risks

| ID | Risk | Evidence |
|---|---|---|
| E-001 | GitHub Actions windows-latest and the school machine both run rust 1.100.x; cargo's test-executable placement changed (build/<hash>/out). Any test that derives sibling paths from `current_exe()` can break again on toolchain updates. Mitigation applied in P00 (upward search). | CI run 44968093273 log vs school-machine p00b log (identical failures) |
| E-002 | rustdoc now errors on bare `<version>` in doc comments under `-D warnings`; other angle-bracket placeholders exist but are already inside code spans. Watch for future rustdoc lint additions. | CI docs job failure |
| E-003 | Fork CI is the acceptance gate; "Code Coverage" workflow fails only at the codecov upload step (no token in fork) — its test step is the signal (per workpack instruction). | coverage.yml |
