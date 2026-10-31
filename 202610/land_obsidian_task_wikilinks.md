---
tier: tale
size: medium
title: Finish landing the (( task wikilink picker (bob-cli-eq)
goal: 'Fix the gaps and regressions that the bob-cli-eq land review found in the ((
  Task Wikilink Picker and the ^^ parity work, make the index performance test measure
  work instead of scheduler noise, then close epic bob-cli-eq.

  '
proposed_by: bbugyi200.apollo.bob-cli-eq.land
proposed_with: claude/opus
bead: bob-cli-eq
status: done
---

- **PARENT:**
  [202610/obsidian_task_wikilinks.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/obsidian_task_wikilinks.md)
- **BEAD:**
  [bob-cli-eq](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-eq/README.md)

# Plan: finish landing `((` (epic `bob-cli-eq`)

## Why

Epic `bob-cli-eq` (plan `plan:202610/obsidian_task_wikilinks.md`, PLAN path shown by
`sase bead read bob-cli-eq`) shipped the Obsidian `((` Task Wikilink Picker in
`plugins/block-id-prompt` of **bob-plugins**. Phase commits: 27acd61, 9dd0374, 4ef761a,
145ea37, 458907b, 4bb8821, and 8ef763a; bob-cli ebd74b7b and 3ca356c6. All seven phases
are closed.

The land review confirmed the defects below in the source, and reproduced most of them
with probes. Each was introduced by the epic, so they remain epic work. Two of them
break `^^` gestures that already worked before the epic. Fix everything here, then do
the closeout (last section). The epic's DECISIONS are final:

- `pomodoro_blocked = raise`;
- `caret_parity = yes`;
- `decision_record = no`. Write no memory. Task `bob-cli-eu` already carries that
  follow-up.

Nothing that landed on either repo since the epic started touches `block-id-prompt`, and
the type-marker change keeps `[type:: X]` storage. No integration edits are needed.

## Ground rules

- Open bob-plugins with `/sase_repo` (`sase repo open bob-plugins -r "..."`) and read
  its `AGENTS.md`.
  - Edit only `plugins/block-id-prompt/src/*.js` fragments, `styles.css`, docs, and
    `scripts/` tests, then run `npm run build`. Never hand-edit `main.js`.
  - Keep each fragment at or under 1000 lines.
  - List any new test file explicitly in `package.json`'s `test` script.
- Line numbers below are from bob-plugins master 8ef763a. Re-locate by symbol if they
  have moved.
- Commit types: `fix(block-id-prompt): …` and `test(block-id-prompt): …`. bob-cli needs
  no change; if you touch it, `just check` is its gate.

## 1. `^^` regressions (high)

### 1a. `^^` naming must keep the typed note and alias

- **Where:** `src/100-plugin-lifecycle.js` `selectTaskLinkTask` (~l.314-325).
- **Bug:** it unconditionally sets
  `targetText: task.linkTarget || taskWikilinkPathStem(task.notePath)`, `blockPrefix`,
  and `aliasSuffix: ""` on the prompt source. `^^` items come from
  `collectTaskPickerItems` / `taskItemFromLine` (`src/030`) and carry neither
  `linkTarget` nor `notePath`.
- **Failure:** naming an ID-less task from `- see [[Tasks#^^]]` writes `[[#^ship]]` (a
  link to the current note) and drops any `|alias`.
- **Fix:** apply those three overrides only when `source.trigger === "(("`. `^^` must
  build its prompt source exactly as before 458907b.
- **Test:** add a `^^` regression to
  `scripts/test-block-id-prompt-task-wikilink-naming.cjs`. Feed a real
  `collectTaskPickerItems` item from a cross-note `[[Tasks#^^|alias]]` marker, and
  assert the final line is `[[Tasks#^ship|alias]]`.

### 1b. `^^` under an open Pomodoro must work

- **Where:** `src/129-plugin-task-wikilink-pomodoro.js` `taskWikilinkPomodoroPrepare`
  (~l.268-275) resolves the target file from `task.notePath`.
- **Failure:** for `^^` items `notePath` is undefined, so every `^^` completion under an
  open Pomodoro is refused with "Task link blocked: target note could not be resolved".
  - With an existing ID, the result is `{stale: true}`, and `TaskLinkPickerModal`
    (`src/080`, ~l.515-522) treats it as done: the picker closes and leaves
    `[[Tasks#^^]]` in place.
  - Reproduce under `## Pomodoros` with `- [ ] Current (09:20-09:50)` and child
    `  - [[Tasks#^^]]`, picking `- [ ] #task Ship it ^ship` from the `^^` picker.
- **Fix:**
  - For `^^` sources (`source.trigger !== "(("`), resolve the file from the `^^`
    picker's destination: `source.destinationPath`, which `openTaskLinkPicker` sets,
    else resolve `source.targetText` the way `readDestinationForValidation` does.
    Alternatively, stamp `notePath` (and `linkTarget` = the typed `targetText`) onto
    each `^^` item when the `^^` picker hands it over.
  - The `^^` link text must stay the user's typed target, so this does not conflict with
    1a.
  - Make a genuinely stale `^^` result behave as the pre-epic `^^` stale path did:
    revert the marker, or keep the picker open with a Notice. Never close silently with
    `^^` still in the note.
- **Tests:** add to `scripts/test-block-id-prompt-task-wikilink-pomodoro.cjs` (or
  `test-block-id-prompt-pomodoro-context.cjs`), using real `collectTaskPickerItems`
  items, not index rows:
  - an existing-ID `^^` link under an open Pomodoro links and promotes to `[*]`;
  - an ID-less one names, then links;
  - a closed target is refused;
  - a prerequisite-blocked target is raised and the `^^` marker reverts;
  - a project-blocked target is refused.

## 2. Trigger grammar (`src/022-task-wikilink-token.js`)

### 2a. Pomodoro-entry false positive

- **Where:** `taskWikilinkContextAt` (~l.451) excludes any line for which
  `isPomodoroEntryLine(lineText)` is true. That helper (`src/040`, l.189) is
  section-blind and matches a `()` placeholder or any time range.
- **Failure:**
  - Obsidian auto-pairs `((` into `(())`, so `- [ ] Follow up (())` and `- [ ] (())` are
    excluded, and the picker never opens on top-level task lines.
  - A task line elsewhere containing `(09:00-09:30)` is also excluded.
- **Fix:** exclude only when both hold:
  - the cursor line is inside the daily note's Pomodoros section
    (`findPomodorosSectionRange`);
  - `isPomodoroEntryLine` is true for the line with the token's replacement span
    removed.
- **Tests:** add trigger/context vectors:
  - `- [ ] Follow up (())` and `- [ ] (())` with the caret inside are allowed;
  - a real entry `- [ ] FIX (09:20-09:50) ((x` under `## Pomodoros` stays excluded;
  - `- [ ] #task Call (09:00-09:30) ((x` outside the Pomodoros section is allowed.

### 2b. Block exclusions leak out of code and frontmatter

- **Where:** `taskWikilinkDollarBlockInside`, `taskWikilinkPercentBlockInside`, and
  `taskWikilinkHtmlCommentInside` (~l.347-394) count `$$`, `%%`, and `<!--` on every
  earlier line.
- **Failure:** fenced `echo $$`, `printf("100%%")`, or `<!-- start` lines, or a `%%` in
  frontmatter, disable `((` for the rest of the note.
- **Fix:** skip frontmatter lines and lines inside fenced code (opening and closing
  fence lines included) when accumulating those states.
- **Tests:** add vectors for each case: a fence or frontmatter holding `$$`, `%%`, or
  `<!--`, followed by an allowed `- [ ] later ((x`.

## 3. Picker and naming (`src/087`, `src/100`, `src/128`)

### 3a. Restore picker state after the naming prompt

- **Where:** `TaskWikilinkPickerModal` (~l.10) takes its query from `source.query` and
  ignores `pickerState.query`. `rowLimit` is not restored, and `scrollTop` is applied to
  the wrong list (~l.112).
- **Failure:** Back, Escape, or an outside click in the naming prompt returns to an
  empty filter at the wrong scroll. The plan requires returning exactly where you were.
- **Fix:**
  - Prefer `source.pickerState.query` (and its `selectedKey` and `rowLimit`) when
    present.
  - Set `inputEl.value` to that query.
  - Apply `scrollTop` after the first render of the restored list.
- **Tests:** add naming tests for Back, Escape, and outside-click restore (query,
  selection, and scroll). Also cover the missing cases from the epic plan's naming list:
  - prefill and full selection;
  - the live insert line;
  - inline validation (`BLOCK_ID_RE` and the duplicate in the note);
  - guarded rows never open the prompt.

### 3b. Page keys must not wrap

- **Where:** `moveSelection` (~l.379) uses modulo for every delta.
- **Fix:** ↑/↓ (±1) still wrap. Page Down / Page Up (±10) clamp to the last / first
  selectable row and extend `rowLimit` as needed.
- **Test:** in `picker-runtime`, Page Down three rows from the end lands on the last
  row.

### 3c. Clear the written ranges on every scan

- **Where:** `inspectActiveEditor` returns early (~`src/100` l.81-102) without clearing
  them: no active Markdown view, an embedded or popover editor, or a non-`.md` file.
- **Fix:** clear the written ranges at the start of every scan, before any early return.
- **Test:** a stale range recorded before an early-return scan never opens the picker
  later.

### 3d. One undo restores `((query))`

- **Where:** the identified-accept splice (`src/128` ~l.412-420) uses
  `editor.transaction` with no user event. CodeMirror can merge it into the preceding
  typing event when it lands within 500 ms, so one undo would also remove the typed
  `((opus`.
- **Fix:** when `editor.cm.dispatch` exists, dispatch the change with
  `selection: { anchor }`, `userEvent: "input.complete"`, and `scrollIntoView: true`.
  Model it on `dispatchFileLinkBlockCompletion` in `src/070`. `input.complete` is not a
  joinable history event. Keep `editor.transaction` / `replaceRange` as the fallback.
- **Scope:** apply the same splice helper to every `((` replacement: identified insert,
  the naming success splice if it uses its own path, and the Pomodoro splice.
- **Test:** with a fake `editor.cm`, assert one dispatch with the right range,
  `userEvent`, and caret.

### 3e. Stale and Backspace guards

- **Wrong notice:** for an ID-less `((` row whose token changed, `selectTaskLinkTask`
  (`src/100` ~l.299) uses `^^`'s `sourceMarkerStillPresent` notice. `((` must show
  `Task wikilink not inserted: the ((…)) text changed`.
- **Backspace without a check:** Backspace on an empty filter (`src/087` ~l.583-588)
  computes the expected token text but never compares it (`void expected`). Verify the
  token text at its range first. If it changed, show the same stale Notice, close, and
  leave the note untouched.
- **Tests:** add one runtime test for each.

## 4. Pomodoro mode (`src/121`, `src/129`, `src/088`/`089`)

### 4a. "Could not be checked" refuses tasks it does not need to check

- **Where:** in `readTaskDependencyDocuments`' consumer (`src/121` ~l.113-122 and
  ~l.171-185), the `returnDetails` gate refuses when there are any read failures or the
  index is not ready, before testing `plan.blocked`.
- **Failure:** one unreadable unrelated note makes every Pomodoro link refuse, and the
  picker tags every row "dependency status unavailable" (`src/129` ~l.219-224).
- **Fix:** refuse as unavailable only when the status is actually unknown: the plan has
  unresolved prerequisite IDs, or the target is not found, while there were read
  failures or the index was not ready. This matches the older non-details gate
  (~l.127-138). A task whose prerequisites all resolved proceeds normally.
- **Test:** an unrelated read failure still links a Ready task with no prerequisites.

### 4b. A `^prj` blocked only by its own prerequisites is raised

- **Where:** `planTaskLinkBlockConsumption` (`src/030` ~l.656-659) sets
  `projectDependencyBlocked` for `target.blockId === "prj" && ownOpen.length > 0`. The
  gate (`src/121` ~l.139-170) and `taskWikilinkPomodoroProjectGuard` (`src/129`
  ~l.148-166) then refuse it.
- **Rule:** under `pomodoro_blocked = raise`, prerequisite-blocked tasks are raised.
  Ctrl+Shift+Enter already raises this case. Only these stay refused:
  - `#prj_is_blocked` residents;
  - inherited project blocks;
  - project dates.
- **Fix:**
  - Distinguish "the `^prj` line's own prerequisites" from a project block, for example
    with a separate plan flag. Keep Ctrl+Shift+Enter's behavior byte-identical.
  - Treat the own-prerequisites case as prerequisite-blocked: the `🔒 waits on N` badge
    and the raise branch.
- **Test:** add a raise test for a `^prj` task with one open prerequisite.

### 4c. Notice when an ID was added in Pomodoro mode

- **Where:** `src/129` ~l.501-503.
- **Fix:** when the commit added an ID, prefix the Pomodoro Notice with
  `Added ^id to note.md · ` as the plan's Notices list requires. The ordinary path
  already does this at `src/110` ~l.288.

### 4d. Raised section lists every promoted task

- **Where:** `src/129` ~l.396-410 lists only direct prerequisites that are now Next.
- **Fix:** list every task PDG1 actually promoted to Next (`plan.promoted`), and base
  the banner count on that same list.
- **Test:** a two-level chain raises the deeper task, which then appears in the section.

### 4e. Cheap guards on every row

- **Where:** the already-in-this-Pomodoro guard and the `#prj_is_blocked` guard are
  computed only for the highlighted row (`src/088` ~l.526-531, ~l.578-580).
- **Fix:** compute those two cheap guards for every row, so the rows render disabled
  with their badges. Dependency-derived details may stay lazy for the selected row.
- **Test:** in `pomodoro`, a non-selected duplicate row renders disabled with
  `Already in this Pomodoro`.

### 4f. Missing Pomodoro tests from the epic plan

Add the cases not covered yet:

- the `((` raise flow end to end: writes, index refresh, the pinned section with the
  first row selected, the banner, then linking a raised prerequisite;
- `((` project refusals: `#prj_is_blocked`, inherited, and project date;
- retype set / clear / keep, and the type clause omitted without the themes API;
- future-link cleanup in the Notice and the note;
- the stale re-check at commit.

## 5. Index performance test (`scripts/test-block-id-prompt-task-wikilink-index-performance.cjs`)

**Problem.** Epic notes #1 and #2 report this test failing under `npm test` and box
load. The cold-build chunk, snapshot, and keystroke budgets all miss there.

**The product is within budget.** The land agent's isolated probe measured:

- rank min 4.3 ms against the 12 ms budget;
- snapshot+order min 23.6 ms against the 50 ms budget;
- worst chunk about 3 ms.

With 40 busy processes on 16 cores, the worst chunk measured 22 ms wall-clock but only
3.7 ms of thread CPU. Rank and snapshot minimums stayed under budget. The failures are
wall-clock preemption noise.

**Fix the measurement, not the budgets:**

- Time every budgeted measurement in this file with the current thread's CPU time:
  `process.threadCpuUsage()` when it exists (Node ≥ 23.9), else `process.cpuUsage()`.
  Sum user + system, in milliseconds. Keep `process.hrtime` only where wall time is the
  point.
- **Chunk ceiling.** Measure per-chunk CPU in the test's injected `schedule` callback:
  the CPU delta from the end of the previous `schedule` callback to the next `schedule`
  call. Assert the minimum, over the 3 builds, of each build's worst chunk ≤ 16 ms. Keep
  the `chunks.length > 1` check from `onChunk`.
- **Cold total, snapshot+order, and keystroke rank.** Use CPU time. Use best-of-5 for
  snapshot and best-of-7 for rank. Await a short real delay (about 10 ms) between
  attempts, so one contention burst cannot cover every sample.
- **Near-linear growth.** Compare the best-of-3 CPU time for 500 and 3,000 notes.
- Update the file's header comment to say why it uses CPU time.
- Do not change `src/036` or the budgets.

**Verify:** run the file 5× in isolation, then 3× while a CPU hog runs, for example
`for i in $(seq 32); do node -e 'const e=Date.now()+90000;while(Date.now()<e){}' & done`.
Kill the hogs afterwards. All runs must pass.

## 6. Real-Obsidian check and docs

- **Extend `scripts/obsidian-task-wikilink-e2e.cjs`.** Keep its self-test,
  `scripts/test-obsidian-task-wikilink-e2e.cjs`, green. Add:
  1. A fast burst: type `((opus`, then Enter within about 150 ms. One undo restores
     `((opus))` (3d).
  2. Escape on a non-empty filter clears it and keeps the picker open. A second Escape
     closes the picker with the note unchanged.
  3. Typing `((` at the end of a top-level `- [ ] Follow up` line auto-pairs and opens
     (2a).
  4. `^^` under an open Pomodoro links an existing-ID task (1b).
- **Run it on athena**, as phase eq.7 did:
  `xvfb-run -a node scripts/obsidian-task-wikilink-e2e.cjs -e /snap/bin/obsidian`. Read
  the `tailnet` reference memory first. If athena or a prerequisite is unavailable, say
  so in the epic close note. Never report a skipped check as a pass.
- **If Escape on a non-empty filter closes the modal** in real Obsidian, route Escape
  through the modal's `scope` so it clears first, then re-run.
- **Docs.** Update `docs/block-id-prompt.md` wherever behavior changed: the 4b `^prj`
  raise, the 4c notice, the 2a/2b trigger exclusions. Keep the Guards and `^^` text
  accurate. Run `npm run docs:gen` if the generated tables change.

## 7. Gates (bob-plugins)

Run these and record each verdict:

- `npm run build`;
- `npm test`. Expect only four failures, all already tracked and not caused by eq:
  - `bob-cli-eh`: note-ready "startup storm", relation-index "registers after
    layout-ready";
  - `bob-cli-et`: freshness-namespace "reload event", gcal-events "create listener".

  Any other failure is yours to fix.

- `npm run validate`;
- `BOB_CLI_DIR=<bob-cli checkout> npm run docs:check`;
- `bob plugins sync --repo <bob-plugins checkout> --no-pull`, per bob-plugins'
  `docs/develop/deployment.md`. Never pass `-F`. If another checkout's uncommitted
  deploy blocks a plugin, report it in the close note.

## 8. Closeout of epic `bob-cli-eq` (final step, same turn as the code)

1. **Symbols.** Run `sase bead epic-symbols bob-cli-eq`.
   - At land review it printed "No --epic-symbol entries for bob-cli-eq".
   - If entries appear now, resolve each: wire it up, privatize it, add a non-test
     pragma, or delete it, per the Symvision epic-whitelist policy.
   - Re-key a Justfile line only to a still-open later bead that needs the exemption.
2. **Close.** Run `sase bead close bob-cli-eq --note "<verification>"`. Summarize:
   - the land review's findings and the fixes above;
   - every gate verdict, with test counts;
   - the athena e2e result, or that it was skipped and why;
   - the plugins sync result;
   - the follow-up triage already recorded on the epic: memory task `bob-cli-eu`, the +1
     on `bob-cli-eh`, new ci task `bob-cli-et`, sync proposals declined, perf test fixed
     here.

   Never use `--force` merely to make the close succeed. If the close is rejected for
   leftover `--epic-symbol` entries, clean them up and close again.

3. **Symvision.** Run `just symvision` in bob-cli if the recipe exists. At land review
   bob-cli's `justfile` had no `symvision` recipe; then say so in your final report.
4. **Plan status.** Set `status: done` in the frontmatter of the epic's plan file: the
   PLAN path shown by `sase bead read bob-cli-eq`,
   `plan:202610/obsidian_task_wikilinks.md` in the plans sidecar. It currently reads
   `status: wip`.
5. **Parent.** `bob-cli-eq` has no `parent_bead`, so the landing ends here.
