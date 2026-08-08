---
tier: tale
title: Distinguish no-Pomodoro and overdue-Pomodoro menu states
goal:
  The Hammerspoon menu bar shows a bold green NO POMODORO only when no Pomodoro ledger
  entry is open, and a bold red OVERDUE POMODORO while an open Pomodoro is at least ten
  minutes overdue.
proposed_by: bbugyi200.athena.vz.f1
create_time: 2026-08-08 19:26:10
status: wip
---

# Plan: Distinguish no-Pomodoro and overdue-Pomodoro menu states

## Objective

Refine the Pomodoro menu-bar warning introduced by chezmoi commit `5f5bd239` so that two
different successful states no longer share one red `NO POMODORO` presentation:

- a successful status check with no open Pomodoro renders exactly `NO POMODORO` in bold
  green;
- an open Pomodoro that is at least 600 seconds overdue renders exactly
  `OVERDUE POMODORO` in the same bold red style previously used for `NO POMODORO`;
- an active countdown and an open Pomodoro less than 600 seconds overdue retain their
  existing plain `M:SS` and regular red `+M:SS` presentations.

The distinction must remain correct across the 15-second refresh, manual refresh,
wake/unlock refresh, and Hammerspoon reload. Closing the final open Pomodoro must turn
an overdue warning into green `NO POMODORO` on the next successful refresh. Operational
failures must continue to hide the menu and log diagnostics rather than masquerading as
either successful state.

## Tier and repository scope

This is a `tale`: one follow-up agent can implement and verify the small, cohesive
status-contract and presentation change. It spans the current `bob-cli` repository and
the linked `chezmoi` repository because the menu cannot reliably distinguish the two
states from the current command output alone.

Open the linked repository before reading or editing it:

```bash
sase repo open chezmoi -r "Implement distinct no-Pomodoro and overdue-Pomodoro menu states"
```

Use the returned path for all `chezmoi` reads, edits, and checks. Do not refer to an
ephemeral numbered workspace path in source, documentation, or commit messages.

## Current behavior and the ambiguity to remove

The relevant command behavior is implemented twice for native/script parity:

- `src/native/pomodoro.rs` finds the latest open Pomodoro in today's ledger and emits
  `[OVERDUE by Nm] ...` only while it is 1 through 599 seconds overdue. At 600 seconds
  and beyond it exits successfully with empty stdout.
- `scripts/bob_pomodoro` has the same cutoff and is selected when
  `BOB_CLI_USE_SCRIPT=1`.
- The same empty successful result also means that the daily note is absent or no
  Pomodoro entry is open. Thus empty stdout alone cannot distinguish a stale open task
  from a closed final task, especially after a Hammerspoon reload.
- `status_from_env()` and `bob tmux-pomodoro` intentionally rely on the existing default
  cutoff; changing the default would alter notification and tmux behavior.

The linked `chezmoi` implementation currently has:

- `home/dot_hammerspoon/pomodoro_countdown.lua`, whose `presentation(remaining_seconds)`
  maps both `nil` and values at or below `-600` to `NO POMODORO` with a `missing`
  appearance;
- `home/dot_hammerspoon/init.lua`, which runs plain `bob pomodoro`, represents empty
  successful output as a renderable state without `endEpoch`, and applies bold red
  attributes to the `missing` appearance;
- `tests/hammerspoon/pomodoro_countdown_spec.lua`, which pins the conflated boundary.

Do not infer stale/open status from retained Hammerspoon memory or diagnostic stderr.
Retaining stale state would show a false overdue warning after the task is closed, and
debug text is not a stable machine-readable contract.

## Implementation

### 1. Add an opt-in stale-open status contract to `bob pomodoro`

Add a public `-s, --show-stale` option to `bob pomodoro` and its `bob_pomodoro`
compatibility binary/script. Follow the CLI memory rules: list the short and long forms
in alphabetically ordered help output and explain the behavior clearly.

In `src/native/pomodoro.rs`:

- extend the parsed options with `show_stale` and pass it into the status formatter;
- preserve the current default: a Pomodoro at 600 seconds overdue or later still
  produces no output unless the option is present;
- with `show_stale`, emit the same normalized
  `[OVERDUE by <whole-minutes>m] HHMM-HHMM <task>` shape for an open Pomodoro at and
  beyond the cutoff that is already used for recent overdue output;
- continue returning empty successful output when the daily note is absent or there is
  no open ledger entry, even when `show_stale` is set;
- keep `status_from_env()`, notification polling, and `run_tmux()` on the non-stale
  default so existing callers do not gain indefinitely stale output.

Mirror the option parsing, help, exact cutoff, and output behavior in
`scripts/bob_pomodoro`. Do not pass the option from `scripts/tmux_bob_pomodoro`.
Document the opt-in behavior near the `bob pomodoro` section of `README.md`, including
that it is intended for consumers that need to distinguish an old open entry from no
open entry.

Add CLI coverage in `tests/cli.rs` for both the native implementation and the script
fallback. Pin all of these cases:

- the existing default remains empty at exactly 600 seconds overdue;
- `--show-stale` reports the still-open fixture at exactly 600 seconds overdue with an
  `[OVERDUE by 10m]` prefix;
- the short `-s` alias reports a value beyond the cutoff;
- the no-open fixture remains empty with the option;
- `bob pomodoro --help`, the legacy binary help, and script-fallback help expose the
  documented short/long option without introducing a long-only public option.

### 2. Consume the opt-in contract from Hammerspoon

In linked-repository `home/dot_hammerspoon/init.lua`, change only the Hammerspoon status
command to invoke `bob pomodoro --show-stale`. The existing parser already accepts
`[OVERDUE by %d+m]`; keep that normalized interface and calculate `endEpoch` as today's
parsed end time so initial sync and reload can immediately identify a stale open
Pomodoro.

Keep successful empty output as the explicit renderable no-current-Pomodoro runtime
state with no `endEpoch`. A later non-empty result replaces it with a countdown/stale
state, and a later empty result replaces an overdue state, which is how closing the
final Pomodoro turns the warning green. Preserve the existing menu details, tooltip,
Last sync row, Refresh action, one-time zero-crossing sync, periodic sync, wake/unlock
sync, and retained runtime-object cleanup.

Do not add a fallback to plain `bob pomodoro`: once this source is deployed, the
installed command must support `--show-stale`; treating an unsupported option as empty
would falsely claim that no Pomodoro is open. Nonzero exits and all other command,
parse, task-creation/start, and callback failures must keep using the existing
hide-and-log paths.

### 3. Separate the pure presentation states and styles

In `home/dot_hammerspoon/pomodoro_countdown.lua`, rename the semantically obsolete
`MISSING_AFTER_SECONDS` constant to an overdue-warning name while retaining one
authoritative value of `10 * 60`. Make `presentation()` return four explicit states:

- `remaining_seconds == nil`: title `NO POMODORO`, appearance `missing`;
- `remaining_seconds >= 0`: existing `M:SS`, appearance `normal`;
- `-599 <= remaining_seconds < 0`: existing `+M:SS`, appearance `overdue`;
- `remaining_seconds <= -600`: title `OVERDUE POMODORO`, appearance `overdue_warning`.

Keep the module free of `hs` so boundary behavior remains testable on non-macOS CI.

In `home/dot_hammerspoon/init.lua`, map these appearances explicitly rather than using
an overdue catch-all:

- `normal`: the existing plain menu-bar title;
- `overdue`: the existing regular menu-bar font in red `#ff453a`;
- `missing`: a bold menu-bar font in green `#30d158`;
- `overdue_warning`: the same bold menu-bar font and red `#ff453a` attributes that
  `NO POMODORO` used before this change.

Derive bold fonts with `hs.styledtext.convertFont` and
`hs.styledtext.fontTraits.boldFont`; do not hard-code a font family or size. Keep the
style names aligned with their semantics so a future state cannot silently inherit the
wrong color.

