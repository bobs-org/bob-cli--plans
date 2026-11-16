---
tier: epic
title: Schema editors that arrange instantly and pick emoji beautifully
goal: "The album Schema Card and the Bank setup card move fields, choices, categories,
  groups, and rules any distance with an instant local preview, then save the whole
  arrangement as one guarded bob transaction with one commit and one Undo. Every emoji
  prompt is a searchable Emoji Slot. Both cards use a calmer, contextual layout. Bob
  stays the only writer, and nothing is saved without a successful preview of the exact
  envelope.

  "
decisions:
  enter_saves:
    ask: How many Enter presses should it take to save a quiet arrangement?
    choices:
      quiet_once:
        Quiet arrangements save on one ⏎; index-grouping, rule, and hotkey changes need
        a second ⏎
      always_twice:
        Every arrangement shows bob's consequence first and saves on a second ⏎
    default: quiet_once
    why: Reorders are cheap to undo with u; loud cases still stop for confirmation
    answer: quiet_once
  cldr_keywords:
    ask:
      Vendor CLDR English emoji keywords (~150–250 KB) into bob-emoji for much richer
      search?
    default: true
    why: gemoji's thin keywords are the biggest search gap (memory→🧠, find→🔍)
    answer: true
  bank_cross_group:
    ask:
      Let a bank category move into another group as one loud arrangement (category.set
      + category.move)?
    default: false
    why:
      Changing a group also changes filing layout; v1 keeps it an explicit edit in the
      edit stage
    answer: false
  decision_record:
    ask: Add a decisions memory record that one schema arrangement is one commit?
    memory:
      - decisions
    requested:
      I agree with all of the requirements recommended in that research file (but keep
      in mind that you are responsible for planning and splitting up this work
      appropriately).
    default: true
    answer: false
phases:
  - id: quick-fixes
    title: Quick fixes that ship before the redesign
    depends_on: []
    size: small
    description:
      "quick-fixes: stop q from cancelling inside bank inputs, keep a bank category's
      emoji when its label is edited, make Alt+J/K work with macOS Option, and port
      bob's split_icon so album choice splitting matches Rust."
  - id: arrange-engine
    title: Pure arrangement engine
    depends_on:
      - quick-fixes
    size: medium
    description:
      "arrange-engine: add a DOM-free projected-order model with nudges, edges, gap
      placement, local undo, anchor rebase, and a save marker. Add album and bank
      envelope compilers checked by a simulator, plus gap-hint labels and FLIP planning."
  - id: emoji-engine
    title: bob-emoji search upgrades and api v1
    depends_on: []
    size: medium
    description:
      "emoji-engine: normalize multi-token queries with tiered ranking, look up pasted
      glyphs, add catalog categories, CLDR keywords when chosen, and label suggestions.
      Expose a frozen versioned api on bob-emoji that keeps the editor :shortcode: menu."
  - id: album-card-shell
    title: Album Schema Card visual refresh and region painting
    depends_on:
      - arrange-engine
    size: medium
    description:
      "album-card-shell: give the album card one title line, the details row as the
      header subtitle, pills, a one-line status rule, and a contextual hint strip with a
      ⋯ action list. Remove Preview this edit, latch Enter, and repaint regions instead
      of rebuilding the modal."
  - id: album-arrange
    title: Album arrangement editing and one-envelope save
    depends_on:
      - album-card-shell
    size: medium
    description:
      "album-arrange: wire the engine into the album card. Alt+J/K, ends, counts, and m
      move-to change a local projection with FLIP, deltas, and a pending bar. A latched
      quiet/loud Enter saves one envelope in the background, and u steps back or stages
      Undo. The save survives closing the card."
  - id: album-emoji-slot
    title: Shared Emoji Slot in the album card
    depends_on:
      - album-arrange
      - emoji-engine
    size: medium
    description:
      "album-emoji-slot: build the shared Emoji Slot (glyph well, preview chip, and
      in-card panel served by the bob-emoji api, with a plain fallback). Wire it into
      album choice add/edit and field-add initial choices with duplicate badges, IME
      guards, and a strict Esc ladder."
  - id: album-place-drag
    title: Add-and-place ghosts and pointer drag
    depends_on:
      - album-emoji-slot
    size: medium
    description:
      "album-place-drag: show a ghost chip or row for choice-add and field-add that the
      move keys place before the single add op. Add shared pointer drag with grip
      handles, an insertion caret, autoscroll, and Esc cancel for album items."
  - id: bank-arrange
    title: Bank setup card arrangement parity
    depends_on:
      - album-place-drag
    size: medium
    description:
      "bank-arrange: bring arrangement editing to bank categories, groups, and rules
      (always loud) without per-move confirm stages. Paint bob's dry-run order and
      hotkey shifts, check bob's order against the local one, and add region painting,
      Ctrl+Z semantics, and saves that survive closing the card."
  - id: bank-emoji-slot
    title: Emoji Slot for bank categories
    depends_on:
      - bank-arrange
    size: small
    description:
      "bank-emoji-slot: use the Emoji Slot for setup-card new/edit category and the
      category card's new-category stage. Compose the emoji and label into bob's fused
      label, keep the emoji unless the user clears it, and draw suggestions from bank
      vocabulary."
  - id: acceptance
    title: End-to-end acceptance, measurements, and decision record
    depends_on:
      - bank-emoji-slot
    size: medium
    description:
      "acceptance: run real-Obsidian walkthroughs that count commits for both cards,
      plus timing budgets and light/dark/reduced-motion screenshots. Do a final
      consistency pass, write the decision record, and propose follow-ups for CLI
      save-latency trims."
proposed_by: bbugyi200.apollo.research.51.linker.w0
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-16 15:38:34
status: wip
---

- **PROMPT:**
  [prompts/202611/schema_editor_arrange_and_emoji.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/schema_editor_arrange_and_emoji.md)

# Plan: Schema editors that arrange instantly and pick emoji beautifully

## Context

Bryan's album Schema Card (screenshot from 2026-11-16) and Bank setup card are hard to
use in two ways:

