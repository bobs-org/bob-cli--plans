---
tier: tale
title: Finish the cycler verdict landing safely
goal:
  Preserve concurrent edits in Undo, report partial failures, align tickler copy, and
  close bob-cli-hn.6.1 and parent phase bob-cli-hn.6.
size: medium
proposed_by: bbugyi200.athena.bob-cli-hn.6.1.land
proposed_with: codex/gpt-6-astra
bead: bob-cli-hn.6.1
create_time: 2026-11-06 00:49:52
status: done
---

- **PARENT:**
  [202611/cycler_offer_verdicts.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/cycler_offer_verdicts.md)
- **BEAD:**
  [bob-cli-hn.6.1](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-hn/bob-cli-hn.6.1.md)

# Finish the cycler verdict landing safely

Repair the remaining correctness and integration gaps in `bob-cli-hn.6.1`, then finish
its landing and close its parent phase `bob-cli-hn.6`. One coding agent can implement
this bounded repair directly. This tale has no separate land agent; its last step owns
the complete closeout below.

Read `bob-cli-hn.6.1`, all four closed children and their notes, and `bob-cli-hn.6`
through `sase bead read`. Read the approved context through
`sase artifact read plan:202611/cycler_offer_verdicts.md` and
`sase artifact read plan:202611/successor_card_verdicts.md`, with reasons. Reuse the
land audit and follow-up disposition notes on hn.6.1 instead of re-filing them. Read the
applicable memory with `/sase_memory_read`, including `sase_beads.md`, `docs.md` before
changing documentation or user-visible CLI help, and the recovery and review-walk
decisions cited by the approved child design.

The parent DECISIONS are unchanged. This child does no memory editing. The rejected new
decision record and glossary edit stay rejected; the parent's separately authorized
existing-decision update remains its land agent's responsibility. Keep the cycler public
API at v4 unless hn.7 has independently advanced it; do not undo that later work or
implement promotion/Mac features here. No release/version edits.

## Verified baseline and boundaries

The lander inspected bob-cli `3fe50bc7` and linked bob-plugins `1c3aaf1`; both matched
their freshly fetched `origin/master`. Epic commits are bob-cli `44c5c8c8` and
bob-plugins `ef9257d`, `3af8d0c`, `1c3aaf1`. The canonical 22-case fixture at
`tests/fixtures/offer_close/vectors.json` matches the copy under
`plugins/task-status-cycler/scripts/fixtures/offer_close/` byte for byte.

The 99 tests in successor-verdicts, successor-apply, successor-wiring, and
navigation-review-advance pass. Plugin `build:check` and the additional checklist
Ctrl+Enter, nav offer-close API, and nav successor-verdict suites pass too. These passes
do not cover the defects below. Both `sase bead epic-symbols bob-cli-hn.6.1` and
`sase bead epic-symbols bob-cli-hn.6` currently return no entries.

Intervening capture work `efbe3470` consumes the repaired Rust descriptors and adds
receipt copy that must now match the canonical contract. Merge-in work
`fbb6da46`/`3fe50bc7` and Gmail work `232e94bb`/`cfdb68c1` need no new feature
integration identified by this audit. Checklist-stage work in plugins
`1017b38`/`2d0348b`/`4c748d9`/`7a8dd15` and bob-cli `81e0d0c2` is incorporated; preserve
its stage completion and PRE/POST boundary semantics. Inspect any further drift before
editing, especially hn.7's wrappers around the same helpers.

Use `/sase_repo` to open `bob-plugins`, read its `AGENTS.md`, and use only the returned
path. Edit committed source fragments, rebuild generated `main.js`, keep each
hand-edited fragment at or below 1000 lines, and register any new test script in
`package.json`. Extract cohesive helpers from `089i` if needed: it is already at the
limit. Sync changes with `bob plugins sync --repo <returned checkout>`.

## 1. Preserve concurrent edits in guarded Undo

In `plugins/task-status-cycler/src/089i-successor-offer-close-apply.js`, successful root
writes record a snapshot whose `before` comes from `providedNotes`, even when the writer
revalidated and recomputed against newer live text. The lander reproduced this using the
actual cycler and nav harnesses:

1. The provided note is `- [ ] #task Offered ^a\nOld unrelated prose\n`.
2. The live vault note has the same task line but `User edited unrelated prose`.
3. `applyOfferClosesNow` closes `Tasks.md#^a`, preserving that live prose.
4. `undo.restores[0].before` still contains the old prose.
5. Nav's real `restoreGuardedSnapshots` accepts Undo and deletes the user's edit.

