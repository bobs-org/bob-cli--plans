---
tier: epic
title: Google Calendar sync stamp under the daily note's Events heading
goal: 'Every daily-note `## Events` section that `bob gcal sync` reconciles carries
  one quiet, Bob-owned line showing which Google account it was synced with and when;
  the line stays honest without churning the note, cannot be silently taken over by
  another account, and Bob Ledger Tools paints it as a live health light that turns
  amber when sync has stopped.

  '
decisions:
  stamp_refresh:
    ask: How often should an otherwise-unchanged sync rewrite the stamp time?
    choices:
      half_hourly: Once it is 30+ min old; ~2 extra note commits per idle hour, stale
        after 45 min
      every_run: Every 5-min run; exact time, but the daily note is rewritten and
        committed all day
      on_change: Only when rows change; no extra writes, but a quiet day looks stale
        while healthy
    default: half_hourly
    why: Keeps the stamp an honest health signal at about a sixth of every-run churn
    answer: on_change
  account_guard:
    ask: Should sync refuse a day whose stamp names a different Google account than
      this run uses?
    default: true
    why: Otherwise a switch cancels the old account's untouched meetings and relabels
      their rows
    answer: true
  decision_record:
    ask: Add a decisions memory strand recording the stamp's ownership, refresh, and
      account rules?
    memory:
    - decisions:gcal-events-stamp-is-bob-owned
    default: false
    answer: false
phases:
- id: cli-stamp
  title: bob gcal sync writes and maintains the Events stamp
  depends_on: []
  size: medium
  description: 'cli-stamp: add the stamp module (render, recognize, parse, refresh
    rule), run it as the planner''s last step, add the account guard, report the stamp
    in human and JSON output, and update the gcal docs and tests.'
- id: plugin-paint
  title: Bob Ledger Tools paints the stamp as a live health line
  depends_on:
  - cli-stamp
  size: small
  description: 'plugin-paint: in bob-plugins, recognize the stamp in Live Preview
    and Reading view, paint it as a muted caption, mark it stale on today''s note,
    refresh it each minute, then test, document, and run `bob plugins sync`.'
proposed_by: bbugyi200.apollo.bh
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-10-30 18:30:32
status: done
bead_id: bob-cli-es
---

- **PROMPT:** [prompts/202610/gcal_events_sync_stamp.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202610/gcal_events_sync_stamp.md)
- **BEAD:** [bob-cli-es](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-es/README.md)

# Plan: Google Calendar sync stamp under the Events heading

## Outcome

Today's note after a sync, in source:

```markdown
## Events

_↻ synced 14:05 ·
[you@example.com](https://calendar.google.com/calendar/r/day/2026/10/30?authuser=you%40example.com)_

- [x] #task **0800-0830** Product interview [📅](…) [📹](…) [root:: job]
      ^product-interview
- **0915-1000** Toddler class · _maybe_ [📅](…)

## Pomodoros
```

In Obsidian Live Preview with Ledger Tools painting, it renders as one small, muted
italic caption between the heading and the rows:

```text
Events
↻ synced 14:05 · you@example.com                    (small, faint; the email opens this day in that account's Google Calendar)
☑ 0800-0830  Product interview  📅 📹
  0915-1000  Toddler class · maybe  📅
```

When today's stamp is too old (sync has stopped), the same line turns the warning color,
gains a painted ` · stale` suffix, and its tooltip reads
`No Google Calendar sync since 14:05 · run bob gcal doctor`. Nothing about staleness is
written to the note.

The stamp costs exactly one line. The blank gap under the heading that
`3f69021a fix(gcal): keep a blank line under Events` introduced stays where it is, and
the stamp is the section's first line.

## Design decisions (made by this plan)

- **Placement: the first line after the heading gap.** A paragraph may sit directly
  above a list. CommonMark lets a bullet list interrupt a paragraph, so the stamp needs
  no blank line below it. That gives a cost of +1 line.
  - Rejected: a heading suffix (`## Events · …`). Four exact-match heading parsers would
    break: `event_task::is_events_heading`, the Ledger Tools section finder and
    reading-view heading check, and the task-status-cycler clock. `#Events` links would
    also change on every sync.
  - Rejected: a footer below the rows. A paragraph directly after a list item is a lazy
    continuation of that item, so a footer needs a blank line before it (+2 lines).
  - Rejected: a callout or blockquote (2+ lines of heavy chrome).