- **Emoji prompts are bare text boxes.** Album `Choice emoji` and bank category labels
  (placeholder `e.g. 🍔 Fast food`) expect a glyph pasted by hand. `bob-emoji` already
  ships a 1,870-entry gemoji catalog with recents, but only its Markdown-editor
  `:shortcode:` suggest uses it.
- **Reordering moves one slot per transaction.** `schemaMoveSelected(±1)` computes the
  move against the _saved_ order and `session.setDraft([op])` _replaces_ the draft. A
  second Alt+K before Apply re-stages the same op, and the chips never show the pending
  order. Each slot then costs about 5.5–6.5 s of guarded transaction (pre-sync, plan,
  commit, verify, post-sync, report refresh). The bank card is stricter still: every
  move opens a confirm stage. In vault history, one thought ("put How I use SASE before
  The Demos") took **five commits in 85 seconds**.

Source research (Bryan agrees with all its adjusted requirements; this plan owns the
split):
`research:202611/schema_editor_emoji_and_multislot_reorder/schema_editor_emoji_and_multislot_reorder.md`.

Facts verified while planning:

- bob-cli writers already do everything v1 needs. `bob albums update` applies an
  envelope's ops **sequentially** and returns reverse-ordered `inverse_ops`.
  `field.move`/`choice.move`/`field.add`/`choice.add` take absolute `position`s.
  `bob bank config update` takes `category.move`/`group.move` with `before|after`
  anchors and `rule.move {index, match_was, position}`, where `index` is the rule's
  index _at that op_. Its dry run returns post-image `categories[]` and
  `hotkeys.shifted`, which the plugin validates but never paints. A 14-slot move
  dry-runs as fast as a 1-slot move; distance is free and transactions are what cost.
  **This epic changes no bob-cli code.**
- **Bank label-edit data loss.** Bank `category.set {label}` replaces the _full_ stored
  label (`full_label` = `emoji + " " + label`). The setup card's edit stage is
  pre-filled with the emoji-less `entry.label`, so any rename silently drops the
  category's emoji.
- **macOS Alt chords.** Album and bank Alt+J/K match only `event.key`. On macOS,
  Option+J/K produce `∆`/`˚`, so the chords need an `event.code` (`KeyJ`/`KeyK`) match.
  Other plugins already do this, for example `652-pomodoro-split.js`.
- **`q` closes from inside inputs.** In `248k-plugin-bank-setup-card.js` (`onKeyDown`)
  and `248g-plugin-bank-category-new-stage.js` (`routeNewCategoryKey`), the `q` check
  runs before the in-input guard. Typing "Liquor" cancels the stage.
- **Icon-split mismatch.** Rust `split_icon` (bob-cli `src/native/albums/output.rs`)
  treats a leading ASCII alphanumeric as label text, so `1️⃣ First` has no icon. It also
  needs no space after the icon. JS `albumSchemaChoiceParts` disagrees on both. When a
  keycap choice's label is edited, the JS split can drop the keycap.
- Cross-plugin APIs follow the frozen `this.api` pattern (`task-status-cycler`,
  `bob-command-mode.api.commandMode`), documented in bob-plugins
  `docs/develop/plugin-apis.md`.
- `242aa-plugin-guarded-update-session.js` discards an apply's result when the draft
  revision changes during the apply. **Never call `session.setDraft` while an apply is
  in flight.**
- Both modals rebuild their entire DOM on every session change (`content.empty()`). That
  breaks FLIP, live popovers, caret, and IME composition.

## Design

### The one rule

> **Arrange locally and instantly. Bob validates once. Save the whole arrangement as one
> transaction with one Undo.**

A reorder gesture never writes. It updates a local _arrangement_: the saved order plus
pending moves. The card paints the projected order in one frame. While you pause, the
compiled envelope goes to the existing guarded session, so bob's dry run checks it
speculatively. ⏎ saves the whole arrangement through the unchanged guarded session.

Invariants every phase keeps:

1. Bob is the only writer. Apply runs only after a successful preview of the exact
   envelope (same revision, expect, and generation). The fingerprint/`expect`,
   save-open-editors, one-flight apply, and inverse-op Undo machinery stay as they are.
2. One arrangement is one envelope, one commit, and one Undo. Returning everything to
   the saved order leaves no draft and runs no transaction.
3. Only reorders batch. Add, relabel, toggle, remove, hide, and restore stay one gesture
   and one commit. While an arrangement is pending, those gestures are unavailable. The
   pending bar says "⏎ save or esc revert first", and pressing one of them flashes the
   bar.
4. Identity is always a stable key: field key, choice value, category key, group name,
   or a rule's simulated index plus `match_was`. Never use a DOM index.
5. Never call `session.setDraft` while an apply is in flight. Moves made during a save
   accumulate locally (see Save pipeline).
6. Choices never move between fields. Bank accounts, the HIDDEN section, and the
   built-in Money group's first place never move.
7. No bob-cli changes, and no skipping the index rebuild. Section and `index_by` order
   _is_ grouping.

### Arrangement interaction (both cards)

| Gesture                                                                 | Effect (local, zero bob calls)                                                                                                                       |
| ----------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------- |
| `⌥J` / `⌥↓`                                                             | Move the selected item one slot later (down or right)                                                                                                |
| `⌥K` / `⌥↑`                                                             | Move it one slot earlier (up or left)                                                                                                                |
| `⌥⇧J` / `⌥⇧K`                                                           | Move it to the last or first slot of its scope                                                                                                       |
| count prefix, e.g. `4⌥K` (album only; bank digits are category hotkeys) | Repeat the nudge N times. A count also applies to the next `j`/`k`/arrow selection move                                                              |
| `m` (album) / `⌥M` (both cards)                                         | **Move to…**: every legal _gap_ in the item's scope gets a hint badge. Typing the badge's label drops the item into that gap. `Esc` leaves hint mode |
| drag a grip handle                                                      | Pointer drag with an accent insertion caret. Feeds the same model                                                                                    |
| `⏎`                                                                     | Save the arrangement (see Save pipeline)                                                                                                             |
| `esc`                                                                   | Revert every pending move (nothing was written)                                                                                                      |
| `u` (album) / `Ctrl+Z` (bank)                                           | With moves pending: step back one local move. With none: stage Undo of the last save, _painted as a projected arrangement_                           |

Alt chords match `event.code` as well as `event.key`. Inside stage inputs, the matched
chords call `preventDefault` so macOS never inserts `∆`/`˚`. At a scope edge, a nudge
does nothing except a subtle 1-frame edge bump and a live-region "Already first".

What you see:

- Moved items keep an accent ring and a **delta badge** (`↑4`, `↓2`, relative to the
  saved slot).
- The hint strip becomes a **pending bar**, for example
  `↕ 1 move · 🔍 How I use SASE 15 → 11 · ⏎ save · esc revert · u step back`. With
  several moves it reads `↕ 3 moves · ⏎ save · esc revert · u step back`.
- Position changes animate with **FLIP** in 140 ms, ease-out. With
  `prefers-reduced-motion`, nothing animates. FLIP measures old rects by
  `data-arrange-id` before the body repaints and inverts the new ones after, so it works
  even across a rebuild.
- A polite live region announces "How I use SASE, position 11 of 16".
- Gap hints are small accent keycap badges in the gaps. In chip clouds, the insertion
  caret is a 2 px underline in the gap. Between rows it is a 2 px line. Labels are
  home-row-first (`asdfghjklqwertyuiopzxcvbnm`) and prefix-free. They become two
  characters only when there are more than 26 gaps, and the first keystroke dims
  non-matching badges.

### Save pipeline

- Each change to the arrangement compiles an envelope. A simulator asserts the compiled
  ops reproduce the projection exactly, then the envelope goes to
  `session.setDraft(ops)`. The existing 250 ms debounce previews it speculatively.
- **Enter latches.** If the preview for the current revision is still running, ⏎ arms
  "Saving as soon as bob confirms…" and applies when that revision's preview succeeds.
  Any further move clears the latch. The latch works card-wide, including stage edits,
  so ⏎ is never inert.
- **Quiet vs loud.** An album preview is _loud_ when `group_before !== group_after`, any
  `per_op.newly_invalid > 0`, or `warnings` is non-empty. A bank preview is loud for any
  `rule.move`, a non-empty `hotkeys.shifted`, or warnings. Loud consequences paint amber
  with bob's consequence text, and a second ⏎ (or the **Confirm save** button) applies.

  > [!decision] enter_saves = quiet_once A quiet preview applies on the first ⏎. Only
  > loud previews ask for the second ⏎.

  > [!decision] enter_saves = always_twice Every arrangement paints its consequence on
  > the first ⏎, and the second ⏎ applies. The same rule covers Undo arrangements.

- **Saving runs in the background.** After ⏎, the projected order stays. Moved items get
  a soft shimmer and the pending bar reads `Saving… · 🔍 How I use SASE → 11`. The card
  stays navigable. New moves build a _follow-on_ arrangement on top of the saving
  projection: an engine save marker tracks which moves are in flight, and the session
  gets nothing until the apply settles.
  - On success, the base advances to the saved projection. Follow-on moves then compile
    and preview against the new fingerprint.
  - On failure, the marker clears and every move stays pending against the old base.
- **Success** paints a green status rule, `Saved · u undo`, which fades to neutral after
  about 3 s. After the post-write report lands, the saved order is compared with the
  projection. If they differ, the card shows bob's order and quietly warns
  `bob saved a different order; showing bob's`, and logs both orders to the console.
  This should never happen.
- **Refusal** keeps the arrangement pending and visibly unsaved, with a red status rule
  and the refusal message and hint. `esc` reverts. Stale and timeout refusals keep
  today's documented contract: the retained draft is never silently rebased, and an
  explicit "Re-preview against the updated schema" gesture rebases the arrangement by
  anchors and previews.
- **Closing during a save** is allowed: ⏎ then `esc` closes immediately. The
  plugin-level apply flight finishes. If no card for that album (or bank) is open, an
  Obsidian Notice reports "Album schema saved" or the refusal. The plugin keeps the last
  `{inverseOps, afterFingerprint}` per album stem (and one for the bank), and a reopened
  card offers Undo when its fresh `expect` equals that `afterFingerprint`. This needs a
  small `init.undo` seed option in `createGuardedUpdateSession`.
- **Timing spans.** Store nudge paint, preview, and apply durations (`performance.now`)
  in a small ring buffer on the plugin, for example `albumSchemaTimings` /
  `bankSetupTimings`, for e2e budgets.

Budgets (acceptance criteria): nudge paint p95 < 50 ms with zero writer calls (expect <
16 ms); emoji filter paint p95 < 50 ms; one write transaction and one revalidation per
arrangement; settled dry run p95 ≤ ~1.3 s on the real vault.

### Emoji Slot

One shared component (model + DOM fragments in bob-ledger-tools) used by every emoji
prompt in both cards.

- **Chip field.** A label input with a fused, square **glyph well** as its leading
  adornment, styled like the chip it will produce: `[🔍│ How I use SASE ]`. A live
  **preview chip** shows the exact token bob will store and render. DOM and focus order
  are label, then well (CSS `order` paints the well first), so "type the label, Tab,
  pick" flows naturally. An empty well shows a dotted smiley placeholder.
- **Opening.** Focusing the well (by Tab or click) and pressing ⏎, Space, or ↓ opens the
  panel with suggestions. Typing a printable character on the well opens the panel with
  that character as the query.
- **The panel** sits _inside the card_ below the field (in flow, max height about 240
  px, internal scroll). It is not a floating popover, so there is no second Escape
  scope. Its search input holds focus, and the result grid uses `aria-activedescendant`
  so screen readers announce the highlighted emoji's name.
- **Empty query** shows, in order:
  1. **For “<label>”**: label suggestions from the api.
  2. **In your albums** / **In your bank**: host vocabulary, overlapping label words
     first.
  3. **Recent**: shared with the editor's `:shortcode:` recents.
  4. A row of 9 category tabs from gemoji `category`.
- **Typing filters live** with no debounce (search costs about 1 ms). A footer line
  names the highlighted result: `🔍 :mag: magnifying glass tilted left · ⏎ pick`.
- **Keys.**
  - Arrows and `Ctrl+N`/`Ctrl+P` move the highlight in two dimensions.
  - ⏎ picks the highlight, closes the panel, and returns focus to the well. It never
    also saves the schema edit.
  - Tab leaves without picking.
  - The first Esc closes the panel only.
  - Backspace on an empty query clears the chosen glyph.
- **Search text and chosen glyph are separate state.** Typing `rocket` never becomes the
  icon.
- **Paste and the OS picker** (Ctrl+Cmd+Space on macOS into the search input): a query
  that is exactly one emoji grapheme sets the glyph directly.
- IME composition guards apply throughout. No repaint replaces the panel's input during
  composition.
- **Duplicate badge.** A small dot marks glyphs already used by a sibling (same field's
  choices, or bank categories), and the footer reads "also used by Draft". It never
  blocks.
- **Unicode.** Preserve picked and pasted grapheme clusters byte-for-byte: ZWJ
  sequences, flags, keycaps, and tone modifiers. Never `slice(0,1)` or count code
  points. Never reject glyphs newer than the catalog, and never narrow bob's permissive
  label contract. Splitting stored labels uses the JS port of Rust `split_icon` from
  quick-fixes. Bob reads a leading _digit_ keycap (`0️⃣`–`9️⃣`) as label text, so the
  panel omits those ten glyphs. A pasted digit keycap is accepted with the muted note
  "bob stores a leading digit keycap as label text".
- **Fallback.** Without `bob-emoji` api v1 (plugin disabled or old), the well becomes a
  plain glyph input with the hint "Enable Bob Emoji for search". Apply is never blocked.

### Visual language (both cards)

Use Obsidian CSS variables only, and verify light and dark themes.

- **Header.** One title line (`🗃️ Zettel`), with a dim breadcrumb below it on one line,
  truncated with an ellipsis (full trail in `title`). The album details row
  (`2 zettels · dated notes`) _is_ the header subtitle and the selectable "Album
  details" target, which drops the duplicate stats row.
- **Field rows.** Bold label, then pills: `required` (accent outline, replacing `*`),
  kind (`text`, `choices`, `number 1–5`), and `groups index` / `sub-groups index`
  (muted). The fill meta (`set on 2 of 2`) is muted and right-aligned.
- **Chips.** Emoji, label, and a muted tabular-numeral count pill. Zero counts fade to
  about 45% opacity. `+ choice` is a small dashed chip at the end of the cloud, and
  `+ field` is a dashed full-width row at the end.
- **Selection.** A 2 px `--interactive-accent` ring with a visible focus state.
- **Status line.** One line with a 3 px left rule: faint (idle), accent (previewing),
  green (ready or saved), amber (needs confirm), red (refused). It is hidden when empty,
  so there is no dead tinted bar.
- **Hint strip.** Replaces the 13-button toolbar. It shows only keys valid for the
  current selection as `<kbd>` keycaps with muted labels, in priority order, wrapping to
  at most two lines. A `⋯` button reveals an "All actions" button list so every gesture
  keeps a visible button equivalent (a documented contract).
- **Stage panel.** A contained section with a heading and a label/control grid. The
  `Preview this edit` button is gone because previews are automatic. The buttons are a
  primary **Save** (shows `⏎`, enabled when `canApply`) and **Cancel** (`esc`).
- **Motion.** FLIP and fades last 120–160 ms. `prefers-reduced-motion: reduce` turns
  them off.

## Working in bob-plugins (every phase except the memory part of acceptance)

- Open the repo with `/sase_repo` (`sase repo open bob-plugins`) and read its
  `AGENTS.md`.
- Edit `plugins/<id>/src/` fragments, never generated `main.js`. Each hand-edited
  fragment must stay at or below 1000 lines. `241y-plugin-album-schema-card.js` is
  already 883 lines, so put new logic in **new fragments** registered in
  `src/fragments.json` at the right load position. Fragments share one script scope:
  `const` and `class` declarations must load before their users.
- Register every new test file in `package.json` `test:unit`, which is one long line.
- Bump the plugin's `manifest.json` minor version per feature phase (patch for fix-only
  phases).
