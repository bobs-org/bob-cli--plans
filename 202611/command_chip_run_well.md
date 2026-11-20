---
tier: tale
size: medium
title: Command chips you can spot at a glance — the solid run well
goal:
  Give every command chip and bob-run card the copy chip's capsule in the run hue — a
  solid accent well holding a white ▶ beside a tinted body — so a chip that runs bob is
  unmistakable at a glance, reads the same in every view, never strands its well, and
  shows each run state in place, in light and dark.
decisions:
  copy_only_look:
    ask:
      Should a command chip that can only copy wear the copy chip's blue well and copy
      glyph?
    choices:
      copy:
        Blue copy well and body, exactly like a copy chip; blue always means a click
        copies
      run:
        Keeps the purple run body with a muted copy well; blue no longer always means
        copy
    default: copy
    why:
      The click copies, so the copy look tells the truth and keeps one meaning per hue
    answer: copy
  glossary_command_chip:
    ask:
      Should the glossary Command Chip strand stop calling the inline chip a quiet pill?
    memory:
      - glossary:command-chip
    corrects: renders the span as a quiet pill with a ▶ glyph
    default: true
    answer: true
proposed_by: bbugyi200.apollo.fa.f0
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-20 15:35:59
status: wip
---

# Plan: Command chips you can spot at a glance — the solid run well

## Goal

Make every command chip (`run:` / `run[LABEL]:` spans and `bob-run` fences, rendered by
the `bob-chips` plugin in the `bob-plugins` linked repo) unmistakable at a glance in
Obsidian, the way the blue copy well did for copy chips
(`plan:202611/copy_chip_blue_well.md`). It must be intuitive (the well's glyph says what
a click does, its color says what happened), reliable (one look in every view, no
reflow, no stranded well, no hover shift), and beautiful in light and dark. Grammar,
gestures, the cursor rule, the run engine, policies, tooltip text, announcements, and
payload handling do not change. This is a look change plus two small DOM/paint tweaks.

Design mockups, light and dark, before vs after for every inline state, the vault-style
task rows, labels and sequences, copy-only and refused chips, the ⌘↵ cues, and every
card state: `file:explicit:4c41c148f7cef40c6f754702` (light) and
`file:explicit:f6553589e913ef0a7323f494` (dark). The prototype CSS behind them (both
`styles.css` command-chip sections, verified in headless Chromium under Obsidian's real
button rules) is `file:explicit:39ee8943991facc192cec1c9`; start from it, but re-verify
everything below. It still positions `.bob-run-chip-action` absolutely; step 2 replaces
that rule. View any of them with `sase artifact open <ref>`.

## Why command chips are hard to spot today (diagnosis)

Measured with the production builders, run store, and painter in headless Chromium under
Obsidian 2.1.2's default palette and its real button rules (`OBSIDIAN_BUTTON_RULES` in
`scripts/preview-copy-chips.cjs`), against `plugins/bob-chips/styles.css` at bob-chips
0.15.0:

1. **The faintest thing on the line.** The ▶ is a hairline stroked outline in
   `--text-faint`: 1.96:1 against its fill in Live Preview and 2.3:1 in Reading view,
   below the 3:1 floor for meaningful icons. The fill is a 7% accent wash over
   `--background-secondary` with a hairline border. Beside it, the vault's cyan inline
   code and the new blue copy well both shout louder than the one thing that runs a
   command.
2. **Lost on task rows.** `gtd_daily`'s three command chips (`#sync`, Bank, `#init`) sit
   on the Blocked/Next row washes, where they read as pale boxes.
3. **Two looks for one thing.** `.bob-run-chip` (0,1,0) sets the wash, but Obsidian's
   `button:not(.clickable-icon)` (0,1,1) wins the background. Reading view, embeds,
   hover previews, and Tasks/Dataview results render command chips as white form buttons
   while Live Preview shows the lilac wash: the computed background differs in every one
   of the 12 state pairs compared.
4. **Stranded glyph.** The host wraps normally, so Chromium breaks between the ▶ and
   the face: 892 of 14,544 sweep layouts (120–420 px, 0–11 lead words, six chips, both
   surfaces). Showing the tooltip moves the layout in 308 more, because its out-of-flow
   placeholder sits after the face.
