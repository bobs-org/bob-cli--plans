---
tier: tale
size: medium
title: Finish successor jump identity and commit parity, then land oe.8 and oe
goal:
  Successor pins preserve task identity, receipt copy, and every commit key across all
  surfaces, and the verified oe.8 and oe plans close normally.
proposed_by: bbugyi200.apollo.bob-cli-oe.8.land
proposed_with: codex/gpt-6-astra
bead: bob-cli-oe.8
create_time: 2026-11-21 08:40:15
status: wip
---

- **PARENT:**
  [202611/successor_jump_repairs.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/successor_jump_repairs.md)
- **BEAD:**
  [bob-cli-oe.8](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-oe/bob-cli-oe.8.md)

# Finish the remaining jump-pin gaps and the landing

Implement these three bounded repairs as one coding agent, then finish the landing. This
tale continues the interrupted landing of `bob-cli-oe.8`; no separate land agent will
resume it. The final step below is part of the implementation, in the same turn as the
final code when verification allows. Do not wait for this tale's own commit, push, or
post-push CI before closing its completed beads.

Read `bob-cli-oe.8` and `bob-cli-oe` with `sase bead read`, then read
`plan:202611/successor_jump_repairs.md` and `plan:202611/successor_jump_pin.md` with
`sase artifact read`. The parent's shared semantics and final DECISIONS remain the
contract: one navigation-only pin, every commit key fires it, every write precedes the
one landing, walk closes park, and one receipt describes the jump. The authorized
walk-park decision is already recorded; the jump glossary decision was **no**. This tale
changes no memory and introduces no new CLI/API contract.

Use `/sase_repo` to open `bob-plugins`, `bob-mac-capture`, and the plans sidecar, and
obey each repository's `AGENTS.md`. Read `docs.md` through `/sase_memory_read` before
any documentation edits. Do not edit generated plugin `main.js` directly or the Rust
package version/CHANGELOG.

## Audit already completed

Read `file:explicit:73581b2443f045fcd5a90e60` with `sase artifact read` for the durable
landing audit, runnable CLI reproduction, observed output, and receipt and Mac source
evidence. The CLI reproduction explicitly distinguishes the installed binary observation
from the checkout source review.

The lander read all four child beads and every note, both plans, and the repairs in
bob-cli `d27041cd`, bob-plugins `9c4b401e`, `767a5970`, `6debb63b`, and Mac `109c30db`.
The checklist regression in epic note #1 is fixed by `767a5970`; the focused checklist
tests now pass. Fifteen focused successor/navigation/cycler/unlink test files and plugin
`build:check` pass. Mac's offline script module passes 43/43, and exact-SHA push CI is
green at <https://github.com/bobs-org/bob-mac-capture/actions/runs/44859450606>. This is
existing-code evidence, not verification of this tale's changes.

Unrelated changes through bob-cli `e11f2a0c` and bob-plugins `6debb63b` were checked:
album `walk_by` (`e11f2a0c`), calendar `navGroups` (`168b665d`), and title-block
rendering (`1a95b292`) neither consume nor duplicate successor jump landing. Mac has no
post-repair drift at `109c30db`. Recheck commits that land after these baselines.

Step-3 follow-up triage is complete and recorded on `bob-cli-oe.8`:

- `oe.8.1` note #1 and `oe.8.2` note #2: duplicate docs budget issue `bob-cli-oc`,
  corroborated with `+1`.
- `oe.8.2` note #1 (also reported by `oe.8.3`): timing nodes `bob-cli-km` and
  `bob-cli-io`, each corroborated with `+1`.
- `oe.8.4` note #1: new small flake task `bob-cli-ok`, covering the three CI-script
  cases sharing the tiny wall-clock harness deadlines. All-status searches, the
  last-week task sweep, and active-epic causal review found no duplicate.
- No proposal was declined. Do not file these again or repair them in this tale.

The original unrelated `bob-cli-oh` panic and `bob-cli-oi` refusal-shake defect also
remain outside scope, as stated in the repair epic. Do not turn baseline gate failures
into successor work.

## 1. Preserve ID-less identity before later items shift the note

Read `src/native/capture/task_complete.rs`, especially `settle_idless_offer_ordinals`,
`idless_same_description_ordinal`, `successor_task_lookup`, and
`successor_task_line_after_write`; also read the per-item planning and net-pass order in
`capture/plan.rs`, and `capture/jump.rs`.

The repair currently settles ordinals only after every item has been planned. It looks
up `row.line` in the final text. Ordinary dependent rows still carry their earlier line,
so a later insertion can put a different equal-description task there. The helper then
certifies that other task's ordinal. This violates the repair plan's explicit
instruction to count identity in the snapshot the offer was planned against.

Reproduction, using the existing `successor_jump.rs` vault helper:

```markdown
# deps.md (three lines; no heading in the actual fixture)

- [ ] #task Above ^above
- [?] #task Beta [dependsOn:: bar]
- [?] #task Beta [dependsOn:: bar]
```

