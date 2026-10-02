---
tier: tale
title: Make every Pomodoro menu-bar color legible on any menu bar
goal:
  The Pomodoro status item paints only colors that pass a tested legibility contract on
  both light and dark macOS menu bars, using one appearance-independent, blue-free,
  isoluminant ten-stop countdown gradient with bold digits.
size: medium
proposed_by: bbugyi200.apollo.48.f1.f1
create_time: 2026-10-02 16:25:42
status: wip
---

# Make every Pomodoro menu-bar color legible on any menu bar

## Outcome and scope

Every color the Hammerspoon Pomodoro status item paints must stay readable whether the
Mac's menu bar is currently rendering light or dark. Replace the appearance-switched
dark/light gradient pair with **one appearance-independent ten-stop palette**. Its stops
share the same luminance, run from teal (fresh) through green, gold, and orange to red
(nearly exhausted), and use no blue. Draw the countdown digits in a bold monospaced face
so the color carries enough ink at menu-bar size. Move every other fixed title color
(overdue red, the flashing `OVERDUE` badge, `NO POMODORO` green) into the same vetted
luminance band. Add a regression test that proves every color any title state can use
meets the legibility contract.

This is a **medium tale** for one coding agent. All source changes belong in the linked
**chezmoi** repository. Start with the `/sase_repo` skill:

```sh
sase repo open chezmoi -r "Implement the approved legible Pomodoro gradient plan"
```

Use the printed checkout path and read its `AGENTS.md`. Paths below are relative to that
checkout. bob-cli, SASE, the ledger, capture contracts, and task statuses need no
changes. Keep everything else from the approved tomato/gradient design: title text and
segment order (`theme (duration) · 🍅 countdown`), the exact-tenths bucket math, the
duration rules, overdue precedence and flashing cadence, sync/wake/refresh lifecycle,
tooltip and dropdown, and the plain-text fallback.

## Diagnosis

At planning time chezmoi `HEAD` is `c7b88df8848f0fdedf3ca653d0f1bcc4a9609046`
(`feat(hammerspoon): put tomato beside countdown with ten-color time gradient`), and
`busted --no-coverage ./tests/hammerspoon` passes **62 successes / 0 failures**.

- `home/dot_hammerspoon/init.lua` picks `GRADIENT_DARK_COLORS` or
  `GRADIENT_LIGHT_COLORS` from `resolveBobPomodoroInterfaceDark()`, which wraps
  `hs.host.interfaceStyle()`. Hammerspoon's source (`extensions/host/libhost.m`) shows
  that call only reads the system-wide `AppleInterfaceStyle` default. It reports the
  system Light/Dark setting, not what the menu bar is drawing.
- On current macOS the menu bar is translucent or transparent and adapts to the
  wallpaper beneath it, so it can render light while the system is Dark, or dark while
  the system is Light. Hammerspoon exposes no effective-appearance API for a status item
  (`extensions/menubar` has none), so there is no reliable signal to switch on.
- Whenever the two disagree, the chosen palette lands on the wrong surface. WCAG
  contrast measured against a light-bar reference `#E6E6E6` and a dark-bar reference
  `#2E2E2E`:
  - dark palette on a light bar: **1.28–2.37:1** (the blue `#65C3ED` is 1.59:1, and
    yellow/lime are about 1.3:1);
  - light palette on a dark bar: **1.84–2.21:1** (the blue `#006381` is 2.01:1).
- The other fixed colors have the same flaw. `NO POMODORO` green `#30d158` is 1.62:1 on
  a light bar. The overdue red `#ff453a` is 2.73:1 on a light bar.

The suspicion about blue is half right. Blue was among the faintest stops, and the eye
resolves short-wavelength detail worst in small text, so thin blue digits read poorly
even at a passing ratio. But dropping blue alone would leave the yellow and green stops
invisible on light bars. The real defect is that each palette was legible on only one
surface while being chosen by a signal that does not describe the menu bar.