5. **States whisper.** Armed, queued, running, failed, and attention only recolor a
   hairline border and a faint glyph.
6. **The card shares all of this,** and its title squeezes letter by letter on a narrow
   card ("Morning / upkeep") because the header never wraps.

## The design: a solid run well

Every inline command chip becomes the copy chip's two-part capsule in the run hue:

- a solid **well** holding a white, filled ▶ (the universal "play"): the pre-attentive
  beacon;
- a lightly tinted **body** holding the face in hue-tinted ink: calm and readable.

This makes one rule for the whole chip family: **a solid well marks a chip you can
click; its glyph says what a click does, and its color says what happened.** Blue well +
copy glyph copies; accent well + ▶ runs; green says it finished; red says it failed;
orange says it needs attention; a pale gray well with ⊘ says it can't run here. Hue is
never the only cue: glyph shape, cursor (`pointer` vs `copy`), and the tooltip all
agree, so blue/purple confusion under color-vision deficiency costs nothing.

The run hue is the theme accent (`var(--interactive-accent)`). Command chips, Command
Mode, and the ⌘↵ cues already wear it, and the copy plan moved copy to blue precisely so
accent could stay "run". On a `gtd_daily` row, accent appears only as link _text_; no
other solid accent fill sits on the line.

### Tokens (inline chip)

Express every color through one custom property, so a state changes only the ink and the
glyph. Every value below is a theme variable; no hex, no `rgb[a](`.

| Part                 | Rule                                                                                                                                                                                                                                                                                                                                                                                                                                          |
| -------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Ink                  | `--bob-run-chip-ink: var(--bob-run-chip-hue, var(--interactive-accent))` on the chip. `--bob-run-chip-hue` is the one user knob (a snippet may set it on `body`); styles.css never sets it.                                                                                                                                                                                                                                                   |
| Body fill            | `color-mix(in srgb, var(--bob-run-chip-ink) 10%, var(--background-primary))`. Opaque, so row washes never muddy it.                                                                                                                                                                                                                                                                                                                           |
| Border               | `1px solid color-mix(in srgb, var(--bob-run-chip-ink) 50%, var(--background-primary))`, radius `var(--radius-s)`                                                                                                                                                                                                                                                                                                                              |
| Face ink             | `color-mix(in srgb, var(--bob-run-chip-ink) 30%, var(--text-normal))`. Literal faces stay monospace at normal weight; labeled faces use the text font with `.bob-run-chip-label` at weight 500.                                                                                                                                                                                                                                               |
| Separators and count | `.bob-run-chip-sep`, `.bob-run-chip-count`: `color-mix(in srgb, var(--bob-run-chip-ink) 40%, var(--text-muted))` (5.1:1 light, 5.3:1 dark; `--text-faint` was below 3:1 on the new body).                                                                                                                                                                                                                                                     |
| Geometry             | The copy chip's: `padding: 0.1em 0.5em 0.12em`, `font-size: 0.92em`, `line-height: inherit` (`normal` on `button.bob-run-chip`), `vertical-align: baseline`, `box-decoration-break: clone` (+ `-webkit-`), `height: auto`, `margin: 0`, `font: inherit`, `text-align: inherit`, `box-shadow: none`.                                                                                                                                           |
| Well                 | `.bob-run-chip > .bob-run-chip-glyph`: `display: inline`; `margin-inline: -0.5em 0.42em`; `padding: 0.1em 0.3em 0.12em`; start-side radii `calc(var(--radius-s) - 1px)`; `background-color: var(--bob-run-chip-ink)`; `color: var(--bob-run-chip-on-ink, var(--text-on-accent))`; `--icon-stroke: 2.25px`. It also resets the shared base `.bob-copy-chip-glyph` rule's `inline-flex` / `width` / `min-width` / `margin-right` / faint color. |
| Glyph                | `.bob-run-chip > .bob-run-chip-glyph svg`: `0.95em` square, `vertical-align: -0.13em`. `svg.lucide-play` gets `fill: currentColor` (a solid ▶); every other glyph stays stroked. The well is padding plus a fixed-size svg, so glyph swaps never reflow.                                                                                                                                                                                     |
| No stranded well     | Host `white-space: nowrap`; `.bob-run-chip-face { white-space: normal; overflow-wrap: anywhere }`, plus the DOM change in step 2.                                                                                                                                                                                                                                                                                                             |
| Cursor               | `pointer` (`copy` for copy-only chips)                                                                                                                                                                                                                                                                                                                                                                                                        |

