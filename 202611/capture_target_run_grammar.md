---
tier: tale
title: Parse capture target runs in execution and editor paths
goal:
  Complete bob-cli-gm.1 with per-target execution fan-out, editor target metadata,
  precise refusals, and the next capture grammar revision.
size: medium
proposed_by: bbugyi200.athena.bob-cli-gm.1
proposed_with: codex/gpt-6-astra
bead: bob-cli-gm.1
create_time: 2026-11-03 14:04:43
status: wip
---

- **PARENT:**
  [202611/capture_multi_target.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/capture_multi_target.md)
- **BEAD:**
  [bob-cli-gm.1](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-gm/bob-cli-gm.1.md)

# Capture target-run grammar for bob-cli-gm.1

## Outcome and scope

Implement the `grammar` phase of `plan:202611/capture_multi_target.md`, assigned bead
`bob-cli-gm.1`. A capture such as `@work+prep @dev+parser Called Morgan` must parse into
two executable items containing the same note, while `capture-parse` keeps one editor
item and reports both targets. Leading and trailing runs, marker-only operations,
authored children, and template rows must obey the existing single-target semantics.

This is a medium tale because the change is one coupled language feature in the existing
capture pipeline. Implement execution and editor parsing together, reusing their
per-item semantic passes. No new storage, commands, or planner architecture is required.
The phase has no earlier implementation or remaining-work notes; the clean tree at
`27f508bd` still selects a single marker in both resolvers and has no `FanoutContext`,
`targets`, or new refusal codes.

The epic's final decisions are binding: refuse split-edge target markers, and make no
memory edits (including `glossary:bulk-capture`). Do not ask those questions again. Read
the assigned bead and approved epic artifact before coding to pick up coordination notes
and concurrent changes.

This phase supplies parsed fan-out metadata to `bob-cli-gm.2` and target ranges to
`bob-cli-gm.3`. The former owns clipboard reuse, resolved-path duplicate checks, grouped
capture output, target-labelled planner errors, and warning deduplication. The latter
owns completion inside runs, rewrite, `@@` absorption, and drawer-menu behavior.
Preserve compilation of their existing callers, but keep those product changes in their
assigned phases. No Mac, plugin, release-version, changelog, or memory changes belong
here.

## Required language contract

A target is a complete or partial token accepted by `is_sub_bullet_marker_candidate`,
including its own glued `#drawer` or terminal `!`. Other route forms (`@route`,
`@route#section`, `@route:id`, `@route^id`, `^route:id`, and `!note:id`) are not members
of a target run.

Select a maximal adjacent run after removing the existing item modifiers: separate
`#type`, `%` directives, `&`/`*`/`~` relationship modifiers, and terminal `s:<N>`/`$<N>`
markers. Those modifiers may interleave with run members when their existing grammar
permits it. Preserve quoted/protected text and the existing escape rules.

| Position                                    | Recognition                                                                |
| ------------------------------------------- | -------------------------------------------------------------------------- |
| Beginning of the parent line                | Consume every consecutive target; remaining tokens are the body.           |
| Entire parent line                          | Consume the run; resolve each marker-only member using the existing rules. |
| End of the parent or an authored-child line | Consume the maximal trailing run when body text precedes it.               |
| Between prose words                         | Keep the markers literal.                                                  |

A non-target ends the run: `@work+a @dev:b note` and `@work+a @bob hi` remain
single-target items. An incomplete or invalid target-shaped member does not end the run
or become prose; report that member's existing error or needs. Forced `--route`,
`--task`, and `--drawer` destinations retain literal `@` tokens as before. A local run
in a list bullet retains `list_bullet_marker`. One run occupies one route/mode slot, so
another marker or run on another physical line still gets `duplicate_capture_marker`.

Two new coded refusals are required:

- `split_capture_targets`: when the parent line has a leading target (one or several),
  intervening body text, and a trailing target, report
  `` `@a+x` and `@b+y` are at opposite ends of the line; put every `@route+id` target in one run at the start or end ``.
  Substitute the offending markers; the editor range is the trailing token's complete
  byte range. A trailing plain `@work` after a leading target remains literal. A
  whole-line run is not a split-edge error.
- `duplicate_capture_target`: compare lower-cased routes and exact block IDs within a
  run, ignoring drawer/toggle suffixes for identity. Report
  `` `@work+a` names ^a twice on this line; name each task once `` with the repeated
  member's token and ID. The editor range covers that repeated token. Alias resolution
  belongs to the next phase, and targets in independent capture items are not rejected
  by this check.

Treat each member exactly like the original capture with only that member's marker
present. A drawer applies to its member alone; `@a+x! @b+y` is an explicit toggle
followed by Ensure Next; an explicit toggle with body text still fails. Resolve bodyless
authored notes/drawers, `%c`, separate types, schedules, priorities, and relationship
ownership per member through existing helpers. One invalid member rejects strict
execution of the entire draft.

