---
tier: epic
title: 'Command-mode sequences: mark with Ctrl+M, run in order'
goal: 'In Bob Command Mode, Ctrl+M marks commands into an ordered sequence that Enter
  or Ctrl+Enter runs one at a time. The status-bar bob chip shows how many steps there
  are and which command is running. RECENT shows each past sequence as one row that
  runs it again.

  '
decisions:
  attention_continues:
    ask: Should a step that ends Needs attention (for example 'Already up to date')
      let the sequence continue?
    default: true
    why: Attention mostly means exit 0 with a note; only failed, unknown, and stopped
      halt, like &&.
    answer: true
  escape_clears_marks:
    ask: Should Esc first clear pending marks, so a second Esc closes the sheet?
    default: true
    why: One stray Esc should never both drop a built sequence and close the sheet.
    answer: true
  rerun_last_sequence:
    ask: 'Should Bob: Rerun last command rerun the whole sequence when the newest
      run was a sequence step?'
    default: true
    why: 'A sequence is one unit everywhere else: RECENT, the chip, and the summary
      toast.'
    answer: true
phases:
- id: engine
  title: Sequence engine, record stamps, and plugin wiring
  depends_on: []
  size: medium
  description: 'engine: add the pure step and history model, stamp step run records,
    add the in-order driver that stops on failure, and wire the plugin to run sequences,
    suppress per-step toasts, and rerun the last sequence.'
- id: feedback
  title: Status chip, summary toast, and pager step label
  depends_on:
  - engine
  size: medium
  description: 'feedback: teach the status-bar chip to show step N/M and the running
    command, replace per-step toasts with one summary card (Output, Resume, Rerun
    all, Copy), label sequence steps in the pager, and add preview frames.'
- id: sheet
  title: Marking, tray, review card, and RECENT sequence rows
  depends_on:
  - engine
  - feedback
  size: medium
  description: 'sheet: add Ctrl+M marking in list and argument stages, the numbered
    sequence tray, Enter and Ctrl+Enter routing with the review card, Backspace and
    Esc rules, and the grouped RECENT sequence row that reruns as one unit.'
- id: docs
  title: Plugin reference, user guide, and deploy
  depends_on:
  - sheet
  size: small
  description: 'docs: document sequences in the bob-plugins plugin reference and the
    bob-cli guide, run the full plugin and docs gates, and deploy with bob plugins
    sync.'
proposed_by: bbugyi200.athena.16k
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-10-30 17:30:02
status: done
bead_id: bob-cli-er
---

- **PROMPT:** [prompts/202610/command_mode_sequences.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202610/command_mode_sequences.md)
- **BEAD:** [bob-cli-er](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-er/README.md)

# Plan: Command-mode sequences

All code lives in the `bob-command-mode` plugin in the linked **bob-plugins** repo
(`plugins/bob-command-mode/`). Open it with your `/sase_repo` skill
(`sase repo open bob-plugins -r "<why>"`) and read its `AGENTS.md` first. Rules from
that file apply to every phase:

- Edit `src/` fragments only. Run `npm run build`, and never hand-edit `main.js`.
  `src/fragments.json` sets the order. All fragments share one script scope. A
  hand-edited fragment must stay at or below 1000 lines, so new code goes into new
  fragments and is added to `fragments.json`.
- Run `npm run build:check` and the relevant
  `node --test scripts/test-command-mode-*.cjs` files. Before you finish, run the full
  `npm test` and `npm run validate`.
- After changing that repo, run `bob plugins sync`.

There are no bob-cli Rust changes. The `bob __catalog` contract, the run-record schema
number (`CM_RECORD_SCHEMA` stays 1; the new field is optional), and `api.commandMode` v1
stay as they are. "The footer `bob` indicator" means the status-bar chip
(`120-status-chip.js`), the spinning ring plus `bob` plus a clock.

## The experience

### Build a sequence

- `Ctrl+M` marks the selected row as the next step. Pressing `Ctrl+M` on a marked row
  unmarks it, and the remaining steps renumber. As with `Ctrl+N`/`Ctrl+P`, match any
  letter case or layout (`key === "m"` or `code === "KeyM"`), and require Ctrl alone:
  `Ctrl+Shift+M` and `Ctrl+Alt+M` are not marks.
- The highlight stays on the row so its new number badge is visible right away. The
  query text becomes fully selected. Typing replaces it to start the next search, while
  `Ctrl+N`/`Ctrl+P` and `Ctrl+M` still work on the current matches. A second `Backspace`
  is needed before the empty-query rules apply.
- Marked rows swap their icon for a filled accent circle holding the step number, and
  get a faint accent tint.
- A **sequence tray** appears directly under the input whenever anything is marked. It
  has a small `SEQUENCE` label, then numbered pill chips in mark order joined by faint
  `›` separators, then a right-aligned `Run N ↵` button for mouse users. Each chip shows
  the command words in monospace, is truncated with an ellipsis, and has a `title`
  holding the step title and full `bob …` argv. Hovering shows a `×` that removes the
  step. New chips fade in over about 140 ms, with no animation under
  `prefers-reduced-motion`. The tray wraps onto more lines instead of scrolling.
