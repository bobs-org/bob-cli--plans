---
tier: tale
title: Complete task drawers and capture grammar revision 3
goal:
  Make task drawer capture, creation, completion, retyping, CLI output, and
  documentation agree with the approved bob-cli-ey.2 contract.
size: medium
proposed_by: bbugyi200.apollo.bob-cli-ey.2
proposed_with: codex/gpt-6-astra
bead: bob-cli-ey.2
create_time: 2026-10-31 23:33:23
status: wip
---

- **PARENT:**
  [202610/capture_drawers_and_directives.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/capture_drawers_and_directives.md)
- **BEAD:**
  [bob-cli-ey.2](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-ey/bob-cli-ey.2.md)

# Complete bob-cli-ey.2: task drawers and grammar revision 3

## Assignment and sizing

Implement and verify the `cli-drawers` phase, **bob-cli-ey.2**, of
`plan:202610/capture_drawers_and_directives.md`. Read that plan with
`sase artifact read` and the assigned bead with `sase bead read` before coding. The
parent plan's §§2–3 and its `cli-drawers` acceptance criteria are normative.

This is a single-agent medium tale after investigation: the task-child scanner,
indentation/insertion helpers, staged transaction machinery, type-field writer,
live-link relocation, and completion infrastructure already exist. The new catalog can
reuse the relation-label cache design. Keep the externally visible grammar, writer,
completion, CLI, documentation, and tests in one coherent change. Splitting the public
switch would publish contradictory meanings for the same `#` token.

The planning checkout was clean at `b290d833`. Prerequisite bob-cli-ey.1 is closed; its
`%` directives, bare-`#` type completion, and grammar revision 2 are present.
`bob-cli-ey.2` is in progress with no prior implementation notes. Its drawer work has
not landed: `capture_task_sections.rs`, prefix matching, attached type markers,
`--task-section`, and `task_section` completion still exist. An initial
`sase bead epic-symbols bob-cli-ey.2` found no entries. Recheck current state before
editing and preserve any subsequently landed prerequisite work.

Only this phase is assigned. Project-task H2 drawer routing belongs to bob-cli-ey.4;
this implementation must retain the specified `^prj` refusal. Mac UI and plugin parity
changes belong to other phases. Do not edit another repository, the Cargo package
version, CHANGELOG.md, immutable plan/research/decision history, or
`tests/fixtures/pomodoro_themes/vectors.json`.

## Result visible to the user

`Postgres 17 minimum @foo+bar#requirements #fix` adds a plain note to the REQUIREMENTS
drawer of `^bar`, creating that drawer if needed, and retypes `^bar` to FIX in the same
transaction. A live Task Link follows the existing type relocation rules. This child
capture does not change the parent's lane.

The glued suffix always means drawer, regardless of body text. An empty `#` opens drawer
completion; a drawer capture with no parent note text is incomplete even if it has
authored children. Separate `#fix` remains a type marker, including on `@foo+bar #fix`
(the existing Ensure Next behavior) and `Fix the parser @sase^parser #fix` (new task
creation).

Completion presents existing drawers, common drawers from other notes, and an explicit
new-drawer row. Preview reports the destination, whether the drawer will be created, and
any type change together. `capture-parse` and `capture-complete` report `grammar: 3`;
their `schema_version` stays 1.

## Existing code and implementation seams

| Concern                                                  | Existing entry points                                                                         |
| -------------------------------------------------------- | --------------------------------------------------------------------------------------------- |
| Task child scanner, title grammar, matching, listing CLI | `src/native/capture_task_sections.rs`                                                         |
| Child insertion and managed-log anchors                  | `src/native/capture/sub_bullet.rs`                                                            |
| Route parsing, name literals, type masking               | `src/native/capture_language/{tokens,type_marker,pomodoro_literal,item}.rs`                   |
| Editor spans, needs, completion field geometry           | `src/native/capture_language/{model,editor_model,editor_classify,editor_parse,completion}.rs` |
| Shared grammar revision                                  | `src/native/capture_language/mod.rs`                                                          |
| Forced targets, child planning, output assembly          | `src/native/capture/{cli,plan,mod,output}.rs`                                                 |
| Existing-task retype and live-link relocation            | `src/native/capture/task_theme.rs`                                                            |
| Menu candidates and type-subject discovery               | `src/native/capture_complete/{candidates,model,engine,task_types,render,shell}.rs`            |
| Cache precedent and successful-commit invalidation       | `src/native/relation_labels/scan.rs`, `src/native/capture/commit.rs`                          |
| Command registration and shell values                    | `src/{native,runner}.rs`, `src/native/completion/{tree,kinds,providers,context,present}.rs`   |

