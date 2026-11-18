---
tier: epic
title: 'bob tick: one ordered, configurable pass for Bob''s frequent background work'
goal: 'One `bob tick` command runs Bob''s more-than-daily background work (gcal, gmail,
  gkeep, refs, projects, tasks) in a fixed order. Each step has its own cadence, set
  in chezmoi''s Bob config. The Mac''s four staggered cron jobs become one 5-minute
  `bob-tick` job. Automated `bob gcal sync` runs stay at least an hour apart, and
  the command''s output, status board, and receipt explain what ran, what waited,
  and why.

  '
decisions:
  command_name:
    ask: Which name should the new orchestrator command use?
    choices:
      tick: One scheduler beat per cron run; crisp in crontab, help, and logs; the
        research pick
      tend: Frequent light care beside nightly; no vocabulary collisions; needs a
        gloss on first read
    default: tick
    why: Says what each run is (run what's due); 4 letters; no command collisions
    answer: tick
  decision_record:
    ask: Add a decisions-web record for the one-ordered-pass design (rejected alternatives
      included)?
    memory:
    - decisions:frequent-work-is-one-ordered-pass
    requested: Review the bob_periodic_orchestrator.md file in the research sidecar
      repo for context and inspiration before planning. I agree with all of the requirements
      recommended in that research file
    default: true
    answer: false
phases:
- id: engine
  title: Engine, CLI, and the three views
  depends_on: []
  size: medium
  description: 'engine: add src/native/tick/ with the strict tick: config block, the
    fixed six-step catalog, the pure planner, atomic host-local state, the run lock,
    subprocess supervision (SIGTERM, grace, SIGKILL; signal forwarding), generic exit-code
    outcomes, the run, -d, -s, -q, and -f json views, the last.json receipt with a
    Darwin notification, writer pinning, CLI registration in the Vault section, generated
    CLI docs, and fake-step integration tests.'
- id: gates
  title: Step gates and per-step adapters
  depends_on:
  - engine
  size: medium
  description: 'gates: add the today''s-note readiness gate for gcal and tasks; gcal''s
    once-a-day yesterday sync and its cooldown shared with gcal''s host status; the
    refs change gate (lib/xlib listing fingerprint, pre-scan hook, 6h ceiling); an
    additive gmail JSON busy field mapped to waiting; per-step JSON adapters with
    one-line summaries; and gate reasons in -d.'
- id: chezmoi_config
  title: Pin cadences in chezmoi
  depends_on:
  - engine
  size: xsmall
  description: 'chezmoi_config: add the commented tick: block (gcal 1h, gmail 1h,
    gkeep manual, refs, projects, tasks 15m) to the chezmoi Bob config, update the
    gcal comment, and validate it against the engine build and an older bob.'
- id: docs
  title: Contract page, decision record, and the bob sync tip
  depends_on:
  - gates
  size: small
  description: 'docs: write the docs/tick.md behavior contract with its mkdocs nav
    entry, add cross-links from the step commands'' pages, add the migration-notes
    line and the unknown `bob sync` tip, and author the decisions-web record, then
    regenerate docs.'
- id: mac_cutover
  title: Mac cron cutover
  depends_on:
  - gates
  - chezmoi_config
  - docs
  size: small
  description: 'mac_cutover: once the Mac''s bob has the gates build (wait with a
    monitor if not), replace the four Mac cron jobs with one */5 bob-tick job, delete
    the Highlights shell wrapper and its test, update the Mac-schedule doc sections,
    converge with cronctl, verify, and remove the stale bob_sync wrappers.'
proposed_by: bbugyi200.athena.1e2
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-18 13:06:49
status: done
bead_id: bob-cli-nb
---

- **PROMPT:** [prompts/202611/bob_tick.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/bob_tick.md)
- **BEAD:** [bob-cli-nb](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-nb/README.md)

# Plan: `bob tick` — one ordered pass for Bob's frequent background work

## Context

This plan implements the design in
`research:202611/bob_periodic_orchestrator/bob_periodic_orchestrator.md` (lead
consolidation of five reports). It builds on `research:202611/bob_sync_task_review.md`.
Bryan agreed to every requirement in the lead report. Where this plan settles something
the report left open, the choice is stated and justified here. Workers may read those
artifacts with `sase artifact read` for background, but this plan is the spec.

Today the Mac runs four staggered cron jobs (from chezmoi `home/.chezmoidata.yaml`):

| Job                   | Schedule              | Command                                  |
| --------------------- | --------------------- | ---------------------------------------- |
| `bob-highlights-sync` | `0,15,30,45 * * * *`  | `maybe_bob_highlights_sync -w`           |
| `bob-projects-sync`   | `5,20,35,50 * * * *`  | `bob projects sync`                      |
| `bob-task-reconcile`  | `10,25,40,55 * * * *` | `bob task reconcile --retry-timeout 120` |
| `bob-gmail-sync`      | `7 * * * *`           | `bob gmail sync -q`                      |

`bob gcal sync` runs only when Ledger Tools sees today's daily note created, plus
Bryan's morning `#gtd #pre #sync` chore. In nine days, 60 of 78 cron failures were false
alarms: 50 were "daily note does not exist yet" and 10 were the staggered jobs colliding
on the shared lock. Also, the Highlights wrapper's change gate never once skipped, which
costs about 1.7 hours of full-library scanning a day.

## Name

