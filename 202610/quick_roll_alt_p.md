---
tier: epic
title: One-key recommended roll (Alt+P) with review-walk previews
goal: 'One keystroke on a task applies exactly what the Task Card''s Ctrl+Enter recommends
  (same-level roll, decay, or a confirmed cancel), and every GTD review landing and
  the review footer say in a few words which priority that key would use and whether
  it decays.

  '
decisions:
  hotkey:
    ask: Which key should run the one-key recommended roll?
    choices:
      alt_p: ⌥P replaces the unused Pomodoro-schedule template; Cmd+P stays Obsidian's
        command palette
      mod_p: ⌘P replaces Obsidian's command palette (Ctrl+P off macOS); ⌥P keeps the
        template
    default: alt_p
    why: The ⌥P template is the Pomodoro binding you don't use; Cmd+P is the command
      palette
    answer: alt_p
  cancel_confirm:
    ask: Should a cancel recommendation wait on the Task Card for a second press before
      cancelling?
    default: true
    why: Every other outcome reschedules; a one-key cancel is the only answer that
      ends a task
    answer: true
  work_summary:
    ask: Should the key still offer the optional Work summary on Next and Pending
      tasks?
    default: true
    why: Identical to Ctrl+Shift+P then Ctrl+Enter; a second press or blank Enter
      skips it
    answer: true
  memory_footer_glossary:
    ask: Update the Morning GTD Review Footer glossary strand to name the roll preview?
    memory:
    - glossary:Morning GTD Review Footer
    default: true
    corrects: on a queue row it reads `⟳ Review r/N · TIER i/M` plus that row's detail
    answer: true
  memory_decision_record:
    ask: Record the one-key roll design as a new decisions-web record?
    memory:
    - decisions
    default: false
    answer: false
phases:
- id: quick-roll-key
  title: The one-key roll command in bob-navigation-hotkeys
  depends_on: []
  size: medium
  description: 'quick-roll-key: add the Task card roll-by-recommendation command on
    the chosen key, run it as a one-shot Task Card intent that stays invisible unless
    input is needed, alias the key inside the card, rebind the vault hotkey, and document
    it.

    '
- id: review-preview
  title: Roll preview on review-walk landings and api.quickRoll
  depends_on:
  - quick-roll-key
  size: medium
  description: 'review-preview: extract one shared recommendation planner, format
    the concise roll preview, add it as the last line of `]s` landing toasts, and
    expose it as the additive nav `api.quickRoll` v1 namespace.

    '
- id: footer-preview
  title: Roll preview in the review footer
  depends_on:
  - review-preview
  size: small
  description: 'footer-preview: bob-ledger-tools shows the compact preview in the
    footer''s hint slot and the full sentence in its tooltip while the cursor is on
    a review row.

    '
- id: review-docs
  title: Review contract, API contract, and memory updates in bob-cli
  depends_on:
  - quick-roll-key
  - review-preview
  - footer-preview
  size: small
  description: 'review-docs: update the freshness review ritual and the nav API contract
    in bob-cli docs, apply the accepted memory decisions, and regenerate generated
    docs.'
proposed_by: bbugyi200.athena.132
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-10-23 06:34:42
status: wip
bead_id: bob-cli-aq
---

- **PROMPT:** [prompts/202610/quick_roll_alt_p.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202610/quick_roll_alt_p.md)
- **BEAD:** [bob-cli-aq](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-aq/README.md)

# Plan: One-key recommended roll (Alt+P) with review-walk previews

## Why

Bryan wants a faster way to say "not now" on a prioritized task. Today that takes
`Ctrl+Shift+P` (open the Task Card) and then `Ctrl+Enter` (take the recommended roll,
decay, or cancel from the priority ladder). This plan turns those two keys into one and
shows, on every review landing, what that key would do. The review is the place where
"not now" is answered dozens of times a morning (`docs/freshness.md` §6 already says
"for a tickler, 'not now' is a priority roll").