### States

No state changes border width, padding, font weight, or the well's size; only colors,
border style, and the glyph change. The glyph map in `COMMAND_CHIP_GLYPHS` stays as it
is.

| State                | Look                                                                                                                                                                                                                                                         |
| -------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| Ready                | Accent well, white solid ▶                                                                                                                                                                                                                                  |
| Hover / `mobile-tap` | Body 17%, border full ink; 120 ms color transition (dropped under reduced motion)                                                                                                                                                                            |
| Active               | Body 24%                                                                                                                                                                                                                                                     |
| Focus-visible        | `outline: 2px solid var(--bob-run-chip-ink); outline-offset: 2px; box-shadow: none`                                                                                                                                                                          |
| Armed (⌘↵ target)    | Border full ink plus a soft halo, `box-shadow: 0 0 0 2px color-mix(in srgb, var(--bob-run-chip-ink) 30%, transparent)`. No size change. The halo must survive hover and focus (order it after them).                                                         |
| Loading              | The ready look with the glyph svg at `opacity: 0.6`, so chips don't flash gray at startup                                                                                                                                                                    |
| Queued               | Hourglass in the well; dashed border in the ink                                                                                                                                                                                                              |
| Running              | Spinning loader in the well. A sequence's 2 px segmented progress bar moves from the host onto the face (see below). The current step reads in the ink.                                                                                                      |
| Completed (~1.6 s)   | Ink `var(--color-green)`: green well, white check, green body; one pulse                                                                                                                                                                                     |
| Needs attention      | Ink `var(--color-orange)` with `--bob-run-chip-on-ink: var(--text-on-accent-inverted)`: a black ▶ on an orange well. White fails 3:1 on orange in both themes (2.95 / 2.34); Obsidian defines the inverted ink as `black` in both and uses it the same way. |
| Failed               | Ink `var(--text-error)`: red well, white ▶, red body; kept until the next run or reload                                                                                                                                                                     |
| Unavailable          | Ink `var(--text-muted)`, dashed `--background-modifier-border` border, `--background-secondary` body, muted face; the well is pale, `color-mix(in srgb, var(--text-muted) 20%, var(--background-primary))`, with a muted ⊘                                   |
| Copy only            | See decision **copy_only_look** below; `cursor: copy`                                                                                                                                                                                                        |
| Copied (~1.2 s)      | Ink `var(--color-green)` plus the pulse                                                                                                                                                                                                                      |
| Copy failed (~2 s)   | The borrowed `is-failed` wins over the copy ink: red well, the copy feedback's alert glyph                                                                                                                                                                   |
| Fired (⌘↵)           | Border full ink and one 0.6 s press: a from-only keyframe starting at `color-mix(in srgb, var(--bob-run-chip-ink) 35%, var(--background-primary))`                                                                                                           |
| Reduced motion       | No spin, pulses, transitions, or timed tooltip reveal (the tooltip shows at once); glyphs and state colors stay                                                                                                                                              |
| Print / PDF          | Plain inline code exactly as today: no border, fill, padding, well, halo, progress, tooltip, or weight; monospace face                                                                                                                                       |

Both pulse keyframes become **from-only**
(`@keyframes bob-run-chip-pulse { from { background-color: var(--bob-yank-flash-bg, var(--text-highlight-bg)); } }`),
so every pulse eases back to the element's own fill rather than the old wash.

> [!decision] copy_only_look = copy Copy-only chips (notes under `mail/`, `ref/`,
> `ref_history/`, `.bob/`, `_conflicts/`, mobile, no Command Mode, unknown query source)
> set `--bob-run-chip-ink: var(--bob-copy-chip-hue, var(--color-blue))`: blue well, copy
> glyph, blue body, identical to a copy chip. A copy-only `bob-run` card does the same.

> [!decision] copy_only_look = run Copy-only chips keep the accent ink for the body and
> border. Only the well changes: pale
> `color-mix(in srgb, var(--bob-run-chip-ink) 20%, var(--background-primary))` with the
> copy glyph in the ink (check ≥ 3:1). Copied and copy-failed states still turn green
> and red. Docs then say a copy-only chip "keeps the run body with a copy well".

