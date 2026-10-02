---
tier: tale
title: Put the tomato beside the countdown and add a ten-color time gradient
goal: Show the tomato immediately before the Pomodoro countdown, keep duration text
  neutral, and color running time with ten reliable SASE-inspired steps.
size: medium
proposed_by: bbugyi200.apollo.48.f1
status: done
---

# Put the tomato beside the countdown and add a ten-color time gradient

## Outcome and scope

The menu bar should read `DEEP WORK (50m) · 🍅 12:34`. The tomato identifies the
countdown immediately to its right. The duration is ordinary, fully opaque text in the
same system foreground as the surrounding context. Only the running countdown digits
receive a ten-color progression, moving from blue toward red as the scheduled time is
consumed. The existing overdue alerts retain priority.

This is a **medium tale**: one coding agent can implement the pure presentation logic,
Hammerspoon integration, regression coverage, documentation, and deployment
verification. All source changes belong in the linked **chezmoi** repository. Start with
the `/sase_repo` skill and run:

```sh
sase repo open chezmoi -r "Implement the approved Pomodoro countdown gradient plan"
```

Use its printed checkout path and read its `AGENTS.md`. Paths below are relative to that
checkout. The SASE repository is a read-only design reference; this task does not need
changes to bob-cli, SASE, the ledger, capture contracts, or task statuses. Retain the
existing range parser, endpoint calculation, midnight-wrapped duration calculation, sync
cadence, missing/error handling, and tooltip/dropdown content.

## Evidence and design choices

At planning time, chezmoi commit `1d0c440ac85d9c617439f4f59a5c9364bfe96f76` contains the
previous tomato/duration change. The relevant files are:

- `home/dot_hammerspoon/pomodoro_countdown.lua`: pure title/segment assembly and
  duration helpers. The icon currently precedes the theme.
- `home/dot_hammerspoon/init.lua`: duration is cached at sync, but only its formatted
  string reaches the renderer; its style uses `secondaryLabelColor`. The running
  countdown currently uses neutral text. Alerts use fixed red `#ff453a`.
- `tests/hammerspoon/pomodoro_countdown_spec.lua` and `tests/hammerspoon/init_spec.lua`:
  title, segment, styling, timer, and failure tests.
- `README.md`, the Pomodoro menu bar section.

The baseline `busted --no-coverage ./tests/hammerspoon` passes: **44 successes, zero
failures/errors**. `busted`, `stylua`, and `prettier` are present. The planner did not
run the full repository gate; `lua-language-server` is currently absent from PATH,
consistent with the earlier implementation's reported `just check` limitation.

SASE commit `34079430e3bf2fd34bd3451631cbdbcd370514d2` supplies the reference in
`src/sase/ace/tui/widgets/_usage_indicator_palette.py`, with tests in
`tests/test_provider_usage_indicator_presentation_style.py`. Its ten hues encode
remaining capacity, with bright tones for dark surfaces and deeper tones for light
surfaces. Copy these small palettes locally, with a provenance comment naming that file
and revision; Hammerspoon must not import SASE or read its checkout at runtime. If
further source inspection is needed, open it using `sase repo open sase` first.

Use **fraction of scheduled time remaining**, not fixed minute thresholds: five minutes
left is the last tenth of a 50-minute session but half of a 10-minute one. Use discrete
colors for the whole numeric token, with no interpolation, per-digit rainbow, pulsing,
progress bar, or new text. SASE buckets its displayed integer percentage; this timer
displays seconds, so intentionally use exact equal tenths of the scheduled duration
instead of copying that integer-rounding detail.

## Presentation contract

| State                         | Plain title                            |
| ----------------------------- | -------------------------------------- |
| Running                       | `DEEP WORK (50m) · 🍅 12:34`           |
| Exactly at the endpoint       | `DEEP WORK (50m) · 🍅 00:00`           |
| Recently overdue              | `DEEP WORK (50m) → 10:15 · 🍅 +00:01`  |
| Last second before escalation | `DEEP WORK (50m) → 10:15 · 🍅 +09:59`  |
| Ten minutes overdue and later | `DEEP WORK (50m) → 10:15 · 🍅 OVERDUE` |
| Unnamed session               | `UNTITLED (50m) · 🍅 12:34`            |
| Unknown duration              | `DEEP WORK → 10:15 · 🍅 12:34`         |
| No current session            | `🍅 NO POMODORO`                       |

