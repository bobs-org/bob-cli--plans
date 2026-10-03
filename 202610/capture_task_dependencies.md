---
tier: epic
title: Capture task dependencies with an ampersand picker
goal: 'Bryan can add prerequisite links to new or explicitly selected existing tasks
  with &note:block-id in bob capture and use a beautiful, responsive vault-wide dependency
  picker and accurate preview in Bob Mac Capture.

  '
phases:
- id: dependency-contract
  title: Define dependency capture grammar and the additive JSON contract
  size: medium
  depends_on: []
  description: 'dependency-contract: implement shared lexical ownership, incomplete
    states, spans, and documented additive dependency wire types.'
- id: dependency-discovery
  title: Discover prerequisite tasks throughout the vault
  size: medium
  depends_on:
  - dependency-contract
  description: 'dependency-discovery: add vault-wide prerequisite discovery, fuzzy
    ranking, exact note identities, and safe block-ID assignment support.'
- id: dependency-writes
  title: Apply dependency captures with staged multi-note writes
  size: medium
  depends_on:
  - dependency-contract
  - dependency-discovery
  description: 'dependency-writes: merge managed dependency lines and derived effects
    through the capture batch planner with validation, rollback, and final task previews.'
- id: dependency-mac
  title: Present the dependency picker and preview in Bob Mac Capture
  size: medium
  depends_on:
  - dependency-contract
  - dependency-discovery
  - dependency-writes
  description: 'dependency-mac: consume Bob''s contract in a dependency picker, block-ID
    flow, semantic highlighting, and accessible task preview.'
- id: dependency-verification
  title: Verify the integrated contract and finish the visual review
  size: small
  depends_on:
  - dependency-contract
  - dependency-discovery
  - dependency-writes
  - dependency-mac
  description: 'dependency-verification: exercise real CLI-to-app fixtures, inspect
    rendered picker states on macOS, and finish coordinated compatibility documentation.'
proposed_by: bbugyi200.apollo.4m
create_time: 2026-10-03 09:11:55
status: done
bead_id: bob-cli-3u
---

- **PROMPT:** [prompts/202610/capture_task_dependencies.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202610/capture_task_dependencies.md)
- **BEAD:** [bob-cli-3u](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-3u/README.md)

# Capture task dependencies with &

## Outcome and scope

`&foo:bar` means “the task I am capturing or explicitly targeting depends on the task at
foo.md / ^bar.” It adds a plain task dependency link, normally `[[foo#^bar]]`, to the
dependent's managed `⛓️ **DEPENDS ON:**` child. The `%` in the request's first link
example is treated as a typo: the existing contract and the request's concrete example
both use `#^`, never `#%^`.

Choose an **epic**, not a tale. This crosses two repositories, introduces a new capture
modifier and existing-task action, needs a writer that coordinates several notes, and
must expand discovery beyond the current `:` picker. Four bounded `medium`
implementation phases and a `small` integration/visual phase make the contracts
reviewable and keep macOS verification explicit. The dependency list is intentionally
serial: the discovery and writer share task identity helpers, and the Mac client should
consume a working backend. No worker needs a second design handoff. No model overrides
are requested.

Only this scratch plan is authored during planning. Implementation follows approval.
This feature does not require a plugin change or a vault migration: existing dependency
chips and hooks already understand the canonical line. Do not edit durable memory,
deploy plugins, install applications, or modify Bryan's real vault as part of tests.

## Evidence and implementation anchors

The following were inspected while planning; paths are relative to their repositories.