The planner rendered a simulated preview on `#F2F2F2`, `#E6E6E6`, `#2E2E2E`, and
`#141414` bars. Both old palettes visibly vanish on their opposite surfaces; the palette
below reads on all four.

## Design

### One palette for every menu bar

Every color in a menu bar that can be either light or dark has to sit in the luminance
middle: dark enough for a light bar, light enough for a dark bar. All ten stops are
**isoluminant at WCAG relative luminance ≈ 0.205**, the balance point between the two
references. The ten hues were sampled at equal perceptual (OKLab) steps along OKLCH hues
27°→185°, with chroma capped at 0.20 so the red does not go neon. Then each stop's
lightness was solved so its luminance lands on the target.

Equal luminance is also the aesthetic choice: no stop looks brighter or dimmer than its
neighbors, so the countdown drifts calmly in hue instead of flickering in brightness.
The order keeps the SASE usage indicator's metaphor (cool means plenty left, red means
nearly exhausted) and adds the familiar traffic-light arc. Copy these values exactly:

| Bucket | Remaining time        | Hue       | Hex       | vs `#E6E6E6` | vs `#2E2E2E` |
| ------ | --------------------- | --------- | --------- | ------------ | ------------ |
| 10     | Above 90%, up to 100% | Teal      | `#008D81` | 3.28:1       | 3.32:1       |
| 9      | Above 80%, up to 90%  | Sea green | `#008F5B` | 3.31:1       | 3.29:1       |
| 8      | Above 70%, up to 80%  | Green     | `#009123` | 3.32:1       | 3.28:1       |
| 7      | Above 60%, up to 70%  | Leaf      | `#4E8C00` | 3.32:1       | 3.28:1       |
| 6      | Above 50%, up to 60%  | Olive     | `#768500` | 3.28:1       | 3.31:1       |
| 5      | Above 40%, up to 50%  | Gold      | `#927C00` | 3.30:1       | 3.30:1       |
| 4      | Above 30%, up to 40%  | Amber     | `#AB7100` | 3.30:1       | 3.29:1       |
| 3      | Above 20%, up to 30%  | Orange    | `#C16400` | 3.31:1       | 3.29:1       |
| 2      | Above 10%, up to 20%  | Vermilion | `#D85100` | 3.30:1       | 3.30:1       |
| 1      | Zero through 10%      | Red       | `#E3413B` | 3.31:1       | 3.29:1       |

Every stop measures 4.10:1 or more against pure white and 5.07:1 or more against pure
black, and its blue channel is never its dominant channel. A 50-minute session reads
`50:00` teal, `45:00` sea green, `40:00` green, down to `05:00` red. Bucket selection,
clamping, and nil handling stay exactly as they are.

### Legibility contract

The tests enforce these rules on every fixed hex color the title can paint:

1. WCAG contrast is **at least 3.0:1 against both** `#E6E6E6` (light-bar reference) and
   `#2E2E2E` (dark-bar reference). Both references are deliberately conservative: real
   light bars are lighter and real dark bars darker, which only raises contrast.
2. Gradient stops keep relative luminance within **0.19–0.22**, so a later retune cannot
   make one stop pop or sink. Within this band rule 1 always holds.
3. No gradient stop is blue-dominant (blue channel greater than both red and green).
4. White badge text on the alert red is at least 3.0:1 (measured 4.13:1).

3:1 is the WCAG AA threshold for large or bold text. That is why the digits become bold.

### Bold countdown digits

Render the countdown status in every countdown state in a bold monospaced font,
`Menlo-Bold` at the menu-bar font size. That covers gradient digits, the neutral
unknown-duration digits, and the overdue `+MM:SS`. Bold roughly doubles the colored
stroke area, which is what keeps hue identifiable at 13–14 pt. Menlo's bold and regular
faces share advance widths, so the digits keep a stable width and the title never
shifts. Resolve the font once per config load with `hs.styledtext.validFont`. If
`Menlo-Bold` is unavailable, fall back to the existing regular monospaced font; never
fail the title over a font. The stop time keeps the regular monospaced font as context.
The theme stays bold and the `OVERDUE` badge keeps its bold system font. The title reads
bold theme, quiet context, then the tomato and bold countdown.

