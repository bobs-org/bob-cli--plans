---
tier: tale
title: Ctrl+Enter adds a labelled task relation in Obsidian and Bob Mac Capture
goal:
  Ctrl+Enter on a relation picker row opens the relation label card in the Obsidian
  Links stages and Bob Mac Capture, replacing Shift+Enter, with every hint, test, and
  doc updated.
size: medium
decisions:
  shift_enter:
    ask:
      Should Shift+Enter stop doing anything in the relation lists once Ctrl+Enter takes
      over?
    choices:
      consume:
        Shift+Enter becomes a no-op in relation lists; it can never fall through to
        plain Enter
      alias:
        Shift+Enter keeps opening the label field as an untaught alias; only hints and
        docs move
    default: consume
    why:
      You asked for instead-of; a no-op can't unlink a CURRENT row via Obsidian's
      plain-Enter fallback
    answer: consume
proposed_by: bbugyi200.athena.19y
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-08 09:07:24
status: wip
---

# Ctrl+Enter adds a labelled task relation (Obsidian and Bob Mac Capture)

## Goal

Today, **Shift+Enter** on a row in a relation list opens the label field (the "relation
card"). Plain Enter links (or unlinks on CURRENT) without a label. The relation lists
are the Obsidian Links to / Linked from stages (Task Card `l` / `⇧L`, Ctrl+Shift+L,
Ctrl+Shift+Alt+L) and Bob Mac Capture's `~` / `~~` relation picker. Move that gesture to
**Ctrl+Enter** on both surfaces, and update every hint, accessibility string, test, and
document that teaches it. Plain Enter, Tab marking, `~` / `=` flips, and everything
inside the relation card stay exactly as they are.

No bob CLI or JSON contract changes. bob-cli changes only its own docs. The key handling
lives in the two linked repos, bob-plugins and bob-mac-capture. Open both with
`/sase_repo` and read each one's `AGENTS.md` first.

## What Ctrl+Enter does today, and what it gives up

- **Obsidian (bob-navigation-hotkeys).** In the Links stage, Ctrl+Enter is currently
  Enter. `isRecommendedRollKeydown` only takes a roll for date properties, and otherwise
  falls through to `FilteredPickerModal`, which treats any modified Enter as Enter.
  Nothing is lost.
- **Bob Mac Capture.** Ctrl+Return in every picker except Go To means "accept, capture,
  and open" (`.acceptPickerRowAndSubmitAndOpen`, hint `⌃↩ Capture & Open`). In the
  relation pickers that one-chord shortcut goes away. Command-Return still accepts and
  captures. Return followed by Ctrl+Return in the editor still captures and opens. Say
  this in the docs.
- Inside the relation card, Ctrl+Return / Ctrl+Enter already commits like Return on both
  surfaces: Mac `isolatedPromptCommand` submits any non-Command Return, and Obsidian's
  label stage falls through to the base Enter handler. Keep it that way. Pressing
  Ctrl+Enter twice therefore opens the card and then commits it. Add a test for this on
  both surfaces.

## Part 1: bob-plugins (`plugins/bob-navigation-hotkeys`)

Edit only `src/` fragments. Then run `npm run build`. Never hand-edit `main.js`.

1. **`src/450-picker-keydown.js`.** The `relationStage && stage === "value"` block that
   matches Enter with `shiftKey` and no Ctrl, Meta, or Alt becomes a Ctrl+Enter block.
   Match with the existing `isRecommendedRollKeydown(event)` predicate (Ctrl or Cmd,
   never Shift or Alt). That matches this picker family's standing "Ctrl+Enter (or
   Cmd+Enter on macOS)" convention. Keep the block where it is: before the
   `isRecommendedRollKeydown(event) && stage === "value"` roll block, so the relation
   stage claims the key first. The body
   (`chooseRelationTask(item, { withLabel: true })`, the `this.opening` guard, close on
   success) does not change.

   > [!decision] shift_enter = consume Add a sibling guard in the same place: on a
   > relation stage (`relationStage && stage === "value"`), Enter with `shiftKey` and no
   > Ctrl, Meta, or Alt calls `preventDefault` and `stopPropagation`, then returns.
   > Nothing is written, and nothing falls through to the base Enter handler.

   > [!decision] shift_enter = alias Keep the Shift+Enter match as a second trigger of
   > the same block. Only the copy changes.

   Update the comments that say `⇧↵` (lines around 23 and 45).

