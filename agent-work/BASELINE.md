# Baseline Audit (P00)

- Date/time: 2026-11-22 (audit), local timezone UTC+8
- Repository path (edit copy): `S:\apps\llm\agentwork\msvc-kit-rs`
- Build/test clone (school machine Win11-HV1, user YiZhidi): `D:\msvc-kit-rs`
- HEAD SHA: `1e24a75a42a259995f2e6f6437654a638519cf28`
- Branch: `main` (tracks `origin/main`)
- Remotes: `origin https://github.com/takomitkm/msvc-kit-rs (fetch/push)` — not modified
- Git status at audit start: clean (only `spec/` workpack + `ci.yml` push trigger ahead of upstream `87dc08c`)
- Rust version (build machine): rustc 1.100.0 (0ef1bc386 2026-11-09), cargo 1.100.0 (3acc79c0c 2026-11-05), stable-x86_64-pc-windows-msvc, MSVC 14.44.35207, Windows SDK 10.0.26100 (all on D:\BuildTools)
- OS (build machine): Windows 11 Pro (HV guest), x64. Local edit machine: Windows x64, Git Bash. **No cargo/rustc runs on the local edit machine.**
- Canonical CI command (`.github/workflows/ci.yml` + `justfile`):
  - `vx just fmt-check` → `cargo fmt --all -- --check`
  - `vx just lint` → `cargo clippy --all-targets --all-features -- -D warnings`
  - `vx just test-all-features` → `cargo test --all-features --verbose`
  - `vx just test-doc` → `cargo test --doc`
  - docs job: `vx just doc-ci` → `cargo doc --no-deps --document-private-items` with `RUSTDOCFLAGS="-D warnings"`
  - `ci-success` gate job requires all three jobs (rust-tests, docs, vx-check) green.

## Relevant files and symbols

Status vocabulary per spec 01 §5: `EXISTING_AND_VERIFIED` / `PARTIAL` / `MISSING` / `BROKEN` / `NOT_TESTED` / `UNKNOWN`.

