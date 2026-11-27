---
tier: tale
size: medium
title: Fix the snooze definition clock's rendering, then land bob-cli-q5.5.1
goal:
  Snoozed task definitions show a one-line, state-tinted Lucide alarm clock with its
  time (and `· due` when due) in real Obsidian, proven by layout and paint e2e checks
  and deployed; then bob-cli-q5.5.1 and its parent phase bob-cli-q5.5 close.
proposed_by: bbugyi200.athena.bob-cli-q5.5.1.land
proposed_with: claude/opus
bead: bob-cli-q5.5.1
create_time: 2026-11-26 22:47:30
status: wip
---

- **PARENT:**
  [202611/snooze_display_marks.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/snooze_display_marks.md)
- **BEAD:**
  [bob-cli-q5.5.1](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-q5/bob-cli-q5.5.1.md)

# Fix the snooze definition clock's rendering, then land bob-cli-q5.5.1

## Why this tale exists

The bob-cli-q5.5.1 land agent verified the child epic "Snooze display marks and plan
chip" (`plan:202611/snooze_display_marks.md`). All six phases are closed, all 70 `SM-*`
vectors run verbatim, and the vault bundle matches bob-plugins `a4bd99f2`. The landing
also committed bob-cli doc integration: the `docs/snooze-marks.md` "JavaScript mirror"
section, a `docs/today-marks.md` cross-link, and a `docs/plan.md` chip pointer.

The land agent's review of the attached real-Obsidian screenshots found that the
**definition clock**, already deployed from bob-plugins `27157a49` (bob-cli-q5.5.1.2),
does not render as the contract requires. These are this epic's defects and must be
fixed before it closes. The visual-proof e2e passed only because it reads class names
and data attributes, not rendered layout or computed paint.

Defects, all in `plugins/bob-ledger-tools` of the bob-plugins linked repo:

1. **The time wraps vertically and overlaps neighbouring lines.** `.bob-snooze-mark` in
   `styles.css` (around lines 3319-3352) copies the Today mark box:
   `display: inline-flex; width: 1.2em; height: 1.2em; justify-content: center`.
   `::after { content: attr(data-bob-snooze-time) }` is squeezed into that 1.2em width,
   so the time renders one character per line above and below the clock. See screenshots
   `e2e-snooze-marks-live-light.png`, `-live-dark.png`, and `-hover.png`, attached to
   bob-cli-q5.5.1.6 notes #3/#4 and to bob-cli-q5.5.
2. **No state color ever paints.** The CSS styles `.bob-snooze-mark--asleep`, `--due`,
   and `--blocked`. But `buildSnoozeMarkElement` (`src/143a-snooze-definition-model.js`)
   and the Reading path (`src/277b-plugin-snooze-marks-reading.js:205`) emit
   `bob-snooze-mark--indigo|amber|muted`. No selector matches.
3. **Due clocks never say `due`.** `buildSnoozeMarkElement` accepts only a bare
   `^\d{2}:\d{2}$` time, and `::after` prints only that. The contract and SM-D02/SM-D03
   expect visible `⏰ 15:30 · due`.
4. **Wrong glyph.** The contract and the epic plan require the Lucide `alarm-clock`
   drawn through a CSS mask from `--bob-snooze-mark-glyph`, tinted `currentColor`. Both
   `.bob-snooze-mark::before` and `.bob-snooze-woke::before` render the color emoji `⏰`
   with `mask: none`, so tone never reaches the glyph.

The rows, heading summary, woken pills' layout, and plan chip render correctly; see
`e2e-snooze-marks-rows*.png`, `-reading-*.png`, and `-plan.png`. Do not restyle them
beyond the shared glyph change below.

## Governing decisions

- `decisions:today-is-read-from-the-ledger`: display-only work. Today membership and the
  🍅-wins-over-clock precedence stay unchanged.
- `decisions:task-lanes-are-sticky`: no display or navigation path changes a lane.
- `decisions:bob-cli-releases-ride-a-daily-train`: never edit `Cargo.toml` versions or
  `CHANGELOG.md` in bob-cli.

The epic's reviewer decisions remain final (`uncommitted_rows=ignore`,
`wake_notice=hud`, `hold_in_meetings=yes`, `glossary_snoozed_task=yes` belongs to the
outer phase bob-cli-q5.9, `decision_record=no`). No memory note changes here.

## Work

