---
tier: tale
title: Limit Mac Pomodoro idle reminders to ten seconds
goal:
  Make each NO POMODORO flash period last ten seconds while preserving Fibonacci rests
  and existing presentation behavior.
size: small
decisions:
  update_mac_pom_glossary:
    ask:
      Update the Mac Pomodoro glossary entry to describe the current reminder schedule
      with ten-second flashes?
    default: false
    memory:
      - glossary:mac-menu-bar-pomodoro-indicator
    answer: false
proposed_by: bbugyi200.athena.0yl
decided_by: auto
create_time: 2026-10-09 02:24:52
status: wip
---

# Limit Mac Pomodoro idle reminders to ten seconds

## Outcome and scope

Each flashing period of the Hammerspoon `NO POMODORO` menu-bar indicator lasts 10
seconds instead of 60. Keep the existing Fibonacci rests of 1, 1, 2, 3, 5, 8, ...
minutes, measured from the end of each flashing period. The first reminder still begins
one minute after the idle label appears. Flash cadence stays at 1 Hz, and the green idle
label remains visible between reminders.

This is a `small` tale: one coding agent can change one policy constant, update the
existing behavioral tests and README, and perform focused verification. There are no
phase dependencies or architecture decisions.

## Repository access and findings

Work begins in the bob-cli project. Use `/sase_repo` and run
`sase repo open chezmoi -r "Implement ten-second Mac Pomodoro idle reminders"`. Use the
returned checkout and read its `AGENTS.md`. Paths below are relative to that linked
checkout unless explicitly marked bob-cli.

- `home/dot_hammerspoon/pomodoro_countdown.lua` defines
  `M.MISSING_REMINDER_FOR_SECONDS = 60` and the separate
  `M.MISSING_REMINDER_WAIT_UNIT_SECONDS = 60`. `missing_reminder_step` adds each
  Fibonacci rest to the previous reminder's end; its active interval includes the start
  and excludes the end. `next_missing_reminder` uses the same schedule.
- `home/dot_hammerspoon/init.lua` renders from that module on a 0.5-second tick. It
  passes elapsed idle time through `missingShownSeconds`, uses the same module for the
  dropdown's next-reminder preview, preserves the idle anchor across successful empty
  refreshes, and resets it when idle reappears or on wake/unlock.
- `tests/hammerspoon/pomodoro_countdown_spec.lua` covers schedule arithmetic,
  boundaries, upcoming reminders, invalid input, and presentation.
  `tests/hammerspoon/init_spec.lua` covers rendered frames, suffix transitions, menu
  previews, refreshes, and reset behavior using a fake clock.
- The README's "Pomodoro menu bar" section specifies the current Fibonacci schedule. The
  bob-cli glossary entry is older: it still describes immediate flashing and a
  ten-minute repeating cycle. Follow the code and README for behavior, and handle the
  glossary only through the decision below.
- The macOS restart hook already hashes `pomodoro_countdown.lua`, so a normal full
  chezmoi update applies this file and restarts a running Hammerspoon.

## Implementation

1. Change only `M.MISSING_REMINDER_FOR_SECONDS` from `60` to `10` in the presentation
   policy. Retain the 60-second wait unit and existing scheduling algorithm. The
   rendering timer, color/font/padding policy, `φ Nm` suffix, refresh/reset behavior,
   running countdown, overdue pulses, and `OVERDUE` escalation keep their existing
   behavior. No bob-cli command or output contract changes are needed.

