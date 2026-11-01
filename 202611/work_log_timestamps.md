---
tier: epic
title: Timestamped Work Log entries
goal: 'Every new Work Log entry, from any of Bob''s four writers, opens with one italic
  `*YYYY-MM-DD HH:MM*` stamp of the minute it was written. Older date-only entries
  stay valid and untouched, and every reader handles both forms the same way.

  '
decisions:
  stamp_style:
    ask: How should the Work Log stamp read?
    choices:
      iso: '`*2026-11-01 14:32* — …`: matches BOB_NOW and gcal''s synced stamp; narrowest
        rows'
      weekday: '`*Sun 2026-11-01 14:32* — …`: adds the weekday for skimming; 4 chars
        wider per row'
    default: iso
    why: Bob already writes date-times as YYYY-MM-DD HH:MM, which sorts and greps
      as text
    answer: iso
  memory_decision_record:
    ask: Add a decisions record that Work Log entries carry a write-instant stamp?
    memory:
    - decisions
    default: false
    answer: false
  memory_glossary_work_log:
    ask: Mention the entry stamp in the Work Log glossary term?
    memory:
    - glossary:work-log
    default: false
    answer: false
phases:
- id: mac-strip
  title: Bob Mac Capture strips both stamp forms
  depends_on: []
  size: small
  description: 'mac-strip: make the close card''s Work Log stripper accept stamped
    and date-only entries strictly, with vector tests and the previews doc, before
    bob starts emitting stamps.'
- id: nav-stamp
  title: Navigation hotkeys read and write stamps
  depends_on: []
  size: medium
  description: 'nav-stamp: teach the lane-walk chip to read and show stamps, then
    stamp every bob-navigation-hotkeys Work Log writer and preview from one commit-time
    clock read per gesture.'
- id: cli-stamp
  title: bob capture close stamps entries and owns the grammar docs
  depends_on:
  - mac-strip
  - nav-stamp
  size: medium
  description: 'cli-stamp: stamp capture `x` close entries from the close''s own `now`,
    publish the canonical entry grammar in vault-format.md, refresh the capture/plan/projects
    docs, and mark the JSON shape change with `!`.'
- id: cycler-stamp
  title: Unlink prompt and Pomodoro checkbox close write stamps
  depends_on:
  - nav-stamp
  size: medium
  description: 'cycler-stamp: stamp block-id-prompt''s unlink Work Log and task-status-cycler''s
    Pomodoro close entries from one commit-time clock read, replacing the daily-note-filename
    date.'
proposed_by: bbugyi200.apollo.c3
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-01 15:12:22
status: done
bead_id: bob-cli-fd
---

- **PROMPT:** [prompts/202611/work_log_timestamps.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/work_log_timestamps.md)
- **BEAD:** [bob-cli-fd](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-fd/README.md)

# Plan: Timestamped Work Log entries

## Why and what exists today

A Work Log entry records only its day: `*2026-10-02* — Finished the API review`. With
several entries on one day, Bryan can't tell when each was written or how they relate to
that day's Pomodoros. Four independent implementations write the same shape. All of them
must change together, and two readers that parse the date back must accept the new form
before any writer emits it.

