---
tier: epic
title: Reorganize bob's command tree with sectioned help, bob task, and bob pomodoro
goal: '`bob -h` and `bob <TAB>` present 14 workflow-ordered commands in five sections
  plus a collapsed Capture protocol section; `bob task {archive,reconcile,reroll}`
  and `bob pomodoro {notify,status,tmux}` replace five opaque top-level names; every
  old spelling keeps working forever as a silent alias with byte-identical behavior;
  and the README, docs, tests, chezmoi, and bob-plugins all teach the canonical names.

  '
phases:
- id: sectioned-help
  title: Sectioned help, help routing, and completion parity
  depends_on: []
  size: medium
  description: 'sectioned-help: replace CompletionTier with workflow Sections, render
    sectioned root help (-h collapses the capture protocol, --help lists it), cut
    the examples, add the argv alias-rewrite table and `bob help <path>` routing,
    hide `freshness seed`, label defaults, group `bob <TAB>` by section, and amend
    cli_rules.md. No command changes behavior.'
- id: command-groups
  title: bob task and bob pomodoro groups with permanent aliases
  depends_on:
  - sectioned-help
  size: medium
  description: 'command-groups: add Leaf/Group targets to the runner table, route
    `bob task` and `bob pomodoro` (bare = status), map the five old names onto canonical
    paths as silent aliases, print canonical names in every leaf''s help, diagnostics,
    logs, and commit subjects, mount the groups in completion, and add alias-parity,
    routing, and help snapshot tests.'
- id: canonical-docs
  title: README, docs, and tests teach the canonical names
  depends_on:
  - command-groups
  size: medium
  description: 'canonical-docs: rewrite the README command reference, task and Pomodoro
    sections, shims table, and migration notes; move docs/ to canonical spellings;
    migrate test invocations to canonical paths; and record the proposed follow-ups.'
- id: downstream-callers
  title: chezmoi and bob-plugins callers move to canonical names
  depends_on:
  - command-groups
  size: small
  description: 'downstream-callers: switch the chezmoi shims, tmux.conf, and obsidian
    memory note to canonical names, fix the stale highlights-ref message, retire the
    stale bob_dataview skill copies, update the bob-plugins notice string, and guard
    the rollout so no host runs new spellings on an old bob.'
proposed_by: bbugyi200.apollo.4y
create_time: 2026-10-04 07:02:04
status: wip
bead_id: bob-cli-46
---

- **PROMPT:** [prompts/202610/bob_command_tree.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202610/bob_command_tree.md)
- **BEAD:** [bob-cli-46](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-46/README.md)

# Plan: Reorganize bob's command tree

## Context

Bryan asked whether `bob` should group its sub-commands (for example under a new
`bob task`). The research report
`research:202610/bob_cli_command_tree_reorganization/bob_cli_command_tree_reorganization.md`
(read it with `sase artifact read`) answered: yes, but mostly by sectioning help rather
than nesting, plus two narrow namespaces. **Bryan accepted every recommendation**,
including its open questions as recommended: `reroll` (not `randomize`), `reconcile`
(not `hooks`), hide `freshness seed`, and keep `plan`, `ready`, and `freshness`
top-level.

Today `src/runner.rs` holds one flat, A–Z table of 28 delegates (`SUBCOMMANDS`), each
forwarding raw `OsString` args to a native module that builds its own parser.
`CompletionTier` tags 18 porcelain and 10 `capture-*` plumbing rows. Two hidden clap
subcommands (`task-status-setter`, `mark-next-tasks`) alias `task-status-hooks`. Root
help is clap's `{all-args}` plus a 29-line `AFTER_HELP` that is mostly protocol calls.
`bob help plan` prints the delegate stub `Usage: bob plan [args]...` instead of the real
help.

## Design

### Final command tree

```text
bob
├── capture [OPTIONS] [TEXT]...            unchanged; free-text grammar untouched
├── completion {bash, install, status*, uninstall, zsh}
├── freshness {list*, seed†}               † hidden from help and completion
├── gkeep {doctor, list*, login, pull}
├── highlights {clip, create, doctor, marker, scan, sync}
├── nightly                                unchanged
├── plan                                   unchanged
├── plugins {list*, sync}
├── pomodoro                               bare = status
│   ├── notify                             ← notify
│   ├── status                             ← pomodoro (bare form unchanged)
│   └── tmux                               ← tmux-pomodoro
├── projects {list, sync}
├── query                                  unchanged
├── ready                                  unchanged
├── task                                   bare = help (exit 2, like bare `bob`)
│   ├── archive                            ← move-done-tasks
│   ├── reconcile                          ← task-status-hooks, task-status-setter, mark-next-tasks
│   └── reroll                             ← randomize
├── vault-sync {run*, status}              bare = run (grandfathered writer default)
└── capture-* (10)                         unchanged names, args, and outputs; last section
                                           (* = default subcommand)
```