Use the existing transaction planner throughout. Drawer menus and writes must share key
normalization, exact matching, refusal decisions, and replacement construction; a menu
choice is never authority to skip commit-time resolution.

## 1. Drawer model, names, and resolver

Rename the task-section module and its capture-specific types/functions to drawer
terminology, preferably `capture_drawers` with small catalog/resolution submodules as
needed. Keep `is_pomodoro_name` and its `+` allowance intact. Rename the title predicate
to `is_drawer_title` without broadening the ALL-CAPS title grammar.

Use `pomodoro_literal.rs::lex_exact_name_after_hash` for quoted names, sharing the
existing lexer rather than inventing a second quoting implementation. Represent selector
intent explicitly: a slug versus a quoted/CLI exact title.

- A slug starts with an ASCII letter and continues with ASCII letters, digits, or
  hyphens. Unquoted punctuation and `+` are invalid drawer syntax.
- ASCII and curly quoted literals uppercase and collapse whitespace, then must pass
  `is_drawer_title`. Examples: `#"NON-GOALS"`, `#"Q&A"`, `#"API (V2)"`.
- A title key is uppercase with collapsed whitespace. A literal matches that exact key.
  A slug tests its own key and the key with hyphens replaced by spaces. Preserve the
  difference between `FUTURE-WORK` and `FUTURE WORK`.
- An empty suffix is incomplete (`drawer`); invalid suffixes use `invalid_drawer` with
  examples of current syntax.

Scan only direct non-checkbox ALL-CAPS child bullets of actual `#task` lines, excluding
fenced text and all managed lines/logs. Retain original title, key, slug,
line/indent/block geometry, and direct entry count. A nested task's drawers belong to
that task; an empty existing drawer is still eligible.

Resolve against the current staged note in this order:

1. Exactly one key match selects that drawer; append before its nested managed log using
   the existing insertion helper.
2. Multiple matches return `ambiguous_drawer` with conflicting line numbers. This
   includes duplicate titles and the two hyphen interpretations.
3. No match and a reserved key returns `reserved_drawer`.
4. No match for a slug that is a strict prefix of, or edit distance 1 from, an existing
   drawer key returns `drawer_near_duplicate`. Suggest the full selector and show a
   quoted literal as the deliberate creation override. Quoted literals and `--drawer`
   skip this guard.
5. Otherwise create a drawer. Literals use their canonical title; slugs prefer the
   catalog's most-used matching spelling, falling back to uppercase with hyphens
   replaced by spaces. Use stable tie breaking for equal frequencies. A new key at edit
   distance at most 2 from a catalog key used in at least two notes emits
   `drawer_near_catalog` with the candidate title.

Derive reserved keys from the existing dependency and relation label constants, managed
log labels, and `TASKS`. Include Cancel Log without making it a new insertion anchor:
`MANAGED_TASK_LOG_LABELS` currently covers Schedule, Work, and Rotten only. Locate the
authoritative cancel label; if a needed label or structural heading is embedded in a
writer, expose a shared constant at its owner rather than introduce a disconnected
duplicate list.

For creation, use the plain-child insertion anchor: after existing ordinary direct
children and a Depends-On first child, before the first direct Schedule/Work/Rotten log.
Use existing indentation and line-ending helpers. Add the new ALL-CAPS bullet and its
first nested entry together. Authored children and clip output nest one level below that
entry. The entry has no checkbox, `#task`, or created field.

Identify a project note's own `^prj` with the existing project predicates, not merely an
arbitrary block ID string. Drawer writes to that target refuse with
`use @<note>#<Heading> for a project section`. Plain `@proj+prj` stays as it is.

## 2. Cached catalog

Implement the catalog using `relation_labels/scan.rs` as the cache precedent:

- `bob_cli_cache_dir()/drawers/catalog-v<N>-<sha256(vault)>.json`, per-note mtime and
  size fingerprints, a 60-second snapshot TTL, atomic replacement, and old cache-version
  cleanup. Inject time or use the existing clock seams in tests.
- Scan task drawers and H2 headings in project notes. Exclude managed/reserved names and
  structural H2s: Tasks, status groups, Pomodoros, Events, Highlights. Reuse the owners'
  constants and project-note recognition.
- Exclude dot directories and `_templates/`. Include `done/` for all-time frequency
  only, never recency. Skip fenced headings and drawer-looking prose.
