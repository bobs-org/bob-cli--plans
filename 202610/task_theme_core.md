---
tier: epic
title: Rust task theme suggestions and canonical task fields
goal: Complete bob-cli-dm.1 with pure suggestion and field rules, a read-only explanation
  CLI, unknown-default hints, shared vectors, preservation coverage, and the published
  contract.
phases:
- id: suggestion-model
  title: Implement task theme rules and shared vectors
  size: medium
  depends_on: []
  description: 'suggestion-model: add validated note defaults and reading config,
    the pure resolver, canonical field edits, and the write rule, with shared Rust
    vectors.'
- id: suggestion-cli
  title: Expose suggestions and unknown note defaults
  size: medium
  depends_on:
  - suggestion-model
  description: 'suggestion-cli: ship read-only themes suggest and theme_default_unknown
    hints, wire discovery and help, and measure warm-list performance.'
- id: preservation-contract
  title: Verify field preservation and publish the contract
  size: medium
  depends_on:
  - suggestion-cli
  description: 'preservation-contract: test existing task writers, fix field loss,
    publish the task-theme contract and vault/config references, and run integrated
    gates.'
proposed_by: bbugyi200.apollo.bob-cli-dm.1
proposed_with: codex/gpt-6-astra
parent_bead: bob-cli-dm.1
create_time: 2026-10-28 09:05:49
status: done
bead_id: bob-cli-dm.1.1
---

- **PROMPT:** [prompts/202610/task_theme_core.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202610/task_theme_core.md)
- **PARENT:** [202610/task_theme_suggestions.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/task_theme_suggestions.md)
- **BEAD:** [bob-cli-dm.1.1](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-dm/bob-cli-dm.1.1.md)

# Task theme core for bob-cli-dm.1

## Objective and scope

Complete only the `suggest-core` phase of the approved epic
`plan:202610/task_theme_suggestions.md`, tracked by `bob-cli-dm.1`. The deliverable is
the Rust source of truth for task theme suggestions and deliberate exception edits, an
explanation command, unknown-default hints, shared fixtures for later JS mirrors,
preservation coverage, and documentation. This child epic has three sequential, bounded
coding phases because the algorithms, public command/performance work, and cross-writer
verification are separate substantial work units. Each phase closes only its own
assigned bead. The child epic land agent verifies completion and closes `bob-cli-dm.1`;
nobody working this child epic closes `bob-cli-dm` or any ancestor of it.

Before starting, read the assigned bead and the parent phase with audited
`sase bead read`, and read the approved design with `sase artifact read`. Recheck
whether earlier work has landed and implement only the remaining scope. At planning time
the clean checkout was at `ec018a5b`, the phase had no prior-work notes, none of the
requested suggestion symbols existed, and `sase bead epic-symbols bob-cli-dm.1` reported
no entries. Do not change bead statuses by hand.

The outer epic's decisions are final: `ledger_theme_pills=yes` and `seed_plex_chore=yes`
belong to other phases; `memory_decision_record=no` and `memory_glossary_term=no` forbid
the proposed memory additions. **Edit no memory files and do not run memory
publication.** No plugin, Mac client, capture-menu, capture-write, rename, or live-vault
seeding work belongs here. Never change the Cargo.toml version or CHANGELOG.md. Keep
wire changes additive at schema version 1.

## Existing code and integration constraints

- `pomodoro_homes/facts.rs` owns `NoteFacts::parse`; `resolve.rs` provides `HomeSource`,
  `DiskHomeSource`, cached facts, and `HomeResolver::resolve_note(...).residence_chain`.
  The walk already caps at 16, detects cycles, passes through wrappers, and gives daily
  notes no residence chain.
- `task_fields::inline_fields` returns exact byte spans for both inline-field forms. It
  currently filters by a case-sensitive key. Share its scanner if an all-fields iterator
  is needed; do not introduce a second incompatible grammar.
- `capture_pomodoros::canonicalize_pomodoro_name` is the naming authority;
  `plan_budget::{normalize_component, split_components}` own identity and splitting.
- `Catalog::theme_ref` already implements configured vocabulary and config-first hue,
  including known configured names without historical use. Preserve this behavior from
  `ec018a5b`. Suggestions never add records or usage to the catalog.
