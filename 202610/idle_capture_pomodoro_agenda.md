---
tier: epic
title: 'Idle Pomodoro agenda: an empty Bob Mac Capture panel shows the running Pomodoro
  and everything queued after it'
goal: 'When the capture panel opens with an empty draft, it already shows today''s
  agenda, painted from memory in the first frame: the running Pomodoro and every future
  Pomodoro, each with its linked tasks at the most detail that fits below a fixed
  eye line without scrolling. The `=x` and `=` numbers match the ones bob will use.
  Detail folds per Pomodoro, farthest first: logs, then one-line tasks, then one row
  per Pomodoro, then a name strip. bob owns every fact; the app owns caching, fitting,
  and pixels.

  '
decisions:
  agenda_decision_memory:
    ask: Add a `decisions` memory record for the idle agenda's caching, folding, and
      eye-line policy?
    memory:
    - decisions
    default: true
    requested: Review the idle_capture_pomodoro_agenda.md file in the research sidecar
      repo for context and inspiration before planning. I agree with all of the requirements
      recommended in that research file.
    answer: true
phases:
- id: cli-agenda
  title: bob capture-pomodoros --tasks returns the resolved agenda
  depends_on: []
  size: medium
  description: 'cli-agenda: add the additive `-t/--tasks` flag to `bob capture-pomodoros`.
    It returns roles, role-specific operator numbers, resolved Task Links with clean
    titles, statuses, and log-tagged block lines, plus ledger notes, session notes,
    retired counts, the date, and a completed summary. One memoized pass, deterministic
    bytes, colored human output, docs, golden fixtures, and a read-count perf gate.'
- id: mac-agenda-models
  title: Agenda JSON models, client call, fake-bob branch, and fixtures
  depends_on:
  - cli-agenda
  size: small
  description: 'mac-agenda-models: add the CaptureCore `CaptureAgendaSnapshot` decoders
    (decodeIfPresent everywhere), `BobProcessClient.captureAgenda(previous:)` with
    a byte-compare short circuit and old-bob detection, a fake-bob `--tasks` branch,
    and fixtures copied from bob-cli goldens, all with tests.'
- id: mac-agenda-store
  title: In-memory agenda store, refresh triggers, path-filtered watcher, count from
    snapshot
  depends_on:
  - mac-agenda-models
  size: medium
  description: 'mac-agenda-store: add the pure refresh state machine and relevance
    filter in CaptureCore, and the @MainActor `CaptureAgendaStore` that refreshes
    on launch, filtered vault events, show, submit, wake, unlock, and midnight. It
    replaces the per-show capture-pomodoros spawn, derives the close-comma count from
    the snapshot, passes FSEvents paths through the watcher, and falls back for an
    old bob.'
- id: mac-agenda-planner
  title: Agenda presentation, inline text, and the focus-gradient fit planner
  depends_on:
  - mac-agenda-models
  size: medium
  description: 'mac-agenda-planner: add the pure CaptureCore `CaptureAgendaPresentation`
    (groups, rows, row keys, duplicates, chips, strings), `CaptureAgendaInlineText`,
    `CaptureAgendaLayoutMetrics`, `CaptureAgendaBudget`, the countdown wording, and
    `CaptureAgendaFitPlanner` (the seven-step farthest-first ladder with manual expansions).
    Table-driven tests run on Linux.'
- id: mac-agenda-view
  title: Agenda view, row measurer, panel integration, and fixed eye line
  depends_on:
  - mac-agenda-store
  - mac-agenda-planner
  size: medium
  description: 'mac-agenda-view: build `CaptureAgendaView` and its row views, the
    offscreen `CaptureAgendaRowMeasurer`, model wiring (visibility, plan, expansions,
    settle while hidden), auxiliary-region integration with a height cap, fixed eye-line
    placement instead of re-centring, and the Settings toggle. Add height-consistency
    and rendered design tests, checked from the CI artifact.'
- id: mac-agenda-polish
  title: Transitions, countdown, stale and error states, accessibility, signposts,
    README
  depends_on:
  - mac-agenda-view
  size: medium
  description: 'mac-agenda-polish: add the first-keystroke dim-hold, the in-place
    cross-fade, the Now countdown, stale, loading, unsupported, and multiple-timed
    states, a Settings diagnostic, accessibility containers and values, `agenda-*`
    signposts, a show-path spawn guard test, and the README rewrite of the no-dead-space
    principle plus a new `## Idle agenda` section.'
- id: closeout
  title: Decisions record, final verification, follow-ups, and Bryan's checklist
  depends_on:
  - mac-agenda-polish
  size: small
  description: 'closeout: write the accepted `decisions` record for the idle agenda
    (if the memory decision is on), confirm docs and README coherence across both
    repos, re-run the final checks, record proposed follow-ups, and leave Bryan the
    manual Mac checklist.'
proposed_by: bbugyi200.athena.research.48.linker.w0
decided_by: auto
create_time: 2026-10-09 17:42:21
status: wip
bead_id: bob-cli-66
---

- **PROMPT:** [prompts/202610/idle_capture_pomodoro_agenda.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202610/idle_capture_pomodoro_agenda.md)
- **BEAD:** [bob-cli-66](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-66/README.md)

# Idle Pomodoro agenda in Bob Mac Capture

## Why

Bryan opens the capture panel dozens of times a day. With an empty draft it shows a
one-line editor and nothing else, even though the question he most often has when he
opens it is "what am I doing, and what's next?" Under
`decisions:today-is-read-from-the-ledger`, Today already means the Task Links under
today's open Pomodoros. Showing that ledger in the empty panel adds no new concept. It
also makes the session operators easier to use: `=x1*2`, `=~2`, and `=#bob~1` refer to
numbers that today stay invisible until you type.

This epic implements the consolidated research report
`research:202610/idle_capture_pomodoro_agenda/idle_capture_pomodoro_agenda.md`
(2026-10-09). Bryan accepted every requirement and adjustment it recommends (A1–A11).
The report's open questions are settled here:

- fixed eye line: yes;
- countdown and overdue in the Now header: yes, at minute granularity, only while
  visible;
- a "5 done · 2h 40m" title-row summary: yes;
- ledger notes as italic secondary lines: yes;
- ladder tiers 4–5 in v1: yes;
- the v1.1 interactions: out of scope (closeout records them as follow-ups).

