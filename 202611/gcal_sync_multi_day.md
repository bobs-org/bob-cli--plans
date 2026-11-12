---
tier: tale
title: Repeatable -D for bob gcal sync
goal:
  bob gcal sync accepts -D multiple times and syncs every requested day in one fast run.
size: medium
decisions:
  fetch_batching:
    ask: Should one run batch Google event fetches across the requested days?
    choices:
      runs:
        One events call per calendar per contiguous day run, partitioned per day in bob
      per_day:
        One events call per calendar per day; only version/account/calendars setup is
        shared
    default: runs
    why:
      The common case (today plus tomorrow) becomes one events call per calendar total
    answer: runs
  json_shape:
    ask: Should multi-day JSON print one wrapper document instead of one line per day?
    choices:
      wrapper:
        One document with a days array holding the unchanged per-day report objects
      jsonl: One per-day report object per line as each day completes
    default: wrapper
    why:
      Every gcal JSON surface today is a single document; single-day output stays
      byte-identical
    answer: wrapper
  error_mode:
    ask: When one day fails mid-run, should the remaining days stop?
    choices:
      stop:
        Stop at the first failing day; completed days stay written; exit with that error
        code
      continue: Sync the remaining days, report every failure, exit non-zero
    default: stop
    why: Post-fetch day failures are rare and usually global; reruns are idempotent
    answer: stop
proposed_by: bbugyi200.apollo.dn
proposed_with: claude/claude-fable-5
decided_by: reviewer
decided_via: tui
create_time: 2026-11-12 10:54:38
status: wip
---

# `bob gcal sync`: repeatable `-D|--day` for multi-day sync

## Problem

`bob gcal sync -D DAY` reconciles exactly one day. Syncing several days (for example
today plus tomorrow after an evening of calendar edits) takes several `bob` runs, and
each run repeats the expensive Google setup: a gog version check, account resolution, a
calendar list call, and one `events` call per configured calendar — each one a gog
subprocess round-trip to Google.

Goal: `-D` may be given multiple times on `sync` (`bob gcal sync -D today -D tomorrow`),
each requested day is reconciled exactly as a single-day run would, and the whole run
does far fewer gog calls than N separate runs.

Non-goals:

- No range syntax (`-D a..b`) and no `--days N` window option.
- `bob gcal list -D` and the top-level `bob gcal -D` stay single-valued.
- No change to what a day's sync writes: plans, journal records, `-F` reset scope,
  stamps, MEET filing, and hold/waiting semantics are untouched per day.
- No cross-repo work: Bob Mac Capture does not drive `bob gcal sync`, and the single-day
  JSON report stays byte-identical, so no contract coordination is needed.

## Behavior contract

- `bob gcal sync` (no `-D`) behaves exactly as today (one day: today).
- `bob gcal sync -D a -D b …` resolves every day upfront (`today`, `tomorrow`,
  `yesterday`, or `YYYY-MM-DD`, per `classify::resolve_day`), rejects the whole run with
  the existing `bad_day` setup error (exit 2, nothing fetched or written) if any value
  is invalid, dedupes resolved dates, and processes them in ascending date order
  regardless of argument order.
- Each day is reconciled with the same bounded, idempotent semantics as a solo run:
  `bob gcal sync -D a -D b` must be observably equivalent to
  `bob gcal sync -D a && bob gcal sync -D b` (same notes, journal records, statuses, and
  exit codes), except that Google data for all days is fetched once upfront.
- `-d`, `-F`, `-q`, `-f`, and `-b` apply to every selected day. `-F` resets each
  selected day — the docs must say so explicitly, since `gcal-force-resets-the-day`
  frames `-F` as a single-day reset.
- A `waiting` or `held` day is not an error: it reports and the run continues to the
  next day (exit 0 overall if nothing errors), matching solo-run exit codes.
- Writer-host and config checks run once, before any fetch, exactly as today.

> [!decision] error_mode = stop A day-scoped error (failed lifecycle lookup, journal
> append, note write) stops the run after that day's error output; earlier days stay
> written. With `continue`, the loop instead records each day's error and keeps going,
> and the process exits with the first error's exit code.

## Performance design

Current per-day cost in `snapshot::fetch_day` (`src/native/gcal/snapshot.rs`):
`Gog::resolve` + `require_version` (1 subprocess), `account::resolve_command` (up to 1),
`gog.calendars` (1), then 1 `gog.events` per configured calendar. A naive loop over
`sync()` would repeat all of it per day.

Refactor `snapshot.rs` so multi-day sync shares the Google-side work:

1. **One fetch context per run.** Extract the transport setup (resolve gog, version
   check, account resolution, `calendars()` list) into a context struct built once in
   `sync::run` and reused for every day. The context also carries the configured
   timezone and `GcalConfig` reference.