Membership rule for the new noun: **`bob task <verb>` rewrites task lines across the
whole vault.** Read-only reports (`plan`, `ready`, `freshness`) stay top-level; no
`task add/list/show` is invented. The capture protocol never nests under `bob capture`,
because its free-text TEXT would swallow any reserved word.

### Aliases are permanent, silent, and byte-identical

| Old spelling                                               | Canonical path         |
| ---------------------------------------------------------- | ---------------------- |
| `bob mark-next-tasks`                                      | `bob task reconcile`   |
| `bob move-done-tasks`                                      | `bob task archive`     |
| `bob notify`                                               | `bob pomodoro notify`  |
| `bob randomize`                                            | `bob task reroll`      |
| `bob task-status-hooks`                                    | `bob task reconcile`   |
| `bob task-status-setter`                                   | `bob task reconcile`   |
| `bob tmux-pomodoro`                                        | `bob pomodoro tmux`    |
| `bob_pomodoro`, `bob_notify`, `tmux_bob_pomodoro` binaries | unchanged, same leaves |

- Aliases are an argv-prefix rewrite applied to the first token after `bob`, before clap
  parses, so the clap tree has no hidden duplicates and the leaf never knows which
  spelling ran. One rewrite function serves dispatch, `bob help`, and completion.
- Preserve non-UTF-8 `OsString` args and `--` boundaries; never route through a shell
  string.
- **No deprecation output**, TTY or not: the hand-installed Mac crontab runs
  `bob task-status-hooks --retry-timeout 120` every 15 minutes and mails stderr.
- Old and new spellings produce byte-identical stdout, stderr, exit codes, and vault
  effects. Every command speaks **only its canonical path** in help, usage, diagnostics,
  log lines, replay hints, and Git commit subjects.

**Persisted identifiers that must not change** (they are data, not command names):

- The Schedule Log reason token `randomize` (`🎲 P2 randomize · in **21** (8–30) days`,
  `capture_schedule_log::randomize_reason`). bob-plugins' keep-streak classifier matches
  `/\brandomize\b/` on it.
- Recovery-record tool ids `task-status-hooks` (`task_status_hooks_write::model::TOOL`)
  and `randomize` (`RANDOMIZE_TOOL`), which name on-disk manifests and prune matching.
- Every command's JSON output (schemas and values), env vars, config keys, and
  lock/state/log file names.
- `[[bin]]` names, the embedded fallback scripts, and their output.
- Rust module names, `NativeCommand` variants, doc file names
  (`docs/task-status-hooks.md`, `docs/randomize.md`), and the "task-status hooks"
  concept name in prose.
- Immutable decision records under `sase/memory/decisions/`, which cite
  `bob task-status-hooks` and stay accurate through the alias.
- The 10 `capture-*` names, arguments, and outputs (Bob Mac Capture installs
  independently; the protocol only grows additively).

### Root help

Sections in workflow order, A–Z within each. Every listing line fits in 80 columns.
Section titles use clap's header style and command names its literal style, built as a
clap `StyledStr` with `cli_styles()` so color follows clap's TTY and `NO_COLOR`
handling. Use clap's short/long split (for example `before_help` versus
`before_long_help`) for the `-h`/`--help` difference; clap 4.6.7 has no per-subcommand
heading, so the listing is rendered from bob's table. Final `bob -h` (after
`command-groups`):

```text
Bob — command-line tools for the Bob Obsidian vault and Pomodoro workflow

Usage: bob <COMMAND>

Daily workflow:
  capture     Capture tasks, bullets, and Pomodoro commands into the vault
  freshness   Walk the tiered freshness review queue
  plan        Show today's plan budget, Today's tasks, and NEXT/PENDING lanes
  pomodoro    Show Pomodoro status, print the tmux line, or notify on completion
  ready       Show each area/project note's Ready lane against the per-note cap

Tasks and projects:
  projects    List and sync project notes via their ^prj tasks
  task        Vault-wide task maintenance: reconcile, reroll, archive

Vault:
  nightly     Run nightly maintenance: vault-sync, task archive, vault-sync
  query       Run Dataview or Tasks queries against the vault
  vault-sync  Reconcile the vault through Git (default: run) or show status

Integrations:
  gkeep       Drain the Google Keep inbox into Obsidian tasks
  highlights  Sync Highlights PDF annotations into reference notes

Setup:
  completion  Install and inspect shell completion for bob
  plugins     List and deploy Bob's custom Obsidian plugins

Capture protocol (Bob Mac Capture JSON endpoints; `bob --help` lists them):
  capture-complete  capture-parse  capture-pomodoro-name  capture-pomodoros
  capture-rewrite  capture-sections  capture-targets  capture-task-id
  capture-task-sections  capture-tasks

Options:
  -h, --help     Print help (see more with '--help')
  -V, --version  Print version

Examples:
  bob capture buy milk @groceries   Capture a task into groceries.md
  bob capture '='                   Start the queued Pomodoro
  bob plan                          Show today's plan budget and lanes
  bob freshness                     List the review queue
  bob ready                         Show Ready lanes against the per-note cap
  bob task reconcile --dry-run      Preview task status reconciliation
  bob query --source '#project'     Print matching note paths
  bob vault-sync status --json      Print the last vault Git sync status

Run 'bob <command> --help' or 'bob help <command>' for more on a command.
```