**Which key.** The vault binds nothing to Cmd+P. Obsidian's core default puts Cmd+P on
the command palette. The binding that "does something with Pomodoros" is **⌥P / Alt+P**:
`~/bob/.obsidian/hotkeys.json` maps `templater-obsidian:_templates/schedule.md` (the
Templater replica of the nvim `se` snippet that inserts `(**HHMM-HHMM** [t:: 25m])`
Pomodoro atoms) to `Alt+P`. The default (`hotkey = alt_p`) takes that slot. It also puts
the new key next to the other one-key review answers, `Alt+F` (keep) and `Alt+N`
(next/release). Every hint string in this plan says `Alt+P` because existing hints spell
keys that way (`Ctrl+Alt+F keep · Alt+N release`). All hint text reads one constant,
`QUICK_ROLL_KEY_LABEL`, so a later rebinding changes one line.

> [!decision] hotkey = mod_p The command's default hotkey is
> `{ modifiers: ["Mod"], key: "P" }` and `QUICK_ROLL_KEY_LABEL` is `Cmd+P` on macOS and
> `Ctrl+P` elsewhere. The vault edit sets `"command-palette:open": []` (leave the
> Templater `Alt+P` entry alone). Inside the card, the alias is `Cmd+P` on macOS only,
> because off macOS `Mod+P` is `Ctrl+P`, which already moves the card selection up.

## Design

### One rule: the key is "Ctrl+Shift+P, then Ctrl+Enter"

The key never gets its own writer. It opens the same `BulletPropertyPickerModal` as
`Ctrl+Shift+P`, through the same `openBulletPropertyPicker` / `openLinkPicker` entry
points, with a one-shot `initialTaskCardIntent: "apply-recommendation"`. This mirrors
how `Ctrl+Shift+B` is "Ctrl+Shift+P then `b`" (`initialTaskCardRow` +
`dispatchInitialTaskCardRow` in `350-picker-task-card.js`). Everything the Ctrl+Enter
path already does is inherited unchanged:

- the live-note recheck (`isSamePriorityRollTarget`), Schedule Log reasons, and project
  `^prj` inline scheduling;
- inbox routing (`runInboxRoutedCommit`), the rich priority notice, undo as one unit,
  and Pomodoro link pruning;
- review-walk capture and settle (`reviewOrigin` / `settleReviewOrigin`, so an answer on
  a landed row advances once), the gesture lock, and the review journal (a roll is
  journaled as `defer` from the line diff).

The date is rolled fresh at the moment of the press, exactly as when the card opens.

### Invisible unless the user is needed

On the immediate path the card must not flash. The picker opens with the class
`bob-task-card-quick-pending` on its modal container (`this.containerEl`). The CSS hides
the whole container (`visibility: hidden`, no animation). The class comes off
(`revealQuickRollCard()`) at the first moment the user is needed, and only then:

