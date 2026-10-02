---
tier: epic
title: Excellent shell completion for bob, plus just install
goal: 'Pressing TAB after `bob` in zsh (and bash) offers grouped, described, vault-aware
  completions computed live by the installed bob binary, so completion can never drift
  from the CLI; `bob completion` installs, inspects, and removes the shell adapter
  safely and honestly; and `just install` installs bob from source and keeps its shell
  completion current in one step.

  '
phases:
- id: one-tree
  title: One composed clap command tree for completion
  depends_on: []
  size: medium
  description: 'one-tree: build an exhaustive, completion-only clap tree from the
    module builders plus descriptors for the five hand-parsed commands and the bare
    default-subcommand forms, guarded by parity, parse-smoke, and help-drift tests,
    with no user-visible change.'
- id: engine
  title: Hidden __complete endpoint, protocol 1, and static value kinds
  depends_on:
  - one-tree
  size: medium
  description: 'engine: pin clap_complete''s dynamic engine behind one module, add
    the early-intercepted hidden __complete endpoint speaking protocol 1, the kinds
    table with static decisions, the presenter rules and groups, coverage and golden
    tests, and the first docs/completion.md.'
- id: zsh-adapter
  title: The bob-owned zsh adapter
  depends_on:
  - engine
  size: medium
  description: 'zsh-adapter: ship the embedded, protocol-stamped _bob zsh adapter
    that renders grouped, described, natively styled menus, completes on the first
    autoloaded TAB, and is covered by stubbed-compsys and real-zpty tests.'
- id: lifecycle
  title: bob completion command, adapter lifecycle, and just install
  depends_on:
  - zsh-adapter
  size: medium
  description: 'lifecycle: add the public bob completion command (status default,
    install, uninstall, zsh) with target discovery, ownership stamp plus manifest,
    atomic writes, real-shell verification and beautiful stacked reports, then the
    just install recipe, install-smoke checks, README, and docs.'
- id: vault-kinds
  title: Vault-aware value kinds with partial-parse context
  depends_on:
  - engine
  size: medium
  description: 'vault-kinds: add partial-parse context and read-only vault providers
    for routes, sections, tasks, task sections, Pomodoro refs, plugins, levels, and
    vault notes, behind a 150 ms deadline, with a read-only enforcement test and fixture-vault
    goldens.'
- id: capture-text
  title: Capture markers inside capture TEXT
  depends_on:
  - vault-kinds
  size: medium
  description: 'capture-text: complete capture markers at the end of the active TEXT
    word through an in-process capture_complete extraction, honoring the release gates
    (boundary, safe rows only, end of word only, no wikilinks), using the !prefix
    directive.'
- id: bash
  title: bash adapter and bash lifecycle support
  depends_on:
  - lifecycle
  - capture-text
  size: medium
  description: 'bash: add the values-only bash adapter with COMP_LINE word reassembly
    and wordbreak-safe replies, wire bash into install/status/uninstall and verification,
    and cover it with real-bash tests.'
- id: polish
  title: End-to-end polish, performance record, and docs finish
  depends_on:
  - bash
  size: small
  description: 'polish: run a sandboxed end-to-end zsh session, record real latency
    on the vault, replace illustrative docs samples with real transcripts from the
    fixture vault, and sweep help text and output for cli_rules and visual consistency.'
proposed_by: bbugyi200.apollo.46
create_time: 2026-10-02 11:06:14
status: wip
bead_id: bob-cli-3j
---

- **PROMPT:** [prompts/202610/bob_shell_completion.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202610/bob_shell_completion.md)
- **BEAD:** [bob-cli-3j](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-3j/README.md)

# Plan: Excellent shell completion for `bob`, plus `just install`

## Background

Bryan asked for sase-grade shell completion for `bob` and a `just install` target that
installs from source and updates completion. The research report
`research:202610/bob_shell_completion_and_just_install/bob_shell_completion_and_just_install.md`
(read it with `sase artifact read <ref> "<reason>"`) recommends a design Bryan fully
agreed with. Its sibling report `…/bob_shell_completion_and_just_install__cld.md` (§5,
§6.6) holds the zsh adapter prototype that rendered real menus in zsh 5.9. This plan
adopts that design, settles the report's open questions, and adds a few protocol details
the prototype lacked.

**The core idea: the `bob` binary is the grammar.** Every TAB runs a hidden
`bob __complete` request against one composed clap tree plus bob's own read-only,
vault-aware value providers. The file installed into the shell is a small, stable,
protocol-versioned adapter that only renders what bob returns. Completion therefore
always matches whichever `bob` is on `PATH`, however it was installed. "Updating
completion" means ensuring the adapter is present and current, not regenerating
anything.

### Decisions on the research's open questions

| Question                                              | Decision                                                                                                 |
| ----------------------------------------------------- | -------------------------------------------------------------------------------------------------------- |
| The ten `capture-*` frontend endpoints in `bob <TAB>` | Keep them, grouped last under a `capture protocol` header. Hiding them would break `bob capture-t<TAB>`. |
| bash support                                          | Supported (values only) in its own phase after zsh. fish is deferred.                                    |
| bob-scoped default zstyles                            | Yes. They apply only when the user has set no matching style, and honor `NO_COLOR`.                      |
| Hand-parsed commands                                  | Completion-only descriptors with drift tests. Their runtime parsers stay untouched.                      |
| chezmoi-managed `_bob`                                | Out of scope. The stable adapter makes it possible later.                                                |
| A decisions-web record                                | Not part of this plan. This plan edits no SASE memory.                                                   |

### Ground truth (verified on master when this plan was written)

- `src/runner.rs` registers each subcommand through `delegate_subcommand()`, a trailing
  var-arg bag. Real flags live in about 22 module `build_cli()` functions. Most are
  private and named for help output, such as `"bob capture"`.
- Five commands use hand parsers:
  - `move-done-tasks`: `collect_done/mod.rs` `parse_args`
  - `nightly`: `nightly.rs` `parse_args`
  - `notify`: `notify.rs` `parse_args`
  - `pomodoro` and `tmux-pomodoro`: `pomodoro.rs` `parse_args` / `parse_tmux_args`
- `HIDDEN_SUBCOMMAND_ALIASES` (`mark-next-tasks`, `task-status-setter`) must never be
  offered.
- Bare default forms are re-dispatched by hand: `bob freshness [list options]` becomes
  `list` (`freshness/cli.rs`), and `bob vault-sync [run options]` becomes `run`
  (`vault_sync.rs` `default_run_args`). `gkeep` and `plugins` already model their bare
  form in clap.
- Capture `TEXT` is `trailing_var_arg(true)` plus `allow_hyphen_values(true)`
  (`capture/cli.rs` `text_arg`). Once text has started, `--r` is text, not an option.
