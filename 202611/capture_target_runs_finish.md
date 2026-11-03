---
tier: epic
title: Complete capture target-run grammar in bounded phases
goal: Complete bob-cli-gm.1 with equivalent per-target execution, one editor item
  with targets, coded refusals, and one grammar revision bump.
phases:
- id: selection
  title: Shared target selection and source-preserving member views
  size: medium
  depends_on: []
  description: 'selection: build and test shared target-run selection, terminal extraction,
    refusal metadata, and byte-preserving member views while keeping public parsing
    behavior unchanged.'
- id: execution
  title: Execution member resolution and draft fan-out
  size: medium
  depends_on:
  - selection
  description: 'execution: resolve each member through existing item semantics, fan
    out ordinary and template drafts, and preserve source metadata and coded errors
    behind the internal staging option.'
- id: editor
  title: Editor member semantics and targets JSON
  size: medium
  depends_on:
  - execution
  description: 'editor: reuse shared selection and existing editor semantics to produce
    one item with all target spans, ordered needs, exact diagnostics, and additive
    targets metadata behind the internal staging option.'
- id: activate
  title: Activate both parsers and publish the grammar contract
  size: medium
  depends_on:
  - editor
  description: 'activate: remove internal staging, activate both resolvers together,
    bump the grammar once above every existing claim, document the contract, and verify
    CLI parity and regressions for the child epic land agent.'
proposed_by: bbugyi200.athena.bob-cli-gm.1
proposed_with: codex/gpt-6-astra
parent_bead: bob-cli-gm.1
create_time: 2026-11-03 16:48:51
status: wip
bead_id: bob-cli-gm.1.1
---

- **PROMPT:** [prompts/202611/capture_target_runs_finish.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/capture_target_runs_finish.md)
- **BEAD:** [bob-cli-gm.1.1](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-gm/bob-cli-gm.1.1.md)

# Complete capture target-run grammar

## Scope, evidence, and ownership

This is a child epic for **bob-cli-gm.1**, the grammar phase of
`plan:202611/capture_multi_target.md`. It replaces the implementation approach in
`plan:202611/capture_target_run_grammar.md`; it does not expand the approved product
scope. The host should attach this child epic to the assigned phase. Do not create beads
manually.

At the inspected base `0b4cebdf`, the tree is clean. Both `resolve_line_inner` and
`select_marker_token` still choose one marker, `ParsedCaptureItem` has no fanout
context, and neither new refusal code exists. The earlier phase proposal was archived,
but its implementation has not landed. The phase history has one discrepancy note and no
completed implementation or verification. There are currently no `--epic-symbol` entries
for bob-cli-gm.1. Recheck current state before implementing; retain any work that lands
meanwhile.

The remaining change crosses two independent semantic passes, ordinary and template
draft assembly, source ranges, JSON serialization, and documentation. Split those
concerns across the four medium phases above instead of repeating the whole tale.
Dependencies are deliberately sequential because these changes share parser models and
entry points. Each phase has focused verification and must leave a buildable tree.

The approved decisions remain final: **split_edges = refuse** and **memory_bulk_capture
= no**. No memory file may be changed. Do not add reviewer decisions that reopen these
choices. Only the child epic's land agent may complete the delegated bob-cli-gm.1 scope
after all child phases and integration evidence are ready; child phase workers close
only their own assigned beads. Never close bob-cli-gm or any other ancestor plan bead.

Keep these outer-epic responsibilities outside this child epic:

- bob-cli-gm.2 owns clipboard memoization/snippet reuse, resolved-path duplicate checks,
  grouped human/JSON capture output, target-labelled planner failures, roll integration
  tests, and warning deduplication.
- bob-cli-gm.3 owns completion inside a run, separator rewrite, @@ absorption, and the
  drawer menu. Keep existing callers compiling and preserve the first-member subject.
- No Mac, plugin, version, CHANGELOG.md, or memory edits belong here.

## Contract every phase must preserve

For `@work+a @dev+b note`, execution ultimately yields two parsed items with body
`note`; capture-parse yields one item whose legacy fields describe work/a and whose
`targets` describes both members.

A member is a token accepted by `is_sub_bullet_marker_candidate`, including a partial
member, its own glued drawer, or its own terminal `!`. Plain routes, section routes,
colon/caret routes, `^route:id`, and `!note:id` are not members. Choose the maximal
adjacent run after the existing type, directive, dependency/relation, and terminal
schedule/priority handling. Keep quoting, protected Markdown, and escape rules.