- `ref_tasks::line` owns `has_ref_token`, task marks, and blockquote handling.
  `task_status_hooks::canonical_daily_date` identifies daily notes.
- Task-toggle lookup is in `capture/task_toggle.rs`: `note_tasks::read_settings`,
  `note_tasks::scan`, `by_block_id`, and `suggest_block_id`. Reuse/extract this lookup
  and error formatting without changing capture semantics.
- `projects/scan.rs` supplies frontmatter helpers and deterministic directory
  discovery/exclusions. Its project scanner filters out area notes, so reuse the
  discovery policy, not that filtered project result, for default hints.
- `projects::task_metadata_insertion_offset` only handles a trailing block ID. It does
  **not** implement the required full trailing-metadata run. Freshness's suffix scanner
  is intentionally limited to recognized Tasks keys; do not broaden it as a shortcut for
  theme placement.
- `ListReport.hints` in `pomodoro_themes/cli.rs` is currently `Vec<CollisionHint>`.
  Introduce an additive hint representation that preserves existing collision JSON
  objects exactly while admitting the new hint object.
- CLI discovery spans `src/runner.rs`, `NativeCommand` dispatch in `src/native.rs`,
  `src/native/completion/tree.rs`, and `src/native/catalog/table.rs`; follow the
  existing themes subcommand wiring.

The approved design calls its new contract PT12, but the checked-out
`docs/pomodoro-themes.md` already ships **PT12 — Configured themes**. Preserve that
section, behavior, and anchor. Publish **PT13 — Task theme suggestions** and use PT13 in
new pointers. Record this numbering correction in the parent phase's completion note so
later outer-epic agents extend PT13. This is a documentation allocation correction; all
approved behavioral requirements remain intact.

## Shared behavioral contract

### Values, fields, and resolution

A valid theme value is trimmed, nonempty, contains no `+`, and passes canonical Pomodoro
naming. Compare normalized keys. Display the catalog spelling when known (including
configured names), otherwise the canonical uppercase spelling. Hidden themes remain
eligible; theme defaults never affect hue, history, or Stats.

Add `theme: Option<NoteTheme>` to note facts, with `Name(String)`, `Off`, and
`Invalid(String)`. Parse scalar frontmatter using the existing frontmatter helpers;
honor quotes. An absent/blank value inherits. ASCII-case-insensitive `off` stops the
walk only when declared by an area/project. Lists, composites, mappings, or bad names
are invalid, skipped, and produce one `theme_default_invalid` warning per visited
declaring note. Non-area/project `theme:` values are ignored, while their scalar
`parent:` links remain traversable. Test both flow and multiline list forms; raw
frontmatter helpers alone must not mistake a multiline value for absence.

Task fields are none, set (exactly one valid `[theme:: X]` or `(theme:: X)`), or repair.
Two or more fields mean `duplicate`; one invalid field means `invalid`. Repair falls
through to inheritance, carries `repair: "duplicate" | "invalid"` when a suggestion
exists, and is replaced by the next eligible writer. Closed tasks preserve their field
as history and have no suggestion. A no-suggestion result still returns warnings
separately; do not fabricate a theme just to carry a repair marker.

Expose pure `suggest_for_task(source, catalog, cfg, residence_path, line)` and
`suggest_for_new_task(...)` APIs in `pomodoro_themes::suggest`. Use a `HomeSource`
adapter so later staged capture callers can provide in-memory contents. Return an
optional suggestion plus warnings. Resolve in this order:

1. A set task field: source `task`, label `this task`.
2. A reading task: an exact `#ref` token, or trailing block ID `ref` / `ref-*`,
   including reread IDs. Use `plan.reading_theme`, default `READ`; `off` skips this
   step. Ordinary annotation follow-up tasks do not qualify.
3. Canonical daily residence plus case-insensitive trailing `gtd`: first `plan.exempt`
   name, source `review`, label `daily review`. Empty exempt skips it.
4. First valid note default on the residence chain: source `note`, label
   `from <lowercase declaring stem>`. Stop on `off`, continue past invalid values, and
   reuse the 16-note/cycle limits. Daily notes have no default walk.