**`bob tick`** (see the `command_name` decision). Each run is one scheduler beat: check
what is due, run it in order, and report. Most runs do little or nothing, which "sync"
would misdescribe. Why not the other names:

- `sync` is a retired top-level name (`docs/develop/migration-notes.md`). It collides
  with `vault-sync` and with eight `<noun> sync` subcommands, and stale `~/bin/bob_sync`
  wrappers would start doing new work.
- `refresh` belongs to Task Freshness (`[refresh::]`).
- `upkeep` is the freshness budget's word.
- `pulse` describes the heartbeat, not the work.
- `tend` is the runner-up, offered in the decision.

Do **not** register `sync` as an alias. CLI alias rules cover renames, not new commands.
The `docs` phase adds a tip to the unknown-command error instead.

> [!decision] command_name = tend Use `tend` instead of `tick` everywhere this plan says
> `tick`:
>
> - the command (`bob tend`) and its `tend:` config block;
> - `$XDG_STATE_HOME/bob-cli/tend/` and `org.bobs.bob.tend-finished`;
> - `docs/tend.md`, the `src/native/tend/` module, `BOB_TEND_*` test hooks, and the
>   `bob-tend` cron job;
> - the `feat(tend)` commit scope.
>
> Behavior is otherwise identical.

## The contract (shared by every phase)

### Membership rule and steps

The membership rule: _a step belongs in `bob tick` if it should run more than once a day
on the writer host._

- Out of scope by that rule: `vault-sync` (the 15-second LaunchAgent), `nightly` (athena
  03:30), `plugins sync`, and every once-a-day job.
- The catalog is closed and the order is fixed. Order is a correctness property, not a
  preference: outside data comes in first, fastest first, and derived state comes last.

|   # | Step       | Kind   | Child command (all get `--bob-dir <vault>`)                           | Default  | Ceiling |
| --: | ---------- | ------ | --------------------------------------------------------------------- | -------- | ------- |
|   1 | `gcal`     | intake | `gcal sync -q -f json` (+ `-n 2` until yesterday is closed out today) | `1h`     | 10m     |
|   2 | `gmail`    | intake | `gmail sync -q -f json`                                               | `1h`     | 20m     |
|   3 | `gkeep`    | intake | `gkeep pull -f json`                                                  | `manual` | 20m     |
|   4 | `refs`     | intake | `ref scan -w -f json`                                                 | `15m`    | 30m     |
|   5 | `projects` | derive | `projects sync -f json`                                               | `15m`    | 10m     |
|   6 | `tasks`    | derive | `task reconcile --retry-timeout 120 -f json`                          | `15m`    | 10m     |

- gkeep is `manual` by default: it clips web pages and needs its own reviewed unattended
  policy.
- With stdin at `/dev/null`, URL-only Keep notes fall back to `gkeep_inbox`, so nothing
  prompts.

### Configuration (`tick:` in `~/.config/bob/config.yml`)

```yaml
tick:
  # writer: <host label>   # defaults to gcal.writer; unset = any host
  steps:
    gcal: 1h
    gmail: 1h
    gkeep: manual
    refs: 15m
    projects: 15m
    tasks: 15m
```

- **Values.** `<N>m`, `<N>h`, or `<N>d`, between 5m and 7d. Also `manual` (runs only
  when named or with `-a`) or `off` (never runs; YAML `false` also means off, as in the
  `gkeep.check_interval` precedent).
- **Defaults.** A missing block, a missing `steps`, or a missing step uses the defaults
  above.
- **Strict inside the block.** An unknown key, an unknown step, or a bad value is exit 2
  before any work, naming the key. Example:
  `tick.steps.gcal: "hourly" is not an interval; use 1h, manual, or off`.
- **Lenient outside it.** `RawConfig` keeps `tick` as a raw `serde_yaml::Value`, and
  only `bob tick` validates it. A bad block never breaks another command, and older
  binaries ignore the key.
- **Cadence only.** No user ordering, arbitrary commands, per-step flags, or per-step
  timeouts in v1. A step value may grow into a mapping later, additively.

### Scheduling rules (exact)

- **Due.** A scheduled step is due when
  `floor_minute(now) − floor_minute(last_started_at) ≥ interval`, or when it has never
  run.
  - Minute flooring absorbs cron start jitter, so `1h` with a 5-minute heartbeat means
    exactly hourly.
  - A `last_started_at` later than now (the clock moved backward) is ignored with a
    warning.
- **Per-step decision.** The decision is made at each step's turn, with `now` re-read
  then, not once per tick.
- **Clock.** A step's clock restarts whenever its child **ran**, even when it failed or
  timed out. That is what makes `gcal: 1h` a real cap: a failing gcal retries next hour,
  not next tick.
  - `waiting` and `skipped` never restart the clock. A step waiting on the daily note
    runs on the first tick after the note is ready.
  - `last_success_at` is tracked separately, so a step that keeps failing still looks
    broken.
- **Triggers.**
  - `scheduled` (bare `bob tick`) runs due steps.
  - `named` (`bob tick gmail gcal`) runs exactly those steps now, in catalog order,
    skipping only due-gates. Naming an `off` step is exit 2.
  - `all` (`-a`) runs every non-`off` step now, `manual` included.
  - Safety gates (writer host, note readiness) always apply. The refs change gate
    applies only to `scheduled`.