- `Cargo.lock` pins clap 4.6.1. `clap_complete` 4.6.11 requires clap ≥ 4.6.6, and the
  local registry cache already holds clap 4.6.7.
- `clap_complete::engine::complete(&mut Command, Vec<OsString>, arg_index, cwd)` returns
  `CompletionCandidate`s carrying `help`, `id` (`arg::<id>` / `command::<name>`), `tag`,
  `display_order`, and `hidden`. That is enough for bob's presenter.
- `ValueCompleter` sees only the current word, so bob supplies context itself.
- Available helpers:
  - `native::style::Styler` (TTY and `NO_COLOR` aware)
  - `native::env::{bob_dir, bob_cli_state_dir, home_dir}`
  - `runner::cli_styles()`, the green/cyan help palette
  - `sha2`, already a dependency
- `decisions:mac-capture-is-a-thin-client` makes bob the only implementation of capture
  grammar and completion data. Shell adapters are thin clients and must never parse bob
  syntax.

## Guardrails for every phase

- **Never run `just install`.** Never run `bob completion install` or `uninstall`
  against the real `$HOME` either. These replace Bryan's machine-wide `~/.cargo/bin/bob`
  and write into his dotfile directories. Use `just install-smoke` (isolated root) and
  tests that set a temporary `HOME`, `XDG_STATE_HOME`, `XDG_DATA_HOME`, and `ZDOTDIR`.
  There is a second reason: SASE sets one shared `CARGO_TARGET_DIR` per agent run, which
  can make `cargo install --path` install a stale binary.
- **Keep the tree green.** Run `just all` (fmt, clippy, test) at the end of every phase.
  Extend `just install-smoke` whenever a phase adds a command.
- **Follow the `cli_rules` reference memory.** Read it with `/sase_memory_read`. Its
  rules:
  - excellent `--help`
  - alphabetical commands and options
  - a short alias for every public long option (the internal `__complete` arguments are
    exempt)
  - color through `Styler`
- **Completion never writes.** No vault writes, `git`, network, clipboard, locks, script
  materialization, or state/log files. The only exception is the explicitly requested
  `BOB_COMPLETE_DEBUG=<file>`.
- **Say "shell completion" in user-facing text**, so it is never confused with
  `capture-complete`.
- **Leave runtime dispatch alone.** `run_bob` → `native::run` stays unchanged except for
  the `__complete` early intercept and the new `completion` subcommand.
- **Never add a value cache** unless measurements demand one.

## Architecture

```text
 zsh/bash ──TAB──▶ adapter (_bob / bob, written by `bob completion install`)
                     │  command bob __complete zsh --protocol 1 --suffix "$SUFFIX" -- <words…> "$PREFIX"
                     ▼
 bob __complete ─▶ early intercept in run_bob (nothing else runs)
                ─▶ completion::tree()            one exhaustive clap tree (one-tree)
                ─▶ context: partial parse         --bob-dir, --route, --task, --repo … (vault-kinds)
                ─▶ engine.rs → clap_complete     subcommands, options, static choices (engine)
                ─▶ kinds + providers              routes, tasks, Pomodoros, capture markers …
                ─▶ present.rs                     presentation rules, human groups, directives
                ─▶ protocol.rs                    value\tdesc\tgroup\tsuffix | !prefix !dirs !files !message
```

Suggested layout. The names are flexible; engine isolation is not:

```text
src/native/completion/
  mod.rs          run_complete(args) entry; module wiring
  tree.rs         composed tree (one-tree)
  engine.rs       the ONLY module that imports clap_complete (engine)
  protocol.rs     request parsing, constants, encoder, sanitization (engine)
  kinds.rs        (command path, arg id) → Kind table (engine; extended later)
  present.rs      post-filter and presenter (engine)
  context.rs      partial-parse context (vault-kinds)
  providers.rs    read-only vault providers (vault-kinds)
  capture_text.rs capture-marker slot (capture-text)
  cli.rs, install.rs, manifest.rs, verify.rs, report.rs  (lifecycle)
  adapters/_bob.zsh (zsh-adapter), adapters/bob.bash (bash)  embedded with include_str!
tests/cli/completion/{mod.rs, protocol.rs, zsh_adapter.rs, lifecycle.rs, vault.rs, capture_text.rs, bash.rs}
docs/completion.md
```

Each phase owns separate test files and docs sections. That way the two parallel tracks
after `engine` (zsh-adapter → lifecycle, and vault-kinds → capture-text) rarely touch
the same lines.

## Protocol 1 (the contract between bob and every adapter)

**Request:**

```text
bob __complete <SHELL> --protocol <N> [--suffix <TEXT>] -- <WORD>...
```

- **`SHELL`** is `zsh` or `bash`. It changes only presentation: bash ignores
  descriptions and groups, so bob may omit them.
- **`WORD`s** are the command line's words up to the cursor, shell-unquoted. The first
  word is the command as typed and is ignored. The last word is the cursor word's text
  _before_ the cursor, and may be empty.
- **`--suffix`** is the unquoted text of the cursor word _after_ the cursor. It is
  absent or empty when the cursor sits at the end of the word.
- **A malformed request** exits 2 with a message on stderr; adapters discard stderr.

**Response.** UTF-8 lines on stdout. Exit 0, including on internal failure, which yields
empty output.

- **Directives** are whole lines, emitted before any candidate:
  - `!prefix <N>`: keep the first N Unicode scalar values of the cursor-word prefix, and
    let candidates replace only the rest. zsh maps it to `compset -P N`; bash prepends
    the kept text. Bob uses it for `--opt=` attached values and for capture markers
    inside a longer word.
  - `!dirs`: complete directories natively.
  - `!files` / `!files <glob>`: complete files natively, optionally filtered.
  - `!files-in <root>\t<glob>`: complete paths relative to `root` (vault notes). The
    separator is a TAB, so roots may contain spaces.
  - `!message <text>`: show a hint and offer no candidates (free-text slots, version
    skew).
- **Candidate lines:** `value<TAB>description<TAB>group<TAB>space|nospace`.
  - `value` is never empty and never contains TAB, LF, or CR; bob drops such candidates.
  - `description` may be empty, and LF/CR/TAB become spaces.
  - An empty `group` means `values`.
  - `nospace` means the user is expected to keep typing, for example after `/`, `=`, or
    `@dev:`.
- **Order is display order.** Groups display in order of first appearance, and adapters
  never re-sort.
- **Unknown directives.** Adapters ignore unknown `!` directives, so an additive
  directive needs no protocol bump; any other change bumps the protocol.
- **Version skew.** The binary supports `[MIN_PROTOCOL, PROTOCOL]`, which is `[1, 1]`
  now. Once protocol 2 exists it must keep 1.
  - A request below the minimum gets
    `!message bob shell completion is out of date — run: bob completion install`.
  - A request above the maximum gets
    `!message this bob is older than its shell completion — reinstall bob (just install)`.
