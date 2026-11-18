---
tier: tale
title: Schema card field and group jumps on Ctrl+J/K
goal:
  On the album Schema Card and the bank Setup card, Ctrl+J/Ctrl+K do the field and group
  jumps that Shift+J/K do today.
size: small
decisions:
  bank_uppercase_hotkeys:
    ask:
      Should bank Setup category hotkeys match uppercase letters again now that
      Shift+J/K no longer jump?
    choices:
      keep:
        Shift+letter and Caps Lock letters select nothing, as the bank docs promise
        today
      restore: Pre-epic case-insensitive matching returns; Shift+K selects 🚗 Car again
    default: keep
    why:
      bob-cli stores hotkeys as lowercase [a-z0-9], and keeping Shift+letter free costs
      nothing
    answer: keep
proposed_by: bbugyi200.apollo.eu
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-17 20:50:40
status: wip
---

# Plan: Move the schema-card field and group jumps from Shift+J/K to Ctrl+J/K

## Context

Epic **bob-cli-mu** (closed) added two sets of keys:

- On the album **Schema Card**, `Shift+J`/`Shift+K` jump to the next/previous field row
  in painted order.
- On the bank **Setup card**'s Categories tab, `Shift+J`/`Shift+K` jump between category
  groups.

Bryan wants both rebound to **`Ctrl+J`** (next) and **`Ctrl+K`** (previous). Before you
start, read the epic and its plan for the shared design:

```bash
sase bead read bob-cli-mu -r "Design of the field/group jumps being rebound to Ctrl+J/K"
sase artifact read plan:202611/schema_card_field_jumps.md "Field/group jump design being rebound"
```

**Why Ctrl.** Bryan's vault binds `Ctrl+J`/`Ctrl+K` to bob-navigation-hotkeys
`jump-to-next-section-header` / `jump-to-prev-section-header` in the editor
(`~/bob/.obsidian/hotkeys.json`). To him, `Ctrl+J/K` already means "next/previous
heading". Field rows and bank group labels are the cards' headings.

**Only the chord changes.** Everything else from the epic stays exactly as it is:

- painted order and parent-first `Ctrl+K` from a chip, `+ choice`, or tile
- clamped ends with the 3 px bump and announcements (`No next field`,
  `Claim, field 2 of 3`, `Bills, group 3 of 4`, and so on)
- silent `event.repeat` clamps and reduced-motion gating
- the `1`–`9` digit jumps and the lowercase `j`/`k`/arrow fine walk
- `Alt+J/K` arrange
- the gates (stages, inputs, IME composition, `applying`, the read-only card)
- jumps never write

**Where the code lives.** All code is in the linked **bob-plugins** repo. Open it with
`/sase_repo` (`sase repo open bob-plugins -r "<why>"`) and read its `AGENTS.md`. Paths
below are relative to that repo, and `F/` means `plugins/bob-ledger-tools/src/`.

- Edit `src/` fragments only. Never hand-edit `main.js`.
- Keep every fragment at or under 1000 lines. Several are close: `241y` is 973 lines,
  `241yb` 934, `241yc` 958, and `248k` 948. Change conditions in place rather than
  adding blocks.
- Ship with `npm run build`, then `npm test`, then `npm run docs:check`, then
  `bob plugins sync`. If sync refuses, follow AGENTS.md's refusal protocol, which never
  uses `-F` on your own.

**Out of scope.** This plan changes no bob-cli code, no CLI/JSON contract, no memory
note, and nothing on the release train. No memory note or bob-cli doc mentions these
keys. The Record Card (`F/242p`) has no field jumps and is untouched.

## The chord rule (both cards)

A field or group jump fires on exactly this test:

```js
event.ctrlKey &&
  !event.metaKey &&
  !event.altKey &&
  !event.shiftKey &&
  (lower === "j" || lower === "k"); // lower = String(event.key || "").toLowerCase()
```

`j` is +1 (next) and `k` is −1 (previous).

- **Literal Control on every platform, macOS included.** `⌘J`/`⌘K` are not aliases. This
  matches the vault's literal `Ctrl` section-header binding, the album card's `Ctrl+D`,
  and the Record Card's Ctrl keys.
- **Caps Lock still counts.** Ctrl+J with Caps Lock on arrives as `key: "J"` with
  `shiftKey: false` and still steps. That is why the test lower-cases the key.
- **`Ctrl+Shift+J/K` stays unbound on both cards.** That chord is the editor's open-task
  jump.
- **No `event.code` fallback.** This matches the epic's rule for `j`/`k`.
- **Bare `J`/`K` (Shift+J/K) become inert.** They get no branch and are not consumed.

**Ordering matters.** Today a real `Ctrl+K` arrives as `key: "k"` and falls into a
branch with no modifier gate:

- on the album card, the `j`/`k` fine walk (`F/241y` ~886);
- on the bank card, the category hotkey branch (`F/248k` ~797), which selects 🚗 Car.

Each new Ctrl branch must sit **before** those branches, in the spot where the `J`/`K`
branch is now, so the jump wins.

## Steps

1. **Album card keys (`F/241y-plugin-album-schema-card.js`, `schemaOnKeyDown` ~876).**
   - Replace the `event.key === "J" || "K"` branch with the chord rule. Call
     `this.schemaFieldStep(lower === "j" ? 1 : -1, event)` after `preventDefault`.
   - Keep the branch where it is: after the Alt branches and before the digit and
     fine-walk branches.
   - Leave the digit branch unchanged.

2. **Album hint mode (`F/241yb-plugin-album-schema-arrange.js` ~870).** Hint mode owns
   the keyboard. Today its swallow branch requires no Ctrl, so `Ctrl+J/K` would fall
   through to the base handler and jump while gap badges are showing.
   - While `this.schemaHintMode` is set (and there is no stage and no focused input),
     also `preventDefault` and return on the chord.
   - Keep swallowing digits and `J`/`K`/`j`/`k` exactly as today.

3. **Album hint strip (`F/241yc-plugin-album-schema-card-regions.js` ~795).**
   - Change the keys to
     `["⌃J/K", n === 1 ? "1" : \`1–${n}\`]`. This uses the same `⌃`glyph as the strip's`⌃D`.
   - Placement and absence rules stay the same.

4. **Bank card keys (`F/248k-plugin-bank-setup-card.js`, `onKeyDown` ~793).**
   - Replace the `J`/`K` branch with the chord rule, for `this.section === "categories"`
     only. Call `consume()` and then `this.jumpGroup(lower === "j" ? 1 : -1, event)`.
   - Keep it in the same spot, before the hotkey branch.
   - On Accounts, Rules, and Checks the chord does not jump.
   - The card's modal-scope catch-all (`F/247` `onOpen`) still consumes every key, so
     nothing leaks to Obsidian hotkeys.
   - Leave the hotkey regex alone unless the decision below says otherwise.

5. **Bank hint mode (`F/248ka-plugin-bank-setup-card-arrange.js` ~743).**
   - In the unmatched `j`/`k` swallow, replace `!mod` with `!event.metaKey`. Hint mode
     then also swallows `Ctrl+J/K` and never jumps or hotkeys.
   - The hint-letter branch above it stays modifier-free.

6. **Bank footer (`F/248kb-plugin-bank-setup-card-regions.js` ~247).**
   - Change the item to `{ keys: "⌃J/K", label: "group" }`.
   - Use the literal `⌃` glyph, not `` `${mod}+` ``. The chord is literal Ctrl even on
     macOS, where `mod` renders as `⌘`.

7. **Comments.** Wherever a code comment says Shift+J/K, make it say Ctrl+J/K:
   - `F/241yf` header
   - `F/241xb` ~301–302 (`schemaCardBump` direction note)
   - `F/242c` ~74
   - `F/248i` ~180
   - `F/248k` ~866

> [!decision] bank_uppercase_hotkeys = restore
>
> Change the hotkey test at `F/248k` ~797 back to `/^[a-z0-9]$/i`.
> `bankSetupCardSelectionByHotkey` already lower-cases. Drop "hotkeys match lowercase
> only" from `docs/bob-ledger-tools.md`. Flip the unit test so Shift+K selects 🚗 Car
> and an uppercase letter selects its category. With `keep`, change none of this.

8. **Unit tests.**
   - **`scripts/test-ledger-tools-album-schema-field-jump.cjs`:**
     - Move every jump press from `{ key: "J" | "K" }` to
       `{ key: "j" | "k", ctrlKey: true }`. Keep every row of the step table and its
       messages, renamed to Ctrl.
     - Caps Lock case: `{ key: "J", ctrlKey: true }` steps.
     - New inert cases, where the selection stays and nothing is announced:
       - `{ key: "J" }` and `{ key: "J", shiftKey: true }`
       - `{ key: "j", ctrlKey: true, shiftKey: true }`
       - `{ key: "j", metaKey: true }`
     - Invert "Ctrl+J does nothing" in the Alt/Ctrl test. Ctrl+J now lands on the next
       field, with `schemaArrangeDirty()` false and zero draft ops.
     - Move the gate, no-writes, and feedback (bump, `event.repeat`, reduced motion)
       cases to the Ctrl chord.
     - Hint mode swallows `Ctrl+J`, and hint mode survives.
     - The hint strip expects `["⌃J/K", "1–3"]`.
   - **`scripts/test-ledger-tools-bank-setup-group-jump.cjs`:**
     - Move every jump press to the Ctrl chord.
     - Rewrite the "Shift+K never fires a hotkey" test:
       - Real `Ctrl+K` (`{ key: "k", ctrlKey: true }`) from Coffee jumps to
         `group:Everyday`, not Car.
       - `k` still picks Car, and `1` still picks Rent.
       - Under `keep`, `{ key: "K" }` and `{ key: "C" }` select nothing.
     - Inert tabs and gates use the Ctrl chord.
     - Hint mode swallows `Ctrl+J` and survives.
     - The footer item is `{ keys: "⌃J/K", label: "group" }`, and it is absent on
       Accounts.