`bob --help` differs only by: the long about (below), and a full `Capture protocol:`
section listing each endpoint with its description (its own name column). The long about
drops the now-ambiguous "Run a task with `bob <command>`":

> Bob tracks a daily Pomodoro ledger inside an Obsidian vault and keeps that vault
> synced through Git. Commands follow the daily workflow: capture, review (plan,
> freshness, ready), run Pomodoros, reconcile tasks, and nightly maintenance. Pass
> `--help` to any command, or run `bob help <command>`, for its own options.

The two protocol writers say so in their verb: `capture-task-id` → "Write a block ID
onto an open capture task"; `capture-pomodoro-name` → "Write a name onto an open unnamed
Pomodoro". Shorten other protocol descriptions only where needed for 80 columns (for
example `capture-targets` → "List inbox, area, and active project capture routes").

### Group behavior

Groups owned by the runner (`task`, `pomodoro`) are pure routing: no options of their
own and no shared parent options (in particular no `task --bob-dir`; `reroll`
deliberately lacks `--bob-dir` because its Git sync binds to `BOB_DIR`).

- `bob <group> <member> ...` runs the member with the rest of argv.
- `bob <group>` with no args: `pomodoro` runs `status`; `task` prints its help to stderr
  and exits 2, exactly like bare `bob`.
- First token decides. A group default receives the args only when the first token is
  absent or starts with `-` and is not `-h`/`--help`. A bare word that is not a member
  gets clap's unrecognized-subcommand error with suggestions (status takes only flags,
  so a bare word can only be a typo). Rule: **a bare group may default only to a
  flag-only, read-only member**; `vault-sync` (bare = `run`) is grandfathered and
  labeled.
- `-h`/`--help` as the first token shows the group help; `bob pomodoro status --help`
  shows the leaf help.
- `help` routing: `bob help [PATH...]` and `bob <group> help [MEMBER]` become
  `bob PATH... --help`. PATH is alias-rewritten, then validated against the composed
  command tree (the same `tree()` completion uses, hidden members included): every token
  must name a subcommand at its level, otherwise exit 2 with an unrecognized-subcommand
  error naming the token. `--help` is appended right after the validated path, so it can
  never be swallowed as free text (`bob help capture buy milk` must exit 2 and write
  nothing). Bare `bob help` prints `bob --help`.

Group help is rendered by clap with the same styles, so it reads like
`bob vault-sync --help`:

```text
Vault-wide task maintenance: reconcile, reroll, archive

Every `bob task` command rewrites task lines across the whole vault.

Usage: bob task <COMMAND>

Commands:
  archive    Move done and canceled tasks into done/ archives and repair links
  reconcile  Reconcile task statuses from the Pomodoro ledger and dependencies
  reroll     Re-roll due prioritized tasks within their priority windows

Options:
  -h, --help  Print help

Examples:
  bob task reconcile --dry-run   Preview task status reconciliation
  bob task reroll --dry-run      Preview re-rolling due prioritized tasks
  bob task archive               Archive done and canceled tasks

See also: bob plan, bob ready, bob freshness, bob projects, bob query
Run 'bob task <command> --help' for more information on a command.
```

```text
Show Pomodoro status, print the tmux line, or notify on completion

Usage: bob pomodoro [COMMAND]

Commands:
  notify  Notify when the current Pomodoro is complete
  status  Show the current Pomodoro status (default)
  tmux    Print the Pomodoro status and plan meter for tmux

Options:
  -h, --help  Print help

Bare `bob pomodoro [-d] [-s] [-v]` runs `bob pomodoro status`.

Examples:
  bob pomodoro                 Show the current Pomodoro status
  bob pomodoro -s              Include a stale open Pomodoro
  bob pomodoro tmux            Print the tmux status-line segment
  bob pomodoro notify 30 300   Check every 30 s; after notifying, wait 300 s

Run 'bob pomodoro <command> --help' for more information on a command.
```

Module-owned groups mark their default member with ` (default)` in their own help:
`completion status`, `freshness list`, `gkeep list`, `plugins list`, `vault-sync run`.

### Completion

`bob <TAB>` groups root candidates by the same sections, using lowercase group labels
(`daily workflow`, `tasks and projects`, `vault`, `integrations`, `setup`,
`capture protocol`), sections in order and A–Z within. Members of any group stay under
`commands`. Aliases and `help` are never offered, but completion of a typed alias works
(`bob randomize --l<TAB>` offers `--level`). `bob pomodoro -<TAB>` offers status's
`-d/-s/-v`. `freshness seed` is not offered.

