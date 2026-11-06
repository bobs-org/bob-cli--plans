---
tier: tale
title: Capture merge-in grammar and editor contract
goal:
  Complete bob-cli-hl.2 with shared merge-in parsing, teaching rewrites, grammar
  revision publication, and a safe execution refusal until the execute phase lands.
size: medium
proposed_by: bbugyi200.athena.bob-cli-hl.2
proposed_with: codex/gpt-6-astra
bead: bob-cli-hl.2
status: done
---

- **PARENT:**
  [202611/capture_merge_in_arrow.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/capture_merge_in_arrow.md)
- **BEAD:**
  [bob-cli-hl.2](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-hl/bob-cli-hl.2.md)
- **AGENTS:**
  - [bbugyi200.athena.bob-cli-hl.2](https://github.com/bobs-org/bob-cli--agents/blob/main/sessions/bbugyi200.athena.bob-cli-hl.2.md)
- **COMMITS:**
  - [53c1d1a](https://github.com/bobs-org/bob-cli/commit/53c1d1ad4cbff2d8567718deb2175b887ebdb686)
    — feat(capture)\!: complete merge-in grammar and teaching refusals

# Capture merge-in grammar and editor contract

## Objective and scope

Complete phase **bob-cli-hl.2** of `plan:202611/capture_merge_in_arrow.md`. Recognize
`New task < @a+x @b+y` as one new-task item carrying an explicit merge-in clause,
consistently in execution parsing and editor parsing. Preserve the new task's own kind,
route, fields, and children. Teach incomplete and invalid input with the approved codes
and explicit draft rewrites. Until phase bob-cli-hl.3 supplies execution, refuse every
complete merge-in item before any vault mutation with `merge_in_unavailable`.

This is one medium implementation tale: one shared scanner, two parser adapters, wire
serialization, a planner refusal, and focused regression coverage. The many
`ParsedCaptureText` and `Diagnostic` initializers need mechanical optional field
additions; they do not represent additional feature work.

The approved parent epic is authoritative. Read it through `sase artifact read`, and
read this phase through `sase bead read` before starting implementation. Do not re-ask
its final decisions: `pull_in_title_count = yes`, `memory_task_merge_glossary = yes`,
`memory_merge_in_decision = no`. This tale changes no memory. The authorized glossary
edit already landed in the contract phase. Execution, source resolution against the
vault, merge result/output fields, shell-redirection detection, picker rows, Similar
tasks, the `+` rewrite toggle, and the Mac app remain their existing later phases' work.

## Verified starting point

Planning inspected clean HEAD `d9e35359`
(`docs(capture): contract for merge-in < capture`). Phase bob-cli-hl.1 is closed; its
docs already contain the claim and context tables, error copy, editor fields, and MI
vectors. bob-cli-hl.2 is in progress with no previous implementation/remainder notes and
no `--epic-symbol` entries. No merge-in Rust implementation exists, and
`capture_language::GRAMMAR_REVISION` is 14.

Before coding, refresh these observations and preserve any newer work. The concurrent
bob-cli-hj epic also touches grammar and rewriting. If it has taken revision 15, choose
the next free revision and use that number consistently. Do not redo the contract phase
or reopen an already in-progress bead.

Relevant implementation seams:

- `src/native/capture_language/transfer.rs`: pure span-aware scanner and source token
  helpers; current helpers return `TransferScan` errors.
- `capture_language/draft.rs`: `parse_capture_item_members` tries transfers before
  legacy target-run fan-out; `capture_item_source_view` and
  `rebind_capture_item_declarations` preserve original offsets and lifetimes. The test
  singleton entry point `parse_capture_text_with_options` has its own transfer-claim
  branch, and templates call the member parser after expansion.
- `capture_language/editor_parse.rs`: `parse_editor_item_with` dispatches transfers
  before target-run parsing and already constructs masked views.
- `capture_language/model.rs` and `editor_model.rs`: parsed task, editor item, span,
  need, and diagnostic types.
- `src/native/capture_parse.rs`: separate result/item wire structs and `wire_item`
  conversion; template instances remap diagnostic ranges. Adding a field only to
  `editor_model.rs` will not publish the JSON contract.
- `src/native/capture/plan.rs::plan_capture_item`: early planner dispatch and the
  temporary no-write refusal.
- `capture_language/completion.rs` and `rewrite.rs`: direct transfer scanner callers
  that currently fall back to ordinary target-run behavior.

## Implementation

### 1. Shared source parsing and merge-in scanner

Extract `is_source_like`, `parse_complete_source_text`, `parse_source_token`, and
`find_lexical_duplicate` into `capture_language/task_sources.rs`, using shared
source/span and lexical-error types that do not depend on either scan enum. Keep
compatible transfer exports/adapters where needed. Preserve the existing `>` codes,
messages, ranges, and behavior, including its escape rule. Reuse the existing
destination parser for the one-token `Use >` rewrite instead of inventing a second
definition of a valid destination.

Add `capture_language/merge_in.rs`, register it in `mod.rs`, and implement
`scan_merge_in_line(parent, source, has_children) -> MergeInScan` with `Unclaimed`,
`Escaped`, `Incomplete`, `Valid { arrow, sources }`, and
`Invalid { code, message, span, rewrite }`. Keep absolute half-open UTF-8 byte ranges.
The scanner owns the claim rules, copy, and lexical rewrites:

1. Scan left to right for the first standalone `<` with a claimable following token: end
   of line, `@route`, a source-like `@` token containing `+`, `:`, or `^`, or a
   `+`/`+query` selector. A prose comparison leaves that arrow unclaimed and scanning
   continues, so a later real clause can still claim.
2. Consume complete `@route+id` sources in typed order. A final bare route, `@route+`,
   or `+query` is incomplete and preserves complete sources already read. Include the
   route-prefix typing state `< @` without letting it fall through to unrelated parsing.
   Terminal `<` is incomplete too.
3. Preserve `transfer_source_marker` for `@route:id`/`@route^id` and
   `transfer_source_suffix` for `#drawer`/`!` suffixes. Duplicate note-and-ID pairs,
   with lowercased routes, use `duplicate_capture_target` at the second occurrence.
   Resolved alias duplicates belong to the execute phase.
4. Other tokens after the sources, a non-final partial, or another `<` produce
   `merge_in_extra_text`. Offer `Move before <`: relocate extra authored tokens before
   the first arrow, retain ordered sources, and drop a stray second arrow. Scope the
   edit to the affected parent line, preserving list prefixes, children, and unrelated
   items.
5. Claim `text<@a+x`, `text <@a+x`, and `text< @a+x` as `merge_in_arrow_spacing` with
   `Add spaces`. Only claim glued near-source shapes; `<=`, `=>`, `<3`, `<<`, and
   ordinary comparisons remain prose.
6. Respect protected markup at the candidate arrow/source ranges: code, wikilinks, and
   autolinks do not form clauses. Protected content elsewhere on a valid task line must
   not disable its later unprotected clause. Do not scan authored child lines as new
   directives.
7. `\<` at the arrow position escapes the **entire rest of that parent line**. Strip
   only that escape backslash; preserve `<` and every subsequent token as literal task
   body text. Nothing in the escaped tail routes, adds markers, or fans out. Parse the
   prefix normally. In particular, `Escaped \< @foo+f @work` creates the inbox task body
   `Escaped < @foo+f @work`. Do not copy `\>`'s narrower semantics.

### 2. Execution adapter and shared context checks

Add `Option<MergeInClause>` to `ParsedCaptureText` containing the arrow and ordered
sources; initialize it to `None` for all existing constructors. Do not introduce a new
`CaptureKind` for merge-in.

After `parse_transfer_claim` returns unclaimed, run the merge-in claim before legacy
fan-out. Skip claiming under forced `-r`/`--task`/`--drawer` destinations, consistently
with transfer parsing. For valid and pending clauses, replace the clause bytes with
equal-length spaces and parse the left side using the ordinary parser. Use source views
and declaration rebinding so masked storage does not leak into borrowed data or original
byte offsets. Do not recursively invoke the same claim on its masked input.

Apply the same context decision to the execution and editor representations:

| Left side result                                                                   | Required handling                                                       |
| ---------------------------------------------------------------------------------- | ----------------------------------------------------------------------- |
| One `Task` or `TaskWithBlockId`                                                    | Allowed                                                                 |
| One `Pomodoro` with neither `start` nor `close`                                    | Allowed; stays a linked new task                                        |
| Task-list bullet under a lone `@route` header                                      | Allowed per bullet with inherited route                                 |
| `@@` inheritance, `%c`, or an expanded `%t` row creating one open task             | Allowed                                                                 |
| An empty left side                                                                 | `merge_in_needs_new_task`: name the new task before `<`                 |
| Existing task, target run, or dependency-only item                                 | `merge_in_needs_new_task`                                               |
| Start or born-closed task                                                          | `merge_in_needs_new_task`: keep it open and start/close in a later item |
| Project creation or its project-task children                                      | `merge_in_needs_new_task`: use project move bullets                     |
| Note/section bullets, refs/URLs, session operators, or any other non-task creation | `merge_in_needs_new_task`: this line creates no task                    |

For the existing-destination refusal, offer `Use >` only when the entire trimmed left
side is one destination accepted by the existing transfer parser: `@n`, `@n+id`,
`@n+id#drawer`, or `@n#section`. Thus `@n+t < @a+x @b+y` rewrites to `@a+x @b+y > @n+t`
and `@db < @a+x` to `@a+x > @db`.

Retain normal parsing of authored children/drawers and left-side `&`, `*`, `~`, `~~`,
schedule, priority, and task-type fields. Refuse a lexical match between a source's
lowercased note plus ID and a prerequisite, dependent, or relation target with
`merge_in_source_also_marked`, naming the relevant role. Configured merge-label
detection and resolved aliases remain execute-phase checks; lexical target equality is
this phase's MI14 coverage.

Attach the clause only to the single valid new-task parse. Incomplete submission returns
`incomplete_merge_in` with the approved “pick tasks to merge in” copy; invalid
submission preserves the shared scanner/context error code. Keep item/line attribution
and the offending source span through diagnostics; do not degrade merge-in errors to
generic legacy fan-out failures. Update the singleton test helper as well as production
draft/template entry points.

For escapes, mask the entire tail during marker parsing, then restore its literal text
into the body in place, without flattening children or re-parsing the restored `@`, `&`,
`#`, or other tokens as directives. Preserve prior prefix markers and following bulk
items.

At the start of `plan_capture_item`, before task creation, clipboard consumption, or
relation work, refuse `parsed.merge_in.is_some()` with `merge_in_unavailable`: “merging
into a new task with < needs a newer bob”. The execute phase will replace this guard.
Fixture tests must prove both dry run and commit, including a later merge-in in a bulk
draft, write no files.

### 3. Editor model, diagnostics, and actual JSON

Use the shared scanner immediately after transfer handling and before editor fan-out.
Reuse the masked left parse and context checks above; preserve its body, route, fields,
spans, children, and `task`/`pomodoro_task` mode.

Add `SpanKind::MergeInArrow`, `SpanKind::MergeInSource`, and `Need::MergeInSource`,
including their string mappings. Add an optional per-item `merge_in` model and wire
object:

```json
{
  "arrow": { "start": 10, "end": 11 },
  "sources": [
    { "start": 12, "end": 16, "marker": "@a+x", "route": "a", "block_id": "x" }
  ]
}
```

Offsets above illustrate shape; runtime values must index the exact draft. Keep the
complete ordered sources on pending input. Add `merge_in_source` to needs and an
`interactive_placeholder` span on its partial token, with **no merge-in error
diagnostic** and no forced `incomplete` item mode. Existing unrelated left-side
diagnostics/needs remain intact. Never emit transfer span kinds for these clauses.

Add a reusable optional `Diagnostic.rewrite` model with `start`, `end`, `replacement`,
and `label`; omit it when absent. Initialize existing diagnostic constructors without
changing old output. Emit only explicit suggestions for `Use >`, `Move before <`, and
`Add spaces`. A rewrite's half-open range must splice into the original draft, not a
masked view, bullet-stripped substring, or expanded template instance. Re-parse applied
suggestions in tests.

Carry `merge_in` through `capture_parse.rs`'s `CaptureParseItem` and `wire_item` and its
template path. Preserve the existing top-level/first-item convention where applicable.
Audit diagnostic cloning/remapping and deduplication: template remapping currently
remaps `range` only, so a new rewrite must also be mapped into an exact original-draft
edit; never publish synthetic expansion coordinates. Where an expanded diagnostic cannot
be represented by one exact original-draft splice, leave the optional rewrite absent and
retain its error. Do not change `schema_version: 1` or serialize absent new fields as
null.

### 4. Protect assistance paths until their phase lands

Audit every `scan_transfer_line` call, including editor aggregation, `completion.rs`,
and both uses in `rewrite.rs`. Explicitly handle merge-in claim states so a source
clause cannot be mistaken for a new route/target run or have its task markers changed by
legacy automatic rewrites. Protect escaped tails too. Preserve ordinary left-side
assistance when it can be scoped correctly; safe absence of source-slot
candidates/rewrites is acceptable until bob-cli-hl.4 lands. Do not implement its picker
or toggle feature here.

### 5. Publish the grammar revision and completed documentation

Bump `GRAMMAR_REVISION` to 15 (or next free revision after synchronizing with concurrent
work). Update current-revision claims in `docs/capture.md`,
`docs/reference/compatibility.md`, and `docs/reference/json-outputs.md`, plus relevant
test expectations. Preserve historical version statements.

Remove or narrow contract-only markers only for the now-implemented grammar,
capture-parse shape, and diagnostic rewrites. Keep execution, merge JSON, pickers, and
Mac behavior explicitly contract-only. Document the transitional execution refusal
accurately. Fill in actual test names for MI7, MI8, MI13's parse portion, and MI14's
lexical portion in `docs/task-merge.md`. Read `docs.md` through `/sase_memory_read`
before editing docs. Preserve token coverage/search-smoke rows and existing reference
word budgets.

## Validation and acceptance

Use fixture vaults only. Add `capture_language/tests/merge_in.rs` and register it; add
wire tests in `capture_parse.rs` and planner/CLI tests using existing capture fixture
helpers. Prefer shared tables for editor/execution agreement:

- MI7: each context rejection, each allowed new-task form, both required `Use >` cases,
  and no rewrite for an arbitrary multi-token left side.
- MI8: the full claim table, later claim after an earlier prose `<`, all protected
  markup forms, glued arrows, duplicate sources, pending source forms, extra
  tokens/second arrow, and complete/pending source order.
- Escape cases: prefix route and type preserved, literal tail markers never route or fan
  out, child text preserved, and no changes to `\>` behavior.
- MI13 parse: multiple task-list bullets with independent merge clauses; header/list
  metadata, children, and absolute source ranges remain correct.
- MI14 lexical: each of `&`, `*`, `~`, `~~` targeting a typed source refuses; distinct
  targets remain allowed; case-folded route matching is consistent.
- UTF-8, leading indentation, CRLF/multi-line and later bulk-item offsets; template
  expansion and original-input rewrite mapping; JSON omits absent keys and exposes the
  new fields, needs, spans, and shipped grammar revision.
- Apply each structured rewrite to the original string and re-parse the result; preserve
  surrounding items, bullets, and children byte-for-byte.
- Complete merge-in is parsed as one task with no source fan-out but rejected in both
  dry-run and commit by the transitional planner guard; a later refused bulk item leaves
  the entire fixture vault unchanged.
- Regression guards for completion and rewrite source slots; existing `>` transfer,
  target-run, list, relation, template, and directive tests retain their prior behavior.

Run focused Rust tests first, then the canonical gates **`just check`** and
**`just docs-check`**. Use `/sase_monitor` for long verification runs and keep waiting
for the monitor staging command itself to exit. No new CLI subcommand or option is
planned. Do not add broad unrelated refactors or disable checks.

Phase hl.1 recorded four pre-existing docs lint errors for `BOB_GMAIL_*` names in
`src/native/google/profile.rs`. This is evidence to investigate if seen, not permission
to assume current failures are baseline. Reproduce any suspected pre-existing failure
identically on a clean base and record a `PROPOSED FOLLOW-UP:` note on bob-cli-hl.2 with
the check, baseline, and any existing tracking bead. Such a proven baseline failure does
not prevent closing this phase. Never create a follow-up bead as this phase's worker.

## Completion and handoff

Recheck current upstream work before landing and reconcile the selected grammar
revision. The implementation commit must be `feat(capture)!: ...`; its body must explain
that `X < @a+x` and `X < @route` previously wrote ordinary text or notes and now claim
merge-in syntax. Never edit Cargo versions or `CHANGELOG.md`; declare the commit through
`/sase_final` rather than manually committing unless explicitly instructed otherwise.

Record the scanner/model entry points and revision in the phase completion note so
execute/assist can consume them. Before closing, run
`sase bead epic-symbols bob-cli-hl.2`; resolve every entry or re-key a genuine
later-phase symbol to a still-open phase/parent. Then close **only bob-cli-hl.2** with
`sase bead close bob-cli-hl.2 --note "<implementation and verification evidence>"`. Do
not close bob-cli-hl or any ancestor plan bead, and do not close bob-cli-hi or other
phases. The epic's land agent owns their settlement. Finish through `/sase_final`.
