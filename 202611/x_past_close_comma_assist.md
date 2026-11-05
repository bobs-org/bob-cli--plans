---
tier: tale
title: Auto-comma and Backspace for x= close lists in Bob Mac Capture
goal:
  Typing task numbers after the second = of an x= past record auto-inserts and
  Backspace-removes commas, as x does.
size: medium
decisions:
  bound_source:
    ask:
      Should bob-cli report the x= lineup bound as a new capture-pomodoros JSON field?
    choices:
      bob_field:
        bob-cli adds max_task_link_count; the x= assist waits for that bob release on
        the Mac
      agenda_items:
        Mac-only; Swift counts each open agenda entry's Task Link items; works with
        today's bob
    default: bob_field
    why:
      bob owns the x= target rules and the close numbering, just as it supplied
      task_link_count for x
    answer: bob_field
proposed_by: bbugyi200.apollo.cl
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-05 09:05:06
status: wip
---

# Auto-comma and Backspace for `x=` close lists in Bob Mac Capture

## Goal

Bob Mac Capture inserts the `,` between close task numbers for `x`: typing `x12!34`
shows `x1,2!3,4`. Backspace right after an auto-inserted `,<digit>` removes both
characters. Add the same behavior to the close part of a past-Pomodoro record,
`x=E[!]=X`. That close part is everything after the second `=`.

| Typed keys                 | Editor shows                                            |
| -------------------------- | ------------------------------------------------------- |
| `x==12`                    | `x==1,2`                                                |
| `x=3=12!34`                | `x=3=1,2!3,4`                                           |
| `x=fix@2=*13`              | `x=fix@2=*1,3`                                          |
| `x=3!=!12~34`              | `x=3!=!1,2~3,4`                                         |
| `x=#"DEEP WORK"=12`        | `x=#"DEEP WORK"=1,2`                                    |
| `x=build@2=12 x=fix@3=*34` | `x=build@2=1,2 x=fix@3=*3,4`                            |
| `x1 x==23` (chain)         | `x1 x==2,3`, and the `x` list uses its own gate (below) |

Backspace after an assisted pair: `x==1,2` becomes `x==1`.

The assist never fires on these, so the digit types normally:

- the block count: `x=3` + `1` gives `x=31`;
- the `@D` count: `x=build@2` + `1` gives `x=build@21`;
- the first number of a group (`x==` + `1`, `x==1!` + `2`);
- the inline Work Log number (`x== 1` + `2`) and Work Log bullets;
- `0`;
- any text that is not a close list.

## What already works, and what is missing

Investigation for this plan found the following.

- **Spans already cover `x=`.** `bob capture-parse` emits the same close-list spans
  inside `x=` that the Swift helper already filters (`pomodoro_close_in_progress`,
  `_park`, `_complete`, `_drop`). Spans verified with the installed bob:
  - `x==1`: `pomodoro_past` [0,2), `pomodoro_close` [2,3), `pomodoro_close_in_progress`
    [3,4). Mode is `pomodoro_past`, with no `items` key.
  - `x=3=1`: `pomodoro_past` [0,3), `pomodoro_close` [3,4), in_progress [4,5).
  - `x=build=1`: past [0,2), `pomodoro_name` [2,7), close [7,8), in_progress [8,9).
  - `x=fix@2=*1`: past [0,2), name [2,5), past [5,7), close [7,8), `pomodoro_close_park`
    [8,10).
  - `x=3!=!1`: past [0,3), `pomodoro_past_repair` [3,4), close [4,5),
    `pomodoro_close_complete` [5,7).
  - `x==1!2`: past [0,2), close [2,3), in_progress [3,4), complete [4,6).
  - `x=#"DEEP WORK"=1`: past [0,2), name [3,14), close [14,15), in_progress [15,16).
- **Backspace needs no new mechanism.** `CaptureCloseTaskCommaProvenance` records every
  pair the controller applied, whatever syntax it belongs to. Once insertion fires
  inside `x=`, Backspace works through the existing `.deleteBackward` path.