- Gate with `npm run build`, `npm test`, and `npm run docs:check`, then deploy with
  `bob plugins sync -p bob-ledger-tools` (or `-p bob-emoji`). Follow AGENTS.md exactly
  if sync refuses an overwrite: never `-F` on your own.
- The ledger-tools phases are deliberately **chained**. Each one edits the same card
  fragments, the generated `main.js`, and the same deployed plugin, so only one checkout
  edits and deploys bob-ledger-tools at a time. `emoji-engine` touches only `bob-emoji`
  and runs in parallel.
- Update the plugin reference docs your phase makes stale: `docs/albums.md` (Schema
  Card), the Setup paragraph in `docs/bob-ledger-tools.md`, `docs/bob-emoji.md`, and
  `docs/develop/plugin-apis.md`. bob-cli's `docs/reference/plugins/` pages are mounted
  copies owned by bob-plugins; never hand-edit them.
- Visual phases capture screenshots with the real-Obsidian harnesses
  (`xvfb-run -a node scripts/obsidian-album-ui-e2e.cjs …`,
  `scripts/obsidian-bank-ui-e2e.cjs`; see `docs/develop/validation.md`) in light and
  dark themes. Review them before landing.

## Phase quick-fixes: Quick fixes that ship before the redesign

Files: `248k-plugin-bank-setup-card.js`, `248g-plugin-bank-category-new-stage.js`,
`248i-plugin-bank-setup-card-model.js`, `241y-plugin-album-schema-card.js`,
`242c-plugin-album-schema-edit.js`, and tests in
`scripts/test-ledger-tools-bank-setup.cjs`,
`scripts/test-ledger-tools-bank-category-card*.cjs`, and
`scripts/test-ledger-tools-album-header.cjs`.

