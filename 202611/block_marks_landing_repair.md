---
tier: tale
size: medium
title: Repair rendered block marks and finish bob-cli-ft.3.1 landing
goal:
  Fix the four reproduced rendered-row defects, replace false-positive real-app checks
  with observable evidence, and close bob-cli-ft.3.1 and its parent phase bob-cli-ft.3.
proposed_by: bbugyi200.athena.bob-cli-ft.3.1.land
proposed_with: codex/gpt-6-astra
bead: bob-cli-ft.3.1
create_time: 2026-11-02 10:38:21
status: wip
---

- **PARENT:**
  [202611/block_marks_editor.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/block_marks_editor.md)
- **BEAD:**
  [bob-cli-ft.3.1](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-ft/bob-cli-ft.3.1.md)

# Remaining blocked-checkbox work

This is a bounded repair and same-turn landing for **bob-cli-ft.3.1**. One coder can
repair the existing reading adapter and harness directly. Do not create another epic or
defer this tale's closeout to a land agent: a tale has no land agent.

## Context and fixed scope

Use `/sase_repo` to open `bob-plugins`; read its `AGENTS.md` and use only the printed
path. Edit its source fragments, then build generated `main.js`. The reading fragment
`plugins/bob-ledger-tools/src/142g-block-marks-reading.js` is already 975 lines; split
cohesive helpers into another registered fragment if needed, keeping every hand-edited
fragment at or below 1000 lines. Read docs memory before editing docs. No memory edits,
CLI/Rust behavior changes, Cargo versions, or CHANGELOG changes are authorized here.

Audited context:

```sh
sase bead read bob-cli-ft.3.1 -r "Need landing audit, follow-up triage, and children"
sase bead read bob-cli-ft.3 -r "Need parent phase scope and final decisions"
sase artifact read plan:202611/block_marks_editor.md "Need remaining rendering contract"
sase artifact read plan:202611/blocked_reason_marks.md "Need parent surface boundaries"
sase artifact read file:explicit:1656bef2dea171eba6964f38 "Need all four failing regression probes"
sase artifact read file:explicit:7a6f9f6a48c085ec27cb91b6 "Need their failing results"
sase artifact read file:explicit:ee8f976abf31be54c7612c4b "Audit the prior real-Obsidian report"
```

The land audit read the epic and every child note, reviewed plugin commits `89dae58`,
`257255e`, `5f1c23c`, `bded172`, and CLI documentation commits `311b1514` and
`a7931a64`. At bob-cli `a7931a64` / bob-plugins `4502b73`, all 46 block-mark
model/render/contrast/editor/reading/offline-harness tests pass. Another 158 neighboring
freshness/priority/date/recurrence/task-tag/calendar tests pass, including consumers of
the changed shared fence guard. Four added landing probes fail on that same tree.

The original decisions remain final: **own_first**, **chevron**, **quiet**, and
**memory_decision_record=no**. Stored Markdown stays `[?]`; the shared classifier
remains the only reason authority. No painting code writes the vault, wraps or replaces
checkboxes, intercepts clicks, or reads files per row. Priority/project folds, Tasks
query CSS/BM24, chip adoption, Task Card adoption, and the outer visual-review phase
remain with `bob-cli-ft.4`–`.8`.

## 1. Repair the rendered-row lifecycle with regressions

In `142g-block-marks-reading.js`, address these reproduced failures and cover the
adjacent ownership cases in `scripts/test-ledger-tools-block-marks-reading.cjs`:

1. **Recycled fallback rows keep their old reason.** Render a `[?] #task` row with raw
   `[scheduled:: 2026-11-20]` and no section info; then replace its own trailing text
   with a new task without a date, leaving the same DOM row and status. A sweep still
   paints `date`: `blockMarkReadingClassifyRow` prefers retained text forever. Retain
   canonical source only while its origin and row identity are still proven. Re-evaluate
   trustworthy current raw fallback text; invalidate on source/path/row changes and
   uncertain reuse, including query rows. Preserve marks through legitimate downstream
   field transformations when the source identity is still known. Do not solve this by
   title matching, file reads, or discarding all retention.
2. **Repeated cleanup overwrites another plugin's metadata.** Paint a row whose checkbox
   has `aria-label="Original"`; turn the gate off and sweep; set a new external aria
   label; sweep again while off. The code writes `Original` back.
   `blockMarkReadingRestoreOwned` treats null applied values as permission to restore,
   and the stale ownership record survives clear. Release ownership once restored;
   restore only attributes still owned, and capture a fresh baseline on reacquisition.
   Handle replacement checkbox nodes without restoring the old node's attributes onto
   the replacement. Cover repeated off sweeps, neutral transitions, unload, external
   edits, and toggle-on/off after the external edit.