5. Otherwise none. New tasks use step 4 from their destination only.

Reading source uses label `reading task`. The inherited suggestion runs the same rule
with step 1 skipped. The serialized suggestion contains `theme: ThemeRef`, `source`, and
`label`. Only source `note` has `from: {path, stem, kind, hue, chain}`; paths are
vault-relative, chain runs residence through declaring note, and hue is the declaring
note's root-area hue. Only source `task` has `inherited`, serialized as a suggestion
without another inherited member, or explicit null. Other optional fields are omitted.
Reuse `ThemeRef` and its `known: false` convention. A theme is known if known to the
existing catalog or present among today's live ledger components, even when the catalog
cache is stale. Unknown names remain reportable; downstream menus must never preselect
them.

Add validated `plan.reading_theme` to `PlanConfig` and its parser: absent defaults to
`READ`, `off` disables the rule, and invalid/non-string values report a normal
`plan.reading_theme` config error. Preserve unrelated config loaders' independence.

### Canonical field edit and deliberate choice

Implement `task_theme` as small pure helpers: parse state, `set_task_theme`,
`clear_task_theme`, and `plan_theme_change`. The change planner takes field state, shown
suggestion/inherited theme, resolved destination name, exempt keys, and a deliberate
flag; it returns no change, set, or clear, with enough before/after data for the later
capture JSON. Avoid a module dependency cycle between suggestion types and field
primitives.

Apply the first matching rule:

1. Not deliberate: no change.
2. Destination is unnamed, all-exempt, or contains other than exactly one non-exempt
   component: no change. `PLAN + BUILD` selects BUILD; `BUILD + FIX` does not write. Use
   normalized component identity.
3. No suggestion was shown: no change. Keep the explicit Task Card edit escape hatch
   separate/explicit for its later caller, where this guard does not apply.
4. Chosen key equals inherited: remove every theme field, no-op if absent.
5. Otherwise set catalog-spelled X, no-op if exactly one valid field already has X's
   key; do not normalize authored bytes on that semantic no-op.

Set removes all theme fields in either form plus one adjacent separating space, then
inserts ` [theme:: NAME]` before the maximal trailing run of inline fields of any key,
tags, and a trailing `^id`. With no run, insert at the trimmed end. Clear only removes
all theme fields and the adjacent separator. Preserve all other bytes, including
indentation, blockquote prefixes, tabs, trailing whitespace, CRLF, and absence of a
final newline. Do not accidentally consume the task checkbox or title as trailing
metadata. Overlapping removals from adjacent fields must be safe.

No existing gesture becomes a writer in this child epic. In the full contract,
deliberate capture means an explicit `#NAME` or equivalent `=name`; the picker means a
changed committed row, nonempty query, or Shift+Enter. Alt+Enter is always link once;
untouched preselection is not deliberate. Guards refuse before any field edit. Theme
edits never affect freshness eligibility. Close, roll, typed `^^`, move, rollover,
reconcile, ordinary successor placement, and ref-today are never writers.

## Phase suggestion-model

Implement the value parser, note facts, reading config, resolver/types, and field
helpers above. Keep the core independent of filesystem I/O beyond `HomeSource` and
independent of UI/menu code. Add only narrow API exposure needed for later phases; do
not introduce broad unused-code allowances.

Extend `tests/fixtures/pomodoro_themes/vectors.json` additively with `suggest`,
`task_field`, and `write_rule`. Keep existing sections unchanged and add Rust tests that
actually consume all three new sections. Each case has a stable descriptive ID and
explicit expected results; `suggest` includes mini-vault files/config, catalog/live-day
facts, residence and task input, expected JSON/warnings.

Required cases: task beats READ; all three reading identifiers; a non-reading follow-up;
reading off; review and empty exempt; nearest/default/off; archive and reference
wrappers; invalid scalar/list skipped with deduplicated warning; cycles and 16-note
boundary; noncanonical daily names; known, unknown, configured-but-unused, and hidden
names; quoted/blank note values; invalid/duplicate/paren fields; field clear/set/no-op
byte fixtures with tags, no ID, blockquotes, CRLF, Unicode title, unknown trailing field
keys, and fresh/Tasks suffixes; deliberate false; no suggestion; clear-to-inherited;
single, composite, exempt, and unnamed destinations.