For every current-session state, assemble roles in this order:
`theme, [gap, duration], [arrow, stop], separator, icon, gap, status`. Exactly one
ordinary space separates the tomato from the numeric countdown. Nothing else sits
between them. The overdue warning retains its existing no-break-space badge padding;
that padding remains confined to the status span. Missing stays `icon, gap, missing`.

Keep duration parentheses, the middle-dot separator, the overdue stop-time rule, and the
unknown-duration stop-time fallback. Keep the native emoji as a text segment so its
position is controllable and the plain-text fallback includes it. Do not use `setIcon`,
which would put it at the status item's outer edge.

Set the duration's attributes to the existing context attributes: system `labelColor`,
alpha 1, ordinary menu-bar font, no background. Remove the separate dimmed-duration
style. Preserve the theme's bold font, the countdown's monospaced font, and the existing
font fallbacks. The tomato, duration, theme, separator, arrow, and stop time never
inherit countdown colors or flashing backgrounds. Do not recolor the emoji.

`presentation.title` must remain the exact concatenation of its segment text in all
states. Styling failure must return this complete title, including the relocated tomato.
Keep the full theme and scheduled stop time in the tooltip and dropdown, without adding
a tomato or gradient there.

## Gradient and state precedence

Let `D` be valid scheduled duration in seconds (`durationMinutes * 60`) and `R` be
remaining seconds. For finite `R >= 0` and finite positive `D`, clamp `R` to `[0, D]`
and choose a one-based bucket:

```text
bucket = max(1, min(10, ceil(10 * clamped_remaining / D)))
```

Thus exactly 90% is bucket 9, exactly 50% is bucket 5, exactly 10% and zero are bucket

1. Time above the scheduled duration clamps to bucket 10. Compute from the numeric
   duration and seconds, never from the displayed `50m`, formatted countdown, elapsed
   tick count, or `bob pomodoro`'s rounded prefix. A retimed session uses its new
   duration and endpoint at the next successful sync; a wake uses the existing wall
   clock.

| Bucket | Remaining time        | Hue          | Dark appearance | Light appearance |
| ------ | --------------------- | ------------ | --------------- | ---------------- |
| 10     | Above 90%, up to 100% | Blue         | `#65C3ED`       | `#006381`        |
| 9      | Above 80%, up to 90%  | Cyan         | `#48CCD0`       | `#006C6C`        |
| 8      | Above 70%, up to 80%  | Teal         | `#4CD4B0`       | `#006E56`        |
| 7      | Above 60%, up to 70%  | Green        | `#78DB8D`       | `#206F3C`        |
| 6      | Above 50%, up to 60%  | Yellow-green | `#AADC64`       | `#456C1B`        |
| 5      | Above 40%, up to 50%  | Yellow       | `#CED44C`       | `#5F6500`        |
| 4      | Above 30%, up to 40%  | Amber        | `#EBC04F`       | `#775800`        |
| 3      | Above 20%, up to 30%  | Orange       | `#FFA552`       | `#8C480E`        |
| 2      | Above 10%, up to 20%  | Coral        | `#FF805F`       | `#A03620`        |
| 1      | Zero through 10%      | Red          | `#FF5F6D`       | `#A22534`        |

A 50-minute session changes colors every five minutes: `50:00` blue, `45:00` cyan,
`40:00` teal, down to `05:00` red. A 25-minute session changes every 2:30. Color is
supplemental: the digits, `+` prefix, and `OVERDUE` label always carry the state.

Apply these rules in priority order:

1. Missing: existing `🍅 NO POMODORO`, including its current missing-state styling.
2. At least ten minutes overdue (`R <= -600`): existing fixed-red/white-on-red flashing
   `OVERDUE` badge. Only its status span flashes; preserve the cadence, font, padding,
   and constant width across phases.