### Rules recorded in cli_rules.md

Replace `sase/memory/cli_rules.md`'s body with (keep its frontmatter):

```markdown
# CLI Rules

When adding or changing CLI subcommands or options:

- Make `-h|--help` output excellent: clear, complete, consistent, and easy to scan.
- Keep options sorted alphabetically. Keep subcommands alphabetical within each help
  section; `bob --help` orders its sections by the daily workflow (Daily workflow, Tasks
  and projects, Vault, Integrations, Setup, Capture protocol), and `bob <TAB>` uses the
  same sections as completion groups.
- Give every public long option a short alias; this does not apply to internal
  subprocess arguments.
- Prefer beautiful, colored output over black-and-white output when color improves
  readability.
- Never hard-rename or remove a command: the old spelling becomes a permanent hidden
  alias with byte-identical behavior and no deprecation output. Help, diagnostics, logs,
  and commit subjects print only the canonical path.
- Nest only under a noun with a stated membership rule. `bob task <verb>` is reserved
  for commands that rewrite task lines across the whole vault; read-only reports stay
  top-level.
- A bare group may default only to a flag-only, read-only member; `vault-sync` (bare =
  `run`) is the grandfathered exception and is labeled in help.
- Never add a subcommand under `bob capture` (its free-text TEXT would swallow it); the
  `capture-*` protocol names are frozen siblings that may only grow additively.
```

## Phase sectioned-help: sectioned help, help routing, and completion parity

No command changes behavior in this phase; only help, completion, and the hidden-alias
mechanism change.

1. **Sections in the table** (`src/runner.rs`). Replace `CompletionTier` with
   `enum Section { DailyWorkflow, TasksAndProjects, Vault, Integrations, Setup, CaptureProtocol }`
   with `title()` ("Daily workflow", …, "Capture protocol") and `completion_group()`
   (lowercase title). One `SECTIONS` order constant drives help and completion. Give
   every row a `section`. Interim placement for the commands that move in
   `command-groups`: `notify` and `tmux-pomodoro` under Daily workflow (beside
   `pomodoro`); `move-done-tasks`, `randomize`, and `task-status-hooks` under Tasks and
   projects. Order the table by section, then name; replace
   `subcommands_are_sorted_alphabetically` with a test that sections are contiguous, in
   `SECTIONS` order, and A–Z within.
2. **About strings.** Set the final root descriptions from the Design section for every
   command that stays top-level (`capture`, `freshness`, `plan`, `ready`, `projects`,
   `query`, `vault-sync`, `plugins`, …). Interim `nightly`: "Run nightly maintenance:
   vault-sync, move-done-tasks, vault-sync". Apply the two protocol-writer descriptions
   and any 80-column shortening.
3. **Alias rewrite.** Replace `HIDDEN_SUBCOMMAND_ALIASES` (hidden clap subcommands) with
   `ALIASES: &[Alias]`, `Alias { from: &'static str, to: &'static [&'static str] }`,
   applied to the first token after `bob` (after the `__complete` intercept, before
   clap). Interim entries: `task-status-setter` and `mark-next-tasks` →
   `["task-status-hooks"]`. Expose it as one function completion can reuse.
4. **`bob help` routing.** `disable_help_subcommand(true)` at the root and implement the
   `bob help [PATH...]` rewrite and validation from the Design section. `help` is no
   longer listed.
5. **Root help rendering.** Render the sectioned listing from the table per the Design
   section (interim names for the five moving commands; the shared name column widens to
   fit `task-status-hooks` until `command-groups`). `-h` collapses the protocol to the
   pointer plus names wrapped at 80 columns in table order; `--help` lists each endpoint
   with its description. Update `LONG_ABOUT` and the footer.
6. **Examples.** Cut `AFTER_HELP` to the eight examples in the Design section, with
   `bob task-status-hooks --dry-run` as the interim reconcile example. Each removed
   `capture-*` example must appear in that endpoint's own `--help` examples (add where
   missing); confirm every other removed example (`completion install`, `gkeep`,
   `gkeep pull --dry-run`, `freshness list -f json`, `randomize --dry-run`, `ready -a`,
   `highlights create`, `highlights scan --dry-run`, `move-done-tasks --threshold 10`,
   `plan -f json`, `plugins list`, `projects list`, `capture '@dev:foobar' …`) appears
   in its command's own help, adding any that do not.
7. **Defaults and hidden seed.** Append ` (default)` to the default member's about in
   `completion`, `freshness`, `gkeep`, `plugins`, and `vault-sync`. Hide
   `freshness seed` (`.hide(true)`): drop it from freshness examples and the root about;
   rewrite the long-about seed sentences to say the hidden `seed` subcommand stamped the
   one-time cutover and must not be re-run. It stays callable with all its guards
   (`bob freshness seed --help`, `--dry-run`).
