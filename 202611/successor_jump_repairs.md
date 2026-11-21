---
tier: epic
title: Jump pin landing repairs
goal: 'Every Ctrl+J jump pin lands exactly once, on the right task, after every write
  of the gesture has landed, on all three surfaces. Bob never lands on an unrelated
  twin, never reports a findable task as lost, never lands before a Task Link strike
  or an inbox-route move, and never drops the pin on the Mac inbox-route path. A pinned
  close from a `]s` landing still records its review answer. The card capsule looks
  the way the design specifies.

  '
phases:
- id: cli-jump-identity
  title: bob capture jump landing finds the right ID-less task
  depends_on: []
  size: small
  description: 'cli-jump-identity: make the top-level jump landing settle a correct
    identity for every ID-less offer row, failing closed to a null landing; use the
    duplicate-refusing block-ID lookup; add the missing JS vectors; and complete §12.24
    with the shared semantics, commit table, and precedence.'
- id: nav-jump-repairs
  title: Obsidian card capsule, walk follow bookkeeping, and receipt copy
  depends_on: []
  size: medium
  description: 'nav-jump-repairs: in bob-navigation-hotkeys, fix the capsule icon
    and its trailing-edge placement, make the reviewWalk follow path record the review
    answer and park only when v1 would advance, show the receipt line once, make landOnTask
    settle before reporting, leave one authoritative applySelection/settle, and fix
    plugin-apis.md''s §12.24 citation and word budget.'
- id: cycler-jump-ordering
  title: Land the jump after every write in task-status-cycler and block-id-prompt
  depends_on:
  - nav-jump-repairs
  size: medium
  description: 'cycler-jump-ordering: make the Task Link Ctrl+Enter close and the
    unlink-plus-inbox-route path land the jump only after their strike or move writes
    and walk settle, stop the walk-return receipt doubling the ↗ line, and build the
    Left identity from the fresh plan row.'
- id: mac-jump-repairs
  title: Mac Pull-In card keeps the pin through inbox routing
  depends_on: []
  size: medium
  description: 'mac-jump-repairs: in bob-mac-capture, carry the jump offer and its
    capability through the inbox-route and other re-dispatch contexts, fix the primary
    label count for hidden highlights, animate the capsule per the design, align the
    pinned footer''s VoiceOver text, treat a missing jump object as lost, and restore
    docs facts the epic trimmed.'
proposed_by: bbugyi200.apollo.bob-cli-oe.land
proposed_with: claude/opus
parent_bead: bob-cli-oe
create_time: 2026-11-20 21:39:26
status: wip
bead_id: bob-cli-oe.8
---

- **PROMPT:** [prompts/202611/successor_jump_repairs.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/successor_jump_repairs.md)
- **PARENT:** [202611/successor_jump_pin.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/successor_jump_pin.md)
- **BEAD:** [bob-cli-oe.8](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-oe/bob-cli-oe.8.md)

# Plan: Jump pin landing repairs

This child epic finishes the parent epic's (bob-cli-oe, "Ctrl+J jump pin on the
successor card") landing. The parent's plan, `plan:202611/successor_jump_pin.md`, stays
the contract. Its "Shared semantics", "Look", and per-surface sections define every
behavior named below. Read it first (`sase bead read bob-cli-oe` shows the path). The
parent's DECISIONS are final. No memory note changes in this epic, and the parent's
`decisions:successor-jump-parks-the-walk` record already describes the corrected
behavior.

A landing review of all seven parent phases found the bugs below. Each item names the
defect, gives evidence, and says what "fixed" means. Where a fix needs a design choice,
the item says which one to make.

Every phase follows its repo's `AGENTS.md`. In bob-plugins:

- edit `src/` fragments and keep each at or below 1000 lines;
- run `npm run build`, then `npm test`, then `npm run docs:check`;
- bump the touched plugins' manifest versions;
- run `bob plugins sync`.

The two known pre-existing gate failures are not this epic's work: the
`docs/reference/vault-format.md` budget (bob-cli-oc) and the ledger-tools 25 ms perf
flakes (bob-cli-km, bob-cli-io). The broken refusal shake is tracked separately as
bob-cli-oi, and the `-X` debug panic on two ID-less dependents as bob-cli-oh. Do not fix
them here.

## cli-jump-identity: bob capture jump landing finds the right ID-less task

