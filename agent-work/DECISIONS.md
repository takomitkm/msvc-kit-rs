# Engineering Decisions

## DEC-001: Repair the CLI-binary lookup in CLI-spawning tests (baseline repair, not feature work)
- Date: 2026-11-22
- Phase: P00
- Context: `cargo test --all-features` fails on rust/cargo 1.100 (both school machine and GitHub windows-latest, CI run 44968093273 at 1e24a75): 17 tests in `tests/cli_exit_code_tests.rs` and 4 in `tests/update_tests.rs` panic with `Os { code: 2, NotFound }` when spawning `msvc-kit.exe`. Root cause proven by experiment: cargo ≥1.100 places test executables under `target\debug\build\<pkg>\<hash>\out\` instead of `target\debug\deps\`, and `get_binary_path()` derived the binary location from `current_exe()` assuming the old deps layout. Copying the binary into the OUT_DIR flipped the suite to 20/20 PASS.
- Options considered: (a) leave failures documented as baseline red — but the per-stage gate requires CI workflow conclusion=success for each push, and main is red before any agent change; (b) pin/downgrade toolchain — rejected, forks runners use latest stable; (c) make the helper search upward from the test exe until it finds `msvc-kit.exe`, falling back to the classic sibling path; (d) rewrite tests to use `CARGO_BIN_EXE_msvc-kit` — larger churn, changes failure modes for all 29 tests, unnecessary.
- Decision: option (c), applied to both duplicated helpers (`tests/cli_exit_code_tests.rs`, `tests/update_tests.rs`).
- Why: restores the green baseline required by the P00 push gate; no assertion or test case is removed or weakened; behavior on older cargo (deps layout) is unchanged (search finds the binary at the same place, first candidate checked is the old target).
- Trade-offs: duplicated helper logic remains in two files (pre-existing duplication, tracked as R-205); if a future cargo moves the binary somewhere unreachable, the fallback still fails loudly with NotFound.
- Evidence/tests: TEST-RESULTS P00-T07 (baseline FAIL), P00-T08 (OUT_DIR experiment), P00-T09 (CI parity), P00-T13 (621 passed / 0 failed after fix).
- Revisit condition: if cargo changes artifact placement again, or if a test suite gains a second binary target.

## DEC-002: Escape `<version>` placeholder in `install_into_vs` doc comment
- Date: 2026-11-22
- Phase: P00
- Context: CI Documentation job (rustdoc with `RUSTDOCFLAGS=-D warnings`) fails on rust 1.100: `unclosed HTML tag 'version'` at `src/install_into_vs/mod.rs:252`. Pre-existing source text + newer rustdoc lint.
- Options considered: (a) leave red; (b) backtick the placeholder (doc-comment only, no code change); (c) reflow the sentence.
- Decision: (b) — `` `<source_dir>/VC/Tools/MSVC/<version>/` ``, matching the already-backticked style at line 314.
- Why: smallest change that restores the CI docs gate; content unchanged.
- Trade-offs: none.
- Evidence/tests: TEST-RESULTS P00-T10 (CI failure), P00-T12 (local `cargo doc` with `-D warnings` PASS).
- Revisit condition: future rustdoc lint additions flagging other placeholders (line 314/315 already inside code spans).

## DEC-003: All compile/test/lint execution on the school machine; local machine is edit-only
- Date: 2026-11-22
- Phase: P00
- Context: Workpack instruction §2 forbids cargo/rustc/clippy/fmt on the local Windows PC; the school machine clone `D:\msvc-kit-rs` is build-only (never produces commits).
- Options considered: n/a (user instruction).
- Decision: long cargo runs go through `schtasks` jobs (`D:\rust\<script>.cmd` scripts with full MSVC/proxy env, per-step logs, `EXITCODE`/step markers in `D:\rust\p00*-status.log`); short interactive commands via ssh + `source /d/rust/msvc-env.sh` equivalents; dirty-tree changes synced as patch files (`git apply`), reverted after each verified run.
- Why: follows the workpack execution protocol; schtasks avoids the known ssh long-run hang.
- Trade-offs: extra sync round-trip per change (patch → apply → run → revert).
- Evidence/tests: TEST-RESULTS P00-T01..T15 all reference school-machine logs.
- Revisit condition: if the school machine becomes unreachable (stop-and-report condition).

## DEC-004: "push per stage" overrides the master prompt's no-commit/no-push rule (B1)
- Date: 2026-11-22
- Phase: P00
- Context: Master prompt B1 forbids commits/pushes; the user's 2026-11-22 workpack instruction §1 explicitly overrides exactly that: after a stage's Exit Gate passes, commit and push, then verify the CI workflow on the pushed sha is green. All other B1 protections remain (no force-push, no remote changes, no PR/issue/release, no hard reset/clean of user files).
- Decision: per-stage commits with conventional prefixes and explicit paths only (no `git add -A`); push with `env -u GITHUB_TOKEN git -c http.proxy=http://127.0.0.1:7897 push origin main`; CI checked via gh api after each push.
- Why: direct user instruction recorded verbatim in the workpack.
- Trade-offs: none beyond what the instruction already specifies.
- Evidence/tests: each stage's phase report records the pushed sha + CI conclusion.
- Revisit condition: only if the user revokes the override.