8. **Completion presentation** (`src/native/completion/present.rs`). Group root
   candidates by `Section` using `completion_group()`, in `SECTIONS` order and table
   order within; keep `commands` for non-root levels. Update the module doc comment
   (rule 6) and `docs/completion.md` (group list and example candidate lines).
9. **Memory.** Using `/sase_memory_write` (this approved plan authorizes it), replace
   the body of `sase/memory/cli_rules.md` with the text in the Design section, then run
   `sase memory init`.
10. **README.** Group the Commands table by help section, in help order, and say that
    the `capture-*` rows form the Capture protocol section. Keep the five moving
    commands under their current names; `canonical-docs` renames them. Update the
    paragraph about hidden aliases to reflect the rewrite table.
11. **Tests.**
    - Replace `top_level_help_lists_commands_alphabetically_with_examples` with exact
      piped snapshots of `bob -h` and `bob --help` (no ANSI), and a test that every
      listing line is at most 80 columns.
    - `bob help <cmd>` is byte-identical to `bob <cmd> --help` for every root command
      and for `bob help vault-sync status`, `bob help freshness seed`, and
      `bob help task-status-setter`. Bare `bob help` equals `bob --help`.
      `bob help nosuch` and `bob help capture buy milk` exit 2, and the latter writes
      nothing.
    - The interim aliases keep routing (existing
      `task_status_hooks_help_is_native_only`) and never appear in help or completion.
    - `freshness seed` is absent from `bob freshness --help` and `bob freshness <TAB>`;
      `bob freshness seed --help` still succeeds.
    - Update `root_empty_offers_commands_then_capture_protocol` to assert section group
      labels and order.
12. **Verify.** `just all` and `just install-smoke` pass. Every existing invocation
    keeps its stdout and exit code; only help text changes.

## Phase command-groups: bob task and bob pomodoro groups with permanent aliases

1. **Table model** (`src/runner.rs`):
   `Subcommand { name, about, section, target: Target }`,
   `enum Target { Leaf(Leaf), Group(Group) }`,
   `Leaf { native_command, script_command }`,
   `Group { members: &'static [Member], default: Option<&'static str>, … help text }`,
   `Member { name, about, leaf: Leaf }`. Final rows match the tree above.
   - `task` (Tasks and projects, no default): `archive` → `MoveDoneTasks`, `reconcile` →
     `TaskStatusHooks`, `reroll` → `Randomize`.
   - `pomodoro` (Daily workflow, default `status`): `notify` → `Notify` / `bob_notify`,
     `status` → `Pomodoro` / `bob_pomodoro`, `tmux` → `TmuxPomodoro` /
     `tmux_bob_pomodoro`.
   - Root and member descriptions exactly as in the Design section (including the final
     `nightly` text).
2. **Aliases.** Final `ALIASES` exactly as the Design table.
3. **Dispatch** (`run_bob`), in order:
   1. `__complete` intercept (unchanged).
   2. Alias rewrite.
   3. `help` routing at the root and directly after a runner group.
   4. Group default insertion per the Design rules.
   5. clap parse over the nested tree: root delegates, plus group commands whose members
      are delegates. Groups disable clap's help subcommand, and bare `bob task` matches
      bare `bob`.
   6. Select the leaf from the parsed path, then apply the `BOB_CLI_USE_SCRIPT` fallback
      with that leaf's `script_command`, so `bob pomodoro tmux` still runs the
      `tmux_bob_pomodoro` asset under the fallback.
4. **Group help** exactly as the Design section's two blocks (`-h` and `--help` may
   share them).
5. **Canonical names in leaves.** Update every user-facing string:
   - `task_status_hooks::COMMAND_NAME` → `bob task reconcile` (usage, diagnostics,
     report header, retry log lines, examples).
   - `randomize::COMMAND_NAME` → `bob task reroll`. This covers usage, examples,
     diagnostics, the 🎲 header, the replay line `bob task reroll --seed …`, the commit
     subject `bob task reroll YYYY-MM-DD: N tasks in M notes`, and the long-about "one
     `bob task reroll` commit".
   - `collect_done::COMMAND_NAME` → `bob task archive` (usage, diagnostics, commit
     subject `bob task archive YYYY-MM-DD`, the `committed:` line).
   - `pomodoro` status help:
     `usage: bob pomodoro [status] [-d|--debug] [-s|--show-stale] [-v|--verbose]` /
     `bob pomodoro status -h`. tmux help: `bob pomodoro tmux` / `bob pomodoro tmux -h`,
     with the hint `Try 'bob pomodoro tmux --help' …`.
   - `notify` usage: `bob pomodoro notify [-v] PRE_CHECK_SLEEP POST_NOTIFY_SLEEP` /
     `bob pomodoro notify -h`. The positional help says "Pomodoro status checks" instead
     of "calls to bob_pomodoro".
   - Native diagnostic prefixes `bob_pomodoro:`, `tmux_bob_pomodoro:`, `bob_notify:` →
     `bob pomodoro status:`, `bob pomodoro tmux:`, `bob pomodoro notify:`. The embedded
     fallback scripts keep their own names.
   - Rename the completion descriptors (`completion_descriptor`,
     `tmux_completion_descriptor`, `notify::completion_descriptor`,
     `collect_done::completion_descriptor`) to the canonical `bob …` paths.
   - `nightly`: step label `task archive`; help workflow text names `task archive`.
   - Hints naming `task-status-hooks` → `bob task reconcile`: projects long about and
     output hint, and capture warnings in `capture/task_toggle.rs`,
     `capture/pomodoro_link.rs`, `capture/ensure_next.rs`, `capture/pomodoro_close.rs`,
     and `capture/output.rs`. These are message text only; the JSON shape is unchanged.
   - Leave every persisted identifier from the Design list untouched. Also leave
     `vault_sync::run_notify`'s subprocess argv (`bob notify`) unchanged: it runs
     whatever `bob` is on PATH, so the permanent old spelling is the most compatible,
     and its pre-existing bug is a follow-up.
   - Add a unit test: every leaf's descriptor name
     (`NativeCommand::command().get_name()`) equals `bob ` + its canonical path.