| Repository      | Existing code/contract                                                                                                                                  | Consequence                                                                                                                                                                                             |
| --------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| bob-cli         | `docs/capture.md`; `src/native/capture_language/{item,line,tokens,markers,completion,editor_parse,editor_classify,editor_model,model,draft,rewrite}.rs` | Execution, lexical preview, rewriting, and cursor completion share Rust grammar. Ordinary body text already creates a task; `!` is not a new task classifier.                                           |
| bob-cli         | `src/native/capture_link_tasks.rs`; `capture_complete/{engine,candidates,model}.rs`                                                                     | `:` has tiered fuzzy ranking and missing-ID candidates, but only scans routable inbox/area/project notes. Reusing its pool unchanged would fail vault-wide discovery.                                   |
| bob-cli         | `src/native/note_tasks.rs`, `vault_links.rs`, `capture_task_id.rs`                                                                                      | Task refs are stale-safe line/digest refs. Existing route tokens allow only ASCII letters, digits, `_`, and `-`; they do not represent arbitrary vault paths.                                           |
| bob-cli         | `src/native/task_dependencies/{mod,parse,format,legacy}.rs`; `task_status_hooks/reconcile{,/fields,/apply}.rs`; `docs/task-dependencies.md`             | Canonical line grammar, note identities, projection fields, legacy adoption, and status semantics already exist. Reuse/extract pure helpers rather than inventing another dependency representation.    |
| bob-cli         | `src/native/capture/{plan,batch,commit,output,task_blocks,block_diff}.rs`; `task_status_hooks_write/`                                                   | Capture stages a full batch and rolls back failed writes; task block tracking already produces final block previews. The current capture commit loop needs read-preimage checks for dependency batches. |
| bob-mac-capture | `Sources/CaptureCore/{TaskLinkPickerPresentation,CapturePickerPresentation,CaptureModels,BobProcessClient,CaptureTextRanges}.swift`                     | Reuse the fetched snapshot, local fuzzy filtering, source-agnostic picker card, typed decoding, and byte-range handling.                                                                                |
| bob-mac-capture | `Sources/BobMacCapture/{CapturePanelModel,CapturePickerState,CapturePickerView,CaptureKeyCommandRouter,CaptureEditorPalette,TaskBlockView}.swift`       | `:` opens from Bob's `task_link` context. It has separate ID assignment and Shift-Return-to-start behavior; neither should accidentally become dependency behavior.                                     |
| bob-mac-capture | `Tests/BobMacCaptureTests/{CapturePickerDesignTests,CaptureTaskLinkPanelTests}.swift`, `justfile`, `.github/workflows/ci.yml`                           | There are model tests and an ImageRenderer review path, enabled by `BOB_MAC_CAPTURE_RENDER_DIR`. Actual Swift/AppKit checks require macOS.                                                              |

Read relevant memory through `/sase_memory_read` during implementation:
`decisions:task-deps-are-depends-on-links`, `decisions:mac-capture-is-a-thin-client`,
`decisions:task-lanes-are-sticky`, `glossary:task-dependency-link`,
`glossary:task-freshness`, and `cli_rules.md` when adding options. The plan follows
those decisions without changing them.

Open the Mac repository through `/sase_repo` before reading or editing it. The
configured `sase repo open bob-mac-capture` failed during planning because its primary
checkout was absent; `sase repo open gh:bobs-org/bob-mac-capture -r "..."` succeeded.
Use whatever checkout that command prints in the implementing agent's workspace; do not
assume planning-time paths or clone it manually. No Mac `AGENTS.md` was present in the
inspected checkout; recheck instructions when opening it.

## User-facing contract

### 1. Select the dependent by the existing capture meaning

Remove recognized dependency modifiers, then use the shared capture grammar to determine
the owner. An explicit `@route+id` selects the existing parent even if there is
child-bullet text. Do not infer a task from punctuation or search for a similar task
title. The following table is normative:

| Capture item                                           | Result                                                                                                                                                  |
| ------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `Buy Groceries! &foo:bar`                              | Create the ordinary inbox task, keeping `Buy Groceries!` as its text, and add foo/bar as its prerequisite.                                              |
| `Buy Groceries! @home &foo:bar`                        | Same, in home.md; routing and dependency markers also work in the reverse trailing order.                                                               |
| `Buy Groceries! @home^groceries &foo:bar &cash:budget` | Create the named task with two prerequisites, preserving typed order.                                                                                   |
| `&foo:bar @body+excercise`                             | Add the dependency to existing body/^excercise. No new task, empty child, link toggle, or Pomodoro operation.                                           |
| `&foo:bar @body:excercise`                             | Same as the preceding row. This narrow alias honors the exact example in the request.                                                                   |
| `@body+excercise &foo:bar`                             | Same dependency-only update, independent of token order.                                                                                                |
| `Waiting for approval @body+excercise &foo:bar`        | Keep existing sub-bullet capture behavior for the prose, and add the dependency to its explicit parent. Preview must show both effects.                 |
| `Buy Groceries! @home:groceries &foo:bar`              | Create a new Pomodoro-linked task using the existing colon meaning and add its prerequisite; dependency preparation precedes ledger/status planning.    |
| `&foo:bar`                                             | Incomplete: a prerequisite has been selected, but no dependent was named. Teach “Add task text or @note+task-id”; never create an ampersand-named task. |
| `&`, `Buy Groceries! &fo`, `Buy Groceries! &foo:`      | Incomplete prerequisite selection; parser reports the need and completion opens. Execution refuses with a useful item/line diagnostic.                  |
| `R&D research`, `Research & Development`               | Literal text; ampersands inside words and a standalone ampersand between prose words are not directives.                                                |
| `Buy Groceries! \&foo:bar`                             | Explicit literal token; the dependency scanner consumes only this escape and leaves the visible `&foo:bar` in task text.                                |