### 4. Update pure-Lua boundary coverage

Revise `tests/hammerspoon/pomodoro_countdown_spec.lua` to assert the new constant,
titles, and appearance keys. Cover at least `601`, `0`, `-1`, `-599`, exact `-600`, a
value beyond the cutoff, and `nil`. The tests must prove that only `nil` maps to green
`NO POMODORO` semantics and only an actual stale countdown maps to red
`OVERDUE POMODORO` semantics.

## Validation

### `bob-cli`

From the current repository:

1. Run `cargo fmt --check` and `just check-scripts`.
2. Run the focused Pomodoro CLI tests, including native and script fallback:
   `cargo test --test cli pomodoro`.
3. Run `just all` for formatting, Clippy, the full Rust/unit/integration/parity suite,
   and documentation tests.
4. Exercise the built command against the open and no-open fixtures with `BOB_NOW`
   pinned at the exact 600-second cutoff. Confirm default stale output is empty,
   `--show-stale` emits `[OVERDUE by 10m]`, and the no-open fixture stays empty.
5. Review `git diff --check` and `git status --short`; only the intended Pomodoro
   command, fallback script, tests, and documentation should be changed.

### Linked `chezmoi`

From the path returned by `sase repo open`:

1. Run `stylua --check` on `home/dot_hammerspoon/init.lua`,
   `home/dot_hammerspoon/pomodoro_countdown.lua`, and
   `tests/hammerspoon/pomodoro_countdown_spec.lua`.
2. Run `luac -p` on both Hammerspoon implementation files.
3. Run `busted tests/hammerspoon/pomodoro_countdown_spec.lua`, then
   `just test-hammerspoon`.
4. Review `git diff --check`, `git status --short`, and a source-scoped `chezmoi diff`
   so the deployment diff is understood and no unrelated managed files are included.

### Deployment and visual smoke test

The Hammerspoon command now depends on a new option. Install/deploy the validated
`bob-cli` revision and confirm `bob pomodoro --show-stale --help` succeeds before
applying the linked `chezmoi` revision. After the normal commit/finalizer step, obey the
linked repository instruction to run `chezmoi update -a --force`; the path watcher
should reload Hammerspoon.

On the MacBook, verify:

- with the final Pomodoro task closed, bold green `NO POMODORO` remains present through
  a periodic refresh and a manual Refresh;
- creating/activating a Pomodoro replaces it with the normal countdown;
- an open Pomodoro from 1 through 599 seconds overdue shows the regular red `+M:SS`;
- at exactly 600 seconds overdue it changes without disappearing to bold red
  `OVERDUE POMODORO`, remains correct through a 15-second refresh and a Hammerspoon
  reload, and keeps Refresh available;
- closing that overdue Pomodoro changes the next successful refresh to bold green
  `NO POMODORO`;
- a deliberately induced command failure clears the menu and logs the error instead of
  showing either successful-state warning.

## Acceptance criteria

- Bold green `NO POMODORO` means the successful status result contains no open Pomodoro;
  it is not used merely because a known open timer is old.
- An open Pomodoro switches from regular red `+9:59`-style countdown text to bold red
  `OVERDUE POMODORO` at exactly 600 seconds overdue and remains so across refresh and
  reload until the entry is closed or replaced.
- Closing the final Pomodoro produces green `NO POMODORO` on the next successful
  refresh, including when the prior display was `OVERDUE POMODORO`.
- Default `bob pomodoro`, notification, and tmux stale-cutoff behavior remains
  backward-compatible; only callers that pass `-s/--show-stale` receive old open
  entries.
- Native and fallback-script command behavior, help, exact cutoff, absence behavior,
  pure-Lua presentation boundaries, full relevant suites, and the MacBook smoke checks
  all pass.
- Operational failures remain hidden and diagnostic rather than being rendered as a
  legitimate no-open or overdue state.