- **The gate is what is missing.** The assist is armed only by
  `currentPomodoroTaskLinkCount`, which counts the numbered Task Links of the _running_
  Pomodoro, and it is nil when no session runs. `x=` usually records work while nothing
  runs, so the assist is off. Even when a session runs, `x=` numbers a different lineup:
  the lineup of its _target row_. The record engine
  (`src/native/capture/pomodoro_past.rs`, `resolve…`/`record_one_member`) picks one of
  these targets:
  - an unnamed token takes the next future untimed placeholder;
  - a named token reuses an open untimed placeholder with that name;
  - otherwise a fresh "again" or "created" row is inserted with no links.

  It then closes that row through the shared close engine. So the right count for an
  `x=` list is an upper bound over the lineups of today's open entries. A fresh row has
  0 links.

## Design

The app stays a thin client (`decisions:mac-capture-is-a-thin-client`). It still never
recognizes close syntax. Bob gives it two facts:

1. **Which item owns the list.** `capture-parse` already reports per-item `mode` and
   UTF-8 `range` in `items[]` (verified):
   - `x1 x==2` → `pomodoro_close` [0,2), `pomodoro_past` [3,7);
   - `x1\n\nx==2` → `pomodoro_close` [0,2), `pomodoro_past` [4,8);
   - `x==1 =\n- 1 did` → `pomodoro_past` [0,14) with a nested `pomodoro_start` [5,6).

   A single-item draft has no `items` key, so the top-level `mode` covers the whole
   draft. Pick the owner like this: take the innermost item range that contains the
   caret's list span. Its `mode == "pomodoro_past"` means the list is an `x=` list.
   Anything else, or no item data at all, means an `x` or link close list, which keeps
   today's gate.

2. **The `x=` lineup bound.** It is "fewer than 10" for the largest numbered Task Link
   lineup among today's open entries (the running session and every placeholder). The
   same `singleDigitLineupLimit` (10) applies. An unknown bound (nil) leaves `x=` lists
   unassisted.

The two gates are independent. In `x1 x==23`, the `x` list uses the running count and
the `x=` list uses the bound. The router and the immediate assist parse are armed when
_either_ gate is armed. The pure helper then picks the gate for the list under the
caret.

**Accepted misses, to document.**

- The bound is conservative. If any open entry has 10 or more numbered links, `x=` lists
  are unassisted, even when the actual target has fewer.
- A same-draft chain whose earlier `x` close (or earlier stack member) carries links
  into the `x=` target can push that target past the bound. Example: 6 running links
  carried into a 6-link placeholder. The original `x` assist accepted the same kind of
  miss for batches that link new tasks before their close.
- Both misses need 10 or more links and a typed two-digit index. Neither can corrupt a
  lineup under 10.

## Part 1: bob-cli, `max_task_link_count` on `bob capture-pomodoros`

> [!decision] bound_source = agenda_items Skip Part 1 entirely and make no bob-cli
> change. In Part 2, compute the bound in Swift as described in step 2b's `agenda_items`
> note.

This lives in the bob-cli workspace.

1. In `src/native/capture_pomodoros.rs`, extract the body of `current_task_link_count`
   into a helper that numbers one entry's lineup. That helper builds the
   `capture_pomodoro_close::RunningPomodoro` (`line`, `name`,
   `time_range.unwrap_or_default()`) and returns `number_task_links(...).len()`. Keep
   `current_task_link_count` on top of it. Add a top-level `max_task_link_count: usize`
   to `CapturePomodorosResult`:
   - It is the maximum over every `scan.entries` entry with
     `state == PomodoroState::Open`, or `0` when none is open.
   - It ignores completed entries even under `--all`.
   - Serialize it always, with or without `--tasks`.
   - Make the missing-daily-note and missing-section success paths report `0` too.
   - Do not touch the shared `PomodoroEntry` or `scan()`, or per-entry
     `task_link_count`.
   - Leave human output unchanged and `schema_version` at 1. The change is additive.
2. Help: in the `long_about`, extend the clause "…and the current entry's numbered Task
   Link count for picker callers" with one clear clause about `max_task_link_count`: the
   largest numbered Task Link lineup among open entries. Then run `just docs-gen` to
   regenerate `docs/reference/cli/capture-pomodoros.md`.