The colon alias applies **only** when at least one dependency modifier exists, there is
no body or authored content, and there is exactly one unsuffixed `@route:id`. Without
`&`, `@route:id` and bare `@route+id` retain their existing link/toggle meanings. The
picker/docs prefer `@route+id` for an existing dependent. Do not accept a
dependency-only colon target with `#name`, `=...`, `=x...`, or an explicit `!` toggle:
diagnose the combination and suggest a separate blank-line item for the Pomodoro action.
New task captures keep their otherwise supported schedule, priority, clipboard, and
Pomodoro start/close modifiers, with the whole draft still one transaction.

Apply the same refusal to `@route+id!` and dependency-only `@route+id#name`:
dependencies never smuggle a toggle/name action into an existing-task update. With
actual child prose, `@route+id#section` retains its normal sub-bullet placement; the
dependency belongs to the parent task and stays outside that child section.
Dependency-only items reject schedule/priority/clipboard modifiers rather than silently
applying task-creation modifiers to a pre-existing task. Conflicting forced destination
flags retain the existing usage-error policy; `--route` can route a new task with
dependencies but never turn an explicit dependency update into a new task.

An `@@route` declaration continues to route new tasks. `@@route+id` continues to select
an inherited existing parent; dependency modifiers on its items apply to that parent. An
explicit local target wins using existing inheritance rules. Dependencies are item-local
and never become a global declaration. A dependency-only item inheriting just `@@route`
still needs task text or a parent ID.

### 2. Marker boundaries and exact note identities

Support one or more dependency modifiers at the leading or trailing marker positions of
the parent line. In the trailing marker run, dependencies interleave with the existing
destination, schedule, priority, and clipboard markers in either order. Also follow
capture's existing item-wide marker convention on authored child lines: a terminal
dependency there belongs to the same capture item's dependent. A child containing only
dependency modifiers contributes no empty bullet; a child with prose keeps that prose.
Never silently attach it to a neighboring nested task. The picker header identifies the
owner. Project-note task construction, section-bullet items, Pomodoro notes, and
standalone session operators have no unambiguous single task owner for this modifier:
reject claimed dependencies in those modes with a targeted diagnostic. Literal
ampersands in code, URLs, wikilinks/aliases, and prose remain text.

Recognize complete `&note:id` tokens and partial query tokens (`&`, `&query`, `&note:`)
only at a whitespace boundary in those positions, outside protected Markdown spans. The
interactive trigger is `&` at the item's beginning or after whitespace at the line end;
editing an existing dependency token can reopen its picker. An unresolved query cannot
silently fall back to prose on submit. `&` in the middle of `Research & Development`
stays literal once followed by ordinary prose. Add specific regression vectors for this
transition and for escaping.

The prerequisite note is a vault-relative identity, **not** a capture destination:

- Simple root notes use `&foo:bar`; nested notes use `&projects/foo:bar`.
- For whitespace or reserved punctuation, use a quoted note component such as
  `&"Projects/Shopping List":bar`. Inside that component only `\"` and `\\` are escapes.
  The Rust formatter selects quoting and escapes; Swift inserts its string verbatim.
  Preserve note-path case and Unicode. Block IDs keep Bob's existing rules.
- The note component is extensionless, like current routes. Resolve an explicit relative
  path first, otherwise a unique basename using the dependency resolver's semantics.
  Disambiguate candidates with full relative paths. Reject traversal, absolute paths,
  invalid empty components, and paths escaping the vault, including through a symlink.
  Do not globally broaden the existing `@route` grammar.
- Do not normalize dependency paths with the existing lowercasing route parser. Display
  locators and generated wikilinks are different strings: the latter use the shortest
  unambiguous form from `task_dependencies::format::canonical_link`.
