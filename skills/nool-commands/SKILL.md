---
name: nool-commands
description: Comprehensive expert guidance for the Nool CLI (v7.3.0). Covers semantic VCS, automated architectural discovery, high-fidelity visualization, semantic planning (RFC-0001), interactive review (RFC-0008), evidence-based transitions, and multi-agent coordination. Optimized for AI coding agents.
license: Apache-2.0
metadata:
  version: "7.3.0"
  author: nool-core-team
compatibility: Requires nool CLI v7.3.0+
allowed-tools: bash(nool *)
---


# Nool CLI

Nool is a Semantic-Agentic Commutative VCS (currently **v7.3.0**). The source of truth is the **Knot DAG**, not text.

**When this applies:** any project tracked by Nool — confirm with `nool status`. Note that the presence of a `.nool/` directory alone is *not* proof: `~/.nool` is the machine-level identity/config dir (`config.toml`, `identity.key`), and tooling can create a bare `.nool/` as a side effect of writing a log there. A real ledger contains artifacts like `knots/`, `nool.db`, `manifest.toon`/`manifest.json`, `git_mirror/`, or `memory/`. When scripting a check, look for one of those, not just the directory. In such a project, **use `nool` for VCS, task management, and debugging** instead of raw `git`. In a repo with no `.nool/`, use git normally (or `nool init` to start tracking). Nothing here is specific to one repository — it is the general workflow for working in any Nool-tracked codebase, in any language.

The `nool` binary is wherever it is installed on PATH (commonly `~/.local/bin/nool` or `/usr/local/bin/nool`). Two flags work on **every** command: `--compact` (concise agent-optimized output) and `--quiet` (errors + results only). Many commands also accept `--json` for machine-readable output — including `status`, `log`, `propose`, `doctor`, and `task list`. When you need to parse output, prefer `--json`; otherwise default to `--compact`.

When unsure of flags or subcommands, run `nool <command> --help` — the CLI is self-documenting. `nool quick-start` and `nool guide <agent|human|onboarding|research|debugging>` give built-in walkthroughs.

## Core mental model

- **Knot** = atomic semantic mutation (the "commit"). Lives in the DAG once solidified.
- **Propose → Solidify** is the commit cycle: `propose` stages a candidate Knot, `solidify` signs + appends it and auto-mirrors to Git (Bifrost).
- **Full mode** (`--full`) = full semantic guarantees, 30–90s, and is **the default**. **Fast mode** (`--fast`) = <5s local iteration with deferred validation — an explicit, unsafe opt-in. Both the AST precheck and reification are toggleable in `nool.toml` (default on) — see Tips.
- **Thread** = intent grouping across Knots. **Task** = tracked unit of work.
- **Souls / agents / fleets** = the v5.0 metaharness: persistent personas and swarms of sovereign agents that run bounded, journaled, budget-gated work over a goal.

## Editions and exit code 4

Nool ships as three binaries built from one source, all named `nool`: **Community** (free forever, 1,000 knots per rolling month, single seat), **Team** (fleets, council, steer, playbooks, announce/leases, personas, eval, PR summaries, exploration, GitHub/Jira import, MCP handoff) and **Enterprise** (Team plus workspace roll-ups, audit, commit templates, governance packs, contracts, plugin install, `console --serve`, air-gapped `nool.lic`). `nool version` and `nool status` print the edition.

- A lower edition answers a higher edition's command with an upgrade hint and **exit 4** (`unavailable`), never a clap usage error: `nool fleet plan` on Community prints ``fleet` is in Nool Team → nool admin account trial · nool.dev/pricing``. With `--json` the denial is `{"outcome":"unavailable","reason_code":"EDITION_REQUIRED",…}`.
- Team and Enterprise binaries are **lease-gated**: without a valid lease their command groups exit 4 with `reason_code: LEASE_REQUIRED`, while `status`, `log`, `task`, `thread`, `propose`, `solidify` and every other single-player command keep working in every edition. Activate with `nool admin account activate <key>`, start a 30-day Team trial with `nool admin account trial`, switch binaries with `nool upgrade --edition team|enterprise`.
- A revoked lease degrades the account to Community; nothing is hard-blocked. A lease bound to a device (`device_id`) verifies only on that machine.

Treat 4 like 2 and 3: retrying unchanged never helps — activate a lease, or install the edition that carries the command.

## The golden path (agent commit flow)

Run these in order to land a change:

```bash
nool status --json                                       # 1. read repo state, pending proposals, threads
nool propose --all --intent "fix rate limiting"           # 2. stage working-tree change as a candidate Knot
nool solidify                                            # 3. sign + append to DAG, auto-commit to Git mirror
nool push <remote>                                       # 4. optional: replicate Knot history
```

Shortcut: `nool propose --all --intent "..." --solidify` does steps 2–3 in one call. Add `--push [remote]` to also push.

**Close the fast-mode loop.** `--fast` is an explicit opt-in, and it *defers* validation rather than skipping it — the deferred half only runs when something calls for it. If you have been landing knots with `--fast`, settle the debt periodically:

```bash
nool validate --all          # run deferred validation for pending fast-mode knots
nool validate <knot-ids>     # or just the ones you care about
```

Nothing runs this for you. A long-lived agent session that only ever proposes `--fast` accumulates knots whose validation never executed, and a failure that would have been caught sits undetected until a release check. Run it at the end of a work session, before `nool doctor`, and before any release.