2. Update the existing policy and runtime test cases to the new schedule. Shorter
   flashing periods bring subsequent reminders earlier because the existing rests begin
   at the end of each period. Use these concrete elapsed times as independent expected
   values:

   | Rest just completed | Flash start          | Flash end, excluded  |
   | ------------------- | -------------------- | -------------------- |
   | 1 minute            | 60 seconds (1:00)    | 70 seconds (1:10)    |
   | 1 minute            | 130 seconds (2:10)   | 140 seconds (2:20)   |
   | 2 minutes           | 260 seconds (4:20)   | 270 seconds (4:30)   |
   | 3 minutes           | 450 seconds (7:30)   | 460 seconds (7:40)   |
   | 5 minutes           | 760 seconds (12:40)  | 770 seconds (12:50)  |
   | 8 minutes           | 1250 seconds (20:50) | 1260 seconds (21:00) |

   Recalculate the longer Fibonacci examples too, retaining coverage beyond the first
   few reminders and keeping the uncapped waits. For each tested period, verify inactive
   just before its start, active at the start and at start + 9 and + 9.5 seconds, and
   inactive at exactly start + 10 seconds. Keep expected times independent of the
   production duration constant so the tests detect a return to 60 seconds.

   Update existing active/resting presentation samples and their descriptions, including
   the runtime cases currently using 119/180/239 seconds. Verify both flash phases
   retain the same padded text and `φ Nm` suffix throughout a period, and that the
   suffix and animation stop at its new end. Use the existing fake-clock boundary cases
   (59, 60, 69, 70 seconds for the first period), with a later period covered as well.

   Verify the next-reminder preview before, during, and after a flash. For example,
   elapsed 0 points to start 60; elapsed 60, 69, and 70 points to start 130 with `φ 1m`;
   elapsed 140 or 250 points to start 260 with `φ 2m`. Update the dropdown's existing
   frozen-clock examples so at least one remains inside an active period; 90 seconds is
   now a rest. Preserve the dropdown's existing `HH:MM` formatting.

3. Update the README to say "10-second flash step" and give start examples
   `1:00, 2:10, 4:20, 7:30, 12:40, 20:50, ...`. Keep the description of Fibonacci rests,
   the suffix, and reset events consistent with the code.

4. Apply the glossary decision only if accepted.

   > [!decision] update_mac_pom_glossary Use `/sase_memory_write` before editing and
   > `/sase_memory_read` to reread `glossary:mac pom`. In bob-cli's existing
   > `sase/memory/glossary/mac-menu-bar-pomodoro-indicator.md`, replace the obsolete
   > idle schedule with an initial one-minute rest, ten-second flashing periods, and
   > uncapped Fibonacci rests between them. Retain the appearance/wake/unlock reset
   > semantics and README pointer. Run `sase memory init` from bob-cli to regenerate
   > memory outputs. Keep this edit confined to the reminder schedule.

   If this decision remains off through `%auto`, follow `/sase_memory_write`: record the
   skipped correction through `/sase_new_task`, corroborating an existing memory task
   where applicable. If a human explicitly declines it, file nothing.

## Validation and acceptance

Run from the linked chezmoi checkout:

```sh
busted tests/hammerspoon/pomodoro_countdown_spec.lua tests/hammerspoon/init_spec.lua
stylua --check home/dot_hammerspoon/pomodoro_countdown.lua tests/hammerspoon/pomodoro_countdown_spec.lua tests/hammerspoon/init_spec.lua
prettier --check --prose-wrap=always --print-width=88 README.md
git diff --check
```

The existing Busted configuration uses `nlua` and coverage; use that harness. Review the
complete diff to confirm that only the idle flash duration and its timing
expectations/documentation changed. Retain the existing tests for idle anchor
continuity, reappearance, wake/unlock, missing styled-text support, invalid inputs, and
running/overdue states in the two full spec files.

Acceptance requires a ten-second active window at each reminder, unchanged Fibonacci
rest lengths and 1 Hz cadence, the correct next-reminder preview, and passing focused
tests/format checks. Report any unavailable tools or unperformed live checks accurately.

Follow the linked repository's post-commit instruction to run
`chezmoi update -a --force` through the normal SASE completion workflow. On macOS, a
full apply runs the existing restart hook; a targeted Hammerspoon apply requires a
separate Reload Config. When validating the deployed Mac UI, observe steady idle text
for the first minute, flashing from approximately 1:00 until 1:10, and the next flashing
period from 2:10 until 2:20. Check that wake/unlock restarts the initial one-minute
rest. Distinguish Linux test results from a live Mac smoke check; do not claim the Mac
was updated or observed without evidence.
