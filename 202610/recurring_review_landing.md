---
tier: tale
title: Restore RECURRING navigation and finish bob-cli-5p landing
goal:
  Deploy the missing recurring navigation and strict hidden-row parity, verify
  integration, and close bob-cli-5p with its plan marked done.
size: medium
proposed_by: bbugyi200.athena.bob-cli-5p.land
bead: bob-cli-5p
create_time: 2026-10-08 12:23:41
status: wip
---

- **PARENT:**
  [202610/recurring_review_tier.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/recurring_review_tier.md)
- **BEAD:**
  [bob-cli-5p](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-5p/README.md)

# Restore RECURRING navigation, fix hidden-reference parity, and land bob-cli-5p

## Scope and verified state

This is the remaining work for epic **bob-cli-5p**, not a new feature or a new
multi-phase epic. One coder must implement these repairs and finish the epic's landing
in the same turn. No step depends on this turn's eventual commit SHA, push, or CI
result. Do not create phases or wait for another land agent.

Read `sase bead read bob-cli-5p -r "Need the verified landing scope and notes"` and
`sase artifact read plan:202610/recurring_review_tier.md "Need the approved RECURRING contract"`.
The original epic has four closed phases, bob-cli-5p.1 through bob-cli-5p.4, and no
parent bead at this audit. Its immutable selected DECISIONS are:

- `tier_position=before_tickler`: PRE → NEW → PROJECTS → PENDING → NEXT → RECURRING →
  TICKLER → REFERENCES → ROTTEN → POST.
- `ctrl_alt_f=refuse`: Alt+F and Ctrl+Alt+F on RECURRING write nothing, give the
  recurring-tier notice, and stay. Alt+Shift+F remains retired.
- `memory_records=no`: do not edit any memory file or generated instructions.

Rust commit **56a5e68** implements schema 11, occurrence-date overlay, queue/counts,
lint and RC1–RC12 in bob-cli. bob-plugins commit **9233b0c** implements the ledger
overlay/footer and freshness namespace v9 with `recurringTier: true`, ledger manifest
1.36.0. Both are present on master and fetched origin/master. There were no non-epic
commits after those first epic commits at the audit. Preserve existing nav
return-to-current-task behavior from **2e2ce32** and all other preexisting navigation,
lane, dependency and Task Card behavior; fetch and inspect any further drift before
implementation and integrate it where necessary.

The nav phase reports commit **515e2e0** and version 2.14.0, but that commit is absent
from the opened linked repo and fetched origin, and audited `stitch:515e2e0` reports
missing. Actual master and deployed nav are **2.13.0** and lack the RECURRING
capability, machine tier, commitment count, gesture refusal, notices and resolution
rules. Restore the missing behavior from the concrete requirements below on the current
base; do not count the phase's close note as implementation evidence.

The lander also reproduced a ledger parity defect. With today 2026-10-08, this task is
excluded by Rust's CLI but becomes RECURRING in JS:

```markdown
- [ ] #task Hidden #hide [repeat:: every week] [scheduled:: 2026-10-01] ^ref
```

In `plugins/bob-ledger-tools/src/160-widgets-and-row.js`, the `tracker === "ref"` hide
bypass makes `laneVisible=true` for this recurring row. The recurring contract requires
ordinary visibility, including exclusion of `#hide`, even on trackers. Rust's
hidden-tracker candidate scan already rejects recurrence.

The audit passed **320/320** existing focused JS freshness/nav/card tests and
`npm run validate` (build check plus 6/6 manifests). Installed bob is schema 11, no
`recurring_undated` warnings; the remaining live overdue occurrences had exact due dates
and overdue days. Bryan was answering the original four census rows during the audit, so
do not require the old count of four. A read-only next-day check (`BOB_NOW=2026-10-09`)
kept `gtd_daily.md` rows in PRE/POST. Morning review already contains the approved
recurring answer text, and vault-sync had matching local/remote SHAs with no errors. Do
not rewrite handled tasks or reinstall a stale plugin source.

## Follow-ups already triaged by the lander

Every child note was reviewed and the outcomes are recorded on bob-cli-5p:

- bob-cli-5p.1 #1, .2 #2, .3 #1, .4 #1 are one deliberately deferred memory proposal.
  **bob-cli-5q**, small memory task, is ready and proposes the new recurring decision,
  supersession of `decisions/review-walk-is-tiered.md`, and `glossary/task-freshness.md`
  update after separate authorization. It has a related link to **bob-cli-3y**, which
  owns different tracker scope/cadence omissions in the same older record. This is the
  explicit approved `memory_records=no` outcome; perform no memory edits here.