[Departures from the research](#departures-from-the-research) lists where this plan goes
further or differs.

Live numbers from athena on 2026-10-09 (Linux, release `bob`):

- `bob capture-pomodoros -f json`: 3 open placeholders (FIX with 5 links, SASE 2, BOB 1)
  and 5 completed entries.
- Benchmarks from the report: `capture-pomodoros` 3.1 ms; `capture --dry-run -- =` 10.5
  ms. The latter resolves 5 links, including the 110 KB `sase.md`, and is the closest
  proxy for `--tasks`.

## The experience (north star)

1. **The capture hotkey.** The panel appears in one frame, exactly where the compact bar
   appears today. The editor is focused, with the cursor in it. Below it, in the same
   `.thinMaterial` pane the live preview uses, sits today's agenda:

   ```
   ╭──────────────────────────────────────────────────────────────────────────╮
   │  Type to capture…                                                        │ editor, focused, fixed eye line
   │ ┌──────────────────────────────────────────────────────────────────────┐ │
   │ │ Today · Fri 9 Oct                                   5 done · 2h 40m  │ │ quiet title row
   │ │┃▶ FIX                               14:10–14:35 · 12m left          │ │ Now: pink rail + faint wash
   │ │┃ ① ◐ Use just install-venv instead of install so install can …      │ │ number badge · status glyph
   │ │┃       epic on athena                                               │ │ ledger note (italic, secondary)
   │ │┃       ⚒ Work log                                                   │ │ log header (footnote, tertiary)
   │ │┃         Oct 8 — Running research on athena.                        │ │
   │ │┃ ② ◉ Add bob mac command to manage bob-mac-capture                  │ │
   │ │┃    ✓ 2 done this session                                           │ │ retired links: one dim line
   │ │  ◌ SASE  Next                                      = starts it      │ │ Next: dashed glyph + capsule
   │ │   ① ◐ Use just install-venv instead of …                ↑ in FIX    │ │ duplicate → one line + pointer
   │ │   ② ◉ Fix muse reply streaming                             ⚒ 2      │ │ folded log → chip
   │ │  ◌ BOB   Dynamic AGENTS.md · Plan epic roadmap for goals   2 tasks  │ │ Later folded to one row
   │ │  Later · READ · GOALS · BLOG · +6                                   │ │ name strip (heavy days only)
   │ └──────────────────────────────────────────────────────────────────────┘ │
   │ Ready                                          Stash 0  Discard  Capture │ footer unchanged
   ╰──────────────────────────────────────────────────────────────────────────╯
   ```

2. **On a normal day nothing folds.** Today's workflow (3–5 open entries, at most 17
   links) fits at full detail about 99% of the time. Every task shows its full
   definition: title, every sub-bullet, Depends-On line, Work Log, and Schedule Log.
3. **On a heavy day the agenda folds and never scrolls.** Detail fades with distance in
   time: first the farthest Pomodoros lose their logs, then their tasks become one line,
   then each becomes one row, then they merge into a name strip. Next and Now fold last.
   Every fold leaves a chip that says what is hidden (`⚒ 3`, `+5 lines`, `4 tasks`), and
   clicking a chip expands that unit in place.
4. **Typing replaces the agenda calmly.** The first keystroke dims the agenda. When the
   live preview arrives (or after 250 ms), the agenda swaps for the preview with one
   height change. The editor never moves. Clearing the draft brings the agenda back
   instantly.
5. **It is always right.** Editing a linked task in Obsidian, submitting a capture,
   waking the Mac, or crossing midnight refreshes the agenda in the background.
   Yesterday's plan is never shown. A broken link shows as a warning row with the raw
   link. A failed refresh keeps the last good agenda and marks it stale.

## Departures from the research

| Research said                                                             | This plan does                                                                                                                                                                                                                                   | Why                                                                                                                                                                                    |
| ------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `index` is the `=x` number on Now, the `=` number on Next, otherwise null | On every non-current open entry, `index` is that entry's `=` lineup number                                                                                                                                                                       | `=#NAME~K` and `==#NAME~K` drop by those same lineup numbers (`docs/capture.md`), so a Later entry's numbers are just as actionable                                                    |
| Task Links are direct-child plain or embedded links                       | `items` are every single-block-link line that `number_task_links` recognizes: any depth, plain, deferred `[[T]]#`, or embedded. Each carries `ledger_depth` and `marker`, and `index` is null where the role's numbering rule does not number it | On the running entry, `=x` numbers nested and deferred links too. Using bob's exact recognizer keeps every shown number truthful and leaves no link line rendered as raw `[[…]]` prose |
| `retired_link` items                                                      | A per-entry `retired_link_count`                                                                                                                                                                                                                 | The view only ever shows "✓ N done", so a count is the whole contract                                                                                                                  |
| `resolution: missing \| ambiguous \| not_a_task \| unreadable`            | `missing_note \| ambiguous_note \| missing_block \| duplicate_block \| not_a_task \| unreadable`, each with a human `warning`                                                                                                                    | The warning row can say exactly what is wrong                                                                                                                                          |
| `status_type` in snake_case                                               | The uppercase labels every other `capture-*` JSON already uses (`capture_tasks::status_type_label`)                                                                                                                                              | One vocabulary across the protocol                                                                                                                                                     |
| Name strip at most 3 lines                                                | Same, but the planner budgets the strip at its height for _all_ Later names, an upper bound                                                                                                                                                      | It never under-budgets, and it needs one measurement instead of one per prefix                                                                                                         |
| A tall agenda's growth is clamped "at the bottom margin"                  | While the agenda owns the auxiliary region, its reported ideal height is capped at the below-eye-line budget. Typed previews keep today's full-screen limit and slide-up behavior                                                                | Guarantees the agenda can never move the editor, without changing how existing previews behave                                                                                         |
| Perf budgets gated in tests                                               | A deterministic gate (each distinct note is read and scanned once) plus a coarse wall-clock guard. Release timings are measured with hyperfine and recorded on the bead                                                                          | Millisecond assertions flake in debug builds and on CI. Read counts catch the real regression, accidental O(links) re-reads                                                            |
| —                                                                         | Additive `starts_at`, `status_name`, line `kind`, and `lines_truncated` fields                                                                                                                                                                   | The countdown, accessibility labels, nested-subtask glyphs, code lines, and bounded payloads need them                                                                                 |

## Ground rules (every phase)

- **Repositories.**
  - bob-cli phases (`cli-agenda`, `closeout`) work in this project's workspace.
  - App code lives in the linked repo **bob-mac-capture**. Open it with
    `sase repo open bob-mac-capture -r "<reason>"`. If that fails because the linked
    checkout is missing on the host, use
    `sase repo open gh:bobs-org/bob-mac-capture -r "<reason>"`. Work only in the path
    the command prints, and read that repo's `AGENTS.md` if the command names one.
- **Thin client (`decisions:mac-capture-is-a-thin-client`).**
  - bob owns every ledger fact: roles, Task Link recognition, numbering, link
    resolution, task blocks, line kinds and depths, log tagging, clean titles, statuses,
    the date, and the completed summary.
  - The app decodes, deduplicates repeated tasks, measures, plans folds, and draws.
  - The app never reads the daily note or any task note, never infers current/next from
    the clock, and never parses list structure. Inline styling of display text (code
    spans, wikilink display text, emphasis, dimmed `[key:: value]` fields) is
    presentation, not grammar. `TaskDisplayText` and
    `CapturePomodoroLineTokens.tokenizeTaskRow` are the precedents.
- **JSON contract.**
  - `capture-pomodoros` stays at `schema_version` 1. Every new field is additive and
    appears only with `--tasks`, so the default output is byte-identical to today's.
  - The app decodes every new field with `decodeIfPresent` and a safe default, and
    rejects any other `schema_version`.
  - The contract below is shared by all phases. Keep the names. If a compile constraint
    forces a rename, add an `INTERFACE CHANGE:` note on the phase bead.