- Steps that can be marked are bob command rows: catalog entries, FOR THIS NOTE and FROM
  CLIPBOARD context rows, the typed `Run typed bob command` row, and RECENT single runs
  that were not previews. All other rows refuse with a warning hint and do not change
  the tray:
  - surface, plain, and Vim rows: "Only bob commands can join a sequence"
  - unavailable rows: their reason
  - refused typed commands: the exclusion reason
  - RECENT previews: "Previews can't join a sequence"
- A row that still needs positionals opens argument mode, just as `Enter` does. In
  argument mode, `Ctrl+M` marks the typed command once it is complete; otherwise the
  hint is `Needs NAME`. The sheet then returns to the list with the query selected.
- Step identity is the step signature (request argv plus entry id), not the row key. A
  RECENT `plan` row and the catalog `plan` row are therefore one step, and both show the
  badge. A sequence never holds the same command twice.
- `Ctrl+M` on a RECENT **sequence** row loads its steps. If every one of its steps is
  already marked, it unmarks them all. Otherwise it appends the missing steps in order.
  Pressing it twice therefore undoes it.
- Marks belong to the open sheet. They are never saved, and closing the sheet drops
  them, so a stale tray can never run later by surprise.

```
 bob                                                       athena · bob 0.9.3
 ❯ plan▌                                    ← query selected after a mark
 SEQUENCE  (1) vault-sync run  ›  (2) task reconcile  ›  (3) plan    Run 3 ↵
 ────────────────────────────────────────────────────────────────────────────
 DAILY WORKFLOW
 (3) plan                Today's plan                           reads  1.2s
  ⓘ  plan budget         Plan budget                            reads
 ────────────────────────────────────────────────────────────────────────────
 ⌃M unmark · ↵ run 3 · ⌃↵ run 3 now · ⌫ unmark last · esc clear marks
```

### Keys while marks are pending

| Key (list stage)              | No marks (unchanged)                         | Marks pending                                                                 |
| ----------------------------- | -------------------------------------------- | ----------------------------------------------------------------------------- |
| `Ctrl+M`                      | mark the selected row                        | toggle the selected row                                                       |
| `Enter`                       | run the selected row                         | run the **marked** steps; the review card appears first if any step would ask |
| `Ctrl+Enter`                  | run now                                      | run the marked steps now, without the review card                             |
| `Shift+Enter`                 | preview                                      | warning hint "⇧↵ previews one command — ↵ runs the sequence"; nothing runs    |
| `Backspace` on an empty query | close                                        | unmark the last step                                                          |
| `Esc` / `Ctrl+[`              | close                                        | see below                                                                     |
| click a row                   | activate                                     | toggle its mark (never runs it)                                               |
| `Ctrl`/`Cmd`+click            | toggle its mark (starts a sequence by mouse) | toggle its mark                                                               |
| `Tab`, `?`, `::`              | unchanged                                    | unchanged (`Esc` from help output returns to the list with marks intact)      |

> [!decision] escape_clears_marks With marks pending, `Esc`/`Ctrl+[` clears every mark
> and keeps the sheet open, with no warning hint. The tray disappearing is the feedback.
> A second `Esc` closes the sheet.

> [!decision] escape_clears_marks = no `Esc`/`Ctrl+[` closes the sheet even with marks
> pending, and the marks are dropped.

In argument mode with marks pending:

- `Ctrl+M` adds the command as the next step.
- `Enter`, when no completion is highlighted, adds the typed command if it is complete
  and not already marked, then runs the sequence.
- `Ctrl+Enter` does the same without the review card.
- `Shift+Enter` shows the same hint as in the list.

The argument footer says this explicitly:
`⌃M add as step 3 · ↵ add & run 3 · ⌃↵ add & run now · esc back`. With no marks,
argument-mode keys keep their current meaning, and the footer gains `⌃M mark`. The
default list footer also gains `⌃M mark`:
`⌃N/⌃P select · ↵ run · ⌃M mark · ⇥ args · ⇧↵ preview · ⌃↵ run now · ? help · :: vim · esc close`.

### Run it

- **One mark** runs exactly as if that row had been activated (`Enter` or `Ctrl+Enter`).
  That includes in-sheet report output, preview-first previews, and confirm cards.
  Nothing is stamped as a sequence. A sequence always has two or more steps.
- **Two or more marks** start a sequence. With `Enter`, if any step's catalog policy is
  `confirm` or `preview-first`, the sheet first shows the **review card**. This reuses
  the existing `confirm` stage and its amber styling:

```
 Run 3 commands in order
  (1) vault-sync run      Sync vault now        syncs
  (2) task archive        Archive done tasks    writes
      ⚠ asks first: Archive every done task?
  (3) ref rename a b      Rename a reference    syncs
      ⚠ applies without its preview
 Stops at the first failure · open notes save before each writer
 [ Run all ⌃↵ ]  [ Back ]
```

