---
tier: tale
title: Reserve Task Link clearing for explicit bang toggles
goal:
  Named task captures ensure Next and relocate their existing Task Link while only the
  bang form can add or clear links.
size: medium
proposed_by: bbugyi200.athena.0mz.f0.f0.f0
create_time: 2026-09-18 13:45:25
status: wip
---

# Plan: Reserve Task Link clearing for `!` and make `#pomodoro` relocate

## Why this exists

The recently swapped task-toggle spellings now make plain `@route+block-id` the safe,
idempotent Ensure Next operation and reserve `@route+block-id!` for the explicit two-way
add/clear toggle. One older branch was deliberately left in place, however: marker-only
`@route+block-id#pomodoro` still uses the two-way toggle. If its task is already Next,
Bob changes it to Ready, ignores the named selector, and deletes matching Task Links
from every open Pomodoro.

That is not the desired final contract. Among these marker-only task operations, `!`
should be the only spelling that may delete a Task Link or synthesize one:

```text
@file+id             ensure Next; move the existing Task Link to implicit current/next
@file+id#pomodoro    ensure Next; move the existing Task Link to the named Pomodoro
@file+id!            explicit two-way toggle; add a missing link or clear open links
```

For `#pomodoro`, “move” means preserving the dedicated Task Link bullet and its complete
descendant subtree. If the link is already under the resolved named Pomodoro, the ledger
is unchanged. Preserve the established named-selector contract: whole-slug matches beat
prefix matches, and a valid selector with no matching open entry (including a
completed-only match) creates the canonical named future Pomodoro before moving the
existing subtree into it. The named Ensure Next form must never create a missing Task
Link; that remains an explicit reason to use `@file+id!`.

This is a coordinated bob-cli and Bob Mac Capture contract change. JSON and UI must
remain behavior-driven so previews, VoiceOver, human output, notifications, and
notification targets describe whether status changed, the link moved, a named Pomodoro
was created, or the request was a true no-op.

## Work

### 1. Make both unsuffixed marker-only forms resolve to Ensure Next

In `src/native/capture_language.rs`, change the final whole-item resolution so both
exact marker-only `@route+block-id` and `@route+block-id#pomodoro` carry
`TaskToggleIntent::EnsureNext`. Keep the optional Pomodoro name on the typed
`CaptureKind::TaskToggle`; the execution layer, rather than punctuation checks, should
choose the implicit versus named relocation destination. Terminal `@route+block-id!`
remains the only `TaskToggleIntent::Toggle` spelling.

Do not broaden marker eligibility:

- `!` remains invalid with `#pomodoro`, body text, authored children, `@@`, clipboard,
  schedule, priority, or forced destination flags.
- Body-bearing `text @route+block-id#section` remains an ordinary child capture into an
  ALL-CAPS task section.
- Empty/incomplete selectors, route/task/Pomodoro completion ranges, UTF-8 offsets,
  leading-`@` completion, and the route-to-task-to-Pomodoro picker handoff stay
  unchanged.

Update execution-parser, editor-parser, and completion tests as a three-spelling matrix.
Pin that plain and named marker-only forms carry Ensure Next, `!` alone carries the
two-way intent, and named forms retain normal route, block-ID, and Pomodoro-name spans
without the explicit-toggle modifier span.

### 2. Generalize relocation to an implicit or named destination

In `src/native/capture_task_toggle.rs`, replace the implicit-only relocation entry point
with a typed destination policy (or equivalent) that distinguishes:

- the existing implicit current/next selection used by plain `@route+id`; and
- named open selection or canonical named-future creation used by `@route+id#pomodoro`.

Keep the relocation planner pure and atomic. It must first find exactly one movable
dedicated Task Link under an open Pomodoro, ignoring completed history and prose
lookalikes. Zero occurrences remain an actionable error; multiple occurrences remain an
invariant error instead of silently deleting duplicates. The source is always moved as a
complete subtree, with only its root indentation adapted to the destination.