Measured in the prototype (Obsidian default palette): face ink 9.9:1 light and 7.8:1
dark; the white ▶ on the well 4.26:1 in both; the well against the page 4.26:1 /
3.91:1; the attention glyph 7.1:1 / 9.0:1; the card title 10.1:1 / 8.0:1 and its Run
button 9.6:1 / 7.7:1. Live Preview and Reading computed 0 differences over 624 property
comparisons.

### Tooltip

Keep the existing reveal, rows, and text exactly. The host is now `nowrap`, so the tip
keeps its own `white-space: normal` (it already sets it). Add the tooltip to the
reduced-motion block so it shows at once.

### One look in every view

Live Preview spans and Reading/embed/hover/Tasks/Dataview buttons must compute identical
fill, border, shadow, ink, padding, font, outline, and well in every state. Name both
tags in every rule: base rules on `:is(span, button).bob-run-chip` (0,1,1) tie
Obsidian's `button:not(.clickable-icon)` and win on source order; state rules reach
(0,2,1); `button.bob-run-chip.mobile-tap` covers the tap state. This replaces the old
`button.bob-run-chip` / `button.bob-run-chip:hover` patch rules. Reading chips remain
atomic `<button>` boxes, so a long sequence still wraps inside its box there; that
layout difference is accepted, as it is for copy chips.

### No stranded well, no hover shift

Host `nowrap` with a wrapping face removes every stranded ▶. The tooltip's out-of-flow
placeholder after the face still let the end cap wrap alone on hover (280 layouts). The
fix that brought every count to 0 is the one copy chips already use: move the visually
hidden `.bob-run-chip-action` ("Run"/"Copy") **after the face, in flow at zero size**.
Give it the `.bob-copy-chip-copy-prefix` recipe:
`font-size: 0; white-space: nowrap; user-select: none`, no absolute positioning or clip.
`aria-labelledby` keeps naming action then face, so the accessible name is unchanged.
Prototype sweep: 0 stranded wells, 0 lone end caps, and 0 hover shifts over 14,544
layouts per pass. Overshoot past the column stays within the end cap's own padding (8.4
px max, as with clone-decorated inline code).

### Progress rides the face

`.bob-run-chip.has-progress` no longer paints the host. Instead
`.bob-run-chip.has-progress > .bob-run-chip-face` takes
`background-image: var(--bob-run-progress)` at `0 100% / 100% 2px no-repeat` with
`padding-block: 0.1em 0.12em`, the chip's own vertical padding, so the bar sits exactly
on the chip's inner bottom edge (measured: the face's box ends at the host's padding
box). The well no longer hides the first segment, and because the face keeps the default
`box-decoration-break: slice`, a wrapped face carries one continuous bar across its
lines instead of a full bar per line. In `commandChipProgressGradient`, the current
segment becomes `var(--bob-run-chip-ink, var(--interactive-accent))` so the knob
recolors it.

### The `bob-run` card

The card takes the same identity at card scale, mirroring the `bob-copy` card:

- **Frame:** `--bob-run-chip-ink` declared on `.bob-run-fence`; border
  `color-mix(in srgb, ink 35%, var(--background-modifier-border))`; background
  `var(--background-primary)` (paper, not `--background-secondary`).
- **Header:** fill at 8% ink, bottom border at 25% ink, and the copy card's wrap rule
  (`flex-wrap: wrap; justify-content: flex-end; gap: 0.4em 0.5em`, title
  `flex: 1 1 auto; min-width: 0`), so on a narrow card the meta and Run button wrap to a
  right-aligned second row instead of squeezing the title.
- **Header glyph:** a solid rounded-square well (`1.5em`, `var(--radius-s)`, ink
  background, `var(--bob-run-chip-on-ink, var(--text-on-accent))` glyph, solid ▶).
- **Title:** face ink, weight 600.
- **Run button** (`:is(span, button).bob-run-fence-run`): the chip body recipe (12%
  fill, 50% border, face ink, weight 500, `cursor: pointer`; hover 17% with a full-ink
  border; active 24%; focus-visible ink outline; no shadow). Width: `min-width: 5.5em`
  holds Run/Output. A copy-only card's button shows Copy / "Copied ✓" / "Copy failed",
  so `.bob-run-fence.is-copy .bob-run-fence-run` takes the copy card's `7.5em`. Each
  kind keeps one width, so no feedback reflows the header.
