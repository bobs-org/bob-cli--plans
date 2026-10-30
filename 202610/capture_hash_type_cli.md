---
tier: tale
title: Migrate bob-cli capture task types to hash markers
goal:
  Complete bob-cli-ep.1 with hash type grammar, configured-type refusal, completion,
  tests, and docs in one coherent change.
size: medium
proposed_by: bbugyi200.apollo.bob-cli-ep.1
proposed_with: codex/gpt-6-astra
bead: bob-cli-ep.1
create_time: 2026-10-30 15:11:10
status: wip
---

- **PARENT:**
  [202610/capture_hash_type_marker.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/capture_hash_type_marker.md)
- **BEAD:**
  [bob-cli-ep.1](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-ep/bob-cli-ep.1.md)

# Implement capture hash task types in bob-cli

Complete phase **bob-cli-ep.1** of **bob-cli-ep**, using the approved design
`plan:202610/capture_hash_type_marker.md`. The result is one coherent bob-cli change:
`#<type>` replaces `t:<type>` in grammar, execution, completion, fixtures, help, and
documentation. Bob Mac Capture work belongs to the dependent phase bob-cli-ep.2.

## Scope and established decisions

The phase was in progress, assigned to bob-cli-ep.1, with no earlier work notes when
inspected. The clean source tree at `3f69021a` still has the old `t:` lexer and
warn-and-write behavior. Commit `6990e89b` already fixed D8 relocation identity and
named-type agreement; preserve those fixes. No implementation of this migration has
landed on this inspected tree. Recheck the bead and source when starting in case that
changes; do not redo completed work or set bead status by hand.

This is a medium tale for one coding agent: the existing shared lexer, type writers,
completion rows, and integration fixtures provide the implementation seams. The changes
need to land together under the parent epic's coherent-master rule. There is no prior
unfinished attempt requiring a child epic.

Final reviewer decisions, with no new choices to approve:

- `bare_hash_menu = letter`: free-standing `#` alone opens no type menu. A letter or
  opening quote starts it; attached empty `#` opens the full menu.
- `retire_t = prose`: `t:` is ordinary text everywhere. Remove its special scanning and
  escaping, and add no `retired_type_marker` diagnostic.
- `memory_record = no`: edit no memory notes, decision records, or generated
  `AGENTS.md`; do not run memory publication.

Keep `schema_version` at 1 and the existing Task type wire fields and presentation. Do
not edit Cargo package versions, CHANGELOG.md, the shared Pomodoro-theme fixture
`tests/fixtures/pomodoro_themes/vectors.json`, or companion repositories. No new CLI
options are needed. Bob remains the only grammar and write authority.

## Observable contract

### Grammar and context ownership

| Input/context                                                                    | Required result                                                                                |
| -------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------- |
| `#fix Fix it`, `Fix it #fix`, `Fix it #fix @sase`                                | Type the new task and remove the marker from its body                                          |
| Child `- check #fix`, continuation `#fix`, or child `- #fix`                     | Type the enclosing item; a marker-only continuation contributes no child bullet                |
| `Fix it @sase^parser#fix`                                                        | New task `^parser` typed FIX; only the suffix is the type marker                               |
| `@sase+parser#fix`                                                               | Retype the existing task, then Ensure Next using its new type                                  |
| `@sase+parser#fix &notes:spec`                                                   | The same existing-task type semantics with the existing relationship/dependency owner behavior |
| `Need details @sase+parser#requirements`, or an item with authored child bullets | Preserve the task-section selector; no attached type marker                                    |
| `^sase:parser #fix`, `@sase+parser! #fix`                                        | Preserve existing explicit-type retyping and relocation                                        |
| `@sase^parser#`, marker-only `@sase+parser#`                                     | Incomplete with need `task_type`; full type completion menu                                    |
| `jot idea #`                                                                     | Pomodoro note, with no type menu                                                               |
| `Ask about #fix today`                                                           | Mid-line prose                                                                                 |
| `Review PR #1234`, `##heading`, `#-value`, non-ASCII-led hash text               | Prose; no type marker or legacy-tag refusal                                                    |
| Edge `\#fix`, `\#"Q&A"`                                                          | Keep literal hash text, remove only the escape backslash; mid-line escapes remain verbatim     |
| `Fix it t:fix`, edge `t:`, `t:"DEEP WORK"`, `\t:fix`                             | Ordinary text with no special `t:` escape removal, span, need, or menu                         |

A slug starts with an ASCII letter and continues with ASCII letters, digits, or `-`.
Quoted literals use the existing `lex_exact_name_after_hash`, including curly quotes and
`\"` / `\\` escapes. Digit-led names require literals such as `#"2FA"`.
Unterminated/empty literals and empty attached suffixes are incomplete. `#fix.`,
`#fix!`, `#a[b]`, and `#bob+decks` are claimed invalid types with teaching errors. One
type marker is allowed per item, including attached plus free-standing duplicates.

