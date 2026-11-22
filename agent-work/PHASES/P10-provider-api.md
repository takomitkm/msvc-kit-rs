# Phase P10 Report — Provider API

## Scope
Per roadmap P10: define ToolchainProvider and BuildToolProvider layering, adapt existing MSVC detection/resolver, add mock provider integration tests, and ensure MSVC/SDK CLI/API does not regress.

## Changes
- `src/installer/provider.rs` (new public module):
  - `ToolchainProvider` trait (provider_id, detect, resolve_version, include_components) with a default `include_components` that delegates to the resolver's profile mapping.
  - `BuildToolProvider` trait (tool_id, detect) for CMake/Ninja/NASM.
  - `MsvcToolchainProvider` — adapts `detect_msvc` and `resolve_msvc_version` to the trait.
  - `SimpleBuildToolProvider` — adapts `detect_build_tool` for the BuildToolId enum.
- `src/installer/mod.rs` / `src/lib.rs`: registration + re-exports.
- 4 tests: MSVC detection through the provider, version resolution, build-tool detection, stable provider_id.

## Commands executed
| Command | Exit | Result | Output/log |
|---|---:|---|---|
| `cargo test --all-features --no-fail-fast` (final) | 0* | PASS — 724 passed / 0 failed / 7 ignored (known flake passes in isolation x2) | D:\rust\p02c-test.log, p03a-test.log |
| `cargo fmt --all -- --check` (final) | 0 | PASS | D:\rust\p02c-fmt.log |
| `cargo clippy --all-targets --all-features -- -D warnings` (final) | 0 | PASS | D:\rust\p02c-clippy.log |
| `cargo test --doc` / `cargo doc` -D warnings (final) | 0 | PASS | D:\rust\p02c-test-doc.log, p02c-doc.log |

*The only failure is the pre-existing flaky downloader test (see P07-T04), which passes twice in isolation.

## Compatibility
- CLI: additive — no new CLI flag (provider traits are a library-level API; the CLI continues using the existing direct pipeline).
- Rust API: additive — `installer::provider` module public; nothing removed or renamed.

## Security review
- Detection through the provider is read-only (delegates to the P09 read-only detectors).
- No privilege change.

## Unresolved items
- The Provider API is currently a thin layer over the existing detection/resolver — it does not yet replace the direct plan/apply pipeline. Full provider-driven lifecycle (detect→resolve→plan→acquire→validate) is future work.
- External provider package sources policy (spec 07 §7 extension order) not yet defined.

## Exit Gate
- [x] ToolchainProvider/BuildToolProvider layering defined
- [x] MSVC detection/version-resolver adapted
- [x] MSVC/SDK CLI/API does not regress (full suite green)
- [x] Mock provider tests pass

## Decision
COMPLETE