1. **`q` only outside inputs.** In `BankSetupCardModal.onKeyDown` and
   `routeNewCategoryKey`, plain `q`/`Q` cancels or closes only when focus is not in an
   input, select, or textarea. `Escape` keeps working everywhere. Add regression tests
   that type `Liquor` and `Equipment` into each stage input and assert the stage
   survives and the value is intact.
2. **Bank edits keep the emoji.** `bankSetupCardSetCategoryOp` sends
   `set.label = "<entry.emoji> <typed label>"` when the category has an emoji and the
   typed label has no leading icon (by the `split_icon` port). Unchanged-label detection
   still compares against `entry.label`. Test: renaming `☕ Coffee` to `Coffee shop`
   sends `☕ Coffee shop`, and an unchanged label sends nothing.
3. **Alt chords on macOS.** Album and bank `Alt+J`/`Alt+K` also match
   `event.code === "KeyJ" / "KeyK"`. Test with
   `{ altKey: true, key: "∆", code: "KeyJ" }`.
4. **One icon splitter.** Add `bobSplitIcon(label) → { emoji, label }`, an exact JS port
   of Rust `split_icon` in bob-cli `src/native/albums/output.rs`:
   - trim the label;
   - a first char that is ASCII alphanumeric means no icon;
   - otherwise the icon is the run up to the first whitespace or ASCII alphanumeric;
   - both parts are trimmed, and an empty icon means no icon.

   `albumSchemaChoiceParts` uses it. Add a shared fixture table that the Emoji Slot
   phases reuse: `✅ Done`, `👩🏽‍💻 Dev`, `🇺🇳 UN`, `1️⃣ First` (no icon), `#️⃣ Tag`, `🌱Seed`
   (no space), `★ Star`, `(x) Foo`, a plain label, and the empty string. Check
   expectations against the Rust function's code; the port must agree.