- **States** swap the card ink exactly like the inline chip: armed (full-ink frame plus
  the 2 px halo), loading (glyph 0.6), queued (dashed frame), running (spinner in the
  well; current step mark and text in the ink), completed (green), attention (orange,
  inverted glyph), failed (red), unavailable (dashed hairline frame,
  `--background-secondary` header, pale well, muted title), copy-only (per the
  decision), copied (green), fired (one header press). Step marks keep their own colors.
- **Output links** keep `--text-accent`; give them the `:is(span, button)` treatment so
  Obsidian's button rules never paint them.
- **Reduced motion** and **print** keep today's behavior. Scope the print hide rules
  with `[class]` (for example `.bob-run-fence[class] .bob-run-fence-glyph`) because the
  new glyph and button rules outrank the old ones. The prototype's first pass printed
  glyphs and buttons until it did this.

In the **Make or edit chip** card, `.bob-copy-author-modal.is-run .bob-copy-author-icon`
becomes the run well: set
`--bob-copy-chip-ink: var(--bob-run-chip-hue, var(--interactive-accent))` there, so the
base icon rule paints the accent well and white glyph, and fill its `svg.lucide-play`.
Its live preview already uses the production builders and inherits the rest. Keep the
inert preview's `pointer-events: none`.

### Explicitly unchanged

- Copy chips and `bob-copy` cards: the CSS text before the command-chip section and from
  `.bob-copy-fence {` up to the `bob-run` card section stays byte-identical. Keep the
  shared base `.bob-copy-chip-glyph` rule as is, and scope every new glyph rule under
  `.bob-run-chip >` or `.bob-run-fence-header`.
- The ⌘↵ keycap capsule (`.bob-run-chip-hint`), tooltip rows and text, announcements,
  ARIA names, the glyph map, grammar, gestures, the cursor rule, the run store and
  engine, policies, Notices, timers, and source mode.
- No new plugin setting. The CSS custom property is the only knob.

## Steps

Work in the `bob-plugins` linked repo (open it with `/sase_repo`; read its `AGENTS.md`),
except step 7. `bob-chips` is a source-fragment plugin: edit `src/` and `styles.css`,
run `npm run build`, never hand-edit `main.js`, and keep every hand-edited fragment at
or under 1000 lines.

1. **Styles.** In `plugins/bob-chips/styles.css`, rewrite the inline command-chip
   section (today `.bob-run-chip {` up to `.bob-copy-fence {`) and the `bob-run` card
   section (its leading comment up to `.modal.bob-copy-author-modal {`) per the design
   above, then change the authoring-card run icon. Open the inline section with a
   leading comment (`/* Command chips: …`) so the slicing tests have a stable anchor.
   Theme variables only, except the existing `--bob-yank-flash-bg` fallback.
2. **DOM and paint.** In `src/041-command-chip-dom.js`:
   - append the children as glyph, face, action, tip (the action moves after the face),
     and style `.bob-run-chip-action` with the in-flow zero-size recipe;
   - change the progress gradient's current segment to
     `var(--bob-run-chip-ink, var(--interactive-accent))`.

   Keep every best-effort `try` guard, the ids, and `aria-labelledby` (action then face)
   / `aria-describedby` exactly.