| Kind          | Where                                                                                                                                                           | Gesture                                                                                                                        |
| ------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------ |
| Writer (Rust) | bob-cli `src/native/capture_work_log.rs` (`write_work_log_group`), called from `src/native/capture_pomodoro_close/linked_tasks.rs`                              | capture `x` close (`=x`, typed `x` logs)                                                                                       |
| Writer (JS)   | bob-plugins `plugins/bob-navigation-hotkeys/src/220-property-targets.js` (`formatLaneWorkLogEntry`, `insertLaneWorkLogEntry`)                                   | Alt+N release, Move to Next, Ctrl+Shift+P scheduling (incl. Task Card), schedule review, Pending keeps, `^prj` subtree writers |
| Writer (JS)   | bob-plugins `plugins/block-id-prompt/src/040-editor-pomodoro-and-logs.js` (`formatWorkLogEntry`, `planWorkLogInsertion`)                                        | Ctrl+Shift+Enter unlink of an In Progress task                                                                                 |
| Writer (JS)   | bob-plugins `plugins/task-status-cycler/src/020-schedule-and-logs.js` (`formatWorkLogEntryBody`, `buildWorkLogEntryLines`, `planPomodoroWorkLogGroupInsertion`) | checking off a Pomodoro in Obsidian                                                                                            |
| Reader        | bob-plugins `plugins/bob-navigation-hotkeys/src/542-lane-walk-model.js` `readLatestWorkLog` (regex `\*(\d{4}-\d{2}-\d{2})\*`)                                   | `]s` lane-walk chip `🛠️ Oct 30 — summary`. **A stamped entry would make this chip silently disappear.**                        |
| Reader        | bob-mac-capture `Sources/CaptureCore/CapturePomodoroClosePresentation.swift` `strippedWorkLogDate` (requires exactly 10 chars)                                  | close card previews. **A stamped entry would show raw `*… 14:32* —` text.**                                                    |

Nothing else parses entry dates. Marker-only consumers (Rotten Log and Schedule Log
placement, drawers, project promotion and reversal, task moves, highlights snapshots,
the `work log N` agenda count) see only the `🛠️ **WORK LOG**` marker or move the subtree
byte for byte, so they need no change.

## Design

### The stamp

Writers emit exactly this, with the marker unchanged:

```markdown
- [/] #task Ship the importer ^ship
  - 🛠️ **WORK LOG**
    - _2026-11-01 14:32_ — Wired the stamp into capture close
    - _2026-11-01 09:50_ — Designed the grammar
      - chose minutes, not seconds
    - _2026-10-28_ — Entries written before this change keep their date
```

Why this shape:

- **Intuitive.** The row still reads "when — what": one italic lead, the same em dash,
  and the summary in normal weight. Old and new entries look like one log.
  `YYYY-MM-DD HH:MM` is the date-time text Bob already uses (`BOB_NOW`, and gcal's
  `↻ synced Thu 2026-10-29 21:40` stamp). It uses a 24-hour clock like every other Bob
  time, and it sorts and greps as plain text.
- **Beautiful.** It adds nothing but the time: no emoji, seconds, offset, or extra
  separator. The italic lead keeps the stamp quieter than the summary. `·` stays free
  for the Rotten Log's `· kept N×` and the notice separators.
- **Exact enough.** Bob's clocks run at minute grain (Pomodoro spans, gcal stamps), so
  seconds would only add noise.

> [!decision] stamp_style = weekday The stamp becomes `Ddd YYYY-MM-DD HH:MM`, with an
> English three-letter weekday from the same instant (`%a` in chrono; a fixed
> `["Sun",…,"Sat"]` table in JS, never locale-dependent). Writers emit
> `*Sun 2026-11-01 14:32* — …`. Readers accept an optional `Ddd ` before the date and
> ignore it for parsing. The lane-walk chip is unchanged (it already prints `Nov 1`),
> and the Mac stripper drops the whole stamp. Every vector below gains the weekday
> column. Everything else in this plan is the same.

### Semantics

Every phase implements these rules exactly:

1. **Write instant.** The stamp is when the entry was written into the task note: the
   gesture's commit (Enter in a prompt, the `bob capture` run, the Pomodoro checkbox).
   It is never the time a prompt opened, the session start, or the daily note's filename
   date.
2. **One instant per gesture.** Date and time come from a single clock read, formatted
   once. Every entry written by one gesture (counted Alt+N, a batch Ctrl+Shift+P, a
   multi-task close) carries a byte-identical stamp.
3. **Clocks stay where they are.** bob-cli uses `env::current_datetime()` (`BOB_NOW`,
   then `DATE`, then the configured `timezone`, then the host zone). Plugins use the
   host's local time, as they already do for every date stamp they write. Adding
   timezone awareness to the plugins is out of scope.
4. **Previews are live.** Prompt previews format the clock at render time with the same
   formatter, and they already re-render on input. The written stamp is read again at
   commit, so a preview left open across a minute boundary may show the previous minute
   until the next keystroke. It can never pair the wrong date with a time.
