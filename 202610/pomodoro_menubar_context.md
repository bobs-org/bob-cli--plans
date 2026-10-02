---
tier: tale
title: Keep the Pomodoro theme and stop time visible in the Mac menu bar
goal:
  Show theme and stop time beside the live countdown or OVERDUE warning while preserving
  NO POMODORO for missing sessions.
size: medium
proposed_by: bbugyi200.athena.0vg
create_time: 2026-10-02 13:06:30
status: wip
---

# Keep the Pomodoro theme and stop time visible in the Mac menu bar

## Outcome and scope

Show the current Pomodoro's theme and scheduled stop time throughout its lifetime,
including when it is at least ten minutes overdue. Retain the live countdown, shorten
the escalated warning to `OVERDUE`, and preserve `NO POMODORO` when no current session
exists. The result should feel like a compact native status item with a clear visual
hierarchy.

This is a medium tale: one implementation agent can deliver the Lua presentation,
integration, regression coverage, documentation, and deployment together. All source
changes belong in the linked **chezmoi** repository. Open it with
`sase repo open chezmoi -r "Implement the approved Pomodoro menu-bar design"`, use the
returned path, and read its `AGENTS.md`. Paths below are relative to that repository
unless explicitly identified as bob-cli context. This planning turn changes only this
scratch plan.

## Findings that determine the implementation

- `home/dot_hammerspoon/init.lua` already runs `bob pomodoro --show-stale`
  asynchronously. `parseBobPomodoroOutput` retains `taskText`, the normalized range, and
  validated `endHour`/`endMinute`; `todayEndEpoch` drives a local countdown.
- `home/dot_hammerspoon/pomodoro_countdown.lua` currently returns only the countdown or
  warning text. At `remaining_seconds <= -600`, it replaces the countdown with
  `OVERDUE POMODORO`. The missing state is separate from command/parse failures.
- The runtime has a 0.5-second render/flash timer, a 15-second sync timer, wake and
  unlock refresh, a one-shot refresh when crossing zero, in-flight task suppression,
  callback guards, and retained-object cleanup on reload. Preserve these behaviors.
- bob-cli's `src/native/pomodoro.rs` already supplies everything needed, in the form
  `[<13m] 0950-1015 — DEEP WORK` or `[OVERDUE by 10m] 0950-1015 — DEEP WORK`. Canonical
  names follow one leading em dash; older entries can contain ordinary task text
  instead. bob-cli's `capture_pomodoros::parse_name_tail` preserves later em dashes
  within a name. No CLI extension, vault parsing, or second subprocess is needed for
  this feature.
- The two current Hammerspoon test files pass: **11 successes, no failures** with
  `busted --no-coverage ./tests/hammerspoon`. `init_spec.lua` currently substitutes a
  simplified presentation implementation, so its existing tests alone cannot verify the
  new end-to-end text.

## Visual design

Use one horizontal title in this fixed order: **theme → stop time · status**. These
examples show visible text; styling distinguishes the segments:

| State                                           | Example title                 |
| ----------------------------------------------- | ----------------------------- |
| Running                                         | `DEEP WORK → 10:15 · 12:34`   |
| At the stop time                                | `DEEP WORK → 10:15 · 00:00`   |
| Recently overdue                                | `DEEP WORK → 10:15 · +00:01`  |
| Last second before escalation                   | `DEEP WORK → 10:15 · +09:59`  |
| Exactly ten minutes overdue and thereafter      | `DEEP WORK → 10:15 · OVERDUE` |
| Existing session without a name or legacy label | `UNTITLED → 10:15 · 12:34`    |
| No current session                              | `NO POMODORO`                 |

