---
tier: epic
title: Finish shell completion — correct results, bash insertion, and an honest lifecycle
goal: 'Every shell-completion reply means what bob itself would do with it, bash inserts
  exactly what bob returned, and `bob completion` / `just install` probe quickly from
  a real terminal and report registration, exit codes, and closers honestly — so epic
  bob-cli-3j can land.

  '
parent_bead: bob-cli-3j
phases:
- id: results
  title: Completion results, bash insertion, and capture-grammar integration
  depends_on: []
  size: medium
  description: 'results: make body-bearing @route: follow capture-complete''s new-ID
    intent, honor ValueHints and positional file slots, fix the stale capture-complete
    TEXT hint, make the bash adapter insert correctly across = wordbreaks, open quotes,
    spaces, and attached !files-in, and add goldens and docs for the =x/=*/=! capture
    changes.'
- id: lifecycle
  title: Honest, fast bob completion lifecycle
  depends_on: []
  size: medium
  description: 'lifecycle: run shell probes in their own session so they never stall
    under a terminal, stop status from probing or failing not-installed shells, honor
    $SHELL alongside owned adapters, align glyphs, exit codes, and closers with registration,
    keep stale compdumps visible, refuse unrecorded stamped files, and add the lifecycle
    tests the original plan required.'
proposed_by: bbugyi200.apollo.bob-cli-3j.land
create_time: 2026-10-02 14:29:06
status: done
bead_id: bob-cli-3j.9
---

- **PROMPT:** [prompts/202610/shell_completion_landing_fixes.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202610/shell_completion_landing_fixes.md)
- **PARENT:** [202610/bob_shell_completion.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/bob_shell_completion.md)
- **BEAD:** [bob-cli-3j.9](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-3j/bob-cli-3j.9.md)

# Plan: Finish shell completion (landing fixes for epic bob-cli-3j)

## Background

Epic bob-cli-3j (plan `202610/bob_shell_completion.md`) built `bob __complete`, the zsh
and bash adapters, `bob completion`, and `just install`. Its eight phases are closed,
`cargo test` is green, and `cargo fmt --check` is clean. The land audit then ran the
built binary against a fixture vault, drove a real interactive bash through a pty, and
ran `bob completion` under a controlling terminal. It confirmed the defects below. All
of them were introduced by the epic and must be fixed before it lands.

This child epic's `parent_bead` is bob-cli-3j. Its land agent resumes that landing:
closing bob-cli-3j, its symvision check, and its plan-file status are deliberately
**not** phases here.

The two phases own disjoint files and can run in parallel:

- **`results`** owns:
  - `present.rs`, `kinds.rs`, `capture_text.rs`, and `adapters/bob.bash`, all under
    `src/native/completion/`
  - `shell_completion` in `src/native/capture_complete.rs`
  - `tests/cli/completion/{capture_text.rs,bash.rs,protocol.rs,vault.rs}` (the COMPREPLY
    and readline cases in `bash.rs`)
  - these sections of `docs/completion.md`: What completes, Capture markers, bash, and
    Live transcripts
- **`lifecycle`** owns:
  - `install.rs`, `verify.rs`, `report.rs`, `manifest.rs`, and `cli.rs`, all under
    `src/native/completion/`
  - `tests/cli/completion/{lifecycle.rs,zsh_adapter.rs}`
  - `README.md`, `justfile`, and `Cargo.toml` / `Cargo.lock`
  - these sections of `docs/completion.md`: Installing, Commands, Status states, and
    Troubleshooting

If `lifecycle` must touch a bash _lifecycle_ test that lives in `bash.rs`, it should
move that test into `lifecycle.rs` rather than edit `bash.rs` in place.

### Fixture used in the expectations below

The expectations below use the fixture vault from `tests/cli/completion/capture_text.rs`
(`fixture()`), with `BOB_NOW="2026-06-01 09:10:01"`. It contains:

- area `cash`, with sections `Shopping` and `Ideas`
- tasks `^buy-milk` and `^fix-sink`; `^fix-sink` has task sections `FIRST STEPS` and
  `SECOND STEPS`
- project `dev`, with one task, `^ship-it`
- a daily note with open Pomodoros `Morning pages` and `DEEP WORK`

## Guardrails (both phases)

- **Never run `just install`.** Never run `bob completion install` or `uninstall`
  against the real `$HOME` either. Tests and manual checks must set a temporary `HOME`,
  `XDG_STATE_HOME`, `XDG_DATA_HOME`, `ZDOTDIR`, and `SHELL`. Use `just install-smoke`
  for the install path.