`Ctrl+Enter` or **Run all** confirms. `Esc` or **Back** returns to the list with the
marks intact. `Enter` alone does nothing on the card, the same as today's confirm card.
If no step is guarded, `Enter` starts right away.

- Starting closes the sheet and hands the steps to the plugin, so the sequence runs in
  the background. Each step goes through
  `runner.run(argv, {entryId, title, trigger: "sequence", skipGuards: true, sequence: stamp})`.
  Confirmation was already given once, by the review card or by `Ctrl+Enter`. The runner
  still saves every open Markdown note before each writer and refuses the step if a save
  fails.
- Steps run strictly in order. Step k+1 starts only after step k's `runner.run` promise
  settles, so reads in a sequence also run one at a time.
- The sequence halts at the first step whose outcome is `failed`, `unknown`, or
  `stopped`. An early runner refusal (no `runId`) counts as `failed` with its reason. No
  later step runs; each later step is reported as skipped.

> [!decision] attention_continues A step that ends `attention` (◐, for example
> `Already up to date` or `Intake is still queued`) lets the sequence continue. The
> summary toast then takes the attention tone.

> [!decision] attention_continues = no Any outcome other than `completed` halts the
> sequence, `attention` included.

- If the same sequence (same signature) is already active, it does not start again: the
  sheet stays open with the hint "That sequence is already running". This also guards
  against a quick double `Enter`.

### While it runs: the status-bar chip

```
 ◠ bob 2/3 Reconcile tasks 0:12          running (the ring spins as today)
 ⧗ bob 2/3 Reconcile tasks 0:12          current step queued behind another writer
 ✓ bob 3/3 Today's plan 0:42             settle, 1.6 s, then fade (as today)
 ✗ bob 2/4 Reconcile tasks 0:35          settle for a halted sequence
```

- The chip anatomy becomes ring, `bob`, **step** (`2/3`), **title** (current step
  title), clock. The step text is the accent color, weight 600, with tabular numbers.
  The title is muted, at most about 160 px, with an ellipsis.
- The clock shows the sequence's wall time, not the current step's time.
- The step and title nodes are created lazily on the first sequence paint, inserted
  before the clock, never removed, and hidden while no sequence is active. Chips that
  never show a sequence keep today's three-node anatomy.
- The existing rules still apply: the 400 ms reveal (measured from the sequence start),
  the overdue amber, the queued dashed ring, and `bob ×N` when several runs are active.
- Between steps the chip stays continuously visible, showing the next step's title. A
  step finishing inside an active sequence never triggers a settle, so the chip does not
  flicker between steps.
- The tooltip (`aria-label`, `data-tooltip-position="top"`) looks like this:

```
 bob · sequence 2 of 3 · 0:42

 ✓ Sync vault now · 0:30
 ● Reconcile tasks · 0:12
   bob task reconcile
   Last: reconciled 12 tasks
 ○ Today's plan

 Click to open output
```

Other active runs follow the sequence block in the existing format.

- Clicking the chip, or pressing `Enter`/`Space` on it, opens the current step's output
  in the pager. Between steps it opens the last finished step. During a settle it opens
  the sequence's **focus run**: the halting step if there is one, otherwise the last
  report step, otherwise the last step.

### When it finishes: one summary toast

Per-step toasts are suppressed for steps of an active sequence. If the pager is open on
such a step, it still refreshes. One card replaces them all:

```
 ✓ Sequence · 3 commands                         0:42
   ✓ Sync vault now                    0:30
   ✓ Reconcile tasks                   0:09
   ✓ Today's plan                      0:03
   [Output] [Rerun] [Copy]

 ✗ Sequence stopped at step 2 of 4               0:35
   ✓ Sync vault now                    0:30
   ✗ Reconcile tasks                   0:05
     Commit pending edits first
   ○ Today's plan                      skipped
   ○ Plan budget                       skipped
   [Output] [Resume from 2] [Rerun all] [Copy]
```

- **Durations and stickiness** reuse `CM_TOAST_META` by sequence outcome: 5 s for
  completed or stopped, 8 s for attention, and sticky for failed or unknown. One sticky
  card is kept per sequence signature.
- **Output** opens the focus run in the pager.
- **Resume from k** reruns the halted step and continues, reusing the same sequence id,
  so history shows one unit.
- **Rerun** / **Rerun all** starts the whole sequence again under a new id.
- **Copy** copies the steps as one shell line joined by `&&`, for example
  `bob vault-sync run && bob task reconcile && bob plan`. That matches the
  stop-on-failure meaning.
- **Click** on the card body opens Output.
- **Replays skip the review card.** Toast reruns and resumes skip it, just as today's
  replays skip their guards.
- **OS notification.** A sequence longer than 15 s that finishes while Obsidian is
  unfocused gets one OS notification under the existing setting. Clicking it opens the
  focus run.

### Run it again: RECENT

- RECENT folds every run record carrying the same `sequence.id` into one row. The row
  takes one of the four RECENT slots, and its steps never appear as separate RECENT
  rows. Rows are deduplicated by sequence signature, so running the same sequence again
  shows one row.