- Quoted-note parsing must preserve original byte offsets and cooperate with the
  existing tokenizer; do not implement it with a global whitespace split. A quoted
  partial remains one incomplete modifier. The CLI examples quote the entire draft
  because a shell treats an unquoted `&` as control syntax.

### 3. Dependency writes and status effects

The canonical source of truth stays the managed child line. Reuse
`docs/task-dependencies.md` sections 2–5 and the DP/DW/DR vectors:

1. Resolve the dependent and every new prerequisite against staged contents. Existing
   dependents must be open tasks; never reopen a completed dependent. Reject missing,
   ambiguous, duplicate-ID, or non-task targets. A task in a prior item of the same
   draft is usable after that item creates its explicit ID; forward references to a
   later item are rejected. This matches capture's sequential planning model.
2. Append new unique prerequisite identities in typed order. Existing links keep order;
   repeats in the draft or already-present links are idempotent, not toggles and not
   reorder operations. Add a dependency through this syntax; removal stays with the
   existing dependency editor. Preserve closed prerequisites already on the line.
3. Merge into one canonical child line, first except after an existing Cancel Log.
   Preserve child indentation, CRLF/LF, final-newline state, and unrelated content. Fold
   contract-defined legacy children when touching the dependent; adopt existing
   field-only dependencies before adding. Never discard an existing prerequisite to make
   the new write succeed. Refuse a malformed or multiple managed line situation with a
   repairable diagnostic. Preserve unresolved links and their healing IDs when safely
   possible under R4; if adoption cannot preserve all existing relationships, refuse the
   edit rather than losing them.
4. Derive `[dependsOn::]` and necessary target `[id::]` from the resulting line inside
   this same plan. Prefer existing valid IDs. Use the shared canonical ID encoder only
   when needed and preserve field placement after freshness and before `^block-id`. An
   unencodable path without an existing valid `[id::]` is a guarded refusal, exactly as
   in the existing dependency editor. Adding `^id` alone cannot cure that case.
5. Reject self-dependencies and any **new** cycle, including a cycle introduced by two
   items in this draft, with a readable path. Existing unrelated graph problems do not
   make a no-op repeat fail. Cycle checks use line/legacy edges, not just stale fields.
6. Apply immediate Blocked/promotion behavior from contract §5 using shared pure
   helpers: an open prerequisite blocks the dependent; Next/In Progress dependents raise
   open prerequisites to at least their applicable lane, with future schedules and
   existing Blocked reasons respected. Never lower sticky lanes, retire a prerequisite's
   scheduled date, or create prerequisite Pomodoro links just because it was selected.
   Closed/cancelled/archive prerequisites do not block. Stamp today's freshness on an
   existing dependent actually edited by this explicit gesture; creation, target-ID
   preparation, prerequisite promotion, and a wholly unchanged duplicate request do not
   stamp freshness. Use the existing capture clock/helper.
7. Keep dependency changes in `CaptureBatchPlanner`. Reuse/extract projection and status
   helpers as needed; do not run the whole-vault hooks command after committing and do
   not let preview mutate anything. No dependency-only capture requires today's daily
   file to exist or rewrites unrelated notes.

All final changes are planned before writing: the dependent, target ID fields, promotion
effects, optional child content, and any normal capture/ledger changes. Use the batch's
current text for same-note edits so a target-ID change cannot overwrite a newly inserted
child. Add read-preimage checking to dependency batches, following
`task_status_hooks_write` patterns: track decision inputs, revalidate before applying,
and refuse a changed source instead of silently overwriting it. Keep the existing
temporary-file/rollback mechanism and exercise failure after at least one replacement.
Do not claim a multi-file rename is filesystem-level atomic; the guarantee is capture's
staged, guarded, rollback-on-failure batch contract.

The previous daily snapshot remains read-only. If a prerequisite there lacks a usable
identity and would require stamping, show a precise refusal, not a link that hooks will
immediately drop. Resolve archive links with explicit `done/...` paths and the archive
catalog's closed semantics. No historical task is made open by this operation. Discovery
should show guarded rows rather than silently excluding a task whose identity cannot
currently be used.

### 4. Discovery and JSON contract

Keep `schema_version: 1`: the change is additive. Freeze example payloads in
`docs/capture.md` during the contract phase and use them in CLI and Swift tests.

