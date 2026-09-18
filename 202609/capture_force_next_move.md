---
tier: tale
title: Add force-Next task-link relocation
goal:
  A terminal bang capture atomically ensures an open task is Next, relocates its
  existing open-Pomodoro Task Link to the current or next slot, and reports the exact
  outcome through the CLI and Mac app.
size: medium
proposed_by: bbugyi200.athena.0mz.f0
create_time: 2026-09-18 12:19:05
status: wip
---

# Plan: Add force-Next task-link relocation with `@route+block-id!`

## Why this exists

The marker-only `@route+block-id` capture is currently a two-way task toggle. Ready and
Blocked tasks become Next and gain a link under today's selected open Pomodoro; Next
tasks become Ready and lose their links from open Pomodoros. That behavior is useful for
starting and clearing work, but it is the wrong operation when a task is already planned
under a later open Pomodoro and the user wants to pull it into the current (or next
future) slot without any risk of toggling an already-Next task off.

Add a terminal `!` modifier for that one-way operation:

```text
@file+id!
```

The modifier means “ensure this open task is Next, and relocate its existing Task Link
to today's implicit current/next open Pomodoro.” It is not an instruction to create a
new link. The route note and daily note must still be planned atomically, and the Bob
Mac Capture preview, status announcement, and post-capture notification must distinguish
a real status change, a real move, and a complete no-op.

The semantic contract for this plan is deliberately narrow:

- `!` is accepted only as the final byte of a marker-only `@route+block-id!` item. It
  does not add a force mode to child-bullet captures, global `@@` destinations, named
  `#pomodoro` selectors, or markers combined with body text, authored children,
  clipboard, schedule, priority, or forced destination flags.
- The destination uses the established implicit selection rule: the single open timed
  Pomodoro when present, otherwise the first open Pomodoro in document order. A missing
  Pomodoros section, no eligible open entry, or multiple open timed entries remains an
  atomic error.
- Only a dedicated Task Link bullet beneath an open Pomodoro is movable. Completed
  Pomodoros are historical and must never be edited by this operation. A link embedded
  in surrounding prose is not a Task Link under the project's canonical definition.
- No matching open Task Link is an actionable, write-free error that tells the user to
  use the ordinary `@route+block-id` toggle if they intend to add one. More than one
  movable occurrence is also a write-free invariant error rather than silently choosing
  which occurrence and descendant notes win.
- When the sole link is already under the selected destination, the daily file remains
  byte-for-byte unchanged. The task-side force-Next plan is independent: it may still
  set a non-Next open status or retire a future schedule, but an already-Next task with
  its link already at the destination is a true no-op.
- When relocation is needed, move the dedicated link bullet and its complete descendant
  subtree. Preserve descendant text and relative nesting, adapt only the root
  indentation to the destination's established child indentation, preserve line endings,
  and leave both Pomodoro entry lines intact. Do not degrade a move into
  remove-plus-new-link in a way that loses notes beneath the Task Link.
- Ready `[ ]`, Blocked `[?]`, In Progress `[/]`, and Next `[*]` are eligible open task
  states and all end as Next. Done, canceled, unknown, missing, non-task, and
  duplicate-ID targets retain actionable errors. Reuse the existing
  single-future-schedule retirement, existing-Schedule-Log update, dependency warning,
  dry-run, and batch rollback rules.

The force-Next status behavior has an existing precedent in the `bob-plugins`
`block-id-prompt` planner, but this syntax and relocation contract belong in `bob-cli`.
The plugin is reference context only and is not changed by this plan.

## Work

### 1. Extend the shared capture grammar without weakening ordinary marker rules

In `src/native/capture_language.rs`, represent task-toggle intent explicitly rather than
inferring it later from a malformed block ID. A small enum or equivalent typed field on
`CaptureKind::TaskToggle` should distinguish the existing two-way toggle from the new
ensure-Next operation.

Teach execution parsing, editor parsing, span generation, and cursor-aware completion
about a terminal `!` on the otherwise complete `@route+block-id` marker:

- Strip the modifier before route/block-ID validation so `id!` is not reported as an
  invalid block ID, but only after proving this is the exact terminal force form.
- Keep the editor mode `task_toggle`, the route and block ID values unchanged, and add a
  dedicated additive span kind for the modifier (for example `task_toggle_force_next`)
  so clients can color it without folding it into the block-ID replacement range.
- Keep route and task completion working while the marker is built. The task completion
  replacement ends before `!`; a caret on or after the completed modifier must not cause
  Bob to replace the modifier as part of the ID.
- Produce focused diagnostics for unsupported near-misses such as a body-bearing
  `text @route+id!`, `@route+id#!`, `@route+id#name!`, `@@route+id!`, repeated `!!`, and
  force markers combined with other item-wide metadata. Ordinary literal exclamation
  points in capture prose remain literal.
