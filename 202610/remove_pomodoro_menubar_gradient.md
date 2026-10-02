---
tier: tale
title: Remove the Pomodoro menu-bar color gradient
goal: The running countdown stays in the system foreground, turns red when overdue,
  and flashes red on OVERDUE.
size: medium
proposed_by: bbugyi200.apollo.4b
status: done
---

# Remove the Pomodoro menu-bar color gradient

## Outcome and scope

The Hammerspoon Pomodoro status item stops changing the countdown color as scheduled
time runs down. While a session is running, including at `00:00` and when the duration
is unknown, the countdown digits use the ordinary system menu-bar foreground. From the
first overdue second through `+09:59`, those digits turn red and stay steady. At ten
minutes overdue and later, the status reads `OVERDUE` and only that badge flashes red.

This is a **medium tale** for one coding agent. All source changes belong in the linked
**chezmoi** repository. Start with the `/sase_repo` skill:

```sh
sase repo open chezmoi -r "Remove the Pomodoro menu-bar color gradient"
```

Use the printed checkout path and read its `AGENTS.md` before editing. Paths below are
relative to that checkout. Do not write a workspace directory into source, tests, or
docs. bob-cli, SASE, the ledger, capture contracts, Neovim keymaps, and the tmux segment
need no changes.

Leave the title contract alone. The visible order stays `theme (duration) · 🍅 status`,
with `→ stop time` inserted once overdue, and `🍅 NO POMODORO` when nothing is current.
Keep the tomato as a text segment immediately before the status, the ordinary-foreground
duration, the tooltip and dropdown, the 15-second poll, wake and unlock refresh, the
manual Refresh item, and the single resync when the countdown crosses zero.

## Diagnosis

At planning time chezmoi `HEAD` is `f40c745929f9a8ef5f03a00c0a99b0f8afc40d85`
(`feat(hammerspoon): make Pomodoro menu-bar gradient legible on any menu bar`). From
that checkout, `busted --no-coverage ./tests/hammerspoon` passes **65 successes / 0
failures**.

Two commits added the gradient and then retuned it:

- `c7b88df8848f0fdedf3ca653d0f1bcc4a9609046` put the tomato beside the countdown and
  colored the running digits from a ten-stop, time-relative scale.
- `f40c745929f9a8ef5f03a00c0a99b0f8afc40d85` replaced the light/dark pair with one
  appearance-independent palette and moved the overdue red, the flashing badge, and
  `NO POMODORO` onto colors that stay readable on either menu bar.

The gradient itself lives in three places:

- `home/dot_hammerspoon/pomodoro_countdown.lua` defines `GRADIENT_COLORS`,
  `gradient_bucket`, and `gradient_color`, and `presentation()` sets `bucket` only when
  `appearance == "normal"` and `durationMinutes` is a finite positive number. Bucket 10
  is a fresh session; bucket 1 is the last tenth, including `00:00`, and uses the same
  hex as the overdue alert (`#E3413B`).
- `home/dot_hammerspoon/init.lua` caches `bobPomodoroGradientTitleAttributes` for
  buckets 1 through 10. In `bobPomodoroMenuTitle`, a normal status span with a valid
  bucket uses that cache. Any other normal status uses `labelColor`. Overdue `+MM:SS`
  already uses `ALERT_COLOR` on the bold mono countdown font. The `OVERDUE` badge
  already flashes between red text and white-on-red. `NO POMODORO` uses `MISSING_COLOR`.
- `tests/hammerspoon/pomodoro_countdown_spec.lua` and the describe block
  `Hammerspoon init Pomodoro countdown gradient` in `tests/hammerspoon/init_spec.lua`
  lock the bucket math, the ten hexes, and the end-to-end color audit. `README.md`
  documents the scale in the Pomodoro menu bar section.

`durationMinutes` is also how the title gets `(50m)`. Sync still computes it from the
ledger range, and `presentation()` still formats it when the context has no duration
string. That path is separate from the bucket.

Before the gradient, running digits were system `labelColor` in regular mono, overdue
digits were `#ff453a`, and `NO POMODORO` was `#30d158`. The later legibility commit
replaced those two hexes because they fail a 3:1 check on one of the menu-bar
references. Those replacements stay. The thing to remove is the running scale, not the
alert colors.

