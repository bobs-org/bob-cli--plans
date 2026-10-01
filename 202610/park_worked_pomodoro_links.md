---
tier: tale
title: Park worked task links with =x* in Bob capture
goal:
  Support composable =x* selections that record normal work without carrying selected
  links into the next Pomodoro, with accurate CLI and Mac previews.
size: medium
proposed_by: bbugyi200.athena.0un.w0
create_time: 2026-10-01 00:35:08
status: wip
---

# Park worked tasks when closing a Pomodoro

## Goal and scope

Add `=x*<N>` to `bob capture` and Bob Mac Capture. It performs the same work-recording
effects as selecting `<N>` in an ordinary close, but does not copy those links into the
next Pomodoro. Bryan can launch an agent swarm, record the work, and let the task wait
in PENDING until it is ready to verify.

Deliver this as one coordinated implementation across bob-cli and its linked
`bob-mac-capture` repository. A single coding agent can implement the bounded grammar,
planner, JSON, and presentation changes; this is a medium tale, not an epic. Implement
the Rust contract before consuming it in Swift. This planning turn changes only this
scratch plan; implementation begins after plan approval.

Use the UI verb **park**: "record work, leave it out of the next session." Parking is a
close outcome, not a new task status. Keep `[/]` and its In Progress status name;
existing dashboard PENDING terminology and sticky-lane behavior already provide the
waiting place. No task-status-hooks, dashboard, plugin, timer, or vault-format redesign
is required. Do not edit memory, install software, deploy the app, or touch the live
vault as part of this implementation.

## Existing architecture and required context

Read the applicable instructions in both repositories. Open the Mac repository with the
`sase_repo` skill and `sase repo open bob-mac-capture` using an audit reason; use only
the returned checkout. Paths below are relative to the repository named.

Before implementation, use `sase_memory_read` to read `decisions:task-lanes-are-sticky`,
`decisions:today-is-read-from-the-ledger`, `decisions:mac-capture-is-a-thin-client`,
`glossary:Pomodoro`, `glossary:Task_Link`, `glossary:Work_Log`, and
`glossary:freshness`. Relevant invariants:

- Rust owns grammar, completion, preview, and vault mutation. Swift renders Bob's
  semantic spans and resolved results and submits a draft as one aggregate capture.
- Closing work normally promotes eligible tasks to `[/]`; removing a link never releases
  a lane. Preserve normal status eligibility, warnings, freshness stamping, and Work Log
  behavior rather than unconditionally assigning a status.
- Today is derived from dedicated links under today's open Pomodoros. Closed history
  does not count. Parking affects this close's carry, not links in other sessions;
  another existing live link can legitimately keep the task in Today.
- JSON schema version stays 1 and grows additively. New Swift properties use
  `decodeIfPresent` and defaults; unknown values retain safe presentation fallbacks.

The existing implementation provides these integration points:

- `src/native/capture_language/close_selection.rs` owns the shared execution/editor
  lexer, duplicate and overlap diagnostics, incomplete lists, and byte ranges.
  `model.rs`, `item.rs`, `tokens.rs`, `editor_model.rs`, `editor_classify.rs`,
  `editor_parse.rs`, `editor_pomodoro.rs`, and `draft.rs` carry the spec through
  whole-item, link/new-task, and same-line-chain forms.
- `src/native/capture_pomodoro_close/selection.rs` numbers links, applies outcomes,
  validates duplicate targets and Work Log entries, and tracks line shifts caused by
  inserting log bullets. `ledger.rs` writes closed history and builds the carry list
  before choosing whether to create a placeholder. `linked_tasks.rs` applies task
  effects and builds task rows. In particular, `apply_startable` currently registers
  `carried: true`; that assumption must become actual carry metadata.
- `src/native/capture/pomodoro_close.rs` adapts specs into the pure planner and emits
  summaries; `output.rs` owns JSON types and human output. `pomodoro_blocks.rs` and the
  plan-budget calculation derive previews from the staged result.
