# P04 — MSI Backend Feasibility Report (native Rust MSI extraction)

Status: **native backend NOT enabled**; `msiexec /a` remains the only active backend. The parity gate defined in spec 05 §3 cannot be met in this repository today, so per the same spec ("如果这些条件无法满足，停止 native extractor 实现，仅交付 backend abstraction + 可行性报告") only the abstraction, the hardened wrapper, and this report are delivered.

## 1. What P04 delivered instead

- `MsiBackendPreference` (`auto` / `msiexec` / `native` / `fallback`) — public, additive; `native` and `fallback` resolve to an explicit parity-gate error instead of silently pretending (spec roadmap: "native 显示 unsupported/experimental").
- `MsiProcessRunner` trait (`run`/`sleep`/`max_retries`) with `SystemRunner` production implementation — the fake-runner seam spec 08 §8 demands.
- `MsiexecBackend` (Windows): argument-array invocation (`/a <msi> /qn TARGETDIR=<dir> /L*V <log>`), verbose log beside the source MSI, configurable 1618 retry budget with recorded delays, exit-code-specific guidance (1603/1619/1620) including the log path, spawn-failure reporting, no `/i`, no shell, no elevation request.
- `extract_msi_sync` now delegates to the backend (lock and progress handling unchanged), removing the duplicated retry loop.
- 6 fake-runner tests: argument shape + no-`/i` assertion, 1618 retry counting and delay recording, exhausted retries, 1603/1619/1620 diagnostics, spawn failure, preference parsing/resolution.

## 2. Candidate crate evaluation

| Crate | Version (2026-11) | License | MSRV | Maintenance | Verdict |
|---|---|---|---|---|---|
| `msi` (rust-msi) | 0.8.x | MIT/Apache-2.0 | low | updates sporadic; MSI database read/write (CFB container, Directory/Component/File/Media tables) | **Spike-only**: database reading is real, but the crate does not implement the full extraction pipeline — CAB sequence mapping (Media.LastSequence vs File.Sequence), embedded `#cabinet` streams, short/long name resolution and feature/condition filtering would all be new code on top |
| `cab` | 0.6 | MIT/Apache-2.0 | low | maintained (already a dependency) | Cabinet decompression works and is already used for direct CAB payloads; cabinet *selection* per MSI Media table would be new logic |
| others (e.g. `cfb` crate alone) | — | — | — | container-level only | insufficient alone |

Key risk: spec 05 §3 item 12 forbids pretending to support per-component extraction ("初期可先提取整个安全的 administrative image"); items 6/7 (embedded CABs + sequence mapping) are exactly where naive implementations break, and MSVC/SDK MSIs exercise them heavily.

## 3. Parity gate — unmet conditions

1. **Legal fixtures**: the repository must not commit Microsoft MSI/CAB payloads (spec 10 §2-3); real fixtures require a user-approved local opt-in store (spec B5/T5). None exists.
2. **Two independent SDK/MSI versions** must pass comparison against `msiexec /a` output (file set, sizes, hashes).
3. **Adversarial cases** (long paths, Unicode, short-name|long-name mapping, missing CAB, wrong CAB hash, malicious paths) need crafted fixtures that only make sense against real MSIs.
4. The school-machine environment provides MSVC 14.44 + SDK 10.0.26100 payloads that could serve as fixtures if the user approves a one-time opt-in download — this requires user approval per the standing instructions and has not been requested.

## 4. Enablement criteria (for a future phase)

1. User-approved real fixture store (two SDK/MSI versions, legal provenance recorded).
2. Implement backend: CFB parse via `msi` crate → Media/File/Directory table walk → per-file CAB selection via Sequence/LastSequence → `cab` extraction → short/long name resolution → administrative-image layout under the staging root.
3. Parity suite vs `msiexec /a` (file set + sizes + SHA256 per file), adversarial matrix, Windows-only run, two fixture versions.
4. Only then flip `MsiBackendPreference::Native` from parity-gate error to real implementation; `Auto` stays `msiexec` until the suite is green in CI.

## 5. UAC / standard-user safety test plan (opt-in, T5)

Documented per spec 05 §4 / 08 §8; requires user approval and a dedicated machine (stop-and-report condition applies):
1. Standard user account runs `msiexec /a` against `%LOCALAPPDATA%` staging for an approved fixture; assert no UAC prompt, no product registration.
2. Admin account, non-elevated token: same run; assert the tool itself never calls `runas` (checked by the wrapper's argument shape and process observation).
3. A fixture with an AdminExecuteSequence custom action (if any is legally available): expect failure with the exit-code report, no elevation attempt, log preserved.
4. Never run in CI; interactive sessions only, with timeouts.
