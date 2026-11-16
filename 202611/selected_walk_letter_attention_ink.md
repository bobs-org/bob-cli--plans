---
tier: tale
title: Selected walk letter wears its attention hue
goal:
  The selected `s`/`w`/`b` selector letter takes its attention dot's hue as readable ink
  whenever that dot would show off-family.
size: medium
decisions:
  walker_footer_glossary:
    ask:
      Should the walker-footer glossary strand also say the selected letter wears its
      attention hue as ink?
    memory:
      - glossary:walker-footer
    default: false
    answer: true
proposed_by: bbugyi200.apollo.ei
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-16 17:53:59
status: wip
---

# Plan: Selected walk letter wears its attention hue

## Goal

The `[[ s w n b ]]` bracket-walk selector, painted by bob-navigation-hotkeys in the
desktop status bar beside the bob-ledger-tools walker footer, already shows a colored
attention dot on the `s`, `w`, and `b` letters. The dot appears only when that family is
**not** the selected walk and it has work to review:

- `s`: the hue of the first due review commitment tier (PRE cyan, NEW red, PROJECTS
  purple, PENDING yellow, NEXT green, RECURRING pink, TICKLER blue).
- `w`: red while any PENDING/NEXT/READY lane is over its cap.
- `b`: amber (`--text-warning`) while bank rows are waiting.

After this change, the **selected** letter itself takes that same hue whenever its dot
would show if the family were not selected. For example, if `s` has a red (NEW) dot and
Bryan presses `]s`, the dot goes away and the `s` letter turns red. The text uses a
readable version of the hue: same hue, lightness adjusted for the theme. The ink follows
the same data as the dot. It changes tier as the walk clears commitments and drops back
to the normal active-letter color once nothing would light a dot.

All code changes are in the `bob-plugins` linked repo. Open it with `/sase_repo`
(`sase repo open bob-plugins -r "..."`) and read its `AGENTS.md` first. Both plugins use
the committed source build: edit `src/` fragments, run `npm run build`, and never
hand-edit `main.js`.

## How it works today

- **Nav view model**: `plugins/bob-navigation-hotkeys/src/547a-bracket-mode-view.js`.
  `buildBracketModeView(type, attention)` sets each segment's `attention` (dot visible)
  to `raw present && item.type !== normalized`. The selected family's attention is
  hidden, and its `attentionTier`/`attentionLabel` are `""`. `bracketModeViewKey(view)`
  only covers segments that have a dot.
- **Nav painter**: `src/547-bracket-mode-indicator.js` `paintBracketModeIndicator`. It
  toggles `has-attention`, sets or removes `data-attention-tier`, and sets
  `title`/`aria-label` from the segment.
- **Nav api**: `src/482-walk-mode-nav-api.js` `createWalkModeApi`, now `version: 5`.
  `setAttention(type, value)` stores attention per family (`s`/`w`/`b`) in
  `plugin.bracketModeAttention` and repaints. The selected letter's attention is already
  stored. The view simply hides it.
- **Nav CSS**: `plugins/bob-navigation-hotkeys/styles.css`.
  - The `button.bob-bracket-mode-seg.has-attention::after` dot uses
    `--bob-review-attention`.
  - Per-tier rules `button.bob-bracket-mode-seg.has-attention[data-attention-tier="…"]`
    set that variable for pre/new/projects/pending/next/recurring/tickler/over (near the
    bracket-mode block).
  - A separate `waiting` rule sits near the bank-walk styles (`--text-warning`).
  - `button.bob-bracket-mode-seg.is-active` sets
    `color: var(--text-normal); font-weight: 600`.
  - The active letter sits on the `.bob-bracket-mode-thumb` pill, a 20%
    `--interactive-accent` wash.
  - The forced-colors block sets the active letter to `HighlightText`.
- **Ledger pushes**: three helpers, each called on every `updateFreshnessStatusBar`
  paint (`src/240-plugin-freshness-status.js`). Each one **explicitly pushes `null` when
  its own family is selected**:
  - `walkerFooterPushReviewAttention` in
    `plugins/bob-ledger-tools/src/122-walker-footer-host.js` (`route.mode === "s"`),
    label `Review: N due · K commitments — TIER first; click to show`.
  - `walkerFooterPushLaneAttention` in `src/123-lane-walker-footer.js`
    (`route.mode === "w"`), label `Lanes over limit: WIP 12/10 · … — click to show`.
  - `walkerFooterPushBankAttention` in `src/125-bank-walker-footer.js`
    (`route.mode === "b"`), label `Bank: N waiting · oldest X — click to show`.
  - The unload callback in `240-plugin-freshness-status.js` clears all three. Keep it.