## Color contract

Apply these rules in order. Color only the status span, except the missing state, which
colors the `NO POMODORO` span. The tomato, theme, duration, gaps, arrow, separator, and
stop time stay on system `labelColor` and never take a background.

| Condition                                                         | Status text                                        | Status color                                                  | Flash      |
| ----------------------------------------------------------------- | -------------------------------------------------- | ------------------------------------------------------------- | ---------- |
| No session (`remaining == nil`)                                   | `NO POMODORO`                                      | `#009123` (`MISSING_COLOR`), bold system font                 | No         |
| Running, `remaining >= 0`, including `00:00` and unknown duration | `MM:SS`                                            | system `labelColor`, bold mono countdown font                 | No         |
| Overdue and above `-600` seconds                                  | `+MM:SS`                                           | `#E3413B` (`ALERT_COLOR`), same countdown font, no background | No         |
| `remaining <= -600` seconds, steady phase                         | `OVERDUE` with the existing no-break-space padding | `#E3413B` text, bold system font, no background               | This phase |
| `remaining <= -600` seconds, flash phase                          | same padded `OVERDUE`                              | `#FFFFFF` text on `#E3413B`                                   | This phase |

The last tenth of a running session is no longer red. Red begins at the first overdue
second (`+00:01`). `00:00` stays in the system foreground. Unknown duration uses that
same foreground. It is not a separate color.

Keep one countdown font for running and overdue digits: `Menlo-Bold` at the menu-bar
size, falling back to the existing regular mono font when bold is unavailable. Menlo's
bold and regular faces share advance widths, so the title does not shift at the zero
crossing, and the overdue red keeps the bold ink it was chosen for (3.31:1 on both
`#E6E6E6` and `#2E2E2E`, which is the large-or-bold threshold). The stop time stays
regular mono. The `OVERDUE` badge stays on the bold system font. Validate fonts once per
config load, as today.

Keep these module constants as the only fixed title colors:

- `ALERT_COLOR = "#E3413B"`
- `MISSING_COLOR = "#009123"`
- `BADGE_TEXT_COLOR = "#FFFFFF"`

Do not restore `#ff453a` or `#30d158`. Do not consult `hs.host.interfaceStyle`. The
Pomodoro path must not call `hs.host` at all.

The flash half-period stays the existing 0.5-second tick. Only the status span changes
between the two warning phases. Context spans, the badge font, and the badge padding
stay constant across phases.

## Implementation

1. **Pure module (`home/dot_hammerspoon/pomodoro_countdown.lua`).** Delete
   `GRADIENT_COLORS`, the palette comment, `gradient_bucket`, and `gradient_color`.
   Delete the `bucket` local in `presentation()` and the `bucket` field on the returned
   table. Keep `ALERT_COLOR`, `MISSING_COLOR`, and `BADGE_TEXT_COLOR`. Keep
   `durationMinutes` on the presentation when the context has a finite number, and keep
   using it only to format a missing duration string. Keep every `appearance` value:
   `missing`, `normal`, `overdue`, `overdue_warning`, and `overdue_warning_flash`. The
   module stays independent of `hs`.
2. **Menu-bar painter (`home/dot_hammerspoon/init.lua`).** Delete
   `bobPomodoroGradientTitleAttributes` and the loop that fills it. In the status-span
   branch, `appearance == "normal"` always uses `bobPomodoroCountdownTitleAttributes`
   (`labelColor` plus the bold-or-regular mono countdown font). Leave the overdue,
   warning, warning-flash, and missing branches on the constants above. Keep passing
   `state.duration` and `state.durationMinutes` into the presentation context, and keep
   computing both at sync time. Keep the styled-text failure path that returns the
   complete plain title.