- In bob-mac-capture, `Sources/CaptureCore/CaptureModels.swift`,
  `CapturePomodoroClosePresentation.swift`, and `CompletionRowContent.swift` decode and
  present results. `Sources/BobMacCapture/CaptureEditorPalette.swift`,
  `CapturePanelView.swift`, and `CapturePanelModel.swift` render outcomes and keep a
  dimmed preview while a list is incomplete. `closePendingTrim` currently accepts comma,
  bang, and tilde placeholders but not star.

## User-visible contract

### Grammar and selection

The grammar is `=x[<N>][*<P>][!<M>][~<K>]`, with case-insensitive `x`. The initial
unprefixed list, when present, always comes first. Each prefixed group may appear once
and `*`, `!`, and `~` groups may follow in any order. Lists contain comma-separated,
positive, 1-based task numbers and no spaces. Preserve raw spelling and sort the parsed
lists as the existing lexer does. `*` starts a list; it is neither a postfix modifier of
the preceding number nor a wildcard.

Keep the existing lineup definition: eligible dedicated links are numbered in ledger
order at any depth; aliases and leading tomato markers do not change their identity, and
struck links, mixed-text mentions, notes, and fenced lines are not numbered.

| Selection                                                 | Close effect                           | History                       | Carried forward |
| --------------------------------------------------------- | -------------------------------------- | ----------------------------- | --------------- |
| `<N>`                                                     | Normal In Progress work                | Normal worked link and notes  | Yes             |
| `*<P>`                                                    | Same normal work effects; parked       | Same worked link and notes    | No              |
| `!<M>`                                                    | Existing completion behavior           | Existing completed history    | No              |
| `~<K>`                                                    | Existing drop behavior; lane unchanged | Existing dropped-link removal | No              |
| Unlisted plain/deferred link with `<N>` or `*<P>` present | Deferred                               | Existing deferred behavior    | Yes             |
| Unlisted embedded link                                    | Existing completion behavior           | Existing completed history    | No              |

Selection mode is active if the initial `<N>` is present **or a nonempty `*<P>` group is
present**. This is essential: `=x*2` has the same unlisted-link outcomes as `=x2`. When
neither is present, unlisted links keep their ledger outcomes as before; for example,
`=x!2~3` does not implicitly defer other worked links.

The literal `0` remains valid only as the entire initial `<N>` list. `=x0*2` is valid:
there are no ordinary continuing selections, task 2 is parked, and other eligible
unlisted links are deferred. `*0`, `!0`, and `~0` are invalid. Preserve existing
zero/leading-zero rules rather than introducing another number normalization rule.

Examples that must be supported:

- `=x*2,3`: work on and park 2 and 3; defer other plain links.
- `=x1*2,3!4,5`: continue 1, park 2 and 3, complete 4 and 5.
- `=x1!4,5*2,3`: exactly the same result as the previous example.
- `=x1*2!3~4`: continue 1, park 2, complete 3, drop 4; defer any remaining plain links.
- `=x0*2`, `=X*2`, and all permutations of the prefixed groups.
- `@route:id=x*2`, `^route:id=x*2`, and `New task @route:id=x*2` use the existing
  post-link/post-create lineup. Do not let marker parsing interpret the star as task
  text or a block-ID character.
- `+2 =x*2`, `=x*2 =`, `=x*2 =#bugs`, and `=x*2 =~1` retain left-to-right atomic chain
  behavior. The last example's `1` refers to the next session's own lineup.
- Child Work Log bullets and nested details work for parked links exactly as for
  ordinary worked links: `=x*2` followed by `- 2 Launched the swarm` is valid.

Do not add star syntax to session starts, normal task markers, or ledger Markdown.
Continue treating unrelated prose shapes as prose under the existing recognition rules.

### The five-link acceptance example

Start with a running Pomodoro containing five dedicated plain links to eligible Next
tasks 1 through 5. Close with `=x1*2,3!4,5`:

- The closed session retains the normal `🍅` work history for 1, 2, and 3, including
  their child notes. Existing completion handling retires links 4 and 5 as usual.
- Tasks 1, 2, and 3 receive exactly the normal In Progress and Work Log effects; tasks 4
  and 5 receive exactly the normal completion effects.
- The newly created placeholder contains only link 1. The JSON carry list, task-row
  `carried` flags, next-session count, rendered blocks, and plan budget all agree.
