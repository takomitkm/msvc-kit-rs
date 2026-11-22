# Interim Status (2026-11-22) — paused after P07 by user request

The user directed (2026-11-22) that the agent complete its current phase and
stop. P07 was complete at that point (d493133, CI success); P08 had NOT been
started (no implementation code written - only non-repo analysis of the real
Visual Studio manifest dependency schema on the build machine). This file
records the pause; `PROGRESS.md` is the authoritative phase log and
`FINAL_REPORT.md` is still reserved for the P11 completion.

## Completed phases (each pushed; CI workflow success on every phase HEAD)

| Phase | Scope | Commits | Phase HEAD (CI green) |
|---|---|---|---|
| P00 | Baseline audit + baseline repair (fmt/clippy/703-test baseline re-established; pre-existing CI red diagnosed as cargo-1.100 test-layout + rustdoc drift) | 5944777, d41a99f, b784452 | b784452 |
| P01 | SafeRelativePath (Windows path semantics), guarded VSIX/CAB extractors (validate-all-then-write), marker v2, internal contract schemas | ca2ec3e lineage: 968826d, 3433791, 448f877 | 448f877 |
| P02 | InstallPlan v1 + planner + plan_id + plan/apply CLI + stale detection | cd08e62, 6514c72, 21e3603 | 21e3603 |
| P03 | Journaled file-level transactions: staging, merge, backup/commit/rollback, recovery, ownership index | 22c6fed, 4b32cb6, 67d87dc | 67d87dc |
| P04 | MSI backend abstraction (runner seam, msiexec /a wrapper hardened with /L*V log + retry tests) + native feasibility report (parity gate unmet, native stays disabled) | f1b1d71, a032e24 | a032e24 |
| P05 | Pruning engine (profiles full/portable/minimal/custom, TOML rules schema, staging-only execution journaled) + --prune-profile CLI | ca2ec3e, e77657c | 6bc0a51 (style fix for newer CI rustfmt) |
| P06 | Content-addressed blob cache (atomic publish, verify+quarantine, source refs, GC mark/sweep with dry-run/pins) + cache CLI | 0cf449a, 6087ebb, f7fc575 | f7fc575 |
| P07 | Toolchain lock v2 (provenance, pruning records, CAB relations, manifest digest) + installed-tree mutation detection | 6cb2356, d493133 | d493133 |

## Paused at: P08 — Build Profile / Dependency Resolver

- No implementation code exists for P08 (verified: nothing written before the
  pause instruction arrived).
- Prepared analysis (not repo work): the real VS 17.14 package manifest
  (17.9 MB) was fetched on the build machine and its dependency JSON schema
  sampled — dependency maps are `{ "<package-id>": { "version": "...",
  "behaviors": "..." } }` with free-form value shapes; this informs the
  strict-schema requirement (unknown dependency shapes must error, not be
  silently ignored, per spec 04 §4.3).

## Remaining phases (not started)

- P08 — Build Profile / Dependency Resolver (capability mapping, closure,
  cycle/conflict, stable topological sort, plan integration)
- P09 — Detection / Reuse (MSVC/SDK/LLVM/MinGW detectors, capability-based
  reuse planner)
- P10 — Provider API (ToolchainProvider/BuildToolProvider layering)
- P11 — Full regression, documentation, release readiness, FINAL_REPORT.md

## Known risks / open pointers

- `agent-work/RISKS.md` — P0 risk register (R-003 partial-install risk
  resolved by P03 transactions; R-005 unowned-overwrite protection landed in
  P03; remaining items tracked there).
- `agent-work/TEST-RESULTS.md` — per-phase verification matrix; one known
  flaky upstream test (`downloader::common_tests::
  a_corrupted_first_transfer_is_refetched_and_succeeds`) fails rarely under
  full-suite parallel load on Windows and passes twice in isolation
  (annotated in-source, P07-T04).
- CI "Code Coverage" workflow: fork lacks Codecov upload credentials; the
  codecov upload step fails there by design — its test steps are the signal
  (per the standing fork-noise instruction).
- rustfmt version drift between CI (rustup-managed, newer) and the build
  machine (1.100) required two style-only fix commits (6bc0a51, f7fc575) and
  was the cause of the single CI failure after P05 (resolved same-session).
