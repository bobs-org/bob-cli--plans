---
tier: tale
size: medium
title: 'Finish and land the Rotten Log epic (bob-cli-as): tooltip line, missing tests,
  card header fix'
goal: The freshness mark tooltip shows the Rotten Log line exactly as docs/freshness.md
  §11 specifies (wording, date, truncation, placement, query rows untouched, bounded
  scan), with tests and a bob-ledger-tools release. The missing nav grammar and Rust
  consequence tests exist, and the decay card header can report an unreadable log.
  Epic bob-cli-as is closed and its plan file is marked done.
proposed_by: bbugyi200.athena.bob-cli-as.land
proposed_with: claude/opus
bead: bob-cli-as
status: done
---

- **PARENT:**
  [202610/rotten_log.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/rotten_log.md)
- **BEAD:**
  [bob-cli-as](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-as/README.md)

# Finish and land the Rotten Log epic (bob-cli-as)

## Why

Epic `bob-cli-as` (plan `plan:202610/rotten_log.md`) has all nine phases closed. The
land agent checked every phase against the plan and the code. Most of it is done and
verified:

- the bob-cli registry, agenda JSON, and snapshots (`just check` passes);
- the nav grammar (the marker parser matches every Rust vector);
- the single, card, and batch keep flows (each has focused tests);
- the Mac chip;
- the docs.

Phase `bob-cli-as.7` (mark-tooltip) does not match the plan, and its docs already
describe the planned behavior. Three earlier phases also left smaller gaps.

What is already done needs no further work:

- Follow-up triage. bob-cli-6v, bob-cli-7i, and bob-cli-3w were corroborated, and
  bob-cli-6w got a note. No new task beads were needed.
- The nav-grammar recogniser audit. block-id-prompt and task-status-cycler use only
  kind-specific Schedule/Work finders, so no change is needed.

Both are recorded as notes on `bob-cli-as`. Do **not** redo them.

Linked repos: open `bob-plugins` with `/sase_repo`
(`sase repo open bob-plugins -r "..."`) and read its `AGENTS.md` first. Plugins with
`src/fragments.json` are built from `src/`: never hand-edit `main.js`, run
`npm run build`, and keep each fragment at 1000 lines or fewer. After any bob-plugins
change, run `bob plugins sync`.

## Gaps found at landing

1. **bob-ledger-tools tooltip line**
   (`plugins/bob-ledger-tools/src/135-freshness-rotten-log.js`,
   `250-plugin-freshness-mark-model.js`, `260-plugin-freshness-mark-render.js`, commit
   `f5557ec`). It differs from the plan and from `docs/freshness.md` §11 in bob-cli:
   - "1 keeps logged" should read "1 keep logged".
   - The date is the raw `2026-10-16`, not `Oct 16`.
   - The reason is not truncated to 48 characters.
   - The line is appended after the action line ("Alt+F to confirm"/"Alt+F to decide").
     It belongs before that line.
   - Live Preview runs `doc.toString()` on every decoration build, then splits the whole
     note once per mark.
   - The entry regex accepts `_date_` and bare dates, but nav never counts those.
   - Reading view excludes only Dataview containers, not Tasks query results.
   - There are no tests, no version bump (still 1.64.0, which is the Alt+P footer
     release), and no update to the manifest description or `docs/bob-ledger-tools.md`.
2. **Nav grammar tests** (phase `bob-cli-as.3`). The plan asked for a dedicated
   `scripts/test-navigation-rotten-log.cjs` in `npm test`. Only one combined test was
   added, inside `test-navigation-hotkeys-project-from-task.cjs`.
3. **Decay card header** (phase `bob-cli-as.5`). `buildFreshnessDecayRottenHeaderLine`
   in `plugins/bob-navigation-hotkeys/src/195-rotten-keep.js` returns `""` when
   `entries` is empty, before it checks for ambiguity. `readRottenLog` returns
   `{status: "unknown", entries: []}` for duplicate markers, so the card can never show
   `🥀 log unreadable`. The card's Keep stage subtitle also uses the generic
   `kept N in a row · …` copy instead of the planned
   `Kept N reviews in a row · this keep makes N+1`.