```
 RECENT
 ≡  vault-sync run › task reconcile › plan      3 steps         ✓  12m  0:42
    Sync vault now → Reconcile tasks → Today's plan          (about line when selected)
```

- **Icon:** Lucide `list-ordered`, with fallback `≡`.
- **Path:** each step's display words in monospace, with faint `›` separators.
- **Title:** `N steps`.
- **Chips:** the sequence glyph, the age, and the wall duration.
- **About line:** the step titles joined by `→`. For a halted sequence it reads
  `Stopped at step 2: <reason>`. For an interrupted one it reads
  `Interrupted after step 2 of 4`.
- **Glyphs:** `✓ ◐ ✗ ? ■` as today, `⟳` while running, and `◌` for **interrupted**. An
  interrupted sequence has fewer step records than steps, no halting outcome, and is not
  active, which is what a plugin reload mid-sequence leaves behind.
- **Enter / Ctrl+Enter** run the whole sequence again under its review rules, the same
  as marked steps. `Ctrl+M` loads its steps (see above). A RECENT sequence that is still
  running gives the "already running" hint.
- If marks are pending, `Enter` always runs the marks, never the highlighted RECENT
  sequence.

### Other surfaces

- **Pager:** the pager header, in both the standalone pager and the sheet's output
  stage, shows a small `step 2 of 3` label for a stamped run. `r` reruns only that step,
  `Y` copies only that step, and `[`/`]` still walk single runs.
- **Rerun last command:**

> [!decision] rerun_last_sequence **Bob: Rerun last command** reruns the whole sequence
> when the newest record carries a sequence stamp. Its steps come from the stamp, they
> run under a new id, and the review card is skipped as for every replay. Otherwise
> behavior is unchanged.

> [!decision] rerun_last_sequence = no **Bob: Rerun last command** keeps rerunning only
> the newest record, even when it is a sequence step.

- **Toast Retry on single runs** is unchanged.

## Architecture

### Step, stamp, and history

- **Step** (stored form): `{ argv, entry_id, title }`. `argv` is the _request_ argv that
  `runner.run` expects: the entry path plus user args, with no `bob` binary, fixed
  words, or preview words. This is the same shape `cmReplayRun` returns.
  - **Display words:** `entry.path + entry.fixed + user args`. Fall back to `argv` when
    the entry is gone.
  - **Title:** the catalog entry title when the entry exists, else the stored title,
    else the display text. The typed row's `Run typed bob command` label is never used
    as a step title.
- **Signature:** `JSON.stringify([argv, entry_id])` per step. A sequence signature is
  the JSON array of its step signatures.
- **Record stamp:** a step's run record gains an optional
  `sequence: { id, index, total, started_at, steps }`. `index` is 1-based, `id` is
  `"seq-" + cmRunId(...)`, and `started_at` is the sequence start as an ISO string.
  Every record carries the **full** step list. RECENT can therefore rebuild the sequence
  from any surviving record, even after retention sweeps some steps or a halt left later
  steps without records.
- **History** (`cmSequenceHistory(records, { activeIds })`, pure) groups records by
  `sequence.id`. Within a group, the newest record per index wins, which makes a resume
  read as one unit. Each group gets a state:
  - `running` if the id is active
  - else the first `failed`/`unknown`/`stopped` step outcome
  - else `attention`/`completed`, if every index has a record
  - else `interrupted`

  Wall duration is the newest `finished_at` minus the stamp's `started_at`.

- **Duplicates:** if a step's argv is already in flight elsewhere, the runner's existing
  `findDuplicate` piggyback still applies. That step's record has no stamp, which the
  model tolerates.

### Driver contract

`cmRunSequence(runner, steps, options)` is an async function that never rejects.

- **Options:**
  `{ id, startedAtMs, startIndex = 0, priorResults = [], trigger, onStepStart(i, step), onStepFinish(i, result), cancelled(), continues(outcome) }`.
- **Each step i:**
  1. Check `cancelled()`. If it is true, stop with outcome `interrupted`.
  2. Call `onStepStart`.
  3. Resolve `entryId`, but only when `entry_id` still exists in `runner.catalog()`.
     Otherwise run by argv so the exclusion guard applies, as `cmReplayRun` does.
  4. `await runner.run(...)` with the stamp.
  5. Build the result `{ index, runId, outcome, reason, durationMs }`, timed in the
     driver.
  6. Call `onStepFinish`.
  7. Halt unless `continues(outcome)`.
- **Hooks** are wrapped in try/catch. A hook error never stops the run or reaches the
  runner.
- **Return value** is a summary:
  `{ id, signature, steps, total, outcome, stoppedAt, results, startedAtMs, finishedAtMs, durationMs }`.
  - `results` has one entry per step. Steps never reached are
    `{ index, skipped: true }`.
  - `stoppedAt` is the 1-based index of the halting step, or `null`.
  - `outcome` is the halting outcome, or `interrupted`, or `attention` when any step was
    attention, or else `completed`.

### Plugin wiring

`BobCommandModePlugin` gains the following.

- **Active state:** a `commandModeSequences` map (id → live state: steps, current index,
  results, start time, signature).
