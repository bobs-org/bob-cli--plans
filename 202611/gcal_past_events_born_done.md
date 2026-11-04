---
tier: tale
title: Born Done — bob gcal sync completes Event Tasks for meetings that already ended
goal:
  A meeting that bob gcal sync first writes after it ended is born [x] with its
  completion date and a struck 🍅 link, so it never parks in Pending.
size: medium
decisions:
  new_decision_record:
    ask:
      Add decisions:ended-synced-meetings-are-born-done recording the new Past-phase
      rule and why it replaced [/]?
    memory:
      - decisions:ended-synced-meetings-are-born-done
    default: true
    requested:
      Can you help me make it so the `bob gcal sync` command creates event tasks with a
      closed status when the event is in the past? We already seem to create the closed
      past pomodoro properly, but the event task is created with a wip/pending status.
    answer: true
  supersede_link_by_phase:
    ask:
      Mark decisions:synced-meetings-link-by-phase superseded in part by the new record,
      with a back-link?
    memory:
      - decisions:synced-meetings-link-by-phase
    default: true
    corrects: A Past row is worked from birth (`[/]` and a closed-entry link)
    answer: true
proposed_by: bbugyi200.athena.17z
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-04 11:24:29
status: wip
---

# Born Done: `bob gcal sync` completes Event Tasks for meetings that already ended

## Problem

When `bob gcal sync` first writes an Event Task after its meeting has ended (the
**Past** phase), it already files the row under a closed
`- [x] (**SSSS-EEEE** [t:: Nm]) — MEET` (or `[off:: Nm]`) entry with a `🍅 [[#^id]]`
link. However, the row itself is born `[/]` (In Progress / Pending / WIP). Bryan's
2026-11-04 note shows the problem: the 08:15–08:30 `^swings` row was synced at 11:06.
Its closed off-clock MEET was correct, but the `[/]` row sits in the Pending lane
(already `PENDING 22/8`, over cap) until he completes it by hand.

Root cause: `status_for_phase` in `src/native/gcal/plan.rs` maps `LinkPhase::Past` to
`'/'`. That was a deliberate choice in `decisions:synced-meetings-link-by-phase`, which
rejected `[x]` ("Sync cannot know Bryan attended"). Bryan now asks for the closed
status. This matches `decisions:ended-off-clock-events-settle`, where reconcile
completes ended off-clock events with `[x]` + completion date because "leaving it `[/]`
would keep a lunch in Pending".

## Target behavior

Only the **Past** phase changes. It applies to every Event Task row this sync run
creates, whether it is a new row (`plan.rows_added`) or a Maybe context row upgraded in
place (`upgrade_context_row` → `plan.rows_upgraded`), on any day (today, or
`-D yesterday`):

| Phase at sync time        | Pomodoro                      | Link line                             | Row checkbox                                       |
| ------------------------- | ----------------------------- | ------------------------------------- | -------------------------------------------------- |
| Past (buffered end ≤ now) | closed MEET entry (unchanged) | `🍅 ~~[[#^id]]~~` (was `🍅 [[#^id]]`) | `[x]` plus `[completion:: YYYY-MM-DD]` (was `[/]`) |
| Current                   | unchanged                     | unchanged                             | `[*]` (unchanged)                                  |
| Future                    | unchanged                     | unchanged                             | `[*]` (unchanged)                                  |

Exact Past row shape (root present / root-less):

```markdown
- [x] #task **0800-0830** Product interview [📅](…) [root:: job] [type:: MEET]
      [completion:: 2026-10-30] ^product-interview
- [x] #task **0815-0830** Swings! [📅](…) #off [completion:: 2026-11-04] ^swings
```

Decisions this plan makes (the implementer should not revisit them):

- **Completion date = the note's day** (`SyncPlanInput.day_string` / `note_day(input)`),
  not the wall-clock day of the sync run. The meeting happened on that day.
  `-D yesterday` therefore counts as Done yesterday, matching reconcile settle, which
  stamps the ledger day.
