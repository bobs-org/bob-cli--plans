---
tier: epic
title: Capture type-marker grammar and new-task typing
goal:
  "Complete bob-cli-ee.1 with one shared offset-preserving t: grammar, editor spans and
  diagnostics, new-task type writes and named-Pomodoro agreement, while preserving
  existing capture behavior and verifying its documentation."
phases:
  - id: language
    title: Shared type-marker lexer and execution grammar
    size: medium
    depends_on: []
    description:
      "language: implement the shared t: lexer, offset-preserving item views,
      original-draft boundary handling, execution metadata and teaching errors, and
      template integration. Preserve unmarked grammar and fail closed at the planner
      until the writes phase enables new-task typing. Add grammar and boundary vectors."
  - id: editor
    title: Editor spans and transparent completion parsing
    size: medium
    depends_on:
      - language
    description:
      "editor: consume the shared masked view before editor claims and existing
      completion detection. Add task_type spans, needs, diagnostics, shared exact-name
      resolution and ThemeRefs. Verify offsets, neighboring completion, rewrite,
      similar-task bodies and the masking invariant without implementing the new
      task_type completion menu owned by bob-cli-ee.3."
  - id: writes
    title: New-task type writes, agreement, output and documentation
    size: medium
    depends_on:
      - editor
    description:
      "writes: enable type markers for newly created tasks, stage the resolved field
      before placement, enforce named-Pomodoro agreement, report set/cleared/kept and
      unconfigured warnings, retain existing-target refusals, update help and
      documentation, and verify the complete bob-cli-ee.1 acceptance matrix."
proposed_by: bbugyi200.apollo.bob-cli-ee.1
proposed_with: codex/gpt-6-astra
parent_bead: bob-cli-ee.1
create_time: 2026-10-30 04:33:09
status: wip
---

- **PROMPT:**
  [prompts/202610/capture_type_grammar.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202610/capture_type_grammar.md)
- **PARENT:**
  [202610/capture_type_marker.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/capture_type_marker.md)

# Capture type-marker grammar and new-task typing

## Ownership and fixed decisions

This child epic implements only the `grammar` phase assigned to **bob-cli-ee.1** under
**bob-cli-ee**. Its governing design is `plan:202610/capture_type_marker.md`, especially
sections 1, 2.1–2.2 and Phase `grammar`. Read that artifact through `sase artifact read`
and read the assigned bead before implementation. Link this proposal under bob-cli-ee.1
through the normal plan-approval workflow; do not hand-create beads.

The parent decisions are final: `unknown_types = warn`, `menu_order = config`,
`memory_record = no`. No memory file may be created, changed or deleted. Record the
skipped decision note as `PROPOSED FOLLOW-UP:` on bob-cli-ee.1; if the note already
exists, do not duplicate it. The menu order is relevant to the later completion phase,
not work to implement here.

Existing-target retyping and D8 relocation belong to bob-cli-ee.2. The new `task_type`
completion context, candidates and zsh rows belong to bob-cli-ee.3. Mac rendering
belongs to bob-cli-ee.4. This epic prepares their shared grammar seam without
implementing those phases. Existing-target marker forms must be recognized and refused
honestly until ee.2 implements them.

Only the child epic's land agent closes bob-cli-ee.1 after all work below is verified.
Child phase workers close only their assigned child phase. Never close bob-cli-ee or any
other ancestor. Follow-ups go in `PROPOSED FOLLOW-UP:` notes, not newly created task
beads. Before every close, run `sase bead epic-symbols` for that bead and resolve or
re-key outstanding entries to a still-open owner.

## Current implementation and why three phases

At inspection, HEAD was `3b0ffa0a`, the checkout was clean, bob-cli-ee.1 was in_progress
without previous-work notes, and no type-marker implementation had landed. Existing
task-type field writers and typed Pomodoro placement already exist. Recheck these facts
before coding so concurrent landed work is reused.

The hard boundary is parsing: `capture_language/item.rs` and `editor_parse.rs` each have
many early whole-item claims, while `draft.rs` splits URL lists and session chains
before those claims. Their outcomes borrow global-declaration tokens from the input. A
late token-removal patch would misclassify whole-item forms and invalidate byte offsets.
The first phase owns that shared source representation. The second integrates editor
consumers. The third uses the established type writers and staged transaction planner.

Every phase must compile and pass its relevant tests on its own. Keep the phase graph
sequential because these phases share model and parser files. Do not weaken lint gates
to cover unused scaffolding; expose only helpers used in the current phase, then extend
them as their callers arrive.