- `capture-parse` stays entirely lexical and filesystem-free. Add optional per-item
  `dependencies` entries with raw/range, note, and block ID, plus optional
  `dependency_target` describing `new_task` or `existing_task` ownership. A complete
  dependency-only action has mode `task_dependency`; new-task/sub-bullet modes keep
  their established value and gain the modifier fields. Incomplete inputs report
  `task_dependency` (prerequisite picker) and/or `dependency_target` needs as
  appropriate. Add non-overlapping semantic spans for dependency sigil, note, and block
  ID, and for an existing-task owner where necessary. All offsets index the original
  draft's UTF-8 bytes, including Unicode, quotes, CRLF, and batches.
- `capture-complete` returns context `task_dependency`, a replacement range covering
  exactly the active ampersand token including its sigil/quoted note, and candidates
  with Bob-authored full `replacement` strings. It also returns the decoded `query` and
  optional owner metadata so the app never has to parse quoted components. Refetching at
  `replacement.start` returns the unfiltered snapshot as `:` does.
- Reuse established task fields (`text`, `ref`, `block_id`, status, section, depth,
  line, `requires_block_id`, `block_id_suggestions`) and add an exact `note_path`
  (including extension), display locator, stable group, `already_dependency`, and
  optional `disabled_reason`. Backend-derived owner/eligibility data is authoritative.
  ID-less or guarded rows never supply an insertable empty replacement. Existing `route`
  remains optional display/backward-compatibility data, not a substitute for the exact
  path. Do not send the Pomodoro `pulls_forward` promise for dependencies.
- Scan all actual task-bearing vault notes using shared task settings/global filter:
  include untyped root notes, nested folders, ref notes, terminal projects, daily tasks,
  hidden tasks, and completed/cancelled/archive tasks. Exclude dot-directories,
  `_templates`, `_generated`, `_conflicts`, fenced code, and non-task block anchors. The
  literal “any task” request must not inherit the `:` route/status restriction. Known
  self/already-selected targets and unusable identities remain visible with an
  explanatory badge/guard. Unreadable notes yield bounded warnings and do not destroy
  good results; a requested write target must fail decisively if unreadable.
- Empty-query order: same-note open tasks, Pending/In Progress, Next, other open tasks
  grouped by note, then Completed/Cancelled (including archive history); hidden tasks
  are subdued and ordered last within their section. Stable path/line order breaks ties.
  Keep completed history in a clearly separate collapsible section; searches include it
  automatically so completed tasks remain findable.
- Reuse the `capture_link_tasks::rank` tiered AND-term matcher for task text, exact note
  locator, block ID, path, and section. Query matches remain the primary ordering within
  the open and completed groups; deterministic empty-query order breaks ties. Use the
  same vectors in Rust and Swift, including UTF-8/highlight edge cases.
- Opening the picker fetches a snapshot; filtering/navigation in its search field remain
  local. Refresh on reopen, vault-change invalidation, and successful ID assignment, not
  on every filter keystroke. Guard async results by generation, draft snapshot,
  replacement range, and picker source. Large candidate counts must not render unbounded
  rows or truncate the searchable dataset.
- Add optional `dependency_update` execution/preview detail: dependent identity and
  text, added/already-present counts, prerequisite summaries (note, block ID, canonical
  link, status and text), open prerequisite count, and resulting dependent status.
  Report final dependent/changed-target blocks through `task_blocks`, adding roles
  `dependency` and `dependency_target`. Deduplicate blocks in a batch and preserve new
  tasks without a user-authored ID. Human output describes “Add dependency to…” or
  “Already depends on…”; it never reports a dependency-only action as a new task.

For missing block IDs, reuse the explicit Add block ID interaction. Extend
`capture-task-id` with a mutually exclusive `--note-path/-n` alternative to `--route/-r`
and an opt-in `--allow-closed/-a` for dependency candidates; the old route/open-task
behavior stays unchanged. Validate the exact path and stale task ref, all anchor
collisions, writability, and dependency eligibility before assignment. Return optional
`note_path` and a backend-formatted `dependency_replacement` so the app cannot lose
quoting/case when resuming. The opt-in does not reopen a closed task. Honor archive and
previous-daily constraints above, and document any guarded refusal. Keep options
alphabetized and help/examples excellent.

This ID assignment is deliberately an explicit, separately completed preparation action,
matching the existing `:` flow. Selecting/highlighting a row writes nothing; the
prompt's button says **Add ID and use task** and explains it edits that note now.
Cancelling the later capture can therefore leave an unused block ID, as today. The
dependency line, derived target `[id::]`, and status effects are written only on final
capture submission. Do not build a second draft-local mutation protocol in this feature.