- **Completion field placement = the existing helper**
  `crate::native::task_complete::tree::add_or_replace_completion_field` (two spaces,
  then `[completion:: DATE]`, before the trailing `^id`). Do not hand-format it. The row
  always carries `#task`, so always add the field. This is the same field the settle
  path and a timed close write.
- **The Past link is struck at birth** (`🍅 ~~[[#^id]]~~`). This is the canonical
  "retired with recorded participation" grammar that `bob task reconcile` would rewrite
  the plain link to anyway, since the target is done and sits under a completed Pomodoro
  (normalized in place, never relocated). It is also exactly what reconcile settle
  writes. Sync's write is therefore already in the hooks' steady state, and reconcile
  makes no follow-up edit. `link_order::sort_event_task_links` already recognizes struck
  links (`decisions:sync-sorts-event-task-links`), and Event Link Times already pills
  struck children (EL2). `pomodoro_link_present` is a substring check and still matches.
- Ownership is unchanged. The vault owns the checkbox after birth. A later Google
  decline, delete, or move leaves a done row alone and reports it once, exactly as for
  the old `[/]` row (`row_untouched` is false for both).
- Rows created _before_ their meeting ended (Current/Future, `[*]`) are unchanged. A
  timed close (`ended_inside`) or reconcile settle (off-clock) still completes them.
  Sync never changes an existing row's checkbox.

## Implementation

### 1. Planner (`src/native/gcal/plan.rs`)

- `status_for_phase`: `Some(LinkPhase::Past) => 'x'`. Keep Current/Future `'*'` and
  context `' '`.
- `render_occurrence` (and the `render::TaskRowInput` / `render_task_row` path it
  calls): Past task rows must carry `[completion:: <note day>]`. Thread the note day in.
  For example, give `render_occurrence` a `completion_day: &str` (or an `Option<&str>`
  set only for Past task rows) and apply `add_or_replace_completion_field` to the
  rendered row. Alternatively, add a `completion: Option<&str>` field to `TaskRowInput`
  that `render_task_row` applies last. Choose whichever keeps all three
  `render_occurrence` call sites simple:
  - L344: the context-row `text_match`. Status is irrelevant there, so any value works,
    but it must still render context rows byte-identically.
  - L482: new rows.
  - L1146: the Maybe upgrade. Context rows never get the field.
- `append_links`: when `tomato` is true (only past clusters pass it), write
  `🍅 ~~[[#^{id}]]~~` instead of `🍅 [[#^{id}]]`. Open-entry links stay plain
  `[[#^id]]`. Update the doc comment on `plan_meet_pomodoros` / the `LinkPhase` enum if
  it describes the Past link or checkbox.

### 2. Row span parser (`src/native/gcal/lifecycle.rs`): required for correctness

`split_suffix` currently starts the Bryan-owned suffix at ` [root::`, else ` [type::`,
else the trailing ` ^id`. On a **root-less** row (no `[root::]`, no `[type::]`; roots
may be null, see `docs/gcal.md` § List), a `[completion:: …]` before `^id` therefore
falls into `middle`. With a `📅` badge it lands in `badges`; without one it lands in the
title. Either way, the next sync sees a span hash that differs from the journal. It then
reports "kept your edit to badges" once, and `apply_spans` stops applying Google's
time/title/badge updates to that row. This bug already exists today for root-less rows
completed by a timed close, settle, or by hand. This plan would make it routine.

Fix: treat ` [completion::` as a third vault-owned suffix marker. Start the suffix at
the **earliest** marker position among `[root::`, `[type::`, and `[completion::`, and
begin it at the whitespace run that precedes the marker. That way the head (`middle`)
carries no trailing whitespace, while the suffix keeps all the whitespace, so
`RowSpans::rebuild()` round-trips byte for byte. Keep the existing root/type behavior
byte-identical for rows that have no completion field. Check `render::set_row_metadata`
with a suffix that starts with whitespace and holds only `  [completion:: …] ^id`:
inserting a root must yield ` [root:: X]  [completion:: …] ^id`. Titles cannot contain
`[completion::` because sync escapes `::` in titles.