- **Plain Markdown that looks right without the plugin.** The stamp is an emphasis line
  holding a text glyph, a 24-hour time, Bob's `·` separator, and a link.
  - It is not a list item, so every row scanner already skips it: `parse_event_span`,
    `gcalEventsScanMarkdown`, the Tasks plugin, and Dataview.
  - It holds no `::`, so Dataview does not read it as a page-level field.
  - It holds no `#`, so it never mints a tag.
  - It holds no Tasks date signifier: no 📅, 📆, 🗓, ✅, ⏳, or 🛫.
  - Its time is `HH:MM`, never `**HHMM-HHMM**`, so it can never look like an Event span.
  - Rejected: HTML `<small>`, which shows raw tags in source mode and nvim.
  - Rejected: Dataview inline fields, which become fields of the daily note.
  - Rejected: a bullet, which looks like an event row.
  - Rejected: the 🔄 emoji, a loud blue tile on macOS. `↻` (U+21BB) takes the
    surrounding text color.
- **The account is a deep link.** It points to
  `https://calendar.google.com/calendar/r/day/<YYYY>/<M>/<D>?authuser=<account>`, which
  opens the note's day in the right account. The link also stops Obsidian from
  auto-linking the bare email as `mailto:`.
- **Absolute time only.** The note says `synced 14:05`. Relative text ("3 min ago")
  would be wrong the moment it was written, so the live judgement belongs to the
  painter.
- **Bob owns the line.** It works like the heading gap: sync re-renders it, normalizes
  edits, and puts back a deleted line. It is chrome, not content, so tombstones and
  "propose once" do not apply.
- **No new config key or CLI flag.**

## Shared stamp contract (both phases)

### Grammar

```text
*↻ synced <when> · [<account>](<day-url>)*
```

- `<when>` is `HH:MM` (24-hour, Bob's configured timezone) when the run happened on the
  note's own day. Otherwise it is `Ddd YYYY-MM-DD HH:MM` (chrono `%a %Y-%m-%d %H:%M`),
  for example `synced Thu 2026-10-29 21:40` on Friday's note after
  `bob gcal sync -D tomorrow`. This matches the `Moved to Fri 2026-10-31` style already
  used in cancel reasons.
- `<account>` is the resolved `snapshot.account`, displayed as resolved. Escape `\`,
  `*`, `_`, `[`, `]`, `` ` ``, `<`, and `>` with a backslash; the parser unescapes them.
- `<day-url>` is
  `https://calendar.google.com/calendar/r/day/<YYYY>/<M>/<D>?authuser=<pct>`. Month and
  day are unpadded. The account is percent-encoded except for unreserved characters, so
  `@` becomes `%40`. Reuse `render::percent_encode_query`, made `pub(crate)`.

### Recognition

A stamp line is a line with all of these properties:

- It is inside the `## Events` section range.
- It is unfenced.
- It is not a list item or heading.
- After trimming leading whitespace and leading `*` and `_` characters, its text starts
  with `↻`.

Bob owns every recognized line.

### Parsing

- Parsing yields the account (the unescaped link text) and the sync time: the
  `NaiveDateTime` for an `HH:MM` stamp on the note's day, or the explicit date-time.
- A recognized line that fails to parse is a malformed stamp.

### Canonical slot

- The stamp sits immediately after the blank line(s) under the heading, before any other
  section line.
- When the next line is non-blank and not a list item, put one blank line between them
  so the two paragraphs do not merge.

### Presence

A stamp exists exactly when the day's note has an `## Events` section after planning.
Sync never creates a section just to hold a stamp. A day with no included events and no
section stays untouched, as it is today. A hand-written section that sync reconciles
gets a stamp: it truthfully says the calendar was checked.

### Refresh rule

Every run first plans everything else: rows, lifecycle, the gap, and MEET links. Then:

1. If that work changed the note text, write a fresh stamp with this run's time (`now`
   truncated to the minute).