For an existing named open entry, reuse `capture_pomodoros::select_named` so whole-slug
matching wins over prefix matching and selection does not depend on the implicit
current-Pomodoro ambiguity guard. If the source already belongs to that entry, return
`already_current` with byte-identical content. Otherwise move the subtree earlier or
later in the ledger and report source/destination endpoints.

For a valid selector with no matching open entry or only a completed match, preserve the
current `NamedOrCreate` behavior: create one open placeholder named with the canonical
ALL-CAPS value, using the existing current / last-completed / first-entry anchoring and
multiple-open-timed validation, then move the existing subtree beneath it. Refactor the
named-entry insertion primitive out of `capture.rs` if needed so relocation and ordinary
Pomodoro-linked capture share one canonical creation implementation. Do not implement
creation by inserting a fresh bare link and deleting the source, because that would lose
descendants.

Extend the relocation result only as needed to carry resolved name, placement, whether
the named Pomodoro was created, and accurate one-based endpoints. Unit tests should
cover existing exact and prefix matches, already-at-destination, completed-only and
missing-name creation, invalid names, existing named selection despite multiple timed
entries, creation rejection with multiple timed entries, earlier/later moves, descendant
preservation, indentation, CRLF/no-final-newline input, same-location no-op,
completed-history exclusion, zero/multiple movable links, and source/destination line
accounting when creation or removal shifts offsets.

### 3. Route named captures through the Ensure Next executor and report truthfully

In `src/native/capture.rs`, pass the optional Pomodoro selector into
`plan_ensure_next_capture` and select the new destination policy there. Named and
implicit forms should share the existing task-side Ensure Next behavior: Ready, Blocked,
In Progress, and Next are eligible; every successful result ends Next; a future schedule
is retired under the existing rules; dependencies still warn; and route-note and
daily-note changes are committed atomically from the batch planner's staged snapshots.

For named results, keep schema version 1 and the behavior-level additive fields:

- `toggle_behavior: "ensure_next"` and `toggle_direction: "next"`;
- `status_changed` and `pomodoro_link_action: "moved"|"already_current"`;
- source and destination endpoint objects;
- `pomodoro_name` set to the resolved canonical destination name;
- `creates_pomodoro: true` only when this operation created the named entry;
- `pomodoro_already_linked: true` only for the already-at-destination no-op; and
- `removed_pomodoro_links: 0` with no `pomodoro_selector_unused` clearing story.

If no movable link exists, fail without writes and point to `@route+block-id!` only when
the user intentionally wants link insertion. Do not stage byte-identical task or day
content. Preserve same-file planning, dry run, multi-item staged snapshots, and
all-or-nothing rollback.

Keep the two-way branch exclusive to `!`. Its Ready/Blocked-to-Next path may add a
missing link and perform its established duplicate cleanup; its Next-to-Ready path may
delete links from open Pomodoros. Add regression coverage proving that neither
unsuffixed form can enter insertion/removal cleanup and that `!` is the only marker-only
syntax that can clear a Next task and delete its open links.

Update human output so a named move identifies both Pomodoros, a created destination is
explicitly identified, and a no-op says the link is already in the resolved name rather
than the generic “current/next Pomodoro.” Expand CLI integration tests across Ready,
Blocked, In Progress, and Next; status-only, move-only, combined, creation, and
total-no-op outcomes; exact/prefix selection; future-schedule handling;
missing/duplicate/completed-only links; completed destinations; dry run; same-note
edits; staged batches and rollback; human output; and the exact additive JSON. Retain
focused `!` tests in both directions.

### 4. Make Bob Mac Capture exercise and present named relocation outcomes

In the linked `bob-mac-capture` repository, continue deriving UI behavior only from
Bob's returned `toggle_behavior`, direction, status, relocation, endpoint, and creation
fields—never by parsing `#` or `!` in the Swift client.