Retain current marker edge ownership: only the existing transparent markers,
dependencies, blocked-task modifiers, relations, and type markers may separate a marker
from its physical line edge. Attached route tokens must also occupy a marker position.
Protected URLs, code, wikilinks, quoted route/modifier text, and `C#` remain protected.
The suffix of `@r^id+#NAME` remains a project-note Pomodoro name; `@r^id#NAME+` retains
the existing misordered-project diagnostic. Refuse `@r+id#NAME!`, teaching:

> explicit toggle `!` cannot follow a task type; write `@<route>+<block-id>! #<type>`

Other hash owners remain unchanged: `@route#Section`, `@route:id#NAME`,
`^route:id#NAME`, `=#NAME`, counted named starts, and project-note Pomodoro names.
`x#bugs` still errors. Pomodoro names retain their existing unknown-name warning.

### Type resolution and refusal

Keep exact-name resolution: configured spelling first, then catalog spelling; literals
match keys, slugs additionally try spaces for hyphens, and prefixes never resolve. Share
a capture-type acceptance predicate between writers and completion:

- With a vocabulary, accept only keys in `Catalog::configured_themes()`.
- Without a vocabulary, accept only types already known by the history catalog.
- Exclude inventory labels. Do not use `Catalog::configured()` or a ThemeRef's
  `configured: true` alone: inventory labels deliberately report true there.

Rejected markers fail the complete capture transaction, including dry-run, with
`type_not_configured` and exit 2 before vault writes. Remove the marker-specific
canonical-write fallback and warning path; loading failures must not bypass type
validation. Keep unrelated Pomodoro warning behavior intact.

Use the approved error copy, substituting the actual raw marker, canonical name, config
path, and up to two similar names from the existing similarity rule:

> `#fxi` is not one of your task types; did you mean `#fix`? Add FXI under `types:` in
> <config> to use it, or write `\#fxi` to keep it as a tag

Without a vocabulary:

> `#snack` is not a task type yet: no Pomodoro has used SNACK and no `types:` vocabulary
> is configured; add `types:` to <config>, or write `\#snack` to keep it as a tag

Omit the suggestion clause if nothing is similar. Append at most one context hint:
attached `@r^id#NAME` teaches `@r^id+#name` for a project note and its Pomodoro;
attached `@r+id#NAME` teaches `^r:id#name` to link under that Pomodoro; a free-standing
type alongside a plain `@route` teaches `@route#name` for a heading bullet. Attached
forms omit the escape-as-tag clause. Preserve literal spelling/escaping in remedies.

All applicability, duplicate, incomplete, and conflict messages must use `#` spellings.
Duplicate label: `type marker (#<type>)`. A new attached block-ID capture without task
text teaches `Fix the parser @sase^parser#fix`. The sub-bullet remedy teaches capturing
`@<route>+<block-id>#<type>` alone to retype the parent. Named Pomodoro disagreement
retains `type_marker_conflict` and teaches which hash selector to drop, naming the
original destination token as in the approved design.

### Editor and completion

`task_type` spans own the whole free-standing marker or only an attached suffix. Their
one ThemeRef owns the name content, excluding hash and quotes. Spans must stay sorted,
non-overlapping, and correct in UTF-8 and CRLF drafts. Adjacent block-ID and type spans
may touch. Incomplete markers report mode `incomplete` and need `task_type`.
`capture-parse` stays lexical: annotate known/configured ThemeRef flags, but emit no
`type_not_configured` diagnostic for partial or unknown names.

`capture-complete` uses the existing `task_type` context and rows. A marker-only
`@r+id#…`, including its incomplete state and relationship-only owner, reports
`subject: "existing"`. The new-task `@r^id#…` reports `new`. Bare free-standing `#`,
mid-line prose, task-section selectors, Pomodoro names, and old `t:` get no type
context. Do not add `initial_selection`.

Replacements are `#slug` only if the slug starts with a letter and round-trips to the
key; otherwise use an escaped `#"NAME"` literal. Leading free-standing markers retain
the existing space insertion rule; attached suffixes never append a space. Empty
attached menus keep config order (or the existing history menu without a vocabulary).
Keep ranking, effects, hiding, and suggestions. Unknown typed rows carry
`refused: "type_not_configured"` and `similar_to`; prefix/substring matches remain
usable. Shell rows retain the `task types` group, with full value `#fix`; verify zsh
quotes insertion so `interactive_comments` cannot consume it.

## Implementation sequence

1. Read
   `sase bead read bob-cli-ep.1 -r "Need current scope and notes before implementation"`
   and the approved artifact through `sase artifact read`. Read the required docs memory
   before editing docs/help. Inspect source and notes for work that landed after
   planning. Capture the current completion/parse latency baseline before the lexer
   change, using the existing read-only benchmark method on apollo.

