---
tier: tale
title: Soft-line fields for the property card editors
goal:
  Long one-line values in the Record Card, Day and Period, and Area and Project card
  editors are easy to read and write in a styled field that wraps and grows, never
  scrolls sideways, and never stores a stray line break.
size: medium
decisions:
  line_breaks:
    ask:
      Should a Record Card text value stay one stored line, with pasted line breaks
      folded into spaces?
    choices:
      fold:
        One stored line; long text wraps, Enter saves, pasted breaks fold to spaces with
        a note
      keep:
        Shift+Enter adds real line breaks, saved as escaped YAML and shown pre-wrapped
        in rows
    default: fold
    why:
      Record rows and Obsidian's own Properties show text as one line; sibling fields
      already fold
    answer: fold
  row_clamp:
    ask: Should long record values clamp to three lines in the Record Card field list?
    choices:
      clamp:
        Rows stay scannable; values past three lines end in an ellipsis until opened
      full:
        Rows wrap the whole value; one paragraph-long field can push others off screen
    default: clamp
    why: The open stage shows the whole value, so the list only needs to stay scannable
    answer: clamp
proposed_by: bbugyi200.athena.1g3
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-25 14:13:52
status: wip
---

# Soft-line fields for the property card editors

## Why

In Bryan's screenshot, the Record Card editor (`==` on an album record) is editing
**Claim** and the stage is broken:

- A bare "Claim text" label sits inline next to a tiny, unstyled 3-row textarea.
- The hint under it reads "Enter saves; newlines are refused."
- The row above shows the value in a single line that overflows the modal edge.
- Every row looks like a gray button.

Root causes, all in `bob-ledger-tools` in the `bob-plugins` linked repo (open it with
`/sase_repo`):

1. **No CSS for any typed stage.** `src/242q-plugin-record-card-stages.js` paints these
   classes, and `styles.css` defines none of them:
   - `bob-record-card-stage-label`, `-stage-input`, `-text`, `-stage-hint`,
     `-stage-actions`
   - `-stage-preview`, `-stage-warning`, `-link-band`
   - `-suggest-list`, `-suggest-row`, `-suggest-label`, `-suggest-count`,
     `-suggest-empty`
   - `-chip-list`, `-chip`, `-chip-text`, `-chip-edit`, `-chip-remove`, `-list-input`

   The text, suggest, date, number, URL, link, and list stages all render with browser
   defaults.

2. **Rows inherit Obsidian's button chrome.** `.bob-record-card-row` is a `<button>`
   styled at specificity 0,1,0, which loses to Obsidian's `button:not(.clickable-icon)`
   (0,1,1). Rows therefore get a fixed height, `nowrap`, and a gray fill, so a long
   value spills out of its row. Other cards already fix this. See the
   `.bob-album-schema-target` rule ("height: auto; white-space: normal") and the 0,2,1
   override comments in `styles.css`.
3. **The hint is false.** bob accepts line breaks in record text: `FieldKind::Text`
   passes them through, and `yaml_string_scalar` writes `"…\n…"`. A pasted multi-line
   text therefore saves silently as an escaped break. `docs/record-card.md` says "Pasted
   newlines are preserved for bob to refuse", which is also wrong. The same doc promises
   an "auto-growing textarea", but no autosize code exists.

## Survey of the other property card editors

| Editor                                                                  | Long-text input today                                                                                                                                                                     | Verdict               |
| ----------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------- |
| Record Card (album records)                                             | Unstyled stages, clipped rows, and a false newline hint (above)                                                                                                                           | **Fix**               |
| Day and Period card editor (`297c*`)                                    | The Intention and Summary stages are one-line `<input>`s. An agent-drafted summary scrolls sideways. A native `<input>` deletes pasted line breaks in Summary, which glues words together | **Fix**: same problem |
| Area and Project card (`==` opens the Property Editor settings, `298y`) | The Summary setting is a one-line `<input>`. Its 140-character guideline is about two lines                                                                                               | **Fix**: same problem |
| Album card editor (schema)                                              | Short identifiers (labels, plural, choice labels) in styled inputs                                                                                                                        | OK                    |
| Bank card editor (setup and row note)                                   | Short names and patterns in styled inputs. The row note is an intentional inline caption input                                                                                            | OK                    |
| Property Editor generic Text stage                                      | A styled, intentionally multi-line textarea (Shift+Enter adds a line)                                                                                                                     | OK, out of scope      |

## The design: a soft-line field

A **soft-line field** stores one line of text but wraps it across as many visual lines
as it needs. The same field is used everywhere a one-line value can be long: the Record
Card Text stage, the Day and Period Intention and Summary stages, and the Area and
Project Summary setting.