The epic contains an inconsistent example: it calls the missing ID need in
`@work+a @dev+ note` `block_id`, whereas the existing classifier and both unit and CLI
tests call a missing `+` target `task`. Follow the epic's normative per-member
equivalence and unchanged-single-target rules by reusing that classifier and unioning
its actual needs. A `PROPOSED FOLLOW-UP:` on `bob-cli-gm.1` records this discrepancy for
the epic land agent to reconcile. Do not rename the singleton need or add a separate
run-only need mapping.

## Implementation

### 1. Share target selection without duplicating semantic rules

Add a small internal representation/helper for an ordered selected run, with each
token's original range, text, and physical line. Reuse it in
`line.rs::resolve_line_inner` and `editor_classify.rs::select_marker_token` (or its
replacement). Keep the ordinary non-target route path and its existing precedence
intact. Candidate recognition must precede strict member parsing, so invalid members are
still selected.

Extend `markers.rs::extract_terminal_markers` to cross a trailing target run while
collecting the same single schedule and priority slots. Preserve its existing
stop-on-duplicate, retired-priority, and escaped-priority behavior. Extend
`tokens.rs::validate_special_terminal_markers_line` to validate every selected boundary
member. In particular, all three arrangements `note s:3 @a+x @b+y`,
`note @a+x @b+y s:3`, and `note @a+x s:3 @b+y` extract one schedule and two targets. Do
the equivalent for `$<N>` and both markers together.

Carry the full run through `LineOutcome` and `AggregateMarkers` as one route slot. Keep
first-member access where a caller needs a single subject. Detect split edges and
lexical duplicates using the shared selection and carry their codes through
`CaptureItemParseError` and `TemplateCodedError`; avoid losing the codes at a `String`
conversion boundary.

### 2. Resolve execution members through the existing item pipeline

Refactor `item.rs::parse_capture_item` / `parse_capture_item_masked` so a run can yield
one semantic outcome per member. Reuse the current masking and per-item passes with each
member selected and the other selected markers removed from the body; do not copy the
toggle, drawer, dependency, or type rules into a second resolver. Byte-preserving masks
or an explicit selected member parameter are suitable; any borrowed declarations must be
remapped to the original source, as the current type/directive masking already does.

Run parent toggle/Pomodoro eligibility, `resolve_sub_bullet_kind`, drawer finishing,
`execution_item_dependencies`, type applicability, and single-note marker restrictions
for each member. A later member's modifier ownership must never reuse the first member's
`dependency_target`. Preserve one original declaration set and the established
cross-line duplicate checks.

In `model.rs`, add `FanoutContext { line, position, count, marker }` and
`ParsedCaptureItem.fanout: Option<FanoutContext>`. `line` and `position` are 1-based;
`marker` is that member's exact text. Only groups of at least two targets receive this
metadata. Initialize every other item to `None`.

In `draft.rs::parse_ordinary_draft`, finalize each member's dependencies and global
inheritance before pushing the members in typed order. Preserve the original `index`,
source start/end, line start/end, list context, and template row for every member.
Sharing `index` preserves the schedule-roll seed used later by planning. Keep ordinary
batch ordering around the group.

Update both `parse_template_expansion` and `parse_single_template_instance` to allow one
template item to produce several parsed members. Keep existing template
clipboard/session/list refusals and row error prefixes; fan-out must not look like
multiple template items and trigger `template_session_operator`. Retain row source
ranges and row identity for all members, and map fan-out line metadata to the
originating physical row consistently with that context.

### 3. Keep one editor item and expose all members

Extend `LineEditorParse` and `parse_editor_line` to retain all selected markers and
remove every member from `body_tokens` / `body`. `SeenMarkers::absorb_route` must be
called once per run, not once per member. Share the selection and diagnostic range rules
with execution.

Through `parse_editor_item_with` / `parse_editor_item_inner`, resolve each member with
the existing toggle, drawer, and dependency-owner logic. Gather every member's spans and
diagnostics, preserve source-byte positions (including after non-ASCII prose), and union
needs in member order without duplicates. Do not duplicate common body or modifier spans
when reusing a per-member pass. The first member remains the legacy
mode/route/section/block-ID, type-marker subject, dependency owner, and
`drawer_bullet_target`. Keep complete local destination marker metadata for all members
so the assist phase can consume it.

Add an editor target representation to `editor_model.rs`, thread it through
`EditorItemParse` and the first-item projection, and serialize it in `capture_parse.rs`
at the top level and on the corresponding `items[]` entry:

```text
targets: [{start, end, marker, mode, route, block_id, section}, ...]
```

Ranges are absolute byte offsets in the original draft, `marker` is exact source text,
and mode/route/ID/section are per-member results. Serialize `targets` only for runs with
at least two members, including partial members. Never fan out editor `items[]`.
Preserve singleton wire shape and the current omission of `items[]` for a single
ordinary item. Apart from the intentional grammar revision, old singleton JSON must
remain unchanged.

### 4. Bump and document the grammar once

