---
tier: epic
title: Private event notes for Google Calendar events
goal: 'Bryan can give any Google Calendar event (including events other people own
  and invitations he has not answered yet) Bob-only metadata at RSVP time: `@route`,
  `#off`/`#on`, `-30 reason`, `+15 reason`, and prose. He can write it from the CLI,
  over SSH, or from Bob Mac Capture. The metadata lands on the event''s Event Task
  exactly as if those lines were in the Google description, and it beats that description
  field by field. Bob never writes to Google: it validates notes at write time, previews
  them faithfully, and never drops one silently.

  '
decisions:
  auto_sync:
    ask: On the writer Mac, should saving or clearing a note immediately sync each
      affected day whose daily note exists?
    default: true
    why: Today's row reflects the note at once instead of waiting for the next sync
    answer: true
  mac_chord:
    ask: Which chord should open the Event note mode in Bob Mac Capture?
    choices:
      cmd_e: ⌘E from any draft; the capture draft waits untouched until you come back
      cmd_e_blank: ⌘E only in an empty draft, matching ⌘J's join-event rule
      leader: Ctrl-G e under the existing editor leader; adds no new top-level chord
    default: cmd_e
    why: E for event sits beside ⌘J, and you write notes right after an RSVP, often
      mid-draft
    answer: cmd_e
  memory_record:
    ask: Record this design as a decision record that partly supersedes the buffer
      and gcal-root records?
    memory:
    - decisions:private-event-notes-beat-google-per-field
    - decisions:event-buffers-widen-the-span
    - decisions:gcal-root-follows-google-until-edited
    default: true
    corrects: Description-only grammar means a description you cannot edit cannot
      take a Google-owned buffer.
    answer: true
  memory_glossary:
    ask: Add a "Private Event Note" glossary term so it isn't confused with daily,
      area, and project notes?
    memory:
    - glossary:private-event-note
    default: false
    answer: false
phases:
- id: engine
  title: Note store and per-field layering engine
  depends_on: []
  size: medium
  description: 'engine: add the `gcal.note` journal record and its last-writer-wins
    fold, parse notes on the plain-text description path, and layer occurrence and
    series notes over Google''s description in `snapshot::fetch_day`. Give the root
    a note-first source, keep note bullets outside the cap, mark notes in `list` and
    `sync`, add list JSON identity fields, and test it all with hand-written journal
    records.'
- id: select
  title: bob gcal note command, event selection and read-only modes
  depends_on:
  - engine
  size: medium
  description: 'select: add the `bob gcal note` command skeleton and its option pre-pass.
    Select an event by title query across a 90-day window, `-D`, a pasted Google Calendar
    URL, or `-i`/`-C`. Add the TTY picker and the This event / All events scope prompt,
    widen the organizer fields, and add the list (`-l`, `-a`) and show modes with
    their JSON contract, help, and catalog row.'
- id: hardening
  title: Note provenance in gcal.row and fail-safe reads
  depends_on:
  - engine
  size: small
  description: 'hardening: journal which fields a private note contributed to each
    `gcal.row`, hold those fields with a warning when a note can no longer be read
    because journal lines are malformed, and word `kept_description` for the input
    that actually changed.'
- id: write
  title: Writing, previewing, and applying notes
  depends_on:
  - select
  size: medium
  description: 'write: normalize one-line and multi-line input into canonical lines
    using shared vectors, and reject bad routes and conflicts at write time. Show
    the faithful ''lands as'' preview and terminal time rail, then save or clear.
    Report saved versus applied, and on the writer run the immediate sync. Add the
    preview and save JSON with highlight spans, golden JSON fixtures for the Mac,
    and the `docs/gcal-notes.md` contract.'
- id: extras
  title: Editor, adopt, doctor check, and fragile-description hint
  depends_on:
  - write
  size: small
  description: 'extras: add `-e` editing in $EDITOR with a scissors help block and
    `-A` to adopt directive and reason lines from a fragile Google description. Add
    a doctor `notes` check for orphans, unresolved routes, and foreign accounts, and
    a `bob gcal list` hint for directives that live only in someone else''s description.'
- id: records
  title: Decision record, glossary term, and guide
  depends_on:
  - write
  - hardening
  size: small
  description: 'records: add the ''Private Event Notes Beat Google Per Field'' decision
    record with supersession marks and the glossary term (each per its memory decision),
    add the ''Plan an event you don''t own'' guide, and record the follow-ups this
    epic leaves out.'
- id: mac
  title: Bob Mac Capture Event note mode
  depends_on:
  - write
  - records
  size: large
  description: 'mac: in the bob-mac-capture linked repo, add an Event note mode to
    the capture panel: open it with a chord, the status menu, or a Calendar-band row;
    pick from upcoming events, edit with bob''s highlight spans, and see a live hatched
    time rail with the ''lands as'' preview. Save, clear, and undo, all through `bob
    gcal note -f json`, with docs, tests, and green exact-SHA macOS CI.'
proposed_by: bbugyi200.apollo.research.3z.linker.w0
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-10 05:59:44
status: wip
bead_id: bob-cli-ji
---

- **PROMPT:** [prompts/202611/private_gcal_event_notes.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/private_gcal_event_notes.md)
- **BEAD:** [bob-cli-ji](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-ji/README.md)

# Plan: Private event notes for Google Calendar events

## Why this design

Bryan wants metadata such as `@body` or `-30 I need to walk home from the park` on the
Event Tasks of Google Calendar events he does not own. He can't edit those events'
titles or descriptions. He also has to set the metadata when he RSVPs, which is weeks
before the daily note (and so the Event Task) exists.

The design follows the research report he approved:
`research:202611/private_notes_for_unowned_gcal_events/private_notes_for_unowned_gcal_events.md`.
Every phase agent should read it first with
`sase artifact read research:202611/private_notes_for_unowned_gcal_events/private_notes_for_unowned_gcal_events.md "<reason>"`.
He accepted all of its requirement adjustments (R1–R10). This plan settles its open
questions as follows:

- **The object is a private event note.** It is a description of a Google event that
  only Bob reads. Bob stores it in the vault's gcal journal, keyed by the event's Google
  identity. Its lifecycle is independent of the Event Task, which is only the note's
  projection. That is what makes "ahead of time" work: Bob never pre-creates daily
  notes.
- **Any event qualifies (R1).** That includes events you own and invitations awaiting a
  reply or answered Maybe (R6). A note never changes inclusion, never RSVPs, and never
  creates a row.
