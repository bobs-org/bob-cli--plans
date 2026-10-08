---
tier: tale
title: "Task tag marks: give the #task hash a distinct identity ink"
goal:
  "The #task hash glyph on open task lines renders in one constant, distinct identity
  ink with a slightly firmer stroke, so a tracked task is recognizable at a glance on
  plain, Next, In Progress, and Blocked lines in light and dark themes; closed tasks
  keep a soft ghost of the same hue; nothing else about task tag marks changes."
size: small
decisions:
  ink:
    ask: "Which hue should the #task hash glyph wear on open task lines?"
    default: teal
    why:
      "Teal complements the red Blocked tint on most open #task lines and parts from
      violet pills."
    choices:
      teal:
        A deep teal no status, lane, warning, or tag uses; crisp on the red Blocked tint
      tag: The theme tag color (violet by default); the glyph reads as the tag, like the
    answer: teal
proposed_by: bbugyi200.apollo.5t.f0
decided_by: auto
create_time: 2026-10-08 08:03:10
status: wip
---

# Task tag marks: give the `#task` hash a distinct identity ink

## Goal

The task tag mark (shipped in bob-ledger-tools 1.35.0) swaps every `#task` pill on a
task line for one slanted hash glyph. Its ink is `--text-faint` at 0.9 opacity, and 0.55
on closed tasks. Beside the muted priority and date marks it nearly disappears, and on
closed lines it is close to invisible. Bryan asked for it to stand out a little more,
with a distinct color. Give the glyph **one constant identity ink**: a hue that means
"tracked task" and nothing else. Also make the stroke slightly firmer so the color reads
at 0.8em, and keep a soft ghost of the hue on closed tasks. Shape, eligibility,
interaction, the toggle, the tooltip, and the hidden tag in Tasks query results all stay
unchanged.

```text
open:    - [?] ⌗ #prj Sase message boards!  ⟳ 3d  ▮▮▮      ⌗ = identity ink, 0.9 → 1 on hover with a 14% capsule
closed:  - [x] ⌗ Better %model support!  ✓ Jun 24            ⌗ = ghost: 40% hue + 60% text-faint, opacity 0.75
plain:   - [ ] Pack charger                                  no glyph (unchanged)
```

## Why a constant identity hue, and which one

Color on a Bob task line already carries state. The new ink must not mean any state, and
it has to hold up where `#task` lines actually appear. These are vault counts from
2026-10-08:

- There are 3,406 `#task` lines, and 2,934 of them are closed (`x`, `-`). The about 470
  open lines are mostly **Blocked** (`[?]`, 383 lines), with a red 13% line tint and a
  red border and checkbox. Next (`[*]`, 19 lines) is green-tinted, In Progress
  (`[/]`, 7) is yellow-tinted, and plain `[ ]` (63) has no tint.
- 656 `#task` lines carry another tag pill, usually `#prj`, rendered in the violet
  accent, directly beside the glyph.

The hues already in use mean these things:

| Hue           | Already means                                                                               |
| ------------- | ------------------------------------------------------------------------------------------- |
| red           | Blocked (`task-statuses.css` tint, blocked chips)                                           |
| orange        | attention: due freshness, ROTTEN, repair flags, the Pending chip                            |
| yellow        | In Progress (line tint, the half-ring progress mark)                                        |
| green         | Next (line tint), fresh `✓ today`                                                           |
| blue          | the Ready lane (Task Card `is-lane-ready` chip, `bob-plan` READY chip)                      |
| violet accent | tags (`#prj`, `#gtd`, …) and links                                                          |
| cyan          | inline code chips (vault snippet `inline-code.css`, `--inline-code-hue: var(--color-cyan)`) |
| grey          | priority and date marks, cancelled                                                          |

Rules for the ink, which apply to both choices: it is **constant**. It never follows
status, lane, priority, or freshness, so the glyph names what the line is (a tracked
task), never what state it is in. That job stays with the checkbox and line tint. It
never uses red, orange, yellow, green, or blue. It must read on the red Blocked tint.
Pink and orchid are rejected: they turn muddy on the red tint and look like a near-miss
beside the violet pills.

> [!decision] ink = teal

The ink is a deep teal, made by blending the theme cyan with the text color so it adapts
to both themes: about `#0a9391` in light (about 3.8:1 on white, above the 3:1 non-text
contrast floor) and a soft aqua of about `#79dedc` in dark. Teal is the complement of
the red Blocked tint, so it is crispest exactly where most open `#task` glyphs sit. It
is clearly apart from the violet `#prj` pill next to it, so the tracked-task mark no
longer looks like just another tag. It shares a hue family only with inline code. Code
is a bordered monospace chip, so the two never read alike, and Bryan already chose cyan
for code, which shows he likes this family.

