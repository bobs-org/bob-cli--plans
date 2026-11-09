---
tier: epic
title: Make bob gcal sync race-proof, resettable with --force, and explicit about
  skips
goal: A daily-note creation race can never again erase synced Event Tasks for good,
  `bob gcal sync --force` rewrites a day from Google as on its first sync, and every
  Google event sync leaves out is listed with a specific reason and fix.
phases:
- id: templater_guard
  title: Refuse to sync into a note Templater has not rendered
  size: small
  depends_on: []
  description: 'templater_guard: bob-cli. Detect pending Templater tags in the daily
    note and return a structured `waiting` (note_missing / pomodoros_missing / note_unrendered)
    instead of writing, including on the locked re-read. See plan section "Phase templater_guard".'
- id: ledger_tools_readiness
  title: Start the create hook only after Templater finishes
  size: small
  depends_on: []
  description: 'ledger_tools_readiness: bob-plugins. Ledger Tools waits for a rendered,
    settled daily note before spawning `bob gcal sync` (and Gmail), calls `--format
    json`, accepts the `held` status, and surfaces waiting/held notices. See plan
    section "Phase ledger_tools_readiness".'
- id: skip_reasons
  title: Name every skipped event and stop re-tombstoning
  size: medium
  depends_on:
  - templater_guard
  description: 'skip_reasons: bob-cli. One shared skip vocabulary for sync and list;
    sync lists every left-out event with reason and hint, reports in-sync counts,
    stops re-tombstoning gone rows every run, and revives a tombstoned row pasted
    back. See plan section "Phase skip_reasons".'
- id: force_reset
  title: Add the --force day reset to bob gcal sync
  size: medium
  depends_on:
  - skip_reasons
  - ledger_tools_readiness
  description: 'force_reset: bob-cli. New force flag, `gcal.reset` journal record
    and fold, a planner reset that restores missing rows with their old block ids
    and re-applies Google over present rows, beautiful reset output, docs, and tests.
    See plan section "Phase force_reset".'
- id: replaced_hold
  title: Hold a replaced Events section and record the deletion policy
  size: small
  depends_on:
  - force_reset
  description: 'replaced_hold: bob-cli. When every row Bob wrote for the day and the
    Events stamp vanish together, do what the replaced_events decision says (default:
    hold with status `held`), update list/doctor/docs, and write the decision records.
    See plan section "Phase replaced_hold".'
decisions:
  force_edits:
    ask: Should `-F` also overwrite your edits to rows still in the note?
    choices:
      reset: Present rows get Google's time, title, badges, and root back; status,
        children, history stay
      keep: Present rows stay exactly as they are; `-F` only restores missing rows
        and links
    default: reset
    why: You asked -F to disregard edits; status, children, and sessions stay safe
      either way
    answer: reset
  replaced_events:
    ask: When all of today's Bob rows and the Events stamp vanish at once, should
      sync hold, not tombstone?
    choices:
      hold: Sync writes nothing that day and says the note was replaced; `bob gcal
        sync -F` restores it
      restore: Sync re-adds the lost rows on its next run; a deliberately wiped section
        comes back once
      tombstone: Keep today's rule; each missing row counts as your deletion and only
        `-F` restores it
    default: hold
    why: Never re-adds without consent, yet never mistakes a clobbered note for your
      deletions
    answer: hold
  force_short:
    ask: Should force use bob's usual `-F|--force` rather than `-f`, which sync already
      uses for `--format`?
    choices:
      capital_f: '`-F|--force`; `-f` stays `--format` as on every gcal command and
        in the create hook'
      lowercase_f: '`-f|--force` as asked; sync''s `--format` moves to `-F`, a `!`
        break for `-f json` callers'
    default: capital_f
    why: bob spells force `-F` wherever `-f` is `--format` (plugins sync, periods,
      freshness)
    answer: capital_f
  memory_force_record:
    ask: Add the decision record `decisions:gcal-force-resets-the-day` for the reset
      and replaced-section rule?
    memory:
    - decisions:gcal-force-resets-the-day
    default: true
    requested: As a part of this change, let's add a new `-f|--force` option to the
      `bob gcal sync` command that disregards any previous edits/deletions and re-writes
      the appropriate event tasks and pomodoros to today's daily file as if it were
      the first time that command were run for that day.
    answer: true
  memory_propose_once_mark:
    ask: Mark `decisions:bob-gcal-is-read-only-and-proposes-once` partly superseded
      by the new record?
    memory:
    - decisions:bob-gcal-is-read-only-and-proposes-once
    default: true
    corrects: deleted rows are tombstoned and never re-added
    answer: true
