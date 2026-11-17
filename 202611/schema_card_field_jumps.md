---
tier: epic
title: Schema card field jumps, numbered fields, and grouped rhythm
goal: 'On the album Schema Card (opened from album and record notes), Shift+J/Shift+K
  step between fields and 1–9 jump straight to a numbered field, with a keycap showing
  each field''s number beside its name and clear space between field groups. On the
  bank Setup card (~/bob/bank.md), Shift+J/Shift+K step between category groups while
  digit hotkeys keep working. Every jump is pure navigation that never writes, and
  the numbers and jumps always follow the order painted on screen.

  '
phases:
- id: field-nav
  title: Album card field navigation on one painted order
  depends_on: []
  size: medium
  description: 'field-nav: route the Schema Card''s targets through one painted-order
    helper shared with the body painter, add Shift+J/K field steps and 1–9 field jumps
    with clamp-bump-announce feedback and reveal scrolling, retire the album digit
    count prefix, add shared bump/reveal helpers, unit tests, a real-key e2e probe,
    and key docs.'
- id: field-visuals
  title: Numbered field keycaps, heading rows, and group rhythm
  depends_on:
  - field-nav
  size: medium
  description: 'field-visuals: paint keycaps for fields 1–9 with current-group emphasis,
    make field rows transparent headings, add dialog-scoped group spacing and a shared
    label axis, add the field hint item and row accessibility, then check geometry
    and screenshots in real Obsidian and update the visual docs.'
- id: bank-groups
  title: Bank Setup card group jumps
  depends_on:
  - field-nav
  size: small
  description: 'bank-groups: add Shift+J/K category-group jumps in the Categories
    tab using painted order, make hotkeys match lowercase only so Shift+K stops selecting
    Car, tint the current group label, add a footer hint, reuse the shared bump/reveal
    helpers, and add tests and docs.'
decisions:
  field_number_record:
    ask: Add a decisions strand saying schema-card digits are painted field numbers?
    memory:
    - decisions:schema-card-digits-are-field-numbers
    default: false
    answer: false
proposed_by: bbugyi200.apollo.research.57.linker.w0
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-17 15:36:18
status: wip
bead_id: bob-cli-mu
---

- **PROMPT:** [prompts/202611/schema_card_field_jumps.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/schema_card_field_jumps.md)
- **BEAD:** [bob-cli-mu](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-mu/README.md)

# Plan: Schema card field jumps, numbered fields, and grouped rhythm

## Context

Bryan asked for `J`/`K` (next/previous field) and `1`–`9` (jump to field N) keymaps on
the schema editing cards, with each field's number shown near its name and more vertical
space before each field's choices. He agreed with **every requirement** in the
consolidated research report. Read it first:

```bash
sase artifact read research:202611/schema_card_field_jump_design/schema_card_field_jump_design.md "Field jump design requirements"
```

The screenshot that motivated this is `~/tmp/screenshots/20261117_110414.png` (Zettel
album: Status, Claim, Section). It shows two problems. Field rows render as filled gray
bars that look like the choice chips. A field with no choices (Claim) sits closer to the
next field (Section) than chips sit to each other.

**Where the code lives.** All code changes are in the linked **bob-plugins** repo. Open
it with `/sase_repo` (`sase repo open bob-plugins -r "<why>"`) and read its `AGENTS.md`.
Paths below are relative to that repo. `F/` stands for `plugins/bob-ledger-tools/src/`.
Repo rules:

- Edit `src/` fragments and never hand-edit the generated `main.js`.
- Every hand-edited fragment must stay **≤ 1000 lines**. Current sizes: `241y` card 953,
  `241yb` 949, `241yc` 928, `241yd` 200, `242c` 250, `241y-…-card-dom` 117, `241xb` 545,
  `248k` 849, `248i` 456.
- A new fragment must be registered in `src/fragments.json`. A prototype mixin goes
  after `241yd`.
- A new test file must be added to `test:unit` in `package.json`.
- Run `npm run build` → `npm test` → `npm run docs:check` → `bob plugins sync`. If sync
  refuses, follow AGENTS.md's refusal protocol, which never uses `-F` on your own.

This plan changes no bob-cli code, no CLI/JSON contract, and nothing on the release
train.

**Scope facts:**

- **Album and record notes share one card.** `F/242d-plugin-album-entrypoints.js`
  resolves a record's `parent:` and opens the same `AlbumSchemaCardModal`, so records
  get everything for free.
