# RESUME — v2 fix round state (written 2026-11-23 ~09:05 local)

Written because the driving session's model quota was exhausted mid-round. Read this first
when resuming; it is the authoritative state of the v2 fix round.

## Where things stand

| Item | State |
|---|---|
| main @ `34b662b` | G-1..G-4 fixed, pushed, fork CI green (Code Coverage in_progress at write time) |
| WIP branch `wip/v2-g5-g6-g7` | unverified in-flight work for G-5 + G-6 (see below), pushed for preservation — **never merged, never compiled** |
| PR #219 (`feat/staged-installer-rework`) | open, still at pre-fix code (`377ce8e`), waiting on maintainer to approve workflow runs (`action_required`) |
| G-5 / G-6 | partial: crash-test harness skeleton + `check_no_reparse_within` helper + 2 call sites, **not compiled, not tested** |
| G-7 / G-8 | not started |
| Report numbers | still inconsistent (FINAL_REPORT 714, TEST-RESULTS 724, measured 727) — must be regenerated from the post-fix tip's full log |

## Gap list (source of truth: agent-work/BASELINE-v2-GAPS.md)

G-1 GC symlink (P0) — DONE in 34b662b
G-2 pins fail-closed (P0) — DONE in 34b662b
G-3 marker bool / mixed cache (P1) — DONE in 34b662b
G-4 MSI SHA256 trust boundary (P1) — DONE in 34b662b
G-5 real crash recovery test (P1, R4) — WIP on wip/v2-g5-g6-g7
G-6 reparse-point write protection (P1, R7) — WIP on wip/v2-g5-g6-g7
G-7 trailing-separator tests (P2, R2) — TODO
G-8 plan --no-cache (P2, R6) — TODO
G-9 snapshot pinning — no code change (already in v2 spec)

## Next actions, in order

1. Resume on the WIP branch, do NOT merge it as-is: run it on the school machine until it compiles
   (`cargo test --all-features --no-fail-fast`), then finish G-5 honesty rules:
   - helper child + kill at journal boundary + independent recovery process
   - any boundary that cannot be injected reliably → mark BLOCKED, never claim crash-safe
2. G-6: verify the reparse check covers every write site (extractor.rs VSIX + CAB `create_dir_all`,
   the `planned_dirs` loop, and the staging root itself); add a Windows junction test.
3. G-7: `dir/` positive, `dir//` and `dir/./` negative tests.
4. G-8: `plan --no-cache` that truly leaves the manifest cache untouched (temp dir fetch).
5. Full verification: fmt + clippy -D warnings + full test on school machine, then push, then fork CI green.
6. Regenerate report numbers from the final log; update FINAL_REPORT.md and TEST-RESULTS.md so both
   state the same count and the same tip sha. Record the known upstream flake
   (`a_corrupted_first_transfer_is_refetched_and_succeeds`, passes in isolation) as environmental.
7. Only then bring the fixes onto `feat/staged-installer-rework` (cherry-pick) so PR #219 shows the
   corrected code, and post a short PR comment listing the five review findings and their fixes.
   Ask the user before pushing PR-branch changes.

## Environment cheat-sheet (unchanged)

- Edit copy: `S:\apps\llm\agentwork\msvc-kit-rs`; specs in `spec/` (v1) and `spec/v2/` (authoritative).
- Never compile locally. School machine (Win11-HV1) only:
  `echo '<cmds>' | MSYS_NO_PATHCONV=1 ssh school-guest-pc2 "D:/yzd/MSYS2/usr/bin/bash.exe --login -s"`
- School clone: `/d/msvc-kit-rs` (build/test only, never commit there).
- Long jobs MUST go through schtasks (ssh dies on long compiles):
  `export MSYS2_ARG_CONV_EXCL="*"; schtasks /create /tn msvc-kit-run /tr "D:\rust\cargo-run.cmd <args>" /sc once /st 23:59 /it /f && schtasks /run /tn msvc-kit-run`
  then poll `/d/rust/cargo-last.log` until an `EXITCODE=` line appears; delete the task afterwards.
- Push: `env -u GITHUB_TOKEN git -c http.proxy=http://127.0.0.1:7897 push origin main`
  (without `env -u` the machine's GITHUB_TOKEN causes 403).
- Fork CI trap: "Code Coverage" fails only at "Generate coverage report" (no codecov token) — its test
  step green is the signal. "Release" workflow is disabled on the fork; do not re-enable it.

## Operational note on this round

The driving agent was repeatedly killed by model-provider rate limits/retry storms (429/1005), and each
restart confirmed state from disk. Nothing was lost; both the P00 gap list and the partial G-5/G-6 work
are now preserved in git.