proposed_by: bbugyi200.athena.1aa
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-09 07:12:08
status: wip
bead_id: bob-cli-iz
---

- **PROMPT:** [prompts/202611/gcal_sync_race_force_and_skip_reasons.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/gcal_sync_race_force_and_skip_reasons.md)
- **BEAD:** [bob-cli-iz](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-iz/README.md)

# Problem and outcome

On 2026-11-09 `bob gcal sync` wrote today's "Haircut" Event Task, its description
bullet, and its MEET link at 05:24:20. Seven seconds later Templater rendered the new
daily note and replaced the whole file with what it had read before the sync, so the
Events section disappeared. The next sync found the journaled row missing, recorded a
`line_removed` tombstone, and has skipped the event ever since. `bob gcal list` shows
"removed by you". The full analysis is in
`research:202611/gcal_haircut_lost_to_daily_note_race.md`; read it with
`sase artifact read`.

Three code facts cause the loss:

- Ledger Tools (`plugins/bob-ledger-tools/src/291-plugin-gcal-events.js`,
  `gcalEventsWaitForPomodoros`) counts the note as ready as soon as it contains
  `## Pomodoros`. Core Daily Notes copies the **raw** template, `<% … %>` tags and
  `## Pomodoros` included, before Templater renders it.
- `bob gcal sync` (`src/native/gcal/sync.rs`) checks only for the note and its Pomodoros
  section, so it writes into an unrendered note that Templater is about to overwrite.
- The planner (`src/native/gcal/plan.rs`, journaled branch of `plan_with_clock`) reads
  any missing journaled row as your deletion. Tombstones are permanent, nothing can undo
  one, and the run reports a title-less `- task removed by you`. Because the fold keeps
  `row` next to `gone`, every later run tombstones the row again: another `gcal.gone`
  line, status `written`, and the same opaque line.

After this epic:

1. The race is closed from both sides. The hook waits for a rendered, settled note. Sync
   itself refuses any note that still has Templater tags and reports `waiting` with the
   reason.
2. `bob gcal sync -F` resets one day, as you asked: it forgets the day's deletions, link
   removals, and edits, then rewrites the day's Event Tasks and MEET filing from Google
   as if this were the first sync.
3. Every sync names every event it leaves out, with a specific reason and, when you can
   act, the exact fix. `bob gcal list` uses the same words.
4. If any writer ever replaces the Events section again, sync detects it and follows the
   `replaced_events` decision. By default it holds rather than tombstoning.

# Design

## Defense in depth

| Layer                 | Where                    | Stops                                                                     |
| --------------------- | ------------------------ | ------------------------------------------------------------------------- |
| Hook readiness        | Ledger Tools             | The hook starting sync before Templater renders                           |
| Unrendered-note guard | `bob gcal sync`          | Any run (hook, manual, cron) writing into a note Templater will overwrite |
| Replaced-section hold | `bob gcal sync`          | Any other whole-note clobber turning into permanent tombstones            |
| `-F` day reset        | `bob gcal sync`          | Being stuck: one command recovers a damaged day                           |
| Skip reasons          | `sync` and `list` output | Silent omissions: every left-out event says why                           |

The guard alone closes the Templater race. Templater reads the raw file and later writes
the rendered one, so a sync that sees the raw file stops, and a sync that sees the
rendered file is safe. The hook change keeps the one-shot create hook from firing too
early and then giving up.

## Shared skip vocabulary

Add one module, `src/native/gcal/skip.rs`, that both `sync` and `list` use. A
`SkipReason` has three parts: a stable JSON `reason` code, a human `detail`, and an
optional `hint`. Precedence: an omission (the Google side) wins over a tombstone, which
wins over a hold.

