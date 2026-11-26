---
tier: tale
title: Finish tie chord dispatch and reversal composition, then land q8
goal:
  Make tie chords, ID prompts, and reversal side effects honor the approved contract,
  then close bob-cli-q8.10 and its ready parent bob-cli-q8.
size: medium
proposed_by: bbugyi200.apollo.bob-cli-q8.10.land
proposed_with: codex/gpt-6-astra
bead: bob-cli-q8.10
create_time: 2026-11-26 06:04:21
status: wip
---

- **PARENT:**
  [202611/finish_plain_link_ties.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/finish_plain_link_ties.md)
- **BEAD:**
  [bob-cli-q8.10](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-q8/bob-cli-q8.10.md)

# Finish tie chord dispatch and reversal composition, then land q8

## Scope and inherited decisions

Finish the remaining implementation defects found while landing **bob-cli-q8.10**. One
coding agent can make these bounded navigation changes and finish the landing. This tale
has no land agent: its final step must close q8.10 and, after the parent readiness audit
below, q8. No step may depend on this tale's eventual commit SHA, push, or CI result.
Closeout happens in the same turn as the final code.

Read these with `sase artifact read`:

- `plan:202611/finish_plain_link_ties.md` — the current epic's acceptance criteria.
- `plan:202611/plain_link_dependency_ties.md` — authoritative behavior, strings, writer
  table, and reliability rules.

Reviewer decisions on bob-cli-q8 remain final: `ctrl_shift_l = yes`,
`memory_decision_record = no`, `memory_glossary_relation = yes`. The approved glossary
sentence already exists exactly once. No memory changes are needed.

Governing decisions:

- `decisions:task-deps-are-depends-on-links`: dependency edits retain the canonical
  Depends-On line and its derived fields; one gesture uses one editor transaction.
- `decisions:task-lanes-are-sticky`: retain dependency lane transfer and recovery.
- `decisions:consumption-keeps-notes-and-recorded-time`: newly Blocked dependents lose
  live Pomodoro links without losing authored notes or recorded time.
- `decisions:projects-block-from-prj-line`: retain resident blocking and cleanup when
  either side of a reversal is `^prj`.
- `decisions:subprojects-inherit-project-blocking`: preserve the existing subtree pass
  and its descendant effects when a project dependency reverses.
- `decisions:bob-cli-releases-ride-a-daily-train`: do not edit bob-cli Cargo.toml
  versions or CHANGELOG.md.

Use `/sase_repo` to open bob-plugins; read its AGENTS.md. Edit navigation source
fragments, never generated main.js. Keep each fragment at most 1000 lines, add new
fragments to fragments.json if needed, rebuild, and deploy with `bob plugins sync`.
Follow its deployment conflict procedure without forcing an overwrite.

## What the lander verified

All notes on q8.10.1–q8.10.3 were read. No `PROPOSED FOLLOW-UP:` entries exist. The epic
has a LAND FOLLOW-UP TRIAGE note recording that outcome; no unrelated task proposal
remains to file. Problems below belong to this epic.

Primary commits reviewed: 6e722fc6, 1c23bce7, 8cb474ba. Plugin commits reviewed:
6fb169bb, e82bd932, b4ab2cbb, 561a9ea7, 2cad24b7, b9b3cec9. PT12b/c and LR27 CRLF, LR31
uncertain/quiet-deferred cases, and LR35 whole-document comparison are present.
Excluding `dry_run`, `applied_files`, and `recovery_directory` from LR35 equality is a
justified, documented distinction between planned effects and apply-only receipts. The
complete JS fixture data equals the Rust JSON fixture. Do not redo these changes.

The lander ran the five tie suites (plain-link-ties, tie-change-writer, tie-stage-copy,
tie-chords, counted-tie-absorption): 62/62 passed, despite the defects below.
`npm run build:check` passed. `sase bead epic-symbols bob-cli-q8.10` found no entries.
The current bob-cli justfile has no symvision recipe; check availability again at
closeout.

Primary `just check` run **231f6b89bd79a9cbc17778a0ba1b3d92** on untouched 292d60c0
completed with exit 101: formatting and gated Clippy passed, and only the lib/CLI
targets failed, reproducing 97 known failures (42/55). Eighty drawer-cutover cases
belong to active bob-cli-ph, 17 settlement cases to active bob-cli-q4. The lander used
`/sase_new_task`, searched all task statuses, swept the past week's tasks, inspected
active epics and the causal ph.10/q4.6 phases, and appended corroboration to ph and q4.
No duplicate task was created. WISPr passes here, so no new q9 corroboration is
warranted. The relation library tests (41), reconcile relation CLI tests (49), and
capture relation tests (33) all pass. Read the named ToolRun for complete output rather
than assuming known unrelated failures are missing q8 work.

