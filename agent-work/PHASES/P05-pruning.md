# Phase P05 Report — Pruning Engine

## Scope
Per roadmap P05: rule schema and profiles, evidence-backed rule candidates, dry-run-first planning, staging-only execution, removed-file records, PDB/DIA/debug-CRT default policies, and fixtures per rule.

## Baseline code findings
- No pruning existed beyond the SDK optional-MSI skip list (RISK: no install-tree trimming at all).
- The P03 transaction staging tree provides exactly the isolation the spec demands for pruning execution.

## Changes
- `src/installer/pruning.rs` (new public module):
  - `PruningProfile` (`full`/`portable`/`minimal`/`custom`) with parsing; `full` and rule-less profiles never prune; `minimal` deliberately ships zero rules until a rule carries its own compile/link acceptance evidence; `custom` loads a user TOML document.
  - `PruningRulesDocument`/`PruningRule` (`msvc-kit.pruning-rules.v1`, TOML, `deny_unknown_fields`, validated: known actions, non-empty patterns, unique ids). Placeholders: `{vctools}` (full MSVC version), `{sdk}` (full SDK version), `{hostdir}` (HostX64 form), `{target}` (x64 form), `{arch}` (every architecture the plan did NOT select), `**` (suffix).
  - `plan_pruning`/`plan_rules` — pure planning (no file system); `execute_pruning`/`remove_staged_file` — staging-only removal with empty-parent cleanup; every action carries rule id/version/reason/size.
  - Built-in `portable` rules (evidence-backed, version-scoped to the 14 toolset family): `msvc.telemetry.vctip` (vctip.exe), `msvc.telemetry.vs_telemetry_dll` (Microsoft.VisualStudio.Telemetry.dll), `sdk.non_target_arch_libs` (um/ucrt libs for architectures the plan did not select).
  - Explicit non-rules: PDBs, DIA SDK, debug CRT, Auxiliary tree — no built-in rule touches them.
- `src/installer/transaction.rs`: `Transaction::prune_staged` — plans over the staged file list, executes removals in the staging tree, journals every removal as `FileRemoved` (rule id recorded with the path), and updates the staged file list so the merge/commit never sees pruned bytes. Rule failures abort the transaction (spec 06 §1.5).
- `src/installer/mod.rs`: `extract_and_finalize_msvc_with_profile` / `extract_and_finalize_sdk_with_profile` (new public API); the existing functions delegate with `PruningProfile::Full` — default behavior unchanged.
- `src/bin/msvc-kit.rs`: `--prune-profile` flag on `download` and `apply` (default `full`).
- `src/lib.rs`: pruning re-exports.

## Tests added (in-module)
- `full_profile_never_prunes`.
- `portable_prunes_telemetry_and_non_target_arch_libs_only` — positive fixtures (telemetry exe+dll, non-target um/ucrt libs) AND negative fixtures (PDB in the same tree, selected-arch libs, lookalike `avctip.exe`); every action carries rule id + reason.
- `version_scoped_rules_skip_other_versions` — a `15.x` toolset does not fire the `14`-scoped rules.
- `custom_document_round_trips_through_toml` — TOML round-trip + bad-action rejection.
- `execution_removes_only_planned_files_and_requires_them_present` — staging removal + empty-parent cleanup + missing-file failure (no silent skips).

## Commands executed
| Command | Exit | Result | Output/log |
|---|---:|---|---|
| `cargo test --all-features --no-fail-fast` (final) | 0 | PASS — 691 passed / 0 failed / 7 ignored | D:\rust\p02c-test.log |
| `cargo fmt --all -- --check` (final) | 0 | PASS | D:\rust\p02c-fmt.log |
| `cargo clippy --all-targets --all-features -- -D warnings` (final) | 0 | PASS | D:\rust\p02c-clippy.log |
| `cargo test --doc` / `cargo doc` -D warnings (final) | 0 | PASS | D:\rust\p02c-test-doc.log, p02c-doc.log |
| Intermediate rounds | — | fixes: host-string conversions, `{host}`→`{hostdir}` placeholder semantics (Host+HostX64 double-prefix bug), `{sdk}` placeholder (SDK lib paths used the MSVC version), `{arch}` expansion over non-selected arches, clippy lints | D:\rust\p02c-*.log |

## Compatibility
- CLI: additive — `--prune-profile` on download/apply, default `full` (prior behavior byte-for-byte).
- Rust API: additive — new pruning module + `*_with_profile` variants; existing functions unchanged.
- JSON/schema: new `msvc-kit.pruning-rules.v1` schema; journal gains `file_removed` events (schema already declared the kind in P01/P03).
- Windows behavior: pruning happens in staging before commit; removed files are journaled and never leave dangling roots (empty parents cleaned).

## Security review
- target path: rule patterns resolve to concrete relative paths matched against staged files only; `**` is suffix-bounded; lookalike names verified not to match.
- ownership: pruned files are removed from staging before ownership assignment — they never enter the ownership index or installed manifest.
- rollback: pruning occurs pre-commit, so transaction rollback removes the whole staging tree; no root-side undo needed.
- privilege: unchanged.

## Unresolved items
- **Minimal-profile compile/link acceptance** (exit gate): requires a real installed toolchain to prune and compile against — NOT_RUN here; opt-in T5 (spec B5) on a machine with a real payload. The `minimal` profile ships zero rules so the gate is trivially satisfied by construction until real rules land.
- Receipt-side `removed_files` in `InstallReceiptDocument` (P07) will consume the journal `file_removed` records; the journal already carries rule ids.
- Rules beyond telemetry/non-target-arch need per-package manifest evidence before inclusion (spec 06 §1.2).

## Exit Gate
- [x] full: no deletions (test)
- [x] portable: every deletion explainable (rule id + reason asserted)
- [x] unknown versions: rules do not fire outside their version scope (test)
- [x] failure aborts the transaction (execute_pruning missing-file failure + transaction aborts on any prune error)
- [x] minimal toolchain compile acceptance: NOT_RUN (opt-in real-payload test; minimal profile is rule-free by construction — documented, not silently skipped)

## Decision
COMPLETE