- bob-cli-5p.2 #1 and .3 #2 are one preexisting scheduling-clock defect. **bob-cli-5r**,
  small CI task, is ready. In `scripts/test-navigation-roll-decay.cjs`, the two nodes
  `picker-single Ctrl+Enter rolls a P2 task in one undo group` and
  `picker-single opens on scheduled so Ctrl+Enter rolls with no navigation` expect `[?]`
  but get `[ ]` for scheduled 2026-10-08. The harness fixes the preview/recommendation
  baseDate to 2026-09-30 but the single scheduling writer still uses the live clock,
  making the date due today. Both failed on two unchanged-tree runs; phase reporters
  reproduced on their clean bases; nav and this test file are unchanged by 9233b0c.
  Evidence: `file:explicit:d973ed74834aac4768839e70`. Keep this separate task's
  test-clock work outside these repairs; distinguish these exact existing failures from
  any new failure in verification. No other proposed follow-up was declined or lost;
  repeated proposals were consolidated into these two outcomes.

## Implementation steps

1. **Open source and recheck drift.** Work in the bob-cli checkout. Use
   `sase repo open bob-plugins -r "Implement bob-cli-5p remaining nav and parity repairs"`
   and use only the path it prints for that repo. Read its AGENTS.md. Review fetched
   master/base commits since 56a5e68 and 9233b0c, excluding epic commits, and integrate
   any newly landed overlapping behavior. Both plugins use committed fragments: edit
   `src/`, respect the 1000-line fragment cap, register a new fragment if needed, and
   regenerate `main.js` with `npm run build`.

2. **Restore missing nav behavior.** In bob-navigation-hotkeys:
   - `470-keydown-and-freshness.js`: add `reviewFreshnessSupportsRecurringTier(api)`
     requiring namespace `version >= 9` and `recurringTier === true`; recognize machine
     tier `recurring`; count it in `reviewIsCommitmentTier` and `reviewWalkRemaining`;
     add a text-first `matchReviewRecurringCursor` patterned on the checklist matcher.
     Preserve the boundary notice's ROTTEN/POST-only trigger.
   - `480-review-jump-and-nav-api.js`: render RECURRING fallback detail as due today or
     N days overdue, with the real completion/Today/reschedule/skip hints. Add the exact
     recurring-tier notice:
     `RECURRING · never stamped — Ctrl+Enter done · Ctrl+Shift+Enter today · Ctrl+Shift+P reschedule · ]s skip`.
   - `520-plugin-lane-links-and-review.js`, `refreshTaskFreshnessOnTasks`: before stamp
     classification, for a single uncounted target matching a live recurring queue row
     and the v9 capability, both Alt+F and Ctrl+Alt+F show that notice, write nothing
     and stay. Preserve non-landing refusals, counted/Task Link batch behavior and
     capability fallback.
   - `538-review-walk-identity.js`, `reviewOutcomeResolves`: completion and `link-today`
     resolve RECURRING; lane and route changes never resolve it. A card resolves only if
     the row closed, its scheduled date moved past today, a new dependency ID appeared,
     or the earliest valid inline scheduled/due/start date moved past today. A freshness
     stamp never resolves it. Keep ordinary tiers and PRE/POST behavior intact.
   - Verify the Task Card's Schedule date pick on a recurring task writes the new
     scheduled date. Its cancel/decay-cancel refusals remain. If rescheduling actually
     refuses, fix the advertised hint on both nav and ledger to say "edit its scheduled
     date"; never advertise an unusable action.
   - Extend `test-navigation-freshness.cjs` for tier/capability/commitment/boundary and
     both no-write gesture paths, and `test-navigation-review-advance.cjs` for the
     resolution table. Add the required Ctrl+Enter fixed-cadence recurrence-insert case:
     completing an overdue weekly Sunday occurrence inserts another still-due occurrence
     above it; the walk must advance once, avoid the completed row, not skip another due
     row, and retain the new occurrence in RECURRING. Test the return-to-current-task
     integration.