4. **Rust consequence tests** (phase `bob-cli-as.2` item 4). No test shows that a
   captured sub-bullet lands above an existing Rotten Log, or that `capture_work_log`
   ignores a Rotten marker.

## Steps

### A. bob-ledger-tools: make the tooltip line match the contract

The contract is the "Rotten-log line" bullet in bob-cli `docs/freshness.md` §11 and the
plan's mark-tooltip phase.

1. **Parser parity.** Make the line-based recogniser in `135-freshness-rotten-log.js`
   byte-compatible with nav's, and add a comment naming the source of truth:
   `ROTTEN_LOG_PARENT_RE` / `ROTTEN_LOG_ENTRY_RE` in
   `plugins/bob-navigation-hotkeys/src/010-requires-and-config.js`, and `readRottenLog`
   / `findManagedTaskLogParents` in `030-bullet-blocks-and-logs.js`.
   - **Marker.** Any list marker (`-`, `*`, `+`, `1.`, `1)`). An optional `🥀`, then an
     optional U+FE0F, then at least one space or tab. Then `**ROTTEN LOG**` or
     `**Rotten log**`, with the optional colon that nav accepts, and only trailing
     spaces or tabs.
     - `🗓️ **ROTTEN LOG**` is not a marker.
     - `🥀**ROTTEN LOG**` (no space) is not a marker.
     - `**rotten log**` and `**Rotten Log:**` are not markers.
   - **Entry.** Only direct children of the marker count, and each must be exactly
     `*YYYY-MM-DD* — <reason> · kept N×` with N from 1 to 999. Drop the `_`/bare-date
     leniency.
   - **Counting.** Two or more direct-child markers → no line ("unknown"). A missing log
     or a log with no valid entries → no line.
2. **Formatter.** Add `freshnessMarkRottenLogLine(history, today)`. It returns:
   - `🥀 N keeps logged · last <date> — <reason>`, or `🥀 1 keep logged · …` for one
     entry;
   - `<date>` as `Oct 16`, using the same `FRESHNESS_MARK_MONTHS` names that
     `freshnessShortDate` uses but without the weekday, with `, YYYY` appended when the
     year differs from today's;
   - the newest entry's reason, truncated to 48 characters plus `…` (count code points,
     so an emoji is never split);
   - `🤷 no reason given` exactly as written;
   - `null` when there is nothing to show.
3. **Placement.** The tooltip order must be:
   1. line 1;
   2. line 2;
   3. the keeps line;
   4. **the rotten line**;
   5. the action line.

   The mark's `aria-label` is built from `model.tooltip`, so it then includes the line.
   One way to do this: let `freshnessMarkModel` take an optional `rottenLogLine` input,
   or put a small helper on the model that splices the line before the action line.
   Query-row consensus candidates must never receive the line, so
   `freshnessMarkConsensus` and Tasks/Dataview marks stay byte-identical.

4. **Live Preview performance.** In `260-plugin-freshness-mark-render.js`, remove the
   per-build `doc.toString()` and the per-mark full-note split. Read only the task's
   child block, and only for lines that produce a mark. Use the CodeMirror `Text` API
   (`doc.line(n)`, `doc.lines`), starting at the task line. Stop at the first nonblank
   line indented at or above the task's indentation.
5. **Reading view.** Keep the DOM-based reader (or switch to `getSectionInfo` source
   lines if that is simpler). It must count the same rendered entries nav counts.
   - Extend `freshnessMarkInTaskQuery`, or an equivalent guard, so that Tasks plugin
     query results are excluded explicitly, as well as Dataview. Include
     `block-language-tasks` and `plugin-tasks-query-result` containers.
   - Tasks query rows, the dashboard, and `rotten.md` results never show the line.