- Count distinct notes per key (including once when a note repeats a title). Track
  spelling usage for creation separately from distinct-note ranking. Rank by distinct
  non-done notes modified within 90 days, then total distinct note count, then latest
  mtime, with a stable final tie breaker.
- Empty-query Recent contains at most eight keys, each used in at least two notes. Typed
  queries search the full catalog, prefix before substring, on key or slug. Remove keys
  already represented on the target task.
- Invalidate after a successful capture commit next to relation-label invalidation.
  Failures and dry runs must not mutate vault files or eagerly invalidate as if a commit
  occurred. Cache files may be refreshed by read-only catalog calls.

The target's drawer scan reads its own note directly and does not depend on catalog
freshness. Reuse cached catalog snapshots for warm completion; do not perform a second
full vault scan to resolve a single target or key. Catalog data guides spelling and
suggestions, never whether a drawer exists on the task.

## 3. Grammar and editor switch

Remove attached-type interpretation from `type_marker.rs`, including
`TypeMarkerAttachment::{ExistingTask,ProjectTask}`, its enum when unused, attached
candidate construction, item-sensitive toggle masking, and attached completion
precedence. Preserve unrelated dependency-target `ExistingTask` variants and behavior.

In `tokens.rs` and `item.rs::resolve_sub_bullet_kind`, a `+id#...` token always stays a
drawer capture. If the parent body is empty after masking all markers, return need
`drawer_text` and refuse execution with an example such as
`add the note text (Postgres 17 minimum @foo+bar#requirements)`. Children, type markers,
relationships, and clip directives must not accidentally send this through Ensure Next
or relationship-only fast paths. Never create an empty drawer.

Refuse `@route^id#...` with the parent plan's R3 message teaching a separate `#fix`.
Preserve `@r^id+#NAME`, its misordered `@r^id#NAME+` error, ordinary `@note#Heading`,
Pomodoro names, separator toggles, and free-standing type markers. Refuse
`@r+id#drawer!` with copy saying that the `!` toggle takes no drawer.

Update editor classification, span serialization, needs, completion fields, template
source mapping, and parse help together:

- `drawer` replaces `sub_bullet_section`; its span covers `#` through the name, includes
  `title` and `key` when valid, and never overlaps the block-ID span.
- Add `drawer` and `drawer_text` needs, removing `task_section`.
- Complete, empty, partial quoted, invalid, and marker-only forms retain usable source
  ranges. Quoted names with spaces remain a single destination token.
- Parsing stays lexical and performs no target/catalog lookup.
- Set the shared `GRAMMAR_REVISION` to 3 for parse and completion; leave
  `schema_version` at 1.

## 4. Staged writes, retyping, and previews

Wire the resolver into `capture/sub_bullet.rs` and the new selector into forced target
handling in `capture/plan.rs`. Re-read staged text for every batch item, including after
any type-field or relationship edit that changes offsets. Two items targeting the same
new drawer must create one header.

Accept a separate `#type` for body-bearing explicit `@route+id` targets, with or without
a drawer. Update shared subject eligibility, forced-target validation, editor
classification, and `capture_complete/task_types.rs` so the subject is `existing` and
reports the actual target's current/inherited type. Avoid relying only on
`dependency_target`, which is absent for ordinary child captures.

Use `CaptureTaskThemeContext::apply_type_marker_to_existing` before the child write and
`relocate_for_retype` for a live link. Preserve the existing rule that choosing an
inherited type clears an explicit override. Reuse the existing summary/relocation
fields. A body-bearing child capture does not Ensure Next, create an unrequested Task
Link, or change the status lane. The separate marker-only `@route+id #type` path retains
its current lane behavior.

Keep all note edits, live-link moves, attachments, and authored children within the
existing batch transaction. Invalid/unconfigured types, closed retype targets,
strict-cap refusals, stale task refs, conflicting drawer matches, or later-item errors
leave the vault unchanged. Retain stale-safe `--task-ref` selection and existing
identity rules; do not silently mint IDs to simplify drawer insertion.

Replace capture output's `parent_section` with per-item `drawer: {title, key, created}`,
including the legacy first-item mirror in batch JSON. Plain-child and drawer captures
carry the type summary when applicable. Human output and dry-run preview show both
effects together, with a new-drawer indicator and any Task Link movement. Keep preview
and commit on the same planner.

## 5. Drawer menu and CLI

Replace `CompletionContext::TaskSection` and its candidate variant/dispatch with
`Drawer`/`drawer` in editor completion, engine, render, and shell code. The replacement
range covers the entire `#name`, including quotes, and replacement text has no trailing
space. Include `cursor_after` from bob.