3. **Tasks exclusion misses the processor root.** Pass a
   `li.task-list-item.plugin-tasks-list-item` directly to `renderBlockMarksIn`, with raw
   task/date text and a source context. It receives `date` because the ancestor skip
   loop stops before examining `root`. Check the root itself, and ensure code/pre
   exclusion has the same boundary correctness. Preserve ordinary nested-row handling.
4. **A query without source identity gets a mark.** In the existing test helpers, create
   `readingEl(doc, "div", {class: "block-language-dataview"})`, append
   `taskRow(doc, {trailing: " Hand [scheduled:: " + FUTURE + "]"})` with no path or
   backlink, and render with `{sourcePath: "Queries/Inbox.md"}`. Expected no mark;
   actual `date`. Require a proven origin for query results before applying or reusing
   marks, and keep the query-note stem out of project reasons. Add origin-loss and
   same-status origin-change cases. This must not stop source-note fallback rendering.

The complete four-probe snapshot is attached above; the earlier result log covers the
first three, and the audit rerun failed all four. Move their substantive assertions into
the permanent suite, using existing helpers rather than duplicating its test setup.

## 2. Make the existing Obsidian verification prove its claims

Repair `scripts/obsidian-block-marks-e2e.cjs` and its offline self-test; use the
existing isolated vault, profile, CDP ownership checks, watchdog, and cleanup. Keep all
scratch data under the active workspace. Supply externally copied assets through
explicit, audited paths when repository access is needed; no live-vault edits. This is
completion of the accepted child plan, not expansion into parent phases.

The prior report says `hover-tooltip-text: ok` while its detail has **`tip: ""`**. The
assertion accepts `aria` OR tooltip text, and its tooltip selector also includes
`[data-tooltip-position]`, which can select the checkbox itself. Drive a real pointer
hover through CDP, wait for a visible Obsidian tooltip, and assert its exact reason text
separately from aria metadata. Remove the aria-only passing path. If this exposes a real
lazy-attachment defect, fix it within the editor adapter using the approved
requestMeasure write-pass fallback; preserve checkbox identity and native clicks.

Close the related false-positive gaps in the existing harness:

- Verify marks inside the actual embed/popover/query container, not any marks on the
  page or mere popover existence. Check a named, provable Dataview row and its own box;
  explicitly distinguish neutral unprovable rows. The old report's `1 marked of 58` is
  aggregate evidence, not proof that a particular source row renders correctly.
- Add a standalone P3 row (P3 currently appears only as a losing pip), and prove owned
  checkbox paint for each key rather than just row attributes. Retain nested and quoted
  cases and both themes/views. For a source form the host does not render as a checkbox,
  report that accurately instead of claiming checkbox paint from a line attribute.
- Disable the actual snippet stylesheet through the scratch app's supported mechanism or
  identified style node, confirm it is absent, and check computed glyph/mask and family
  paint. Removing only matching `link` elements and checking a nontransparent background
  does not prove this. Restore the scratch setting afterward.
- Make the natural refresh change a reason/key or exact tooltip date. The existing edit
  uses whole-note `text.replace`, can hit an earlier equal date, and only checks that a
  date mark remains a date. Target the exact fixture row and prove its changed result
  without reopening or directly invoking refresh methods.
- Actually exercise BM6 cold-to-ready cache, BM22 changed ladder configuration, and BM23
  a controlled date boundary through normal invalidation seams. Current static
  today/future rows and a ready-cache precondition are not those scenarios. Label
  controlled-clock evidence accurately; no need to wait for real midnight. Prove
  toggle-off removes owned tooltip metadata and restores computed appearance, then
  reactivates in place; keep the native on/off click comparison.

Extract pure assertion/report helpers where useful and make the offline self-test reject
absent/empty tooltip text, unchanged refresh evidence, wrong host origins, and falsely
claimed snippet removal. It must remain clearly labeled offline.

Update `docs/blocked-marks.md` and plugin docs only to match the evidence actually
obtained. Fix any rendering defect these strengthened checks expose within this scope;
do not weaken assertions to regain green or mark required unrun coverage as passed. Read
and compare the new light/dark screenshots with the accepted mockup
`file:default:6f2a31a39e300befd5a88907` through audited artifact reads.

## 3. Integrate and verify the repaired tree

Recheck drift since the audit's two SHAs. The audit found no new consumer needing the
miniature API outside the explicitly deferred parent phases. Intervening plugin
Stats-clock paint (`75e7875`), off-clock tag rendering (`66741d4`), and
relation-successor changes (`65d25d9`, `f4550e2`, `2ead3a8`, `e9f2fb7`, `4502b73`)
preserve the block-mark contract. In particular, starts-after relations do not become
blocking prerequisites. Intervening CLI section completion, event-link ordering/default
starts, successor planning, text escaping, and CANCEL LOG indentation do not consume
this display API. Both audited branches were master with no additional origin commits
after fetching. Keep these integrations intact and inspect any newer overlap before
closing.