3. Recently overdue (`-600 < R < 0`): existing fixed-red `+MM:SS`, without flashing.
4. Running or exactly zero: apply the selected palette color to the status span only.
5. Running with unavailable duration or appearance information: neutral system
   foreground for the countdown, retaining all title text and normal font styling.

These preserve the established alerts instead of letting a clamped gradient obscure
overdue state. The ten-color requirement describes the running scale; alert red is an
explicit state override.

## Implementation

1. **Pure presentation and palette.** In `pomodoro_countdown.lua`, reorder segments as
   specified. Add a small pure bucket helper accepting remaining seconds and numeric
   duration minutes, plus local palette lookup exposed through a narrow helper. Return
   `nil` for an unavailable bucket: missing/nonnumeric/nonfinite duration, nonpositive
   duration, nonfinite computed seconds, or nonfinite/nonnumeric remaining input. Handle
   negative remaining through the alert path, not normal gradient selection. Keep this
   module independent of `hs`. Add optional bucket metadata to the presentation; keep
   existing `appearance` values and fields intact. Existing string-only duration
   contexts still render, but receive no invented percentage. If guarding numeric
   duration requires sharing a finite-number check with `format_duration`, make that
   small change so an invalid number cannot crash the presentation fallback; do not
   expand into a parser rewrite.
2. **Carry numeric duration into render.** In `init.lua`, pass cached
   `state.durationMinutes` alongside `state.duration` in the context. Keep duration
   derivation at sync time; rendering only computes the ratio and bucket. Replace the
   duration style with the ordinary context attributes. Resolve the normal status color
   from the new bucket while retaining the existing overdue/missing branches and
   plain-text fallback.
