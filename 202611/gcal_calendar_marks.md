---
tier: epic
title: Multi-calendar gcal sync with calendar marks, starting with Birthdays
goal: '`bob gcal sync` syncs several Google Calendars from one account, chosen by
  id. Every row from a non-primary calendar starts its title with that calendar''s
  configured mark, so `- **all day** 🎂 Ava''s Birthday [📅](…)` lands in 2026/20261114.md
  and the source calendar is clear in the note, every CLI surface, and JSON. Removing
  a calendar from config freezes its old rows instead of cancelling them.

  '
decisions:
  birthday_mark:
    ask: Which mark should the Birthdays calendar wear?
    choices:
      cake: 🎂 reads as a birthday at a glance and echoes Google's own birthday art
      gift: 🎁 hints at a gift to buy, but reads less plainly as a birthday
      party: 🥳 is festive, but reads as any celebration, not only a birthday
    default: cake
    why: 🎂 is the most legible birthday glyph at the start of a row
    answer: cake
  mark_decision_record:
    ask: Add a decisions record saying every non-primary calendar's rows wear its
      configured mark?
    memory:
    - decisions:non-primary-calendars-wear-a-mark
    default: true
    requested: I agree with all of the requirements recommended in that research file
      (but keep in mind that you are responsible for planning and splitting up this
      work appropriately).
    answer: false
phases:
- id: calendar-config
  title: Calendar marks in config, a resolved calendar set, and per-calendar fake
    gog
  depends_on: []
  size: medium
  description: 'calendar-config: add the required, validated `mark` key to `gcal.calendars`,
    check the calendar set against the account''s calendar list before any fetch (missing
    calendars, primary/email aliases, unmarked non-primary), carry a resolved calendar
    identity on every snapshot event, and teach the test fake gog to serve events
    and lookups per calendar.'
- id: marked-rows
  title: Marked titles in the daily note
  depends_on:
  - calendar-config
  size: medium
  description: 'marked-rows: render the mark at the front of the Google-owned title
    span through one shared helper used for rendering and journal span hashes. New
    rows are born marked; untouched rows follow mark changes in place; hand-edited
    titles are kept and reported once; block ids and probes stay on the bare title.'
- id: deconfigured-freeze
  title: Freeze rows from calendars that left the config
  depends_on:
  - calendar-config
  size: small
  description: 'deconfigured-freeze: journaled rows whose calendar is no longer configured
    are left alone, reported once, and never looked up, which removes the false Moved
    and Deleted verdicts; re-adding the calendar resumes the same identities.'
- id: calendar-surfaces
  title: Calendar identity on every CLI surface and in JSON
  depends_on:
  - marked-rows
  size: medium
  description: 'calendar-surfaces: show marks and the sync set in `bob gcal calendars`,
    `list`, `sync`, `doctor`, and the ack and note human lines; add additive `mark`
    and `calendar` JSON fields; update the gcal and JSON reference docs.'
- id: enable-birthdays
  title: Decision record, setup guide, and the Birthdays config row
  depends_on:
  - calendar-surfaces
  - deconfigured-freeze
  size: small
  description: 'enable-birthdays: write the calendar-mark decision record, add the
    add-another-calendar step to the setup guide, enable the Birthdays calendar by
    id with its mark in the chezmoi bob config, and record follow-ups.'
- id: live-acceptance
  title: Install, apply, and prove Ava's birthday lands
  depends_on:
  - enable-birthdays
  size: small
  description: 'live-acceptance: install the new bob and apply the chezmoi config
    on the hosts that hold the Google grant, then run doctor, calendars, list, a dry
    run, the real `-D yesterday` sync, and an idempotent rerun. Ava''s birthday must
    land as exactly one marked all-day bullet.'
proposed_by: bbugyi200.apollo.research.4q.linker.w1
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-15 12:24:18
status: wip
bead_id: bob-cli-lr
---

- **PROMPT:** [prompts/202611/gcal_calendar_marks.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/gcal_calendar_marks.md)
- **BEAD:** [bob-cli-lr](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-lr/README.md)

# Plan: Multi-calendar gcal sync with calendar marks, starting with Birthdays

## Context

Bryan keeps birthdays in a separate Google Calendar named **Birthdays** under the same
account as his primary calendar. He wants `bob gcal sync` to sync it too, and he wants
it to always be clear which calendar a row came from. The consolidated research report
`research:202611/gcal_multi_calendar_sync/gcal_multi_calendar_sync.md` (read it with
`sase artifact read`) recommends a design that Bryan accepted in full. This plan
implements it.