3. **Static tests.**
   - Add `scripts/test-command-chips-look.cjs` and register it in `package.json`
     `test:unit`, modeled on `scripts/test-copy-chips-look.cjs`. Over the command-chip
     sections it asserts:
     - no fixed colors;
     - the `var(--bob-run-chip-hue, var(--interactive-accent))` default on the chip and
       the card;
     - a nowrap host with a normal face;
     - a well scoped under `.bob-run-chip >`, and the filled `lucide-play`;
     - the attention `--text-on-accent-inverted` ink;
     - the progress rule on the face;
     - from-only pulse and fire keyframes;
     - no `border-width` or `border: [2-9]px` in state rules;
     - every `COMMAND_CHIP_FACES` face styled;
     - reduced-motion and print blocks covering chip, well, svg, tooltip, and card parts
       with `[class]` scoping;
     - the 5.5em / 7.5em button widths;
     - the run author icon's ink.
   - Update the pinned expectations in `scripts/test-command-chips-live-preview.cjs`
     ("styles use theme variables…") and `scripts/test-copy-chips-look.cjs` (the slice
     anchors, "the command-chip slice is untouched", and the
     `.is-run .bob-copy-author-icon` assertion) to the new tokens. Their old assertions
     pin the quiet look on purpose; replace them, don't delete them.
   - Extend the DOM tests (the closest existing command-chip DOM test) with these
     checks:
     - child order glyph, face, action, tip;
     - the action's text still flips Run/Copy;
     - `aria-labelledby` still names action then face;
     - the progress gradient uses the ink variable;
     - the payload still appears in no attribute.
