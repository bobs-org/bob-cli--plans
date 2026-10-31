---
tier: tale
title: Reroute unedited gcal Event Task roots when Google's route changes
goal:
  Adding @route to an existing event reroutes its unedited [root::] on the next bob gcal
  sync.
size: medium
decisions:
  reroute_sources:
    ask:
      Should colour, calendar, and default changes reroute unedited rows too, not just
      @route tokens?
    choices:
      google:
        Any non-series-memory source reroutes unedited rows, including token removal
      token_only:
        Only a changed @route token reroutes; other sources affect new rows only
    default: google
    why: Mirrors how time and title already follow Google while unedited
    answer: google
  worked_rows:
    ask: Should rerouting also apply to done or already-worked rows?
    choices:
      any:
        Any row whose [root::] you did not edit is rerouted, fixing a finished meeting
        too
      open_only:
        Only open, unworked rows reroute; logged meetings keep their attribution
    default: any
    why:
      Spans already update regardless of checkbox; a late token should fix attribution
    answer: any
  memory_root_decision:
    ask:
      Add a decisions record that [root::] follows Google until edited, superseding the
      old clause in part?
    memory:
      - decisions
    default: true
    corrects: the vault owns the checkbox, `[root::]`, children, and the row itself
    answer: true
proposed_by: bbugyi200.apollo.br
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-10-31 12:21:20
status: wip
---

# Reroute unedited gcal Event Task roots when Google's route changes

## Problem

Bryan added `@love` to the title and description of the Google Calendar event "Halloween
at Aunt Jenny's!" (2026-10-31). The daily-note row still reads:

```markdown
- [*] #task **1330-1930** Halloween at Aunt Jenny's! [📅](…) [📹](…) [root:: meet]
  ^halloween-aunt-jenny
```

### Root cause (verified)

- The row was created by `bob gcal sync` at 07:44 with `[root:: meet]`. Its journal
  record in `.bob/gcal/athena.jsonl` carries
  `"root":{"source":"default","value":"meet"}`. A sync at 11:11 updated only the time
  span and kept the same root and source. The research report recommending `@love` was
  ingested at 11:54, so the token arrived after the row existed.
- `apply_span_updates` (`src/native/gcal/plan.rs:467-584`) refreshes only the time,
  title, and badge spans. It always keeps `root: journaled.root.clone()`. Its doc
  comment says: "The checkbox, `[root::]`, `[type::]`, and block ID are never
  rewritten."
- `choose_event_root` (`src/native/gcal/root.rs:73-115`) resolves `@love` and strips it
  from the title, so the fresh title equals the existing one. The plan is a no-op:
  nothing is written, and no warning or kept notice is shown. The failure is silent, and
  the stripped title makes it look as though Bob accepted the token.
- `bob gcal list` (`src/native/gcal/list.rs:358-371`, `:429-476`) reports the freshly
  chosen root (`● love`, `@love`), while the vault row keeps `meet`. That contradiction
  adds to the confusion.
- `docs/gcal.md` (§Who owns each part of a row, §Reconciling Google-side changes) and
  decision `bob-gcal-is-read-only-and-proposes-once` document "written once". The
  decision exists so that Bob never fights Bryan's edits. Rerouting a root that Bryan
  has not edited does not fight an edit.

## Goal

An Event Task's `[root::]` follows Google's chosen root while Bryan has not edited it,
the same way the time, title, and badge spans already do. After the fix, the next
`bob gcal sync` turns the Halloween row into `[root:: love] [type:: MEET]` and reports
it as `~ 1330-1930 Halloween at Aunt Jenny's!  ● love @love`. Bryan's own root edits
still always win. When Google later disagrees with one of those edits, sync reports it
once instead of failing silently.

## The rule

For a journaled row where `journaled.row_kind == "task"` and `occurrence.task`, let:

- `current` = `lifecycle::line_root(line)` (`None` when the row has no root),
- `recorded` = `journaled.root`,
- `fresh` = `occurrence.root`, together with its source and whether it came from series
  memory.

Then:

1. **Skip** when `fresh` is `None` (no valid candidate; never delete a root), when
   `fresh` came from series memory (source 1, `learned from your … edit`), or when
   `fresh == current`.
   - Series memory must be excluded. If it were not, editing one occurrence back to its
     original value would look "unedited" (`current == recorded`) and get rewritten to
     the stale learned root on the next run. It would then flip back once the series
     re-learns. Trace `series_edit_teaches_and_reteaches_the_root` in
     `tests/gcal_lifecycle.rs:608` against any design change. Series memory keeps
     applying to new rows only, as documented today.
   - Under `> [!decision] reroute_sources = token_only`, also skip unless the fresh
     source is an `@route` token.