Build restores from the exact preimage actually accepted by each successful guarded
write. A root snapshot's `before` must be that write's `guardBefore`; later recovery
writes in that file retain that first preimage and extend only `after`. For a file first
touched by recovery, use its verified write preimage. Never adopt a changed task line as
authorization to close it. Preserve per-row refusal, joint recovery, editor-first reads,
and all-or-nothing Undo postimage checks.

Add regression coverage through the real writer and nav restore, for vault and editor
paths, including an unrelated edit arriving between the original snapshot and final root
revalidation. Undo must retain that edit and earlier ordinary pick/predecessor-recovery
effects. Add a two-file restore case where one later edit refuses all restores with
neither file partially undone. Keep LF/CRLF and final newline bytes intact. Audit every
snapshot creation branch, not only the common successful-root assignment.

## 2. Make partial recovery failures visible in the v4 receipt

The cycler includes `failures` and its plain fallback prints them, but
`plugins/bob-navigation-hotkeys/src/713-nav-offer-close-apis.js` drops that field.
Calling the real `buildUnblockedNoticeModel` with a successful `offer_closes` row and
`failures: [{kind: "recovery", path: "Tasks.md", reason: "disk full"}]` returns
success-only text and `failure: null`. `presentSuccessorModel` considers that successful
rich presentation, so the user never sees the failed dependent write.

Normalize and render verdict-step write/recovery failures in the existing v4 receipt,
preserving its Undo action and one-notice behavior. Handle mixed success and failure as
well as failure-only payloads. Preserve older payload behavior and safe text rendering.
Keep the cycler's plain fallback equivalent; the rich API must never silently discard a
failed recovery. Add an integration test that makes an actual dependent write fail, then
passes the actual result into the real notice model/render helper. Assert the failure is
visible, the root remains closed, the dependent is not reported as recovered, and Undo
covers only successful writes. Exercise the walk-return route so the receipt still
appears once with Undo.

## 3. Finish the shared tickler contract and its intervening consumer

In `src/native/task_complete/offer_close.rs`, `consequence_suffix` counts every
`[unblocked:: today]` occurrence across whole changed files, then compares that count
with the number of recovered rows. It must inspect the actual recovered rows instead. An
unrelated already-stamped row in the same file currently suppresses the suffix when a
new dependent is stamped (2 stamps versus 1 recovery), and can invent the suffix when
the recovered NEW row has no stamp but the unrelated row has one (1 stamp versus 1
recovery). Preserve WRITE skips and independent single-root predictions; receipt
recovery attribution still counts the actual joint result once.

Add executable Rust regressions for both cases, including a stamped unrelated task in
the same note and a NEW dependent. Add shared vectors without renumbering the existing
22 cases, copy canonical bytes to plugins, and make both engines consume the new
expectations. Update fixture provenance and remove assumptions that the case count stays
exactly 22. Reuse the JS row-level stamp checks rather than introducing whole-note
counting there.

Align tickler receipt text with the already-approved parent table and
`docs/task-dependencies.md` §12.22:

- One stamped recovery: `↳ “{D}” back tomorrow as a tickler`.
- Multiple stamped recoveries: `↳ {k} unblocked tasks back tomorrow as ticklers`.
- No stamped recovery: no tickler line.

The intervening hn.3 consumer `src/native/capture/offer_verdicts.rs::tickler_lines`
currently says `also unblocked` and inserts a middle dot. Cycler
`137a-successor-offer-close-integration.js` and nav `713` omit the quotes and always
emit one line per row. Fix those concrete seams and their expectations together. Keep
receipts truthful for mixed stamped/unstamped recoveries; preserve schema, flags,
dry-run behavior, and eligibility. Extend narrow copy tests rather than rewriting
unrelated capture planning. Correct docs only if their canonical table has drifted; read
docs memory first.

## 4. Close the remaining gesture-level acceptance gaps

Extend existing registered wiring/apply tests with the approved scenarios that the
current narrow SV14/SV15 checks do not establish:

- Last project task Done, then offered `^prj` Done: one picker, one receipt, no extra
  offers. Same-day predecessor reopen must leave the verdict root closed.
- Mixed ordinary pick, Done, and reasoned Cancel, on both a managed Done gesture and the
  parked relation-offer route: only the pick links, roots close once, dependent cascades
  recover only, and the receipt is combined. Reopen takes back only untouched ordinary
  picks. Undo closes keeps the predecessor and ordinary pick state, including when picks
  and closes share a note.