2. Otherwise leave the existing stamp byte-identical unless it is due. A stamp is due
   when any of these holds:
   - It is missing or malformed.
   - It is not byte-identical to its canonical rendering, as after a user edit.
   - It is not in the canonical slot.
   - It is duplicated.
   - It is later than `now`.
   - It has aged out under the decision below.

   A due stamp is rewritten with this run's time.

> [!decision] stamp_refresh = half_hourly A stamp is aged out when
> `now − synced ≥ 30 min`. At the LaunchAgent's 5-minute cadence, a healthy today's
> stamp is never more than about 35 minutes old. Docs and the painter call a stamp on
> today's note **stale** after 45 minutes.

> [!decision] stamp_refresh = every_run A stamp is aged out when `now > synced` (minute
> precision), so every run at a new minute rewrites it. Docs and the painter call
> today's stamp stale after 15 minutes. Docs must say plainly that the daily note is
> rewritten, and so committed by vault-sync, about every 5 minutes.

> [!decision] stamp_refresh = on_change Age never makes a stamp due; only the structural
> reasons in step 2 do. The stamp means "last change". Docs say so, and the painter
> never marks a stamp stale.

A journal-only run (tombstones, flags) does not count as a note change. Dry runs apply
the same rule, so the diff shows exactly what a real run would write.

### Account guard

> [!decision] account_guard After reading the note and before loading the journal or
> planning, sync parses the first recognized stamp. When it parses and names a different
> account than this run's resolved account (compared case-insensitively), sync writes
> nothing and fails with `GcalError::setup("account_changed", …)` (exit 2). The failure
> is recorded through the existing `record_last_error`, so `doctor`'s last-sync check
> shows it. Dry runs fail the same way. The retry path re-reads the note, so it checks
> the guard again.
>
> - Message:
>   `2026/20261030.md was synced with a@example.com; this run uses b@example.com`
> - Hint:
>   `delete the ↻ line under ## Events to hand this day to b@example.com, or set gcal.account: a@example.com`
>
> A malformed stamp has no account, so the guard does not apply and the stamp is
> rewritten. The guard exists because an account switch makes every journaled `primary`
> occurrence 404 under the new account, which cancels untouched meetings as "Deleted in
> Google Calendar". It also stops a relabelled stamp from claiming rows the old account
> created.

> [!decision] account_guard = no No guard. A different account simply re-renders the
> stamp, which is due because its text changed. Document in Known limits that switching
> accounts mid-day relabels the section.

## Phase cli-stamp: bob gcal sync writes and maintains the Events stamp

Work in bob-cli. Read the reference memory `docs.md` before editing docs. Do not touch
`Cargo.toml`'s version or `CHANGELOG.md`; releases ride the daily train. Use a
`feat(gcal): …` commit.

### Code

1. **New `src/native/gcal/stamp.rs`** (register it in `gcal/mod.rs`). It holds:
   - `Stamp { account, synced_at: NaiveDateTime }`
   - `render(account, synced_at, note_day) -> String`
   - `recognize(lines, events_range) -> Vec<usize>`, which respects fences the same way
     `event_task::events_section_range` does
   - `parse(line, note_day) -> Option<Stamp>`
   - `is_due(...)`, which implements the refresh rule
   - the heartbeat and stale-threshold constants for the chosen `stamp_refresh`, with
     doc comments

   Unit-test round trips. Cover a same-day stamp, another-day stamp, escaped accounts
   (for example `first_last+cal@x.com`), malformed lines, and lines that must not be
   recognized: list rows, fenced lines, and a `↻` outside Events.