5. **A deliberate behavior change.** Pomodoro close entries (bob `x` and
   task-status-cycler) used to take their date from the daily note's filename. They now
   take date and time from the write instant. In normal use the day is the same. Closing
   yesterday's forgotten session this morning now stamps this morning. Under the old
   rule it would have joined yesterday's date to today's time, an instant that never
   happened.
6. **History is immutable.** Legacy `*YYYY-MM-DD* —` entries are never backfilled,
   rewritten, or flagged. Their times are unknowable. Moves, promotions, and reversals
   already carry Work Log subtrees verbatim.
7. **Strict writers, tolerant readers.** Writers emit only the stamp above. Readers
   accept exactly two forms: stamped and legacy date-only, both inside `*…*`. They
   reject seconds, `T` separators, one-digit hours, and offsets.
8. **Work Log only.** Rotten, Schedule, and Cancel Log entries keep their date-only
   grammar. Those dates are day-grained facts with their own parsers and keep-counting
   rules.
9. **DST.** Stamps use wall-clock time. During the repeated fall-back hour, stamps can
   repeat or go backwards. Entry order (newest first, by insertion) stays the authority.

### Conformance vectors

Each phase copies the vectors for its own implementation into its tests verbatim. These
tables are the parity contract between the Rust, JS, and Swift copies, the same way
`capture_schedule_log.rs` mirrors the picker's Schedule Log bytes.

**Stamp formatter** (instant in the writer's clock → stamp; truncate seconds, never
round; zero-pad everything):

| Instant             | Stamp (`iso`)      | Stamp (`weekday`)      |
| ------------------- | ------------------ | ---------------------- |
| 2026-11-01 14:32:59 | `2026-11-01 14:32` | `Sun 2026-11-01 14:32` |
| 2026-11-01 09:05:00 | `2026-11-01 09:05` | `Sun 2026-11-01 09:05` |
| 2026-12-31 23:59:59 | `2026-12-31 23:59` | `Thu 2026-12-31 23:59` |
| 2027-01-01 00:00:00 | `2027-01-01 00:00` | `Fri 2027-01-01 00:00` |

**Entry body**: summary `Finished the API review` with stamp `2026-11-01 14:32` →
`*2026-11-01 14:32* — Finished the API review`. Each writer keeps its existing summary
normalization: the plugins collapse whitespace, and Rust trims.

**Reader** (newest entry line → parsed):

| Line                                                 | Result                                                             |
| ---------------------------------------------------- | ------------------------------------------------------------------ |
| `\t\t- *2026-11-01 14:32* — Finished the API review` | date `2026-11-01`, time `14:32`, summary `Finished the API review` |
| `\t\t- *2026-10-28* — Older entry`                   | date `2026-10-28`, time none, summary `Older entry`                |
| `\t\t* *2026-11-01 14:32* — Starred bullet`          | parsed (any list marker, as today)                                 |
| `\t\t- *2026-11-01 9:05* — x`                        | rejected                                                           |
| `\t\t- *2026-11-01 14:32:10* — x`                    | rejected                                                           |
| `\t\t- *2026-11-01T14:32* — x`                       | rejected                                                           |

**Mac stripper** (bob JSON string → card text):

| Input                                       | Output                 |
| ------------------------------------------- | ---------------------- |
| `*2026-11-01 14:32* — Designed the grammar` | `Designed the grammar` |
| `*2026-09-28* — Designed the grammar`       | `Designed the grammar` |
| `*2026-11-01 14:32:10* — x`                 | unchanged              |
| `*soon* — x`                                | unchanged              |

**Lane-walk chip** (newest entry → chip/text fragment):

| Newest entry                                   | Fragment                                   |
| ---------------------------------------------- | ------------------------------------------ |
| `*2026-11-01 14:32* — Finished the API review` | `🛠️ Nov 1 14:32 — Finished the API review` |
| `*2026-10-28* — Older entry`                   | `🛠️ Oct 28 — Older entry`                  |

### Phase graph