Preview `!sase:bar` with no picks and take the **second Beta's opaque offer key**
(`deps.md@70339031~2` in the existing fixture). Submit `-p none -j KEY` with:

```text
!sase:bar

@deps+above
- extra note
```

The current installed `bob 0.5.0` exits 0 with that same `~2` key, but landing line 3.
Final `deps.md` has Above at 1, its new child at 2, first Beta at 3, second Beta at 4.
Expected landing is **4**. Source inspection at `e11f2a0c` confirms the matching
final-line-based mechanism; reproduce against the checkout-built binary in a test.

Capture the ID-less row's identity in the correct pre-pick/per-item snapshot, then carry
it through staged writes. Reuse existing planner/task tracking where suitable, and the
shared counting helper; do not recompute identity from a stale final line. Earlier twins
that gain block IDs must leave the ID-less population; unrelated twins still count.
Preserve ordinary status-lookup semantics, note-pull/project behavior, and the
duplicate-refusing block-ID resolver. If identity cannot be established, fail closed to
a null landing, never another task's line. Do not mint an ID merely to navigate.

Add the bulk-insertion regression to `tests/cli/capture/successor_jump.rs`, retaining
JS1–JS15. Include an insertion plus an earlier picked twin when supported by the
existing fixture, so both line movement and minted-ID population changes are covered.
Keep unique-key selection separate from display-number renumbering. Add a concise vector
description to `docs/task-dependencies.md` §12.24 within its word budget.

## 2. Preserve one receipt when a non-walk caller defers its landing

In `bob-plugins`, read:

- `task-status-cycler/src/137c-successor-jump.js` and `160-plugin-completion.js`;
- `task-status-cycler/src/135-plugin-successors.js` (`presentSuccessorModel`,
  `landSuccessorJump`) and the parked-successor caller;
- `block-id-prompt/src/124-plugin-pomodoro-inbox-route.js` and
  `130-plugin-task-link-open-and-notices.js`.

`finalizeTaskLinkCloseForJump` now sets `successorJumpTarget: "return"` even for
ordinary Task Link closes, to wait until the link is struck. Unlink routing uses the
same transport to wait for the move. However, `presentSuccessorModel` removes
`noticeModel.jump` on **every** return target. Those non-walk callers later invoke
`landOnTask` directly, which emits no success receipt. The user gets no `↗ “S”` line. A
current-source harness confirmed that the same model carries a jump receipt with `{}`,
loses it with `{successorJumpTarget: "return", successorJumpBox: {}}`, and emits no card
for a normal walk-return context without close verdicts.

Separate deferred target transport from receipt ownership using existing context signals
where sufficient, or one explicit internal context signal if necessary. Suppress the
receipt only when an actual follow/composing owner will emit it. Preserve all ordering
repairs: write close, strike/unstrike or route, then exactly one landing; walk callers
park and use one follow toast; older cyclers retain their supported behavior. Avoid
showing two receipts for verdict-bearing walks or routed unlinks.

Extend caller-level tests in the cycler task-link/wiring suites and the block-id
unlink/parked suites. Cover ordinary Task Link close, ordinary routed and unrouted
parked offers, and walk-owned variants. Assert exactly one jump receipt and one landing
in addition to the existing write-order assertions. The existing synthetic "non-walk
receipt" test with `{}` misses the new caller context; retain it but add the actual
deferred context.

Edit source fragments, rebuild, bump touched plugin manifests, regenerate inventory when
needed, and deploy through `bob plugins sync` following its provenance guard. No forced
sync. Update API docs only if an observable/API description needs it; keep
`plugin-apis.md` within 8000 words.

## 3. Make Mac note-pull-only Shift-Return honor the staged pin

At `109c30db`, `CaptureKeyCommandRouter` maps Shift-Return to `chooseNoPullInOffers`. In
`CapturePanelModel.swift`, that method returns early on
`choice.isOnlyNotePull && choice.closeCount == 0` without considering `jumpKey`. Thus a
pin-only `@sase` card never submits. In `CapturePullInChoiceView.swift`, the footer
applies the same unqualified guard before rendering the skip button and before selecting
`jumpIntentAccessibilityText`, so it displays and speaks the unpinned "Back to draft"
path. This is a source-confirmed gap in the parent commit rule and this repair epic's
pinned footer requirement.

Preserve the old guard for an **unpinned** card, but let a staged valid pin commit with
`.none`, retain its capability and opaque key, and jump on Shift-Return. Give the pinned
footer's button and accessibility text precedence over the unpinned note-pull-only
explanation. Preserve capability gating and the no-pin tests.

Extend `CapturePullInChoiceModelTests` with a note-pull-only fixture advertising jump
support: pin, Shift-Return through the controller/router, assert
`--pull-in none --jump-offer KEY`, zero inferred picks/closes, and one jump URL even
when open-notes was requested. Cover pin plus marks to show Shift-Return skips the
marks, and retain the existing unpinned note-pull test. Add the corresponding footer
copy/accessibility regression. No new protocol fields or grammar versions.

