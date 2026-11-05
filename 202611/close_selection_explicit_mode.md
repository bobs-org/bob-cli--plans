---
tier: tale
size: medium
title: Close selections defer every unlisted link once any task number is typed
goal:
  An `x` or `x=` close includes a Task Link by default only when no task number is
  typed; otherwise only the links it references are included.
decisions:
  drop_lists_explicit:
    ask:
      Should a `~<K>` list alone (for example `x~2`) also switch the close to explicit
      mode?
    choices:
      any_number:
        x~2 drops 2 and defers the rest (same as x0~2); write x1,3~2 to drop 2 and work
        1 and 3
      drop_only:
        "`~` only removes: x~2 drops 2 and every other link keeps its ledger default"
    default: any_number
    why: One rule (any number means you are listing) with no per-marker exception
    answer: any_number
  event_clock_in_explicit:
    ask:
      In explicit mode, should unlisted Event Task links defer instead of taking their
      clock default?
    choices:
      defer_unlisted:
        x!3 defers an unlisted meeting that ended inside the session; write x!1,3 to
        count it
      keep_clock:
        Unlisted Event Tasks keep clock defaults in every mode, as a documented
        exception
    default: defer_unlisted
    why:
      A clock default is still a default include; keeping it would restore an exception
    answer: keep_clock
  memory_close_selection_record:
    ask:
      Add decision record close-selections-list-explicitly and mark the Event Task clock
      record superseded-in-part?
    memory:
      - decisions:close-selections-list-explicitly
      - decisions:event-task-links-close-by-the-clock
    default: true
    corrects:
      On a timed Pomodoro close, an unlisted plain Task Link to an Event Task
      ([[glossary:event-task]]) in the same canonical daily note is classified by its
      local span and the session start `S` and close time `C`
    answer: true
proposed_by: bbugyi200.athena.18o
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-05 13:24:02
status: wip
---

# Close selections: any typed task number means "only what I listed"

## Problem

`x[<N>][*<P>][!<M>][~<K>][@<N>]` (whole-item `x`, link suffixes `=x…`, and the `x=E=X`
past close) decides each numbered Task Link's outcome. Today a close switches into
"selection mode" (unlisted plain links are **deferred**) only when `<N>` is typed
(including `0` / `@<N>`) or `*<P>` has numbers. With `!<M>` or `~<K>` alone, every
unlisted plain link keeps its **ledger default**: In Progress, 🍅 in the closed entry,
carried, task started.

So `x*2` counts only task 2, while `x!2` counts task 2 **and** every other plain link.
In selection mode, unlisted Event Task links also keep their clock defaults: `x2` still
completes an unlisted meeting that ended inside the session.

## Policy

1. **Default mode**: no task number typed anywhere (`x`, `x*`, `x!`, `x` with only a
   Work Log). Every link not covered by a wildcard takes its ledger default, or its
   Event Task clock default. This is unchanged.
2. **Explicit mode**: any task number typed in `<N>` (including `0` and `@<N>`), `*<P>`,
   `!<M>`, or `~<K>` (`~` per `drop_lists_explicit`). Only referenced links are
   included:
   - `<N>` stays in progress.
   - `*<P>` parks.
   - `!<M>` completes.
   - `~<K>` drops.
   - An empty `*` / `!` wildcard references "every remaining link", so `x1*`, `x*!2`,
     and `x!~2` keep today's meaning.
   - Every other plain link is **deferred**: no 🍅, carried, not started. That includes
     unlisted Event Task links of any clock class (per `event_clock_in_explicit`).
   - Authored markers keep their authored meaning, as they do today: `[[T]]#` stays
     deferred, and a hand-written `![[T]]` stays complete. The person already made an
     explicit choice by hand, so this is not a default include.
3. All close forms share one resolver, so the rule covers the following with no per-form
   code. Verify each with a test.
   - whole-item `x…`
   - `@route:id=x…`, `^route:id=x…`, and `<text> @route:id=x…`
   - past closes `x=E=X`

> [!decision] drop_lists_explicit = any_number `~<K>` counts as a typed task number in
> explicit mode above. Under `drop_only`, a close with only `~<K>` lists stays in
> default mode.

> [!decision] event_clock_in_explicit = defer_unlisted Unlisted Event Task links defer
> in explicit mode. Under `keep_clock`, they keep their clock outcome in every mode, and
> docs name that as the one exception.

Consequences to state in docs:

- `x0` followed by any other list (`x0!2`, `x0~2`, `x0*2`) now equals the same token
  without `0`. These spellings stay valid.
- A plain note-free `x0` is still the reset.

## Implementation

### 1. One explicit-mode predicate (Rust)

Add one canonical predicate and route every copy of the old rule through it. Example
names are `CloseSelection::is_explicit()` and `PomodoroCloseSpec::is_explicit()`, both
backed by one free function in `src/native/capture_language/close_log.rs` or `model.rs`.
The predicate is:

```
in_progress.is_some() || !park.is_empty() || !complete.is_empty()
    || (DROP_IS_EXPLICIT && !drop.is_empty())
```

Wildcards are handled separately where they matter: lexical positions defer to the
runtime lineup. Inline the `drop` term according to `drop_lists_explicit`; do not add a
runtime flag.

The old rule lives in these places. Replace each one:

- `src/native/capture_pomodoro_close/selection.rs`
  - `CloseSelection::has_work_selection()` (also controls the `ledger` vs `unlisted`
    source label for Event Tasks). Make it `is_explicit() || park_all || complete_all`.
  - `outcome_for()`: replace the two tail branches (`if let Some(in_progress)…` and
    `if !selection.park.is_empty()…`) with one explicit-mode branch: embedded →
    Complete/Unlisted, otherwise Deferred/Unlisted.
  - If `event_clock_in_explicit = defer_unlisted`, move the Event Task clock branch
    below the explicit check. Clock defaults then apply only in default mode and for the
    wildcard-skipped `not_started` case. Under `keep_clock`, keep the clock branch where
    it is.
  - Keep the `not_started` wildcard exclusion. In explicit mode the skipped event
    resolves to Deferred either way.
- `src/native/capture_language/close_log.rs`
  - `is_loggable()`, `default_log_index()`, and `lexical_worked_tasks()` use the
    predicate.
  - `x!2 note` now logs lexically to task 2.
  - `x~2 note` (if `~` is explicit) fails with the existing "works no task" error,
    exactly like `x0~2 note`.
  - Fix `not_worked_suggestions()`'s `None` / empty-park arm, which becomes reachable
    for `x!2` or `x~3` logging to an unlisted task. It currently returns `x{i}` /
    `x!{i}` and drops the typed `!`/`~` lists. It should keep them: `x!2~3` logging to 1
    suggests `x1!2~3` / `x!1,2~3`.
- `src/native/capture_parse.rs`
  - `format_pomodoro_close` and `format_past_close_summary` duplicate the predicate. Use
    the shared one.
  - `x!2` then reads `x!2 (complete 2 · defer the rest)`.
  - Update the `capture-parse` help text near line 297–306 if its wording names the old
    trigger.