- The app summary reads `Continue 1 · Parked 2, 3 · Complete 4, 5`; parked rows visibly
  say `Parked · not carried` and preserve the actual status transition.

Do not erase links 2 and 3 from closed history or unlink copies in other open Pomodoros.
The request is to suppress copying these selected lines into the new session. Use the
existing placeholder rule after filtering carry: when nothing is carried and a later
entry exists, use that entry without creating an extra one; when no later entry exists,
create the usual empty placeholder with its stub.

### Errors and editing states

Reject a repeated star group, duplicate numbers, a number appearing in two outcome
groups, zero in `*`, overflow, malformed lists, empty interior groups, and out-of-range
selections. No last-wins precedence. Diagnostics must identify the offending number or
separator and use the existing `invalid_pomodoro_close` family and shared byte-range
conventions. Examples: `=x*1*2`, `=x*1,1`, `=x1*1`, `=x*1!1`, `=x*1~1`, `=x*0`, `=x*!2`,
`=x*1,,2`, and `=x*6` against a five-link lineup.

`=x*`, `=x1*`, `=x*2,`, and a valid selection followed by `*` are editor-incomplete:
retain the parsed prefix, report `pomodoro_close_task`, and put an
`interactive_placeholder` span on the dangling separator. Execution rejects them without
writes. A prior lexical error still takes precedence over incompleteness. Use the same
behavior for link forms, chains, and drafts with Unicode before the operator. Never turn
an unfinished or malformed new close into a captured task.

Two numbered occurrences of the same target must agree on the entire outcome, including
carry. Thus an ordinary worked selection and a parked selection for the same task
conflict; two parked occurrences are allowed. Retain the existing duplicate guard and
ensure resolved aliases cannot merge incompatible parked/continuing rows silently.
Extend resolved-identity checks where parking is involved without expanding the
semantics of unrelated existing selections. Validate before any staged effects can be
committed. A later failed item in an aggregate draft rolls back the whole draft using
the existing transaction path.

## Implementation

### 1. Extend the shared grammar and protocol

Add `park: Vec<u32>` to the Rust close spec and the parse/capture JSON close objects,
omitted when empty, and a corresponding set in the planner selection. Preserve the typed
meaning of `in_progress`: it stays null if no initial list was typed, even for `=x*2`;
do not secretly union parked numbers into it. Update `plain`, `has_selection`, spec
conversions, incomplete specs, and all struct initializers.

Extend the lexer with a third labeled group rather than adding another copy of the
grammar. A small shared scan of prefixed segments can replace the current two-marker
branch table, while retaining diagnostics for all existing inputs. Enforce unique groups
and disjoint numbers, preserve absolute byte offsets, and propagate `park` and its range
through every whole-item and suffix editor path. Add `pomodoro_close_park` spanning `*`
and its list, analogous to complete/drop spans. Use existing completion/rewrite
infrastructure; verify the new close is classified there correctly without inventing a
task-number picker.

Keep schema version 1. Add `task_links[].outcome: "parked"` with `source: "listed"` for
star-selected lines. Existing `marker` remains the original ledger marker;
`tasks[].role` remains `worked` for resolved parked worked rows (or the existing
unresolved role), because parking does not change task resolution or status semantics.
The existing task `carried` boolean and top-level `carried` list report actual carry.
Older output without `park` remains unchanged, including omitted empty fields.

### 2. Separate work recording from carry in the pure close planner

Add an explicit `Parked` selection outcome. For marker rewriting, parked behaves like In
Progress: normalize a selected deferred/embedded numbered line to a plain link, then use
normal close handling to create history, start eligible tasks, stamp freshness when the
current start path does, and collect Work Logs. Allow parked links wherever typed logs
allow worked links, while retaining nested-link and lineup-change guards. Preserve
existing blocked/done/unresolved task warnings; parking cannot force those targets to
become In Progress or claim they did.

