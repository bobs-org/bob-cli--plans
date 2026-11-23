---
tier: tale
title: Link to daily-note Event Tasks from the Links stage
goal: The Ctrl+Shift+L Links stage finds, names, and links Event Tasks from daily notes.
size: medium
decisions:
  daily_scope:
    ask: Should the Links stage admit only Event Tasks from daily notes?
    choices:
      events:
        Only `## Events` rows led by `**HHMM-HHMM**` join; `^gtd` rows and old ledger
        tasks stay out
      all:
        Every daily-note task joins, like Mac Capture's `~` catalog, adding a `^gtd` row
        per day
    default: events
    why:
      Event Tasks are what was asked for; other daily-note tasks add hundreds of
      look-alike rows.
    answer: events
  day_window:
    ask: Should Event Tasks from every past day be linkable?
    choices:
      all_days:
        Every daily note's Event Tasks join; equal-score rows rank newest day first
      recent:
        Only today, future days, and the last 30 days join; older meetings need a
        hand-written link
    default: all_days
    why:
      Relations are history and capture `~` reaches any day; newest-first ranking sinks
      old rows.
    answer: all_days
  chip_names:
    ask: Should LINKS and LINKED FROM chips also name Event Tasks as `HHMM-HHMM Title`?
    choices:
      clean:
        Chips drop the 📅/📹/📍 link markdown and fields, matching the Links stage rows
      leave:
        Chips keep their current text, such as `1100-1130 Phone Call w/ Dan
        [📅](https:/…`
    default: clean
    why:
      Otherwise every new meeting link shows the raw gcal link markdown seen in the
      screencast.
    answer: clean
proposed_by: bbugyi200.apollo.fr
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-23 11:42:26
status: wip
---

# Links stage: link to Event Tasks in daily notes

## Context

**What the screencast shows.** On the job note's `Notion Software Engineer…` task,
`Ctrl+Shift+L` opens `🪢 Links · …` ("searching 5115 tasks"). Typing `derrick blake`
never surfaces today's meeting `1500-1530 Bryan Bugyi and Derrick Blakely`. That meeting
is an Event Task in `2026/20261123.md`. Paging with `Ctrl+N` shows only unrelated
subsequence matches.

**Root cause.** The Links stage lives in bob-plugins `plugins/bob-navigation-hotkeys`.
In `src/304-relation-stage-view.js`, `isRelationStageExcludedPath` drops every
`YYYY/YYYYMMDD.md` path. Both pool builders call it:

- `collectVaultRelationCandidatesFromSnapshots`, for buffered notes and the cold-cache
  vault scan;
- `mergeRelationStageCacheCandidates`, for Obsidian Tasks cache records.

An Event Task is an ordinary `#task` row with a gcal-minted `^block-id`. Nothing else
keeps it out of the pool.

**Everything downstream already accepts daily-note targets.** Only the pool needs to
change:

- **bob's contract.** A relation target may live in any daily note. See
  `docs/capture.md` "Targets" and `docs/task-relations.md` §5–§7. Reconcile writes
  LINKED FROM on writable targets, which covers every daily note except the previous
  one. Bob Mac Capture's `~` catalog (`capture_dependency_tasks::discover`) already
  lists daily-note tasks. **No bob-cli code change is needed.**
- **Nav commit path.** The mirror is already skipped for `done/` and for the previous
  daily note. See `relationPathIsWritable` in `src/086-relation-line-plan.js`,
  `src/594-plugin-relation-edit.js`, and `src/595-relation-follow-ups.js`.
- **gcal sync.** It keeps managed first children (LINKS, LINKED FROM) under Event Tasks.
  A cross-day move cancels the old row in place, so a link never dangles (see
  `docs/gcal.md`).
- **Live relation index.** bob-ledger-tools already resolves daily-note ends. The
  screencast's job note shows `INSPIRED BY → 1100-1130 Phone Call w/ Dan …`.

**Event Task definition** (glossary; bob `src/native/event_task.rs`):

- It is a checkbox `#task` row inside a canonical daily note's `## Events` section.
- `events_section_range` defines that section: it runs to the next level-two heading and
  skips frontmatter and fenced code.