2. **`src/426-picker-relation-stage.js`.** In the footer hints
   (`getRelationStageFooterHints`), the mention-row `⇧↵ add relation` and the list
   `⇧↵ with a relation` become `⌃↵`. That is the same `⌃` glyph family as the label
   card's `⌃D` / `⌃H` / `⌃R`. Update the comments at the top of the file, around
   `chooseRelationTask`, and around the relation field.
3. **`src/305-relation-stage-copy.js`.** Change the row subtitles:
   `↵ opens it in its note · ⌃↵ adds a relation`, `↵ removes this source · ⌃↵ relabels`,
   and `↵ unlinks · ⌃↵ relabels`.
4. **`manifest.json`.** In the description, "Shift+Enter edits a relation label" becomes
   "Ctrl+Enter edits a relation label". Bump the minor version (currently 2.75.0, so
   2.76.0, or one minor above whatever is current). Then run `npm run docs:gen` so
   `docs/bob-navigation-hotkeys.md` picks it up.
5. **`docs/task-relations.md`** (the plugin reference, mounted into the Bob site). Every
   `⇧↵` / `Shift+Enter` that opens the relation field, relabels CURRENT, or adds a
   relation from a mention row becomes `⌃↵` / `Ctrl+Enter` (around lines 86, 118, 133,
   and 134). Grep the rest of `docs/` for any other relation-field `⇧↵`. Leave the
   unrelated `⇧↵` uses alone: inbox route stay, Pomodoro picker "new NAME", schedule
   "Skip reason", and Command Mode preview. Under `consume`, add one sentence:
   Shift+Enter in those stages does nothing.
6. **Tests** (`scripts/`). Update the copy assertions in
   `test-navigation-relation-mentions.cjs` (around 461) and
   `test-navigation-relation-links-stage.cjs` (around 1364). Rename the mention test
   title at about 486 to say `⌃↵`. Add keydown tests through `picker.handleKeydown` in
   `test-navigation-relation-links-stage.cjs`, reusing its `tildeKeyEvent`-style event
   helper:
   - Ctrl+Enter on a RESULTS row opens the `relation-label` stage. Cmd+Enter does the
     same.
   - Ctrl+Enter on a CURRENT row opens the relabel stage. It does not unlink.
   - Ctrl+Shift+Enter and Alt+Ctrl+Enter do not open the label stage.
   - **consume:** Shift+Enter on RESULTS and on CURRENT calls `preventDefault`, leaves
     `stage === "value"`, and writes nothing. CURRENT stays linked. **alias:**
     Shift+Enter still opens the label stage.
   - Ctrl+Enter in the `relation-label` stage commits like Enter.
   - In a non-relation value stage on `scheduled`, Ctrl+Enter still takes the
     recommended roll (regression). If a new test file is needed, add it to the `test`
     script in `package.json`.
7. Run `npm test` (it includes `build:check`), `npm run validate`, and
   `npm run docs:check`. Then deploy the change, as `AGENTS.md` requires:
   `bob plugins sync --repo <the opened bob-plugins checkout> -p bob-navigation-hotkeys`.
   If a provenance guard refuses, report it and do not pass `--force`.

## Part 2: bob-mac-capture

