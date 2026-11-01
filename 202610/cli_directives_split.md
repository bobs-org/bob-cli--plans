---
tier: epic
title: 'cli-directives, split: % directives, bare-# type marker, %t templates, and
  @@ cleanup'
goal: 'Deliver phase cli-directives (bead bob-cli-ey.1) of the "Capture drawers and
  % directives" epic as a chain of coherent bob-cli changes, each small enough for
  one coding agent: delete `@@route+id`; add the `%` directive lexer with `%c`, `%n`,
  and `%l`; split clipboard parsing from reading; make a bare edge `#` an incomplete
  type marker; replace `@@template:`/`@@@` with `%t`; then add the `directive` completion
  context, `initial_selection`, and `grammar: 2`. The land agent closes bob-cli-ey.1.

  '
phases:
- id: global-route-id-removal
  title: Delete the @@route+block-id draft-wide form
  depends_on: []
  size: medium
  description: 'global-route-id-removal: delete `@@route+id` end to end (parent plan
    §1.5): `ParsedGlobalDestination.block_id`/`mode`, the `draft.rs` `+` branch, the
    `GlobalSubBulletRoute`/`GlobalSubBulletBlockId` spans and `Need::Task` from `classify_global_token`,
    `global_destination.mode`/`block_id` in capture-parse and capture JSON, the "under
    ^id" human output, `capture_related/draft.rs` handling, and its tests and docs.
    `@@route+id` now fails with the generic `@@` declaration error. Commit as `feat(capture)!:`.'
- id: directive-lexer-clip
  title: '% directive lexer with %c/%c<N>, and clipboard parsing split from reading'
  depends_on:
  - global-route-id-removal
  size: medium
  description: 'directive-lexer-clip: add `capture_language/directive.rs` (edge position
    rule, transparency, bare `%` incomplete with need `directive`, `unknown_directive`,
    `\%` escape, duplicate refusal, `directive` span) recognizing `%c` and `%c<N>`;
    delete `%`/`%N`/`%header`, `parse_clip_token`, `ClipRequest` headers, `is_valid_header`,
    `ClipOutput.header`, and `CaptureParseOptions.parse_clip_markers`; make `-n/--no-clip`
    mean never-read (`clip.status: not_read`, `clip_not_read` refusal) and `-c/--clip`
    a plain flag; add a "Directives (`%`)" docs section. Commit as `feat(capture)!:`.'
- id: note-and-listen-directives
  title: '%n replaces the trailing # Pomodoro note; %l replaces URL %/URL @'
  depends_on:
  - directive-lexer-clip
  size: medium
  description: 'note-and-listen-directives: teach the directive lexer `%n` and `%l`;
    route `%n` to the existing Pomodoro-note path with its refusals (routes, forced
    route, `s:`, `$`, `#type`; `@@route` does not apply) and delete the trailing-`#`
    marker; route `%l` to ref listen (bare-URL items only, else `directive_not_applicable`)
    and delete `is_standalone_listen_marker` everywhere, including `gkeep/plan.rs`;
    update help, docs, and tests. Commit as `feat(capture)!:`.'
- id: bare-hash-type-marker
  title: A bare edge
  depends_on:
  - note-and-listen-directives
  size: small
  description: 'bare-hash-type-marker: a free-standing bare `#` at an item edge becomes
    an incomplete `task_type` marker (span on the `#`, need `task_type` mode `incomplete`,
    execution refuses with copy that teaches `#fix` and `\#`); delete the "bare hash
    stays the Pomodoro note" branch in `type_marker.rs` and the bare `#` in `is_trailing_transparent`;
    leave `#`+digit, `##…`, `#-…`, mid-line, and glued `#` unchanged. Commit as `feat(capture)!:`.'