2. **Batched events fetch.** Group the resolved days into contiguous runs (consecutive
   calendar dates). For each run and each configured calendar, make one `gog.events`
   call spanning `DayWindow::new(first_day).from_rfc3339()` →
   `DayWindow::new(last_day).to_rfc3339()`. Then partition the returned raw events into
   per-day candidate sets by replicating Google's `timeMin`/`timeMax` overlap semantics
   against each day's window:
   - timed event: include for day `d` iff
     `end > window(d).from && start < window(d).to`;
   - all-day event (exclusive end date): include iff `start_date <= d < end_date`.

   An event spanning midnight lands in both days' sets, exactly as two solo fetches
   would return it; `classify` already handles per-day inclusion (`outside_day` etc.) on
   top of that, so classification inputs are identical to today's.

   Parse each raw event once per day it lands in (via the existing
   `GcalEvent::from_google`), keeping the current per-day assembly code unchanged; dedup
   of parsing work across days is not worth the plumbing.

3. **Vault-local state stays per-day.** Series roots, the notes book, and the journal
   fold are cheap local reads, and day A's sync can _learn_ a series root or append
   journal rows that change day B's plan. To keep sequential-run equivalence, the
   per-day loop reloads `journal::load_state`, `journal::load_notes`, and
   `journal::load_series_roots`, and performs snapshot assembly (classify, root choice,
   clock resolution via `HomeResolver`) fresh for each day, in date order, _after_ the
   previous day's write. Call `resolver.refresh()` between days (or build a fresh
   resolver per day) so day A's note writes are visible to day B's home resolution.

4. **Everything else stays per-day**: the hold detection, lifecycle lookups for absent
   journaled rows (`gog.event`, rare), plan, vault lock acquisition, preimage-checked
   write with one retry, journal append, and `record_status`.

Net effect for `-D today -D tomorrow` with one calendar: 1 version + 1 account + 1
calendars + 1 events call (vs 8 subprocess calls for two solo runs today).

Keep a thin `fetch_day` wrapper with the current signature so `bob gcal list`
(`src/native/gcal/list.rs`) and single-day callers are untouched: it builds a context
for one day and assembles that day.

> [!decision] fetch_batching = per_day Skip step 2: keep one `gog.events` call per
> calendar per day, reusing only the shared context from step 1. No partitioning code;
> `fetch_day` keeps its current body minus the setup. Steps 1, 3, and 4 are unchanged.

Staleness note: all days' Google data is fetched before the first write, so a later
day's snapshot can be seconds older than in sequential solo runs. The existing locked
re-read/re-plan already tolerates note-side drift; Google-side drift within one run is
accepted (same exposure as a solo run's fetch-to-write window).

## CLI changes (`src/native/gcal/cli.rs`)

Per `cli_rules` memory: options stay alphabetical, the short alias stays, help stays
excellent, and nothing is renamed.

- Give `sync` its own day arg (`sync_day_arg()`): same `--day`/`-D`/`DAY`/default
  `today`, plus `.action(ArgAction::Append)`, help:
  `"Local day: today, tomorrow, yesterday, or YYYY-MM-DD (repeat to sync several days)"`.
  The top-level and `list` `day_arg()` keep single occurrence.
- `SyncArgs.day: String` → `days: Vec<String>` (trimmed values via `get_many`,
  defaulting to `["today"]`). Update the `cli.rs` unit tests (`sync_args_pin_options`)
  and add a repeated-`-D` parse test.
- `sync_after_help()` gains a multi-day example (`bob gcal sync -D today -D tomorrow`),
  and the `bob gcal` top-level examples add the same line if it fits the existing
  layout.
- `apply_note_day` (`src/native/gcal/sync.rs`) builds
  `SyncArgs { days: vec![day.to_string()], … }`.

## Sync orchestration (`src/native/gcal/sync.rs`)

- `run(args)` becomes: resolve config + writer check once → resolve/dedupe/sort days
  (fail fast on `bad_day`) → build the fetch context and batch-fetch raw events for all
  days (any fetch failure fails the whole run before any write, preserving the
  incomplete-snapshot doctrine from `bob-gcal-is-read-only-and-proposes-once`) → loop
  days in date order, running the existing single-day body (everything from the note
  read through `record_status`) against that day's prefetched events.
- Extract the current `sync()` body into a per-day function taking the shared context
  plus one day's candidate events; `apply_note_day` and the single-day path reuse it.
- `record_last_error` records against the day being processed when a day-scoped error
  stops the run; pre-loop failures record against the first resolved day (current
  behavior records against the lone `-D` day; with no resolvable day it already returns
  early).
- `is_today`, `LinkPhase`, stamps, and holds all already derive from the per-day
  `day`/`now` values — no change.

## Output

- **Human**: print each day's existing report block in day order as it completes, with
  one blank line between consecutive day blocks. Single-day output is unchanged.
- **Dry-run (`-d`, human)**: per-day unified diffs concatenate naturally (each starts
  with its own `---`/`+++` header); warnings stay on stderr per day.
- **Quiet (`-q`)**: unchanged per day (warnings only).
- **JSON (`-f json`)**: a run whose argv resolves to one day prints today's exact
  single-object document — byte-identical, no contract break (and no `!` on the
  release-train commit). A multi-day run prints one wrapper document at the end:

  ```json
  {"schema_version": 1, "ok": true, "days": [ <per-day report objects, unchanged shape> ]}
  ```

  When the run stops on a day error (`error_mode = stop`), the wrapper carries the
  completed days plus a top-level `"ok": false` and
  `"error": {"day": "YYYY-MM-DD", "kind": …, "message": …, "hint": …}`, and the process
  exits with that error's exit code. Pre-loop/global failures (bad day, writer host,
  config, fetch) keep today's existing top-level error document from `ui::report_error`,
  unchanged.