- **Intake, then derive.**
  - In a `scheduled` tick, `projects` and `tasks` (when their cadence is an interval)
    also run if any intake step ran earlier in that tick.
  - When an intake step ran but a derive step did not (it waited, was not named, or the
    process died), the derive step gets a persisted `follow_up_pending` flag. The next
    scheduled tick then runs it even if it is not due. Running the step clears the flag.
  - Named runs never broaden their own scope.
- **The gcal cap counts other callers.** gcal's due-gate uses the later of tick's own
  last start and the `ts` in gcal's host status (`$XDG_STATE_HOME/bob/gcal/status.json`,
  via `gcal::journal::read_status`) when that status's `outcome` is not `waiting`.
  - So the Ledger Tools note-creation run, or a manual `bob gcal sync`, pushes the next
    automated run out by an hour.
  - Explicit runs (`bob tick gcal`, `-a`, direct `bob gcal sync`) are not throttled.
    They count toward the next automated hour.
- **Times.**
  - State stores UTC RFC 3339 instants. "Today" and "yesterday" use Bob's configured
    timezone.
  - The scheduler clock is `Utc::now()`, overridable only by a hidden test hook
    `BOB_TICK_NOW` (RFC 3339). `BOB_NOW` stays the children's business and never moves
    scheduler state.

### Outcomes, exit codes, and gates

- **Per-step outcome.** Each step records one of:
  - `ok`
  - `failed`
  - `waiting`: a precondition is not met; retried next tick.
  - `skipped`: not due, `manual`, `off`, or nothing to do.
- **Reason codes.** Each outcome carries a stable `reason`: `due`, `never_ran`,
  `follows_intake`, `follow_up`, `named`, `all`, `not_due`, `manual`, `off`,
  `unchanged`, `note_missing`, `note_unrendered`, `pomodoros_missing`, `vault_busy`,
  `exit_nonzero`, `timeout`, `spawn_failed`, `interrupted`.
- **Exit status.**
  - **0** when every step that ran succeeded or only waited, including "nothing due" and
    "already running".
  - **1** when any step failed.
  - **2** for usage or config errors and for a wrong-host refusal.
  - Cron mail therefore always means a real problem.
- **Wrong host refuses.**
  - The host label is `pref_journal::host_label()`, the same one `bob gcal doctor`
    shows. The pin is `tick.writer`, falling back to `gcal.writer`; with neither set,
    there is no pin.
  - On a mismatch, run modes print `bob tick: this host is <h>; tick.writer is <w>` with
    a hint and exit 2.
  - This is a deliberate refinement of the report's "exit 0": if a pinned Mac's label
    drifts (`bob-cli-ew`), cron should mail, not go silently idle.
  - `-d` and `-s` work anywhere and show a banner.
- **Gates (the `gates` phase).**
  - **Readiness.** gcal and tasks wait (`note_missing` / `note_unrendered` /
    `pomodoros_missing`) until today's daily note exists, is Templater-rendered, and has
    `## Pomodoros`. This uses gcal's `WaitingReason` rules and is checked **before** any
    child or Google fetch.
  - **Change gate.** refs skips (`unchanged`) unless its change gate opens.
  - **Gmail busy.** A gmail busy-skip maps to `waiting` / `vault_busy` and does not
    restart the clock. gmail's own two-hour stale-busy failure still passes through as
    `failed`.
  - **gcal's own waiting.** A `waiting` result from the gcal child itself (a race after
    the preflight) is shown as `waiting` but **does** restart the clock, because it
    already fetched.

### Concurrency and supervision

- **No outer vault lock.** `bob tick` never holds `bob_sync.lock` across steps. Each
  child takes it itself with its existing bounded waits, so capture and gestures stay
  responsive.
- **Run lock.** `flock` on `$XDG_STATE_HOME/bob-cli/tick/run.lock` (fs2, via
  `bob_env::create_state_lock_file`). The holder writes `running.json` with pid,
  `started_at`, trigger, and the current step.
  - A contended non-interactive run prints `already running · started 10:15 · on refs`
    and exits 0. It writes no state and no receipt.
  - A contended interactive run (stdin and stdout both TTYs) waits with the existing
    spinner naming the step in flight, then proceeds.
- **Spawning.**
  - Children are `std::env::current_exe()`, or the hidden test hook `BOB_TICK_STEP_EXE`.
  - Each child gets explicit argv, no shell, and stdin `/dev/null`, in its own process
    group.
  - Use `subprocess_runner::run_raw` with bounded stdout (4 MiB) and stderr (64 KiB)
    caps.
- **Ceilings.** Ceilings are measured with `Instant`, which does not advance while macOS
  sleeps, so they count awake time.
  - At a ceiling, the step's group gets SIGTERM, then SIGKILL after a 30-second grace.
    The outcome is `failed` / `timeout`, never `waiting`.
  - Later steps still run once the group is reaped.
  - Add an optional `term_grace` (and an interrupt flag the wait loop polls) to
    `RawRunSpec`. Existing callers keep today's immediate SIGKILL.
- **Signal forwarding.** On SIGINT, SIGTERM, or SIGHUP, `bob tick` terminates the
  running step's group (SIGTERM, grace, SIGKILL), records `failed` / `interrupted`,
  writes state, and exits with 128+signal, so no step outlives its tick.
  - Use a `libc::sigaction` handler that sets an `AtomicBool`; there is no signal crate
    in the tree.
  - Document the one gap: SIGKILL of `bob tick` itself orphans the child.