- **`commandModeRunSequence(steps, opts)`:**
  - Refuse an empty list. Refuse a duplicate signature, returning
    `{ ok: false, reason: "already running" }` with a Notice.
  - Otherwise register the sequence, call the driver, unregister in `finally`, and call
    `commandModeSequenceFinished(summary)`.
  - Skip the finish handling when the plugin was unloaded, so there is no toast.
- **Read helpers:** `commandModeSequenceIsActive(signature)` and
  `commandModeSequenceSnapshots()`.
- **Notify wrapper:** wraps `commandModeNotify`. It returns null for a notice whose
  `record.sequence.id` is active, after refreshing a watching pager.
- **Chip-settle wrapper:** wraps `commandModeRecordChipSettle`. It skips runs whose
  record (`cmPagerFindRecord`) belongs to an active sequence.
- **`onunload` wrapper:** sets an unloaded flag that `cancelled()` reads. The in-flight
  detached child still finishes and is adopted as today. Steps after it never start.

### Fragment map

| Fragment                                        | Phase            | Role                                                                                |
| ----------------------------------------------- | ---------------- | ----------------------------------------------------------------------------------- |
| `050-run-store.js` (edit)                       | engine           | `cmNewRecord` copies `options.sequence` when given                                  |
| `056-sequence-model.js` (new, after 050)        | engine           | pure step, signature, stamp, guards, history, focus run, copy text                  |
| `060-runner.js` (edit)                          | engine           | `runInner` passes `opts.sequence` to `cmNewRecord`                                  |
| `065-sequence-runner.js` (new, after 060)       | engine           | `cmRunSequence`                                                                     |
| `090-plugin.js` (edit)                          | engine, feedback | `rerun-last` calls `commandModeRerunLast()`; chip refresh passes sequence snapshots |
| `095-plugin-sequences.js` (new, after 090)      | engine, feedback | plugin prototype patches listed above, then finish feedback                         |
| `115-sequence-toast.js` (new, after 110)        | feedback         | summary card model and DOM fragment                                                 |
| `120-status-chip.js` (edit)                     | feedback         | sequence model, lazy step and title nodes, tooltip, settle                          |
| `140-ansi-and-links.js` / `150-pager.js` (edit) | feedback         | `step i of n` header label                                                          |
| `160-sheet-rank.js` (edit)                      | sheet            | RECENT grouping, default footer adds `⌃M mark`                                      |
| `177-sheet-sequence.js` (new, after 176)        | sheet            | marks, tray, keys, review card, sequence row activation                             |
| `990-exports.js` (edit)                         | all              | export new pure helpers for tests                                                   |
| `styles.css` (edit)                             | feedback, sheet  | chip, toast, tray, badge, review, and sequence row styles                           |

`177-sheet-sequence.js` sits after `176-sheet-keys.js`, so its `handleSheetKeydown`
wrapper is outermost and sees **raw** events. Use
`cmCommandModeKeyAlias(event) === "Escape"` to treat `Ctrl+[` as `Esc`. Honor the
`cmHandledSheetKeys` WeakSet the same way 176 does (check it first, and add to it when
handled), so the modal listener and the scope fallback never act twice.

### Invariants (test each one)

1. Step k+1 starts only after step k settles. Steps run in mark order.
2. After a halting outcome no later step starts, and later steps are reported as
   skipped.
3. Every bob step goes through `runner.run`, keeping the save-before-write guard.
   Confirm and preview-first are satisfied once, by the review card or `Ctrl+Enter`.
4. Marks never outlive the sheet. A single mark never stamps a sequence.
5. Every stamped record carries the full step list. History groups by id, and the newest
   record per index wins.
6. Only one active sequence runs per signature.
7. Unload stops before the next step and emits no toast.
8. Sequence code never throws into the runner, the chip timer, or `commandModeNotify`.

## Phase engine: Sequence engine, record stamps, and plugin wiring

1. `050-run-store.js`: in `cmNewRecord`, when `options.sequence` is an object, set
   `record.sequence` to a sanitized copy. Strings become strings; steps become
   `{ argv: string[], entry_id, title }`. Leave the field **absent** otherwise.
2. `060-runner.js`: pass `sequence: opts.sequence` into `cmNewRecord` in `runInner`. Do
   not change the guards.
3. New `056-sequence-model.js` with pure helpers:
   - `cmSequenceStep`, `cmSequenceStepSignature`, `cmSequenceSignature`
   - `cmSequenceStepDisplay(step, catalog)`, `cmSequenceStepTitle(step, catalog)`
   - `cmSequenceStamp(id, index0, steps, startedAtMs)`
   - `cmSequenceContinues(outcome)`: `completed` always continues, `attention` continues
     per the `attention_continues` decision, and everything else halts
   - `cmSequenceGuards(steps, catalog)`: returns
     `[{ index, kind: "confirm" | "preview-first", text }]` from current catalog
     policies, where `text` is the entry's confirm text or `applies without its preview`
   - `cmSequenceHistory(records, { activeIds })`
   - `cmSequenceFocusRunId(summaryOrGroup, catalog)`
   - `cmSequenceCopyText(steps, catalog)`: `cmFormatArgv(["bob", ...display])` joined
     with `&&`
   - `cmSequenceStepsFromRecord(record)`
