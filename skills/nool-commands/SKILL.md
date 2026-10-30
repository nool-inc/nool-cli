---
name: nool-commands
description: Expert guidance for the Nool CLI (v7.23.0) in any Nool-tracked repository - landing changes (propose/solidify), reading code by symbol, multi-agent coordination (announce, leases, try worktrees), tasks, bugs and the knowledge ledger, debugging, and the nool code agent harness. The full command reference is Commands.md beside this file.
license: Apache-2.0
metadata:
  version: "7.23.0"
  author: nool-core-team
compatibility: Requires nool CLI v7.23.0+
allowed-tools: bash(nool *)
---

# Nool CLI (v7.23.0)

Nool is a semantic version control system: the source of truth is the **Knot DAG**, not text, and every landed change is a signed **knot** mirrored to Git. In a Nool-tracked repository (confirm with `nool status`), use `nool` for version control, tasks, bugs and debugging instead of raw `git`; raw history-writing git verbs are refused there.

This file is the short version, the parts an agent needs on every task. **`Commands.md` in this directory documents every command and flag**: read the section you need rather than guessing, or run `nool <command> --help`. `--compact` (concise output) and `--quiet` work on every command; prefer `--json` when you parse output.

## Exit codes

- `0` success.
- `1` error: read the message.
- `2` blocked by a gate or policy: fix the cause; retrying unchanged never helps.
- `3` contention: a lease, an overlapping announcement or a promote conflict. Back off, wait (`nool announce wait <scope> --timeout-ms N`) or narrow your scope.
- `4` unavailable: the command belongs to a higher edition (Community / Team / Enterprise) or needs a lease. Activate one with `nool admin account activate <key>`, or install the edition with `nool upgrade --edition team|enterprise`.

## Read code by symbol, not by file

```bash
nool outline <file>                  # one line per symbol: start-end kind owner::name (or Markdown headings)
nool show <file>::<Type>::<method>   # one symbol's source with line numbers and doc comment
nool show docs/x.md::Setup           # one Markdown section
```

Both commands parse the file live and need no index. Use them instead of reading a large file whole, and instead of `grep -n` followed by a ranged read. `nool show` prints the outline when the name is not found. For orientation on a task, use `nool context "<task>" --cite`, which returns `path:start-end` citations within a token budget, or `nool ground "<task>"`. Before a broad edit, run `nool debug blast-radius <path>`.

## Land a change (the golden path)

```bash
nool status --compact
nool announce intent --intent "<what>" --target-nodes <paths>   # shared tree: declare what you touch
nool discover conflicts <paths>                                   # exit 3 = someone else holds it
nool propose --path <file> [--path <file>…] --intent "<what>" --solidify
```

- **Name your files.** `--all` sweeps every dirty file, including other agents' unfinished work. Use it only in a `nool try` worktree or a tree nobody else edits. To land only some hunks of a shared file: `--hunks f.rs#10-24`.
- **Land code that builds.** Full validation (the default) runs the language's integrity driver ("ghost run": build plus the selected tests) on HEAD plus your files, in a private worktree. Other sessions' uncommitted edits never affect it. A refusal names the failing tests. An identical re-propose (same HEAD, same files, same toolchain) reuses the earlier passing verdict instead of rebuilding.
- `--fast` defers validation and is an explicit, unsafe opt-in. Settle the debt with `nool validate --all` before you stop.
- Without `--solidify`, `propose` only stages a candidate; seal your own with `nool solidify --candidate <id>`. A plain `solidify` takes the oldest candidate, which may belong to someone else.
- A broad change needs `--justification "<why>"`. A steer gate prints the exact `nool steer …` command that clears it.
- A long landing need not block you: `nool propose … --solidify --detach` prints an id at once, and `nool land wait <id>` later prints one verdict line with the landing's exit code. Do not poll a log in a sleep loop.
- Replicate with `nool push <remote>` / `nool pull <remote>`; accept a branch with `nool merge <branch>`.

## Choose the verb by situation

| Situation | Do |
|---|---|
| Ordinary change you keep | `propose --path … --solidify` |
| Work you may throw away, or need isolated | `nool try new <name> --worktree --nodes <paths>` → work there → `try impact` → `try promote` or `try discard` |
| Other agents may touch the same paths | `announce intent --target-nodes` → `discover conflicts` → propose |
| Several independent tasks in parallel | `nool fleet advise` → `fleet plan` → `fleet start` |
| Reach the agent you collided with | `nool msg send --to <agent> "<message>"` (`nool msg inbox` to read yours) |

Try worktrees live at `.nool/try/<name>/`. Run `propose --try-branch <name>` (or `--all`) inside one. `nool try rebase <name>` merges main's drift in before `promote`. Leases: `nool lease status`, `nool lease wait <paths>`, `nool lease release <id>`; `--as-agent` or `NOOL_AGENT_ID` give co-located agents distinct identities. An announcement without `--target-nodes` protects nothing.

## Tasks

Lifecycle: Open → Claimed → InProgress → InReview → Done.

```bash
nool task create --name "…" --desc "…" [--acceptance "<criterion>"] --solidify
nool task inbox | mine | board | show --id <id>
nool task pick --id <id> -s ; nool task start --id <id> -s
nool task finish --id <id> --landed-knot <knot id, short id ok> --test-note "<what proved it>" -s
```

A task change without `--solidify` is only staged. `nool task criteria inbox|set|review` manages acceptance criteria. Edit, cancel, relate, block, assign, import and sync are in Commands.md §9.

## Bugs and knowledge

```bash
nool bug report --title "…" --reproduction "…" -S   # then: nool bug link <bug> --fix <knot>; nool bug list|show
nool learn --about <subject> --content "…"        # record what you learned (a signed Finding knot)
nool findings <subject>                           # recall it later; nool findings inbox for candidates
```

Record non-obvious discoveries as you go: the next agent recalls them instead of re-deriving them.

## Debug and inspect

```bash
nool log | nool why <id> | nool thread show <name> --full | nool diff <a> <b>
nool query history <paths>          # who changed these paths, when and why
nool debug replay HEAD | blame | bisect
nool doctor [--strict] [--fix]
```

Undo safely with `nool plan pluck` (preview) and then `nool pluck <id>`, which includes descendants. `nool rewind --knot <id>` returns the workspace to a checkpoint.

## Agent harness and fleets

`nool code` is Nool's own coding agent (TUI, `--exec` headless, ACP). It lands its work through the same governed path and has `outline`/`read_symbol` tools. Fleets, personas, providers, councils, playbooks and evaluation are in Commands.md §14 and §19.

## Tips

- Unsure of a flag? `nool <command> --help`, or the matching Commands.md section.
- Old names still work as aliases: `soul`, `inquiry`, `flow` → `persona`, `explore`, `order`.
- `[reification].enabled` and `[analysis].ast_enabled` in `nool.toml` toggle the integrity driver and the syntax precheck; the Aram policy gate always runs.
- Release: `nool doctor --strict` → `nool checkpoint X.Y.Z`.
