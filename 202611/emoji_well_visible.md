---
tier: tale
size: medium
title: Make the Emoji Slot's glyph well visible and discoverable
goal:
  In the album Schema Card, Bank setup card, and Category card, the emoji control beside
  a choice or category label reads as an obvious, clickable emoji box, and a one-line
  hint says how to use it, so nobody has to ask where the emoji input is.
decisions:
  suggest_followup:
    ask:
      Should the implementer file a follow-up task bead to improve bob-emoji's label
      suggestions?
    choices:
      file: File a large task bead with the measured noisy examples; no code change here
      skip: File nothing; the noisy "For <label>" row stays as it is
    default: file
    why: Once people can find the panel they will see its noisy first row more often.
    answer: file
proposed_by: bbugyi200.athena.1da.f0
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-17 08:16:53
status: wip
---

# Make the Emoji Slot's glyph well visible and discoverable

## Problem

Bryan's screencast of the album Schema Card's **Add choice** stage shows the label input
(`foo`) with a flat gray square at its left holding a faint ☻. Nothing reads as an emoji
input, so he asked where the emoji input box is.

That square is the Emoji Slot's **glyph well**, a `<button data-emoji-well>` from the
bob-cli-mf epic (`242g-plugin-emoji-slot-dom.js`). Clicking it, or pressing Tab from the
label and then Enter, Space, ↓, or any printable character, opens the in-card search
panel. The panel stays hidden until then. The same well appears in the Bank setup card's
new/edit category stages and in the Category card's new-category stage
(`242h-plugin-bank-emoji-slot.js`).

Two defects hide it:

1. **Obsidian's button chrome overrides the well's CSS.** Obsidian's `app.css` has
   `button:not(.clickable-icon){background-color:var(--interactive-normal);box-shadow:var(--input-shadow);padding:var(--size-4-1) var(--size-4-5);width:100%}`.
   That selector has specificity (0,1,1), which beats the well's single-class
   `.bob-emoji-slot-well` rules (0,1,0) in `plugins/bob-ledger-tools/styles.css`. The
   result is the gray `--interactive-normal` fill with a shadow, not the intended
   `--background-primary` tile. The empty state's dotted border does apply, but it is
   `--background-modifier-border` on that gray fill, so it vanishes. The glyph is
   `--text-faint`, so the tile looks like a disabled avatar. The screencast frame shows
   exactly this. Other plugins in this repo already override the chrome with
   `button.<class>` selectors and `box-shadow: none`; see `button.bob-bracket-mode-seg`
   in `plugins/bob-navigation-hotkeys/styles.css`.
2. **Nothing teaches the gesture.** The empty well has no visible hint and no tooltip,
   and the card footer says nothing about emoji. The real-Obsidian e2e never caught
   this: its only probe (`schema-emoji-slot-panel` in
   `scripts/obsidian-album-ui-e2e.cjs`) calls `well.click()` from script and never
   checks the well's computed style or a real key path.

The panel's behavior, key isolation, Esc ladder, fused-token compose rules, and the
three hosts' save paths are correct and stay unchanged.

## Changes (all in the `bob-plugins` linked repo, plugin `bob-ledger-tools`)

Edit `src/` fragments and run `npm run build`; never hand-edit `main.js`. Every
hand-edited fragment must stay at or below 1000 lines.

### 1. Well styling that beats Obsidian's button chrome (`styles.css`)

- Restyle the well with a selector that outranks `button:not(.clickable-icon)`, for
  example `button.bob-emoji-slot-well` (0,1,1, later in the cascade) or
  `.bob-emoji-slot-fused > button.bob-emoji-slot-well`. Set explicitly:
  `background-color`, `box-shadow: none`, `padding`, `width`, `height`, `border`,
  `color`, `cursor: pointer`.
- Make it look like a sibling of the label input in the fused field. Read the label
  input's computed background and border in real Obsidian, and match them with the same
  CSS variables (likely `--background-modifier-form-field` and
  `--background-modifier-border`).