6. **Completion** (`src/native/completion/`):
   - `tree()` mounts `task` and `pomodoro` nodes with member descriptors renamed to
     member names. Add status's options at the `pomodoro` group level with
     `args_conflicts_with_subcommands(true)`, as `freshness::cli::completion_command`
     does, so the bare form completes.
   - Every completion entry point that walks the tree (`present::complete_request`,
     `context.rs`, and any other) first applies the shared alias rewrite to the words.
   - `kinds.rs` has no path entries for the moved commands today. Confirm that, and add
     entries only if a moved leaf needs one.
   - Update `tree.rs` tests to the canonical spellings: `parse_smoke`
     (`task archive -t 10`, `pomodoro notify -vv 5 10`, `pomodoro -s`,
     `pomodoro status -s`), `hand_descriptor_drift` labels, and the hidden-name list
     (now the `ALIASES` names).
   - Root `task` sits under `tasks and projects` and `pomodoro` under `daily workflow`;
     members sit under `commands`.
7. **Tests** (add `tests/cli/aliases.rs` and extend existing suites):
   - **Parity.** For every alias, run the old spelling and the canonical path with the
     same args and env on fresh fixture copies. Assert byte-identical stdout, stderr,
     exit code, and resulting files. Cover:
     - `--help`, plus one invalid-option diagnostic, for every alias.
     - `task-status-hooks`, `task-status-setter`, and `mark-next-tasks`:
       `--dry-run -f json` and one live write.
     - `move-done-tasks` on a Git fixture (reuse `tests/cli/move_done.rs` helpers).
     - `randomize --dry-run --seed …`.
     - `tmux-pomodoro` on the Pomodoro fixture.
     - `notify --help` and a bad-arity call.
     - Timer parity for `bob pomodoro`, `bob pomodoro status`, and `bob_pomodoro`, each
       with and without `--show-stale`.
   - **Fallback.** Under `BOB_CLI_USE_SCRIPT=1`, `bob pomodoro status|tmux|notify` match
     the old spellings and run the embedded assets; the legacy binaries still work.
   - **Reachability.** Every `NativeCommand` is reachable through exactly one canonical
     path. Every alias target resolves to a leaf, and no alias collides with a root
     name.
   - **Routing.**
     - Bare `bob task` prints help on stderr and exits 2.
     - `bob task -h`, `bob task --help`, `bob task help reroll`, and
       `bob help task reroll` all work.
     - `bob task nosuch` and `bob pomodoro statsu` exit 2 with a suggestion.
     - `bob pomodoro` equals `bob pomodoro status`, and `bob pomodoro -s` routes to
       status.
     - `bob pomodoro --help` shows the group help.
   - **Capture grammar guard.** `bob capture --dry-run --format json` with `parse`,
     `complete`, `tasks`, `parse invoices`, and `api design` still produces ordinary
     task captures with exit 0.
   - **Snapshots.** Final `bob -h`/`bob --help` snapshots, plus exact `bob task --help`
     and `bob pomodoro --help` snapshots, all within 80 columns.
   - **Completion.** `bob task <TAB>` offers archive/reconcile/reroll.
     `bob pomodoro <TAB>` offers notify/status/tmux, and `bob pomodoro -<TAB>` offers
     `-d/-s/-v`. `bob randomize --l<TAB>` offers `--level`. No alias is ever a root
     candidate.
   - Fix existing tests broken by canonical output strings. These include
     task-status-hooks report headers, randomize headers, replay lines, and commit
     subjects, move-done commit subjects, nightly step labels, the help markers in
     `tests/cli/help.rs`, and `pomodoro_help_documents_show_stale_option` (now
     `bob pomodoro status --help`). Test invocations may keep old spellings for now;
     `canonical-docs` migrates them.
