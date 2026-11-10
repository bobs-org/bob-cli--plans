---
tier: tale
title: Complete notebook capture assistance
goal:
  Finish notebook completion, note-bullet ID assignment, and the approved global-route
  collapse assist for bob-cli-j8.5.
size: medium
proposed_by: bbugyi200.apollo.bob-cli-j8.5
proposed_with: codex/gpt-6-astra
bead: bob-cli-j8.5
status: done
---

- **PARENT:**
  [202611/bob_notebooks.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/bob_notebooks.md)
- **BEAD:**
  [bob-cli-j8.5](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-j8/bob-cli-j8.5.md)
- **AGENTS:**
  - [bbugyi200.apollo.bob-cli-j8.5](https://github.com/bobs-org/bob-cli--agents/blob/main/sessions/bbugyi200.apollo.bob-cli-j8.5.md)
- **COMMITS:**
  - [93cf930](https://github.com/bobs-org/bob-cli/commit/93cf930ca2a8221bdbe4b1dad602ef49669f3f18)
    — feat(capture)\!: complete notebook capture assistance

# Complete notebook capture assistance (bob-cli-j8.5)

## Objective and authority

Finish the bob-cli capture protocol for notebook routes: section choices and creation,
named-note suggestions, selecting and assigning IDs to note bullets, the notebook
creation directive, and the lexical `@@route` collapse assist. Complete and close only
`bob-cli-j8.5` after verification.

The binding design is `plan:202611/bob_notebooks.md`, especially
`capture-notebook-assist`, the wire additions, and compatibility sections. Its research
is `research:202611/bob_notebooks/bob_notebooks.md`, Capture and findings V1–V9. Read
both through `sase artifact read` and the phase through
`sase bead read bob-cli-j8.5 -r "Need current scope and implementation notes"`. The epic
takes precedence over the research: `/` and `\` both participate in collapse, and ID
assignment extends the existing `capture-task-id` command.

Inherited, final choices are `collapse_assist = yes` and
`memory_glossary_corrections = yes`. The latter already authorizes the exact
`glossary:capture-alias` correction below; do not ask the reviewer to choose again. The
other accepted memory changes belong to the epic's wrapup. Application UI, plugin
changes, vault migration, and notebook extraction belong to other phases.

This is one medium coding task: the parser, notebook writer, catalog, route policy, and
creation transaction already exist. There are no prior-work notes on this phase and no
epic-symbol entries as of planning. Recheck current notes and source before editing,
preserving any newly landed assistance work.

## Existing implementation to extend

- `12692ebb` and its follow-up `843642f9` already landed notebook capture targets,
  metadata, human/shell presentation, and route completion tests. The opt-in scanner is
  `capture_targets::scan_capture_targets_with_notebooks`; the base scanner must continue
  excluding notebooks from task-only consumers.
- `ab021e2d` implements `@parent^id@ %n`, including provisional notebook policy for
  routes created earlier in a draft. Preserve its parsing and transaction semantics.
- `capture_complete/engine.rs` builds the notebook policy, but section candidates still
  use ordinary section matching and note-scoped `@route+` still calls `task_candidates`.
  This is `CompletionContext::Task`, distinct from the vault-wide bare-plus `TaskParent`
  picker.
- `capture_language/notebook.rs` provides `notebook_sections` and
  `resolve_notebook_section` with Existing/Create/NearMiss outcomes. Reuse them.
- `capture/notebook_note.rs` already defines which bullets may be `@nb+id` parents,
  rejects duplicate IDs, and skips fenced examples. Share that eligibility logic with
  completion and ID assignment rather than defining another Markdown dialect.
- `capture_block_ids.rs` has suggestions and `collect_used`, including non-task anchors,
  but its body extraction uses an editor parse without notebook options.
- `capture_task_id.rs` already accepts a stale-safe `line:digest` ref, validates IDs,
  preserves line endings, and returns `parent_replacement`. Its assignment path
  currently scans only tasks.
- `capture_language/rewrite.rs` has a cursor-only rewrite chain before bare `@@`
  absorption. There is no `collapse_global_destination` rule yet.

## Implementation

### 1. Finish route and section choices

Retain notebook route ordering, canonical replacement, kind, title, summary, owner, and
breadcrumb. Finish the epic's presentation contract in
`capture_complete/{model,candidates,render,shell}.rs`: notebook rows have a 📓 badge,
the breadcrumb as the displayed label, and summary as detail. Keep the route and file
identity separate from presentation and preserve ordinary route serialization. Reuse
existing metadata instead of rescanning for every row. Existing terminal and shell badge
support should be retained and tested rather than rebuilt.

For a notebook `@nb#` field, derive only real H2 sections from the same notebook
policy/snapshot as parse. Filter by the query while retaining document order; never
offer the H1 title, H3s, or fenced headings. Ordinary area/project section prefix
matching and the record-create row remain unchanged.

For a nonempty query that has no exact title/whole-slug match, append one
`New section “Query”` row. Compute its proposed title and selector with
`resolve_notebook_section`. Use a bare slug when the resolver would create it; use a
quoted selector when an unquoted slug would be a near miss. Preserve authored quoted
titles. Do not offer an empty section create. Represent the create row additively in
`SectionMenuRow`, distinct from `record_create`, and handle it in every renderer. Apply
the response's replacement range to prove the resulting draft has exactly one `#`,
retains any `^id` suffix, and is accepted by dry-run with the advertised section title.
Continue using the shared resolver at commit time.

### 2. Make named-note ID suggestions notebook-aware

Thread the already-built notebook parse context into the block-ID builder/body
extraction. For `@nb^` and `@nb#section^`, keep `intent: "new"`, and add optional
`destination_kind: "notebook"`. Preserve marker ranges, allowed alphabet, body, used
IDs, and suggestions. Leave the new field absent for ordinary task fields. Do not
introduce a new intent or change the sealed project-note intent.

Suggestions must derive from the note's text without route, section, or directive
syntax, and avoid every actual ID in the file, including headings and plain bullets. Use
`collect_used` and the established suggestion engine. Keep draft-created notebooks
recognized by the existing policy; do not write files during completion.

### 3. Select notebook note bullets and assign missing IDs

Add a notebook-specific candidate branch only for note-scoped `@nb+`. Keep bare `+`,
task links, dependencies, relations, and task search on their existing task-only
scanners. Extract a small shared notebook bullet scan/lookup helper from the writer as
needed, used by completion, the existing sub-note lookup, and assignment.

Eligible rows are note bullets at any depth under the current writer's rules. Exclude
checkbox lines, headings, fenced examples, and prose. Surface duplicate target IDs as
refusals or disabled rows, never selectable ambiguous destinations. Show ID'd matches
first, then ID-less matches, with stable document order within those groups and the
existing text/ID query behavior. ID-less rows are offered in this notebook picker
without making the user request a task-only flag.

Carry additive `item_kind: "note"`, route, text, line, section, depth, child count,
`ref`, `block_id`, `requires_block_id`, and `block_id_suggestions` using the existing
candidate envelope. For existing IDs, replacement is the ID for the established
note-scoped range. ID-less rows require assignment before insertion. Keep the existing
task-row fields/serialization compatible; note handling must not cause task status,
ledger, or relationship operations. The Mac presentation is handled by `bob-cli-j8.8`
using the note discriminator.

Extend `capture_task_id::plan_assignment` after resolving and reading its target:
recognize actual notebook frontmatter, resolve the candidate's `line:digest` against the
same eligible bullet snapshot, and append the chosen ID only to that bullet. Reuse
`note_tasks::line_digest` and the existing ref semantics: exact line+digest first,
otherwise exactly one matching digest after a line shift; stale or ambiguous matches
refuse. Keep existing task assignment behavior and command/flag names.

For notebook assignment, reject an existing ID, a duplicate chosen ID anywhere in the
note, a notebook retyped since selection, and non-note target lines. Preserve
read-only/path guards, CRLF/BOM, indentation, trailing text, and unrelated bytes.
Dry-run must return the same proposed result without writing. Return the resolved
line/ref and backend-authored `parent_replacement: "@nb+ID"` so the client can resume
the selection; identify the notebook note additively in the result and document its
shape. Preserve the old task success shape. Do not make note rows eligible for task-only
commands by synthesizing a task in the shared task scan.

### 4. Offer the correct `%n` directive

In `capture_complete/directives.rs`, classify the candidate draft using the same parser
and notebook policy as strict capture. On a complete caret-sealed notebook creation
shape, enable `%n` with notebook-creation name, summary, and detail. Do not depend on
today's Pomodoro availability for this row. A draft with an empty summary may still
choose the directive; the normal creation validator then teaches the required summary.

Keep the existing Pomodoro-note row and availability checks for ordinary `%n`. Retain
refusals for colon seals, incomplete caret seals, `#name` after the seal, duplicate
directives, and other invalid combinations. Cover leading and trailing seals and parents
that are areas, projects, or notebooks. Avoid broad error-code suppression merely to
enable a menu row.

### 5. Implement the approved lexical collapse

Add `RewriteRule::CollapseGlobalDestination` with wire code
`collapse_global_destination`. With an explicit cursor immediately after a trigger glued
to a complete `@@route` satisfying `is_route_token`, return a single native edit and
mapped UTF-8 cursor as follows:

| Trigger       | Replacement |
| ------------- | ----------- |
| `#`, `/`, `\` | `@route#`   |
| `^`, `.`      | `@route^`   |
| `+`, `,`      | `@route+`   |

This works for every syntactically valid route without reading the vault. Run it ahead
of the dot fallback and ordinary aliases, under the existing template guard. It is its
own rewrite rule, not `expand_sigil_alias`, so it does not emit an alias stamp or a
misleading alias tip. Preserve the response edit protocol: applying the edit yields the
returned text, and Undo can restore the exact typed input.

Require the trigger at the caret and the complete token boundary. Do not collapse bare
`@@`, spaced triggers, invalid routes, already-suffixed globals, escaped or quoted
prose/code, or a trigger earlier than the caret. Preserve template and exact-name
exclusions and the existing bare-declaration absorption behavior. `capture`,
`capture-parse`, pasted text, and no-cursor rewrite calls retain strict grammar; this
assist does not legalize suffixed globals. Update tests that currently pin `@@r.`,
`@@cash,`, and `@@cash/` as literal cursor-typing cases.

### 6. Documentation and the inherited memory correction

Follow `cli_rules.md` and `docs.md` through `/sase_memory_read` before editing help or
documentation. Update `docs/capture.md` under capture-complete, capture-rewrite, and
capture-task-id; document the additive JSON fields, section-create row, assignment
response, and rewrite rule in `docs/reference/json-outputs.md`. Link to
`docs/notebooks.md` as the contract hub and add a concise completion pointer there.
Update companion CLI help that now incorrectly claims task-only assignment or omits the
rewrite, then regenerate its reference fixtures with `just docs-gen`. Keep schema
version 1 for these additive protocol changes; preserve the already landed strict
capture grammar revision. Do not edit versions or CHANGELOG.md.

Use `/sase_memory_write` for the inherited `memory_glossary_corrections = yes` change to
`glossary:capture-alias`: remove `@@` routes from the list where `/` stays literal, and
name the `@@` collapse beside `@@` absorption among rewrites that are not aliases. This
corrects the current claim
`` `/` stays literal where it is already field text (`=/`, `[[folder/`, `@@` routes). ``
once the approved collapse ships. Apply exactly that correction and run
`sase memory init` to republish generated instructions. Cite its completion in the phase
note so the epic wrapup can recognize that this particular correction landed.

## Validation and completion

Add focused unit/CLI tests using temporary vaults, fixed `BOB_NOW`, and isolated
configuration. Register new test modules in the existing module files.

- Route rows retain canonical replacements and expose the notebook label/detail; the
  task-only scanner still excludes notebooks.
- Sections: H2-only/document-order output; exact title and slug; near-miss create
  quoting; unrelated new slug; quoted title; empty query; `#section^id`; fences;
  ordinary area/project and record section regressions. Apply candidates and verify
  dry-run/commit agree with their advertised action.
- IDs: `@nb^` and `#section^` remain new, expose notebook kind, and avoid both task and
  non-task IDs; ordinary fields remain compatible. Check Unicode body/cursors.
- Parents: nested and ID-less bullets, duplicate target IDs, excluded checkboxes,
  headings and fences, filtering/order, and an end-to-end select → assign ID → capture
  child flow. Verify no daily ledger or task metadata is changed.
- Assignment: dry-run no-op; actual write; stale and shifted refs; ambiguous equal
  lines; already assigned and colliding IDs; retyped target; read-only file; CRLF, BOM,
  and existing task-assignment regressions.
- `%n`: complete caret seals enabled even without a Pomodoro; ordinary Pomodoro
  availability preserved; invalid seals/duplicates remain refused.
- Collapse: all seven triggers on short/underscored/hyphenated routes, leading and
  trailing positions, non-ASCII surrounding text, exact edits/cursor, no-cursor and
  non-firing cases, bare `@@` absorption, ordinary aliases, and alias-tip regressions.

Run focused tests while implementing, then `cargo fmt --all`, `just check` (the full
fmt/clippy/tests gate), and `just docs-check`. Run `just docs-gen` before the
documentation gate whenever help changes. Use `/sase_monitor` for long commands and wait
for each handoff command itself to exit before ending a turn.

Record results on `bob-cli-j8.5`. For a failure, identify whether this change caused it;
if it reproduces identically on the clean base, append
`PROPOSED FOLLOW-UP: <summary — reproduction and evidence, including any existing task ID>`
to this phase, as the user instructed, and do not leave a completed phase open for that
unrelated failure. Record any other discovered work the same way; create no task beads.

Before closing, run `sase bead epic-symbols bob-cli-j8.5`; resolve its symbols or re-key
surviving justfile entries to a still-open appropriate epic/phase. Then run
`sase bead close bob-cli-j8.5 --note "<implemented behavior and verified results>"`.
Never close the parent epic or an ancestor plan bead. Finish the provider turn with
`/sase_final`; the planning turn keeps the phase open while handing this tale off.