- After the optional `#task` tag, the row's text starts with `**HHMM-HHMM**`
  (`parse_event_span`).
- Its title is `clean_event_title(text after the span)`.
- JS twins exist in task-status-cycler `src/061-event-task-clock.js` and in
  bob-ledger-tools `src/035-meet-event-run.js` (`eventRunCleanTitle`).
  bob-navigation-hotkeys has none.

## Goal

The Links stage gains three abilities:

- Typing finds Event Tasks from daily notes.
- Each one appears as `HHMM-HHMM Title` with its day.
- `↵` links it with today's mirror behavior.

Everything else in the Links and Linked-from stages stays as it is.

## Design (bob-plugins)

### 1. Event Task twin in nav

Add the new fragment `src/304a-relation-event-targets.js` and list it after
`304-relation-stage-view.js` in `fragments.json`. Fragment 304 is already 875 lines, and
the cap is 1000. The fragment holds pure helpers whose comments cite the Rust functions,
following the style of `061-event-task-clock.js`:

- `isRelationStageDailyNotePath(path)`: the existing `^\d{4}\/\d{8}\.md$` test.
- `relationStageEventsSectionRange(lines)`: twin of `events_section_range`.
- `relationStageEventTaskSpan(rawLine)`: twin of `parse_event_span`. It returns
  `"HHMM-HHMM"` or `null`.
- `relationStageEventTaskName(rawLine)`: returns `span + " " + title`, where title twins
  `clean_event_title`. It strips the trailing `^id`. It then strips trailing `[k:: v]`
  fields, off-clock tags, and Markdown links, repeating until nothing changes. Finally
  it drops a trailing ` · *…*` tail. It returns `null` for a line that is not an Event
  Task.
  - Example: the Derrick row
    `- [*] #task **1500-1530** Bryan Bugyi and Derrick Blakely [📅](…) [📹](…) [📍](…) [buffer:: -15m] [root:: job] [type:: MEET] ^bryan-bugyi-derrick`
    becomes `1500-1530 Bryan Bugyi and Derrick Blakely`.
- `relationStageDailyAdmission(path, entry, context)`: decides whether a daily-note row
  joins the pool. It applies `daily_scope` and `day_window`; `context` carries `today`
  and the section verdict. An admitted row is returned decorated with:
  - `dailyNote: true`;
  - `eventTask` (boolean);
  - `noteDay` (`"2026-11-23"`, parsed from the file name);
  - `displayText` and `text` set to the event name when it is an Event Task.

### 2. Pool admission (`304`)

- **Keep `isRelationStageExcludedPath` byte-for-byte.** It still answers `true` for
  daily notes, so the Linked-from fallback scan
  (`collectRelationStageIncomingFromSnapshots`) and the existing assertions stay
  unchanged.
- **Route daily notes through admission instead.** In both pool collectors, check a
  daily-note path first and send each of its rows through `relationStageDailyAdmission`.
  Every other path still goes through `isRelationStageExcludedPath`.
- **Buffered and scanned notes are classified exactly.** Use the snapshot's `lines` with
  `relationStageEventsSectionRange`; the snapshot already skips fenced code and
  frontmatter rows.