- **Speed is a feature (the user's hard requirement: "MUST be blazing fast").**
  - Nothing new runs between the hotkey and `makeKeyAndOrderFront` except reading
    already-planned state. No disk reads, no process waits, no hashing, no measuring,
    and no planning on the show path.
  - The process spawns per show stay at one: the agenda refresh replaces today's
    `capture-pomodoros` spawn.
  - Byte-identical bob output is a no-op: no decode, no publish, no re-layout.
- **No Swift toolchain on the Mac agent path.**
  - GitHub Actions (`.github/workflows/ci.yml`, `macos-26`) is the only compiler for the
    AppKit targets, so write conservatively:
    - Swift 5 language mode;
    - `@MainActor` on every AppKit-touching type;
    - `@available(macOS 26.0, *)` on views, as the panel does;
    - long-stable APIs only;
    - swift-format default style (4-space indent, lines ≤ 100 columns);
    - no long `+` or `??` expression chains (the type-checker times out on them).
  - athena has Linux Swift (`/usr/bin/swift`). CaptureCore phases run `swift test` on a
    scratch copy of the tree (`--filter CaptureCoreTests` where useful). Linux builds
    only `CaptureCore`, `RefsCore`, and `refs-rank`.
- **Commit and CI loop for bob-mac-capture phases.** This plan explicitly authorizes
  every bob-mac-capture phase to do the following:
  1. `git pull --rebase` on `master`. `mac-agenda-store` and `mac-agenda-planner` may
     run in parallel and touch disjoint files.
  2. Commit to `master` with `/sase_git_commit` from the bob-mac-capture checkout, using
     conventional headers (`feat(agenda): …`, `test(agenda): …`).
  3. Find the CI run with
     `gh run list -R bobs-org/bob-mac-capture --workflow CI --commit <sha> --json databaseId`.
  4. Watch it through `/sase_monitor`
     (`gh run watch <id> -R bobs-org/bob-mac-capture --exit-status`).
  5. Fix forward until the whole job is green. On failure, read
     `gh run view <id> --log-failed` and grep for ` error:`.

  A phase is not done until its CI run is green. Its bead note records the run URL and
  the SHA.

- **Look at the pixels.** Every phase that changes agenda visuals downloads
  `render-fixtures`
  (`gh run download <id> -R bobs-org/bob-mac-capture -n render-fixtures -D <tmpdir>`),
  opens its `agenda-*` PNGs with the Read tool, and checks them against
  [Visual design](#6-visual-design) before declaring done. Fix any misalignment,
  clipping, contrast problem, or unintended truncation it sees.
- **Fixtures are synthetic.** Never commit real vault titles or paths. Mac fixtures are
  copied from the bob-cli goldens, which are generated from synthetic vaults.
- **Privacy.** Signposts carry names and counts only, never titles or paths.
- **Tests match their neighbors.** Use the existing `fakeBobPath()`, `waitUntil`,
  `#filePath` fixture loading, `RenderFixtureWriter`, and `UserDefaults(suiteName:)`
  patterns. In bob-cli, use the in-module `TempDir` tests and
  `crate::native::env::with_var` (thread-local, never process env), as
  `capture_pomodoros.rs` does.

## Design specification (decided; implement exactly)

### 1. bob contract: `bob capture-pomodoros --tasks` (`-t`)

**CLI.**

- `-t, --tasks`, a `SetTrue` flag. Help: "Resolve each open entry's Task Links into
  tasks for agenda views (roles, operator numbers, task blocks)".
- Declare it after `help_arg()` so the options stay alphabetical: `-a -b -f -h -t`.
- Extend `long_about` with one paragraph on `--tasks`. Add
  `bob capture-pomodoros -t -f json` to the examples.
- Keep `about` unchanged, so the root help fixtures stay byte-identical.
- `--all --tasks` is allowed: completed entries get `role: "completed"` and empty
  `notes` and `items`. The app never passes `--all`.

**Top-level additions** (only with `--tasks`):

| Field               | Type               | Meaning                                                                                                                                                                  |
| ------------------- | ------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `date`              | `"YYYY-MM-DD"`     | The daily note's date (`pomodoro::parse_day_file_date`, falling back to `env::current_datetime().date()`). The app never shows a snapshot whose `date` is not today      |
| `completed_summary` | `{count, minutes}` | Completed entries in today's ledger, and the summed minutes of their time ranges (reuse the range-duration helper in `capture/pomodoro_adjust.rs`; widen its visibility) |

**Per-entry additions** (on every listed entry, only with `--tasks`):

| Field                  | Type                                            | Meaning                                                                                                                                                                                    |
| ---------------------- | ----------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `role`                 | `current \| next \| later \| open \| completed` | `current` = `is_current`; `next` = `next_future_pomodoro`; `open` = an open timed entry when several are open (nothing is current); `later` = every other open entry; `completed` = closed |
| `starts_at`, `ends_at` | `"YYYY-MM-DDTHH:MM"` or null                    | Naive local datetimes for timed entries. An end before the start rolls to the next day                                                                                                     |
| `retired_link_count`   | int                                             | Struck single-link lines (`~~[[T]]~~`, already done) in the entry's sub-bullet range                                                                                                       |
| `notes`                | `[Line]`                                        | Session notes: the entry's sub-bullet lines that are neither items nor retired links nor descendants of an item, in ledger order, with `depth` relative to the entry (direct child = 1)    |
| `items`                | `[Item]`                                        | See below, in ledger order                                                                                                                                                                 |

**Item** = one line that `number_task_links`'s recognizer accepts: after stripping 🍅
markers, the list body is exactly one block link (`[[T]]`, `[[T]]#`, `![[T]]`, or
`![[T]]#`), unstruck, outside fences, at any depth in the entry's sub-bullet range.

| Field                                         | Type                                                                                                         | Meaning                                                                                                                                                                                                                                                |
| --------------------------------------------- | ------------------------------------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `index`                                       | int or null                                                                                                  | Current entry: the `=x` number from `number_task_links`. Every other open entry: the `=` lineup number from `list_queued_links` matched by ledger line. Null when that rule does not number the line (deferred or nested links on a non-current entry) |
| `ledger_line`                                 | int                                                                                                          | 1-based line in the daily note                                                                                                                                                                                                                         |
| `ledger_depth`                                | int                                                                                                          | 1 = a direct child of the entry                                                                                                                                                                                                                        |
| `marker`                                      | `plain \| deferred \| embedded`                                                                              | `TaskLinkMarker::as_str()`                                                                                                                                                                                                                             |
| `block_link`                                  | string                                                                                                       | Verbatim `[[path#^id\|alias]]` without `!`                                                                                                                                                                                                             |
| `resolution`                                  | `resolved \| missing_note \| ambiguous_note \| missing_block \| duplicate_block \| not_a_task \| unreadable` | How the link resolved                                                                                                                                                                                                                                  |
| `relative_target`                             | string or null                                                                                               | Vault-relative note path; for `[[#^id]]`, the daily note                                                                                                                                                                                               |
| `line`                                        | int or null                                                                                                  | 1-based task line in the target note                                                                                                                                                                                                                   |
| `block_id`                                    | string                                                                                                       | The link's block id                                                                                                                                                                                                                                    |
| `text`                                        | string or null                                                                                               | Clean title (`NoteTask.description`, with `#task` dropped for filter-cleared embedded lookups as `close_task_text` does)                                                                                                                               |
| `status_symbol`, `status_name`, `status_type` | string or null                                                                                               | From the resolved `NoteTask`. `status_type` uses `capture_tasks::status_type_label`                                                                                                                                                                    |
| `lines`                                       | `[Line]`                                                                                                     | The task's block (`NoteTask.block_end`) minus the task line, in document order. Blank lines are dropped; `depth` is relative to the task (direct child = 1)                                                                                            |
| `lines_truncated`                             | int                                                                                                          | Lines omitted after the first 150 (0 normally)                                                                                                                                                                                                         |
| `ledger_notes`                                | `[Line]`                                                                                                     | Descendants of the link line in the ledger that are not items themselves; `depth` is relative to the link line                                                                                                                                         |
| `warning`                                     | string or null                                                                                               | Human sentence for any non-`resolved` resolution (bounded with `bounded_warning`)                                                                                                                                                                      |

**Line:**

| Field           | Type                                           | Meaning                                                                                                                                                                                          |
| --------------- | ---------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `text`          | string                                         | Display body. `task` lines: `clean_description` (no `#task`, inline fields, or `^id`). `bullet` lines: the body after the list marker. `text` and `code` lines: the line without its indentation |
| `depth`         | int                                            | ≥ 1, relative to the owner. List items count parent hops (as `capture::block_depths` does; make it `pub(crate)`); continuation lines get their parent's depth + 1                                |
| `kind`          | `bullet \| task \| text \| code \| log_marker` | `task` = a checkbox list item. `code` = a line inside a fence (fence delimiter lines are omitted). `log_marker` = `parse_managed_task_log_marker` matched (all four accepted spellings)          |
| `status_symbol` | string or null                                 | `task` lines only                                                                                                                                                                                |
| `log`           | `work \| schedule \| null`                     | Set on a log marker line and on every line of its subtree, including logs inside nested tasks. Prose that merely mentions "work log" is not tagged                                               |

**Resolution rules.**

- Build one `VaultLinkResolver` for the run. Empty targets (`[[#^gtd]]`) resolve to the
  daily note and use the already-read contents.
- Memoize by `(path, filter_cleared)`: read each distinct note once, and run
  `note_tasks::scan` once per distinct note and filter mode. Embedded links resolve with
  the global filter cleared, as `resolve_queued_links` does.
- Read `note_tasks::read_settings` once per run.
- Never touch `TaskIndex`, Dataview, or `bob plan`.
- Fail soft. A missing daily note or `## Pomodoros` section keeps today's behavior:
  `ok`, an empty list, and a warning. A problem with one link becomes that item's
  `resolution` and `warning`, never a command failure.
- Determinism: the same vault bytes produce the same output bytes. No field depends on
  the clock, except `date` when the day file's name is not a date.

**Human output with `--tasks`.** Keep the existing header, and append
`· Fri 2026-10-09 · 5 done (2h 40m)`. Then, per shown entry:

- a role-badged header: `▶ FIX 14:10-14:35 now` (green bold badge), `◌ SASE next`
  (cyan), `◌ BOB later` (dim), `◆ … open` (yellow);
- one row per item: the number (or blank), `[symbol]`, the clean title, dim
  `relative_target:line`, and a dim summary such as `+4 lines · work log 2`;
- unresolved items as a yellow `!` row with the raw link and its warning;
- `✓ N done` when `retired_link_count > 0`.

Colors come only from `Styler`. Plain output stays readable.

**Docs.** In `docs/capture.md` → Discovery commands, update the usage line and add a
`--tasks` subsection with this contract, the numbering rules (and why Later entries are
numbered), and the fail-soft rules. Update the `README.md` command mentions if they list
options.

### 2. App data and freshness

**Layers.**

| Layer                | Holds                                                                                                         | Invalidated by                                                |
| -------------------- | ------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------- |
| View                 | The planned, laid-out agenda inside the pre-warmed panel; measured row heights keyed by row content and width | A new snapshot, a width change, a budget change, an expansion |
| `CaptureAgendaStore` | The last good decoded snapshot and the raw stdout bytes it came from                                          | The triggers below; byte-identical output is a no-op          |
| bob                  | Nothing persistent                                                                                            | —                                                             |

**Triggers.** All share the `agenda` lane. One refresh is in flight at a time. A trigger
during a refresh schedules exactly one follow-up. A generation counter discards results
that started before a reset (Recheck Bob, vault or executable change).

1. App launch, after `prewarm()`, so the first hotkey after login already has data.
2. Vault FSEvents, whether or not the panel is visible, when
   `CaptureAgendaRefreshFilter.isRelevant` accepts the batch (below).
3. Every panel show, as stale-while-revalidate. The cached agenda paints first, then one
   background refresh checks it. This **replaces**
   `refreshCurrentPomodoroTaskLinkCount()`'s spawn. It runs for retained drafts too,
   because the close-comma count needs it.
4. After a successful submit.
5. `NSWorkspace.didWakeNotification`, `NSWorkspace.sessionDidBecomeActiveNotification`,
   `.NSCalendarDayChanged`, and `.NSSystemClockDidChange`.
6. Recheck Bob and changes to the `bobDirectory` or `bobExecutableOverride` settings.
   These reset the store (snapshot, bytes, capability) and then refresh.

**Relevance filter** (`CaptureAgendaRefreshFilter`, pure, in CaptureCore). A batch is
relevant when any of these hold:

- a flag says must-rescan, root-changed, or events dropped (user or kernel);
- a path ends in `.md` and has no path component starting with `.` relative to the vault
  root;
- a path is `.obsidian/plugins/obsidian-tasks-plugin/data.json` (the global filter);
- a directory under the vault was renamed or removed (folder moves change basename
  resolution).

Everything else (`.git/`, `.sase/`, `.obsidian/workspace*.json`, `.trash/`, PDFs,
images) is ignored.

**Watcher.** `VaultTargetWatcher` passes paths and flags through
(`kFSEventStreamCreateFlagUseCFTypes`). It accumulates the union of paths and the OR of
flags across its trailing debounce window and delivers one `VaultChangeBatch` per
debounce. The targets cache keeps refreshing on every batch, exactly as today.

**Date guard.** A snapshot whose `date` differs from the app's local `yyyy-MM-dd` (using
an injected clock) is never shown. The agenda shows one "Loading today…" line until the
refresh lands.

**Old bob.** If `capture-pomodoros --tasks` exits non-zero and stderr mentions `--tasks`
(clap's unexpected-argument error), mark the capability `.unsupported` for that
executable path. Do not retry `--tasks` until Recheck Bob or an executable change. Run
the plain `capture-pomodoros` on the `pomodoros` lane for the close-comma count only,
and hide the agenda. Settings shows the diagnostic.

**Close-comma count.** `currentPomodoroTaskLinkCount` comes from the published snapshot
(its current entry's `task_link_count`). It is nil when the snapshot is not today's, or
when the latest refresh failed, exactly as the failed spawn clears it today.

### 3. Presentation (`CaptureAgendaPresentation`, pure, in CaptureCore)

**Groups.** Built from open entries in this order:

1. the `current` entry, or every `open` entry (multiple timed) in ledger order;
2. the `next` entry;
3. `later` entries in ledger order.

| Role    | Header                                                    | Trailing                                                          |
| ------- | --------------------------------------------------------- | ----------------------------------------------------------------- |
| current | `play.circle.fill` (pink) + name, or "Untitled Pomodoro"  | `14:10–14:35` plus the countdown (§7)                             |
| open    | `exclamationmark.circle` (orange) + name + "Open" capsule | time range                                                        |
| next    | `circle.dashed` + name + "Next" capsule                   | `=` starts it (the `=` in the editor's pomodoroStart token color) |
| later   | `circle.dashed` (tertiary) + name                         | `=#slug` in tertiary monospaced when `selectable`, else nothing   |

With several open timed entries, an orange warning row heads the agenda: "Multiple open
timed sessions — close one so `=x` knows which is running".

**Rows**, per group in order:

1. the header;
2. session notes (secondary bullets);
3. items:
   - **Resolved task:** a headline with the number badge, status glyph, and title, then
     its ledger notes (italic secondary), then its child lines. Log subtrees render as a
     log header ("⚒ Work log" with `hammer`, or "Schedule log" with `calendar`) plus
     entries in footnote tertiary.
   - **Resolved, but DONE or CANCELLED:** one struck, dimmed line.
   - **Any other resolution:** one warning row (`exclamationmark.triangle`, the raw
     `block_link`, and the warning, truncated).
   - **A repeat of a task already shown in full earlier in the agenda** (same
     `relative_target` + `block_id`): one line with a trailing `↑ in FIX`.
4. `✓ N done this session` (Now) or `✓ N done` (others) when `retired_link_count > 0`.

Empty groups show "No linked tasks" (tertiary).

- **Markers.** On the current entry only, a trailing tertiary caption shows `deferred`
  for deferred items and `completes on close` for embedded ones.
- **Truncation.** `lines_truncated > 0` adds a final `+N more lines` row.

**Row keys.** Every row has a `CaptureAgendaRowKey` (`Hashable`): the row kind, the
display text, depth, line limit, and accessory presence. The measurer caches heights by
key and width. Unchanged rows across snapshots are never re-measured.

**Inline text** (`CaptureAgendaInlineText`). Extend `TaskDisplayText`'s scanner with
these segment kinds:

- `code`: monospaced;
- `link`: display text only, link-tinted, never clickable in v1;
- `strong` and `emphasis`: from `**…**` and `*…*` / `_…_`;
- `field`: `[key:: value]`, dimmed;
- `tag`: `#tag`, secondary.

A trailing `^block-id` is hidden. Unmatched delimiters stay literal. Segments tile the
text exactly, as `TaskDisplayText` guarantees.

**Strings.** All user-visible wording lives in the presentation (headers, hints, chips,
accessibility labels), so views never branch on JSON.

### 4. Fit planner (`CaptureAgendaFitPlanner`, pure, in CaptureCore)

**Unit states.**

| Unit                     | States, in folding order                                  |
| ------------------------ | --------------------------------------------------------- |
| Task                     | `full` → `noLogs` → `oneLine`                             |
| Now, Open, or Next group | `full` → `noLogs` → `oneLineTasks`                        |
| Later group              | `full` → `noLogs` → `oneLineTasks` → `oneRow` → `inStrip` |

What each state shows:

- **Task `full`:** headline, ledger notes, and every child line.
- **Task `noLogs`:** headline with log chips (`⚒ 3`, `🗓 2`; the count is the log's
  direct entries), ledger notes, and the non-log child lines.
- **Task `oneLine`:** one line with a `+N lines` chip (N = the hidden child lines plus
  ledger notes).
- **Group `oneLineTasks`:** session notes collapse to a header chip (`2 notes`).
- **Group `oneRow`:** a single row: glyph, name, the task titles joined by `·`
  (truncated), and an `N tasks` chip.
- **Group `inStrip`:** the group is a name in the strip `Later · READ · GOALS · +6`. The
  strip is a flow of names, at most 3 lines, with a `+N` overflow token.

Warning rows, struck rows, duplicate rows, and retired rows are always one line.

**Ladder.** Stop at the first step whose total height ≤ budget. Each numbered bullet is
one step applied to one group, farthest group first; groups with nothing to fold are
skipped.

1. Everything `full`.
2. `noLogs`: Later groups (last to first), then Next, then Now/Open.
3. `oneLineTasks`: Later groups (last to first).
4. `oneRow`: Later groups (last to first).
5. `inStrip`: Later groups (last to first). The strip is budgeted at its measured height
   with all Later names (an upper bound).
6. `oneLineTasks`: Next, then Now/Open (last to first).
7. Overflow. The plan is marked `overflows`, and the view scrolls with a bottom fade and
   an `N more` cue.

This keeps the user's priority order everywhere it matters: no task goes to one line
while any task still shows logs. Steps 4–5 fold only Later entries, so on a backlog day
Now and Next keep detail.

**Manual expansion.** `expanded: Set<CaptureAgendaUnitID>` (a task, a group, or the
strip). Expanded units are pinned at `full`; an expanded strip shows its groups as
`oneRow`. Pinned units are skipped by every step. If the result overflows, the region
scrolls, because Bryan asked for it. Expansions reset on hide and when the snapshot
bytes change.

**Height model.**

```text
total = titleRow
      + warningRow (if any)
      + Σ visible groups (groupInsets(role) + Σ row heights)
      + groupSpacing × (visibleGroups − 1)
      + strip (if any)
```

All constants live in `CaptureAgendaLayoutMetrics` and are shared by the view and the
planner: paddings, spacings, the 72 pt trailing accessory column, the 26 pt badge
column, the 16 pt glyph column, the 14 pt indent step, the line-limit clamps (headline
3, child 4, one-line 1), and the 3 pt rail. A height-consistency test (macOS CI) proves
`fittingSize` of the rendered plan equals the planner's total within 2 pt.

**Budget** (`CaptureAgendaBudget`, pure):

```text
budget = eyeLineTop − visibleFrame.minY − screenMargin(24)
       − (frame chrome + titlebarDragInset + compact editor height
          + 2 × sectionSpacing + measured footer height + rootPadding(bottom)
          + preview-pane insets + one row of slack)
```

The budget never depends on the panel's current height, which avoids resize loops.

**Determinism.** Same inputs, same plan. The planner is O(rows) arithmetic per step.

### 5. Window placement: a fixed eye line

- **Show-time placement.** `replayLatestContentMetricsForPresentation` no longer
  re-centres the panel at its target height. `CapturePanelPlacement` computes the eye
  line, the top edge `center()` gives the _compact_ panel (editor + footer, no auxiliary
  region), and caches it per screen visible frame. The controller then places the panel
  at that top with the target height. The compact bar sits exactly where it does today.
  Every show, with or without the agenda or a retained draft, puts the input line in the
  same place.
- **Height while the agenda shows.** The auxiliary ideal height is capped at the agenda
  budget, so the panel never slides up for the agenda.
- **Typed previews are unchanged.** They grow downward from the same top and keep
  today's `CapturePanelWindowSizer` clamp and slide-up rule.
- **Settle while hidden.** When a new plan is published while the panel is hidden, the
  controller calls `layoutSubtreeIfNeeded()`. `latestContentMetrics` then already holds
  the agenda height, and the next show replays it in one frame.
- **Screen changes.** If the screen at show time differs from the one planned for, only
  the budget changes. Re-planning is arithmetic on cached heights.

### 6. Visual design

**Container.** The agenda lives in the same `.thinMaterial`, 8 pt-radius, 10 pt-padded
pane as `PreviewPane`, so the panel reads as one family. It is proportional text, not
monospaced diff cards, and has no diff gutter and no change washes. That signals
read-only.

**Title row.** "Today" (`.subheadline.weight(.semibold)`) plus "· Fri 9 Oct"
(secondary), formatted from `date` with the user's locale. When nothing is running, " ·
Nothing running" follows. On the right, in tertiary with `.monospacedDigit()`, is
`5 done · 2h 40m`, omitted when the count is 0.

**Now card.** A 3 pt pink rail (`CaptureEditorPalette.color(for: .pomodoroStart)`) and a
faint pink wash (opacity 0.06; 0.12 under Increase Contrast), with 6 pt inner padding
and an 8 pt corner radius. This is the only accent in the agenda.

**Typography.**

| Element                      | Font                                           |
| ---------------------------- | ---------------------------------------------- |
| Pomodoro names               | `.subheadline.weight(.semibold)`, tracking 0.3 |
| Times                        | `.callout.monospacedDigit()`, secondary        |
| Task titles                  | `.callout`, primary                            |
| Child lines and ledger notes | `.callout`, secondary (ledger notes italic)    |
| Logs                         | `.footnote`, tertiary                          |
| Inline code                  | monospaced, same size                          |
| Chips and captions           | `.caption2.weight(.medium)`                    |

**Badges and glyphs (reused, not invented).**

- Number badges reuse the start/close badge (`Image(systemName: "\(n).circle")`, with
  the capsule fallback above 50). Extract one shared `CaptureNumberBadge` view if the
  two existing copies are identical.
- Status glyphs and colors come from `CaptureEditorPalette.taskStatus`.
- Nested subtask lines use the same glyphs at footnote size.
- Bullets are a tertiary `•`.

**Chips.** Capsules with `.quaternary` fill and secondary text, using an SF Symbol plus
a count (`hammer` 3, `calendar` 2, `+5 lines`, `4 tasks`, `2 notes`). They are plain
buttons that never take focus, with a hover highlight. Clicking one expands its unit and
returns focus to the editor.

**Spacing.** 10 pt between groups, 2 pt between rows, and 4 pt after a group header.
Indentation is 14 pt per depth, measured from the title's text column.

**States** (one quiet line each, inside the pane):

| State                                         | Line                                                                                          |
| --------------------------------------------- | --------------------------------------------------------------------------------------------- |
| No open Pomodoros                             | `No Pomodoros planned · =#NAME starts one`                                                    |
| No daily note                                 | `No daily note for today yet`                                                                 |
| Snapshot not today's, or none yet             | `Loading today…`                                                                              |
| Refresh failed with a good snapshot for today | The agenda plus `clock.badge.exclamationmark` and "Couldn't refresh" on the title row's right |
| bob too old, or the setting off               | No agenda; the panel is the compact bar exactly as today                                      |

**Light and dark, Increase Contrast, Reduce Transparency.** Use only system semantic
colors and materials. Every design fixture renders in both appearances.

### 7. Motion, countdown, and transitions

- **Show:** no animation. Cached content is in the first frame.
- **First keystroke (dim-hold):**
  - When the draft turns non-blank while the agenda is visible, the agenda fades to 35%
    opacity (100 ms) and stays in place.
  - The hold ends at the first of: `previewState` turning `.ready` or `.failed`; any
    picker, completion, prompt, stash, or error taking the auxiliary region; 250 ms
    elapsing; or the draft returning to blank, which cancels the hold.
  - The region then swaps to whatever owns it now, with one height change. The live
    preview's `CapturePreviewPaneHeightPolicy` settled height is reset at the swap, so a
    still-loading pane uses the 92 pt floor rather than the agenda's height.
- **Clearing to empty:** the agenda reappears immediately from the cached plan, followed
  by a background revalidation.
- **In-place updates while visible:** a 120 ms opacity cross-fade. Height is never
  animated.
- **Reduce Motion** disables the fade and cross-fade. The dim still applies, without
  animation.
- **Countdown:** `TimelineView(.everyMinute)` exists only while the panel is visible and
  a current entry has `ends_at`. Wording comes from the pure
  `CaptureAgendaClock.remainingText(endsAt:now:)`:
  - `12m left` or `1h 05m left`;
  - `ending now` within the last minute;
  - `overdue 8m` in orange.

### 8. Accessibility

- Each group is one accessibility container. Example label: "Running Pomodoro FIX, 14:10
  to 14:35, 2 tasks". The countdown is `.accessibilityValue`, so it is not re-announced
  every minute.
- Task rows read "Task 1, In Progress, <title>". Chips read "Show work log, 3 entries",
  "Show 5 hidden lines", and so on.
- Return in an empty editor never acts on an agenda row. Keyboard focus never leaves the
  editor in v1.

### 9. Settings and diagnostics

- **Toggle.** `AppSettings.agendaEnabled` (key `agendaEnabled`, default on via the
  `defaults.object(forKey:) == nil` pattern), shown in a new "Agenda" section: "Show
  today's Pomodoros when the draft is empty".
- **Off.** The store keeps refreshing for the close-comma count, but nothing is
  measured, planned, or shown.
- **Diagnostic line.** "Last refreshed 14:32", "Couldn't refresh: <message>", or "Update
  bob to show the agenda (needs `capture-pomodoros --tasks`)".

### 10. Signposts and performance acceptance

- **Signposts:** intervals `agenda-refresh`, `agenda-measure`, and `agenda-plan`; events
  `agenda-unchanged`, `agenda-published`, and `agenda-hold-released`. Metadata only.
- **Show path.** No new work. A model test proves that one show invokes bob exactly once
  (`capture-pomodoros … --tasks`) and nothing else, via `FAKE_BOB_RECORD_PATH`.
- **Freshness.**
  - An event batch touching a `.md` file refreshes a hidden panel's agenda.
  - A submit refreshes before the next show.
  - Yesterday's snapshot is never shown.
- **Fit.**
  - Zero overflow for the current-period and September-shaped synthetic fixtures at the
    13" (467 pt) and 14" (533 pt) budgets.
  - Now and Next keep at least `noLogs` on the September-shaped fixture at 533 pt.

## Phase `cli-agenda`: bob capture-pomodoros --tasks

Work in bob-cli.

1. Read `src/native/capture_pomodoros.rs`,
   `capture_pomodoro_close/{selection,ledger,links,linked_tasks}.rs`,
   `capture_pomodoro_start.rs` (`list_queued_links`, `resolve_queued_links`),
   `vault_links.rs`, `note_tasks.rs`, `capture/sub_bullet.rs`
   (`parse_managed_task_log_marker`), `capture/block_diff.rs` (`block_depths`), and
   `plan_budget/today.rs`.
2. Add a submodule (for example `src/native/capture_pomodoros/agenda.rs`, converting the
   file into a module directory if that is cleaner) that builds the §1 payload.
   Re-export what it needs, and widen visibility with `pub(crate)` only:
   `NumberedTaskLink`, `TaskLinkMarker`, `block_depths`, and the range-duration helper.
   Avoid O(entries × file) work where it is cheap to avoid: compute `line_spans` and
   fences once and pass them down, adding a variant of the numbering walk if needed.
3. Add the memoized note reader (`AgendaNotes`) with a `#[cfg(test)]` read counter.
4. Wire `-t/--tasks` into `build_cli`, the request, JSON, and human output, per §1.
5. **Tests** (in-module, following the existing style):
   - **Every semantic fixture:**
     - a running entry plus queued ones;
     - several open timed entries;
     - unnamed and empty placeholders;
     - a task repeated across entries;
     - struck links, embeds, deferred links, nested links under links;
     - mixed prose, fenced lines, and 🍅 markers;
     - `[[#^self]]` links;
     - missing note, ambiguous basename, missing block, duplicate block id, and non-task
       block targets;
     - closed linked tasks;
     - Work and Schedule Logs, including logs inside nested tasks, the legacy
       `**Work log**` spelling, and prose mentioning "work log";
     - CRLF line endings and no final newline;
     - a 200-line task block (truncation);
     - `--all --tasks`;
     - the default output being byte-identical without `--tasks`.
   - **Numbering parity:** the current entry's indexes equal `capture --dry-run -- =x`'s
     `pomodoro_close.task_links` numbers, and the next entry's equal
     `capture --dry-run -- =`'s start rows. Extend the existing parity test in
     `tests/cli/capture/pomodoro_name.rs`.
   - **Read-count gate:** a vault with 10 entries and 25 links spread over 4 notes reads
     each note exactly once per filter mode.
   - **Coarse wall-clock guard:** a September-shaped synthetic vault (24 entries, 78
     links over 6 notes) finishes in < 250 ms in the test build.
   - **Help:** add `-t, --tasks` to `tests/cli/help_options.rs`.
6. **Goldens.** Add `tests/cli/capture/pomodoros_agenda.rs`. It builds synthetic vaults
   and compares JSON (with the temp root normalized to `/Users/test/bob`) against
   committed goldens in `tests/fixtures/capture_pomodoros/`. Setting
   `BOB_UPDATE_GOLDENS=1` regenerates them; reuse an existing golden mechanism if the
   tests already have one. The goldens:
   - `agenda-current.json`: Now with logs, a ledger note, a duplicate, a retired link,
     and a deferred link; Next; two Later entries; one missing link;
   - `agenda-nothing-running.json`;
   - `agenda-empty.json`;
   - `agenda-no-daily-note.json`;
   - `agenda-multiple-timed.json`;
   - `agenda-heavy.json`: the September shape.
7. **Docs.** Update `docs/capture.md` as §1 says.
8. **Measure.** Build release and run `hyperfine -N 'bob capture-pomodoros -t -f json'`
   against the live vault on athena. Record the mean and p95 in the bead note. The
   targets are ≤ 15 ms (today's ledger) and ≤ 30 ms (the heavy fixture vault, via `-b`).
   If a target is missed, profile and fix before closing.
9. `just check` passes.

## Phase `mac-agenda-models`: models, client call, fake bob, fixtures

Work in bob-mac-capture.

1. **Models.** `Sources/CaptureCore/CaptureAgendaModels.swift`:
   - `CaptureAgendaSnapshot: Decodable, Equatable, Sendable, SchemaVersioned`;
   - `CaptureAgendaPomodoro`, `CaptureAgendaItem`, `CaptureAgendaLine`,
     `CaptureAgendaCompletedSummary`;
   - enums `CaptureAgendaRole`, `CaptureAgendaLineKind`, `CaptureAgendaLog`,
     `CaptureAgendaResolution`, `CaptureAgendaMarker`, each with an `.other(String)`
     case or a safe default for unknown values.
   - Every field uses `decodeIfPresent` with a default. Add
     `var currentTaskLinkCount: Int?`, matching `CapturePomodorosResponse`.
2. **Client.**
   `BobProcessClient.captureAgenda(previous: Data?) async throws -> CaptureAgendaFetch`
   (`.unchanged` or `.changed(snapshot:bytes:)`):
   - args `["capture-pomodoros", "--format", "json", "--tasks"]`, lane `agenda`,
     expected schema 1;
   - compare stdout bytes with `previous` before decoding;
   - map clap's unexpected-argument failure to
     `BobClientError.unsupportedOption("--tasks")` (a new case), or the closest existing
     pattern.
3. **fake-bob.** In the `capture-pomodoros` branch, when argv contains `--tasks`:
   - `cat` `FAKE_BOB_AGENDA_FIXTURE` (default `agenda-current.json`);
   - `FAKE_BOB_AGENDA_UNSUPPORTED=1` prints clap's
     `error: unexpected argument '--tasks' found` to stderr and exits 2;
   - `FAKE_BOB_AGENDA_FAIL=1` exits 1.

   Without `--tasks`, behavior is unchanged.

4. **Fixtures.** Copy the six bob-cli goldens into `Tests/Fixtures/`, plus a seventh,
   `agenda-yesterday.json`: `agenda-current.json` with `date` moved back one day.
5. **Tests** (`CaptureAgendaModelsTests`, `BobProcessClientTests`):
   - every fixture decodes;
   - unknown enum values and missing fields decode safely;
   - a legacy payload without `--tasks` fields decodes with empty items;
   - the wrong schema is rejected;
   - identical bytes return `.unchanged`;
   - the unsupported path maps correctly;
   - argv is recorded.

## Phase `mac-agenda-store`: store, triggers, watcher, count

Work in bob-mac-capture. Runs in parallel with `mac-agenda-planner`; touch only the
files below.

1. **Pure pieces in CaptureCore.**
   - `CaptureAgendaRefreshFilter`, per §2.
   - `CaptureAgendaRefreshState`: single-flight plus one follow-up, the generation
     counter, last bytes, `capability` (`.unknown`, `.supported`, `.unsupported`), and
     the date guard via an injected `today: () -> String`.
   - Table-driven tests for both on Linux.
2. **`VaultTargetWatcher`.** Add path and flag passthrough and the `VaultChangeBatch`
   aggregation (§2). Keep the existing callback semantics for the targets cache.
3. **`CaptureAgendaStore`** (`@MainActor`, `ObservableObject`, app target):
   - `@Published` `snapshot`, `status`, and `lastRefreshedAt`;
   - `refresh(reason:)`, `vaultDidChange(_:)`, and `reset()`;
   - the wake, unlock, day-changed, and clock-changed observers (inject the notification
     centers, as `RefsLibrary` does);
   - the old-bob fallback.
4. **Wiring (`AppDelegate`, `CapturePanelController`, `CapturePanelModel`):**
   - prefetch at launch;
   - feed every watcher batch to the store;
   - `show()` calls `agendaStore.refresh(reason: .show)` instead of
     `model.refreshCurrentPomodoroTaskLinkCount()`, and the old method's spawn is
     removed;
   - refresh after a successful submit;
   - reset on Recheck Bob and on the bob settings changes;
   - the model derives `currentPomodoroTaskLinkCount` from the store (§2).

   Keep `setCurrentPomodoroTaskLinkCountForTests` working.

5. **Tests:**
   - store refresh coalescing;
   - an unchanged-bytes no-op (no publish);
   - a failure keeps the snapshot, sets the status, and nils the count;
   - yesterday's snapshot is hidden;
   - unsupported falls back to the plain call and is not retried until reset;
   - a watcher batch with `.git/` paths does nothing, and one with a `.md` path
     refreshes;
   - a show triggers exactly one bob call, and the close-comma tests still pass (update
     `CaptureCloseTaskCommaTests` to drive the count through the store).
6. **README.** Update the `capture-pomodoros` freshness paragraph (around the current
   lines 328–336) to describe the new triggers. Leave the full agenda section to the
   polish phase.

## Phase `mac-agenda-planner`: presentation, inline text, planner

Work in bob-mac-capture, CaptureCore only. Runs in parallel with `mac-agenda-store`.

1. `CaptureAgendaPresentation` (§3), including duplicates, struck closed tasks, warning
   rows, the retired line, marker captions, truncation rows, session notes, the
   multiple-timed warning, the states (§6), every string, and row keys.
2. `CaptureAgendaInlineText` (§3), built on `TaskDisplayText`'s scanner without changing
   `TaskDisplayText`'s behavior for its existing callers.
3. `CaptureAgendaLayoutMetrics` (all constants), `CaptureAgendaBudget` (§4), and
   `CaptureAgendaClock.remainingText` (§7).
4. `CaptureAgendaFitPlanner.plan(presentation:heights:budget:expanded:) -> CaptureAgendaPlan`.
   `heights` is a lookup from `CaptureAgendaRowKey` to height. The plan exposes:
   - per-group and per-task states;
   - the strip's member groups;
   - `totalHeight`, `overflows`, and `hiddenCount` (for the `N more` cue);
   - an ordered list of render rows the view iterates without branching on folding
     logic.
5. **Tests** (`CaptureAgendaPresentationTests`, `CaptureAgendaInlineTextTests`,
   `CaptureAgendaFitPlannerTests`), table-driven with synthetic heights:
   - every ladder step in order;
   - farthest-first within a step;
   - skipping groups with nothing to fold;
   - duplicates are full once;
   - expansions are pinned, and overflow follows;
   - the strip upper bound;
   - empty days;
   - determinism, including identical plans from identical inputs;
   - the two acceptance fits (§10), using the September-shaped fixture with a row height
     model of 19 pt per row, 15 pt per extra wrapped line, and about 105 characters per
     line;
   - countdown wording at the boundaries.

   Run `swift test` on Linux.

## Phase `mac-agenda-view`: view, measurer, integration, eye line

Work in bob-mac-capture.

1. **Views.** `Sources/BobMacCapture/CaptureAgendaView.swift`, with row views per §6,
   rendering `CaptureAgendaPlan` rows. The views use only `CaptureAgendaLayoutMetrics`
   for spacing and sizes.
2. **Measurer.** `CaptureAgendaRowMeasurer` (`@MainActor`) keeps one reusable offscreen
   `NSHostingView` whose `rootView` is swapped per row. It returns `fittingSize.height`
   at the agenda's content width, is cached by `(row key, width rounded to pixel)`, and
   is bounded (clear it on snapshot change when the cache exceeds 2,000 entries). It
   measures the same row views the agenda renders.
3. **Model wiring.**
   - `agendaVisible` (§2/§3 conditions: setting on, blank draft, nothing else owning the
     region, `previewState == .idle`, a snapshot or a state line to show);
   - `agendaPlan`, `agendaExpanded`, and `expand(_:)`;
   - on snapshot publish, width change (at `windowDidEndLiveResize`), or screen change:
     measure what is missing, plan, and publish;
   - while hidden, the controller settles the layout (§5).
4. **Panel integration.**
   - `hasAuxiliaryContent` includes `agendaVisible`.
   - `auxiliaryContent` renders the agenda pane where `PreviewPane` would be.
   - `CapturePanelContentHeightPolicy` caps the agenda's ideal height at the budget.
   - The `onChange` resets also cover the agenda.
5. **Eye line.**
   - Add `CapturePanelPlacement` and replace the show-time `panel.center()` with it
     (§5).
   - Update the controller tests that assumed re-centring, and add tests: the top is
     identical across shows with and without the agenda; it is identical with a
     retained-draft preview; and typed previews still grow downward and slide up only at
     the bottom.
6. **Settings.** Add `AppSettings.agendaEnabled` and the Settings "Agenda" section
   toggle (§9), observed live from `AppDelegate`.
7. **Tests.**
   - `CaptureAgendaHeightConsistencyTests`: for the current, heavy-folded, and expanded
     fixtures at 724 pt and 584 pt content widths, hosted `fittingSize` equals
     `plan.totalHeight` ± 2 pt.
   - `CaptureAgendaDesignTests`: render `agenda-current`, `agenda-nothing-running`,
     `agenda-heavy` (folded at the 533 pt budget), `agenda-empty`, and
     `agenda-multiple-timed` at 760 and 620 widths, light and dark, through
     `RenderFixtureWriter` as `agenda-*.png`.
   - Model tests for the visibility conditions and the settle-while-hidden metrics.
8. Review the PNGs from CI (ground rules), and iterate on spacing and type until they
   match §6 and look calm and deliberate.

## Phase `mac-agenda-polish`: transitions, countdown, states, a11y, signposts, README

Work in bob-mac-capture.

1. The dim-hold and the cross-fade (§7), including the preview settled-height reset at
   the swap. Add model tests for every hold-release path and for blank-draft
   cancellation.
2. The countdown (§7), with `TimelineView` only while visible.
3. The stale title-row marker, "Loading today…", the multiple-timed warning row, and the
   Settings diagnostic line (§6, §9).
4. Accessibility (§8). Assert the labels in model or presentation tests where possible.
5. Signposts (§10).
6. Add the show-path spawn guard test (§10) if `mac-agenda-store` did not already.
7. Add design fixtures for the stale and overdue states.
8. **README.**
   - Rewrite the principle at the current lines 290–292: "A fresh popup is a compact,
     Spotlight-like bar … no empty preview placeholder, no dead space". It becomes "a
     compact bar whose empty-draft auxiliary region shows today's agenda; with the
     agenda off or nothing to show, it is the compact bar exactly as before".
   - Fix the "re-centred on show" wording to describe the fixed eye line.
   - Add `## Idle agenda`, covering:
     - what it shows, and the roles and numbers;
     - freshness and caching (A1);
     - the fold ladder and chips;
     - the eye line;
     - transitions;
     - the setting and the diagnostics;
     - the old-bob behavior;
     - signposts.
9. Review the PNGs from CI.

## Phase `closeout`: decisions record, verification, follow-ups

Work in bob-cli. Open bob-mac-capture only to read.

1. **Coherence.** Read `docs/capture.md`'s `--tasks` section and the Mac README's
   `## Idle agenda` end to end. Make sure the field names, the numbering rules, and the
   freshness story agree across the two repos.
2. **Memory decision `agenda_decision_memory`:**

   > [!decision] agenda_decision_memory Write the `idle-capture-shows-ledger-agenda`
   > decision strand described below.

   > [!decision] agenda_decision_memory = no Write no memory. Handle the off case as
   > described below.
   - **If accepted:** use `/sase_memory_write` to add the `decisions` strand
     `idle-capture-shows-ledger-agenda`.
     - Keyword: "An Empty Capture Draft Shows Today's Ledger Agenda".
     - Roster summary, a rule: "An empty capture draft shows bob's
       `capture-pomodoros --tasks` agenda (Now, Next, Later), cached in app memory and
       revalidated on filtered vault events and show; folds go farthest-first (logs,
       one-line, one-row, name strip) below a fixed eye line; the app never reads the
       vault."
     - It follows the decisions-web template: Applies to, Claim, Why (with the rejected
       alternatives from the report: a daily-file cache key, a global fold ladder, a
       horizon cap, Swift-side vault reads, building on `bob plan`, and a fixed-height
       scroll list), Cost, Reopens when, and Evidence. The evidence is the research
       report, this plan, and the phase commits.
     - Link `[[mac-capture-is-a-thin-client]]` and `[[today-is-read-from-the-ledger]]`.

     Then run `sase memory init`.

   - **If a human turned it off:** do nothing.
   - **If `%auto` left it off:** record a `PROPOSED FOLLOW-UP:` note on this phase's
     bead.

3. **Final checks:**
   - bob-mac-capture CI is green on the final `master`;
   - download `render-fixtures` and review every `agenda-*` PNG one last time;
   - `just check` passes in bob-cli;
   - Linux `swift test` passes on a scratch copy;
   - `bob capture-pomodoros -t` (human and JSON) looks right against the live vault.
4. **Proposed follow-ups.** Record each as a `PROPOSED FOLLOW-UP:` note on this phase
   bead; do not create beads.
   - Click a task row to open it in Obsidian (`ObsidianOpenURL`).
   - Hold ⌥ to see every task at full detail in a scroll view.
   - ⌘1…9 to insert the Nth link into the draft.
   - Plan-budget capsules in the title row.
   - `sources[]` stat fingerprints, only if Mac signposts show unfiltered refreshes cost
     something.
5. **Leave Bryan the checklist below** in the final report. State plainly what was and
   was not verified.

## Manual verification for Bryan (after installing the new bob on the Mac and `just install` in bob-mac-capture)

1. Open the panel: the agenda is there in the first frame, and the editor sits exactly
   where the compact bar used to.
2. Edit a linked task's sub-bullet in Obsidian, wait about a second, and reopen: the
   change is shown.
3. Type a character: the agenda dims, then swaps for the preview with one resize. Delete
   it: the agenda returns instantly.
4. `=x`, `=`, and `=#name~K` numbers match the agenda's badges.
5. Queue many placeholders in a scratch daily note (`BOB_DAY_FILE`, or a test vault via
   Settings): the fold order is logs, then one-line, then one-row, then the strip, and
   nothing scrolls. Clicking a chip expands that unit.
6. Light, dark, Increase Contrast, Reduce Transparency, and Reduce Motion all look
   right. The countdown ticks and turns orange when overdue.
7. Turn the Settings toggle off: the panel is the compact bar exactly as before.
8. Optional: Instruments' `agenda-*` signposts show no work on `panel-order`.

## Out of scope

- Anything that writes the vault from the agenda.
- Keyboard navigation into agenda rows, row clicks, ⌥ full-detail view, and ⌘1…9
  insertion (closeout follow-ups).
- Plan-budget capsules.
- A disk snapshot, a daemon, or FFI. The app is resident and prefetches at launch.
- Completed Pomodoros in the agenda.
