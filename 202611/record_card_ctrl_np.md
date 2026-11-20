---
tier: tale
title: Record Card Ctrl+N / Ctrl+P navigation
goal:
  Ctrl+N / Ctrl+P move the Record Card's highlight exactly like ↓ / ↑, including inside
  the choice filter on macOS.
size: small
decisions:
  browse_ctrl_np:
    ask:
      Should Ctrl+N / Ctrl+P also step between fields when no stage is open (browse
      mode)?
    choices:
      everywhere:
        Ctrl+N/P also move the field selection in browse; a full ↓/↑ synonym on the card
      pickers_only:
        Ctrl+N/P move only picker highlights; browse keeps j/k, arrows, and Tab
    default: everywhere
    why: One rule (Ctrl+N/P = ↓/↑ wherever arrows move) matches Bob's other pickers
    answer: everywhere
proposed_by: bbugyi200.athena.1ej
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-20 09:04:54
status: wip
---

# Record Card: Ctrl+N / Ctrl+P move the highlight like ↓ / ↑

## Problem

Bryan's screencast (`~/tmp/screencasts/20261120_085557.mp4`, macOS) shows the Record
Card editor on a `type: [[record]]` note. The **Status** choices stage is open with the
**Filter choices** input focused and the highlight on the current choice (Draft). He
presses `⌃P`, then `⌃N` three times, and the highlight never moves. He then presses `↓`
three times (the highlight moves Draft → Ready → Placed → Later) and `↑` three times
(back to Draft). Arrows work, but Ctrl+N and Ctrl+P do nothing. Every other Bob picker
(relation card, bank subscription linker, bank category card, Emoji Slot, and
`FilteredPickerModal` subclasses) already treats Ctrl+N / Ctrl+P as ↓ / ↑.

## Root cause

All code lives in the `bob-plugins` linked repo. Open it with `/sase_repo`
(`sase repo open bob-plugins -r "..."`) and read its `AGENTS.md` first.
`plugins/bob-ledger-tools` uses the committed fragment build: edit `src/`, run
`npm run build`, and never hand-edit `main.js`.

- The live key handler is `recordOnKeyDown` in
  `plugins/bob-ledger-tools/src/242r-plugin-record-card-typed-stages.js`.
  `Object.assign(RecordCardModal.prototype, …)` replaces the class-body copy in
  `242p-plugin-record-card.js`, which is dead at runtime. Do not edit the 242p copy.
- **Branch with focus in an input** (`this.recordInInput(event)`): with the
  `choice-filter` input or a `choices` stage, only `ArrowDown` / `ArrowUp` call
  `this.recordMoveChoice(±1)`. With a `suggest` / `link` stage, only arrows call
  `this.recordMoveSuggest(±1)`. Ctrl+N / Ctrl+P fall through to the bare `return`
  without `preventDefault`. On macOS, Chromium then runs its native Emacs caret binding
  (caret to start or end of the filter text), and the highlight never moves. That is the
  screencast's symptom.
- **Choices-stage branch with focus outside the input**: only `ArrowDown`/`j` and
  `ArrowUp`/`k` move.
- **Browse branch with no stage**: only arrows, `j`/`k`, and Tab move the field
  selection (`this.recordMove(±1)`).
- No record card fragment has ever bound Ctrl+N/P (`git log -S"Ctrl+N"` over the
  242n–242s fragments and `docs/record-card.md` is empty). The scan found no other
  plugin capture-phase listener that consumes plain Ctrl+N/P: the navigation-hotkeys
  capture handlers all require Shift or other keys. The record card simply ignores the
  chord.

## Changes

### 1. Pure chord helper (`242n-plugin-record-card-model.js`)

Add `recordCardCtrlNavDelta(event)`. It returns `1` for the exact Ctrl+N chord, `-1` for
the exact Ctrl+P chord, and `0` otherwise. Match the semantics of
`albumEmojiSlotMoveKey` in `242f-plugin-emoji-slot-model.js`:

- Require `event.ctrlKey`.
- Reject `altKey`, `metaKey`, `shiftKey`, and `isComposing`.
- Match on `key` (`n`/`N`/`p`/`P`) or `code` (`KeyN`/`KeyP`), so Caps Lock and
  non-QWERTY layouts work.
- Rejecting Shift keeps `Ctrl+Shift+P`, the Record Card's own entry chord, out of
  navigation.

Give it a one-line comment. Export it from `src/372-exports-record-card.js` alongside
`recordCardHomeEnd`.

### 2. Bind the chord where arrows move a highlight (`242r` `recordOnKeyDown`)