- **Persistence.**
  - State is written atomically (temp file plus rename, dir 0700, file 0600) **after
    every step**, so a crash loses at most the step in flight.
  - Unreadable or corrupt state prints a one-line warning, treats every step as never
    run, and is rewritten. gcal stays protected by its host-status cooldown.

### CLI

```text
Usage: bob tick [OPTIONS] [STEP]...

Run Bob's due background work in one ordered pass: gcal → gmail → gkeep → refs →
projects → tasks. A step runs when its tick.steps interval has passed since it
last ran on this host, so cron can call bob tick every few minutes while each
step keeps its own pace. bob tick never holds the shared vault lock across steps.

Arguments:
  [STEP]...  Run only these steps now (due times skipped, safety gates kept):
             gcal, gmail, gkeep, refs, projects, tasks

Options:
  -a, --all              Run every step that is not off now, manual steps included
  -b, --bob-dir <DIR>    Bob vault root; defaults to BOB_DIR or ~/bob
  -d, --dry-run          Show what would run and why; run nothing, write nothing
  -f, --format <FORMAT>  Output format: human or json [default: human]
  -h, --help             Show help
  -q, --quiet            Print only failures
  -s, --status           Show each step's last result, last success, and next run

Steps, in run order (default cadence · ceiling):
  gcal      bob gcal sync       1h · 10m   today; yesterday too on the day's first run
  gmail     bob gmail sync      1h · 20m   file bob-labelled threads, watch replies
  gkeep     bob gkeep pull      manual     drain the Keep inbox
  refs      bob ref scan -w     15m · 30m  when lib/ or xlib/ changed, or every 6h
  projects  bob projects sync   15m · 10m  also after any intake step ran
  tasks     bob task reconcile  15m · 10m  also after any intake step ran

Examples:
  bob tick              Run every due step (what cron runs)
  bob tick -d           Show what is due, and why
  bob tick -s           Show the status board
  bob tick -a           Run everything now (the morning catch-up)
  bob tick gmail gcal   Run Gmail and Calendar now
  bob tick -f json      Print the run receipt

For vault Git sync, use `bob vault-sync`.
```

- **Help placement.** A leaf verb in the **Vault** help section, next to `nightly` (a
  group whose bare form mutates would break the CLI rules).
- **Options.** Sorted alphabetically, every long option with a short alias. `STEP`
  values complete from the catalog.
- **Conflicts.** `-d` and `-s` exclude each other. `-a` excludes `STEP`. `-s` excludes
  `-a`, and with `STEP` it filters rows.

### Output (the "beautiful" bar)

Glyphs are fixed, and the text stays readable with no color (cron logs, `NO_COLOR`):

| Glyph | Meaning             | Color  |
| ----- | ------------------- | ------ |
| `✓`   | ok                  | green  |
| `✗`   | failed              | red    |
| `◌`   | waiting             | yellow |
| `▸`   | would run           | cyan   |
| `·`   | not due / unchanged | dim    |
| `–`   | manual / off        | dim    |

Color goes through the existing `Styler` and only on a TTY. Columns are aligned. On a
TTY, the running step shows a spinner line with elapsed seconds that the result line
replaces.

**A run**: the header, then a line for each step that ran, waited, or failed, then the
summary **last**, so cronrun's 40-line failure mail always contains it:

```text
bob tick · Wed Nov 18 10:15 · kellys-macbook-pro-local
  ✓ gcal      today · 2 added · 1 link closed          2.1s
  ◌ gmail     vault busy · retrying next tick
  ✓ refs      2 PDFs changed · 1 note updated · 2 tasks  14s
  ✓ projects  no changes                               4.9s
  ✗ tasks     exit 1 · vault changed mid-write          11s
      │ bob task reconcile: guarded write refused: …
      │ hint: the next tick retries
  failed · 4 ran in 32s · next refs, projects, tasks 10:30 · gcal, gmail 11:15
```

- **Nothing due**: a run with nothing due prints one dim line:
  `bob tick · 10:20 · nothing due · next refs 10:30 · gcal, gmail 11:15`.
- **`-q`**: prints nothing unless a step failed; then it prints only the failed lines
  and the summary.
- **Failure detail**: a failed step shows up to 8 stderr lines behind a dim `│` gutter,
  plus the child's own hint when its JSON carries one.

**`-d` (plan)** shows all six steps and writes nothing:

```text
bob tick · plan · Wed 10:20 · kellys-macbook-pro-local
  · gcal      1h      next 11:15     last ✓ 10:15 · today · 2 added
  · gmail     1h      next 11:15     last ◌ 10:15 · vault busy
  – gkeep     manual                 last ✓ 07:31 (bob tick -a)
  ▸ refs      15m     due            lib/ changed since 10:15
  ▸ projects  15m     follows refs
  ◌ tasks     15m     waiting        today's note doesn't exist yet
  2 steps would run · dry run, nothing written
```

**`-s` (status board)** is read-only and works on any host:

```text
bob tick · status · kellys-macbook-pro-local · last tick 10:15 (failed, 32s)
  STEP      EVERY   LAST     RESULT                         LAST OK  NEXT
  gcal      1h      10:15 ✓  today · 2 added                10:15    11:15
  gmail     1h      10:15 ◌  vault busy                     09:15    due
  gkeep     manual  07:31 ✓  2 notes pulled                 07:31    —
  refs      15m     10:15 ✓  2 PDFs changed                 10:15    10:30 if changed · full 16:15
  projects  15m     10:15 ✓  no changes                     10:15    10:30
  tasks     15m     10:15 ✗  exit 1 · vault changed …       10:00    10:30
```