Open bob-plugins with `/sase_repo` (`sase repo open bob-plugins -r "<why>"`) and read
its `AGENTS.md`. Edit `src/` fragments and `styles.css`, then run `npm run build`. Never
hand-edit `main.js`. Keep every hand-edited fragment at or below 1,000 lines (`277a` is
727, `143a` is 500).

### 1. Clock box and text (defects 1 and 3)

- Let the definition clock grow with its text while keeping the line box: keep
  `height: 1.2em`, `font-size: 0.85em`, `line-height: 1`, `vertical-align: -0.14em`, and
  the margins. Replace the fixed `width: 1.2em` with `min-width: 1.2em; width: auto`,
  add `white-space: nowrap`, a small `gap`/`padding-inline`, and
  `justify-content: flex-start`. Mirror the already-correct `.bob-snooze-woke` box.
- Render the visible text from the SM-D model: `15:30` for asleep and blocked,
  `15:30 · due` for due. Keep `data-bob-snooze-time` as the bare `HH:MM`, because
  existing tests and the e2e read it. Either add a separate `data-bob-snooze-label` that
  `::after` prints, or use a tone-scoped `::after` content suffix. Live Preview (`277a`)
  and Reading (`277b`) must produce identical hosts. Widget `eq` must still swap the DOM
  when the label changes and must never replay the pulse.

### 2. Tone classes (defect 2)

- Make CSS and markup agree on one vocabulary. Emitting the state names
  (`--asleep|--due|--blocked`) is preferred because it matches the contract prose. Map
  model tones `indigo→asleep`, `amber→due`, `muted→blocked` in one shared helper used by
  `143a` and `277b`. Keep the `tone` fields in model output unchanged, because the SM-D
  vectors assert them verbatim.
- Update the e2e's `toneOf` reading accordingly and assert paint, not class names
  (section 4).

### 3. Shared Lucide alarm glyph (defect 4)

- Define `--bob-snooze-mark-glyph` on `body` as a `url("data:image/svg+xml,…")` mask
  image of Lucide `alarm-clock` with an opaque stroke: `circle cx=12 cy=13 r=8`, paths
  `M12 9v4l2 2`, `M5 3 2 6`, `m22 6-3-3`, `M6.38 18.7 4 21`, `M17.64 18.67 20 21`,
  `viewBox 0 0 24 24`,
  `fill='none' stroke='black' stroke-width='2' stroke-linecap='round' stroke-linejoin='round'`.
  Follow the existing `--bob-priority-glyph-*` mask-variable pattern in `styles.css`.
- `.bob-snooze-mark::before` and `.bob-snooze-woke::before` become `content: ""`, an
  inline-block `1em × 1em` box with `background-color: currentColor`,
  `-webkit-mask-image`/`mask-image: var(--bob-snooze-mark-glyph)`, and
  `mask-size: contain`, `no-repeat`, `center`. Drop the emoji font stack.
- In the plan chip (`src/290c-snoozed-plan-chip.js`), render the model's leading `⏰ `
  through the same masked glyph span followed by the remaining text. Keep
  `snoozePlanMeterModel` `text` unchanged, because SM-P vectors assert it. Keep the
  chip's `aria-label` tooltip exactly as today.
- Forced colors: keep a visible glyph
  (`forced-color-adjust: none; background-color: CanvasText` on the mask box), the
  existing `1px solid CanvasText` outline, and readable state text. Reduced motion stays
  animation-free.
- Update the CSS header comments that currently claim a mask is used.

### 4. Tests that catch this class of defect

- Unit (`scripts/test-ledger-tools-snooze-definition-marks.cjs`): assert the emitted
  state class for each SM-D tone, and the due label/attribute. Assert that Live Preview
  and Reading hosts carry identical attributes.
- Real Obsidian (`scripts/obsidian-snooze-marks-e2e.cjs`): add checks that fail on the
  current bundle and pass after the fix.
  - Each visible definition clock is one line: `getBoundingClientRect().height` is at
    most the host line's single-line height, its width exceeds 2.5em, and the row's
    line-height check still passes.
  - `getComputedStyle(el, "::after").content` contains `· due` for the due clock and not
    for asleep/blocked.
  - Computed `color` differs between asleep, due, and blocked clocks.
  - `getComputedStyle(el, "::before")` has a non-`none` `mask-image` (or
    `webkitMaskImage`) for definition clocks, woken pills, and the plan-chip glyph.
  - The forced-colors pass still shows a non-transparent glyph background.