Ctrl+N must do exactly what `ArrowDown` does, and Ctrl+P what `ArrowUp` does, in each
branch below. Call `event.preventDefault?.()` exactly as the arrow branches do. That
suppresses the macOS caret movement. Keep every existing early guard ahead of the new
checks: composition / `keyCode 229`, `defaultPrevented`, `pendingApply`, and the
Esc/`q`/Ctrl+[ close ladder. IME and in-flight applies must stay inert.

- **In-input branch, choices**: wherever `inputKind === "choice-filter"` or the stage is
  `choices`, map ArrowDown / ArrowUp / Ctrl+N / Ctrl+P to one delta, call
  `this.recordMoveChoice(delta)`, and return. This also covers a list stage's choice
  filter, because the existing condition already routes it here.
- **In-input branch, suggest / link stage**: route the same delta to
  `this.recordMoveSuggest(delta)`.
- **Choices-stage branch, focus outside the input**: add Ctrl+N → `recordMoveChoice(1)`
  and Ctrl+P → `recordMoveChoice(-1)` next to the `ArrowDown`/`j` and `ArrowUp`/`k`
  checks.
- **Browse branch** (decision `browse_ctrl_np`): see the callouts below. Place the new
  check after the `this.stage && this.stage.kind !== "choices"` early return. Typed
  stages with focus off their input then stay inert, as they are for arrows. Place it
  before the Ctrl+D / Ctrl+Z / Ctrl+E checks; those keys do not overlap with N/P.
- Leave every other input unchanged: text, date, number, and URL stage inputs and the
  list-item input. Ctrl+N/P keep native text behavior there, because arrows move no
  highlight in those inputs either.
- Do not change the footer hints. They list no arrow keys today (`recordCardHintItems`
  shows `⏎ pick`, `esc/q cancel`, `y YAML`).

> [!decision] browse_ctrl_np = everywhere In the browse branch, add Ctrl+N →
> `this.recordMove(1)` and Ctrl+P → `this.recordMove(-1)`, with the same wrapping as the
> arrow and `j`/`k` row checks. Update the Browse keys table (step 4).

> [!decision] browse_ctrl_np = pickers_only Skip the browse-branch binding. Add a test
> that browse Ctrl+N/P neither move `selectedKey` nor call `preventDefault`. Leave the
> Browse keys table unchanged.

### 3. Tests (`scripts/test-ledger-tools-record-card.cjs`)

Use the existing `openCard()` world and the `keyEvent(key, { ctrlKey: true, target })`
helper. Add focused tests:

- **Helper vectors**:
  - `1`: `n`, `N` (Caps Lock), and `code: "KeyN"` with a layout-shifted `key`.
  - `-1`: the `p` / `P` / `KeyP` equivalents.
  - `0`: Ctrl+Shift+N, Ctrl+Alt+N, Ctrl+Meta+N, Meta+N, plain `n`, a composing Ctrl+N,
    and a missing event.
- **Choices stage, filter focused** (status field, current `to_try`):
  - From the `[data-record-input='choice-filter']` input, Ctrl+N moves `selectedChoice`
    to the same value `ArrowDown` reaches, and Ctrl+P matches `ArrowUp`.
  - Both wrap at the ends.
  - Each sets `defaultPrevented` and leaves `choiceFilter` and the input value
    untouched.
  - With a filter typed, Ctrl+N walks only the visible choices.
  - Enter after Ctrl+N applies the highlighted choice as exactly one `field.set`.
- **Choices stage, focus off the input**: Ctrl+N/P move like arrows.
- **Modifier variants**: Ctrl+Shift+N, Ctrl+Alt+N, and Meta+N in the filter neither move
  the highlight nor call `preventDefault`.
- **Suggest stage**: give a fixture field `kind: "text", suggest: true`. The `field()`
  helper already accepts `{ suggest: true }`; add report `values` as
  `recordCardSuggestValues` expects. Ctrl+N/P cycle `stageDraftText` exactly as
  ArrowDown/ArrowUp do. Cover the link stage too if the fixture can supply
  `recordCardLinkCandidates` cheaply; otherwise skip it.
- **Typed text stage**: Ctrl+N in a `stage-text` input is not prevented and leaves the
  draft unchanged.
- **IME**: a composing Ctrl+N is inert.
- **Browse**: assert the chosen `browse_ctrl_np` behavior against `selectedKey`,
  including ghost rows and wrapping when bound.

### 4. Docs (`docs/record-card.md`, the plugin reference for this hotkey)

- **Browse keys** table, first row (only if `browse_ctrl_np = everywhere`): "`j` / `k`,
  arrows, Ctrl+N / Ctrl+P, Tab / Shift+Tab".
- **Choices** section: say that `j`/`k` outside the filter, arrows and Ctrl+N / Ctrl+P
  (also inside the filter), and Enter pick from the list.
- **Stages** table, Suggest and Link rows: add that arrows or Ctrl+N / Ctrl+P step
  through candidates.
- Keep the edits to these lines. Run `npm run docs:check`.

### 5. Version, build, validate, deploy, commit (in `bob-plugins`)

1. Bump `plugins/bob-ledger-tools/manifest.json` to the next **patch** version above
   master's at implementation time (currently `1.141.0` → `1.141.1`). Run
   `npm run docs:gen` to refresh the generated version rows in `docs/index.md` and
   `docs/bob-ledger-tools.md`.
2. Run `npm run build`, then `npm run build:check`.
3. Run
   `node --test scripts/test-ledger-tools-record-card.cjs scripts/test-ledger-tools-record-card-stages.cjs scripts/test-obsidian-record-card-e2e.cjs`,
   then the full `npm test` and `npm run validate`. If unrelated timing tests fail,
   rerun them on a pristine master checkout and report them as pre-existing only if they
   also fail there.
4. Run `bob plugins sync`. Follow the AGENTS.md refusal protocol and never pass `-F` on
   your own.
5. Commit the `bob-plugins` changes as one `fix(record-card): …` commit. The
   `/sase_final` repo obligation covers it.

A real-Obsidian acceptance run (`docs/develop/validation.md`) is **not** required. Linux
Electron has no native Ctrl+N/P caret binding, so it cannot reproduce the macOS symptom,
and the handler-level unit tests are the gate. State in the final report that Bryan
should confirm on the Mac after sync: in the Status choices stage, Ctrl+N/P should move
the highlight like ↓/↑.

## Out of scope

- The album schema card (`241y`), bank setup card, and record walk rail do not bind
  Ctrl+N/P either. Leave them unchanged. If you think they should match, file a task
  bead with `/sase_new_task` rather than widening this change.
- No change to bob-cli, Bob Mac Capture, or any memory note.