Add explicit fake-Bob parse, preview, and committed capture responses for a named
`@file+id#pomodoro` Ensure Next request. Cover an actual move, an already-at-name no-op,
and named-Pomodoro creation. The panel should retain the Pomodoro completion flow,
submit the exact named draft, show **Ensure Next** rather than **Set Open**, and
announce the status and named relocation result.

Extend `CaptureTogglePresentation` only where the existing behavior-driven presentation
is insufficient: no-op copy should name the destination, creation should be visible in
concise preview/VoiceOver/notification copy, and the daily note should be included in
notification actions exactly when relocation or named creation changed it. Notification
titles/bodies must distinguish combined status-and-move, move-only,
status-only/already-at-name, and complete no-op outcomes without showing legacy “removed
links” or “selector unused” copy. Preserve tolerant decoding of older Bob results and
the existing same-path de-duplication for notification targets.

Add focused `CaptureCore`, panel-model, and notification tests for these named responses
plus a regression that `@file+id!` still uses Set Next/Open and is the only form whose
returned result can present link deletion.

### 5. Align documentation and verify both repositories

Update bob-cli's `README.md`, `docs/capture.md`, and `bob capture --help` so the
task-operation table and prose state the final contract consistently. Remove the
now-stale description of marker-only `@route+id#pomodoro` as a two-way toggle, its
inert-on-clear behavior, and `pomodoro_selector_unused` for that spelling. Document
exact/prefix named selection, create-on-miss, the existing-link precondition,
subtree-preserving move, named no-op, and `!` as the sole explicit add/clear escape
hatch. Keep body-bearing `#section` and `@route:block-id#pomodoro` creation docs
distinct.

Update the Bob Mac Capture README/minimum-Bob capability notes and interaction examples
to the same three-spelling matrix and behavior-driven compatibility story.

Run bob-cli verification:

```sh
cargo fmt --check
cargo clippy --all-targets --all-features
cargo test
just check-scripts
```

Run the linked Mac app's macOS 26 Apple-toolchain verification:

```sh
just format-lint
just build
just test
just bundle
```

Finally use a disposable vault to exercise all three spellings end to end: move a
descendant-bearing link to an existing named Pomodoro, repeat for a true no-op, create a
missing named future Pomodoro and move into it, confirm plain syntax still targets
implicit current/next, and confirm only `!` can insert a missing link or clear a Next
task's open links. Spot-check notification copy and Open Note/Open Notes targets for
move, creation, and no-op outcomes.

## Done when

- Marker-only `@route+block-id#pomodoro` ensures every eligible open task is Next and
  moves its sole existing open-Pomodoro Task Link subtree to the resolved named open
  Pomodoro, or creates that named future Pomodoro and moves the subtree there.
- Repeating the named request when the task is already Next and its link is already
  under that destination performs no writes and reports the named no-op.
- Plain and named Ensure Next never synthesize or delete Task Links, never edit
  completed history, and fail atomically on missing or ambiguous movable links.
- `@route+block-id!` is the only marker-only task syntax that performs the established
  two-way add/clear behavior, including open-link deletion when clearing Next.
- Parser intent, CLI execution, JSON, human output, Mac preview, status and VoiceOver,
  notifications, notification targets, help, and docs all agree on the three-spelling
  contract.
- All Rust/script and macOS app verification passes, and disposable-vault checks
  demonstrate named move, named creation, named no-op, implicit relocation, and explicit
  `!` insertion/removal without losing descendant notes.

## Out of scope

- Allowing `!` together with `#pomodoro`, `@@`, body-bearing sub-bullet capture,
  authored children, or other item-wide metadata.
- Creating a missing Task Link in either Ensure Next form, moving links out of completed
  Pomodoros, silently repairing duplicate links, or changing the broader
  `task-status-hooks` reconciliation.
- Changing body-bearing `@route+id#section`, new-task `@route:id#pomodoro`, the Pomodoro
  completion/name-assignment workflows, or general notification design outside accurate
  task-relocation presentation.