## DEC-005: P00 treats the two baseline repairs as in-scope despite "don't modify core code"
- Date: 2026-11-22
- Phase: P00
- Context: Master prompt P00 says "不改核心代码" (no core code changes) while requiring the baseline test record; the workpack requires CI=success per pushed stage. main was already red at HEAD for toolchain-drift reasons (DEC-001/DEC-002).
- Options considered: (a) push docs-only and accept red CI → stage gate unfulfillable; (b) defer fixes to P01 → same red-CI problem for the P00 push; (c) repair now as two minimal, behavior-preserving commits (`fix(tests)`, `docs:`).
- Decision: (c). Neither fix touches runtime behavior: one is test infrastructure, one is a doc comment.
- Why: satisfies both constraints simultaneously and is the minimal intervention that unblocks the staged plan.
- Trade-offs: P00's diff is not literally zero; the phase report lists both commits explicitly.
- Evidence/tests: see DEC-001/DEC-002 evidence.
- Revisit condition: user objection to including repairs in P00.

## DEC-006: Add `unicode-normalization 0.1` dependency for archive collision keys
- Date: 2026-11-22
- Phase: P01
- Context: Spec 08 §3 requires rejecting paths that collide after Unicode normalization; Windows does not normalize file names, but two entries spelling the same name in NFC and NFD plus differing bytes must not silently become one file. Implementing NFC without tables is not credible.
- Dependency review (spec 10 §6): `unicode-normalization` 0.1.26 (latest is 0.1.x; "0.2" does not exist on crates.io); purpose: NFC normalization in `SafeRelativePath::collision_key`; existing dependency could not do it (serde/toml/etc. have no normalization); license MIT/Apache-2.0; MSRV well below 1.100; actively maintained (used by idna/regex ecosystem); no default features, pure std, adds ~two table modules (~200 KB source, no transitive deps); compiles on Windows (verified) and Linux (std-only); parser risk none (no parsing of untrusted input beyond Unicode tables); testable with real fixtures (done — NFC/NFD collision test).
- Decision: add the dependency; collision key = ` NFC(name).to_lowercase()`.
- Why: the only credible way to satisfy the spec matrix without hand-rolled Unicode.
- Trade-offs: one more dependency in the lockfile; NFC approximation of Windows upcase tables (documented limitation — full Windows case folding is not implemented by any mainstream Rust crate either).
- Evidence/tests: `cargo test` green incl. `zip_unicode_normalization_duplicate_paths_with_different_bytes_are_rejected`; Cargo.lock committed.
- Revisit condition: if MSRV/size pressure appears, or Windows ships normalization-aware APIs.

## DEC-007: Safety rejection policy — validate-all-then-write, content collisions resolved in the validation pass
- Date: 2026-11-22
- Phase: P01
- Context: Spec P01 exit gate demands "安全拒绝时 staging 外零变更". With no staging yet (P03), extraction still writes into the live root, so the only way to honor the gate is to fully validate an archive (names, symlink modes, size caps, file/dir conflicts, duplicate-key content identity) before the first write. The zip reader already refuses byte-identical duplicate names at open; case-fold/NFC duplicates must therefore be compared by content.
- Options considered: (a) detect collisions mid-write (rejection after partial writes); (b) hash every ZIP entry during validation and decide dedupe/conflict upfront (archive read twice); (c) defer duplicate handling to P03 transactions.
- Decision: (b) for ZIP — pass 1 hashes every entry (streaming SHA256) and keeps key→(first path, hash); equal bytes skip the duplicate, differing bytes reject the whole archive before any write; pass 2 writes only survivors. CAB rejects duplicate names outright (the cab crate resolves entries by name, so duplicates are unreadable unambiguously). The per-entry hashes computed in pass 1 are exactly what P03's InstalledManifest needs next.
- Why: strongest guarantee available pre-transaction; the extra read pass is I/O on archives that must be decompressed once more anyway for extraction.
- Trade-offs: double decompression I/O for ZIP extraction; CAB duplicate policy is "reject" rather than "dedupe" (documented in tests).
- Evidence/tests: all rejection tests assert byte-identical target trees; full suite green (D:\rust\p01c-test.log).
- Revisit condition: measured extraction overhead on real payloads justifies a staged single-pass design in P03.