Post-start integration reviewed through bob-cli 292d60c0 and bob-plugins 4ae966f4:
snooze capture and completion, planned-today review exclusion, forecast enrichment,
settle acceptance, caret/drawer documentation, navigation settled review identities,
Bank acceptance and home-card ownership. No additional tie writer or conflicting tie
representation appeared. Preserve the settled editor commit adapter and review identity
behavior while fixing the composition below. Inspect any newer drift before closing.

## 1. Make chord dispatch agree with its displayed result

In `plugins/bob-navigation-hotkeys/src/451-picker-tie-chords.js`, `tieCommitChord`
refuses every `item.disabled` before evaluating the desired-state change. But a disabled
native-add row may have a legal reversing chord. The current strip can correctly say
`⟲ It waits on this task · ⌃B reverses` while the key only notices
`would create a cycle` and performs zero writes.

Reproduction in the existing navigation harness: owner O and other X share n.md; X has
`Depends-On: [[#^o]]`; the Depends-on row for X is disabled with
`disabledReason: "would create a cycle"` and `tieInfo.waitedOnBy = true`.
`dependencyStageConsequenceStrip` offers the reversal, but dispatching the Ctrl+B
handler never calls `applyTieChange`. The symmetric Blocks case needs Ctrl+Shift+B.

- Keep Enter's native-add cycle refusal. Admit a chord only when its desired-state plan
  is valid, including the trial graph that removes the reversed edge. Retain
  eligibility, stale-row, malformed input, project, and third-task cycle guards. Do not
  simply clear `disabled` or permit all disabled rows.
- In `tieChordHintsForItem`, do not unconditionally omit `chord.to === enterTo`: a
  reversing chord has a different effect from the disabled native Enter. The current
  reproduction's footer lists only Ctrl+L and Ctrl+Shift+L, omitting the promised
  Ctrl+B. Show the legal reversal in both dependency-stage footers and suppress guarded,
  stale, batch/count-ineligible, and no-op chords.
- Test real keyboard/modal dispatch through the writer, not just helper strings. Cover
  both directions, all four chord modifiers, a third-task cycle, ordinary eligibility
  refusal, and a stale selection. Assert committed bytes or zero writes.

## 2. Complete the block-ID prompt flow before writing

`tieCommitChord` checks missing IDs before `tieChordRefusal`. A marked or counted
ID-less row therefore opens the prompt despite R8. Move the batch/count guard ahead of
prompt setup and retain the pressed chord's correct refusal name.

`430-picker-vault-batch.js::confirmTieChangeBlockId` writes the first missing ID, clears
`pendingTieChange`, and calls `applyTieChange` with `promptBlockId: null`. When both
owner and target lack IDs, it returns false with only the owner's ID written, no tie,
and no pending prompt. Reproduction: two plain open `#task` rows without trailing IDs;
confirm `owner` in the first prompt. The resulting first row is
`- [ ] #task Owner [id:: n__owner] ^owner`, the second is unchanged, and the operation
reports failure.

- Reuse the existing prompt flow, gathering every required ID before committing
  preparations and the desired tie. Retain reserved-ID collision checks and fresh
  snapshot validation; same-note ID preparations should join the final transaction.
- Esc before final confirmation must write neither IDs nor tie bytes, including
  cancellation at the second prompt. Reset pending state cleanly on cancellation,
  navigation, and stage re-entry.
- Cover owner-only, target-only, both missing, same-note/cross-note, duplicate ID, stale
  note while prompting, cancellation at each prompt, and marked/counted refusal. Invoke
  actual confirmation handlers and assert note bytes, notices, and transaction counts.
  Replace the current tests that merely regex-match `confirmTieChangeBlockId` or the
  comment `Esc writes nothing`.

## 3. Compose the full reversal effects into one transaction

`580-plugin-properties-and-dependency-edit.js` combines a second dependent plan by
spreading the first plan and replacing only some fields. This preserves the first plan's
`summary` and `parentLineNext`, although the second plan may insert/delete lines and
create the newly Blocked dependent.

Concrete reproduction using the actual writer and TransactionEditor from
`scripts/navigation-hotkeys-harness.cjs`:

```markdown
- [ ] #task Owner ^owner
- [?] #task Other [dependsOn:: Tasks__owner] ^other
  - ⛓️ **DEPENDS ON:** [[#^owner]]
```

Reverse to owner-waits. With `stampDependencyParentLine` instrumented to record its
arguments and return unchanged text, the writer stamps line 1 (now Owner's Depends-On
drawer) and line 0 (Owner); Other has shifted to line 2 and is missed. Instrumenting
`readDeferredPomodoroSnapshot` records zero calls: the second add's
`summary.pomodoroTargets` is discarded, so Owner's newly blocked live link is not
consumed. The existing test only checks the edge and undo group, so it passes.

- Compose effects from both plans explicitly. Resolve both dependent identities against
  the final postimage before freshness stamping (including shifts from tie absorption),
  union ordinary and project Pomodoro targets, and preserve both sides' project/subtree,
  lane, preparation, mirror, and recovery effects.