- **Silence.** No stderr on normal paths. A silent panic hook plus `catch_unwind` turns
  panics into empty output.
- **Debugging.** `BOB_COMPLETE_DEBUG=<file>` appends the request, the response, the
  elapsed milliseconds, and any error or timeout.
- **The shell filters, not bob.** Bob never prefix-filters within a slot: it resolves
  the slot from the cursor word and returns that slot's full candidate set, so Bryan's
  `matcher-list` keeps working.
  - Call the engine with a slot-normalized current word: empty for values and commands,
    `-` or `--` for options.
  - Serve attached `--opt=` values with `!prefix`.

## Presentation rules

1. **Empty cursor word.** Offer subcommands and positional values. Offer options only
   when nothing else applies, for example `bob capture-sections <TAB>`.
2. **A lone `-`.** Offer short and long forms adjacent, with identical descriptions.
   zsh's `list-grouped` then shows `-b  --bob-dir  -- Bob vault root` on one row. A
   `--…` word gets long forms only.
3. **Filter what no longer applies.**
   - Drop options already present, unless they are `Append` or `Count`.
   - Drop options that conflict with present ones.
   - Drop `-h/--help` and `-V/--version` once other arguments exist.
   - Never offer hidden arguments, hidden subcommands, aliases, or clap's auto `help`
     subcommand.
4. **Never re-sort what bob ranked.**
   - Subcommands follow `SUBCOMMANDS` order.
   - Options follow help order.
   - Routes follow capture-targets order.
   - Tasks follow capture-complete ranking.
5. **Descriptions add information** and never repeat the value.
6. **Group headers are human words:**
   - `commands`, then `capture protocol` (the ten frontend endpoints)
   - `options`
   - per-slot groups such as `format`, `inbox`, `areas`, `projects`, `sections in dev`,
     `tasks in dev`, and `open Pomodoros`
7. **Text boundary.** Once a trailing var-arg `TEXT` has started, or after `--`, offer
   no options or subcommands.

Target experience in zsh. Headers render bold green, matching `bob --help`:

```text
% bob <TAB>
── commands ──
capture      -- Capture a task or bullet into the Bob vault
completion   -- Install and inspect shell completion for bob
freshness    -- Walk the tiered freshness review queue and seed the cutover
…
── capture protocol ──
capture-complete  -- Complete the capture marker at the cursor
…

% bob capture-sections --route <TAB>
── inbox ──
mac_inbox  -- inbox · default capture target
── areas ──
cash  dev  fun  gtd  job  …                     -- area
── projects ──
bob  bob_git  bob_gtd  …                        -- project · wip

% bob capture fix it @dev:<TAB>
── tasks in dev ──
@dev:remote-power   -- Enable remote power on for Athena!
@dev:menu-bar-ping  -- Render tmux_ping in menu bar!

% bob capture --format <TAB>
── format ──
human  -- Colored text for people
json   -- Machine-readable JSON
```

## The `bob completion` command (final surface)

```text
bob completion [COMMAND]

Install and inspect shell completion for bob.

Every <TAB> is answered live by the `bob` on your PATH, so completion always
matches the installed binary: commands, options, and vault values such as
capture routes, tasks, and open Pomodoros. The file installed into your shell
is a small, stable adapter that rarely changes. `bob completion install`
keeps it current and never edits your rc files.

Commands:
  bash       Print the bash completion adapter
  install    Install or refresh the completion adapter for your shells
  status     Show installed adapters and the bob they call (default)
  uninstall  Remove completion adapters that bob installed
  zsh        Print the zsh completion adapter

Examples:
  bob completion                   Same as `bob completion status`
  bob completion install           Install for $SHELL; refresh every bob-owned adapter
  bob completion install zsh -d    Show the zsh plan without writing anything
  bob completion status -v         Also check registration in a real shell
  bob completion zsh -o ~/.zfunc/_bob
                                   Write the zsh adapter yourself
```

Subcommands and options, all alphabetical:

- `install [SHELL]...`
  - `-d/--dry-run`: show the plan; write nothing, not even directories or the manifest.
  - `-f/--force`: replace a file bob did not write, or one edited since install.
  - `-n/--no-verify`: skip the real-shell check.
  - `-q/--quiet`: print only warnings and errors.
  - `-t/--target DIR`: install into DIR. Combining it with more than one shell is a
    usage error.
- `status`
  - `-j/--json`
  - `-v/--verify`: probe a real shell now.
  - Bare `bob completion [-j] [-v]` is `status`, and the composed tree models that form.
    `list` is a hidden alias.
- `uninstall [SHELL]...`
  - `-d/--dry-run`
- `bash` / `zsh`
  - `-o/--output FILE`
- `SHELL` is a possible-values argument with `PossibleValue::help`. It lists `zsh` until
  the bash phase adds `bash`.
- Exit codes:
  - `0` for success or an explicit no-op
  - `1` when an install or uninstall failed, an install is not registered, or
    `status -v` found a broken install
  - `2` for usage errors

**Report design.** Use stacked per-shell blocks, never a wide table: narrow terminals
must not shred columns, which was sase's anti-lesson. Use color on a TTY through
`Styler`, and plain text when piped or under `NO_COLOR`. The glyphs: `✓` green, `·` dim,
`→` cyan, `⚠` yellow, `✗` red. Show the home directory as `~`. Sample outputs:

```text
$ bob completion                       # status (default)
🐚  Shell completion · bob 0.1.0 · ~/.cargo/bin/bob

  ✓  zsh    current · protocol 1 · registered as _bob
            ~/.zfunc/_bob
  ·  bash   not installed              → bob completion install bash

$ bob completion install               # first run
🐚  Shell completion · bob 0.1.0 · ~/.cargo/bin/bob

  ✓  zsh    installed · protocol 1 · registered as _bob
            ~/.zfunc/_bob  (first writable fpath entry)

  Open a new shell (or run `exec zsh`) to start using it.

$ just install                         # later runs: adapter unchanged
  ✓  zsh    unchanged · protocol 1 · registered as _bob
            ~/.zfunc/_bob

  Completion is live: every <TAB> asks ~/.cargo/bin/bob, so new commands work now.

$ bob completion install zsh -d
🐚  Shell completion · dry run — nothing is written

  →  zsh    would install · protocol 1
            ~/.zfunc/_bob  (first writable fpath entry)

  ✗  zsh    installed · not registered
            ~/.zfunc/_bob
            ~/.zfunc is not on your fpath. Add this line to ~/.zshrc before compinit:
              fpath=(~/.zfunc $fpath)

  ⚠  `bob` on PATH is /usr/local/bin/bob, not this binary (~/.cargo/bin/bob).
     <TAB> asks the bob on PATH. Fix PATH or reinstall.
```

**Status states.**

- `not installed`: hint `→ bob completion install <shell>`.
- `current`: bob-owned, and the bytes equal what this binary writes.
- `outdated`: bob-owned, but the bytes differ from this binary's adapter. Hint
  `→ bob completion install`.