## DEC-008: Extraction marker v2 with legacy v1 ignored-and-upgraded
- Date: 2026-11-22
- Phase: P01
- Context: Spec P01: "现有 marker 不再作为唯一提取完成凭据；旧 marker 要安全地 invalid/legacy handle". v1 markers (path+size+mtime fingerprint, body `ok`) prove nothing about source content and their write failures were silently ignored.
- Options considered: (a) keep v1 names but verify output tree on hit (expensive, still P03 territory); (b) v2 JSON markers keyed by source SHA256; v1 markers never match the new name pattern, so the first post-upgrade run re-extracts and upgrades.
- Decision: (b). Marker name `<name>-v2-<sha256(name|sha)[:16]>.done`, body = JSON `{schema, source_sha256, source_name}`; skip requires schema + digest match; writes via `storage::atomic_write`; write failure logs a warning and only costs a re-extraction (never a false skip).
- Why: makes the skip decision depend on source content identity; legacy markers are handled safely (ignored, preserved on disk).
- Trade-offs: one full-hash pass per archive per install run to compute the marker name (accepted; can be cached in P06); one-time re-extraction after upgrade.
- Evidence/tests: `extraction_markers_record_source_sha256`, `legacy_ok_markers_are_ignored_and_upgraded`.
- Revisit condition: P03 replaces markers with receipts/InstalledManifest; marker code then becomes a compatibility shim.

## DEC-009: Data contracts stay `pub(crate)` with module-level dead_code allow
- Date: 2026-11-22
- Phase: P01
- Context: Spec P01 asks for minimal schemas "可先内部" (may be internal first). Publishing them now would freeze shapes before P02/P03 wire them to real behavior, and `MsvcKitError` variants for the new failure classes (spec 03 §6) are also deferred to avoid breaking external exhaustive matches.
- Decision: `src/installer/contracts.rs` is `pub(crate)` with `#![allow(dead_code)]` and a comment stating the allow expires when P02 consumers land; path-safety errors surface as contextual `MsvcKitError::Other` messages for now.
- Why: additive-only public API review passes (zero public surface change in P01).
- Trade-offs: a temporary allow lint; revisited at P02.
- Evidence/tests: clippy `-D warnings` green; golden tests cover the full shape.
- Revisit condition: P02 install-plan CLI lands.

## DEC-010: Install plan document becomes public API; internal schemas stay pub(crate)
- Date: 2026-11-22
- Phase: P02
- Context: The CLI (separate binary crate) can only see public library items, and the plan/apply commands must serialize the plan. The receipt/ownership/journal schemas still have no consumers (P03).
- Decision: move InstallPlanDocument and its companion types from `installer::contracts` into a new public `installer::plan` module; keep the other three documents internal. `MsvcKitError` still unchanged (stale/conflict surfaced as contextual `Other` errors).
- Why: additive public surface; avoids freezing the transaction-era schemas early.
- Trade-offs: two modules for related documents until P03 merges them if needed.
- Evidence/tests: golden round-trip tests in both modules; clippy -D warnings green.
- Revisit condition: P03 transaction work lands.

## DEC-011: Apply semantics for P02 - verify, revalidate, then execute through the existing pipeline pinned by the plan
- Date: 2026-11-22
- Phase: P02
- Context: Roadmap P02 requires apply-side plan verification and stale detection, but the transactional installer is P03. The existing download pipeline (lock + downloader + P01 safe extractors + receipt recording) is the only execution path today.
- Decision: apply = load/verify plan -> refuse blocking conflicts -> revalidate_plan (manifest digest, cached payload digests, free space) -> on stale fail with a StaleInstallPlan report -> otherwise execute with the plan's pinned selectors (never re-resolving latest), cross-check every downloaded payload against the plan pins before extraction, then extract and record receipts exactly like download.
- Why: satisfies the P02 exit gate now; the P03 transaction manager takes over the execution stage without changing the verification protocol.
- Trade-offs: apply is not yet transactional (half-installs on failure remain possible until P03 - risk R-003 unchanged); the pin check closes the silently-switched-packages gap.
- Evidence/tests: plan_tests stale/conflict/pin tests; full suite green.
- Revisit condition: P03 replaces execution internals.