1. **`Sources/BobMacCapture/CaptureKeyCommandRouter.swift`, `pickerCommand`, Return /
   keypad Enter.** Inside the `keys == .control` branch, check
   `context.pickerSourceIsRelation` first and return `.openRelationLabelPrompt` whether
   or not a row is selected. The controller already no-ops without `selectedRowID`. Go
   To and every other source keep their current Ctrl+Return behavior.

   > [!decision] shift_enter = consume Delete the `pickerSourceIsRelation` arm from the
   > `keys == .shift` branch, so Shift-Return falls to its existing `.consumeKey`.

   > [!decision] shift_enter = alias Keep that arm and update the comment to call it an
   > untaught alias.

2. **`Sources/CaptureCore/CapturePickerPresentation.swift`.**
   - In `coreKeyHintItems`, the `.linksTo, .linkedFrom` entry becomes
     `("⌃↩", "Link with a relation")` in place of `⇧↩`. Fix the stale "later phase"
     comment.
   - `controlReturnHint` returns `nil` for `.linksTo` / `.linkedFrom`, so
     `⌃↩ Capture & Open` is never listed there.
   - `keyHintVariants` must stop removing every `⌃↩`. The compact variant drops only the
     source's Control-Return _open_ hint (the value of `controlReturnHint`, when there
     is one). The relation hint must survive narrow layouts.
   - The `.linksTo, .linkedFrom` `cardAccessibilityHint` becomes: "Arrow keys move,
     Return inserts the relation, Control-Return links with a relation, Command Return
     inserts and captures, tilde or equals flips direction, Escape cancels."
3. **`Sources/BobMacCapture/CapturePickerView.swift`, around line 1320.** The
   `ViewThatFits` fallback filters `$0.keys != "⌃↩"` by itself. Route it through
   `source.keyHintVariants(...)`, or the same predicate, so it also keeps the relation
   hint and cannot drift from `CaptureCore`.
4. **`Sources/CaptureCore/DependencyPickerPresentation.swift`, `actionLine`.** The line
   becomes `↩ link · ⌃↩ link with a relation · ~ or = flips direction`. The `=` opener
   rewrite in `CapturePanelModel.swift` (around line 1731) replaces only the
   `~ or = flips direction` suffix. Confirm it still produces
   `↩ link · ⌃↩ link with a relation · == types = · ~ flips`.
5. **Comments in `CapturePanelModel.swift`.** In `openRelationLabelPromptFromPickerRow`
   and the design-test helper around line 13265, "Shift-Return relation pick" becomes
   Control-Return. The model logic does not change.
6. **Tests.**
   - `CaptureRelationPanelTests.testShiftReturnOpensRelationCard` becomes a
     Control-Return test (`modifierFlags: .control`, keyCode 36). Add cases:
     - keypad Enter (keyCode 76) with Control opens the card;
     - Control-Return with `pickerHasSelection: false` still returns
       `.openRelationLabelPrompt`;
     - Control-Return in a non-relation picker (for example `.dependency` with a
       selection) still returns `.acceptPickerRowAndSubmitAndOpen`;
     - **consume:** Shift-Return in a relation picker returns `.consumeKey`.
   - Update `CapturePickerDesignTests` (around line 277) to the `.linksTo` hint keys
     `["↑↓", "↩", "⌃↩", "~", "⌘↩", "esc"]` and the matching actions. Add a
     `keyHintVariants` assertion that the relation source's variant(s) keep `⌃↩`, and
     that `.activeTask`'s compact variant still drops its `⌃↩`.
   - Update `DependencyPickerPresentationTests` (around line 479) and any equals-opener
     assertion of the `== types =` line.
   - Grep `Tests/` for other relation tests that drive Shift-Return (keyCode 36 with
     `.shift` alongside `openRelationLabelPrompt` or relation pickers, for example in
     `CaptureSeparatorToggleTests` and `CaptureRelationEqualsAliasTests`) and move them
     to Control.