- **Empty state** (`.is-empty`): a 1px **dashed** border in a visible color, for example
  `--text-muted` or a `color-mix` of `--interactive-accent`, never
  `--background-modifier-border`. The ☻ glyph uses `--text-muted`, not `--text-faint`.
  Add a small accent "+" corner badge with `::after` so it reads as "add an emoji".
- **Filled state:** a solid border and the glyph in `--text-normal`.
- **Hover, `:focus-visible`, and `[aria-expanded="true"]`:** an `--interactive-accent`
  border plus a visible focus ring, for example
  `box-shadow: 0 0 0 2px color-mix(in srgb, var(--interactive-accent) 35%, transparent)`.
- **Disabled:** dimmed, with no hover ring.
- Check the fallback `<input class="bob-emoji-slot-fallback">` (shown when Bob Emoji v1
  is missing) against Obsidian's input rules the same way. Fix it only if it is
  similarly overridden.
- Use Obsidian CSS variables only. The stylesheet-contract tests forbid hex colors,
  including `var(--x, #hex)` fallbacks. Verify light and dark themes.

### 2. Teach the gesture (`242g-plugin-emoji-slot-dom.js`)

- Give the well a `title` tooltip, for example "Choose an emoji: click, or Tab here and
  type to search". Keep the existing `aria-label` semantics.
- Add a muted one-line **hint** element (for example `.bob-emoji-slot-hint`) inside the
  fused field, below the label and preview chip. Example text: "Emoji: click ☻ or press
  Tab, then type to search".
  - Show it only when the slot is not the fallback, not disabled, the glyph is empty,
    and the panel is closed.
  - Hide it with the `hidden` attribute, not empty text, so it takes no space.
  - Update visibility in `albumEmojiSlotUpdatePreview`, `albumEmojiSlotOpen`, and
    `albumEmojiSlotClose`, so picking, clearing (Backspace on an empty query), opening,
    and closing all refresh it.
  - Give it a grid placement that never overlaps the preview chip, the digit-keycap
    note, or the panel.
- The hint must be true in every host. Today Tab from the label reaches the well in all
  three: by DOM order in the album card and the Bank setup stage, and through the label
  → emoji → group → hotkey cycle in the Category card's `routeNewCategoryKey` (`248g`).
  Typing a printable character on the well opens the panel with that character as the
  query. Do not change those key paths. If any host turns out not to satisfy the hint,
  adjust the hint wording, not the key routing.
- Do not auto-open the panel on focus. Tab out of the search closes the panel and
  refocuses the well, so auto-open would trap Tab.

### 3. Tests

- **Unit** (in the existing Emoji Slot suites
  `scripts/test-ledger-tools-album-schema-card-shell.cjs` and
  `scripts/test-ledger-tools-bank-emoji-slot.cjs`):
  - The hint shows for an empty, enabled, closed slot.
  - It hides after a pick, while the panel is open, for a pre-filled glyph, for a
    disabled slot, and in fallback mode.
  - It reappears after Backspace clears the glyph.
  - The well has its `title`.
  - Cover album choice-add, album field-add initial choices, Bank setup new category,
    and Category card new category.
- **CSS contract:** a test that reads `plugins/bob-ledger-tools/styles.css` and asserts:
  - a well rule whose selector outranks `button:not(.clickable-icon)` sets
    `box-shadow: none`, a background, `padding`, and `width`;
  - the empty-state rule sets a dashed border;
  - no hex color appears in the new rules.