Facts verified while planning:

- **Multi-calendar fetch already exists.** `gcal.calendars` accepts several entries
  (`src/native/config/gcal.rs::parse_calendars`),
  `snapshot.rs::FetchContext::fetch_days` fetches each configured calendar, and journal
  keys are `calendar_id + \u{1f} + event_id`, using the _configured_ spelling
  (`primary`).
- **No calendar identity is visible.** `SnapshotEvent.calendar_name` is computed but is
  `#[allow(dead_code)]`. No vault row, list cell, or sync JSON row names its calendar,
  and `ReportRow` in `sync.rs` has no calendar field.
- **The event.** The calendar is `Birthdays`, id
  `kjs04hafl3btrilujllivid3s8@group.calendar.google.com`, role owner, not primary. The
  event is `Ava's Birthday`, all-day 2026-11-14 → 2026-11-15 (exclusive end), annual,
  `eventType: default`, transparent, with no attendees. All-day events already classify
  as **context rows**: no checkbox, no MEET, no root, and ack skips them. So **no
  classifier change** is needed.
- **The second "Birthdays" entry** in Google's sidebar is the contacts-birthday layer.
  It is not in the CalendarList API and is out of scope. Calendars are always chosen by
  id, never by display name.
- **Hosts.** apollo has **no** bob-gcal OAuth client (`bob gcal doctor` fails on client,
  account, and token) and cannot run a live sync. athena passes doctor and syncs only
  `primary` today. The Mac (`ssh mac`) is reachable when its lid is open. Ledger Tools
  runs `bob gcal sync` on the Mac when today's note is created. Every host installs bob
  from `~/projects/github/bobs-org/bob-cli` with `just install`.
- **Config.** `~/.config/bob/config.yml` is managed by chezmoi
  (`home/dot_config/bob/config.yml` in the `chezmoi` linked repo). Its `calendars:` key
  is commented out, so production syncs only `primary`.
- **Known hazards made reachable:**
  - `parse_calendars` accepts duplicates, so `primary` plus the primary's email would
    double-fetch.
  - `sync.rs::resolve_absent` still runs a per-row gog lookup for journaled rows whose
    calendar left the config. Day 2+ of a multi-day all-day event then fails
    `starts != day`, gives a false `Moved`, and the row is removed. A 404 gives
    `Deleted`.
  - `tests/gcal_support/fake_gog.py` serves one events fixture whatever calendar is
    asked for, so multi-calendar tests would prove nothing.

## Design

### The rule

> Bob syncs the calendars listed by id in `gcal.calendars`. Every row from a non-primary
> calendar starts its title with that calendar's configured **mark**, a short emoji that
> is unique per calendar. Primary rows stay unmarked, so an unmarked row always means
> "my own calendar".

The mark sits **inside the Google-owned title span**. Every parser in bob-cli,
bob-plugins (Ledger Tools), and Bob Mac Capture already treats that span as plain title
text. So no parse contract changes in any repo, and the existing three-way span merge
handles all migration. Marks are presentation only. They never select calendars, and
never affect roots, task type, recurrence, time accounting, block ids, or journal
identity.

### Config grammar

```yaml
gcal:
  root: meet_inbox
  calendars:
    - id: primary # unmarked: your own calendar
    - id: kjs04hafl3btrilujllivid3s8@group.calendar.google.com # Birthdays
      mark: 🎂
      # root: love   # optional; affects only timed events on this calendar
```

Validation happens when config loads. Every failure is a setup error (exit 2) that names
`gcal.calendars[N]` and gives a fix:

- `mark` is **required** on every entry except the account's primary calendar: an id of
  `primary`, or an id equal to `gcal.account` (case-insensitive). It is optional on
  primary. A bare string entry is still valid only for `primary` or the account email.
  Any other bare string fails with "needs a mark".
- Mark shape: trimmed, 1–8 Unicode scalars. No ASCII, no whitespace or control
  characters, no U+200B. ZWJ sequences, variation selectors, and flags are allowed.
