---
tier: tale
size: medium
title: "Land bob-cli-f6: fix the epic's Bob Mac Capture breakages, then close the epic"
goal:
  bob-mac-capture compiles and passes the Calendar band tests the epic added, the idle
  agenda no longer respawns bob in a loop after a failed boundary refresh, and epic
  bob-cli-f6 is closed with its plan marked done.
proposed_by: bbugyi200.apollo.bob-cli-f6.land
proposed_with: claude/opus
bead: bob-cli-f6
create_time: 2026-11-01 21:02:56
status: wip
---

- **PARENT:**
  [202611/untracked_calendar_events.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/untracked_calendar_events.md)
- **BEAD:**
  [bob-cli-f6](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-f6/README.md)

# Land bob-cli-f6: fix the epic's Bob Mac Capture breakages, then close the epic

## Context

Epic **bob-cli-f6** ("Untracked calendar events, calendar-aware MEET starts, and a
Calendar band") has all eight phases closed. Its land agent verified the work:

- **bob-cli is complete.** It checked commits 6d39305a, a6671b24, 83e11967, eda4dfc3,
  273e09a7, 39804896 and 96155ed5 against the plan and DECISIONS (untracked_counts =
  exclude, obsidian_meet_start = full, warn_untracked = all). On master d712168f, fmt,
  clippy and every test binary pass. An end-to-end run on a scratch vault produced the
  results the plan expects:
  - `=` at 11:58 wrote an untracked `(**1200-1300**) — MEET`.
  - `x` at 13:00 closed the lunch and carried Standup and the 1:1.
  - `=` then started `(**1300-1400** [t:: 60m]) — MEET`.
  - `completed_summary` and `bob pomodoro --worked` left the lunch out.
  - BOB at 12:45 warned with `=3`.
- **bob-plugins is complete** (7dbbb6d). Every spec item is in source, the vectors match
  bob-cli byte for byte, and no npm test failure comes from the epic.
- **bob-mac-capture is not.** It has epic-caused defects that keep master CI red and one
  runtime bug. This tale fixes them and then closes the epic. All follow-up triage is
  already recorded on bob-cli-f6, so do not file beads for the items below.

All code changes are in the linked repo **bob-mac-capture**. Open it with `/sase_repo`
(`sase repo open bob-mac-capture -r "<why>"`), work only at the path it prints, and read
its `AGENTS.md` first. In particular: never land Swift you have not compiled; format
only the lines you change; run `just portable-check` on a host with Swift (athena if
`swift` is missing locally); for `Sources/BobMacCapture*` and `Tests/BobMacCapture*`,
type-check a scratch copy on the mac (rsync without `.build`/`.git`, `bundle.sh`,
`just typecheck-tests`, then delete the copy; never use Bryan's checkout).

## Work

### 1. Fix the `CaptureAgendaDesignTests` compile error (blocks the whole macOS Test step)

`Tests/BobMacCaptureTests/CaptureAgendaDesignTests.swift` (the `render(...)` helper,
around line 203) calls `.environment(\.colorSchemeContrast, contrast)`. That key path is
read-only. CI run 41369991233 fails with "cannot convert value of type
'KeyPath<EnvironmentValues, ColorSchemeContrast>' to expected argument type
'WritableKeyPath<…>'".

- Use the writable `\._colorSchemeContrast` key path. Confirm it with
  `just typecheck-tests` on the mac scratch copy.
- If that does not type-check, drive the variant some other way that compiles. For
  example, render the high-contrast pass with the `accessibilityHighContrast*`
  appearances; if you do, update `RenderFixtureWriter`'s `appearance == .darkAqua` dark
  check to match.
- If neither compiles, drop the `agenda-events-high-contrast` render and its `contrast:`
  parameter instead of leaving the target broken.

### 2. Fix the wrong Command-J router assertion

`testKeyRouterJoinsCalendarEventFromBlankDraft` in
`Tests/BobMacCaptureTests/BobMacCaptureTests.swift` asserts that Control-J (key code 38,
`.control`) returns `nil`. `CaptureKeyCommandRouter` correctly returns
`.insertBulletNewline` for Control-J, so the test fails on CI runs 41345683097
and 41348314986.

- Change that line to assert `.insertBulletNewline`.
- Keep the Command-J assertions: blank draft → `.joinCalendarEvent`, typed draft →
  `nil`, plain J → `nil`.

### 3. Stop the boundary-refresh respawn loop

`CapturePanelModel.scheduleCalendarBoundaryRefresh()`
(`Sources/BobMacCapture/CapturePanelModel.swift`, around lines 1083–1110) calls
`agendaStore?.refresh(reason: .eventBoundary)` immediately whenever the band's
`refresh_at` is already in the past. If that refresh fails, this loop runs:

1. `CaptureAgendaStore.finishFailure` sets `status = .stale(message)`. `@Published`
   emits even when the value is unchanged.
2. The `$snapshot.combineLatest($status)` sink calls `refreshAgendaPlan()`.
3. `refreshAgendaPlan()` calls `scheduleCalendarBoundaryRefresh()` again.
4. The snapshot is unchanged, so `refresh_at` is still in the past, and the app spawns
   bob again.

The result is back-to-back bob processes for as long as bob keeps failing.

Fix: refresh at most once per boundary.

- Remember the `refresh_at` value already refreshed for, whether it fired immediately or
  from the sleeping task. Skip the immediate refresh when the past `refresh_at` equals
  it.
- A new `refresh_at` re-arms normally.
- Reset the memory when the panel hides, or the day or snapshot changes, so the next
  show still refreshes.
- Prefer a small pure helper in `CaptureCore` that decides "none / immediate /
  sleep(delay)" from `(refreshAt, lastFiredBoundary, now)`. Unit-test it in
  `Tests/CaptureCoreTests` so `just portable-check` covers it: past boundary fires once,
  repeated past boundary does not, future boundary sleeps, new boundary re-arms.

Do not add a timer or backoff beyond this.

### 4. Restore the byte-identical `agenda-events.json` fixture

The plan says the Mac fixture is a byte copy of bob-cli's golden
`tests/fixtures/capture_pomodoros/agenda-events.json`. c4d8cf0 hand-added
`"tracked": false,` to the running BOB entry (around line 187 of
`Tests/Fixtures/agenda-events.json`). bob never emits that state, because non-MEET
sessions are always tracked.

- Remove that line so `cmp` against bob-cli's golden passes. Read the golden from the
  bob-cli checkout; it has been unchanged since 83e11967.
- The untracked Now-card pill keeps its coverage in the inline
  `testNowCardShowsFutureStartAndUntrackedSession` (`CaptureAgendaCalendarTests`).
- Check that no other test relied on the edited line.

### 5. Far-run banner polish

- **Literal backticks.** The `pomodoro_event_run_later` banner
  (`CapturePanelView.swift`, `eventRunLaterWarning(in:)` and its call site in the start
  card, around lines 4383 and 4660) shows bob's message verbatim. Bob's message is
  "MEET's next event, Lunch, starts at 1400 (2h 10m from now) · capture `=5#meet` to
  start 25 minutes now", so the backticks appear on screen. Render inline code with
  `AttributedString(markdown:options: .init(interpretedSyntax: .inlineOnlyPreservingWhitespace))`,
  falling back to the plain string on parse failure. Keep `accessibilityText` without
  backticks.
- **One banner per batch.** The warning is batch-level, so a multi-start batch shows it
  on every start card. Attach it only to start cards whose `eventRunText` is non-nil,
  meaning bob reported an event run for that start.

### 6. Keep the all-day line in the "Calendar clear" state

In `Sources/CaptureCore/CaptureAgendaPresentation.swift`, the clear path in
`makeCalendar` (`if upcoming.isEmpty { … }`, around line 1540) builds
`CaptureAgendaCalendar` without `allDayRow`. Its `fullRows(showEnded:)` clear branch
returns only `[headerRow, clearRow]`. A day whose timed events have all ended, or that
has only all-day events, therefore drops its all-day line ("School holiday · day 2 of
3").

- Pass `allDayRow: Self.makeCalendarAllDayRow(events.allDay)` in the clear path.
- In the `fullRows` clear branch, return header, then the all-day row when present, then
  the clear row.
- Leave `compactRows` and the oneRow fold unchanged.
- Add a `CaptureAgendaCalendarTests` case for an all-day-only events block and for "all
  timed events ended plus one all-day".

### 7. Verify and document

- `just portable-check`, on athena if needed. It must pass, including the new
  CaptureCore tests.
- `just typecheck-tests` on a mac scratch copy for the `BobMacCapture*` changes (items
  1, 2, 3 and 5). If the mac is offline, say so in the bead note and final report: the
  macOS targets are unverified.
- `just docs-check`. Update `docs/idle-agenda.md` only if it describes the clear state
  or boundary refresh in a way these changes make false.
- Do not wait for macOS CI. The 7 baseline XCTest failures are tracked as bob-cli-f3,
  and the docs image refresh is tracked as bob-cli-fg.

## Closeout of epic bob-cli-f6 (final step, same turn as the code)

1. Run `sase bead epic-symbols bob-cli-f6`. At planning time it printed "No
   --epic-symbol entries for bob-cli-f6". If entries appear, resolve each symbol (wire
   it up, privatize it, add a non-test pragma, or delete it per the Symvision
   epic-whitelist policy). Re-key a Justfile line to an open bead only when a still-open
   later bead needs the exemption.
2. Close the epic. Never use `--force` to make the close succeed.

   ```bash
   sase bead close bob-cli-f6 --note "Landed. Verified bob-cli (6d39305a a6671b24 83e11967 eda4dfc3 273e09a7 39804896 96155ed5) against the plan and DECISIONS: just check fmt+clippy+tests green on master d712168f; scratch-vault E2E (untracked lunch run, carried Standup, tracked 1300-1400 MEET, worked/completed_summary exclude lunch, BOB =3 warning). bob-plugins 7dbbb6d complete, vectors byte-identical. bob-mac-capture landing fixes: <list items 1-6 with commit SHA and verification: portable-check result, mac typecheck-tests result or 'mac offline, unverified'>. Follow-up triage recorded on the epic (bob-cli-ff, bob-cli-fg new; +1 on f1/et/eh/f3; DISCOVERED ISSUE on bob-cli-fb; declined items with reasons)."
   ```

3. Run `just symvision` in bob-cli if the recipe exists. At planning time bob-cli's
   justfile had no `symvision` recipe; if it is still absent, say so.
4. Set `status: done` in the frontmatter of the epic's plan file: the PLAN path printed
   by `sase bead read bob-cli-f6 -r "<why>"`. It lives in the `plans` sidecar; use
   `/sase_repo` (`sase repo open plans`) before editing it.
5. bob-cli-f6 has no `parent_bead`, so there is no ancestor to close.
6. Commit the bob-mac-capture changes and the plan-file status through the turn's
   finalizer (`/sase_final` repository obligations).
