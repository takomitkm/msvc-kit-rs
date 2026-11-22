# Phase P02 Report — Canonical InstallPlan / CLI Dry Run

## Scope
Per roadmap P02: unified InstallPlan document and planner, stable plan identity, `plan` CLI with JSON output, plan-file verification and stale detection on `apply`, plan-pinned execution, compatibility of the existing preview/API surface.

## Baseline code findings
- `DownloadPreview` (library-only) already reuses the manifest fetch and selection helpers; the planner reuses exactly those code paths (no duplicate manifest logic).
- `VsManifest::fetch_with_selection` did not expose the manifest identity (URL + official digest) that plans must pin; the fetch internals already verified it.

## Changes
- `src/downloader/manifest.rs`: `ManifestIdentity` (public) + `VsManifest::fetch_with_identity` returning the parsed manifest, the serving channel and the manifest identity (URL, official SHA256, file name); internal `fetch_channel_inner`/`fetch_auto_inner` refactor, existing functions unchanged in behavior. Test call sites updated for the tuple return.
- `src/installer/plan.rs` (new public module):
  - `InstallPlanDocument` (`msvc-kit.install-plan.v1`) with `PlanRequest`/`ResolvedSummary`/`PlanPayload`/`PlanObservations`/`PlanAction`/`DiskUsageEstimate`/`PrivilegeRequirement` — moved from the internal contracts module and made public (the CLI serializes it); `deny_unknown_fields` throughout.
  - `plan_id` = SHA256 over the canonical serialization without `created_at` (struct-declaration field order = stable canonical form); `verify_plan_id` detects any tampering.
  - `create_install_plan(&PlanOptions)`: fetches the manifest once (cache-assisted), selects MSVC/SDK packages with the existing `find_*_packages` helpers, probes the payload cache (existence + size + SHA256), and emits per-payload actions (`download_payload` / `reuse_cached_payload` / `retain_source_media` / `skip_optional_package`) plus per-package `extract_package` actions with dependency links. Payloads without an official SHA256 and conflicting payload identities become blocking conflicts; corrupt cache hits produce a warning and a redownload action. Disk estimate reports measured download/reuse bytes, `null` for staging/backup/final and `bounded` confidence (extract size unknowable from the manifest).
  - `load_and_verify_plan(path)` — schema check + `plan_id` recompute.
  - `revalidate_plan(plan, manifest_cache_dir)` — re-fetches the manifest (cache-assisted) and compares digests, re-hashes every `reuse_cached_payload` target, and compares free space against the download estimate. Returns `PlanRevalidation` with explicit stale reasons.
  - `verify_downloaded_against_plan(component, files, plan)` — name+SHA256 pin check of downloaded files before extraction.
  - Free-space probe via `GetDiskFreeSpaceExW` (Windows; other platforms leave the observation unset).
- `src/installer/contracts.rs`: now holds only the internal receipt/ownership/journal schemas (golden tests retained).
- `src/bin/msvc-kit.rs`: new `plan` subcommand (same selectors as `download`, `--format text|json`, `--output FILE`; JSON to stdout only, status lines to stderr) and new `apply` subcommand: load + verify plan → refuse blocking conflicts → revalidate (stale ⇒ `StaleInstallPlan` report listing reasons) → `--check-only` exits → otherwise execute with the plan's pinned selectors (no re-resolution of latest), verify each downloaded payload against the plan pins, extract, record receipts. Exit code 1 on conflicts/stale.
- `src/vs_channel.rs`: channel selector accepts explicit `http(s)://` URLs (custom channels get a URL-derived cache slug); documented as the escape hatch for unpublished/mirrored channels.
- `Cargo.toml`: `windows-sys` feature `Win32_Storage_FileSystem` for the free-space probe.
- `tests/plan_tests.rs` (new, 7 tests) + `tests/cli_exit_code_tests.rs` (3 CLI smoke tests for plan/apply).