Readers go first and writers follow, so no window ever shows a stamped entry to a reader
that can't parse it. `mac-strip` and `nav-stamp` run in parallel. `cli-stamp` waits for
both readers. `cycler-stamp` waits for `nav-stamp`, which owns the lane-walk reader, and
then runs alongside `cli-stamp`. Plugin phases deploy only their own plugins
(`bob plugins sync --repo <checkout> -p <id>`, once per plugin id), so they never roll
back each other's deploy. bob and the Mac app install independently. An older app with a
newer bob only shows the raw stamp in the card, a cosmetic problem that never breaks
decoding.

## Phase mac-strip: Bob Mac Capture strips both stamp forms

Work in the bob-mac-capture linked repo. Open it with `/sase_repo` and read its
`AGENTS.md` first.

- In `Sources/CaptureCore/CapturePomodoroClosePresentation.swift`, rename
  `strippedWorkLogDate` to `strippedWorkLogStamp` and update its three call sites (typed
  previews, the matched loop, and the other previews). Strip only when the text between
  the leading `*` and `* — ` matches `YYYY-MM-DD` or `YYYY-MM-DD HH:MM` exactly,
  checking digit positions. Leave every other entry unchanged. Update the doc comment to
  say Work Log entries arrive stamped (`*2026-11-01 14:32* — …`) or, from older bob,
  dated.
- In `Tests/CaptureCoreTests/CapturePomodoroClosePresentationTests.swift`, add the Mac
  stripper vectors. Add one presentation test that decodes a close whose `work_log` and
  `typed_work_log` carry stamped entries and asserts unstamped previews. Existing
  date-only fixtures must keep passing unchanged.
- In docs, change `docs/capture-previews.md` from "with the date stripped" to "with its
  date or date-time stamp stripped". Run `just docs-check`.
- Verify with `just portable-check` (CaptureCore is portable). Do not touch
  `Sources/BobMacCapture*`. If you must, follow AGENTS.md's Mac compile procedure, and
  report the macOS targets as unverified if the Mac is offline.

## Phase nav-stamp: Navigation hotkeys read and write stamps

Work in the bob-plugins linked repo, opened with `/sase_repo`, in plugin
`bob-navigation-hotkeys`. Edit `src/` fragments only, then run `npm run build`. Keep
every fragment at or below 1000 lines.

Reader (do this first):

- `542-lane-walk-model.js` `readLatestWorkLog`: accept an optional ` HH:MM` after the
  date inside the italics. Return `{ date, time, summary }`, with `time: null` for
  legacy entries, and update the comment that names the grammar.
- Add one helper, for example `laneWalkWorkLogLabel(workLog)`, that renders
  `🛠️ ${reviewShortDate(date)}${time ? " " + time : ""} — ${summary}`. Use it in both
  the text model (`542`) and the DOM notice (`543-lane-walk-notice.js`) so they can't
  drift.

Writers:

- In `220-property-targets.js`, add `formatWorkLogStamp(date)`: local getters,
  zero-padded, minutes truncated, plus a comment pointing at bob-cli's
  `docs/reference/vault-format.md` grammar. Rename the last parameter of
  `formatLaneWorkLogEntry` and `insertLaneWorkLogEntry` from `dateText` to `stampText`.
  Replace `resolveSchedulingWorkLogDateText` with a stamp resolver (`workLog.stampText`,
  then a caller-supplied stamp, then `formatWorkLogStamp(new Date())`).
- Stamp at commit on every writer path, reading the clock once per gesture through an
  injectable `now` (an options field or plugin seam that defaults to `new Date()`). Then
  pass the same `stampText` to every target in the batch:
  - `195-rotten-keep.js`: Pending keep summary.
  - `219-plugin-project-subtree-writers.js`: both `insertLaneWorkLogEntry` sites.
  - `230-schedule-review.js`: both sites.
  - `245-task-link-lane.js`: Alt+N release and Move to Next.
  - `570-plugin-link-roll-and-schedule.js`: Ctrl+Shift+P and the Task Card schedule
    stage.

  `dateText` keeps every other job it has (fresh stamps, Schedule Log, keep counting).
  Don't touch those.