| Feature | Path | Symbol | Existing tests | Status | Evidence |
|---|---|---|---|---|---|
| VS channel manifest fetch/parse | `src/downloader/manifest.rs` | `VsManifest::fetch_with_selection`, `VsManifest` (camelCase serde) | `manifest_integrity_tests` (4 PASS), fixture `tests/fixtures/visualstudio-17-release.channel.json` | EXISTING_AND_VERIFIED (offline) | suite green 2026-11-22; network paths `#[ignore]` in `tests/e2e_tests.rs:431-472` |
| Channel trust / signed catalog | `src/downloader/manifest_trust.rs`, `manifest_trust_windows.rs` | `verify_catalog` | `downloader::cache::tests::signed_catalog_cache_...` (`#[ignore]`, network+cert) | PARTIAL (Windows signature path NOT_TESTED offline) | `src/downloader/cache.rs:452` test marked `requires current Microsoft manifests and online certificate revocation` |
| MSVC/SDK payload download | `src/downloader/common.rs` | `CommonDownloader::download_packages`, `download_file_with_streaming_hash` | `downloader_tests` (50 PASS), `common_tests` (in-lib) | EXISTING_AND_VERIFIED | streaming SHA256, NamedTempFile+`persist` atomic publish, exponential retry (`src/downloader/common.rs:436-546`) |
| Payload path validation (download side) | `src/downloader/common.rs` | `payload_path` | `traversal_and_absolute_payloads_are_rejected` + Windows variants | EXISTING_AND_VERIFIED | rejects `..`, `/abs`, `C:/abs`, empty (`src/downloader/common.rs:274-289`) |
| Download index (redb) | `src/downloader/index.rs` | `DownloadIndex`, `IndexEntry`, `DownloadStatus` | `e2e_tests`, `toolchain_lock_tests` (8 PASS) | EXISTING_AND_VERIFIED | key = `file_name` per per-version dir `index.db`; corrupted DB → rename `.bak` + recreate (`index.rs:81-101`) |
| Hash verification | `src/downloader/hash.rs` + `common.rs` | `compute_file_hash`, sha256 compare | `downloader_tests` | EXISTING_AND_VERIFIED | missing official SHA256 + verify → hard error (`common.rs:379-388`) |
| Manifest cache (ETag/LM + digest) | `src/downloader/cache.rs` | `fetch_with_integrity`, `ManifestCacheMeta` | mockito cache tests (3 PASS in lib) | EXISTING_AND_VERIFIED | cached bytes rehashed; 304 only with validator + digest valid; atomic publish |
| HTTP/TLS config | `src/downloader/http.rs` | `create_http_client` | indirect | EXISTING_AND_VERIFIED | native-tls default, rustls feature opt-in (`Cargo.toml:39`) |
| MSVC component selection | `src/downloader/manifest.rs` | `find_msvc_packages` | `downloader_tests` | PARTIAL | ID substring/prefix matching only (`manifest.rs:477+`); no dependency semantics |
| Host/Target architecture | `src/version/mod.rs` | `Architecture`, `msvc_host_dir/msvc_target_dir` | `version_tests` (42 PASS) | EXISTING_AND_VERIFIED | cross arch exercised in `sdk.rs` selection tests |
| Concurrency adaptation | `src/downloader/common.rs` | batch throughput heuristic | `common_tests` | EXISTING_AND_VERIFIED | constants in `src/constants.rs` (`download` mod) |
| VSIX/ZIP extraction | `src/installer/extractor.rs` | `extract_vsix_sync` | none for path safety | **PARTIAL / FEATURE GAP** | `out_path = target_dir.join(relative_path)` at `extractor.rs:223` — no traversal/absolute/ADS/device-name guard, no entry collision check, no size ratio limit |
| CAB extraction | `src/installer/extractor.rs` | `extract_cab_sync` | none for path safety | **PARTIAL / FEATURE GAP** | `out_path = target_dir.join(name)` at `extractor.rs:473` — same gaps as VSIX |
| MSI extraction (Windows) | `src/installer/extractor.rs` | `extract_msi_sync` | code-explanation unit tests only | PARTIAL | `msiexec /a <msi> /qn TARGETDIR=<dir>` via arg array (`extractor.rs:319-321`); 1618 retry ×5 @2 s; **no `/L*V` verbose log, no fake process-runner tests for retry/timeout paths** |
| MSI extraction (non-Windows) | `src/installer/extractor.rs` | `msiextract` branch | none | NOT_TESTED | `extractor.rs:365-408` |
| SDK MSI handling | `src/installer/mod.rs` | `classify_sdk_payload_for_extraction` | `sdk_extraction_keeps_cabs_and_skips_optional_msi_payloads` (PASS) | EXISTING_AND_VERIFIED | CAB kept as source media; "application verifier"/"winrt intellisense" MSIs skipped (`installer/mod.rs:24,34`) |
| Extraction marker | `src/installer/mod.rs` | `extraction_marker_path` | `extraction_markers_include_source_fingerprint` (PASS) | **PARTIAL (weak)** | fingerprint = SHA256(path+size+mtime), 16 hex chars (`installer/mod.rs:121-143`); marker write failure ignored (`:225 let _`); marker hit skips re-extract **without verifying output tree** |
| Parallel direct-to-target extraction | `src/installer/mod.rs` | `extract_packages_with_progress` | none | **BROKEN (design gap)** | `buffer_unordered(min(cpus,4))` writes straight into shared target root (`installer/mod.rs:208-244`); failure mid-way leaves partial files; no staging, no journal, no rollback |
| Installation lock | `src/storage.rs` | `lock_file` (std `File::lock`), `toolchain_lock::lock_installation` | none for concurrency | PARTIAL (basic) | OS lock released on crash (`storage.rs:10-22`); no owner token/liveness protocol; `.download.lock`, `manifests/*.lock`, `locks/msi-extract.lock` all use it |
| Atomic write | `src/storage.rs` | `atomic_write` | used by receipts/meta | EXISTING_AND_VERIFIED | NamedTempFile + `sync_all` + `persist` |
| Toolchain Lock v1 | `src/toolchain_lock.rs` | `ToolchainLock` (`msvc-kit.toolchain-lock.v1`), `deny_unknown_fields` | `toolchain_lock_tests` (8 PASS) | EXISTING_AND_VERIFIED | strict version/fingerprint/payload validation (`toolchain_lock.rs:228-304`) |
| Installation Receipt v2 | `src/toolchain_lock.rs` | `InstallationReceipt` (`msvc-kit.installation-receipt.v2`), `record_installation` | `toolchain_lock_tests` | EXISTING_AND_VERIFIED (acquisition-only) | rehashes every archive, requires official SHA256 verified (`toolchain_lock.rs:309-384`); **does not attest the extracted tree** |
| Receipts location | `<root>/.msvc-kit/receipts/<component>-<version>-<host>-<arch>.json` | — | — | EXISTING_AND_VERIFIED | `toolchain_lock.rs:91-101` |
| Dry Run / preview | `src/downloader/mod.rs`, `msvc.rs`, `sdk.rs` | `DownloadOptions.dry_run`, `DownloadPreview`, `preview()` | `downloader_tests` | PARTIAL | library-only preview (package count/size); CLI `download` hard-codes `dry_run: false` (`bin/msvc-kit.rs`, Download arm); no disk/privilege/conflict/prune fields |
| VS instance detection | `src/install_into_vs/mod.rs` | `find_vs_instances` (vswhere → registry → common paths) | `cli_exit_code_tests::test_install_into_vs_check_exits_zero` (smoke) | PARTIAL | VS/UBT install-target discovery only; not a toolchain detection framework |
| install-into-vs | `src/install_into_vs/mod.rs` | `install_into_vs`, `can_write_to_vs` | indirect CLI tests | PARTIAL | privileged flow, separately gated |
| doctor | `src/doctor/mod.rs` | `doctor`, `doctor_with_sdk_root` | `doctor_tests` (7 PASS) | EXISTING_AND_VERIFIED | schema `msvc-kit.doctor.v1`; `--compile` path in `execution.rs` |
| query/env/run/cmake | `src/query/mod.rs`, `src/env/`, `src/execution.rs`, CLI | `query_installation`, `get_env_vars`, `run` | `query_tests` (47 PASS), `env_tests` (36 PASS), `execution_tests` (3 PASS) | EXISTING_AND_VERIFIED | JSON schemas stable per docs/guide/toolchain-contracts.md |
| Bundle | `src/bundle/` | `create_bundle`, `BundleLayout` | `bundle_tests` (24 PASS) | EXISTING_AND_VERIFIED | portable bundle layout + scripts |
| Config/portable mode | `src/config/mod.rs` | `load_config`, `enable_portable_mode` | `config_tests` (29 PASS), `config_env_tests` (11 PASS) | EXISTING_AND_VERIFIED | cache root via `directories` ProjectDirs |
| Dependency graph resolution | — | `VsPackage.dependencies: HashMap<String,Value>` parsed (`manifest.rs:87`) but **never read** | none | **MISSING** | no closure/cycle/conflict/topo-sort code anywhere (rg: `dependencies` only in struct def + test literals) |
| InstallPlan / apply / plan CLI | — | — | none | **MISSING** | no `plan`/`apply` subcommands (`Commands` enum: Download, Setup, List, Clean, Config, Env, Query, Doctor, Run, Lock, Cmake, InstallIntoVs, Bundle, Update) |
| OwnershipIndex / InstalledManifest / TransactionJournal | — | — | none | **MISSING** | no such types |
| CAS blob cache + GC | — | index keyed by `file_name` per version dir | none | **MISSING** (redb exists but not content-addressed global store) | `index.rs` |
| Pruning engine | `src/installer/mod.rs` | only optional-MSI skip list | 1 unit test | **MISSING** (no install-tree pruning) | `installer/mod.rs:24` |
| Provider API | `src/downloader/traits.rs` | `ComponentDownloader`, `ComponentType::{Msvc,Sdk}` | `reexports_tests` (14 PASS) | PARTIAL (download-centric only) | no detect/resolve/plan/apply/verify lifecycle |
| Detection framework | `src/install_into_vs/mod.rs` | `find_vs_instances` | smoke | PARTIAL | see VS detection row |
| CLI exit-code contract | `src/bin/msvc-kit.rs` | `Commands` | `cli_exit_code_tests` (20 PASS after P00 fix) | EXISTING_AND_VERIFIED | winget-compatible exit codes |
| Self-update | `src/bin/msvc-kit.rs` (Update arm), `tests/update_tests.rs` | `axoupdater` | `update_tests` (9 PASS after P00 fix) | EXISTING_AND_VERIFIED | help/flag coverage only; network update NOT_TESTED |

