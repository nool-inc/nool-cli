# Changelog

All notable changes to Nool are documented in this file.

## [Unreleased]

## [7.13.1] - 2026-09-29

Binaries: https://github.com/nool-inc/nool-cli/releases/tag/v7.13.1

`nool code` sessions that ride out provider hiccups, size their context to the
model, stay off other agents' files, follow repository playbooks and say up
front which approvals a change will need; plus more model providers and a
doom-loop stop. (7.12.0 and 7.13.0 were never published.)

### Added
- **More providers.** Anthropic, OpenAI, Google Gemini, Azure OpenAI and xAI, alongside OpenRouter, Mistral, AWS Bedrock, Cloudflare Workers AI and Ollama (`nool code provider add <id>`). `anthropic/claude-sonnet-5` still names an OpenRouter model; `anthropic:claude-sonnet-5` reaches the direct API.
- **Retries.** Rate limits, overloads, 5xx and timeouts are re-sent with backoff up to `[code] retry_attempts` (default 3); a context-length error compacts once and re-sends; anything else pauses with the reason.
- **Context sized to the model.** The primed context gets 1% of the window (1500 to 6000 tokens), or `[code] prime_budget_tokens`.
- **No writes over other agents' leases.** A write outside the declared scope to a file another agent holds is refused with the holder's name.
- **Playbooks.** `[code.playbooks.<name>]` with `steps` and `checks`; `nool code --playbook <name>` or `/playbook <name>`. The checks must pass before the goal lands.
- **Steering up front.** `[steer]` checkpoints appear in the session state, and the completion check names the approval landing will need.
- **Doom-loop stop.** The same edit, write or shell call three steps running pauses the session.

## [7.11.0] - 2026-09-24

Binaries: https://github.com/nool-inc/nool-cli/releases/tag/v7.11.0

A new `nool code` TUI: readable output, live status, a command palette, and a
view of the whole fleet. The governance screens (plan contracts, lease
conflicts, goal review) stay as they were, and the start-up splash is
unchanged and on by default.

### Added
- **Reading what happened.** Every overlay (`/diff`, `/help`, `/status`, `/mcp`, …) is now a scrollable, searchable pager (`/` to search, `n`/`N` to jump, `q`/Esc to close). Any tool's full output is one key away with Alt+↑/↓ and Enter. Assistant replies render as markdown, and diffs everywhere show old/new line numbers with word-level changes side by side from 120 columns.
- **Knowing what it is doing.** The status line shows the current phase, elapsed time, and stalls. `/savings` and the goal review show what Nool saved (context tokens avoided, output kept out of context, prompt-cache reads, tests skipped), each labelled as measured or estimated.
- **Getting around.** Ctrl+K opens a command palette over every command, skill and key. `/model` with no argument opens a picker with your recent models first. `/copy reply|code|diff|tool` copies over OSC 52 and the system clipboard.
- **Fleets.** A fleet strip in the chat TUI (Alt+N) shows one row per child with its phase, current tool and cost, and flags when a child edits inside another child's declared scope.

### Changed
- **Starting anywhere.** `nool code` runs from any subfolder of a Nool repository and keeps that subfolder as the session's focus.
- **Repository trust no longer blocks.** The `[y/N]` prompt before the TUI is replaced by a card inside it; the session starts with the repository's hooks off, and trust now records a fingerprint of the trusted commands and asks again when they change.
- **Redraws are adaptive.** No fixed tick: the TUI draws when something changes, and animations pause while the terminal is unfocused or you are typing.

## [7.10.0] - 2026-09-24

Binaries: https://github.com/nool-inc/nool-cli/releases/tag/v7.10.0

`nool code` grows into a full coding agent: an Agent Client Protocol server
for editors, voice mode, more model providers with a saved default model, and
notifications when a session's work is done.