- Previews: `192-freshness-refresh-summary-modal.js`, `196-task-link-lane-modal.js` with
  `taskLinkLanePromptState` (`245`), `370-picker-schedule-work-log.js`, and
  `380-picker-cancel-and-lane.js`. Each renders `formatWorkLogStamp(now)` at render
  time.
- Guard: when done, every call that
  `rg -n "insertLaneWorkLogEntry\(|formatLaneWorkLogEntry\(" plugins/bob-navigation-hotkeys/src`
  finds must pass a stamp produced by `formatWorkLogStamp`. None may pass a bare date.

Tests:

- Add the reader and chip vectors in `scripts/test-navigation-lane-walk.cjs`, plus
  `scripts/test-ledger-tools-lane-walk.cjs` if it covers the chip.
- Add formatter vectors.
- With a fixed `now`, assert exact stamped lines in the writer suites
  (`test-navigation-hotkeys-schedule-work-log.cjs`,
  `test-navigation-hotkeys-task-link-lane.cjs`, `test-navigation-rotten-keep.cjs`,
  `test-navigation-hotkeys-schedule-review.cjs`,
  `test-navigation-project-subtree-gestures.cjs`, `test-navigation-refresh-modal.cjs`,
  and any other suite that pins an entry).
- Add one batch test that every target shares one stamp.
- Add one test where a prompt opens at 23:59 and commits at 00:01: it stamps the new
  date at `00:01`.

Docs: in `docs/bob-navigation-hotkeys.md` (and `docs/task-card.md` if it shows the
entry), update the entry previews and the lane-walk chip time. Run `npm run docs:check`.

Verify: `npm run build`, `npm test`, then
`bob plugins sync --repo <this checkout> -p bob-navigation-hotkeys`.

## Phase cli-stamp: bob capture close stamps entries and owns the grammar docs

Work in bob-cli.

- `src/native/capture_work_log.rs`:
  - Add `pub(crate) fn work_log_stamp(now: NaiveDateTime) -> String` (`%Y-%m-%d %H:%M`).
  - Rename `write_work_log_group`'s `date` parameter to `stamp`, and rename `dated` to
    `stamped` in locals and docs.
  - Change the module doc to "stamped Pomodoro Work Log groups".
- `src/native/capture_pomodoro_close/linked_tasks.rs`:
  - `ClosePlanner` gains `work_log_stamp`, computed once in `ClosePlanner::new` from the
    `now` the close already receives, and `write_logs` passes it to
    `write_work_log_group`.
  - `self.date`, the day-file date, keeps its completion-date jobs
    (`complete_embedded_trees` and the completion parse) unchanged.
  - Update the `work_log` and `typed_work_log` doc comments to say "stamped".
- There is no other Rust writer. The only non-test `WORK LOG**` construction is in
  `capture_work_log.rs`. `capture/output.rs` prints entries as given.
- Tests:
  - Add formatter vectors as a unit test.
  - Update `capture_work_log.rs`, `capture_pomodoro_close/linked_task_tests.rs`,
    `selection_tests.rs`, and `tests/cli/capture/pomodoro_close*.rs`, plus any other
    suite that asserts a newly written entry.
  - Fixture entries that already existed stay date-only. Only written entries gain the
    stamp, and every such CLI test pins `BOB_NOW`.
  - Add one CLI test: closing a session in the 2026-09-28 day file with
    `BOB_NOW=2026-09-29 00:10:00` writes `*2026-09-29 00:10* — …`, and the human output
    prints the stamped entry.
- Docs:
  - `docs/reference/vault-format.md`: add the canonical Work Log entry grammar beside
    the managed-children registry. Cover the writer form, the legacy form readers
    accept, write-instant and one-instant-per-gesture semantics, which clock each side
    uses, and "never backfilled".
  - `docs/capture.md`: the close prose (~3660), the `typed_work_log` field note
    (~3704–3705: `*YYYY-MM-DD HH:MM* — text`), and the worked example (~3834–3836).
  - `docs/plan.md`: ~436 and the ~515 table row.
  - `docs/projects.md`: ~1103 and the example at ~1119–1126.
  - Run `just docs-check`.
