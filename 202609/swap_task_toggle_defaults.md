---
tier: tale
title: Make Ensure Next the default task-toggle capture
goal:
  Plain marker-only task captures safely ensure Next by default, while a terminal
  exclamation point explicitly selects the prior two-way toggle across bob-cli and Bob
  Mac Capture.
size: medium
proposed_by: bbugyi200.athena.0mz.f0.f0
create_time: 2026-09-18 13:17:27
status: wip
---

# Plan: Make Ensure Next the default task-toggle capture

## Why this exists

The two marker-only task operations currently assign the safer, idempotent behavior to
the longer spelling:

```text
@file+id    -> two-way Ready/Blocked <-> Next toggle
@file+id!   -> ensure Next and move the existing Task Link to current/next
```

Reverse those exact spellings so the common/default action cannot accidentally clear an
already-Next task:

```text
@file+id    -> ensure Next and move the existing Task Link to current/next
@file+id!   -> explicit two-way Ready/Blocked <-> Next toggle
```

This is intentionally a grammar-to-intent swap, not a redesign of either operation. The
existing ensure-Next relocation planner, atomicity rules, no-op behavior, result
metadata, human output, and Mac notification presentation remain the source of truth for
that behavior. The existing two-way branch still adds a missing link when setting Next,
can select/create a named Pomodoro through the separate marker-only
`@route+block-id#pomodoro` form, and removes open-Pomodoro links when clearing Next.

The semantic boundary is:

- An exact marker-only `@route+block-id` uses ensure-Next. Ready, Blocked, In Progress,
  and Next are eligible; it requires exactly one movable Task Link under an open
  Pomodoro, relocates its full subtree to the implicit current/next Pomodoro, and is a
  true no-op when status and location are already correct.
- An exact marker-only `@route+block-id!` opts into the established two-way toggle.
  Ready and Blocked become Next and use the existing link-insertion path; Next becomes
  Ready and removes matching links from open Pomodoros; In Progress and closed states
  retain the existing toggle errors.
- Marker-only `@route+block-id#pomodoro` remains the existing named two-way toggle. It
  is a distinct syntax whose selector is meaningful to link insertion but not to the
  implicit relocation operation. `!` remains invalid when combined with `#pomodoro`.
- Body-bearing `text @route+block-id` and `text @route+block-id#section` remain ordinary
  existing-task sub-bullet captures. The swap applies only after the whole item proves
  it is marker-only, with no authored children or other item-wide metadata.
- `@@` declarations, repeated `!!`, body text, authored children, clipboard, schedule,
  priority, and forced destination flags remain unsupported with the `!` modifier.

Because the capture result already describes behavior independently of spelling, keep
`toggle_behavior: "ensure_next"` on whichever input actually ran ensure-Next and keep it
absent for the two-way toggle. That lets the CLI, Bob Mac Capture, VoiceOver, and
notifications continue to report the real outcome without guessing from the draft.

## Work

### 1. Reverse the shared grammar's exact-marker intent mapping

In `src/native/capture_language.rs`, keep `TaskToggleIntent` as the behavior-level
contract but change how marker-only inputs resolve to it:

- Resolve plain `@route+block-id` with no `#pomodoro` to `TaskToggleIntent::EnsureNext`.
- Resolve terminal `@route+block-id!` to `TaskToggleIntent::Toggle` after stripping the
  suffix for route and block-ID validation.
- Resolve plain `@route+block-id#pomodoro` to `TaskToggleIntent::Toggle`, preserving its
  existing named-Pomodoro semantics and completion context.
- Keep the decision in the shared execution/editor grammar so `bob capture`,
  `capture-parse`, and `capture-complete` cannot disagree. In particular, do not let a
  body-bearing sub-bullet inherit ensure-Next merely because its spelling has no `!`.

Rename force-Next-specific parser concepts so the code and editor contract describe the
new meaning rather than preserving misleading names. The terminal suffix helper and
diagnostics should describe an explicit two-way toggle, and the suffix should emit an
accurate additive span kind such as `task_toggle_explicit_toggle` instead of
`task_toggle_force_next`. Keep focused diagnostics for invalid `!` combinations, with
examples that point to the valid marker-only `@<route>+<block-id>!` spelling without
calling it force-Next.