## Contract to preserve

- Lowercase `t:` claims a token at either edge of any physical line, including an
  authored child bullet's body after its prefix. Between it and that edge may stand only
  existing trailing-transparent markers (`@…`, `@@…`, bare `#`, `s:`, `s:N`,
  priority/escaped-priority and clipboard markers), complete or partial `&` / `*` / `~`
  / `~~` modifiers, or other type markers. Uppercase `T:` and mid-line `t:` stay prose.
  Respect protected code and wikilink spans and quoted modifier contents rather than
  finding embedded tokens in them.
- Slugs contain ASCII letters, digits and hyphens, starting with a letter or digit.
  Literals reuse `pomodoro_literal.rs`'s straight/curly quotes and quote/backslash
  decoding. A literal spans through its closing quote. Empty or unterminated names are
  incomplete; composites containing `+` and invalid remainders are errors, never
  silently recaptured as text.
- An edge `\t:…` keeps `t:…` in the body with only the escape removed. Mid-line `\t:`
  stays verbatim. Track escape positions like priority escapes; do not globally replace
  strings or shift the parser's source ranges.
- Exactly one type marker is allowed per item across all its lines. A later bare `t:fix`
  or `- t:fix` types its existing item without contributing a child bullet. It does not
  become a blank-line separator or a new task.
- All pre-existing claims see the same item with marker bytes replaced by ASCII spaces,
  keeping UTF-8 byte lengths and line endings intact. Item boundaries and source
  positions refer to the original draft. Every valid masked item's kind, body,
  destination, other markers and semantic spans retain their pre-marker meaning.
- The marker types only the subject task. New ordinary/routed/forced-route,
  explicit-block-id, `-R` URL-as-task, template and Pomodoro-linked tasks are supported.
  Dependency, blocked-task and relation targets never receive it. Existing link, Ensure
  Next/toggle and relationship-owner subjects remain temporarily unsupported in this
  phase.
- Plain bullets, sub-bullets with text, project notes, queued references, Pomodoro
  notes, completion/close/start/adjust/shift/chain gestures have no eligible subject. A
  marker alone, even above children, needs task text. Preserve the masked item's own
  malformed-grammar error; if it parses as an ineligible kind, report the type-marker
  applicability error.

Use the parent's exact teaching copy and codes: need `task_type`, `invalid_type_marker`,
`duplicate_capture_marker` with label `type marker (t:<type>)`,
`type_marker_needs_task`, `type_marker_not_applicable`, and writer
`type_marker_conflict`.

## Phase language

Primary files: new `src/native/capture_language/type_marker.rs`, plus `mod.rs`,
`model.rs`, `draft.rs`, `item.rs`, `template.rs`, relevant tests, and a small temporary
guard in `capture/plan.rs`.

1. Build one lexical scanner returning complete, incomplete, invalid and escaped
   occurrences with owned decoded names, literal flag, original token range and
   name-content range. Reuse quote decoding, protected ranges and modifier scanners.
   Make leading/trailing transparency a shared predicate; account for multi-token quoted
   literals and modifiers before walking edges. Collect duplicates across the whole
   logical item.
2. Introduce owned backing storage for the byte-preserving masked view and construct
   borrowed `CaptureItem`/`RawLine` views over it only while that storage lives. Do not
   leak buffers or return references to local strings. Keep original metadata alongside
   the masked representation. Returned declarations must borrow original text
   reconstructed by their unchanged ranges (or become owned consistently); they cannot
   borrow the temporary mask. Audit both `item.source` slicing and `line.raw.text`
   callers.
3. Group blank-line items and recognize authored line boundaries from the original draft
   first. Keep marker-only continuation lines attached to their owner and suppress them
   as semantic child bodies. For URL-list and session-chain/inline-close split probes,
   use masked line views so `t:` does not turn a URL list or operator chain into prose.
   Map their resulting ranges back to original text. A marker on a URL line belongs to
   that URL; a marker-only continuation belongs to the preceding subject. For a chain,
   retain ordinary operator splitting and carry its marker onto an operator item so the
   batch refuses rather than executing a partial chain. Pin the attachment and original
   ranges with tests; never re-split the entire masked string on newly empty lines.