5. Bump bob-ledger-tools by one patch version, then build, test, and sync.

## Phase arrange-engine: Pure arrangement engine

Add a new DOM-free fragment in bob-ledger-tools that loads before the album and bank
card fragments, for example `241xa-plugin-schema-arrange-model.js`. Add a small
DOM-helper fragment for FLIP planning only if the pure parts need a home; the runner
lands in album-arrange. Add `scripts/test-ledger-tools-schema-arrange.cjs`.

The model:

- `createArrangement({ scopes, fixed })`. `scopes` maps `scopeId → ids[]` in saved
  order: album `fields` and `choices:<fieldKey>`; bank `groups`, `group:<name>`, and
  `rules`. `fixed` marks immovable ids or slots, such as the Money group's first place.
- Gestures:
  - `nudge(scope, id, delta)` → `{ ok, clamped }`;
  - `toEdge(scope, id, "first"|"last")`;
  - `placeAtGap(scope, id, gap)`, where a gap index counts the projected order;
  - `legalGaps(scope, id)` excludes the two no-op gaps next to the item and any fixed
    slots.
- Queries: `projected(scope)`, `delta(scope, id)` (signed slots against the saved
  order), `dirty()`, `movedIds()`, `summary()` (for the pending bar).
- `stepBack()` pops one gesture from the local undo stack. `revert()` clears every move.
  Coalesce key-repeat nudges into one undo step per held key.
- `rebase(newScopes)` → `{ dropped }`. Each move records an anchor: the item's projected
  predecessor, or `start`. On rebase, start from the new saved order, drop vanished ids,
  and re-insert moved items in gesture order after their anchor. When the anchor
  vanished, use the nearest surviving earlier item in the old projection.
- Save marker: `beginSave()` snapshots the projection being saved, `settleSave(true)`
  makes the snapshot the new base and keeps later moves pending, and `settleSave(false)`
  clears the marker.

The compilers:

- `compileAlbumArrangement(arr)` → `field.move {field, position}` and
  `choice.move {field, value, position}` ops.
- `compileBankArrangement(arr, { categories, rules })` →
  `category.move {key, before|after}`, `group.move {group, before|after}`, and
  `rule.move {index, match_was, position}`. Each rule op carries the rule's index _at
  that op_ from the simulation and its match text.
- **Minimal moves.** The moved set is the complement of a longest increasing subsequence
  of saved indices in projected order, so the op count is n − LIS. Each moved item is
  placed relative to a neighbour that is already final.
- `simulateAlbumOps` and `simulateBankOps` replay ops with the writers' semantics. For
  bank, simulate a flat ordered category list where `group.move` moves a group's block
  in order. Each compiler asserts that `simulate(compile(arr))` equals the projection
  and otherwise returns `{ ok: false, code: "compile-mismatch" }`; never ship a
  mismatched envelope. Simulating `inverse_ops` lets Undo paint as a projected
  arrangement.

The helpers:

- `arrangeGapHintLabels(count)`: home-row-first, prefix-free labels.
- `arrangeFlipPlan(beforeRects, afterRects)` → per-id `{dx, dy}` transforms.

Tests:

- every permutation for n ≤ 6 compiles, simulates exactly, and uses n − LIS ops;
- a 100-item random fuzz;
- first↔last moves, a same-slot no-op, and a return to the original order giving zero
  ops;
- coalescing and `stepBack`;
- rebase with deleted, added, and anchor-vanished items;
- `rule.move` index tracking with duplicate `match` texts;
- group block moves, and the fixed Money group;
- save-marker success and failure, including follow-on moves;
- gap-hint labels for 1, 26, 27, and 200 gaps.

Wire nothing into the cards in this phase. Bump the minor version, build, test, and
sync.

## Phase emoji-engine: bob-emoji search upgrades and api v1

Files: `plugins/bob-emoji/src/030-emoji-model.js` (or new fragments next to it),
`050-plugin.js`, `060-exports.js`, `scripts/gen-emoji-catalog.mjs`,
`plugins/bob-emoji/vendor/`, `scripts/test-bob-emoji-*.cjs`, `docs/bob-emoji.md`, and
`docs/develop/plugin-apis.md`.

