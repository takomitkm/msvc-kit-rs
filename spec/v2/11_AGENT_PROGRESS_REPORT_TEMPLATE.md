# Agent 进度、决策与测试报告模板

Agent 必须持续更新，不要只在最终阶段写报告。

## 1. BASELINE.md 模板

```markdown
# Baseline Audit

- Date/time:
- Repository path:
- HEAD SHA:
- Branch:
- Git status:
- Rust version:
- Cargo version:
- OS:
- Canonical CI command:

## Relevant files and symbols
| Feature | Path | Symbol | Existing tests | Status | Evidence |
|---|---|---|---|---|---|

## Current install flow
[从 CLI/API → Manifest → Resolve → Download → Cache → Extract → receipt → env 的真实调用路径]

## Target root layout
[列出实际目录，并区分 download cache / installed files / receipts]

## Baseline test results
| Command | Exit | PASS/FAIL/BLOCKED | Log | Notes |
|---|---:|---|---|---|

## Risks
[按 P0/P1/P2]

## Unknowns
[没有证据的事项]
```

## 2. DECISIONS.md 模板

```markdown
# Engineering Decisions

## DEC-001: <title>
- Date:
- Phase:
- Context:
- Options considered:
- Decision:
- Why:
- Trade-offs:
- Evidence/tests:
- Revisit condition:
```

所有偏离规格的设计必须记录决策和理由。

## 3. PROGRESS.md 模板

```markdown
# Progress

## Current phase
Pxx — title

## Status
NOT_STARTED / IN_PROGRESS / BLOCKED / COMPLETE

## Completed
- [x] ...

## Remaining
- [ ] ...

## Files changed
- `path`: reason

## Tests
- PASS: command
- FAIL: command + reason
- BLOCKED: command + missing environment

## Risks / regressions
...

## Next exact action
[写出下一步要改哪个文件/添加哪个测试，不写模糊愿望]
```

## 4. TEST-RESULTS.md 模板

| Test ID | Phase | Command/Test | OS/Privilege | Result | Evidence/log | Notes |
|---|---|---|---|---|---|---|

结果只能为：
- `PASS`
- `FAIL`
- `BLOCKED_ENVIRONMENT`
- `NOT_RUN`

不得把 `NOT_RUN` 或 `BLOCKED_ENVIRONMENT` 统计为 PASS。

## 5. 每阶段报告模板

```markdown
# Phase Pxx Report

## Scope

## Baseline code findings
- path/symbol:
- existing behavior:
- test evidence:

## Changes
- path:
- changed symbol:
- behavior:

## Tests added
- test name:
- bug it detects:
- why it would fail before change:

## Commands executed
| Command | Exit | Result | Output/log |
|---|---:|---|---|

## Compatibility
- CLI:
- Rust API:
- JSON/schema:
- Windows behavior:

## Security review
- target path:
- path traversal:
- ownership:
- rollback:
- privilege:

## Unresolved items

## Exit Gate
- [ ] all criteria checked

## Decision
COMPLETE / BLOCKED / NEEDS_REVISION
```

## 6. FINAL_REPORT.md 模板

```markdown
# Final Engineering Report

## Summary

## Delivered phases

## Not delivered

## Architecture changes

## Public CLI/API/schema changes

## Attribution / license changes

## Test matrix
| Layer | OS | Command | Passed | Failed | Blocked | Not run |
|---|---|---|---:|---:|---:|---:|

## Fault injection summary
- total points:
- passed:
- failed:
- blocked:

## Real Windows/MSI acceptance
- MSI used:
- source/license:
- Windows version:
- privilege context:
- msiexec args:
- exit code:
- log path:
- output validation:
- UAC observed:

## Known risks

## Manual actions needed from user

## Exact git diff summary
```
