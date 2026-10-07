---
tier: tale
title: Repair Tasks date marks and finish bob-cli-53 landing
goal:
  Tasks date marks work with real DOM collections, queued frames stop after shutdown,
  and verified epic bob-cli-53 is closed with its original plan marked done.
size: medium
proposed_by: bbugyi200.athena.bob-cli-53.land
bead: bob-cli-53
status: done
---

- **PARENT:**
  [202610/task_date_marks.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/task_date_marks.md)
- **BEAD:**
  [bob-cli-53](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-53/README.md)
- **AGENTS:**
  - [bbugyi200.athena.bob-cli-53.land](https://github.com/bobs-org/bob-cli--agents/blob/main/sessions/bbugyi200.athena.bob-cli-53.land.md)
- **COMMITS:**
  - [01431e8](https://github.com/bobs-org/bob-cli--plans/commit/01431e8c66867ee7f85aaa479f6fff1380ef9566)
    — docs(plans): mark task date marks plan done

# Finish task date marks and land bob-cli-53

Repair two verified defects introduced by epic **bob-cli-53**, deploy the fixed plugin,
and finish that epic's landing in this coder turn. This is one direct implementation
task. The epic's two original phases are already closed; do not wait for another land
agent or for this tale's eventual commit, SHA, push, or CI.

## Verified scope and context

Read `sase bead read bob-cli-53 -r "Need landing verification and remaining issues"` and
the approved artifact with
`sase artifact read plan:202610/task_date_marks.md "Need the original display contract and closeout requirements"`.
The original plan is `plan:202610/task_date_marks.md`, stored under the plans sidecar as
`202610/task_date_marks.md`. Use the path returned by the current `sase bead read` or an
audited `sase repo open plans`; never assume the lander's workspace path.

Open **bob-plugins** with `/sase_repo` and read its `AGENTS.md` before working there.
That repo is the source of truth. Edit fragments under `plugins/bob-ledger-tools/src/`,
never generated `main.js`; rebuild it normally. Each hand-edited fragment must stay at
or below 1000 lines. Fragment 266 currently has 999 lines. Do not refactor freshness or
priority marks.

The lander reviewed every original child and note:

- `bob-cli-53.1`, note #1: implementation, tests, metadata, docs, and deployment;
  primary commit `67f0cbb` and bob-plugins commit `d29034b`.
- `bob-cli-53.2`, note #1: bounded Tasks-result pass, CSS, tests, docs, deployment;
  primary commit `b5a0258` and bob-plugins commit `ced2675`.

Current date-mark suites pass **46/46**, `npm run build:check` passes, and
`npm run validate` passes **6/6**, but the browser defect below defeats the fake-DOM
tests. `bob plugins list --no-pull --repo <opened-bob-plugins-path> --format json`
confirmed **1.32.0**, enabled and byte-identical to the installed vault plugin.
Installed Tasks **8.4.0** confirms the documented row classes, date hosts, final
`data-task` completion signal, and date click/contextmenu handlers.

Integration was checked through fetched primary `origin/master`: non-epic commits since
creation (2026-10-07 08:19:06 EDT) are `3232214` (uv/URL hardening), `2eafe60` (URL
ingest), and `df9d504` (URL routing). They do not change Obsidian rendering or consume
the dateMarks API, so no integration edits are needed for these commits. Linked
bob-plugins has no non-epic commits in that interval. Recheck newer drift before
closing, and integrate only actual overlap.

**Follow-up triage is finished and recorded on bob-cli-53.** Neither child has a
`PROPOSED FOLLOW-UP:` note. There are no distinct non-epic proposals to file,
corroborate, attach, or decline. Both defects below remain epic work; do not turn them
into task beads. If new unrelated follow-ups are discovered, use `/sase_new_task` and
record every outcome on bob-cli-53 before closeout.

The approved Done when criteria explicitly allow the **live Obsidian checklist to remain
pending for Bryan**. Preserve that distinction; do not claim the live UI or iOS was
verified. At landing inspection, bob-cli-53 had **no parent_bead** and
`sase bead epic-symbols bob-cli-53` reported **no entries**.

## 1. Make the Tasks date pass work with real DOM children

In `plugins/bob-ledger-tools/src/267-plugin-date-marks-tasks.js`,
`decorateTasksResultDates` initializes its walk with `(li.childNodes || []).slice()`.
Browser `childNodes` is a **NodeList**, which has no `.slice()`. The catch returns 0
before visiting any date host, so full-mode Tasks results retain native emoji dates
everywhere.

This was reproduced with the exact fragment in headless Chrome:
`typeof li.childNodes.slice` was `undefined`, the decoration count was 0, and no marks
were appended. A reproduction using the existing phase test fixture returned 1 for the
array control, and 0/no `data-bob-date-mark` for the same complete row with an indexed,
iterable, NodeList-like `childNodes` object.

Use a NodeList-compatible snapshot, such as `Array.from(li.childNodes || [])`, and audit
the new date-mark traversal paths for the same array assumption. Keep the bounded walk,
per-host flags, row deduplication, invalid/short-mode fallback, display-only contract,
and Tasks-owned nodes/listeners intact.

Strengthen `scripts/test-ledger-tools-date-marks-tasks.cjs` with meaningful regression
coverage whose children are an indexed DOM collection **without array methods**. It must
fail on the old code, exercise the queued frame pass as well as direct decoration,
produce the correct labels for full-mode hosts, and remain idempotent while preserving
the native inner span. Update fixture helpers as needed so the test does not
accidentally restore `.slice()` on childNodes. Avoid a permanent browser dependency; a
faithful collection fixture is sufficient.

## 2. Stop queued frames after toggle-off and unload

Another existing-fixture reproduction:

1. Create a complete scheduled row and inject the existing manual frame scheduler.
2. Call `scheduleTasksResultDateMarks(row.body)`.
3. Call `toggleDateMarks()` to set `dateMarksEnabled` false.
4. Drain the previously queued callback.

Current code appends one mark even though marks are off. `runTasksResultDateMarkFrame`
checks no enabled state, and `onunload` only clears `dateMarksDay`/the body class,
leaving the enabled state and Tasks queue active.

Guard the frame execution against disabled marks and clear stale pending queue state. On
unload, disable date marks and clear the Tasks queue/pending/dedup state so late
callbacks cannot decorate detached or live rows after the plugin unloads. Keep
scheduling bounded and synchronous/never-throw; a callback that is already scheduled can
safely become a no-op. Verify off/on can still decorate new rows without duplicate marks
or a permanently pending scheduler.

Add regression tests for toggling off before a queued frame runs, a pending-row retry
followed by disabling, and unloading before a callback runs. Assert no DOM write after
shutdown and correct later re-enable behavior. Keep fragment 266 within the 1000-line
limit rather than restructuring unrelated mark systems. There is no native
requestAnimationFrame binding defect: unbound native rAF was confirmed to schedule
successfully in Chrome; do not fix an unverified problem.

## 3. Build, verify, and deploy

Bump bob-ledger-tools to **1.32.1** (or the next unused patch if newer upstream drift
already uses it), update its README version row, and update `docs/date-marks.md` in
bob-cli only if needed to describe the shutdown guarantee or regression coverage. Keep
namespace v1 and top-level API v3.

In the audited bob-plugins checkout, run `npm run build`, the focused date-mark suites,
`npm test`, `npm run validate`, and `npm run build:check`. Fix failures caused by these
repairs. Inspect the generated changes and run
`bob plugins sync --no-pull --repo <opened-bob-plugins-path>` to deploy **this
checkout**. Confirm its deployment with the matching plugins list command. Do not edit
the installed custom-plugin files directly.

Run **`just check`** for file-change verification when the repository provides it.
During landing audit it was attempted in bob-cli and exited with
`justfile does not contain recipe check`; bob-plugins had no justfile. If still absent,
record that limitation and the successful plugin gates rather than adding unrelated
build infrastructure. **Do not run `just check-full`.** If a true long verification
command needs a handoff, use `/sase_monitor`; its successor must finish this tale's epic
closeout. Do not declare completion with running commands.

Refresh the primary base branch and the audited linked checkout's history before
closeout. Review new non-epic changes since the recorded integration audit; resolve any
actual consumers, duplication, or conflicts with date marks and reverify only the
affected work. Read both original child beads again if their notes changed.

## 4. Finish bob-cli-53's closeout in this same coder turn

This is the final implementation step. There is no later land agent for this tale. Do
**not** order any step after this tale's own commit, push, SHA lookup, or CI. The host
commits the final code after this turn ends.

1. Run `sase bead epic-symbols bob-cli-53`. For **every** listed entry belonging to this
   epic or its phases, resolve it (wire up, privatize, add an appropriate non-test
   pragma, or delete per Symvision policy). Re-key an exemption only when a concrete
   **still-open** later bead still needs it. Do not leave any stale entry. Re-run the
   command after edits.
2. Ensure all descendants, both original child notes, original linked-plan acceptance
   criteria, the browser traversal fix, queued-frame shutdown fix, verification,
   deployment, and post-start integration are complete. Include the finished follow-up
   triage outcome and the pending live checklist in the close note. Then close normally:
   `sase bead close bob-cli-53 --note "<specific code, tests, deployment, integration, and follow-up verification>"`.
   If leftover epic-symbols reject closure, finish their cleanup and retry. If a named
   phase is incomplete, finish or reopen it. Never use `--force` to obtain a successful
   close or to advance a successful nested landing; only a deliberate
   canceled/superseded outcome could use
   `--force --reason ... --resolution canceled|superseded`.
3. After successful closure, run `just symvision` **if available**, fixing any remaining
   stale whitelist issue. At audit neither repository offered this recipe; record its
   absence if that remains true.
4. Open the plans sidecar through `/sase_repo` if needed, use audited artifact access to
   the original plan, and set **`status: done` in the frontmatter of
   `202610/task_date_marks.md`** at the current PLAN path returned by bead read. This is
   the original epic plan, not merely this repair tale. Its changed sidecar belongs in
   the final declaration along with every repo you changed.
5. Run `sase bead read bob-cli-53 -r "Need the parent link"` after closing. It had no
   parent at audit, so normally finish here. If a parent link was added in the meantime,
   inspect that concrete bead before acting: for a phase, verify the child plan
   fulfilled it and close only that phase normally, leaving its epic to its waiting land
   agent. For a parent plan, review the previous landing note, all descendants/notes,
   linked plan, and post-child drift; rerun readiness checks, retire all of its
   epic-symbol exemptions, close it normally, run available symvision, mark its plan
   done, and repeat only through fully complete directly parented plan ancestors. At the
   first incomplete/ambiguous parent, append a blocker note there and report it. Do not
   force ancestor closure.

Use `/sase_final` as the last action before the final response, with commit declarations
for every modified repository (including plans). Report the epic's actual close result,
verification/deployment, and that Bryan's live checklist is still pending.