### 3. Tests

Update the assertions that pin today's Past behavior. Leave hand-written `[/]` fixture
rows alone; only assertions about what sync _writes_ change:

- `src/native/gcal/plan/tests.rs`: the past/future split test (~L880–910: `standup` →
  `- [x] #task`, contains `[completion:: <day>]`, link `\t- 🍅 ~~[[#^standup]]~~`), the
  cluster test (~L950: struck links), and the ~L1285–1293 case.
- `tests/gcal_sync.rs`: ~L110–150 (interview row `[x]`, completion date, still
  `ends_with("^product-interview")`, struck closed-MEET link), ~L480 (`-D yesterday`:
  `[x]` rows with _yesterday's_ completion date), ~L805–816, ~L1862–1878, ~L1983 (the
  off-clock `swings` case: `[x]` under the `[off:: Nm]` MEET).
- `tests/gcal_lifecycle.rs`: ~L366, ~L1053–1061, ~L1126, ~L1220, ~L1303. These are
  past-born rows asserted `[/]`; they become `[x]`, and "worked row stays" semantics are
  unchanged. Their behavior (left alone and reported once, `kept_off_clock`) must still
  hold.
- Grep for any other `🍅 [[#^` / `"- [/]` expectations on sync output
  (`rg -n '🍅 \[\[#\^|\[/\]' tests/gcal_*.rs src/native/gcal`).

New tests:

1. **Lifecycle unit (`lifecycle.rs`)**: for a root-less completed row
   (`… [📅](url) #off  [completion:: 2026-11-04] ^swings`) and a rooted one,
   `split_row_spans(...).hashes()` equals the hashes of the same row without the
   completion field, and `rebuild()` returns the input unchanged.
2. **Planner unit**: a Past root-less occurrence renders
   `- [x] #task … [completion:: <day_string>] ^id` with a struck 🍅 link. A Past
   Maybe-upgrade renders the same. Current/Future rows have no completion field. Context
   rows are unchanged.
3. **Integration (`tests/gcal_sync.rs`)**: sync a past root-less event, then rerun at a
   later minute. The second run reports no `kept_edit` and leaves the row untouched
   apart from the stamp. Then change the event title in the fake Google feed. The next
   sync updates the title in place on the `[x]` row and keeps `[completion::]` and
   `^id`.
4. **Steady state**: if the integration harness can run `bob task reconcile` on the
   synced vault (look for an existing helper; skip this test if none exists), assert
   that reconcile leaves the synced daily note's `## Events` rows and closed MEET links
   byte-identical.

### 4. Docs (behavior change: contract/reference pages only)

- `docs/gcal.md` § Sync: Past row of the phase table → link `🍅 ~~[[#^id]]~~`, checkbox
  `[x]` with `[completion:: YYYY-MM-DD]` (the note's day). Change the "A past day
  (`-D yesterday`) gains closed MEET entries and `[/]` rows" sentence to `[x]` rows
  completed on that day.
- `docs/gcal.md` § Who owns each part of a row ("Untouched" paragraph): replace "A Past
  row is worked from birth (`[/]` and a closed-entry link) … `[/]` rows count in the
  Pending lane and show in `list`'s `past_open` footer until they are completed or
  cancelled" with: a Past row is born done (`[x]`, completion date, struck 🍅 link under
  its closed MEET), so a later Google decline, delete, or move leaves it alone and
  reports it once. Sync cannot learn attendance, so uncheck or cancel a skipped meeting
  by hand.