Use the system menu-bar font at its native size. Give the theme restrained emphasis with
the existing validated bold-font resolver; use regular weight for the arrow and stop
time. Use native, appearance-aware foreground colors for context, not hard-coded black
or white. Keep the time clearly readable rather than making it tiny or faint. The stop
time is always zero-padded 24-hour `HH:MM`, matching the ledger's convention; the arrow
means the session's scheduled endpoint. Do not replace it with the current clock or
recompute it by adding the rounded CLI minute count.

Keep the live countdown, with at least two minute digits and two second digits (`09:59`,
`00:00`, `+00:01`; allow more minute digits when necessary). Use a validated monospaced
font such as Menlo at the menu-bar font size for numeric runs, falling back to the
system menu-bar font. Resolve fonts once per load, not every tick.

Before the ten-minute cutoff, only the overdue counter turns the existing red. At and
after the cutoff, only the `OVERDUE` badge alternates between the existing bold red
foreground and white-on-red warning styles. Preserve the current 0.5-second half-period
and identical badge padding, text, and font in both phases. Theme, stop time, separator,
and their colors stay constant. The whole title must never blink off. Keep the current
green bold `NO POMODORO` styling and show it by itself.

Bound the theme segment to **24 Unicode code points including a final ellipsis** when
needed: keep the first 23 code points, trim trailing whitespace, then append `…`. Short
names remain complete. Never byte-slice a UTF-8 sequence, truncate the stop time or
status, or rotate between different pieces of information. Keep the full normalized
theme in the tooltip and dropdown. This deliberately bounds typical menu-bar use without
introducing screen-layout monitoring; visually verify wide-character names and the
MacBook's available space during the smoke check.

The tooltip should lead with the full theme and `Stops at HH:MM`. The dropdown should
expose the same full context, retain the raw CLI status as secondary detail, and keep
Last sync and Refresh. The missing state keeps its existing no-current-session details
and has no leftover theme or stop time. Do not add capture, rename, or timer controls.

## Implementation

1. **Separate semantic text from Hammerspoon styling.** Extend `pomodoro_countdown.lua`
   with pure helpers for context normalization and the title model. Pass context
   explicitly into presentation rather than re-parsing the raw output every tick. Return
   the complete plain title plus the segment information needed to style theme,
   endpoint, separator, and status independently. Keep remaining seconds and the
   existing appearance states as the sole warning-state inputs.

2. **Normalize the existing payload conservatively.** After a successful parse, trim and
   collapse whitespace in the task label, remove exactly one leading em dash and
   surrounding whitespace for canonical names, and preserve internal punctuation, later
   em dashes, combined names such as `BOB + SASE`, and authored case. For a legacy
   nonempty label without the leading em dash, show that label. If nothing remains, use
   `UNTITLED`; a valid range without a theme is still a current session. This is display
   cleanup, not a new Markdown or capture-grammar parser. Preserve `rawOutput` and the
   existing parser's range validation and empty-output handling. Derive the fixed
   stop-time string directly from parsed `endHour` and `endMinute`. Cache the full theme
   and shortened theme when a successful sync replaces the state.

3. **Compose one attributed title in `init.lua`.** Replace the whole-title warning
   styling in `bobPomodoroMenuTitle` with segment-specific attributes using
   `hs.styledtext`, then pass it to the existing `setTitle`. Use its documented table
   representation or supported composition APIs; ensure attribute spans remain correct
   around multibyte text. Explicitly set the base menu-bar font and an appropriate
   system foreground for context, since styled-text defaults need not match a menu bar.
   Apply warning backgrounds exclusively to the padded warning segment. Keep the
   existing font validation and graceful fallback. If richer styling fails, fall back to
   the complete plain title so presentation failure cannot discard valid context. Update
   tooltip/menu details together with the new state.

4. **Preserve synchronization and lifecycle guarantees.** Poll the same command at the
   same cadence; local ticks must not spawn extra tasks or perform font discovery.
   Renaming a theme or adjusting the endpoint must replace all displayed context on the
   next successful sync. Empty successful output clears old context and displays
   `NO POMODORO`. Command failure, malformed output, and task-start failure retain the
   existing hide/log behavior, without relabeling failures as no session or displaying
   stale context. Retain reload cleanup, obsolete-callback rejection, manual Refresh,
   and wake/unlock handling. Do not change date selection, cross-midnight semantics,
   task status, or the vault.