Build and validate the plugin, bump its current manifest once for this repair,
regenerate docs inventory, and run focused block-mark plus affected shared-helper tests.
Run `npm test`, `npm run validate`, `npm run docs:check`, and primary `just docs-check`;
run **`just check`** for file-change verification, never `just check-full`. Use
`/sase_monitor` for known long commands, with an explicit continuation that completes
this tale and landing. A monitor is not an excuse to end with an unobserved command
still running. Preserve new real-Obsidian report and screenshots as artifact references
on this epic, and inspect the screenshots.

Run `bob plugins sync` after plugin changes per the linked repo instructions. Respect
its dirty-deployment protection; do not force over another agent's work. The prior
phase's final note deferred deployment of 1.90.0, so verify deployment of the repaired
build rather than assuming that note's version is current. No requirement below waits
for this turn's eventual commit SHA, push, or CI result: the host commits after the
turn.

External follow-ups have already been triaged by the lander using `/sase_new_task`. Do
not file duplicates or turn these into epic work:

| Proposing child notes                  | Existing outcome                                                                                                                        |
| -------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------- |
| `.1 #1`, `.2 #1`, `.3 #4`, `.4 #1`     | Corroborated `bob-cli-et`, real-date freshness reload fixture                                                                           |
| `.1 #1`, `.2 #1`, `.3 #1`, `.4 #1`     | Corroborated `bob-cli-f7`, UTC/local Keep notice fixture                                                                                |
| `.1 #1`, `.2 #1`, `.3 #3`, `.4 #1`     | Corroborated `bob-cli-eh`, two startup callback fake failures                                                                           |
| `.1 #1`, `.2 #1`, `.3 #2`, `.4 #1`     | Corroborated `bob-cli-ff`, expired future schedule date                                                                                 |
| `.2 #1`, `.4 #1`                       | Corroborated `bob-cli-f8`, snapshot/rank CPU investigation; cold-chunk observation retained without claiming identical cause            |
| `.2 #1`, base-only reduced-motion miss | Declined a separate task: focused current motion tests pass 2/2, as did the phase control; no independently reproduced distinct failure |

Independent evidence: `file:explicit:55a612058c1373e9ca6880b1` (112/119 passing; the
five known deterministic nodes plus warm snapshot 75.0ms and cold chunk 16.8ms; rank
passes this run). The tests and index code were unchanged by this child epic. Do not
call all phase full-suite failures identical or assume all are load-only flakes. All
active epic scopes were checked; no active epic causally owns these external failures.
No new task was created. Carry every disposition into the close note.

## 4. Finish this epic and its parent phase in this same coding turn

After the repaired scope is verified, reread the epic, all four children and notes, and
linked plan to confirm no new unresolved epic work. Their original phases are already
closed; this tale completes the landing gaps. Then perform these concrete actions
without waiting for this turn's own commit:

1. Run `sase bead epic-symbols bob-cli-ft.3.1`. Resolve every entry or re-key its
   Justfile exemption only to a still-open later bead that demonstrably needs it. The
   audit found none, but rerun now. Do not leave a closing bead's exemption stale.
2. Close normally:
   `sase bead close bob-cli-ft.3.1 --note "<verification of all child scope and notes, decisions, four rendering repairs, real-app evidence, intervening integration, checks, and every follow-up disposition above>"`.
   If refused, finish the named work or symbol cleanup and retry. Never force merely to
   succeed and never use `--force` to advance this successful nested landing.
3. Run `just symvision` if the recipe is available (the audit found none), including in
   changed repos where provided. Open the plans repo through `/sase_repo`, resolve the
   actual PLAN path reported by the epic read, and set `status: done` in
   **`202611/block_marks_editor.md`** frontmatter. Do not mark the outer plan done.
4. Reread `bob-cli-ft.3.1` for its parent link and `bob-cli-ft.3` for readiness. Its
   parent is the **phase `bob-cli-ft.3`**, not a plan ancestor. Verify that the closed
   child covers that phase's source rendering, CSS/API, lifecycle/tooltips, contrast,
   and real-Obsidian harness. Check/retire any `sase bead epic-symbols bob-cli-ft.3`
   entries, then close only that phase normally:
   `sase bead close bob-cli-ft.3 --note "Verified child epic bob-cli-ft.3.1 and its landing repair complete the editor_marks scope; <concrete checks and evidence>"`.
   Leave **`bob-cli-ft`** and `202611/blocked_reason_marks.md` to the already-waiting
   outer land agent. If a new parent-phase blocker appears, record it on that phase and
   report it instead of forcing closure.
5. Use `/sase_final` as the final action; include every changed repo (plugins, primary
   docs, and plans) in the host declaration. Report the actual completed bead IDs and
   material verification limits. Do not wait for or manually make a commit.