| `reason`            | Source                                                               | `detail` (human)                                                 | `hint`                                                  |
| ------------------- | -------------------------------------------------------------------- | ---------------------------------------------------------------- | ------------------------------------------------------- |
| `declined`          | `Omitted("declined")`                                                | declined in Google Calendar                                      | —                                                       |
| `cancelled`         | `Omitted("cancelled")`                                               | cancelled in Google Calendar                                     | —                                                       |
| `awaiting_reply`    | `Omitted("reply in Calendar")`                                       | awaiting your reply                                              | RSVP in Google Calendar to add it                       |
| `event_type`        | `Omitted("event type: X")`                                           | Google `X` events are not synced                                 | —                                                       |
| `outside_day`       | `Omitted("does not cover/overlap this day", "timed events belong…")` | belongs to `Sun 2026-11-08`'s note (or: does not cover this day) | `bob gcal sync -D <day>` when it belongs to another day |
| `invalid_time`      | `Omitted("invalid …", "end is before start")`                        | Google sent an invalid time range                                | —                                                       |
| `removed_by_you`    | journal `gone` with `line_removed`                                   | removed by you at `05:28`                                        | restore with `bob gcal sync -F`                         |
| `removed_in_google` | journal `gone` with a lifecycle reason                               | the existing reason text                                         | —                                                       |
| `events_replaced`   | the replaced-section hold                                            | lost when the Events section was replaced                        | restore with `bob gcal sync -F`                         |

Waiting reasons are day-level, not per-event. They use the same type with codes
`note_missing`, `pomodoros_missing`, and `note_unrendered`.

Keep the classify reason strings unchanged. `list -f json` still prints the raw
`reason`. The friendlier detail is added beside it.

## Output design

Rows keep the current layout, `  {sigil} {kind:<7} {time}  {title}  {extras}`. Skipped
events use the kind `skipped`, which is exactly 7 characters. Colour follows `Styler`:
actionable skips (`removed_by_you`, `awaiting_reply`, `events_replaced`) use a yellow
`⊘`, by-design omissions use a dim `⊘`, the detail is dim, and any command in a hint is
cyan. Skipped rows sort by time and come after the change rows and MEET lines. The
summary line stays the last result line, followed by the stamp echo. `-q` still prints
nothing on success, no-op, waiting, or held. Warnings stay on stderr.

A normal run:

```text
bob gcal sync · 2026/20261109.md
  + task    16:00–17:00 −15m  Haircut (The Wolves Den)  ● body ≡ 1
  + MEET    () — MEET · linked 1
  ⊘ skipped 09:00–09:30  Team sync  declined in Google Calendar
  ⊘ skipped 12:00–12:30  Lunch with Sam  awaiting your reply · RSVP in Google Calendar to add it
✓ Events +1 task · Pomodoros +1 entry, +1 link · 3 in sync · 2 skipped
↻ synced 05:49 · you@example.com
```

An unchanged run that answers "where is my event?":

```text
bob gcal sync · 2026/20261109.md
  ⊘ skipped 16:00–17:00  Haircut (The Wolves Den)  removed by you at 05:28 · restore with bob gcal sync -F
✓ Events unchanged · Pomodoros unchanged · 3 in sync · 1 skipped
```

Waiting keeps the `waiting for <path>` prefix that the docs and tests already use:

```text
bob gcal sync · 2026/20261109.md
waiting for 2026/20261109.md · Templater hasn't finished rendering it (line 3 still has <% … %>)
  hint: nothing was written; rerun bob gcal sync once the note is rendered
```

A reset (`-F`) run:

```text
bob gcal sync · 2026/20261109.md · reset
  ↺ task    16:00–17:00 −15m  Haircut (The Wolves Den)  ● body ≡ 1  restored
  ~ task    09:00–09:30  Standup  ● work  reset your title edit
  + MEET    () — MEET · linked 1
✓ Events reset · ↺1 restored, ~1 reset · Pomodoros +1 link · 2 in sync · 1 skipped
↻ synced 06:02 · you@example.com
```

JSON changes are additive within `schema_version: 1`. Each phase lists the fields it
adds.

# Phase templater_guard

**Repo:** bob-cli. **Depends on:** nothing.

1. Add a pure detector, for example `src/native/templater.rs`, exposing
   `first_pending_tag(contents) -> Option<PendingTag { line: usize /* 1-based */, excerpt: String }>`.
   A pending tag is a `<%` opening delimiter, which covers `<%*`, `<%+`, `<%-`, and
   `<%_`. Scan everything, YAML frontmatter included, but skip fenced code blocks
   (backtick and tilde fences, matching the closing fence like `gcalEventsFindSection`
   does) and inline code spans. Trim the excerpt to about 60 characters. Keep this
   module generic and outside `gcal/` so other daily-note writers can reuse it.