### Added
- **`nool code acp`: an Agent Client Protocol (ACP v1) agent** over stdio (JSON-RPC 2.0, one message per line). Editors that speak ACP, such as Zed, run Nool as an agent with no other setup: `"agent_servers": {"Nool": {"command": "nool", "args": ["code", "acp"]}}`. Session steps stream as ACP updates (message and thought chunks, tool calls, the plan, mode and usage), and permission requests go to the editor for approval.
- **`nool code voice`: voice mode.** Speech-to-text and text-to-speech through local OpenAI-compatible endpoints (whisper.cpp for STT; Kokoro-82M or Speaches for TTS), with a zero-download fallback to the OS speech engine (`say` on macOS, `piper`/`espeak-ng` on Linux, `System.Speech` on Windows). Code blocks, diffs and formatting are stripped before anything is spoken. Verbs: `doctor`, `setup`, `speak`, `transcribe`, `config`; `/voice` in an interactive session.
- **More model providers.** Cloudflare Workers AI, AWS Bedrock and Mistral join OpenRouter and Ollama (`--backend openrouter|ollama|cloudflare|mistral|bedrock`, or `--model provider:model`). `nool code provider add|list|test|remove` stores keys in `~/.nool/credentials.toml` (mode 0600; an environment variable always wins) and checks them with one free request.
- **A saved default model.** `nool code model list|set|show` keeps the model the next runs use, globally or per repository (`set --repo`); `/model` switches it inside a session.
- **Notifications.** `nool code notify list|test|enable|disable|add|remove` sends a notice when a session's work is done, to sound, desktop, webhook (optionally HMAC-signed), Slack, Discord, Teams or a shell command. Configured in `~/.nool/code.toml` `[notify]`, or `/notify` in a session.
- **`nool code doctor`, `trust`, `mcp`, `plugin`.** See every MCP server, instruction file, skill, hook and plugin a session imports and why anything was skipped; trust a repository before its project-level MCP servers, hooks and plugins load; add catalog or custom MCP servers and sign in to OAuth ones (tokens go to the OS credential store); install Claude Code-format plugins.
- ASCII animations for session events in the interactive view.

### Changed
- **Licences are now signed by the Nool hub**, which also serves the installers. `curl -fsSL https://hub.nool.dev/install.sh | sh` installs the Community edition alongside the existing `https://www.nool.dev/nool-install.sh`.

