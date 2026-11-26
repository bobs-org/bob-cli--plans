---
tier: tale
title: Sync the US Holidays Google Calendar into Bob
goal:
  bob gcal sync writes each Holidays in United States event as a marked all-day row in
  Events.
size: small
decisions:
  mark:
    ask: Which emoji should mark rows from the Holidays in United States calendar?
    choices:
      us_flag:
        🇺🇸 names the source calendar; some Linux terminal fonts show it as the letters U
        S
      party: 🎉 renders everywhere but reads as any celebration, close to 🎂 Birthdays
    default: us_flag
    why:
      🇺🇸 passed a live dry run, and only a flag sets holidays apart from personal
      celebrations.
    answer: us_flag
proposed_by: bbugyi200.athena.1gc
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-26 05:28:03
status: wip
---

# Sync the "Holidays in United States" Google Calendar into Bob

## Goal

Bryan's screencast shows the subscribed Google calendar **Holidays in United States**.
Its calendar id is `en.usa#holiday@group.v.calendar.google.com` (Settings → Integrate
calendar). `bob gcal sync` should copy its events into each daily note's `## Events`
section the same way the Birthdays calendar already works: one all-day context row per
event, tagged with a calendar mark.

## Findings: no bob-cli change is needed

All of these were checked live against Bryan's account through a temporary copy of
`~/.config/bob/config.yml` (`BOB_CONFIG_FILE=…`, read-only commands and `--dry-run`
only):

- `bob gcal calendars` already lists the calendar. Its role is `reader` and it shows
  `SYNC no`. The existing "id must be in the account's calendar list" check
  (`calendar_not_found`) therefore passes for this subscribed calendar.
- `gcal.calendars` accepts `{id, mark, root?}` maps (`src/native/config/gcal.rs`,
  `parse_calendar_item`). Every non-primary calendar needs a unique mark
  (`src/native/gcal/config.rs`, `validate_calendars` / `validate_mark`). Neither 🇺🇸 nor
  🎉 is in `TASKS_SIGNIFIER_CHARS` or `BOB_MARK_RESERVED_CHARS`. Multi-scalar marks such
  as the flag's two regional indicators are allowed: up to 8 characters, see the
  `zwj_mark_is_accepted` test.
- With the entry added, `bob gcal list -D 2026-11-26` showed
  `all day  🇺🇸 Thanksgiving Day … context`. The JSON output showed `inclusion: context`,
  `reason: all-day`, `ack.state: not_required`, and `description_lines: 0`.
- `bob gcal sync --dry-run -D 2026-11-26` would add only:
  `- **all day** 🇺🇸 Thanksgiving Day [📅](https://www.google.com/calendar/event?eid=…)`.
  That has the same shape as the Birthdays row in
  `tests/gcal_sync.rs::marked_birthday_row_is_born_marked_and_idempotent`. It has no
  checkbox, `#task`, `[root::]`, MEET link, or description bullets, even though Google
  gives each event a description (`Public holiday`, or
  `Observance\nTo hide observances…`).
- A YAML plain scalar treats `#` as a comment only after whitespace, so the unquoted id
  parsed correctly. The plan still quotes it so readers don't mistake the `#` for a
  comment.
- A per-calendar `root` affects only timed events. Every holiday is an all-day event, so
  the entry leaves `root` unset, as Birthdays does.

## Change (chezmoi linked repo only)

`~/.config/bob/config.yml` is managed by chezmoi. Do not edit the live file, and do not
edit anything in bob-cli.

1. Open the linked repo with `sase repo open chezmoi -r "<reason>"`. Use the path it
   prints, and read that repo's `AGENTS.md`.
2. In `home/dot_config/bob/config.yml`, under `gcal.calendars`, add one entry right
   after the Birthdays entry and its commented `# root: love` line. Match the Birthdays
   entry's style:

   ```yaml
   - id: "en.usa#holiday@group.v.calendar.google.com" # Holidays in United States (subscribed, read-only)
     mark: 🇺🇸 # all-day context rows; root is unused for all-day events
   ```

   > [!decision] mark = party Use `mark: 🎉` instead of `mark: 🇺🇸`, with the same
   > comment.

   Leave every other line of the file unchanged.

## Verification (before the commit lands)

The file is plain YAML, not a chezmoi template, so point bob at the edited source file
directly. Run all of these with
`BOB_CONFIG_FILE=<opened chezmoi path>/home/dot_config/bob/config.yml`:

- `bob gcal doctor` → the `calendars` check reads
  `✓ 3 synced · <account>, 🎂 Birthdays, <mark> Holidays in United States`.
- `bob gcal calendars` → the Holidays row shows `SYNC yes` and the chosen mark.
- `bob gcal list -D 2026-11-26` → a `<mark> Thanksgiving Day` all-day `context` row.
  `-D 2026-12-25` → `<mark> Christmas Day`.
- `bob gcal sync --dry-run -D 2026-11-26` → the diff adds only the
  `- **all day** <mark> Thanksgiving Day [📅](…)` row, plus the `## Events` header and
  `↻ synced` stamp if today's note has none yet. It must show no description bullets.

Do **not** run a real `bob gcal sync`. The hourly `bob tick` gcal step on the Mac writes
the rows once the Mac has the new config. Per
`decisions:bob-gcal-is-read-only-and-proposes-once`, the Mac is the single writer.

## Rollout

- The chezmoi checkout is a commit obligation: land it through the turn's normal SASE
  final declaration, as a `chore(bob): …` commit in the dotfiles repo.
- The chezmoi repo's gotcha requires `chezmoi update -a --force` after the commit lands.
  If the turn cannot run it after the finalizer's commit, say so in the final response
  so Bryan runs it on athena and on the Mac. The Mac is the host whose `bob tick` writes
  the rows.

## Out of scope

- **Observances.** Google's feed includes observances as well as public holidays:
  Halloween, Election Day, Black Friday, Christmas Eve, Daylight Saving Time ends, and
  so on. Bob syncs whatever Google returns. To hide them, Bryan can use Google Calendar
  → Settings → Holidays in United States → "Holiday calendar content". Filtering in Bob
  would be a new feature, not part of this change.
- No bob-cli code, docs, tests, or memory changes. Nothing in bob-cli becomes stale.

## Governing decisions

- `decisions:calendar-events-are-daily-note-event-tasks`: synced events live in the
  daily note's `## Events` section. Holiday rows land there as marked all-day context
  rows.
- `decisions:bob-gcal-is-read-only-and-proposes-once`: Google stays read-only and the
  Mac is the single writer. The plan verifies with read-only commands and `--dry-run`
  only.