- Commit subject: `feat(capture)!: …`. The documented `work_log` and `typed_work_log`
  string shape changes, which the release-train decision counts as a JSON contract
  change. Never touch `Cargo.toml`'s version or `CHANGELOG.md`.
- Verify: `just check`.

> [!decision] memory_decision_record Use `/sase_memory_write` to add strand
> `sase/memory/decisions/work-log-entries-are-stamped-at-write.md` (status accepted,
> decided 2026-11-01). It applies to bob-cli, bob-plugins, Bob Mac Capture, and the
> vault. Claim: semantics 1, 2, 6, and 7. Why: the problem statement above. It cites
> this plan and the phase commits. Rejected alternatives: seconds, UTC offsets,
> session-span stamps, backfilling, and stamping the Rotten, Schedule, and Cancel Logs
> too. Then run `sase memory init`.

> [!decision] memory_glossary_work_log Use `/sase_memory_write` to add one sentence to
> the `glossary:work-log` strand: "Each entry opens with an italic `*YYYY-MM-DD HH:MM*`
> stamp of the minute it was written; entries from before stamps keep `*YYYY-MM-DD*`."
> Then run `sase memory init` and `just docs-gen` to regenerate
> `docs/reference/glossary.md`.

## Phase cycler-stamp: Unlink prompt and Pomodoro checkbox close write stamps

Work in the bob-plugins linked repo, opened with `/sase_repo`, in plugins
`block-id-prompt` and `task-status-cycler`. Edit `src/` fragments, then run
`npm run build`.

- block-id-prompt:
  - Add `formatWorkLogStamp(date)` with the same contract.
  - `formatWorkLogEntry(summary, stampText)`.
  - `workSummaryPromptState` derives its preview stamp from `options.now`.
  - `planWorkLogInsertion` takes `options.stamp` in place of `options.date`, and its
    result reports `workLogStamp` instead of `workLogDate`.
  - The callers in `120-plugin-pomodoro-links.js` (Ctrl+Shift+Enter unlink) and
    `130-plugin-task-link-open-and-notices.js` pass
    `options.workLogStamp || formatWorkLogStamp(this.now())`, read once at commit.
  - Update `090-prompt-modals.js` (preview) and `170-exports.js`.
- task-status-cycler:
  - Add `formatWorkLogStamp(date)`.
  - `formatWorkLogEntryBody`, `buildWorkLogEntryLines`, and
    `planPomodoroWorkLogGroupInsertion` take the stamp.
  - In `170-plugin-pomodoro.js`, read it once per close through a new overridable seam,
    `getPomodoroWorkLogStamp()`. It replaces `getPomodoroWorkLogDateString(activePath)`
    in `190-plugin-targets.js`, so the daily-note filename is no longer the source. Pass
    it to every group and target of that close.
  - Update `210-exports.js`.
- Tests: add formatter vectors. Update `test-block-id-prompt-work-log.cjs`,
  `test-block-id-prompt-pomodoro-unlink-runtime.cjs`,
  `test-task-status-cycler-pomodoro-work-log.cjs`, and any other suite that pins an
  entry to a fixed `now`. Add three tests:
  - Two Task Links to one task in one close share a stamp.
  - Closing yesterday's daily note stamps the write instant.
  - The unlink preview shows the stamp.
- Docs: update `docs/block-id-prompt.md` (~80 example) and `docs/task-status-cycler.md`
  (~43, ~138). Run `npm run docs:check`.
- Verify: `npm run build`, `npm test`, then run
  `bob plugins sync --repo <this checkout> -p block-id-prompt` and the same command with
  `-p task-status-cycler`.

## Non-goals

- Backfilling, reformatting, or relative-time rendering ("2h ago") of entries in the
  vault.
- Plugin timezone awareness: plugins keep host-local time, as for every other plugin
  date stamp.
- Stamping the Rotten, Schedule, or Cancel Logs.
- New CLI flags or JSON fields. The JSON strings stay the exact vault entry text.