- Doc comments that state the old rule:
  - `close_selection.rs`: the `CloseSelectionLex` doc ("keep their ledger outcome unless
    `*<P>` is present").
  - `model.rs`: around line 266.
  - `close_log.rs`: module docs, lines 24–38.
- Search for stragglers before finishing:
  `rg -n 'in_progress.is_some\(\)|park.is_empty\(\)' src/native`, then inspect
  `editor_pomodoro.rs` hints and completion text for "ledger outcome", "unlisted", or
  the old trigger wording.

### 2. Tests

Update the expectations the change breaks; most single-link `x!1` / `x~1` cases are
unchanged. Add focused coverage:

- `src/native/capture_pomodoro_close/selection_tests.rs`, with a 3-link lineup:
  - `x!2` defers plain 1 and 3 (source `unlisted`).
  - `x~2` defers 1 and 3 (or keeps their ledger outcomes under
    `drop_lists_explicit = drop_only`).
  - `x!2` keeps an unlisted `![[T]]` complete and an unlisted `[[T]]#` deferred.
  - These are unchanged: `x`, `x!`, `x*`, `x1*`, `x*!2`, `x!~2`, `x2!1`.
  - Event Tasks: `x!3` with unlisted ended_inside and running meetings gives
    deferred/deferred (or complete/in_progress under `keep_clock`).
- `tests/fixtures/event_task_close/vectors.json`, if
  `event_clock_in_explicit = defer_unlisted`:
  - The rust_only vector
    `explicit-x2-wins-for-not-started-event-and-clock-applies-to-unlisted` expects
    `["deferred", "in_progress"]`. Rename its id to say unlisted links defer.
  - Add a rust_only `x!2` vector with an unlisted running meeting.
- `tests/fixtures/pomodoro_clock/vectors.json`, if `drop_lists_explicit = any_number`:
  - Vector `close-mixed-after-dropping-on-clock` uses `x~2` to test the off-clock
    restamp after dropping an on-clock link. Change its token to `x1~2` so it keeps
    testing that. Keep the id and expected day.
- Close log tests (`tests/cli/capture/pomodoro_close_log.rs` and the lexer tests):
  - `x!2 note` logs to 2.
  - `x!2` with `- 1 note` reports "task 1 isn't worked" with suggestions `x1!2` /
    `x!1,2`.
  - `x~2 note` (explicit) reports "works no task".
- `capture-parse` summaries (`tests/cli/capture/parse_pomodoro_close.rs`,
  `parse_pomodoro_past.rs`): `x!2` and `x~2` end with `defer the rest`.
- One CLI dry-run per close form on a multi-link fixture, asserting that unlisted links
  are deferred and carried:
  - whole-item `x!2`
  - a link form `^route:id=x!N` (the linked task numbers last)
  - a past close `x=fix=!2` (or the current past-close spelling in
    `docs/reference/capture-syntax.md`)
- Close names (`tests/cli/capture/pomodoro_close_names.rs`): on a `BUILD + FIX` session
  where `x!2` completes the BUILD link, the closed entry settles to `BUILD`, and the
  carried FIX link keeps FIX on the continuation.

Run `just` targets the repo uses for CI, at least `cargo test --all-targets`, clippy,
and fmt. All must pass.

### 3. Docs

Read the `docs.md` reference memory first
(`sase memory read docs.md -r "updating capture close docs"`).

Then update these:

- `docs/capture.md` § Choosing each Task Link's outcome:
  - Rewrite the mode paragraph ("`<N>` omitted leaves unlisted links at their ledger
    outcome unless `*<P>` is present…").
  - Update the **Outcomes** table rows (`<N>` typed or `*<P>` present → "any task number
    typed").
  - Update "Unlisted plain links with `<N>` or `*<P>` present defer…", "`x!2~3` without
    `<N>` or `*<P>` keeps ledger outcomes…", and "An omitted `<N>` keeps the ledger
    default…".
  - Note that `x0!2` ≡ `x!2`.
- `docs/capture.md` § Event Task links close by the clock: clock defaults apply only
  when no task number is typed. Per decision, update "unlisted Event Tasks use the clock
  defaults above" and the explicit-numbered sentence.
- `docs/capture.md` § Logging work while closing: the selection-mode wording for
  positional bullets.
- `docs/reference/capture-syntax.md`: § `x` (selection pieces), § `x!<M>` ("`x!1`
  narrows to task 1" → completes 1 and defers the rest), § `x~<K>`, and the Work Log
  section.
- `docs/reference/cli/capture.md`, `docs/reference/cli/capture-parse.md`, and
  `docs/reference/json-outputs.md`: wherever they describe `source: ledger|unlisted` or
  the summary text.
- Grep `docs/` for `ledger outcome`, `selection mode`, `unlisted`, and `*<P>` is
  present`, and fix every stale statement. Run the docs build or check from the `docs.md`
  memory.
- Do not edit `CHANGELOG.md` or the `Cargo.toml` version (release train).

### 4. Cross-repo checks (read-only unless a pin breaks)

Open each repo with `/sase_repo`.

- **bob-plugins**: confirm the plugins do not parse `x` selections. The shared selection
  vectors are all `rust_only`. Check how `event_task_close/vectors.json` and
  `pomodoro_clock/vectors.json` are vendored. If a byte-parity or sync check would fail,
  update the vendored copy there, run that repo's tests, and commit it in that repo. No
  plugin deploy is needed for a fixture-only change.
- **bob-mac-capture**: it is a thin client, but grep its tests and fixtures for pinned
  `capture-parse` close summaries (`defer the rest`, `x!`, `x~`) and for `source`
  handling. If anything is pinned, do not push a Mac change blind. File a follow-up task
  bead through `/sase_new_task` naming the pinned strings.

### 5. Memory

> [!decision] memory_close_selection_record

Use `/sase_memory_write` before editing. This one toggle covers both notes, because an
accepted record is immutable: the only way to correct the event record's claim is a
superseding record plus a mark on the old one.

- New strand `sase/memory/decisions/close-selections-list-explicitly.md`:
  - Claim: any typed task number makes a close explicit; only referenced links are
    included, unlisted plain links defer, and authored markers and wildcards keep their
    meaning.
  - Why it beat the alternatives: `*`-only triggering (today), and `~` never triggering,
    if that was rejected.
  - Cost: `x~2` / `x!2` now need the worked tasks listed.
  - Reopens when: listing every worked link becomes routine friction.
  - Evidence: this plan, the `docs/capture.md` section, and the vectors.
  - Record the reviewer's answers to the two policy decisions.
  - Link `[[decisions/event-task-links-close-by-the-clock]]`.
- Under `event_clock_in_explicit = defer_unlisted`, mark
  `sase/memory/decisions/event-task-links-close-by-the-clock.md` with
  `metadata.status: superseded-in-part` and
  `superseded_by: [[decisions/close-selections-list-explicitly]]`. Add one back-link
  sentence saying unlisted Event Task links defer in explicit mode. Do not otherwise
  edit the accepted body. Under `keep_clock`, leave that record untouched.
- Run `sase memory init`.
- If this toggle is off, file one `memory` task bead through `/sase_new_task` with the
  proposed record and mark instead.

### 6. Commit

One commit, marked breaking because `bob capture` behavior changes, for example:
`feat(capture)!: defer unlisted links whenever a close lists any task`.
