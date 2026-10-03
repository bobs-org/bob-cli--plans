---
tier: epic
title: Fuzzy task pickers for scoped and vault-wide plus capture
goal:
  Scoped @file+ and leading or prose-terminal + gestures open the shared native fuzzy
  task picker, insert the correct @file+id marker, and preserve existing capture
  semantics, Pomodoro operators, and stale-safe task identity.
phases:
  - id: plus_completion_contract
    title: Define plus task discovery and cursor contract in bob-cli
    size: medium
    depends_on: []
    description:
      "plus_completion_contract: add the shared selector classifier, additive completion
      metadata, vault-wide discovery adapter, backend-authored ID assignment
      replacement, and Rust contract/regression tests."
  - id: plus_picker_mac
    title: Present scoped and vault-wide plus task pickers in Bob Mac Capture
    size: medium
    depends_on:
      - plus_completion_contract
    description:
      "plus_picker_mac: connect both plus scopes to the shared picker lifecycle and
      fuzzy presentation, including keys, focus, snapshot guards, ID naming,
      compatibility fallback, accessible visuals, and macOS tests."
  - id: plus_picker_integration
    title: Verify the combined feature and polish the picker on macOS
    size: small
    depends_on:
      - plus_completion_contract
      - plus_picker_mac
    description:
      "plus_picker_integration: exercise backend and app together in a fixture vault,
      verify operator and compatibility behavior, inspect native visuals and
      accessibility on macOS, and correct feature-specific integration issues."
proposed_by: bbugyi200.athena.0vw
create_time: 2026-10-03 16:24:16
status: wip
---

- **PROMPT:**
  [prompts/202610/plus_task_picker.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202610/plus_task_picker.md)

# Fuzzy task selection for capture's plus syntax

## Outcome and scope

Typing `@file+` opens a fuzzy task picker scoped to `file.md`. Choosing the task with
block ID `^id` produces `@file+id`. Typing `+` at the start of an otherwise empty
capture item, or after a space at the end of a capture line, opens the same picker
across notes. Choosing a task produces Bob's complete `@file+id` marker and leaves the
surrounding draft intact. Selection only prepares the draft; the existing aggregate
`bob capture` transaction performs the requested action.

Use the existing native picker card and interaction model from `:` and `^`. Provide the
full eligible snapshot once, then filter locally so typing is instant and the panel does
not flicker or change height. Bob owns trigger recognition, candidate discovery,
eligibility, replacement ranges, and marker formatting. Swift owns presentation, local
fuzzy ranking, focus, and keyboard navigation.

For this feature, vault-wide means the same eligible-task scope as the existing `:` Task
Link Picker: all open tasks in the routable inbox, area, and non-terminal project notes,
across every such note rather than only the current destination. The existing scanner
accepts Ready, Blocked, Next, and In Progress tasks and includes tasks without block
IDs. It excludes done/canceled tasks, terminal projects, and notes outside that
capture-target catalog. Show the scope honestly as **All capture notes** in the
detail/help text. Do not extend route grammar to arbitrary nested or quoted note paths,
add support for closed-task capture, or silently relocate tasks as part of this
completion feature.

The scoped `@file+` picker retains its current `capture-tasks` eligibility: all open
tasks in that addressed note, including configured open statuses and ID-less tasks with
`--all-tasks`. Do not narrow it to the colon link-status predicate. Searching never
writes to the real vault. An explicitly accepted ID-less row uses the existing
stale-safe Add block ID flow.

## Why this is an epic

The backend owns a new bare-plus completion context and the ambiguity with Pomodoro
operators. The Mac frontend then adopts that contract across its picker, key routing, ID
prompt, focus, and async snapshot paths. A final phase verifies the real combination on
macOS and corrects interaction/visual issues. These are separate bounded deliverables
with an explicit dependency chain; implementation starts only after this plan is
approved. Planning is xlarge work; each coding phase is sized for direct implementation
from this plan.

## Grounded implementation context