- `edited`: the stamp is present, but the digest differs from the manifest. Hint: `-f`
  replaces it.
- `foreign`: no bob stamp. Hint: `-f` replaces it.
- `current (externally managed)`: an unrecorded file whose bytes equal this adapter. Bob
  reports it and never adopts it.
- `missing`: the manifest records an install, but the file is gone.

**Registration is never claimed from file existence.**

- Without `-v`, status shows the verification recorded at install, but only if that
  record's digest and path match the file. Otherwise it shows
  `· registration not checked → bob completion status -v`.
- With `-v`, status reports the live probe:
  - `registered as _bob`
  - `not registered` (with a remedy)
  - `shadowed by <file>`
  - `bob is bound to <fn>`
  - `unverified (<reason>)`

`status -j` emits `schema_version: 1` with `protocol`, a `bob` object (version, running
path, path of `bob` on PATH), and a `shells` array. Each shell entry has `shell`,
`state`, `path`, `protocol`, `owned`, `registration`, `registration_checked` (`install`
| `now` | null), `target_reason`, and `remedy`.

## one-tree: One composed clap command tree

Build `native::completion::tree() -> clap::Command`, the full bob grammar used only for
completion. There is no user-visible change in this phase.

1. **Module builders.** Make every module `build_cli()` `pub(crate)`, along with the
   highlights `create::command()` and `clip::command()` as needed. Adding a
   `pub(crate) fn completion_command()` wrapper is also fine. Do not change any
   builder's runtime behavior or help output.
2. **Exhaustive mapping.** Add `NativeCommand::command(self) -> clap::Command` as an
   exhaustive `match` with no `_` arm, so a new `NativeCommand` cannot compile without a
   tree entry.
3. **`tree()`.**
   - Root `bob`: version, about, `disable_help_subcommand(true)` applied recursively.
   - For each `SUBCOMMANDS` entry in order, mount
     `entry.native_command.command().name(entry.name).about(entry.about)`.
   - Expose the table from `runner` with a `pub(crate)` accessor, or move it into a
     shared module. Never include `HIDDEN_SUBCOMMAND_ALIASES`.
4. **Bare default forms.** `bob freshness [list options]` and
   `bob vault-sync [run options]` must parse. For example, mount the default
   subcommand's args on the parent with `args_conflicts_with_subcommands(true)`. Confirm
   `gkeep` and `plugins` bare forms parse as they are.
5. **Descriptors for hand parsers.** Add a `pub(crate) fn completion_descriptor()` next
   to each hand parser, and confirm each against the parser source:
   - `move-done-tasks`: `-t/--threshold <N>`
   - `nightly`: none
   - `notify`: `-v/--verbose` (Count), plus positionals `PRE_CHECK_SLEEP` and
     `POST_NOTIFY_SLEEP`
   - `pomodoro`: `-d/--debug`, `-v/--verbose`, `-s/--show-stale`
   - `tmux-pomodoro`: none

   All include `-h/--help`. Refactor each hand parser's `print_help()` into a
   `help_text() -> String` that the printer uses. The output must stay byte-identical,
   so tests can read it.

6. **Unit tests in `tree.rs`:**
   - The mounted names equal the `SUBCOMMANDS` names, in order. No mounted name contains
     a space. Hidden aliases and `help` subcommands are absent.
   - `tree().debug_assert()` passes.
   - Parse smoke tests:
     - `bob capture --route cash fix it`
     - `bob capture -- --2`
     - `bob freshness -f json`
     - `bob freshness list -f json`
     - `bob vault-sync --dry-run`
     - `bob gkeep pull --dry-run`
     - `bob move-done-tasks -t 10`
     - `bob notify -vv 5 10`
     - `bob pomodoro -s`
   - Descriptor drift: every option token in each descriptor appears in that command's
     `help_text()`, and every `-x` / `--long` token in `help_text()` appears in the
     descriptor.
7. **No help changes.** All existing help tests in `tests/cli/help*.rs` must pass
   unchanged.

## engine: Hidden `__complete`, protocol 1, and static kinds

1. **Dependency.**
   - Add `clap_complete = { version = "=4.6.11", features = ["unstable-dynamic"] }` with
     a comment explaining the exact pin (`unstable-dynamic` is semver-exempt and also
     enables `clap/unstable-ext`).
   - Raise `clap` to `"4.6.6"` in `Cargo.toml`.
   - Update `Cargo.lock` with `cargo update -p clap --precise <≥4.6.6>` plus the new
     package. Keep unrelated lockfile churn out.
2. **`engine.rs`.** This is the only `clap_complete` import. It wraps `engine::complete`
   and returns bob-owned candidate structs (value, help, id, tag, order, hidden).
   Document the fallback in a module comment: if upstream breaks, replace this file with
   an in-house walker over clap's public introspection, behind the same interface.
3. **Early intercept.** When `argv[1] == "__complete"`, `runner::run_bob` calls
   `native::completion::run_complete(argv[2..])` and returns. This happens before the
   root clap parse and before the `BOB_CLI_USE_SCRIPT` fallback. `__complete` appears
   neither in `bob --help` nor in the tree.
4. **`protocol.rs`.** Request parsing, `PROTOCOL` / `MIN_PROTOCOL`, the skew messages,
   and the line encoder with sanitization, all unit-tested.
5. **`kinds.rs`.**
   - `Kind` covers `Choices`, `Dirs`, `Files(Option<glob>)`, and `FreeText`. `FreeText`
     emits `!message <VALUE_NAME> — <arg help>`. Later phases add vault kinds.
   - The table is keyed by (command path pattern, arg id), most specific entry first.
     The key needs the path because one ID can mean two things: `--block-id` is a _new_
     ID in `capture-task-id` but an _existing_ task in `capture-task-sections`.
   - A module builder may instead set a clap `ValueHint` on a path argument (`DirPath` →
     `!dirs`, `FilePath`/`AnyPath` → `!files`). That needs no table entry.
   - Attach directives by annotating the tree with `ArgValueCompleter`s that return
     sentinel candidates, which `present.rs` turns into directive lines, or something
     equivalent.

   v1 decisions for this phase:
   - **Choices:** every argument with clap possible values (`--format`, `--engine`,
     highlights `--status`, `--prefer`, gkeep `--source`, …). Add `PossibleValue::help`
     where it adds information.
   - **Dirs:** `--bob-dir`, `--repo`, `--backup-dir`, `--lib-dir`, `--ref-dir`,
     `--xlib-dir`, `query --vault`.
   - **Files:**
     - `--query-file`, `--tasks-file`
     - highlights markdown inputs (`*.md`)
     - PDF arguments and PDF `--output` (`*.pdf`)
     - clip `--html` (`*.html`)
   - **FreeText:** everything else, for example `--message`, `--name`, `--title`,
     `--seed`, `--cursor`, `--limit`, `--until`, gkeep `--id` and `--email`, and
     `capture-task-id --block-id`.
   - **Temporarily FreeText**, with an informative message until the later phases
     upgrade them: every vault slot (`--route`, `--section`, `--task`, `--task-section`,
     `capture-task-sections --block-id`, `--pomodoro-ref`, `--plugin`, `--level`,
     `--tasks-note`, `--origin`) and capture `TEXT`.

