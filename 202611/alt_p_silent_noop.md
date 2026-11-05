---
tier: tale
title: Make Alt+P never fail silently
goal:
  Alt+P on a Task Card row always writes, shows a visible card, or shows a notice; never
  nothing.
size: medium
decisions:
  busy_notice:
    ask:
      Should a Task Card key swallowed by the review-walk settle lock show a short
      notice?
    choices:
      notice:
        Key is still swallowed, but a toast says "press Alt+P again"; double presses
        show it too
      silent:
        Stays silent; a fast press right after a landing still looks like a broken key
    default: notice
    why: A silent swallow looks exactly like a broken keymap, which is this bug report
    answer: notice
  e2e:
    ask: Should the fix ship with a committed real-Obsidian regression e2e for Alt+P?
    choices:
      repo_e2e:
        Commit a real-Obsidian e2e plus offline self-test in npm test, like the
        decay-ladder e2e
      one_off: Verify once in a throwaway run; commit only unit tests
    default: repo_e2e
    why:
      Unit stubs never evaluate CSS, which is how the invisible card shipped with green
      tests
    answer: repo_e2e
proposed_by: bbugyi200.athena.18h
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-05 07:11:10
status: wip
---

# Fix: Alt+P silently does nothing (invisible quick-roll card + silent walk-lock swallow)

All code changes are in the **bob-plugins** linked repo (open it with `/sase_repo`),
plugin **bob-navigation-hotkeys**. Read that repo's `AGENTS.md` first: edit `src/`
fragments only, run `npm run build`, never hand-edit `main.js`, keep every hand-edited
fragment at or below 1000 lines, and run `bob plugins sync` when done.

## What Bryan saw

On 2026-11-05 at 06:44 on the Mac, `]s` landed on the NEW row `sase.md` line 34
`- [ ] #task The \`sase final\` command … [created::2026-11-04]
^final-for-bare-agents`. The row was Ready, had no priority, and the review footer read `NEW
1/6 · Alt+P set
P1`. He pressed Alt+P. No card, no notice, and no write followed. Vault history shows the line unchanged until a later P1 write at 06:47:56, which the review journal records as `defer`. In between, he filed "Fix `<alt+p>`
Obsidian keymap again!".

## Diagnosis

### Ruled out (with evidence)

- **Planner / write logic.** With the unit harness
  (`scripts/navigation-hotkeys-harness.cjs`), Alt+P on the exact `sase.md` snapshot with
  the real `~/.config/bob/config.yml` writes `[priority:: high]` plus a 3–8 day
  `scheduled` date, a `P0 → P1` Schedule Log, and the P1 notice.
- **macOS key shape.** Obsidian 1.15's keymap matches a hotkey on `vkey` (from
  `keyCode`, 80 → `"P"`) _or_ on `event.key`. So Option+P (`key: "π"`, `code: "KeyP"`)
  still runs `bob-navigation-hotkeys:roll-by-recommendation`. A real-Obsidian run with a
  Mac-shaped CDP key event confirmed this and wrote P1.
- **Hotkey conflict.** The vault's `.obsidian/hotkeys.json` binds nothing else to Alt+P.
  The old Templater `schedule.md` binding was removed on Oct 23.