## DEC-012: Channel selector accepts explicit http(s) URLs with URL-derived cache slugs
- Date: 2026-11-22
- Phase: P02
- Context: Plan/apply tests (and users targeting unpublished or mirrored channels) need to pin a channel by URL; the selector previously accepted only registry majors/years/auto.
- Decision: parse_channel_selector returns a custom VsChannelSpec (major 0) for http(s) selectors; cache_slug derives a stable 12-hex suffix from the URL so custom channels cannot collide on cache files. The manifest trust chain (catalog digest vs. fresh channel bytes) applies unchanged.
- Why: minimal escape hatch, additive, no behavior change for existing selectors.
- Trade-offs: users can point at arbitrary servers; trust still comes from the fresh-channel digest + payload SHA256s, and HTTP is accepted for test fixtures only in practice.
- Evidence/tests: plan tests all use the URL selector; vs_channel unit coverage.
- Revisit condition: if a provider-API channel registry (P10) supersedes URL pinning.

## DEC-013: Transaction design - per-package staging, in-memory merge, journal-protected file commit
- Date: 2026-11-22
- Phase: P03
- Context: Spec 05 section 6 suggests a merged-tree directory; the current flow extracts one package per component (MSVC, SDK) per transaction, and spec I-002 forbids replacing the shared root.
- Decision: extract each package into packages/<safe-id>/ staging, merge in memory (deterministic, collision-rejecting), then commit file-by-file with write-ahead journal events (will_backup/backup_done/will_install/install_done) and an explicit committed event as the point of no return. The lock protocol is documented as: the caller holds .msvc-kit/installation.lock for the whole transaction lifecycle; no separate lock hierarchy.
- Why: file-level commit satisfies I-001/I-002/I-006 without a generation-directory migration; in-memory merge is sufficient for one package per component today.
- Trade-offs: no materialized merged-tree dir (revisit for multi-package merges); journal appends are fsynced per event (I/O cost acceptable at this granularity).
- Evidence/tests: transaction_tests 9/9; full suite green.
- Revisit condition: multi-package transactions, pruning integration (P05), or CAS pinning (P06).

## DEC-014: Legacy root adoption with backup instead of hard refusal
- Date: 2026-11-22
- Phase: P03
- Context: Spec 05 section 7 refuses unowned overwrites; but every pre-P03 msvc-kit root has neither ownership.json nor installed manifests, so strict refusal would break re-installs (I-013 CLI compat).
- Decision: the commit pre-pass classifies existing files as ReplaceOwned when (a) the ownership index records the current digest, or (b) the root carries legacy msvc-kit evidence (.msvc-kit/receipts or .msvc-kit-extracted markers). Replace paths always back up the original bytes into the transaction backup dir before installing. Truly unowned files (no evidence) remain UnownedConflict refusals.
- Why: keeps the migration one-way and auditable (journal records every adoption), never deletes unknown files, and preserves CLI compatibility.
- Trade-offs: the legacy evidence check is root-granular, not file-granular (markers are per-archive); backup files live in the transaction dir until commit cleanup.
- Evidence/tests: replace_owned test, unowned refusal test.
- Revisit condition: P07 installed manifests supersede legacy evidence.

## DEC-015: Recovery semantics for write-ahead events and post-Committed faults
- Date: 2026-11-22
- Phase: P03
- Context: Spec 05 section 8.3: recovery must distinguish journaled-but-not-executed from executed-but-not-journaled, and default to rollback after executed actions.
- Decision: recovery groups journal events per path and reconstructs intent: WillBackup/BackupDone carry the old digest (metadata: None means did-not-exist); WillInstall/InstallDone carry the new digest. Rollback removes created files only when they still match the installed digest, restores replaced files from backup (refusing with needs_recovery when a third party modified them), reverts the ownership index, and voids the transaction installed manifest. A fault after the committed event is treated as a durable install (wiring logs and continues; recovery only cleans backups).
- Why: hash-gating makes recovery idempotent and safe under concurrent third-party writes; the committed event is the single point of no return.
- Trade-offs: recovery cannot restore bytes if the backup file was lost with the transaction dir in a pre-fix journal format - the fault matrix caught exactly this class (journal/backup name case mismatch) and the format was fixed before shipping.
- Evidence/tests: fault_injection_matrix (11 points), backup_fault_points, explicit_rollback tests.
- Revisit condition: pruning or CAS operations join the journal (P05/P06).