## 4. Verify the final tree and recheck drift

Run targeted regressions and then the required repository gates once the code is ready.
Use `/sase_monitor` for long commands; never end a turn with an unobserved running
process. Resume the same verification run when it escalates.

- bob-cli: `cargo test --test cli successor_jump`, relevant note-pull/capture tests, and
  **`just check`** (via `sase tool run check` when available). Do not run
  `just check-full`. A check-full failure after a passing check is infrastructure, not
  remaining epic work. Run `just docs-check` for the vector-doc edit.
- bob-plugins: the changed focused suites plus checklist/review/landing coverage,
  `npm run build:check`, `npm test`, `npm run docs:check`, and required manifest
  validation/inventory checks. Known unrelated `km`/`io` timing failures and `oc`
  docs-budget failure remain separately tracked; verify their provenance if seen.
- Mac: obey its compile requirement. Run `just portable-check` (on athena through the
  documented tailnet workflow if this host lacks Swift), best-effort scratch Mac
  `bundle.sh` and `just typecheck-tests`, and the repository's `Scripts/macos-ci-check`
  precommit gate. If the Mac is offline, state the missing local macOS proof honestly;
  the exact-tree macOS CI gate still must pass. Do not skip Swift compilation. After any
  push, watch that exact-SHA run through `/sase_monitor`, but never order this tale's
  closeout after its own final push.

Re-read the current epic notes and inspect newly landed commits in all three opened
repositories. Finish any interaction caused by these repairs before closing. Record new
unrelated discoveries through `/sase_new_task`, without duplicating the completed triage
above.

## 5. Complete this tale and land both parent plans normally

This step is mandatory; there is no waiting land agent for this tale. Code and
verification can be completed and beads closed in the same provider turn before
`/sase_final` commits that code. No dependency on this tale's final SHA or push.

1. Read the coder's assigned bead and descendants. If the host created a separate tale
   bead below `bob-cli-oe.8`, close that tale normally once its implementation and gates
   are complete, so it cannot block descendant readiness. If the coder remains assigned
   to `bob-cli-oe.8`, leave it open until step 3. Mark this tale's linked plan done
   through the standard plan workflow when its work is complete. Do not force-close a
   successful nested landing.
2. Run `sase bead epic-symbols bob-cli-oe.8`. For **every** listed entry, wire up,
   privatize, add an appropriate non-test pragma, or delete it under the Symvision
   policy; re-key only if a still-open later bead actually needs it. The audit found
   none, but rerun this check. Read all four phase notes and the tale notes, and confirm
   all descendants and the linked plan's acceptance are ready.
3. Close normally:
   `sase bead close bob-cli-oe.8 --note "<verified fixes, tests, integration drift, and all follow-up dispositions>"`.
   Include the `oc`, `km`, `io`, `ok` outcomes above, and resolution of epic note #1. If
   stale symbols reject the close, clean them and retry. If a named phase is incomplete,
   finish or reopen it; use no force for a successful landing. Only a genuinely
   canceled/superseded outcome may use the explicit force/reason/resolution procedure
   from the user prompt.
4. Run `just symvision` if the recipe exists (currently it does not), and set
   `status: done` in the frontmatter of `plan:202611/successor_jump_repairs.md`
   (`202611/successor_jump_repairs.md` in the opened plans repo). Resolve/read it
   through the audited artifact tools.
5. Read `bob-cli-oe.8` again for its parent link, concretely **`bob-cli-oe`**, a plan
   bead. Review that parent's previous LAND TRIAGE and LAND VERIFICATION notes, every
   descendant and note (including phases `bob-cli-oe.1` through `.7`, repair epic `.8`
   and its descendants), `plan:202611/successor_jump_pin.md`, original implementation
   commits, and post-child drift. Rerun descendant and linked-plan readiness checks; do
   not infer parent completion solely from this child closing.
6. If `bob-cli-oe` is fully complete, run `sase bead epic-symbols bob-cli-oe`, retire or
   properly re-key every entry, then
   `sase bead close bob-cli-oe --note "<what was rechecked against the parent contract, descendants, notes, decisions, and integration drift>"`.
   Run `just symvision` when available and set `status: done` in
   `plan:202611/successor_jump_pin.md` (`202611/successor_jump_pin.md` in plans).
   Inspect its parent link and repeat only through directly parented complete plan
   ancestors. If a parent is a phase, close only that phase normally after verifying its
   scope; leave its containing epic to the waiting land agent. At the first incomplete
   or ambiguous parent, record the concrete blocker there and report it.
7. Finish with `/sase_final`, declaring every changed repository, including plans. The
   final report must state which beads actually closed, verification results, and any
   parent blocker or missing local Mac proof.