Preserve cursor-aware completion across the swap: route and task completion work while
building either form, task-ID replacement ends before `!`, and a caret after the
completed suffix does not replace or absorb it. Keep leading-`@` route completion and
the `+` route-to-task handoff unchanged.

Update grammar, parse, and completion tests to cover both exact forms, multi-item
drafts, named selectors, body-bearing sub-bullets, authored children/metadata conflicts,
repeated or misplaced `!`, UTF-8 offsets, and task completion immediately before the
suffix. Pin the new suffix span and prove the plain marker has the normal task-toggle
route/block-ID spans with no modifier span.

### 2. Preserve behavior-level execution and swap all user-visible CLI expectations

In `src/native/capture.rs`, continue dispatching solely on `TaskToggleIntent`; do not
duplicate or merge the mature two-way and ensure-Next planners. With the grammar mapping
reversed, verify that:

- Plain `@route+block-id` enters `plan_ensure_next_capture`, accepts all four open task
  states, requires an existing unique movable open-Pomodoro Task Link, relocates from
  the batch planner's staged daily-note snapshot, and reports status-only, move-only,
  combined, and total-no-op outcomes exactly as today.
- Suffixed `@route+block-id!` enters the two-way branch, including Ready/Blocked-to-Next
  insertion, Next-to-Ready cleanup, future-schedule retirement, dependency warnings,
  dry-run behavior, named-selector exclusion, and atomic batch rollback.
- Marker-only `@route+block-id#pomodoro` still enters the two-way branch and retains
  exact/prefix selection, creation, and inert-selector-on-clear behavior.

Update the no-movable-link error for the new default ensure operation to direct users to
`@route+block-id!` when they intentionally want the two-way operation to add a missing
Task Link. Audit any invariant and usage text that currently equates `!` with
force-Next.

Keep JSON schema version 1 and the existing behavior-based fields. A plain ensure-Next
result carries `toggle_behavior: "ensure_next"`, `status_changed`,
`pomodoro_link_action`, and source/destination endpoints. A suffixed two-way result
keeps `toggle_behavior` absent and uses the legacy direction/link fields. Do not invert
the field's meaning or make clients infer behavior from the suffix. Human output should
therefore continue to say `ensured`/`would ensure` for the plain default and
`toggled`/`would toggle` for the suffixed operation.

Rework CLI integration coverage as a spelling/behavior matrix rather than only renaming
tests. Cover plain ensure-Next for Ready, Blocked, In Progress, and Next; combined,
status-only, move-only, and total-no-op outcomes; missing/duplicate/completed-only
links; same-note edits; staged multi-item moves and rollback; future-schedule handling;
human output; and the exact additive JSON. Cover suffixed two-way behavior in both
directions, including link insertion/removal, In Progress rejection, named-selector
rejection, dry run, warnings, and JSON with no ensure-only fields. Retain a named
unsuffixed toggle regression so the deliberate third syntax cannot silently become
relocation.

### 3. Make Bob Mac Capture follow returned behavior while presenting the new syntax

In the linked `bob-mac-capture` repository, keep `CaptureTogglePresentation` driven by
`toggle_behavior`, `toggle_direction`, `status_changed`, and
`pomodoro_link_action`—never by checking whether the draft contains `!`. This should
allow its existing Ensure Next status/relocation copy, no-op notification, VoiceOver
announcement, and changed-note routing to work for the new plain spelling without
changing the wire model.

Update the editor semantic-span mapping for `task_toggle_explicit_toggle` and rename the
local semantic category/palette comments away from force-Next. For tolerance with older
Bob binaries, continue recognizing legacy `task_toggle_force_next` as a modifier span;
the live preview/result metadata, not its highlight category, determines the action and
notification copy.