- Reserved glyphs, rejected wherever they appear in a mark, ignoring U+FE0F:
  - every Tasks signifier in `render.rs::neutralize_tasks_emoji`
    (`🔁 ⏫ 🔼 🔽 🔺 ⏬ 🛫 ➕ ⏳ ⌛ 📅 📆 🗓 ✅ ❌ 🏁 🆔 ⛔`)
  - Bob's own glyphs (`📹 📍 🍅 ✎ ↤ ↦ ↻ ⚠ ⚑ ≡ 🥀 · ● — ✓ ✗ ✔ ↺`)

  Keep the Tasks list in one shared constant so the escaper and the validator cannot
  drift.

- Marks are unique across entries. Entry ids are unique (exact match after trimming).
- Example errors (exact wording is the implementer's, but keep this shape):

  ```
  gcal.calendars[1] kjs04hafl3btrilujllivid3s8@group.calendar.google.com needs a mark
    hint: add `mark: 🎂` (any short emoji) under that entry so its rows say which calendar they came from
  gcal.calendars[1].mark "📅" is reserved (Tasks due-date signifier); choose another emoji
  gcal.calendars[2].mark "🎂" is already used by gcal.calendars[1]
  ```

This is nominally a breaking config change (`calendars: [primary, work]` now fails), so
the calendar-config commit subject is `feat(gcal)!: …`
(`decisions:bob-cli-releases-ride-a-daily-train`). Never edit the Cargo version or
CHANGELOG.

Fetch-time checks run in `FetchContext::new`, which already has gog's calendar list, so
they cover `list`, `sync`, `ack`, and `note`. `bob gcal calendars` stays usable as the
fix tool. Each failure is a setup error (exit 2) and fails before any events fetch:

- A configured id that matches no listed calendar:
  `gcal.calendars[1] kjs04… is not in you@example.com's calendar list · run bob gcal calendars`
  (kind `calendar_not_found`).
- Two entries that resolve to the same Google calendar, such as `primary` plus the
  primary's email (kind `calendar_duplicate`).
- An unmarked entry that resolves to a non-primary calendar (kind `config`).

### What it looks like

The daily note (2026/20261114.md after acceptance):

```markdown
## Events

_↻ synced Sun 2026-11-15 11:40 ·
[bryanbugyi34@gmail.com](https://calendar.google.com/calendar/r/day/2026/11/14?authuser=…)_

- **all day** 🎂 Ava's Birthday [📅](https://www.google.com/calendar/event?eid=…)
- [x] #task **0900-0915** Swings! [📅](…) [root:: abby] [type:: MEET] … ^swings
```

The all-day row sorts directly under the stamp. A multi-day row keeps its qualifier:
`- **all day** 🎂 Trip · *day 2 of 3* [📅](…)`. A timed event on a marked calendar
becomes `- [ ] #task **0700-0800** 🏄 Dawn patrol [📅](…) [root:: body] ^dawn-patrol`,
with the block id minted from the bare title.

`bob gcal calendars`: synced calendars first, in config order; the rest follow in
Google's order and are dimmed. A row that is not synced shows `—` for MARK and ROOT. The
ID column comes last so it is still easy to copy.

```
Google Calendar · calendars · bryanbugyi34@gmail.com
NAME                    SYNC  MARK  ROOT        ROLE    COLOUR   ID
bryanbugyi34@gmail.com  yes   —     meet_inbox  owner   #9fc6e7  bryanbugyi34@gmail.com
Birthdays               yes   🎂    meet_inbox  owner   #7bd148  kjs04hafl3btrilujllivid3s8@group.calendar.google.com
Bloomberg               no    —     —           reader  #ac725e  …
```

`bob gcal list` header and row:
`Google Calendar · Sat 2026-11-14 · bryanbugyi34@gmail.com + 🎂 Birthdays · synced …`,
with the EVENT cell `🎂 Ava's Birthday`.

`bob gcal sync`: `  + context all day  🎂 Ava's Birthday`.

`bob gcal doctor`: `✓ calendars  2 synced · bryanbugyi34@gmail.com, 🎂 Birthdays`.

### Non-goals (v1)

- Making birthdays Event Tasks, or any classifier or `eventType` change.
- Syncing every visible calendar, or choosing calendars by display name.
- A `bob gcal sync -C ID` flag. A single-calendar snapshot would make every other
  calendar's journaled rows look absent.
- Deduplicating across calendars by title or iCalUID.
- A named badge such as `[📅 Birthdays](…)`. That would mean four parse sites across
  three repos.
- A stamp legend, mark-derived colours, and per-calendar ack or row-mode policy.
- Changes to bob-plugins or Bob Mac Capture. The mark is title text they already show.
- bob writing its own config. chezmoi owns `~/.config/bob/config.yml`.

## Phase: calendar-config

Goal: the config knows each calendar's mark. A fetch knows exactly which Google calendar
each configured entry is. Tests can serve different events per calendar.

1. `src/native/config/gcal.rs`: add `mark: Option<String>` to `GcalCalendarSettings`.
   Parse it in the map form, where a wrong type is an error naming
   `gcal.calendars[N].mark`. Keep unknown keys ignored.
2. `src/native/gcal/config.rs::GcalConfig::from_settings`: enforce the rules in **Config
   grammar**: required mark, mark shape, reserved glyphs, unique marks, unique ids, and
   the bare-string rule. Add accessors such as `mark_for(calendar_id)`. The default
   config (`[primary]`, no mark) must still resolve. Put the reserved Tasks glyph list
   in one constant shared with `render.rs::neutralize_tasks_emoji`.
3. `src/native/gcal/snapshot.rs`:
   - Resolve the configured set against `gog.calendars()` in `FetchContext::new` into a
     `SyncedCalendar { configured_id, google_id, name, mark, primary }` list, in config
     order. Apply the three fetch-time checks there.
   - Replace `SnapshotEvent.calendar_name: String` with the event's `SyncedCalendar`, a
     clone or shared reference. Remove its `dead_code` allow and keep a `name()`
     accessor for the root chooser.
   - Expose `DaySnapshot.calendars: Vec<SyncedCalendar>` so surfaces never fall back to
     the raw id just because a calendar had no events that day.
4. `tests/gcal_support/fake_gog.py` and `tests/gcal_support/mod.rs`:
   - Serve `calendar events <calendarId>` from
     `BOB_GCAL_FAKE_EVENTS_DIR/<calendarId>.json` when that directory is set. A missing
     file means no events. A `{"_fail": {...}}` payload fails that calendar only.
   - Keep `BOB_GCAL_FAKE_EVENTS` as the fallback for every calendar, so existing tests
     do not change.
   - Serve single-event lookups from
     `BOB_GCAL_FAKE_EVENT_DIR/<calendarId>/<eventId>.json` first, then the current
     `<eventId>.json`.
   - Add harness helpers to set per-calendar fixtures and to add a non-primary calendar
     (for example `Birthdays`, `bdays@group.calendar.google.com`) to the fake calendar
     list. Don't churn existing snapshots unless a test needs it.
5. Tests:
   - Config units: missing mark, bare non-primary string, ASCII mark, whitespace mark,
     over-long mark, reserved glyph (`📅`, `🍅`), duplicate mark, duplicate id. Also
     check that `primary` and the `gcal.account` email may be unmarked, and that a ZWJ
     mark is accepted.
   - Update `calendars_accept_string_ids`.
   - Integration: a configured-but-missing calendar fails with exit 2 and the note stays
     byte-identical. `primary` plus the account email fails as a duplicate. An unmarked
     entry that resolves to a non-primary calendar fails.
   - Integration: when the second calendar's fetch fails, nothing is written, even
     though the first calendar succeeded.
   - Integration: the same event id on two calendars gives two separate rows with
     independent journal state.
6. Docs:
   - `docs/gcal.md` § Configuration: the `mark` bullet and rules, and the updated
     `calendars` bullet.
   - `docs/gcal.md` § How it works: the primary/email alias rule.
   - `docs/gcal.md` § Exit status: the new kinds.
   - `docs/reference/configuration.md#gcal`: the `gcal.calendars` row becomes "Sequence
     of `primary`/account-email ids or `{id, mark, root?}` maps".

Done when `just check` passes and the default single-`primary` config still behaves
exactly as before. Existing gcal tests must pass unchanged, apart from the deliberate
string-id test update.

## Phase: marked-rows

Goal: every row from a marked calendar carries the mark at the front of its title, and
mark changes migrate through the existing ownership rules.

1. `src/native/gcal/render.rs`: add one helper,
   `row_title(mark: Option<&str>, title: &str) -> String`. It returns
   `"{mark} {escape_title(title)}"`, or just the mark if the escaped title is empty, or
   `escape_title(title)` when there is no mark. The mark itself is never escaped because
   it was validated. Add `mark` to `TaskRowInput` and `ContextRowInput`, and make
   `render_task_row` and `render_context_row` use `row_title`.
2. `src/native/gcal/plan.rs`:
   - Add `mark: Option<String>` to `PlannedOccurrence` and pass it through
     `render_occurrence`.
   - In the span-update path (the `fresh` array near the `("title", escape_title(…))`
     entry), the fresh title span becomes
     `row_title(occurrence.mark, &occurrence.title)`.
3. `src/native/gcal/sync.rs::build_occurrences`: fill `mark` from the snapshot event's
   `SyncedCalendar`, and hash the title span with `row_title`. Keep `id_hint`, the
   `needs_meet_type` probe, `@route` stripping, and the similar-task probes on the
   **bare** `event.title`. Pin that with tests.
4. Audit every other site that builds or compares a title span: adoption by URL or block
   id, force reset (`-F`), `replaced.rs`, and `plan/tests.rs` helpers. Make them all go
   through `row_title` so a marked row never looks edited to Bob.
5. Migration behaviour comes from the existing three-way span merge. Tests prove it:
   - New rows are born marked.
   - Adding, changing, or removing a calendar's mark updates **untouched** rows in place
     on the next sync.
   - A hand-edited title keeps Bryan's text and reports `kept_edit` once.
   - Primary rows are byte-identical to before (zero churn).
6. More tests (fake gog, two calendars):
   - The Ava fixture: a recurring all-day `eventType: default`, transparent event. It
     gives exactly `- **all day** 🎂 Ava's Birthday [📅](…)` directly under the stamp,
     with no checkbox, `#task`, MEET, or root. A rerun is idempotent.
   - A multi-day all-day marked row keeps its `· *day N of M*` qualifier after the mark.
   - A timed event on a marked calendar becomes a marked Event Task. Its block id is
     minted from the bare title and MEET filing is unchanged.
   - The next-year occurrence of an annual event (`…_20271114`) is a distinct row
     identity. Trust Google's instance dates; add no recurrence engine.
7. Docs:
   - `docs/gcal.md` § Events: a marked example row.
   - `docs/gcal.md` § Who owns each part of a row: a new "Calendar mark" row. Owner:
     your config. It is part of the Google-owned title span and follows config while the
     title is untouched.
   - `docs/reference/vault-format.md` § Events section and Event Tasks: the mark
     grammar.

Done when `just check` passes and an unchanged single-calendar vault sees no diff.

## Phase: deconfigured-freeze

Goal: removing a calendar from `gcal.calendars` means "stop managing this source" and
never "the events were deleted".

1. `src/native/gcal/sync.rs::resolve_absent`: pass in the resolved `GcalConfig` (or the
   snapshot's calendar set) instead of resolving config lazily. Some journaled keys on
   this day have a calendar id that matches no configured entry's configured id. For
   each one, return `AbsentVerdict::LeftAlone("calendar no longer synced")` **without**
   a gog lookup. `LeftAlone` already reports once per detail flag.
2. Audit the other journal-driven paths for rows that are not in the snapshot: the
   past-day close of Bob-filed meetings, MEET refiling, and description or ack
   reconciliation. Make sure a frozen row is never cancelled, removed, retimed, refiled,
   or closed because its calendar left the config. Link sorting may still reorder its
   link.
3. Tests, with a two-calendar journal history and the second calendar then removed from
   config:
   - A frozen timed row and a day-2 row of a multi-day all-day event both stay
     byte-identical.
   - The fake gog log shows **no** `calendar event` lookup for them.
   - The notice appears once, and a rerun stays quiet.
   - Re-adding the calendar resumes the same rows with no duplicates.
4. Docs:
   - `docs/gcal.md` § Reconciling Google-side changes: frozen rows.
   - `docs/gcal.md` § Known limits:
     - Cross-calendar duplicates are not merged.
     - Switching `primary` to the literal email (or back) changes journal identity. Pick
       one spelling.
     - The contacts-birthday layer is not a syncable calendar.

Done when `just check` passes.

## Phase: calendar-surfaces

Goal: every surface that names a synced event or calendar shows the same mark the note
shows, and JSON says which calendar each row came from. All JSON changes are additive
and `schema_version` stays 1.

1. `src/native/gcal/calendars.rs`, table and JSON:
   - Table columns: NAME, SYNC, MARK, ROOT, ROLE, COLOUR, ID.
   - Synced calendars come first, in config order. The rest follow in Google's order and
     are dimmed.
   - Use `—` for no mark, and for ROOT on calendars that are not synced.
   - The header names the account.
   - JSON rows gain `"mark"` (null when none). Existing fields are unchanged.
   - Use `style::display_width` so emoji columns align.
2. `src/native/gcal/list.rs`:
   - The header names the sync set from `DaySnapshot.calendars`, as
     `bryanbugyi34@gmail.com + 🎂 Birthdays`.
   - The EVENT cell shows `mark title` in both the plain and styled variants. Keep the
     pre-styling width logic.
   - JSON `calendars[]` gain `mark`. `events[]` gain `calendar: {id, name, mark}`, where
     `id` is the configured id. `calendar_id` stays, and `title` stays bare.
3. `src/native/gcal/sync.rs`:
   - `ReportRow` gains the row's calendar identity.
   - Human rows show the mark exactly once. Rows whose title comes from the note line
     (`journaled_title`, which is used for removed and kept rows) already carry it. Rows
     built from Google titles get the mark prefixed.
   - JSON `rows[]` gain `calendar_id` and `calendar: {id, name, mark}`. `name` falls
     back to the id and `mark` is null for calendars that are not configured.
4. `src/native/gcal/doctor.rs`, calendars check:
   - OK summary: `N synced · <name>, 🎂 <name>`.
   - On failure, name the missing or duplicate entries, with the fix
     `run bob gcal calendars and set gcal.calendars to ids you own`.
5. Human ack and note lines (`ack.rs`, `note.rs`) that print an event title prefix the
   calendar's mark through a single display helper. Do not change ack or note JSON
   beyond what already exists. If this stops being a mechanical substitution, record a
   `PROPOSED FOLLOW-UP:` on the phase bead instead.
6. Tests:
   - Human table snapshots and JSON assertions for `calendars`, `list`, `sync` (added,
     kept, and removed rows on a marked calendar), and `doctor`.
   - Every primary-only output is unchanged.
7. Docs:
   - `docs/gcal.md` § Calendars, § List, § Doctor, § Sync, and § JSON output.
   - `docs/reference/json-outputs.md`: the gcal calendars, list, and sync rows name
     `mark` and `calendar`.
   - Run `just docs-check`. There are no new flags, so no `docs-gen` change is expected;
     run it if help text changed.

Done when `just check` and `just docs-check` pass.

## Phase: enable-birthdays

Goal: the design is recorded, users can follow it, and Bryan's config enables the
Birthdays calendar.

> [!decision] mark_decision_record Use `/sase_memory_write`. Add the strand
> `decisions:non-primary-calendars-wear-a-mark` (keyword "Non-Primary Calendars Wear A
> Mark"), in the format of the existing `decisions/*.md` records:
>
> - **Applies to:** bob-cli and the vault.
> - **Claim:** calendars are selected by id. Every non-primary calendar's rows begin
>   their Google-owned title with that calendar's configured, unique mark. Primary rows
>   are unmarked. Marks are presentation only. A calendar that leaves the config is
>   frozen, not cancelled.
> - **Rejected:** named badge, qualifier chip, Obsidian tag, inline field, per-calendar
>   sections, plugin-only paint, stamp-only legend, selection by name, colour-derived
>   marks.
> - **Cost:** marks churn untouched rows when changed; edited titles keep the old mark.
> - **Reopens when:** marks need machine parsing, or a stamp legend lands.
> - **Evidence:** the archived plan, the research ref above, `docs/gcal.md` § Who owns
>   each part of a row, and the gcal tests.
>
> Link `[[decisions:bob-gcal-is-read-only-and-proposes-once]]`. Then run
> `sase memory init`.

1. `docs/guides/connect-google-calendar.md` step "3. Copy calendar ids into config": add
   another calendar by id with a mark, using Birthdays/🎂 as the example. Then preview
   with `bob gcal sync -d`.
2. Chezmoi (open it with `/sase_repo` as `chezmoi`), `home/dot_config/bob/config.yml`.
   Uncomment `calendars:` in the `gcal:` block and make it:

   ```yaml
   calendars: # default: [primary]; every other calendar needs a unique mark
     - id: primary # unmarked: your own calendar
     - id: kjs04hafl3btrilujllivid3s8@group.calendar.google.com # Birthdays
       mark: 🎂
   ```

   > [!decision] birthday_mark = cake `mark: 🎂`, as shown above.

   > [!decision] birthday_mark = gift `mark: 🎁`. live-acceptance expects 🎁 wherever
   > this plan shows 🎂 for Birthdays.

   > [!decision] birthday_mark = party `mark: 🥳`. live-acceptance expects 🥳 wherever
   > this plan shows 🎂 for Birthdays.

   Keep the commented `root:` example. Old binaries ignore `mark` and would write
   unmarked rows, which the new binary then marks in place while they are untouched.
   That outcome is benign, but live-acceptance installs the new binaries anyway before
   it applies this config.

3. Record `PROPOSED FOLLOW-UP:` notes on this phase's bead:
   - cross-calendar iCalUID dedupe;
   - an opt-in for the contacts-birthday layer (`eventType: birthday` as context);
   - a stamp legend once stamp parsers accept trailing segments;
   - a per-calendar row policy before enabling timed-event calendars such as Time
     Blocks.

Done when `just docs-check` passes, `sase memory init` is clean, and the chezmoi change
is committed through the normal final declaration.

## Phase: live-acceptance

Goal: Bryan's real 2026/20261114.md gains exactly one
`- **all day** 🎂 Ava's Birthday [📅](…)` bullet, written by the **updated** binary. Use
the glyph from `birthday_mark`.

apollo has no bob-gcal grant, so all Google-touching commands run over SSH:

- **The Mac** (`ssh mac`, user `bbugyi`) is preferred. Use a login shell (`zsh -lc '…'`)
  so PATH and the gog keyring env load.
- **athena** (`ssh athena`) is the fallback. Use it when the Mac times out or its
  `bob gcal doctor` cannot reach Google.

Builds take minutes. Run every command in the foreground with a generous timeout.

1. Confirm that origin/master contains the earlier phases' commits and that the chezmoi
   remote contains the config change. If either is missing, stop and report.
2. On every reachable syncing host (Mac, athena), update the binary first:
   - `cd ~/projects/github/bobs-org/bob-cli && git pull --ff-only && just install`
     (report and skip that host on a non-fast-forward).
   - `bob --version`, and confirm the new behaviour with `bob gcal calendars -f json`,
     which shows a `mark` field.
3. Then apply only the bob config on each host:
   - `chezmoi git -- pull --ff-only`, then `chezmoi diff ~/.config/bob/config.yml`.
   - If the diff would only bring the target up to the chezmoi source, run
     `chezmoi apply ~/.config/bob/config.yml`.
   - If it would delete host-local edits, do not apply. Report instead.
4. On the acceptance host:
   1. `bob gcal doctor` must show calendars ✓ with 2 synced, including 🎂 Birthdays.
   2. `bob gcal calendars` must show the Birthdays row as `yes` with 🎂.
   3. `bob gcal list -D 2026-11-14 -a` must show `🎂 Ava's Birthday` as context.
5. Run `bob gcal sync -D 2026-11-14 -d` and inspect the **whole** diff.
   - Expect the stamp refresh plus `+- **all day** 🎂 Ava's Birthday [📅](…eid=…)`
     directly under the stamp, with Swings! and the "Brave" row byte-identical.
   - Past-day sync may also legitimately settle eligible untouched Bob-filed meetings.
     Report any such change.
   - If anything is removed or looks wrong, stop and report without writing.
   - If the note is missing, waiting, or held, report that. It is not a pass.
   - Never use `-F`.
6. The real run is exactly `bob gcal sync -D yesterday` when the acceptance host's
   `date +%F` is `2026-11-15`, as Bryan asked. Otherwise run
   `bob gcal sync -D 2026-11-14` and say why.
7. Rerun the same command. There must be no duplicate row, and only the stamp minute may
   change.
8. Verify the result:
   - `grep -n "Ava" ~/bob/2026/20261114.md` shows exactly one marked all-day bullet.
   - Show the final `## Events` excerpt.
   - If the run happened on athena, run `bob vault-sync` once there so the note reaches
     the Mac promptly.
9. Optionally run `bob gcal list -D 2026-11-09` (read-only) to show `🎂 Dad's Birthday`.
   Make no other day writes.

Report:

- which host ran the acceptance;
- the binary version on each host;
- the dry-run diff;
- the real-run and rerun outputs;
- the final Events excerpt.

Done when Ava's birthday is in the note exactly once, marked, as a context bullet, and
the rerun is clean.