2. In `sync.rs`, replace the bare `waiting(...)` calls with a typed `WaitingReason`, and
   check in this order: note missing → `note_missing`; pending tag → `note_unrendered`
   (with line and excerpt); no `## Pomodoros` → `pomodoros_missing`. The tag check must
   come before the Pomodoros check, because the raw template already has Pomodoros. Run
   the same guard on the locked re-read after a `NoteChanged` retry. If that re-read is
   unrendered, return `waiting` and write nothing. `-d` (dry run) gets the same waiting
   report.
3. Output. The human form is `waiting for <path> · <detail>` plus a dim `hint:` line,
   using the per-reason text from the design. JSON adds a top-level `waiting`, either
   `{"reason", "detail", "hint", "line"}` or `null`. `status` stays `"waiting"`.
   `record_status` writes an additive `reason` into `status.json`.
4. `bob gcal list`: when the note is unrendered, the header reads
   `waiting for Templater` and the VAULT cells read `waiting for Templater` instead of
   showing Bob state. `bob gcal doctor`'s `last_sync` check shows the waiting reason
   when present.
5. Docs: in `docs/gcal.md`, update § Sync (the waiting sentence and the reasons), the §
   Exit status `0` bullet, and § JSON output. Update the clap `long_about` for `sync`.
6. Tests. Detector unit tests: a frontmatter tag, a body tag, `<%*` variants, a fenced
   block ignored, inline code ignored, and a rendered note returning none. Integration
   tests in `tests/gcal_sync.rs`: a raw-template note with Pomodoros is left
   byte-identical, the journal is unchanged, the JSON reason and line are correct, and
   once rendered the same fake snapshot writes normally. Also cover the retry path if
   the existing harness can simulate a concurrent edit; otherwise unit-test the guard on
   the re-read helper.

Done when `cargo test` passes for the gcal suites, `just docs-check` passes, and
`bob gcal sync -d` on a raw-template note prints the waiting reason.

# Phase ledger_tools_readiness

**Repo:** bob-plugins. Open it with `/sase_repo` and read its `AGENTS.md`. **Depends
on:** nothing.

1. Replace the readiness predicate with `gcalEventsNoteReady(contents)`. It requires a
   `## Pomodoros` section and no pending Templater tag, using the same rule as the bob
   detector: `<%` outside fenced and inline code, frontmatter included. Rename the
   waiter to `gcalEventsWaitForRenderedNote` and keep `gcalEventsWaitForPomodoros` as an
   export alias so existing callers and tests keep working.
2. Settle: after the first ready read, require one more identical ready read at least
   500 ms later before returning ready. Raise the bound from 15 s to 60 s, because the
   2026-11-09 Templater render came at least 7 s late. Update the Gmail hook's "within
   15 seconds" log text to match.
3. On timeout, the calendar hook still spawns sync once, because bob's guard is the
   authority. If bob reports `waiting` with `waiting.reason == "note_unrendered"`, show
   a Notice once per day: "Calendar sync is waiting for Templater to render today's note
   · run `bob gcal sync` when it's ready". The Gmail hook keeps its current skip on
   timeout.
4. Spawn `["gcal", "sync", "-q", "--format", "json"]`. The long form works with every
   bob version and with either `force_short` answer.
5. `gcalEventsDecodeSync` (`291c-plugin-gcal-event-clock.js`) also accepts
   `status: "held"`. On `held`, show a Notice once per day: "Calendar sync held: today's
   Events section was replaced · run `bob gcal sync -F`". Unknown future fields are
   ignored.
6. Tests: extend `scripts/test-ledger-tools-gcal-events.cjs` to cover a raw template
   (tags plus Pomodoros) as not ready, a rendered note as ready, tags in code fences
   ignored, settle with an injected clock and delay, the timeout path still spawning,
   the decoder accepting `held`, and the exact spawn args. Run `npm run build`, the
   relevant `node --test` files, `npm run build:check`, and `npm run docs:check`, then
   update `docs/bob-ledger-tools.md` where it describes the create hook. Finish with
   `bob plugins sync` to deploy to the vault.

# Phase skip_reasons

**Repo:** bob-cli. **Depends on:** templater_guard.

1. Journal fold (`journal.rs`): `OccurrenceState` gains `gone_at` (the first tombstone's
   timestamp) and `gone_reason`. A `gcal.row` folded **after** a `gcal.gone` for the
   same key and day revives the occurrence by clearing `gone`, `gone_at`, and
   `gone_reason`. Add unit tests for both orders.