- bob-cli is Rust. `src/native/capture_language/completion.rs` determines cursor fields;
  `CompletionContext::Task` already represents the right side of `@route+id`. Its
  replacement range covers the ID alone and stops before `#`.
  `src/native/capture_complete/candidates.rs::task_candidates` already scans and
  searches task ID, text, section, and status; `--all-tasks` includes ID-less rows.
- `src/native/capture_complete/engine.rs`, `model.rs`, `render.rs`, and `shell.rs`
  produce JSON, human output, and safe shell candidates. Schema version 1 grows through
  optional fields. Shell completion must stay read-only, omit rows requiring ID
  assignment, and retain its existing cursor/deadline gates.
- The vault-wide colon scanner/ranker is `src/native/capture_link_tasks.rs`; its
  completion adapter is `task_link_candidates`. It already provides stable task refs,
  block-ID suggestions, note groups, and Pomodoro metadata. Its `replacement()`
  currently hardcodes `@route:id`; reuse discovery and identity without copying that
  colon insertion policy into plus semantics.
- `src/native/capture_language/tokens.rs::task_link_query_token` only claims a solo
  colon item. Plus needs a different terminal-token predicate, including a prose line
  ending in a selector. Shared protected-span/token infrastructure is preferable to a
  second regex grammar.
- Bare `+` is already a valid five-minute Pomodoro adjustment; `+N` resizes, `++N`
  shifts, and operator chains such as `+2 =x` split into action items.
  `completion_field_at` currently suppresses these before task completion. Later lines
  are classified as authored bullets, and `+` can be their bullet marker. Work Log text
  suppresses marker completion. Respect all of these.
- Open the linked Mac repository with `sase repo open bob-mac-capture -r <reason>` and
  work only in the returned checkout. Do not copy any numbered workspace path into
  implementation instructions. There was no repository `AGENTS.md` at inspection; check
  applicable instructions when opening it during implementation.
- In that repository, `CapturePanelModel.handleCompletionResponse` routes `task_link`,
  `active_task`, dependency, and block-ID contexts into the modal picker card, but
  leaves `task` on the inline list. `CapturePickerState`, `CapturePickerIndex`,
  `CapturePickerSource`, and `CapturePickerView` already share the lifecycle and
  rendering. `TaskLinkPickerIndex` has local weighted fuzzy matching, note/Pomodoro
  groups, stable `route|ref` identities, and pending block-ID rows. `BlockIDPickerIndex`
  provides scoped task row presentation.
- Existing task-link snapshots refetch at `replacement.start` to recover a full list
  when the user types before the popup opens; async guards compare both generation and
  draft. Existing sources also support a reopen chip, marker highlighting, Escape
  suppression, and fixed visible-row budgets.
- `BobProcessClient.captureComplete` already passes `--all-tasks`. Existing
  `CaptureTaskIDPromptPurpose.parentTask` and `taskLink` flows share ID assignment but
  differ in return state and formatting; preserve cancel/back navigation.
- Mac checks use `./Scripts/xcode-swift.sh build` and `test` plus formatting;
  `.github/workflows/ci.yml` uses macOS 26. Linux source inspection alone cannot
  establish that the AppKit/SwiftUI interaction works.

## Product interaction contract

| Situation                                  | Picker scope                    | Accepted draft                       |
| ------------------------------------------ | ------------------------------- | ------------------------------------ |
| `@cash+`                                   | Open tasks in cash.md           | `@cash+goog-exit`                    |
| `Called the bank @cash+`                   | Open tasks in cash.md           | `Called the bank @cash+goog-exit`    |
| `+` in a fresh item                        | Open tasks across capture notes | `@cash+goog-exit`                    |
| `Called the bank +`                        | Open tasks across capture notes | `Called the bank @cash+goog-exit`    |
| `@cash+go#requirements`, caret inside `go` | Open tasks in cash.md           | `@cash+goog-exit#requirements`       |
| Exact identified `@cash+goog-exit`         | No auto-open                    | Draft unchanged; Tab/chip can browse |
| `+2`, `++`, `++3`, or `+2 =x`              | No plus task auto-open          | Existing Pomodoro operator behavior  |

### Trigger and ambiguity rules

