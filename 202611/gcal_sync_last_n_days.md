---
tier: tale
size: small
title: "`bob gcal sync -n|--last-n-days N`: sync the last N days in one run"
goal:
  "Add a repeat-free way to catch up on recent days: `bob gcal sync -n N` syncs the N
  local days ending today, exactly as the equivalent repeated `-D` spelling would, in
  one batched run."
decisions:
  day_interaction:
    ask: Should -n combine with explicit -D days into one deduped run?
    choices:
      union:
        Merge into one sorted, deduped day set; -n 2 -D tomorrow syncs yesterday through
        tomorrow
      conflict: Reject -n together with an explicit -D; each run uses one spelling
    default: union
    why:
      The day set already dedupes and sorts, so union composes with zero special cases.
    answer: union
  n_bound:
    ask: Should -n reject values above 366?
    choices:
      capped: N must be 1..=366; larger values fail fast with a hint
      uncapped: Any positive N is accepted
    default: capped
    why:
      A full year covers any real catch-up and catches pasted-year typos like -n 2026.
    answer: capped
proposed_by: bbugyi200.apollo.dn.f1
proposed_with: claude/claude-fable-5
decided_by: reviewer
decided_via: tui
create_time: 2026-11-12 13:14:46
status: wip
---

# `bob gcal sync -n|--last-n-days N`: sync the last N days in one run

## Problem

Catching up after days away from the writer host means spelling every day by hand:
`bob gcal sync -D today -D yesterday -D 2026-11-10 …`. A count is the natural gesture —
"sync the last week" should be one short flag, not seven `-D` values, and it should ride
the multi-day batching that already makes one run cheaper than N runs.

## Design

### Semantics

`-n|--last-n-days N` selects the N local days **ending today**: today, yesterday, …,
today−(N−1). So:

- `bob gcal sync -n 1` ≡ `bob gcal sync -D today`
- `bob gcal sync -n 2` ≡ `bob gcal sync -D today -D yesterday`
- `bob gcal sync -n 7` ≡ the last week including today

**The governing invariant: `bob gcal sync -n N <flags>` is observably equivalent to the
expanded repeated-`-D` spelling** — identical day order, identical writes, identical
human and JSON output bytes, identical gog calls. Everything else is inherited, not
reimplemented:

- Days resolve, dedupe, and sort upfront into the existing `BTreeSet<NaiveDate>` in
  `resolve_days` (`src/native/gcal/sync.rs`); reconciliation stays oldest-first with
  stop-on-first-failure.
- The last N days are by construction **one contiguous date run ending today**, so the
  existing contiguous-run batching in `snapshot::FetchContext::fetch_days` already
  fetches them with exactly one events call per configured calendar — the optimal case,
  with no new fetch code.
- One resolved day prints the frozen single-day JSON shape; more than one prints the
  existing `schema_version: 1` wrapper with its `days` array (`selected_days > 1` in
  `run`). `-n 1` therefore stays byte-identical to `-D today`.
- `-F`, `-d`, `-q`, and past-day close semantics apply per selected day exactly as they
  do for repeated `-D`.
- "today" means `bob_env::current_datetime().date()`, so `BOB_NOW` pins it in tests,
  matching `-D today`.

> [!decision] day_interaction = union `-n` days and `-D` days merge into the same
> `BTreeSet`. `-n` always includes today, so the clap `default_value("today")` on the
> sync `day` arg is absorbed by the union with no special-casing: `-n 2` alone resolves
> to {yesterday, today} even though `args.days` carries the default "today".

> [!decision] day_interaction = conflict `SyncArgs::from_matches` must distinguish an
> explicit `-D` from the clap default (via `ArgMatches::value_source`) and
> `resolve_days` returns a `GcalError::setup` when both are explicit. The default
> "today" never conflicts.

### Validation and errors

Parse N in bob, not with a clap `value_parser!`, so a bad value is a `GcalError` that
honors `-f json` and carries a hint — exactly like a bad `-D` day from
`classify::resolve_day` — instead of a plain clap text error:

- The arg stays an `OsStringValueParser` string; `SyncArgs` gains
  `last_n_days: Option<String>` (trimmed, like `days`).
- `resolve_days` parses it: an integer in `1..=366` (decision `n_bound`) expands into
  today−(N−1)..=today inserted into the day set; anything else — `0`, negatives,
  non-integers, out-of-range — fails the **whole run before any fetch or write** with
  `GcalError::setup("bad_last_n_days", …)` and the hint
  `use an integer between 1 and 366 (1 = today only)`.

> [!decision] n_bound = uncapped Accept any N ≥ 1 that fits the date range; only `0` and
> non-integers error, and the hint drops the upper bound.

`record_last_error`'s fallback day (used when the failure precedes day resolution)
extends to consider a parseable `-n` expansion alongside `-D` values, so `doctor`'s
last-error day stays sensible for `-n`-only runs.