1. the dispatch renders any stage or re-renders the card (Work summary stage, the "Task
   changed while the picker was open" rebuild, a refusal that returns home);
2. the inbox-route picker opens or returns (`suspend`/resume pair);
3. the one-shot dispatch settles (resolves, refuses, or throws) and the picker is still
   open. This is the safety net: the card is never left open and invisible.

While the dispatch is in flight, the existing `taskCardDispatching` guard already
refuses every other card intent. Stray keys typed during a write therefore do nothing.

### What the key does

| Situation (cursor task)                                    | Result                                                                                                                                                                                                                        |
| ---------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Open task, recommendation is a **roll** or **decay**       | Writes at once through the card's Ctrl+Enter path; card never visible; rich priority notice; landed row advances the walk.                                                                                                    |
| Same, task is **Next or Pending**                          | Work summary stage appears (as with Ctrl+Enter). Enter, blank Enter, or a second press of the key submits.                                                                                                                    |
| Inbox note                                                 | Route prompt first, then the write (as with Ctrl+Enter). Backing out of the route prompt returns to the now-visible card.                                                                                                     |
| Recommendation is **cancel** (decayed past the last level) | Card appears on its danger banner and nothing is written. A second press (or Ctrl+Enter) cancels with the existing decay-cancel reason; Esc keeps the task. Notice: `Decayed past P4 — Alt+P again cancels it, Esc keeps it`. |
| No priority (implicit P0)                                  | Card appears and nothing is written. Notice: `No priority to roll — pick a level (1–4)`.                                                                                                                                      |
| Recurring task at the last level (`unavailable`)           | Card appears with the existing unavailable reason; nothing is written.                                                                                                                                                        |
| Counted `N` + key in Vim normal mode                       | Counted session, like `N<Ctrl+Shift+P>` then Ctrl+Enter (count read and reset in the command path, as `toggleTaskLane` does). Waits on the card if any target would cancel.                                                   |
| Pomodoro Task Link bullet                                  | Link session for the linked task(s), like Ctrl+Shift+P then Ctrl+Enter there; same cancel rule.                                                                                                                               |
| Depends-On / BLOCKS / LINKS / LINKED FROM line             | Acts on the owning task: the cursor moves to the parent, then the plain-task path runs. It never opens those stages.                                                                                                          |
| Not a bullet, selection spans tasks, mobile                | Same refusal notices as Ctrl+Shift+P.                                                                                                                                                                                         |
| Inside an open Task Card                                   | The key is an alias of Ctrl+Enter (held-key repeat ignored, like the other action keys).                                                                                                                                      |

> [!decision] cancel_confirm = no A cancel recommendation is applied immediately,
> exactly like Ctrl+Enter (still behind the inbox route prompt). Only "no priority" and
> "unavailable" reveal the card. Batches containing cancels also apply immediately. The
> preview says `cancel` without the `?`.

> [!decision] work_summary = no The key never offers the Work summary: the one-shot
> intent tells `maybeOfferRecommendedWorkLog` (and the counted/link equivalents) to
> dispatch with a null summary. Ctrl+Enter inside the card keeps offering it. Drop the
> "second press submits the Work summary" behavior.

The Task Card banner teaches the key: next to its `Ctrl+Enter` keycap it gains a second,
quieter keycap showing `QUICK_ROLL_KEY_LABEL` (class `is-alias`, reduced opacity in
`styles.css`), only when the recommendation is available.

### The preview: concise, truthful, and identical to what the key will do

Review rows show what the key would do in at most one short line. The words come
straight from the recommendation the key will use: the same planner (see
`review-preview`), so the preview can never disagree with the write. It names the level,
the roll step or decay, and the rolled **window**. It never names an exact date, because
the date is drawn when the key is pressed. The rich notice after the write shows the
exact date.

| Recommendation                               | Toast line (`line`)               | Footer hint (`compact`) | Tooltip / aria (`detail`)                                    |
| -------------------------------------------- | --------------------------------- | ----------------------- | ------------------------------------------------------------ |
| roll, limit 1, step 1                        | `Alt+P roll P2 1/1 · 8–30d`       | `Alt+P roll P2 1/1`     | `Alt+P: roll P2 again (1 of 1), 8–30 days out`               |
| roll, decay disabled                         | `Alt+P roll P2 · 8–30d`           | `Alt+P roll P2`         | `Alt+P: roll P2 again, 8–30 days out`                        |
| decay                                        | `Alt+P decay P2 → P3 · 31–90d`    | `Alt+P decay P2→P3`     | `Alt+P: decay P2 → P3, 31–90 days out`                       |
| cancel (confirm)                             | `Alt+P cancel? · decayed past P4` | `Alt+P cancel?`         | `Alt+P: cancel — decayed past P4 (confirm on the Task Card)` |
| unavailable, no priority, closed, not a task | none                              | none                    | none                                                         |

The window comes from `min_days`–`max_days` of the level the date is drawn from: the
current level for a roll, the target level for a decay. Use an en dash, `8d` for a
one-day window, and omit the window when the level has no bounds. Previews are never
shown on PRE/POST checklist rows, which resolve only by completion. The tier gate lives
in nav, so the toast and the footer share it.

Example landing toast (ROTTEN):

```
Review 14/31 · ROTTEN 2/9 · rotten 5d · every 7d
Alt+P roll P2 1/1 · 8–30d
```

Example landing toast (NEXT; the existing action-hint line stays and the preview
follows):

```
Review 5/31 · NEXT 2/6 · confirmed yesterday
Still next? Ctrl+Alt+F keep · Alt+N release · Ctrl+Shift+Enter today
Alt+P decay P1 → P2 · 8–30d
```

Example footer on a review row:
`⟳ Review 14/31 · ROTTEN 2/9 · rotten 5d · Alt+P roll P2 1/1 · TICKS 2 · ✓ 4/10 today`

### Rejected alternatives (for the record and for implementers)

- **A separate headless writer.** It would fork inbox routing, Work Log, project
  scheduling, undo, and walk settle. Reusing the card's path is what makes the key
  reliable.
- **Exact dates in previews.** That would need a frozen pre-roll per landing, carried
  into the card and invalidated on every edit. The window answers "which priority, and
  does it decay" truthfully.
- **An inline ghost chip on the task line.** It would put a new editor decoration on
  every landing. The toast and footer are the walk's established surfaces.
- **Binding "Ctrl+Shift+P, Ctrl+Enter" as a macro.** It shows the card, can't hide it,
  and can't apply the cancel rule.

## Phase `quick-roll-key`: the one-key roll command in bob-navigation-hotkeys

Work in the linked `bob-plugins` repo (open it with `sase repo open bob-plugins`; read
its `AGENTS.md`). Edit `plugins/bob-navigation-hotkeys/src/` fragments, register new
ones in `src/fragments.json`, and run `npm run build`. Never hand-edit `main.js`. Each
hand-edited fragment must stay ≤ 1000 lines. `310-task-card-previews.js` (984),
`480-review-jump-and-nav-api.js` (987), and `536-plugin-review-advance.js` (976) are
near the limit, so put new code in new fragments and keep edits there to a few lines.

1. **Command** (`490-plugin-lifecycle.js`, next to the other `task-card-*` commands): id
   `roll-by-recommendation`, name `Task card: Roll by recommendation`, default hotkey
   `{ modifiers: ["Alt"], key: "P" }`,
   `editorCallback: (editor) => this.rollByRecommendation(editor)`.
2. **Plugin mixin** (new fragment, for example `681-plugin-quick-roll.js`, class
   `BobNavigationHotkeysQuickRollMixin`, registered in `690-install-plugin-mixins.js`):
   `rollByRecommendation(cm)` reads and resets a pending Vim count exactly like
   `toggleTaskLane` (`510-plugin-link-commit-and-lane.js`). It then calls
   `openBulletPropertyPicker(cm, { initialTaskCardIntent: "apply-recommendation", countExplicit, additionalTaskCount })`.
   If a bullet-property picker is already open
   (`activeBulletPropertyPicker?.pickerOpen`), the command does nothing. The open card's
   own alias handles the key, so the chord never reopens a card or applies twice. Define
   `QUICK_ROLL_KEY_LABEL` here.
3. **Thread the option** through `openBulletPropertyPicker`
   (`550-plugin-cancel-and-counted-property.js`) and `openLinkPicker`
   (`500-plugin-transclusion-and-link-picker.js`) next to `initialTaskCardRow`. Also
   store it in the picker constructor (`340-picker-core.js`). With the intent set, the
   managed-line redirect only moves the cursor to the owning task and then continues on
   the plain-task path; it never sets `reopen*Stage` or `initialProperty`. Review
   capture stays exactly where it is (plain-task path only).
4. **Picker mixin** (new fragment before `460-install-picker-mixins.js`, for example
   `457-picker-quick-roll.js`, class `BulletPropertyPickerQuickRollMixin`, added to the
   `installBulletPropertyPickerMixins` list). Method names must not collide (the
   installer throws on duplicates):
   - `dispatchInitialTaskCardIntent()`: a one-shot like `dispatchInitialTaskCardRow`,
     called from the same two sites after `open()`. Link sessions wait for link
     resolution exactly as the row dispatch does. It decides between applying at once
     and waiting on the card: reveal and wait when the model's recommendation is
     unavailable (notice per the table: the "no priority" text when the task has no
     priority field, otherwise `unavailableReason`), or when `cancel_confirm` holds and
     any target would cancel (`effects.cancel > 0`). Otherwise it dispatches
     `{ type: "apply-recommendation" }` and reveals in `finally` if the picker is still
     open.
   - `revealQuickRollCard()` (idempotent), plus the reveal hooks from "Invisible unless
     the user is needed". Hook them with one-line calls in the existing stage/card
     render paths and the inbox-route suspend/resume. Do not reorder those methods.
   - Add the pending class before the modal paints (set it in `onOpen` before render, or
     immediately after `open()`, whichever the harness shows avoids a painted frame) and
     the CSS rule in `plugins/bob-navigation-hotkeys/styles.css`.
5. **In-card alias.** In `resolveTaskCardKey` (`310-task-card-previews.js`), treat
   `altKey && !ctrlKey && !metaKey && !shiftKey && event.code === "KeyP"` exactly like
   Ctrl+Enter. Use `event.code`, because ⌥P types `π` on macOS. Include it in the
   `actionKey` repeat guard. In the Work summary stage opened by this key, the same
   chord submits like Enter (`450-picker-keydown.js`), so "press twice" means "roll, no
   summary". Never type `π` into the field.
6. **Banner keycap** (`330-task-card-view.js`): a second
   `appendTaskCardKeycap(bannerMain, QUICK_ROLL_KEY_LABEL)` with class `is-alias`, plus
   its `styles.css` rule.
7. **Vault hotkey.** In `~/bob/.obsidian/hotkeys.json`, set
   `"templater-obsidian:_templates/schedule.md": []` so `Alt+P` has one owner. Keep the
   file's existing 2-space JSON formatting and validate it parses. `bob vault-sync`
   carries the change to the other machines (no manual vault commit). The template stays
   runnable through Templater's insert command.
8. **Docs** (bob-plugins `docs/`): in `docs/task-card.md`, add an `Alt+P` row to the
   gesture table ("same as Ctrl+Enter") and a short "One-key roll" section covering the
   behavior table above. Add one sentence to the nav `manifest.json` description. Run
   `npm run docs:gen` (the commands table picks up the new command and default hotkey).
   Bump the nav manifest minor version.
9. **Tests**: a new `scripts/test-navigation-quick-roll.cjs` built on
   `scripts/navigation-hotkeys-harness.cjs` / `scripts/modal-harness.cjs`, added to the
   `npm test` list in `package.json`. Equivalence is the core assertion. With a fixed
   `random` and `baseDate`, the key and "open card + Ctrl+Enter" must produce
   byte-identical note content and the same notice model for: roll, decay, `^prj`
   project-inline roll, counted batch, and Task Link batch. Also cover:
   - The container carries the pending class, is never revealed on the immediate path,
     and the picker closes.
   - Next/Pending shows the revealed Work summary stage, and a second press submits
     blank.
   - Cancel: revealed, nothing written, second press cancels, Esc writes nothing.
   - No priority and recurring-unavailable: revealed, nothing written.
   - A batch containing a cancel waits.
   - Inbox note: route prompt before any write.
   - Depends-On line acts on the owner.
   - A landed review row advances once (reuse the review-advance fixtures).
   - In-card alias and repeat guard.
   - A throwing dispatch leaves the card revealed, not hidden.
10. **Ship**: `npm run build`, `npm test`, `npm run validate`, `npm run docs:check`,
    then `bob plugins sync`.

## Phase `review-preview`: roll preview on review-walk landings and `api.quickRoll`

Same repo and fragment rules.

1. **One planner.** Extract the body of `computePriorityRollRecommendation`
   (`390-picker-roll-write.js`) into a pure
   `planPriorityRollRecommendationAtLine({ config, content, line, baseDate, random })`
   in a new fragment (for example `396-quick-roll-preview.js`). It returns today's
   frozen object, including `priorityName`, `schedulesName`, and `taskLine`, or `null`.
   The picker method becomes a thin wrapper. Behavior must not change; parity tests
   prove it.
2. **Preview model** in the same fragment:
   `planQuickRollPreview(recommendation, { keyLabel, cancelConfirms })` returns a frozen
   `{ ok: true, kind, line, compact, detail }` or `{ ok: false, reason }`, producing
   exactly the strings in the preview table.
   `quickRollPreviewForLine({ config, content, line, tier })` combines the planner (with
   a constant `random`, since the date is unused) and the tier gate (`pre`/`post` →
   `{ ok: false }`).
3. **Config without noise.** Previews load the config with `showNotice: () => {}` and
   memoize it by the config file's mtime. A missing or broken config yields
   `{ ok: false }` and never a Notice.
4. **Landing toasts.** A plugin method `reviewQuickRollLine(entry)` reads the landed
   note's content: from the active editor when it shows `entry.path`, else via
   `app.vault.cachedRead`. It locates the row with the existing pure
   `resolveReviewQueueLine`, takes the tier from `reviewEntryMachineTier(entry)`, and
   returns `preview.line` or `""`. A helper `appendQuickRollLine(notice, line)` appends
   `"\n" + line` after everything the notice already contains (including a wrapped tail
   or a POST tail), so notices without a preview stay byte-for-byte unchanged. Call it
   at the landing sites that can land on a non-checklist row: the `]s` jump
   (`536-plugin-review-advance.js`, the `buildReviewJumpNotice` call near the end), the
   answer-advance landing (same fragment, before `composeReviewAdvanceNotice`), and the
   walk return (`539-plugin-review-walk-return.js`). Skip the checklist walk
   (`535-plugin-review-checklist-walk.js`, PRE/POST only). Each call site adds at most
   about three lines. Split a fragment if it would pass 1000 lines.
5. **`api.quickRoll` v1** on `createDependencyNavApi` (one added line in `480`; factory
   `createQuickRollApi(plugin)` in the new fragment). The root `version` stays 3. The
   shape is `{ version: 1, keyLabel, preview({ content | lines, line, tier }) }`.
   `preview` is synchronous and never throws, returning the frozen preview object. With
   a null plugin (after unload) it returns `{ ok: false, reason: "unavailable" }`.
6. **Docs**: add `quickRoll` to the nav table in `docs/develop/plugin-apis.md`, add one
   sentence about the landing preview line in `docs/bob-navigation-hotkeys.md`, and
   update the manifest description. Run `npm run docs:gen` and bump the nav minor
   version.
7. **Tests**: a new `scripts/test-navigation-quick-roll-preview.cjs` in the `npm test`
   list, covering:
   - Planner parity with the old method over roll/decay/cancel/unavailable/P0/closed/
     `^prj` fixtures.
   - Every preview-table string, including no-limit, one-day window, no bounds, and
     cancel with and without confirm.
   - PRE/POST and no-priority give `{ ok: false }`.
   - A silent config failure gives `{ ok: false }` with no Notice.
   - Toast composition: ROTTEN gets the second line; NEXT keeps its action-hint line,
     then the preview; wrapped and POST tails are untouched; no-priority rows are
     byte-identical to today.
   - The `api.quickRoll` shape and the null plugin.
   - **Preview truth**: for each fixture, the previewed kind, level, and step match what
     `rollByRecommendation` then writes, and the written date falls inside the previewed
     window.
8. **Ship**: build, `npm test`, `npm run validate`, `npm run docs:check`,
   `bob plugins sync`.

## Phase `footer-preview`: roll preview in the review footer

Same repo; plugin `bob-ledger-tools` (`120-freshness-footer.js` is at 946 lines, so keep
edits small or add a fragment).

1. In `updateFreshnessStatusBar` (`240-plugin-freshness-status.js`), only when `current`
   matched a queue row and nav exposes `api.quickRoll?.version >= 1`, do the following.
   Reuse the existing `dependencyNavApi()` accessor pattern from
   `270-plugin-dependency-model.js`. Get the cursor's source lines with
   `freshnessFooterCursor(true)` and call `preview({ lines, line: line0, tier })`. Pass
   the result to `freshnessFooterView` as `options.quickRoll`. Wrap the call in
   try/catch; the footer never throws.
2. In `freshnessFooterView`, on a queue row with `quickRoll.ok`, set
   `hintText = quickRoll.compact` and push `quickRoll.detail` onto the tooltip lines
   next to the existing `actionHint`. The aria label already includes `hintText`, and
   the paint key already includes `hintText`, so repaint and fitting (which drops the
   hint before the meter) need no change. Off-row hints (`]s next`) are unchanged.
3. Without nav, or on PRE/POST rows, the footer is byte-identical to today.
4. **Docs**: one sentence in `docs/bob-ledger-tools.md` and the manifest description;
   `npm run docs:gen`; bump the ledger-tools minor version.
5. **Tests**: extend `scripts/test-ledger-tools-freshness-footer.cjs` with a stub nav
   `quickRoll` to cover:
   - the hint and tooltip on a ROTTEN row;
   - no nav, a v0 namespace, a throwing preview, and a PRE row, all unchanged;
   - the paint key changes when the preview changes.
6. **Ship**: build, `npm test`, `npm run validate`, `npm run docs:check`,
   `bob plugins sync`.

## Phase `review-docs`: review contract, API contract, and memory updates in bob-cli

Work in this bob-cli repo. Read the `docs.md` reference memory first. Rules: references
own exact behavior, and guides change only where a step became incorrect. bob-cli docs
under `docs/reference/plugins/` are assembled from bob-plugins and are not edited here.

1. `docs/freshness.md` §6:
   - In "Review outcomes", "not now" becomes `Alt+P` (the Task Card's recommended
     roll/decay in one key → next; a cancel waits on the card), with `Ctrl+Shift+P 1–4`
     kept for choosing a level.
   - "For a tickler, 'not now' is a priority roll" names `Alt+P`.
   - The footer paragraph mentions the roll preview on review rows.
   - §8's nav and ledger-tools rows note the landing preview and footer hint.
2. `docs/task-dependencies.md` §9 (nav API contract): add the additive `quickRoll` v1
   namespace in the same style as `inboxRoute`: shape, sync/never-throws, the
   null-plugin result, the PRE/POST gate, and that `api.version` stays 3.
3. > [!decision] memory_footer_glossary Use `/sase_memory_write`. In
   > `sase/memory/glossary/morning-gtd-review-footer.md`, the queue-row sentence
   > becomes: on a queue row it reads `⟳ Review r/N · TIER i/M` plus that row's detail
   > and, when the row has a recommended roll, the `Alt+P` preview in the hint slot (for
   > example `Alt+P roll P2 1/1`). Keep the rest of the strand.
4. > [!decision] memory_decision_record Use `/sase_memory_write` to add a decisions-web
   > strand (keyword "Alt+P Is The Task Card Recommendation In One Key"). It covers
   > where the decision applies (bob-plugins nav and ledger-tools, bob-cli docs, the
   > vault review ritual), the claim (the key runs the card's own Ctrl+Enter path, stays
   > invisible unless input is needed, cancels only on a second press, and previews show
   > level/step/decay/window, never a date), the rejected alternatives above, the cost,
   > when it would be reopened, and evidence (this plan's ref and the nav/ledger-tools
   > versions that shipped).
5. If any memory changed, run `sase memory init`. Then run `just docs-gen` to regenerate
   `docs/reference/glossary.md` and `docs/develop/decisions.md`, and `just docs-check`.
   This is docs-only: never edit `Cargo.toml` versions or `CHANGELOG.md`.

## Done means

- In Obsidian, ⌥P on a prioritized task rolls or decays it with no visible card and
  shows the usual rich notice. On a landed review row it advances the walk once.
- A decayed-past-last-level task shows the danger banner and waits for a second ⌥P.
- `]s` landings on eligible rows end with an `Alt+P …` line. The footer shows the
  compact preview while the cursor is on that row.
- The Pomodoro-schedule template no longer owns Alt+P. Cmd+P still opens the command
  palette (under the default `hotkey` answer).
- All bob-plugins tests and docs checks pass, `bob plugins sync` has run, and bob-cli
  `just docs-check` passes. Bryan reloads Obsidian on the Mac to pick up the new plugins
  and hotkeys.