5. **Document the result.** Add a short Pomodoro menu-bar section to chezmoi's
   `README.md` with the state examples, arrow meaning, 24-hour clock, truncation/full
   details behavior, warning threshold, and refresh behavior. Keep screenshots,
   generated assets, new configuration knobs, and bob-cli/mac-capture changes outside
   this bounded feature.

The native title API accepts attributed text, and the attributed-text API supports font
and per-range color/background attributes; use the documented contracts:
[menu-bar titles](https://www.hammerspoon.org/docs/hs.menubar.html#setTitle),
[styled text](https://www.hammerspoon.org/docs/hs.styledtext.html), and
[system color lists](https://www.hammerspoon.org/docs/hs.drawing.color.html).

## Verification and acceptance

Extend `tests/hammerspoon/pomodoro_countdown_spec.lua` with table-driven semantic cases:
positive countdowns, zero, -1, -599, -600, and well past the cutoff; both flash phases;
canonical and legacy labels; empty or em-dash-only labels; whitespace normalization;
combined and internally dashed names; 24-character and over-limit names; multibyte
characters around the truncation boundary; and endpoint formatting at midnight, noon,
and 23:59. Assert that theme and stop time survive every current-session state and that
`OVERDUE POMODORO` is no longer the warning status.

Extend `tests/hammerspoon/init_spec.lua` with real presentation-module integration
coverage through the mocked task-completion callback. Upgrade the styled-text double to
record composed text and attribute spans. Freeze the clock for boundary assertions and
restore it afterwards. Verify:

- Named active and stale-overdue CLI payloads reach the menu with their full context;
  the invoked command still contains `--show-stale`.
- Flash phases have identical text, padding, font metrics, and context attributes; only
  the warning segment's foreground/background changes.
- Font rejection or styling failure leaves a complete readable title.
- A later renamed/retimed result updates title, tooltip, and menu; an empty success
  becomes exactly `NO POMODORO` with old context removed; malformed/nonzero results
  clear state as before and a later valid result recovers.
- Existing one-task-at-a-time, zero-crossing refresh, wake/unlock, manual refresh, stale
  callback, and reload guarantees still hold. Retain the startup regressions.

Run `busted --no-coverage ./tests/hammerspoon`, a targeted `stylua --check` on changed
Lua files, `prettier --check --prose-wrap=always --print-width=88 README.md`, and
`git diff --check`. Apply formatting only to changed files. There is no Rust behavior
change requiring a bob-cli build. Inspect the final diff for accidental hotkey or
screenshot changes.

After the approved implementation is committed through the prescribed SASE finalizer,
follow chezmoi's mandatory `chezmoi update -a --force` apply workflow. Deploy the
committed source on the Mac through the established SSH alias `mac`; consult the
`tailnet.md` memory before remote access and use a bounded connection timeout. The
Hammerspoon path watcher should reload applied configuration automatically. Do not edit
deployed `~/.hammerspoon` files directly.

Perform a Mac smoke check in light and dark appearances: short and long themes, a
wide-character label, running/zero/recent-overdue/escalated/missing states, hover and
dropdown details, both warning phases, and reload/wake recovery. Use a temporary
in-memory presentation preview, restoring live state afterward, rather than editing the
real Pomodoro ledger to simulate states. Confirm the time remains readable on the
MacBook display, no warning background spills into context, and flashing does not move
adjacent items. Automated Linux tests do not establish native visual correctness. If the
Mac is offline or GUI inspection is unavailable, report that exact remaining
verification step and provide the deployment/smoke-check instructions; do not claim it
was visually verified or deployed.