6. **`present.rs`.**
   - Implement the presentation rules 1–7, slot normalization, and `!prefix` for
     `--opt=value`.
   - Add a tier field to runner's `Subcommand` table, for example
     `tier: Porcelain | Plumbing`, and mark the ten `capture-*` endpoints `Plumbing` so
     grouping metadata lives with the table.
   - Use `options` as the options header, whatever clap's help headings say.
   - Name a value group after the slot, for example the lowercased value name `format`.
7. **Coverage tests.**
   - (a) Every value-taking argument in `tree()` has a decision: possible values, a
     `ValueHint` other than `Unknown`, a kinds entry, or explicit `FreeText`.
   - (b) Every kinds entry matches a real (path, arg) in `tree()`.
8. **Golden integration tests.** Register `tests/cli/completion/mod.rs` and add
   `protocol.rs`. The tests run the built binary on these cases:
   - `bob ''`, which yields `commands`, then `capture protocol`, and no options
   - `bob cap`, which yields the full command set, unfiltered
   - `bob capture -` (paired forms), `bob capture --`
   - `bob capture --route cash --`, where `--route` is dropped
   - `bob capture --section x --`, where conflicting `--task` is dropped
   - `bob capture-sections --format `
   - `bob freshness -f `
   - `bob query --query-file `, which yields `!files`
   - `bob capture --bob-dir `, which yields `!dirs`
   - `bob capture --format=`, which yields `!prefix 9` with values `human` and `json`
   - `bob capture fix -`, where TEXT has started, so the slot gets only a message and no
     options
   - `bob gkeep `
   - `bob mark-` (hidden aliases absent)
   - `--protocol 0` and `--protocol 99` skew messages
   - a malformed request, which exits 2
   - `BOB_COMPLETE_DEBUG` writes the log file and nothing else
9. **Latency check.** A warn-only check: structural requests should have p50 ≤ 10 ms. It
   prints a warning and never fails.
10. **Docs.** Create `docs/completion.md` with these sections: Overview (the runtime
    model), Protocol 1, and What completes. Add a row to `docs/README.md`.

## zsh-adapter: The bob-owned zsh adapter

1. **Embedding.** Add `src/native/completion/adapters/_bob.zsh`, embedded with
   `include_str!` behind `pub(crate) fn zsh_adapter() -> &'static str`.
   - Line 1 is `#compdef bob`.
   - Line 2 is the ownership stamp:
     `# Generated by bob completion (protocol 1). Do not edit: run `bob completion
     install`.`
   - A unit test asserts that the stamp's protocol equals `protocol::PROTOCOL`.
2. **Reference implementation.** Start from this. It extends the tested cld §6.6 shape
   with `--suffix`, unquoting, `!prefix`, TAB-separated `!files-in`, `NO_COLOR`, and
   empty-output safety:

   ```zsh
   #compdef bob
   # Generated by bob completion (protocol 1). Do not edit: run `bob completion install`.
   # Each <TAB> asks the `bob` on your PATH, so completion always matches the installed binary.

   _bob() {
     local -a lines fields groups
     local -A arrays
     local line value help group suffix array header ret=1

     lines=(${(f)"$(command bob __complete zsh --protocol 1 --suffix "$SUFFIX" -- \
       "${(@Q)words[1,CURRENT-1]}" "$PREFIX" 2>/dev/null)"})
     (( $#lines )) || return 1

     # Bob-scoped presentation defaults, applied only where you have set no style.
     if ! zstyle -m ":completion:${curcontext}:descriptions" format '*'; then
       header='%B%F{green}── %d ──%f%b'
       (( ${+NO_COLOR} )) && header='── %d ──'
       zstyle ':completion:*:*:bob:*:descriptions' format "$header"
     fi
     zstyle -m ":completion:${curcontext}:" group-name '*' ||
       zstyle ':completion:*:*:bob:*' group-name ''

     for line in $lines; do
       case $line in
         ('!prefix '<->) compset -P ${line#!prefix }; continue ;;
         ('!dirs')       _files -/ && ret=0; continue ;;
         ('!files')      _files && ret=0; continue ;;
         ('!files '*)    _files -g "${line#!files }" && ret=0; continue ;;
         ('!files-in '*) fields=("${(@ps:\t:)${line#!files-in }}")
                         _files -W "$fields[1]" -g "$fields[2]" && ret=0; continue ;;
         ('!message '*)  _message -r "${line#!message }"; continue ;;
         ('!'*)          continue ;;  # a directive from a newer bob: ignore it
       esac
       fields=("${(@ps:\t:)line}")
       value=${fields[1]//:/\\:} help=$fields[2] group=${fields[3]:-values} suffix=${fields[4]:-space}
       array=$arrays[$group/$suffix]
       if [[ -z $array ]]; then
         array=_bob_group_$#groups
         arrays[$group/$suffix]=$array
         groups+=("$group/$suffix")
         local -a $array
       fi
       set -A $array "${(@P)array}" "$value${help:+:$help}"
     done

     for group in $groups; do
       array=$arrays[$group]
       suffix=${group##*/} group=${group%/*}
       if [[ $suffix == nospace ]]; then
         _describe -V -t "${group// /-}" "$group" $array -S '' && ret=0
       else
         _describe -V -t "${group// /-}" "$group" $array && ret=0
       fi
     done
     return ret
   }

   if [[ $zsh_eval_context[-1] == loadautofunc ]]; then
     _bob "$@"
   else
     compdef _bob bob
   fi
   ```

   **Required properties.** Each one needs a test:
   - The first autoloaded TAB completes immediately.
   - No `emulate -L zsh`, which breaks `_describe`.
   - TAB splitting preserves empty fields.
   - `:` in values is escaped.
   - Empty output returns 1, so zsh falls through.
   - User styles win.
   - Directives go through `_files` and `_message`, so native colors, quoting,
     `list-colors`, and `menu select` keep working.
   - stderr is discarded.
   - The adapter never parses bob syntax.

   Verified on zsh 5.9: `${(f)"$(…)"}` yields zero elements for empty output, while the
   quoted `"${(@f)…}"` form yields one. `(@Q)` unquotes words. `(@ps:\t:)` keeps empty
   fields.