2. Update `src/native/capture_language/type_marker.rs` and its exports in `mod.rs`.
   Replace sigil, openers, literal/query handling, marker escaping, and error copy.
   Extend the offset-preserving scanner to attached suffixes. Introduce one shared
   item-level predicate for the `+id` marker-only decision, based on the body and
   authored children after masking other markers. Use it in scanner, completion, and
   editor classification rather than three approximate heuristics. Keep this helper
   below the high-level parser so masking cannot recursively invoke itself. Preserve
   suffix-only masks and marker-only continuation removal through execution, editor,
   completion, templates, rewrite, and draft splitting. Review all callers of
   `scan_line`, `collect_item_with_clip_control`, and masked-token helpers so attached
   suffixes do not leak through a line-only fast path.

3. Update the grammar consumers under `capture_language/`: `tokens.rs`, `markers.rs`,
   `line.rs`, `item.rs`, `draft.rs`, `editor_classify.rs`, `editor_parse.rs`,
   `completion.rs`, `rewrite.rs`, and `template.rs` as necessary. Remove
   `reject_legacy_bullet_markers`, `legacy_bullet_marker_diagnostic`, and the old
   diagnostic. Remove all special `t:` handling. Delete the named Ensure Next path:
   `CaptureKind::TaskToggle`'s Pomodoro-name fields, the literal-to-toggle shortcut in
   `parse_sub_bullet_route_token`, the obsolete `TaskTogglePomodoroName` span kind, and
   corresponding branches in `capture/task_toggle.rs`, `capture/ensure_next.rs`, and
   their consumers. Ensure masked marker-only `+id` still uses normal Ensure Next;
   body/children retain the strict task-section charset. Preserve explicit toggle
   behavior and its new remedy.

4. Add shared acceptance/refusal support at the existing exact-name/catalog seam. Use it
   in both new-task and existing-task paths of `capture/task_theme.rs` and completion;
   pass origin/route information from `capture/plan.rs` as needed for context hints.
   Keep subject restrictions, field exception semantics (`set`/`cleared`/`kept`), batch
   atomicity, named agreement, and the D8 relocation applier. Do not change unrelated
   type inference or ledger behavior. Remove the two current warn-and-write branches and
   the config/catalog failure fallbacks that currently write an arbitrary marker name.

5. Update `capture_parse.rs` ThemeRef decoration and span parsing, plus
   `capture_complete/task_types.rs`, `engine.rs`, `shell.rs`, and `model.rs` only as
   needed. Fix the current existing-subject detection (it currently recognizes only
   relationship targets and whole-item PomodoroLink/Incomplete forms). Reuse the shared
   acceptance rule for refused typed rows and inventory exclusions. Keep
   rendering/grouping stable. Exercise the generated zsh adapter through
   `tests/cli/completion/zsh_adapter.rs` or the existing interactive completion harness;
   checking raw completion text alone is insufficient evidence of quoting.

6. Convert and extend the existing tests with the verification matrix below. Use valid
   configured types/history in fixtures testing unrelated behavior; do not simply rename
   old unknown-name success tests, since these must now refuse. Update
   `tests/fixtures/capture_type_marker/vectors.json`; leave the Pomodoro-theme vectors
   intact. Compute byte offsets from the new input where practical, and retain exact
   range assertions for attached and Unicode forms.

7. Update help in `capture/cli.rs`, `capture_parse.rs`, and `capture_complete/cli.rs`,
   including the overloaded `+id#` examples and incomplete needs. Update
   `docs/capture.md` (grammar table, hash ownership, typing, retired forms, #now, Ensure
   Next, spans/needs/completion), `docs/reference/capture-syntax.md` (token index,
   overloaded hash rows, `plus-child-section`, `plus-ensure-next`, and `hash-type`
   replacing `t-type` with inbound links fixed), `docs/task-types.md` (PT12 refusal
   exception, typed captures, remove Ensure Next from named-capture forms, routing
   table), `docs/completion.md`, and `docs/_meta/search_smoke.yml`. Treat `t:` as prose
   in all retirement copy. Run `just docs-gen` for generated CLI references and indexes,
   reviewing its diff so it does not introduce memory/source changes or edits to mounted
   sibling docs.

8. Remeasure warm `capture-complete`, `capture-parse`, and the shell task-type slot for
   `Fix the parser #f` on apollo using the documented debug-build method (one warmup, 20
   samples). Compare equivalent current baseline timings, retain PT3's +5 ms p95 budget,
   and update the measured row/date/method in `docs/completion.md`. This is read-only
   against the vault; use isolated temporary vaults for all write tests. If measurement
   is unavailable, report that limitation honestly rather than inventing values.