1. `@route+` and edits to its non-exact ID component open the scoped picker. Cursor-only
   movement into the component shows a compact reopen chip instead of stealing focus.
   Editing the route remains route completion; typing `+` while route completion is open
   retains the existing selected-route acceptance behavior, then transitions into the
   scoped picker.
2. A bare-plus selector is a terminal whitespace-delimited token, beginning at the start
   of a capture item's parent line or following a separating space at that line's end.
   The initial gesture is a literal `+`; text typed immediately afterward before the
   response arrives becomes the seed query (`+bank`). Once open, filter spaces are AND
   terms and never enter the draft.
3. Also support a terminal plus selector in the body of an eligible later authored
   bullet, applying only that physical line's byte range. An authored `+` bullet marker
   itself is never a trigger; `- note +` can be. In a later blank-line-separated item, a
   leading `+` works like it does in the first. A blank editor line at EOF must not lose
   the trigger through draft splitting.
4. Never trigger in a protected code span/fence, escaped `\+`, wikilink/alias, URL, Work
   Log entry, literal mid-prose plus, `C++`, `a+b`, a project-note `@route^id+` sigil, a
   Pomodoro name containing `+`, or a global `@@` declaration. Preserve scoped task
   completion for an existing `@@route+...` declaration where Bob already returns
   context `task`.
5. A lone whole-item `+` is intentionally dual-use: `capture-complete` offers task
   candidates, while `capture-parse` and `bob capture '+'` retain their existing valid
   adjustment semantics. The picker footer teaches **Esc to extend Pomodoro +5m** for
   that exact item. Escape leaves the draft unchanged; submitting it then extends as
   before. Do not mark it incomplete, suppress its valid dry-run, or change operator
   execution.
6. Complete numeric/double-plus action tokens and their invalid numeric near misses keep
   action parsing and request no task picker. Preserve the next keystroke regardless of
   whether the popup response has arrived: for the exact whole-item `+` with an empty
   filter, the first unmodified digit or second `+` closes the picker and continues the
   operator in the editor. Bob supplies those action-continuation hints in the
   descriptor; Swift only routes the key and does not classify capture syntax. Suppress
   reopening for that continuation. This prevents `+2`/`++3` behaving differently with
   fast and slow typing. Once task-search text exists, digits are ordinary filter text;
   paste is normal filter input, including a purely numeric search. Scoped and
   prose-terminal pickers never use this operator handoff. A prose-terminal `+` is a
   selector gesture, never an adjustment.
7. A prose-terminal `+` should show **Choose a task to append to** while the selector is
   unresolved, and must not preview or submit it as an ordinary inbox task by accident.
   Bob supplies an additive editor completion need/span for this new non-operator
   selector shape, with a useful selection-required execution error if it reaches
   `bob capture`. Share its lexical predicate across editor, execution, and completion;
   do not reinterpret protected or mid-prose text. A whole-item `+` remains the
   exception in rule 5.

### Appearance, feedback, and keys

- Use one existing panel card, its established corner radius, materials, spacing,
  palette, and task status glyphs. Add a scope capsule `@cash+ · cash.md` or
  `+ · All capture notes`. Use an **Append / Select Task** identity and neutral accent
  consistent with existing task/block-ID highlighting; do not borrow the colon-only Link
  & Start action or schedule pull-forward warning.
- Task text is the primary row content. Show a quieter `route+id` locator and useful
  note/section/Pomodoro context. ID-less rows have an Add ID affordance, never an empty
  selectable insertion. Accent fuzzy matches with existing semibold highlights, keep
  status understandable without relying on color, and truncate secondary text before the
  task title.
- Scoped empty-query rows keep Bob's document order. Vault-wide empty-query rows retain
  familiar colon groups (queued Pomodoros, In Progress, Next, notes). Nonempty queries
  show one ranked list with spaces as AND terms across task text, ID, note, section, and
  status; preserve Bob's stable order for ties.
- The detail strip always says **Inserts @cash+goog-exit**. Once selected, the existing
  live preview communicates the actual outcome: a body-bearing marker appends a note; a
  solo marker ensures Next and its Pomodoro placement. Selection must never imply that
  it already changed status or captured content.