- `/tmp` on athena is a full 32G tmpfs. Run the e2e with `TMPDIR` pointing at a
  directory on the home disk, e.g.
  `mkdir -p ~/.cache/sn-e2e-tmp && TMPDIR=~/.cache/sn-e2e-tmp xvfb-run -a node scripts/obsidian-snooze-marks-e2e.cjs --out <dir> --report <file>`.
  Rust or npm suites that hard-code `/tmp` may fail with ENOSPC unrelated to this work.
  Record any such failure with its command rather than treating it as yours.

### 5. Contract wording in bob-cli

Read `docs.md` through `/sase_memory_read`, then amend `docs/snooze-marks.md` "Glyph
anatomy" (and the woken-pill anatomy sentence if needed) to match the shipped design:

- `--bob-snooze-mark-glyph` holds the mask image, tinted `currentColor`.
- The box keeps the Today mark's height, margins, and alignment but grows in width to
  fit the time and `· due`. Do not claim a fixed `1.2em × 1.2em` box.
- State classes are `bob-snooze-mark--asleep|due|blocked`.

Do not change `tests/fixtures/snooze/marks.json`; the vectors are already correct. Run
`just docs-check`. Its lint stage currently fails only on the pre-existing
`docs/reference/vault-format.md` budget (tracked as bob-cli-oc). When that is still the
only lint error, also run `uv run --project tools/docs bob-docs build`,
`… bob-docs links`, and `… bob-docs urls` to cover the later stages.

### 6. Gates and deploy

In bob-plugins, with `BOB_CLI_DIR` set to this bob-cli checkout:

- `npm run build`, `npm run build:check`, `npm run validate`
- `node --test scripts/test-ledger-tools-snooze-*.cjs scripts/test-obsidian-snooze-marks-e2e.cjs`
- `npm test`; record environmental ENOSPC failures separately
- the real-Obsidian e2e above. All checks must pass except `reading-tomato-regression`,
  a pre-existing Today Reading defect tracked as bob-cli-qs and reproduced on the
  pre-epic bundle.

Inspect the new light/dark Live Preview, Reading, hover, narrow, and a11y screenshots
yourself. Confirm each clock reads `⏰ HH:MM` (or `⏰ HH:MM · due`) on one line in its
tone. Attach the report and screenshots with
`sase bead attach bob-cli-q5.5 -K -n "<what they prove>" <files…>`. Then run
`bob plugins sync` from the bob-plugins checkout and follow its `AGENTS.md` refusal
rules (never `-F` on your own). Confirm the vault's
`.obsidian/plugins/bob-ledger-tools/main.js` and `styles.css` match the repo build.

Use `/sase_monitor` for anything that could outlive a foreground call. Do not leave
subprocesses running.

## Closeout (final step, same turn as the code)

Do this after the gates pass, without waiting for this work's own commit, push, or CI:

1. Run `sase bead epic-symbols bob-cli-q5.5.1`. Resolve every listed `--epic-symbol`
   entry by wiring it up, privatizing it, adding a non-test pragma, or deleting it, or
   re-key it to a still-open bead that needs it. At planning time there were none.
2. Close the epic:
   `sase bead close bob-cli-q5.5.1 --note "<verified: phases, SM groups, the clock rendering fix with before/after e2e evidence, gates, deploy, follow-up triage recorded in note by bob-cli-q5.5.1.land>"`.
   Never use `--force` merely to make it succeed.
3. Run `just symvision` if the recipe exists (bob-cli had none at planning time).
4. Set `status: done` in the frontmatter of the epic's plan file
   (`plan:202611/snooze_display_marks.md`, the PLAN path shown by
   `sase bead read bob-cli-q5.5.1 -r "<why>"`).
5. The epic's `parent_bead` is the **phase** bob-cli-q5.5 of outer epic bob-cli-q5.
   Confirm its scope is met: definition clock, woken pill, row and heading paint, plan
   chip, the `docs/snooze-marks.md` contract with vectors, and the design-bar evidence
   now attached to it. Run `sase bead epic-symbols bob-cli-q5.5` and resolve or re-key
   any entries. Then close only that phase with
   `sase bead close bob-cli-q5.5 --note "<what you verified>"`. Do **not** close
   bob-cli-q5 or any other ancestor; its own land agent owns it.

## Out of scope

- The Reading-view Today 🍅 defect (bob-cli-qs) and the vault-format.md budget
  (bob-cli-oc) are tracked elsewhere.
- No CLI, wake-writer, lane, Today-membership, release-version, or changelog changes.