6. **Tests.** Extend `scripts/test-ledger-tools-freshness-mark.cjs` and
   `scripts/test-ledger-tools-freshness-mark-surfaces.cjs`, which are already in
   `npm test`, to cover:
   - wording for 0, 1, and many entries;
   - the date format, including the different-year case;
   - 48-character truncation;
   - `🤷` shown as is;
   - an ambiguous log, and invalid or nested entries that are not counted;
   - the mismatched-emoji and no-space marker vectors;
   - placement before the action line, and the line present in `aria-label`;
   - the Live Preview surface and the reading-view surface;
   - Dataview and Tasks query-row marks byte-identical with and without a Rotten Log
     under the task;
   - a performance bound, for example a 5,000-line child block and many marked tasks
     building decorations within a generous time limit, with `doc.toString` never
     called.
7. **Version and docs.**
   - Bump `plugins/bob-ledger-tools/manifest.json` from 1.64.0 to **1.65.0** (minor: new
     behavior).
   - Append one sentence to the manifest description, for example: "In source-note Live
     Preview and Reading view, the freshness mark tooltip adds a
     `🥀 N keeps logged · last <date> — <reason>` line from the task's Rotten Log; Tasks
     and Dataview query rows never show it."
   - Add the same behavior to the freshness-mark bullet in `docs/bob-ledger-tools.md`.
   - Run `npm run docs:gen` so `docs/index.md` and the plugin page tables pick up the
     version and description.

### B. bob-navigation-hotkeys: grammar tests and the card header

1. **New test file.** Create `scripts/test-navigation-rotten-log.cjs` and add it to the
   `npm test` list in `package.json`. It covers:
   - **Marker vectors**, copied verbatim from the Rotten cases in bob-cli
     `src/native/capture/tests/assembly.rs`. Accepted spellings: `-`, `*`, `+`, `1.`,
     and `2)` markers; emoji and no emoji; VS16; colon; legacy `Rotten log`. Rejected
     spellings: trailing text, lowercase, `Rotten Log:`, `🗓️`/`🛠️` + ROTTEN, `🥀` +
     SCHEDULE/WORK, `🥀**ROTTEN LOG**`, and a task line containing the marker.
   - **Entry parsing.** Valid entries parse. `kept 0×`, `kept 1000×`, and a missing tail
     are rejected.
   - **`readRottenLog` statuses.** `missing`, `empty`, `ok`, and `unknown`. Hand-note
     children and nested bullets are not counted.
   - **`planManagedTaskLogEntry`.**
     - A new log is created as the last direct child, after nested child tasks.
     - A new entry is prepended under an existing marker, inheriting its entry indent
       and list marker (also with space-indented and `*` lists).
     - CRLF content and a missing final newline both work.
     - A `❌ **CANCEL LOG**` or `⛓️ **DEPENDS ON:**` first child stays first.
     - Ambiguous markers return `ambiguous-markers`.
   - **Work Log regression.** `insertLaneWorkLogEntry` output stays byte-identical on a
     task that also has a Rotten Log. A Work entry is never filed under the Rotten
     marker.
2. **Card header.** In `buildFreshnessDecayRottenHeaderLine`, check the
   `unknown`/`ambiguous` status before the empty-entries early return, so the card
   header shows `🥀 log unreadable`. Add a test to
   `scripts/test-navigation-decision-card-keep.cjs`.
3. **Card Keep stage subtitle.** When the stage is mounted by the decay card, the streak
   line reads `Kept N reviews in a row · this keep makes N+1`, followed by the
   logged-count text (`M keeps logged` / `none logged yet` / `log unreadable`). The
   Alt+F refresh-modal copy stays unchanged. Use an option on
   `buildRottenKeepPromptModel` that `060-decay-card-modal.js` passes. Update the
   stage/model tests.