2. **Planner (`plan.rs`).**
   - Add `account: &str` and `now: NaiveDateTime` to `SyncPlanInput`.
   - Add `stamp: Option<Stamp>` (the stamp the note carries after the plan) and
     `stamp_refreshed: bool` to `SyncPlan`.
   - Run the stamp step **last**, after `plan_meet_links`. Compute "other work changed
     the note" as `working.finish() != input.contents` before the step.
   - The step removes every recognized stamp line and writes the single canonical line
     at the slot, or keeps the existing one when it is not due.
   - When the step inserts or removes lines above the MEET entry, shift
     `pomodoro_entry_line` so the 1-based report stays correct. A hand-written Events
     section may sit below Pomodoros, so compare indexes rather than assuming an order.
   - Make `events_top_index` skip a recognized stamp so a new earliest row lands below
     it.
   - The no-op branch that clears plan fields must keep `stamp` (the existing parsed
     stamp) for reporting.
   - `plan.changed` needs no new logic: a stamp rewrite changes the contents.
3. **Sync (`sync.rs`).**
   - Pass `&snapshot.account` and `now` into both `plan_sync` calls. The retry re-plan
     reuses the same `now`.
   - Run the account guard (see the decision above) right after reading the note, and
     again after the retry's re-read.
   - Add `stamp` to `SyncReport`.
4. **Output.**
   - JSON (additive):
     `"stamp": {"account": "you@example.com", "synced_at": "2026-10-30T14:05", "refreshed": true}`.
     `synced_at` is local wall time to the minute, the same instant the note shows.
     `refreshed` is true when this run wrote the stamp, or would write it under `-d`.
     The value is `null` when the note has no Events section and for `waiting`.
   - Do not add a `status` value. Ledger Tools' decoder accepts only `written`,
     `unchanged`, and `waiting`, so a stamp-only write reports `written` with empty
     `rows`.
   - Human: when a stamp exists, end the report with a dim echo
     `↻ synced 14:05 · you@example.com` (no URL).
   - A written run with no row or link changes summarizes as
     `✓ Events unchanged · stamp refreshed`.
   - `-q` still prints nothing.

### Tests

- **`src/native/gcal/plan/tests.rs`** (extend the `run` helper with an account and
  `now`). Cover:
  - A new section reads heading, gap, stamp, rows.
  - A rerun at the same `now` is byte-identical.
  - The heartbeat boundary for the chosen decision.
  - A future stamp is rewritten.
  - Edited, misplaced, and duplicated stamps are normalized to one canonical line.
  - A new earliest row lands below the stamp.
  - A stamp above a non-list paragraph gets a blank line between them.
  - `pomodoro_entry_line` stays correct when the stamp is inserted in the same plan as a
    new MEET link.
  - A hand-written section gains a stamp.
  - A note with no section and no events gains nothing.
- **`tests/gcal_sync.rs`** (update fixtures that assert the exact Events head, such as
  `first_sync_creates_events_meet_and_links` and the heading-gap test).
  - Use `run_bob` with explicit `BOB_NOW` for a sync at 12:00, then 12:20 (`unchanged`,
    byte-identical), then the heartbeat time (`written`, only the stamp line differs,
    journal line count unchanged).
  - Add JSON `stamp` assertions.
  - Add the human echo line.
  - The dry-run diff shows the `+*↻ synced` line.
  - Run `-D tomorrow` from a 21:40 `BOB_NOW` and expect the dated form.
  - Guard on: seed the note with an `other@example.com` stamp and expect exit 2, kind
    `account_changed` in `-f json`, an unchanged note, and an untouched journal.
- **`tests/gcal_lifecycle.rs`:** fix any exact-text Events assertions the stamp shifts.

### Docs

- **`docs/gcal.md`** § Sync:
  - Add a short "Events stamp" subsection: the source example, the grammar in words, the
    refresh rule, what stale means on today's note (with the threshold from the
    decision), and that the line is Bob's (edits are replaced, a deleted line returns on
    the next sync).
  - Add a stamp row to the "Who owns each part of a row" table.
  - Reword "A byte-identical plan writes nothing": an unchanged calendar can still write
    the stamp under the refresh rule.
  - Add `stamp` to the sync JSON field lists.
  - Exit status: add `account_changed` to code 2 when the guard is on.
- **`docs/reference/vault-format.md`:** add the stamp line to the Events example and its
  one-sentence description.
- **`docs/reference/json-outputs.md`:** add `stamp` to the `bob gcal sync` row.
- Run `just docs-check`, then `just all` (fmt, lint, test).