Carry suppression must be explicit planner metadata, not a new durable Markdown marker
and not a string-deletion pass over an already-built placeholder. Derive a set of parked
source lines from the selected lineup after typed Work Log insertion has recomputed its
line positions. Pass that information into ledger planning (an internal options-aware
helper can retain `plan_ledger_close`'s current empty-options behavior). Skip exactly
these worked lines while collecting carry and set their classified-link carry flags
false, before the next-placeholder decision. Keep them in startable targets and history;
preserve the relative order of all remaining carried lines.

Thread actual carry into `apply_startable` and any subsequent reference registration or
Work Log row merging. The existing OR-merge of task-row carry flags must not turn a
parked task back into carried merely because it was started. Other independently carried
references retain their existing effects; suppress only the selected lines. Apply the
existing duplicate-target validation to the new outcome and, when distinct path
spellings resolve to the same target, check compatible outcomes before effects.

Update summary construction and human output. Parked task rows retain their real
transition and log counts, gain a concise `parked · not carried` caption, and produce a
`Parked 2, 3` selection summary. Warnings remain visible on unresolved or ineligible
tasks. Keep old output stable for captures without star selections. Ensure preview
blocks and budget meters come from the final staged ledger with no client-side fixes.

### 3. Present parking clearly in Bob Mac Capture

Decode `park` in both `PomodoroCloseSpec` and `PomodoroCloseSummary`, defaulting to an
empty array, and add a presentation outcome for `parked`. Update initializers and
encoding keys as well as decoders. Teach span-category mapping and the palette about
`pomodoro_close_park`; use system teal consistently for the star group, numbered badge,
and parking accent, with semantic text so color is never the only signal.

Preserve the existing compact close-card layout and numbered-row alignment. A parked row
keeps normal readable, unstruck task text, its actual transition, and Work Log previews.
Add a small `pause.circle` accent beside `Parked · not carried`; do not replace the task
transition with a fictional status or reuse the dropped-row strike/dimming. VoiceOver
should say the task number, text, actual status result, and "parked, not carried to the
next Pomodoro." Keep warnings separately visible and announced.

For a star-bearing selection, use `Continue`, `Parked`, `Complete`, `Deferred`, and
`Dropped` groups as applicable, in that order with sorted numbers. `Continue` means
ordinary worked links carried onward; it does not name a task status. Preserve current
summary wording for old syntax. `=x0*2` may say `Continue none · Parked 2` but must not
say `In progress none` while task 2 is being worked. Do not imply a parked task was
removed from all of Today; only the selected session's carry is affected.

Add a concise teaching example, `=x*2 parks 2 (not carried)`, to the existing hint
system, using only numbers available in the current lineup (use 1 for a one-link
session). Avoid a new panel or modal. Allow the hint/summary to wrap at narrow widths;
the task text and outcome take priority over locators.

Extend `closePendingTrim`'s existing placeholder handling to recognize the star that Bob
has already marked incomplete. Preserve the dimmed prefix preview, say
`Type a task number after *`, and disable Close until the draft is complete. The trimmed
draft is for preview only; capture must submit the original complete draft in one call.
Do not parse selection lists or infer outcomes in Swift.

New Mac builds must still decode old fixtures, absent `park`, and unknown outcome/span
values. New syntax requires an updated Bob binary: document CLI-first rollout, because
an older Bob may classify an unknown syntax as ordinary task text. Do not claim old Bob
supports or reliably rejects star syntax, and do not add a second grammar in Swift to
compensate. Keep schema-version rejection and generic error handling unchanged.

### 4. Document and verify the complete behavior

Update bob-cli `docs/capture.md` (syntax tables, numbered selection/outcome rules,
five-link example, Work Log eligibility, JSON additions, error/incomplete examples, and
single-quoted shell invocations) and relevant `src/native/capture_parse.rs` help. The
old statement that every selection is only hand-edited markers plus an unchanged close
must explicitly account for star's carry metadata. Update the linked Mac `README.md`
with parking, preview behavior, and the minimum Bob capability requirement. No new CLI
subcommand or flag is needed.

Add focused behavioral coverage to existing suites, using temporary fixture vaults and a
fixed `BOB_NOW`:

1. **Grammar/editor parity:** star-only and mixed lists, all six prefixed-group orders,
   uppercase X, `=x0*2`, sorted lists/raw preservation, every invalid category above,
   incomplete states and retained prefix fields, absolute byte spans with Unicode,
   whole-item/link/new-task forms, and chains. Keep existing close/start/prose tests.
   Relevant suites are `capture_language/close_selection.rs` tests,
   `capture_language/tests/{grammar,editor_modes,chain,completion,rewrite}.rs`, and
   `tests/cli/capture/parse_pomodoro_close.rs`.
2. **Real effects:** the exact five-link example and a four-group example, comparing
   daily and task-note bytes, work history, freshness, completion, ordered carry,
   placeholder creation, and actual status warnings. Include all-parked with and without
   a later placeholder; star-only selection deferring unlisted plain links; explicit
   star overrides of deferred and embedded markers; named/unnamed sessions; duplicate
   targets (same outcome and conflicts, including aliases); unresolved, Blocked, and
   already-done targets; tabs/spaces, aliases, CRLF, and no final newline. Build on
   `capture_pomodoro_close/{selection_tests,linked_task_tests,tests}.rs` and
   `tests/cli/capture/pomodoro_close_selection.rs` rather than duplicating all fixtures.
3. **Logs and transactions:** existing descendants plus multiple typed entries and
   nested details on parked links; entries for an earlier task shifting a later parked
   line; nested-link log rejection and log text that would change numbering; actual
   dry-run/write parity; invalid later aggregate item leaving every file unchanged.
   Exercise new and existing task link-close forms and `=x*2 =` / named-switch chains.
   Assert parked task rows remain `carried: false` after status changes and log merges
   when no other independently carried reference to that target exists.
4. **Derived results:** `pomodoro_blocks`, next-session counts and plan-budget links
   agree with the new ledger. On an area/project task with no other live link, verify it
   leaves Today while retaining the normal sticky Pending lane through hook
   reconciliation. Include a remaining live link elsewhere to confirm parking does not
   globally remove Today membership. Preserve the canonical-daily exception.
5. **Cross-repository presentation:** produce Mac JSON fixtures from the implemented
   Rust parse and dry-run endpoints, including the five-link example, star-only, Work
   Logs, all-parked/empty carry, incomplete, and invalid cases. Keep fixture generation
   inputs/time documented. Extend `CapturePomodoroClosePresentationTests`,
   model/span-category tests, and `CapturePanelModelTests` to assert optional decoding,
   real status text, unstruck/undimmed parked rows, captions, sorted summaries,
   VoiceOver, disabled submission while incomplete, and original-draft submission.

Run focused Rust tests during development, then `just all` in bob-cli once the change is
complete (`cargo fmt --check`, Clippy for all targets/features, and `cargo test`). Do
not write to the real vault for validation.

Run the Mac repository's `just all` on macOS 26 with its selected Apple toolchain; it
covers formatting, build, tests, and an ad-hoc bundle. Use the existing macOS CI when
the coding host lacks that toolchain, following the project's authorized workflow and
SASE monitor for long waits. Never report Linux-only inspection as passing Swift
validation or claim an unobserved CI result.

Extend the existing `PomodoroBlockDesignTests` rendering harness with mixed parking and
incomplete-parking previews. With `BOB_MAC_CAPTURE_RENDER_DIR` set, run
`./Scripts/xcode-swift.sh test --filter PomodoroBlockDesignTests` on macOS and inspect
the generated light/dark PNGs at both 620pt and 760pt widths. Verify task text,
transition, parking caption, Work Log nesting, wrapping, and the matching closed/next
blocks; make spacing/contrast corrections before considering the UI validated. Keep
generated images out of source control unless using an established artifact workflow. If
macOS execution or visual review is unavailable, state that explicitly as remaining
validation rather than silently waiving it.

## Completion criteria

The five-link example works in the CLI and app with precisely one carried link; parked
work preserves normal status/history/log behavior; every existing `=x` form still
composes correctly; malformed or incomplete input cannot partially mutate a draft; and
previews truthfully distinguish continuing, parked, completed, deferred, and dropped
work. Rust checks, macOS checks, fixture compatibility, and visual review have concrete
results recorded. The approved implementation changes are confined to bob-cli and
bob-mac-capture code, tests, fixtures, and documentation.