4. New `065-sequence-runner.js` with `cmRunSequence`, per the driver contract.
5. New `095-plugin-sequences.js`:
   - Add the plugin wiring above.
   - Add a minimal `commandModeSequenceFinished(summary)` that shows one plain `Notice`,
     for example `bob sequence · 3/3 completed in 0:42` or
     `bob sequence stopped at 2/4 · <reason>`. Feedback replaces it with the card.
   - Add `commandModeRerunLast()`. Per `rerun_last_sequence`, it reruns
     `cmSequenceStepsFromRecord(newest)` through `commandModeRunSequence` when the
     newest record has a stamp, and otherwise calls `commandModeRerun(newest)`.
   - Point the `rerun-last` command callback in `090-plugin.js` at
     `commandModeRerunLast()`.
6. Export the new pure helpers and `cmRunSequence` from `990-exports.js`, and add both
   new fragments to `fragments.json`.
7. Extend `scripts/fixtures/fake-bob.cjs`:
   - `FAKE_BOB_EXIT_FOR`: a JSON object mapping the space-joined argv after `bob` to an
     exit code, overriding `FAKE_BOB_EXIT`.
   - `FAKE_BOB_LOG_FILE`: append one JSON line `{argv, start, end}` per invocation.
8. Add `scripts/test-command-mode-sequence-engine.cjs` to the `npm test` list. Use
   `makeRunner` from `command-mode-harness.cjs` and the fake bob. Cover:
   - in-order execution (each logged `start` ≥ the previous `end`), including for reads
   - stamps on every record (id, 1-based index, total, full steps)
   - a halt on a failing step 2 (step 3 never logged, `stoppedAt: 2`, skipped results)
   - attention continuing per the decision
   - an early refusal (a `saveAllMarkdown` failure) halting with its reason
   - a stale `entry_id` falling back to argv
   - a resume (`startIndex`, `priorResults`, same id) folding into one history group
   - the history state rules, including `interrupted`, and the newest record per index
   - duplicate-signature refusal
   - `cancelled()` stopping before the next step
   - per-step `commandModeNotify` suppression and the chip-settle skip, through a
     `BobCommandModePlugin` built the way `test-command-mode-outcomes-api.cjs` builds it
   - `commandModeRerunLast` on a stamped newest record
   - copy text and focus-run selection
   - hooks that throw not breaking the run

## Phase feedback: Status chip, summary toast, and pager step label

1. `120-status-chip.js`:
   - `cmStatusChipModel(records, nowMs, state, progressFor, options)` reads
     `options.sequences`, the snapshots
     `{ id, total, index, steps: [{ title, display }], results, startedAtMs }`.
   - With an active sequence, it returns the extra model fields `step: "2/3"`,
     `stepTitle`, `sequenceId`, the sequence clock, the sequence tone, the tooltip, and
     `runId`.
     - **Tone:** queued when the current step's active record (found by
       `record.sequence.id`) is queued; overdue when any active run is overdue;
       otherwise running.
     - **`runId`:** the current step's active record, else the last finished step's
       `runId`.
   - The sequence is visible once its age is at least the reveal time, even when no run
     is active between steps.
   - The settle state grows an optional `sequence: { step, stepTitle, total }` that
     paints `✓ bob 3/3 <title> 0:42`.
   - `cmRenderStatusChip(..., state, options)` takes a trailing `options` argument and
     passes it through. It adds sequence ids to `state.shown` and lazily creates the
     `bob-cmd-chip-step` and `bob-cmd-chip-title` spans before the clock, as described
     above.
   - `commandModeRefreshStatusChip` in `090-plugin.js` passes
     `{ sequences: this.commandModeSequenceSnapshots() }`.
2. In `095-plugin-sequences.js`, make `commandModeSequenceFinished(summary)` do three
   things:
   - **Chip settle:** record the settle only if the sequence id is in `chipState.shown`,
     with the focus run as the click target.
   - **Card:** show it through the new fragment, keeping one sticky card per signature,
     with a plain `Notice` fallback when there is no document.
   - **OS notification:** for runs over 15 s while unfocused, as in
     `commandModeMaybeNotifyOS`.

   Add `commandModeRunSequenceToastAction(action, summary)` for `output`, `resume`,
   `rerun`, and `copy`.

3. New `115-sequence-toast.js`:
   - `cmSequenceToastModel(summary, catalog)` returns the tone, header text, and step
     lines (glyph, title, duration or `skipped`, and the first line of a halting
     reason), plus the actions. Resume appears only for a halt.
   - `cmCreateSequenceToastFragment(model, onAction, onOutput)` uses the existing
     `bob-cmd-toast` card classes, plus a `bob-cmd-toast-steps` list.
   - Set `role` and `aria-live` as `cmCreateToastFragment` does.