1. **Catalog.** The generator also emits each entry's gemoji `category`, which is
   already in `vendor/gemoji-emoji.json`.

   > [!decision] cldr_keywords Vendor the CLDR English annotations (`annotations` plus
   > `annotationsDerived` `en.json`) from `unicode-org/cldr-json`. Open that repo with
   > `/sase_repo` (`sase repo open gh:unicode-org/cldr-json`); never web-fetch its
   > files. Pin the commit and copy the Unicode License into `vendor/`, and record both
   > in `vendor/README.md`. The generator merges each catalog glyph's CLDR keywords and
   > tts name into a `keywords` array (VS16-insensitive match, deduped against aliases
   > and tags). Keep the generated fragments split, and check that the bundle grows by
   > no more than about 250 KB. If the data cannot be obtained, ship the code-only
   > upgrades and record a PROPOSED FOLLOW-UP.

   > [!decision] cldr_keywords = no Skip CLDR. Keywords come from gemoji `tags` only.

2. **Search ranking**, one engine shared by the editor suggest and the api:
   - normalize: lowercase, strip surrounding colons, and treat `_`, `-`, and whitespace
     as token separators;
   - a query of several tokens matches only when _every_ token matches;
   - tiers, best first: exact alias or keyword, alias prefix, alias word-start or
     keyword prefix, alias substring, description word prefix, description substring,
     then conservative fuzzy (edit distance 1 for tokens of 4+ chars, only when fewer
     than 8 results);
   - recents and caller-supplied `boost` glyphs reorder only _within_ a tier;
   - a query that is exactly one emoji grapheme returns that entry first
     (VS16-insensitive).

   Regression set: `:fire:`, `white check mark`, `thumbs up`, `+1`, `memory` (🧠 only
   with CLDR), a pasted `🔥`, and the typo `rokcet`. Keep the editor menu's trigger
   grammar and empty-query popular rows. Update existing suggest tests only where
   ranking intentionally changed.

3. **Label suggestions.** `suggestForLabel(label)` tokenizes the label, drops a small
   stop-word list, lightly stems (`-s`, `-es`, `-ing`, `-ed`), and ranks by matched
   tokens, then tier. Examples to test: "Automation (service)" includes ⚙️ or 🛠️; "Seed"
   includes 🌱; "Ready" includes ✅.
4. **api v1.** In `onload`, set
   `this.api = Object.freeze({ version: 1, search(query, {limit, boost}), suggestForLabel(label, {limit, exclude}), categories(), browse(categoryId, {limit, offset}), info(glyph), recents({limit}), remember(glyphOrRow) })`.
   Rows are `{ emoji, name, alias, aliases, category, keywords, matchTier }`. `remember`
   shares the editor's persisted recents, mapping a glyph to its primary alias. The api
   exists even when `EditorSuggest` is unavailable, because the index builds anyway.
   Every method is total: it returns empty arrays or `null` and never throws.
5. Document the api in `docs/develop/plugin-apis.md` (new "Bob Emoji" section) and the
   search behavior in `docs/bob-emoji.md`. Bump to 0.2.0, then build, test, and run
   `bob plugins sync -p bob-emoji`.

## Phase album-card-shell: Album Schema Card visual refresh and region painting

Files: `241y-plugin-album-schema-card(-dom).js`, `242c-plugin-album-schema-edit.js`,
`241w-plugin-album-schema-model.js`, new fragments as needed,
`plugins/bob-ledger-tools/styles.css`, `scripts/test-ledger-tools-album-header.cjs`, and
`docs/albums.md`.

1. **Region painting.** Split `paintSchemaCard()` into stable regions: header, lint,
   body, stage, status, and hints/more, each with its own painter.
   - A session `onChange` repaints status, hints, and the stage's status sub-part only.
   - A selection change toggles classes and repaints hints.
   - The stage panel is built once per stage open (kind and target). Its inputs are
     never replaced while the stage is open, so caret, focus, and IME composition
     survive previews.
   - The read-only (non-editable) path keeps working.

   Tests assert input _node identity_ and selection range survive a preview round trip,
   and that no repaint replaces an input during composition.

2. **Visual refresh** per Design › Visual language. The details target becomes the
   header subtitle, and the duplicate stats row goes. Add pills, count pills with faded
   zeros, dashed `+ choice`/`+ field`, the single status line with its colored rule
   (hidden when empty), and the contextual hint strip with the `⋯` all-actions list.
   Remove `Preview this edit` and keep Save and Cancel. Refusal identity, hint, affected
   counts, derived keys, and the sync-warning **Retry sync** keep their information
   inside the status region.
3. **Enter latch** for stage edits: ⏎ during a pending preview applies when the current
   revision's preview succeeds, and any edit clears the latch.
4. Keep every documented contract: the Esc ladder, `y` jumps, refusal and stale
   behavior, Undo via `u`, and visible button equivalents (through `⋯`). Update
   `docs/albums.md` for the new layout, Enter latch, and hint strip.
5. Capture light and dark screenshots of browsing, a stage, a refusal, and the `⋯` list.
   Review them against the Visual language section before landing. Bump the minor
   version, then build, test, and sync.

## Phase album-arrange: Album arrangement editing and one-envelope save

Files: `241y-plugin-album-schema-card.js` (keep it ≤ 1000 lines by moving logic into new
fragments, e.g. `241yb-plugin-album-schema-arrange.js` and a shared
`241xb-plugin-schema-arrange-dom.js`), `242aa-plugin-guarded-update-session.js`
(`init.undo` seed), `242a-plugin-album-update-service.js` (per-stem last-undo record and
completion Notice), `styles.css`, tests, and `docs/albums.md`.

1. **Model.** The card owns an arrangement built from `schemaModel`: a `fields` scope
   and a `choices:<key>` scope per choices field. The body paints _projected_ order.
   Replace `schemaMoveSelected` with engine nudges. `schemaAlbumState` treats a pending
   arrangement like a draft: it stores `latestSchemaModel` instead of swapping the model
   underneath the user. After the card's own save settles, it adopts the fresh model and
   settles the save marker.
2. **Gestures** per the Arrangement interaction table: `⌥J/K`, `⌥↑/↓`, `⌥⇧J/K`, counts,
   `m`/`⌥M` gap hints, `esc` revert, and `u` step back or projected Undo. Non-reorder
   gestures are blocked while moves are pending (Design invariant 3).