4. **Preview harness.**
   - **Shared baseline.** Extract the Obsidian baseline (`OBSIDIAN_BUTTON_RULES`, the
     root variables, and Obsidian's default light/dark palettes, including
     `--color-blue`, `--text-on-accent`, and `--text-on-accent-inverted: black`) into
     one module, for example `scripts/preview-obsidian-baseline.cjs`.
     `preview-copy-chips.cjs`, `preview-command-chips.cjs`, and
     `preview-command-chips-author.cjs` all use it, so the palettes can never drift
     again. This fixes **bob-cli-od**.
   - **Context rows.** Add vault-style context to the command preview, as the copy
     preview has:
     - Blocked, Next, and WIP row washes;
     - cyan inline code;
     - an accent link;
     - copy chips beside command chips, including rows shaped like `gtd_daily`'s `#sync`
       and `#init` rows.
   - **Checks.** Extend `scripts/preview-command-chips.cjs` so the run exits nonzero on
     any failure. Check:
     1. **Parity** of span vs button computed styles (host, well, face) in every face,
        plus forced `:hover`, `:active`, `:focus-visible` via `CSS.forcePseudoState`,
        under the real button rules.
     2. **Geometry.** Chip and well rects are identical across every face of one chip.
     3. **Sweep** (120–420 px in 3 px steps, 0–11 lead words, a label, a short literal,
        a long literal sequence, a running sequence, an unavailable chip, and a
        copy-only chip, on both surfaces): no stranded well, no lone end cap, overshoot
        within the end cap's padding, long faces still wrap, and no layout change
        between rest and a shown tooltip.
     4. **Contrast** in light and dark. At rest, face vs body is at least 4.5:1, glyph
        vs well at least 3:1, and well vs page at least 3:1 for ready. Glyph vs well is
        at least 3:1 for attention, failed, unavailable, and copy-only. Completed,
        copied, and copy-failed are transient and exempt. Parse both `rgb()` and
        `color(srgb …)`.
     5. **Progress.** The bar is on the face, ends at the host's padding box, and no
        segment sits under the well.
     6. **Card button width.** One width per card kind: Run/Output for run cards, and
        Copy/"Copied ✓"/"Copy failed" for copy-only cards. This replaces today's
        single-width check, which fails by design under the 5.5em/7.5em split.
     7. **Card header.** A narrow card never wraps the title mid-word.
     8. **Print and reduced motion**, as today plus the new parts.
     9. **Attributes.** No payload appears in an attribute, and ARIA names are
        unchanged.
   - **Screenshots.** Keep the existing ones (`command-chips-{light,dark}.png`,
     `command-fences-{light,dark}.png`, narrow, reduced-motion, print). Keep the offline
     `--self-test` passing and update `scripts/test-command-chips-preview.cjs` if the
     matrix shape changes.
   - **Review.** Run the real preview (`CHROMIUM_PATH` when none is on `PATH`;
     Playwright's Chromium works), inspect every screenshot yourself, and tune within
     the token tables only when a capture shows a real defect. Record any tuned value in
     the docs. Rerun `preview-copy-chips.cjs` and `preview-command-chips-author.cjs`;
     both must pass.
5. **Real-Obsidian e2e.** Add `scripts/obsidian-command-chips-look-e2e.cjs`, modeled on
   `scripts/obsidian-copy-chips-look-e2e.cjs`, and call it from
   `scripts/obsidian-command-chips-e2e.cjs` (plus its offline self-test). In the
   isolated scratch vault with the stub `bob`, check:
   - one labeled ready chip as a Live Preview span and as a Reading button computes one
     look at rest and after a trusted CDP hover;
   - its well's background is the computed `--interactive-accent`;
   - the tooltip is visible about 0.5 s after hovering;
   - a trusted click that the stub completes turns the well green;
   - a chip in a `ref/` note shows the decided copy-only look.

   Save screenshots to `--out`. Run it on athena: copy the working tree to a fresh
   `/tmp` directory (never into `~/bob`) and run
   `xvfb-run -a node --experimental-websocket scripts/obsidian-command-chips-e2e.cjs --out /tmp/command-chips-e2e`.
   If it cannot run, say so plainly in the final report; never claim it passed.

6. **Docs, version, build, deploy.** Keep `docs/bob-chips.md` under the reference cap:
   it is at 5,434 of 6,000 lint words, so keep the net growth under about 300 words.
   - In `docs/bob-chips.md`:
     - **Command chips:** change "a quiet play pill" to the solid run well.
     - **States table:** rewrite it per this plan's States table, as "Well and ink".
     - **Geometry:** replace the "quiet geometry" paragraph with the capsule recipe, the
       `--bob-run-chip-hue` knob, the one-look rule, the no-stranded-well rule, and the
       progress bar on the face.
     - **⌘↵ and the hint:** update the armed bullet (full-ink border and a soft halo).
     - **The bob-run card:** describe the well, tinted header, Run button, state inks,
       and narrow wrap.
     - **Look and feedback:** update the Make-or-edit sentence ("the COMMAND CHIP header
       is the solid run well").
   - In `docs/develop/validation.md`, update "Command chip preview and e2e" for the
     shared baseline, the new checks, and the look cases.
   - Bump `plugins/bob-chips/manifest.json` to the next free minor (0.16.0 unless master
     has moved).
   - Run `npm run build`, `npm run docs:gen`, `npm test`, `npm run validate`, and
     `npm run docs:check`.
   - Commit, then run `bob plugins sync` (follow the repo's refusal rules; never `-F` on
     your own).
   - Close **bob-cli-od** with
     `sase bead close bob-cli-od --note "<what you verified>"`, citing the shared
     baseline and the rerun previews.
7. **Glossary (bob-cli; decision `glossary_command_chip`).**
   - In `sase/memory/glossary/command-chip.md`, replace "renders the span as a quiet
     pill with a ▶ glyph" with the new look, e.g. "renders the span as a pill with a
     solid accent ▶ run well". Change nothing else in the strand.
   - Run `sase memory init`, then `uv run --project tools/docs bob-docs gen` to
     regenerate `docs/reference/glossary.md`, then `just docs-check`. A
     `vault-format.md` word-budget failure there predates this work (bob-cli-oc); report
     it, don't fix it here.

## Acceptance

- In the vault, `gtd_daily`'s `#sync`, Bank, and `#init` chips show a solid accent ▶
  well and a tinted body. They read the same in Live Preview and in Tasks/Reading
  renders, and stand apart from the blue copy wells, cyan inline code, and purple links
  on Blocked, Next, and WIP rows, in light and dark.
- Every run state is legible from the well alone:
  - armed: halo;
  - queued: hourglass and dashed border;
  - running: spinner and the progress bar on the face;
  - completed: green check;
  - attention: orange well, black ▶;
  - failed: red well;
  - unavailable: pale ⊘;
  - copy-only: per the decision.
- No layout shift on hover, focus, any state change, or copy feedback. The well never
  sits alone at a line end, the end cap never wraps alone, and long faces still wrap.
- `bob-run` cards carry the same identity. Their titles never squeeze on a narrow card,
  and their Run button never reflows the header.
- Print/PDF output is unchanged plain inline code and plain listings; with the plugin
  off, chips are unchanged inline code.
- Copy chips and `bob-copy` cards are byte-identical in CSS and pass their preview.
- `npm test`, `npm run validate`, `npm run docs:check`, all three previews, and (where
  runnable) the real-Obsidian command-chip e2e pass. Bundle and docs are fresh, the
  plugin is deployed with `bob plugins sync`, and bob-cli-od is closed.