2. Planner (`plan.rs`):
   - A gone occurrence whose line is still missing is skipped. It gets no new
     `rows_gone`, no record, and no report row, so a rerun after a tombstone is
     `unchanged` and appends nothing.
   - A gone occurrence whose line is back (matched by block id, then URL or text) is
     adopted again (`rows_adopted`, with `link_adopted` from the note). Its `gcal.row`
     revives it, so MEET-link eligibility and `list` return to normal.
   - `lifecycle` already filters gone rows. Keep that.
   - `GoneRow` carries the occurrence's time label and title so the tombstoning run can
     report
     `- task    14:00–15:00  Design review  removed by you · restore with bob gcal sync -F`.
3. Skip vocabulary: implement `src/native/gcal/skip.rs` as in the design table. Build
   the day's skipped list from **every** snapshot event, omitted ones included:
   omissions first, then gone/removed occurrences. Count `in_sync` as included
   occurrences that are present in the note and appear in no change row.
4. `SyncReport` gains
   `skipped: Vec<SkippedEvent { time, title, calendar_id, event_id, reason: SkipReason }>`
   and `in_sync: usize`. Render them as in the design: skipped rows after the changes
   and MEET lines, and `· N in sync · M skipped` appended to every summary form
   (`unchanged`, sort-only, refile-only, stamp-only, and the general one). Drop zero
   counts so a quiet day stays short.
5. JSON adds `in_sync`, a `skipped` array of objects
   `{time, title, calendar_id, event_id, reason, detail, hint}`, and a `detail` field
   (string or null) on every `rows` object. For removed rows, `detail` is
   `removed by you` or the lifecycle reason.
6. `bob gcal list`: compute VAULT text from `skip.rs`. A tombstoned row reads
   `removed by you 05:28`. Add one footer line when any row is removed:
   `N removed by you · bob gcal sync -F restores today's Events`. List JSON events gain
   a `skip` object `{reason, detail, hint}` or `null`. Keep the existing `vault` strings
   and `reason` values.
7. Docs: update § Sync (output, skipped list, rerun behaviour), § List, § JSON output,
   and the "The whole row" row of § Who owns each part of a row. Pasting a row back now
   revives it; `-F` arrives in the next phase.
8. Tests:
   - `deleted_row_and_removed_link_are_never_readded`, plus a new test that runs sync a
     third time and asserts no new `gcal.gone`, status `unchanged`, and a skipped entry
     with `removed_by_you` and the time.
   - A paste-back revive test: the row is managed again, a later Google title change
     applies, and `list` shows `task`.
   - Fixture events covering declined, no-reply, `event type`, and outside-day, each in
     `skipped` with its code.
   - Human output snapshots for the summary forms.

# Phase force_reset

**Repo:** bob-cli. **Depends on:** skip_reasons and ledger_tools_readiness, so the hook
already uses `--format json`.

> [!decision] force_short = capital_f Add `Arg::new("force").long("force").short('F')`.
> Sync options stay alphabetical by long name in help. **If `lowercase_f`:** add
> `.short('f')` for force and move sync's `--format` short alias to `-F` (sync only).
> Before committing, run `rg -n "gcal sync"` over the bob-plugins, bob-mac-capture, and
> chezmoi checkouts, opened with `/sase_repo` (use the `/crontab` skill for cron
> entries). Switch every `-f json` caller to `--format json`. Mark the commit
> `feat(gcal)!:`.

1. CLI (`cli.rs`):
   - Add `SyncArgs.force`. Help: "Reset this day: forget your deletions and edits and
     rewrite its Events and MEET links from Google, as on a first sync".
   - `sync_after_help` examples gain `bob gcal sync -F` and `bob gcal sync -F -d`
     (preview a reset). The top-level `bob gcal` examples gain
     `bob gcal sync -F    Rewrite today's Events from Google as on a first sync`.
   - Update `long_about`.
   - Extend `sync_args_pin_options`.
   - Run `just docs-gen`.