3. **Shared arrangement DOM.** Put these in a fragment the bank card reuses: the FLIP
   runner (keyed by `data-arrange-id`, reduced-motion aware), delta badges, the
   moved-item accent, the gap-hint overlay (positioned from rects inside the body's
   scroll container and recomputed on resize), the pending bar, and the live announcer.
4. **Save pipeline** per Design › Save pipeline. Implement:
   - compile, simulate, and `setDraft` on settle;
   - the Enter latch, and quiet/loud detection from the album report;
   - background save with the shimmer;
   - follow-on moves through the save marker, never calling `setDraft` mid-apply;
   - refusal that stays pending;
   - stale explicit re-preview with an anchor rebase;
   - the predicted-equals-actual check;
   - close-during-save with the Notice and an Undo that survives a reopen;
   - timing spans.

   > [!decision] enter_saves = always_twice The first ⏎ always arms the amber
   > consequence, and quiet detection only chooses the wording.

5. **Tests** (unit, fake DOM): a multi-slot move yields exactly one envelope with
   compiled positions; chips repaint in projected order with zero writer calls; a return
   to the original order clears the draft; the latch; quiet versus loud (fake
   `group_before/after`); follow-on moves during an in-flight apply never call
   `setDraft` until it settles; refusal keeps pending; Undo paints the inverse
   arrangement; close-during-save records the plugin-level undo; `u`/`esc` ladders;
   gap-hint keystrokes; macOS `code` chords. Extend
   `scripts/obsidian-album-schema-writer-e2e.cjs` against real bob: a 3-move arrangement
   is one commit, `u` then ⏎ restores it in one commit, and the saved order equals the
   projection.
6. Document the gestures, pending bar, save and Undo semantics, and close-during-save in
   `docs/albums.md`. Capture screenshots (pending, hint mode, saving, saved) in both
   themes. Bump the minor version, then build, test, and sync.

## Phase album-emoji-slot: Shared Emoji Slot in the album card

Files: new fragments, for example `242f-plugin-emoji-slot-model.js` (pure state, result
assembly, 2-D navigation by column count, glyph detection via `Intl.Segmenter` or the
grapheme regex already used, duplicate and keycap rules) and
`242g-plugin-emoji-slot-dom.js` (chip field, panel, a11y); `242c` stage builders;
`styles.css`; tests; and `docs/albums.md`.

1. **Build the slot** per Design › Emoji Slot. It resolves
   `app.plugins.plugins["bob-emoji"]?.api` at use time, requires `version >= 1`, and
   falls back to the plain input otherwise. The host passes `label`, `vocabulary`
   (`{emoji, label}[]`), `siblings`, and `onPick`. Picks call `api.remember`.
2. **Wire into album stages.** Choice edit (pre-filled from the `split_icon` port),
   choice-add, and every field-add initial-choice row. The stage ops are unchanged:
   `choice.set {emoji}`, `choice.add {emoji}`, `field.add {choices[].emoji}`. An empty
   glyph on edit sends `emoji: ""`, which removes the icon (bob's contract). Vocabulary
   is every choice icon across the album report's albums. Siblings are the same field's
   choices.
3. **Esc ladder:** panel, then hint mode, then stage, then pending arrangement, then
   close. ⏎ in the panel never reaches the stage's save.
4. **Tests.** Filtering paint stays within budget over the full catalog; label
   suggestions appear on the empty query; Enter picks without applying; Tab leaves
   without picking; a pasted glyph sets directly; composition guards; the duplicate
   badge; digit keycaps are omitted and a pasted one gets the note; the round-trip
   fixtures from quick-fixes; fallback when the api is absent or v0. Run the
   schema-writer e2e with ZWJ, flag, and keycap emoji, and assert byte-exact
   round-trips.
5. Docs, and screenshots of the panel (empty query, filtered, browse) in both themes.
   Bump the minor version, then build, test, and sync.

## Phase album-place-drag: Add-and-place ghosts and pointer drag

1. **Ghost placement.** When `choice-add` opens, a dashed accent **ghost chip** appears
   at its default slot (after the selected choice, or at the end). It mirrors the slot's
   live preview chip. `⌥J/K`, `⌥⇧J/K` (matched by code and `preventDefault`ed inside
   inputs), and `⌥M` gap hints move the ghost while focus stays in the stage. The single
   `choice.add {position}` carries the placement. `field-add` gets a ghost **row** that
   places `field.add {position}` the same way. The ghost is an engine item with a
   temporary id that the compiler excludes. Save still means one op and one commit.
2. **Pointer drag.**
   - A grip handle (⠿) appears on hover and focus at a chip's or row's leading edge.
   - Dragging lifts the item with a subtle shadow and shows the 2 px insertion caret.
     The body autoscrolls near its edges.
   - A drop outside the item's scope (another field's chips) shows a not-allowed caret
     and snaps back.
   - Esc cancels the drag.
   - A drop calls `placeAtGap`, feeding the same pending arrangement and save pipeline.

   Put the drag controller in the shared arrangement DOM fragment so the bank card
   reuses it.

3. **Tests.** Ghost position compiles to the right `position`; ⌥J inside the label input
   moves the ghost and inserts no `∆`; drag start, move, and drop produce the same
   engine calls as the keyboard; an out-of-scope drop is rejected; Esc cancels. Run the
   real-Obsidian album UI e2e for a drag and a ghost add. Docs and screenshots. Bump the
   minor version, then build, test, and sync.

## Phase bank-arrange: Bank setup card arrangement parity

Files: `248k`/`248j`/`248i`/`248e`, the shared arrangement DOM fragment, `styles.css`,
`scripts/test-ledger-tools-bank-setup.cjs`, `scripts/obsidian-bank-ui-e2e.cjs`, and the
Setup paragraph of `docs/bob-ledger-tools.md`.