- **Readable-ink recipe**: ledger already paints readable tier text in the review footer
  (`plugins/bob-ledger-tools/styles.css`, `.bob-freshness [data-review-tier]` and
  `.bob-freshness-answer`):
  - Fallback: `color-mix(in srgb, hue 60%, var(--text-normal))`.
  - Under `@supports (color: oklch(from red l c h))`:
    `oklch(from hue L min(c, 0.16) h)`.
  - The answer chip, which sits on a tinted wash like the selector thumb, uses
    `L = 0.44` light / `0.84` dark (`.theme-dark`).

## Design

### 1. Ledger: keep pushing attention for the selected family

In each of the three push helpers, delete the `route.mode === <family>` branch that
pushes `null`. Compute the attention exactly as for the off-family case, then push it.
Keep every other `null` path unchanged: no route or a legacy route, tasks/lanes/bank
unavailable, zero commitments, no over-limit lane, no waiting rows, ROTTEN/POST-only
queues. Keep the existing version gates (`>= 3` for `s`, `>= 5` for `w`, `>= 4` for `b`)
and the unload clearing.

When the family is selected, push the same `tier` with the click hint removed from the
label, because clicking the selected letter does nothing:

| Family | Off-family label (unchanged)                                                             | Selected-family label                                    |
| ------ | ---------------------------------------------------------------------------------------- | -------------------------------------------------------- |
| `s`    | `Review: 9 due · 5 commitments — NEW first; click to show`                               | `Review: 9 due · 5 commitments — NEW first`              |
| `w`    | `Lanes over limit: WIP 12/10 — click to show`                                            | `Lanes over limit: WIP 12/10`                            |
| `b`    | `Bank: 3 waiting · oldest Nov 14 — click to show` (or `Bank: 3 waiting — click to show`) | `Bank: 3 waiting · oldest Nov 14` (or `Bank: 3 waiting`) |

Build the labels from their parts with an optional suffix. Do not strip text from a
finished string. No new nav-version gate is needed, because both mixed deployments are
harmless:

- Older nav (v3–v5) already hides the selected family's attention.
- Older ledger keeps pushing `null` for the selected family, so new nav shows no ink.

Update the helper comments to say this.

### 2. Nav view model: dot off-family, ink on the selected family

In `buildBracketModeView`:

- Let `present = raw is a non-null object`, and `selected = item.type === normalized`.
- Keep `attention` meaning "dot visible": `present && !selected`. Existing callers, such
  as the walker-footer preview template and e2e queries, depend on it.
- Add `attentionInk: present && selected`.
- Fill `attentionTier` and `attentionLabel` when `present`, whether the segment shows a
  dot or ink. Keep the per-family default labels.
- Set `ariaLabel` to `${segmentAria}. ${label}` when `present`.
- With mode unset (`none`), no letter is selected, so only dots show. `n` never gets
  attention, because `setAttention` refuses it.

Update `bracketModeViewKey` so that every segment with a dot **or** ink contributes its
kind (`dot`/`ink`), tier, and label. A tier or label change on the selected letter must
then repaint, and an unchanged view must still skip DOM writes.

### 3. Nav painter

In `paintBracketModeIndicator`:

- Toggle a new `has-attention-ink` class from `attentionInk`. Leave `has-attention` for
  the dot only.
- Set `data-attention-tier` while a dot **or** ink shows, and remove it otherwise.
- Set `title` to the attention label while a dot or ink shows, and remove it otherwise.
- Changing mode must move a letter cleanly from ink to dot and back, with no stale class
  or attribute. For example, select `s`, then `w`, then `s`.

### 4. Nav CSS: same hue, readable ink

In `plugins/bob-navigation-hotkeys/styles.css`:

- Make every tier hue rule apply to both dot and ink, including `waiting` near the
  bank-walk styles. Either list `.has-attention-ink[data-attention-tier="…"]` next to
  each `.has-attention[...]` selector, or key the rules on
  `button.bob-bracket-mode-seg[data-attention-tier="…"]` alone, since the painter sets
  that attribute only while a dot or ink shows. Update the existing CSS-regex tests to
  match whichever form you choose.