> [!decision] json_shape = jsonl Instead of the wrapper, a multi-day JSON run prints one
> per-day report object per line as each day completes, and a failing day prints the
> existing error document (plus a `"day"` key) as its line before the process exits. No
> buffering; document the line-oriented shape in `docs/gcal.md`.

## Tests

Integration tests in `tests/gcal_sync.rs` using the existing `gcal_support` harness
(`FakeGog` already logs every argv line to `BOB_GCAL_FAKE_LOG`, and serves an events
fixture independent of `--from`/`--to`, so one fixture can carry several days' events):

1. **Two-day write**: events on day A and day B in one fixture; `sync -D A -D B` writes
   each event into its own daily note only (partitioning correct), both
   journals/statuses recorded, exit 0.
2. **Call count**: the same run makes exactly one `calendar events` call per calendar
   (with `fetch_batching = runs`; two with `per_day`), and exactly one `calendars` call
   and one version check — assert on the fake log.
3. **Non-contiguous days** (`runs` only): `-D A -D C` with a gap makes two events calls
   per calendar, and no day between is written.
4. **Order and dedupe**: `-D B -D A -D A` processes A then B, syncing A once.
5. **Bad day fails fast**: `-D A -D nonsense` exits 2 with `bad_day`, no gog events
   call, no note or journal writes.
6. **Waiting day doesn't block**: day A's note missing, day B's present → A reports
   waiting, B is written, exit 0.
7. **Sequential equivalence for learning**: day A's note edit teaches a series root;
   `-D A -D B` gives day B's new row the learned root, matching two solo runs.
8. **Multi-day dry-run**: two diffs printed, nothing written, no journal appends.
9. **Multi-day `-F`**: both selected days reset (and only those days).
10. **JSON shapes**: multi-day wrapper has `schema_version`, `ok`, `days[]` with the
    unchanged per-day shape; single-day JSON remains covered by existing tests untouched
    (byte-compat guard). With `error_mode = stop`, a mid-run failure test asserts the
    wrapper's `error.day` and the exit code.
11. **Midnight-spanning event** (`runs` only): a timed event crossing the A→B boundary
    appears in both days' candidate sets exactly as two solo fetches return it (assert
    both notes' rows match solo-run output).

Unit tests: repeated `-D` parsing and `SyncArgs.days` in `src/native/gcal/cli.rs`;
window-overlap partition rules (timed boundary touch, all-day exclusive end) next to the
new partition code in `snapshot.rs`.

## Docs

Per the docs memory routing (new CLI flag → `just docs-gen` + clap help; behavior change
→ reference):

- `docs/gcal.md` § Sync: update the usage line (`-D … (repeatable)`), add a short "Sync
  several days" paragraph (order, dedupe, one fetch pass, waiting days don't block, `-F`
  resets every selected day), add the example, and document the multi-day JSON shape
  next to the existing report fields.
- Regenerate the CLI reference with `just docs-gen`; verify with `just docs-check`.
- No memory-note edits are needed: no decision record states single-day-only syncing as
  a rule, and `gcal-force-resets-the-day` stays true per selected day.

## Validation

- `just fmt lint` and `cargo test --test gcal_sync --test gcal_cli` plus the `cli.rs`
  unit tests; `just test` for the full suite.
- `just docs-gen && just docs-check`.
- Release-train rules: no version or CHANGELOG edits; the change is additive, so no `!`
  contract marker.