3. **Docs (`README.md`, Pomodoro menu bar section).** Leave the state table and the
   duration, theme-length, tooltip, dropdown, polling, and failure sentences as they
   are. Replace the paragraph that begins "The running countdown digits carry one
   ten-color time-relative gradient" with this contract, in prose that prettier can wrap
   at 88 columns:

   The running countdown digits stay in the ordinary menu-bar foreground for the whole
   session, including `00:00` and a session whose duration is unknown. They use a bold
   monospaced face, falling back to the regular monospaced face when bold is
   unavailable. From the first overdue second through `+09:59`, the `+MM:SS` countdown
   is alert red (`#E3413B`) and does not flash. At and after ten minutes overdue, the
   status is `OVERDUE` and only that badge flashes between red text and white
   (`#FFFFFF`) on the same red. `NO POMODORO` is green (`#009123`). Theme, duration,
   separator, arrow, stop time, and the tomato stay in the ordinary foreground. A
   mid-tone wallpaper directly under a transparent menu bar can undercut any fixed
   color, including the system's own labels; turning on Accessibility › Display › Reduce
   transparency restores a uniform surface.

   The later lifecycle sentence that already says only the `OVERDUE` badge flashes may
   stay. Do not leave a second description of tenths, hues, isoluminance, or a ten-color
   scale.

## Tests

Update the two Hammerspoon spec files. Leave their other describe blocks passing without
weakening them. In particular, keep the existing coverage for title text, the one-space
tomato gap, `OVERDUE` at exactly `-600`, the nine-span flash regression, font validation
(`Menlo-Bold` checked at load, missing green `#009123`, both badge colors), styling
failure, zero-crossing resync, polling, wake, refresh, and reload cleanup.

`tests/hammerspoon/pomodoro_countdown_spec.lua`:

- Delete the bucket-boundary, five-minute walk, two-and-a-half-minute walk, clamp,
  cross-length equivalence, nil-input, and ten-stop palette tests.
- Replace "meets the menu-bar legibility contract" so it no longer mentions a gradient.
  Keep the local WCAG helpers and their white/black and identical-color self-checks.
  Assert `ALERT_COLOR` and `MISSING_COLOR` are each at least 3.0:1 against both
  `#E6E6E6` and `#2E2E2E`. Assert `BADGE_TEXT_COLOR` on `ALERT_COLOR` is at least 3.0:1.
  Keep the three retired-color witnesses: `#65C3ED` against `#E6E6E6`, `#006381` against
  `#2E2E2E`, and `#30d158` against `#E6E6E6` all stay below 3.0:1.
- Replace "attaches the gradient bucket..." with a presentation test that every state
  (running with numeric duration, zero, recent overdue, warning, string-only duration,
  unknown duration, missing) has a nil `bucket`. A running context with
  `durationMinutes = 50` still reports `durationMinutes == 50` and a `(50m)` title.
  String-only and missing contexts still have nil `durationMinutes`.

`tests/hammerspoon/init_spec.lua`, rename the gradient describe block so its name no
longer says gradient:

- Running `50:00` is `labelColor` with `Menlo-Bold`, duration `(50m)` stays `labelColor`
  on the system font, no span has a background, and `hs.host` is not called.
- The same running title, colors, and fonts result when `interface_style` is `"Dark"`,
  nil, `"Light"`, or `"Solarized"`, when `hs.host` is removed, and when `interfaceStyle`
  throws. `env.host_calls` stays 0.
- Unknown duration (`DEEP WORK → 10:15 · 🍅 01:00`) uses the same `labelColor` and
  `Menlo-Bold` countdown font.
- Recent overdue `+MM:SS` uses `ALERT_COLOR` and `Menlo-Bold` with no background. Steady
  `OVERDUE` uses `ALERT_COLOR` text and no background. The flash phase uses
  `BADGE_TEXT_COLOR` on `ALERT_COLOR`. Missing text uses `MISSING_COLOR`.
- With `Menlo-Bold` marked invalid, running, unknown-duration, and recent-overdue
  countdowns use `Menlo-Regular`. Running and unknown-duration stay `labelColor`. Recent
  overdue stays `ALERT_COLOR`. The bold font is validated at load, not per tick.
- Sync still stores `durationMinutes` and paints `(25m)` without a further `bob` request
  when the tick moves from `25:00` to `22:30`. Both countdowns are `labelColor`. Retime
  still updates the theme, duration, and remaining text; rename still updates only the
  theme. In both cases the countdown color stays `labelColor`.