3. **Tests** in `tests/cli/completion/zsh_adapter.rs`. Skip them with a printed note
   when `zsh` is absent.
   - (a) **Stubbed compsys.**
     - Setup: in `zsh -f`, source the adapter file (read through `CARGO_MANIFEST_DIR`).
       Stub `_describe`, `_files`, `_message`, `compset`, and `compdef` so they record
       their arguments. Put a fake `bob` on PATH that prints canned protocol lines and
       records its argv. Set `words`, `CURRENT`, `PREFIX`, `SUFFIX`, and `curcontext`,
       then call `_bob`.
     - Assert the request argv: unquoted words, `--suffix`, `--protocol 1`.
     - Assert group order and that `-S ''` applies to nospace candidates.
     - Assert colon escaping and empty descriptions.
     - Assert each directive's mapping, including `!prefix 7` → `compset -P 7`.
     - Assert unknown directives are ignored, and empty output returns 1.
   - (b) **Real zsh through `zmodload zsh/zpty`.** Add no new crate dependencies.
     - Setup: start `zsh -f` in a pty with `fpath=(<tmp> $fpath)` and
       `autoload -Uz compinit && compinit -D`. Put the test-built bob on PATH by
       symlinking `CARGO_BIN_EXE_bob` as `bob`, and copy the adapter to `<tmp>/_bob`.
     - Typing `bob cap<TAB>` as the very first completion shows `capture` rows, which
       proves the first autoloaded TAB completes.
     - `bob capture -<TAB>` shows `--route`.
     - `bob capture --format <TAB>` shows the `format` header.
   - (c) **Styles.** With no user zstyle, bob's format is set. With a user
     `descriptions` format, bob leaves it alone. With `NO_COLOR`, the plain header is
     used.
4. **Docs.** Add a Styling section to `docs/completion.md`:
   - what bob sets by default
   - how to override it with your own `zstyle`
   - `NO_COLOR`

## lifecycle: `bob completion`, adapter lifecycle, and `just install`

1. **Register the command.**
   - Add `NativeCommand::Completion`.
   - Add a `SUBCOMMANDS` entry `completion` (Porcelain) with about "Install and inspect
     shell completion for bob", sorted between `capture-tasks` and `freshness`.
   - Mount it in the tree through the exhaustive match, modeling the bare `status` form.
   - Add `bob completion install` to the root `AFTER_HELP` examples, and update the
     top-level help tests.
2. **CLI.** Build exactly the surface in "The `bob completion` command" with zsh only.
   Design the `Shell` enum so bash slots in later. Use `ValueHint`/possible values for
   its own arguments rather than kinds-table entries: this keeps `kinds.rs` free of
   merge conflicts with the parallel vault-kinds phase. Write help text and examples per
   `cli_rules`.
3. **Shell selection.**
   - Use the explicit `SHELL` arguments if given.
   - Otherwise use the basename of `$SHELL` _plus_ every shell that has a bob-owned
     adapter in the manifest. Never use parent-process detection: just recipes run under
     bash.
   - An unsupported `$SHELL` with no owned adapters exits 1 with "pass a shell:
     `bob completion install zsh`".
4. **Target directory.** The first match wins, and bob records the reason it chose:
   1. `-t`
   2. the location recorded in the manifest, which keeps the location stable
   3. the first fpath entry that is under `$HOME`, user-owned, writable, and not under
      the oh-my-zsh root, a `plugins` directory, a cache directory, or a plugin-manager
      tree. Read the fpath with a bounded `zsh -ic` probe. On Bryan's hosts this is
      `~/.zfunc`.
   4. `${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/completions`, when oh-my-zsh is present
   5. `~/.zfunc`, which bob creates, then prints the exact `fpath=(~/.zfunc $fpath)`
      line to add before compinit

   The zsh file name is `_bob`.

5. **Ownership.**
   - The manifest lives at `$XDG_STATE_HOME/bob-cli/completion/manifest.json` (through
     `bob_cli_state_dir()`), with `schema_version: 1`. Each shell entry records:
     - `path`, `protocol`, `sha256`
     - `bob_version`, `installed_at`, `target_reason`
     - `verification`: result plus the digest it applied to
   - A foreign file, an edited file, or a symlink is refused without `-f`.
   - A foreign file whose bytes equal the current adapter is reported as
     `current (externally managed)` and is not adopted.
6. **Write.**
   - Write atomically: a temp file in the same directory, then rename.
   - Skip the write when the bytes are unchanged, and report `unchanged`.
   - Remove a stale `_bob.zwc` next to the file.
   - A dry run creates nothing.
7. **Verify.** This is the default on install; `-n` skips it.
   - Run one bounded `zsh -ic` probe:
     - its own process group, stdin `/dev/null`, `BOB_COMPLETION_PROBE=1`
     - killed at the deadline: 8 s by default, with a hidden
       `BOB_COMPLETION_PROBE_TIMEOUT_MS` override for tests
     - machine-parseable output that is robust to spaces in paths
   - The probe reads three things:
     - `$fpath`
     - `${_comps[bob]}`
     - `$functions_source[_bob]` after `autoload +X _bob`
   - Classify the result:
     - `registered`: `_comps[bob]` is `_bob` and the source is this file
     - `shadowed by <file>`
     - `bob is bound to <fn>`
     - `not registered`, with a remedy:
       - if the directory is not on fpath: the exact `fpath` line
       - if it is on fpath: a stale compinit dump, so
         `rm -f "${ZDOTDIR:-$HOME}"/.zcompdump* && exec zsh`
     - `unverified (timed out | zsh not found)`: a warning, never success
   - Also warn when `command -v bob` (resolved) is not `current_exe()`.
   - Record the result in the manifest.
8. **Report.** Follow the report design and samples above. The closing hint depends on
   what happened:
   - when something was installed or updated: "Open a new shell (or run `exec zsh`)"
   - when everything was unchanged: "Completion is live: every TAB asks <path>"
9. **`uninstall [SHELL]...`.**
   - With no SHELL, it targets every bob-owned adapter.
   - It removes only files whose stamp and manifest digest prove that bob wrote them,
     plus any `_bob.zwc`, and drops their manifest entries.
   - It refuses an edited file and prints the exact `rm` command instead.
   - It closes with "Open a new shell to drop the loaded completion."