4. Wrap `parse_capture_item` before every whole-item claim, then parse the masked view
   through the existing logic. Add `type_marker: Option<TypeMarker>` to
   `ParsedCaptureText` with owned name, literal flag and span metadata, setting every
   constructor explicitly. Resolve applicability after masked kind/subject and global
   destination inheritance are known. Empty parent plus children remains needs-task.
   Centralize marker error construction for the editor to reuse.
5. Apply the same preparation when template rows enter parsing. Preserve the
   expansion-to-authored source map; a marker inside a template belongs to its expanded
   row. Markers must not pollute template-token recognition. Do not invent draft-wide
   `@@t:` or template invocation propagation semantics.
6. Until phase writes is complete, the planner must reject marker-bearing eligible new
   tasks explicitly as not yet supported instead of silently discarding their type.
   Preserve ordinary captures. Permanent non-task and existing-target errors already use
   the normative codes.
7. Add `capture_language/tests/type_marker.rs` and a separate fixture under
   `tests/fixtures/capture_type_marker/`. Cover both edges, marker runs, escapes,
   literals, duplicates, children, continuation lines, bulk items, templates, URL lists,
   chains, Unicode and CRLF. Test malformed masked forms retaining their old errors and
   valid ineligible forms receiving the new teaching error. Run the capture-language
   tests and `just check`.

## Phase editor

Primary files: `capture_language/editor_model.rs`, `editor_parse.rs`, `completion.rs`,
`capture_parse.rs`, the shared type-marker module and tests.

1. Use the prepared masked view before `parse_editor_item_with`'s early `/`, `:`, `!`,
   session, parent-search and reference claims. Carry marker metadata through every
   return. Add `SpanKind::TaskType` and `Need::TaskType`; incomplete markers put the
   item in incomplete mode. Integrate duplicate handling with `SeenMarkers` and reuse
   the execution teaching messages. Preserve original source offsets and original
   displayed input.
2. Mask type markers before the existing field detector's global/whole-item claims,
   close-log checks and route/modifier detection. Reuse the original item topology and
   source map. At a cursor on `@sa` in `Fix it @sa t:fix`, route completion still
   returns the original replacement range. Cover `:query`, `/query`, `!query`, `+query`,
   `^r:id` and child-line neighbors. Do not add `CompletionContext::TaskType` or menu
   candidates here; ee.3 consumes the exported scanner to implement that context.
3. Implement a shared exact type-name resolution helper suitable for both editor lookup
   and the writer: configured vocabulary first, then catalog; literal normalized-key
   equality; slug key equality before substituting spaces for hyphens. Never resolve
   prefixes. Preserve configured/catalog spelling, otherwise use `canonical_theme_name`.
   Test vocabulary/catalog collisions and `deep-work` versus `DEEP WORK`.
4. Generalize `capture_parse.rs` catalog loading and `attach_theme_refs` for
   `task_type`. Whole-marker spans include sigil and quotes, with one ThemeRef over only
   the source name content. Do not split on `+` as Pomodoro-name spans do. Use decoded
   content for identity but authored byte ranges for highlighting, including escaped
   characters and curly quotes. Incomplete or invalid names get their
   span/need/diagnostic without fabricating a resolved type. Unknown complete names
   carry appropriate known/configured flags and existing hue policy. Keep
   schema_version 1.
5. Verify `body` excludes markers everywhere, including `capture-related`. Add a
   `capture-rewrite` regression proving separator toggling leaves `t:` tokens intact.
6. Build a table-driven property test over existing valid grammar families: insert
   markers at legal leading/trailing/child positions and compare the masked result with
   the baseline after removing marker-specific metadata. Compare ordinary spans by
   translating insertion offsets, not by assuming numeric offsets are identical between
   differently sized input strings. Keep separate assertions for expected new needs and
   applicability errors. Verify execution/editor parity and template source-map ranges.
   Run focused language, capture-parse, rewrite and completion tests, then `just check`.

## Phase writes

Primary files: `capture/plan.rs`, `capture/task_theme.rs`, `capture/pomodoro_link.rs` as
needed, `capture/batch.rs`, `capture/output.rs`, `capture/cli.rs`, tests and the
documentation listed below.

1. Replace the temporary new-task refusal with staging. Resolve the explicit type using
   the shared helper and loaded vocabulary/catalog. In the batch's
   `CaptureTaskThemeContext`, compute inherited suggestion I from the staged formatted
   task line in its actual destination, skipping its own type field. Honor reading
   precedence and `[root::]`, forced/global routes and earlier staged note changes. Use
   `set_task_theme`/`clear_task_theme` to preserve suffix metadata; do not implement a
   second field rewriter.