2. Journal:
   - Add `append_reset(vault, host, day)`, which writes
     `{"v":1,"ts":…,"host":…,"kind":"gcal.reset","day":"YYYY-MM-DD","reason":"force"}`.
   - Fold `gcal.reset` in writer order: drop every `(key, day)` entry accumulated so far
     for that day. Records after it apply normally, and series roots are untouched.
   - The key-less record must not count as malformed: restructure `parse_record` so the
     kind is checked first, and update `malformed_count` for `doctor`.
   - Older binaries warn about one unknown line and ignore it. That is acceptable
     because the Mac is the single writer. Note it in docs § Known limits.
3. Planner reset. Add `SyncPlanInput.reset: Option<DayReset>`, built from the day's
   pre-reset state. Planning runs against a synthesized journal for the day:
   - The flags `gone`, `link`, and once-only notices are dropped.
   - A prior `row` record is kept only when its line is present in the note. Match by
     the prior block id first, then URL or text.
   - With `force_edits = reset`, the kept record's `spans` and `root` are replaced with
     the row's **current** span hashes and `[root::]`. The existing three-way merge
     (`apply_span_updates`, `root_update_action`) then sees the row as untouched and
     applies Google's time, title, badges, and root. The description record keeps its
     prior state, so bullets follow the normal rules: untouched bullets follow Google
     and edited ones stay yours. A clock flip on a worked row is still kept, as today.
   - With `force_edits = keep`, present rows go through crash-recovery adoption
     unchanged.
   - Present rows whose link `[[#^id]]` already sits in Pomodoros count as linked, so no
     second link is filed.
   - A missing row is added as on a first sync: fresh bullets, filing by phase, and a
     past row is born done. It **reuses its prior block id** when no other line carries
     that id as a trailing block id, so links elsewhere in the vault resolve again. If
     today's Pomodoros already link that id, that link is kept and no duplicate is
     filed. Harden `collect_wanted_links` so no sync, forced or not, ever files a second
     link to an id that Pomodoros already links. Report these rows with action
     `restored` when prior state existed, otherwise `added`.
   - Lifecycle runs against the synthesized journal, so a reset never cancels.
   - Every included occurrence present after the run ends with one current `gcal.row`,
     which makes an immediate normal rerun byte-identical with no appends.
4. `sync.rs`:
   - `-F` still respects the writer check, the account guard, and every waiting reason.
     It never creates a note, and it works with `-D` for any day whose note exists.
   - After the note write succeeds, append `gcal.reset` (only when the day had prior
     journal state), then the plan's records, in that order.
   - `-F -d` previews the same diff and appends nothing.
   - A force run whose note is already right still appends the reset when prior state
     existed. It reports status `written` and the summary
     `✓ Events reset · nothing to rewrite`.
5. Output, as shown in the design:
   - The header gets the suffix ` · reset`.
   - Sigil `↺` marks restored rows, and forced span resets show `reset your <span> edit`
     in `detail`.
   - The summary leads with `✓ Events reset`.
   - JSON adds top-level `force: bool`, the row action `restored`, and `detail` on
     updated rows.
   - `status.json` adds `forced: true`.
6. Docs (`docs/gcal.md`):
   - New § Sync › "Reset a day (`-F`)", with a table of what is forgotten (tombstones,
     removed links, once-only notices, your span and root edits per `force_edits`), what
     is kept (checkbox, block ids, child notes, `[type::]`, Pomodoro sessions, series
     memory), and the dry-run tip.
   - Update the "The whole row" ownership row to "tombstoned; only `bob gcal sync -F`
     re-adds it".
   - Update § Reconciling, § JSON output, and § Known limits.
   - Replace any hand-paste recovery advice with `-F`.
7. Tests:
   - The 2026-11-09 scenario end to end: sync, clobber the note back to the rendered
     template, sync (tombstone), `-F` (row restored under its old block id with its MEET
     link, a past row born done), then a normal rerun is unchanged.
   - An edited title or root on a present row is reset; checkbox, children, and 🍅
     history survive.
   - A removed link is re-filed.
   - A dangling link is reused without a duplicate.
   - `-F -d` writes and appends nothing.
   - `-F` waits on an unrendered note.
   - `-F` on another host is refused.
   - Fold unit tests: reset ordering, and records after the reset survive.

# Phase replaced_hold

**Repo:** bob-cli. **Depends on:** force_reset.

The trigger: not under `-F`, the day has at least one journaled, non-gone row, none of
those rows is found in the note, and the note carries no recognized Events stamp (or no
Events section at all).