- **Look.** It starts at one line and grows with its content up to about six lines. Past
  that it scrolls internally. Text wraps at word boundaries, or anywhere for long URLs.
  It never scrolls sideways and has no resize grip.
- **Keys.**
  - Enter saves, and so does Shift+Enter. A soft-line field can never hold a line break.
  - With a LINK band open, each editor's existing band rules apply unchanged.
  - Esc, `q`, and Ctrl+[ behave as they do today.
- **Line breaks.**
  - Pasted text that contains line breaks is trimmed. Each run of breaks, together with
    any spaces or tabs touching it, becomes one space.
  - A short quiet note then appears, for example
    `Line breaks became spaces — Claim is one line.`
  - Any other path that slips a break in (drop, IME, autocorrect) folds the whole value
    the same way, and the caret is kept.
  - A seeded value that already has breaks is shown verbatim until the user edits it.
- **Undo.** The paste fold inserts through `execCommand("insertText")`, so one Ctrl+Z
  undoes the paste natively.

> [!decision] line_breaks = keep Record Card Text alone becomes a multi-line field. Do
> not call `bobSoftLineAttach` for it. Instead, autosize it with the same CSS
> (`.bob-soft-line` sizing without the fold listeners). Enter saves, and Shift+Enter
> inserts a line break: the record keydown handler must let Shift+Enter through when no
> band is open. Rows render values with `white-space: pre-wrap`. The docs say that bob
> stores breaks as quoted `\n` YAML. The Day and Period and the Area and Project Summary
> fields still fold, because bob refuses multi-line summaries there.

### Record Card editor look

While a stage is open, the field's row and its stage form one accent-outlined unit:

```
  Status *        🌱 Seed
╭──────────────────────────────────────────────────────╮
│ Claim *                                              │  ← row: accent border, value hidden
│ ┌──────────────────────────────────────────────────┐ │
│ │ AGENTS.md files are not enough. SASE adds        │ │  ← soft-line field, full width
│ │ structure to agentic memory.▍                    │ │
│ └──────────────────────────────────────────────────┘ │
│ ▏claim: "AGENTS.md …" → "AGENTS.md …"                │  ← bob preview echo (muted, accent rule)
│                                  [ Cancel ] [ Save ] │  ← Save is mod-cta
╰──────────────────────────────────────────────────────╯
  Section *       🤖 Agent CLIs
```

- The row stays the header. Its value hides while the stage is open, because the field
  shows the full value and nothing is duplicated.
- The visible "Claim text" label goes away, because the row already names the field. The
  `<label for>` stays and becomes visually hidden, so screen readers and the existing
  `textContent` tests still see "Claim text".
- In-stage buttons shrink to **Cancel** and **Save** (`mod-cta`).
  - Save + next and Save + prev stay on Tab and Shift+Tab, in the hint strip, and in All
    actions (`···`), which already lists Save, Save + next, Save + prev, and Cancel.
  - Save keeps `data-record-input="stage-save"`.
- The text kind gets no standing hint, because the hint strip already shows ⏎ save, ⇥
  save+next, and esc cancel. Other kinds keep their guidance hints (date grammar, number
  range, URL, link forms, list keys, suggest count).
- The fold note replaces the hint while it applies.
- An empty text field shows the placeholder `<Label>…`.
- The field uses the card's theme variables, so it works in light and dark mode. It also
  gets a soft accent focus ring and readable line-height (1.5).
- When a stage opens, the stage scrolls into view with `block: "nearest"`.

## Implementation (the `bob-plugins` linked repo, plugin `bob-ledger-tools`)

Read `AGENTS.md` in the repo first. Edit only the `src/` fragments, `styles.css`, tests,
and docs, then rebuild `main.js` with `npm run build`. Keep every hand-edited fragment
at or below 1000 lines.

### 1. Shared helper: new fragment `src/241xd-plugin-soft-line-field.js`

Add the fragment to `src/fragments.json` right after `241xc-plugin-card-close-keys.js`.
Add a test-facing exports fragment `src/370a-exports-soft-line-field.js` right after
`370-exports-card-close-keys.js`, following that file's pattern. The helper has no
top-level DOM access and fails open: every DOM call sits in `try`/`catch`.

- `bobSoftLineFold(text) → { value, folded }`
  - Replaces each run of line breaks (CRLF, CR, LF, U+2028, U+2029), together with the
    spaces and tabs touching it, with one space.
  - Changes no other whitespace.
- `bobSoftLinePasteText(text) → { value, folded }`
  - Text without a line break is returned unchanged with `folded: false`.
  - Otherwise the text is trimmed and then folded.
- `bobSoftLineAttach(textarea, { skip, onFold } = {})`
  - Idempotent: mark the element with `data-soft-line="true"` and wire it once.
  - Adds class `bob-soft-line` and sets `rows="1"`, `wrap="soft"`, and
    `enterkeyhint="done"`.
  - Do not set `aria-multiline`.
  - Callers keep creating the element themselves, so the `// bob-text:` role markers
    stay at their creation sites. `scripts/test-prose-field-roles.cjs` enforces those
    markers. Do not create text controls inside the helper.
  - Element-level listeners. These run at the target before the editors' delegated
    `input` listeners on `contentEl`, so a draft never sees a line break.
    - `beforeinput`: `preventDefault()` for `insertLineBreak` and `insertParagraph`.
    - `paste`: unless `skip?.()` is true (an open proseField NAME stage owns paste),
      read `text/plain`. If it has a line break:
      1. `preventDefault()`.
      2. Insert `bobSoftLinePasteText(...).value`, trying in order:
         `doc.execCommand("insertText", false, value)`, then
         `setRangeText(value, start, end, "end")` plus a bubbling `input` event, then a
         value splice with the caret set.
      3. Call `onFold?.("paste")`.
    - `input`: if the value still contains a break, fold the whole value, put the caret
      at the folded length of the text before it, and call `onFold?.("input")`. Then
      autosize.
- `bobSoftLineAutosize(textarea) → "css" | "js" | null`
  - Returns `"css"` when `CSS.supports("field-sizing", "content")` is true.
  - Otherwise sets `style.height = "auto"`, then
    `scrollHeight + (offsetHeight - clientHeight)`, and preserves the selection. The CSS
    `max-height` still caps the height.
  - Feature detection matters because Obsidian's Electron version follows the installer,
    which can be old on a Mac.
- CSS base in `styles.css`:
  - `textarea.bob-soft-line { display: block; box-sizing: border-box; width: 100%; min-width: 0; height: auto; resize: none; field-sizing: content; overflow-y: auto; line-height: 1.5; white-space: pre-wrap; overflow-wrap: anywhere; max-height: min(40vh, 11.5em); }`
  - Add a one-line `min-height` that covers the padding and border.

### 2. Record Card editor

`src/242q-plugin-record-card-stages.js`:

- **Text kind.** Keep creating the `<textarea>` with its `bob-text` marker. Drop
  `rows="3"`. Per **`line_breaks`** (default `fold`), call `bobSoftLineAttach` with:
  - `skip`: returns whether the record proseField handle reports `isNaming()`.
  - `onFold`: sets the modal's stage note and repaints.
  - Set `placeholder` to `${label}…`.
- **Label.** For every non-list kind, the `<label>` gains `is-visually-hidden`. The list
  stage keeps its visible caption and its "Add item" label, both restyled.
- **Hint.** `recordCardStageHintText("text")` returns `""`. The painter appends a hint
  only when it has text. When `options.stageNote` is set, paint
  `<p class="bob-record-card-stage-hint is-note" role="status">` in its place.
- **Actions.** `recordCardPaintStageActions` paints Cancel and then Save. Save gets
  `mod-cta` and keeps `data-record-input="stage-save"`. Disabled-while-applying behaves
  as today.

Modal state (`242r`/`242s`/`242p` as fits):

- Add `stageNote`. It is set by `onFold` to
  `Line breaks became spaces — ${label} is one line.` and cleared by `recordOpenStage`,
  `recordCancelStage`, and any stage change or successful save. Thread it into the
  `recordCardPaintStage` and `recordCardPaintTypedStage` options built in `242o`.
- After every `paintRecordCard`, call `bobSoftLineAutosize` on the live text input.
- On stage open only, not on every keystroke repaint, call
  `scrollIntoView?.({ block: "nearest" })` on the inline stage element.

`src/242o-plugin-record-card-dom.js`:

- `recordCardPaintRow` adds `is-editing` when `options.stage?.field === target.key`.
  Check both the rebuild path and the kept-stage repaint path.

`styles.css`, in the Record Card block around the `.bob-record-card-modal` rules:

- Scope the row, choice, and action rules under `.bob-record-card` so they beat
  `button:not(.clickable-icon)`. At rest, rows use:
  - `height: auto`, `min-height: var(--input-height, 2.2em)`, `white-space: normal`, and
    `line-height: 1.4`
  - `background: transparent` and `box-shadow: none`
  - the hover and selected treatments that exist today
- `.bob-record-card-value`, per **`row_clamp`**:
  - `clamp`:
    `display: -webkit-box; -webkit-box-orient: vertical; -webkit-line-clamp: 3; overflow: hidden`
  - `full`: wrap only
- Editing unit:
  - `.bob-record-card-row.is-editing`: accent border, square bottom corners, value
    hidden.
  - `.bob-record-card-row.is-editing + .bob-record-card-stage`: accent border with no
    top edge, rounded bottom corners, a light accent tint (`color-mix(... 4–6% ...)`),
    and padding of about `0.2em 0.55em 0.6em`.
  - Use a negative top margin equal to the list gap so the two pieces butt together.
  - Lay the stage out as a grid with a gap of about 0.45em.
  - Give the non-inline `.bob-record-card-stage-host` the same panel look without the
    join.
- `.bob-record-card-stage-input` (all kinds):
  - Full width, `box-sizing: border-box`, and about `0.45em 0.6em` of padding.
  - A `--background-modifier-border` border, `var(--radius-s)`, and
    `--background-primary`.
  - `font: inherit`. Single-line kinds use `height: var(--input-height)`.
  - On `:focus-visible`, an accent border plus
    `box-shadow: 0 0 0 2px color-mix(in srgb, var(--interactive-accent) 25%, transparent)`.
- `.is-visually-hidden`: the usual sr-only clip pattern, scoped to the card.
- Hint and note: about 0.82em, `--text-faint`. The note uses `--text-muted`.
- Preview and warning: about 0.86em with a 2px left rule, in the accent color for the
  preview and in `--text-warning` for the warning.
- Suggest list and rows: reuse the `.bob-record-card-choice*` look and give the list a
  max-height with scrolling.
- List chips:
  - Wrapping flex row.
  - Pill chips (`999px` radius, border, `--background-primary`).
  - Quiet edit and remove buttons with Obsidian chrome reset (`height: auto`,
    transparent background, `box-shadow: none`).
- Link band: copy the intention editor's hide-when-empty rule.
- Stage actions: a right-aligned flex row with a gap of about 0.4em. At 375px and below
  the buttons stretch.
- Add the stage controls to the existing `forced-colors` and reduced-motion handling.
  Match the existing dark-mode mix. Add no animations: choices stages rebuild on every
  paint and would flicker.

### 3. Day and Period card editor

- `297cc`: the `intention` and `summary` stages create a `<textarea>` with class
  `bob-intention-stage-input is-line` and keep the `bob-text: prose S14` marker. Then
  call `bobSoftLineAttach` with:
  - `skip: () => intentionCardEditorProseIsNaming(modal)`
  - an `onFold` that sets `stageNote` to the intention note or a summary note
    (`Line breaks became spaces — a summary is one line.`)

  Reflection stays the multi-line textarea it is today.

- Retire the intention-only paste path in `297ca` (`intentionCardEditorPaste` and its
  host `paste` listener) in favor of the shared fold, so a paste is not handled twice.
  Keep `INTENTION_CARD_EDITOR_PASTE_NOTE` as the intention note. Update exports and
  tests to match.
- Update the note in place where possible, so one Ctrl+Z still undoes the paste. If a
  full repaint is unavoidable, caret restoration must still hold.
- `297cd` `intentionCardEditorProseKey`: mirror the Record Card guard. When the band was
  open and declines an Enter (Shift+Enter in a textarea), treat it as consumed: close
  the band and never save a half-typed link.
- Autosize: call `bobSoftLineAutosize` after `paintIntentionCard` and in
  `intentionOnStageInput`. Reflection keeps `intentionCardEditorAutogrow`.
- CSS: `.bob-intention-stage-input.is-line` takes the soft-line base. Do not let the
  generic `min-height`/`resize: vertical` rule fight it.

### 4. Area and Project Summary setting (Property Editor)

- `298y` `propertyEditorPaintSummaryStage`:
  - Create a `<textarea>` and keep the `bob-text: literal one-line area/project summary`
    marker.
  - Keep its classes, kept-input reuse, and live `N/140` counter.
  - Call `bobSoftLineAttach` with an `onFold` that adds the note
    `Line breaks became spaces — a summary is one line.` through the setting notes.
- `298i` stage keydown: the `this.stage.kind === "text" && event.shiftKey` pass-through
  must skip `[data-soft-line]` inputs, so Shift+Enter saves like Enter there. The
  generic multi-line Text stage keeps Shift+Enter as a new line.
- Autosize after each paint of the setting stage.

### 5. Tests (`node --test`, FakeNode harnesses)

- New `scripts/test-ledger-tools-soft-line-field.cjs`, added to `test:unit` in
  `package.json`. Cover:
  - fold vectors: CRLF, CR, LF, U+2028/9, breaks surrounded by spaces, runs, and leading
    or trailing breaks
  - paste trim
  - the attach attributes and idempotence
  - `beforeinput` prevention
  - paste through the `execCommand`, `setRangeText`, and value-splice fallbacks, with
    `onFold`
  - `skip` leaves the paste native
  - the `input` safety net keeps the caret
  - autosize returns `"js"` and `"css"`, and `null` on a bare node
- Record Card (`test-ledger-tools-record-card-stages.cjs` and `-prose-field.cjs`):
  - the textarea has `data-soft-line`, no `rows="3"`, and the placeholder
  - the hidden label still provides "Claim text" in `textContent`
  - no text hint, and the fold note when one applies
  - Cancel and Save only, with Save as `mod-cta`
  - the `is-editing` row
  - a multi-line paste produces one folded draft and a `field.set` value with no `\n`
  - the band's Shift+Enter guard stays green
- Intention editor DOM and prose tests: the Intention and Summary stages are `TEXTAREA`
  with `data-soft-line`, a summary paste folds with the summary note, and Shift+Enter
  with the band open never saves. Update assertions that expected `<input>` or the old
  paste path.
- Property Editor settings tests: the Summary stage is a soft-line textarea, the counter
  is unchanged, and Shift+Enter saves.
- Run `npm test` (it starts with `build:check`) and `npm run docs:check`. If a test
  outside this change fails, check whether it is a flake or pre-existing before touching
  it.

### 6. Prove the look in real Obsidian

Extend the Record Card real-Obsidian proof (`scripts/obsidian-record-card-e2e.cjs` and
its workflow lib, which already has `capture(...)`). Give a fixture record a text field
holding about 300 characters, then add:

- screenshots of the open Text stage in light at 1400px, dark at 1400px, and light at
  375px
- the 6-line cap with internal scroll
- a trusted-key multi-line paste that asserts the folded draft and the note
- a closed-stage screenshot that shows the clamped row

Run it as `docs/develop/validation.md` describes, with xvfb-run, the snap Obsidian
binary, and `BOB_BIN`/`BOB_CLI_DIR` with provenance. Also capture the Day and Period
Summary stage and the Area and Project Summary setting with a long value. Use their
existing runners (`obsidian-intention-card-e2e.cjs`, `obsidian-property-editor-e2e.cjs`)
if they reach those stages. Otherwise a scratch capture run is fine and need not be
committed. Look at every screenshot yourself and fix anything that looks off before
finishing. Report the screenshot paths.

### 7. Docs

Update the user-facing docs in the same change:

- `docs/record-card.md`:
  - Rewrite the Text row as a soft-line field (wraps, grows to about six lines, Enter
    saves, line breaks fold to spaces with a note).
  - Describe the joined row and stage, the hidden duplicate value, and Cancel and Save.
    Save + next and Save + prev move to Tab, Shift+Tab, and All actions.
  - Remove "preserved for bob to refuse".
- `docs/intention-card.md`: the Intention and Summary fields wrap as soft-line fields,
  and pasted breaks fold in both.
- `docs/property-editor.md`: Summary edits one line in a wrapping field.

Bump the `bob-ledger-tools` manifest minor version, following recent feature commits.
Then run `npm run build`, the tests, and `bob plugins sync`. Follow the AGENTS.md rules
if sync refuses an overwrite, and never pass `-F` on your own.

## Acceptance

- In real Obsidian, a 300-character Claim opens as a full-width, wrapped field joined to
  its row. Nothing overflows the modal and nothing scrolls sideways, in light, dark, and
  at 375px.
- A paste of `a\nb\r\nc` saves `a b c`, the note shows, and one Ctrl+Z restores the
  pre-paste text.
- No record text stage save sends a value containing `\r`, `\n`, U+2028, or U+2029.
- Day and Period Intention and Summary, and the Area and Project Summary, use the same
  soft-line field and behavior.
- Every Record Card stage kind (text, suggest, date, number, URL, link, list, and
  choices) renders styled. The audit of `src/242q`/`242o` classes against `styles.css`
  shows no undefined stage classes.
- `npm test` and `npm run docs:check` pass, and the vault is synced.

## Out of scope

- bob-cli itself. A bob-side refusal of multi-line record text would change the CLI
  contract, and the widget now guarantees one line.
- The Property Editor's generic multi-line Text stage.
- Album schema notice paragraphs that render as plain text. They are cosmetic and not
  this problem.