- **Real-Obsidian e2e** (`scripts/obsidian-album-ui-e2e.cjs` and
  `scripts/obsidian-bank-ui-e2e-setup.cjs`), next to the existing emoji probes:
  - **Style probe:** on an empty well, `getComputedStyle` shows `box-shadow` `none`, a
    `dashed` top border style, and a background equal to the fused label input's
    computed background, not `--interactive-normal`.
  - **Keyboard probe (album card)** with real CDP keys, using the
    `Input.dispatchKeyEvent` pattern from `scripts/obsidian-bank-ui-e2e.cjs`:
    - focus the choice label, press Tab, and assert `document.activeElement` is
      `[data-emoji-well]` and the hint is visible;
    - type `r` and assert the panel is open with search value `r`;
    - press Esc and assert the panel closed, the stage is still open, and the label text
      is unchanged.
  - **Screenshots:** empty well with hint, and focused well, in light and dark.
  - Run both e2es on a host with Obsidian and Xvfb, as in `docs/develop/validation.md`.
    For example:
    `BOB_BIN=<bob> xvfb-run -a node scripts/obsidian-album-ui-e2e.cjs -e /snap/bin/obsidian -r /tmp/album-schema.json`
    and `xvfb-run -a node scripts/obsidian-bank-ui-e2e.cjs`.
  - If an e2e cannot run, say so in the final report. Do not claim it passed.

### 4. Docs, version, deploy

- Update the Emoji Slot paragraph in `docs/albums.md` ("type the label, Tab to the
  square glyph well…"). It should say the dashed ☻ tile left of the label is the emoji
  box: click it, or Tab to it and type to search. Mention the hint.
- Touch the bank line in `docs/bob-ledger-tools.md` only if it needs to change. Keep
  every capped doc at or below its current word count. `npm run docs:check` enforces the
  caps and the generated inventory.
- Bump `plugins/bob-ledger-tools/manifest.json` by one patch version from whatever
  `HEAD` holds (1.127.3 when this plan was written). Then run `npm run build`,
  `npm run build:check`, the touched suites, and `npm run docs:check`.
- Run `npm test`. The four pre-existing red files (bob-cli-ju, bob-cli-md, bob-cli-me,
  bob-cli-m8) may stay red. Any new failure must be fixed.
- Deploy with `bob plugins sync`. If sync refuses to overwrite another checkout's deploy
  of an unrelated plugin, follow the refusal protocol in `AGENTS.md` and
  `docs/develop/deployment.md`. Never pass `-F` on your own. Syncing only
  `bob-ledger-tools` (`-p bob-ledger-tools`) is fine.

> [!decision] suggest_followup = file Use `/sase_new_task` to check for duplicates, then
> file one `large` task bead in bob-cli. Problem: bob-emoji `suggestForLabel` ranks
> noisy prefix matches first. Measured with the vendored catalog:
>
> - "How I use SASE" → 👤 👥 😐 😒 🤢 😵‍💫
> - "foo" → 🦶 👣 🏈 😋 🦵 🥭
> - "Dining out" → ↘️ ⬇️ ↙️ 😵 📤 ☺️
> - "Draft", "Groceries", and "Conclusion" → nothing
> - Good matches: "Coffee" → ☕, "Memory" → 🧠
>
> The panel's first "For “<label>”" row shows these picks. Relate it to the closed epic
> bob-cli-mf, where api v1 and the CLDR keywords landed. No code change in this tale.

> [!decision] suggest_followup = skip File nothing about suggestion quality.

## Out of scope

- A `:shortcode` trigger inside label inputs, and an always-visible inline suggestion
  strip. Label suggestions are too noisy today to surface without opening the panel.
- Changing bob-emoji search or ranking. That is the optional follow-up above.
- Album-level emoji (bob-cli-ml). It will reuse this well and inherit the fix.
- Any bob-cli code change. This is plugin CSS and DOM only.

## Acceptance

- In real Obsidian, light and dark, the empty well beside a choice or category label
  reads as an input-like dashed tile with a muted ☻ and a "+" badge, not a gray button.
  Its computed background matches the label input, and it has no Obsidian button shadow.
- The hint appears under an empty slot in all three hosts. It disappears once an emoji
  is picked or the panel is open, and comes back when the emoji is cleared.
- Label → Tab → type `r` opens the panel with query `r` in real Obsidian. Esc closes
  only the panel. Save/apply paths, Enter isolation, and the Esc ladder are unchanged.
- Build, build:check, the touched suites, docs:check, and the e2e probes pass, or each
  skipped e2e is reported as skipped. `bob-ledger-tools` is deployed via
  `bob plugins sync`.