The completion envelope adds `target: {route, block_id, text}` for the drawer context.
An empty drawer query sets `initial_selection: "none"`; handle this explicitly because
the existing helper requires a whitespace boundary before bare `#`, whereas a drawer `#`
is glued to an ID. Existing bare type/directive selection behavior remains correct.

Return rows in groups:

1. `task`: matching drawers on the target, in document order;
2. `recent`: matching catalog keys absent from the task, in catalog order;
3. `new`: one nonempty typed query with no exact key among the other rows.

Rows contain `title`, `key`, `group`, task `entry_count` or catalog
`note_count`/`last_seen` as appropriate, `detail`, `creates`, optional `refused`,
`consequence`, `replacement`, and `cursor_after`. Write singular/plural-aware copy in
bob. Examples are `3 notes`, `used in 12 notes · 3 days ago`,
`Adds after "2 GB RAM" in REQUIREMENTS`, and `Creates FUTURE WORK under ^bar`. Use the
resolver to mark reserved/near-duplicate/ambiguous choices refused. A missing,
duplicate, non-task, or unsupported project target produces a diagnostic and no
actionable creation row.

Generate `#slug` only when it resolves unambiguously to the intended title; otherwise
use `#"TITLE"`. Check ambiguity against the full target, not just filtered rows. The
quoted override remains available for deliberate creation.

Replace `-S/--task-section` with `-D/--drawer TITLE`, requiring `--route` and `--task`
or `--task-ref`. Treat the CLI value as an explicit title: exact key matching and
creation, with reserved/ambiguous refusals but no near-duplicate guard. Delete the old
option and update every conflict/teaching message.

Register canonical `bob capture-drawers` alphabetically in CaptureProtocol. Keep
`capture-task-sections` only as the permanent hidden command alias, with byte-identical
canonical behavior and output, including diagnostics/help. The arguments remain those of
the existing listing command. Its JSON is:

```text
{ok, schema_version: 1, route, block_id, ref, count,
 drawers: [{title, key, slug, line, entry_count}],
 recent: [{title, key, note_count, last_seen}]}
```

Update top-level registration, runner dispatch, shell command/value completion,
`--drawer` suggestions, `justfile` install smoke, help-order tests, and fixtures. Apply
`cli_rules.md`: alphabetical options, short aliases, canonical names in messages, and no
capture subcommands.

## 6. Documentation and inherited memory decisions

Update affected help and contracts in the same implementation: `docs/capture.md`,
`docs/reference/capture-syntax.md`, `docs/completion.md`, `docs/task-types.md`,
`docs/projects.md`, `docs/reference/json-outputs.md`, and `docs/_meta/search_smoke.yml`.
Replace attached-type examples with separate types and describe drawers in
promotion/reversal documentation. Preserve plain headings, annotation sections, Pomodoro
names, and status-group terminology.

Generate `docs/reference/cli/capture-drawers.md` in place of the old command page,
update `mkdocs.yml` navigation, and add the published-page redirect from
`reference/cli/capture-task-sections.md`. Regenerate help fixtures through the existing
tooling rather than editing generated pages manually.

The parent epic's memory decisions are final and inherited; do not re-ask them. Use
`/sase_memory_write` before the following authorized edits:

| Accepted decision               | This phase's authorized strand and content                                                                                                                                                                                                                                                                     |
| ------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `memory_drawer_term = yes`      | Add `glossary:task-drawer`, alias drawer: definition, nonempty promotion/reversal, exact capture/create/menu behavior, managed exclusions, and distinctions from headings, Annotation Section, Pomodoro, and Task Type. Link the relevant terms. Omit the later phase's `^prj` H2 sentence.                    |
| `memory_grammar_record = yes`   | Add `decisions:capture-sigils-say-where-what-how`: `@` where, glued `#` place/free `#` type, `%` directives, exact commit matching and creation, menu-only prefix assistance. Include applicability, alternatives, costs, reopening condition, and evidence from the approved parent plan and research report. |
| `memory_glossary_strands = yes` | In `glossary:annotation-section`, replace `Not a Markdown ## heading or a bob capture target section` with the equivalent formatted sentence linking `[[task-drawer]]`.                                                                                                                                        |

Read those canonical notes only through `sase memory read`. Use `sase artifact read` for
`research:202610/capture_drawer_and_directive_grammar/capture_drawer_and_directive_grammar.md`
before citing its rationale. Capture and Bulk Capture already reflect `%c`/`%t` from
bob-cli-ey.1 and need no further edits here. Older accepted records remain immutable.
Run `sase memory init` after the authorized edits, then `just docs-gen`.