- Add the ink, reusing ledger's readable-ink recipe with the answer-chip lightness,
  because the letter sits on the accent-tinted thumb:
  ```css
  .bob-bracket-mode {
    --bob-attention-ink-l: 0.44;
  }
  .theme-dark .bob-bracket-mode {
    --bob-attention-ink-l: 0.84;
  }
  button.bob-bracket-mode-seg.is-active.has-attention-ink {
    color: color-mix(
      in srgb,
      var(--bob-review-attention, var(--text-accent)) 60%,
      var(--text-normal)
    );
  }
  @supports (color: oklch(from red l c h)) {
    button.bob-bracket-mode-seg.is-active.has-attention-ink {
      color: oklch(
        from var(--bob-review-attention, var(--text-accent)) var(--bob-attention-ink-l)
          min(c, 0.16) h
      );
    }
  }
  ```
  The selector `.is-active.has-attention-ink` outranks the `:hover` rule and the
  `data-bracket-flash="empty"` rule, so hovering never drops the ink. Keep the thumb
  pill, font weight, and geometry unchanged. Add no animation or transition.
- In `@media (forced-colors: active)`, add
  `button.bob-bracket-mode-seg.is-active.has-attention-ink { color: HighlightText; }` so
  forced colors still win over the ink.

### 5. Nav api version

Bump `createWalkModeApi` to `version: 6` and extend its header comment: "v6 paints the
selected family's attention as letter ink instead of hiding it; ledger keeps pushing
attention for the selected family". Ledger does not gate on v6, as explained in §1.

## Tests

Run the targeted files while iterating, then the full gates listed under "Build, docs,
deploy".

`scripts/test-navigation-bracket-mode-indicator.cjs`:

- Rewrite "per-family attention lands on s and b only while another family is selected":
  - Off-family segments keep the dot.
  - The selected family has `attention === false`, `attentionInk === true`, and its tier
    and label filled in.
  - `w`/`n` segments without attention stay empty.
  - Unset mode has no ink.
- Apply the same rewrite to the `w` test ("the w dot paints only while mode is not w…").
- Painter: with mode `s` and `s` attention, the `s` button has `has-attention-ink`, does
  not have `has-attention`, and has `data-attention-tier` and `title`. Selecting `s` →
  `w` → `s` swaps ink and dot cleanly. Clearing attention removes the ink class, the
  attribute, and the title.
- View key: a tier change on the selected letter changes the key. An identical view
  still causes zero DOM writes.
- CSS:
  - The ink rule exists, reads `--bob-review-attention`, and has no
    `animation`/`transition`.
  - The `@supports` oklch variant exists.
  - Every tier hue (including `waiting` and `over`) applies to `.has-attention-ink`.
  - The forced-colors block covers the ink with `HighlightText`.
  - Keep "s dot hues match the review footer tier hues" passing.
- walkMode test: `api.version === 6`.

`scripts/test-ledger-tools-walker-footer.cjs`:

- "mode s, legacy routes, and unavailable tasks push null": on `s`, expect a non-null
  push with the first commitment tier and a label that does **not** match
  `/click to show/`. Legacy and unavailable cases stay `null`.
- "updateFreshnessStatusBar pushes non-null off-s and null on s": expect non-null on `s`
  too, with the same tier.
- "bank attention is waiting-amber off-b and null on b…": on `b`, expect
  `{tier: "waiting"}` with a label lacking `click to show`. Zero rows and old nav are
  unchanged.
- "lane attention lights off-w and clears on w…": on `w`, expect `{tier: "over"}` with
  `Lanes over limit: WIP 12/10` and no click hint. Other cases are unchanged.
- Rename these tests to match the new behavior. Unload tests stay as they are.

`scripts/obsidian-walker-footer-e2e.cjs`: `dotOnS === false` still holds, because `s`
shows ink instead of a dot. Add an ink check: after `api.select("s")` with the fixture's
commitments due, `button[data-bracket-type="s"].has-attention-ink` exists and its
`data-attention-tier` equals the `dotTier` seen off-`s`. Include it in the pass verdict.
If the fixture has no due commitments on `s`, assert the `b`/`w` ink instead where the
fixture makes that deterministic. Run the live e2e if the local Obsidian harness is
available (see `docs/develop/validation.md`). Otherwise keep its `--self-test` green and
report that the live run was skipped.