## Tests added
- `plan_is_stable_round_trips_and_never_downloads`: two plans from the same state share `plan_id`; JSON round-trip; tampered plan fails verification; planning downloads nothing (payload route `expect(0)`) and never creates the target root.
- `plan_reports_verified_cache_hits`: verified cache hit → `reuse_cached_payload`, download bytes 0; revalidation passes; corrupting the cache makes revalidation stale with a digest-mismatch reason.
- `plan_downloads_against_corrupted_cache_with_warning`: right-size/wrong-bytes cache → redownload action + warning.
- `plan_marks_conflicting_payload_identities_blocking` / `plan_blocks_payloads_without_official_sha256`: blocking conflicts recorded (and `apply` refuses them before any download).
- `apply_revalidation_detects_changed_manifest`: republished manifest → stale reason "manifest changed".
- `downloaded_payloads_match_the_plan_pins`: the downloader writes exactly where the plan predicted (layout invariant), pin check passes, tampering fails it.
- CLI smoke: `plan --help`, `apply --help` exit 0; `apply` on a missing file exits non-zero.

## Commands executed
| Command | Exit | Result | Output/log |
|---|---:|---|---|
| `cargo test --all-features --no-fail-fast` (round 3) | 0 | PASS — 671 passed / 0 failed / 7 ignored | D:\rust\p02c-test.log |
| `cargo fmt --all -- --check` (round 3) | 0 | PASS | D:\rust\p02c-fmt.log |
| `cargo clippy --all-targets --all-features -- -D warnings` (round 3) | 0 | PASS | D:\rust\p02c-clippy.log |
| `cargo test --doc` (round 3) | 0 | PASS 23 | D:\rust\p02c-test-doc.log |
| `cargo doc --no-deps --document-private-items` + `-D warnings` | 0 | PASS | D:\rust\p02c-doc.log |
| `cargo test --lib a_corrupted_first_transfer` (isolation check) | 0 | PASS 1/1 | ssh transcript |
| Intermediate rounds | 101/1 | fixed: plan module re-exports, `VsChannelSelection` import path, renamed `fetch_auto_inner` call sites (6), tuple destructuring (3), clippy casts, conflict-test fixture (TargetArm64 package never matched the x64 target filter) | D:\rust\p01b-*.log |

## Compatibility
- CLI: additive — `plan` and `apply` are new subcommands; `download` untouched (verified by the untouched download tests).
- Rust API: additive — new public items (plan module, `ManifestIdentity`, `fetch_with_identity`); nothing removed or renamed publicly; `fetch_auto_from` → `fetch_auto_inner` is a private rename.
- JSON/schema: new `msvc-kit.install-plan.v1` schema with golden round-trip tests; lock v1 / receipt v2 untouched.
- Windows behavior: plan/apply use the same download/extract/receipt pipeline as `download` (including the installation lock and P01 safe extractors).

## Security review
- target path: planning writes nothing; apply reuses the P01 safe extractors and the plan pins download paths (verified by the layout-invariance test).
- path traversal: unchanged guarantees; plan payload names go through the same normalization as the downloader.
- ownership: plan records observations only; ownership model remains P03.
- rollback: unchanged (P03).
- privilege: `privilege.requires_elevation=false` with reason "user-writable target"; no elevation logic introduced.

## Unresolved items
- `a_corrupted_first_transfer_is_refetched_and_succeeds` (upstream test, c0b9e0e) failed once under full-suite parallel load and passed in isolation plus the two following full runs — recorded as a load-dependent parallel flake, not caused by P02 (no download-path changes); monitor in later phases.
- Disk estimate staging/backup/final remain `null` until P03/P06 provide staging sizes and extracted-size data.
- `plan --profile`/capability mapping and stable-`plan_id`-across-machines (path-independent identity) defer to P08/P03 respectively; the current identity includes the normalized target root string.

## Exit Gate
- [x] Dry Run does not modify any target directory or cache payload (asserted: target root not created; payload route expect(0))
- [x] Plan JSON round-trips (golden test + round-trip assertion)
- [x] Same state → same plan_id (stability test; created_at excluded from identity)
- [x] CLI old download preview/API compatible (existing preview tests unchanged and green)
- [x] All blocking conflicts block Apply (apply refuses non-empty `blocking_conflicts` before downloads; tested)

## Decision
COMPLETE