- Card-open task edits and failed ordinary recovery writes cannot authorize a later
  verdict; unaffected rows still apply. Retain the existing stale-walk exclusion,
  old-nav/missing-cancel capability fallback, and checklist-boundary tests.

Reuse `pickSuccessorOfferChoices`, `applySuccessorOfferCloseChoices`, and internal
`applyOfferClosesNow`. Never nest queued `finalizeClosedTasks`, public recovery, or nav
post-cancel APIs inside the mutation queue. Use bounded async tests so a queue deadlock
fails promptly. Fix any demonstrated violations of these existing accepted invariants in
this tale; do not add product scope.

## 5. Verify and deploy the repair

Run focused Rust offer-close/copy tests and the relevant CLI verdict tests, plugin
successor/apply/wiring/nav receipt/review-walk/checklist tests, then the required
repository gates. File-change verification is `just check`, never `just check-full`.
Plugin gates are `npm run build`, `npm test`, and `npm run docs:check`; sync the changed
plugins afterward. Use `/sase_monitor` for long commands and await staging commands
until they exit. Report actual results; do not call a known-red broad gate green. A
failure that reproduces independently on the clean base is infrastructure work, not a
reason to expand this repair or leave completed epic work open.

The lander completed all eight original proposed-follow-up dispositions on hn.6.1:

- hn.6.1.1 #1 unused `TODAY`: removed by `fbb6da46`; no task needed.
- hn.6.1.1 #2 clipboard registration: corroborated `bob-cli-hp`.
- hn.6.1.1 #3, hn.6.1.2 #1, hn.6.1.3 #1 task-merge link: one resolved proposal;
  `3fe50bc7` replaced the obsolete link. No full docs pass was claimed.
- hn.6.1.3 #2 reduced-motion screenshot timeout: corroborated `bob-cli-fz`.
- hn.6.1.4 #2 warm snapshot/ranking budgets: corroborated `bob-cli-f8`, retaining
  secondary timing misses as observations without assuming the same cause.
- hn.6.1.4 #1 stale README hash: independently reproduced; new small ready CI task
  `bob-cli-hs`, related to closed checklist rollout `bob-cli-ho`.

Do not duplicate those tasks. Recheck their status before citing them as current
failures. New independent follow-ups use `/sase_new_task`; epic-caused issues remain
this tale's work. Record final outcomes and verification in the epic close note.

## 6. Finish this landing in the final code turn

Do not wait for this tale's own commit SHA, push, CI result, or another land agent.
After the repair and checks, recheck descendant readiness, linked plan, and drift. If
further work remains, report it accurately instead of forcing a close.

1. Run `sase bead epic-symbols bob-cli-hn.6.1`. Resolve every listed entry by wiring,
   privatizing, a valid non-test pragma, or deletion under the Symvision policy; re-key
   only when a still-open later bead actually needs it. Also check
   `sase bead epic-symbols bob-cli-hn.6` for the upcoming parent-phase close.
2. Close normally with
   `sase bead close bob-cli-hn.6.1 --note "<verified source, fixes, integration, tests, and all follow-up dispositions>"`.
   If rejected for symbols or incomplete descendants, finish that work and retry
   normally. Never use `--force` to make a successful nested landing advance.
3. Run `just symvision` when the recipe is available; if absent, record that fact and
   the clean epic-symbol checks. Open the plans sidecar with `/sase_repo` before writing
   and read its instructions; use the audited artifact read for its body. Set
   `status: done` in the frontmatter of `plan:202611/cycler_offer_verdicts.md` (resolve
   its path with `sase artifact path`, not a numbered workspace path).
4. Read `sase bead read bob-cli-hn.6.1 -r "Need the parent link"` and confirm the
   concrete parent remains phase `bob-cli-hn.6`. Verify that the child and this repair
   fulfill that phase's descriptor/apply/receipt/Undo/reopen/walk scope. Then close only
   that phase with
   `sase bead close bob-cli-hn.6 --note "<verified child scope and repair gates>"`.
   Leave containing epic `bob-cli-hn` and its linked plan to its already-waiting land
   agent. If the parent scope or readiness is ambiguous, leave it open and append the
   concrete blocker there.
5. Finish through `/sase_final`, declaring every changed repository, including the plans
   sidecar. The host commits this turn's final code after declaration; that is
   compatible with completing the epic and its parent phase in this turn.