| Position                    | Meaning                                               |
| --------------------------- | ----------------------------------------------------- |
| Start of parent line        | Consume the run; the remaining tokens form the note.  |
| Entire parent line          | Resolve each member's existing marker-only operation. |
| End of parent or child line | Consume the run only when body text precedes it.      |
| Between prose words         | Preserve literal text.                                |

One run consumes one route/mode slot. A second route-bearing physical line retains the
existing duplicate-marker refusal. A local run in a capture-list bullet retains
`list_bullet_marker`. Forced route/task/drawer destinations keep @ tokens literal.
Non-target tokens end the run, so `@a+x @b:y note` and `@a+x @bob hi` remain
single-target captures. A malformed target-shaped member is selected and receives its
existing strict error or editor needs, never silently treated as prose.

Each member must equal parsing the same item with only that target retained. Reuse the
existing item semantic passes for drawers, note-less authored children, explicit toggle
versus Ensure Next, separate type markers, clipboard directives, schedule/priority, and
dependency/relation ownership. For example, `@a+x! @b+y` has different toggle intents
per member, and `@a+x! @b+y note` must retain the explicit-toggle-with-body refusal. An
invalid member rejects the whole strict draft before planning writes anything.

New refusals are coded through `CaptureItemParseError` and `TemplateCodedError`:

- `duplicate_capture_target`: compare normalized lower-case route plus exact block ID
  within a run, ignoring drawer/toggle suffixes. Use the repeated token's complete byte
  range and the message `` `@work+a` names ^a twice on this line; name each task once ``
  with actual values. Separate draft items may name the same task; aliases are the outer
  execute phase's responsibility.
- `split_capture_targets`: a leading target and a trailing target separated by prose are
  refused, even when either edge contains only one target. Use the trailing offending
  token's complete byte range and the message
  `` `@a+x` and `@b+y` are at opposite ends of the line; put every `@route+id` target in one run at the start or end ``.
  An all-target line is not split; `@a+x note @work` retains the trailing plain route as
  literal text.

The parent design's incomplete-member example says `block_id`, but the current singleton
classifier returns `Need::Task` for `@dev+`. The phase's existing PROPOSED FOLLOW-UP
records this inconsistency. Preserve the stronger per-member equivalence and singleton
compatibility requirements: union the existing classifier's needs, including `task`,
without a run-specific renaming.

## Phase selection

Own a shared internal target-run representation and source transformation, preferably in
`capture_language/target_runs.rs`, with focused tests alongside the existing grammar
tests. Keep lexical selection separate from strict parsing so incomplete/invalid members
are included. Retain original marker text, absolute byte ranges, physical line, and
source order. Share split-edge and duplicate identity/message logic; adapters turn the
same evidence into strict errors or editor diagnostics.

Provide an offset-preserving view of one item for each member: remove the other run
members with byte-preserving masks, retaining authored children and the original item
identity. Never compact the source before computing spans. Reuse existing type/directive
masking and declaration-remapping conventions. Preserve one declaration set; masking
must not create multiple @@ declarations. Reuse the existing line tokenization and
dependency stripping rather than introducing a second lexer.

Extend terminal extraction to cross a target run and collect schedule/priority on either
side or between its members. `note s:3 @a+x @b+y`, `note @a+x @b+y s:3`, and
`note @a+x s:3 @b+y` must select the same targets and schedule. Cover equivalent
priority cases and both marker kinds. Preserve existing stop-on-duplicate,
escaped-priority, retired-priority, and non-target-route behavior. Extend
special-terminal validation through the same selected boundary tokens.

To keep public execution/editor behavior in lockstep during separate commits, use one
temporary **internal parser option**, default false at all production entry points, with
explicit opt-in for new grammar unit tests. Carry it through CaptureParseOptions and
EditorParseOptions (or an equally small private parse context). Parameterize shared
selection/extraction only where needed; keep one implementation of each semantic rule.
Do not add a CLI flag, environment variable, persistent setting, or permanent feature
gate. The activate phase removes this option and its legacy-only branches. Existing
singleton tests and production CLI results remain unchanged until then. This staging is
part of the phase dependency contract, not a user-visible grammar mode.