- For the dual-use lone `+`, show a quiet **Type a number or + to adjust; Esc to extend
  +5m** hint. Continuing an operator restores the native editor caret and applies the
  triggering key once, without modifying another item.
- Return, Tab, or click accepts and returns focus to the editor at Bob's range.
  Command-Return accepts then captures. Shift-Return/Option-Return are consumed as they
  are for non-link sources; they never append `=` to a plus marker. Reuse
  wrap/page/Home/End/Ctrl-N/P navigation and native filter editing.
- Escape first clears a nonempty filter, then cancels with draft/caret restored,
  suppresses auto-open for that token, and shows the reopen chip. Tab/Down/Ctrl-N on the
  chip reopen. Backspace on an empty filter deletes bare `+query`, or removes scoped
  `+query` while leaving `@route` available for route completion. Bob supplies the exact
  deletion range; Swift must not hunt for the marker.
- Keep the card height fixed during filtering, show loading without a false empty state,
  distinguish **No tasks in cash.md** from **No matches**, and surface bounded discovery
  warnings without losing the draft. Match existing VoiceOver announcements, accessible
  row labels, light/dark themes, contrast, and Reduce Motion. Do not expose JSON or
  subprocess details in the UI.

## Phase plus_completion_contract

**Dependencies:** none. **Size:** medium. **Repository:** bob-cli.

Deliver the complete server contract, documented and tested, before the app changes.
Reuse existing scanners and canonical task identity.

1. Add one shared lexical classifier for the bare-plus selector and wire it into
   completion before the lone-plus action early return, preserving every other operator
   exclusion. Apply the same predicate to the unresolved prose-terminal selector
   editor/execution behavior described above. Add a new completion context `task_parent`
   for vault-wide plus; retain `task` and its ID-only range and candidate replacement
   for scoped `@route+` / `@@route+` callers.
2. Grow completion schema 1 additively with a `picker` descriptor on these plus
   responses. It carries `kind: parent_task`, `scope: note | vault`, the
   backend-authored scope token, the exact note target for scoped responses, a full
   semantic marker range for highlighting, and an exact trigger-removal range. For a
   dual-use lone `+` only, include optional action-continuation key hints (ASCII digits
   and `+`) for native editor handoff. Also serve optional `query` for both plus
   contexts, empty on refetch at `replacement.start`; other contexts retain their prior
   output. Document exact names, null/omission rules, and UTF-8 byte semantics in
   `docs/capture.md`. The range for bare plus covers the whole `+query` token, including
   its sigil; the range for scoped task completion still covers only the ID, excluding
   `@route+`, trailing `!`, `#section`, and neighboring text.
3. Adapt the existing vault-wide discovery into `task_parent` candidates with
   server-formatted `@route+id` replacements. Share candidate data/ref/suggestions and
   ranker where suitable; represent ID-less rows with `requires_block_id` and no
   insertable replacement. Suppress colon-specific `pulls_forward` messaging because
   appending a note is not linking. Scoped candidates retain their old wire shape and
   `--all-tasks` behavior; add suggestions only as an optional field if the ID prompt
   needs them.
4. Make the Add block ID response supply an optional backend-formatted
   `parent_replacement` (`@route+id`) alongside its existing fields, so the Mac app can
   use Bob's marker verbatim for a vault-wide accepted ID-less row. The scoped picker
   inserts the validated returned ID into its ID-only range. Use the existing
   `capture-task-id --route --task-ref` operation and stale-safe recovery; add no new
   public CLI option or command. Prompt cancellation never writes; an explicit
   successful ID assignment can remain after later draft cancellation, matching existing
   picker behavior.
5. Update human completion rendering and shell extraction for `task_parent`. Shell Tab
   can list identified `@route+id` rows for eligible bare-plus shapes; it never names
   ID-less tasks. Direct execution of `+`, `+N`, and `++N` remains unchanged. Keep
   discovery bounded warnings, no sensitive draft logging, stable order, and existing
   shell performance/read-only guarantees.