- Preserve byte-accurate UTF-8 spans and execution/editor grammar parity. Existing
  `@route+id`, `@route+id#name`, ordinary child-bullet, global destination, rewrite, and
  completion behavior must remain unchanged.

Update `capture-parse` contract tests to pin the new mode, block ID, force-modifier
span, empty body, and absence of diagnostics for the valid form. Add execution parser
and completion tests for leading/trailing placement rules, multi-item drafts, invalid
combinations, UTF-8 offsets, and a block-ID completion immediately before the suffix.

### 2. Add an open-Pomodoro Task Link relocation planner

In `src/native/capture_task_toggle.rs`, add a pure planner dedicated to moving an
existing canonical Task Link. Reuse the Pomodoro scanner and the same implicit
current/next destination selection used by ordinary toggle insertion, but keep movement
separate from `plan_link_insertion`: the bang form must never synthesize a missing link
or create a named future Pomodoro.

The planner should:

1. Validate the Pomodoros section and resolve the destination before returning any
   postimage.
2. Inspect direct child list items of open Pomodoro entries for a dedicated canonical
   `[[route#^block-id]]` Task Link, using the existing list-item/subtree boundaries and
   fenced/nested exclusions. Completed entries are ignored for both source and cleanup.
3. Return typed errors for zero or multiple movable occurrences. If only completed or
   non-dedicated occurrences exist, the error should explain that no movable open
   Pomodoro Task Link was found.
4. Return an `already_current` outcome with the original content when the source owner
   is the selected destination.
5. Otherwise remove the entire source list-item subtree, reindent it under the selected
   destination while preserving relative descendant indentation and newline style, and
   insert it at the end of that destination's child block. Account for source-before-
   destination offset shifts and same-file EOF/CRLF cases without overlapping edits.

Return enough structured metadata for callers to report the result without reparsing:
whether content changed, `moved` versus `already_current`, insertion placement, and
source/destination endpoint data (one-based line plus optional Pomodoro name and time
range). Unit-test movement both earlier and later in the ledger, the timed-current and
first-open fallback choices, an unnamed endpoint, a descendant-bearing Task Link,
established indentation, CRLF/no-final-newline input, completed-history exclusion,
missing and duplicate links, mixed-text lookalikes, and the already-current no-op.

### 3. Wire ensure-Next and relocation into atomic capture planning and output

Refactor `plan_task_toggle_capture` in `src/native/capture.rs` around the typed toggle
intent while leaving the existing two-way branch behavior intact.

For ensure-Next:

- Resolve and validate the route task exactly as today, accepting all four open task
  symbols and rejecting closed/unknown targets.
- Preflight the daily ledger relocation from the batch planner's current staged
  snapshot. A relocation error must abort before commit, so even a task-side
  status/schedule plan is rolled back. This must also work when the route note is the
  daily note and when an earlier item in the same batch staged either file.
- Apply the existing `plan_task_next` semantics idempotently: already Next stays Next;
  the other eligible open states become Next; a single future schedule is retired and
  logged only when an existing Schedule Log is present. Keep the dependency warning.
- Stage the daily file only for `moved`; do not stage a byte-identical `already_current`
  result. Keep dry-run and all-or-nothing multi-item behavior.

Extend the additive success JSON rather than changing schema version 1 or repurposing
the old toggle fields. Continue reporting `kind: "task_toggle"`,
`toggle_direction: "next"`, before/after task lines/statuses, block link, schedule
effects, and destination day file. Add optional fields that let old clients decode the
response unchanged and new clients render it precisely:

- `toggle_behavior: "ensure_next"` for the bang form (absence continues to mean the
  legacy two-way toggle);
- `status_changed: true|false`;
- `pomodoro_link_action: "moved"|"already_current"`; and
- structured `pomodoro_link_source` and `pomodoro_link_destination` endpoints carrying
  line, optional name, and optional time range.

For compatibility, keep `pomodoro_name` as the resolved destination name when one
exists, set `creates_pomodoro: false`, set `pomodoro_already_linked: true` only for the
already-current outcome, and report destination placement only for an actual move. Do
not describe relocation as a “later duplicate removal” through the legacy
`removed_pomodoro_links` wording.

Update human output so the bang form says `would ensure`/`ensured` rather than
`would toggle`/`toggled`, distinguishes “set Next” from “already Next,” and prints
either the source-to-destination Pomodoro move or “Task Link already in current/next
Pomodoro; no ledger change.” A total no-op must be visibly reported as such.