- **Same grammar (R2).** A note uses the description grammar Bryan already types. The
  note beats Google per field (R3). Google always keeps the time, the title, and the
  📅/📹/📍 links.
- **No new ownership rule.** The note is layered over the description before rendering.
  The row then follows the unchanged rule: it follows while untouched, your edit wins,
  and the edit is reported once. A note written after the row exists therefore behaves
  like a Google-side description change.
- **Google stays read-only** (`decisions:bob-gcal-is-read-only-and-proposes-once`).
  RSVP-in-Bob, the Keep phone path, and other extensions are follow-ups (see the end of
  this plan).
- **Two surfaces.** The CLI (`bob gcal note`) makes the feature work everywhere,
  including over SSH. The Mac Capture Event note mode is the beautiful surface, and it
  is required, not polish.
- **Decided here, not asked:**
  - The command is `bob gcal note`, and the glossary term is "private event note".
  - `gcal.description: off` hides note bullets too: one kill switch. Note _directives_
    still apply, just as Google's do.
  - Both scopes ship in v1: This event and All events.
  - Notes are never pruned silently.
  - `sync -F` keeps notes and reapplies them.

## Shared design (every phase follows this)

### Storage: one new journal record

Notes live in the existing per-host journal, `.bob/gcal/<host>.jsonl` (union-merged and
already tracked, including the doctor's gitignore rule checks). Any host may append a
note. Only the `gcal.writer` host applies notes, through sync.

```json
{
  "v": 1,
  "ts": "2026-11-09T18:02:11-05:00",
  "host": "athena",
  "kind": "gcal.note",
  "account": "you@example.com",
  "key": { "calendar_id": "primary", "event_id": "st49ogf0k0jiciob1ebgj1aupo" },
  "scope": "event",
  "text": "@family\n-30 Drive to Deerwood\n+15 Drive home",
  "seen": {
    "title": "Deerwood. Christmas breakfast",
    "start": "2026-12-13T10:00:00-05:00",
    "ical_uid": "…",
    "original_start": null
  }
}
```

- **Key.**
  - `scope: "event"` keys on `{calendar_id, event_id}`. A recurring instance's id
    already embeds its original start.
  - `scope: "series"` keys on `{calendar_id, series}`, where `series` is Google's
    `recurringEventId`. That is the same shape as `gcal.series_root`.
  - Use the configured calendar id, as `gcal.row` does.
  - Never key by title, date, path, or block id.
- **Account guard.** A note whose `account` differs from the snapshot's resolved account
  is never applied. List and doctor surface it instead.
- **Fold.** Last writer wins, ordered by `(ts, host, path, line)` exactly like
  `load_series_roots`.
  - `text: ""` is a clear. The fold remembers that a key was cleared, as opposed to
    never noted.
  - Records are append-only.
  - Text larger than 2048 bytes, an unknown scope, or a missing key part makes the
    record malformed.
- **`seen`** is for display and repair only. Never use it to move a note to another
  event.
- **Compatibility.**
  - `load_state` must skip `gcal.note`: it must not fold it as an occurrence or move
    `newest`.
  - `malformed_count` must accept valid `gcal.note` lines.
  - Older binaries print one "skipping malformed gcal journal line" warning per note
    line until upgraded. This is accepted, like `gcal.reset`, because the daily release
    train keeps the window short.
- **`sync -F`.** Reset clears day state only. Notes survive, and the rebuilt rows
  include them.

### Grammar

A note's canonical text is description grammar, parsed by the **plain-text** tokenizer
(never the HTML sniffing path):

```text
@body
-30 I need to walk home from the park
Bring the stroller
```

- **Directive line** (`@route`, `#off`, `#on`, `-N`, `+M`, `-0`/`+0`): sets the root,
  the clock tag, and the buffers. `-0`/`+0` explicitly clear a side that Google's
  description set.
- **Reason line** (`-30 …`, `+15 …`): sets one side and writes its `↤`/`↦` bullet.
- **Any other line** is prose and becomes a bullet, with the same inert escaping Google
  bullets get.

Canonical text holds the directive line first. It uses canonical spellings: `#off`,
`#on`, ASCII `-`, no `m` suffix, and the resolved route name. The reason lines follow,
before then after, then the prose lines in input order. Directive lines never render, so
the reordering is invisible in the vault.

### Precedence: each field takes the first layer that sets it

| Field                    | 1                                                                       | 2            | 3                               | then (unchanged)                                                |
| ------------------------ | ----------------------------------------------------------------------- | ------------ | ------------------------------- | --------------------------------------------------------------- |
| Root                     | ✎ this event                                                            | ✎ all events | learned series memory           | title `@route`, description `@route`, colour, calendar, default |
| Clock tag (`#off`/`#on`) | ✎ this event                                                            | ✎ all events | Google title or description tag | root walk, then on-clock                                        |
| Before buffer            | ✎ this event                                                            | ✎ all events | Google description              | none                                                            |
| After buffer             | ✎ this event                                                            | ✎ all events | Google description              | none                                                            |
| Reason, per side         | travels with the winning value                                          |              |                                 |                                                                 |
| Prose bullets            | ✎ this event lines, then ✎ all events lines, then Google's copied lines |              |                                 |                                                                 |

- A layer _sets_ a field as follows:
  - root: it has a route;
  - clock: it has `#off` or `#on`;
  - a side: it has a directive or reason value for that side, `0` included.
- If a note sets a side without a reason, Google's reason for that side is dropped, not
  inherited.
- Within one layer, today's description rules hold: `#on` beats off, and the first value
  per side wins. Conflicts inside a note are rejected at write time (see **write**).
- **Root source.** Note roots use `RootSourceKind::Note`, ranked first. It behaves like
  `Token` for rerouting: an untouched `[root::]` follows a note-derived root, while your
  `[root::]` edit wins and is reported once (`kept_root`).
  - A note route that no longer validates is a warning and falls through to the next
    candidate.
  - When a note route wins and the title ends in a resolvable `@route` token, that token
    is still stripped from the displayed title, because it was routing, not title text.
  - Source labels are `✎ @family` (this event) and `✎ all events @abby` (series).
- **Bullets.** The block order is:
  1. `↤` reason, then `↦` reason;
  2. note prose (event scope, then series);
  3. Google's copied lines, capped by `gcal.description_lines`;
  4. the overflow line.

  Note reasons and prose never count toward the cap. For an oversized Google
  description, the block is the reasons, the note prose, then the existing fallback
  bullet. `gcal.description: off` writes no bullets at all, note bullets included.

- **Inclusion.** Notes never change inclusion. They only affect Event Task rows, since
  context rows carry no root, buffer, tag, or bullets. They apply when an awaiting-reply
  or Maybe event becomes a task (Yes, or `gcal.tentative: task`), including the
  Maybe→Yes upgrade in place.

### Ownership in the row: unchanged rules, one column changes

The vault row is byte-for-byte the row the same lines would produce in a Google
description. There is no provenance glyph in the vault: titles can forge glyphs, and
`decisions:event-buffers-widen-the-span` rejected glyph tokens. Ledger Tools, the
agenda, and Mac Capture's rail already read `[buffer::]`/`[root::]`, so plugins need
nothing.

Only the Owner column in `docs/gcal.md` § Who owns each part of a row changes. It
becomes "your private note, then Google, until you edit it" for `[root::]`,
`[buffer::]`, `#off`/`#on`, and the description bullets.

Accepted v1 cost: the badges span is one hash, so a note changed after the row's birth
reaches the buffer and clock tag only while the time, title, and badges are all
untouched. Most notes land at birth. Splitting the badge hash is a recorded follow-up.

### Privacy in machine output

Note text never appears in `bob gcal list` or `bob gcal sync` JSON, which carry only
summaries. Only `bob gcal note` itself and `sync -d` diffs show the words. `GcalEvent`'s
`Debug` must redact note text, as it already redacts descriptions.

## Phase engine: note store and per-field layering engine

Repo: bob-cli. Main files:
`src/native/gcal/{journal,description,model,snapshot,root,plan,sync,list}.rs` and
`src/native/gcal/description/bullets.rs`. Suggested names are only suggestions.

1. **Journal** (`journal.rs`):
   - Add `NoteScope { Event, Series }` and a folded `NoteBook`, with lookups by
     `(calendar_id, event_id)` and `(calendar_id, series)`. Each entry carries the
     record, a `cleared` flag, and `seen`.
   - `load_notes(vault) -> io::Result<(NoteBook, Vec<String> warnings)>`.
   - `append_note(vault, host, &NewNote) -> io::Result<String>`, a single `O_APPEND`
     write via `pref_journal::append_line`.
   - Exclude `gcal.note` from `load_state`'s occurrence fold and from `newest`, and
     teach `malformed_count` to accept a valid note.
   - Update the module doc comment's record list.
2. **Parsing** (`description.rs`):
   - Add `parse_note(text, ctx) -> EventDescription` on the plain tokenizer.
   - Give `DescriptionDirectives` per-side _set_ flags, so "set to 0" differs from
     "unset". Keep today's behavior byte-identical for Google descriptions; the fixtures
     in `tests/fixtures/gcal_description/` must still pass.
   - Add a separate `note_lines` to `EventDescription`, so `render_bullets` can exempt
     note prose from the cap.
3. **Layering** (`model.rs`):
   - Add
     `GcalEvent::apply_notes(occurrence: Option<&ParsedNote>, series: Option<&ParsedNote>)`.
   - It implements the precedence table above. It mutates `clock_tag`/`off_clock`/
     `untracked`, `buffer`, `description.buffer_reasons`, and `description.note_lines`.
   - It records `note: Option<AppliedNote>`: the scopes present, the note route with its
     scope, the set of fields it contributed (`root`, `clock`, `before`, `after`,
     `bullets`), and a short hash of the applied canonical texts.
   - Redact note text in `Debug`.
4. **Snapshot** (`snapshot.rs`):
   - Load the `NoteBook` next to `series_roots`, for the snapshot's account only.
   - Apply notes right after `GcalEvent::from_google`, before root selection and clock
     resolution, so `list`, `sync`, and the later `bob gcal note` preview agree by
     construction.
   - Mirror today's series-roots stance: an unreadable journal degrades to no notes
     here, because sync's own strict `load_state` read fails loudly on I/O errors. The
     malformed-line case is **hardening**'s job.
5. **Root** (`root.rs`):
   - `choose_event_root` takes the note route (with scope) as the first candidate, with
     `RootSourceKind::Note` and the labels above.
   - Strip a resolvable title route token when a note route wins.
   - In `plan.rs`, `root_update_action` treats `Note` like `Token`.
6. **Bullets** (`description/bullets.rs`): render the order above.
   `sync.rs::build_occurrences` needs no other change, because it reads the layered
   event.
7. **Visibility** (`list.rs`, `sync.rs`).
   - **List table:** a dim `✎` after the EVENT title, before `≡ N`. The root source
     reads `✎ @family`.
   - **List JSON:** each event gains `calendar_id`, `event_id`, `root.kind` (`note`,
     `series_note`, `learned`, `token`, `colour`, `calendar`, `default`), and
     `note: null | {"scope": "event"|"series"|"both", "fields": [...], "bullet_lines": N}`.
   - **Sync human rows:** `● family  via ✎ @family`.
   - **Sync JSON:** rows gain the same `note` summary.
   - No text in either JSON.
8. **Docs.**
   - Create the contract page `docs/gcal-notes.md` ("Private event notes"), with the
     Model, Grammar, Precedence, Storage, and Ownership sections drawn from this plan.
     The **write** phase adds the command sections. Add the page to `mkdocs.yml` next to
     `gcal.md`.
   - In `docs/gcal.md`, update:
     - § Who owns each part of a row (Owner column);
     - § Root selection (note candidates first);
     - § Description bullets and § Buffers (one sentence each, linking the new page);
     - § List and § JSON output (the new fields).
   - Run `just docs-check`.
9. **Tests** (`tests/gcal_notes.rs`, with the fake gog and hand-written `gcal.note`
   journal lines):
   1. A locked guest event with a note is born with its root, `[buffer::]`, and `↤`/`↦`
      reasons.
   2. A note added after birth updates an untouched row in place and re-files its
      Bob-owned MEET, through the existing `buffer_changed` → `MovedRow` path.
   3. A row-edited root is kept and reported once with `kept_root`.
   4. Precedence:
      - occurrence beats series beats Google, each side independently;
      - `-0` clears Google's side;
      - Google's reason is dropped for a side the note overrides;
      - a note `#on` beats a Google `#off`.
   5. Clearing a note (`text: ""`) restores Google's values on an untouched row.
   6. A note in another host's journal file applies, and the last writer wins.
   7. `sync -F` keeps and reapplies notes.
   8. A note on an awaiting-reply event waits, applies on Yes, and never applies when
      the event is declined.
   9. A note follows an event moved to another day.
   10. A `gcal.note` line doesn't move `newest` and doesn't count as malformed.
   11. `list` and `sync` JSON carry no note text.
   12. Note prose is exempt from `description_lines`. `description: off` writes no note
       bullets but still applies the note's directives.
   13. A note under a different account is ignored.
   14. An incomplete snapshot still writes nothing.

Gate: `just check` (or the repo's standard fmt, clippy, and test targets) passes.

## Phase select: bob gcal note command, event selection and read-only modes

Read the `cli_rules` and `docs` reference memory before starting.

### Command shape

```text
bob gcal note [-A|--adopt] [-a|--all] [-b|--bob-dir DIR] [-C|--calendar ID] [-c|--clear]
              [-D|--day DAY] [-d|--dry-run] [-e|--edit] [-f|--format human|json]
              [-h|--help] [-i|--event-id ID] [-l|--list] [-s|--scope event|series]
              [QUERY...] [[--] TEXT...]
```

This phase registers every option so that the help text and completion are final. `-A`,
`-e`, `-c`, `-d`, and TEXT behavior land in **write** and **extras**. Until then they
exit 2 with "not available yet". Keep options alphabetical and give every long option
its short alias (`cli_rules`). Add `note` to `src/native/gcal/mod.rs`, `cli.rs`, the
`bob gcal` top help examples, and the completion tree tests. Add it to
`src/native/catalog/table.rs` as a `writes` row (network, `PreviewFirst`, dry run `-d`),
or as an `Excluded` entry with a stated reason if the catalog cannot express QUERY +
TEXT. The coverage tests require one or the other.

- **Argv pre-pass.** Run it before clap, so that `-30`, `-30m`, and `+15` are never read
  as flags, and so that `bob gcal note deerwood -c` still works.
  - Tokens that are this command's options are honored anywhere before the first `--`.
    That covers `-x`, `-x VALUE`, `--long`, and `--long=VALUE`.
  - A token matching `^[-−–+]\d` is never an option.
  - After `--`, every token is TEXT, verbatim.
  - The positional words before TEXT form QUERY. TEXT starts at the first directive
    token (`@word`, `#off`, `#on`, a deprecated off alias, or `±N[m]`) or after `--`.
  - An unknown option-like token is a clap usage error (exit 2).
- **Modes.**
  - Bare `bob gcal note`, or `-l`: list notes.
  - `-l -a`: list every upcoming event, noted or not. This is the Mac picker source.
  - `-l QUERY`: filter the listing.
  - QUERY without TEXT: show that event.
  - QUERY with TEXT, or `-i … -- TEXT`: write.
  - `-c`: clear.
- **Exit status.**
  - 0: success.
  - 1: gog or network failure.
  - 2: usage or setup error, no match, an ambiguous match off a TTY, a scope missing off
    a TTY, or (in **write**) an invalid note.

### Selecting the event

- **Window.** Fetch from today's local midnight to +90 days, per configured calendar,
  with all pages, using the existing `Gog::events` and `GcalEvent::from_google`. Parse
  every event and apply notes (so `list` and the picker show each event's effective
  note). Classify each event with its own day's `DayWindow`. `-D DAY` narrows the window
  to one day, past days included.
- **Eligible events.** Timed, busy, under-24-hour events of event type `default` or
  `fromGmail`, answered Yes, answered Maybe, or awaiting a reply. Exclude declined,
  cancelled, all-day, multi-day, and free events: they can never become Event Tasks.
- **Matching.** Every QUERY word must appear case-insensitively in the displayed title
  (clock tokens stripped). Rank an exact title first, then a prefix match, then a
  contains match, then soonest first. Recurring instances of one series group into one
  candidate: "Little Gym · ↻ weekly · next Fri Nov 13".
- **Exact selection.**
  - QUERY may be a pasted Google Calendar URL with `eid=`. Base64url-decode it, take the
    first space-separated token as the event id, and look that id up with
    `gog calendar event <cal> <id>` across the configured calendars. The calendar half
    of an `eid` is abbreviated (`@m`, `@g`) and undocumented, so never trust it. If no
    configured calendar holds the id, fall back to a title search with a warning.
  - `-i ID` uses the same lookup.
  - `-C ID` pins the calendar for `-i` or a URL. An id found in two configured calendars
    without `-C` is exit 2 with both calendars listed.
  - Exact selection works outside the 90-day window. That matters for the Mac
    Calendar-band entry, which passes the row's 📅 URL.
- **Results.**
  - Zero matches: exit 2, with a hint (`-D DAY`, `bob gcal list -D …`).
  - One match: proceed.
  - Several matches on a TTY: a numbered picker, soonest first, showing day, time,
    title, "from <organizer>", the RSVP chip, the ↻ group, and the `✎` summary when a
    note exists.
  - Several matches off a TTY: exit 2, listing candidates with their `-i`/`-C` values.
    In JSON, `ok: false`, `error.kind: "ambiguous"`, and a `candidates` array.
  - Reuse the existing TTY test hook (`BOB_GCAL_ASSUME_TTY`).
- **Scope.**
  - A recurring match on a TTY asks:

    ```text
    Little Gym repeats weekly · Fridays 09:15–10:00 · guest · Maybe
      1  This event   Fri Nov 13
      2  All events   every Little Gym
    Which? [1] ›
    ```

    "This event" means the soonest matching occurrence, or the `-D` day's occurrence.

  - Off a TTY, a recurring match without `-s` exits 2.
  - `-s series` on a one-off event exits 2.

- **Organizer.** Widen `gog.rs` `FIELDS` from `organizer(self)` to
  `organizer(self,displayName,email)`. Store the organizer on `GcalEvent` as a display
  name, falling back to the email. JSON carries `{name, self}` only, never the email.

### List and show output

Human output uses the existing `ui.rs` styler and hue dots, in colour on a TTY and plain
text off a TTY:

```console
$ bob gcal note
✎ 3 private notes · next 90 days
  Fri Nov 13  09:15  Little Gym                     all events  ● abby    −30m       waiting for Yes
  Sun Dec 13  10:00  Deerwood. Christmas breakfast  this event  ● family  −30m +15m
  ⚠           Interview with PermitFlow             this event  no longer on your calendar · bob gcal note -c -i 7k2…

$ bob gcal note deerwood
✎ Deerwood. Christmas breakfast
  Sun Dec 13 · 10:00–12:00 · from Kelly · Yes
  no private note yet · add one: bob gcal note deerwood @family -30 Drive there
```

- Status words come from the event's state:
  - `waiting for Yes` for Maybe events that won't be tasks;
  - `waiting for your reply`;
  - `lands Sun Dec 13`;
  - `on today's note`.
- Notes whose event is outside the window are looked up by id, capped at 25 lookups.
  - Past events are hidden.
  - Cancelled, declined, or not-found events show `⚠ no longer on your calendar` with
    the one-line clear command.
  - A series note with no occurrence in the window looks up its series master. A gone
    master is an orphan.
- Show mode prints the current canonical note lines and the effective root, buffer, and
  clock. The **write** phase adds the "lands as" block to show mode.

### JSON (`schema_version: 1`)

Every event object, in the list, show, candidates, and later write outputs:

```json
{
  "calendar_id": "primary",
  "calendar_name": "Bryan",
  "event_id": "st49…",
  "series_id": null,
  "recurring": false,
  "title": "Deerwood. Christmas breakfast",
  "day": "2026-12-13",
  "start": "2026-12-13T10:00:00-05:00",
  "end": "2026-12-13T12:00:00-05:00",
  "time": "10:00-12:00",
  "answer": "yes",
  "organizer": { "name": "Kelly", "self": false },
  "url": "https://www.google.com/calendar/event?eid=…",
  "lands": "task|context|waiting_reply|waiting_maybe",
  "note": null
}
```

- `note` holds a summary in listings:
  `{"scope","root":{"value","hue"},"buffer","clock","bullet_lines"}`. It never holds
  text.
- Show mode adds `note.text` and `note.lines`. Show is the only read path that prints
  the words.
- The list document is
  `{schema_version, ok, window:{from,to}, notes|events, orphans, warnings}`.
- Document the command in `docs/gcal-notes.md` § Command and JSON, add a
  `docs/reference/json-outputs.md` row, and run `just docs-gen` and `just docs-check`.

### Tests (`tests/gcal_note_cli.rs`)

Cover:

- argv pre-pass vectors, including `-30m`, `−30`, `-c` after the query, and `--`;
- matching and ranking;
- series grouping;
- URL `eid` decoding, with the verify-and-fallback path;
- `-i`/`-C` and ambiguous ids;
- the TTY picker and scope prompt;
- off-TTY exit 2 with candidates;
- excluded declined and all-day events;
- list orphans and past-hiding;
- JSON shapes, with no note text in list JSON.

Extend `tests/gcal_support/fake_gog.py` to honour `--from`/`--to` windows if the tests
need it.

## Phase hardening: note provenance in gcal.row and fail-safe reads

Repo: bob-cli (`journal.rs`, `plan.rs`, `sync.rs`).

1. **Provenance.** `gcal.row` gains an additive
   `note: {"hash": "<16 hex>", "fields": ["root","clock","before","after","bullets"]}`
   whenever a private note contributed to the row this sync wrote or tracked. The field
   is absent otherwise. Older binaries read fields with `object.get` and ignore it.
   `RowRecord` parses it.
2. **Fail-safe hold.** A journaled row may carry a `note` record with fields, while the
   current fold has neither a live note nor a deliberate clear for that key, and the
   journal reports malformed gcal lines. In that case, hold the row:
   - skip its span, root, and description updates;
   - carry its journal record forward unchanged;
   - warn on every run while the condition lasts:
     `held Deerwood. Christmas breakfast: its private note could not be read (2 malformed gcal journal lines) · bob gcal doctor`.

   A deliberate clear is never held. Report held rows in human and JSON sync output
   (`warnings`, and the row's `detail`).

3. **Accurate wording.** When `kept_description` fires and the journaled note hash
   differs from the fresh one, say
   `kept your edited description bullets; your private note changed`. Otherwise keep
   today's Google wording.
4. **Docs.** Add one paragraph each to `docs/gcal-notes.md` § Reliability (create it if
   **write** hasn't yet) and to `docs/gcal.md` § Reconciling Google-side changes.
5. **Tests.** Cover:
   - the provenance record shape;
   - holding on a truncated note line versus re-applying Google's values after a
     deliberate clear;
   - no hold when nothing is malformed;
   - the `kept_description` wording for a note-only change.

## Phase write: writing, previewing, and applying notes

Repo: bob-cli (a new `src/native/gcal/note*.rs`, plus `sync.rs`, `render.rs`, docs).

### Normalization

Rules apply to every input line. Argv TEXT words are joined with single spaces into one
line. Text containing newlines (from the Mac, or from `-e` in **extras**) is split into
lines first.

1. A line that starts with directive tokens is scanned token by token.
   - A `±N[m]` token immediately followed by a non-directive word starts a **reason**.
   - A `±N[m]` token followed by a directive token or the end of the line is a bare
     buffer.
   - A reason runs until a `±N[m]` token for the _other_, still-unset side, which starts
     the next reason, or until `;`.
   - `@` and `#` inside a reason are text.
2. A non-directive word that follows the leading directives and is not part of a reason
   starts a **prose** line that runs to the end of the line, or to `;`.
3. `;` breaks a line only in a line that started with a directive token. A line that
   starts with prose is one bullet, verbatim. That keeps "Bring snacks; and water"
   whole.
4. The result is re-emitted as canonical text (see Shared design § Grammar).

So:

- `@body -30 I need to walk home from the park` → `@body` /
  `-30 I need to walk home from the park`;
- `-30 Drive there +15 Drive home` → two reasons;
- `@family #off -30 +15` → one directive line.

Commit the cases as shared vectors in `tests/fixtures/gcal_note/normalize.json`
(`{input, canonical, errors}`), and test them in Rust. Bob Mac Capture never parses
grammar; it relies on these through bob's JSON.

### Validation (all errors at once, exit 2, nothing saved)

- **Route.** Resolve it with `parent_notes::resolve_parent` plus
  `root::validate_root_with`, against the occurrence day's daily-note path. That works
  before the note file exists. An unknown or invalid route lists the closest routes
  ("did you mean body (area · Body)?").
- **Conflicts.** Two different routes, `#off` together with `#on`, and two different
  values for one side are each errors that name both tokens.
- **Minutes and size.** Minutes must be 0–720. A reason needs minutes > 0. The canonical
  text must be ≤ 2048 bytes.
- **Warnings, not errors.**
  - A `#word` that isn't a directive, which becomes an escaped bullet ("`#foo` is not a
    Bob directive; it will be a bullet").
  - A prose line whose first token looks like a directive.
  - Option-like tokens inside TEXT such as `-d` ("put options before the event query").
  - A deprecated off alias, which is canonicalized to `#off` with a notice.

### Preview, save, clear

- **One occurrence builder.** Factor sync's per-event work in `build_occurrences` (root,
  `needs_meet_type`, badges, buffer, bullets) into one function. The note preview calls
  it on the event with the _proposed_ note applied. What you see is what sync writes.
- **"Lands as" block.** `render_task_row` plus the materialized bullets, dimmed on a
  TTY. The block id is minted from the title as sync would mint it at birth.
- **Saving.** Append a `gcal.note` record with `seen`. Writing canonical text identical
  to the current note appends nothing (`unchanged`). `-c` appends a clear. For a
  recurring event, `-c` offers only the scopes that have a note. `-d` validates and
  previews but writes nothing.
- **Saved versus applied.** Every write reports exactly one of:
  - `lands in 2026/20261213.md when that day syncs`;
  - `you haven't replied: lands when you say Yes`;
  - `you answered Maybe: lands when you say Yes` (unless `gcal.tentative: task`);
  - `↻ applied to 2026/20261109.md`;
  - `saved · 2026/20261109.md updates on the next sync on the-mac` (off the writer).

  A series note reports "applies to every Little Gym" plus the next occurrence's status.

> [!decision] auto_sync On the writer host (`gcal.writer` unset or equal to this host),
> after a successful save or clear, run the existing sync in-process (quiet) for each
> affected day whose daily note exists and is rendered (normally only today). Report
> `↻ applied to …` or relay the sync's warnings (`kept_edit`, `kept_root`,
> `waiting for Templater`, …). A sync failure never undoes the saved note: exit 0, print
> the failure as a warning, and hint `bob gcal sync`.

> [!decision] auto_sync = no Never sync from `bob gcal note`. When an affected day's
> note exists, report `saved · run bob gcal sync to apply it to 2026/20261109.md now`.

### Human output (beautiful on a TTY, plain off it)

```console
$ bob gcal note deerwood @family -30 Drive to Deerwood +15 Drive home
✎ Deerwood. Christmas breakfast
  Sun Dec 13 · 10:00–12:00 · from Kelly · Yes

  @family
  -30 Drive to Deerwood
  +15 Drive home

  ● family  09:30 ╱╱╱╱╱╱ 10:00 ━━━━━━━━━━━━━━━━━━━━━━━━ 12:00 ╱╱╱ 12:15

  lands as
  - [ ] #task **1000-1200** Deerwood. Christmas breakfast [📅](…)
    [buffer:: -30m +15m] [root:: family] [type:: MEET] ^deerwood-christmas-breakfast
    - ↤ *30m before* · Drive to Deerwood
    - ↦ *15m after* · Drive home

✓ Noted · only in Bob · lands in 2026/20261213.md when that day syncs
```

- **Canonical lines.** Colour them with the same span kinds the JSON reports: the route
  in its root hue, the buffer tokens, the reason text, prose.
- **The time rail** is one line, at most 48 columns:
  - the wings are `╱` in the dim root hue;
  - the meeting is `━` in the root hue;
  - widths are proportional, with at least one character for a nonzero wing;
  - a midnight clip shows `(clipped at 00:00)`.

  The rail makes a wrong sign obvious: `-30` reserves time _before_ the meeting, and Bob
  never guesses a sign from prose.

- **Clearing** prints `✓ Cleared · Google's description decides again`.
- Show mode now also prints the rail and the "lands as" block.

### Write JSON (the Mac contract)

```json
{"schema_version":1,"ok":true,"action":"preview|saved|unchanged|cleared|show",
 "event":{…event object…},"scope":"event",
 "input":{"text":"@family -30 Drive to Deerwood +15 Drive home"},
 "spans":[{"start":0,"end":7,"kind":"route"},{"start":8,"end":11,"kind":"buffer"},
          {"start":12,"end":28,"kind":"reason"}],
 "note":{"text":"@family\n-30 Drive to Deerwood\n+15 Drive home",
         "lines":[{"kind":"directives","text":"@family"},
                  {"kind":"reason","side":"before","minutes":30,"text":"-30 Drive to Deerwood"},
                  {"kind":"reason","side":"after","minutes":15,"text":"+15 Drive home"}]},
 "effective":{"root":{"value":"family","source":"✎ @family","kind":"note","hue":{…}},
              "clock":null,"buffer":{"before_minutes":30,"after_minutes":15},
              "meeting":{"start":"10:00","end":"12:00"},
              "occupied":{"start":"09:30","end":"12:15","clipped":false},
              "fields":["root","before","after","bullets"]},
 "preview":{"path":"2026/20261213.md","row":"- [ ] #task **1000-1200** …","bullets":["- ↤ *30m before* · Drive to Deerwood","- ↦ *15m after* · Drive home"]},
 "status":{"state":"waiting_note|applied|saved_elsewhere|waiting_reply|waiting_maybe|sync_failed","detail":"…","path":"…"},
 "errors":[],"warnings":[]}
```

- **`spans`** are half-open UTF-8 byte offsets into `input.text`, the same convention as
  `capture-parse`. Kinds: `route`, `clock`, `buffer`, `reason_sign`, `reason`, `prose`,
  `separator`, `invalid`.
- **Invalid input** returns `ok: false` with exit 2. The output still has `spans`, plus
  `error: {kind: "invalid_note", message, hint}` and an
  `errors: [{start, end, kind, message, suggestions}]` array, so the Mac can underline
  the problem.
- **Golden fixtures.** Commit representative outputs as golden fixtures in
  `tests/fixtures/gcal_note/json/` (`list_all`, `show`, `preview`, `saved`, `unchanged`,
  `cleared`, `invalid`, `ambiguous`) and assert them in tests with timestamps and URLs
  normalized. The **mac** phase copies them into its fake bob.

### Docs

- Finish `docs/gcal-notes.md`: § Command (examples above), § When a note lands (the
  research's table), § JSON, and § Reliability.
- Update `docs/gcal.md`:
  - § Commands block, Contents, and Known limits. Replace the "description you cannot
    edit" limit with a pointer to private notes, and add the badge-hash cost.
  - § Exit status.
- Run `just docs-gen` and `just docs-check`.

### Tests

Cover:

- the normalization vectors;
- every validation error and warning;
- the preview equals the row sync later writes (an end-to-end test: write the note,
  create the daily note, sync, then compare the vault row to `preview.row` and
  `preview.bullets`);
- `unchanged` idempotency;
- clear;
- series scope;
- saved-versus-applied states;
- the writer-host immediate sync, under the chosen `auto_sync` branch;
- the off-writer message;
- a sync failure after a save still exits 0;
- the golden JSON fixtures.

## Phase extras: editor, adopt, doctor check, and fragile-description hint

Repo: bob-cli.

1. **`-e|--edit`.**
   - Open `$VISUAL`, then `$EDITOR`, then `vi`, on a temp file holding the current
     canonical note. A TTY is required (exit 2 with a hint otherwise). `#` starts
     directives, so comment lines are impossible. Use a git-style scissors block
     instead:

     ```text
     @family
     -30 Drive to Deerwood
     ------------------------ >8 ------------------------
     Do not modify or remove the line above; everything below it is ignored.
     Private note for Sun Dec 13 10:00–12:00 Deerwood. Christmas breakfast (from Kelly · Yes)
       @route  #off  #on  -30  +15        directives: root, clock, buffers
       -30 Drive there   +15 Drive home   buffer reasons (bullets)
       anything else                      bullets on the Event Task
     An empty note clears it.
     ```

   - On save, run the same normalization, validation, and preview path as **write**.
   - On validation errors, print them and ask `Edit again? [Y/n]`. Answering `n` exits 2
     and saves nothing.
   - Empty text clears the note. Unchanged text reports `unchanged`.

2. **`-A|--adopt`.**
   - Copy the directive and reason lines from the event's _current Google description_
     into a private note, after normalization and validation. Prose stays in Google.
   - If the event already has a note in the chosen scope, exit 2 with "edit it with
     `-e`". Combining `-A -e` opens the editor prefilled with the adopted lines.
   - Output says the lines "now live in Bob too: they survive the organizer's next
     update".
3. **Doctor `notes` check** (`doctor.rs`, JSON id `notes`):
   - ok: `3 private notes`.
   - warn when any of these exist:
     - orphans: the event is not found, cancelled, or declined, or the series is gone;
       capped at 25 lookups, with "N not checked" beyond that;
     - routes that no longer resolve against today's note path;
     - notes under a different account;
     - malformed note records.
   - Each warning comes with a one-line fix (`bob gcal note -c -i …` or
     `bob gcal note -e -i …`). Update `docs/gcal.md` § Doctor.
4. **Fragile-description hint.** In `bob gcal list`, flag an event Bob would make a task
   when all of these hold:
   - you don't organize it, and it has guests;
   - its Google description carries Bob directives or reasons;
   - it has no private note.

   The hint reads:

   ```text
   ⚑ Interview with PermitFlow: its Bob lines live in the organizer's description and may vanish on their next update · bob gcal note -A -i …
   ```

   JSON gains `fragile_directives: true` on that event.

5. Tests for each item, plus docs in `docs/gcal-notes.md` and the help text. Then run
   `just docs-gen` and `just docs-check`.

## Phase records: decision record, glossary term, and guide

Use `/sase_memory_write` before any memory edit, and touch only what an accepted memory
decision covers. Run `sase memory init` after memory edits.

> [!decision] memory_record Add a new `decisions` web strand,
> `decisions:private-event-notes-beat-google-per-field`, titled "Private Event Notes
> Beat Google Per Field", in the decisions web's format: Applies to, Claim, Why,
> Rejected alternatives, Cost, Reopens when, Evidence. Status is `accepted`, with
> today's date.
>
> - **Claim.** A private event note is vault-stored, Bob-only description text keyed by
>   the Google event. It beats Google's description per field (this event, then all
>   events, then Google; a reason travels with its side's winning value). The row keeps
>   one ownership rule.
> - **Rejected alternatives** (see the research's options table): writing to Google or
>   to `extendedProperties`, the RSVP comment, pre-touched overrides, seed-once, a
>   separate ownership block, shadow events, pre-created daily notes, Markdown holding
>   notes, and a capture sigil for the Mac surface (a dedicated mode instead).
> - **Cost:** the single badge hash, old-binary warnings, and Bob occupancy that is not
>   Google free/busy.
> - **Reopens when:** RSVP-in-Bob is wanted, reasons must survive hand-edited blocks, or
>   notes must reach rows with edited titles.
> - **Evidence:** this plan, the research ref, `docs/gcal-notes.md`, and
>   `tests/gcal_notes.rs`.
>
> Mark `decisions:event-buffers-widen-the-span` (append to its existing `superseded_by`
> list) and `decisions:gcal-root-follows-google-until-edited` (status
> `superseded-in-part`) with `superseded_by` pointing at the new record. Add one
> `[[decisions:private-event-notes-beat-google-per-field]]` back-link sentence to each,
> stating what was retired:
>
> - the "description you cannot edit cannot take a Google-owned buffer" cost;
> - the chosen root now begins with the note candidates.
>
> Link `[[decisions:buffer-reasons-ride-description-bullets]]` from the new record
> rather than marking it: its claim stays true.

> [!decision] memory_record = no Write no decision memory. Record
> `PROPOSED FOLLOW-UP: decision record for private event notes` on this phase's bead.

> [!decision] memory_glossary Add a new `glossary` web strand,
> `glossary:private-event-note` ("Private Event Note"; aliases: private note, ✎ note).
> It is a Bob-only description of one Google Calendar event, or of all events in its
> series, stored in the vault's gcal journal. It is never a vault Markdown note and
> never sent to Google. Link `[[event-task]]`.

> [!decision] memory_glossary = no Record
> `PROPOSED FOLLOW-UP: glossary term Private Event Note` on this phase's bead.

Also:

- **Guide.** Add `docs/guides/plan-events-you-dont-own.md` ("Plan an event you don't
  own"), under the guides' word budget. Cover:
  - RSVP, then `bob gcal note <query> …`;
  - recurring scope;
  - checking with `bob gcal note` and `bob gcal list`;
  - adopting fragile external-invite lines with `-A`;
  - the `gcal.colors` stopgap for routing from the phone;
  - what "lands" means.

  Add it to the guides index and `mkdocs.yml`, then run `just docs-check`.

- **Follow-ups.** Record each item under "Follow-ups" below as a `PROPOSED FOLLOW-UP:`
  note on this phase's bead. Do not create beads.

## Phase mac: Bob Mac Capture Event note mode

Open the `bob-mac-capture` linked repo with `/sase_repo` and read its `AGENTS.md`. Its
rules apply: compile all Swift, `just portable-check`, typecheck on the mac,
`Scripts/macos-ci-check` before pushing, and watch the exact-SHA macOS run to green with
`/sase_monitor`. This phase is `large`: plan it first. Your plan may split it further.

The app stays a thin client (`decisions:mac-capture-is-a-thin-client`). It never parses
`@route` or buffers and never writes the vault. Every list, highlight, preview, save,
clear, and undo goes through `bob gcal note … -f json`. Add a new `SchemaVersioned`
response and use `decodeReport`, because invalid notes print JSON with exit 2. Seed the
fake bob from bob-cli's golden fixtures in `tests/fixtures/gcal_note/json/`.

Design brief (the beautiful surface):

- **Entry points.**
  - The chord below.
  - A status-menu item, "Event Note…", which opens the panel straight into the mode.
  - On an idle-agenda Calendar-band row, the same chord opens that event's editor
    directly. The agenda's `calendar_url` (the 📅 link) goes to bob as the URL QUERY.

  The mode is a surface inside the capture panel. The capture draft and its undo stack
  wait untouched, and Esc returns to them. It is never a capture sigil
  (`decisions:capture-sigils-say-where-what-how`).

> [!decision] mac_chord = cmd_e ⌘E opens the mode from any draft state. Route it in
> `CaptureKeyCommandRouter` and verify nothing else claims ⌘E.

> [!decision] mac_chord = cmd_e_blank ⌘E opens the mode only when the draft is blank,
> exactly like ⌘J; otherwise the key stays with AppKit.

> [!decision] mac_chord = leader Add `"e"` to `editorLeaderBindings` (Ctrl-G e).

- **Picker** (`bob gcal note -l -a -f json`, its own lane, loaded on entry with a quiet
  spinner):
  - Soonest first, under day headers (Today, Tomorrow, Sun Dec 13).
  - Each row shows the time, title, "from Kelly" or the calendar name, an RSVP chip
    (Yes, Maybe, Awaiting reply), ↻ for recurring events, and, when noted, a chip in the
    root hue (`✎ ● family · −30m +15m`).
  - Typing filters locally and never chooses by itself.
  - Ctrl-N/P and the arrows move the selection, Return opens the editor, and Esc leaves.
- **Editor.**
  - A fixed identity header card: day, time, title, organizer, RSVP chip. You always
    know which event you are changing.
  - For recurring events, a This event / All events segmented control. Pick a key that
    is free inside the mode, and document it.
  - A multi-line field. It is painted from bob's `spans` (UTF-8 byte offsets, through
    the existing `validatedSpanRanges`): the route in the root hue, buffer tokens,
    reason text, prose, and red underlines with messages for `errors`.
  - Preview with `-d`, debounced about 150 ms, on a lane that cancels the previous run.
- **Preview.**
  - A time rail with the meeting as a solid capsule in the root hue and real diagonal
    _hatched_ wings for buffers, labelled with each reason. Below it,
    `Occupied 09:30–12:15 · Meeting 10:00–12:00`.
  - Then a dimmed "Lands as" rendering of `preview.row` and `preview.bullets`.
  - Extract the agenda's private rail views (`CaptureAgendaView.swift` `railWingView`
    and related) into a shared component. Use it in both the agenda and here, so the
    preview reads like the day it describes. Today's agenda wings are a 0.35-opacity
    fill, despite their doc comment; adopt true hatching in both places unless the
    agenda's render tests show a regression you can't fix.
  - Every state has a text label, so nothing depends on colour alone. Add VoiceOver
    labels.
- **Footer and save.**
  - The footer shows `Only in Bob · This event`, with key hints
    `⏎ Save · ⇧⏎ New line · ⌘⌫ Clear · ⎋ Back`. Follow the capture panel's
    Return/Shift-Return convention.
  - After a save, show the status bob reports ("✓ Saved · lands Sun Dec 13" or "↻
    Applied to today's note") and return to the picker, with the event now wearing its
    chip. That way several RSVPs can be noted in a row.
  - ⌘Z, while the mode is open, undoes the last save or clear by re-saving the previous
    text, or clearing it.
- **Old bob.** If bob is too old (`isUnsupported*Failure`), show a friendly empty state
  ("Event notes need a newer bob") and hide the menu item.
- **Docs.**
  - A new page, `docs/event-notes.md`, with a `nav.yml` entry. The existing reference
    pages are at the 6,000-word cap, so add only short cross-links to `keyboard.md`,
    `protocol.md` (the new optional command plus its compatibility row), and
    `idle-agenda.md`.
  - Add render fixtures for the rail.
  - Add a short "On your Mac" section to the bob-cli guide from **records**.
- **Tests.**
  - CaptureCore decoding tests against the golden fixtures.
  - Router tests for the chosen chord.
  - Hosted panel tests for the picker, editor, save, undo, and old-bob states.
  - A `gcal note` case in `Tests/Fixtures/fake-bob`.

## Testing and gates (all bob-cli phases)

- Tests never touch Google: use the fake gog (`BOB_GCAL_GOG`, `BOB_GCAL_FAKE_EVENTS`,
  `BOB_GCAL_FAKE_EVENT_DIR`).
- Each phase leaves `cargo fmt`, clippy, the full test suite, `just docs-check`, and
  (when help changes) `just docs-gen` clean.
- Never edit `Cargo.toml`'s version or `CHANGELOG.md`
  (`decisions:bob-cli-releases-ride-a-daily-train`). These are additive features, so use
  `feat(gcal): …` commit subjects, not `!`.

## Follow-ups (not in this epic)

- Split the badges hash (📅/📹/📍 versus clock tag and `[buffer::]`), so a note reaches
  rows with an edited title or time.
- A Google Keep phone path (`gcal <query>` plus directives in a Keep note, applied only
  when it resolves to exactly one event), if phone RSVPs turn out to be common. Until
  then, configure `gcal.colors` and recolour the event on the phone.
- "This and following" scope for series splits.
- A Ledger Tools pill tooltip: "from your private note".
- RSVP from Bob. This reopens `bob-gcal-is-read-only-and-proposes-once`. The narrower
  `calendar.events.owned` scope would suffice.