Validation: focused config, home resolver, theme vector, task-field, and freshness
tests, then formatting and applicable lint gate. Record the exported API and fixture
shapes in the phase completion evidence. This phase does not add menus or capture
writes.

## Phase suggestion-cli

Implement `bob themes suggest TARGET [-b/--bob-dir DIR] [-f/--format human|json]`.
`route:block-id` uses the task-toggle lookup, including missing/duplicate/not-task and
close-match diagnostics. Bare route reports a new task's default. Resolve paths with
existing capture-route validation and do not allow path traversal. Report closed tasks
successfully with no suggestion and one `closed_task` warning.

JSON is
`{schema_version: 1, ok, target: {route, block_id, path, line, new_task}, theme_suggestion: <object or null>, warnings}`.
Use null block ID/line for new tasks, and the existing human-facing one-based line
convention for a found task. Keep error output structured and consistent with adjacent
commands. Human output shows the swatch/name and `suggested · <label>`; note defaults
show declaring path and chain, and task overrides show
`without it: <swatch> WRITE from sase_blog` or the equivalent inherited reason/none.
Respect NO_COLOR and broken pipes.

Wire dispatch, alphabetical help/subcommands/options, short aliases, completion tree,
and a read-only `themes-suggest` catalog entry. Update catalog/protocol count assertions
and CLI goldens. Run `just docs-gen`; inspect generated diffs and preserve canonical
memory sources. Cover command discovery/help, valid/new/closed/error targets, JSON
omission/null rules, hidden/unknown names, and byte-identical vault contents before and
after a query.

Add `theme_default_unknown` to list hints, with
`{code: "theme_default_unknown", path, name, similar_to}` for each typed area/project
note whose valid default is unknown. Render an actionable human hint. No hint for
absent, blank, off, invalid, ignored note types, or known defaults. Use shared
similarity rules, retain collision hints' existing bytes/shapes, and produce
deterministic hint order. A `--query`, `--limit`, or hidden filter must not silently
hide the vault-wide default diagnostic.

The scan follows project discovery exclusions and never writes the vault. Measure before
selecting a cache strategy: a warm full-content scan is acceptable only if it meets the
+5 ms p95 bound. If needed, add a disposable per-vault note-facts cache with per-file
mtime_ns/size and directory discovery fingerprints; edits, additions, deletions,
retyping, and default removal must be visible on the next list, and corruption/`--fresh`
must rebuild. Re-evaluate knownness from current catalog/config so new ledger use or
vocabulary changes clear hints without rewriting defaults. Do not load default-hint
scans into completion/parse hot paths.

Capture same-host optimized before/after measurements on apollo with fixed date, config,
vault snapshot, and warmed caches: at least 10 warmups and 100 samples per binary,
alternating runs when possible, report sample count, baseline/revised p95, and delta.
Use the clean pre-change revision as baseline, not an arbitrary installed bob. Use
authorized repo access for any vault/source outside the checkout; use `/sase_repo` first
and the returned path, and only edit a scratch copy inside the workspace. No live-vault
mutations. Preserve the benchmark evidence on the assigned bead. If a long build/check
needs a handoff, use `/sase_monitor` and wait for the handoff command itself to exit.

Validation: focused themes CLI and completion/catalog/help tests, hint invalidation
tests, read-only scratch-vault smoke tests, measured warm-list delta <=5 ms p95, and the
appropriate formatting/lint checks.

## Phase preservation-contract

Add meaningful end-to-end or existing writer-level regressions proving a distinctive
authored field (including spelling/spacing) survives these operations:

- task move;
- archive and reopen;
- Highlights reading-status checkbox edit;
- ref rename's alias overwrite;
- task-status-hooks reconcile;
- freshness stamping, with the theme still before the fresh/Tasks suffix.

Exercise the actual operation, not a parallel test-only formatter. Assert that only the
operation's intended bytes change; the task field is preserved, not invented,
normalized, or stripped. Where a test exposes loss, fix that writer narrowly while
retaining its existing behavior, and test closed/history fields as applicable.