Add CLI integration coverage for Ready, Blocked, In Progress, and already-Next tasks;
status-only, move-only, combined, and total-no-op outcomes; future-schedule retirement;
missing/duplicate/completed-only link failures with zero writes; same-note edits; dry
run; dependency warnings; batch staged-snapshot behavior and rollback; human output; and
the exact additive JSON shape. Keep the legacy toggle regression matrix to prove that
omitting `!` can still clear Next to Ready and retains named-Pomodoro behavior.

### 4. Make Bob Mac Capture preview and notifications outcome-aware

In the linked `bob-mac-capture` repository, decode the new optional response fields in
`Sources/CaptureCore/CaptureModels.swift`, including a typed endpoint model. Missing
fields must preserve compatibility with older Bob binaries and existing toggle results.

Extend `CaptureTogglePresentation` as the single presentation source for the panel,
VoiceOver, and notifications:

- A bang preview/footer uses **Ensure Next**, never **Set Open**.
- Present status and relocation independently: “Ready → Next” versus “Next unchanged,”
  and “Moved LATER → CURRENT” versus “Already in CURRENT; no Pomodoro changes.” Use the
  endpoint name when present, the time range when unnamed, and a concise line-based
  fallback only when neither exists.
- Keep schedule-retirement and Schedule Log chips, but do not show the legacy “adds
  link” or “removed later links” rows for relocation.
- Compute `dayFileChanged` from `pomodoro_link_action == moved`, so notification click/
  Open Notes includes the daily note only when it changed. Preserve the route-note and
  same-path de-duplication behavior.
- Make committed notification titles summarize the actual result: combined status and
  move, status-only, move-only, or already-Next/already-current no-op. The body should
  name the task destination and give the before/after status plus the Pomodoro outcome;
  status text and VoiceOver should carry the same information in concise spoken form.

Map the new parse span to an intentional semantic color category rather than relying on
the unknown-span fallback. Extend `Tests/Fixtures/fake-bob` with explicit parse,
preview, and committed capture responses for `@file+id!`. Add `CaptureCore` tests for
tolerant decoding and every presentation outcome, `NotificationServiceTests` for
title/body and one-versus-two changed-note targets, and focused `CapturePanelModelTests`
proving the modifier keeps the completed marker valid, changes the primary action to
Ensure Next, surfaces the preview outcome, submits the exact draft, announces the
committed result, and does not regress the leading-`@` route-to-task completion flow.

### 5. Document and verify the coordinated contract

Update bob-cli's `README.md`, `docs/capture.md`, and command examples/help to document
the new marker, fixed implicit destination, open-only existing-link precondition,
one-way status rules, completed-history protection, no-op behavior, failure cases, human
output, parse span, and additive JSON fields. Update bob-mac-capture's README to state
the minimum Bob capability and describe the Ensure Next preview/notification contract.

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

Finally exercise the installed/development app against a disposable vault: move a
descendant-bearing Task Link from a later open Pomodoro into the current timed one,
repeat the same `@file+id!` request to confirm a clear total no-op, and confirm the
notification accurately reports both runs. Also spot-check plain `@file+id` in both
directions, task completion from a leading `@`, an In Progress bang request, a missing
link failure that preserves both files, and notification Open Note/Open Notes routing.

## Done when

- `@route+block-id!` is a valid, highlighted, completable marker-only capture that can
  never toggle an already-Next task back to Ready.
- The command atomically ensures an eligible task is Next and moves its sole existing
  open-Pomodoro Task Link subtree to the implicit current/next destination, while never
  editing completed history or creating a missing link.
- Repeating the command with an already-Next task whose Task Link is already at the
  destination performs no writes and reports that no-op truthfully.
- Missing, ambiguous, malformed, and unsupported forms fail with actionable messages and
  leave every staged file unchanged.
- JSON, human CLI output, Mac preview/status/VoiceOver, notifications, and notification
  target actions all agree on whether status changed, the Task Link moved, or nothing
  changed.
- Existing task-toggle, named-Pomodoro, child-bullet, batch, completion, and leading-`@`
  behavior remains covered and unchanged when `!` is absent.
- All Rust/script and macOS app verification commands pass.

## Out of scope

- Adding `!` to `@@` declarations, `@route+id#name`, ordinary child-bullet captures,
  Pomodoro-linked task creation, or any other marker family.
- Creating a missing Task Link, creating/naming a Pomodoro, or moving links out of a
  completed Pomodoro; the ordinary toggle remains the explicit add-link workflow.
- Changing `bob-plugins` keymaps or their existing force-Next implementation.
- Repairing arbitrary duplicate/mixed-text links, deleting an emptied source Pomodoro,
  or running the broader `task-status-hooks` reconciliation as part of capture.
- Redesigning the app's general notification categories, completion UI, or batch
  presentation beyond decoding and presenting this additive toggle outcome.