- id: percent-t-templates
  title: '%t frames templates; delete @@template:, @@@, and the @@ template candidate'
  depends_on:
  - note-and-listen-directives
  size: medium
  description: 'percent-t-templates: add `%t` to the directive lexer and frame the
    draft''s first item as the template (parent plan §1.4) with offset-preserving
    masking; `template_directive_misplaced`; refuse row values that inject `%c`/`%t`;
    render shelf `show`/`draft` with ` %t` and keep shelf IDs unchanged; delete the
    `@@template:` header, the `template_header` span, `@@@`/`start_template` rewrite,
    and the `@@` template completion candidate; update help, docs, the calibrate script,
    the runner example, the Bulk Capture glossary strand, and note open shelf beads.
    Commit as `feat(capture)!:`.'
- id: directive-completion-grammar
  title: directive completion context, bare-# menu, initial_selection, and grammar
    2
  depends_on:
  - bare-hash-type-marker
  - percent-t-templates
  size: medium
  description: 'directive-completion-grammar: add the `directive` completion context
    (`capture_complete/directives.rs`: rows c, n, t, then l on bare-URL items, bob-authored
    names/summaries/details/disabled reasons, no clipboard read); open the `task_type`
    context on a bare edge `#` with every row; emit `initial_selection: "none"` only
    for an empty query opened by bare `#` or `%`; add `GRAMMAR_REVISION = 2` as top-level
    `grammar` in capture-parse and capture-complete; zsh `%<TAB>` and bare `#<TAB>`;
    finish docs (Grammar at a glance), the Capture glossary strand, and the retirement
    grep. Commit as `feat(capture):`.'
proposed_by: bbugyi200.apollo.bob-cli-ey.1
proposed_with: claude/opus
parent_bead: bob-cli-ey.1
create_time: 2026-10-31 12:58:12
status: done
bead_id: bob-cli-ey.1.1
---

- **PROMPT:** [prompts/202610/cli_directives_split.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202610/cli_directives_split.md)
- **PARENT:** [202610/capture_drawers_and_directives.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/capture_drawers_and_directives.md)
- **BEAD:** [bob-cli-ey.1.1](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-ey/bob-cli-ey.1.1.md)

# Plan: `cli-directives`, split into coherent bob-cli changes

## Context

Bead `bob-cli-ey.1` is phase `cli-directives` of epic `bob-cli-ey`
(`plan:202610/capture_drawers_and_directives.md`, "Capture drawers and `%` directives").
That epic plan is the **normative contract** for every phase below: read its "The
language after this epic" table, the "Ground rules for every phase", §1 (`%`
directives), §5 (unchanged on purpose), §7 (memory), and §8 "Phase `cli-directives`"
before starting. This child plan does not restate the contract; it only slices it into
pieces one coding agent can finish, and says which section each piece owns. Where this
plan and the parent plan seem to differ on contract details, follow the parent plan.

The phase touches the whole capture language (`src/native/capture_language/`, ~29k
lines), `capture`, `capture-parse`, `capture-complete`, `capture-rewrite`,
`capture-templates`, `capture-related`, `gkeep`, zsh completion, and the 7,400-line
`docs/capture.md`. One agent cannot do all of it, so it ships as six commits on master.

The epic's DECISIONS are final and already answered (`memory_glossary_strands = yes`
authorizes the **Capture** and **Bulk Capture** strand edits in this phase; the drawer
term, grammar record, and Annotation Section strand belong to later sibling phases and
are **not** edited here). No other memory may be edited.

## Coherence rule for this split

Bryan's hosts track master, and the parent plan requires that grammar, writers,
completion, help, docs, and tests never disagree at any pushed commit. Each phase below
is therefore a self-contained grammar change:

- Each phase updates the code, `--help` text (then `just docs-gen`), the hand-written
  docs sections it affects, and the tests for exactly the forms it touches, and leaves
  every form it does not touch working and documented as it is today.
- The directive lexer grows one letter per phase. A letter a phase has not added yet is
  simply `unknown_directive` (a valid current-grammar error), and the docs do not
  mention it until the phase that adds it.