## DEC-016: MSI backend abstraction without a CLI flag; native stays behind the parity gate
- Date: 2026-11-22
- Phase: P04
- Context: The roadmap asks for an auto/msiexec/native/fallback abstraction and fake-runner tests; native parity cannot be met without user-approved real fixtures (spec 05 section 3, spec 10 section 2-3). Adding an msi-backend field to DownloadOptions would break external struct literals (I-013/B7).
- Decision: ship the preference enum and runner seam as a public library API (msvc_kit::installer::msi / lib re-exports) with native+fallback returning an explicit parity-gate error; extract_msi_sync delegates to the backend internally; no CLI flag or DownloadOptions field yet.
- Why: removes the duplicated retry loop, makes the wrapper testable per spec 08 section 8, and defers the option-surface decision to the options-builder strategy phase.
- Trade-offs: users cannot select a backend from the CLI yet (library callers can); native remains a documented, tested-as-error path.
- Evidence/tests: msi backend suite 10/10; feasibility report lists enablement criteria.
- Revisit condition: parity fixtures approved, or the options-builder lands (P07+).

## DEC-017: Pruning placeholder semantics and conservative built-in rule set
- Date: 2026-11-22
- Phase: P05
- Context: The spec example patterns use Host{host} (double-prefix bug) and {vctools} for SDK lib paths (wrong version namespace); {arch} must expand to every architecture the plan did NOT select, which requires the planner to know selected arches.
- Decision: placeholders are {vctools}, {sdk}, {hostdir} (full HostX64 form), {target}, {arch} (non-selected arches from context.selected_arches) and ** suffix wildcards; the built-in portable profile ships exactly three evidence-backed rules (vctip.exe, VisualStudio.Telemetry.dll, non-target-arch um/ucrt libs) scoped to the 14 toolset family; minimal ships zero rules until evidence exists; custom profiles load user TOML.
- Why: matches spec 06 section 1.2/1.4 (telemetry removal is portablemsvc-evidenced; PDB/DIA/debug-CRT/Auxiliary are explicit non-rules; rules must be version-scoped and auditable).
- Trade-offs: pattern matching is exact-or-suffix (no mid-path globs) - deliberately conservative; {arch} expansion requires selected_arches to be provided or the rule cannot fire.
- Evidence/tests: portable_prunes_telemetry_and_non_target_arch_libs_only (positive + negative fixtures), version_scoped_rules_skip_other_versions.
- Revisit condition: per-package manifest evidence for additional rules; real-payload verification (opt-in).

## DEC-018: CAS as an overlay store with CLI-managed import/GC (downloader layout unchanged)
- Date: 2026-11-22
- Phase: P06
- Context: Spec 06 section 2.5 requires migration to be re-runnable and the per-version download layout to keep working; converting the downloader to CAS-native downloads touches receipts, extraction, bundle and every consumer.
- Decision: the BlobStore is an overlay beside the existing caches - import_file ingests existing downloads (hash -> atomic publish -> source refs in redb), gc() sweeps blobs whose sources vanished (respecting pins and a retention window), and the cache CLI subcommand manages import/verify/gc. Downloader-native CAS writes and validator-aware Range resume stay future work (RISKS R-104).
- Why: satisfies every P06 exit gate without touching the install path; blobs on disk survive index corruption (index renamed aside, imports re-register).
- Trade-offs: dedup happens only for imported files until the downloader publishes into CAS directly; pinning is manual until transaction integration.
- Evidence/tests: 6 CAS tests incl. dedup, quarantine, GC dry-run/pins/retention/idempotence, re-runnable migration.
- Revisit condition: plan/apply integration (P08) or downloader rework.