## Visual acceptance

Extend `scripts/preview-bracket-mode.cjs` with cases for:

- mode `s` with `s` attention for each commitment tier: one row per tier.
- mode `w` with `w: over`.
- mode `b` with `b: waiting`.
- mode `s` with all three attentions: ink on `s`, dots on `w` and `b`.

Render light and dark. Dark must also carry the `theme-dark` class, so that
`--bob-attention-ink-l` switches. The preview body currently uses only `data-theme`.

In the preview's `measure()`, record each inked letter's contrast against the
thumb-over-status-bar color. Resolve computed colors to sRGB, for example with a canvas
`fillStyle` read-back, and require at least 4.5:1 in both themes for every tier. If a
tier falls short, adjust `--bob-attention-ink-l` (lower in light, higher in dark)
instead of changing the hue. Look at the screenshots before finishing.

## Build, docs, deploy

1. Bump minor versions from whatever is current when you start: bob-navigation-hotkeys
   (2.85.2 → 2.86.0 at the time of writing) and bob-ledger-tools (1.122.0 → 1.123.0).
2. Update the nav `manifest.json` description: "per-family `s/w/b` attention through nav
   `api.walkMode` v5" becomes "per-family `s/w/b` attention (a dot off-family, its hue
   as letter ink on the selected family) through nav `api.walkMode` v6".
3. Update the hand-written docs:
   - `docs/bob-navigation-hotkeys.md` "Footer indicator":
     - Add v6 to the walkMode list.
     - Replace "Dots clear on selection or when nothing waits" with: on selection, a
       family's dot becomes readable ink on its own letter; dots and ink clear when
       nothing waits. Labels appear in tooltips; `Highlight`/`HighlightText` apply when
       colors are forced.
   - `docs/bob-ledger-tools.md` walker-footer paragraph: ledger pushes `s`/`w`/`b`
     attention for every family, including the selected one, and selected-family labels
     drop the click hint.
   - `docs/develop/plugin-apis.md` walkMode paragraph: describe v6's ink semantics and
     note that ledger needs no v6 gate.
4. Run `npm run build`, `npm run docs:gen`, `npm test`, and `npm run docs:check`.
5. Run `bob plugins sync`. If it refuses an overwrite, follow `AGENTS.md` exactly: never
   pass `-F` on your own.

## Memory

> [!decision] walker_footer_glossary Use `/sase_memory_write` first. In bob-cli's
> `sase/memory/glossary/walker-footer.md`, extend the sentence "Off-family dots mark the
> selector: `s` while review commitments are due, `w` in the badge red while any
> PENDING/NEXT/READY lane is over its limit, and `b` in amber while bank rows wait."
> with one clause: the selected family's own letter wears that same hue as readable ink
> instead of a dot. Then run `sase memory init`, then `just docs-gen` so
> `docs/reference/glossary.md` regenerates, and run `just docs-check`.

> [!decision] walker_footer_glossary = no Make no memory edits.

## Acceptance criteria

- Pressing `]s`, or clicking `s`, while a review commitment is due turns the `s` letter
  into readable ink of that dot's tier hue (for example, NEW → red) and shows no `s`
  dot. As the walk clears tiers, the ink follows the first due commitment tier. With
  only ROTTEN/POST left, or nothing due, the letter returns to the normal active color.
- With `w` selected and a lane over its cap, `w` is red ink. With `b` selected and
  waiting bank rows, `b` is amber ink. `n` never inks. Unset mode shows dots only.
- Switching families moves the indicator at once: the old letter regains its dot and the
  new letter inks, with no flicker of stale classes.
- The inked letter's tooltip and aria label give the attention label without "click to
  show". Hover keeps the ink. Forced colors use `HighlightText`. Nothing animates.
- Legacy routing (nav without `walkMode`) and plugin unload still clear all attention.
  Mixed old/new plugin versions degrade to today's behavior.
- Ink contrast is at least 4.5:1 against the thumb in light and dark for every tier. All
  unit tests, build/docs checks, and `bob plugins sync` pass.

## Out of scope

- Tinting the thumb pill, changing the dot itself, or adding attention for `n`.
- Changing which tiers or conditions raise attention.
- Walker-footer panel changes.
