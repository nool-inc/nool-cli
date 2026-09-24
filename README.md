# Nool

[![Version](https://img.shields.io/badge/version-7.10.0-blue.svg)](https://github.com/nool-inc/nool-cli/releases/latest)
[![Status](https://img.shields.io/badge/status-production-green.svg)](https://github.com/theswiftway/nool-cli)

Nool is a version control system engineered for the era of AI-authored code. It acts as the control plane between an AI coding agent's intent and the codebase it modifies, ensuring changes are intentional, inspectable, and governed before they become canonical.

Get started with the free Community edition: 1,000 knots a month, one engineer, no card, no signup.

## Core Capabilities

Nool provides agentic change control for AI coding-agent workflows:

- **Intent Tracking**: Record engineering intent before code acceptance.
- **Blast Radius Analysis**: Compute the semantic impact of every change.
- **Semantic Impact Envelope Enforcement**: Flag modifications that exceed declared scope.
- **Causal Justification**: Require evidence when implementation exceeds the declared envelope.
- **Policy and Security Gates**: Validate changes before they are accepted into the repo.
- **Durable State**: Preserve intent, impact, findings, and justifications alongside the code.

Git remains the ecosystem compatibility and storage layer. Nool governs the evolution of AI-authored code.

## Install

Every installer below puts the **Community** edition on your PATH: free forever, one engineer, 1,000 knots a month, no email and no card. The binary is called `nool` in every edition; `nool version` prints which one you have.

**macOS**
```bash
curl -fsSL "https://www.nool.dev/nool-install.sh?utm_source=homepage&utm_medium=copy_button&utm_campaign=install" | bash
```
**linux**
```bash
curl -fsSL "https://www.nool.dev/nool-install.sh?utm_source=homepage&utm_medium=copy_button&utm_campaign=install" | bash
```
**windows**
```bash
irm "https://www.nool.dev/nool-install.ps1?utm_source=homepage&utm_medium=copy_button&utm_campaign=install" | iex
```

From 7.10.0 the Nool hub also serves the installer directly (macOS and Linux); both paths install the same Community build:
```bash
curl -fsSL https://hub.nool.dev/install.sh | sh
```

Release archives are named `nool-<version>-community-<target>.tar.gz` (`.zip` on Windows) and ship with a `.sha256` next to each archive plus a run-wide `SHA256SUMS` manifest; the installer verifies the archive against its checksum before extracting. The manifest is minisign-signed (`SHA256SUMS.minisig`, public key `RWT0vGBCEIz27HfLSal/dVFhklQJmgDGIIA9mq9O7MhtUdSiawHtyn9J`); the installers verify it when `minisign` is present, and `NOOL_REQUIRE_SIGNATURE=1` makes that mandatory. Team and Enterprise archives follow the same naming (`-team-`, `-enterprise-`) but are not published here: the licence hub serves them to subscribers.

### Upgrading to Team or Enterprise

A Community binary answers Team and Enterprise commands (`nool fleet`, `nool announce`, `nool workspace`, ...) with an upgrade hint and **exit code 4**; nothing else changes. To move up:

```bash
nool admin account trial                  # 30-day Team trial, no card: a lease arrives by email
nool admin account activate <key>         # activate a purchased Team or Enterprise lease
nool upgrade --edition team               # download and install the Team binary for your platform
nool upgrade --edition enterprise         # same for Enterprise (contract customers)
```

Leases are signed tokens that verify offline; the hub only issues and renews them. When a Team lease lapses the binary degrades to Community rather than locking you out. Buy Team seats at https://www.nool.dev/pricing; Enterprise is by contract at https://www.nool.dev/contact.

## Licensing

Nool is commercial software. Community is free forever. Enterprise contracts include source escrow, so if Nool ever stops trading or stops supporting the product, you receive the source.

- **Community** (this download): free forever. One engineer, 1,000 knots a month, core VCS, semantic intelligence and tasks. Governed by [LICENSE](./LICENSE).
- **Team**: $49 per engineer per month billed annually ($59 monthly), 3-50 seats. Self-serve at https://www.nool.dev/pricing, 30-day trial without a card.
- **Enterprise**: from $89 per engineer per month billed annually, 50-seat minimum. Source escrow, offline and air-gapped licences. Contact sales at https://www.nool.dev/contact.

## Quick Links

- [Skills.md](./Skills.md): Full CLI command reference for the installed surface.
- [skills/nool-commands/SKILL.md](./skills/nool-commands/SKILL.md): Agent-optimized Nool skill file.
- [skills/nool-commands/Commands.md](./skills/nool-commands/Commands.md): The same command reference, alongside the skill so an agent can load both.
- [CHANGELOG.md](./CHANGELOG.md): What changed in each release.
- [Releases](https://github.com/nool-inc/nool-cli/releases): Signed archives for macOS, Linux (glibc/musl) and Windows, each with a `.sha256`.
- [docs/index.html](./docs/index.html): Static documentation landing page.

## Adopting Nool as Your VCS

Nool can absorb an existing Git history and lift the current project structure into the semantic ledger:

```bash
nool init --from-git main
nool discover features
nool discover lift --solidify
nool status --compact
```

If you are starting fresh, run `nool init` without `--from-git`.

## Core Workflow

Use this sequence for most code changes:

```bash
nool status --compact
nool discover features
nool announce intent --intent "Refine command documentation"
nool work start --intent "Refresh docs from installed CLI"
nool task create --name "Update docs for Nool 7.10.0" --solidify
nool propose --all --intent "Refresh docs from installed CLI" --fast
nool solidify
```

For planned state transitions and safer staged execution:

```bash
nool plan replay --target <op_ids>
nool review <plan_id>
nool apply --plan-id <plan_id>
```

## Command Map

### Core Version Control

- `nool init`: Initialize a Nool ledger and identity in a repository.
- `nool propose`: Generate a candidate Knot from file changes and intent.
- `nool solidify`: Sign and append the candidate Knot to the DAG.
- `nool reify`: Inspect a reified bundle and validate syntax before solidifying.
- `nool plan`: Create semantic replay, pluck, merge, and rebase plans.
- `nool apply`: Execute an approved or draft semantic plan.
- `nool verify`: Run structural invariants against current or planned state.
- `nool explain`: Explain identities, dependencies, and reasons for a semantic object.
- `nool evidence`: Show why an AI-authored transition was accepted or rejected.

### Sync, History, and Release

- `nool push`: Replicate unpushed Knots to a remote replica.
- `nool pull`: Fetch and replay Knots from a remote replica.
- `nool sync`: Run bidirectional semantic sync.
- `nool log`: Show the canonical replay log.
- `nool diff`: Show file-content diffs between two Knots.
- `nool changelog`: Generate a semantic changelog.
- `nool tag`: Create a semantic tag.
- `nool checkpoint`: Mark the current state as a checkpoint or release label.
- `nool approve`: Approve a Knot or intent thread.
- `nool promote`: Promote a local Knot to staged or synced status.

### Discovery and Inspection

- `nool status`: Repository health, DAG state, licensing, and pending proposals.
- `nool doctor`: Release-readiness and repository health checks.
- `nool dag`: Visualize the DAG.
- `nool visualize`: Visualize history, graphs, ROI, and relational artifacts.
- `nool why`: Walk the causal chain of a change.
- `nool query`: Run semantic queries over the Knot DAG.
- `nool discover`: Restore context, find conflicts, discover features, and lift them into Knots.
- `nool insights`: Show project insights, blast radius stats, and time-saved metrics.
- `nool review`: Open the interactive review surface for candidate changes.
- `nool audit`: Generate intent, authorship, and release compliance reports.

### Threads, Tasks, and Knowledge

- `nool work`: Start a new piece of work, with optional parallel subtasks.
- `nool thread`: Manage intent threads.
- `nool task`: Manage task lifecycle.
- `nool inbox`: Open the unified notification center.
- `nool learn`: Record a knowledge finding, dependency insight, or reasoning note.
- `nool findings`: Retrieve recorded findings for a file, thread, or topic.
- `nool link`: Retroactively link a solidified Knot to intent or thread metadata.
- `nool announce`: Coordinate work across multiple agents before edits begin.
- `nool bug`: Report, link, list, and inspect bugs.

### Workspaces

- `nool workspace status`: Show the project tree and dependency order.
- `nool workspace doctor`: Reconcile declared workspace config against discovered projects.
- `nool workspace goal`: Decompose or absorb cross-project goals.
- `nool workspace goals`: List persisted workspace goals and rollups.
- `nool workspace goal-status`: Show completion status across projects.
- `nool workspace insights`: Aggregate insights across child projects.
- `nool workspace pull`: Run `nool pull` across the workspace in dependency order.

### Runtime, Governance, and Administration

- `nool bridge`: Manage the Git Bifrost bridge and LFS integration.
- `nool daemon`: Launch the background sync daemon.
- `nool console`: Launch the web console and v5.0 control-plane API.
- `nool ui`: Launch the interactive TUI DAG explorer.
- `nool untrack`: Stop tracking files in Git while keeping them locally.
- `nool validate`: Run background validation for quarantined fast-mode Knots.
- `nool admin`: Account, team, plugin, and billing administration.
- `nool config`: Show and manage effective system configuration.
- `nool languages`: List supported languages and validator availability.
- `nool usage`: Show token budgets and agent performance metrics.
- `nool prune`: Clean temporary and cached files.
- `nool migrate`: Migrate Nool-generated files into the canonical layout.
- `nool upgrade`: Upgrade the CLI.
- `nool uninstall`: Remove the CLI and local identity keys.
- `nool completion`: Generate shell completion scripts.
- `nool quick-start`: Show the beginner quick-start guide.
- `nool guide`: Show the detailed command guide.
- `nool inquiry`: Inspect the Inquiry Tree of directions, evidence, and distilled insights.
- `nool council`: Run the configured multi-model council over the working-tree diff.
- `nool agent`: Inspect declarative agent specs.
- `nool fleet`: Plan and run fleets of sovereign agents over a goal; `nool fleet capacity` reports how wide this machine and budget allow, and which limit binds.
- `nool harness`: Report health of the swappable model backends.
- `nool soul`: Create and manage persistent model personas.
- `nool enrich`: Recall knowledge for a query and perform bounded enrichment on misses.

### Coding Sessions (`nool code`)

- `nool code --exec "<goal>"`: Run a goal headless on a model in a try branch; it lands only when Nool's completion check passes (exit 0 verified, 2 blocked, 3 lease conflict, 1 paused or failed).
- `nool code acp`: Run Nool as an Agent Client Protocol agent over stdio for editors such as Zed.
- `nool code provider` / `nool code model`: Set up OpenRouter, Ollama, Cloudflare Workers AI, AWS Bedrock or Mistral keys, and save the default model.
- `nool code voice`: Local speech-to-text and text-to-speech, with an offline OS-speech fallback.
- `nool code notify`: Sound, desktop, webhook, Slack, Discord, Teams or command notifications when a session's work is done.
- `nool code doctor` / `trust` / `mcp` / `plugin`: Inspect what a session imports, trust a repository's MCP servers and hooks, manage MCP servers and plugins. A `Stop` hook never reopens a goal that already passed its completion check; its block is shown as a notice and the goal lands.
- Sandbox: a session's shell runs in a jail that already fits real toolchains (Xcode, Swift, Go, Gradle, Maven, Dart and more, with a shared build cache in `~/.nool/code-cache`). Loopback is on by default, so local test servers and build daemons work; the outside network stays off. Widen or tighten it only from `~/.nool/code.toml`; a repository cannot widen its own jail, and a refusal names the setting that allows it:

  ```toml
  [sandbox]
  write_paths = ["~/work/shared-fixtures"]   # extra writable roots
  network = "off"                            # strict: no loopback either
  # mode = "off"                             # run commands unjailed
  ```
- In a session: Tab autocompletes slash commands and their arguments; `/changes` (Ctrl+G) opens a change explorer with each edited file's diff. Plain `nool …` shell commands run as the session's agent, and commands that land or undo work (`solidify`, `try promote`, `checkpoint`, `push`) always ask. The `list` tool hides dot entries unless called with `hidden: true`.

## Notes

- Documented for the 7.10.0 Community build: `Nool CLI v7.10.0`.
- `nool checkpoint` is the primary release-label command; `nool release` remains a backward-compatible alias.
- The current quick-start path is `status --compact -> discover features -> announce intent -> work start -> task create -> propose -> solidify`.
- For agent workflows, prefer `--compact` on `status`, `log`, `dag`, and `plan status`.

## Learn More

- Product site: https://www.nool.dev
- Why Nool exists: https://www.nool.dev/why-nool
- Semantic Impact Envelope: https://www.nool.dev/semantic-impact-envelope
- Research: https://www.nool.dev/research
- Setup: https://www.nool.dev/setup