2. **Reroute** when `current == recorded` (both `None` counts as equal, so a row born
   without a root gains one):
   - Under `> [!decision] worked_rows = open_only`, also require
     `lifecycle::row_untouched(...)` (already used for cancels). Otherwise skip
     silently.
   - Rewrite the root with `task_root::set_task_root`.
   - If `occurrence.needs_type` is true and the line has no `[type::]` field, also
     insert `[type:: MEET]` with `task_theme::set_task_theme`. Apply the theme first,
     then the root, exactly as `render::render_task_row` does, so the line ends in
     `… [root:: love] [type:: MEET] ^block-id`. Without this, rerouting to `love`
     (`theme: CHORE` in `love.md`) would silently turn the meeting into a CHORE Pomodoro
     theme. Never rewrite or remove an existing `[type::]`.
   - Record the row in `plan.rows_updated` with `root = fresh` and
     `source = fresh source`. The appended `gcal.row` journal record then holds the new
     root. The next run therefore sees `current == recorded`, and `learn_series_root`
     does not mistake Bob's own rewrite for an edit by Bryan.
3. **Keep and report once** when `current != recorded` (Bryan edited the root) and
   `fresh != recorded` (Google's routing changed too). Push a `KeptNotice` with a new
   flag `kept_root` and a detail like
   `kept your [root:: meet]; Google now routes to love (@love)`, using the existing
   flag-once pattern (journal `state.flags`). When Bryan edited the root but Google did
   not change (`fresh == recorded`), say nothing: that is an ordinary edit.

The root decision is independent of the span three-way merge:

- A kept title, time, or badge edit (the current early `return` when `edited` is
  non-empty) must not block the reroute. When spans are edited, keep them as they are,
  keep `journaled.spans` in the record, and still push the existing `kept_edit` notice.
- A root edit by Bryan must not block span updates. Do not fold root into the
  all-or-nothing `edited` list, because that would stop time changes from reaching every
  manually rerouted meeting.

`learn_series_root` must keep running first, on the pre-rewrite line (it already does,
at `plan.rs:299`).

## Implementation steps

1. **Carry the root kind into planning.**
   - Expose whether the chosen root came from series memory, without string-matching the
     `"learned from"` label. For example, add a small `RootSourceKind` enum (or a
     `learned: bool`) to `root::EventRoot`, set it in `choose_event_root`'s branches,
     and copy it onto `PlannedOccurrence`.
   - Thread it through where `PlannedOccurrence` is built (`src/native/gcal/sync.rs`
     around `:397-481`) and through the test builder `occurrence(...)` in
     `src/native/gcal/plan/tests.rs`.
2. **Apply the rule in `src/native/gcal/plan.rs`.**
   - Restructure `apply_span_updates`, or add an `apply_root_update` step it calls, so
     that both of these happen:
     - one row rebuild and one `rows_updated` entry cover span changes and/or the root
       change;
     - the kept-edit early return no longer drops the root change.
   - Update the function's doc comment and the comment block above `plan_sync`'s no-op
     check if needed.
   - Keep `plan.changed` and the no-op detection correct: a reroute changes contents, so
     it is never a no-op.
3. **Report it.** No new JSON action is needed. Rerouted rows report as `updated`, with
   `root` and `source` already filled from `PlannedRow` (`sync.rs:743-755`, human sigil
   `~`). Kept-root notices flow through `plan.kept` → `warnings` (`sync.rs:321`) and the
   journal `kept` record (`sync.rs:990`) like `kept_edit`. Confirm that `-q` and
   `-f json` still behave, and that `-d` shows the `[root::]` diff.
4. **Tests.**
   - Unit tests (`src/native/gcal/plan/tests.rs`):
     - an unedited `[root:: meet]` row plus a fresh `love` from `@love` is rewritten to
       `[root:: love] [type:: MEET] ^id` when `needs_type`, and without `[type::]` when
       not;
     - an existing `[type:: X]` is kept;
     - an edited root (`current != recorded`) plus a changed fresh root is kept, with
       one `kept_root` notice that is not repeated when the flag is already in the
       journal;
     - a series-memory fresh root never rewrites;
     - `fresh == None` never removes a root;
     - a row whose title edit is kept still gets the reroute, and a row whose root was
       edited still gets a time update;
     - a missing root (`None == None`) gains one;
     - the second run is a no-op (idempotent).
   - Integration tests (`tests/gcal_lifecycle.rs` and/or `tests/gcal_sync.rs`, using the
     existing `FakeGog` / `GcalEnv` helpers):
     - sync an event, add ` @body` to its title in the fake events JSON, and sync again.
       The row now carries `[root:: body]`, the title stays without the token, the
       journal's last `gcal.row` has `root.value == "body"` and
       `root.source == "@body"`, and the JSON row action is `updated` with root `body`;
     - a recurring-series variant confirms that no `gcal.series_root` is learned from
       Bob's own rewrite;
     - `series_edit_teaches_and_reteaches_the_root` still passes unchanged.
   - Add tests for whichever non-default decision branches were chosen.
5. **Docs.** First read the `docs.md` reference memory via `/sase_memory_read`. Then
   update `docs/gcal.md`:
   - §Who owns each part of a row: change the `[root::]` row from "Written once; your
     edits win, are never rewritten, and teach the series" to describe the new
     behaviour. It follows the chosen root (not series memory) until you edit it, and
     adds `[type:: MEET]` when needed. Your edits win, are reported once when Google
     later routes elsewhere, and teach the series.
   - §Reconciling Google-side changes: replace "The checkbox, `[root::]`, `[type::]`,
     and block ID are never rewritten" with the new root bullet, including the
     `kept_root` notice.
   - §Root selection: state that adding, changing, or removing a token (or a colour,
     calendar, or default change under `reroute_sources = google`) reroutes unedited
     existing rows on the next sync. Sync has no periodic trigger, so run
     `bob gcal sync` or wait for the next daily-note sync.
   - Update any `bob gcal sync` CLI help text that claims roots are never rewritten.
     None was found in `src/native/gcal/cli.rs`, but re-check.
6. **Memory** (only per the memory decisions; use `/sase_memory_write`):
   - `memory_root_decision` (when accepted), part 1: create a decisions-web strand
     `decisions:gcal-root-follows-google-until-edited`. The claim: an Event Task's
     `[root::]` follows Google's chosen root (never series memory) while it still
     matches what Bob last wrote; Bryan's edits always win and are reported once on
     conflict; a reroute adds `[type:: MEET]` when needed. Record the alternatives that
     were rejected:
     - write-once roots, which led to the silent Halloween failure;
     - rerouting from series memory, which flip-flops;
     - folding root into the span merge, which would block time updates on manually
       rerouted rows.

     Cite this plan and the `.bob/gcal` journal evidence above. Follow the web's record
     shape: Applies to, Claim, Why, Cost, Reopens when.

   - `memory_root_decision` (when accepted), part 2: mark
     `decisions:bob-gcal-is-read-only-and-proposes-once` with
     `metadata.status: superseded-in-part` and `superseded_by` pointing at the new
     strand. Add a `[[...]]` back-link stating that the `[root::]` ownership clause is
     retired. Do not reword its accepted body.
   - When `memory_root_decision` is accepted, run `sase memory init` to republish.

7. **Release hygiene.** Per `decisions:bob-cli-releases-ride-a-daily-train`, do not
   touch the `Cargo.toml` version or `CHANGELOG.md`. Use a `feat(gcal): …` commit with
   no `!`, because the JSON contract is unchanged.

## Verification

- `cargo fmt --check`, `cargo clippy --all-targets -- -D warnings`, and the full
  `cargo test` suite (at minimum `cargo test gcal`), or the repo's `just`/CI equivalent
  if one exists.
- Manually, with the fake-gog integration harness rather than the live vault, run
  `bob gcal sync -d` on a fixture where the title gains `@body`. The diff should show
  only the `[root::]` (and, when needed, `[type::]`) change on that row.

## Out of scope

- Editing Bryan's vault. The Halloween row is fixed either by the next sync after this
  ships or by hand.
- Making `bob gcal list` show a vault row's edited root beside the chosen root. The
  `kept_root` sync notice covers the conflict case. File a task bead if the list
  mismatch still confuses after this lands.
- HTML-formatted description tokens (`Agenda<br>@bob`), which the research report noted
  as unverified.