- **The bank card is a separate class.** `~/bob/bank.md` opens `BankSetupCardModal`
  (`F/248k`).
- **Some surfaces must not change.** The read-only record sheet (`F/242i–242k`) and the
  inline album header (`F/241v`, `F/241x`) reuse `.bob-album-schema-field` classes.
  Every CSS change must therefore be scoped to the dialog.
- **The DOM stays flat.** Ghost insertion, `m` gap badges, and drag-caret math all
  assume field rows and chip clouds are direct siblings in the body (`F/241yc`,
  `F/241xb`). No wrapper elements.

## Design (shared by all phases)

### The one rule

> **`⇧J` / `⇧K` select the next / previous field row in painted order, counted from the
> current selection. `1`–`9` select field N in painted order.** Lowercase `j`/`k` and
> the arrows keep the fine walk over every target, and it still wraps.

Every row of the research key table follows from this rule:

| From                              | `⇧J`                            | `⇧K`                             |
| --------------------------------- | ------------------------------- | -------------------------------- |
| Album details                     | field 1                         | stays (bump)                     |
| field _n_                         | field _n+1_ (last: stays, bump) | field _n−1_ (first: stays, bump) |
| a chip or `+ choice` of field _n_ | field _n+1_                     | **field _n_** (parent first)     |
| a body issue or `+ field`         | stays (bump)                    | last field                       |

Field jumps clamp and never wrap. A digit with no field N leaves the selection where it
is, bumps, and announces. With zero fields, every one of these keys is a bump plus an
announcement.

### Painted order

**Painted order** means the order of the `[data-target-key]` elements as painted:

1. the Album details row,
2. for each field in **projected** order (the pending arrangement while it is dirty):
   the field row, its chips in projected order, then `+ choice`,
3. the body's issue rows,
4. `+ field`.

The current split is a bug the research confirmed. `schemaTargets()` reads the saved
`model.fields` and lists `add-field` before issues. The body paints projected order and
puts issues before `+ field`. While a reorder is pending, `j`/`k` can therefore move
backwards on screen. The fix is to make the body painter and the target list **call the
same order functions**, so they cannot drift apart again.

### Feedback vocabulary (shared by both cards)

- **Jump.** The selection ring moves. The view scrolls to show the selection
  (`scrollIntoView({ block: "nearest" })`). A **group jump** shows the whole group:
  scroll the group's chip cloud (or tile block) into view first, then the heading row,
  so the heading is always visible and as many choices as fit are too. A polite live
  announcement follows: `Claim, field 2 of 3` or `Bills, group 3 of 4`.
- **Refusal** (clamp, missing digit, zero fields). The selection stays where it is. The
  selected element gets a **3 px vertical bump** (90 ms, `translateY`, downward for `⇧J`
  and missing digits, upward for `⇧K`), plus an announcement:
  - `No next field` / `No previous field`
  - `No field 7 · this album has 3 fields` (use `1 field` when there is one)
  - `This album has no fields yet`
- **No jitter.** A held key that keeps hitting an end (`event.repeat`) clamps silently
  after the first bump. `prefers-reduced-motion` turns off every bump.
- **Reach screen readers.** The shared announcer (`schemaArrangeAnnounce`,
  `F/241xb:280`) sets the `hidden` attribute, which removes the live region from the
  accessibility tree. Its CSS (`.bob-album-schema-arrange-live`) already hides it
  visually, so drop the attribute. Then the existing nudge announcements and the new
  ones all reach screen readers.

### Non-goals (deliberate)

- **Read-only card.** The no-session fallback card gets no new keys and no keycaps. Its
  rows are not targets today, and it is a short-lived state.
- **`0` stays unbound.** No numbers for fields 10 and up (they get an invisible
  placeholder that keeps the column).
- **No new key aliases.** No `{`/`}` aliases. No `event.code` fallback for `J`/`K`,
  matching `j`/`k`.
- **Untouched surfaces.** No wrapper DOM, and no change to the record sheet or the
  inline header.

## Phase field-nav: Album card field navigation on one painted order

