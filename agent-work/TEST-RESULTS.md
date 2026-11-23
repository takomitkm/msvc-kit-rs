# Test Results (P00)

Result vocabulary: PASS / FAIL / BLOCKED_ENVIRONMENT / NOT_RUN. NOT_RUN and BLOCKED are never counted as PASS.
All cargo commands were executed on the school machine (Win11-HV1, cargo 1.100.0, MSVC 14.44.35207, SDK 10.0.26100, standard user) via scheduled task; the local edit machine never compiles.

| Test ID | Phase | Command / Test | OS / Privilege | Result | Evidence/log | Notes |
|---|---|---|---|---|---|---|
| P00-T01 | P00 | `cargo fmt --all -- --check` (initial, pre-component-install) | Win11, user | BLOCKED_ENVIRONMENT | D:\rust\p00-fmt.log | `rustfmt` component missing; installed via `rustup component add rustfmt clippy` |
| P00-T02 | P00 | `cargo clippy --all-targets --all-features -- -D warnings` (initial) | Win11, user | BLOCKED_ENVIRONMENT | D:\rust\p00-clippy.log | `clippy` component missing at that moment |
| P00-T03 | P00 | `cargo test --all-features` (initial, fail-fast) | Win11, user | FAIL | D:\rust\p00-test.log | exit 101; `cli_exit_code_tests` 3 passed/17 failed; remaining suites not run (fail-fast) |
| P00-T04 | P00 | `cargo test --doc` (initial) | Win11, user | PASS | D:\rust\p00-test-doc.log | 23 passed |
| P00-T05 | P00 | `cargo fmt --all -- --check` | Win11, user | PASS | D:\rust\p00b-fmt.log | after component install |
| P00-T06 | P00 | `cargo clippy --all-targets --all-features -- -D warnings` | Win11, user | PASS | D:\rust\p00b-clippy.log | |
| P00-T07 | P00 | `cargo test --all-features --no-fail-fast` (baseline, before P00 fixes) | Win11, user | FAIL | D:\rust\p00b-test.log | exit 101; 24 suites: 22 green (incl. lib 218, downloader 50, query 47, toolchain_lock 8, e2e 16+4 ign, bundle 24), `cli_exit_code_tests` 3/17 FAIL, `update_tests` 5/4 FAIL |
| P00-T08 | P00 | experiment: copy `target/debug/msvc-kit.exe` into `target/debug/build/msvc-kit/640bc89b8bbaff73/out/`, rerun `cargo test --all-features --test cli_exit_code_tests` | Win11, user | PASS (20/20) | ssh transcript | proves root cause: cargo ≥1.100 runs test exes from `build\<pkg>\<hash>\out\`, helper assumed `deps\`; binary copied out again after the run |
| P00-T09 | P00 | GitHub Actions CI run 44968093273 @ 1e24a75 (windows-latest): Rust Tests job | CI, user | FAIL (same 17) | `agent-work/../../.tmp/msvc-kit/ci-rust-tests.log` (local), gh api | identical failures to P00-T07 → toolchain-drift baseline failure on main, not machine-specific |
| P00-T10 | P00 | GitHub Actions CI run 44968093273: Documentation job (`cargo doc` + `-D warnings`) | CI, user | FAIL | `.tmp/msvc-kit/ci-docs.log` | `unclosed HTML tag 'version'` at `src\install_into_vs\mod.rs:252` |
| P00-T11 | P00 | fix verification: `cargo fmt --all -- --check` (after P00 fixes) | Win11, user | PASS | D:\rust\p00c-fmt.log | fixes: robust binary lookup in `tests/cli_exit_code_tests.rs` + `tests/update_tests.rs`; doc comment escaped |
| P00-T12 | P00 | fix verification: `cargo doc --no-deps --document-private-items` with `RUSTDOCFLAGS=-D warnings` | Win11, user | PASS | D:\rust\p00c-doc.log | reproduces CI docs job locally |
| P00-T13 | P00 | fix verification: `cargo test --all-features --no-fail-fast` | Win11, user | PASS | D:\rust\p00c-test.log | 28 suites, 621 passed / 0 failed / 7 ignored (network-gated `#[ignore]`) |
| P00-T14 | P00 | fix verification: `cargo test --doc` | Win11, user | PASS | D:\rust\p00c-test-doc.log | 23 passed |
| P00-T15 | P00 | clippy re-check on fixed tree | Win11, user | PASS | D:\rust\p00d-clippy.log | exit 0 |
| P00-T16 | P00 | `cargo test --all-features --no-fail-fast` ignored-test audit | — | NOT_RUN | `tests/e2e_tests.rs:431-472` | 4 network-heavy e2e + 1 signed-catalog cache test + 2 preview-dir tests are `#[ignore]` by default; per spec B6 they stay opt-in; not executed in P00 |
| P01-T01 | P01 | `cargo test --all-features --test installer_safety_tests` (round 1, pre-implementation, tests-only patch) | Win11, user | FAIL (intended) | D:\rust\p01a-test.log | 4 passed / 19 failed — traversal escaped, ADS created, reserved names written, conflicts overwritten, legacy marker skipped extraction: suite proven to detect the bugs |
| P01-T02 | P01 | `cargo test --all-features --test installer_safety_tests` (round 2, post-implementation) | Win11, user | PASS | D:\rust\p01b-safety.log | 24/24 (incl. empty-dir policy added after clippy dead-code note) |
| P01-T03 | P01 | `cargo fmt --all -- --check` (round 3, after fmt pass) | Win11, user | PASS | D:\rust\p01c-fmt.log | |
| P01-T04 | P01 | `cargo clippy --all-targets --all-features -- -D warnings` (round 3) | Win11, user | PASS | D:\rust\p01c-clippy.log | round-2 clippy found dead_code (contracts, expected) + cloned_ref_to_slice_refs; fixed |
| P01-T05 | P01 | `cargo test --all-features --no-fail-fast` (round 3) | Win11, user | PASS | D:\rust\p01c-test.log | 29 suites, 660 passed / 7 ignored / 0 failed; includes path_safety unit matrix and contracts golden tests |
| P01-T06 | P01 | `cargo test --doc` (round 3) | Win11, user | PASS | D:\rust\p01c-test-doc.log | 23 passed |
| P01-T07 | P01 | `cargo doc --no-deps --document-private-items` + `RUSTDOCFLAGS=-D warnings` (round 3) | Win11, user | PASS | D:\rust\p01c-doc.log | |
| P01-T08 | P01 | `cargo test --all-features --no-fail-fast` (round 2 intermediate) | Win11, user | PASS | D:\rust\p01b-test.log | suite green before fmt normalization; round 3 re-verified after fmt-only diff |