1. **Scopes.** `groups`: user groups only; Money stays first and is not a drop target
   before itself, and HIDDEN never moves. `group:<name>`: the declared visible
   categories in that group. `rules`. Accounts are never reorderable.

   > [!decision] bank_cross_group `⌥J/K` at a group's edge, gap hints, and drag may
   > cross into the adjacent group. Such a move compiles to `category.set {group}` plus
   > `category.move` in the same envelope and is always loud. Engine scopes for
   > categories become one categories scope partitioned by group, and the simulator
   > checks the group change.

   > [!decision] bank_cross_group = no Category moves stop at their group's edge with
   > the edge bump and the hint "⏎ edit to change group".

2. **Remove per-move confirm stages.** `beginSelectedGesture("move")` and confirm-stage
   moves go away. Every reorder is local, with the pending bar and status line from the
   shared arrangement DOM, and gap hints use `⌥M` (plain letters stay hotkey selection).
   Rule rows repaint their numbers in projected order with delta badges.
3. **Paint bob's dry run.** On preview, assert that the dry-run `categories[]` order
   equals the projection. A mismatch is red "bob disagrees with this order" and blocks
   save. Tiles whose derived hotkey shifts (`hotkeys.shifted`) get an amber `G→H` badge,
   and the status line explains it. Hotkey shifts, any rule reorder, and warnings are
   loud (Design › Save pipeline).

   > [!decision] enter_saves = always_twice All bank arrangements confirm on a second ⏎.

4. **Region painting** for the bank card: the tabs, panel, stage, status, and hints
   regions mirror album-card-shell. Stage inputs survive previews and composition. The
   contextual hint strip replaces the "Letters select a category…" note and footer. Port
   the Enter latch to `applyStage`.
5. **Ctrl+Z** steps back a local move while moves are pending, and otherwise stages the
   server Undo painted as a projected arrangement. Add close-during-save with the
   Notice, plugin-level last undo, and timing spans, as in album-arrange.
6. **Tests:** category, group, and rule multi-moves each make one envelope; the dry-run
   order check; `G→H` badges; rule moves are loud; no confirm stage; Ctrl+Z ladders;
   region identity; Alt chords by code; `⌥M` hints never hijack hotkey letters outside
   hint mode. Real-bob e2e (`scripts/obsidian-bank-ui-e2e.cjs` against its scratch
   vault): a category 3-slot move plus a group move is one commit, and a rule move
   confirms loudly and is one commit; Undo restores each. Docs, screenshots, minor
   version bump, then build, test, and sync.

## Phase bank-emoji-slot: Emoji Slot for bank categories

1. **Setup card.** The new and edit category stages use the Emoji Slot. The well is
   pre-filled from `entry.emoji` and the label input from `entry.label`. The op label is
   composed as `${emoji} ${label}` (or just the label when the well is empty), which
   supersedes the quick-fixes compose rule. Clearing the well explicitly removes the
   icon. Vocabulary is every category's emoji and label. Siblings are all visible
   categories. The placeholder no longer teaches hand-typing emoji.
2. **Category card.** The new-category stage (`248g`) uses the same slot. Its custom key
   routing (Tab cycle, character append outside inputs, group cycling) must hand every
   key to an open panel first. ⏎ in the panel picks and never runs
   `applyNewCategoryAndFile`.
3. **Tests:** compose and split round trips (shared fixtures, ZWJ, flags), the
   emoji-retained rename, explicit clear, panel key isolation in `248g`, and fallback
   without the api. Docs, screenshots, minor version bump, then build, test, and sync.

## Phase acceptance: End-to-end acceptance, measurements, and decision record

1. **Album walkthrough** in real Obsidian (the album UI e2e on its scratch vault),
   replaying Bryan's session the right way:
   - add a Section choice by typing "How I use SASE", Tab into the well, and pick 🔍;
   - place its ghost before 🎬 The Demos with `⌥M` plus a hint;
   - press ⏎ once: one preview, one commit, and the chip already in place;
   - `u` then ⏎ restores the previous state in one commit.

   **Bank walkthrough:** a category arrangement and a rule move, one commit each. Assert
   commit counts from the scratch vault's git log.

2. **Budgets.** Report nudge paint, emoji filter, dry run, and save p50/p95 from the
   timing spans against the Design budgets. Re-check the edge cases: about 100 choices,
   duplicate labels with distinct values, a stale-vault rebase mid-arrangement, rapid
   double ⏎, close during save, keyboard-only use, reduced motion, and both themes.
   Capture a final screenshot set for both cards.
3. **Consistency pass.** Check the two cards for consistency (hint strip wording,
   keycaps, colors, motion) and fix small drift inside bob-plugins. Confirm all four
   docs pages match shipped behavior, then run `npm run docs:check` and sync.
4. **Decision record.**

   > [!decision] decision_record Use `/sase_memory_write` (this plan's accepted memory
   > decision authorizes it). Add the bob-cli `sase/memory/decisions/` strand
   > `schema-arrangement-is-one-commit`. It applies to the album Schema Card and Bank
   > setup card. Its claim: any number of reorder moves is one locally composed
   > arrangement, previewed once and saved as one guarded envelope with one commit and
   > one Undo, while non-reorder gestures stay one gesture and one commit. Record the
   > rejected alternatives: per-slot transactions, a faster per-move apply, mixed
   > add/rename batching, and `--offline` interactive saves. Record the cost, cite the
   > research ref and the landing commits as evidence, and state what reopens it. Then
   > run `sase memory init`.

   > [!decision] decision_record = no Write no memory. Record a PROPOSED FOLLOW-UP
   > naming the decision record instead.

5. **Follow-ups.** On the phase bead, record PROPOSED FOLLOW-UP notes with the measured
   numbers for bob-cli save-latency trims: embed the post-write album entry and bank
   config report in the update JSON to drop the blocking `bob albums list` (~0.65 s);
   scope `verify_installed` to the edited album; let bob declare quietness
   (`index_changed`); and treat `--offline` interactive saves as a separate decision.
   Also record album-level emoji (`album.set` has no `emoji` today).

## Non-goals

- Changing any bob-cli writer, adding offline commits, or skipping the index rebuild.
- Mixing reorders with adds, renames, toggles, or removals in one envelope.
- Multi-select bulk moves, album-level emoji, a skin-tone picker, or reordering bank
  accounts.
- Removing the YAML `y` / `Ctrl+E` excursions.
