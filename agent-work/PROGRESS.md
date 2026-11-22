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

## Remaining
- [ ] P08 - Build Profile / Dependency Resolver (in progress)
- [ ] P09 - Detection / Reuse
- [ ] P10 - Provider API
- [ ] P11 - Full regression / docs / release readiness / FINAL_REPORT.md

## Files changed (P07)
- `src/installer/lock_v2.rs`: new public module (v2 schema, verify_installed_tree, mutation detection)
- `src/toolchain_lock.rs`: LOCK_V2_SCHEMA const
- `src/installer/contracts.rs`: InstalledManifest public
- `src/installer/mod.rs`, `src/lib.rs`: registration + re-exports
- `src/downloader/common_tests.rs`: flaky test annotated (pre-existing parallel-load flake, passes twice in isolation)
- `agent-work/INTERIM-STATUS.md`: pause record

## Tests (final P07 round)
- PASS: `cargo fmt --all -- --check` (D:/rust/p02c-fmt.log)
- PASS: `cargo clippy --all-targets --all-features -- -D warnings` (D:/rust/p02c-clippy.log)
- PASS: `cargo test --all-features --no-fail-fast` - 703 passed / 0 failed / 7 ignored (D:/rust/p02c-test.log)
- PASS: `cargo test --doc` - 23 passed; `cargo doc` -D warnings (D:/rust/p02c-test-doc.log, p02c-doc.log)
- Known flake (pre-existing, annotated): `a_corrupted_first_transfer_is_refetched_and_succeeds` - fails rarely under full-suite parallel load; passes twice in isolation.

## Risks / regressions
- See `agent-work/RISKS.md` and `agent-work/INTERIM-STATUS.md`.
- rustfmt drift between CI and the build machine caused two style-only fix commits (6bc0a51, f7fc575); the CI-failing shas e77657c and 6087ebb were repaired same-session.

## Next exact action
P08: write failing resolver fixture tests (DAG/cycle/conflict/unknown-shape/determinism/profiles), then implement installer::resolver (strict dependency schema, closure, stable topo sort) and wire plan integration. Prepared input: the real VS
17.14 package manifest dependency schema was sampled on the build machine
(`{ "<package-id>": { "version": "...", "behaviors": "..." } }` free-form
shapes) - the resolver must validate dependency shapes strictly (unknown
shapes error, never silently ignored, spec 04 section 4.3). Then P09 detection, P10
provider API, P11 full regression + docs + FINAL_REPORT.md.