> [!decision] decision_record Add the decisions strand
> `decisions:gcal-events-stamp-is-bob-owned`, modeled on the existing gcal strands'
> frontmatter and body shape (Applies to / Claim / Why / Cost / Reopens when /
> Evidence).
>
> - **Claim:** Bob owns one stamp line at the top of each reconciled Events section. It
>   is rewritten whenever the note is written, plus the chosen refresh rule, and the
>   account guard as decided.
> - **Why:** the rejected placements and formats above, and churn against vault-sync's
>   15-second commits.
> - **Cost:** about two idle commits an hour, or as decided.
> - **Reopens when:** Bob can render live state without writing the note.
> - **Evidence:** this plan and `docs/gcal.md`.
>
> Follow the `/sase_memory_write` edit-and-republish steps, `sase memory init`, then
> `just docs-gen` so `docs/develop/decisions.md` regenerates.

## Phase plugin-paint: Bob Ledger Tools paints the stamp as a live health line

Open the linked `bob-plugins` repo with `/sase_repo`, and follow its `AGENTS.md`: edit
`src/` fragments, `npm run build`, never hand-edit `main.js`, and keep each fragment at
1000 lines or fewer. Read the stamp implementation from the cli-stamp phase in bob-cli
(`src/native/gcal/stamp.rs` and its tests) and match the grammar exactly.

1. **Scan.** Extend `gcalEventsScanMarkdown` in
   `plugins/bob-ledger-tools/src/291-plugin-gcal-events.js`, or add a sibling scanner,
   to return `stamp: {line, glyphStart, synced: {date|null, minutes}} | null`. Use the
   recognition rule above: inside the Events section, unfenced, not a list line, and
   starting with `↻` after `*`/`_`.
2. **Freshness.** Add `gcalEventsStampState(stamp, {isToday, now})`.
   - It returns `"stale"` only on today's note, when `now − synced` exceeds the
     threshold from `stamp_refresh` (45 min for half_hourly, 15 min for every_run, never
     for on_change).
   - Otherwise it returns `"fresh"`. A future stamp is fresh.
   - Dated stamps compare the full date-time.
3. **Live Preview.** In `buildCalendarEventDecorations`, stop returning
   `Decoration.none` when there are no rows but a stamp exists. Add
   `Decoration.line({class: "bob-calendar-sync-stamp", attributes: {"data-state": state, title}})`.
   - Fresh title: `Google Calendar · synced 14:05`.
   - Stale title: `No Google Calendar sync since 14:05 · run bob gcal doctor`.
   - The existing 60-second refresh effect re-evaluates the state.
4. **Reading view.** In `renderCalendarEventsIn`, handle a `p` whose text starts with
   `↻ synced`.
   - Confirm that it belongs to the Events section with `ctx.getSectionInfo(el)` and the
     source lines when available. Otherwise fall back to the same previous-heading walk
     lists use.
   - Add the class, `data-state`, and title, and register it with the minute refresh the
     way time pills are.
5. **CSS (`styles.css`, gated by `.bob-calendar-events`).**
   - The stamp uses `var(--text-faint)` and `var(--font-ui-smaller)`, keeps the italic,
     and gets a little top spacing.
   - Its link inherits the muted color, with no accent, and is underlined on hover.
   - `[data-state="stale"]` uses `var(--text-warning)` and adds a `::after` with
     `" · stale"`, for both `.cm-line` and the reading-view `p`.
   - Check that it reads well in both light and dark themes.
   - The existing "Toggle calendar event painting" command covers it.
6. **Tests.** Extend `scripts/test-ledger-tools-gcal-events.cjs` with stamp vectors:
   - same-day and dated stamps
   - fresh, stale, and future states
   - a stamp in a section with no rows
   - a `↻` line outside Events, list rows, and fenced lines, none of which count
   - stamp detection leaving the existing row scan unchanged
7. **Docs and deploy.**
   - Add two sentences to `docs/bob-ledger-tools.md` § Calendar Events about the stamp
     painting and what stale means.
   - Run `npm run build`, `npm test`, and `npm run docs:check`.
   - Then run `bob plugins sync`, as the repo's `AGENTS.md` requires.
