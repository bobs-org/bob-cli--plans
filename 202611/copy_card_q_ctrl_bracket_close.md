---
tier: tale
title: "Copy card: q and Ctrl+[ close the Cmd+Y card"
goal:
  Bare q (and Shift+Q) and Ctrl+[ close the Obsidian Cmd+Y Copy card exactly like Esc,
  with no copy and no write.
size: small
proposed_by: bbugyi200.apollo.di
proposed_with: claude/opus
create_time: 2026-11-12 08:25:42
status: wip
---

# Plan: Copy card closes on `q` and `Ctrl+[`

## Goal

The `Cmd+Y` Copy card (Obsidian, plugin `bob-navigation-hotkeys`) closes only on `Esc`
today. Make bare `q` (and `Shift+Q`) and `Ctrl+[` close it too, exactly like `Esc`: no
copy, no clipboard change, no vault write. This matches the Task Card and the image zoom
viewer, which already close on `Esc` / `q` / `Ctrl+[` through the shared
`isTaskCardCloseKeydown` helper.

## Where the work lives

All code, tests, and docs are in the **`bob-plugins` linked repo**. Open it with
`/sase_repo` (`sase repo open bob-plugins -r "..."`) and read its `AGENTS.md` first.
`bob-navigation-hotkeys` uses the committed source build: edit
`plugins/bob-navigation-hotkeys/src/*.js` fragments, run `npm run build`, and never
hand-edit the generated `main.js`. Each hand-edited fragment must stay at or below 1000
lines (`101-copy-card-model.js` is about 955 lines, so keep additions small). The
bob-cli repo needs no change.

## Root cause

- `src/103-copy-card-modal.js` registers a catch-all scope handler
  (`scope.register(null, null, …)`). `handleCopyCardKey` calls `preventDefault` and
  `stopPropagation` on every key, then acts only on the action returned by
  `resolveCopyCardKey`.
- `resolveCopyCardKey` in `src/101a-copy-card-keys.js` returns `close` only for an
  unmodified `Escape`.
  - `Ctrl+[` falls into the "any Ctrl/Alt/Meta chord → `null`" branch. The modal
    swallows it and nothing happens.
  - `q` reaches row-key matching. No built-in row owns `q`, so it returns `null` and is
    swallowed. A `copy.templates` entry with `key: q` would copy instead.
- The shared helpers already exist in `src/470-keydown-and-freshness.js`:
  `isCtrlLeftBracketKeydown` and `isTaskCardCloseKeydown`. The second accepts unmodified
  `Escape`, `q` / `Q` with no Ctrl, Meta, or Alt, and exact `Ctrl+[` (by
  `code === "BracketLeft"` or `key === "["`). Fragments share one script scope and
  function declarations are hoisted, so a fragment can call a function declared in a
  later fragment. `src/207-image-zoom-modal.js` already calls `isTaskCardCloseKeydown`
  this way.

## Design

1. **Key resolution** (`src/101a-copy-card-keys.js`, `resolveCopyCardKey`):
   - Keep the existing order: busy → consume, composition → `null`, repeat → `null`,
     bare modifier → `null`, then the `Escape` branch (unchanged, so `Alt+Escape` and
     `Shift+Escape` still do nothing).
   - Right after the `Escape` branch, before the Mod+Y, arrow, `Ctrl+N` / `Ctrl+P`,
     Enter, and row-key checks, add
     `if (isTaskCardCloseKeydown(event)) return copyCardKeyAction("close");`. That
     gives:
     - `q` and `Shift+Q` close. This is the one exception to "Shift+letter is ignored",
       and it matches the Task Card.
     - `Ctrl+[` closes. `Ctrl+Shift+[`, `Alt+[`, `Meta+[`, and `Ctrl+Alt+[` still do
       nothing.
     - While a copy is in flight (busy), `q` and `Ctrl+[` are consumed, the same as
       `Esc` is today.
     - Auto-repeat and IME composition are still ignored.
2. **`q` is reserved for close** (`src/101-copy-card-model.js`):
   - Add `const COPY_CARD_CLOSE_KEY = "q";`.
   - In `normalizeCopyCardTemplates`, seed the `owners` map with
     `{ kind: "close", label: "Close", index: -1 }` for `COPY_CARD_CLOSE_KEY`.
   - Generalize the `others` filter so non-template owners always count
     (`owner.kind !== "template" || owner.index !== template.index`).
   - When a template collides with the close owner, its reason is
     `key q closes the card`. Precedence for the reason: builtin, then close, then other
     template.
   - A template with an explicit `key: q` becomes a `collision` row with `key: null`.
     Like any collision row, it stays copyable by `↑↓ ↵` or click. So `q` never changes
     meaning based on config, the same principle that already reserves `P A F N I`.
     Bryan's current config uses only `key: t`, so nothing he uses changes. Positional
     keys are digits and never reach `q`.
   - Leave `COPY_CARD_RESERVED_KEYS` unchanged. It means "built-in row keys", and a
     model test pins it to `["p","a","f","n","i"]`. Export `COPY_CARD_CLOSE_KEY` through
     `src/717-copy-card-exports.js` so tests can name it.
3. **Footer hint** (`src/103-copy-card-modal.js`, `renderFooter`): change the close chip
   from `{ keys: ["esc"], label: "close" }` to `{ keys: ["esc", "q"], label: "close" }`.
   This renders two `kbd` caps. `Ctrl+[` is left out of the footer, the same way
   `Ctrl+N` / `Ctrl+P` are left out next to `↑↓`, and `docs/copy-card.md` documents it.
4. **No other modal changes.** `handleCopyCardKey` already turns `close` into
   `settled = true; this.close()`. The keydown target is fixed when the event is
   dispatched, and the event is `preventDefault`ed. So a `q` keydown that closes the
   card cannot also reach the CodeMirror editor (for example, start a Vim macro
   recording). This is the same path the Task Card's `q` already uses.

## Tests

- `scripts/test-navigation-copy-card-model.cjs`:
  - `q`, `Q` (with and without `shiftKey`), `Ctrl+[` given as `key: "["`, and `Ctrl+[`
    given as `code: "BracketLeft"` with another `key` all resolve to
    `{ type: "close" }`.
  - These return `null`: `Ctrl+Shift+[`, `Alt+[`, `Meta+[`, `Ctrl+q`, `Meta+q`, `Alt+q`,
    a repeated `q`, and a composing `q`.
  - Extend "busy consumes every key": `q` and `Ctrl+[` → `consume`.
  - A template with `key: "q"` (and `key: "Q"`) is a `collision` with `key: null` and
    reason `key q closes the card`, and pressing `q` with that model still closes.
  - Two templates both on `q` both collide with the close reason.
  - Existing `p`-collision reasons are unchanged.
- `scripts/test-navigation-copy-card-view.cjs`:
  - Next to "Esc closes without writing", add the same check for `q` and for `Ctrl+[`
    (both the `key: "["` and the `code: "BracketLeft"` forms): the modal closes and
    `onCopy` is never called.
  - Footer test: the footer text contains both `esc` and `q` before `close`.
- `scripts/obsidian-copy-card-e2e.cjs` (real Obsidian over CDP):
  - Let `keyDispatchEvents` take optional `code` (already supported) and `vk` overrides.
    The derived VK for `[` is 91 (VK_LWIN), which is wrong.
  - Add two scenarios modelled on `escape-closes-without-copy`:
    - `q-closes-without-copy`: `{ key: "q" }`.
    - `ctrl-bracket-closes-without-copy`:
      `{ key: "[", code: "BracketLeft", vk: 219, ctrl: true }`.
    - Each seeds a clipboard sentinel, opens the card, presses the key, waits for the
      card to close, and asserts the sentinel is unchanged.
  - Add an offline `--self-test` assertion that the `Ctrl+[` dispatch events carry
    `code: "BracketLeft"`, `windowsVirtualKeyCode: 219`, and the Ctrl modifier bit.
  - The existing `scratch-note-bytes-unchanged` check covers "no vault write".
  - The real run needs `/snap/bin/obsidian` (or `-e PATH`) and `xvfb-run`. If they are
    unavailable on the host, run only the self-test and say so plainly in the final
    report. Do not claim a real-Obsidian pass.

## Docs

`docs/copy-card.md` (plugin reference, mounted into the Bob docs site), Keys section:

- Add a close row or sentence: `Esc`, `q`, and `Ctrl+[` close the card without copying.
- Amend "`Shift+letter` is ignored" to note that `Shift+Q` closes like `q`.
- Amend "Other modified chords are ignored except exact `Ctrl+N` / `Ctrl+P`" to also
  list exact `Ctrl+[`.
- State that `q` is reserved like `P A F N I`. In the template-grammar paragraph, add
  the `key q closes the card` collision next to `key p is Path's`.
- Keep the page's prose tight. `docs/bob-navigation-hotkeys.md` needs no change.

## Verification

From the `bob-plugins` checkout:

1. `npm run build`, then `npm run build:check`.
2. `node --test scripts/test-navigation-copy-card-model.cjs scripts/test-navigation-copy-card-view.cjs scripts/test-obsidian-copy-card-e2e.cjs`
3. `npm test`, which includes the build check and the full unit suite.
4. `npm run docs:check`. If no sibling bob-cli checkout is found, point `BOB_CLI_DIR` at
   a bob-cli checkout.
5. If Obsidian and `xvfb-run` are available:
   `xvfb-run -a node scripts/obsidian-copy-card-e2e.cjs --out /tmp/copy-card-e2e`. All
   records must pass, including the two new close records.
6. `bob plugins sync` to deploy to the vault (required by the repo's AGENTS.md). Follow
   AGENTS.md exactly if sync refuses an overwrite. Never pass `-F` on your own.

Commit the `bob-plugins` changes, including the regenerated `main.js`, as one
`fix(copy-card): …` commit through the normal `/sase_final` repo obligation. Do not bump
the plugin manifest version.

## Out of scope

- Renaming `isTaskCardCloseKeydown` to a generic name.
- Changing any other modal's close keys.
- Bob Mac Capture's separate copy surfaces.