- **`<A-1>-` at the left of the status bar.** That is obsidian-vimrc-support's chord
  display (`displayChord: true`): a stale, unmatched Vim key. It is not the screenshot
  hotkey (Hammerspoon's Ctrl+Alt+Shift+S) and is unrelated to Alt+P.
- **Successor-picker walk hold.** Today's daily note has no Task Link to either the
  dropped row or the target row.
- **Real Obsidian, realistic timing.** In an isolated real Obsidian 1.15 (xvfb + CDP,
  real config, vimMode, Tasks/dataview/vimrc plugins), Alt+P on this row wrote P1 in
  three cases: plainly, after a `]s` landing, and 3 s after cancelling the previous NEW
  row.

### Root cause: two code paths make Alt+P fail with zero feedback

Both were reproduced in that real Obsidian run.

1. **The "revealed" quick-roll card stays invisible (primary defect, since `92d5809`,
   Oct 23).**
   - Alt+P opens the Task Card hidden with class `bob-task-card-quick-pending` and
     applies the recommendation. Any path that cannot finish in one shot calls
     `revealQuickRollCard()` to show the card. Those paths include:
     - a write that returns `false` and leaves the picker open;
     - a model that is not ready, or `opening` / `taskCardDispatching` / `linkResolving`
       set, when `dispatchInitialTaskCardIntent` runs;
     - the Next/Pending Work-summary stage;
     - the decay-cancel banner;
     - an unavailable recommendation;
     - an inbox route back-out;
     - a throwing dispatch.
   - The class is added to **two** elements: the constructor's
     `setQuickRollPendingClass(this, true)` (`src/340-picker-core.js:55-60`) tags
     `containerEl`, and `onOpen` (`src/350-picker-task-card.js:724-728`) also tags
     `modalEl`.
   - `setQuickRollPendingClass` (`src/457-picker-quick-roll.js:14-29`) only touches
     `picker.containerEl || picker.modalEl`, which is the container. So the reveal never
     clears `modalEl`, and `styles.css`
     (`.modal.bob-cnp-modal.bob-task-card-quick-pending { visibility: hidden }`) keeps
     the card invisible.
   - Real-Obsidian evidence, on a Pending row: stage `schedule-work-log`,
     `quickRollCardPending` false, `containerEl` = `modal-container mod-dim` (visible),
     `modalEl` = `modal bob-task-card-modal bob-task-card-quick-pending bob-cnp-modal`
     with computed `visibility: hidden`, focus on `body`, and the picker still open
     after 2 s with no notice and no write.
   - The invisible modal also holds the keyboard until Esc. While it is registered,
     every later Alt+P from the editor is swallowed by `rollByRecommendation`'s "live
     picker" early return (`src/681-plugin-quick-roll.js:16-18`).
   - Every "shows the card" row of the documented Alt+P table (`docs/task-card.md`,
     _One-key roll_) is broken.
   - The unit tests missed it because they only assert on `containerEl.classes`:
     `scripts/test-navigation-quick-roll.cjs` "a throwing one-shot dispatch always
     reveals…" and "Alt+P hides immediate rolls and reveals the optional Next Work
     summary".
2. **Silent review-walk lock swallow.**
   - `openBulletPropertyPicker` (`src/550-plugin-cancel-and-counted-property.js`, the
     `captured.busy === true` branch near line 548) returns `true` with no notice while
     `reviewWalkBusy()` holds. That covers the 3000 ms in-flight lock and the 350 ms
     settle window that starts right after the landing toast, which itself advertises
     `Alt+P set P1` (`src/536-plugin-review-advance.js:707-716`).
   - Real-Obsidian evidence: Alt+P about 150 ms after the previous answer gives
     `captureReviewGesture → {busy:true}`, no picker, no notice, and the line unchanged.
   - The swallow itself is accepted design (decision `answering-advances-the-walk`:
     "keys are swallowed for about 350 ms after each advance"). Only its silence is the
     defect.

Nothing in the vault records which of the two fired at 06:44: there is no keystroke
telemetry, and both give the identical symptom. After this fix, **every Alt+P exit is
visible**: a write plus its notice, a visible card, or a notice. If the bug recurs, the
user will see which branch fired.

## Changes (bob-plugins, bob-navigation-hotkeys)

1. **One owner for the hidden state.**
   - Make `setQuickRollPendingClass` in `src/457-picker-quick-roll.js` add or remove
     `bob-task-card-quick-pending` on **both** `containerEl` and `modalEl`, each when
     present, using `addClass/removeClass` with a `classList` fallback.
   - Replace the ad-hoc `this.modalEl.addClass("bob-task-card-quick-pending")` in
     `onOpen` (`src/350-picker-task-card.js:726-728`) with a call to the helper. That
     file is at 998/1000 lines, so the edit must not add net lines.
   - After the change, no code outside the helper may touch that class name. Grep
     `plugins/bob-navigation-hotkeys/src` to confirm.
   - `revealQuickRollCard()` then clears both elements through the helper.
2. **Busy feedback.**

   > [!decision] busy_notice = notice
   >
   > - In the busy branch of `openBulletPropertyPicker`, still return `true` (keep the
   >   swallow, keep the double-press guard) but first show one short Notice:
   >   `Review walk still settling — press Alt+P again` when
   >   `options.initialTaskCardIntent === "apply-recommendation"`, else
   >   `Review walk still settling — press again`.
   > - `src/550-…` is at 999/1000 lines, so put the notice in a small method on the
   >   quick-roll mixin (`src/681-plugin-quick-roll.js`, e.g.
   >   `noticeReviewWalkBusy(options)` returning `true`) and make the branch a one-line
   >   `return this.noticeReviewWalkBusy(options);`.
   > - Do not change lock durations or the other busy sites (task split, task move, lane
   >   toggle).

   > [!decision] busy_notice = silent Skip this step; leave the busy branch
   > byte-identical.

3. **Unit tests** in `scripts/test-navigation-quick-roll.cjs`. The harness `ModalStub`
   in `scripts/modal-harness.cjs` already has separate `containerEl` and `modalEl`.
   - Every existing reveal assertion that checks `containerEl.classes` must also assert
     that `modalEl.classes` lacks `bob-task-card-quick-pending` after the reveal. That
     covers the throwing dispatch, the Next Work summary, the cancel banner, the
     unavailable P1 level, unavailable recurring, and the inbox route back-out.
   - The "hides immediate rolls" test must assert that both elements carry the class
     while the write is pending.
   - Add a focused test: open via `rollByRecommendation`, force a write that resolves
     `false` with the picker left open, then assert that the card is revealed on both
     elements and the picker is still open.
   - If `busy_notice = notice`: add a test where `captureReviewGesture` returns
     `{ busy: true }`. Alt+P then creates no picker, shows exactly one notice containing
     `press Alt+P again`, and writes nothing. Ctrl+Shift+P (`openBulletPropertyPicker`
     without the intent) shows the generic text.
   - Add a CSS guard next to "quick-roll visibility CSS disables the modal entrance
     animation". For each class that `styles.css` uses to hide the quick card, assert
     that the reveal helper removes it from the element that selector targets.
4. **Real-Obsidian regression e2e.**

   > [!decision] e2e = repo_e2e
   >
   > - Add `scripts/obsidian-quick-roll-e2e.cjs` plus `-lib.cjs`, modeled on
   >   `scripts/obsidian-decay-ladder-e2e.cjs`: isolated scratch vault and profile,
   >   owned Obsidian process, CDP, and never the live vault.
   > - Add the offline `--self-test` wrapper `scripts/test-obsidian-quick-roll-e2e.cjs`
   >   to the `npm test` list, mirroring `scripts/test-obsidian-decay-ladder-e2e.cjs`.
   > - Drive real `Input.dispatchKeyEvent` Alt+P events in the Mac shape (`key "π"`,
   >   `code "KeyP"`, `windowsVirtualKeyCode 80`, Alt modifier with a real Alt keyup).
   > - Scenario a: a Ready task with no priority gets `[priority:: high]` and a
   >   P1-window `scheduled`, with no modal left open.
   > - Scenario b: a Pending `[/]` task shows the Work-summary stage with **computed**
   >   `visibility: visible` on `modalEl`. A second Alt+P then submits and writes.
   > - Scenario c (only if `busy_notice = notice`): Alt+P inside the settle window right
   >   after a `]s` answer shows the settling notice and writes nothing. Alt+P again
   >   after the window writes P1.
   > - Run it under `xvfb-run -a` on a host with Obsidian, and record the command and
   >   result in the commit message body.

   > [!decision] e2e = one_off Run the same three scenarios once in a throwaway script
   > outside the repo, and record the observations in the commit message. Commit only
   > the unit tests.

5. **Docs, version, deploy.**
   - Bump `plugins/bob-navigation-hotkeys/manifest.json` with a patch bump (2.73.3 →
     2.73.4). Use a minor bump instead if `busy_notice = notice`, since that adds
     user-visible behavior.
   - In `docs/task-card.md` _One-key roll_, add the row "Pressed while the review walk
     is still settling after an answer → shows _Review walk still settling — press Alt+P
     again_, writes nothing" (notice branch only), and keep the other rows (now true
     again).
   - Run `npm run build`, `npm run docs:gen`, `npm test`, `npm run validate`, and
     `npm run docs:check`. All must pass. Report any pre-existing unrelated failures
     verbatim instead of fixing them.
   - Then run `bob plugins sync`.
   - No bob-cli or SASE memory changes are needed.
     `decisions:answering-advances-the-walk` and bob-cli `docs/freshness.md` §"settle
     window that swallows double presses" stay true, because the key is still swallowed.

## Out of scope

- An Alt+P capture-phase physical-key fallback like Alt+F/Alt+N. Obsidian's `vkey` match
  already dispatches Option+P, and the existing test "the registered command owns Alt+P"
  pins command ownership.
- Changing lock durations or the double-press guard.

## Acceptance

- `rg bob-task-card-quick-pending plugins/bob-navigation-hotkeys/src` shows only the
  helper.
- Unit tests prove that both `containerEl` and `modalEl` are cleared on every reveal
  path, and (notice branch) that the busy swallow shows exactly one notice.
- (e2e branch) The real-Obsidian e2e passes all scenarios, including computed
  `visibility: visible` for the revealed Pending card.
- `npm test`, `npm run validate`, and `npm run docs:check` are green, and
  `bob plugins sync` has deployed bob-navigation-hotkeys to the vault.
- Bryan's next check on the Mac: Alt+P on a landed Ready no-priority row writes P1 and
  advances. Alt+P on a Pending row shows the summary stage. If a key is ever swallowed,
  a visible notice now says so.