- `docs/gcal.md` § Known limits: add one bullet. A meeting first synced after it ended
  is born done and counted (Done, plus its closed MEET's minutes); sync cannot tell
  whether you attended.
- `docs/gcal.md` § Events stamp example: optionally add the `[completion:: …]` field to
  the `[x]` example row so it matches real output.
- `docs/reference/vault-format.md` § Events section and Event Tasks: change the
  example's `- [/] #task **0800-0830** Product interview …` to the `[x]` +
  `[completion:: …]` shape and `🍅 [[#^product-interview]]` to
  `🍅 ~~[[#^product-interview]]~~`. Change "a row first synced after its event ended
  shows `[/]` under a closed MEET" to `[x]` (completed on the note's day).
- Run `just docs-check`.

### 5. Memory (only under the accepted decisions above)

> [!decision] new_decision_record

- Add `sase/memory/decisions/ended-synced-meetings-are-born-done.md`, modeled on
  `sync-sorts-event-task-links.md` (frontmatter: `keyword`, `aliases`, a one-sentence
  `summary`, `metadata.status: accepted`, `decided: 2026-11-04`, `title`). Body
  sections:
  - **Applies to**: bob-cli and the vault.
  - **Claim**: the Past row rule above (`[x]`, note-day completion, struck 🍅 link, new
    and upgraded rows, any day).
  - **Why**: Bryan's 2026-11-04 request. The `2026/20261104.md` `^swings` row sat `[/]`
    in an over-cap Pending lane even though its closed MEET was right. It also matches
    reconcile settle's completion of ended off-clock events.
  - **Rejected alternatives**:
    - keep `[/]`: it parks finished meetings in Pending;
    - complete with the sync day: it miscounts `-D yesterday`;
    - leave the 🍅 link plain: reconcile rewrites it a moment later.
  - **Cost**: a skipped meeting is born done and counted; uncheck or cancel it by hand.
  - **Reopens when**: sync can learn attendance, or skipped meetings should stay open
    for review.
  - **Evidence**: this plan's archived `plan:` ref (confirm the exact ref),
    `docs/gcal.md` § Sync, `tests/gcal_sync.rs`.
  - Back-links: `[[decisions:synced-meetings-link-by-phase]]`,
    `[[decisions:ended-off-clock-events-settle]]`, `[[glossary:event-task]]`.

> [!decision] supersede_link_by_phase

- In `sase/memory/decisions/synced-meetings-link-by-phase.md`, append
  `decisions:ended-synced-meetings-are-born-done` to `metadata.superseded_by` (status
  stays `superseded-in-part`). Add one closing "Superseded in part:" paragraph like the
  existing ones: Past rows are now born `[x]` with a completion date and a struck 🍅
  link, and the `[x]` rejection is retired. See
  `[[decisions:ended-synced-meetings-are-born-done]]`. Do not reword the accepted body.

After either accepted change, run `sase memory init` to regenerate `AGENTS.md`, the
provider shims, and the decisions roster. If a memory decision was declined, skip that
note, and follow `/sase_memory_write` § Declined Changes.

## Out of scope

- Migrating rows already synced as `[/]`, including today's `^swings` row. Sync never
  changes a checkbox after birth; Bryan completes those by hand.
- Current/Future rows, timed-close classification, reconcile settle, and every
  bob-plugins / Bob Mac Capture surface. None of them changes; the JSON and human `sync`
  output keep their schema.
- `CHANGELOG.md` and the `Cargo.toml` version (the release train owns them). Commit as
  `feat(gcal): …` (no `!`: no CLI or JSON contract breaks).

## Verification

```bash
cargo test gcal                      # planner + lifecycle unit tests
cargo test --test gcal_sync --test gcal_lifecycle --test gcal_list
just lint
just docs-check
sase memory init                     # only if the memory decisions are accepted
```

Spot-check a dry run in a scratch vault, or read the integration test's note. A past
event shows `- [x] #task **HHMM-HHMM** … [completion:: <note day>] ^id` in `## Events`
and `\t- 🍅 ~~[[#^id]]~~` under its closed MEET, and an immediate rerun writes nothing
beyond the per-minute stamp.