1. **Shared order helpers (`F/242c`).**
   - Add pure helpers:
     - `albumSchemaPaintedFieldOrder(model, projected)`: projected `fields` filtered to
       known keys, else model order.
     - `albumSchemaPaintedChoiceOrder(field, projected)`: projected `choices:<key>` when
       it covers every choice, else declared order.
     - `albumSchemaPaintedTargets(model, projected)`: stats → fields with chips and
       `+ choice` → issues → `add-field`.
   - Also add `albumSchemaFieldStepKey(targets, selectedKey, direction)`, which returns
     the next/previous `field:` key strictly after/before the current index, or `null`.
   - Also add `albumSchemaFieldNumberKey(targets, n)`.
   - Make `albumSchemaCardPaintBody` (`F/241yc:281`) use the two order helpers in both
     its hint-mode and normal loops, with no behavior change.
   - The body builds chips from `albumHeaderPresentField(field, model).choices`. Confirm
     that this list has the same members as `field.choices` (or derive targets from the
     presented field) so that keys and DOM cannot differ.
   - Export the new helpers next to `albumSchemaTargets` in `F/352-exports-album.js` for
     tests.
2. **Targets follow paint (`F/241y`).**
   - `schemaTargets()` returns
     `albumSchemaPaintedTargets(this.schemaModel, projection)`.
   - `projection` is `this.schemaArrangement.scopes()` only when that arrangement
     already exists and `dirty()` is true. Read it directly. Do not call
     `schemaEnsureArrangement` from target lookup, which avoids lazy creation and
     recursion.
   - Leave `albumSchemaTargetByKey` (key lookups) alone.
3. **Keys.**
   - In the base `schemaOnKeyDown` (`F/241y:820`), add two branches after the Alt
     branches and before the `j`/`k` lines, gated by no Ctrl/Meta/Alt:
     - `event.key === "J" | "K"` (Shift is implied, and Caps Lock counts) →
       `this.schemaFieldStep(±1, event)`.
     - `/^[1-9]$/.test(event.key)` (**ignore `shiftKey`**: US `⇧1` arrives as `!`, and
       AZERTY digits need Shift) → `this.schemaFieldNumber(n, event)`.
   - Put the methods in a **new fragment** `F/241yf-plugin-album-schema-field-jump.js`
     (`Object.assign(AlbumSchemaCardModal.prototype, …)`, registered after `241yd`).
   - Both methods are no-ops without `this.session`.
   - On a hit they set `selectedKey`/`selectedIndex`, call `paintSchemaCard("select")`,
     reveal the group, and announce `<label>, field N of M`.
   - On a miss they bump and announce (see the feedback vocabulary).
   - They never touch the session, previews, writers, or the arrangement.
   - Existing gates already make them inert in inputs, stages, IME composition
     (`isComposing`, `keyCode 229`), and `applying`.
4. **Retire the count prefix (`F/241yb`).**
   - Delete `schemaArrangeCountBuffer` and `schemaArrangeConsumeCount`.
     `schemaArrangeNudge` steps exactly `direction`, and its signature stays.
   - Replace the digit-buffer branch (`F/241yb:884-890`) with **hint-mode swallowing**:
     while `this.schemaHintMode` is set, digits and any `J`/`K` that
     `schemaArrangeHintKey` did not consume are `preventDefault`ed and return. Hint mode
     owns the keyboard.
   - Outside hint mode, digits fall through to the base handler.
5. **Shared motion helpers (`F/241xb`).**
   - Add `schemaCardBump(node, sign, { axis = "y" })`. It is gated on
     `schemaArrangeReducedMotion()`, uses `node.animate` when present, and does nothing
     otherwise.
   - Add `schemaCardRevealGroup(head, body)`, which runs
     `body?.scrollIntoView({ block: "nearest" })` and then
     `head.scrollIntoView({ block: "nearest" })`, each optional-chained in try.
   - Route the album nudge clamp shake (`F/241yb:119-130`) through
     `schemaCardBump(…, { axis: "x" })`. The look is unchanged, but it is now
     reduced-motion gated.
   - Make `paintSchemaCard("select")` (`F/241yd:99`) reveal the selected element
     (nearest), so `j`/`k` and clicks never strand the selection off-screen.
   - Remove `hidden` from the announcer.