### 5. Mac experience

Use the existing picker card, adaptive palette, focus handling, and row metrics. Add a
`.dependency` picker source and a dependency presentation/index backed by Bob's fields.
Shared task-row rendering/fuzzy primitives should be factored only where useful; do not
copy the entire task-link state machine or fork its grammar.

Illustrative layout, using the existing card's typography and spacing:

```text
  ⛓  Depends on                         18 tasks
     For: Buy Groceries!
  ⌕  Search tasks by text, note, or ^id

  IN THIS NOTE
  ○  Confirm grocery budget        cash · ^budget
  ◐  Finish meal plan              home · ^meals
  …

  Selected: Confirm grocery budget
  Open · will block Buy Groceries! until finished
  ↵ Use task    ⌘↵ Capture    ↑↓ Navigate    Esc Back
```

- The dependent header shows its task text and location when known. For an ownerless
  leading `&`, say **Choose a prerequisite**, with “Then add task text or @note+id”;
  choosing a prerequisite is allowed before specifying the dependent.
- Rows lead with readable task text and an existing status glyph; secondary text carries
  note/section and a compact `^id`. Highlight matched characters. Use subtle chain
  accents consistent with dependency highlighting, no new arbitrary palette. Full
  text/path and guard reasons appear in the detail strip/accessibility label.
  Completed/cancelled history is visually distinct and described as non-blocking.
- Selected/already-present prerequisites display **Already added**. Accepting such a row
  cannot remove/reorder anything; normally disable redundant accepts with the
  explanation. Backend idempotence remains the last defense for typed duplicates.
- Return or Tab accepts one task, replaces only Bob's range, and returns focus to the
  editor. At a terminal token, leave one separating space for typing another `&`;
  preserve existing suffix whitespace and surrounding text when editing in place.
  Command-Return follows the existing accept-and-submit convention only if the dependent
  is complete. Shift-Return has **no start-session behavior** for this source. Escape
  restores draft/caret and suppresses immediate auto-reopening; a small **Choose
  dependency** chip reopens it. Empty-filter Backspace removes only the active
  trigger/query using the source's existing guarded behavior.
- Arrow keys and Control-N/P navigate. Search never loses the selected row unnecessarily
  during refresh. Loading, empty vault, no matches, guarded rows, partial-read warnings,
  and ID-assignment errors each have a useful state. An ID prompt cancelled with Escape
  restores the dependency picker/filter/selection; assignment failure inserts nothing.
- Highlight `&` and its locator through Bob's semantic spans. Never claim ampersands
  inside ordinary text, a wikilink, or code with a Swift-side regex. Multiword search
  belongs in the dedicated filter field, not in a second Swift capture language.
- Preview shows **New task · depends on …** or **Add dependency to “Exercise”**, the
  actual resulting managed child, waiting count, and Blocked/closed status distinction.
  Reuse final task-block cards/diffs to show target ID/status effects, avoiding
  duplicate rendering. A dependency-only action must never be labeled “Create task” or
  “Link to Pomodoro.” An error leaves the draft open; one successful aggregate capture
  hides it.
- Decode optional fields with `decodeIfPresent`; retain schema checks and all legacy
  flows. Add the new needs/spans to completion request routing and markers to editor
  presentation. Keep grammar, candidate guards, locator formatting, and vault writes in
  Bob. The new syntax requires the updated CLI; coordinate CLI-first installation and
  document that an older CLI does not understand `&`. Do not claim old binaries can
  safely execute the new syntax merely because JSON decoding is compatible.

## Phases

### dependency-contract

Implement the normative grammar/lexical contract above and its documentation first. Work
in bob-cli `capture_language/`, `capture_parse.rs`, and the wire-model definitions
needed by completion/output; add a dedicated dependency token/parser module rather than
scattering suffix stripping. Account for raw spans, quotes, protected text, item-local
accumulation, existing-target precedence, global inheritance, and rewrite preservation
together. `capture-rewrite` must neither absorb `&` into `@@` nor turn the colon alias
into an action with different ownership.