9. Run focused tests during development, then `just check`, `just docs-check`, and
   `just all`, as the epic requires. Use `/sase_monitor` for long verification runs.
   Review the complete diff and audit old type spellings over capture source, docs, and
   tests with `rg`; remaining `t:` matches must be intentional prose-regression cases,
   migration documentation, or unrelated syntax such as `[t:: 30m]`.

## Verification matrix

Extend `capture_language/tests/{type_marker,type_marker_editor,completion,rewrite}.rs`,
the nearby grammar/editor tests, `capture/tests/type_marker.rs`, the tests inside
`capture_parse.rs`,
`tests/cli/capture/{capture_type_marker,type_marker_existing,complete_task_types}.rs`,
and `tests/cli/completion/` as appropriate. Each behavior should be tested at its owning
layer; use integration tests for writes, rollback, and wire contracts.

| Area                | Required evidence                                                                                                                                                                                                                                                                    |
| ------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| Free markers        | Leading/trailing/child/literal/escaped cases, one-marker duplicate errors, malformed claimed tokens, protection inside code/URLs/quoted tokens                                                                                                                                       |
| Attached markers    | New block-ID and existing-task suffixes, empty and literal suffixes, exact non-overlapping spans/ThemeRef name ranges, Unicode and CRLF, mixed attached/free duplicates                                                                                                              |
| Item ownership      | Marker-only `+id`, modifiers on either side, authored children and body choosing sections, marker-only continuation lines not accidentally creating children, project `id+#name` and misordered `id#name+`                                                                           |
| Retirement          | `t:` stays literal text at either edge and mid-line with no special escape processing, needs, or completion; removed legacy-tag errors stay removed                                                                                                                                  |
| Other hash contexts | Bare-hash Pomodoro note, no candidates, `Fix it #fix #` not applicable, digit/hash/non-ASCII prose, heading sections and task sections unaffected                                                                                                                                    |
| Resolution/refusal  | Configured success including never-used types, exact versus prefix, history success without vocabulary, unknown refusal in both configurations, inventory refusal, at most two suggestions, every context hint, malformed/unavailable config cannot silently write an arbitrary type |
| Write invariants    | New and existing set/cleared/kept, closed-target refusal, Ensure Next placement and D8 identity, relationship-only owner, named agreement, entire-batch rollback on unknown type, dry-run output equals real-run apart from `dry_run`                                                |
| Editor contract     | No configured-type diagnostics while typing; correct `task_type` flags, need and mode; masking property extended to attached forms; rewrite/template/draft/bare-URL handling still correct                                                                                           |
| Completion          | Existing subject for empty/partial/complete `+id#` and relationship owner, new subject for `^id#`, digit-led literal replacements, attached replacement without trailing space, refused typed row, unchanged ranking/hide/effect/group                                               |
| Preserved behavior  | Pomodoro-name unknowns still warn and write, `@r:id#` and `=#` name menus remain correct, `tests/cli/task_move.rs` stays green                                                                                                                                                       |
| Shell               | Task-type group and values plus actual zsh insertion quoting under `interactive_comments`                                                                                                                                                                                            |

Useful focused commands are `cargo test --lib type_marker`,
`cargo test --lib capture_language`, `cargo test --lib capture_parse`,
`cargo test --test cli capture::`, `cargo test --test cli completion::`, and
`cargo test --test cli task_move`; finish with the required full gates.

## Completion and handoff

Do not create beads manually. Append discovered out-of-scope work using
`sase bead note bob-cli-ep.1 'PROPOSED FOLLOW-UP: <summary — evidence>'` for the epic
land agent. A failure reproduced identically on the clean base tree does not prevent
closure: record the command, failure, base evidence, and an existing tracking bead when
present. Fix failures introduced by this change.

Before closing, run `sase bead epic-symbols bob-cli-ep.1`. The planning-time check found
none, but recheck at completion. Resolve any added symbol or re-key its justfile entry
to the still-open parent/later phase before closure. Then close only `bob-cli-ep.1` with
`sase bead close bob-cli-ep.1 --note "<verified behavior and gates>"`. Do not close
bob-cli-ep or any ancestor plan bead. Do not hand-edit status. If a host-declared closed
launch is genuinely unfinished, follow the user's reopen rule; otherwise leave
already-completed work closed.

For the completed implementation's host commit declaration use the required subject:

`feat(capture)!: type tasks with #<type> and retire t:`

Include a `BREAKING CHANGE:` footer explaining that marker-only `@route+id#NAME` now
types the task instead of naming its Pomodoro, and `t:` now captures as ordinary prose.
Keep grammar, writers, completion, help, and docs in that coherent landing. Record
verification and the stable fixture/wire contract on this phase so the Mac phase can
regenerate its fixtures against the landed CLI. Finish with `/sase_final`; do not
manually commit or close ancestors.