### Every other title color joins the same band

| Use                                     | Old        | New                       |
| --------------------------------------- | ---------- | ------------------------- |
| Overdue `+MM:SS` digits                 | `#ff453a`  | `#E3413B` (alert red)     |
| Steady `OVERDUE` badge text             | `#ff453a`  | `#E3413B`                 |
| Flashing `OVERDUE` badge background     | `#ff453a`  | `#E3413B`                 |
| Flashing `OVERDUE` badge text           | `#ffffff`  | `#FFFFFF` (4.13:1 on red) |
| `NO POMODORO` (bold)                    | `#30d158`  | `#009123` (green)         |
| Theme, duration, separator, arrow, stop | labelColor | unchanged (labelColor)    |

Alert red deliberately equals bucket 1. The countdown is already red in its last tenth,
stays red when it crosses zero (the `+` prefix carries that change), and escalates to
the flashing badge at ten minutes overdue. `NO POMODORO` reuses bucket 8 so the whole
item draws from one family. Context spans keep the dynamic system `labelColor`, which
macOS resolves against the menu bar's real appearance at draw time. Do not color the
tomato emoji.

### Rejected alternatives

- **Detecting the menu bar's real appearance.** There is no Hammerspoon API for it.
  Screen sampling needs Screen Recording consent, adds a capture indicator, and lags
  wallpaper or Space changes. In-process Objective-C introspection is undocumented and
  fragile.
- **Apple's dynamic `system*Color` list colors.** They adapt correctly, but several
  light-appearance variants are fill colors, not text colors (systemYellow is about
  1.4:1 on a light bar), and there are not ten in order.
- **A colored background chip behind every running countdown.** It is always legible,
  but heavy for a constantly visible item, and it would blunt the flashing `OVERDUE`
  badge, which must stay the one alarm-shaped element.
- **Halos or shadows around the digits.** They look cheap in a native menu bar.

Known limit, to document: a mid-tone wallpaper directly under a transparent menu bar can
undercut any fixed color, and macOS's own labels too. Turning on Accessibility › Display
› Reduce transparency (or the menu bar background option, where the macOS version offers
one) restores a uniform surface.

## Implementation

1. **Pure palette module (`home/dot_hammerspoon/pomodoro_countdown.lua`).** Replace
   `GRADIENT_DARK_COLORS` and `GRADIENT_LIGHT_COLORS` with one `M.GRADIENT_COLORS`
   (index 1 = red, 10 = teal, the exact values above). Add `M.ALERT_COLOR = "#E3413B"`,
   `M.MISSING_COLOR = "#009123"`, and `M.BADGE_TEXT_COLOR = "#FFFFFF"` as the single
   source of truth for every fixed title color. Change `M.gradient_color(bucket)` to one
   argument; a stray extra argument must not change its result. Keep its nil handling.
   Replace the "copied from SASE" provenance comment with a short one: hue order
   inspired by SASE's usage indicator; values derived for menu-bar legibility
   (isoluminant ≈ 0.205, OKLCH 27°→185°, chroma ≤ 0.20, no blue); the legibility
   contract is enforced in the spec. Leave `presentation()` and the bucket math
   untouched. The module stays independent of `hs`.
2. **Hammerspoon integration (`home/dot_hammerspoon/init.lua`).**
   - Delete `resolveBobPomodoroInterfaceDark`, the interface-style resolution inside
     `bobPomodoroMenuTitle`, and both per-appearance attribute caches. The Pomodoro path
     must not call `hs.host` at all.
   - Cache one ten-entry gradient attribute table per config load from
     `PomodoroCountdown.gradient_color(bucket)`, using the bold monospaced countdown
     font. A normal status with a valid bucket uses its entry; otherwise the neutral
     countdown attributes (`labelColor`, same countdown font) apply.
   - Add a validated bold monospaced resolver next to the existing mono resolver
     (`Menlo-Bold` at menu-bar size, else nil) and use bold-or-regular mono for the
     neutral, gradient, and overdue countdown attributes. Leave stop-time attributes on
     the regular mono font.
   - Build the overdue countdown, steady `OVERDUE`, flashing badge background, badge
     text, and missing attributes from the module constants (alpha 1), instead of local
     hex literals.
   - Keep the styled-text failure fallback that returns the complete plain title.