## Current install flow (download → extract → receipt)

```text
CLI `download` (src/bin/msvc-kit.rs, Commands::Download arm)
  ├─ ToolchainLock::load (if --lockfile)          → v1 lock strict validation
  ├─ lock_installation(target_dir)               → <root>/.msvc-kit/installation.lock (OS file lock)
  ├─ download_msvc(&DownloadOptions)             → MsvcDownloader::download_impl
  │    ├─ VsManifest::fetch_with_selection       → channel fetch, cache + digest/signature verify
  │    ├─ manifest.find_msvc_packages            → ID substring match (host/target/components)
  │    └─ CommonDownloader::download_packages    → <root>/downloads/msvc/<ver>_<host>_<target>/
  │         ├─ payload_path (validate relative name)
  │         ├─ .download.lock + DownloadIndex (index.db, redb)
  │         └─ per payload: validated cache hit | stream→NamedTempFile→SHA256→persist
  ├─ lock.verify_download (if lockfile)          → rehash archives vs receipt
  ├─ extract_and_finalize_msvc(&mut InstallInfo) → extract_packages_with_progress(files, target_dir)   ← DIRECT to shared root
  │    ├─ marker skip via .msvc-kit-extracted/<name>-<fp16>.done (path+size+mtime fingerprint)
  │    └─ parallel: vsix/zip join(name); msi→msiexec /a /qn TARGETDIR=root; cab join(name)
  ├─ exact-version checks vs request/lock
  ├─ record_installation(...)                    → receipt v2 (rehash + index provenance) at .msvc-kit/receipts/
  └─ same sequence for SDK (downloads/sdk/<build>_<target>/, CABs kept beside MSIs)
```