## Suite totals after P00 fixes (P00-T13)

lib unittests 218, bin unittests 7, integration 24+20+11+29+7+50+16+36+3+6+11+5+4+0+5+47+14+2+3+8+8+5+9+42+8, doc 23 → **621 passed, 0 failed, 7 ignored** (unit+integration) plus doc 23.

## Baseline failure disposition

- All 21 pre-fix failures (`cli_exit_code_tests` 17, `update_tests` 4) shared one root cause (P00-T08, P00-T09) and were repaired by the P00 baseline-repair commit `fix(tests)` — no test was deleted or weakened; all 24 originally-registered test functions still run (20 in cli suite incl. the same assertions, 9 in update suite).
- The docs failure was repaired by escaping the placeholder in the doc comment; content unchanged.
| P02-T01 | P02 | `cargo test --all-features --no-fail-fast` (round 3) | Win11, user | PASS | D:/rust/p02c-test.log | 671 passed / 0 failed / 7 ignored; adds plan_tests (7) + plan/apply CLI smoke (3) |
| P02-T02 | P02 | `cargo fmt --all -- --check` (round 3) | Win11, user | PASS | D:/rust/p02c-fmt.log |  |
| P02-T03 | P02 | `cargo clippy --all-targets --all-features -- -D warnings` (round 3) | Win11, user | PASS | D:/rust/p02c-clippy.log |  |
| P02-T04 | P02 | `cargo test --doc` (round 3) | Win11, user | PASS | D:/rust/p02c-test-doc.log | 23 passed |
| P02-T05 | P02 | `cargo doc` + `RUSTDOCFLAGS=-D warnings` (round 3) | Win11, user | PASS | D:/rust/p02c-doc.log |  |
| P02-T06 | P02 | `cargo test --lib a_corrupted_first_transfer` (isolation rerun) | Win11, user | PASS | ssh transcript | full-suite failure earlier was a load-dependent parallel flake (pre-existing upstream test from c0b9e0e); not reproduced in the two following full runs |
| P02-T07 | P02 | mockito fetch-count choreography audit | - | PASS | tests/plan_tests.rs `FetchCounts` | channel always fully fetched (no independent digest); vsman full once then 304-conditionals; asserts encode exact expectations |
| P03-T01 | P03 | cargo test --all-features --test transaction_tests (final) | Win11, user | PASS | D:/rust/p03a-test.log | 9/9 incl. 11-point fault-injection matrix, backup restore, unowned refusal, coexistence, needs-recovery blocking |
| P03-T02 | P03 | cargo fmt --all -- --check (final) | Win11, user | PASS | D:/rust/p02c-fmt.log |  |
| P03-T03 | P03 | cargo clippy --all-targets --all-features -- -D warnings (final) | Win11, user | PASS | D:/rust/p02c-clippy.log |  |
| P03-T04 | P03 | cargo test --all-features --no-fail-fast (final) | Win11, user | PASS | D:/rust/p02c-test.log | 681 passed / 0 failed / 7 ignored |
| P03-T05 | P03 | cargo test --doc + cargo doc -D warnings (final) | Win11, user | PASS | D:/rust/p02c-test-doc.log, p02c-doc.log |  |
| P03-T06 | P03 | fault-matrix bug verification (backup-name case mismatch) | Win11, user | PASS (after fix) | D:/rust/p03a-test.log history | matrix caught journal-lowercase vs backup-original-case mismatch that lost original bytes on recovery; fixed and re-verified |
| P04-T01 | P04 | cargo test --all-features --lib msi (backend suite) | Win11, user | PASS | D:/rust/p04a-msi.log | 10/10 incl. 6 fake-runner tests: arg shape/no-/i, 1618 retry counts+delays, exhausted retries, 1603/1619/1620 diagnostics with log path, spawn failure, preference parse/resolve |
| P04-T02 | P04 | cargo fmt --all -- --check (final) | Win11, user | PASS | D:/rust/p02c-fmt.log |  |
| P04-T03 | P04 | cargo clippy --all-targets --all-features -- -D warnings (final) | Win11, user | PASS | D:/rust/p02c-clippy.log |  |
| P04-T04 | P04 | cargo test --all-features --no-fail-fast (final) | Win11, user | PASS | D:/rust/p02c-test.log | 686 passed / 0 failed / 7 ignored |
| P04-T05 | P04 | cargo test --doc + cargo doc -D warnings (final) | Win11, user | PASS | D:/rust/p02c-test-doc.log, p02c-doc.log |  |
| P04-T06 | P04 | real-msi msiexec /a acceptance | - | NOT_RUN | agent-work/PHASES/P04-msi-feasibility.md | opt-in T5; needs user-approved fixture store (spec B5/B6) |
| P05-T01 | P05 | cargo test --all-features --no-fail-fast (final) | Win11, user | PASS | D:/rust/p02c-test.log | 691 passed / 0 failed / 7 ignored; adds 5 pruning tests (profiles, TOML schema, execution, version scoping, negative fixtures) |
| P05-T02 | P05 | cargo fmt --all -- --check (final) | Win11, user | PASS | D:/rust/p02c-fmt.log |  |
| P05-T03 | P05 | cargo clippy --all-targets --all-features -- -D warnings (final) | Win11, user | PASS | D:/rust/p02c-clippy.log |  |
| P05-T04 | P05 | minimal profile compile/link acceptance | - | NOT_RUN | agent-work/PHASES/P05-pruning.md | opt-in T5: requires a real installed toolchain; minimal profile is rule-free by construction |
| P05-T05 | P05 | real-payload prune verification | - | NOT_RUN | agent-work/PHASES/P05-pruning.md | telemetry/arch rules validated on synthetic trees; real-MSVC verification is opt-in |
| P06-T01 | P06 | cargo test --all-features --no-fail-fast (final) | Win11, user | PASS | D:/rust/p02c-test.log | 697 passed / 0 failed / 7 ignored; adds 6 CAS tests (dedup, quarantine, GC dry-run/pins/retention/idempotence, re-runnable migration, digest validation) |
| P06-T02 | P06 | cargo fmt --all -- --check (final) | Win11, user | PASS | D:/rust/p02c-fmt.log |  |
| P06-T03 | P06 | cargo clippy --all-targets --all-features -- -D warnings (final) | Win11, user | PASS | D:/rust/p02c-clippy.log |  |
| P06-T04 | P06 | Windows sync_all discovery | Win11, user | PASS (after fix) | D:/rust/p02c-test.log history | sync_all on a read-only handle fails with Access Denied on Windows; partial-file fsync now opens in write mode |
| P07-T01 | P07 | cargo test --all-features --no-fail-fast (final) | Win11, user | PASS | D:/rust/p02c-test.log | 703 passed / 0 failed / 7 ignored; adds 6 lock-v2 tests (v1 compat, round-trip, provenance contradiction, schema rejection, mutation detection) |
| P07-T02 | P07 | cargo fmt --all -- --check (final) | Win11, user | PASS | D:/rust/p02c-fmt.log |  |
| P07-T03 | P07 | cargo clippy --all-targets --all-features -- -D warnings (final) | Win11, user | PASS | D:/rust/p02c-clippy.log |  |
| P07-T04 | P07 | flaky downloader test isolation runs (x2) | Win11, user | PASS | ssh transcript | a_corrupted_first_transfer passed twice in isolation; annotated as known parallel-load flake (pre-existing upstream test) |
| P08-T01 | P08 | cargo test --all-features --no-fail-fast (final) | Win11, user | PASS | D:/rust/p02c-test.log | 714 passed / 0 failed / 7 ignored; adds 11 resolver tests (chain, diamond, determinism, cycle, missing dep, 4 unknown-shape rejections, version conflict, prefix match, fragment+exclude seeds, profile fragments, TOML round-trip, realistic desktop set) |
| P08-T02 | P08 | cargo fmt --all -- --check (final) | Win11, user | PASS | D:/rust/p02c-fmt.log |  |
| P08-T03 | P08 | cargo clippy --all-targets --all-features -- -D warnings (final) | Win11, user | PASS | D:/rust/p02c-clippy.log |  |
| P08-T04 | P08 | stack overflow diagnosis + fix | Win11, user | PASS (after fix) | D:/rust/p08diag-status.log | bisected to #[tokio::main] embedding Plan arm futures on 1 MB main-thread stack; fixed by 16 MB linker stack + Box::pin |
| P09-T01 | P09 | cargo test --all-features --no-fail-fast (final) | Win11, user | PASS | D:/rust/p02c-test.log | 724 passed / 0 failed / 7 ignored; adds 9 detection tests (MSVC/SDK/build tools, false-positive, read-only, reuse states) |
| P09-T02 | P09 | cargo fmt --all -- --check (final) | Win11, user | PASS | D:/rust/p02c-fmt.log |  |
| P09-T03 | P09 | cargo clippy --all-targets --all-features -- -D warnings (final) | Win11, user | PASS | D:/rust/p02c-clippy.log |  |
| P10-T01 | P10 | cargo test --all-features --no-fail-fast (final) | Win11, user | PASS | D:/rust/p02c-test.log | 724 passed / 0 failed / 7 ignored; adds 4 provider tests (MSVC detect, version resolve, build tool detect, provider_id) |
| P10-T02 | P10 | cargo fmt --all -- --check (final) | Win11, user | PASS | D:/rust/p02c-fmt.log |  |
| P10-T03 | P10 | cargo clippy --all-targets --all-features -- -D warnings (final) | Win11, user | PASS | D:/rust/p02c-clippy.log |  |