8. **install-smoke** (`justfile`): add `bob task --help`,
   `bob task archive|reconcile|reroll --help`, `bob pomodoro --help`,
   `bob pomodoro notify|status|tmux --help`, and `bob help plan`. Keep the old-spelling
   `--help` lines as alias smoke.
9. **Verify.** `just all` and `just install-smoke` pass.

## Phase canonical-docs: README, docs, and tests teach the canonical names

1. **README.md**
   - Contents list and every anchor link.
   - Daily workflow steps:
     - Step 2: `bob task reroll --dry-run`, then `bob task reroll --seed <seed>`.
     - Step 3: `bob pomodoro`, `bob pomodoro tmux`, `bob pomodoro notify`.
     - Step 4: `bob task reconcile`.
     - Step 5: nightly wording.
   - Vault layout: `done/` is written by `bob task archive`.
   - Commands table: the final 14 rows by section, plus the Capture protocol rows. Drop
     the hidden-alias paragraph and point to Migration notes instead.
   - Replace `## Task status hooks`, `## Randomize`, and `## Move done tasks` with
     `## Task maintenance`. It opens with the membership rule and has
     `bob task reconcile`, `bob task reroll`, and `bob task archive` subsections that
     keep the existing content.
   - Replace `## Pomodoro status` with `## Pomodoro` covering bare/`status`, `tmux`, and
     `notify` with canonical usage lines. Drop the now-false "Help text still uses the
     legacy binary name `bob_notify`".
   - Compatibility shims table: `bob_notify` → `bob pomodoro notify`, `bob_pomodoro` →
     `bob pomodoro`, `tmux_bob_pomodoro` → `bob pomodoro tmux`.
   - Migration notes:
     - Lead with the old → canonical table.
     - State that old spellings are permanent hidden aliases with identical behavior and
       no notices, and that new integrations should use canonical names.
     - Note that reroll and archive commits are now labeled `bob task reroll` and
       `bob task archive`; older history says `bob randomize` and `bob move-done-tasks`.
     - Keep the historical hard-rename paragraph, but `bob collect-done` now maps to
       `bob task archive`.
     - Update the "new integrations" sentence.
   - Fix every link to a renamed README anchor across `README.md` and `docs/`. Grep for
     `#task-status-hooks`, `#randomize`, `#move-done-tasks`, and `#pomodoro-status`.