`extract_and_finalize_sdk` is invoked with `&InstallInfo` (immutable) and CABs are never extracted (kept as msiexec source media).

## Target root layout (actual, shared root)

```text
<install-root>/                          # == DownloadOptions.target_dir; one shared mutable root
├── downloads/                           # download cache (inside the SAME root)
│   ├── msvc/<ver>_<host>_<target>/      # archives + index.db + .download.lock
│   └── sdk/<build>_<target>/            # archives (+ Installers/*.msi, *.cab) + index.db + .download.lock
├── VC/Tools/MSVC/<full-version>/        # extracted MSVC tree
├── Windows Kits/10/...                  # extracted SDK tree
├── .msvc-kit/
│   ├── installation.lock                # OS-level lock during download
│   └── receipts/<comp>-<ver>-<host>-<arch>.json   # receipt v2 (acquisition provenance)
└── .msvc-kit-extracted/                 # weak extraction markers (path+size+mtime hash)
```

Outside the install root:
- Manifest cache: `default_cache_root()/manifests` (`directories::ProjectDirs("com/loonghao/msvc-kit")`, config override `MsvcKitConfig::cache_dir`)
- MSI extraction lock: `<cache-root>/locks/msi-extract.lock` (`extractor.rs:304-306`)

Key structural facts (confirmed by code, matching spec 01 §1 gap list):
1. `target_dir` is simultaneously install root, download cache root, receipt store, marker store.
2. Extraction writes into the live shared root with parallel workers; no staging.
3. Download cache is per version/arch directory keyed by `file_name` — not a global content-addressed store.
4. Receipt proves acquisition (URL+SHA256), not the installed tree.

## Baseline test results (build machine D:\msvc-kit-rs, cargo 1.100.0)

See `TEST-RESULTS.md` for the full command matrix. Summary:
- Initial run: `cargo test --all-features` → exit 101: `cli_exit_code_tests` 3 passed / 17 failed (fail-fast stopped the remaining suites).
- `--no-fail-fast` run: additionally `update_tests` 5 passed / 4 failed. Same root cause in all 21 failures.
- Root cause (proven by experiment): cargo ≥1.100 runs test executables from `target\debug\build\<pkg>\<hash>\out\`, not `target\debug\deps\`; the tests' `get_binary_path()` assumed the deps layout, so `msvc-kit.exe` was "not found" at spawn. Copying `msvc-kit.exe` into the OUT_DIR made the suite pass 20/20.
- GitHub CI run 44968093273 (workflow CI, sha 1e24a75, windows-latest) failed with the **identical 17 failures** — this is an upstream-toolchain-drift baseline failure on main, not machine-specific. The CI Documentation job also failed (`unclosed HTML tag 'version'` in `src/install_into_vs/mod.rs:252` under rustdoc `-D warnings`).
- After the P00 baseline-repair fixes (test helper path search + doc-comment escaping): fmt PASS, clippy `-D warnings` PASS, `cargo test --all-features --no-fail-fast` 621 passed / 0 failed (7 ignored, all network-gated), `cargo test --doc` 23 passed, `cargo doc` with `-D warnings` PASS.

## Unknowns

- Windows-signed catalog fallback (`manifest_trust_windows.rs`) — code exists with unit tests for parts; the full trust chain requires Microsoft manifests + online CRL: NOT_TESTED here (network-gated, matches spec B6).
- `msiexec /a` against real MSIs — no real-MSI fixture in repo (correctly so per spec §10); NOT_RUN (opt-in T5).
- `install_into_vs` mutation paths against a real VS instance — NOT_RUN (privileged, spec B5).
- Whether `vx cargo` (CI wrapper) changes any behavior vs plain cargo — CI logs show identical test behavior to school-machine plain cargo, so treated as equivalent for P00.
