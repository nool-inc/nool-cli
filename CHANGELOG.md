# Changelog

All notable changes to Nool are documented in this file.

## [Unreleased]

## [7.2.0] - 2026-09-06

Binaries: https://github.com/theswiftway/nool-cli/releases/tag/v7.2.0

### Added
- **`nool fleet capacity`**: report the wave-concurrency ceiling for this machine and budget without planning anything — each probe's width (CPU, memory, budget), which one binds, and the `[fleet]` settings that produced it.
- **`--width auto`** on `nool fleet plan`, now the default. Derived from usable cores (minus `[fleet] reserved_cpus`), available memory over `memory_per_agent_mb`, and the remaining `budget_usd_per_day` over the builder's `per_run_usd`. Precedence is `--width` > `[fleet] width` > `auto`. The `Ceiling:` line and the `--json` `width_decision` name the binding constraint.
- **Provider rate limits as a capacity dimension**: `[fleet] provider_requests_per_minute` and `provider_tokens_per_minute`, divided by what one agent draws. Undeclared limits do not bind.
- **`[fleet] launch_stagger_ms`** (default 250) rate-limits the ramp so a wave's startup allocations do not land in the same instant.

### Changed
- **Admission control**: capacity is re-measured before every wave rather than once at the start, since memory moves while a fleet runs. A pinned `--width` shapes the plan but does not raise admission.
- **The executor declares what a unit of concurrency costs.** Vendor-CLI backends are subprocesses with their own Node/V8 heap and declare ~512 MiB; the in-process `host` backend declares negligible, removing memory from the ceiling entirely.
- `fleet plan --width` was previously a hardcoded 8 — the same number on a four-core laptop as on a 64-core builder.
- **Documentation synced to the 7.2.0 command surface**: `Skills.md` and `skills/nool-commands/{SKILL.md,Commands.md}` are now the authored reference shipped with the CLI (previously a separate, stale 5.9.1 write-up); `README.md`, `CLAUDE.md` and `docs/index.html` state 7.2.0. `Commands.md` is added here for parity with the published docs.

### Fixed
- `nool query context` traverses structural and semantic edges, not `DependsOn` alone. A file reaches the symbols it defines by `Owns`, so a file-level query returned the file and nothing else while the symbol-level query answered correctly.
- Entity edges are stored one row per fact: repeated discovery passes no longer inflate the graph with duplicates, and a reindex no longer drops entities it should have kept.
- Lease supersession is applied correctly.
- `nool admin plugin init` scaffolds against the SDK vendored in the release archive, searching upward for `sdk/`, instead of emitting a `crates.io` dependency that cannot resolve (Nool's crates are `publish = false`).

## [3.2.0] - 2026-05-24

### Changed
- **Documentation Sync**:
  - Updated `README.md`, `Skills.md`, `CLAUDE.md`, and `skills/nool-commands/SKILL.md` to match the installed `Nool CLI v3.2.0` command surface.
  - Corrected stale examples for `nool announce intent`, `nool thread create --name`, `nool thread status --name --status`, `nool task pick --id`, `nool task finish --id`, and `nool debug bisect --bad`.
  - Removed outdated references to proposal and discovery flags that no longer match the generated help output.

## [2.3.5] - 2026-05-22

### Added
- **Manual Plugin & Policy Management**:
  - Added `nool admin plugin install <path> [--policy]` to allow manual installation of WASM plugins and policies.
  - Added `nool admin plugin uninstall <name> [--policy]` to allow manual uninstallation.
  - Added `nool admin plugin list` to view all active governance and language plugins.
- **Multi-Stage Lifecycle Hooks**:
  - Implemented `PrePropose`, `PostPropose`, `PreSolidify`, and `PrePush` stages in the Aram governance substrate.
  - Aram policies now receive the current `LifecycleStage` via the updated WIT interface.
  - Added **Installation Hints**: Automated hints for missing hook-required policies.
- **Agent Persona Hardening**:
  - `nool propose` now supports non-blocking **Agent Auto-Justification** when blast-radius warnings are triggered.
  - Added `--justification` flag for manual causal reasoning during proposals.
  - Improved persona prioritization from `nool.toml`.

## [2.2.4] - 2026-05-21

### Added
- **Source-Derived Thread Summaries**:
  - `nool thread show <name> --full` now includes touched paths, directory footprint, AST-aware public API hints, and dependency signals.
  - Added **Internal Dependency Map** to visualize relationships between files touched within a thread.
  - Added **Transitive Dependency Closure** (via `ImpactAnalyzer`) to identify contextually related files outside the current thread.
  - Upgraded `ImpactAnalyzer` with language-specific resolution heuristics (Rust crates, group imports, naming conventions).
- **Git Index Escape Hatch**:
  - Added `nool untrack <path>...` as the Nool-native replacement for `git rm --cached`.
- **Binary Size Optimization**:
  - Implemented aggressive size reduction strategy: Fat LTO, single codegen units, size-optimized levels ('z'), and abort-on-panic behavior.
  - Pruned heavy dependency features in `tokio`, `wasmtime`, `lancedb`, and `async-stripe`.
  - Binary size reduced from 166MB to ~115MB.
- **Multi-Platform Release Support**:
  - Enhanced release scripts with cross-platform `sed` compatibility and support for multiple Rust targets.
  - Automated packaging for macOS, Linux, and Windows with SHA256 checksum generation.

### Changed
- **Proposal Ergonomics**:
  - Added `nool propose --all` to stage modified, deleted, staged, and untracked Git worktree paths without repeating `--path`.

## [1.31.0] - 2026-05-11

### Added
- **Auto-Sync Background Daemon**:
  - `nool bridge watch` now spawns a detached background process for continuous replication.
- **Ephemeral Branching (nool try)**:
  - Verified and stabilized `nool try` for safe, isolated experimentation.