- **Completion never writes.** No vault writes, `git`, network, locks, or state files.
  The read-only enforcement test in `tests/cli/completion/vault.rs` must stay green.
- **Adapters stay thin clients.** Adapters never parse bob syntax beyond shell-word
  reassembly (`decisions:mac-capture-is-a-thin-client`). Capture-marker semantics must
  come from the shared `capture_complete` / `capture_language` code, never a local copy.
- **Follow `cli_rules`.** Help stays alphabetical and excellent, and color goes through
  `Styler`.
- **Verification at the end of each phase:**
  - `cargo fmt --check`
  - `cargo clippy --all-targets --all-features`. Its only allowed error is the
    pre-existing deny at `tests/cli/capture/pomodoro_name.rs:808`, which epic bob-cli-28
    owns; do not fix it here. The phase must add no new completion-module warnings.
  - `cargo test`
  - `just install-smoke`

## Phase `results`: completion results, bash insertion, capture integration

### 1. Body-bearing `@route:` follows capture-complete's intent

**Defect.** `shell_completion` in `src/native/capture_complete.rs`, in its
`CompletionContext::PomodoroBlockId` arm, always offers existing linkable tasks. It
skips the `capture_block_ids::build_block_id_field` intent that `build_result` uses.

For `bob capture fix it @dev:<TAB>` it offers `@dev:ship-it`. But
`<text> @route:block-id` creates a _new_ task with that ID (docs/capture.md, "Linking
and starting existing tasks"). Accepting the row therefore fails:

```text
bob capture: … block ID ^ship-it already exists in …/dev.md
```

`bob capture-complete` on the same text returns intent `new`, no candidates, and the
suggestions `["fix"]`.

**Fix.**

- Build the block-ID field exactly as `build_result` does: call `build_block_id_field`
  with the field's route, replacement, and context, then branch on `block_field.intent`.
- **Link intent** (a solo `@route:` item): offer existing identified linkable tasks
  exactly as today, in the group `tasks in <route>`.
- **New or ProjectNote intent:** never offer existing tasks. Offer each of the field's
  `suggestions` as a row:
  - value `<marker_prefix>@<route>:<suggestion>` (built the same way the current rows
    are)
  - description `new block ID`
  - group `new task ID`
  - `space`
- Delete the comment that says shell completion "always offers linkable tasks".

**Expected:**

| Words after `bob capture` | Reply                                                      |
| ------------------------- | ---------------------------------------------------------- |
| `@dev:`                   | `@dev:ship-it` (tasks in dev)                              |
| `fix it @dev:`            | `@dev:fix` (new task ID)                                   |
| `'fix it @dev:'`          | `!prefix 7`, then `@dev:fix`                               |
| `'café @dev:'`            | `!prefix 5`, then `@dev:caf`                               |
| any of the above          | never an ID listed in `capture-complete`'s `block_id.used` |

**Tests and docs.**

- Update the goldens in `tests/cli/completion/capture_text.rs` (the `fix it @dev:`,
  quoted, and café cases).
- Add a solo `@dev:` golden that lists `@dev:ship-it`.
- Add a test asserting that for body-bearing text, shell completion and
  `capture-complete -f json` agree on the intent.
- Update `docs/completion.md`:
  - the `bob capture fix it @dev:<TAB>` transcript and examples: use a solo
    `bob capture @dev:<TAB>` for the task list, and show the new-ID suggestion for the
    body-bearing form
  - the "can never disagree" claim, which is true again after this fix

### 2. Stale `capture-complete` TEXT hint

**Defect.** The generic `text` entry in `src/native/completion/kinds.rs` is still
`Kind::VaultSoon("TEXT — in-progress text; marker completion lands in a later phase")`.
So `bob capture-complete fix <TAB>` shows a stale promise.

**Fix.**

- Make that generic entry an accurate free-text message, for example
  `TEXT — draft text`.
- Keep the capture-marker slots on `capture`, `capture-parse`, and `capture-rewrite`
  unchanged.
- Remove `Kind::VaultSoon` if nothing uses it anymore.
- Keep the coverage tests (a) and (b) green.

### 3. Honor `ValueHint`s; scope the PDF `output` entry

**Defect.** `value_lines` in `src/native/completion/present.rs` never reads
`Arg::get_value_hint()`. The coverage test still counts a hint as a decision. As a
result:

- `bob completion install -t <TAB>` returns `!message DIR — Install into DIR …` instead
  of directories.
- The generic `output` entry (`Kind::Files(Some("*.pdf"))`) applies to every command, so
  `bob completion zsh -o <TAB>` and `bob completion bash --output <TAB>` return
  `!files *.pdf`.

**Fix.**

- **Order of precedence.** A path-specific kinds entry beats a `ValueHint`. A
  `ValueHint` other than `Unknown` / `Other` beats a generic (`path: &[]`) kinds entry
  and the FreeText fallback.
- **Hint mapping:**
  - `DirPath` → `!dirs`
  - `FilePath`, `AnyPath`, and `ExecutablePath` → `!files`
- **Scope the PDF entry.** Make the `*.pdf` `output` decision path-specific to the
  highlights commands whose `--output` is a PDF; check the builders. Every other
  `output` falls through to its hint or to FreeText.

**Expected:**

- `bob completion install -t ''` → `!dirs`
- `bob completion zsh -o ''` → `!files`
- the highlights PDF `--output` still returns `!files *.pdf`
- Add these as goldens in `tests/cli/completion/protocol.rs`.

### 4. Positional value slots with an empty cursor word

**Defect.** In `present.rs`, around the `subs.is_empty() && values.is_empty()` branch,
an empty cursor word at a positional slot falls back to options. So
`bob highlights create <TAB>` lists `-b`/`--bob-dir`… instead of `!files *.md`. That
breaks presentation rule 1: offer positional values first, and options only when nothing
else applies.

**Fix.**

- When the engine returns no subcommands or values, resolve the next unfilled
  positional's decision through the same `value_lines` path. If that decision is a
  directive or value kind, emit those lines:
  - files, dirs, vault notes, choices, or a `ValueHint`
- FreeText positionals keep the options fallback.
- Capture `TEXT` slots keep their current behavior.

**Expected:**

- `bob highlights create ''` → `!files *.md`
- `bob capture-sections ''` → options, unchanged
- `bob notify ''` → options, unchanged
- Add these as goldens.

### 5. bash adapter inserts exactly what bob returned

`src/native/completion/adapters/bob.bash`. Readline replaces only its own current word.
The audit drove a real interactive `bash --norc --noprofile -i` with the adapter
sourced, typed the line, and pressed TAB. It got:

| Typed                                                       | Result today                          | Must become            |
| ----------------------------------------------------------- | ------------------------------------- | ---------------------- |
| `bob capture =#`                                            | `==#deep-work`                        | `=#deep-work`          |
| `bob capture 'fix it @ca` (open quote)                      | the quoted text replaced by the value | `fix it @cash`         |
| `bob capture --route cash --task fix-sink --task-section F` | two args `FIRST` `STEPS`              | one arg `FIRST STEPS`  |
| `bob query --tasks-note=c`                                  | nothing                               | `--tasks-note=cash.md` |

These already work and must stay that way:

- `--format=j` → `--format=json`
- `@dev:` colon split → `@dev:ship-it`
- `--route ca` → `cash`

**Fixes:**

- **a. Every wordbreak, not just `:`.** When the cursor word is unquoted, readline
  replaces only the text after the last character of the cursor word that appears in
  `$COMP_WORDBREAKS`.
  - Build each full reply as the kept text (`${prefix:0:keep}`) plus the value.
  - Strip from it the cursor-word text up to and including that last wordbreak
    character, whatever it is (`=`, `:`, …).
  - This generalizes the current `:`-only strip.
- **b. Open quote.** When the cursor word began with a quote that is still open at the
  cursor (the reassembly loop knows this through `in_single` / `in_double`):
  - readline replaces the whole quoted text, so reply with the kept text plus the value
  - no wordbreak stripping and no escaping
  - readline adds the closing quote itself
- **c. Escaping.** For unquoted words, escape values that contain spaces or shell
  metacharacters (`printf -v q %q`), so each value stays one argument.
- **d. `!files-in` with `!prefix`.**
  - Filter the relative note names by the text after the kept prefix (`${prefix:keep}`),
    not by the whole prefix.
  - Apply the same wordbreak rule as in (a).
- **e. Keep the adapter's contract:**
  - the ownership stamp
  - `complete -F _bob bob` with no `-o default`
  - `compopt -o nospace` only when every candidate is nospace
  - no bob-syntax parsing

**Tests in `tests/cli/completion/bash.rs`:**

- **COMPREPLY expectations.** Update the existing COMPREPLY cases to the corrected
  replies. The quoted `fix it @dev:` case now serves the new-ID suggestion; use
  `'fix it @ca` for an open-quote route case.
- **Readline end-to-end.** Add tests that drive a real interactive bash through zsh's
  `zpty` module, in the same harness style as `zsh_adapter.rs` and with no new crates:
  - Start `bash --norc --noprofile -i` with `INPUTRC=/dev/null`, `TERM=dumb`, the
    fixture env, and the test-built bob on `PATH`, then source the adapter.
  - Type the line and send TAB.
  - Send C-a and type `printf '<%s>\n' ` in front of the line, then Enter.
  - Assert the printed arguments for every row of the table above, plus `--format=j` and
    solo `@dev:`.
  - Skip with a printed note when zsh or bash is absent.

### 6. Integration with capture changes that landed during the epic

Two capture commits landed while the epic was in flight:

- 74f47d4: the inline single-entry close on the `=x` line, where Work Log text is
  literal
- f589d07: the `=*` and `=!` close shorthands

Shell completion inherits both through the shared `capture_language`
`completion_field_at`. The audit verified:

| Words after `bob capture` | Reply                            |
| ------------------------- | -------------------------------- |
| `'=x done @'`             | nothing                          |
| `=*`                      | nothing                          |
| `=!`                      | nothing                          |
| `'=x wired it =#'`        | `!prefix 12`, then `=#deep-work` |

**Tests.** Lock these four cases in as goldens in
`tests/cli/completion/capture_text.rs`.

**Docs.** In the Capture markers section of `docs/completion.md`, say that action items
offer nothing, exactly as in `capture-complete`:

- closes (`=x…`, `=*`, `=!`)
- starts (`=`, `=<X>`)
- `+N` / `-N` adjustments and `++N` / `--N` shifts
- Work Log text on or below the `=x` line

## Phase `lifecycle`: honest, fast `bob completion`

### 1. Probes never stall under a terminal

**Defect.** `run_bounded_shell` in `src/native/completion/verify.rs` starts `zsh -i -c`
/ `bash -i -c` with `process_group(0)`, so the probe still has the user's controlling
terminal. The interactive shell opens `/dev/tty` from a background process group and is
stopped by `SIGTTIN` / `SIGTTOU`. Reproduced under a pty with a sandboxed HOME:

- `bob completion status -v` takes 24 s and reports `unverified (timed out)` for every
  shell.
- Plain `bob completion` takes 8 s, because of the fpath probe.
- Without a tty, the same commands finish in 0.6 s.

`just install` from a terminal hits this on every run.

**Fix.**

- **New session.** Start the probe in a new session with no controlling terminal: use
  `CommandExt::pre_exec` to call `libc::setsid()`.
  - Add `libc = "0.2"` as a direct dependency. It is already in `Cargo.lock` at 0.2.186;
    keep lockfile churn to that.
  - Keep stdin as `/dev/null` and keep `BOB_COMPLETION_PROBE=1`.
- **Kill the whole group on timeout.** Use `libc::kill(-pid, SIGKILL)`, not only the
  child.
- **Drain stdout concurrently.** Read stdout on a reader thread while waiting, and stop
  at the deadline. Then more than 64 KiB of rc output cannot block the probe, and a
  background job that inherits the pipe cannot hang bob after the shell exits.

**Regression test.**

- Run bob with a controlling terminal by driving it through zsh's `zpty` from
  `tests/cli/completion/lifecycle.rs`.
- Use a fake `zsh` probe that first tries to read from `/dev/tty`. That read stops a
  background-process-group reader, but fails immediately with no controlling terminal.
- Assert the probe classifies well within the timeout instead of reporting `timed out`.
- Skip when zsh is missing.

### 2. Keep stale compdumps visible

**Defect.** `probe_zsh` always runs `compinit -D` itself. That rebuilds `_comps` from
`fpath`, so a stale compdump from the user's rc is never observed, and the
`rm -f "${ZDOTDIR:-$HOME}"/.zcompdump* && exec zsh` remedy can almost never fire.

**Fix.** Initialize completion only when the rc did not:

```zsh
(( ${+_comps} )) || { autoload -Uz compinit; compinit -D; }
```

Deferred-compinit setups still verify. Adjust the fake-zsh tests, and cover the
on-fpath-but-unregistered stale-dump remedy.

### 3. `status` never fails, or probes, a shell that is not installed

**Defects:**

- In `status_one` in `src/native/completion/install.rs`, `status -v` probes
  not-installed shells, renders them `✗ … not installed · not registered`, and exits 1.
  A zsh-only user always gets exit 1.
- Plain `status` (no `-v`) runs the up-to-8 s `zsh -ic` fpath probe just to show where
  zsh _would_ install.

**Fix.**

- A shell with no file and no manifest entry renders the plan's form, with no probe in
  any mode, and does not count toward exit 1:

  ```text
  ·  bash   not installed              → bob completion install bash
  ```

- `missing` (the manifest records an install but the file is gone) stays a failure.
- `status` without `-v` never spawns a shell.

**Tests.**

- Plain `bob completion`, `status -j`, and `status -v` never invoke a recording fake
  shell for a not-installed shell.
- `status -v` exits 0 when every installed adapter is registered.

### 4. `$SHELL` is honored alongside owned adapters

**Defect.** `run_install` calls `selected_or_all(matches, StatusDefault::Owned(..))`,
which returns the owned shells as if they were explicit, so `select_for_install` returns
before consulting `$SHELL`. Reproduced: with bash owned and `SHELL=/bin/zsh`,
`bob completion install -n` refreshed bash and never installed zsh.

**Fix.**

- With no SHELL arguments, select `$SHELL`'s basename plus every owned shell.
- With `-t` and no SHELL arguments, select only `$SHELL`'s shell. Then `-t` does not hit
  the two-shell usage error just because another adapter is owned.
- `-t` with two explicit shells still exits 2.
- Add tests for both rules.

### 5. Glyphs and exit codes match registration

**Defect.** An install whose live verification is `not registered` renders
`✓ zsh installed · protocol 1 · not registered` and exits 0. The README and the
`just install` recipe both promise exit 1 when completion needs attention. The original
plan's exit codes and sample (`✗ zsh installed · not registered`) say the same.

**Fix.**

- **Install:**
  - `not registered`, `shadowed by …`, or `bob is bound to …` render `✗` and make the
    command exit 1.
  - `unverified (timed out | … not found)` renders `⚠` and exits 0.
  - `-n` renders `· registration not checked → bob completion status -v` instead of
    `unverified (skipped (--no-verify))`.
- **Status without `-v`:** a recorded unhealthy verification renders `⚠`, never `✓`.
  The exit code stays 0.
- **Docs.** Update the `install.rs` module comment, the README Installation text, and
  the Status states and Troubleshooting sections of `docs/completion.md`.

### 6. Closers tell the truth

**Defect.** A dry run prints "Completion is live: every <TAB> asks …". So does an
unchanged install that is not registered.

**Fix.**

- A dry run never prints that closer.
- "Completion is live" prints only when every row is unchanged and registered.
- When nothing changed but a row is unhealthy, print no closer: the row notes carry the
  remedy.

### 7. Unrecorded stamped files are not adopted

**Defect.** `classify` treats a stamped file with no manifest record and different bytes
as `outdated`. Install then overwrites and adopts it without `-f`. This happens, for
example, to a file the user wrote with `bob completion zsh -o`, after bob upgrades.

**Fix.**

- Classify that file as `outdated (externally managed)`.
- Refuse it without `-f`, with the same refusal and hint as a foreign file.
- Never adopt it into the manifest.
- `current (externally managed)` is unchanged.
- Add tests.

### 8. `-t` moving an owned install

**Defect.** When `-t` installs to a new path while the manifest records a bob-owned
adapter elsewhere, the old file is left behind, untracked.

**Fix.**

- After the new write succeeds, remove the old file and any `.zwc` next to it if its
  digest still matches the manifest, and say so in the row notes.
- If the old file was edited, leave it and print the exact `rm` command.
- Add a test.

### 9. The home-default `fpath` line always prints

When the target reason is the `~/.zfunc` home default (target rule 5), always print the
`fpath=(~/.zfunc $fpath)` line to add before compinit. Today it appears only through a
not-registered probe result. Print it with `-n` and in a dry run too, and add a test.

### 10. Remaining required tests and checks

- **Clippy.** Fix `needless_option_as_deref` at `verify.rs:257`.
- **Missing tests from the original plan,** in `tests/cli/completion/lifecycle.rs`:
  - plain output with no ANSI when piped
  - zsh `bob is bound to <fn>`
  - the previous-install (manifest) target rule and its recorded reason
- **Fake bash.** The fake bash in `lifecycle.rs` always prints `complete -F _bob bob`.
  Make it branch on whether the adapter exists, so a not-installed bash is never
  reported registered.
- **zsh header.** The real-zsh zpty test in `zsh_adapter.rs` must also assert the
  `format` group header for `bob capture --format <TAB>`.
- **`just install-smoke`.** The dry-run step must also assert that no manifest file was
  created.