Publish PT13 in `docs/pomodoro-themes.md`, preserving PT12 configured themes and all
existing published anchors. Include storage/validity/repair, precedence and inheritance,
JSON, placement, write rule/never-writers, unknown/hidden behavior, `themes suggest`,
list hints, and the Dataview caveat: `task.theme` can conflate frontmatter defaults with
task fields, so use `bob themes suggest`. Document the outer epic's shared menu contract
explicitly as the cross-client contract; avoid claiming other clients already ship
unfinished outer phases.

Menu contract: exact entries contain only S after removing exempt keys other than S;
prefer running exact, else first exact in ledger order, else first containing. Keep
Today order. Otherwise known S may create a top `New S Pomodoro` row if a Pomodoros
section exists, at most one timed open entry exists, and strict theme caps permit it;
remove its Recent duplicate. Otherwise preserve today's default and report
`unknown_theme`, `creation_blocked`, or `strict_cap`; ref-today additionally uses
`already_queued`. Nonempty query retains PT7 ranking/first-row behavior while annotating
at most one best matching row. Ctrl+D on suggested creation is consumed without writing.
Later completion phase owns implementation and `preselect` vectors.

Pin shared reason/copy: `Suggested · this task`, `Suggested · reading task`,
`Suggested · daily review`, `Suggested · from <stem>` (root-area dot where supported),
`reading tasks suggest READ`, `daily review suggests PLAN`, `<stem> suggests WRITE`, and
`no default here`. Consequences: `this task becomes FIX · <stem> suggests WRITE` and
`task theme cleared · <stem> suggests WRITE`; notices:
`Linked under FIX · task theme FIX` / `Linked under WRITE · task theme cleared`.
Accessible example: `Pomodoro WRITE, suggested, from sase_blog`. Explicit task fields
alone get PT6 pills; inherited values do not become ghost pills. Unknown and repair
styling follows the parent contract. The later capture JSON addition is
`task_theme: {action: "set" | "cleared", theme, previous, inherited}`, omitted on no
change. Later listing rows add `suggested`/`suggestion_label`; suggestions may gain
`fallback` and ref-today `create` without a schema bump.

Update the shared-vector index, the authored/derived field rows in
`docs/reference/vault-format.md`, R1's pointer in `docs/pomodoro-homes.md`,
`plan.reading_theme` in `docs/plan.md` and `docs/reference/configuration.md`, and
generated CLI reference/navigation through `just docs-gen`. Read `docs.md` and
`cli_rules.md` through `/sase_memory_read` before relevant edits. Change no memory
source or generated memory content to evade the outer decisions.

Run the full canonical `just check` (covers the formatting/lint/tests requested by the
parent's `just all`), `just docs-check`, and a scratch-vault human/JSON smoke test with
a temporary note default. Record exact outcomes; do not claim unrun checks. A failure
reproduced identically on the clean base is a `PROPOSED FOLLOW-UP:` note on the assigned
phase (and summarized for `bob-cli-dm.1`), with any already-known task ID, rather than a
reason to strand a completed phase. Do not create follow-up beads. Check/re-key any
`--epic-symbol` entries before closing the assigned child phase.

## Child epic land agent acceptance

Verify all three phases landed and inspect their evidence, especially all shared
vectors, the six preservation paths, list timing, and documentation generation. Re-run
integrated gates only if merges/new changes or unresolved evidence require it.
Resolve/re-key temporary symbols before closing this child epic, and then run
`sase bead epic-symbols bob-cli-dm.1` before closing that phase. If unresolved symbols
are needed only by later outer phases, re-key them to `bob-cli-dm` or the appropriate
still-open phase; never leave a closed-bead exemption.

Close the child epic through its normal land flow, then close **only the delegated outer
phase** with
`sase bead close bob-cli-dm.1 --note "<verified behavior, checks, timing evidence, exported interfaces, PT13 numbering correction, and any proven baseline failures>"`.
Leave `bob-cli-dm` and every ancestor plan open for their own land agent. This is the
explicit delegated closure of the original phase requested by the user; child phase
workers have no ancestor-close authorization.
