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