- Keep the final settled editor commit in one transaction; preserve dependency-first
  cross-note ordering and the existing partial-failure notice for cross-note reversal.
- Test both task orders and both reversal directions, meaningful freshness stamps on
  exactly the task rows, daily link consumption for the newly Blocked side, authored
  child notes, recorded closed time, and a `^prj` side with residents. Use a valid
  status registry and the active editor binding in the harness. Verify final bytes and
  one undo restores the entire same-note preimage.
- Finish the promised counted-writer coverage in
  `scripts/test-navigation-counted-tie-absorption.cjs`: its cross-note case currently
  invokes only `absorbTiePairsFromContents`, not the 420 counted executor. Exercise real
  counted/marked entrypoints, same-note and cross-note absorption, mirrors, malformed
  LINKS/follow-up failure notices, lane transfer, and consumption. Fix any defects those
  bounded acceptance cases expose using the existing absorption helpers. Do not build a
  new tie engine.
- Retain the stage copy promises while touching these paths. In
  `dependencyStageConsequenceStrip`, blocked-owner and closed-prerequisite early returns
  currently omit the plain-link replacement prefix and kept-label suffix. Compose
  applicable segments from the planned result in those branches too; add focused
  assertions rather than preserving the incomplete early-return expectations.

## 4. Verify and deploy

Add behavioral cases to the existing suites (or register focused new files in
`test:unit`). Read `docs.md` through `/sase_memory_read` before updating contract docs
or the validation inventory if needed.

Run `npm run build`, the affected tie/dependency/prompt suites, `npm test`, and
`npm run docs:check` in bob-plugins. Run `bob plugins sync` under the linked repo's
deployment rules. For primary file changes, run `just check` through the SASE tool
runner and `just docs-check` for docs changes. Never run `just check-full` for this
tale. Existing unrelated failures already recorded on other epics are not q8 work; new
failures in touched behavior must be resolved. Use `/sase_new_task` for genuinely new
unrelated findings and record every triage outcome on q8.10 before closing.

## 5. Finish this landing and its parent in the same turn

No SHA, push, CI, or returning land agent is a prerequisite for these steps.

1. Re-read `sase bead read bob-cli-q8.10 -r "Verify final epic readiness"` and all
   descendants and notes, the linked plan, this tale, and post-audit commits in both
   repos. Verify the remaining fixes above and the original child scopes. If this tale
   is tracked as an open descendant, settle/close its implemented work normally before
   closing its ancestor so there is no unfinished descendant; never force.
2. Run `sase bead epic-symbols bob-cli-q8.10`. For every entry, resolve it by wiring,
   privatizing, adding an appropriate non-test pragma, or deletion, or re-key it only to
   a still-open later bead that genuinely needs the exemption. Finish the cleanup
   yourself. Then run
   `sase bead close bob-cli-q8.10 --note "<verified scopes, decisions, integration, tests, and complete follow-up triage outcomes>"`.
   If closure reports unfinished named phases, finish or reopen them; do not force a
   successful nested landing. Use forced canceled/superseded resolution only for a
   genuinely deliberate abandoned scope, never merely to make close succeed.
3. Run `just symvision` if the recipe exists. Mark `status: done` in the frontmatter of
   **plan:202611/finish_plain_link_ties.md**, using `/sase_repo` to open the plans repo
   and the PLAN path from the audited read, not a sibling checkout.
4. Read q8.10 again for its parent link; the concrete parent is **bob-cli-q8**, a
   plan/epic, not a phase. Re-read q8's previous LAND TRIAGE and LAND VERIFICATION
   notes, every descendant and note (q8.1–q8.9, q8.10 and its descendants), and
   **plan:202611/plain_link_dependency_ties.md**. Recheck descendant and linked-plan
   readiness and all drift since the previous landing. Preserve already-recorded
   follow-up outcomes; q8's prior note routed unrelated failures to q4/q4.10.1/ph and
   the Mac convert-key feature to bob-cli-qb. Verify the original Mac scope through its
   linked repo and q8.9's recorded CI evidence if unchanged; do not demand this tale's
   future commit SHA or unrelated Mac CI.
5. If q8 remains fully complete, run `sase bead epic-symbols bob-cli-q8`, retire all
   entries, close normally with
   `sase bead close bob-cli-q8 --note "<what was rechecked and verified>"`, run
   `just symvision` if available, and set `status: done` in
   **plan:202611/plain_link_dependency_ties.md**. Repeat only for directly parented plan
   ancestors that are fully complete. If an ancestor is incomplete or ambiguous, leave
   it open, add a concrete blocker note, and report that outcome. If a parent is instead
   a phase, verify and close only that phase normally and leave its containing epic to
   its land agent.
6. Finish through `/sase_final` with commit decisions for every edited repository and a
   final response naming the closed epics or the precise remaining blocker.