3. **Appearance and resilient styling.** Use the documented native
   [`hs.host.interfaceStyle()`](https://www.hammerspoon.org/docs/hs.host.html#interfaceStyle)
   in a guarded helper: `"Dark"` selects the dark palette; a successful `nil` result
   means the default/light style (also accept `"Light"`). A missing API, exception, or
   unrecognized value returns unknown and uses neutral countdown text. Resolve this once
   per render inside the existing half-second tick so appearance switches take effect
   without a reload or a new watcher, timer, shell command, or network request. This API
   reports interface style, not sampled menu-bar backdrop colors. Cache the two sets of
   ten color/font attribute tables once per config load; do not mutate shared context
   attributes when selecting one. Appearance detection failure should preserve styled
   context rather than fail the whole title. Any actual styled-text construction failure
   still returns the complete plain title.
4. **Document.** Update the README state table and prose for the new order, normal
   duration foreground, time-relative scale, exact thresholds, light/dark variants,
   unknown-duration/appearance fallback, and preserved alert behavior. Include the
   palette provenance and the 50-minute example. Keep docs consistent with code.

Use the existing `hs.styledtext` composition: Hammerspoon explicitly supports it in
[`hs.menubar:setTitle`](https://www.hammerspoon.org/docs/hs.menubar.html#setTitle). No
image asset or external dependency is required.

## Verification and acceptance

Update the existing two spec files, preserving their unrelated coverage. Add tests where
a regression could change behavior:

- Exact title and role order for running, zero, recent overdue, both warning phases,
  missing, untitled, unknown duration, and long/multibyte themes. Assert the suffix
  `separator, icon, gap, status` and the title/segment-concatenation invariant.
- The bucket helper reaches exactly ten buckets. Check every boundary at one second
  above, exactly on, and one second below for a 50-minute session, plus midpoints; check
  100%, above 100%, zero, subsecond positive values, and a 25-minute session. Equivalent
  remaining fractions for 5-, 25-, 50-, and 120-minute sessions must select the same
  bucket. Check nil, strings, zero/negative duration, NaN, and both infinities without
  throwing. Keep the midnight-wrapped duration unit test.
- Verify all ten hex colors for both appearances. Mock `hs.host.interfaceStyle` in the
  integration harness and test `"Dark"`, nil/default, `"Light"`, unknown return, absent
  API, and thrown error. Change the mocked style between render ticks and verify the
  palette changes while text, layout, and fonts stay the same.
- Exercise a successful parsed payload through sync to confirm `durationMinutes` reaches
  gradient selection. Drive a color boundary using the existing render timer without an
  extra `bob` request. Retiming must update both the title and denominator at the next
  sync, while rename-only changes keep the same color.
- Inspect styled spans: only normal `status` gets a gradient color; `(50m)` and every
  non-status span keep the appropriate context style, with no background. Check both
  appearance palettes and neutral fallbacks. Update the current nine-span overdue
  regression to the new order and duration foreground. Both flash phases must leave all
  eight non-status spans unchanged, including the adjacent tomato.
- Keep `00:00` normal, then `+00:01` overdue, then escalation at exactly `-600`.
  Preserve the single zero-crossing resync, polling, wake, manual refresh, reload
  cleanup, missing/recovery, command/parse failures, and font fallback regressions.
- Inject styling failure in running, overdue, and missing states and assert complete
  plain titles with the correct tomato position. Appearance lookup failure alone should
  leave a styled title with neutral running digits.

Run from the opened chezmoi checkout:

```sh
busted --no-coverage ./tests/hammerspoon
stylua --check home/dot_hammerspoon/pomodoro_countdown.lua home/dot_hammerspoon/init.lua tests/hammerspoon/pomodoro_countdown_spec.lua tests/hammerspoon/init_spec.lua
prettier --check --prose-wrap=always --print-width=88 README.md
git diff --check
```

Run the repository's `just check` gate as well, using `/sase_monitor` if it needs a
long-running handoff. Record exact outcomes. If the Lua language server or other
existing tooling prevents completion, report the actual blocker and the passing focused
checks separately; do not call the full gate green or broaden this task into unrelated
tooling repairs.

## Deployment and visual verification

Follow the opened repository's current finalization instructions and required
post-commit `chezmoi update -a --force`. Read `/sase_final` at execution time: the
current skill makes the host commit after the provider turn and exposes no arbitrary
post-commit-command field. Do not invent one, manually commit, or run more commands
after a final declaration. Use an already configured post-commit apply workflow if
available. Otherwise explicitly report the required apply and Mac verification as
pending after the host commit, with the command and expected revision; do not pull an
older source tree and claim the new revision is deployed. Source completion and live
deployment must be reported separately when that sequencing prevents both in the
implementation turn.

Read `tailnet.md` with `/sase_memory_read` before Mac access. Use the configured `mac`
SSH alias with bounded connection attempts; the Mac can be offline. After the landed
revision is available, apply the chezmoi source on the Mac, verify the deployed files
match the new revision, and confirm Hammerspoon's existing watcher reloads them.

Use a temporary in-memory preview, or the existing runtime with timers paused and
restored in a guaranteed cleanup path, to inspect all ten colors and each title state.
Do not edit the Pomodoro ledger or leave debug switches, preview timers, or fake state
behind. Reload and resync the normal runtime after the preview. Check:

- Tomato baseline and one-space adjacency to digits; uncluttered theme/duration
  grouping; duration matches ordinary text; monospaced digits keep stable width.
- All ten stops in both palette variants at real menu-bar size, including the
  yellow/amber middle stops, using the actual Mac appearance and backdrop. Preview
  variants without silently changing Bryan's system-wide appearance settings.
- Running to zero to recent overdue to both warning phases; only the badge flashes.
  Verify a long theme, missing state, tooltip, dropdown, and refreshed real session.

SASE's contrast measurements apply to its opaque TUI badge surfaces, not macOS's
translucent menu bar. Do not claim those ratios as measured Mac results. The palette
variants and visual check are required; if a tone is unreadable in the actual menu bar,
adjust its lightness within the same hue/order and update its tests/docs before calling
visual verification complete. Do not add a background block to every running countdown
merely to copy the TUI. If Mac access or GUI inspection is unavailable, state which
checks were performed and which remain unverified instead of claiming a visual pass.

## Completion criteria

The source and README agree on `theme (duration) · 🍅 countdown`; duration is neutral;
the running countdown uses exactly ten predictable time-relative colors in each
appearance; missing/unknown inputs degrade readably; existing overdue behavior and
runtime lifecycle remain correct. Focused checks pass, the full-gate result is reported
accurately, and deployment/visual verification has either concrete evidence or an
explicit remaining blocker.