### State and receipt

- **Location.** Everything lives under `$XDG_STATE_HOME/bob-cli/tick/`
  (`bob_env::bob_cli_state_dir()`): `state.json` (scheduler memory), `last.json`
  (receipt), `run.lock`, and `running.json`. State is per host.
- **`state.json`.** `schema_version: 1`, `host`, `vault`, and `steps.<name>` with:
  - `last_started_at`, `last_finished_at`, `last_outcome`, `last_reason`,
    `last_summary`, `last_exit_code`, `last_success_at`, `follow_up_pending`;
  - step-specific additive fields from `gates`: gcal `yesterday_synced_on`; refs
    `fingerprint` and `last_full_scan_at`.
- **Receipt.** `-f json` prints, and every locked non-dry run writes atomically to
  `last.json`:
  - top level: `schema_version: 1`, `command: "tick"`, `host`, `vault`,
    `trigger: scheduled|named|all`, `started_at`, `finished_at`, `exit_code`, `steps`;
  - per step: `name`, `cadence`, `outcome`, `reason`, `detail`, `started_at`,
    `duration_ms`, `exit_code`, `summary`, `next_due_at`, `last_success_at`, and `gate`
    once `gates` lands.
- **Darwin notification.** On macOS, `bob tick` then posts `org.bobs.bob.tick-finished`.
  Move the `notify_post` FFI from `src/native/highlights_ref/scan_receipt.rs` into a
  shared helper rather than copying it. A future Bob Mac Capture health badge then needs
  no new protocol.
- **JSON for the other modes.** `-d -f json` and `-s -f json` print the same per-step
  shape, with `mode: plan|status`.
- **Privacy.** No raw mail or calendar text goes into receipts. Summaries are counts.

## Phase `engine` — Engine, CLI, and the three views

Build `src/native/tick/`. It ships a working command with generic step handling. Every
step here uses the default adapter: exit 0 → `ok` with summary `done`; anything else →
`failed` (`exit_nonzero`) with a stderr tail. The `gates` phase adds the smarts.

Suggested layout:

| Module        | Contents                                                                                               |
| ------------- | ------------------------------------------------------------------------------------------------------ |
| `mod.rs`      | `run(args) -> i32`, `help_text`, `completion_descriptor`                                               |
| `config.rs`   | `tick:` parse and validate, the interval grammar                                                       |
| `catalog.rs`  | The six `StepSpec`s: name, kind, default cadence, ceiling, blurb, argv builder, gate and adapter hooks |
| `schedule.rs` | **Pure** planner: `decide(step, cadence, state, now, trigger, tick_context) -> Selection`              |
| `state.rs`    | Atomic `state.json`                                                                                    |
| `lock.rs`     | Run lock, `running.json`, TTY wait                                                                     |
| `run.rs`      | Supervision loop and signal forwarding                                                                 |
| `render.rs`   | Run, `-d`, `-s`, `-q`                                                                                  |
| `receipt.rs`  | JSON shapes, `last.json`, the notification                                                             |

- **Hooks for `gates`.** Give `StepSpec` a `gate` hook (returns open / waiting{reason,
  detail} / skipped{reason, detail}) and an `adapter` hook (returns outcome, reason,
  summary, `consumes_interval`). Phase 1 supplies defaults, so `gates` only fills them
  in.
- **Dead code.** The lints run `-D unused`, so add only what this phase calls.
- **Registration.** Register `NativeCommand::Tick` in `src/runner.rs` as a Vault leaf
  with about-text
  `Run Bob's due background work in order: gcal, gmail, gkeep, refs, projects, tasks`.
  Add completion (STEP values), help tests, and option-order and short-alias checks
  (`tests/cli/help.rs` and `help_options.rs` patterns).
- **Docs.** Run `just docs-gen` so `docs/reference/cli/tick.md` and its nav entry exist.
- **Supervision.** Extend `src/native/subprocess_runner.rs` with `term_grace` and the
  interrupt flag, with unit tests next to the existing ones.
- **Test hooks.** Hidden variables, documented only in test code:
  - `BOB_TICK_STEP_EXE`: the fake child;
  - `BOB_TICK_NOW`: the scheduler clock;
  - `BOB_TICK_STEP_TIMEOUT_SECS`: shrinks every ceiling.

**Tests.** Unit-test `config.rs` and `schedule.rs` exhaustively with fixed clocks. Add
integration tests in `tests/cli/tick.rs` with a fake step program that echoes its argv,
sleeps, or exits N under test control. Isolate with temp `XDG_STATE_HOME`,
`BOB_CONFIG_FILE`, and vault. They must prove:

1. Repeated 5-minute ticks give refs, projects, and tasks every 15m and gcal and gmail
   hourly. Cron-jitter seconds do not slip an hour.
2. A failure restarts the clock: a failing gcal is not retried until the next hour.
   `last_success_at` stays old.
3. Named steps bypass due times. A named `off` step is exit 2 before anything runs. `-a`
   runs `manual`.
4. Follow-up: an intake step that ran pulls in projects and tasks. A named intake-only
   run sets `follow_up_pending`, which survives a killed run and fires on the next
   scheduled tick.
5. Run lock: a contended non-TTY run exits 0 with "already running" and leaves
   `last.json` and `state.json` untouched.
6. Timeout: a sleeping fake step is `failed` / `timeout`, its process group is gone, and
   later steps still run.