### CLI surface (per cli_rules memory)

- Long option `--last-n-days` with short alias `-n` (free on `sync`: b/D/d/F/f/h/q are
  taken), `value_name("N")`.
- Help stays alphabetically sorted: insert `.arg(last_n_days_arg())` between
  `help_arg()` and `quiet_arg()` in `sync_command()` (`src/native/gcal/cli.rs`).
- Arg help:
  `Sync the last N days ending today (2 = today and yesterday; combines with -D)` —
  adjusted to the chosen decisions.
- `sync` long_about gains one clause mentioning `-n` next to the repeat-`-D` clause;
  `sync_after_help` examples gain `bob gcal sync -n 7`; the top-level `bob gcal`
  after_help example list gains the same line with a short caption
  (`Catch up the last week, today included`).
- `list` and top-level `bob gcal` do **not** gain `-n`: they are single-day reports;
  widening them is out of scope.
- No rename or removal anywhere, so no alias work.

### Completion

Add an `Entry { path: &[], arg: "last-n-days", kind: Kind::FreeText }` with a short
comment to `src/native/completion/kinds.rs`, next to the existing `day` entry. The value
is a count, so free text is right; no new completion kind.

### Docs (per docs memory)

- `docs/gcal.md` synopsis line for `sync` gains `[-n|--last-n-days N]` in alphabetical
  position; while editing that line, make it list the full current option set (it is
  missing `-F|--force` and `-h|--help` today — verify against live `-h` before writing).
- `docs/gcal.md` § Sync: extend the existing "Repeated `-D` values resolve before any
  Google fetch; duplicate dates collapse" paragraph with the `-n` expansion rule (last N
  days ending today, union with `-D`, the `-n 2` ≡ `-D today -D yesterday` equivalence)
  and add `bob gcal sync -n 7` to the example block.
- `just docs-gen` regenerates the CLI reference from clap help; commit the regenerated
  pages.
- `just docs-check` currently fails on a pre-existing `bob-plugins` mount collision at
  `docs/reference/plugins/index.md` (tracked on the `bob-cli-it` bead). Treat that exact
  failure as pre-existing; any **other** docs-check diagnostic must be fixed.

## Implementation steps

1. **`src/native/gcal/cli.rs`** — add `last_n_days_arg()`; wire it into `sync_command()`
   in alphabetical position; add `last_n_days: Option<String>` to `SyncArgs` and
   `from_matches`; update `sync` long_about, `sync_after_help`, and `top_after_help`.
   Unit tests beside the existing `sync_args_accept_repeated_days`: `-n 3`,
   `--last-n-days 3`, `-n` combined with `-D`, and absence ⇒ `None`.
2. **`src/native/gcal/sync.rs`** — extend `resolve_days` with parse + expansion per the
   decisions; add the `bad_last_n_days` error with hint; extend `record_last_error`'s
   fallback-day scan. New unit tests for `resolve_days`: expansion, union/dedupe with
   `-D`, each rejection class.
3. **`src/native/completion/kinds.rs`** — the `last-n-days` entry.
4. **`docs/gcal.md`** — synopsis + § Sync prose + example, then `just docs-gen`.
5. **Integration tests in `tests/gcal_sync.rs`** (reusing `FakeGog`,
   `calendar_event_call_count`, and `BOB_NOW` pinning):
   - **Equivalence**: identical fixtures run with `-n 2` vs `-D today -D yesterday`
     produce byte-identical human output, identical final note bytes and journal state,
     and equal `calendar_event_call_count` (1, since the run is contiguous).
   - **Single-day shape**: `-n 1 -f json` output is byte-identical to
     `-D today -f json`.
   - **Union** (if `day_interaction = union`): `-n 2 -D tomorrow` reconciles yesterday →
     today → tomorrow in date order with `calendar_event_call_count == 1` (one
     contiguous run); the JSON wrapper lists three `days`. (If `conflict`: the
     combination errors before any fetch instead.)
   - **Rejections**: `-n 0`, `-n abc`, and (if capped) `-n 367` each exit with the
     `bad_last_n_days` setup error, print the JSON error shape under `-f json`, and make
     **zero** gog calls.
   - **Dry run**: `-n 2 -d` prints two day plans and writes nothing.
6. **Gates** — `cargo check`, focused suites (`gcal_sync`, `gcal_cli`, the `cli.rs` and
   `sync.rs` unit tests), `just fmt lint`, `just docs-gen`, `just docs-check`
   (pre-existing plugins-index failure excepted), `git diff --check`.

## Out of scope

- `-n` on `bob gcal list` or bare `bob gcal` (single-day reports today).
- Any change to fetch batching, the multi-day JSON wrapper schema, force semantics, or
  the sync journal.
- A "next N days" or arbitrary-range flag; `-D` already covers future days.