```css
--bob-task-tag-hue: color-mix(
  in srgb,
  var(--color-cyan, #00bfbc) 72%,
  var(--text-normal)
);
```

The 72% weight is tunable by ±8 during live verification. Record the final value in the
CSS and the doc together.

> [!decision] ink = tag

The ink is the theme's tag color, so the glyph reads as the `#task` tag itself,
distilled to a hash. It follows any accent or theme tag customization and matches the
`#prj` and `#gtd` pills beside it. It must be captured on `body`, never on the host,
because the `a.tag` host overrides `--tag-color` to `transparent` on itself. A `body`
declaration computes with `body`'s values, and descendants inherit the result.

```css
--bob-task-tag-hue: var(--tag-color, var(--text-accent));
```

## Design (plugin CSS, `plugins/bob-ledger-tools/styles.css`, task-tag-marks block)

1. **Hue variable.** Declare `--bob-task-tag-hue` in the existing `body { … }` rule that
   defines `--bob-task-tag-glyph`, with the value chosen by decision `ink`. The theme
   hook `--bob-task-tag-color` stays, is unset by default, and still wins when set.
2. **Firmer stroke.** Change the glyph's `stroke-width='1.6'` to `stroke-width='1.8'` in
   the `--bob-task-tag-glyph` data URI. The path is unchanged. At 0.8em, a 1.6 stroke
   renders at about 1.4px, and antialiasing washes the color out at that width. At 1.8
   it is about 1.55px, still lighter than the family's 1.9.
3. **Rest (open task).** Set
   `--bob-task-tag-ink: var(--bob-task-tag-color, var(--bob-task-tag-hue))` in the
   shared host rule (`body.bob-task-tag-marks .bob-task-tag-mark`), with opacity 0.9
   (unchanged). There is still no resting background or pill.
4. **Hover.** The ink is unchanged, opacity goes to 1, and the capsule is
   `color-mix(in srgb, var(--bob-task-tag-ink) 14%, transparent)`, a soft wash of the
   same hue in place of today's grey. It must actually show on **both** hosts. Today the
   `a.tag` override's `background: none` (specificity 0,3,2) beats the hover rule
   (0,3,1), so reading view never shows a capsule. Fix this by adding
   `body.bob-task-tag-marks a.tag.bob-task-tag-mark:hover` to the hover rule's selector
   list, and set the a.tag override's `--tag-background-hover` to the same 14% mix.
5. **Resting (closed `x`, `X`, `-`).** Set
   `--bob-task-tag-ink: color-mix(in srgb, var(--bob-task-tag-color, var(--bob-task-tag-hue)) 40%, var(--text-faint))`
   with opacity 0.75, the family's resting opacity, in place of 0.55. When a task
   closes, the hash softens to a ghost of the hue rather than vanishing. Resting must
   win on **both** hosts. Today the `a.tag` override block redeclares
   `--bob-task-tag-ink` at 0,3,2, which beats the 0,3,1 resting selectors. That was
   harmless while both values were `--text-faint`, but with distinct inks, closed tasks
   in reading view would stay full-color. Delete that redeclaration (the shared host
   rule already sets the rest ink), and keep the resting rule after the hover rule.
6. **Comments.** Update the block's header comment from "one small, faint, monochrome
   hash" to the identity-ink wording, and name `--bob-task-tag-hue` and the stroke
   width. Leave the Tasks-results hide rule, reduced motion, and print rules unchanged.

No JavaScript changes: the widget, post-processor, tooltip, and toggle stay as they are.

## Implementation steps

### bob-plugins

Open it with `sase repo open bob-plugins` and read its `AGENTS.md`. `bob-ledger-tools`
is fragment-built, but this change touches only `styles.css`, which is not generated.

1. Apply design items 1–6 to the task-tag-marks block in `styles.css`.
2. `scripts/test-ledger-tools-task-tag-marks.cjs`: extend the `styles.css contract`
   test:
   - `--bob-task-tag-hue:` is declared exactly once, with the decision's value (teal:
     contains `var(--color-cyan, #00bfbc)` and `var(--text-normal)`; tag: contains
     `var(--tag-color, var(--text-accent))`).
   - The glyph data URI contains `stroke-width='1.8'` and not `stroke-width='1.6'`.
   - The rest ink is `var(--bob-task-tag-color, var(--bob-task-tag-hue))`.
   - The selector `body.bob-task-tag-marks a.tag.bob-task-tag-mark:hover` exists, and
     the hover capsule mixes `var(--bob-task-tag-ink) 14%`.
   - The resting rule mixes 40% of the ink source with `var(--text-faint)` at opacity
     0.75.
   - The `body.bob-task-tag-marks a.tag.bob-task-tag-mark {` block does not declare
     `--bob-task-tag-ink`.
   - The task-tag-marks block contains none of `--color-red|orange|yellow|green|blue` or
     `--task-status-`; slice the block from its `Task tag marks (task-tag-marks)` header
     to the next block's `Task Link In Progress marks` header. This is the
     constant-identity rule.

   Rename the test title to mention ink and hover.