6. **Tests.** Add a new `scripts/test-ledger-tools-album-schema-field-jump.cjs`
   (registered in `test:unit`), using the FakeNode harness the shell/arrange suites use.
   It must cover:
   - **Order invariant.** Target keys equal body+header `[data-target-key]` DOM order,
     both clean and after moving Claim below Section. `j` then walks the order on
     screen.
   - **`⇧J`/`⇧K`.** Every row of the table above, including clamps, parent-first, and
     `+ field` to the last field.
   - **Digits.**
     - `3` selects Section.
     - `7` on three fields stays put and announces
       `No field 7 · this album has 3 fields`.
     - Digits follow projected order while dirty.
     - `0` is inert and buffers nothing.
     - `3` then `⌥K` nudges field 3 exactly one slot.
   - **Modifiers.** `⇧1` = `!` still stages required. `{ key: "1", shiftKey: true }`
     jumps. `⌥J`/`⌥⇧J`/Ctrl+J keep their meanings.
   - **Gates.** Keys are inert in a stage, an input, composition, `applying`, the
     read-only card, and hint mode. Hint mode survives a swallowed digit.
   - **No writes.** Spy on session preview/apply and `schemaArrangement.nudge`; they get
     zero calls for any jump.
   - **Feedback.**
     - `scrollIntoView` is called (and a missing one doesn't throw).
     - `animate` is called once on a clamp, not on `event.repeat`, and not under reduced
       motion.
     - The existing nudge shake is now gated.
     - The announcer has no `hidden` attribute.

   Update any existing test that relied on the count buffer or the old target order.

7. **Real-key e2e.**
   - Add a CDP probe to `scripts/obsidian-album-ui-e2e.cjs`. Send
     `Input.dispatchKeyEvent` the way `scripts/obsidian-bank-ui-e2e.cjs:384` does, with
     `key: "J"`, `code: "KeyJ"`, Shift modifier, `text: "J"`.
   - From Status, `⇧J` lands on the next field. `3` lands on field 3. The bob shim log
     shows no `albums update` apply.
   - Keep `--self-test` and `scripts/test-obsidian-album-ui-e2e.cjs` green.
   - Run the real e2e under `xvfb-run -a` when Obsidian is installed, and report if it
     isn't.
8. **Docs (`docs/albums.md`).**
   - Remove the count-prefix sentence (line ~139) and "counts" from the keyboard
     paragraph (~186).
   - Add: `Shift+J`/`Shift+K` jump to the next/previous field (from a chip, `Shift+K`
     first returns to its own field, and jumps stop at the ends). `1`–`9` jump straight
     to field N as painted, pending moves included.
9. **Verify and ship.** `npm run build`, `npm test`, `npm run docs:check`,
   `bob plugins sync`.

## Phase field-visuals: Numbered field keycaps, heading rows, and group rhythm

Field anatomy (the dialog only):

```text
 ⠿ [1] Status   required                        set on 4 of 4
       [🌱 Seed 0] [✍️ Draft 4] [✅ Ready 0] … [+ choice]
                                                    ← 0.8em group break
 ⠿ [2] Claim    required · text · sub-groups    set on 4 of 4
                                                    ← 0.8em group break
 ⠿ [3] Section  sub-groups index                set on 4 of 4
       [🪝 Intro 0] [🧱 What is SASE? 1] … [+ choice]
                                                    ← 0.8em
       [+ field]
 ⏎ edit · ⇧J/K 1–3 field · a add field · … · esc/q close · ⋯
```

1. **Keycap.**
   - Use `<span class="bob-album-schema-field-key" aria-hidden="true">N</span>`, placed
     after the grip and before `.bob-album-schema-field-main`, on editable field rows
     only.
   - Keep the builder out of near-full `241yc`: put it in
     `F/241y-plugin-album-schema-card-dom.js` or `241yf`.
   - `albumSchemaCardPaintFieldRow` takes `fieldNumber` from the painted field index, so
     numbers follow projected order while dirty.
   - Fields 10+ get the same element, empty, with `is-unnumbered` (`visibility: hidden`)
     to keep the column.
   - Style follows the bank keycap recipe:
     - about a 1.45 row-em square, 1px `--background-modifier-border`, radius 0.3em,
       `--background-secondary` fill;
     - a `--text-muted` monospace digit at 0.78em, weight 650, `tabular-nums`, optically
       centered on the label;
     - 120 ms color transition, which reduced motion turns off.

     Watch em scoping: the digit's smaller font must not shrink the box.

2. **"You are here."**
   - `albumSchemaCardPaintSelection` also toggles `is-current-group` on the field row
     whose chip or `+ choice` is selected.
   - To make that possible, give `+ choice` a `data-field-key`, and read the owner from
     the selected element's `data-field-key`.
   - The keycap of a selected or current-group field gets an accent border, an accent
     digit, and a 10% accent tint. It shows where `⇧K` will land before you press it.
3. **Field rows become headings.**
   - Obsidian's `button:not(.clickable-icon)` fill (specificity 0,1,1) currently beats
     the intended transparent `.bob-album-schema-target`.
   - Under `.bob-album-schema-dialog button.bob-album-schema-field` and the same for
     `.bob-album-schema-stats-target`, set the rest state to transparent background, no
     box-shadow, and a transparent border.
   - **Re-assert** hover (`--background-modifier-hover`) and selected (2px
     `--interactive-accent` ring plus an 11% accent tint) at that same specificity, or
     the rest rule erases them.
   - Chips keep today's fill. This brings back the row/chip contrast the stylesheet
     already intends.
4. **Rhythm and one label axis.** All of this is scoped to
   `.bob-album-schema-dialog-body`.
   - Set fixed inline sizes for the grip and the keycap, and derive
     `--bob-schema-gutter` from them plus the row padding and gaps.
   - **Label axis.** `.bob-album-schema-label`, the chip-cloud left edge, `+ field`, and
     ghost-row text all start on one vertical axis, with keycaps hanging in the gutter.
     The cloud uses `padding-inline-start: var(--bob-schema-gutter)`. `+ field` uses
     `margin-inline-start: var(--bob-schema-gutter); width: auto`.
   - **Group break.** Put a `0.8em` break before every group: each field row or ghost
     row that is not the body's first child, a gap badge that leads a field, and the
     trailing block (first legacy chip, first issue, or `+ field`). A field row that
     follows its own gap badge sits `0.2em` below the badge.
   - **Cloud spacing.** The cloud hugs its row (`0.2em`). Zero the chip margins and let
     the cloud's `gap: 0.3em` own spacing inside the cloud.
5. **Hint strip (`albumSchemaCardHintItems` and `albumSchemaCardPaintHints`,
   `F/241yc`).**
   - The renderer accepts `keys` as a string or an array. An array paints adjacent
     `<kbd>`s.
   - In plain browsing with at least one field, insert
     `{ keys: ["⇧J/K", "1–N"], label: "field" }` right after the `⏎` item. N is
     `min(fields, 9)`, and a single field shows `1`.
   - Leave it out with zero fields, in a stage, in hint mode, on the pending bar, while
     saving, in a lock wait, and on the read-only card.
6. **Accessibility.**
   - The editable field row's `aria-label` becomes
     `Field 2 of 3, Claim, required, text`.
   - Add `aria-keyshortcuts="N"` on fields 1–9. The keycap stays `aria-hidden`.
7. **Tests.** Add these to the field-jump test file or the shell suite:
   - Keycaps on fields 1–9 only, with `is-unnumbered` on the 10th.
   - Numbers re-flow after a nudge.
   - `is-current-group` follows chip and `+ choice` selection and clears on field or
     stats selection.
   - Hint item placement, its `1–N` text, and every absence case. The renderer paints
     two `<kbd>`s.
   - `aria-label` and `aria-keyshortcuts`.
   - The inline album header and record sheet paint no keycaps.

   The existing place-drag, arrange, and ghost suites stay green.

8. **Real-Obsidian geometry and look (`scripts/obsidian-album-ui-e2e.cjs`).** Assert
   in-page:
   - The field row's computed background at rest is transparent while a chip's is not,
     and the selected row's `box-shadow` is not `none`.
   - Label left equals first-chip left equals `+ field` left (±1px).
   - The space between a group's last element and the next field row is at least 0.6×
     the row font size, including the Claim→Section case with no choices. A
     row-to-its-cloud space is at most 0.35×.
   - `esc/q close` and `⋯` lie inside the hint strip's bounds.

   Capture light, dark, and 375 px screenshots of the Zettel-like fixture with a chip
   selected. **Open the PNGs and judge them.** Keycaps should be crisp and aligned,
   groups clearly separated, chips clearly under their field, and the selection ring
   unclipped. Tune values until they read well.

9. **Docs.** In `docs/albums.md`'s Schema Card paragraph (~82), describe numbered field
   keycaps in a hanging gutter, heading rows with indented choices, the accent keycap
   for the current group, and the `⇧J/K 1–N field` hint.

> [!decision] field_number_record If accepted, this phase also authors a decision strand
> `sase/memory/decisions/schema-card-digits-are-field-numbers.md` in bob-cli, using the
> `/sase_memory_write` skill and then `sase memory init`.
>
> - **Applies to:** every schema-editing card (the album Schema Card, the bank Setup
>   card, and the future Record Card).
> - **Claim:** on schema-editing cards, `⇧J`/`⇧K` step between field rows and `1`–`9`
>   select field N, both in painted order (pending moves included), clamped. A card
>   whose digits are already hotkeys keeps them and gets only `⇧J`/`⇧K` group jumps.
>   Lowercase `j`/`k` remain the fine walk.
> - **Why:** fields are the unit you act on, and numbers must match what is on screen.
> - **Evidence:** the research ref above and this epic's commits.
> - **Reopen if:** an album needs more than 9 fields every day.

## Phase bank-groups: Bank Setup card group jumps

1. **Painted items (`F/248i`).** Add
   `bankSetupCardPaintedCategoryItems(model, arrange)`. It returns group-label and tile
   items in painted order: the groups and tiles from `bankSetupCardProjectedSections`
   (`F/248j:63`), then `group:hidden` and its tiles. Also add
   `bankSetupCardGroupStep(items, selection, direction)`, which returns the
   next/previous `type: "group"` item strictly after/before the current one (parent
   first, clamped).
2. **Keys (`F/248k:667`).**
   - In Categories, with no Ctrl/Meta/Alt, handle `key === "J" | "K"` before the hotkey
     branch → `this.jumpGroup(±1)`. On a hit: select the item, paint, reveal the label
     plus its tiles with `schemaCardRevealGroup`, and announce `Bills, group 3 of 4`. On
     a miss: bump and announce `No next group` / `No previous group`, with the same
     repeat and reduced-motion rules.
   - The keys are inert (not consumed) on Accounts, Rules, and Checks. They are also
     inert in stages, inputs, and `applying`.
   - In bank hint mode (`F/248ka:720`), unmatched `J`/`K` are swallowed and never reach
     hotkeys.
   - Tighten the hotkey match at `F/248k:793` to `/^[a-z0-9]$/` with no `i` flag.
     bob-cli stores hotkeys as lowercase `[a-z0-9]`. `⇧K` stops selecting 🚗 Car, while
     `k` and the Bills digits `1`–`8` still work.
   - Make `moveSelection` in Categories walk the same painted items. This fixes the
     dirty-arrangement arrow drift, which is the same bug class as the album card's.
3. **Look.** Give a group label whose tile is selected `is-current-group`:
   `--text-muted`, up from `--text-faint`, with no underline. Keep `aria-pressed` for
   the label itself. Route the bank nudge clamp shake (`F/248ka:130`) through
   `schemaCardBump(…, { axis: "x" })`.
4. **Footer.** `bankSetupCardHintItems` (`F/248kb:205`) adds
   `{ keys: "⇧J/K", label: "group" }` after `⏎ edit` in Categories browsing only.
5. **Tests.** Add a new `scripts/test-ledger-tools-bank-setup-group-jump.cjs`
   (registered in `test:unit`) covering:
   - every step and clamp, parent-first `⇧K`, and HIDDEN as the last group;
   - `K` doesn't select Car, while `k` and `1`–`8` still do;
   - an uppercase letter selects nothing;
   - inert tabs and gates, including hint-mode swallowing;
   - projected group order while dirty for both `⇧J` and the arrows;
   - the current-group label class, the footer item, no writer calls, and reduced-motion
     bumps.

   Add a real-key probe (`Input.dispatchKeyEvent` Shift+K and Shift+J) to
   `scripts/obsidian-bank-ui-e2e-setup.cjs` and keep its self-test green.

6. **Docs.** Update the Setup line in `docs/bob-ledger-tools.md` (~78): Categories
   `Shift+J/K` jump between groups, and hotkeys match lowercase only.
7. **Verify and ship.** `npm run build`, `npm test`, `npm run docs:check`,
   `bob plugins sync`.

## Acceptance (final manual pass, live vault)

- **Zettel album** (`==` on `sase_blog_zettel.md`):
  - Status → `⇧J` → Claim, and `3` → Section.
  - From a Section chip, `⇧K` → Section and the keycap lights up first.
  - Claim and Section are clearly separate groups, and the chips sit under their labels.
- **Record note:** `==` opens the same card with the same keys.
- **Nine-field `subscriptions` album:** `9` reaches the last field and scrolls it into
  view.
- **Bank** (`==` on `bank.md`): `⇧J` walks Money moves → Everyday → Bills → HIDDEN. `k`
  still picks Car and `1` still picks the first Bills category.
- **Themes:** light and dark both read well, and so does a narrow window.