9. **Real-key e2e.**
   - **`scripts/obsidian-album-ui-e2e.cjs` (~1576).**
     - Send Ctrl+J as `Input.dispatchKeyEvent`
       `{ key: "j", code: "KeyJ", windowsVirtualKeyCode: 74, nativeVirtualKeyCode: 74, modifiers: 2 }`,
       with no `text`.
     - The synthetic fallback dispatches
       `{ key: "j", code: "KeyJ", ctrlKey: true, bubbles: true }`.
     - Rename the record to `schema-field-jump-ctrl-j`.
     - In `buildAuthoredFixture`, write `.obsidian/hotkeys.json` that mirrors the vault:
       `bob-navigation-hotkeys:jump-to-next-section-header` →
       `[{ "modifiers": ["Ctrl"], "key": "J" }]`, and `…prev…` → `K`. This fixture
       enables every repo plugin, so the probe reproduces Bryan's real conflict.
     - Report whether the real key (not only the fallback) moved the selection.
     - If a Markdown editor is active behind the card, record its cursor before and
       after and assert it did not move.
   - **`scripts/obsidian-bank-ui-e2e-setup.cjs` (~1391).**
     - Send `Ctrl+K`, then `Ctrl+J`, the same way (`modifiers: 2`, lowercase key, no
       text, synthetic fallback with `ctrlKey`).
     - Rename the records to `setup-group-jump-ctrl-k` and `setup-group-jump-ctrl-j`.
   - Keep both `--self-test`s and their `scripts/test-obsidian-*-e2e*.cjs` wrappers
     green.
   - Run the real e2e under `xvfb-run -a` wherever Obsidian is installed. It is
     installed on athena; see the tailnet memory for access. If you cannot run it, say
     so.
   - **Contingency.** Both cards are default Obsidian `Modal`s, whose scope has no
     parent, so vault hotkeys should not fire while they are open. If the real-key probe
     shows otherwise, register the chord on the album modal's `this.scope` the way
     `bobCardWireCloseKeys` (`F/241xc`) does, and re-run. The two signs are that the
     card never sees `Ctrl+J/K`, or that the editor behind it moves.

10. **Docs.**
    - **`docs/albums.md`:**
      - ~89: "so `Ctrl+K` from a chip is visible".
      - ~94: the strip includes `⌃J/K 1–N field`.
      - ~195–196: `Ctrl+J`/`Ctrl+K` jump to the next/previous field (from a chip,
        `Ctrl+K` first returns to its own field, and jumps stop at the ends).
    - **`docs/bob-ledger-tools.md` ~79:** "Ctrl+J/K jump between groups".
    - Keep word counts flat so `npm run docs:check` budgets hold.

11. **Verify and ship.**
    - Run `npm run build`, then the focused suites, then `npm test`, then
      `npm run docs:check`, then `bob plugins sync`. The focused suites are the field
      jump, bank group jump, album shell/arrange/place-drag, bank arrange, and both e2e
      self-tests.
    - Known load-timing reds that are unrelated to this change: event-link-times,
      frees-marks, keep-moving-marks(-surface), and off-clock-link-chips. Report them as
      unrelated rather than "fixing" them.

## Acceptance (live vault, after sync)

- **Zettel album** (`==` on `sase_blog_zettel.md`):
  - From Status, `Ctrl+J` goes to Claim, and `3` still goes to Section.
  - From a Section chip, `Ctrl+K` goes to Section, and its keycap lights first.
  - `Shift+J` and `Ctrl+Shift+J` do nothing.
  - The note behind the card does not jump headings.
  - The hint strip reads `⌃J/K 1–3 field`.
- **Bank** (`==` on `bank.md`):
  - `Ctrl+J` walks Money moves → Everyday → Bills → HIDDEN.
  - From a tile, `Ctrl+K` lands on its own group.
  - `k` still picks Car, and `1` still picks the first Bills category.
  - `Shift+K` selects nothing (or Car under `restore`).
  - The footer shows `⌃J/K group`.