3. **Fix recurring visibility parity.** Restrict ledger's hidden exact-`^ref` exemption
   to non-recurring rows in `160-widgets-and-row.js` (or preserve separate strict
   recurring visibility with an equally narrow implementation). Do not change ordinary
   hidden reference review or checklist visibility. Extend
   `test-ledger-tools-freshness-recurring.cjs` through the real row adapter and queue,
   testing hidden recurring `^ref` exclusion, visible recurring `^ref` membership, and
   ordinary non-recurring hidden `^ref` retaining REFERENCES. Add a Rust CLI parity
   regression in `tests/cli/freshness.rs` for the same hidden/visible recurring
   reference rows; Rust should already pass without a behavior change. Keep RC1–RC12,
   tracker and checklist vectors green.

4. **Versions, docs, and verification.** Bump nav from 2.13.0 to the next unused minor
   (expected 2.14.0), and ledger from 1.36.0 to an appropriate unused patch for the
   visibility fix (expected 1.36.1). Update root plugin README and manifest descriptions
   for the ten-tier order, RECURRING no-stamp answers, recurring capability and final
   versions. Run `npm run build`, `npm run build:check`, `npm run validate`, focused
   freshness/navigation/card tests, and `npm test`; address all newly caused failures.
   If the full plugin suite still has only the two independently confirmed roll-decay
   failures tracked by bob-cli-5r, document those exact nodes and baseline evidence
   rather than expanding this feature to that separate task. In bob-cli run
   **`just check`**, the canonical file-change gate. **Never run `just check-full`**.
   For genuinely long verification commands use the SASE monitor workflow and preserve
   the remaining closeout instructions; do not end while an unmonitored command or
   handoff command remains running.

5. **Deploy and verify live behavior.** From the opened current plugin checkout run
   `bob plugins sync --repo <printed-repo-path>` after building. Compare both deployed
   main.js and manifests to this checkout and record final versions. Deployment must
   include this turn's verified uncommitted repairs; do not wait for their future
   commit. Run `bob freshness list -f json` and verify schema 11, all still-open
   eligible recurring rows, occurrence dates/overdue counts, checklist precedence, and
   any undated warnings. Respect user changes to the vault; Morning review was already
   updated and synced. Recheck the existing epic checklist and add a short correction
   note with the actual final versions and live remaining count.

6. **Finish bob-cli-5p landing in this same turn.** This is the final step:
   - Reread the epic and all children/notes for readiness and new issues; all originally
     closed phase scopes must now be implemented and verified. The original PLAN is
     `plan:202610/recurring_review_tier.md`.
   - Run `sase bead epic-symbols bob-cli-5p`. For every listed entry keyed to this epic
     or its phases, resolve it by wiring, privatizing, adding an allowed non-test pragma
     or deleting it under the Symvision policy; re-key only to a still-open later bead
     that truly requires the exemption. Never leave an entry keyed to a bead about to
     close. At the audit there were none.
   - Close normally with
     `sase bead close bob-cli-5p --note "<source/commit and all-note verification; restored nav and hidden-reference parity; integrated post-start changes; test results and baseline CI exception; deployed versions/live check; follow-up outcomes 5q and 5r; symbol cleanup>"`.
     If symbols block the close, clean them and retry. If named phases are incomplete,
     finish/reopen them and complete them normally. Never use `--force` just to succeed
     or to advance a successful nested landing. If SASE gives this tale its own
     still-open descendant bead, complete and close that satisfied tale bead normally
     before the containing epic close; do not wait for this turn's commit to do so.
   - After closing, run `just symvision` if the recipe is available; if absent, record
     that the repo exposes no such recipe. Then open the plans sidecar with
     `sase repo open plans -r "Mark the verified bob-cli-5p plan done"`, use its printed
     path, and set `status: done` in the frontmatter of
     `202610/recurring_review_tier.md`. Read sidecar artifact content only with
     `sase artifact read`, and declare any edited repo in the finalizer.
   - Run `sase bead read bob-cli-5p -r "Need the parent link"`. It had no parent at the
     audit, so normally finish here. If a parent was added, a parent phase gets only its
     verified normal close and its containing epic waits for its own land agent. For a
     directly parented plan, reread its previous landing note, every descendant/note and
     linked plan, inspect post-child drift and rerun readiness checks. Close only when
     fully complete, retire its epic-symbol entries first, run available symvision, mark
     its own plan done, and repeat through complete directly parented plan ancestors.
     Stop on ambiguity or incompleteness, note the blocker on that parent, and report
     it. Do not force successful ancestor completion.
   - End using `/sase_final` so the host commits bob-cli, bob-plugins and any changed
     plan sidecar. Completion must depend on verified source and tests, not on a commit
     the host creates only after this turn ends.