Update `capture_block_ids.rs` intent determination with the shared parsed ownership:
`&foo:bar @body:` must complete an existing dependent, not offer new-task IDs just
because the raw draft contains ampersand text. `New task @body: &foo:bar` retains
new-task ID behavior. Dependencies must not enter the task title used for new-ID
suggestions. Freeze tests for both cases and for partially typed owner selectors.

Publish concrete v1 examples for new-task dependencies, dependency-only update,
ownerless/incomplete draft, Unicode quoted locator, ID-less candidate, guarded
candidate, and final preview. Update `docs/capture.md` and capture/parse/completion help
without advertising unimplemented execution as working. Until the writer phase,
recognized executable dependency input must fail closed with a temporary explicit
unsupported-action error rather than drop modifiers and create a normal task.

Tests: grammar units plus `tests/cli/capture/{parse,rewrite,batch}.rs` and a focused
dependency grammar integration module. Cover every contract-table row; reversed marker
order; multiple `&`; child-line markers; priority/schedule/clipboard modifiers;
`@@route`/`@@route+id`; invalid routes/suffixes; mode-specific rejection; offsets in
emoji/Unicode/CRLF/multi-item input; literal ampersands; malformed quotes and escapes.
Prove `capture-parse` is vault/clipboard independent, and all preexisting `:`/`^`/ `@`
tests retain their behavior. Exit criterion: documented JSON matches tested output and
dependency input cannot yet produce partial mutations.

### dependency-discovery

Implement shared dependency note locator resolution/formatting and a vault-wide task
catalog in bob-cli, reusing `note_tasks`, task settings, path exclusions, link
resolution, and rank helpers. Add `task_dependency` completion in `capture_complete/`
and shell completion coverage as applicable. Do not change the `:` discovery pool or its
ranking. Provide the prerequisite metadata and known owner guard data specified above;
graph guards may use an extracted pure dependency graph builder that the writer also
uses.

Implement the narrowly scoped task-ID endpoint additions and their stale/path checks. Do
not use an ordinary route to round-trip nested, case-sensitive, or quoted paths. Do not
require a ledger file just to discover a dependency candidate.

Tests: temporary vaults containing untyped root/nested/ref notes, terminal projects,
hidden tasks, daily/history/archive tasks, custom statuses, fenced code, non-task IDs,
excluded paths, symlinks, duplicate basenames, duplicate block IDs, unreadable notes,
and tasks without IDs. Prove exact-path and unique-basename resolution agree with
writer/link resolver semantics, and completion is read-only. Cover replacement ranges at
all cursor positions, full-list refetch, quoted paths, multiple dependencies in one
item, and no completion in literal protected text. Reuse DK ranking vectors, add
dependency groups/duplicate guards, and test stable order. ID-assignment tests cover
dry-run, stale ref, ID collision, closed opt-in, previous-daily guards, and every
unrelated byte preserved. Exit criterion: all returned selectable candidates are
representable by an authoritative dependency replacement or an explicit ID flow.

### dependency-writes

Implement a focused `capture/dependencies.rs` planner around the shared dependency
helpers, with minimal exports/extractions from reconciliation/status code. Wire it into
new-task and existing-task capture paths at the correct planning stage, removing the
temporary fail-closed execution guard. Resolve all edits from staged snapshots,
including newly captured tasks and same-note prerequisites; retain sequential batch
semantics. Add scoped preimage checks and rollback tests, dependency output details,
final task-block roles, human messages, and dry-run parity. Update dependency writer
documentation/conformance vectors where new capture-specific behavior needs examples.

Tests: both exact user examples plus `@body+excercise`; default/routed/named/Pomodoro
new tasks; parent-targeted prose; multiple prerequisites; repeated additions;
same-note/full-path/ambiguous-basename links; existing valid IDs; custom fields and
freshness placement; closed prerequisites; malformed/multiple lines; field-only and
legacy adoption; unresolved old links; direct/transitive/self/batch cycles; CRLF and
no-final-newline; Cancel/Schedule/Work Log placement. Prove a dependency-only capture
does not create a task/empty child, alter ledger links, pull schedules forward, or
require a daily note. Verify immediate status/promotion agrees with a subsequent hooks
reconciliation, including blocked/future-scheduled prerequisites and sticky lanes.

Exercise missing/stale/unreadable targets, changed input preimages, a later failing
batch item, failure after one file replacement, and cleanup of temporary/backup files.
Snapshot the whole fixture vault before each failure and compare afterwards. Confirm
dry-run returns the same planned final blocks as execution without writing and that
dependency projection is idempotent on a subsequent hooks run. These are correctness
tests for persistent writes, not tests mirroring private helper implementations. Exit
criterion: no partial dependency state, accurate final previews, and unchanged legacy
capture/task-status test behavior.