- Rewrite the every-color audit. Render running remainders `3000`, `1500`, `300`, and
  `0` on a 50-minute session, plus unknown duration, recent overdue, both warning
  phases, and missing. Every foreground is either the `labelColor` table or one of
  `ALERT_COLOR`, `MISSING_COLOR`, and `BADGE_TEXT_COLOR`. White appears only as
  foreground on an `ALERT_COLOR` background. The only background is `ALERT_COLOR`, and
  only on the flashing badge. Do not require ten distinct stops. `hs.host` is not
  called.
- Styling failure in running, overdue, and missing states still returns the complete
  plain title, including the tomato position.

## Verification

From the opened chezmoi checkout:

```sh
busted --no-coverage ./tests/hammerspoon
stylua --check home/dot_hammerspoon/pomodoro_countdown.lua home/dot_hammerspoon/init.lua tests/hammerspoon/pomodoro_countdown_spec.lua tests/hammerspoon/init_spec.lua
prettier --check --prose-wrap=always --print-width=88 README.md
git diff --check
grep -nE "gradient|GRADIENT_|interfaceStyle|#D85100|#C16400|#AB7100|#927C00|#768500|#4E8C00|#008F5B|#008D81|#ff453a|#30d158" \
  home/dot_hammerspoon/pomodoro_countdown.lua \
  home/dot_hammerspoon/init.lua \
  tests/hammerspoon/pomodoro_countdown_spec.lua \
  tests/hammerspoon/init_spec.lua \
  README.md
```

The `grep` must print nothing. Also run `just check`, using `/sase_monitor` if it needs
a long-running handoff, and report the exact outcome. `lint-lua` does not cover
`home/dot_hammerspoon` or `tests/hammerspoon`; `llscheck` and `luacheck` run only on
Neovim config, Neovim tests, and `home/lib`. If a missing `lua-language-server` or
`llscheck` is the only failure, report it separately from the passing focused checks. Do
not call the full gate green, and do not expand this task into tooling repairs.

## Deployment and visual verification

Follow the opened repository's finalization instructions and its required post-commit
`chezmoi update -a --force`. Read `/sase_final` at execution time. The host commits
after the provider turn, so do not invent a post-commit hook, commit manually, or run
commands after the final declaration. If the apply cannot happen inside the turn, report
it as pending with the command and the expected revision. Source completion and live
deployment are separate when that sequencing blocks both.

Read `tailnet.md` with `/sase_memory_read` before Mac access. The Mac SSH alias is `mac`
(Tailscale name `kellys-macbook-pro`, user `bbugyi`). It is offline unless the lid is
open. Use a bounded `ConnectTimeout` and a small number of attempts.

Once the landed revision is on the Mac, apply it, confirm the deployed `~/.hammerspoon`
files match, and confirm Hammerspoon's watcher reloaded them. For the visual pass, use a
temporary in-memory preview or the live runtime with timers paused and restored in a
guaranteed cleanup path. Do not edit the Pomodoro ledger, and do not change wallpaper or
appearance settings. Confirm:

- Running digits match the surrounding foreground at the start of a session, halfway
  through, in the last minute, and at `00:00`.
- `+MM:SS` is red and does not flash. Only the `OVERDUE` badge flashes, between red text
  and white-on-red.
- `NO POMODORO` stays green. The tomato is not recolored.
- Digit width stays stable across the zero crossing.

Remove any preview canvas, timer, or fake state, then reload and resync the real
runtime. If Mac access or GUI inspection is unavailable, say which checks ran and which
remain unverified.

## Completion criteria

- No countdown color depends on remaining fraction, duration, or system appearance.
- Running digits, including `00:00` and unknown duration, use `labelColor` and the bold
  mono countdown font, with the regular mono fallback.
- Overdue `+MM:SS` is steady `#E3413B`. `OVERDUE` at `remaining <= -600` flashes between
  `#E3413B` text and `#FFFFFF` on `#E3413B`, and nothing else flashes.
- `NO POMODORO` stays `#009123`. Title text, segment order, and the sync lifecycle are
  unchanged.
- The palette, bucket helpers, bucket field, gradient attribute cache, and the retired
  hexes are gone from the module, the painter, both specs, and the README.
- Focused checks pass, and the full-gate result is reported accurately.
- Deployment and visual verification have concrete evidence or an explicit remaining
  blocker.