- `grammar: 2` is emitted only by the last phase, because Bob Mac Capture gates on it to
  mean "all of `cli-directives` is present".
- Breaking phases commit with `feat(capture)!:` and a `BREAKING CHANGE:` footer listing
  the removed forms and wire renames of that phase. Never edit `Cargo.toml`'s version or
  `CHANGELOG.md`.
- Follow "No compatibility" in the parent plan: delete retired code, spans, codes,
  options, docs, and tests outright; no aliases; no error copy naming a retired
  spelling. Narrow negative tests phrased with the current error codes are fine.
- Every phase ends with `just check` and `just docs-check` passing. Read
  `sase memory read cli_rules.md` before touching CLI options and
  `sase memory read docs.md` before touching docs.

## Phase details

### 1. `global-route-id-removal` (parent §1.5)

Delete the `@@route+block-id` form; `@@route` (draft-wide route) is unchanged.

- `capture_language/draft.rs`: the `+` branch (~1259–1300) and
  `ParsedGlobalDestination.block_id`/`mode`; every consumer (`draft.rs` ~698, ~904).
- Spans `GlobalSubBulletRoute`/`GlobalSubBulletBlockId` and `Need::Task` from
  `classify_global_token` (`editor_classify.rs`, `editor_model.rs`, `editor_parse.rs`).
- JSON: `capture_parse.rs` `GlobalDestinationParse` and `capture/output.rs`
  `GlobalDestinationSummary` lose `mode` and `block_id`; the "under ^id" human output
  goes.
- `capture_related/draft.rs` handling; `capture_language/tests/globals.rs` `+` coverage;
  CLI tests that use it.
- `@@route+id` now fails with ``invalid `@@` declaration: `@@` takes a route (`@@foo`)``
  (add one narrow negative test).
- Docs: `docs/capture.md` "Global destination", `docs/reference/capture-syntax.md`,
  `docs/reference/json-outputs.md`.

### 2. `directive-lexer-clip` (parent §1.1, §1.2 for `%c`, §1.3)

Build `src/native/capture_language/directive.rs`: an offset-preserving masking lexer in
the style of `type_marker.rs`, owning the position rule, complete / incomplete / unknown
/ escaped tokens, spans, needs, errors, and an item-level `Directives { clip, … }`
result that later phases extend with `note`, `template`, `listen`.

- **Position and claim** exactly as §1.1: either edge of any physical line of the item,
  with only markers between it and the edge. Directives join the transparent set in
  `dependencies.rs` `is_trailing_transparent` (replacing the clip tokens there; the bare
  `#` stays until phase 4). A line of only directives (or `- %c`) adds no bullet.
  Mid-line `%` is prose; `50%`, URL `%25` stay prose; edge `\%…` captures without the
  backslash, mid-line `\%` is verbatim.
- **Tokens this phase:** `%c`, `%c<N>` (`%c1` ≡ `%c`; `%c0`, leading zero, N above the
  history limit invalid), bare `%` (incomplete: `directive` span without `directive`,
  need `directive`, execution refuses), anything else `unknown_directive` — including
  `%n`, `%t`, `%l` until their phases. Duplicates: `duplicate_capture_marker` with label
  `directive (%c)`.
- **Execution, editor, completion, templates, rewrite, and `capture-related` all see the
  masked view.** Route the `%c` result into the existing clip plan (`capture/plan.rs`
  ~1360–1410) with today's composition refusals, now with `%c` copy.
- **Delete** `markers.rs` `parse_clip_token` and the clip part of
  `extract_terminal_markers`, `ClipRequest::Current{header}`, `capture_clip`
  `is_valid_header`, `ClipOutput.header`, `CaptureParseOptions.parse_clip_markers` (≈55
  call sites across `type_marker.rs`, `markers.rs`, `draft.rs`, `line.rs`, `item.rs`,
  `model.rs`, `capture/plan.rs`, tests) and the template force-on path, and
  `--clip=HEADER`.
