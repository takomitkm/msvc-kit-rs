# Phase P07 Report — Toolchain Lock v2 / Installed-tree Verification

## Scope
Per roadmap P07: lockfile v2 schema (source pinning, CAB relations, pruning records, installed-manifest digest), v1 reader compatibility, selection-only marking, and installed-tree mutation detection (modified/missing/extra).

## Baseline code findings
- ToolchainLock v1 (selection + receipts, deny_unknown_fields) existed and is kept unchanged as the v1 reader.
- InstalledManifest existed internally (P03) but was not part of any verification flow.

## Changes
- `src/installer/lock_v2.rs` (new public module):
  - `ToolchainLockV2` (`msvc-kit.toolchain-lock.v2`, deny_unknown_fields): flattens the v1 selection core and adds `provenance` (`source-pinned` / `selection-only`), `pruning` records (path/rule-id/version/reason), `installed_manifest_digest`, and `cab_relations` (MSI payload → CAB source media).
  - Validation rejects unknown schemas/provenances and provenance/receipt contradictions (a `source-pinned` lock must carry receipts; a `selection-only` lock must not claim them).
  - `from_v1` builds v2 from the v1 reader output plus install provenance.
  - `verify_installed_tree` re-hashes every manifest entry and reports `Modified`/`Missing`/`Extra` diffs; extra detection skips `.msvc-kit`, `.msvc-kit-extracted` and `downloads` (msvc-kit control dirs); verification never deletes (I-001).
  - `load_installed_manifest` (latest or by install id) + `detect_installed_mutation` compose the flows.
- `src/toolchain_lock.rs`: `LOCK_V2_SCHEMA` const (module-level; v1 reader untouched).
- `src/installer/contracts.rs`: `InstalledManifest`/`InstalledFileEntry`/`INSTALLED_MANIFEST_SCHEMA` made public (the verify contract).
- `src/lib.rs` / `src/installer/mod.rs`: registration + re-exports.

## Tests added (in-module, 6)
- `v1_lock_file_is_still_readable_by_the_v1_reader` (v1 compat).
- `v2_round_trips_and_carries_provenance` (schema, selection-only marking, pruning + CAB records + manifest digest).
- `source_pinned_requires_receipts` (contradiction rejected; consistent provenance accepted).
- `unknown_schema_and_provenance_are_rejected`.
- `installed_tree_mutation_is_detected` (pristine → empty; modified/missing/extra each detected; `.msvc-kit` internals skipped).
- `detect_installed_mutation_loads_latest_manifest`.

## Commands executed
| Command | Exit | Result | Output/log |
|---|---:|---|---|
| `cargo test --all-features --no-fail-fast` (final) | 0 | PASS — 703 passed / 0 failed / 7 ignored | D:\rust\p02c-test.log |
| `cargo fmt --all -- --check` (final) | 0 | PASS | D:\rust\p02c-fmt.log |
| `cargo clippy --all-targets --all-features -- -D warnings` (final) | 0 | PASS | D:\rust\p02c-clippy.log |
| `cargo test --doc` / `cargo doc` -D warnings | 0 | PASS | D:\rust\p02c-test-doc.log, p02c-doc.log |
| Intermediate rounds | — | fixes: LOCK_V2_SCHEMA moved out of an impl block, SystemTime path, InstalledManifest made public (verify contract), provenance/receipt contradiction logic corrected, rustfmt reformatting, flaky downloader test annotated (known pre-existing parallel-load flake, passes twice in isolation) | D:\rust\p02c-*.log, p03a-test.log |

## Compatibility
- CLI: unchanged (verify/doctor flag wiring lands with the plan/apply integration).
- Rust API: additive — `installer::lock_v2` module public; `InstalledManifest` types exported.
- JSON/schema: new `msvc-kit.toolchain-lock.v2` (v1 files still load through the v1 reader; v2 files declare their schema explicitly).

## Security review
- source-pinned vs selection-only enforced by validation — a selection-only lock can never claim pinned bytes.
- Verification is read-only; extra files are reported, never deleted (I-001).
- Re-hashing every entry: no size/mtime shortcuts (I-004 layer 6).

## Unresolved items
- `verify`/`doctor --lockfile` CLI wiring (schema + tree) is staged for the P08 plan/apply integration where the flows are finalized.
- Lock v2 production writes (from transactions) land with the same integration.

## Exit Gate
- [x] v1 lock readable (v1 reader untouched, test)
- [x] v2 rebuild carries the exact selection (flattened v1 core) + provenance records
- [x] source-pinned vs selection-only enforced (validation + tests)
- [x] installed mutation detected (modified/missing/extra tests)
- [x] Receipt/lock consistency (v2 carries receipts verbatim from v1; contradiction validated)

## Decision
COMPLETE
