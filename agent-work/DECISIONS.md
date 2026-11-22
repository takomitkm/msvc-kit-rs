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