4. Bump `plugins/bob-navigation-hotkeys/manifest.json` from 2.56.0 to **2.56.1**
   (patch). Then run `npm run docs:gen`.

### C. bob-cli: Rust consequence tests

1. In `src/native/capture/tests/assembly.rs`, add Rotten cases next to
   `finds_the_earliest_direct_child_managed_log`:
   - on a task with only a Rotten Log, the anchor is the Rotten marker;
   - with a Rotten Log before a Work Log, the anchor is the Rotten marker.
2. In the `src/native/capture/task_blocks.rs` tests, mirror
   `plain_insertion_reports_one_added_row_before_schedule_log`. A plain capture
   sub-bullet to an existing task that has a `🥀 **ROTTEN LOG**` lands above the Rotten
   marker.
3. In the `src/native/capture_work_log.rs` tests, cover a task with only a Rotten Log. A
   new `🛠️ **WORK LOG**` marker is created as the last direct child, and the Rotten Log
   is untouched.

### D. bob-cli docs: shipped versions

In bob-cli `docs/freshness.md`, change the ledger-tools version from 1.64.0 to
**1.65.0** in three places:

- the §8 surface table row "freshness mark tooltip rotten line";
- the §11 "Rotten-log line (shipped ledger-tools …)" bullet;
- the §13 2026-10-23 rollout entry.

In that same rollout entry, add nav 2.56.1 (card header fix). Then run
`just docs-check`.

### E. Verification

- **bob-plugins.** Run `npm run build`, `npm test`, `npm run docs:check`, and
  `bob plugins sync`.
  - The full `npm test` baseline at master `f5557ec` is 3505/3510 passing. The five
    known pre-existing failures are tracked elsewhere: bob-cli-6v (the four
    unblocked-glyph tests) and bob-cli-7i ("forward conversion carries a Depends-On line
    and reverse restores it").
  - bob-cli-3w's stage-ranker timing test may also flake under load.
  - Any other failure is yours to fix. Never change those beads' statuses.
- **bob-cli.** Run `just check` and `just docs-check`. Do not run `just check-full`.
- **Live check.** If Obsidian is running on athena, check the tooltip in Live Preview
  and Reading view. Otherwise say on the epic that it was not live-verified.

### F. Epic closeout (final step: do it in this same turn)

`bob-cli-as` has no `parent_bead`, so the landing ends with this epic.

1. Run `sase bead epic-symbols bob-cli-as`. At landing it reported none. If entries
   appear, resolve each one: wire it up, privatize it, add a non-test pragma, or delete
   it. Do not leave any behind.
2. Close the epic. Never use `--force`. Run:

   ```bash
   sase bead close bob-cli-as --note "<verification>"
   ```

   The note should summarize:
   - **Land verification.** All nine phases were checked against the plan and the code.
     bob-cli `just check` passes. bob-plugins `npm test` has only the known 6v/7i
     failures. The nav marker grammar matches every Rust vector. The
     block-id-prompt/task-status-cycler/ledger audit needed no change. No commits landed
     outside the epic in bob-cli, bob-plugins, or bob-mac-capture that needed
     integration (the bob-plugins quick-roll fix `ceb311b` predates the nav phases,
     which were built on it; bob-mac-capture `3343c5e` is unrelated).
   - **Follow-up triage.** 6v, 7i, and 3w were +1'd; 6w got a note; no new tasks.
   - **Fixes in this tale.** Summarize sections A–D and their verification results.

3. If `just symvision` exists in bob-cli, run it to confirm the whitelist is clean. At
   landing time bob-cli's `justfile` has no such recipe; if so, say so in your final
   report.
4. Set `status: done` in the frontmatter of the epic's plan file. That is the PLAN path
   shown by `sase bead read bob-cli-as` (`plan:202610/rotten_log.md`, at
   `sase/repos/plans/202610/rotten_log.md` relative to the bob-cli checkout). Today it
   says `status: wip`.