7. SIGTERM to `bob tick` kills the child's group and records `interrupted`.
8. Config errors (unknown step, unknown key, `1hr`, `0m`, `8d`) exit 2, naming the key,
   before any spawn. Another command (for example `bob plan`) still works with that bad
   block.
9. A writer mismatch is exit 2, while `-d` and `-s` still render with the banner.
10. `-d` and `-s` spawn nothing and write nothing.
11. A corrupt `state.json` warns and recovers.
12. The JSON receipt matches the schema, `last.json` is written, and the notification is
    `cfg(target_os = "macos")` only.
13. The human run, nothing-due, `-q`, `-d`, and `-s` output is snapshot-tested with
    color off.

**Gate.** `just fmt lint test`. The `cfg(macos)` code must compile on macOS: run
`scripts/dev/macos-ci-check` or watch the exact-SHA macOS CI run. Commit as
`feat(tick): …`. The release train ships it; never edit the Cargo version or CHANGELOG.

## Phase `gates` — Step gates and per-step adapters

- **Readiness gate.** Expose a small shared helper over gcal's `WaitingReason` (in
  `src/native/gcal/sync.rs`) that classifies today's daily note.
  - Use the same path `bob task reconcile` reads (`native_pomodoro::day_file_for`), and
    verify it matches the path gcal resolves for today.
  - Wire it as the gate for `gcal` and `tasks`. A closed gate is `waiting` with the
    code, spawns nothing, and leaves the clock alone.
  - `bob task reconcile`'s own CLI contract is unchanged.
- **gcal specifics.**
  - Add `-n 2` until a gcal run succeeds on the current local day, then record
    `yesterday_synced_on`. A failed first run retries `-n 2`.
  - Read the host-status cooldown described in the contract.
  - Map gcal's own `waiting` JSON (single-day and multi-day shapes) to `waiting`, with
    `consumes_interval: true`.
- **refs change gate** (scheduled trigger only). Run when due **and** any of the
  following holds; otherwise `skipped` / `unchanged` (the clock is untouched, and the
  gate is re-checked every tick):
  1. The listing fingerprint differs from the stored one. The fingerprint is a SHA-256
     over the sorted (relative path, size, mtime in ns) of every regular file under the
     resolved `lib/` and `xlib/`. Resolve those directories exactly as `bob ref scan`
     does (config, `BOB_HIGHLIGHTS_LIB_DIR` / `BOB_HIGHLIGHTS_XLIB_DIR`).
  2. A pre-scan hook is configured (`highlights.pre_scan_hook`, or a non-empty
     `BOB_HIGHLIGHTS_PRE_SCAN_HOOK`).
  3. The last successful full scan is older than 6h.

  More rules for the gate:
  - Store the fingerprint computed **before** the scan, and only on success. A change
    made mid-scan is never lost; a scan that writes markers costs at most one confirming
    rescan.
  - Named `refs` and `-a` always scan.
  - `-d` shows `lib/ changed since 10:15`, `unchanged since 10:15 · full scan by 16:15`,
    or `pre-scan hook configured`.

- **gmail busy.** Add an additive `"busy": <bool>` key to `bob gmail sync -f json` in
  `src/native/gmail/sync/report.rs` `json_document`. Cover it in `tests/gmail_sync.rs`.
  Map a busy, ok result to `waiting` / `vault_busy` without consuming the interval.
- **Per-step adapters.**
  - **Parsing.** Parse each child's fresh stdout JSON, never its last-result files.
    Malformed JSON with exit 0 is `ok` with summary `done (unreadable JSON)` and a
    warning. Nonzero is always `failed`.
  - **Child hints.** Surface the child's `hint` and error `code` when its JSON carries
    them.
  - **Concise summaries** (counts only):
    - gcal: added, refiled, closed, and `yesterday + today` when `-n 2` ran;
    - gmail: filed, watching, awaiting, archived;
    - gkeep: notes pulled;
    - refs: PDFs changed, notes created or updated, markers, tasks, and `held` in yellow
      when greater than 0;
    - projects: notes changed;
    - tasks: status changes.
  - **Fixtures.** Build them from each command's real `-f json` output.
- **Gate JSON.** Add the per-step `gate` object to the receipt and the `-d`/`-s` JSON,
  with `kind`, `open`, `reason`, and `detail`; refs adds `fingerprint_changed`, `hook`,
  and `full_scan_due_at`. The `mac_cutover` phase probes for this object.

**Tests.**

- **Readiness.** A missing note makes gcal and tasks wait. Once the note appears, the
  next tick runs them. An unrendered `<%` note waits with `note_unrendered`.
- **gcal.** `-n 2` once per local day, including the retry after a failed first run. A
  recent `synced` status suppresses an automated gcal; a `waiting` status does not.
- **gmail.** A busy result is not consumed.
- **refs.**
  - Each gate outcome: unchanged, PDF touched, xlib file added, hook, 6h ceiling, named.
  - The stored fingerprint is the pre-scan one.
- **Summaries.** Every adapter's summary, from the fixtures.

**Gate.** Same as `engine`; commit as `feat(tick): …`. The gmail JSON key is additive,
not a `!` break.

## Phase `chezmoi_config` — Pin cadences in chezmoi

Open the repo with `sase repo open chezmoi -r "…"`. Edit
`home/dot_config/bob/config.yml`.

