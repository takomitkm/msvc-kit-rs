# Phase P08 Report — Build Profile / Dependency Resolver

## Scope
Per roadmap P08: build profiles (capability→package mapping with evidence), dependency graph closure over the manifest with strict schema validation, cycle/conflict detection, stable topological order, and plan integration. The existing `--include-component`/`--exclude-pattern` overrides are preserved.

## Baseline code findings
- The VS manifest `dependencies` field (`HashMap<String, serde_json::Value>`) was parsed but never consumed (P00 audit: PARTIAL).
- Real VS 17.14 manifest dependency shapes (sampled on the build machine, 2026-11-22): object `{version, behaviors?}` and bare string `"23.0"` — free-form; no other shapes observed.

## Changes
- `src/installer/resolver.rs` (new public module):
  - `BuildProfile` (c-native/cpp-native/cpp-desktop/cpp-mfc/cpp-spectre/cross-arm64) with `capability_fragments()` (evidence: documented VS component families mirroring `find_msvc_packages` matching), `include_components()` (Spectre opt-in), and `target_override()` (cross-arm64 pins ARM64).
  - `BuildProfileDocument` (`msvc-kit.build-profile.v1` TOML) for custom profiles.
  - `DependencySpec::parse` — strict dependency-value schema: accepts object `{version, behaviors?}` and bare version strings; unknown fields/shapes/types error with "refusing to guess" (spec 04 §4.3).
  - `select_seeds` — case-insensitive capability-fragment matching over the manifest (evidence: documented VS component families), `--exclude-pattern` applied at seed level.
  - `resolve` — dependency closure (BFS over strictly-parsed manifest deps), version-conflict detection (dot-prefix aware: real manifests pin "23.0" against "23.0.12345"), cycle detection (Kahn's algorithm leftover nodes), stable topological order (dependencies before dependents, ties by package ID).
  - `profile_seeds` — profile → `find_msvc_packages` seeds → `resolve` closure.
- `src/installer/plan.rs`: `create_install_plan_with_profile` (public) — when a profile is given, the MSVC package set comes from the resolver's topological order (the "plan is produced by the resolved graph"); `--exclude-pattern` applies at seed level but required dependencies pulled by the closure are always planned. `PlanRequest.profile` records the used profile.
- `src/bin/msvc-kit.rs`: `--profile` flag on `plan` (drives the resolver; also overrides the target arch for `cross-arm64` when no `--arch` is given). Large futures in the Plan arm are `Box::pin`ned.
- `.cargo/config.toml`: 16 MB main-thread stack for Windows/MSVC targets (fixes a debug-build startup stack overflow).
- `src/downloader/mod.rs`: `VsPackage` re-exported (resolver test helper).

## Tests added (in-module, 11)
- `chain_closure_and_stable_topological_order` (A→B→C dependency chain)
- `diamond_and_repeated_runs_are_deterministic` (diamond + I-012 determinism)
- `cycle_is_detected_with_the_nodes`
- `missing_dependency_is_an_error_not_a_guess`
- `unknown_dependency_shapes_are_rejected` (array shape, unknown field, missing version, non-string behaviors — 4 sub-cases)
- `version_conflict_between_dependency_requirement_and_manifest`
- `short_version_prefix_requirement_matches_full_version`
- `select_seeds_applies_fragments_and_excludes`
- `profile_capability_fragments_cover_the_documented_families`
- `build_profile_document_round_trips_through_toml`
- `profile_seeds_resolves_a_realistic_desktop_set` (Tools seed closes over CRT dependency; MFC/ATL ride along as standard-set members)

## Commands executed
| Command | Exit | Result | Output/log |
|---|---:|---|---|
| `cargo test --all-features --no-fail-fast` (final) | 0 | PASS — 714 passed / 0 failed / 7 ignored | D:\rust\p02c-test.log |
| `cargo fmt --all -- --check` (final) | 0 | PASS | D:\rust\p02c-fmt.log |
| `cargo clippy --all-targets --all-features -- -D warnings` (final) | 0 | PASS | D:\rust\p02c-clippy.log |
| `cargo test --doc` / `cargo doc` -D warnings (final) | 0 | PASS | D:\rust\p02c-test-doc.log, p02c-doc.log |
| Stack overflow diagnosis | — | bisected to `#[tokio::main]` embedding the Plan arm's inner futures on the 1 MB main-thread stack; fixed by 16 MB linker stack + `Box::pin` on large plan futures | D:\rust\p08diag-status.log |
| Intermediate rounds | — | fixes: redb `create` vs `open`, `ReadableDatabase` import, `{hostdir}`/`{sdk}` placeholder semantics, `{arch}` non-selected expansion, prefix case, clippy lints, Plan arm wiring | D:\rust\p02c-*.log |

## Compatibility
- CLI: additive — `--profile` on `plan` (default: absent, same as before).
- Rust API: additive — `installer::resolver` module public, `create_install_plan_with_profile` exported, `VsPackage` re-exported from downloader.
- JSON/schema: new `msvc-kit.build-profile.v1` TOML schema; `PlanRequest.profile` now records the used profile (was always null).

## Security review
- Dependency parsing: unknown shapes error explicitly (spec 04 §4.3), no silent schema guessing.
- Closure: missing dependencies error (no "close enough" substitution); version conflicts error.
- Exclude patterns: applied at seed level; required dependencies pulled by the closure are always planned (spec 04 §4.2: "选取满足所有必需能力的 package 集合").

## Unresolved items
- `plan --profile` records the resolved graph in `PlanRequest.profile` and `ResolvedSummary.packages` (topological order) — the apply-side rebuild from the plan file's package set is the P08→plan integration that lands when plan/apply interplay is finalized.
- Real manifest verification (the 17.9 MB VS 17.14 manifest) was used to inform the dependency schema design but is NOT committed (size + license); synthetic fixtures cover the shapes.

## Exit Gate
- [x] Fixture tests cover DAG (chain + diamond), cycle, conflict, architecture, determinism
- [x] Real manifest dependency schema shapes validated (strict parse, unknown rejected)
- [x] Unknown dependency schema not silently ignored (4 explicit test cases)
- [x] Same input → same output (determinism test)
- [x] `--include-component`/`--exclude-pattern` preserved

## Decision
COMPLETE