2. **docs/**: use canonical invocations in `getting-started.md`, `task-status-hooks.md`
   (keep its file name and concept title, with one "formerly `bob task-status-hooks`,
   still accepted" note), `randomize.md` (canonical command; the Schedule Log token
   stays `randomize`; the commit subject change and finding older `bob randomize`
   commits), `vault-git-sync.md` (Mac crontab line as
   `bob task reconcile --retry-timeout 120` with a note that an installed old line keeps
   working; the nightly description), `projects.md`, `plan.md`, `capture.md`,
   `freshness.md` (seed is hidden and still guarded), `task-dependencies.md`,
   `README.md`, and `completion.md`. Give at most one "formerly" note per doc. Leave
   historical narrative and `sase/memory/decisions/` untouched.
3. **Tests.** Migrate test invocations of old spellings to canonical paths, for example
   `.arg("task-status-hooks")` → `.args(["task", "reconcile"])`. Do this across
   `tests/cli/task_status_hooks/`, `tests/cli/move_done.rs`, `tests/randomize.rs`,
   `tests/cli/pomodoro.rs`, `tests/cli/help*.rs`, and the rest. Update test names and
   messages that name the command. Keep deliberate old-spelling coverage (the alias
   parity suite, `renamed_old_top_level_commands_are_unknown`, the legacy binary and
   script-fallback tests). Do not rename fixture directories.
4. **Residual sweep.** Run
   `rg -n "task-status-hooks|task-status-setter|mark-next-tasks|move-done-tasks|bob randomize|tmux-pomodoro|bob notify" README.md docs src tests justfile`.
   Every remaining hit must be one of these:
   - an alias table or parity test;
   - a persisted identifier;
   - an internal module or file name;
   - `vault_sync::run_notify`;
   - historical narrative;
   - a single "formerly" note.
5. **Follow-ups.** Record each as a `PROPOSED FOLLOW-UP:` note on this phase's bead (do
   not create beads):
   - bug: `vault_sync::run_notify` runs `bob notify` with no `PRE_CHECK_SLEEP`/
     `POST_NOTIFY_SLEEP`, so conflict notification always fails with a usage error and
     never notifies.
   - feature (R8): unify `-j/--json` with `-f/--format json` across commands.
   - feature (R8): normalize singular/plural command nouns (`projects`, `plugins` versus
     `task`).
   - feature (R8): restyle the hand-parsed help (`nightly`,
     `pomodoro status|tmux|notify`, `task archive`) to match clap-rendered help.
   - memory: propose a `decisions` record for the command-tree policy. It covers
     sectioned help over nesting, the narrow `bob task` rule, permanent silent aliases,
     and the frozen `capture-*` protocol. It names the rejected options (a broad
     `bob task`, `capture-api`, `bob vault`, deprecation hints) and cites the research
     report.
6. **Verify.** `just all` passes.

## Phase downstream-callers: chezmoi and bob-plugins callers move to canonical names

Open each repo with `/sase_repo` (`sase repo open chezmoi …`,
`sase repo open bob-plugins …`), read its `AGENTS.md`, and work only in the printed
paths. Bob Mac Capture needs no change: it spawns only `capture` and `capture-*`, which
are frozen.

**Rollout guard first.** New spellings fail on a `bob` built before `command-groups`,
while old spellings work on every version. Before applying chezmoi, confirm the
installed `bob` resolves the new paths:
`bob task --help && bob pomodoro tmux --help && bob pomodoro notify --help` (quietly).
If any fails, run `just install` from a bob-cli checkout that contains the
`command-groups` commit. That is safe, because every existing caller's spelling keeps
working. Then re-check.

**chezmoi**

- `home/bin/executable_bob_notify`: both `exec … notify "$@"` lines →
  `pomodoro notify "$@"`.
- `home/bin/executable_tmux_bob_pomodoro`: `tmux-pomodoro` → `pomodoro tmux`.
- `home/bin/executable_bob_pomodoro`: leave as is. Bare `bob pomodoro` is the canonical
  status form, and it works on old and new binaries.
- `home/dot_config/tmux/tmux.conf`: `#(bob tmux-pomodoro)` → `#(bob pomodoro tmux)`.
- `home/sase/memory/obsidian.md`: "runs `bob move-done-tasks`" → "runs
  `bob task archive`". Make the edit through `/sase_memory_write` (authorized by this
  plan) and republish per that repo's conventions.
- Stale caller from an earlier hard rename: in
  `home/bin/executable_maybe_bob_highlights_sync`, the failure message "bob
  highlights-ref scan failed" → "bob highlights scan failed".
- Retire the stale `bob_dataview` skill copies. They were installed before the
  `bob_query` rename and are not in `home/.sase-skills-manifest.json`, so
  `sase skill init` never removes them. Add a `home/.chezmoiremove` listing exactly:
  - `.claude/skills/bob_dataview`
  - `.codex/skills/bob_dataview`
  - `.gemini/skills/bob_dataview`
  - `.gemini/config/skills/bob_dataview`
  - `.gemini/jetski/skills/bob_dataview`
  - `.gemini/antigravity-cli/skills/bob_dataview`
  - `.grok/skills/bob_dataview`
  - `.qwen/skills/bob_dataview`
  - `.config/muse/skills/bob_dataview`
  - `.config/opencode/skills/bob_dataview`

  Use no `**` globs. If chezmoi v2.70 prefers another removal mechanism, use that
  instead. Confirm with `chezmoi apply --dry-run --verbose` (or `chezmoi diff`) that
  exactly these targets go and nothing else changes unexpectedly.

- Leave these alone: Hammerspoon's `bob pomodoro --show-stale` (already canonical),
  `bob_vault_sync_watch`, the `bob highlights scan` call, the nvim
  `bob_pomodoro_keymaps` Lua module, and the README's `bob pomodoro` mentions.
- Run the repo's tests that cover the touched files (for example the Hammerspoon and
  shim specs, if any reference them), commit, then run `chezmoi update -a --force` as
  chezmoi's `AGENTS.md` requires. Verify `tmux_bob_pomodoro` and `bob_notify --help`
  exit 0.

**bob-plugins**

- `plugins/bob-navigation-hotkeys/main.js`: notice "deferred to bob task-status-hooks" →
  "deferred to bob task reconcile". Also update the comments naming
  `bob task-status-hooks` there and in `plugins/task-status-cycler/main.js` and
  `plugins/block-id-prompt/main.js`. Do **not** touch the Schedule Log `randomize`
  classifier (`/\brandomize\b/`); it matches persisted vault data.
- Bump bob-navigation-hotkeys' patch version (2.1.0 → 2.1.1) if the repo's convention
  versions user-visible string changes. Comment-only edits need no bump.
- Run `npm test` and `npm run validate`, updating any test that pins the notice. Commit,
  then run `bob plugins sync` (repo `AGENTS.md` rule).

## Rollout note for Bryan

After the epic lands, reinstall `bob` on athena, apollo, and the Mac (`just install`)
before or when running `chezmoi update` on each host. The new tmux line and shims need
the new binary; old spellings keep working everywhere, so installing first is always
safe. The hand-installed Mac crontab can keep `bob task-status-hooks` forever or switch
to `bob task reconcile` once the Mac's `bob` is updated.