## DEC-019: Lock v2 flattens the v1 core instead of duplicating fields; InstalledManifest becomes the public verify contract
- Date: 2026-11-22
- Phase: P07
- Context: v2 could either re-declare v1 fields (duplication drift) or flatten the v1 core struct; and tree verification needs InstalledManifest to be public (it was pub(crate) from P03).
- Decision: ToolchainLockV2 #[serde(flatten)]s a LockedSelectionCore (v1 fields verbatim) and adds provenance/pruning/manifest-digest/CAB-relations; InstalledManifest + InstalledFileEntry + INSTALLED_MANIFEST_SCHEMA are re-exported through installer. verify_installed_tree re-hashes every manifest entry and reports Modified/Missing/Extra, skipping .msvc-kit/.msvc-kit-extracted/downloads; it never deletes.
- Why: one source of truth for selection fields; the verify contract is the natural public surface for mutation detection.
- Trade-offs: flattening couples v2 to v1 field names (acceptable - the v1 core is frozen); extra-file reporting is informational only.
- Evidence/tests: 6 lock-v2 tests; 703 passed / 0 failed.
- Revisit condition: verify/doctor CLI wiring (P08+).

## DEC-020: Resolver uses find_msvc_packages for seeds, not naive fragment matching; 16 MB main-thread stack for the CLI
- Date: 2026-11-22
- Phase: P08
- Context: The resolver's capability-to-package mapping could use naive substring matching or the battle-tested find_msvc_packages (arch filtering, component opt-ins, exclude patterns). Also, the CLI binary's main async block embeds every match arm's inner futures on the 1 MB main-thread stack (Windows); adding create_install_plan_with_profile pushed it past the limit in debug builds.
- Decision: (1) profile seeds come from find_msvc_packages (including the profile component opt-ins and exclude patterns), then the resolver closes over manifest dependencies with strict schema validation; (2) the main-thread stack is set to 16 MB via .cargo/config.toml linker flags for Windows/MSVC targets; (3) large plan futures are Box::pinned to heap-allocate their state machines.
- Why: (1) avoids reimplementing arch filtering and component logic that find_msvc_packages already handles correctly; (2) the stack overflow affects --version (main frame allocated at entry), not just the plan path; (3) heap-allocating large futures is the standard tokio fix.
- Trade-offs: (1) the resolver's fragment matching is only exercised for dependency closure, not primary selection; (2) 16 MB stack is generous but safe for debug builds with large futures; (3) Box::pin adds a heap allocation per plan call.
- Evidence/tests: 11 resolver tests + stack overflow diagnosis (D:/rust/p08diag-status.log).
- Revisit condition: if CI reports stack overflow on other arms, box their futures too; if the resolver's fragment matching needs arch filtering for the closure, add it.

## DEC-021: Detection is strictly read-only with kind-prefix reuse matching
- Date: 2026-11-22
- Phase: P09
- Context: Spec 07 section 4 requires the reuse planner to distinguish ReusableExact/ReusableWithAdditions/Incompatible/Unknown; the naive approach checks only the detected toolchain's own validation checks (which would give false positives when the required capability belongs to a different kind).
- Decision: (1) all detection functions are strictly read-only (is_file/is_dir/read_dir only); (2) plan_reuse compares each required capability's prefix against the detected kind's namespace before consulting validation results - a mismatch is Incompatible regardless of the detected toolchain's own health; (3) unit tests use temp-dir fixtures, never PATH scanning.
- Why: prevents a false ReusableExact when a user points the detector at a Windows SDK root and asks for MSVC capabilities.
- Trade-offs: the planner does not yet handle cross-kind reuse (e.g. SDK root serving the SDK part of an MSVC-only capability set) - that is the plan integration's job.
- Evidence/tests: reuse_planner_incompatible_for_wrong_kind (SDK root + MSVC capability = Incompatible).
- Revisit condition: cross-kind reuse planner (P10+).

## DEC-022: Provider API is a thin trait layer, not a full lifecycle replacement
- Date: 2026-11-22
- Phase: P10
- Context: The spec suggests ToolchainProvider with detect/resolve/acquire/validate/environment lifecycle methods. The existing code has direct pipeline calls (download, extract, record) that work correctly; replacing them wholesale would be a large refactor with high regression risk.
- Decision: define ToolchainProvider and BuildToolProvider as trait interfaces for detect/resolve_version/include_components/validate_downloaded, with MsvcToolchainProvider and SimpleBuildToolProvider as concrete implementations. The shared infrastructure (Plan, Cache, Transaction, Receipt) remains owned by the caller.
- Why: additive, low-risk; the traits can grow without breaking existing callers; the full lifecycle replacement is future work.
- Trade-offs: the Provider API does not yet drive the CLI pipeline; external providers cannot plug in until the plan/apply pipeline accepts them.
- Evidence/tests: 4 provider tests; full suite green.
- Revisit condition: when external provider support is needed.