## 7. Verification and acceptance evidence

Extend/rename existing scanner, grammar, completion, CLI, and type-marker tests; do not
retain parallel obsolete behavior suites. Cover these independent failure modes with
focused tests and a representative end-to-end fixture:

- Exact slug and literal matching; case/whitespace normalization; duplicate titles;
  `FUTURE WORK` versus `FUTURE-WORK`; prefix/one-edit refusals; quoted and CLI override;
  reserved names including Cancel Log; catalog spelling and `drawer_near_catalog`;
  literal punctuation and invalid unquoted punctuation.
- Empty `#` and marker-only named drawers, with/without body, children, `%c`,
  relationships, and separate type markers; no toggle or empty-drawer fallback. Assert
  non-overlapping byte ranges with Unicode before and inside names.
- The user's Postgres example with existing and missing REQUIREMENTS; inherited and
  explicit FIX; live-link relocation; lane preservation; closed target, unconfigured
  type, and strict-cap refusal with no changed vault bytes.
- Tabs/spaces, CRLF, EOF without newline, nested tasks, fences, empty drawers, managed
  logs inside the drawer and under its parent, and Depends-On first child.
- Two batch/template rows creating one drawer, later failure rollback, authored children
  and `%c` nesting, and dry-run/commit result agreement. Stub clipboard commands; never
  read or mutate the user's clipboard/vault for fixtures.
- R3 refusal, `!` refusal, `^prj` refusal, unchanged project-note Pomodoro forms, plain
  child behavior, separator toggles retaining the glued suffix, and separate
  new-task/Ensure Next typing.
- Catalog cold/warm behavior, TTL, note changes/deletion, commit invalidation, cache
  version cleanup, exclusions, distinct-note counts, done-only history, 90-day ranking,
  empty-query threshold/limit, and typed prefix/substring lookup.
- Menu group order, bob-authored copy, refused rows, full-name replacement, quoted
  ambiguous names, `cursor_after`, `initial_selection`, current type subject, and
  unresolved-target diagnostics without creation rows.
- `-D/--drawer`, removed `-S`, canonical command/help visibility, hidden-alias byte
  equality, JSON fields, shell completion, and grammar 3 in both endpoints.

Use the current test harnesses under `capture_language/tests`, `capture_complete/tests`,
`capture/tests`, `tests/cli/capture`, `tests/cli/completion`, and
`tests/cli/help_options.rs`. Run focused tests while developing, then `cargo fmt --all`,
`just docs-gen`, `just check`, and `just docs-check`. Hand genuinely long gates to
`/sase_monitor` according to its instructions; wait for the handoff command itself to
exit.

Measure `capture-complete` from the built binary on an isolated apollo-sized fixture
vault; record note/task counts and setup so results are reproducible. Compare cold and
warm drawer completion with task-type completion for the same parent and equivalent
draft. Report sample count, median, and a tail percentile; exclude process build time
and distinguish first scan from warm cache. Target warm drawer overhead within about 20
ms of the type menu. Record measurements on bob-cli-ey.2 and investigate a material miss
before calling performance done.

Audit retired capture identifiers and copy with `rg`, excluding immutable history and
plan artifacts. Remove capture `task_section`, `task-section`, `TaskSection`,
`sub_bullet_section`, `SubBulletSection`, and attached-type remnants. Allowed
compatibility references are the hidden alias and its focused test, plus the published
URL redirect. Keep unrelated `task_status_hooks::grouped_task_sections` and unrelated
dependency-target `ExistingTask` variants; explain such broad-search false positives in
verification evidence rather than rewriting other domains.

## Completion protocol

Do not change the assigned bead's status by hand. Do not create follow-up beads; record
discovered work as `PROPOSED FOLLOW-UP: ...` notes on bob-cli-ey.2. If a failing check
reproduces identically on the clean base, record that evidence and any existing tracking
bead; that baseline failure does not keep this phase open.

Before closing, rerun `sase bead epic-symbols bob-cli-ey.2`. Resolve leftovers or re-key
them to a genuinely still-open owner so closing cannot stale `just check`. Once the
whole phase is implemented and verified, close **only bob-cli-ey.2** with
`sase bead close bob-cli-ey.2 --note "<implementation, checks, timing, and any baseline-failure evidence>"`.
Never close bob-cli-ey or another ancestor. Finish through `/sase_final`, using a
`feat(capture)!:` subject and a `BREAKING CHANGE:` footer naming removed attached type
forms, `-S/--task-section`, and the span/need/context/output renames. No manual commit
command is needed for the finalizer.
