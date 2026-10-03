---
tier: tale
title: Restore the original idle Pomodoro green and remove the idle tomato
goal: Display NO POMODORO in the original green without a tomato when no session exists.
size: small
proposed_by: bbugyi200.apollo.4d
status: done
---

# Restore the original idle Pomodoro green and remove the idle tomato

## Outcome

When Hammerspoon's successful `bob pomodoro --show-stale` query returns no current
session, the menu-bar item must display exactly `NO POMODORO`, in the original green
`#30d158` at full opacity. It must have no tomato, leading space, trailing space, or
icon segment. Running and overdue sessions retain their existing tomato and layout.

This is a focused presentation change in the linked **chezmoi** repository, suitable for
one coding agent and one implementation pass (`tale`, `small`).

## Repository access and confirmed history

From the implementation workspace, use `/sase_repo` and run
`sase repo open chezmoi -r "Restore the original idle Hammerspoon Pomodoro presentation"`.
Use only the checkout path it prints, and read that checkout's `AGENTS.md` before
editing. All file paths below are relative to that checkout.

On October 2, 2026 at 20:31:51 UTC, commit `f40c745929f9a8ef5f03a00c0a99b0f8afc40d85`
(`feat(hammerspoon): make Pomodoro menu-bar gradient legible on any menu bar`) changed
the idle foreground from the literal `#30d158` in `home/dot_hammerspoon/init.lua` to
`PomodoroCountdown.MISSING_COLOR`, introduced as `#009123` in
`home/dot_hammerspoon/pomodoro_countdown.lua`. The parent version of `init.lua` confirms
the precise original shade. Commit `319674bdc877e6d7963e8055b857ae7342acd38f`
subsequently removed the countdown gradient but retained this darker idle color.

The current missing-session branch of `presentation()` returns a title prefixed by
`M.ICON .. GAP`, `icon = M.ICON`, and three segments: icon, gap, and missing text. The
renderer in `init.lua` already accepts a missing presentation consisting only of the
missing-text segment and falls back to `presentation.title` if styling fails. The
successful empty-output callback already clears prior session context and creates the
missing state; command and parse errors separately hide the item.

Baseline validation in the inspected checkout: `busted ./tests/hammerspoon` passed with
57 successes and zero failures or errors.

## Implementation

1. In `home/dot_hammerspoon/pomodoro_countdown.lua`, set `M.MISSING_COLOR` to `#30d158`.
   Change only the `remaining_seconds == nil` presentation to return
   `title = M.NO_POMODORO_TITLE`, retain its `appearance`, `status`, and absent
   duration, omit its `icon` field, and supply exactly one segment:
   `{ text = M.NO_POMODORO_TITLE, role = "missing" }`. Keep `M.ICON` and the
   current-session branches intact. The existing missing-title attributes in `init.lua`
   already consume the constant, so no renderer rewrite is needed. Preserve its
   bold-font fallback and full-opacity green styling.

2. Update existing assertions in `tests/hammerspoon/pomodoro_countdown_spec.lua` and
   `tests/hammerspoon/init_spec.lua` for the exact new idle text and original green.
   Cover both flash phases, absent and stale context, the single missing segment,
   segment concatenation matching the title, and absent icon metadata. Update the
   invariant test that currently expects the first missing segment to be a tomato.
   Strengthen the existing styled missing-state check to verify one text span, green
   `#30d158` with alpha 1, and no background badge. Keep the successful empty-output
   transition/recovery and styling-failure fallback checks asserting exactly
   `NO POMODORO`. Existing active and overdue assertions must continue to verify their
   tomato and countdown layout.

   The current `meets the menu-bar legibility contract` test enforces 3:1 contrast
   against both reference backgrounds for `MISSING_COLOR` and explicitly shows that
   `#30d158` is below 3:1 on the light reference. Adjust that test to keep the existing
   alert-red and white-on-red contrast checks, while verifying the idle color directly
   as the user-selected `#30d158`. Remove the obsolete missing-green contrast
   requirement and replace its old-green rejection assertion with the exact color check.
   Keep the numeric contrast helper sanity checks and current alert thresholds intact.
   Rename the test to describe the alert contrast contract accurately.

3. Update the Pomodoro menu-bar section of `README.md`: replace the idle example and
   empty-result description with `NO POMODORO`, remove the claim that the tomato is a
   constant idle anchor, and document idle green as `#30d158`. Explain that the tomato
   accompanies current-session countdowns and overdue status. Keep documentation of
   durations, stop times, fonts, polling, tooltips, dropdowns, errors, and overdue
   flashing consistent with their existing behavior.

## Validation and acceptance

- Run `busted ./tests/hammerspoon` from the chezmoi checkout after the changes. All
  existing and updated Hammerspoon tests must pass. Exercise the real empty-result
  callback and plain-title styling fallback through the existing mocks.
- Run
  `stylua --check home/dot_hammerspoon/pomodoro_countdown.lua tests/hammerspoon/pomodoro_countdown_spec.lua tests/hammerspoon/init_spec.lua`
  and `git diff --check`. Include any additional changed Lua file in the format check.
  Inspect the diff and search the scoped source, tests, and README for stale `#009123`
  and `🍅 NO POMODORO` expectations.
- Acceptance: an idle title is exactly `NO POMODORO`; styled text uses
  `{ hex = "#30d158", alpha = 1 }`; idle title, segments, and metadata contain no tomato
  or residual gap; active and overdue titles still contain their tomato; alert colors,
  fonts, flashing, and successful-empty versus error behavior pass their existing
  regression checks.
- Follow the chezmoi checkout's `AGENTS.md` requirement to run
  `chezmoi update -a --force` after any implementation commit, including a SASE
  finalizer commit. For visual acceptance, apply the updated source on the Mac running
  Hammerspoon and let its existing config watcher reload. Check idle text and the
  restored green, then a running session and the return to idle. If Mac access requires
  SSH, first read the `tailnet.md` reference with `/sase_memory_read` and use its
  documented connection procedure. Report automated checks and any unavailable Mac smoke
  check distinctly; applying on another host alone does not establish that the Mac menu
  bar has updated.

## Scope

Limit source changes to the missing-session presentation and its color constant, with
corresponding test and README updates. Preserve the current system-foreground running
countdown, fixed overdue alert colors, refresh lifecycle, and session parsing. The
historical color change was bundled with other styling work; use a targeted edit rather
than reverting that entire commit.