10. **`bob completion zsh [-o FILE]`** prints the adapter, or writes it to FILE.
11. **justfile.** Add the recipe next to `install-smoke`:

    ```just
    # Install bob and its shims from this checkout, then install or refresh shell
    # completion. Pass shells to choose explicitly: `just install zsh bash`.
    [positional-arguments]
    install *shells: (_banner "35" "📦" "INSTALL")
        #!/usr/bin/env bash
        set -euo pipefail
        root="${CARGO_INSTALL_ROOT:-${CARGO_HOME:-$HOME/.cargo}}"
        cargo install --path . --locked --root "$root"
        if ! "$root/bin/bob" completion install "$@"; then
            printf '\nbob is installed at %s; shell completion needs attention (see above).\n' \
                "$root/bin/bob" >&2
            exit 1
        fi
    ```

    - There is no `--force`: cargo replaces the same package from any checkout, and
      `--force` would only let it overwrite _another_ crate's `bob`.
    - The explicit `--root` makes the recipe call exactly the binary it installed. It
      also overrides cargo's `install.root` config; document that.
    - Extend `install-smoke`:
      - add `--help` checks for `completion`, `completion install`, `completion status`,
        `completion uninstall`, and `completion zsh`
      - add
        `"${root}/bin/bob" __complete zsh --protocol 1 -- bob cap | grep -q '^capture'`
      - under `env HOME="$root/home" XDG_STATE_HOME=… XDG_DATA_HOME=… SHELL=/bin/zsh`,
        run:
        - `completion install zsh -d -t "$root/zfunc"`, asserting nothing was created
        - `completion install zsh -n -t "$root/zfunc"`
        - `completion status -j`
        - `completion uninstall zsh`

12. **README.**
    - Make Installation lead with `just install`, saying what it does and which root it
      uses.
    - The Git-remote line becomes
      `cargo install --git git@github.com:bobs-org/bob-cli.git --locked bob-cli && bob completion install`.
    - Drop `--force` everywhere and explain why.
    - Note that automation and agents use `just install-smoke`.
    - Add a short "Shell completion" subsection, a command-index entry for `completion`,
      and the `docs/completion.md` link in the command-contract table.
13. **`docs/completion.md`.** Add these sections:
    - Installing, covering `just install`, `bob completion install`, and remote installs
    - Commands
    - Status states
    - Troubleshooting: `status -v`, `BOB_COMPLETE_DEBUG`, the fpath line, a stale
      compdump, PATH shadowing, and skew messages
    - Why no rc edits and no `eval`
14. **Tests** in `tests/cli/completion/lifecycle.rs`. Every test sets a temporary
    `HOME`, `XDG_STATE_HOME`, `XDG_DATA_HOME`, `ZDOTDIR`, and `SHELL`. Probe logic is
    deterministic through a fake `zsh` on PATH that branches on the probe script. Cover:
    - first install, then idempotent `unchanged`
    - `outdated` → updated
    - refusals of foreign, edited, and symlinked files, and `-f` replacing them
    - externally managed files
    - the dry run writing nothing (snapshot the temp tree)
    - each target rule and its recorded reason
    - `-t` with two shells → exit 2
    - registered, shadowed, not registered (both remedies), and timed out → unverified
    - the PATH-shadow warning
    - status states, `-j` JSON shape, and `-v` exit codes
    - uninstall, including refusal of an edited file
    - plain output without ANSI when piped
    - `.zwc` cleanup
    - `bob completion` bare equals `status`
    - the `list` alias

## vault-kinds: Vault-aware value kinds with context

1. **`context.rs`.**
   - Partially parse the words before the cursor with
     `tree().ignore_errors(true).try_get_matches_from(…)`.
   - Walk down to the deepest subcommand matches.
   - Expose:
     - the subcommand path, used for kinds lookup
     - `bob_dir`: `--bob-dir`, else `native::env::bob_dir()`, which honors `BOB_DIR`
     - `route`, `task`, `repo`, and any other needed values

   Short clusters (`-rcash`), `--route=cash`, and environment defaults then behave
   exactly as at runtime. Value completers capture an `Arc<Context>`.

2. **Providers.** They are read-only, reuse the existing scanners in-process, and never
   spawn `bob`:

   | Kind         | Slots                                                                                                   | Source                                                            | Presentation                                                                                                                         |
   | ------------ | ------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------ |
   | route        | `--route` on `capture`, `capture-sections`, `capture-tasks`, `capture-task-id`, `capture-task-sections` | capture-targets scan                                              | groups `inbox` / `areas` / `projects` in scan order. Descriptions: `inbox · default capture target`, `area`, or `project · <status>` |
   | section      | `capture --section` (needs route)                                                                       | capture-sections scan                                             | group `sections in <route>`                                                                                                          |
   | task         | `capture --task`, `capture-task-sections --block-id` (needs route)                                      | capture-tasks scan                                                | value = block ID as the flag expects it; description = task text; capture-complete ranking; group `tasks in <route>`                 |
   | task section | `capture --task-section` (needs route + task)                                                           | capture-task-sections scan                                        | value = exact TITLE the flag expects                                                                                                 |
   | Pomodoro ref | `capture-pomodoro-name --pomodoro-ref`                                                                  | capture-pomodoros scan                                            | open entries; description such as `0700-0725 · running`; group `open Pomodoros`                                                      |
   | plugin       | `plugins sync --plugin`                                                                                 | directory names under `<--repo or env::plugins_dir()>/plugins/*/` | **never** git                                                                                                                        |
   | level        | `randomize --level`                                                                                     | config `randomize.levels[].label`                                 | description = its window                                                                                                             |
   | vault notes  | `query --tasks-note`, `--origin`                                                                        | `!files-in <bob_dir>\t*.md`                                       | —                                                                                                                                    |
   - A dependent slot with its prerequisite missing returns
     `!message pass --route first`.
   - Decide `highlights --parent` after reading its help: vault note or free text.

3. **Deadline.**
   - Run the engine call on a worker thread, and have the main thread wait at most 150
     ms. A hidden `BOB_COMPLETE_DEADLINE_MS` override exists for tests.
   - On timeout, print nothing, log to the debug file if one is set, and exit
     immediately so a straggling worker cannot delay the prompt.
4. **Read-only enforcement test.**
   - Copy a fixture vault to a temp directory and `chmod -R a-w` it.
   - Put a fake `git` on PATH that records any call.
   - Use empty XDG directories.
   - Run `__complete` for every vault slot.
   - Assert identical file-tree hashes, zero git calls, and untouched XDG directories.
5. **Goldens** in `tests/cli/completion/vault.rs`, using a fixture vault and `BOB_NOW`:
   - routes, grouped and ordered
   - `--route cash --section `, `-rcash --section `, and `--route=cash --task `
   - task sections
   - Pomodoro refs
   - plugins from a fixture repo directory
   - the missing-prerequisite message
6. **Latency check.** A warn-only check: vault slots should have p50 ≤ 30 ms on the
   fixture.
7. **Docs.** Extend the What completes section of `docs/completion.md`.

## capture-text: Capture markers inside TEXT

1. **Extraction.** Extract a `pub(crate)` service from `capture_complete`, for example
   `shell_completion(bob_dir, raw_text, cursor) -> Result<Option<ShellCompletion>>`.
   - It returns the replacement byte range plus rows of (replacement, description,
     context group, continues).
   - Build it on `build_result`'s logic. Never re-implement grammar.
   - It must return before `NoteIndex::read` for wikilink contexts: wikilinks are
     deferred because they take about 300 ms.