6. Update `docs/capture.md` and `docs/completion.md` with scoped/bare examples, the
   exact search scope, action ambiguity, schema examples, and safe shell behavior.
   Verify enum matches, serializers, and renderer branches throughout the repository
   rather than only adding one parser branch.

**Validation and exit criteria:** Unit tests for lexical boundaries and UTF-8 ranges;
CLI fixtures for candidate discovery, JSON/human/shell output, ID-less assignment and
round-trip capture; no real-vault writes. At minimum cover bare `+`, typed `+bank`,
prose-terminal `+`, scoped/global scoped `@...+`, empty ID range, later item/child
lines, LF/CRLF, emoji before the trigger, protected spans, literal plus, Work Logs,
numeric/double-plus operators/chains, and valid typed suffix preservation. Verify that
choosing an identified task produces exactly the draft that current `bob capture` parses
as append/Ensure Next. Run focused tests first, then `cargo fmt --check`,
`cargo clippy --all-targets --all-features`, and `cargo test` (or `just all`). Finish
only with a documented contract and passing appropriate Rust checks.

## Phase plus_picker_mac

**Dependencies:** plus_completion_contract. **Size:** medium. **Repository:** linked
bob-mac-capture.

Implement both plus scopes together against the phase-one contract, keeping the existing
picker infrastructure shared.

1. Decode the optional picker descriptor, query, suggestions, and `parent_replacement`
   with `decodeIfPresent`; maintain schema version 1 and supported older-bob behavior.
   Introduce one parent-task picker source with explicit scoped/vault context. Use a
   configurable task-search index or small shared row/ranking helpers from
   TaskLink/BlockID indexes; do not fork a second complete filter/navigation/card
   implementation or reuse colon action labels.
2. Route `task_parent` and scoped `task` responses to the card, including empty lists,
   with server ranges/query as authority. If an older Bob returns `task` without the
   optional descriptor, preserve its working inline parent-task flow; do not guess
   scoped syntax in Swift. Old `:`/`^`/`&` sources keep their interaction/formatting
   contracts.
3. Preserve full-snapshot refetch at `replacement.start`, verify matching
   context/scope/range, seed fast-typed queries, and reject stale generations or changed
   drafts. Refetch failure uses the existing partial-snapshot warning, never a silent
   empty catalog. An exact identified component is recognized from a fresh full snapshot
   before deciding whether to auto-open; if a filtered response already contains the
   exact candidate, suppress immediately. Selection movements offer chips rather than
   opening a modal. For an unresolved body selector, skip the doomed dry-run using Bob's
   new selection need; a solo `+` continues to preview the adjustment and shows the
   Escape hint.
4. Extend source/index switches, marker highlight, reopen suppression, filter
   focus/input lock, fixed row budget, loading/error state, keyboard router, acceptance,
   cancel, chip reopen, and Backspace using descriptor ranges. Preserve route-plus
   acceptance and all suffixes. Ordinary selection inserts only the candidate
   replacement into Bob's range; no `=`/`!` is synthesized. Apply Bob's lone-plus
   action-continuation hints before native filter editing: an unmodified
   digit/second-plus key with an empty filter restores the editor, inserts that key once
   at the restored caret, and suppresses reopening. Leave paste, modifier editing, and
   keys after nonempty search text native.
5. Generalize the ID prompt's parent-task purpose to retain the return picker, scope,
   suggestions, and none/submit follow-up. ID-less rows are keyed by stable note/route
   plus task ref, never the empty replacement. Explicit accept opens **Add block ID**,
   selects a suggested ID, supports suggestion cycling, and previews the final
   `@route+id`. A failed/stale assignment keeps the prompt open with a recoverable
   error; Escape restores the picker/filter/selection. On success scoped accepts use the
   returned ID, vault accepts require Bob's `parent_replacement`. If that field is
   absent, ask the user to update Bob in the existing status style and preserve the
   draft rather than generating it. Recheck draft identity after the async assignment
   before splicing/submitting.