1. Add a `tick:` block after the `gmail:` block, with all six steps pinned explicitly,
   so the hourly-gcal requirement lives in the source of truth, not in a default. Leave
   `writer` commented until `bob-cli-ew` fixes the Mac's drifting host label. Use this
   header comment:

   ```yaml
   # `bob tick` runs Bob's frequent background work in one fixed order:
   # gcal → gmail → gkeep → refs → projects → tasks. Mac cron (bob-tick) calls it
   # every 5 minutes; a step runs only when its interval has passed since it last
   # ran on this host (projects and tasks also follow any intake step that ran).
   # Values: an interval (15m, 1h, 1d), manual (only `bob tick -a` or
   # `bob tick <step>`), or off. `bob tick -d` shows what is due and why; see
   # docs/tick.md in bob-cli. Older bob builds ignore this block.
   tick:
     # writer: <label from `bob gcal doctor`>  # set once bob-cli-ew pins the Mac's label
     steps:
       gcal: 1h # never more often: each run rewrites today's ↻ synced line
       gmail: 1h # stays inside Gmail's two-hour staleness warning
       gkeep: manual # clips web pages; runs with the morning `bob tick -a`
       refs: 15m # scans only when lib/ or xlib/ changed (full scan ≥ every 6h)
       projects: 15m
       tasks: 15m
   ```

2. Update the `gcal:` comment so it no longer says the Mac syncs only by hand. New text:
   `bob tick` runs it at most hourly on the Mac (`tick.steps.gcal`). Ledger Tools also
   runs it once when today's note is created, and manual runs count toward the hour.
3. Validate with a bob-cli build that contains `engine`:
   `BOB_CONFIG_FILE=<checkout>/home/dot_config/bob/config.yml cargo run -q -- tick -d`
   shows the six cadences with no config error. Confirm the installed (older) `bob`
   still runs `bob plan` with that file.
4. Commit to chezmoi (for example
   `feat(bob): pin bob tick cadences; gcal at most hourly`). Do not run `chezmoi apply`
   or touch any machine; `mac_cutover` converges the Mac.

## Phase `docs` — Contract page, decision record, and the `bob sync` tip

Read `sase/memory/docs.md` (via `/sase_memory_read`) first.

- **`docs/tick.md`.** The behavior contract, added to the mkdocs **Contracts** nav. It
  covers:
  - the purpose and membership rule, and the steps table;
  - config grammar and defaults, and the exact scheduling rules (due, clock, triggers,
    follow-up, gcal cooldown);
  - gates, outcomes, reasons, and exit codes;
  - locking, supervision, and signals;
  - host pinning, output, the receipt schema, and the notification;
  - examples, and troubleshooting (`bob tick -d`, `bob tick -s`,
    `cronctl logs bob-tick --machine mac`).
- **Cross-links.** In `docs/gcal.md`, `docs/gmail.md`, `docs/gkeep.md`,
  `docs/highlights-ref-sync.md`, `docs/projects.md`, and `docs/task-status-hooks.md`,
  add one sentence each saying the command can run as a `bob tick` step, with a link. Do
  **not** describe the Mac's live schedule here; the `mac_cutover` phase owns those
  sections, so the docs never claim a schedule the Mac isn't running yet.
- **`bob sync` tip.** `docs/develop/migration-notes.md` gets one line: `bob sync` stays
  retired; the periodic orchestrator is `bob tick`. When the unknown top-level command
  is exactly `sync`, append to clap's unrecognized-subcommand error:
  ``tip: for vault Git use `bob vault-sync`; for Bob's periodic background work use `bob tick` ``.
  The exit code is unchanged. Test it next to the existing help and alias tests.
- **Decision record.**

  > [!decision] decision_record Use `/sase_memory_write`, then add the strand
  > `decisions:frequent-work-is-one-ordered-pass` with a one-sentence roster summary.
  >
  > **Applies to.** bob-cli, chezmoi (the Mac cron), and the vault.
  >
  > **Claim.** Bob's more-than-daily work on the writer host is one `bob tick` pass, run
  > from one 5-minute Mac cron line in the fixed order gcal → gmail → gkeep → refs →
  > projects → tasks.
  >
  > - Config sets only each step's cadence (interval, `manual`, or `off`); membership,
  >   order, and arguments are code.
  > - Steps are supervised subprocesses that take the vault lock themselves.
  > - Gates make a step wait or skip without restarting its clock. A step that ran
  >   restarts its clock even when it failed, so automated gcal runs stay at least an
  >   hour apart, and other gcal callers count toward that hour.
  >
  > **Why.** Cite
  > `research:202611/bob_periodic_orchestrator/bob_periodic_orchestrator.md` and
  > `research:202611/bob_sync_task_review.md`: 60 of 78 failures were false alarms;
  > minute offsets could not guarantee the fetch → file → tidy order; and gcal had no
  > periodic sync. Rejected:
  >
  > - four jobs plus a fifth gcal job (no order, no catch-up after wake);
  > - launchd agents (not managed by cronctl);
  > - a daemon or file watcher;
  > - folding the work into `bob task reconcile` (CLI rules);
  > - in-process steps like `nightly` (no real timeout);
  > - user-configurable order or commands;
  > - reusing `bob sync`.
  >
  > **Cost.** Cadence lives in config, not the crontab. A wedged step delays later steps
  > up to its ceiling. Manual runs are not throttled. State is host-local.
  >
  > **Reopens when.** A step must run on a different host than the others, measured
  > post-wake latency makes cron unacceptable, or a step earns typed per-step options.
  >
  > **Evidence.** This plan, `docs/tick.md`, and `tests/cli/tick.rs`.
  >
  > Run `sase memory init`, then `just docs-gen` (which regenerates
  > `docs/develop/decisions.md`).