4. Pager: add `sequence: { index, total }` to `cmBuildPagerView` when the record has a
   stamp. `cmRenderPagerView` renders a `bob-cmd-pager-seq` span `step 2 of 3` after the
   argv.
5. `styles.css`:
   - Chip step and title, with no animation under reduced motion.
   - Toast step rows: outcome-colored glyph column, muted duration, faint `skipped`, and
     a reason line in muted text.
   - The pager `step` label, styled as a quiet pill like `.bob-cmd-pager-index`.

   Use theme variables only, matching the existing palette (`--interactive-accent`,
   `--text-muted`, `--color-green/orange/red`).

6. Add chip frames (running `2/3`, queued, a settle `✓ 3/3`, a halted `✗ 2/4`) and both
   summary cards to `scripts/preview-command-mode.cjs` and its template. Render them
   with `node scripts/preview-command-mode.cjs --out <tmpdir>`, and look at the
   screenshots in light and dark themes. Fix spacing, truncation, and alignment until
   they look native next to the existing single-run frames.
7. Extend `scripts/test-command-mode-status-chip.cjs`:
   - the sequence model: step text, title, sequence clock, the tooltip block
   - visibility with zero active runs between steps, and no settle between steps
   - lazy nodes inserted before the clock, with an idempotent repaint
   - the plain anatomy unchanged without sequences
   - settle and click target

   Add `scripts/test-command-mode-sequence-feedback.cjs` to the `npm test` list. Cover:
   - the toast model for completed, attention, halted, and stopped outcomes
   - stickiness and replacement per signature
   - each action (resume keeps the id and skips completed steps; rerun gets a new id;
     copy text)
   - Output targeting the focus run
   - the pager `step i of n` label

## Phase sheet: Marking, tray, review card, and RECENT sequence rows

1. `160-sheet-rank.js`:
   - **Grouping:** `cmSheetRecentRows(runs, nowMs, catalog, options)` gets an optional
     `options.activeSequenceIds`. Walk newest-first. The first record of each sequence
     id emits one row built from `cmSequenceHistory`, and later records of that id are
     skipped.
   - **Sequence row fields:** `kind: "recent"`, `key: "recent-seq:" + signature`,
     `steps`, `sequenceId`, `icon: "sequence"` (add `sequence: "list-ordered"` to
     `CM_SHEET_ICONS` and `≡` to the fallback), `path` (display words with `›` separator
     words), `title: "N steps"`, `about`, `glyph` (add `interrupted: "◌"` to
     `CM_SHEET_OUTCOME_GLYPH`), `age`, `duration`, `required: []`.
   - **Shared logic:** single rows and sequence rows share one dedupe set and the 4-row
     limit. Keeping `kind: "recent"` reuses RECENT grouping plus the glyph and age
     rendering.
   - **Footer:** add `⌃M mark` to the default list footer hint.
2. New `177-sheet-sequence.js`, which patches `CommandModeSheetModal.prototype`:
   - **State:** `this.sequenceMarks` is an ordered array of tray steps. Each tray step
     is the stored step plus `key` (signature), `display`, `effect`, `policy`,
     `confirm`, `hasPreview`, `presentation`, and `row` (a snapshot used for single-mark
     activation; argument-mode marks build it the way `runArgumentStage` does).
   - **`cmSheetSequenceStep(row, catalog)`:** pure; returns a tray step or
     `{ refused: reason }` using the refusal rules above. Export it for tests.
   - **Keys:** an outermost `handleSheetKeydown` wrapper implements the key table and
     the argument-mode rules. It runs only in the `list` and `args` stages; `confirm`,
     `output`, and `vim` pass through, except that the review card's `Ctrl+Enter` starts
     the sequence.
   - **Select-all:** after any mark change from the list, select the whole query
     (`setSelectionRange(0, len)` after `render()`). The 176 render wrapper then keeps
     that selection across `Ctrl+N`/`Ctrl+P` re-renders.
   - **Rendering:**
     - Wrap `renderHeader` to append the tray after the input row in the `list` and
       `args` stages when marks exist.
     - Wrap `renderRow` to add `is-marked` and a `bob-cmd-row-mark` badge (CSS
       `order: -1`; the row icon is hidden while marked), to rebuild a sequence row's
       path with `bob-cmd-row-seq-sep` spans, and to make plain clicks toggle while
       marks are pending, and `Ctrl`/`Cmd`+click toggle always.
     - Wrap `renderFooter` for the marks-pending footers. A warning hint or the running
       line still wins.
   - **Review card:** set `this.confirmSequence = { steps, guards, force }`,
     `this.confirmRow = null`, and `stage = "confirm"`.
     - Wrap `renderConfirm` and `confirmRun` so they handle the sequence when
       `confirmSequence && !confirmRow`, and fall back to the originals otherwise.
     - Clear `confirmSequence` whenever the confirm stage is left.
     - Footer: `⌃↵ run all · esc back`.
   - **Start:**
     - If `plugin.commandModeSequenceIsActive(signature)`, show the hint and stay open.
     - Otherwise clear the marks, `close()`, and call
       `plugin.commandModeRunSequence(steps, { trigger: "sheet-sequence" })`.
     - Headless fallback: when the plugin lacks the method, call
       `cmRunSequence(this.runner(), steps, {})`.
   - **Single mark:** call the next `activateRow` in the chain (175 → 170), not the 177
     wrapper, with `{}` or `{ force: true }`.
   - **RECENT sequence rows:** a wrapped `activateRow` runs them as a sequence under the
     same rules. `Ctrl+M` group-toggles their steps.