Useful `propose` flags: `--all` (prefer over enumerating paths), `--dry-run` (preview without creating a stash), `--amend` (modify the pending candidate), `--abort` (discard a stuck/mistaken candidate that's blocking solidify), `--breaking`, `--issue <ref>`, `--test-note <note>`, `--tag <tag>`, `--kind <fn|test|config|doc>`. For compiled languages, `--bundle <files>` + `--project-root` enable project-level reification. For binaries use `--binary <skip|raw|lfs>`.

## Choosing the right verb

Most mistakes here are reaching for coordination machinery on solo work, or skipping it on shared work. Pick by situation, not by habit:

| Situation | Verb |
|---|---|
| Ordinary change you intend to keep | `propose --all --intent "..." --solidify` |
| Speculative work you might throw away wholesale | `try new <name>` → work → `try diff`/`try impact` → `promote` or `discard` |
| Broad or risky edit | `debug blast-radius <path>` first, then propose with `--full` |
| Another agent or human may touch the same files | `announce intent --target-nodes <paths>` → `discover conflicts <paths>` → propose |
| Several tasks to run in parallel | `fleet advise` → `fleet plan` (computes NodeID-disjoint waves) → `fleet start` |
| Need to reach the agent you collided with | `nool msg <agent-key> "<message>"` |

### `try` — speculative work, with optional isolation

`nool try new <name>` opens an **ephemeral branch in the DAG**. Its knots stay out of main history until `try promote`, and `try discard` throws the whole line of work away.

Use it when the *outcome* is uncertain — a spike, a risky refactor, an approach you may abandon. Review it with `try diff` and `try impact` before promoting.

By default (`[try] mode = "shared"`) a try branch is **metadata only**: edits live in the working tree every agent shares. For real isolation add `--worktree` (v7.0):

```bash
nool try new spike --worktree --nodes src/auth/session.rs   # checkout at .nool/try/spike/, lease filed (exit 3 if held)
cd .nool/try/spike && nool propose --all --intent "..."          # stash lands under the branch; the lease renews
cd - && nool try impact spike --json                       # computed drift + conflicts, ready_to_promote
nool try promote spike        # 3-way merge onto current heads; conflict = exit 3 + diff3 markers in the worktree
nool try discard spike        # real undo: worktree torn down, lease released
```

`try promote` merges drift on main into the branch; a same-symbol conflict names the symbol and the owning knot/agent. Fleets get all of this per task with `nool fleet plan --task id=paths --run <builder.yaml> --isolation worktree` (`--overlap parallel` orders overlapping tasks with `--after` and merges at promote), and the quorum fleet's builder gets it with `nool fleet start --isolation worktree --nodes <paths>` (approved ⇒ the branch is promoted; blocked ⇒ it stays as the review surface). Isolated builders' `fs.*`/`shell.*` sandboxes and the `[harness.sandbox]` write whitelist are rooted at their worktree.

### `announce` — only useful with target nodes

```bash
nool announce intent --intent "refactor parser" --target-nodes src/parser.py,src/lexer.py
nool discover conflicts src/parser.py src/lexer.py     # exit 3 = genuine conflict
```

Two things make this work or fail:

- **`--target-nodes` is what gets matched.** `discover conflicts` compares node sets. An announcement with no nodes is visible to humans but participates in no conflict detection at all — it protects nothing.
- **Check the exit code, not the output.** `discover conflicts <nodes>` exits **3** on a genuine conflict (contention: back off and retry) and **0** otherwise. `announce intent` and `propose` exit 3 the same way when a foreign lease covers the scope. Called with *no* node ids it prints a usage hint and exits 0 — so any "did it print something?" test reports a conflict on every call.

Announce before parallel or shared work; skip it for solo sequential work, where it is pure overhead. On a real conflict: see who holds it with `nool announce status [--json]`, block until it frees with `nool announce wait <scope> --timeout-ms N` (exit 3 on timeout; under `backend = "hub"` it long-polls the hub and wakes on release — `nool announce status --follow --timeout-ms N` streams the hub's grants/releases live), or narrow to a disjoint set. Leases carry a fencing number (`grant_seq`) that the seal checks, renew with `nool announce renew <id>`, and — with `[coordination] backend = "git"` or `"hub"` — replicate to other machines (`--as-agent`/`NOOL_AGENT_ID` give co-located agents distinct identities).

## Task management

Lifecycle: **Open → Claimed → InProgress → InReview (QA) → Done**. QA is where acceptance criteria are verified.

```bash
nool task create --name "..." --desc "..." [--priority N] [--thread NAME] --solidify --json
nool task board                       # Kanban view (Open → Claimed → InProgress → QA → Done)
nool task inbox                       # unassigned open tasks + criteria attention state
nool task mine                        # active tasks assigned to or created by me
nool task pick --id <id> [--solidify] # claim a task
nool task start --id <id>             # Claimed → InProgress
nool task qa --id <id>                # InProgress → InReview (submit for QA; criteria verified here)
nool task finish --id <id> --landed-knot <knot> -s   # mark done (advisory unless criteria review is required)
nool task finish --id <id> --landed-knot <knot> --through   # from Open/Claimed/Blocked: pick+start land first, one command
nool task verify-done --id <id>       # mark verified-done directly from any active state
nool task block --id <id>             # record a blocker
nool task show --id <id>              # quickest next step after any task change
nool task telemetry --id <id>         # token usage + cost attributed to a task
nool task assign --id <id> ...        # assign to a user or agent identity
nool task relate --id <id> ...        # link to another existing task retroactively
nool task cancel --id <id>            # no longer intended (keeps auditable history)
nool task remove --id <id>            # drop from active views (keeps auditable history)
```

Acceptance criteria (advisory, never block execution):

```bash
nool task criteria inbox              # tasks with missing/draft/stale criteria
nool task criteria set --id <id> ...  # replace all criteria in one operation
nool task criteria review --id <id>   # advisory review (optionally replacing criteria first)
```

Import from issue trackers: `nool task list-github` / `import-github`, `list-jira` / `import-jira`, and `nool task sync` to pull assigned tasks from configured providers.

Start a larger unit of work (creates intent + optional sub-tasks):

```bash
nool work start --intent "..." [--parallel N] [--teams a,b]
```

## v5.0 metaharness (personas, agents, fleets)

Run autonomous, journaled, budget-gated work. Backends ("L3 harnesses") are swappable — `host` is always ready; others are ready when their wrapper is on PATH.

**Renamed in v6.x** (old names still resolve as aliases, so existing scripts keep working — but prefer the new ones): `soul` → **`persona`**, `inquiry` → **`explore`**, `flow render` → **`order render`**.

```bash
nool harness list                     # which model backends are healthy/ready

nool agent list                       # list + validate agent specs (*.yaml in a dir)
nool agent validate <file>            # validate one spec, print resolved config
nool agent run --spec <file> --intent "..."   # run one agent (budget-gated, journaled to .nool/runs)

nool fleet advise --task id=nodes --blast <n> [--tools <n>]   # recommend architecture before spending (v5.8)
nool fleet capacity [--width auto] [--builder <spec>]         # how wide this machine/budget allows, and what bounds it
nool fleet plan  --task id=nodes [--width <n>|auto]           # preview NodeID-disjoint parallel waves (no spend)
nool fleet start --builder <b.yaml> --reviewer <r.yaml> --intent "..." [--blast <n>] [--finish-task <id>]

nool persona create <name> --model <catalog-entry> --charter <path> [--memory] [--budget-usd-per-day <n>]
nool persona list | status            # list / inspect personas
nool persona run <name>               # one bounded autonomous pass, replayable evidence

nool eval set-create <name> ...       # define a named eval set (goal + criteria)
nool eval run --run <id> --set <name> # score a recorded persona/agent run against it
nool eval report                      # past eval results for a run or task
```

`nool eval` grades structural criteria automatically (terminal state, budget respected, tools used, knots produced); free-form criteria come back **Unknown** for manual or LLM follow-up rather than being guessed. Use it to tell whether an autonomous run actually did the job, instead of trusting that it exited cleanly.

**The wave concurrency ceiling sizes itself to the machine by default.** `--width` takes an integer or `auto`; the default is **`auto`**. Precedence is `--width` > `[fleet] width` > `auto`. Pin an integer to opt out — a pinned width is used verbatim, and is *reported* when it exceeds what the host can carry rather than silently over-subscribing.

**Three controls, deliberately separate.** Wave membership is NodeID *disjointness* — a property of the work, not of the machine. The other two are capacity:

- **Admission** caps how many agents of a wave run at once, re-measured *before every wave* (memory moves while a fleet runs). A pinned `--width` shapes the plan but does **not** raise admission: on a host with no cgroups and no working `RLIMIT_AS`, admission is the only lever that protects the machine. If the ceiling looks too low, fix the *estimate* — `[fleet] memory_per_agent_mb`, or use a backend that declares its own cost — rather than overriding the measurement.
- **`[fleet] launch_stagger_ms`** (default 250) rate-limits the *ramp*: without it every agent in a wave is created in the same instant and their startup allocations land on top of each other, which spikes well above steady state. A cap and a stagger solve different problems; you want both. Set 0 for the old burst.

**Provider rate limits are a fourth dimension**, and the only one the host cannot see. Set `[fleet] provider_requests_per_minute` and/or `provider_tokens_per_minute` and `auto` divides them by what one agent draws (`agent_requests_per_minute`, default 45; `agent_tokens_per_minute`, default 45000 — the p90s over 5,851 benchmark agent runs, cache reads excluded since providers meter those separately). Undeclared limits do **not** bind: unknown is not unlimited, but throttling against a guess is worse than not throttling. This matters most once agents stop being subprocesses — with memory out of the picture, the rate limit is what 429s a wave.

**The executor declares what a unit of concurrency costs.** `nool-harness` abstracts which vendor runs an agent; it now also says what one costs. A vendor-CLI backend (`claude-sdk`, `codex`, `gemini`, `copilot`, `ollama`, `deepseek`, `openrouter`) is a subprocess with its own Node/V8 heap and declares ~512 MiB; `host` runs in-process via `nool_core::agent::run_agent_loop` and declares *negligible*, which removes memory from the ceiling entirely rather than shrinking it. So the same repo on the same box admits 4 subprocess agents and 10 in-process ones — capacity follows the executor, not a blanket guess.

Why `auto` rather than a fixed number: across a 142-run fleet corpus (N=1..45), realized concurrency plateaus at **~5-7x however many workers are requested** (N=10 achieved 4.0x, N=45 achieved 6.2x), while **25 simultaneous agent processes are on record exhausting a developer machine's RAM in ~9s**. A fixed ceiling cannot know either fact. `auto` gives up no measured throughput and removes the cliff.

```toml
[fleet]
width = "auto"            # or an integer to pin it
reserved_cpus = 2         # cores left for the operator and the OS
memory_per_agent_mb = 512 # raise it when builders compile
min_width = 1
max_width = 16            # default: bounds `auto` only, not a pinned width
budget_usd_per_day = 5.0  # optional third ceiling
```

`auto` takes the **minimum** of three measurements and names the one that bound it:

- **CPU** — logical cores (cgroup- and affinity-aware) minus `reserved_cpus`.
- **Memory** — memory the host reports *available* (not total) divided by `memory_per_agent_mb`.
- **Budget** — `budget_usd_per_day` minus today's spend since 00:00 UTC, divided by the builder spec's `budget.per_run_usd`. It binds only when a builder spec is supplied (`fleet plan --run`, `fleet advise --dispatch`); without a per-run price the remaining allowance buys an *unknown* number of agents, so budget drops out of the minimum rather than pinning the fleet to one.

`nool fleet capacity` answers the ceiling question on its own — no tasks, no spend — printing every probe's number and the one that binds, so you know what to raise next. `fleet plan` prints the same decision, and `--json` carries it as `width_decision`:

```
Fleet plan (3 task(s), max width 4):
  Ceiling: width 4 (auto: bounded by remaining daily budget — cpu 10, memory 17, budget 4)
```

A probe that cannot answer on a platform drops out instead of contributing a zero; if none answers, `auto` falls back to 8.

`nool fleet advise` (v5.8) is a pure recommender: from task signals (count, parallelism = widest disjoint wave ÷ task count, tool density, blast) it returns **Single** (one agent), **ParallelIndependent** (decomposable, low-risk), or **CentralizedQuorum** (coupled or high-risk → builder + reviewer quorum), with rationale and the exact command to run. `--json` for automation. Run it before `fleet plan`/`start` to pick the right shape.

`nool council --consortium <name>` runs a configured multi-model LLM council on the working-tree diff (or `--diff-file`) and prints each member's APPROVE/DENY verdict plus the quorum/veto outcome. Consortiums are defined in `[consortium.<name>]` in `nool.toml`.

### Model providers, cost & souls (config-driven)

Model backends are declared in `nool.toml` and bound to souls/consortiums by catalog name:

```toml
[models.catalog.deepseek-cheap]            # a cost-tier entry
provider = "openrouter"                     # claude|codex|gemini|copilot|ollama|openrouter (v5.7: openrouter first-class)
model    = "deepseek/deepseek-chat"
auth_env = "OPENROUTER_API_KEY"
fallback = "host"

[telemetry.pricing_overrides."deepseek/deepseek-chat"]   # accurate cost accounting
input_cost_per_million_tokens  = 0.27
output_cost_per_million_tokens = 1.10

[soul.builder]                              # a soul bound to a catalog entry + daily budget cap
model = "deepseek-cheap"
charter = "charters/builder.md"
budget_usd_per_day = 0.50

[consortium.review-council]                 # cross-vendor review gate
members = ["deepseek-cheap", "kimi-quality"]
quorum  = 1.0
veto    = ["kimi-quality"]
budget_usd = 0.10
```

Agent specs (`*.yaml`) bind an executor backend directly. OpenRouter routes both agent runs and council members; for the OpenAI-compatible backends the base URL is overridable via `<PROVIDER>_BASE_URL` (e.g. `OPENROUTER_BASE_URL`). Agent-run cost is priced from the catalog/pricing overrides and recorded so `nool usage usage` reflects direct runs (v5.7).

### Setting up & running a fleet (worked example)

A fleet = one **builder** agent produces work, then a **cross-vendor reviewer quorum** judges it (dynamic quorum tightens with blast; a high-trust reviewer can veto). Steps:

**1. Make the model's key visible to the process env** (not just your interactive shell — non-interactive shells read `~/.zshenv`, not `~/.zshrc`):
```bash
echo 'export OPENROUTER_API_KEY=sk-or-...' >> ~/.zshenv   # then restart the session
```

**2. Write agent specs** (one builder, one or more reviewers; put them anywhere, e.g. `agents/`). Use **different vendors/models** for reviewers so the quorum is genuinely cross-vendor:
```yaml
# agents/builder.yaml — cheap, fast worker
name: builder
executor: { harness: openrouter, model: deepseek/deepseek-chat, auth: env:OPENROUTER_API_KEY }
budget:   { per_run_usd: 0.05, max_turns: 2 }
policy:   { trust_tier: 3 }
```
```yaml
# agents/reviewer-kimi.yaml — higher-quality reviewer (high trust tier can veto)
name: reviewer-kimi
executor: { harness: openrouter, model: moonshotai/kimi-k2, auth: env:OPENROUTER_API_KEY }
budget:   { per_run_usd: 0.10, max_turns: 1 }
policy:   { trust_tier: 5 }
```
Validate before running: `nool agent validate agents/builder.yaml`. `harness:` is one of `claude-sdk|codex|gemini|copilot|ollama|deepseek|openrouter|host`; `host` is the always-ready in-process fallback (no key, no spend).

**3. (Optional) pick the shape:** `nool fleet advise --task t1=nodeA --task t2=nodeB --blast <n>` → Single / ParallelIndependent / CentralizedQuorum.

**4. Run the fleet:**
```bash
nool fleet start \
  --builder agents/builder.yaml \
  --reviewer agents/reviewer-kimi.yaml --reviewer agents/reviewer-ds.yaml \
  --intent "Add a validate_email helper" \
  --blast 12 [--finish-task <id>]
```
The builder produces output; each reviewer returns APPROVE/DENY; the **dynamic quorum** (base `--base-quorum`, default 0.6, tightened by `--blast`) decides Pass/Blocked, and a trust-tier-≥5 reviewer's DENY is a binding veto. Each run is journaled to `.nool/runs/`; cost rolls into `nool usage usage`. A cross-vendor warning fires if all reviewers share one vendor (correlated votes). `--finish-task` marks a tracked task done when the quorum approves.

**Budget/quality knobs:** per-agent `budget.per_run_usd` + `[telemetry.pricing_overrides]` control cost; `policy.trust_tier` controls veto power; reviewer model choice controls quality. Preview parallelism for many tasks with `nool fleet plan` first.

### Driving Codex subagents from Nool

There are two distinct ways Nool and Codex compose, and they are easy to conflate:

**1. Nool dispatches to Codex** — bind an agent spec to the `codex` harness and Nool runs a Codex session per agent, budget-gated and journaled:

```bash
nool harness list                                   # codex shows `ready` when the CLI is on PATH
nool agent init builder --harness codex             # scaffold a spec bound to Codex
nool agent run --spec agents/builder.yaml --intent "..."
nool fleet start --builder agents/builder.yaml --reviewer agents/reviewer.yaml --intent "..."
```

Each Nool agent is a separate Codex process. Use `fleet plan` first to get NodeID-disjoint waves so the processes never contend for the same files.

**2. Codex's own subagents run under Nool governance** — inside one Codex session, the model calls `spawn_agent` to fan out. The hook bridge (`nool-bridge/`) makes those subagents first-class Nool participants:

- at the `spawn_agent` call, the task text is read and any **real** file paths it names become `--target-nodes`; if another agent already announced those nodes, the spawn is **denied** with nool's conflict report
- once the subagent has an id, its intent is announced so *sibling* spawns can be blocked against it
- when it stops, its findings are recorded against the **files** it worked on, so the next agent to touch those files retrieves them via `nool findings <path>`

The ordering matters: the conflict gate runs at spawn time, before the announcement exists. Checking afterwards would match the agent's own claim and report every subagent as racing itself.

`nool explore` (was `nool inquiry`) inspects/drives the **exploration tree** (structured record of agent directions, evidence, insights): `view`, `frontier`, `constraints`, `open`, `dispatch`, `record`, `prune`, `merge`, `run`.

### Playbooks & governance (v6.x)

A **playbook** is a named, ordered recipe of existing verbs, gated by the org's steering rules — the way to make a repeatable loop reproducible instead of retyped. Defined as `[playbook.<name>]` in `nool.toml`.

```bash
nool playbook list                    # named playbooks defined in nool.toml
nool playbook plan <name>             # resolved ordered steps, read-only, no execution
nool playbook run <name>              # execute in order, halting on the first gate denial
nool playbook cost <name> [--cadence] # project token/USD cost before scheduling it recurring
nool playbook audit [--suggest]       # governance/loop-readiness score 0-100 + concrete next actions
nool playbook suggest <name>          # propose a diff to a fleet step from observed run history
```

`nool playbook audit` is the fastest way to see whether a repo's agent loops are actually governed: it checks that `[steer]`/`[trust]` are enabled, that every playbook resolves, and that each step uses a verb with a reliable halt signal. A fresh repo scores 0/100. Scaffold the missing config with:

```bash
nool config init-governance           # print a governance scaffold for review
nool config init-governance --write   # append it to nool.toml
```

That writes org roles, responsibilities, controls, and DSL policies — the `[steer]` checkpoints and `[trust]` progressive-trust settings that turn agent commits from advisory into gated.

**LLM judgment gating (v6.6).** `[gating]` now carries a judgment tier that can escalate a decision to a human role instead of silently blocking or allowing:

```toml
[gating]
judgment_blocking            = false          # does a judgment failure block?
judgment_tools               = []
judgment_budget_usd          = 0.1
judgment_escalation_blocking = false
judgment_escalation_role     = "tech-lead"    # who a judgment escalates to
judgment_escalation_dir      = ".nool/judgment-review"
judgment_retain_days         = 90
judgment_backlog_limit       = 25
```

Resolve a raised checkpoint with `nool steer`. Beyond the per-point form, v6.x adds `--cascade` to settle a whole parent→child approval chain in one command (e.g. a child run that halted and dragged its parent down):

```bash
nool steer --cascade <id> --action approve|reject
```

## Coordinated work (multi-agent / shared paths)

```bash
nool announce intent --intent "..." --target-nodes <ids> [--thread NAME]   # declare what you'll touch
nool announce with-context ...  # same, plus a captured context snapshot
nool msg <agent-key> "<msg>"    # direct 1:1 message to a conflicting agent
nool discover conflicts <nodes> # check overlap (exit 3 = conflict); needs node ids
nool discover similar          # find precedent (similar work by topic/approach)
nool discover context          # retrieve a context snapshot from previous work
nool discover learnings        # extract learnings/decisions from a thread
nool discover features         # map logical feature boundaries (the "guardian" map step)
nool discover lift             # save discovered feature boundaries into the graph
```

Sequence for shared work: `announce intent --target-nodes <paths>` → `discover conflicts <paths>` → `work start` or `propose`. Both halves need the same node set; see "Choosing the right verb" above for why an announcement without `--target-nodes` protects nothing.

## Workspace (fractal multi-project coordination)

For a polyglot workspace of nested Nool projects (Org→Dept→Team→Project):

```bash
nool workspace init            # generate workspace.toml, pre-populated with discovered projects
nool workspace status          # project tree, declared edges, order
nool workspace doctor          # reconcile declared config vs discovered projects
nool workspace goal --intent "..." --decompose project=task [--decompose ...] [--save]   # fan a goal across child projects (creates a real task per project; --intent required)
nool workspace goals | goal-status                # list goals / per-project completion
nool workspace insights | telemetry               # roll up insights / cost across children
nool workspace pull | sync     # propagate DAG changes across children in dependency order
nool workspace console         # roll-up web console (port 4002)
```

## Inspection & history

```bash
nool log                       # canonical replay log (semantic history)
nool status                    # DAG state, pending proposals, active threads
nool thread show <name> --full # touched paths, deps, recorded findings
nool why <id>                  # walk the causal chain
nool dag                       # visualize the DAG
nool diff <knotA> <knotB>      # file-content diff between two Knots
nool compare <left> <right>    # compare threads or releases (--shared shows common knots)
nool changelog                 # semantic changelog
nool inbox                     # notification inbox
```

## Debugging & root cause

```bash
nool debug replay HEAD            # interactive replay of a Git ref or agent run
nool debug step | diff | edit | rerun   # inspect/constrain/replay specific steps
nool debug blame                  # find root cause (causal chain from a failure)
nool debug bisect                 # binary-search which Knot introduced a regression
nool debug blast-radius <path>    # downstream impact / risk analysis for a file or knot
nool doctor                       # repo health + release-readiness
nool doctor --strict              # treat warnings as release-blocking
nool doctor --traceability --since 7d [--strict]   # every landed knot in the window: intent ✓ test note ✓ gate ✓ (exit 2 with --strict otherwise)
nool doctor --fix [--git-fallback]   # auto-repair issues where possible (recovery)
nool doctor --fix --heads-only        # consolidate fragmented DAG heads only; skip pruning replay-rejected knots
```

Scope doctor with `--fs-only`, `--semantic-only`, `--artifacts`, or `--architecture`; add `--json` for automation.

## Semantic queries (agent context retrieval)

`nool context "<task description>"` (v6.x) is the one-call replacement for the resolve-intent → query context → blast-radius → Read dance: it assembles matching entities, their blast radius, and prior findings into one token-budgeted packet (`--budget <n>`, default 2000). Add `--skeletons` for AST skeletons with bodies elided, or `--cite` to get `path:start-end` citations instead of inline code — orientation at a fraction of the tokens. Reach for this first; fall back to the individual `query` subcommands below when you need one specific view. Add `--include-history` to also see entities that are known history — superseded, removed, or externalized — instead of excluding them by default.

`nool ground "<task>"` (v6.14) goes one step further for routing-change-style tasks — "which command handles the deploy subcommand": one ranked, budgeted GroundingPacket combining matching entities, blast radius, and prior findings — the same composition `nool context` uses — plus the causal chain from `nool why` for the most relevant prior knot. Accepts a natural-language description or a knot ID; `--budget-tokens <n>` (default 2000), `--json`.

```bash
nool query context <id> --depth <n>   # token-optimized BFS context for an entity/feature (~95% noise reduction)
nool query search "<natural language intent>"
nool query resolve-intent "<intent>"
nool query neighbors <knot>
nool query blast-radius <knot>
nool query recent-knots [--thread NAME]
nool query materialize <knots>        # reconstruct content for knots
nool query runtime-evidence           # search runtime evidence sidecars
nool query validate <path>            # validate files without proposing
```

## Knowledge ledger (record what you learn)

Use Nool as the project's knowledge store, not just VCS/tasks:

```bash
nool learn ...                 # record a knowledge finding
nool findings <topic>          # retrieve findings for a file, thread, or topic
nool enrich "<query>"          # recall; on a miss, run bounded self-healing enrichment + record the gap
nool bug report|investigate|link|list|show|wont-fix|duplicate   # track bugs, link the fixing Knot
```

## Plans, review & verification

```bash
nool plan replay              # compute ops to reach a target semantic state (RFC-0001)
nool plan merge               # plan a merge of divergent semantic branches
nool plan pluck               # plan a selective undo before executing it
nool plan status              # current plan status and steps
nool apply <plan-id>          # execute an approved/draft semantic plan
nool review <plan-id|thread>  # review candidate changes before they're finalized
nool approve --id <id> --comment "..." [--reject] [--solidify]   # record approval/rejection
nool verify --all             # run structural Relational Invariants against current/planned state
nool evidence plan|knot|merge <id>   # see why a transition was accepted/rejected
nool explain <id> [--closure] # explain an identity's dependencies and reasons
nool audit report|export      # compliance report / export with framework validation
```

### Attestations (signed verdicts about a knot)

An attestation is a signed verdict — tests, scans, reviews — *about* a knot. It is exported ref-only (never a commit on the branch) and travels on this writer's `refs/nool/detached/<writer>` chain, so CI can attest without touching history.

```bash
nool attest record --from-head --checker github-actions \
  --assertion unit-tests --verdict pass --scope full --push origin
nool attest show <knot>          # what has been attested, with trust resolved
nool attest obligations <knot>   # which assertions this knot's paths require, and what is missing
nool attest sweep                # Done/InReview -> VerifiedDone where obligations are now met
nool attest wait <knot>          # block until a trusted pass exists
```

`nool push` refuses a knot with unmet obligations when `[attestation] enforce = "block"` — read `nool attest obligations <knot>` to see which assertion is missing rather than guessing. In CI set `NOOL_IDENTITY_KEY`; `--from-head` reads `.nool/knot.bin`, so no ledger is needed.

### Impact-scoped tests (CI)

```bash
nool test select --knot <id>     # the tests this knot's changes affect
```

Uses the same dependents-closure selection as `propose`'s ghost run, so CI runs what the change actually landed with. It reads `nool discover`'s entity graph: a checkout that never ran `discover` has no index, and selection falls back to the full suite with an explicit reason rather than silently under-selecting.

## Undo, merging & branching

```bash
nool pluck <id>          # selective undo — includes transitive descendants (causal integrity)
nool rewind --knot <id>  # rewind the workspace to a previous Knot ...
nool rewind --intent "…" # ... or to a previous Intent checkpoint
nool merge <branch>      # ingest an incoming git branch (e.g. a PR) as candidate Knots
nool try ...             # ephemeral scratch branch, never enters DAG until promote
nool promote <id>        # promote a Local knot to Staged/Synced (validates + git commit)
nool checkpoint <label>  # mark a checkpoint; semver-shaped label = release (alias: nool release)
nool tag <name>          # semantic tag
nool link <id> ...       # attach existing history to intent or thread metadata
nool untrack <paths>     # stop tracking paths in Git while keeping them in the working tree
```

**`rewind` vs `pluck`** — `pluck` removes one thread from the active timeline and takes its transitive descendants with it, preserving causal integrity; `rewind` moves the whole workspace back to a named Knot or Intent checkpoint. Reach for `rewind` when an agent turn went wrong and you want the state before it; reach for `pluck` when one landed change must go but everything after it should stay.

**`merge`** ingests a divergent git branch's commits into candidate Knots, converges them commutatively, checks for semantic conflicts, and runs the full gate stack — the supported path for accepting a PR into a Nool repo. Do not merge with raw git: the DAG would not learn about it.

## Architecture recovery, assertions & export (v6.x)

Automated discovery proposes architecture facts as candidates; `assert` is how a human or agent turns a candidate (or any other judgment) into durable, signed knowledge the DAG will not silently re-derive away.

```bash
nool architecture review              # candidates awaiting review, highest downstream leverage first
nool architecture accept <id>         # confirm a candidate -> records it as an Assertion Knot
nool architecture reject <id>         # reject; a counter-assertion is recorded, the observation is kept

nool assert <subject> <predicate> <object> --justification "<why>"   # e.g. must_not_depend_on
# --inheritance rebind|lineage (default rebind): does the assertion survive split/merge, not just rename/move?
# --revokes <knot-id>: supersede an earlier assertion. -s/--solidify: save immediately.
```

`nool candidate` inspects and clears the queue of pending candidates awaiting `solidify` — the supported way out of a stuck queue (never delete files under `.nool/` by hand):

```bash
nool candidate list    # every pending candidate, across every queue
nool candidate drop <id>
nool candidate clear   # drop them all
```

`nool export` serializes non-canonical views of the semantic architecture model for external tooling: `nool export c4` (deterministic JSON, exact-level C4), `rdf` (Turtle + W3C PROV-O), `jsonld`, `cypher` (Neo4j/openCypher script), `graphml`.

## Other surfaces

- `nool visualize -k history|graph|roi|relational [-f tui|html]` — visualize project evolution and artifact graphs.
- `nool console` — local web dashboard + control-plane API (default port 4001: `/api/status`, `/api/dag`, `/api/tasks`). `--serve` runs a headless daemon (the surface the desktop app embeds).
- `nool ui` — interactive TUI DAG explorer.
- `nool daemon` — background sync daemon: signs git-mirror commits and holds a single-instance lock. Auto-starts on the machine's first *interactive* `nool init` (suppressed under CI, a non-TTY stdout, or `NOOL_NO_DAEMON`); manage with `nool daemon start|stop|status`.
- `nool insights [justifications|loops|conflicts]` — blast-radius stats, agent justifications, ROI/time-saved metrics.
- `nool usage [usage|budget-set|budget-status|analytics|agent|thread|dashboard]` — token usage, budgets, per-thread cost.
- `nool bridge` — manage the Git Bifrost mirror and large-file storage. `nool bridge reindex` fills the commit-to-knot index once so `bisect`/`try`/`compare` stop walking history; `nool bridge prune-knot-refs [--apply] [--remote]` retires the legacy `refs/knots/*` that older builds minted per branch knot.
- Mirror layout (7.1.x): each branch owns `refs/nool/mirror/<branch>`, moved in the same ref transaction as its solidify commit; `refs/nool/git-mirror` is a read-only alias of the default branch; detached knots append to `refs/nool/detached/<writer>`. `push` sends the checked-out branch, its pointer, then the ref groups; `pull` ingests incrementally from the recorded tip.
- `nool sync` / `nool pull` / `nool push` — replica sync.
- `nool languages` — list supported languages and their validation status.
- `nool config` — manage system configuration.
- `nool admin` — account settings, plugins, billing.
- `nool migrate` — move Nool files from legacy locations into the current canonical layout.
- `nool prune` — clean temporary and cached files.
- `nool reify` — inspect bundles and validate syntax.
- `nool order render --agent <spec.yaml> --intent "..." [--path p] [--node id] [--forbid p] [--hash]` — render a compact, model-facing TOON work order from an agent spec (was `nool flow render`; read-only; `--hash` prints the canonical Blake3 work-order hash).
- `nool hooks install|uninstall` — install/remove the active coding-agent guard: git history-verb blocking + session context that redirects raw git to Nool equivalents (currently supports Claude Code; `uninstall` removes exactly what `install` added).
- `nool pr summary --base <base-knot-id> [--thread <name>]` — render the knots not yet in `--base` as a markdown review-context summary (semantic signal, blast radius, findings, justifications per knot) for a CI job to post as a GitHub PR comment.
- `nool commit-template` — validate and preview the effective enterprise commit-message template.
- `nool telemetry` — whether Nool sends anonymous usage analytics (which commands run, coarse error categories, timing). Separate and unrelated to `nool usage analytics`, which is LLM token-cost analytics.
- `nool feedback` — rate Nool / give quick feedback.
- `nool completion` — generate shell completion scripts.
- `nool upgrade` / `nool uninstall` / `nool version` — CLI lifecycle.

## Scenario quick-reference

- **Land a small change** → `nool propose --all --intent "..." --solidify`.
- **Risky/broad edit** → `nool debug blast-radius <path>` first, then propose with `--full`.
- **Touching shared paths with other agents** → `announce intent --target-nodes <paths>` → `discover conflicts <paths>` → propose.
- **Spiking something you may throw away** → `try new <name>` → work → `try impact` → `promote` or `discard`.
- **Collided with another agent** → `nool msg <agent-key> "..."`, or narrow `--target-nodes` to a disjoint set.
- **Pick up tracked work** → `task inbox`/`task mine` → `task pick` → `task start` → … → `task qa` → `task finish`.
- **Find why something broke** → `debug replay HEAD` → `debug blame`; if a regression, `debug bisect`.
- **Undo a landed change safely** → `plan pluck` (preview) → `pluck <id>` (includes descendants).
- **Back out a bad agent turn** → `nool rewind --knot <id>` (or `--intent "..."`) to return the workspace to a known-good checkpoint.
- **Accept an incoming PR branch** → `nool merge <branch>` (never raw `git merge`).
- **Settle deferred validation after fast-mode work** → `nool validate --all`, then `nool doctor`.
- **Check whether agent loops are actually governed** → `nool playbook audit --suggest`; scaffold what it names with `nool config init-governance --write`.
- **Score an autonomous run** → `nool eval run --run <id> --set <name>`.
- **Release** → `nool doctor --strict` → `nool checkpoint X.Y.Z` (release). If health checks flag fixable issues, `nool doctor --fix`.
- **Recover a corrupted/stuck repo** → `nool doctor --fix --git-fallback`.
- **Fan a goal across nested projects** → `nool workspace goal --decompose <target>=<task>`.
- **Get a multi-model review of the working diff** → `nool council --consortium <name>`.
- **Decide single vs multi-agent before spending** → `nool fleet advise --task id=nodes --blast <n>`; then `agent run` (Single), `fleet plan` (ParallelIndependent), or `fleet start` (CentralizedQuorum).
- **Run a cheap builder + quality reviewer across vendors** → catalog two entries (e.g. `openrouter` deepseek + kimi) → `nool fleet start --builder <b> --reviewer <r>`.
- **Cost-control agents** → set `[telemetry.pricing_overrides]` + per-soul `budget_usd_per_day` / consortium `budget_usd`; inspect with `nool usage usage`.

## Tips

- When unsure of flags, run `nool <command> --help` — the CLI is self-documenting.
- Use `--json` for any command whose output you need to parse; `--compact` otherwise.
- **Markdown gotcha:** `nool propose` rejects `.md` commits unless `(){}[]` balance file-wide (code fences included). Balance brackets before proposing docs.
- **Fresh project:** `nool init` sets up `.nool/` but not a git worktree; run `git init` before `nool propose --all` (it needs a worktree to collect changes).
- **Blast-radius gate:** a broad change is blocked non-interactively until you pass `nool propose ... --justification "<reason>"` (or `--auto-justify`).
- **Pre-solidify steer gate:** when `[steer].enabled` and a signal like `coupling_slope_gt` fires, `solidify` blocks with `Steering intervention required at PreSolidify`. The block message prints the exact **steerable subject id** and a ready-to-run command; clear it non-interactively with `nool steer --point pre-solidify --role <role> --action approve --target <subject> --attestation "<signed note>"` (the `--attestation` satisfies the high-risk challenge without stdin), then re-run `solidify`. The subject is a stable content hash, so approving it once is enough even though the approval knot advances the head.
- **Live model runs:** vendor backends need their key in the **process env** (e.g. `OPENROUTER_API_KEY`). A non-interactive shell sources `~/.zshenv`, not `~/.zshrc` — put keys there. Missing key → silent fallback to the in-process `host` backend.
- **Release ordering:** after bumping versions, regenerate any lockfiles/generated artifacts (e.g. build once) and commit them *together with* the version change *before* `nool checkpoint X.Y.Z` — otherwise the regenerated file lands after the checkpoint and `nool doctor` flags a modified tracked file (NOT_RELEASABLE).
- **Fast mode defers, it does not skip:** `--fast` is an explicit opt-in (`--full` is the default), and nothing runs the deferred validation for you. An agent loop that only ever proposes `--fast` builds up knots whose validation never executed. Run `nool validate --all` at the end of a work session and before `nool doctor`/release.
- **Old command names still work:** `soul`, `inquiry`, and `flow` remain as aliases for `persona`, `explore`, and `order`. Prefer the new names in anything you write, but do not "fix" existing scripts on sight — they are not broken.
- **Validation is configurable (v5.9.1):** `[reification].enabled` (native integrity drivers like `cargo check`) and `[analysis].ast_enabled` (in-process AST/syntax precheck) both default to `true` in `nool.toml`. Set either to `false` to skip that check — the Aram policy gate still runs. Useful for repos/languages without a working toolchain, where `--full` reification would otherwise error.