## v2 fix round (R1–R7, gaps G-1..G-8) — unified final numbers

All commands on the school machine (Win11, cargo 1.100.0, MSVC 14.44.35207) via
scheduled task. G-1..G-4 landed on `main` (`34b662b`); G-5..G-8 on branch
`wip/v2-g5-g6-g7` (`ebf7d38` + `3a632fb`). This final run (`3a632fb`) is the
authoritative unified count that supersedes the per-phase numbers above.

| Test ID | Phase | Command / Test | OS / Privilege | Result | Evidence/log | Notes |
|---|---|---|---|---|---|---|
| V2-T01 | G-1..G-4 | `cargo test --all-features --no-fail-fast` @ 34b662b | Win11, user | PASS | D:/rust/g14-test.log | G-1 GC reparse skip, G-2 pins fail-closed, G-3 marker `extracted_any`, G-4 MSI SHA256 boundary. Final count recorded in D:/rust/g14-test.log (34b662b = 731 minus the 3 tests added later by G-5/G-6/G-7) |
| V2-T02 | G-5 | `cargo test --all-features --test transaction_tests real_process_crash_recovery_restores_original_bytes` | Win11, user | PASS | D:/rust/v2wip-test.log | real cross-process crash: helper child aborts at AfterInstall journal boundary, parent `recover_pending` restores original bytes, recovery idempotent |
| V2-T03 | G-6 | `cargo test --all-features` (path_safety unit + extractor) | Win11, user | PASS | D:/rust/v2wip-test.log | `check_no_reparse_within` covers staging root, planned_dirs, file out_path; directory-symlink detection test (skips cleanly if dev mode/elevation unavailable) |
| V2-T04 | G-7 | `cargo test --all-features --lib path_safety g7_trailing_separator` | Win11, user | PASS | D:/rust/v2wip-test.log | `dir/` strips one separator and resolves; `dir//` (empty component) and `dir/./` (`.` component) still rejected |
| V2-T05 | G-8 | `cargo test --all-features` (plan CLI) + `plan --no-cache` path | Win11, user | PASS | D:/rust/v2wip-test.log | `--no-cache` points manifest cache at a fresh temp dir; persistent cache untouched |
| V2-T06 | G-5..G-8 | `cargo fmt --all -- --check` | Win11, user | PASS | D:/rust/v2wip-fmt-check.log | exit 0 |
| V2-T07 | G-5..G-8 | `cargo clippy --all-targets --all-features -- -D warnings` | Win11, user | PASS | D:/rust/v2wip-clippy.log | exit 0 (after async fix to G-5 helper) |
| V2-T08 | G-5..G-8 | `cargo test --all-features --no-fail-fast` (final) | Win11, user | PASS | D:/rust/v2wip-test.log | **731 passed / 0 failed / 7 ignored** (lib 284 + bin 7 + integration 440) |
| V2-T09 | G-5..G-8 | `cargo test --doc` | Win11, user | PASS | D:/rust/v2wip-test-doc.log | 23 passed |

**Unified total: 731 passed / 0 failed / 7 ignored (network-gated) + 23 doc tests**
at tip `3a632fb`. Known upstream flake `a_corrupted_first_transfer_is_refetched_and_succeeds`
did not fail in this run (passes in isolation).
