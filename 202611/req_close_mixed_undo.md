---
tier: tale
size: small
title: Repair mixed close-card Undo and finish the Obsidian requirement landing
goal:
  Make one guarded Undo restore ordinary offer-close and requirement verdicts together,
  then finish bob-cli-on.11.1.7 and its verified parent closeout.
proposed_by: bbugyi200.athena.bob-cli-on.11.1.7.land
proposed_with: codex/gpt-6-astra
bead: bob-cli-on.11.1.7
status: done
---

- **PARENT:**
  [202611/obsidian_req_close_card_repairs.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/obsidian_req_close_card_repairs.md)
- **BEAD:**
  [bob-cli-on.11.1.7](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-on/bob-cli-on.11.1.7.md)
- **AGENTS:**
  - [bbugyi200.athena.bob-cli-on.11.1.7.land](https://github.com/bobs-org/bob-cli--agents/blob/main/sessions/bbugyi200.athena.bob-cli-on.11.1.7.land.md)
- **COMMITS:**
  - [8eb9c65](https://github.com/bobs-org/bob-cli--plans/commit/8eb9c6521c4471844f62ca84055af8dc4a205968)
    — docs(plans): mark req close-card mixed-undo landing done

# Repair mixed close-card Undo and finish the landing

This is the remaining work from the landing of **bob-cli-on.11.1.7**. One coding agent
can fix the known composition defect, add the missing public-path regression, verify,
deploy, and finish the nested landing. This tale has no land agent; its final step must
perform the closeout below. Do not wait for another agent to resume it.

Read the epic and its new LAND notes with
`sase bead read bob-cli-on.11.1.7 -r "Need remaining landing scope"`. Read these plans
through `sase artifact read`:

- `plan:202611/obsidian_req_close_card_repairs.md`
- `plan:202611/obsidian_requirement_close_card.md`

The parent is plan bead **bob-cli-on.11.1**, whose parent is phase **bob-cli-on.11** of
**bob-cli-on**. Do not close bob-cli-on. The approved decisions and existing SP/RV
contracts remain authoritative. In particular, NOTES is born with the first note,
Pending requirements are implicitly Met without writes, the initiating owner stays Done
on card Undo, and a jump happens once after writes settle. This tale edits no memory,
Cargo version, CHANGELOG, or CLI interface.

## Verified baseline and integration review

The lander reviewed every note on child phases on.11.1.7.1/.2, the parent plan's six
phases on.11.1.1-.6, and the previous parent landing notes. At bob-plugins `e1ed13f` /
bob-cli `39661003`:

- 244 focused tests passed: SP vectors/carry API/gesture, RV vectors, twin parity,
  picker/verdict/receipt, req apply/pull-in/landing/integration/public paths.
- An additional public-path probe of idless Carry plus pin passed.
- Child repair commits `44db8ce`, `69a7ca0`, `0bddc2e`, and `21331f3` fix selector
  mapping, synchronous final-text landing, owner snapshot guards, counted and idless
  closes, and the actual v6 row payload. `e1ed13f` isolates the navigation stage
  performance check. Bob-cli `bf005290` corrects carry-now and card docs.
- Completed monitor `16hpvrb8fdz2` reports plugin build/build:check, docs gates, six
  real-Obsidian close-card scenarios, and bob-cli `just check` passing. Its only
  full-suite failures were the two tracked timing nodes, each passing isolated. The
  lander inspected the E2E implementation and monitor result; it did not rerun the live
  harness or claim live-vault testing.
- `bob plugins list` confirmed task-status-cycler 1.58.1 and navigation 2.107.0 synced
  and enabled. Unrelated ledger-tools drift belongs to concurrent work.
- Since this child epic began, plugin commits `c8deb53`, `bb57281`, `8ac4790`, `a5c2269`
  change property-card/ledger-tools surfaces. Bob-cli's props and highlights scan
  changes are separate; `ef640fde` correctly adds the Mac card consumer to shared docs.
  No integration conflict or duplicate implementation was found. Refresh this drift
  check before final close.
- `sase bead epic-symbols` was empty for on.11.1.7, on.11.1, and on.11. Rerun at
  closeout; do not assume the exemption lists stay empty.

Follow-up triage is already complete and recorded on on.11.1.7. Immediate children have
no PROPOSED FOLLOW-UP entries. Parent phase proposals .1 #1, .3 #1, .5 #2, .6 #1 refer
to event-link-times -> **bob-cli-km**; .2 #1, .3 #1, .6 #1 refer to frees-marks ->
**bob-cli-io**. The lander used sase_new_task, searched duplicates, swept recent tasks
and active epics, and added attributed notes without claiming a new reproduction or +1.
No duplicate task is warranted. Parent notes #3-4's pre-existing advisory documentation
headroom stays with **bob-cli-ob**; preserve published API facts. This Undo defect is
epic work, not a follow-up task.

## 1. Repair the mixed-verdict restore composition

Open bob-plugins through `/sase_repo` and read its AGENTS.md. Edit source fragments, not
generated main.js; keep each fragment at or below 1000 lines.

The public managed Done path applies `applySuccessorOfferCloseChoices` before
`applyManagedReqCloseCard` in
`plugins/task-status-cycler/src/137-plugin-successor-project.js`. However, around line
781 it combines restores in the reverse order:

```js
reqCloseIntegrationComposeRestores([...reqRestores, ...closeRestores]);
```

`reqCloseIntegrationComposeRestores` in `198g-req-close-integration.js` keeps the first
before and the last after for a path. Thus the older ordinary-close after overwrites the
final requirement after when both touch one note. The returned Undo guard is stale
immediately. Trace the complete receipt pipeline when fixing it; preserve the earliest
post-initiating-close before and the final successful after, including created flags and
subsequent pull-in writes. Do not weaken the guard, re-read live text to fabricate a
postimage, or reopen the initiating owner.

Reproduction on the public harness at e1ed13f:

1. Use `exporterNote()` plus this sibling in `Tasks.md`:
   `- [ ] #task Offered [dependsOn:: export__task] [id:: offered] ^offered`.
2. Use `makePlugin`, `installReqCloseNav`, an active text editor, `dayNote()` and
   `inbox.md` exactly as in
   `scripts/test-task-status-cycler-req-close-public-paths.cjs`.
3. **Also install the real cancel capability**:
   `plugin.app.plugins.plugins["bob-navigation-hotkeys"].api.cancel = navHarness.helpers.createCancelApi()`.
   Without it the current helper cannot exercise a successful ordinary offer-close. A
   dummy throwing planCancel also refuses the row; do not mistake that for a passing
   mixed test.
4. In the picker, select the `Offered` ordinary row with its close descriptor and the
   `Handles empty input` requirement. Assert `offer.close.complete === true`. Return:

   ```js
   {
     closes: [{ key: offer.key, outcome: "done" }],
     reqs: [{ key: req.key, verdict: "drop", reason: "mixed undo" }],
   }
   ```

5. Call `toggleActiveCheckboxOpenDoneAndPropagate`. Both owner and Offered become `[x]`;
   the requirement becomes `[-]` with its Cancel Log. Both receipts report success. Yet
   `shown[0].undo.restores` records a Tasks.md after with the requirement still `[ ]`
   and no Cancel Log. The real nav `restoreGuardedSnapshots` returns false immediately,
   even though no user edit occurred.

This regression originates in the parent wiring commit `c91ebae`. The child's required
mixed Undo acceptance was not actually tested: public-path test 12 applies only a
requirement Drop, and test 6 selects an ordinary pull-in rather than closing an ordinary
offer. Finish that acceptance here.

## 2. Add public-path regression coverage and accurate Undo prose

Extend the public harness to expose real ordinary-close capability. Add meaningful tests
that run the actual managed Done orchestration and real guarded restore:

- Ordinary Done plus requirement Drop in the **same file**: both writes and receipts
  succeed; exactly one restore per path has the real final after; Undo restores both
  verdicts while the initiating owner stays Done. Include a reason log so shifted lines
  participate.
- Ordinary close plus requirement Carry, with shared source and a separate/created
  destination where appropriate: restore composition includes source, destination, and
  any pull-in writes once; successful Undo retains no orphaned task or stub.
- Change a recorded postimage before Undo and prove it refuses the entire operation
  before any note is written. Retain correct behavior when only one verdict family
  produced successful writes, and preserve actual partial-failure receipts.
- Make the existing offer-close exclusion case genuinely close its chosen ordinary task
  containing Pending requirements, then assert no recursive card opens. Do not accept a
  missing capability or a refused ordinary close as evidence of exclusion.

Keep existing 244 focused tests green. Use the real-Obsidian fixture harness for one
mixed ordinary-close/requirement verdict plus Notice Undo if practical within its
existing setup; at minimum rerun all six existing close-card scenarios after the fix and
record precisely which paths have live versus harness evidence.

Read `docs.md` memory before editing documentation. Correct only the relevant Undo
sentences in bob-cli `docs/task-requirements.md` section 7 and bob-plugins
`docs/task-status-cycler.md`: one card Undo covers its requirement writes and ordinary
offer-close writes together, with exact postimage guards, and keeps the initiating owner
Done. Preserve the separate same-day reopen behavior and avoid unrelated documentation
restructuring or API headroom work.

## 3. Verify and deploy

Run focused req-close/public-path/successor-apply tests, plugin build and build:check,
`npm test`, the real-Obsidian close-card E2E, and plugin docs:check. Run bob-cli
`just docs-check` and **`just check`** for file-change verification. Do not run
`just check-full`. Known event-link-times and frees-marks 25 ms failures may only be
classified as those tracked flakes after the exact nodes pass unchanged in isolation;
record raw and isolated results honestly. Do not blanket-ignore other failures.

Use `/sase_monitor` for long verification as required. Its successor must retain all of
step 4; verification handoff does not assign a new land agent. Deploy changed plugin IDs
with `bob plugins sync --repo <opened-repo> -p task-status-cycler` (and navigation only
if it changed). Never force. Follow AGENTS.md reconciliation if a newer deployment is
protected. Confirm both required plugins remain synced. No step depends on this work's
own commit SHA, push, or post-push CI.

## 4. Finish the nested landing in the same completion turn

This final step belongs to the tale coder after passing verification, including any
monitor successor. Do not stop after implementing or wait for another lander. Closing
the epic in the same turn as the final code is normal; `/sase_final` commits afterward.

1. Re-read `bob-cli-on.11.1.7` and both children with reasons, confirm every note,
   approved decision and acceptance item is settled, and inspect commits/changes since
   the baseline above in both repositories. Record the fix and verification, deployment,
   integration review, and the completed follow-up dispositions above.
2. Run `sase bead epic-symbols bob-cli-on.11.1.7`. Resolve each listed exemption by
   wiring, privatizing, adding a legitimate non-test pragma, or removing the symbol;
   re-key a Justfile entry only to a still-open later bead that needs it. Then run
   `sase bead close bob-cli-on.11.1.7 --note "<verification and triage outcomes>"`. If
   rejected for leftover symbols, fix them and retry. Never use `--force` to advance a
   successful landing or merely to make close succeed. An incomplete phase must actually
   be finished/reopened; only a deliberately canceled or superseded outcome permits the
   corresponding force/reason/resolution procedure.
3. After close, run `just symvision` if available (the inspected bob-cli justfile has no
   symvision recipe; record that limitation rather than inventing a pass). Open the
   plans repository with `/sase_repo`, use audited artifact reads, and set
   `status: done` in `202611/obsidian_req_close_card_repairs.md`, the PLAN path from the
   bead. Preserve all other frontmatter.
4. Run `sase bead read bob-cli-on.11.1.7 -r "Need the parent link"`. Its parent is
   **bob-cli-on.11.1**, a plan bead. Re-read that parent's previous landing notes, all
   descendants and notes, `plan:202611/obsidian_requirement_close_card.md`, and
   post-child drift. Rerun descendant and linked-plan readiness checks; successful phase
   statuses alone are insufficient. If complete, retire/re-key every
   `sase bead epic-symbols bob-cli-on.11.1` entry and close normally with
   `sase bead close bob-cli-on.11.1 --note "<what was rechecked>"`; run `just symvision`
   if available and set `status: done` in its linked plan
   `202611/obsidian_requirement_close_card.md`. If incomplete or ambiguous, record a
   blocker note on on.11.1 and stop ancestor closure; report the blocker.
5. The parent's parent is phase **bob-cli-on.11**. Read it, verify this work fulfills
   its carry-now and close-card scope, retire any `sase bead epic-symbols bob-cli-on.11`
   entries, and close only that phase normally:
   `sase bead close bob-cli-on.11 --note "<verification and deployment evidence>"`.
   Leave containing epic **bob-cli-on** and its plan status to its waiting land agent.
   If the phase is no longer complete, note the blocker and leave it open.
6. Finish through `/sase_final` with commit decisions for every changed repository. The
   final response must distinguish completed closures from any remaining blocker.
