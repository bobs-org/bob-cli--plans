---
tier: tale
title: Finish Event link times documentation and land bob-cli-gg
goal:
  Correct the two remaining copy defects, verify and deploy the combined feature, and
  close bob-cli-gg.
size: small
proposed_by: bbugyi200.apollo.bob-cli-gg.land
proposed_with: codex/gpt-6-astra
bead: bob-cli-gg
status: done
---

- **PARENT:**
  [202611/event_link_times.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/event_link_times.md)
- **BEAD:**
  [bob-cli-gg](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-gg/README.md)
- **AGENTS:**
  - [bbugyi200.apollo.bob-cli-gg.land](https://github.com/bobs-org/bob-cli--agents/blob/main/sessions/bbugyi200.apollo.bob-cli-gg.land.md)
- **COMMITS:**
  - [ee5c078](https://github.com/bobs-org/bob-cli/commit/ee5c0781caa7c748713ab685de9a9dc7da000f78)
    — docs(task-link-marks): describe event pill and off chip as additive

# Finish Event link times documentation and land bob-cli-gg

## Goal

Finish the two copy defects found during landing, verify the combined feature, deploy
the corrected plugin description safely, and close **bob-cli-gg** in this same tale.
This tale's coder owns the epic closeout; no separate land agent resumes afterward. Do
not wait for this tale's own commit, push, or CI: the host commits after the turn.

## Verified context

The original plan is `plan:202611/event_link_times.md`. Its final decision is
**placement = before**. All three phase beads (`bob-cli-gg.1`, `.2`, `.3`) are closed
with resolution done. The lander reviewed every note, the approved plan, the actual
implementation, and these commits:

- bob-cli: `b9c482da` (contract) and `a5e1c519` (live verification report).
- bob-plugins: `8ce2feb` (pill implementation and shared clock fix) and `9b5b285`
  (isolated Obsidian harness).
- Concurrent off-clock chips: `23d401b`, `46b25c6`, and land integration `84ea548`. The
  last commit corrects bob-ledger-tools to **1.97.0** and adds a paired production
  Reading-sweep test for ordering, clock parity, repeated refresh, and independent
  gates. Preserve that version or any newer version already on master.

Both ledger host classes, both source-fill refresh callbacks, the single cache freshness
fix, shared off-clock icon, source fragments, exports, generated build, test
registrations, and shared CSS are integrated. bob-cli's later period-note commit
`020a1f6d` retains both contract registrations and the correct count (38); period notes
do not qualify as daily-note event-pill surfaces.

The lander ran the event-link, off-clock-link, gcal-event, and event E2E self-test
suites together on bob-plugins `84ea548`: **85/85 passed**. `npm run build:check`
passed. A guarded sync of that clean revision succeeded with all three plugin assets
unchanged and clean deployment provenance, resolving gg.2 note #2 and gg.3 note #3.

The original plan explicitly permits a skipped real-Obsidian run when Obsidian or Xvfb
is unavailable. That is this host's situation. The existing contract accurately records
the offline self-test and the absence of screenshots and live timings; keep that
truthful limitation. Do not require installing Obsidian to close this tale.

## Follow-up triage already completed

`bob-cli-gg.2` note #3 and `bob-cli-gg.3` note #2 both propose regenerating
`docs/reference/glossary.md`. They are one issue caused by active epic **bob-cli-gh**,
whose phase `.1` changed Period Note, Stats Strip, and Day Strip in `2a9640a6` /
`4ce79094` without regenerating the published glossary. The lander used `sase_new_task`,
searched all task statuses, swept recent tasks and active epics, read gh and gh.1, and
appended independent reproduction to gh as a `DISCOVERED ISSUE`. No duplicate task was
created. This outcome and the remaining work are recorded in the gg landing note.
Preserve this disposition in the close note; do not create a second glossary task or
edit canonical memory.

The current docs gate passes lint and plugin inventory, then fails only that
generated-glossary drift. If still present during verification, record its attribution
to gh rather than treating it as unfinished gg work.

## Implementation

1. Read `sase bead read bob-cli-gg -r "Need landing notes and parent link"` and its
   phase notes, and use
   `sase artifact read plan:202611/event_link_times.md "Need approved contract and closeout context"`.
   Use `sase_repo` to open `bob-plugins` and read its `AGENTS.md`. Read the docs memory
   through `sase_memory_read`. Review any commits newer than the verified revisions
   above for changes to these same files, preserving both features.

2. In bob-cli `docs/task-link-marks.md`, rule 5 currently adds a second "instead":
   "Ledger rows that link one of the note's Event Tasks wear that event's time pill
   instead of a ring." This violates gg.1 note #2's explicit coordination instruction.
   Keep both contract links, retain the no-ring rule, and state the additive behavior
   directly: an Event Task link wears its time pill before the link; an off-clock one
   also wears the off chip after it. A row can wear both. Do not alter behavior or the
   chosen before placement.

3. In bob-plugins `plugins/bob-ledger-tools/manifest.json`, replace the event-time
   description clause. Its decoded JSON currently contains a literal backslash-u2019 in
   "note\\u2019s" and says "a session toggle shared with calendar event painting". Use a
   normal apostrophe and accurate prose, for example: "Render the same Event time pill
   before ledger Task Links to one of the note's Event Tasks, with its own session
   toggle; calendar event painting must also be on." Preserve the rest of the
   description, including off-clock chips, and retain the current version. Run
   `npm run docs:gen` to update generated inventory in `docs/bob-ledger-tools.md` and
   any other changed generated files. Never hand-edit generated tables or `main.js`. No
   new behavioral test is needed for these copy corrections.

## Verification and deployment

4. In bob-plugins, run `npm run build`, `npm run build:check`, `npm run validate`, and
   the focused suite:

   ```sh
   node --test scripts/test-ledger-tools-event-link-times.cjs \
     scripts/test-ledger-tools-off-clock-link-chips.cjs \
     scripts/test-ledger-tools-gcal-events.cjs \
     scripts/test-obsidian-event-link-times-e2e.cjs
   ```

   Confirm the generated inventory contains the corrected apostrophe and toggle wording.
   Run `npm run docs:check` with `BOB_CLI_DIR` explicitly pointing at your current
   bob-cli checkout; this prevents the script from cloning or resetting its fallback
   cache. Run bob-cli `just docs-check` with the opened plugin checkout mounted through
   `BOB_DOCS_BOB_PLUGINS_DIR`. Attribute only the independently reproduced glossary
   drift to gh; investigate any new failure caused by these edits. Run **`just check`**,
   using `sase_monitor` if the command needs a handoff. Do not run `just check-full`. A
   check-full discrepancy is test infrastructure, not gg work. Review `git diff --check`
   and the final diff in every edited repo.

5. Run
   `bob plugins sync --repo <opened-bob-plugins-path> --no-pull --plugin bob-ledger-tools`
   after the plugin changes. Do not use force against another checkout's uncommitted
   deploy. The lander already sealed the exact combined 84ea548 assets as a clean
   deploy; an incoming checkout containing that revision can safely update the
   description through the normal ancestry guard. If deployment changes concurrently,
   retain both features and resolve safely; record a real blocker rather than claiming a
   refused sync succeeded. No dependency on this tale's future commit is needed for
   guarded deployment.

## Final step: close the epic in this tale

6. Re-read `bob-cli-gg` and all descendants so newly added work or notes are accounted
   for. Verify the linked original plan is ready. Run
   `sase bead epic-symbols bob-cli-gg` **before closing**. Resolve every returned entry
   by wiring, privatizing, adding a valid non-test pragma, or deleting the symbol under
   the Symvision policy; re-key only when a still-open later bead actually needs the
   exemption. The lander's list was empty. Do not leave any gg- or phase-keyed exemption
   behind.

   Close normally with `sase bead close bob-cli-gg --note "<verification>"`. The note
   must cover reviewed phase notes and commits; before placement; combined pill/chip
   integration and version; these two copy corrections; actual test and deployment
   results; the plan-authorized live-test skip; and both glossary proposals routed to gh
   without duplicate tasks. State any known unrelated gate failure accurately. If
   symbols reject the close, clean them and retry. If named phases are incomplete,
   finish or reopen them. Never force to make the close succeed, and never force a
   successful nested landing. Cancellation or supersession requires a deliberate
   non-done outcome and reason, not this success path.

   After closing, run `just symvision` if available; the current bob-cli justfile has no
   such recipe, so record that fact if it remains absent. Open the plans repository
   through `sase_repo` and use the path it prints. Set `status: done` in the frontmatter
   of the original `202611/event_link_times.md`, corresponding to
   `plan:202611/event_link_times.md` (its PLAN location is
   `sase/repos/plans/202611/event_link_times.md` relative to bob-cli). Do not mark it
   done before the epic closes.

   Then run `sase bead read bob-cli-gg -r "Need the parent link"`. There was **no
   parent_bead** at landing audit, so no ancestor closure is currently required. If a
   parent has since appeared, handle it by type: verify and close only a parent phase;
   for a parent plan, review its prior landing note, all descendants and notes, linked
   plan and post-child drift, then rerun readiness and epic-symbol checks before a
   normal close, Symvision check, and plan status update. Repeat only through fully
   complete directly parented plan ancestors. At the first ambiguous or incomplete
   ancestor, leave it open and record its blocker. Do not use force.

   Finish through `sase_final`, including commit decisions for every edited repository
   (bob-cli, bob-plugins, and plans). The host's later commit is the expected order; the
   epic closes in the same turn as the final changes.