- **Tasks cache records use the record's heading.** Admit a record when its `section`
  (Tasks' `precedingHeader`) is `Events` and its `rawLine` (`originalMarkdown`) passes
  the span test.
  - Bob writes `## Events` with no sub-headings, so in practice this matches the exact
    range.
  - Buffered notes still win per note, as they do today.

> [!decision] daily_scope = all Admit every task row in a daily note. Event Tasks still
> get the event name and `eventTask: true`; all other rows keep their current text.

> [!decision] day_window = recent Also drop rows whose `noteDay` is before today − 30
> days. Today and future days always pass. Pass `today` through the collector options so
> tests can pin it.

- Admitted daily rows keep `closed` from their status and are never `archived`.
- `poolSize` and the subtitle `searching P tasks` count the admitted rows.

### 3. Ranking and the empty filter (`304` / `290`)

**Typed query.** Score still leads. Give `compareDependencyStageCanonical` an opt-in
option, for example `dailyNewestFirst`, that only `compareRelationStageCanonical`
passes. Its sort key becomes:

`(owner note, #hide, lane, isDailyNote, isDailyNote ? noteDay descending : path, line)`

- This keeps a strict total order. A daily row is never compared to another row by path.
  Path order would put `2026/…` ahead of letters and the oldest day first.
- Within a status lane, non-daily rows therefore come first. Admitted daily rows follow
  with the newest day first.
- Today's `[*]` meeting still sits in the Next lane and beats a closed mail task of
  equal score.

**Empty query.** Admitted daily rows join only through the two existing rules: the
owner's note, and the In Progress / Next statuses. The closed or archived rule never
admits them, so past meetings never flood CLOSED.

- Today's `[*]` meeting shows.
- Yesterday's `[x]` call needs typing.

### 4. Linked from stays editable sources only

`planLinkedFromStageView` skips rows with `dailyNote: true`. This matches capture `~~`,
which refuses daily-note sources (LX7).

- Flipping with `~` drops those rows.
- Linked-from behavior is otherwise unchanged.

### 5. One name everywhere in the stages

Every Links / Linked-from surface that names an Event Task uses
`relationStageEventTaskName`:

- RESULTS rows;
- CURRENT rows on both stages (`resolveRelationStageCurrent`,
  `relationStageIncomingRowsFromEdges`);
- the stage title when the owner itself is an Event Task;
- the link notice name.

The day is already visible: the subtitle's note label reads `20261123`.

Because `text` is now the clean name, gcal URLs no longer feed subsequence matches.
`route` (`2026/20261123`) and `section` (`Events`) stay searchable, so typing
`events derrick` narrows the list to meetings.

### 6. Previous daily note targets

Pass `previousDailyPath` into the Links-to stage. Compute it in
`buildVaultRelationStageFromNotes` the way `decorateLinkedFromVaultStage` already does
(`scheduledRecoveryDailyPaths`), then pass it on to `planRelationStageView`.

- **Linking works; the mirror waits.** An Event Task in the previous daily note links
  normally, and its mirror is skipped as today.
  - The notice suffix for this case changes from `· archived: shown live only` to
    `· previous daily note: 🪢 LINKED FROM follows after rollover`.
  - Reconcile writes the mirror once that note stops being the previous one.
  - `done/` targets keep the archived suffix, and a batch with both kinds shows both
    suffixes.
- **Id-less rows there are disabled.** An Event Task in the previous daily note that has
  no block id is disabled with `previous daily note is read-only`, because the block-ID
  prompt would write there. Id-less Event Tasks in other daily notes keep the usual
  `＋ id` prompt.

### 7. Chip names (bob-ledger-tools)

> [!decision] chip_names = clean In `src/146-relation-chip-model.js`,
> `relationFullTextFor` checks for a canonical daily-note path whose line is an Event
> Task line. In that case it returns
> `timeRange + " " + eventRunCleanTitle(text after the span)`, reusing the existing twin
> in `035-meet-event-run.js`. This covers both LINKS and LINKED FROM chips, and the
> 40-character cut still applies. Under `leave`, skip this section.

### 8. Contract and docs

**bob-cli `docs/task-relations.md` §15 LG table.** Add these vectors, and keep the LG24
wording style:

| Vector | Situation                                                         | Expected                                                                                                  |
| ------ | ----------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------- |
| LG25   | A Links query matches an Event Task in a daily note               | It ranks in RESULTS as `HHMM-HHMM Title`, and `↵` links it with the cross-note mirror follow-up           |
| LG26   | A daily-note task that is not an Event Task (`^gtd`, ledger rows) | Never listed (under `daily_scope = events`)                                                               |
| LG27   | `↵` on an Event Task in the previous daily note                   | Linked with no mirror, plus the new suffix; an id-less row is disabled                                    |
| LG28   | Empty filter or equal scores                                      | Closed Event Tasks outside the owner's note are not listed; equal-score Event Tasks list newest day first |
| LG29   | Linked-from stage                                                 | Never offers a daily-note source                                                                          |

The doc's own rule requires changing it in the same change as the implementation.

**bob-plugins `docs/task-relations.md` ("Linking from the Task Card").** RESULTS now
also covers Event Tasks from daily notes. Describe:

- the `HHMM-HHMM Title` name;
- newest-first ranking;
- the `events` search;
- the previous-daily-note suffix;
- under `chip_names = clean`, a chip-name bullet.

Then run `npm run docs:gen` if the inventory changes.

**Code comment.** Fix the stale comment above `isRelationStageExcludedPath`.

**Do not edit** bob-cli `docs/reference/plugins/*`. It is a git-ignored mount of the
bob-plugins docs.

## Tests (bob-plugins)

### Links-stage suite

Extend `scripts/test-navigation-relation-links-stage.cjs`, citing LG ids.

**Pool admission:**

- **Kept:**
  - The Derrick `## Events` row joins with `displayText` set to
    `1500-1530 Bryan Bugyi and Derrick Blakely`.
  - It carries `eventTask: true` and `noteDay: "2026-11-23"`.
- **Dropped:**
  - a span-less row under `## Events`;
  - a span row under `## Pomodoros`;
  - a `^gtd` row before any heading;
  - a fenced row inside Events;
  - a row after the next `##` heading;
  - `_templates`.
- **Unchanged:** `isRelationStageExcludedPath("2026/20261020.md")` still returns `true`.

**Cache merge:**

- A record with `precedingHeader: "Events"` and a span-led `originalMarkdown` is
  admitted.
- The same record under another heading is dropped.
- A buffered copy wins over the cache record.

**Ranking:**

- `derrick blake` ranks the meeting first, above the closed mail task.
- The same title on three days ranks newest day first.
- An equal-score non-daily row precedes a daily row in the same lane.
- Sorting a shuffled mixed set (including `2026/2026-Q1.md` and `2025/…` rows) twice
  gives the same order.

**Empty filter:**

- Today's `[*]` meeting is listed.
- An `[x]` meeting from another day is not listed.
- When the owner sits in that daily note, its same-note meetings are listed.

**Linked from:** `planLinkedFromStageView` never lists a daily row.

**`day_window = recent` only:** a 31-day-old meeting is dropped and a 30-day-old one is
kept.

**Name twin:**

- off-clock tag;
- interleaved `[buffer::]` fields and links;
- trailing ` · *…*`;
- non-event lines return `null`.

### Commit, chips, and exports

**Commit** (writer or links-stage suite):

- A meeting in today's note gets its cross-note LINKED FROM follow-up.
- A previous-daily meeting gets no mirror and the new suffix.
- An id-less previous-daily meeting is disabled.

**Chips (`chip_names = clean`):** in `scripts/test-ledger-tools-relation-chips.cjs`,
cover the text of a chip whose target or source is an Event Task.

**Exports:** export the new helpers from a new
`src/732-relation-event-target-exports.js`. `700-exports.js` is at 999 lines. Follow the
pattern in `731-project-seed-reqs-exports.js` and add the file to `fragments.json`.

## Verification

**bob-plugins:**

- `npm run build`;
- `node --test` on each touched suite;
- `npm test`;
- `npm run docs:check`;
- then `bob plugins sync`. If sync refuses an overwrite, follow `AGENTS.md`; never pass
  `-F` on your own.

**bob-cli:** `just docs-check`.

**Manual check in Obsidian:**

1. On the screencast's job-note task, press `Ctrl+Shift+L` and type `derrick blake`.
2. The first row reads `1500-1530 Bryan Bugyi and Derrick Blakely` with note `20261123`.
3. `↵` links it, and the meeting row gains `🪢 LINKED FROM` (or reconcile adds it).

**Commits:** commit bob-plugins as `feat(nav): …`. Commit bob-cli as
`docs(task-relations): …`.

## Non-goals

- The Depends on / Blocks stages keep excluding daily notes.
- No daily-note sources appear in Linked from, and capture `~~` is unchanged.
- No bob-cli code, Mac Capture, or relation-index changes.
- No memory note becomes false. The glossary's Event Task and Task Relation entries
  still hold.