3. **Docs (`README.md`, Pomodoro menu bar section).** Rewrite the gradient paragraph:
   one palette for every menu bar, and why (the menu bar follows its backdrop, not the
   system appearance); teal → red with no blue; the 50-minute example; bold digits; the
   legibility contract with its two references; alert red and the `NO POMODORO` green in
   the same band; neutral digits when the duration is unknown; and the mid-tone
   wallpaper limit with the Reduce transparency remedy. Remove the light/dark variant,
   appearance-fallback, and "palette copied from SASE / SASE contrast measurements"
   sentences. Keep the state table unchanged: text does not change.

## Tests

Update both spec files and keep their unrelated coverage.

`tests/hammerspoon/pomodoro_countdown_spec.lua`:

- Replace `DARK_GRADIENT`/`LIGHT_GRADIENT` with one expected `GRADIENT` table. Assert
  `countdown.GRADIENT_COLORS` equals it exactly. Assert `gradient_color(b)` returns each
  stop, and that `gradient_color(b, true)` and `gradient_color(b, false)` return the
  same stop. Keep the existing nil cases (0, 11, 1.5, `"3"`, nil). Assert the ten stops
  are distinct.
- Add a **legibility contract** block with local spec-only WCAG helpers (sRGB to linear,
  relative luminance, contrast ratio). Self-check them: white/black is 21:1 and
  identical colors are 1:1. For every gradient stop plus `ALERT_COLOR` and
  `MISSING_COLOR`, assert at least 3.0:1 against both `#E6E6E6` and `#2E2E2E`. Assert
  every gradient stop's luminance is within 0.19–0.22 and that no stop is blue-dominant.
  Assert `BADGE_TEXT_COLOR` on `ALERT_COLOR` is at least 3.0:1, and that `ALERT_COLOR`
  equals bucket 1. Include regression witnesses showing the gate rejects the retired
  colors: `#65C3ED` against `#E6E6E6`, `#006381` against `#2E2E2E`, and `#30d158`
  against `#E6E6E6` all fall below 3.0:1.
- Keep every bucket-boundary, equivalence, and nil-input test as is.

`tests/hammerspoon/init_spec.lua`:

- Replace the dark-palette, light-palette, unknown-appearance, and palette-switch tests.
  The running countdown's span color is `{ hex = GRADIENT[bucket], alpha = 1 }` and its
  font is `Menlo-Bold`. The span text, colors, and fonts are identical for
  `interface_style` `"Dark"`, nil, `"Light"`, and `"Solarized"`, with `hs.host` removed,
  and with `interfaceStyle` throwing. Add a call counter to the `hs.host` mock and
  assert the Pomodoro renderer never calls it.
- Keep the existing sync-to-gradient, boundary, retime, and rename tests, updating
  expected hexes to the new palette (for example `50:00` is `#008D81`, a 45:00 boundary
  is `#008F5B`).
- Neutral fallback (unknown duration): `labelColor` with `Menlo-Bold`.
- Overdue `+MM:SS` uses `ALERT_COLOR` and `Menlo-Bold`. Steady `OVERDUE` uses
  `ALERT_COLOR`; flashing uses `BADGE_TEXT_COLOR` on `ALERT_COLOR`; missing uses
  `MISSING_COLOR`. Update the font-resolution, nine-span flash, and missing-style tests
  that hard-code `#ff453a`, `#ffffff`, and `#30d158`. Flash phases must still leave all
  eight non-status spans unchanged and keep the badge font and padding identical.