3. Docs:
   - In `docs/capture.md`, find the Discovery-commands paragraph that defines
     `task_link_count` (search "the size of the numbered Task Link lineup that a plain
     `x` close indexes"). Add the top-level field there. It is the largest such lineup
     among today's open entries (the running session and every placeholder), `0` when
     none is open, and an integer regardless of `--tasks`. An `x=` record closes one of
     those rows or a fresh row with no links, so editors use it to decide whether an
     `x=` close task number can be multi-digit.
   - Add `max_task_link_count` to the `capture-pomodoros` row in
     `docs/reference/json-outputs.md`.
4. Tests:
   - Unit tests beside the existing `task_link_count_*` tests in `capture_pomodoros.rs`:
     - no running session, with placeholders whose numbered lineups are 2 and 3. Include
       a struck link and a prose note, which are not counted. The result is 3.
     - a running session with 4 links plus a 2-link placeholder gives 4.
     - a completed entry with 12 links is ignored, with and without `include_all`.
     - no open entries gives 0.
     - an open placeholder with 12 links reports 12.
   - A CLI parity test next to
     `capture_pomodoros_task_link_count_matches_dry_run_lineup` in
     `tests/cli/capture/pomodoro_name.rs`:
     - Fixture: no running session, and the first future placeholder holds the largest
       lineup.
     - Assert that `max_task_link_count` from `bob capture-pomodoros -f json` (and with
       `-t`) equals the length of `pomodoro_past.close.task_links` from
       `bob capture --dry-run --no-clip -f json -- 'x=='`, using the same
       `BOB_DAY_FILE`/`BOB_NOW`.
     - Also assert that `capture-pomodoro-name` JSON still lacks the field.
5. Run `just check` and `just docs-check`.
6. Commit subject:
   `feat(capture-pomodoros): report max_task_link_count for x= close lists`. It has no
   `!` because the change is additive. Do not edit the `Cargo.toml` version or
   `CHANGELOG.md`; the release train ships it.

## Part 2: Bob Mac Capture

Open the repo with `/sase_repo`:
`sase repo open bob-mac-capture -r "Add x= close-list auto-comma"`. Use only the path it
prints, and read its `AGENTS.md` first. Its rules are binding:

- compile every Swift change;
- format only the lines you change (4-space indentation, 100 columns);
- `just docs-check` for `docs/`.

Use `decodeIfPresent` so an older bob without the field decodes to nil and leaves `x=`
unassisted.

### 2a. CaptureCore: snapshot items and gate selection (`Sources/CaptureCore/CaptureCloseTaskCommaAssist.swift`)

- `CaptureParseSnapshot` gains `items: [CaptureParseSnapshot.Item]`. `Item` holds
  `start`, `end` (UTF-8 bytes, like spans) and `mode`. The memberwise init takes
  `items: [Item] = []` so existing call sites and tests compile unchanged.
- Add `init(draft: String, parse: CaptureParseResponse)`:
  - non-empty `parse.items` maps each `CaptureParseItem` to
    `Item(range.start, range.end, mode)`;
  - otherwise use one item `0..<draft.utf8.count` with `parse.mode`.
- `CaptureCloseTaskCommaAssist`:
  - Add `static let pastItemMode = "pomodoro_past"`.
  - Add `static func isArmed(taskLinkCount: Int?, pastTaskLinkCount: Int?) -> Bool`,
    which is true when either count is armed. Keep the single-argument `isArmed`.
  - `edit(typed:text:selectedRange:snapshot:taskLinkCount:pastTaskLinkCount: Int? = nil)`:
    - Keep the digit, collapsed-selection, current-snapshot, UTF-16 to UTF-8 caret, and
      previous-byte `1`–`9` checks.
    - Find the list span with `kind ∈ listSpanKinds` and `start < c <= end`.
    - Then pick its owner: the smallest `snapshot.items` range with
      `item.start <= span.start && span.end <= item.end`.
    - Gate with `isArmed(taskLinkCount: pastTaskLinkCount)` when the owner's mode is
      `pastItemMode`. Otherwise gate with `isArmed(taskLinkCount: taskLinkCount)`.
    - The early `isArmed` guard moves after the span lookup.
  - Update the doc comments: the helper still only filters Bob's spans and item modes.
- Agenda bound:
  - `bob_field`: `CaptureAgendaSnapshot`
    (`Sources/CaptureCore/CaptureAgendaModels.swift`) decodes `maxTaskLinkCount: Int?`
    from `max_task_link_count` with `decodeIfPresent`. Add an init parameter defaulting
    to nil.

> [!decision] bound_source = agenda_items Instead of decoding, add a computed
> `CaptureAgendaSnapshot.maxTaskLinkCount: Int`: the max over `pomodoros` whose
> `role != .completed` of `max(items.count, taskLinkCount ?? 0)`, or `0` when there are
> none. Do this because bob's agenda numbering walk (`number_task_links_in_range`) emits
> one item per numbered lineup link for every open entry. Only the `--tasks` snapshot
> has items, so the plain-lane fallback leaves the bound nil.

### 2b. Store and model

- `Sources/BobMacCapture/CaptureAgendaStore.swift`:
  - Add `@Published public private(set) var maxTaskLinkCount: Int?`.
  - Set it in `publish(_:)` from the snapshot, only on change.
  - Nil it everywhere `currentTaskLinkCount` is nilled: the `processClient` `didSet`,
    `reset()`, the non-today `.changed` branch, and `finishFailure`.
  - The old-bob plain lane leaves it nil.
  - Update the type's doc comment.
- `Sources/BobMacCapture/CapturePanelModel.swift`:
  - Add `private(set) var maxTaskLinkCount: Int?`, fed by a second subscription in
    `subscribeToAgendaStore()` and cleared in `setProcessClient` when the client
    changes. Do this next to `currentPomodoroTaskLinkCount = nil`.
  - `closeTaskCommaArmed` uses the two-argument `isArmed`. That arms the router context
    (`CapturePanelController`'s `closeTaskCommaArmed:`) and the immediate assist parse
    in `editorTextDidChange` with no further change.
  - `closeTaskCommaEdit` passes `pastTaskLinkCount: maxTaskLinkCount`.
  - Both snapshot sites switch to `CaptureParseSnapshot(draft:parse:)`: the
    `close-task-comma` lane in `startCloseListAssistParse` and the debounced
    `applyParse` path.
  - Add `setMaxTaskLinkCountForTests(_:)` beside
    `setCurrentPomodoroTaskLinkCountForTests`.
  - Update the comments that say the assist depends only on the running Pomodoro.
- `CaptureKeyCommandRouter` and the controller's insert and Backspace helpers need no
  logic change. Touch only comments that would become false.

### 2c. Tests and fixtures

- `Tests/CaptureCoreTests/CaptureCloseTaskCommaAssistTests.swift`: use spans and items
  copied from the real outputs above. For single-item drafts, use one whole-draft item
  with mode `pomodoro_past`.
  - Fires with `taskLinkCount: nil, pastTaskLinkCount: 3`:
    - `x==1` + `2`;
    - `x=3=1` + `2`;
    - `x=build=1` + `2`;
    - `x=fix@2=*1` + `3` (park);
    - `x=3!=!1` + `2` (complete);
    - caret after `1` in `x==1!2` + `3`;
    - `x=#"DEEP WORK"=1` + `2`;
    - nested `x==1 =\n- 1 did` with the caret after `1` + `2`.
  - Routing:
    - In `x1 x==2` and `x1\n\nx==2`: with running nil and past 3, the `x=` list fires
      and the `x` list declines. With running 3 and past 12, the `x` list fires and the
      `x=` list declines.
    - Existing snapshots without items keep the running gate. All current tests stay
      green unchanged.
  - Declines:
    - past nil, 10, and 12;
    - `x=3` + `1` (only a `pomodoro_past` span);
    - `x=build@2` + `1`;
    - `x==` + `1`;
    - `x== 1` + `2` (no list span);
    - `0`.
  - Two-argument `isArmed` truth table.
  - `init(draft:parse:)` decodes the existing `pomodoro-past-parse-close-mix.json`
    (items `pomodoro_close` [0,1), `pomodoro_past` [2,11)) and
    `pomodoro-past-parse-single.json` (no items, so one whole-draft `pomodoro_past`
    item).
- Agenda decoding (`Tests/CaptureCoreTests/CaptureAgendaModelsTests.swift`):
  - Add new fixtures `agenda-nothing-running-past-3.json` and `agenda-past-12.json`.
    Copy `agenda-nothing-running.json`, add `max_task_link_count` 3 / 12, and keep the
    date. Do not edit existing fixtures: design and layout tests enumerate them.
  - Existing fixtures must decode nil (`bob_field`) or their computed bound
    (`agenda_items`).
- `Tests/BobMacCaptureTests/CaptureAgendaStoreTests.swift`: the store publishes, then
  clears, `maxTaskLinkCount` on failure, a non-today snapshot, `reset()`, and client
  replacement, mirroring the `currentTaskLinkCount` tests.
- `Tests/BobMacCaptureTests/CaptureCloseTaskCommaTests.swift`:
  - `refreshModel(agendaFixture: "agenda-nothing-running-past-3.json")` arms
    `closeTaskCommaArmed` with `currentPomodoroTaskLinkCount == nil`.
  - The 12 fixture leaves it disarmed.
  - Mirror `testKeyDrivenAssistParseServesCommaEdit` for `x==1` + `2`, setting only
    `setMaxTaskLinkCountForTests(3)`. Serve the parse through fake-bob, using
    `FAKE_BOB_PARSE_FIXTURE` with a new recorded `x==1` parse fixture if the dynamic
    fake parse does not cover `x=`.
  - Backspace removes the assisted pair in `x==1,2` (→ `x==1`).
  - Replacing the client clears `maxTaskLinkCount`, like
    `testOldCountNeverPublishesAfterReplacement`.
- The router tests need no change, because the router still reads one boolean.

### 2d. Docs (`docs/` is mounted into Bob's site)

- `docs/capture-panel.md`, the auto-comma paragraph (search "auto-inserts `,` between
  close task numbers"):
  - Add the `x=` example (`x==1` + `2` → `x==1,2`).
  - State the two gates: `x` and link lists use the running Pomodoro's count; `x=` lists
    use the largest open lineup.
  - Document the accepted misses and that an older bob disables only the `x=` gate.
- `docs/protocol.md`:
  - In the `capture-pomodoros` lane line, mention `max_task_link_count`.
  - In the "Close-comma assist field" row, name both fields.
- `docs/idle-agenda.md`: fix any sentence that says the store serves only "the
  close-comma count" if it now reads wrong.
- Run `just docs-check`.

## Verification and landing

- bob-cli: `just check` and `just docs-check` must pass.
- Mac:
  - Run `just portable-check` on a host with Swift. If `swift` is missing here, use
    athena.
  - Then compile the `BobMacCapture*` targets on the mac in a scratch copy, as AGENTS.md
    describes: rsync without `.build`/`.git`, `Scripts/bundle.sh`,
    `just typecheck-tests`, then delete the copy. Never use Bryan's checkout or
    `just install`.
  - Before declaring the Mac commit, run `Scripts/macos-ci-check` from the Mac checkout.
    It runs exact-tree macOS CI (lint, build, tests, bundle, install). Run it in the
    foreground with a large timeout, or hand it to `/sase_monitor`. Every macOS step
    must be green. After a red lint step, inspect the later steps too.
  - If the mac is offline, say in the final report that the macOS targets are
    unverified.
- Manual check, if a Mac session is possible: with nothing running and placeholders
  under 10 links, type `x==12` and see `x==1,2`, then Backspace back to `x==1`. Also
  type `x=3` + `1` and see `x=31`.
- Commit each repository through `/sase_final` repository declarations: the bob-cli
  workspace and the opened bob-mac-capture checkout. Do not push by hand or open PRs.
  Suggested Mac subject: `feat(capture): auto-insert commas in x= close lists`. Neither
  side depends on the other at build time. The Mac reads an absent field as nil and
  leaves `x=` lists unassisted until the bob release that has it.