3. `plugins/bob-ledger-tools/manifest.json`: bump the patch version (1.35.0 → 1.35.1 at
   planning time; if it has moved, bump the patch of the current version). In the
   description, change "a quiet hash task-tag mark" to "a teal hash task-tag mark" (tag
   branch: "an accent hash task-tag mark").
4. `README.md`: update the bob-ledger-tools plugins-table row (version and the same
   description wording) and the task-tag-mark paragraph. Replace "one small, faint,
   monochrome hash glyph" with the identity-ink wording, and add a sentence that closed
   tasks keep a faint ghost of the hue.
5. Run `npm run build:check`, `npm test`, and `npm run validate` from the bob-plugins
   root. Run the task-tag suite directly too
   (`node scripts/test-ledger-tools-task-tag-marks.cjs`). Two
   `test-navigation-roll-decay` failures predate this work. If they still fail, confirm
   they also fail on a clean tree (`git stash`) and report them as pre-existing; every
   other test must pass. Then deploy with
   `bob plugins sync --repo <the bob-plugins checkout path> -p bob-ledger-tools`. A bare
   sync from a linked checkout is refused by the foreign-checkout guard.

### bob-cli (docs only, no Rust)

Update `docs/task-tag-marks.md`, the authoritative contract:

- **What you see**: change "one small, faint, monochrome hash glyph" to the identity-ink
  wording, and change the diagram note `(⌗ ≈ the faint hash glyph)` to name the ink.
- **Principles**: replace "Quietest mark in the family: … faint ink and no resting
  background" with: small but distinct; no pill or resting background, because it
  repeats on every task line; one constant identity ink that never encodes state.
- **The glyph**: `stroke-width 1.8`, plus the updated CSS custom property string.
- **New section "Ink"**, after The glyph: the hue table, the constraints (constant;
  never red, orange, yellow, green, or blue; legible on the Blocked tint), the vault
  counts above, the chosen value with its light and dark approximations, the
  `body`-level `--bob-task-tag-hue` definition, and the `--bob-task-tag-color` hook.
- **Anatomy and tones**: update the table rows to rest
  `var(--bob-task-tag-color, var(--bob-task-tag-hue))` at opacity 0.9; hover with the
  same ink at opacity 1 and a 14% ink capsule on both hosts; resting with the 40%
  hue/faint mix at opacity 0.75, winning on both hosts. Replace "The mark ignores status
  accent colors (shape carries meaning; color whispers)" with "The ink is constant: it
  names what the line is, never its state."
- **Rejected alternatives**: replace the "**Accent or status hues** compete with the
  status line tints" clause with an ink bullet. It should reject faint grey (the 1.35.0
  ink: lost beside the grey marks and invisible on closed lines); status hues and blue
  (they would claim a state or the Ready lane); orange (attention and repair); pink and
  orchid (muddy on the Blocked tint, a near-miss beside violet pills); and the
  non-chosen `ink` option (teal branch: the tag accent makes the mark read as one more
  tag pill; tag branch: teal shares the inline-code hue and reads as decoration rather
  than as the tag).
- **Live verification**: replace "The glyph looks right beside the checkbox in light and
  dark themes" with checks that the ink reads clearly yet stays secondary to the title
  in light and dark themes; that it stays legible on Blocked (red tint), Next (green
  tint), and In Progress (yellow tint) lines; that it sits well beside a `#prj` pill;
  that the hover capsule appears in both Live Preview and reading view; and change
  "Closed tasks rest." to "Closed tasks show a faint ghost of the hue (reading view and
  Live Preview)."

Update `docs/README.md` too: change the guide-table description to "Task tag marks: the
teal hash glyph that replaces the `#task` pill" (tag branch: "accent hash glyph"). The
README.md contracts-table row needs no change.

## Acceptance

- Open-task `#task` glyphs render in the chosen identity ink at the 1.8 stroke in Live
  Preview and rendered views, and hover shows a same-hue capsule on both hosts.
- Closed-task glyphs show the 40% ghost at opacity 0.75 on both hosts.
- Nothing else changes: plain checkboxes, non-task lines, Tasks results (still hidden),
  the toggle, the tooltip, and eligibility.
- The task-tag-marks CSS block uses no status, lane, or warning color variable, and the
  extended contract test enforces this.
- `npm run build:check`, `npm test` (except the pre-existing failures, verified against
  a clean tree), and `npm run validate` pass. The plugin is deployed with
  `bob plugins sync --repo … -p bob-ledger-tools`. `docs/task-tag-marks.md` and
  `docs/README.md` describe the new ink.