Re-scan current code and docs immediately before the bump. At planning time
`GRAMMAR_REVISION` is 6, parse help claims 7, and the move-bullet contract already
claims 8 in `docs/capture.md` and `docs/reference/compatibility.md`. Therefore the
required revision is currently **9**, not the epic's provisional 8: take one more than
the highest code or documentation claim at implementation start, and never lower it
after integrating concurrent work.

Update the constant and hard-coded current-revision expectations/examples in
`capture_parse.rs`, `capture_complete/tests/output.rs`,
`tests/cli/capture/complete_drawers.rs`,
`tests/cli/capture/complete_relationship_owner.rs`, and any additional search hits. Fix
the stale grammar-4 help in `capture_complete/cli.rs`. Keep historical revision
descriptions accurate; do not renumber the history of note-pull or move-bullet features,
`schema_version: 1`, or `relation_grammar: 1`.

Read `docs.md` through `/sase_memory_read`. Update routing precedence and the
authored-child one-slot rule in `docs/capture.md`. Add the epic's “Several tasks on one
line” subsection with the recognition table, equivalence principle, per-member operation
table, refusals, and examples. Document `targets[]` and both new diagnostic codes in the
parse section. Update the capture-parse row and sample revision in
`docs/reference/json-outputs.md`, and add this break to the grammar history in
`docs/reference/compatibility.md`. Do not overwrite the adjacent contract work owned by
the execute/assist phases.

Regenerate help/reference outputs with `just docs-gen` after help edits. Inspect
generated changes; do not hand-edit canonical memory. The note-pull epic `bob-cli-gj`
was still in progress when this plan was written. If it is still open after the actual
bump, append a coordination note naming the taken revision as the approved epic
requires. Record the revision on this phase as well for later workers.

## Verification and completion

Extend the existing capture-language test modules and CLI fixtures; avoid a second
parser test harness. Use table-driven cases where useful.

- Execution (`capture_language/tests/grammar.rs`): leading, trailing, marker-only,
  child-line and three-member runs; each member's drawer and `!`; mixed toggles; a
  separate type; clipboard forms; `&`, `*`, `~`, and `~~`; schedule/priority before,
  after, and between members; template fan-out; duplicate routes differing only by case;
  duplicates with different suffixes; split edges; malformed second/third members;
  cross-line route duplication; list-bullet refusal; and forced destinations keeping
  markers literal.
- Assert full per-member semantics against the equivalent single-target parses,
  including kind, route, body, authored children, type, modifiers, dependency ownership,
  shared indices/ranges, and precise fan-out metadata. Cover a multi-note refusal and a
  member-specific self-dependency refusal.
- Editor (`editor_spans.rs`, `editor_modes.rs`): mirror the boundary/error matrix,
  verify bodies and every target range, mixed member modes, unioned incomplete needs,
  first-member legacy fields, and one editor item per draft item. Include Unicode before
  a trailing run and a later blank-separated item to catch character-offset and
  item-relative-offset mistakes.
- CLI (`tests/cli/capture/parse.rs`): JSON for leading/trailing/partial runs, both coded
  refusals, a batch containing a run, and a singleton compatibility fixture. Ensure the
  strict capture JSON also preserves the two new codes.
- Regression literals: `Discuss @dev+id later`, `@a+x @b:y note`, `@a+x @bob hi`, and
  `@a+x note @work`; existing singleton tests must pass unchanged except their explicit
  revision assertions.

Run focused capture-language and capture-parse tests while iterating. Then run
`just fmt`, `just lint`, `just test`, and `just docs-check` (the first three are also
covered by `just check`). Use `/sase_monitor` for long verification commands and follow
its staging/finalization procedure. `just fmt` checks formatting; use `cargo fmt --all`
to repair formatting when needed.

With the binary built from the checkout, also run
`bob capture-parse -f json -- '@work+a @dev+b note'` and confirm body `note`, two
targets, and the selected revision. In an isolated scratch vault with open `work.md ^a`
and `dev.md ^b` tasks, run `bob capture --dry-run -f json -- '@work+a @dev+b note'`;
confirm two `sub_bullet` captures and unchanged vault bytes. Use existing test fixture
configuration patterns, not the user's real vault. Do not require public capture
`fanout` output until the execute phase implements it.

Before closing, inspect the final diff and run `sase bead epic-symbols bob-cli-gm.1`
again (there were no entries when planned). Resolve leftovers or re-key them to the
still-open parent/later phase. Record verification, the actual revision, and any
limitations with `sase bead close bob-cli-gm.1 --note "<what was verified>"` only after
this whole phase is implemented. Never close `bob-cli-gm` or an ancestor plan bead. Do
not hand-set status, reopen an already active phase, or create task beads. Record
discovered work with `PROPOSED FOLLOW-UP:` notes on this phase. A gate failure
reproduced identically on the clean base is recorded there with evidence and any
existing tracking bead, then does not block phase completion.

Finalize implementation through `/sase_final` with a `feat(capture)!:` subject and a
`BREAKING CHANGE:` footer explaining that adjacent target markers now fan out and
split-edge targets are refused. Leave Cargo versions and `CHANGELOG.md` to the release
train.
