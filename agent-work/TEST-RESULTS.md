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