Tests must exercise leading/trailing/child boundaries, two and three members, mixed
suffixes, partial and malformed members, Unicode offsets, modifier interleaving,
duplicate identity, split edges, and the literal examples above. Demonstrate that the
public/default path is still unchanged. Run focused capture-language tests and format
and lint checks before closing this child phase.

## Phase execution

Using selection's member views, add an execution adapter that returns an ordered set of
item outcomes by invoking the existing parse_capture_item pipeline per member. Retain
the singleton implementation as the semantic engine; do not clone the toggle, drawer,
type, dependency, or reference logic. Select the run before early whole-item claims can
misinterpret another marker as note body, but preserve their precedence on unrelated
input. Carry errors without losing codes at String conversions. Keep list refusals and
forced destinations ahead of fan-out detection.

Add `FanoutContext { line, position, count, marker }` and
`ParsedCaptureItem.fanout: Option<FanoutContext>`. Line and position are 1-based; marker
is exact source text. It is None for singletons and other item kinds. Keep all
constructors compiling. Members share original index, start/end, line_start/end, list,
and template_row; they differ in route, kind, and dependency_target. Sharing the index
preserves the planner's roll seed. Preserve ref_source where relevant.

In parse_ordinary_draft, finish global inheritance and dependency ownership for every
member, then append in typed order. Do not duplicate global declarations or common
source warnings simply because an item expanded. Keep other batch items in order.
AggregateMarkers treats the selected run as one route slot; cross-line duplicates remain
errors. Masking can retain the existing singleton line resolver as the member engine as
long as shared selection drives the entire expansion and validation.

Update both parse_template_expansion and parse_single_template_instance. One source
template item may yield several members without violating the existing exactly-one
source-item/session-operator guard. Each member gets the template row's original
range/index/line and same TemplateRowSummary; map fanout.line to that originating row.
Retain template clipboard, list, and session refusals and row-prefixed coded errors.

Tests use the internal opt-in and compare each full semantic result against its
single-target equivalent. Cover member-local drawer/!, marker-only, bodyless bullets,
clipboard, type, schedule/priority, all relationship modifier directions, self-target
refusals, a later invalid member, batch ordering, and template rows. Assert all shared
source metadata and fanout fields. Run focused grammar/template tests and formatting and
lint. Production remains on the existing behavior until activate.

## Phase editor

Use the same selection and member views in parse_editor_item_with/inner. Reuse the
existing classifier and semantic resolution once per member, then combine into one
EditorItemParse. The leading/terminal selection and removal of every selected marker
must match execution. SeenMarkers claims the run once; it must not flag the second
member as a cross-line duplicate. A separate route-bearing line still does.

Collect every member's sub_bullet, task_toggle, and drawer spans and exact diagnostics.
Union needs in member order without duplicates; do not duplicate shared type, directive,
dependency, or body spans. Keep all positions as absolute original-draft byte offsets,
including after Unicode and in later items. Legacy mode/route/section/ block_id,
dependency owner, type-marker subject, and drawer_bullet_target remain the first
member's resolved values. Retain all local_destination_markers for the outer assist
phase, without implementing its completion/rewrite behavior here.

Add the editor target model and thread it through EditorItemParse, EditorParse's
first-item projection, and capture_parse.rs's CaptureParseResult and CaptureParseItem:

```text
targets: [{start, end, marker, mode, route, block_id, section}, ...]
```

Emit it only for runs of at least two selected members, including partial members. Never
expand editor items[]. Preserve its existing omission for an ordinary one-item draft and
the complete singleton JSON shape. New diagnostics use the common coded refusal evidence
from selection. The internal staging option still defaults false.

Mirror execution's cases in editor_modes/editor_spans tests, including mixed toggle
intents, later-member errors, unioned needs, exact target text/ranges, second-item
offsets, and duplicate/split diagnostics. Test serialization via the existing result
builder with opt-in, and keep ordinary CLI behavior unchanged. Run focused editor,
parse, completion, and rewrite regression tests plus formatting and lint; completion and
rewrite product expansion still belongs to the outer assist phase.

## Phase activate

Verify both adapters are complete, then remove the temporary option and old staging
branches in one change. Default public execution, editor parsing, and template paths
must now use the same target selection. Remove temporary interfaces that have no
remaining caller; do not leave disabled production logic or tests that require a special
mode. Keep single-marker and non-target route behavior unchanged except the approved
split-edge refusal.

