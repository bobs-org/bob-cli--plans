---
tier: tale
title: Add whole-item Pomodoro split capture
goal:
  Complete bob-cli-fe.3 with grammar 5 split execution, editor support, completion,
  previews, documentation, and regression coverage.
size: medium
proposed_by: bbugyi200.apollo.bob-cli-fe.3
proposed_with: codex/gpt-6-astra
bead: bob-cli-fe.3
status: done
---

- **PARENT:**
  [202611/merged_pomodoros.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/merged_pomodoros.md)
- **BEAD:**
  [bob-cli-fe.3](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-fe/bob-cli-fe.3.md)
- **AGENTS:**
  - [bbugyi200.apollo.bob-cli-fe.3](https://github.com/bobs-org/bob-cli--agents/blob/main/sessions/bbugyi200.apollo.bob-cli-fe.3.md)
- **COMMITS:**
  - [ccd1ada](https://github.com/bobs-org/bob-cli/commit/ccd1ada109a634bf58b2261b5f3c0a2a84daa5ef)
    — feat(capture)\!: add =/ split syntax end to end

# Implement `=/` Pomodoro splitting for bob-cli-fe.3

Complete only the `capture-split` phase of `plan:202611/merged_pomodoros.md`. `=/`
splits the current merged Pomodoro into planned entries by task type, with matching
editor spans, completion, preview, and committed output. This is one bounded session
action built on the pure planners already landed in `675330dd`; one follow-up coding
agent can implement and verify it.

## Starting evidence and boundaries

- `sase bead read bob-cli-fe.3 -r "Need the phase scope and design file"` reports
  `in_progress`, assigned to `bob-cli-fe.3`, with no previous-work notes. Do not change
  its status by hand or reopen it unless host-authorized recovery requires it.
- Dependency `bob-cli-fe.1` is closed. Its commit provides `split_plan`,
  `select_by_type`, `SelectFilter::MergedBase`, and shared vector coverage in
  `src/native/pomodoro_themes/merged.rs` and `routing.rs`. Use these implementations; do
  not redo the core phase or change its shared vector sections.
- Capture still has `GRAMMAR_REVISION = 4`; `EqualsToken` has only Close and Start.
  There is no split action integration yet. Recheck the incoming tree before coding
  because other epic phases may have landed after this inspection.
- `sase bead epic-symbols bob-cli-fe.3` currently reports no entries. Recheck before
  closing, after implementation.
- The epic decisions are final: `new_task_named_link=merge`, `shed_on_move=yes`,
  `memory_merged_pomodoro=yes`, and `memory_links_merge_record=no`. Their integration
  belongs to sibling phases. This phase edits no memory. It does not implement `=*`,
  landing changes, close-name changes, Obsidian, or Mac UI behavior.
- Keep `Cargo.toml` version and `CHANGELOG.md` unchanged. The host commit for the
  implementation should use `feat(capture)!:` for grammar 5.

Read the parent plan through `sase artifact read` and relevant reference memory through
`/sase_memory_read` when beginning implementation. In particular, consult `docs.md`
before help and documentation edits, and use the existing Pomodoro, Task Link, and Task
Type definitions. No external checkout is needed.

## Observable contract

| Input              | Result                                                                                                |
| ------------------ | ----------------------------------------------------------------------------------------------------- |
| `=/`               | Split all merged-in types from the running entry, else the next planned entry a bare `=` would start. |
| `=/build`          | Select an open merged entry by its base type; split all its merged-in types.                          |
| `=/build#fix#read` | Split only those selected merged-in types from an entry with at least three types.                    |
| `=/#fix`           | Select the current entry and split FIX; it must have at least three types.                            |

The base is the first **type** component, not necessarily the first name component:
labels do not count. An explicit base uses whole-key matching before prefix matching,
prefers the running matching entry, then ledger order, exactly as
`select_by_type(..., SelectFilter::MergedBase)` specifies. No selector creates or starts
an entry. Current selection must reuse start's running/next rules, including the
existing ambiguity safeguards, rather than choosing an arbitrary timed entry.

Only top-level child bullets whose entire body is a Task Link move. Recognize the close
classifier's `[[T]]`, `[[T]]#`, `![[T]]`, and `![[T]]#` forms, including same-note
`[[#^id]]`, and move each qualifying bullet's complete subtree. Resolve the task's
effective type from staged note text. Prose, mentions, non-task links,
unresolved/untyped links, drifted types, base-type links, and unselected types stay.

The source keeps its position, checkbox, timespan, and other line content. Its name
loses only the selected type components; labels and remaining spelling/order stay.
Insert one untimed planned `- [ ] () — T` immediately below the source block for each
selected type, in source-name order, even for types with zero links. Existing planned
entries of the same type are not reused. A running source keeps running. Task
fields/statuses are not changed by splitting.

Use `SplitError::code()` and teaching messages for `split_closed`, `split_meet`,
`split_not_merged`, `split_base_type`, `split_type_unknown`, `split_type_repeated`, and
`split_type_not_needed`; missing target uses `split_target_missing`. Explicit `#type` is
refused for a two-type target, teaching `=/build` or `=/`. A missing explicit base must
not silently fall back to current, and closed entries never participate in selection.
Missing-target guidance lists today's eligible merged entries with valid examples. MEET
remains protected.

## Implementation

### 1. Add the shared syntax and editor representation

Extend `capture_language/model.rs`, `item.rs`, and `mod.rs` with a split spec and
`CaptureKind::PomodoroSplit` / `EqualsToken::Split`. Keep selector spelling and
literalness available to the planner, along with exact source offsets for editors. Use
`bare_start_name_len` for the optional base; reuse the named-start lexer for each
`#name` or `#"SASE V18"` split selector, including its supported quote forms. Do not let
adjacent `#` selectors be swallowed into one name or interpreted as ordinary task-type
markers.

Claim leading `=/` before generic start/prose handling. A split is a whole-item session
action: extra prose, child lines, malformed selectors, route/section/task destination
flags, and other conflicting capture modifiers must produce clear diagnostics with no
task creation. A mid-body `=/` remains prose. Preserve ordinary `=` starts, bare `x`
closes, and all existing retired-close behavior, including `=*`.

Include valid split tokens in `is_session_chain_token`. Chains such as `x =/` and `=/ =`
must retain per-token ranges and execute left to right. Use the existing quote-aware
tokenization so whitespace inside literals does not split a chain.

In `capture_language/editor_model.rs`, editor classification/parsing helpers, and
`capture_parse.rs`, add `EditorMode::PomodoroSplit`, the serialized split spec, and
`SpanKind::PomodoroSplit` covering only `=/`. Emit `pomodoro_name` spans on the base and
split selector names with ranges compatible with `attach_theme_refs`;
`is_theme_name_span` already recognizes that span kind. A trailing `#` produces need
`pomodoro_split_type`. Preserve the editor's incomplete-literal behavior. Update
single-item and batch projections and human parse output together.

Bump `GRAMMAR_REVISION` to 5 once. Find and update exact grammar assertions and goldens,
including the literal JSON in `tests/cli/capture/complete_relationship_owner.rs`. Keep
unrelated schema versions.

### 2. Apply the split through the staged batch planner

Add a focused module such as `capture/pomodoro_split.rs`. Wire dispatch and conflict
checks through `capture/mod.rs`, `plan.rs`, and `batch.rs`, following the existing
start/adjust/shift action pattern. Populate every required plan/result constructor and
kind label so splitting never falls into ordinary task planning.

Read today's daily note through `CaptureBatchPlanner`. Build shared selector input from
its scan and the same `ThemeSources` used by type routing. Resolve the target, then
build the source's top-level Task Link list with original line/subtree identity. Reuse
or narrowly expose the existing close classifier; do not introduce a looser wikilink
recognizer. Resolve link targets relative to the daily note, read their current staged
task lines, and call `CaptureTaskThemeContext::placement_for_task`. In particular, a
preceding item that changes a task type must affect this split.

Pass the ordered link keys and selector set to `merged::split_plan`. Map its guard
errors to structured capture errors with precise messages. Treat missing/unresolvable
links conservatively as staying links, using established warning handling.

Reuse `pomodoro_themes/relocation_apply.rs::rename_entry`,
`capture_task_toggle::insert_named_placeholder`, and `move_subtree_to_entry`, or extract
a narrow shared helper when their current insertion API cannot express "directly below
this source block." Preserve newline convention, separators, indentation, nested
content, duplicate bullets, and neighboring sections. Unlike merge, split does not
deduplicate links.

Avoid stale line indices: perform the transformations against a single staged snapshot
with explicit offset accounting, or rescan/rebase after each structural edit. Preserve
link order inside each destination and destination order from the pure split plan. Stage
the final daily-note bytes only through the batch planner so dry-run, optimistic
preimage validation, atomic commit, and rollback remain shared.

### 3. Return complete previews and human output

Add the optional output field to the appropriate detail, item, and single-result
structures in `batch.rs` / `output.rs`:

```text
pomodoro_split: {
  path, entry_line, before, after, base: ThemeRef,
  created: [{type: ThemeRef, line, name, moved_links}], stayed_links
}
```

Use existing path and one-based line conventions. Counts refer to top-level Task Links;
nested lines are preserved but do not inflate counts. `stayed_links` counts all source
Task Links left behind, including unresolved/untyped ones. Derive the base and created
type pills through the existing ThemeRef/catalog machinery. Omit `pomodoro_split` on
unrelated result kinds.

Human output should make the result visible without exposing planner internals:

```text
split  BUILD + FIX + READ → BUILD · 20261101.md line 14
  ↓ FIX   new Pomodoro · 2 links
  ↓ READ  new Pomodoro · no links
```

Report the source and every created entry through `pomodoro_blocks` refs, including
created entries with no links. Extend role handling only as needed and document any new
role. Feed correct before/after indices into `PomodoroBlockTracker` so a later chain
action rebases the same blocks rather than losing or duplicating them. Keep dry-run JSON
and human previews consistent with committed output and bytes.

### 4. Complete split bases and selected types

Add `CompletionContext::PomodoroSplitBase` and `PomodoroSplitType` in
`capture_language/completion.rs`. Recognize these before the session-action early return
and before generic task-type masking can consume a split `#`. Detect the active token
inside a chain and return exact UTF-8 byte replacement ranges that preserve surrounding
selectors, quotes, and subsequent actions.

Wire discovery in `capture_complete/engine.rs` with focused helpers alongside
`pomodoros.rs`, updating model, rendering, shell adapters, context labels, and help.
Base rows offer today's open merged entries using the base selector as the insertion and
the full entry name as the visible label, plus a current choice that inserts the empty
base (it must not introduce a literal `current` type). Do not offer creation or
historical entries. Selector acceptance must round-trip to the same target the execution
resolver selects; handle duplicate bases consistently.

Type rows resolve the selected/current target with the shared resolver, offer its
merged-in types in name order, exclude the base and already-selected keys, and show Task
Link counts (including zero). Count with the same classification and effective-type
rules as execution. Quoted literals match whole component keys; prefix matching applies
only to nonliterals. For two-type targets, preserve the implicit-split guidance rather
than making a forbidden explicit selection look valid. Completion stays read-only and
uses the existing discovery snapshot model.

### 5. Document and verify the complete contract

Update capture, capture-parse, and capture-complete help. In `docs/capture.md`, add
grammar-at-a-glance rows, "You typed" examples, a "Splitting a merged Pomodoro" section
next to named starts, chain rules, and split `pomodoro_blocks` examples. Update
`docs/reference/json-outputs.md` with grammar 5, mode/span/need/context names, result
fields, and error codes. Regenerate affected CLI references with `just docs-gen`;
inspect generated changes for relevance. The shared split rules are already documented
in `docs/task-types.md`; keep that contract consistent.

Add targeted unit tests in the existing capture-language and completion suites, and CLI
tests under `tests/cli/capture/` with reusable fixture data under
`tests/fixtures/capture_pomodoros/`. Include these meaningful cases:

- Two-type implicit split; three-plus types split-all and selected subsets; selector
  order differing from name order; labels/exempt PLAN; zero-link new entries.
- Running/current and next-planned fallback; explicit base exact/prefix selection and
  running preference; missing targets, no eligible entry, and MEET protection.
- Every reachable guard code, duplicate selector keys, base/unknown selections, and
  two-type explicit-selector refusal; all refusals leave vault bytes untouched.
- All four Task Link forms, same-note links, aliases, nested subtrees, prose and
  mentions, unresolved/untyped/drifted links, CRLF, and neighboring entries/sections.
- Whole-item claiming, malformed or incomplete selectors/literals, trailing `#`,
  conflicting flags, children, mid-body prose, and unchanged starts/retired closes.
- Editor mode/spec/spans/pills and UTF-8 ranges; both completion contexts, counts,
  omitted-base insertion, literal insertion, exclusion, and chain-local replacement.
- Dry-run leaves every file unchanged and describes the same transformation a real
  capture commits. Assert exact daily-note postimages and unchanged task-note bytes.
- `x =/`, `=/ =`, and an earlier staged type change followed by split. A later failing
  item rolls back the entire batch. Block previews identify all changed and created
  entries after later chain edits.

Run focused capture-language and new split CLI tests while implementing, then
`just check` and `just docs-check`. Existing Rust shared-vector tests must still pass
unchanged. Follow `/sase_monitor` for long checks when a handoff is needed; never end a
turn with an unmanaged command still running.

## Completion and bead ownership

The implementation is complete when the documented split spellings work through CLI
capture, parse, completion, dry-run, real commits, and chain previews with the tests
above, and required gates pass or their clean-base failures are evidenced.

Before closing, run `sase bead epic-symbols bob-cli-fe.3`. Resolve each remaining entry
or re-key it to a still-open parent/later phase as appropriate. Record any out-of-scope
work only with `sase bead note bob-cli-fe.3 'PROPOSED FOLLOW-UP: ...'`; do not create
task beads. A gate failure reproduced identically on the clean base does not keep this
phase open: record the reproduction, and cite an existing task bead when one already
tracks it.

Close only `bob-cli-fe.3` using `sase bead close bob-cli-fe.3 --note` with concrete
verification evidence. Do not close `bob-cli-fe` or any ancestor plan bead. Use
`/sase_final` for the final host declaration; the planning turn keeps the assigned phase
open, and the implementation turn closes it only once its full scope is done.