**Gate.** `just docs-check` and `just test` for the tip. Commit as `docs(tick): …` (and
`feat(cli): …` for the tip if it is separate).

## Phase `mac_cutover` — Mac cron cutover

Use the `/crontab` skill (and `/sase_monitor` for every wait) and follow it exactly.
Never use raw `crontab`.

1. **Binary check.** Run:

   ```bash
   ssh -o BatchMode=yes -o ConnectTimeout=10 mac \
     'PATH="$HOME/bin:$HOME/.cargo/bin:/opt/homebrew/bin:/usr/bin:/bin" bob tick -d -f json'
   ```

   It must succeed, and its `refs` step must carry a `gate` object, which proves the
   `gates` build is installed.
   - If not, or if the Mac is asleep, stage a `/sase_monitor` polling loop (every 10
     minutes, up to 72h) and end the turn. Its `--next` resumes this phase.
   - Bryan installs bob on the Mac (`just install-all` there, or the release train).
     Never install it yourself.
   - A timeout means: report `pending` and stop.

2. **chezmoi.** In `home/.chezmoidata.yaml`, replace the four `mac` jobs and their
   staggering comment with:

   ```yaml
   # mac: one ordered pass. `bob tick` decides what is due from tick.steps in
   # ~/.config/bob/config.yml (gcal and gmail hourly; refs, projects, tasks
   # every 15 minutes; gkeep manual) and runs it gcal → gmail → gkeep → refs →
   # projects → tasks. See bob-cli docs/tick.md; `bob tick -d` shows the plan.
   - name: bob-tick
     machines: [mac]
     schedule: "*/5 * * * *"
     description: Run Bob's due background work in one ordered pass
     command: bob tick
   ```

   Plain `bob tick`, not `-q`: a nothing-due tick is one line, so
   `cronctl logs bob-tick` stays a readable history, and cron mails only on exit 1 or 2.

   Delete `home/bin/executable_maybe_bob_highlights_sync` and
   `tests/bash/maybe_bob_highlights_sync_test.sh`, and grep the repo for leftover
   references. The tick's refs gate replaces both.

3. **Check.** Run `cronctl check --source <checkout>`, then
   `cronctl diff mac --source <checkout>`. Expect four jobs out and one in. Then dry-run
   the job by streaming the checkout's cronrun to the Mac, as the skill shows
   (`--dry-run bob-tick 'bob tick'`).
4. **bob-cli docs** (they now describe the live schedule). Rewrite:
   - `docs/vault-git-sync.md` § Mac scheduled maintenance, plus the later "exact
     schedule" mention;
   - `docs/gcal.md` § Where it runs ("no periodic sync" becomes "hourly via `bob tick`;
     Ledger Tools still runs it at note creation");
   - `docs/gmail.md` § Where it runs;
   - `docs/highlights-ref-sync.md` § Scheduled Scan (the wrapper becomes the tick refs
     gate).

   Run `just docs-check`. Commit as `docs(tick): …`.

5. **Converge.** Stage `cronctl apply mac` with the skill's post-commit monitor, then
   finish with commit decisions for chezmoi and bob-cli. The successor reports
   `cronctl status mac` as `in_sync`, or honestly as `pending` if the Mac sleeps. Where
   reachable, it verifies:
   - `ssh mac '… grep -A9 "^tick:" ~/.config/bob/config.yml'` shows `gcal: 1h`;
   - after the first tick, `ssh mac '… bob tick -s'` shows steps running.

   If `config.yml` did not converge, Bryan runs `chezmoi update -a` on the Mac. Say so;
   don't fake it.

6. **Stale wrappers.** On athena (local) and apollo, delete `~/bin/bob_sync` only if its
   content is exactly the retired `exec bob sync "$@"` wrapper. Report what was removed.
7. **Rollback.** Revert the cron commit and `cronctl apply mac`. The individual commands
   remain installed.
8. **Follow-ups.** Record these as `PROPOSED FOLLOW-UP:` notes on the phase bead:
   - change the `gtd_daily` `#pre #sync` chore to `bob tick -a` after a soak week;
   - set `tick.writer` and `gcal.writer` once `bob-cli-ew` lands;
   - a Bob Mac Capture health badge reading `last.json` and
     `org.bobs.bob.tick-finished`;
   - a per-PDF unchanged-skip cache inside `bob ref scan`.

## Out of scope

- Changing `vault-sync`, `nightly`, athena and apollo intake timers, or
  `bob task reconcile`'s own exit contract.
- Throttling direct or manual `bob gcal sync` calls (the lead report's reading of "not
  more than hourly" applies to automated runs; manual runs count toward the hour).
- Unattended gkeep, configurable timeouts or order, a launchd scheduler, and a Mac
  Capture UI.

## Verification (epic done)

- On the Mac, `cronctl status mac` is `in_sync` with exactly one `bob-tick` job.
- `bob tick -s` there shows gcal and gmail hourly and refs, projects, and tasks on
  15-minute cadence.
- Overnight, the readiness gates produce `waiting` instead of mail.
- In `cronctl logs bob-tick --machine mac`, gcal runs are at least 60 minutes apart, and
  refs scans appear only after library changes or the 6h ceiling.
- `just fmt lint test` and `just docs-check` pass on bob-cli master.