This phase runs in bob-cli. Read `src/native/capture/jump.rs` and the lookup helpers in
`src/native/capture/task_complete.rs`: `successor_task_line_after_write`,
`successor_idless_task_lookup`, and `offer_key_duplicate_ordinal`.

**Bug: wrong task or false "lost" for ID-less offers (shared semantics #8).**

`build_landing` resolves every ID-less row by cleaned description plus
`offer.settled_ordinal`, falling back to the ordinal inside the offer key. Only
note-pull and project rows get a settled ordinal; see `note_pull.rs` (around 915-960)
and `project_note.rs` `settled_ordinal_for`. Ordinary unblocked dependents and
keep-moving rows therefore fall back to the key ordinal. That ordinal counts only the
_offered_ ID-less rows _before_ picks mint IDs, while the lookup counts _all_ ID-less
same-description tasks in the final text.

These cases reproduce in a scratch vault on master 8f703ccb:

- An unrelated `- [ ] #task Beta` sits above `- [?] #task Beta [dependsOn:: bar]`.
  Running `-p none -j 1 -- '!sase:bar'` returns landing line 1, the unrelated task,
  instead of line 2.
- Three ID-less `Beta` dependents with `-p 1 -j 2` land on the third Beta instead of the
  second.
- Two ID-less `Beta` dependents with `-p 1 -j 2` return `landing: null` and print the
  "not found after writes" receipt, although the task is at line 2.

**Fix:**

- Settle a correct final-text identity for every ID-less offer row the jump can target,
  not only note-pull and project rows:
  - Count the row's ordinal among ID-less same-description tasks in the text the offer
    was planned against. `row.line` is valid there.
  - Then carry that ordinal into the final staged text. Earlier twins that minted a
    block ID in this batch leave the ID-less set. Unrelated twins that are not offers
    still count on both sides.
  - Follow the existing `settled_ordinal` precedent in `note_pull.rs` and
    `project_note.rs`. Do not fork the rule: put the shared counting in one helper that
    the status lookup can keep using.
- When the identity is still ambiguous, return `landing: null`, which gives the lost
  receipt. Ambiguous means the counted ordinal is out of range, or the row's own line no
  longer matches. Never return another task's line.
- The status lookup (`successor_task_lookup`) must keep its current results for ordinary
  dependents. It may keep using `row.line` there.
- Use the duplicate-refusing block-ID resolver for the block-ID step
  (`line_for_block_id` or its equivalent). A duplicated `^id` must not land on the first
  copy.

**Vectors.** Add these to `tests/cli/capture/successor_jump.rs` and list them under
§12.24:

| #    | Case                                                                                         | Expect                                                     |
| ---- | -------------------------------------------------------------------------------------------- | ---------------------------------------------------------- |
| JS11 | unrelated ID-less twin above the offered dependent, `-p none -j 1`                           | landing is the dependent's line, not the twin's            |
| JS12 | three ID-less same-description dependents, `-p 1 -j 2`                                       | landing is the second dependent's post-write line          |
| JS13 | two ID-less same-description dependents, `-p 1 -j 2`                                         | landing found (line 2), not null                           |
| JS14 | a target whose identity cannot be settled (unit-test the resolver if no CLI case reaches it) | `landing: null`, exit 0, writes kept, lost receipt printed |
| JS15 | `-j 9` while offers exist                                                                    | `offer_jump_stale`, exit 2, vault byte-identical           |

Also tighten two existing vectors:

- JS1 must assert that the landing line holds the recovered post-write text: the status
  flipped from `?`, plus the `[unblocked::]` stamp if the fixture's settings write one.
- JS9 must actually move the row, or add a duplicate, so the description fallback is
  exercised.

**Docs.** In `docs/task-dependencies.md` §12.24:

- Add a compact summary of the parent plan's shared semantics:
  - one pin per card, keyed by offer key, with Ctrl+J toggle/move/clear;
  - pinnable rows only;
  - lifetime and fail-closed refresh;
  - a pin alone never arms the two-step Esc.
- Add the commit table and the precedence rules: the pin replaces Mac Capture's
  open-notes landing, parks a `]s` walk, and replaces project promotion's `^prj`
  landing.
- Fix the JS9 row text so it no longer claims keep-moving rows carry a settled ordinal,
  unless the fix gives them one.
- Fix the `-j` help line's alignment and order in §12.14's help block.
- Keep `just docs-check` budget-clean for every page you touch. Read the `docs.md`
  memory first (`sase memory read docs.md`).

Do not edit `Cargo.toml`'s version or `CHANGELOG.md`. Use a `fix(capture):` subject.

**Verification:**

- `cargo test --test cli successor_jump`;
- the capture, `help_options`, `flag_guard`, and `note_pull` CLI suites;
- `cargo test --lib -- capture`;
- clippy as CI runs it;
- `just check`;
- `just docs-check`, where only the known vault-format budget failure may remain.

## nav-jump-repairs: Obsidian card capsule, walk follow bookkeeping, and receipt copy

This phase runs in bob-plugins, plugin `bob-navigation-hotkeys`.

1. **Capsule icon.** `206e-successor-picker-jump.js` calls
   `applyIcon(capsule, "arrow-up-forward")`. That is the Mac SF Symbol name, not a
   Lucide icon. It also sets the icon on the span holding the "Jump" text, and
   Obsidian's `setIcon` replaces the element's children.
   - Render a dedicated icon span with Lucide `arrow-up-right`, followed by a text span
     "Jump", inside the capsule.
   - Test that both survive in the rendered DOM.
2. **Capsule placement.** The capsule is created by `successorRenderJumpRowAdornments`
   before the row's `bob-successor-row-content` div, and that div has `flex: 1 1 auto`.
   As a result, `margin-left: auto` cannot push the pill to the trailing edge, and it
   renders between the mark and the title.
   - Make the capsule sit at the row's trailing edge, aligned with the title line, as
     the parent's "Look" section requires: create it after the content div, or give it a
     flex `order`.
   - Keep the hover actions where they are today.
   - Pin the DOM order or class contract in a test.
3. **Follow path bookkeeping.** `followReviewWalkAfter` (536b) parks the walk but never
   calls `addReviewAnsweredKeys` / `recordReviewAnswer`, which the v1 continue path does
   (536, around 506-513). A pinned Ctrl+Enter close from a `]s` landing is therefore
   missing from the review journal and stats. The follow path also parks without
   checking that the outcome resolves the row, so a relation-complete close of a
   _different_ task parks the walk past an unanswered row.
   - On a fresh origin, make exactly the resolve decision v1 makes. Reuse the v1
     helpers; do not copy them. Cover both the `reviewOutcomeResolves` result and v1's
     PRE/POST complete rule.
   - Only when v1 would advance: record the answered keys and the journal entry exactly
     as v1 does, then park and land.
   - When v1 would not advance: do not park or record anything, but still land on
     `follow` (explicit intent) and show the one composed toast.
   - Stale and off-landing origins keep today's behavior.
   - Extend `test-navigation-review-advance-gestures.cjs` to cover all three cases.
4. **Receipt line rendered once.**
   - Notice v5 (725) currently renders a `↗` glyph span _and_ the text `↗ “S”`, which
     shows a doubled arrow. Render one arrow.
   - A jump-only notice builds a fallback model whose glyph, title, and receipt all
     repeat the receipt, so it shows about four copies. Make a jump-only notice show the
     `↗ “S”` line once, with a sensible title.
   - Update `test-navigation-hotkeys-unblocked-notice.cjs` to match.
5. **`landOnTask` reports only settled outcomes.**
   - When no Vim jump context is available, `focusTaskAnchor` returns the immediate
     placement result and keeps retrying on later frames (650, around 563-569 and
     781-826). `landOnTask` can then show the "Couldn't find" notice and still move the
     cursor afterwards. Make `landOnTask` await the final placement, or cancel the
     retries before it reports lost.
   - Also move the 1-based `line - 1` guess (676a, around 63-69) after the block-ID
     step, so an identical line above the target can never beat the block ID.
   - Extend `test-navigation-task-landing.cjs`.
6. **One authoritative `applySelection` / `settle`.** 206e replaces both methods
   wholesale with copies, so 206's versions (around 728-814) are dead code, and a later
   fix there would silently do nothing.
   - Leave exactly one implementation of each: one-line hooks in 206 calling 206e
     helpers, or the jump-aware bodies moved into 206 (it is at 815 lines). Delete the
     other copy.
   - In the same code, make a pin dropped at commit time fail closed. It must not fall
     through to the highlight fallback and pull in the highlighted row.
   - Also fix 206f's header comment, which still says v4.
7. **Docs.** bob-plugins `docs/develop/plugin-apis.md`:
   - It cites `docs/task-dependencies.md` §12.22 for the successors v5 jump pin. The
     contract is §12.24, so fix that citation.
   - The parent epic grew this page from 7,832 to 8,044 lint words, over the 8,000-word
     `develop/` cap. Trim it back to 8,000 or fewer without dropping contract facts.
   - `docs/bob-navigation-hotkeys.md` has about 3 words of headroom under its cap. Avoid
     edits there, or trim to pay for them (bob-cli-ob tracks the headroom itself).

**Verification:**

- `npm run build:check`;
- the touched test files plus the successor picker/verdicts, task-landing,
  review-advance(-gestures), unblocked-notice, and checklist-ctrl-enter suites;
- `npm test`, which may show only the known perf flakes;
- `npm run docs:check`, where plugin-apis.md must no longer warn;
- the manifest bump and `bob plugins sync`.

## cycler-jump-ordering: Land the jump after every write in task-status-cycler and block-id-prompt

This phase runs in bob-plugins, in plugins `task-status-cycler` and `block-id-prompt`.
It depends on nav-jump-repairs, which changes the walk follow toast this phase composes
with. Shared semantics #8 says writes land first, and #7 says exactly one landing per
gesture.

1. **Task Link Ctrl+Enter close lands before the strike.**
   - `handleActiveTaskBlockLinkOpenDone` (`160-plugin-completion.js`, around 581-745)
     awaits `finalizeClosedTasks`, which now self-lands. Only after that does it call
     `strikeActiveSelectedTaskBlockLink(editor, …)`.
   - When the pinned successor lives in another note, the landing switches that tab. The
     strike then reads the wrong document and silently does nothing, leaving the closed
     task's Task Link unstruck.
   - Fix: when the caller did not already ask for `successorJumpTarget: "return"`, run
     finalize in `"return"` mode with a local `successorJumpBox`. Then land the boxed
     jump through `landSuccessorJump` with the caller's non-return context, after the
     strike (and the unstrike/reopen branches).
   - Walk callers that already pass `"return"` stay as they are: nav lands via `follow`
     after this method returns.
   - Cover the pomodoro-transclusion branch too.
   - Test: a pinned close from a Task Link strikes the link, then lands, in that order.
2. **Unlink plus inbox route lands before the move.**
   - `finishPomodoroUnlinkWithInboxRoute` (block-id-prompt
     `124-plugin-pomodoro-inbox-route.js`, around 133-200) awaits the parked successor
     offer before `commitPomodoroToggleInboxRoute`.
   - A landing there can switch the tab, so the move fails closed ("Not moved: Routed
     tasks changed").
   - On a walk origin, the later `route` settle then also advances the walk: two
     landings.
   - Fix: ask the parked offer to return the jump instead of landing. That means
     `offerParkedSuccessorsAfterUnlink` passes `successorJumpTarget: "return"` and a box
     through to cycler `offerParkedSuccessors`; extend that context plumbing in 139 if
     needed. Then run the route move.
   - After the move:
     - On a walk origin, settle the walk with `follow: box.jump`, which parks and lands
       once.
     - Otherwise, show the notice, then land once.
     - The no-route and no-write early returns land after their notice.
   - Feature-detect: an older cycler without the return plumbing keeps today's order.
   - Test both orders, with and without a walk origin.
3. **Parked offers and the work-summary prompt.** Pending (`/`) unlinks reach the parked
   offer while the work-summary prompt (block-id-prompt `090-prompt-modals.js`, around
   578-583) is still closing. Make the landing run after that prompt has closed, so a
   modal focus restore cannot undo it. Pin the order in
   `test-block-id-prompt-parked-successors` (or its equivalent), which the parent epic
   never updated.
4. **No doubled receipt on walk returns.** With `successorJumpTarget === "return"`, the
   cycler's own receipt card must not add the `↗ “S”` line, because nav's follow toast
   adds it. Non-walk receipts keep it.
5. **Fresh Left identity.**
   - `137c-successor-jump.js` (around 54) looks up the plan edit for a left (unpicked,
     unclosed) offer at the line from when the card opened. After a replan, those edits
     come from fresh text, so a shifted note can give another task's line as `text`, and
     `landOnTask` would match it first.
   - Build the Left identity from the fresh plan's row for the same offer key, falling
     back to the original offer only when the fresh plan lacks the key.
   - Add tests for this case, for the refused-stale-pick-becomes-Left case, and for the
     cancelled outcome.

Update `docs/task-status-cycler.md` and bob-plugins `docs/develop/plugin-apis.md` only
where behavior or API changed, and keep plugin-apis.md at or below its cap.

**Verification:**

- `npm run build:check`;
- the cycler successor-wiring/apply/verdicts/parked-successors/ctrl-enter suites;
- the block-id-prompt inbox-route and parked-successor suites;
- `npm test`, which may show only the known perf flakes;
- `npm run docs:check`;
- the manifest bumps for both plugins and `bob plugins sync`.

## mac-jump-repairs: Mac Pull-In card keeps the pin through inbox routing

This phase runs in bob-mac-capture. Read its `AGENTS.md`: never land Swift you have not
compiled, and the macOS CI gate applies.

1. **Pin dropped on the inbox-route re-dispatch (the JS5 case).**
   - A pinned commit first sends `--jump-offer` (`CapturePanelModel.swift`, around
     4235). When bob answers `inbox_route_choice_required`, `submitInboxRouteAnswers`
     (around 5086-5105) re-dispatches without `jumpOffer` or its capability.
     `CaptureInboxRouteSubmitContext` (around 560-578) has no field for either.
   - The real write therefore goes out unpinned. The Mac falls back to opening notes
     with no lost line.
   - Fix: add `jumpOffer` and `jumpCapabilityAdvertised` to that context, next to
     `offerVerdicts`, and pass them on the re-dispatch.
   - Audit every other re-dispatch path for the same drop: the move-in choice context,
     refusal reopens, and stale refreshes.
   - Test that a forward submit through the route choice sends `--jump-offer KEY` and
     opens the one-entry jump URL.
2. **Primary label count.**
   - `primaryCommitLabel` now reads the core outcomes' label. That count includes a
     highlighted row hidden in a collapsed group, so collapsing it with nothing staged
     shows "Pull In 1" on a disabled button. Before the change it showed "Pull In".
   - Count only visible rows, as the card's own count does, and keep the jump labels
     from the copy table.
   - Add a test.
3. **Capsule motion.** Nothing animates the pin change today, so the capsule pops.
   - Animate `jumpKey` changes so `matchedGeometryEffect(id: "jump-pin")` glides between
     rows.
   - Under Reduce Motion, use a cross-fade instead of the geometry effect.
4. **Pinned footer accessibility and icons.**
   - While a pin is staged, the footer's VoiceOver text and the buttons' trailing glyph
     must describe the jump, not opening notes.
   - Hide or disable the "Jump Here After Capture" accessibility action when bob does
     not advertise `pull_in.jump.supported`.
5. **Missing `jump` object.** If a pinned commit succeeds but the decoded result has no
   top-level `jump`, treat it as lost: fall back to today's open-notes behavior and add
   the notification's lost line.
6. **Docs facts lost in the parent commit.** bf590f32 trimmed these from
   `docs/capture-previews.md` and `docs/protocol.md`:
   - the Control-X / Control-Shift-X / Shift-Return / two-Esc lines;
   - "rows show `N× · age`, nothing is preselected";
   - "blur never commits a click";
   - Grammar 12's "no synthesized top-level close".

   Restore each fact that no other Mac doc still states, within the docs budget.

7. **Test nits.**
   - Rename `testJumpWithoutCapabilityRefusesNothingAndMarksNothing` to say what it
     asserts.
   - Fix the `pullInSelection: .none` optional warning in `CapturePullInClientTests`.

**Verification:**

- `just portable-check`;
- `just docs-check`, where only the known bob-cli vault-format budget failure may
  remain;
- a best-effort mac compile (`bundle.sh` plus `just typecheck-tests`), saying so if the
  mac is offline;
- `scripts/dev/macos-ci-check` before landing;
- after the push, watch the exact-SHA macOS run to green with `/sase_monitor`.

## Out of scope

- The parent epic's own closeout. Its land agent resumes once this child epic lands.
- bob-cli-oh, bob-cli-oi, bob-cli-oc, bob-cli-ob, bob-cli-km, and bob-cli-io.
- Muted-row pins and any vault-persisted jump state, as in the parent.