- **Clipboard reading (§1.3):** `-n, --no-clip` = never read; dry run reports
  `clip: {"status": "not_read", "count": N}`; real capture refuses `clip_not_read` with
  the parent's exact copy. `-c, --clip` is a plain flag forcing a read as if `%c` were
  written. Read clips report `"status": "read"` plus today's fields, no `header`. Dry
  runs never write snippet or attachment files (add a test). Preview and capture use the
  same `capture_clip/clipboard.rs` path (`BOB_CLIPBOARD_CMD`,
  `BOB_CLIPBOARD_HISTORY_CMD`). Update `capture_flag_guard.rs`, `completion/kinds.rs`.
- **capture-parse spans:** add the `directive` span (`directive: "clip"`, optional
  `count`); remove the `clipboard` span.
- **Docs/help:** `capture/cli.rs` help (the `--no-clip`/`--clip` paragraphs ~112–116),
  `capture_parse.rs` Modes/Needs list; `docs/capture.md` gets a short "Directives (`%`)"
  section (that later phases extend) and its Clipboard section and options/JSON sections
  are rewritten for `%c`;
  `docs/reference/{capture-syntax, json-outputs,configuration}.md`;
  `docs/getting-started.md` and `docs/guides/recover-mac-capture.md` where they show
  `%`.
- **Tests:** rewrite `tests/cli/capture/clip.rs`, the clip parts of `bare.rs`,
  `batch.rs`, `parse.rs`, `capture_language/tests/*`; cover bare `%`, unknown letters,
  `%c3`/`%c1`/`%c0`, duplicates, `\%`, `50%`, URL `%25`, a `not_read` dry run, a
  resolved dry run through the env commands writing no files, the `--no-clip` + `%c`
  refusal, and non-overlapping spans for every placement.

### 3. `note-and-listen-directives` (parent §1.2 for `%n`/`%l`)

- **`%n`:** add to the lexer; route to the existing Pomodoro-note path
  (`PomodoroSelection::CurrentOrLastCompleted`, unchanged). Refuses routes, forced
  `--route`, `s:`, `$`, and `#type` with today's Pomodoro-note conflict codes and `%n`
  copy; a draft-wide `@@route` does not apply to a `%n` item; `%n %c` composes. Delete
  `markers.rs` `is_pomodoro_note_marker`, the `line.rs` 62–99 branch, and the
  `editor_classify.rs` `pomodoro_note_marker_parse` slot. Remove the `pomodoro_note`
  span (keep the `pomodoro_note` editor mode and capture kind).
- **`%l`:** add to the lexer; applies only to an item the reference claim takes (a bare
  URL, or one line of a bare-URL list), else `directive_not_applicable`
  (`` `%l` queues companion audio for a bare URL ``). Delete `url_routing/intent.rs`
  `is_standalone_listen_marker` and every caller (`item.rs`, `editor_parse.rs`,
  `completion.rs` `cursor_in_listen_marker`, `gkeep/plan.rs`), re-expressing them
  through the directive result. Remove the `ref_listen` span; keep the semantic
  `ref_listen` boolean with its doc comment updated to `%l`.
- **Docs/help/tests:** `docs/capture.md` Pomodoro notes and reading-queue listen
  sections (linking the Directives section), `docs/gkeep.md` if it shows `URL %`,
  `docs/reference/capture-syntax.md`; rewrite `tests/cli/capture/ref.rs`, Pomodoro-note
  suites, `capture_language/tests/ref_grammar.rs`, gkeep tests. Cover the `%n` fallback
  order and no-Pomodoro message, `%n` refusals, `%l` only on URLs.

### 4. `bare-hash-type-marker` (parent §1.6, lexing half)

- A free-standing bare `#` at an item edge is an incomplete `task_type` marker:
  `task_type` span on the `#`, need `task_type` (mode `incomplete`), execution refuses
  with copy like
  `` `#` is incomplete: add a task type (`#fix`), or write `\#` to keep a literal `#` ``.
  A leading `# Heading text` is incomplete; `\# Heading text` is literal.
