# Progress

## Current phase
P08 — Build Profile / Dependency Resolver (IN_PROGRESS; resumed 2026-11-22 with user approval to continue P08-P11).

## Status
P00-P07 all COMPLETE, each pushed with CI workflow success on the phase HEAD
(see `INTERIM-STATUS.md` for the phase/commit table and the pause reason).

- P00 b784452 - baseline audit + baseline repair
- P01 448f877 - path security + safe extractors + marker v2
- P02 21e3603 - InstallPlan + plan/apply CLI
- P03 67d87dc - journaled file-level transactions
- P04 a032e24 - MSI backend abstraction + feasibility report
- P05 6bc0a51 - pruning engine
- P06 f7fc575 - CAS blob cache
- P07 d493133 - lock v2 + installed-tree verification

## Completed (P07, the final phase of this run)
- [x] ToolchainLockV2 (msvc-kit.toolchain-lock.v2): flattened v1 core + provenance (source-pinned/selection-only) + pruning records + installed-manifest digest + CAB relations
- [x] Provenance/receipt contradiction validation
- [x] verify_installed_tree: re-hash every entry; Modified/Missing/Extra diffs; control dirs skipped; never deletes
- [x] detect_installed_mutation + load_installed_manifest (latest or by id)
- [x] InstalledManifest made public (verify contract)
- [x] 6 tests; full verification green (703 passed / 0 failed / 7 ignored)

## Completed (P08)
- [x] BuildProfile (c-native/cpp-native/cpp-desktop/cpp-mfc/cpp-spectre/cross-arm64) with evidence-backed capability fragments
- [x] DependencySpec strict parse (object{version,behaviors?} + bare string; unknown shapes error)
- [x] Dependency closure + version conflict + cycle detection + stable topological order
- [x] profile_seeds integrating find_msvc_packages (standard matcher) with resolver closure
- [x] create_install_plan_with_profile (plan from resolved graph) + --profile CLI flag
- [x] 16 MB main-thread stack fix (Windows debug-build stack overflow from embedded futures)
- [x] 11 resolver tests; full verification green (714 passed / 0 failed)

## Completed (P09)
- [x] detection module: DetectedToolchain/ValidationSummary/ReuseDecision types
- [x] detect_msvc (cl/link/lib/headers/libs validation per target arch)
- [x] detect_windows_sdk (um/ucrt libs + headers validation)
- [x] detect_build_tool (CMake/Ninja/NASM existence)
- [x] plan_reuse (capability prefix vs kind mismatch = Incompatible; all present = ReusableExact; some missing = ReusableWithAdditions; no requirements = Unknown)
- [x] 9 fixture-based tests (no PATH, read-only verified, false-positive, reuse states)
- [x] Full verification green (724 passed / 0 failed)

## Remaining
- [ ] push P09 commits; verify CI workflow conclusion=success on the pushed sha
- [ ] P10 (next): Provider API
- [ ] P09 - Detection / Reuse
- [ ] P10 - Provider API
- [ ] P11 - Full regression / docs / release readiness / FINAL_REPORT.md

## Files changed (P09)
- `src/installer/detection.rs`: new public module
- `src/installer/mod.rs`: registration
- `src/lib.rs`: re-exports
- `agent-work/PHASES/P09-detection.md`: phase report

## Files changed (P08)
- `src/installer/resolver.rs`: new public module
- `src/installer/plan.rs`: create_install_plan_with_profile
- `src/bin/msvc-kit.rs`: --profile CLI + Box::pin large futures
- `src/downloader/mod.rs`: VsPackage re-export
- `.cargo/config.toml`: 16 MB main-thread stack (Windows MSVC)
- `agent-work/PHASES/P08-resolver.md`: phase report

## Files changed (P07)
- `src/installer/lock_v2.rs`: new public module (v2 schema, verify_installed_tree, mutation detection)
- `src/toolchain_lock.rs`: LOCK_V2_SCHEMA const
- `src/installer/contracts.rs`: InstalledManifest public
- `src/installer/mod.rs`, `src/lib.rs`: registration + re-exports
- `src/downloader/common_tests.rs`: flaky test annotated (pre-existing parallel-load flake, passes twice in isolation)
- `agent-work/INTERIM-STATUS.md`: pause record

## Tests
- PASS: `cargo fmt --all -- --check` (D:/rust/p02c-fmt.log, final P09 round)
- PASS: `cargo clippy --all-targets --all-features -- -D warnings` (D:/rust/p02c-clippy.log)
- PASS: `cargo test --all-features --no-fail-fast` - 724 passed / 0 failed / 7 ignored (D:/rust/p02c-test.log)
- PASS: `cargo test --doc` - 23 passed; `cargo doc` -D warnings (D:/rust/p02c-test-doc.log, p02c-doc.log)
- Known flake (pre-existing, annotated): `a_corrupted_first_transfer_is_refetched_and_succeeds` - fails rarely under full-suite parallel load; passes twice in isolation.

## Risks / regressions
- See `agent-work/RISKS.md` and `agent-work/INTERIM-STATUS.md`.
- rustfmt drift between CI and the build machine caused two style-only fix commits (6bc0a51, f7fc575); the CI-failing shas e77657c and 6087ebb were repaired same-session.

## Next exact action
Commit P09 as `feat(detection)` and `docs(agent-work)`; push; poll CI for conclusion=success. Then P10: Provider API (ToolchainProvider/BuildToolProvider layering, MSVC/SDK provider adaptation). Prepared input: the real VS
17.14 package manifest dependency schema was sampled on the build machine
(`{ "<package-id>": { "version": "...", "behaviors": "..." } }` free-form
shapes) - the resolver must validate dependency shapes strictly (unknown
shapes error, never silently ignored, spec 04 section 4.3). Then P09 detection, P10
provider API, P11 full regression + docs + FINAL_REPORT.md.