Reverse the explicit fake-Bob parse, completion, dry-run, and committed response cases:
plain `@file+goog-exit` should return the existing ensure-Next moved result, while
`@file+goog-exit!` should return a representative two-way result without ensure-only
fields. Keep body-bearing sub-bullet fixture behavior distinct.

Update focused app tests to prove:

- Plain `@file+id` remains valid through leading route completion and the `+`
  task-picker handoff, previews **Ensure Next**, submits the exact unsuffixed draft,
  announces the actual status/relocation outcome, and includes the daily note in
  notification actions only when the link moved.
- Suffixed `@file+id!` highlights the explicit-toggle modifier, previews **Set Next** or
  **Set Open** from Bob's returned direction, submits the suffix unchanged, and uses the
  established two-way notification/link presentation rather than Ensure Next copy.
- Behavior-level presentation tests still decode both old and new Bob response shapes
  tolerantly; missing `toggle_behavior` continues to mean two-way and `"ensure_next"`
  continues to mean idempotent relocation regardless of spelling.

### 4. Document the migration and verify both repositories

Update bob-cli's `README.md`, `docs/capture.md`, and `bob capture --help` examples and
contract text so every table, workflow, example, diagnostic, parse-span list, JSON note,
and compatibility statement consistently describes plain Ensure Next and suffixed
two-way toggle. Explicitly call out the behavior reversal for users upgrading from the
initial `!` implementation, the unchanged named `#pomodoro` form, the existing-link
precondition for the new default, and `!` as the opt-in escape hatch when insertion or
clearing is intended.

Update bob-mac-capture's README/minimum-Bob capability text and interaction examples to
the same contract. Explain that action labels and notifications come from Bob's returned
behavior metadata, so a mismatched older Bob may implement the previous spellings but
the app still reports that older binary's actual plan truthfully.

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

Finally exercise the installed/development app against a disposable vault: use plain
`@file+id` to move a descendant-bearing Task Link into the current timed Pomodoro,
repeat it to confirm a reported no-op, use `@file+id!` to clear that Next task and
remove its open links, then use the suffixed form on a Ready task to confirm link
insertion. Also spot-check a plain In Progress task, a missing-link default failure that
preserves both files and recommends `!`, the unchanged `@file+id#name` toggle, task
completion before `!`, leading-`@` completion, notification copy, and Open Note/Open
Notes targets.

## Done when

- Marker-only `@route+block-id` is the idempotent default: it ensures any eligible open
  task is Next and relocates its sole existing open-Pomodoro Task Link subtree to the
  implicit current/next Pomodoro without creating a link or touching completed history.
- Marker-only `@route+block-id!` explicitly runs the prior two-way toggle, including
  adding a missing link when setting Next and removing open-Pomodoro links when clearing
  Next.
- Marker-only `@route+block-id#pomodoro` and all body-bearing sub-bullet forms retain
  their existing behavior.
- Parse diagnostics, spans, completion ranges, CLI help, human output, JSON, Mac
  preview, status/VoiceOver, notifications, and notification targets all agree with the
  executed behavior; no layer infers the operation from punctuation alone.
- Plain ensure-Next failures are atomic and direct users to the suffixed toggle only
  when that is the appropriate way to create a missing link.
- All Rust/script and macOS app verification commands pass, and disposable-vault checks
  demonstrate the swapped operations and accurate notifications end to end.

## Out of scope

- Changing the underlying ensure-Next relocation algorithm, implicit Pomodoro selection,
  completed-history protection, or two-way toggle planner beyond wiring the spellings.
- Adding `!` to `@@` declarations, named `#pomodoro` selectors, body-bearing sub-bullet
  captures, or any other marker family.
- Making named `@route+block-id#pomodoro` idempotent or adding a named relocation mode.
- Creating a missing Task Link during ensure-Next, moving links out of completed
  Pomodoros, repairing arbitrary duplicate/mixed-text links, or changing
  `task-status-hooks` reconciliation.
- Redesigning the Mac app's general notification categories, completion UI, or batch
  presentation beyond the swapped syntax, accurate modifier highlighting, and contract
  documentation.