2. **Slots.** `TEXT` on `capture`, `capture-parse`, and `capture-rewrite`. Confirm that
   they join TEXT words with single spaces. The cursor word is TEXT when the resolved
   command's trailing TEXT has already received a word, when `--` precedes it, or when
   it sits at the TEXT position and does not start with `-`.
   - Build `raw_text` from the TEXT words before the cursor, joined with spaces, plus
     the cursor prefix.
   - Set `cursor = raw_text.len()`.
3. **Release gates.** Each gate needs a test.
   - **Safe rows only.** Omit rows that carry `requires_block_id` or `requires_name`.
     Create-Pomodoro rows only canonicalize text, so they stay, described as
     `new Pomodoro`.
   - **End of the active word only.** A non-empty `--suffix` returns nothing. The
     replacement must end at the cursor and start inside the cursor word. A cursor word
     containing a newline returns nothing.
   - **Nothing beyond completion.** Never call `capture-task-id`,
     `capture-pomodoro-name`, any write handler, or any dry run.
4. **Presentation.**
   - Emit `!prefix N`, where N is the number of chars of the cursor word before the
     replacement start.
   - Values are replacement texts, such as `@dev:remote-power`.
   - Descriptions come from the row: task text, route kind, or Pomodoro time and name.
   - Groups are human words: `inbox` / `areas` / `projects`, `sections in dev`,
     `tasks in dev`, `task sections`, `open Pomodoros`, `active tasks`.
   - A replacement ending in a continuation character (`:` `+` `#` `=` `^`) is
     `nospace`. A complete marker gets a space.
5. **Goldens** in `tests/cli/completion/capture_text.rs`, using a fixture vault and
   `BOB_NOW`:
   - `bob capture fix it @` (routes)
   - `@ca`, which returns the full set
   - `@dev:` (tasks)
   - `@dev#` (sections)
   - `@dev+<id>#` (task sections)
   - `^` (active tasks)
   - `=#` (start names)
   - the single quoted word `fix it @dev:`, which yields `!prefix 7`
   - Unicode before a marker, such as `café @dev:`, where the prefix counts chars
   - `--suffix x`, which returns nothing
   - a newline in the word, which returns nothing
   - `bob capture fix --r`, which yields no `--route`
   - `[[`, which returns nothing, and quickly
   - every TEXT slot added to the read-only enforcement test
6. **Docs.** Add a Capture markers section to `docs/completion.md`, and a cross-link
   from the capture-complete section of `docs/capture.md`: shell completion is another
   thin client of that service.

## bash: bash adapter and lifecycle support

1. **Adapter.** Add `adapters/bob.bash`, about 40–70 lines, with the ownership stamp in
   its leading comment lines. It registers with `complete -F _bob bob` (no `-o default`,
   so free-text slots never fall back to filenames).
   - **Word reassembly.** Rebuild the prior words and the cursor word from
     `COMP_LINE`/`COMP_POINT`, rejoining tokens that `COMP_WORDBREAKS` split at `:` and
     `=`. Unquote simple quoting, and send the text after the cursor as `--suffix`.
   - **`!prefix N`.** Apply it, then strip the part bash already broke off at a
     wordbreak, as `__ltrim_colon_completions` does.
   - **Directive mapping:**
     - `!dirs` → `compgen -d`
     - `!files [glob]` → `compgen -f`, filtered by the glob
     - `!files-in` → relative names under root
     - `!message` → an empty reply
   - **Options.** Set `compopt -o nospace` only when every candidate is nospace. Set
     `compopt -o filenames` for file directives.
2. **Lifecycle.**
   - `Shell` gains `bash`, with the target
     `${BASH_COMPLETION_USER_DIR:-${XDG_DATA_HOME:-~/.local/share}/bash-completion}/completions/bob`.
   - Add `bob completion bash [-o]`, and bash rows in install, status, uninstall, and
     JSON.
   - **Verify** with a bounded `bash -ic` probe. It triggers bash-completion's lazy
     loader (`_comp_load bob` in bash-completion ≥ 2.12, else `__load_completion bob`),
     then checks that `complete -p bob` names `_bob`.
   - Without bash-completion, report `not registered` with the exact `source <path>`
     line for `~/.bashrc`, since bob never edits rc files.
3. **Tests** in `tests/cli/completion/bash.rs`:
   - **Real bash.** In `bash --norc --noprofile` with the test-built bob on PATH, set
     `COMP_LINE`, `COMP_POINT`, `COMP_WORDS`, and `COMP_CWORD` as bash would split them,
     call `_bob`, and assert `COMPREPLY`. Cases:
     - `bob cap`
     - `bob capture --format=j`
     - `bob capture fix @dev:` with a colon split
     - the quoted word `fix it @dev:`
     - Unicode
     - options after TEXT
     - `!dirs`
   - **Lifecycle.** Install, status, and uninstall tests for bash, with a fake `bash`
     for the probe.
   - **Selection.** `SHELL=/bin/bash` selection.
4. **justfile and docs.** Add `install-smoke` checks for `completion bash --help` and
   `__complete bash`, and add a bash section to `docs/completion.md`.

## polish: End-to-end polish and performance record

1. **Sandboxed session.** Use a temporary HOME and the zpty harness against the
   _fixture_ vault, so no private vault content lands in docs. Install with
   `bob completion install zsh -t <tmp>`, then capture real menus:
   - `bob <TAB>`
   - `bob capture-sections --route <TAB>`
   - `bob capture fix it @dev:<TAB>`
   - `bob capture --format <TAB>`
   - `bob completion <TAB>`

   Replace the illustrative samples in `docs/completion.md` with these transcripts.

2. **Performance record.** Measure p50 and p95 on the real vault (`BOB_DIR=~/bob`). This
   is safe because `__complete` never writes, and the read-only test enforces that.
   - Measure `bob <TAB>`, `--route`, `--task`, `@dev:`, and `--pomodoro-ref`.
   - Record the numbers, host, and date in a Performance section.
   - If structural p95 ≥ 20 ms or vault p95 ≥ 75 ms, fix it when the fix is small.
     Otherwise file a task bead through `/sase_new_task`.
3. **Consistency sweep.**
   - Every new `--help` against `cli_rules`.
   - The "shell completion" wording.
   - Install and status output on a TTY, piped, under `NO_COLOR`, and at `COLUMNS=60`,
     which must never break the stacked layout.
   - The README index, the `bob --help` example, and the `docs/README.md` row.
4. **Final check.** `just all` and `just install-smoke` must pass.

## Out of scope and follow-ups

- fish adapter, once a host has fish
- wikilink completion, which needs an index cache because it takes about 300 ms
- a chezmoi-managed `_bob` for athena and the Mac
- moving the runtime parsers onto the composed tree
- upstreaming tag-grouping to clap_complete's own zsh adapter
- a decisions-web record ("shell completion is a thin client of bob"), only if Bryan
  asks for it
