---
tier: tale
size: small
title:
  Land bob-cli-du — honor the shared done vectors in cycler api v3, sweep stale §18
  markers, close the epic
goal:
  The task-status-cycler api v3 `completeTaskLineText` honors every case in the shared
  relation_complete `done` vectors. Already-closed lines stay unchanged; recurring and
  non-task lines are refused. Nav consumes that result. docs/task-relations.md no longer
  calls shipped work planned. Epic bob-cli-du is closed and its plan marked done.
proposed_by: bbugyi200.apollo.bob-cli-du.land
proposed_with: claude/opus
bead: bob-cli-du
status: done
---

- **PARENT:**
  [202610/relation_complete_labels.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/relation_complete_labels.md)
- **BEAD:**
  [bob-cli-du](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-du/README.md)
- **AGENTS:**
  - [bbugyi200.apollo.bob-cli-du.land](https://github.com/bobs-org/bob-cli--agents/blob/main/sessions/bbugyi200.apollo.bob-cli-du.land.md)
- **COMMITS:**
  - [68f0443](https://github.com/bobs-org/bob-plugins/commit/68f0443af3c11dd0588e99234bfb625e8647240b)
    — fix(task-status-cycler): guard complete-line api results

# Land bob-cli-du: finish the cycler done-vector contract, sweep docs, close the epic

## Context

Epic **bob-cli-du** ("Complete labels — relation labels that complete their source
task") has every phase closed (bob-cli-du.1 … .9, including child epic bob-cli-du.2.1).
The land agent verified the work and found two remaining gaps, both caused by the epic.
Both are small:

1. **Cycler api v3 does not honor the shared `done` vectors.**
   - The source of truth is bob-cli `tests/fixtures/relation_complete/vectors.json`.
     bob-plugins keeps a byte-for-byte copy at
     `plugins/bob-navigation-hotkeys/scripts/fixtures/relation_complete/vectors.json`,
     which must stay unchanged.
   - Its `done` section has 17 cases. 12 expect
     `{"ok": true, "unchanged": false, "line": <postimage>}`.
   - `already-done` and `already-cancelled` expect
     `{"ok": true, "unchanged": true, "line": <same line>}`.
   - The three `recurring-*` cases (`[repeat:: …]`, `(repeat:: …)`, `🔁`) expect
     `{"ok": false, "reason": "recurring", "line": <same line>}`.
   - The shipped `completeTaskLineText(lineText, {date})` lives in bob-plugins
     `plugins/task-status-cycler/src/160-plugin-completion.js`, around line 224. It
     returns a bare string from `rewriteTaskLineForLocalFallback(line, "x", date)`, so a
     Cancelled or recurring line comes back as `[x]` with a new completion date.
   - `scripts/test-task-status-cycler-complete-vectors.cjs` skips the 5
     unchanged/refused cases. For those it only compares the vector's fields with
     themselves.
   - The epic plan's `cycler-done-api` phase required `{ok, lineText}` /
     `{ok: false, reason: "recurring"|"not_completable"}`, and its acceptance requires
     "Rust and JS pass every shared vector".
   - Nav is safe today: its planner gates on its own verdict, and
     `completeTaskLineForRelation` re-checks the output. But the exported api violates
     CP5 ("never Cancelled → Done").
   - **Important:** Ctrl+Enter's local-fallback Done write in
     `plugins/task-status-cycler/src/180-plugin-editor-edits.js`
     (`setActiveCheckboxStatusLocalWithTaskMetadata`, around line 476) also calls
     `this.completeTaskLineText(...)` and needs a **string**. Ctrl+Enter behavior must
     stay byte-identical, including any recurring or non-open line that path writes
     today. So the guard belongs only on the api surface. Ctrl+Enter keeps the unguarded
     shared rewrite.
2. **Stale status markers in bob-cli `docs/task-relations.md`.** The rollout's docs
   sweep missed these:
   - §6, the LR7 bullet (around line 271), still says "([RV10](#rv10-reversing-a-label),
     planned, phase `reverse-rename`)". RV10 is shipped (`cba7c47d`).
   - CP2 status (around line 1382) says "implemented in capture; live config rollout
     stays with the outer phases." Rollout is done: chezmoi commit `6a7ba18` enables
     `implemented by` and `verified by`.
   - CP6 status (around line 1536) says "implemented in capture; planned
     (`nav-complete-cue`, `mac-complete-cue`)." Both shipped: bob-plugins `7f42bb8` and
     bob-mac-capture `072055d`.
   - CP7 status (around line 1574) says "capture fields implemented." The
     `capture-complete`, `capture-parse`, and `bob relations list` fields also shipped
     (`387a4fc2`).
   - The CP8 bullet says `completeTaskLineText` "returns the exact fallback bytes". It
     must describe the result object after fix 1.

Already decided (do **not** redo):

- The `finalizeClosedTasks` result shape stays as shipped and documented in bob-plugins
  `docs/develop/plugin-apis.md`:
  `{reopened, retired, recoveryFailures, retirementFailures, successors, successorNotice}`.
- No glossary or other memory edits.
- The pre-existing bob-mac-capture CI failure is tracked as bob-cli-dz. Do not touch
  bob-mac-capture.
- Follow-up triage is already recorded on bob-cli-du. Create no beads.

Epic decisions stay in force: offer_successors = yes. Memory edits are authorized only
for glossary:task-relation (already done), and none are needed here. No decision record
is added.

## Steps

### 1. bob-plugins: guard the api, keep Ctrl+Enter unchanged

Open the repo with `/sase_repo` (`sase repo open bob-plugins -r "..."`) and read its
`AGENTS.md`/`CLAUDE.md`. Edit `src/` fragments only, never generated `main.js`, and keep
each fragment at or below 1000 lines.

1. In `plugins/task-status-cycler/src/160-plugin-completion.js`:
   - Add a pure, never-throwing helper, `rewriteTaskLineAsDone(lineText, date)`. It
     returns the string
     `rewriteTaskLineForLocalFallback(String(lineText || ""), "x", date)`, with the
     current try/catch fallback to the original string.
   - Point Ctrl+Enter's `180-plugin-editor-edits.js` call at it, passing the same date
     argument. Ctrl+Enter then writes exactly the same bytes as today. Keep its comment
     about one shared byte source accurate.
   - Re-implement `completeTaskLineText(lineText, options = {})` as the guarded api. The
     date defaults exactly as today (`options.date` string, else
     `this.getCompletionDateString()`). It returns a frozen object:
     - `getTaskStatusForLine(line, 0)` is null →
       `{ok: false, reason: "not_completable", lineText: line}`;
     - the status symbol (normalized `x`/`X`) is not one of `" "`, `"*"`, `"/"`, `"?"` →
       `{ok: true, unchanged: true, lineText: line}`. That covers Done, Cancelled, and
       any other symbol, matching CP5 `already_closed`. Check this **before**
       recurrence, as Rust `task_cancel::verdicts` does.
     - recurring → `{ok: false, reason: "recurring", lineText: line}`. A line is
       recurring if it matches `/\[repeat\s*::/i` or `/\(repeat\s*::/i`, or contains
       `🔁`. These are the same three markers as Rust `is_recurring_task_line` and nav's
       `isRecurringTaskLine`. Add a small local pure helper in the cycler; plugins never
       import each other's `main.js`.
     - otherwise →
       `{ok: true, unchanged: false, lineText: rewriteTaskLineAsDone(line, date)}`;
     - any exception → `{ok: false, reason: "not_completable", lineText: line}`.
   - Leave the api version at 3 and leave the `110-plugin-lifecycle.js` member list
     unchanged.
2. In `plugins/bob-navigation-hotkeys/src/597-relation-complete-compose.js`, update
   `completeTaskLineForRelation` (around line 290) to consume the object:
   - Accept only
     `result && result.ok === true && result.unchanged === false && typeof result.lineText === "string" && result.lineText !== before`.
     Anything else returns the existing
     `{ok: false, reason: "unexpected-cycler-result"}`. A pre-fix cycler that returns a
     bare string therefore fails safe: it writes nothing. Add no string compatibility
     branch.
   - Keep the existing `[x]`, completion-date, block-id, and task-at-line post-checks on
     `result.lineText`. Keep the `cycler-api` reason for a missing api or a thrown call.
3. Tests:
   - `scripts/test-task-status-cycler-complete-vectors.cjs`: call
     `plugin.completeTaskLineText(vector.line, {date: vector.date})` for **every**
     `done` case. Assert `ok`, `lineText === expected.line`, `unchanged` for ok cases,
     and `reason` for refused cases. Add a non-task line case expecting
     `{ok: false, reason: "not_completable"}`. Keep the api-surface test and the
     "Ctrl+Enter local fallback shares the api Done bytes" parity test. Change that test
     so it compares the editor result with `expected.line` and with
     `plugin.completeTaskLineText(...).lineText`.
   - `scripts/test-navigation-relation-complete-writer.cjs`: the real-cycler cases must
     pass through the new object. Add one case with an injected `completeTaskLineText`
     that returns `{ok: false, reason: "recurring", lineText}`. The writer must refuse
     (`relation_complete_write_failed`) and leave every note byte-identical. Add one
     case with a bare-string return that also writes nothing.
   - Grep `scripts/` for other `completeTaskLineText` mocks (for example
     `test-navigation-relation-links-stage.cjs`, around line 308) and keep them
     consistent with the object shape.
4. Docs:
   - In `docs/develop/plugin-apis.md` (around line 322), describe the result object and
     its three outcomes.
   - In `docs/task-status-cycler.md` (around line 140), make the same change in one
     sentence: "leaves already-closed lines unchanged and refuses recurring and non-task
     lines".
5. Run `npm run build`, `npm test`, and `npm run docs:check`, then `bob plugins sync`.
   Compare any failure with a clean-base run. Unrelated timing flakes have been seen in
   ledger-tools suites; rerun once before treating a failure as real. Confirm the
   bob-plugins vectors copy is still byte-identical to bob-cli's (`cmp`).

### 2. bob-cli: sweep the stale §18 / LR7 markers

In bob-cli `docs/task-relations.md`:

- **§6 LR7:** drop ", planned, phase `reverse-rename`" from the RV10 parenthetical.
- **CP2 status:** `*Status: implemented; chezmoi enables `implemented by`and`verified
  by`.*` This mirrors CL2's line.
- **CP6 status:** `*Status: implemented in capture, nav, and Mac.*`
- **CP7 status:** `*Status: implemented.*` First confirm each CP7 bullet exists in code.
  `rg -n "completes_source|relation_complete\b|complete_labels|relation_completes|complete_effect|relation_completed_source" src/`
  should hit `capture_complete/`, `capture_parse.rs`, `capture/output.rs`, and
  `relation_labels/cli.rs`. If any bullet is genuinely unimplemented, say so in that
  status instead.
- **CP8:** replace "returns the exact fallback bytes" with a description of the result:
  `{ok: true, unchanged, lineText}` carrying the exact fallback bytes; already-closed
  lines `unchanged: true`; recurring and non-task lines `{ok: false, reason}`;
  Ctrl+Enter shares the same rewrite.
- Run
  `rg -n "planned" docs/task-relations.md docs/capture.md docs/reference/json-outputs.md docs/reference/configuration.md docs/projects.md docs/task-dependencies.md`.
  Clear any other leftover complete-label "planned" marker the same way. The unrelated
  "apply the planned contents" prose in §17 stays.

Then run `just docs-check` and `just check` in bob-cli. Do not run `just check-full`.
Never edit `Cargo.toml` versions or `CHANGELOG.md`.

### 3. Close out epic bob-cli-du (final step)

1. Run `sase bead epic-symbols bob-cli-du`. It reported no entries at landing time. If
   any appear, resolve each one before continuing: wire it up, privatize it, add a
   non-test pragma, or delete it per the Symvision epic-whitelist policy. bob-cli-du has
   no open later bead to re-key to.
2. Close the epic. Fill in the real test counts and the bob-plugins/bob-cli state:

   ```bash
   sase bead close bob-cli-du --note "Landing verified: all 9 phases (and child epic bob-cli-du.2.1) closed with evidence; bob-cli commits 847a3c53 a32a32fb 4c55a595 4be02241 87b6e608 d1f445ca 387a4fc2 cba7c47d 7d3ce705, bob-plugins 3c84ee3 7c088a5 7f42bb8 (+1da19e9 drift fix), bob-mac-capture 072055d, chezmoi 6a7ba18, vault migration 2c963688 reviewed. Landing tale: cycler api v3 completeTaskLineText now returns {ok, unchanged, lineText}/{ok:false, reason} and passes all 17 shared done vectors (already-closed unchanged, recurring and non-task refused) while Ctrl+Enter keeps byte-identical bytes; nav 597 consumes the object and fails safe; docs/task-relations.md LR7/CP2/CP6/CP7/CP8 markers swept. npm test <N> pass, docs:check, bob plugins sync ok; just check and just docs-check green. Follow-ups: bob-cli-du.1 glossary proposal declined (memory not authorized; task-relation strand covers both exceptions); bob-cli-du.6 finalizeClosedTasks naming declined (shipped shape documented); Mac CI's only failure is pre-existing bob-cli-dz (+1 recorded). Integration drift since 847a3c53 reviewed; no other changes needed."
   ```

   Never use `--force`. If the close is refused, fix the named cause and close again.

3. Run `just symvision` if the bob-cli justfile has that recipe. At landing time it did
   not; if it is still missing, say so.
4. Mark the plan done. Open the plans sidecar with `/sase_repo`
   (`sase repo open plans -r "..."`), edit `202610/relation_complete_labels.md` (the
   PLAN path shown by `sase bead read bob-cli-du`), and change its frontmatter
   `status: wip` to `status: done`. Change nothing else.
5. bob-cli-du has no `parent_bead`, so there is nothing further to close.

## Acceptance

- Every one of the 17 `done` vectors passes through the cycler api
  `completeTaskLineText` with matching `ok`/`unchanged`/`reason`/`lineText`.
- Ctrl+Enter local-fallback bytes are unchanged, and the existing cycler suites pass
  unmodified apart from the vector test.
- The nav writer completes exactly as before with the real cycler. Refused and legacy
  string results write nothing.
- `docs/task-relations.md` has no "planned" marker for shipped complete-label or RV10
  work, and CP8 matches the api.
- `npm test`, `npm run docs:check`, `bob plugins sync`, `just docs-check`, and
  `just check` are green, with no `just check-full` run.
- bob-cli-du is closed normally, and `relation_complete_labels.md` frontmatter reads
  `status: done`.

## Commits

- bob-plugins:
  `fix(task-status-cycler): honor relation_complete done vectors in api v3 completeTaskLineText`
  (covers the cycler, nav 597, tests, and docs).
- bob-cli: `docs(relations): mark complete-label rollout and surfaces shipped`.
- plans sidecar: the `status: done` flip.

There is no `!`; nothing here is a CLI or JSON contract break.