> [!decision] replaced_events = hold Sync writes nothing that day: no note changes, no
> journal records, no new rows for other events. It returns status `held`. In human
> form:
> `held · today's Events section was replaced: the N rows Bob wrote at HH:MM and its ↻ stamp are gone`
> plus a hint line:
> `another writer (Templater, a sync conflict) likely replaced the note; nothing was tombstoned · bob gcal sync -F rewrites today's Events (then delete any rows you don't want)`.
> Every lost event appears in `skipped` with `events_replaced`. JSON adds
> `held: {rows, written_at, detail, hint}`. `status.json` records outcome `held`. `list`
> shows header `held · Events replaced` and VAULT `lost · run sync -F`. Doctor's
> `last_sync` warns. Mark the commit `feat(gcal)!:` because strict consumers reject the
> new status; Ledger Tools already accepts it. **If `restore`:** plan the lost
> occurrences as if unjournaled, re-adding them with their prior block ids exactly like
> the force restore (no reset record), report them as
> `↺ restored · Events section was replaced`, and add no new status. **If `tombstone`:**
> implement no detection; this phase only writes the decision record and docs.

1. Put detection in one helper shared by `sync` and `list`, with unit tests for each
   trigger condition. A deliberately emptied section that keeps the stamp is not
   replaced. A partial loss tombstones as usual.
2. Docs: add a new § "Replaced Events sections" next to § Reconciling, and update § Exit
   status, § JSON output, and § Doctor.
3. Tests: the clobber scenario with no `-F` (held, nothing written or appended, skipped
   reasons present); `-F` then clears the hold; a single deleted row with the stamp
   intact still tombstones.
4. Memory, following `/sase_memory_write` and each memory decision:
   - `memory_force_record`: add the new strand `decisions:gcal-force-resets-the-day`
     (title "A Forced Sync Resets The Day"; frontmatter shaped like its siblings:
     `origin: requested`, `scope: project`, aliases `gcal sync -F` and
     `gcal day reset`). **Applies to** bob-cli and the vault. The claim, adjusted to the
     accepted `replaced_events` answer: a journaled row missing from the note is your
     deletion and is tombstoned. A replaced Events section (every row and the stamp gone
     at once) holds instead. Only `bob gcal sync -F` re-adds: it resets the day's
     journal and rewrites the day from Google as on a first sync, following the accepted
     `force_edits` answer. **Why:** cite the Haircut race research and the rejected
     alternatives (auto-restore re-adding without consent, a permanent tombstone with
     only hand-paste recovery, and a sticky tombstone with no revive). **Cost:** a held
     day gets no new events until `-F`. **Reopens when:** Bob can tell an editor clobber
     from a deletion directly. **Evidence:**
     `research:202611/gcal_haircut_lost_to_daily_note_race.md`, this plan,
     `docs/gcal.md` § Sync › Reset a day, and `tests/gcal_sync.rs`.
   - `memory_propose_once_mark`: in
     `sase/memory/decisions/bob-gcal-is-read-only-and-proposes-once.md`, append
     `decisions:gcal-force-resets-the-day` to `metadata.superseded_by`, and add a
     back-link paragraph: "Superseded in part: tombstoned rows are no longer 'never
     re-added'; `bob gcal sync -F` re-adds them and a replaced Events section holds
     instead of tombstoning. See [[decisions:gcal-force-resets-the-day]]." Do not
     otherwise edit the accepted body.
   - Run `sase memory init` to regenerate `AGENTS.md` and the shims.

# Cross-cutting rules for every phase

- Read the `cli_rules.md` and `docs.md` reference memory before touching CLI help or
  docs. Every public long option needs a short alias, and options stay alphabetical.
- Follow `decisions:bob-cli-releases-ride-a-daily-train`: never edit the `Cargo.toml`
  version or `CHANGELOG.md`. Mark CLI or JSON contract breaks with `!`. Additive JSON
  fields are not breaks.
- Keep `plan.rs` pure (no I/O). Keep files under the repo's size norms; split helpers
  into sibling modules such as `skip.rs` and `reset.rs` rather than growing `plan.rs`
  and `sync.rs`.
- Run `cargo fmt`, `cargo clippy --all-targets`, the gcal test suites (`tests/gcal_*.rs`
  plus module unit tests), and `just docs-check` before declaring a phase done.
- Bob never creates daily notes, and the Google grant stays read-only.
- If `bob gmail sync` also writes into the daily note, file a follow-up task bead for
  reusing the Templater guard there. Do not widen this epic.