3. `styles.css`:
   - **Tray:** `.bob-cmd-seq-tray`, `-label`, `-chip`, `-num`, `-sep`, `-remove`,
     `-run`. Pill chips use `--background-secondary` and `--background-modifier-border`.
     Numbers sit in 15–16 px `--interactive-accent` circles with `--text-on-accent`
     text.
   - **Rows and badge:** the `is-marked` row tint
     (`color-mix(in srgb, var(--interactive-accent) 7%, transparent)`), which must stay
     legible when also `is-selected`, and the `.bob-cmd-row-mark` badge.
   - **Review card:** `.bob-cmd-seq-review` rows with amber notes that reuse the confirm
     palette.
   - **Sequence row:** `.bob-cmd-row-seq-sep` in `--text-faint`.
   - **Motion:** a chip-in animation disabled under `prefers-reduced-motion`.
4. Add `scripts/test-command-mode-sheet-sequence.cjs` to the `npm test` list, reusing
   the `sheetHarness` and `openSheet` patterns from `test-command-mode-sheet.cjs`.
   Cover:
   - marking, toggling, and renumbering
   - the badge and tray text
   - query select-all
   - each refusal reason
   - `Ctrl+M` into argument mode, then marking from it
   - `Backspace` popping the last step
   - `Esc` per `escape_clears_marks`
   - click and `Ctrl`+click toggles
   - `Shift+Enter` with marks running nothing
   - one mark activating like `Enter` (a report opens the output stage, a confirm row
     opens its card)
   - `Enter` with unguarded steps starting at once and closing the sheet
   - `Enter` with confirm or preview-first steps showing the review card (`Esc` keeps
     the marks; `Ctrl+Enter` starts)
   - `Ctrl+Enter` skipping the card
   - argument-mode add & run
   - the already-running hint
   - RECENT grouping (one row, 4-slot limit, dedupe, interrupted glyph, about line),
     plus `Enter` rerunning all steps and `Ctrl+M` loading and unloading them

   Extend `scripts/test-command-mode-sheet-keys.cjs` for `Ctrl+M` with Caps Lock and the
   `KeyM` code, `Ctrl+Shift+M` ignored, and `Ctrl+[` matching `Esc` with marks pending.

## Phase docs: Plugin reference, user guide, and deploy

1. In bob-plugins `docs/bob-command-mode.md`:
   - **Keyboard and Vim:** add `Ctrl+M` and the marks-pending keys.
   - **New `## Sequences` section** (after Command sheet) covering marking, the tray,
     running, the review card, halting, Resume, Copy, and RECENT rows.
   - **Feedback:** the chip's `2/3` and title, and the summary card.
   - **Run engine:** records carry an optional `sequence` stamp; steps run one at a
     time.

   Keep the generated blocks untouched, and run `npm run docs:check`.

2. In bob-cli (this repo), add a short `### Run several commands in order` subsection
   under "Open the command sheet" in `docs/guides/run-bob-from-obsidian.md`: mark with
   `Ctrl+M`, run with `Enter` or `Ctrl+Enter`, watch the chip, and run again from
   RECENT. Fix any guide sentence the feature makes incorrect. Run `just docs-check`.
   The mirrored `docs/reference/plugins/*.md` copies are gitignored and synced from
   bob-plugins, so do not edit them.
3. Run the full `npm test`, `npm run validate`, and `npm run build:check` in
   bob-plugins. Then run `bob plugins sync` and confirm `bob plugins list` shows
   `bob-command-mode` synced. If the Obsidian startup smoke runs on this host, run
   `npm run smoke:obsidian`.

## Verification

- Every phase: run `npm run build`, then `npm run build:check`, its new or changed test
  files, then the full `npm test` and `npm run validate`, then `bob plugins sync`.
  Phases engine, feedback, and sheet each keep every hand-edited fragment at or below
  1000 lines.
- Feedback: look at the preview screenshots before finishing.
- Sheet: run a manual pass in Obsidian if one is reachable. Mark `vault-sync`,
  `task reconcile`, and `plan`, and press `Enter`. Confirm the chip reads `bob 1/3 …`
  through `3/3`, one summary toast appears, and RECENT shows one sequence row that
  reruns on `Enter`.

## Out of scope

- Surface (Obsidian view) or Vim steps inside a sequence.
- Reordering steps in the tray. To reorder, unmark and mark again.
- Previewing a whole sequence.
- Resuming a sequence after a plugin reload. RECENT marks it interrupted honestly.
- An `api.commandMode.runSequence` method.
- Searching sequences by query, since RECENT shows only for an empty query.