2. Apply the parent's field table: X equals I means no field for an untyped new task
   (`kept`) or removal of an authored exception (`cleared`); otherwise a valid field
   already equal to X is `kept`, and any other state is `set`. A marker always reports
   `{action, theme, previous, inherited}`. A kept result includes the effective ThemeRef
   and the right source label.
3. Apply the field before assembling the capture block and before bare Pomodoro
   placement reads its task line. Cover `format_task_line`,
   `format_task_with_block_id_line` and `format_pomodoro_task_line`, including tasks
   with dependency/relation augmentation. No block ID should be minted solely to type a
   plain inbox task. Ensure all later preview/task-block snapshots reflect the edited
   line.
4. Resolve an explicitly named destination with the current name-selection rules. If it
   is a container, accept X; if X is a component, accept X as the explicit pick;
   otherwise refuse with `type_marker_conflict` before commit. The explicit marker is
   the authority: prevent the later `apply_new_task` named-retype path from overwriting
   or clearing it. Apply this to open matches, completed-again and newly created names
   without changing destination selection. Name resolution may be staged; no vault bytes
   can be committed before agreement succeeds.
5. Unknown names proceed with canonical spelling and the existing structured
   `theme_not_configured` warning and similarity copy. Keep warning handling consistent
   with existing typed placement; do not implement the rejected refusal decision.
   Type-only changes introduce no freshness stamp, lane change or dependency guard
   consumption; the enclosing capture's existing gesture effects remain intact.
6. Preserve the temporary existing-target refusal for ee.2 in one obvious seam with an
   honest message. Permanent non-task refusals must identify the gesture and the
   parent's remedy. Dry-run and real-run use the same planner and agree except for
   `dry_run`; conflict in any item rolls back the batch.
7. Add the human `kept` row: `type   ■ FIX · <label>` with `this task`, `from <stem>`,
   `default type` or `reading task` as appropriate. Preserve existing `set`/`cleared`
   output and legacy no-marker behavior.
8. Add `tests/cli/capture/type_marker.rs`, registering it in the test module. Test
   inbox, routed, forced/global-route, explicit ID, `-R`, templates, linked tasks,
   inherited/root/reading suggestions, set/cleared/kept,
   configured/collision/unknown/no-prefix resolution, modifiers typing only their owner,
   blocked creations, named containers/composites/conflicts, every refusal family,
   Unicode ranges, batch rollback and dry-run equality. Reuse the temporary vault/config
   patterns in `task_theme_writes.rs` and `typed_placement.rs`; never write the real
   vault.
9. Read docs memory before editing. Update `docs/capture.md` with the grammar table,
   edge rule, a `Typing tasks with t:` section after scheduling and priority, errors,
   editor spans/needs and task-type change output. Correct the stale `type` JSON example
   key to the actual `theme`. Update `docs/reference/capture-syntax.md` with
   `### t:<type> {#t-type}` and `docs/task-types.md` PT13 with typed captures and the
   writer boundary. Document existing-target support as pending this epic's later ee.2
   phase; do not claim menu functionality that ee.3 has not landed. Update capture and
   capture-parse marker help, then run `just docs-gen` and review generated changes. Do
   not change shared pomodoro theme vectors, versions or changelog.
10. Run focused capture regressions, `just check`, `just all` and `just docs-check`,
    using SASE monitor for long checks when necessary. Record exact verification
    evidence on the assigned child phase.

## Land and completion

The land agent reconciles the three phases and audits the full grammar-phase checklist
from the parent design. Re-run required gates on the integrated tree; retain
schema_version 1, shared vectors unchanged, and no memory edits. Inspect
`git diff --check` and ensure temporary new-task refusals are gone while the intentional
existing-target refusal remains. Record the public helper/model names so ee.2 and ee.3
can consume them.

Check failures that reproduce identically on the clean base do not strand the phase.
Record a `PROPOSED FOLLOW-UP:` with reproduction and any existing task bead ID, then
finish the otherwise verified work. Never create task beads from this phase. Use
approved repository access for any separate checkout.

After child phases are complete, run `sase bead epic-symbols bob-cli-ee.1`,
resolve/re-key leftovers, then close only the original assigned phase with
`sase bead close bob-cli-ee.1 --note "<implemented contract and verified checks>"`. The
child epic's own completion follows its normal land workflow. Do not close the outer
epic bob-cli-ee. The original phase stays in progress during this planning handoff; its
status is runtime-owned.