- Delete the "bare hash stays the Pomodoro note" branch in `type_marker.rs` (~151–159)
  and `text == "#"` in `is_trailing_transparent`.
- Unchanged: `#`+digit, `##…`, `#-…`, mid-line `#`, every glued `#`.
- Docs: the `#<type>` sections of `docs/capture.md`, `docs/task-types.md`,
  `docs/reference/capture-syntax.md`. Tests in `capture_language/tests/type_marker*.rs`
  and `tests/cli/capture/capture_type_marker.rs`.
- The completion menu for a bare `#` is phase 6.

### 5. `percent-t-templates` (parent §1.4)

- Add `%t` to the lexer. Frame the draft's first item (parent line plus authored
  children) as the definition when `%t` sits at either edge of any of its lines; rows
  are the nonblank lines after the first blank line. Mask the token with offsets
  preserved; hand `capture_language/template.rs` the item with the token and one
  adjacent separator space removed. All row rules, limits, `-E/--expand`, and capture(D)
  = capture(expand(D)) are unchanged.
- `%t` in a later item: `template_directive_misplaced`
  (`` `%t` marks the first item as the template ``), replacing
  `template_header_misplaced`/`template_header_missing`; `capture_parse.rs`
  `headerless_template_hint` teaches `%t`. `%t` refuses `%c` (`template_clipboard`),
  session operators, and `@@` declarations with today's `template_*` codes; `%n`/`%l`
  inside a template apply per expanded row. A row value injecting `%c` or `%t` at an
  expanded item's edge is refused.
- **Shelf (`capture_templates/command.rs`, `store.rs`):** IDs must not change — test
  that `{0} @x %t` + rows yields the same ID as the stored definition `{0} @x`. `show`
  and `draft` render the definition with ` %t` appended to its first line, then a blank
  line. `save` refuses non-template drafts with copy teaching `%t`.
- **Delete:** `template.rs` `HEADER` and all `@@template:` handling; the
  `template_header` span; `RewriteRule::StartTemplate`/`start_template` (`@@@`) in
  `capture_rewrite.rs` and `capture_language/rewrite.rs` (separator toggles stay); the
  `@@` template completion candidate (`capture_complete/engine.rs`
  `template_completion_gate`, `RouteCandidateKind::Template`, `shell.rs` template
  branches).
- Update `scripts/dev/capture_related_calibrate` and the `src/runner.rs` template
  example; help in `capture_rewrite.rs`, `capture-templates`, `capture_parse.rs`;
  `docs/capture.md` Bulk templates / Saved templates / capture-rewrite /
  capture-templates sections; `docs/reference/cli/*` via `just docs-gen`;
  `docs/_meta/search_smoke.yml` (add a `%t` query).
- **Memory (authorized by `memory_glossary_strands`):** through `/sase_memory_write`,
  update `glossary:bulk-capture` exactly as parent §7 says, then `sase memory init` and
  `just docs-gen`.
- **Beads:** add a note to each open bead that still names `@@template:` (at planning
  time `bob-cli-82`, `bob-cli-83`, `bob-cli-8h`, `bob-cli-8i`; check `bob-cli-8b`'s land
  too) saying `%t` replaced it and shelf IDs are unchanged.
- **Tests:** rewrite
  `tests/cli/capture/{template,parse_template,template_expand, rewrite}.rs`,
  `capture_language/tests/{template,rewrite}.rs`, `tests/cli/capture_templates.rs`, and
  the `@@` parts of `capture_complete/tests/routes_tasks.rs`; cover `%t` leading,
  trailing, and on a child line; misplaced `%t`; injected `%c`/`%t`; limits; `-E`
  parity; shelf IDs unchanged.

### 6. `directive-completion-grammar` (parent §1.6 completion half, §1.7)