6. Implement the appearance and accessibility contract above in the existing card. Add
   synthetic preview fixtures for scoped/vault, grouped/filtered, ID-less/empty/warning,
   and lone-plus operator hint states. Update the Mac README with examples, keys, scope,
   and minimum backend capability. Installation and deployment are outside this plan; do
   not deploy automatically.

**Validation and exit criteria:** Extend CaptureCore presentation/decoding tests and
AppKit panel/key-router tests using backend-authored fixture responses. Assert the
distinct ID-only/full-token insertions, exact-match suppression, query seeding, full
refetch, changed-draft race, ID assignment success/failure and
changed-draft-after-assignment race, cancel/reopen/delete behavior, route-accept-plus
transition, later-line range preservation, and that Shift-Return cannot start a
plus-selected task. Test operator continuation both before the completion response and
after the picker has focus, including key replay exactly once, numeric paste as filter
text, and no operator handoff for scoped/prose-terminal pickers. Existing
colon/caret/dependency tests must still pass. Run formatting plus
`./Scripts/xcode-swift.sh build` and `./Scripts/xcode-swift.sh test` on macOS 26 or the
existing Mac CI; source inspection is not a substitute for those checks. Where the
worker host is Linux, record that limitation and arrange supported macOS checks using
the project's existing workflow; do not claim AppKit verification locally.

## Phase plus_picker_integration

**Dependencies:** plus_completion_contract, plus_picker_mac. **Size:** small.
**Repositories:** bob-cli and linked bob-mac-capture.

Verify the built app with the corresponding backend and finish the visual quality work.
Correct issues within this feature; do not expand the grammar or redesign unrelated
pickers.

1. Exercise a disposable fixture vault with several notes, status variants, duplicate
   task titles, multiple queued Pomodoros, a long task, Unicode text, an ID-less task, a
   missing/empty note, and a non-capture note. Compare the backend JSON with the app's
   unfiltered full list and selected insertion. Confirm the exact documented catalog
   boundary rather than advertising an all-Markdown-note search that the backend cannot
   insert.
2. Walk `@file+` -> fuzzy search -> accept -> preview -> capture; prose-terminal `+` ->
   search another note -> accept -> append; leading `+` -> accept -> Ensure Next;
   leading `+` -> Escape -> submit -> existing adjustment; and `+N`/`++N` -> existing
   operators with both fast and slow keystrokes. Repeat in a later batch item and an
   eligible authored child line, including ID naming and typed suffixes. Verify capture
   occurs once and failure retains the draft.
3. Inspect both scopes in light/dark and narrow/wide panels, grouped/filtering states,
   no-match/warning/ID prompt states, keyboard-only and VoiceOver use. Check native
   filter editing/undo/paste, no focus theft on caret movement, no reopening after
   accept/cancel, readable match highlights, quiet secondary locator, accurate action
   details, fixed-height filtering, and graceful long text. Use existing preview/demo
   tooling for reviewable visual evidence when available; do not create generated images
   or assets for a native widget.
4. Check compatibility fixtures for new app/older Bob (inline scoped fallback) and new
   Bob/current schema decoder; document minimum versions/capabilities accurately. Run
   remaining relevant cross-repo checks, including the existing macOS CI build/test and
   bundle smoke checks if packaging sources changed. Record what was executed, results,
   any environment limitations, and concrete manual/macOS evidence; unresolved required
   Mac checks mean verification is incomplete.

**Exit criteria:** Both entry points meet the interaction table and data contract; the
operator escape hatch works; existing `:`/`^`/`&` behavior and aggregate capture remain
intact; Rust and supported macOS checks pass; reviewable visual evidence or an explicit
macOS inspection record establishes the UI quality.

## Overall acceptance

An identified selection from `@file+` expands to `@file+id`; an identified selection
from terminal/leading `+` expands to `@file+id` from any eligible capture note. Both use
the same fuzzy picker language and shared lifecycle. Filtering is local and write-free;
insertions obey backend UTF-8 ranges; ID-less selection is explicit and stale-safe.
Ordinary plus text, authored bullet markers, operator commands, typed suffixes, exact
existing selectors, and old clients have intentional tested behavior. The UI clearly
shows scope and the insert action, and is verified on macOS before the epic is declared
complete.
