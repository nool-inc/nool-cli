# Nool Commands Reference

**Version**: 7.3.0

This document is a command reference for the Nool CLI, organized by skill category. For narrative guidance, see the companion `SKILL.md`.

**Two flags work on every command:**
- `--compact` — concise output optimized for automated agents.
- `--quiet` — show only errors and command results.

Many commands also accept `--json` for machine-readable output (e.g. `status`, `log`, `propose`, `doctor`, `task list`). Run `nool <command> --help` any time you need exact flags.

**Exit codes** (every command): `0` ok · `1` error (escalate) · `2` blocked (change the content, retry) · `3` conflict (back off, retry unchanged) · `4` unavailable (the command is in a higher edition, or this edition's lease is missing — activate or upgrade). `nool doctor` keeps its own documented codes.

**Editions**: the same source builds a Community, a Team and an Enterprise `nool`. Commands marked *[Team]* or *[Enterprise]* below exist only in that edition and above; a lower edition prints the upgrade hint and exits 4, and the Team/Enterprise binaries verify a lease before running them (`nool admin account trial`, `nool admin account activate <key>`, `nool upgrade --edition team|enterprise`). Team: `agent`, `announce`, `council`, `eval`, `explore`/`inquiry`, `fleet`, `order`/`flow`, `harness`, `msg`, `persona`/`soul`, `playbook`, `pr`, `review`, `steer`, `task list-github|import-github|list-jira|import-jira`. Enterprise: `audit`, `commit-template`, `workspace`, `admin pack|contract`, `admin plugin install`, `console --serve`.

---

## 1. Initialize & Bootstrap

### `nool init`
Initialize a new Nool ledger and generate an identity key.
- `--from-git <branch>`: import existing git history on this branch.
- `--transport <git|pijul>`: transport backend (default `git`).

### `nool migrate`
Move Nool files from legacy/previous-version locations (root-level `nool.db`/`knots.redb`, `.claude/nool.db`, in-project license keys) into the current canonical layout.
- `--dry-run`: show what would be migrated without changing anything.
- `-y, --yes`: skip the confirmation prompt.

### `nool rewind`
Rewind workspace state to a previous Knot or Intent checkpoint.
- `--knot <KNOT>`: target Knot ID (hex or prefix).
- `--intent <INTENT>`: target Intent description.

---

## 2. Work & Intent Management

### `nool work start`
Start a new work item with optional parallel subtasks. Follow `nool announce intent` for shared work, then split into `nool task create` or `nool try new`.

### `nool announce intent`
Announce intent before starting work. Files a fencing lease (v7.0 "leases v2"): a `grant_seq` is assigned at grant and re-checked at `solidify`, so an expired or superseded lease is refused even if it looked valid when the work started.
- `--intent "<description>"`: declare what you'll touch.
- `-k, --target-nodes <ids>`: semantic paths that will be touched — this is what `discover conflicts` actually matches on; an announcement with no nodes protects nothing.
- `--mode <exclusive|shared>`: `exclusive` (default, you're editing) or `shared` (read/review; shared holders coexist but block exclusive ones).
- `--agent-id <id>`: coordination identity, so co-located agents on one machine aren't indistinguishable (defaults to `$NOOL_AGENT_ID`).
- `-t, --thread <name>`: optional thread association.
- `nool announce with-context`: announce intent and capture a context snapshot (decisions, constraints, patterns, test gaps).
- `nool announce status [--json] [--follow]`: who holds what, in which mode, with fencing number, claim state and origin. `--follow` streams grants/releases live from the hub backend.
- `nool announce wait <scope> --timeout-ms <n>`: block until the scope frees, or exit 3 on timeout. Under `[coordination] backend = "hub"` this long-polls the hub instead of spin-polling.
- `nool announce renew <id>` / `nool announce release <id>`: extend or give up a held lease early.

### `nool discover`
Find conflicts, context, learnings, and similar work.
- `nool discover conflicts`: check for overlapping active announcements before proposing.
- `nool discover context`: retrieve a context snapshot from previous work.
- `nool discover learnings`: extract learnings/decisions from a thread.
- `nool discover similar "<query>"`: find similar work by topic or approach.
- `nool discover features`: discover logical feature boundaries in the project.
- `nool discover lift`: save discovered feature boundaries into the graph.

`nool discover conflicts <nodes>` exits **3** on a genuine conflict (back off and retry) and `0` otherwise — check the exit code, not the printed output. `announce intent` and `try new --nodes` exit 3 the same way when a foreign lease already covers the scope. Called with no node ids `discover conflicts` prints a usage hint and exits `0`, so an announcement made without `--target-nodes` participates in no conflict detection at all.

With `[coordination] backend = "git"` or `"hub"`, leases replicate beyond one machine: `"git"` as commit-backed `refs/nool/claims/*` (every write is a force-with-lease CAS — stale writes are refused, never silently clobbered); `"hub"` as atomic, expiring grants from a `nool-hub-server` with OIDC principals and org/repo ACLs.

### `nool msg <recipient_key> <message>`
Send a direct 1:1 message to another agent — the way to coordinate after `nool discover conflicts` reports an overlap.
- `-t, --ttl <ttl>`: message lifetime.

---

## 3. Propose & Solidify (the commit cycle)

### `nool propose`
Generate a candidate Knot.
- `-i, --intent "<description>"`: what you're changing.
- `--all`: use all modified, deleted, staged, and untracked worktree paths.
- `--path <file>...`: specific changed files.
- `-t, --thread <name>`: associate with a thread.
- `-k, --kind <fn|test|config|doc>`: knot kind (auto-detected if omitted).
- `--fast`: fast mode, <5s, deferred validation. An explicit, unsafe opt-in — `--full` is the default.
- `--full`: full semantic guarantees (30–90s).
- `-s, --solidify`: save immediately if validation passes.
- `--push [<remote>]`: push after saving.
- `--bundle <files>` / `--project-root <root>`: project-level reification for compiled languages.
- `--binary <skip|raw|lfs>`: how to handle binary files.
- `--breaking`, `--issue <ref>`, `--test-note <note>`, `--tag <tag>`.
- `--interactive`: guided mode for beginners.
- `--detailed`: show all affected nodes in blast-radius warnings.

Shortcut: `nool propose --all --intent "..." --solidify`.

### `nool solidify`
Finalize the current candidate Knot: sign + append to the DAG, auto-mirror to Git (Bifrost).
- `-t, --thread <name>`: target a specific thread.
- `--local`: save locally without a git commit.
- `--fast` / `--full`: validation mode; `--full` is the default.
- `--push [<remote>]`: push after saving.

### `nool validate`
Run background validation for pending fast-mode knots.
- `[KNOT_IDS]...`: validate specific knots.
- `--all`: validate all pending fast knots.

### `nool candidate`
Inspect and drop queued candidates awaiting `solidify` — the supported way out of a stuck queue. Never delete files under `.nool/` by hand.
- `nool candidate list`: every pending candidate, across every queue.
- `nool candidate drop <id>`: drop one pending candidate by id.
- `nool candidate clear`: drop all pending candidates.

---

## 4. Try Branches, Checkpoints, Tags & Promote

### `nool try`
Short-lived experiment branches; never enter the DAG until promoted.
- `nool try new <name> [--from <knot>] [--intent "..."]`: start a branch. Metadata-only by default (`[try] mode = "shared"`) — edits stay in the shared working tree.
- `nool try new <name> --worktree [--nodes <scope>]`: materialize an isolated git worktree under `.nool/try/<name>/` (v7.0). `--nodes` files a fencing lease for the scope atomically — a foreign hold refuses with exit 3 and creates no branch. Work from inside the checkout; every `propose` there renews the lease.
- `nool try list`: list active try branches.
- `nool try show <name>`: status of a try branch.
- `nool try diff <name>`: unified diff.
- `nool try impact <name>`: impact analysis, drift, and `ready_to_promote`.
- `nool try promote <name>`: 3-way merge into main history against the recorded base — drift on main is computed per-path; a real conflict exits 3 and leaves `diff3` markers in the worktree instead of guessing.
- `nool try discard <name>`: discard, release the lease, and tear down the worktree.

Fleets get this per task: `nool fleet plan --task id=paths --run <builder.yaml> --isolation worktree` (`--overlap parallel` orders overlapping tasks with `--after`), and `nool fleet start --isolation worktree --nodes <paths>` gives the quorum builder its own checkout, with `[harness.sandbox]`/`allowed_roots` rooted at the worktree.

### `nool checkpoint <label>` (alias `nool release`)
Mark the current state as a checkpoint or release label. A semver-shaped label (X.Y.Z) is treated as a release on a release branch.
- `-i, --include <thread>` / `-e, --exclude <thread>`: select threads.
- `-c, --channel <channel>`: release channel/label.
- `-s, --solidify`: save immediately instead of leaving pending.

### `nool tag <name>`
Create a semantic tag for the current state. `-s, --solidify` saves immediately.

### `nool promote <knot_id>`
Promote a local knot to the next saved state.
- `-p, --push <remote>`: push to a remote after promotion.

### `nool approve`
Record an approval or rejection for a knot or thread.
- `--id <id>`, `-c, --comment "<note>"`, `-r, --reject`, `-s, --solidify`.

---

## 5. Remote Synchronization

### `nool push <remote>`
Push all unsent changes to a remote (URL or name).

### `nool pull <remote>`
Pull changes from a remote and apply them locally.

### `nool sync <remote>`
Synchronize with a remote in both directions.

### `nool merge <branch>`
Merge an incoming git branch (e.g. from a PR): ingest divergent commits into candidate Knots, converge commutatively, check for semantic conflicts, and run the gate stack.

### `nool daemon`
Background synchronization daemon (signs git-mirror commits, single-instance lock).
- `nool daemon start | stop | status`.

### `nool bridge`
Manage the Git bridge and large-file storage.
- `lfs`, `mirror-repair`, `recover-ledger`, `status`, `add-remote`, `remove-remote`, `watch`.
- `nool bridge reindex`: fill the ledger's commit-to-knot index from the mirror pointer's history, the legacy per-knot refs and the detached chains, once. Lookups (`bisect`, `try`, `compare`) answer from the index afterwards instead of walking history. Run it after upgrading a repository that predates the index, or after `recover-ledger`.
- `nool bridge prune-knot-refs`: delete `refs/knots/*` refs whose commit the mirror branch already carries. Branch knots no longer mint a ref each (one ref per commit made every push, fetch and hook run scale with history); detached knots keep theirs. Lists by default; `--apply` deletes locally, `--remote` also on that remote.

**The mirror layout (7.1.x).** Each branch owns its pointer `refs/nool/mirror/<branch>`, moved in the same ref transaction as the solidify commit; `refs/nool/git-mirror` remains a read-only alias of the default branch. Detached knots (attestations) append to per-writer chains `refs/nool/detached/<writer>`. `nool push` sends the checked-out branch, then its pointer, then the legacy refs and chains as separate atomic groups; `nool pull` ingests incrementally from the recorded tip.

---

## 6. Status, Health & Verification

### `nool status`
Show current DAG heads, pending proposals, and thread state.
- `--json`: structured output for automation.
- `--unstaged`: semantic diff of unstaged changes.
- `--limit <n>` / `--no-threads`: control output.

### `nool doctor`
Release-readiness and repository health checks.
- `--strict`: treat warnings as release-blocking.
- `--fix [--git-fallback]`: auto-repair issues where possible.
- Scope: `--fs-only`, `--semantic-only`, `--artifacts`, `--architecture`; add `--json` for automation.
- `--fix --heads-only`: only consolidate fragmented DAG heads into one merge knot, skipping the prune of replay-rejected knots. Use this when the rejected set is large enough that a blanket prune would delete real, still-referenced history (e.g. active tasks).

- `--traceability [--since <n>d|h|m|s] [--strict] [--json]` — governance report over the landed work knots in the window: each one's intent, test note, gate verdict and trusted attestations. Exit 2 with `--strict` when any knot lacks an intent or test note or carries a failed attestation. Streams the ledger from the window start, so it is cheap on large repositories.

### `nool verify`
Run structural invariants against the current or planned state.
- `--target <id>`: knot or plan ID.
- `--all`: run all invariants.

### `nool validate`
Run background validation for pending fast-mode knots (see §3).

---

## 7. Query & Analysis (Agent-First)

### `nool context <task>`
Assemble budgeted context for a task in one call: matching entities, their blast radius, prior findings, and optional AST skeletons — the one-call replacement for the resolve-intent → query context → blast-radius → Read dance, inside an explicit token budget.
- `--budget <n>` (default 2000): token budget for the assembled packet.
- `--skeletons`: include AST skeletons for matched source files (bodies elided).
- `--cite`: emit `path:start-end` citations instead of code.
- `--include-history`: include entities that are known history (superseded, removed, or externalized) instead of excluding them by default.
- `--json`: emit the packet as JSON.

### `nool ground <task>`
Assemble a single ranked, budgeted GroundingPacket for a routing-change-style task (e.g. "which command handles the deploy subcommand"): matching entities, blast radius, prior findings — via the same composition `nool context` uses — plus the causal chain (`why`) for the most relevant prior knot. Accepts a natural-language description or a knot ID.
- `--budget-tokens <n>` (default 2000): token budget for the assembled packet.
- `--json`: emit the packet as JSON.

### `nool query`
Runtime semantic queries.
- `nool query context <id> [--depth <n>]`: token-optimized semantic neighborhood for an entity/feature.
- `nool query resolve-intent "<query>"`: search knots matching a natural-language intent.
- `nool query search "<text>"`: semantic + fuzzy hybrid search.
- `nool query blast-radius <ids>`: compute causal blast radius.
- `nool query neighbors <id>`: show causal neighbors.
- `nool query recent-knots [--thread <name>]`: recent knots.
- `nool query materialize <ids>`: reconstruct content for knots.
- `nool query runtime-evidence`: search runtime evidence sidecars.
- `nool query validate <paths>`: validate files without proposing.

---

## 8. History & Explanation

### `nool log`
Show the canonical replay log.
- `--skip <n>` / `--limit <m>`: pagination.
- `--intent <intent>` / `--fuzzy <fuzzy>`: filters.
- `--thread <name>`: thread filter.
- `--json`: structured output.

### `nool why <node_id>`
Walk the causal chain of a Knot.
- `-d, --depth <n>` (default 3), `--json`.

### `nool explain <subject>`
Explain identities, dependencies, and reasons.
- `-c, --closure`: full dependency closure.

### `nool changelog`
Show the semantic changelog. `-s, --since <hlc>` filters by timestamp.

### `nool compare <left> <right>`
Compare changes between threads or releases. `--shared` also prints common knots.

### `nool diff [left] [right]`
File-content differences between two states (defaults: working tree vs HEAD).

### `nool link <knot_id>`
Link existing history to intent or thread metadata. `-i, --intent`, `-t, --thread`.

### `nool pluck <thread_name>`
Remove a thread from the active timeline while retaining history.

### `nool rewind`
Rewind workspace state (see §1).

---

## 9. Tasks, Threads & Bugs

### `nool task`
Track units of work. Lifecycle: Open → Claimed → InProgress → InReview (QA) → Done.
- `nool task create --name "<name>" [--acceptance "<criterion>"] [--solidify]`: create with explicit or generated draft acceptance criteria.
- `nool task list` / `board` / `inbox` / `mine` / `show --id <id>`.
- `nool task pick --id <id>`: claim. `nool task start --id <id>`: Claimed → InProgress.
- `nool task qa --id <id>`: submit for QA review (criteria verified here).
- `nool task finish --id <id>`: mark done (advisory unless criteria review is required).
- `nool task verify-done --id <id>`: mark verified-done directly from any state.
- `nool task assign`, `relate`, `block`, `cancel`, `telemetry --id <id>`.
- `nool task list-github` / `import-github`, `list-jira` / `import-jira`: import issues as tasks.
- `nool task criteria inbox|set|review`: acceptance-criteria management (advisory, never blocks execution).

### `nool thread`
Manage intent threads.
- `nool thread create | list | show [--full] | status | chat | handoff`.

### `nool bug`
Track bugs.
- `nool bug report | investigate | link | list | show | wont-fix | duplicate`.

### `nool inbox`
Show the notification inbox.

---

## 10. Knowledge & Discovery

### `nool learn`
Record a knowledge finding.
- `--about <subject>` (alias `--topic`), `--kind <root_cause|finding|dependency_insight|reasoning_note>`, `--content "<text>"`, `--knot <id>`, `--supersedes <id>`.

### `nool findings [subject]`
Retrieve recorded findings for a file, thread, or topic.
- `--all`, `--limit <n>`, `--json`, `--kind <kind>`, `--sort <order>`.

### `nool enrich "<query>"`
Recall knowledge; on a miss (below the memory similarity threshold), run bounded self-healing enrichment and record the gap.
- `--soul <name>`, `--max-turns <n>` (default 6).

### `nool discover learnings | similar`
Extract learnings and find similar work (see §2).

---

## 11. Planning, Review & Evidence

### `nool plan`
Compute semantic operations (RFC-0001).
- `nool plan replay --target <ids>`: ops to reach a target semantic state.
- `nool plan pluck`: plan a selective undo.
- `nool plan merge`: plan a merge of divergent branches.
- `nool plan status`: current plan status.

### `nool apply`
Execute an approved or draft semantic plan.
- `--plan-id <id>` (required), `--approval-id <id>`, `--staging-root <dir>`.

### `nool review <target>`
Review candidate changes (plan ID or thread name) before they are finalized.

### `nool pr`
Render nool's review context for a GitHub PR. Nool renders the facts; CI decides how to post them.
- `nool pr summary --base <base-knot-id> [--thread <name>]`: render the knots not yet in `--base` as a markdown review-context summary (semantic signal, blast radius, findings, and recorded justifications per knot), suitable for a CI job to post as a PR comment, e.g. `nool pr summary --base <id> > summary.md && gh pr comment <PR> --body-file summary.md`.

### `nool evidence`
Show evidence for why a transition was accepted or rejected.
- `nool evidence plan|knot|merge <id>`, `export`, `attach-runtime <file>`.

### `nool attest`
Record or inspect verification attestations — signed verdicts (tests, scans, reviews) about a knot. An attestation is exported ref-only: it is never a commit on the branch, and travels on this writer's `refs/nool/detached/<writer>` chain.
- `nool attest record --checker <who> --assertion <what> --verdict <pass|fail|error>`: mint one. Target with `--target <knot>` or `--from-head` (reads `.nool/knot.bin`, so CI needs no ledger). `--scope <check|selected|full>`, `--detail <text>`, `--evidence <file.json>` (stored locally, referenced by hash), `--environment <local|ci|...>`, `--push <remote>`.
- `nool attest show <knot>`: list what has been attested about a knot, with trust resolved from `[attestation] checker_keys`.
- `nool attest obligations <knot>`: the proof-obligation matrix — which assertions the knot's touched paths require (`[attestation.obligations]`) and which are satisfied, missing, or failed. `nool push` refuses unmet obligations when `[attestation] enforce = "block"`.
- `nool attest sweep`: move every `Done`/`InReview` task whose landed knot now carries the required trusted attestations to `VerifiedDone`.
- `nool attest wait <knot>`: block until a trusted `pass` exists, fetching from a remote between polls.

In CI, with `NOOL_IDENTITY_KEY` set:
```bash
nool attest record --from-head --checker github-actions --assertion unit-tests \
  --verdict pass --scope full --push origin
```

### `nool test`
Impact-scoped test selection, for CI.
- `nool test select --knot <id>`: print the tests the knot's changes affect, using the same dependents-closure selection `nool propose`'s ghost-run uses, so a CI run matches what the change actually landed with. Deterministic for a given knot and semantic index.
- Selection reads `nool discover`'s entity graph. A checkout that only restores plain git history and never runs `nool discover` has an empty or stale index; selection notices and falls back to the full suite with an explicit reason rather than silently under-selecting. CI that wants real selection needs the index present at checkout.

### `nool audit`
Produce a compliance report.
- `report`, `export` (with framework validation), `steering`.

---

## 12. Architecture Recovery, Assertions & Export

### `nool architecture`
Review recovered architecture and record accept/reject decisions.
- `nool architecture review`: list candidates awaiting review, highest downstream leverage first.
- `nool architecture accept <id>`: confirm a candidate; records it as an Assertion Knot.
- `nool architecture reject <id>`: reject a candidate; records a counter-assertion, keeps the observation.

### `nool assert <subject> <predicate> <object>`
Record architectural judgment as a durable, revocable assertion Knot — how curated knowledge enters the semantic architecture model, signed and replayable, unlike derived facts which are rebuilt wholesale.
- `--justification "<text>"` (required): why this is true.
- `--inheritance <rebind|lineage>` (default `rebind`): whether the assertion survives split/merge as well as rename/move.
- `--revokes <knot_id>`: an earlier assertion this one supersedes.
- `-s, --solidify`: save immediately instead of leaving it pending for review.

### `nool export`
Serialize non-canonical views of the semantic architecture model.
- `nool export c4`: exact-level C4 architecture view as deterministic JSON.
- `nool export rdf`: entity graph as RDF 1.1 Turtle, with W3C PROV-O provenance.
- `nool export jsonld`: entity graph as JSON-LD 1.1, with W3C PROV-O provenance.
- `nool export cypher`: entity graph as a Cypher script (Neo4j / openCypher).
- `nool export graphml`: entity graph as GraphML.

---

## 13. Usage & Token Analytics

### `nool usage`
Show token usage, budgets, and thread costs. With no subcommand, prints per-provider/per-model token consumption.
- `nool usage usage | budget-set | budget-status | analytics | agent | thread | dashboard`.

### `nool insights`
Show project insights, blast radius stats, and time-saved metrics.
- `nool insights justifications | loops | conflicts | consolidate`.
- `-f, --format <json|toon|html|text>`, `--audience <exec|leader|manager|engineer|agent>`.

---

## 14. Multi-Agent (v5.0 Metaharness)

### `nool harness`
Show which execution harnesses (multi-agent backends) are available and ready.
- `nool harness list`, `nool harness eval-gates`.

### `nool agent`
Inspect declarative agent specs.
- `nool agent init`: scaffold a spec YAML from the built-in template.
- `nool agent list`: list and validate specs in a directory of `*.yaml`.
- `nool agent validate <file>`: validate one spec and print resolved config.
- `nool agent run --spec <file> --intent "..."`: run one agent (budget-gated, journaled).

### `nool fleet`
Plan and run fleets of sovereign agents over a goal.
- `nool fleet plan --task id=paths [--width <n>|auto] [--run <builder.yaml>] [--isolation shared|worktree]`: preview the NodeID-disjoint parallel wave partition (no spend). `--width` caps concurrency per wave: an integer, or `auto` to take the minimum of usable cores (minus `[fleet] reserved_cpus`), available memory over `memory_per_agent_mb`, and the remaining `budget_usd_per_day` over the builder's `per_run_usd`; the printed `Ceiling:` line and the `--json` `width_decision` name which one bound it. Defaults to `[fleet] width`, else `auto` (bounded by `[fleet] max_width`, 16). `--run` executes it, giving each task its own try branch, agent id, and lease. `--isolation worktree` gives each task its own `.nool/try/<run>-<task>/` checkout instead of the shared tree; `--overlap parallel` (worktree only) runs overlapping tasks at once and orders their promotes with `--after`.
- `nool fleet capacity [--width <n>|auto] [--builder <spec>] [--json]`: report the wave-concurrency ceiling for this machine and budget without planning anything — each probe's width (CPU, memory, budget), which one binds, and the `[fleet]` settings that produced it. `--builder` supplies the `budget.per_run_usd` that lets the daily allowance bind.
- `nool fleet advise`: recommend Single / ParallelIndependent / CentralizedQuorum architecture from measurable task signals, before spending.
- `nool fleet start --builder <b.yaml> --reviewer <r.yaml> --intent "..." [--isolation shared|worktree] [--nodes <scope>]`: builder produces work, cross-vendor reviewer quorum judges it with dynamic quorum. `--isolation worktree` roots the builder's `fs`/`shell` sandboxes and `[harness.sandbox]` write whitelist at its own checkout; `--nodes` files its lease scope (exit 3 on a foreign hold). Quorum approval promotes the branch; a block leaves it as the review surface.

### `nool persona`
Create and manage persistent model personas — budget-capped identities.
- `nool persona create | status | edit | list | run`.

### `nool council`
Run a configured consortium (multi-model LLM council) on the working-tree diff and print each member's verdict plus the quorum/veto outcome.
- `--consortium <name>` (required), `--diff-file <file>`, `--json`.

### `nool explore`
Inspect the structured exploration tree — candidate directions, evidence, distilled insights.
- `nool explore view | frontier | constraints | open | dispatch | record | prune | merge | run`.

### `nool order render`
Render a model-facing TOON work order for an agent (read-only).
- `--agent <spec.yaml> --intent "..." [--path p] [--node id] [--forbid p] [--hash]`.

### `nool playbook`
Inspect declarative playbooks — named ordered recipes of existing verbs, gated by steering.
- `nool playbook list | plan | run | cost | audit | suggest`.

### `nool steer`
Intervene and steer a role-based checkpoint.
- `--point <pre-propose|post-propose|pre-solidify|pre-push|doctor|goal-complete>`, `--role <...>`, `--action <...>`, `--target <id>`, `--value <...>`, `--attestation <...>`, `--cascade <id>`.

### `nool eval`
Score a recorded soul/agent run against a named eval set.
- `nool eval set-create | set-list | set-show | run | report`.

### `nool order`
Render the model-facing TOON work order for an agent (see above).

---

## 15. Workspace (Fractal Multi-Project Coordination)

### `nool workspace`
Coordinate a polyglot workspace of nested Nool projects (Org→Dept→Team→Project).
- `nool workspace init`: generate `workspace.toml`, pre-populated with discovered projects.
- `nool workspace status`: project tree, declared edges, order.
- `nool workspace doctor`: reconcile declared config vs discovered projects.
- `nool workspace goal [--decompose <target>=<task>]`: build a goal (fan-out or absorb).
- `nool workspace goals | goal-status`: list goals / per-project completion.
- `nool workspace insights | telemetry`: roll up insights / cost across children.
- `nool workspace visualize`: visualize the Org→Dept→Team→Project tree.
- `nool workspace pull | sync`: propagate DAG changes across children in dependency order.
- `nool workspace console`: roll-up web console (port 4002).

---

## 16. Governance & Configuration

### `nool config`
Manage system configuration.
- `nool config init-governance`: scaffold a team governance config (org roles, responsibilities, controls, DSL policies) with sensible defaults and inline guidance. Prints to stdout by default; `--write` appends to `nool.toml`.
- `nool config show`: show the effective configuration after project, workspace, and organization layers resolve.
- `nool config explain <key>`: explain which layer an effective configuration value came from.

**LLM judgment gating.** The `[gating]` block carries a judgment tier that can escalate a decision to a human role instead of silently blocking or allowing it: `judgment_blocking`, `judgment_tools`, `judgment_budget_usd`, `judgment_escalation_blocking`, `judgment_escalation_role`, `judgment_escalation_dir`, `judgment_retain_days`, and `judgment_backlog_limit`. Resolve a raised checkpoint with `nool steer`, or settle a whole parent→child approval chain at once with `nool steer --cascade <id> --action approve|reject`.

### `nool hooks`
Install or uninstall the active coding-agent guard: git history-verb blocking plus session context that redirects raw git to Nool equivalents.
- `nool hooks install`: install the guard for a supported coding agent (currently `claude`, i.e. Claude Code).
- `nool hooks uninstall`: remove exactly what `install` added, leaving any other hooks or settings untouched.

### `nool commit-template`
Validate and preview the effective enterprise commit-message template.

### `nool admin`
Manage account settings, plugins, and billing.
- `account`, `plugin`, `pack` (verify/install governance packs), `gc`, `channel`, `team`, `train-dict`, `reconcile`, `reindex-graph`, `contract`.

### `nool audit`
Produce a compliance report (see §11).

---

## 17. Debugging & Root Cause

### `nool debug`
Inspect, replay, and troubleshoot repository state.
- `nool debug replay <ref>`: interactive replay of a Git ref or agent run.
- `nool debug step | diff | edit | rerun`: inspect/constrain/replay steps.
- `nool debug blame`: find root cause (causal chain from a failure).
- `nool debug bisect`: binary-search which Knot introduced a regression.
- `nool debug blast-radius <change>`: semantic blast radius and risk analysis.

### `nool reify`
Inspect bundles and validate syntax.
- `nool reify inspect <bundle>`: inspect a reified proposal or knot.

### `nool verify`
Run structural invariants (see §6).

---

## 18. Other Utilities

### `nool prune`
Clean temporary and cached files. `-a, --all` removes all known cache and build artifacts.

### `nool untrack <paths>...`
Stop tracking file paths in Git while keeping them in the working tree.

### `nool languages`
List supported languages and their validation status. `--check-toolchains` also probes installed toolchains; `--json` for structured output.

### `nool visualize`
Visualize project evolution and artifact graphs.
- `-k, --kind <history|graph|roi|relational>` (default `history`).
- `-f, --format <tui|html>` (default `tui`), `-o, --output <file>`.

### `nool dag`
Visualize the DAG.

### `nool ui`
Launch the interactive DAG explorer (TUI). Navigate knots, parents and payloads without leaving the terminal; `nool console` is the browser equivalent.

### `nool console`
Launch the web console (and v5.0 control-plane API).
- `--serve`: headless daemon (no browser). `--port <port>` (default 4001), `--token <token>`.

### `nool reify`
Inspect bundles and validate syntax (see §17).

### `nool order`
Render the model-facing TOON work order for an agent (see §14).

### `nool changelog`
Show the semantic changelog (see §8).

### `nool completion <shell>`
Generate shell completion scripts for `bash`, `zsh`, `fish`, or `powershell`.

### `nool quick-start` (alias `nool quickstart`)
Show the quick-start guide.

### `nool guide`
Show the detailed guide and examples (e.g. `nool guide multi-agent`, `nool guide governance`).

### `nool telemetry`
Show or change whether Nool sends anonymous usage analytics (which commands run, coarse error categories, timing). Enabled by default. Separate and unrelated to `nool usage analytics`, which is LLM token-cost analytics.

### `nool feedback`
Rate Nool or give quick feedback, in response to the occasional non-blocking prompt shown after using it for a while.

### `nool version`
Print version information.

### `nool upgrade`
Upgrade the Nool CLI to the latest version.

### `nool uninstall`
Uninstall the Nool CLI and remove local identity keys.

---

*Last updated: September 10, 2026 for Nool v7.3.0*