- **`directive` context:** new `capture_complete/directives.rs` and the
  `CompletionContext`/`Candidates`/engine/`render.rs`/`shell.rs` plumbing. Rows have
  `token`, `name` ("Clip", "Pomodoro note", "Template", "Listen"), `summary`, optional
  `detail`, optional `disabled_reason`, `replacement`, `cursor_after`; always in order
  c, n, t, with l only on bare-URL items; a typed letter filters. Details are cheap and
  bob-authored: `%c` explains `%c3`; `%n` names the receiving Pomodoro (for example
  `→ 14:00–14:25 BUILD · running`) or why there is none; `%t` explains rows; `%l` says
  whether a listen command is configured. Never read the clipboard. Disabled rows
  explain composition rules (`%n` next to a route, `%t` on a later item, `%c` in a
  template).
- **Bare `#`:** the `task_type` context with an empty query and every row in config
  order; subjects that cannot take a type (`%n` items, session operators) report through
  the existing refused-subject machinery.
- **`initial_selection: "none"`** top-level, only for an empty query opened by a bare
  `#` or `%` (the `@note+id#` case belongs to sibling phase `cli-drawers`); omitted
  otherwise.
- **Grammar revision:** one constant (for example
  `capture_language::GRAMMAR_REVISION = 2`) feeding top-level `grammar` in
  `capture-parse` and `capture-complete` JSON; `schema_version` stays 1.
- **zsh:** `%<TAB>` completes c, n, t, l with descriptions; a bare edge `#<TAB>`
  completes types.
- **Docs:** `docs/capture.md` "Grammar at a glance" gets the three rules from the parent
  plan ("@ says where, # names, % says how"); `docs/completion.md`;
  `docs/reference/{json-outputs,compatibility}.md`; `capture_complete/cli.rs` help;
  `docs/_meta/search_smoke.yml` directive queries.
- **Memory (authorized by `memory_glossary_strands`):** through `/sase_memory_write`,
  update `glossary:capture`'s live-preview clause exactly as parent §7 says, then
  `sase memory init` and `just docs-gen`.
- **Tests:** `capture_complete/tests/*` and `tests/cli/capture/complete_*.rs`: directive
  rows, filtering, URL-only `%l`, disabled reasons, `%n` detail with and without a
  Pomodoro, bare-`#` rows, `initial_selection` present only on empty `#`/`%`, `grammar`
  on both commands.
- **Retirement sweep:** the parent's done-when `git grep` (outside `CHANGELOG.md` and
  `sase/`) finds only narrow negative tests.

## Land agent

After all six phases close:

1. Run `just check` and `just docs-check` on master.
2. Run the parent's done-when grep:
   `git grep -n -e '@@template' -e '@@@' -e start_template -e template_header -e TemplateHeader -e global_sub_bullet -e GlobalSubBullet -e parse_clip_token -e is_pomodoro_note_marker -e is_standalone_listen_marker -e 'clip=HEADER'`
   outside `CHANGELOG.md` and `sase/`; only narrow negative tests may remain.
3. Smoke against a scratch vault with `bob capture --dry-run`, `capture-parse`, and
   `capture-complete`: `jot idea %n`, `Build output %c3` (with
   `BOB_CLIPBOARD_HISTORY_CMD`), `{0} @job+search %t` plus rows,
   `https://example.com/essay %l`, a bare `#`, a bare `%` (both report
   `initial_selection: "none"`), and confirm `grammar: 2`.
4. Confirm no `Cargo.toml` version or `CHANGELOG.md` edits and the `!` subjects on the
   breaking commits.
5. Run `sase bead epic-symbols bob-cli-ey.1`; resolve or re-key any leftovers to a
   still-open bead.
6. Close phase bead `bob-cli-ey.1` with a note summarizing the verification, and record
   a note on it that `mac-directives` may now start (the Mac needs `grammar >= 2`). Do
   **not** close epic `bob-cli-ey`.