### dependency-mac

Open bob-mac-capture through SASE. Implement optional dependency decode models and
process-client support for exact-path ID assignment, then source-specific presentation
and routing through the existing picker state machine. Add shared row/index helpers only
where this reduces duplication. Consume the backend query, guards, spans, and
replacement; never rebuild a locator with string interpolation as the existing `:` ID
callback currently does. Add dependency-specific ID prompt purpose/return flow, accept
semantics, owner details, preview summaries, and task-block deduplication. Update app
README examples/help and accessibility descriptions.

Tests: new/old JSON fixtures; unknown optional fields; Unicode byte-range splicing;
leading `&`, terminal ` &`, existing tokens, multiple items, leading dependency then
owner; local multiword fuzzy ranking; completed/hidden/disabled rows; loading/empty/
error states; stale async results; selection-only cursor movement versus edit-trigger
opening; Escape and chip reopening; empty-filter Backspace; Return/Tab/Command-Return;
Shift-Return never appending `=`; no submit while owner incomplete; ID prompt cancel,
stale success, failure, nested/quoted note identity; one aggregate capture and hide only
on success. Preserve all `:` and `^` panel tests and the existing start-session
shortcut.

Add deterministic render fixtures for dependency picker and preview at 760 and 620 pt,
light/dark and increased contrast, scale 2. Include long task/path text, match
highlights, no owner, several dependencies, already-added/cycle guards, ID prompt,
completed history, and an existing-dependent diff. Check VoiceOver labels and keyboard
focus. Exit criterion: macOS build/tests pass and the feature uses the existing visual
language coherently.

### dependency-verification

Use a fixture vault and real updated Bob binary to generate the final parser,
completion, ID-assignment dry-run, and preview payloads; decode them with Mac tests
instead of relying solely on handwritten Swift fixtures. Exercise the two user examples
end to end and repeated multi-dependency input; verify the resulting files against
`docs/task-dependencies.md` and run hooks parity once.

Run bob-cli `cargo fmt --check`, `cargo clippy --all-targets --all-features`, and
`cargo test` (the repository's `just all` equivalent). Run bob-mac-capture `just all` on
macOS, using its Xcode wrapper and bundle checks. For long commands or CI waits, use
`/sase_monitor` and wait for its handoff command to exit. A Linux-only inspection does
not count as Swift/AppKit verification; use the repo's macOS CI and report any remaining
visual/manual evidence explicitly.

Set `BOB_MAC_CAPTURE_RENDER_DIR` for the dependency render tests, inspect actual PNGs,
and iterate on spacing, contrast, truncation, and focus before completion. Preserve
review images as artifacts only via the applicable artifact skill/memory workflow.
Measure a large synthetic vault once: record candidate count and scan time, verify
filter/navigation causes no subprocess launches, and verify bounded view rendering while
all tasks stay searchable. Recheck the supported feature on an actual Mac panel when
available; if CI supplies renders but no interactive session, distinguish that
limitation rather than claiming manual keyboard/VoiceOver testing.

Finalize CLI and app documentation, exact supported syntax/composition matrix, and
CLI-first rollout instructions. New app decoders must handle old CLI payloads for
existing features; old app decoders must ignore additive fields from new Bob. New
dependency syntax itself requires the updated Bob binary. No auto-update/install,
real-vault mutation, or plugin deployment belongs to this verification phase.

## Completion checklist

- Both requested examples and canonical `@file+id` targeting work, with no incidental
  task, child, ledger, schedule, or toggle effects on dependency-only captures.
- Typed and picked dependencies generate the same canonical line, fields, and status
  effects; repeated additions are harmless and failures leave the fixture vault intact.
- The `&` picker searches beyond capture destinations, finds ID-less and historical
  tasks, explains guards, and never silently substitutes another note/task.
- Bob owns syntax, replacement, preview, and mutations. Swift owns presentation and
  asynchronous interaction; the JSON contract grows additively.
- Actual Mac renders have been reviewed, Rust and macOS checks pass, and any unavailable
  interactive verification is recorded precisely. Implementation stays within the two
  repos and the requested feature; discovered unrelated issues are not bundled into it.