7. **Docs** (mounted into the Bob site). Update:
   - `docs/relationship-pickers.md`: the action lines around lines 113 and 134–136,
     "Escape from a Shift-Return card", and the "Relation Picker keys" summary around
     line 187. Control-Return opens the relation card. Drop "Control-Return Insert,
     Capture & Open" for relation pickers. Note that Return then Control-Return still
     captures and opens.
   - `docs/keyboard.md`: the Control-Return row of the editor/completion table gets a
     relation-picker exception, line 122's "Shift-Return inserts the token and opens its
     label card" changes, and the relation-card Escape row says "after Control-Return".
   - `docs/pickers.md` line 12, if its generic Control-Return claim needs the relation
     exception. Grep `docs/` and `README.md` for any remaining relation-card
     Shift-Return. Leave the unrelated Shift-Return uses alone (Link & Start, Insert &
     next, Move & add another, merge-in continuation, list header, pull-in none). Run
     `just docs-check`.
8. **Verify.** Follow `AGENTS.md`. Never land Swift you have not compiled.
   - Run `just portable-check` on athena.
   - Compile a scratch copy on the mac: rsync without `.build` / `.git`, then
     `bundle.sh` and `just typecheck-tests`, then delete the copy. Read the `tailnet.md`
     reference memory for mac SSH access. Never use Bryan's checkout or `just install`.
   - Run `Scripts/macos-ci-check` before landing.
   - After the push, watch the exact-SHA macOS CI run to green with `/sase_monitor`.
   - If the mac is offline, say in the final report that the macOS targets are
     unverified.

## Part 3: bob-cli docs (this repo)

- `docs/task-relations.md`. Change the LG table: LG3 `⌃↵ then a label / an empty label`,
  and LG4 `CURRENT ↵ / ⌃↵`. Under `consume`, add a row LG23:
  `⇧↵ in a Links / Linked-from stage` → `no-op: nothing links, unlinks, or opens`.
  Change the "Explicit gestures" lines (around lines 1189, 1492, and 2344: `l` / `⇧L` →
  `⌃↵`) and the who-is-source tables (around lines 1266–1267, 1572–1573, and 2412–2413:
  `Task Card l → ⌃↵`). Grep the file for any other relation-field `⇧↵`.
- `docs/guides/link-related-tasks.md`, line 52: "press `⌃↩` to open the relation card".
  Mention that it is Ctrl+Enter in Obsidian's Links stages too.
- Do not edit `docs/reference/plugins/*` or `docs/reference/mac-capture/*`. They are
  gitignored mounts owned by the sibling repos.
- Run `just docs-check` after Parts 1 and 2, so the mounted sibling pages are checked
  too.

No SASE memory note states the Shift+Enter opener: the relation-card decision records
and the Task Relation glossary entry name no opener key. A keymap is not a decision
record. No memory change is planned.

## Landing

Make one commit per repo, using conventional commits:

- `feat(nav): open the relation label field with Ctrl+Enter`
- `feat(mac-capture): open the relation card with Control-Return`
- `docs(relations): teach Ctrl+Enter for labelled relations`

Do not edit bob-cli's `Cargo.toml` version or `CHANGELOG.md`. No `!` is needed: no CLI
or JSON contract changes.

## Acceptance

- In Obsidian Links to / Linked from (single, counted, and mention rows), Ctrl+Enter,
  and Cmd+Enter on macOS, opens the label field on RESULTS and mention rows, and
  relabels on CURRENT. Plain Enter is unchanged. Footers and row subtitles teach `⌃↵`.
- In Bob Mac Capture's `~` / `~~` picker, Control-Return inserts the token and opens the
  relation card. Hints, the action line, and VoiceOver teach `⌃↩` even in narrow
  layouts. Other pickers keep `⌃↩ Capture & Open`.
- Shift+Enter / Shift-Return follows the `shift_enter` decision on both surfaces.
- `npm test`, `npm run validate`, `npm run docs:check`, `just portable-check`, the mac
  scratch compile and `typecheck-tests`, the exact-SHA macOS CI run, and bob-cli
  `just docs-check` all pass. The plugin is deployed with `bob plugins sync`.