### `nool code`: sandbox, tools and UX
- **Real toolchains work inside the sandbox.** A session's `shell` jail now writes the per-user temp and cache dirs your tools need (Xcode DerivedData, Swift and clang module caches, `~/.dart-tool`) and a shared, persistent build cache, `~/.nool/code-cache`, that Go, npm, pip, Gradle, Maven and `XDG_CACHE_HOME` point into. Checked against Rust, Go, Node, Bun, Python, uv, Swift, Xcode, C/C++, Java, Kotlin, Gradle, Maven, Ruby, Zig, Dart, Elixir and OpenTofu. Unix sockets work under the worktree and the private `$TMPDIR` (local test servers, a private ssh-agent), never host sockets such as Docker's. The completion check's test runs use the same jail.
- **Choose your sandbox level** with `[sandbox]` in `~/.nool/code.toml` (user-level only; a repository cannot widen its own jail): `write_paths = [...]` adds writable roots, `network = "off"` turns loopback off too, and `mode = "off"` runs commands unjailed. Loopback is on by default, so integration tests and build daemons (mix, gradle's daemon, sccache, `pytest-rerunfailures`) just work; the outside network stays off and Nool's console ports (4001, 4002) stay unreachable. `network = "localhost"` is still accepted as the default spelled out.
- **Refusals explain themselves.** A refused command ends with a `[sandbox]` note naming the writable roots and the setting that widens them, and the TUI shows the same warning. Denials start with `denied:` and say what to do instead; the agent looks for an allowed alternative, otherwise asks for exactly what it needs and keeps working on the rest.
- **`Stop` hooks no longer reopen a verified goal.** Once the completion check passes, a `Stop` hook still runs, but its block is reported as a notice and the engine lands the goal itself.
- **The agent can run `nool` commands.** A plain `nool …` line in `shell` runs as the session's agent: reads always, other commands (`task create`, `learn`, `bug report`, …) by mode, and commands that land or undo work (`solidify`, `try promote`, `checkpoint`, `push`, …) always ask. Previously every state-changing nool command was refused.
- **Nool-first orientation.** The agent reaches for Nool's context, grounding, blast-radius and findings tools before `grep`. `grep` skips hidden tool and editor directories; `list` counts hidden entries and shows them only with `hidden: true`.
- **Cleaner output.** Tool lines name the command and its result (`✗ shell xcodegen generate — exit 70`, `⊘` for refusals), and a successful command shows its last meaningful line instead of `exit 0`. `--exec` without an API key fails fast with the fix, and a rejected key says so. Budget, kept-branch and headless-verified outcomes name the next command.
- **TUI.** Slash commands and their arguments autocomplete with Tab (`/mode`, `/model`, `/attach` session ids, `/voice`, custom commands and skills). `/changes` (Ctrl+G) opens a change explorer: every file the session edited with A/M/D, +/− counts, its checkpoint, and the file's diff.

### Fixed
- `nool admin account activate`, `sync` and `trial` now work in any directory, not only inside a Nool repository, so a licence can reach a new machine before its first repo.
- `nool upgrade` keeps the edition you're running: a Team or Enterprise binary no longer downgrades to Community.
- Paying customers are no longer pointed at a trial. When your licence covers a higher edition than the installed binary, `activate`, `sync` and edition-only commands tell you to run `nool upgrade --edition <plan>`; a refused lease with a key present suggests `nool admin account sync`.
- A licence is no longer erased when a refresh is refused for a billing hiccup, such as a card being retried. Only an explicit revocation removes it; otherwise it simply runs to its normal expiry.
- Subscription changes that failed to apply are now retried instead of being skipped as duplicates.
- The installers now replace any older `nool` found elsewhere on your PATH (for example `/usr/local/bin` or `~/.cargo/bin`), so a stale copy can't shadow the new one. `scripts/install.sh --no-update-others` opts out.

## [7.9.2] - 2026-09-22

### Fixed
- `propose --as-agent <label>` was refused by the label's own lease on single-file proposals.
- A multi-file proposal is now checked against other agents' leases exactly as a single-file one is.

## [7.9.1] - 2026-09-22

### Fixed
- `nool code` checkpoints no longer name files `propose` refuses (a worktree's own `.nool/` state, collapsed untracked directories) in repositories whose ignore rules were never landed, and never write Python bytecode into the worktree.
- A test run that collected nothing now reads "ran no tests: unchecked" instead of failing the completion check or minting a passing `unit-tests` attestation. Skipped checks (config files, unknown types, TypeScript without a tsconfig, Go outside a module) say "Unchecked: no tests ran".

## [7.9.0] - 2026-09-22

### Added
- **`nool code`: a headless coding harness** (Community). `nool code --exec "<goal>"` runs a goal on a model through Nool's session engine in a `nool try` worktree under its own lease, and lands it only when Nool's completion check passes: a full-mode checkpoint, the affected tests run in a write/network jail, the task's acceptance criteria (`--task`), and try-impact readiness. It then promotes, records a signed `goal-complete` attestation and finishes the task. Exit codes: 0 verified, 2 blocked, 3 lease conflict, 1 paused or failed. `nool code --serve` runs the per-repo session API; `--list` lists sessions. Sessions can never write `.nool/` or `.git`, and raw git history verbs and state-changing `nool` verbs are refused.
- `nool debug blast-radius --json`, `nool try new --json` and `nool try promote --json`.
- **Lease holders are named.** A declared agent label is recorded with its leases: `nool announce status` shows `agent-a (1a2b3c4d)` and `nool discover conflicts` names the holder (`agent_label` in JSON).

### Changed
- `nool try promote` exits 3 on a merge conflict (`MERGE_CONFLICT`); `try discard` of a missing branch exits 1.
- Test selection is honest: package-scoped `cargo test -p`, an explicit run-all mode for unknown footprints, and zero collected Python tests never count as a pass.

### Fixed
- `propose` prechecked only the primary file of a multi-file proposal; every file is now prechecked, and `nool query validate` exits 2 on a syntax error.
- `nool init` no longer leaves a repository-wide lease behind when its import cannot seal (e.g. no git `user.name`/`user.email`).
- `propose --all` kept negated ignore rules.

## [7.8.1] - 2026-09-16

### Fixed
- `nool push` with steering enabled recomputed the steering rollup once per unpushed knot and could run for over an hour on a large backlog; it now computes it once per push.

## [7.8.0] - 2026-09-16

### Added
- **Incremental admission control.** The propose-time gate derives the blast envelope from the affected symbols instead of loading the whole graph, reports the contracts a change threatens (asserted facts, invariants, attestation obligations, steer-sensitive paths) in `propose --json`, and resolves a deterministic evidence plan for the change. Relational invariants are scoped to the proposal's region.
- **Per-stage timing:** `propose --json` emits `proposal_stages` with millisecond timings for each gate stage on propose and solidify.
- **A resumable `nool init`.** Each onboarding phase records its own completion, so a killed `init` resumes where it stopped, and every phase reports progress even when output is piped.

### Changed
- `[analysis] field_signal` (default off) takes whole-graph spectral analysis off the propose path; it remains in `debug blast-radius` and `insights`.

## [7.7.0] - 2026-09-15

### Added
- **Value receipts on human terminals.** `solidify`, `propose`, `try`, `work`, `task`, `checkpoint`, `blast-radius`, `doctor`, `merge`, `explain`, `insights`, `status`, `workspace` and `announce` show what landed, what it protected and what to do next. Agents, pipes and CI keep the compact form; `solidify`'s compact headline is now `outcome=ok knot_id=<id> title="…"`.
- Ghost runs build only the test targets that hold the selected tests and have their own timeout (`NOOL_GHOST_RUN_TIMEOUT_SECS`, 600 s floor); a timeout is reported as such, not as a failing change.

### Changed
- The seal trusts a Full validation `propose` already recorded instead of re-running it, cutting a one-file full-mode knot from minutes of redundant testing.

## [7.6.0] - 2026-09-14

### Added
- **Module-altitude architecture review** and **enforced module invariants:** accepted module assertions become `module_depends_on` / `module_must_not_depend_on` rules that gate proposals at the seal.
- A structural coupling metric (cross-module dependency ratio) for steering and dynamic quorum.
- Traceable `insights` health-grade deductions.

### Changed
- The seal graph, discovery and bootstrap skip generated, minified and vendored assets.
- Blast radius's semantic field is computed from the entity graph.

### Fixed
- Uncommitted architecture decisions on unbound subjects no longer collide on one identity.

## [7.5.0] - 2026-09-14

Binaries: https://github.com/nool-inc/nool-cli/releases/tag/v7.5.0

The dependency graph becomes current and symbol-granular. Supersedes 7.4.0,
whose archives were built but never released; its minisign pinning ships here.

### Added
- **A symbol-granular entity graph, maintained at the seal.** One sealed file is one unit of work: its content is re-parsed and the facts it contributes are reconciled against the graph. A file `Owns` each definition it holds, `Uses` the specific definitions it names in the files it imports, and an interface or base-class method `Calls` the same-named method of every implementing type. `nool query context <path>` reaches the symbols an agent actually asks about.
- **`nool debug blast-radius --symbol <name>`** lists the consumers of that one symbol rather than every importer of the file that declares it; `--symbol path#Name` anchors on one declaring file. A file the graph holds no symbol-level facts for falls back to file-level dependents and says so.
- **Symbol facts for every built-in code language.** Go same-package references, Swift same-module types, Elixir `alias` and Erlang `-import` / `mod:fun()` now produce edges, with exact-case probes on case-insensitive filesystems. Config and markup formats keep their file entity and import edges and contribute no symbols.

### Changed
- **Blast radius reports the dependents closure, not everything that landed later.** In a linear timeline the raw causal set was simply all subsequent history. Descendants are now narrowed to the knots touching the target or one of its dependents; the rest are counted and named as such.
- **A full graph rebuild is the per-file seal pass run over the tree.** `nool admin reindex-graph`, `nool init` and standalone `nool solidify` derive the same facts the seal does and retire what deleted files left behind, so an incrementally maintained graph and a rebuilt one agree.

### Fixed
- **The seal did not fence a sibling label's lease.** A proposal admitted under `--as-agent me` sealed straight through a lease announced as `--agent-id other` on the same file. A declared label now owns only itself, plus the environment's label when one is set.
- **Edges an edit removed survived until the next full rebuild.** A dropped import, a deleted definition and a retargeted `impl` now leave the graph with the knot that removed them, their history interval closed so an as-of query still sees them where they were true.
- **`nool query context` presented an echoed heading as a result.** An unresolvable target now says so and exits non-zero unless `--skeleton` or `--include-runtime` can answer from the file or the knot itself.
- **`nool query materialize` printed a bare header for any multi-file knot.** Every file of a Synthesis knot is now printed, and a knot carrying no file content says so.

### Distribution
- Community archives are published on `nool-inc/nool-cli` from 7.3.0 on; www.nool.dev/artifacts routes by version. Every release carries `SHA256SUMS` and `SHA256SUMS.minisig` (public key `RWT0vGBCEIz27HfLSal/dVFhklQJmgDGIIA9mq9O7MhtUdSiawHtyn9J`).

## [7.3.0] - 2026-09-10

Binaries: https://github.com/theswiftway/nool-cli/releases/tag/v7.3.0

The Community / Team / Enterprise split. This is the first release whose
archives carry the edition in the name: `nool-7.3.0-community-<target>`.

### Added
- **Three editions from one source.** Community (free forever, 1,000 knots a rolling month, one seat, one device), Team (fleets, council, steer, playbooks, announce/leases, personas, eval, PR summaries, exploration, GitHub/Jira import, MCP handoff) and Enterprise (Team plus workspace roll-ups, audit, commit templates, governance packs, contracts, plugin install, `console --serve`, air-gapped `nool.lic`). The paid surface is compiled out of Community, not switched off at runtime.
- **`nool version` and `nool status` name the edition** you are running.
- **Exit code 4 with an upgrade hint** instead of a usage error. `nool fleet` on Community prints ``fleet` is in Nool Team → nool admin account trial · nool.dev/pricing`; `--json` gives `{"outcome":"unavailable","reason_code":"EDITION_REQUIRED",…}`. Retrying unchanged never helps — treat 4 like 2 and 3.
- **`nool admin account trial [--email]`** takes an OTP-verified 30-day Team lease; **`nool admin team invite <email>`** and **`nool admin team list`** manage seats against the limit you bought (the hub returns 409 at N+1); **`nool upgrade --edition team|enterprise`** swaps in the licensed binary for your platform.

### Changed
- **A lapsed or revoked lease degrades to Community rather than refusing to run.** Your history stays readable and your working tree stays yours; previously a revoked licence was a hard stop indistinguishable from a corrupt install. Read paths — `status`, `log`, `query`, `context`, replay, blame — are never gated in any edition, and `task`/`thread`/`admin` lost their blunt trial checks because tasks are a Community floor.
- **A licence is bound to the machine that activated it.** Exceeding your device limit now fails activation with a clear message instead of warning and continuing.

### Fixed
- **`nool task list` no longer lists tasks replay cannot reach.** It read SQLite metadata directly while `task show`/`pick`/`cancel` resolve through the replay engine, so a repository carrying deferred history could show hundreds of tasks — some as InReview or Claimed — that then reported "Task not found".
- **`nool doctor` separates already-deferred history from newly-rejected knots**, so a familiar warning count no longer reads as a fresh regression.
- **`nool status` no longer advises merging a head replay will refuse.** A second head that is itself a `doctor:consolidate-heads` knot drew the generic "merge before release" line; merging it again only produces another rejected knot.

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