Bump GRAMMAR_REVISION exactly once in this phase, to one greater than the highest
revision claimed by code or docs at activation time. At planning time the constant is 7,
parse help claims 7, completion help still claims 4, and docs claim 8; thus the next
revision is currently **9**, not the parent plan's provisional 8. Re-scan before the
actual bump and never lower it while integrating concurrent work. Read bob-cli-gj; if
still open, append the parent-required coordination note naming the taken revision.
Record it on bob-cli-gm.1 as well.

Search for every hard-coded current revision in source, tests, docs, and generated
reference help. Known locations include capture_parse.rs, capture_complete/cli.rs,
capture_complete/tests/output.rs, tests/cli/capture/complete_drawers.rs and
complete_relationship_owner.rs, docs/capture.md, docs/reference/compatibility.md, and
docs/reference/json-outputs.md. Correct current samples and statements; preserve
historical revision descriptions and schema_version/relation_grammar version 1.

Read docs.md through /sase_memory_read before documentation changes. In docs/capture.md,
update routing precedence and the authored-child route-slot sentence; add the parent
epic's "Several tasks on one line" contract with recognition table, equivalence,
operation table, and refusals. Document targets and both diagnostic codes in the parse
section. Update the JSON reference's capture-parse row and compatibility history. Keep
clipboard/output completion details owned by the later outer phases. Regenerate CLI
reference help with `just docs-gen` after help edits; inspect generated changes. No
memory edits are authorized.

Add public CLI cases in tests/cli/capture/parse.rs for leading and trailing runs, an
incomplete later member, duplicate and split refusals, and a batch containing a run.
Check strict capture JSON preserves both error codes. Assert singleton wire shape stays
identical apart from the intentional grammar revision. Run the complete semantic matrix
through the now-default paths, not only the intermediate adapters.

In an isolated HOME/XDG scratch vault using existing CLI fixture patterns and
BOB_REF_JOBS_KICK=off, create work.md ^a and dev.md ^b open tasks. With the locally
built binary, verify capture-parse for `@work+a @dev+b note` returns body note and two
targets; capture --dry-run -f json plans two sub_bullet captures without changing vault
bytes. Do not use Bryan's vault. Public captures[].fanout belongs to bob-cli-gm.2 and is
not a requirement for this phase.

Run `just fmt`, `just lint`, `just test`, and `just docs-check`. The first three may be
run through `just check`; `just fmt` checks formatting, so repair with cargo fmt when
needed. Use /sase_monitor for long verification and settle its staged handoff correctly.
Record results and any clean-base failures for the child epic land agent. Commit the
activation through /sase_final with `feat(capture)!:` and a BREAKING CHANGE footer
explaining fan-out and split-edge refusal. Never edit Cargo.toml's version or
CHANGELOG.md.

## Child epic landing and completion

The land agent verifies every child phase is closed, the temporary staging option is
gone, both public parsers implement the shared contract, template fan-out works, the
actual grammar revision is consistent, and required checks/live scratch checks have
recorded evidence. Re-run checks only after new changes or unresolved failures justify
it. Preserve the earlier incomplete-need discrepancy note for the outer epic land agent.

Run `sase bead epic-symbols` for each bead immediately before closing it, especially
`sase bead epic-symbols bob-cli-gm.1`. Resolve or re-key leftovers to a genuinely
still-open owner. Temporary symbols introduced in this child epic must be consumed or
removed before landing; avoid broad warning suppressions. Do not hand-set statuses or
reopen the already-active parent phase.

Child workers report discovered follow-ups on their own assigned phase with
`PROPOSED FOLLOW-UP:` notes; the child land agent carries relevant unresolved findings
to bob-cli-gm.1 for the outer land agent. Do not create task beads. A check failure
reproducing identically on the clean base must be recorded with reproduction evidence
and any existing tracking bead; it does not leave a completed phase open.

Once all required work is finished, the child epic land agent settles its own epic under
its host authorization, then closes **only the delegated outer phase** with
`sase bead close bob-cli-gm.1 --note "<implementation, revision, checks, and scratch-vault evidence>"`.
The child epic must be closed before this ancestor phase can close, because closing is
non-cascading. This instruction is not permission for a child phase worker to close an
ancestor. Leave bob-cli-gm and other ancestor plans to their own land agents. Finish
each root turn through /sase_final, using keep until the assigned scope is complete.