- Font fallback: extend the styled-text mock so a test can mark `Menlo-Bold` invalid.
  Then every countdown state falls back to `Menlo-Regular` and still renders. Assert
  `Menlo-Bold` is validated at load, not per tick.
- **Every-color audit (end to end).** Render a running title in each of the ten buckets
  (drive remaining time through the real tick), plus the neutral fallback, recent
  overdue, both warning phases, and missing. Collect every `color` and `backgroundColor`
  from every styled span. Each must be either the dynamic `labelColor` table or a hex in
  the vetted set: the ten stops, `ALERT_COLOR`, `MISSING_COLOR`, and `BADGE_TEXT_COLOR`
  (the last only as foreground on an `ALERT_COLOR` background). Assert all ten stops
  appeared, so a future color added anywhere in the title fails this test until it joins
  the contract.
- Keep the styling-failure fallback, zero crossing, single resync, polling, wake,
  refresh, reload cleanup, and font-validation regressions passing.

## Verification commands

From the opened chezmoi checkout:

```sh
busted --no-coverage ./tests/hammerspoon
stylua --check home/dot_hammerspoon/pomodoro_countdown.lua home/dot_hammerspoon/init.lua tests/hammerspoon/pomodoro_countdown_spec.lua tests/hammerspoon/init_spec.lua
prettier --check --prose-wrap=always --print-width=88 README.md
git diff --check
grep -n "interfaceStyle\|GRADIENT_DARK\|GRADIENT_LIGHT\|ff453a\|30d158" home/dot_hammerspoon/*.lua README.md
```

The final `grep` must print nothing. Also run `just check`, using `/sase_monitor` if it
needs a long-running handoff, and report the exact outcome. Earlier runs found
`lint-lua` blocked because `lua-language-server`/`llscheck` is absent and that recipe
does not cover `home/dot_hammerspoon`. If that is still the only failure, report it
separately from the passing focused checks; do not call the full gate green and do not
broaden into tooling repairs.

## Deployment and visual verification

Follow the opened repository's finalization instructions and its required post-commit
`chezmoi update -a --force`. Read `/sase_final` at execution time: the host commits
after the provider turn, so do not invent a post-commit hook, commit manually, or run
commands after the final declaration. If the apply cannot happen inside the turn, report
it as pending with the command and expected revision.

Read the `tailnet.md` reference memory with `/sase_memory_read` before Mac access. Use
the configured `mac` SSH alias with bounded connection attempts; the Mac is often
offline. Once the landed revision is available there, apply it, confirm the deployed
`~/.hammerspoon` files match, and confirm Hammerspoon's watcher reloaded them.

For the visual pass, use a temporary in-memory preview, never the Pomodoro ledger. For
example, an `hs.canvas` strip at real menu-bar font size drawing all ten stops, alert
red, the badge, and `NO POMODORO` on `#E6E6E6` and `#2E2E2E`, plus the live status item
on the real menu bar. Remove any preview canvas, timer, or fake state afterwards and
resync the real runtime. Do not change Bryan's wallpaper or appearance settings to
manufacture a second backdrop; if a second display or Space already shows the other bar
appearance, check it there too. Confirm each stop is clearly readable, digits keep a
stable width across buckets and the zero crossing, and only the `OVERDUE` badge flashes.

If a stop proves unreadable on the real bar, retune it within the same hue order and the
0.19–0.22 luminance band, then update the tests and README. Never reintroduce appearance
switching. If Mac access or GUI inspection is unavailable, say exactly which checks ran
and which remain unverified instead of claiming a visual pass.

## Completion criteria

- The countdown uses exactly ten time-relative colors from one appearance-independent
  palette.
- Every fixed color in any title state passes the tested legibility contract against
  both light and dark menu-bar references, and no Pomodoro code consults the system
  appearance.
- Countdown digits are bold monospaced with a regular-mono fallback.
- Title text, segment order, bucket math, overdue precedence, and runtime lifecycle are
  unchanged.
- README matches the code.
- Focused checks pass and the full-gate result is reported accurately.
- Deployment and visual verification have concrete evidence or an explicit remaining
  blocker.
