---
tier: epic
title: Complete ref create force-mode parsing and rereads
goal: 'Complete bob-cli-c9.3 so an explicit reread creates a separate reading with
  durable lineage, a new reading task, and no changes to earlier readings, across
  every supported create route and queued intake.

  '
phases:
- id: force_parser
  title: Parse recapture modes and preserve existing force behavior
  size: medium
  depends_on: []
  description: 'force_parser: implement the force-mode parser, option plumbing, completion,
    and parser tests; retain filed-reference refusals until the reread writers land.'
- id: family_planning
  title: Resolve capture families and allocate complete reread bundles
  size: medium
  depends_on:
  - force_parser
  description: 'family_planning: add shared capture-family classification, whole-bundle
    __vN allocation, and refusal metadata without enabling reread writes.'
- id: reading_birth
  title: Derive lineage and preserve reading identity during intake
  size: medium
  depends_on:
  - family_planning
  description: 'reading_birth: derive earlier at birth, preserve reading task suffixes
    and aliases, and recheck reread occupancy under the intake locks for inline and
    queued filing.'
- id: single_sources
  title: Enable rereads for single files, URLs, and note snapshots
  size: medium
  depends_on:
  - reading_birth
  description: 'single_sources: wire the shared reread plan into every single-source
    route with family defaults, flag guards, audio behavior, dry runs, and zero changes
    to earlier readings.'
- id: collections_acceptance
  title: Finish collection rereads and the public contract
  size: medium
  depends_on:
  - single_sources
  description: 'collections_acceptance: enable exact URL-set collection rereads, complete
    help and documentation, and verify the phase-wide acceptance matrix.'
proposed_by: bbugyi200.athena.bob-cli-c9.3
proposed_with: codex/gpt-6-astra
parent_bead: bob-cli-c9.3
create_time: 2026-10-25 15:22:56
status: done
bead_id: bob-cli-c9.3.1
---

- **PROMPT:** [prompts/202610/ref_create_reread.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202610/ref_create_reread.md)
- **PARENT:** [202610/ref_create_force_recapture.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/ref_create_force_recapture.md)
- **BEAD:** [bob-cli-c9.3.1](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-c9/bob-cli-c9.3.1.md)

# Complete bob-cli-c9.3: force-mode parsing and rereads

This child epic implements the reread phase of
`plan:202610/ref_create_force_recapture.md`. Its land agent closes only the assigned
phase `bob-cli-c9.3` after checking the complete contract and clearing its epic symbols.
It must not close `bob-cli-c9` or any ancestor plan bead.

## Scope, inherited decisions, and sizing

The assigned bead is in progress, with no earlier implementation or remainder notes.
Prerequisites are already present: `16f86b7d` adds the library family model and
`e852198d` adds the shared duplicate renderer and duplicate-before-parent checks. At
planning time `sase bead epic-symbols bob-cli-c9.3` reports no entries.

This is an epic because parsing and option migration, family classification, allocation,
sync birth and task identity, and six capture routes each have distinct failure modes.
Each phase below is bounded direct implementation work (`medium`), with explicit
interfaces and tests. Dependencies are deliberately serial because these phases share
`create.rs`, `stamp.rs`, and the large create test suite. Do not replace this with
another single-agent implementation of the whole remainder.

The original epic's decisions are final:

- `legacy_force = no`: a PDF-less legacy match keeps warn-and-capture-fresh behavior
  under bare `-f`; it does not start a reading family. A legacy-only source does not
  provide an overwrite target.
- `keep_created = yes`: preserve this decision for the later overwrite phase. This child
  epic never implements overwrite or `recaptured:`.
- `memory_decision_record = no`: no decisions record.
- `memory_glossary_ref_task = yes`: the original epic's `record` phase owns that
  authorized correction. This child epic changes no memory, including the glossary.

Before implementing, read the assigned bead and the parent plan through their audited
commands. Read `cli_rules.md`, `docs.md`, and the applicable reference/intake decisions
with `/sase_memory_read`. No other repositories or the real vault are needed.

## Shared contract and implementation seams

Keep the family read model already in `src/native/ref_library/lineage.rs` as authority:
`RefRow.family` identifies a component's root, and the reading number is one plus the
count of `earlier:` entries under `ref/`, including dangling reading entries. Never
infer an existing family's reading number just from a filename. A natural `protocol__v2`
without lineage is reading 1.

Introduce one small recapture planning module under `src/native/highlights_ref/` (for
example `recapture.rs`), rather than handling duplicate errors by parsing their rendered
strings. It should carry typed match results for no filed match, queued only, a selected
live family, ambiguous unrelated families, and partial URL overlap. The selected-family
result carries root/newest/selected members, normalized source keys, task residence,
reference type, existing audio, and display metadata. A reread plan adds the final
`TargetPlan` and the metadata needed for a `reread:` report. Exact Rust names are
implementation choices; keep source classification and rendering separate from mutation,
and let overwrite consume this seam later.

Existing seams to reuse:

- `create.rs::run_create`, `check_duplicate_before_parent`, the Markdown/PDF prechecks,
  and `resolve_parent_for_target`. The current precheck only runs when `-P` is absent
  and sometimes executes `-L` immediately; replace that duplication with a reusable
  preflight result so recapture defaults can precede parent resolution.
- `sources.rs` collects every URL of both scalar and collection references. Preserve
  syntactic URL/arXiv normalization and its automatic-producer collision strings.
- `stamp.rs::DefaultTargetOccupancy` already sees sibling type directories and
  case-insensitive reference ids, but must also cover audio and `.assets/`.
- `duplicate.rs` owns `AlreadyCaptured`, `CapturedCopy`, `Remedy`, and the shared
  multiline printer. Its copies currently have no reading number; enrich them from the
  index instead of implementing a second family parser.
- `collection.rs::check_collection_overlap` currently considers only one path an exact
  match. A valid multi-reading family must count as one identity for exact-set matching.
- `sync.rs::plan_pdf_sync_v2`, `PipelineMetadata`, `reading_plan.rs`, and
  `reading_execute.rs` own birth, preview, task insertion, and repeat-sync behavior.
- `intake.rs::IntakeSession::ingest` runs after pre-sync with both locks held. Both
  inline intake and `drain_queue` pass through it, before `perform_moves`.

The low-level boolean overwrite permissions for exact output, queued PDFs, and audio can
remain booleans; convert an explicit mode to `force.is_some()` at those boundaries. The
user-facing create/clip/collection options carry `Option<ForceMode>`. Automatic
producers keep no force intent, including when they call lower-level functions directly.

No reread writer is enabled until `reading_birth` is complete. Earlier child phases must
compile and preserve successful existing creates, queued replacement, exact-output
replacement, and listen attachment. Their temporary filed-reference refusal/help must be
truthful; never silently interpret an unavailable mode as permission to overwrite.

## Phase force_parser

Implement the parser and mechanical option migration; do not enable a filed reread.

1. Define `ForceMode::{Reread, Overwrite}` and one decoder shared by create, its hidden
   clip alias, and collections. Replace the force clap argument with:

   ```rust
   Arg::new("force").long("force").short('f').value_name("MODE")
       .num_args(0..=1).require_equals(true)
       .default_missing_value("reread")
       .value_parser(["reread", "overwrite"]).ignore_case(true)
   ```

   Absent force stays `None`; do not add a default value. Decode and remove adopted mode
   tokens before choosing single-target versus collection dispatch.

2. Adopt a case-insensitive `reread` or `overwrite` TARGET only when directly adjacent
   to a bare force flag according to clap indices. An explicit equals-value never
   adopts. Preserve all other TARGET bytes as `OsString` and their original order.
3. Guard a bare TARGET equal to, or within edit distance 2 of, either mode when it has
   no `/`, `.`, or `[[`, is not an existing path, and is not a URL. Exit 2 with
   `` `overwite` looks like a --force mode; use -f overwrite (or --force=overwrite) ``.
   Existing paths, `./overwrite`, and `[[overwrite]]` retain their target meanings. If
   adoption empties TARGET, produce the normal missing-TARGET usage error.
4. Replace force booleans in `CreateOptions` and `ClipOptions` (collections use
   `CreateOptions`), and every create/clip/collection `get_flag("force")`. Leave
   unrelated commands' boolean force flags alone. Preserve old permissions through
   `is_some()` at low-level calls; producer retries never acquire a reread mode.
5. Overwrite must refuse filed references with exit 2; initially use a truthful
   unavailable message. For a fresh capture with no applicable existing replacement,
   return exit 1: `nothing to overwrite: <source> is not a reference yet`, with the hint
   to drop `overwrite`. A queued-only exact match still accepts either mode's old
   replacement permission. Keep the old exact-output and companion-audio replacement
   permissions under either explicit mode; do not mistake an existing exact-output
   replacement for a fresh capture. Preserve `-L` without force attachment. Do not fall
   through into fresh capture for an unhandled overwrite route.
6. Add static shell-completion coverage for `--force=<MODE>` using the existing clap
   choice providers and `!prefix` protocol. Avoid bespoke shell parsing when the
   existing provider suffices. Keep public help alphabetical, document supported
   spelling and escapes, and be honest that filed reread wiring is still pending.
   Refresh generated CLI docs for any help change.

Lock the complete argv matrix with parser-level tests and representative actual CLI
tests (same results for hidden `bob ref clip`):

| Arguments after create                                          | Result                                    |
| --------------------------------------------------------------- | ----------------------------------------- |
| `URL`                                                           | no force                                  |
| `URL -f`, `-f URL`, `--force URL`                               | reread                                    |
| `-f overwrite URL`, `URL -f overwrite`, `URL --force overwrite` | adopted overwrite                         |
| `--force=overwrite URL`, `-f=overwrite URL`                     | explicit overwrite                        |
| `-f Overwrite URL`                                              | adopted overwrite, case-insensitive       |
| `-fd URL`, `-df URL`, `-f -d URL`                               | reread and dry run                        |
| `URL -f overwrite -d`                                           | overwrite and dry run                     |
| `-f overwite URL`                                               | exit 2, stray-mode guard                  |
| `-fd overwrite URL`                                             | exit 2, not adjacent                      |
| `--force=reread overwrite URL`                                  | exit 2, no adoption                       |
| `-f overwrite`                                                  | missing TARGET                            |
| `-f overwrite -f URL`, `--force=reread --force=overwrite URL`   | clap usage error                          |
| `URL1 URL2 -f overwrite`                                        | overwrite, exactly two collection targets |
| `-foverwrite URL`                                               | clap UnknownArgument                      |

Also test explicit case-insensitive values, invalid equals-modes, exact guard words,
near-miss words, and the documented escapes. CLI tests must prove usage failures run no
fetch/adapter/listen command and write nothing. Reuse the fake tools in
`tests/cli/highlights/create.rs` and the completion protocol tests.

## Phase family_planning

Implement the read-only family plan and allocator, keeping filed writes disabled.

1. Resolve live filed hits through the library index and the original route identity.
   URL queries compare complete normalized URL-key sets for collections, including
   one-URL queries that hit a collection. Title routes retain their existing stem probe:
   a free explicit `-N` is an ordinary new capture. Legacy-only hits retain their
   warning behavior and are not roots for the selected decision.
2. Group same-family hits and select newest by the existing computed reading number.
   Unrelated duplicate families remain an explicit ambiguity: list all paths and ask for
   `-N <stem>`; a valid member selector disambiguates without joining families. Preserve
   the chosen family root even when the requested target is a later reading.
3. Extend `DefaultTargetOccupancy`, not a separate filesystem walk. Reserve names across
   every type under `ref/`, `lib/`, and `xlib/`: notes, PDFs, sidecars, `.textbundle`,
   known audio extensions, `.assets/`, and case-insensitive reference ids. Provide a
   recheck form which can ignore the current queued unit's own files.
4. Allocate `<root>__vN` from max family reading + 1 (at least 2), skipping occupied
   candidates through 999. Never use generic `_N` collision fallback for rereads. A
   family's later stem does not become the next root. Keep ordinary target naming
   unchanged except for the required more complete occupancy protection.
5. On a captured URL, `-N` must name a member of the selected family; otherwise exit 2
   with `-N other is not a reading of foo; readings are named foo__vN`. `-o` on a
   captured reread exits 2: `a reread is named <root>__vN; drop -o`.
6. Supply defaults from the newest reading's actual located task residence and type,
   respecting explicit `-P`/`-t`. Reuse existing parent validation/fallback behavior; do
   not trust a stale authored parent over the located task. Default status remains
   ready, with an explicit `-s` honored. Do not copy companion audio automatically.
7. Enrich the common duplicate report with family reading numbers and newest-first
   order. Make the computed next name available to remedies and overwrite's interim
   refusal. Do not advertise a successful reread remedy until its route is enabled. Keep
   queued-only remedies and `already captured` / `already queued` substrings.

Tests cover a family of one, two and three readings, a dangling middle member, unrelated
same-identity families and `-N`, sidecar/audio/assets/id-only collisions across types,
cap exhaustion, unrelated `foo_2`, and natural `protocol__v2`. Include the parent's
`__v2`/`__v4` occupancy case yielding `__v5` with the fixture's family reading metadata
made explicit. A real reading `x__v2` rereads as `x__v3`, while an unrelated natural
version-looking root retains its full root name. Rendering tests cover actual computed
stems, all detail/remedy rows, and plain piped output.

## Phase reading_birth

Make manually queued reread-shaped PDFs safe to file, before create can generate them.

1. Derive birth metadata once in the shared single-PDF sync plan when the note is new.
   Only probe possible lineage for `<base>__vN` with N >= 2. Require a live `<base>` or
   lower `<base>__vM` sharing identity: equal normalized URL dedupe keys (complete sets
   for collections), else equal `source_note`, else equal normalized titles. Reuse the
   selected family's root and entries; never link unrelated occupants. Ordinary births
   should not trigger a new whole-library lookup.
2. Write block-style `earlier:` with every live family member, newest first, labelled
   `vM · <captured date>`, using the existing lineage renderer. Carry it in
   `PipelineMetadata` or equivalent birth-only plan state, outside the PDF projection
   and hash. Preserve it verbatim thereafter; no repeat-sync re-derivation and no
   backlinks written into old notes. Dry-run planning and execution must agree.
3. Carry confirmed reading metadata into task creation and previews. For rereads,
   reserve the `-vN` component in `^ref-<root-slug>-vN` before truncating to 44 chars.
   On task-id collision the ordinary `-2` suffix follows the version component;
   truncation must preserve both. Audit `ref_tasks/line.rs`, `insert.rs`, sync preview
   reservation, `reading_execute.rs`, and `ref_library/migrate_tasks/plan.rs`. Do not
   change already-existing task IDs or turn a natural suffix into family lineage. Use
   lineage-derived reading numbers for the library model; do not replace the parent's
   number definition with filename parsing when occupied names are skipped.
4. Render `Title (vN)` for confirmed later readings. Apply the same alias rule wherever
   a later sync or reopen renders that reading's task; keep existing task identity. Test
   long titles as well as long stems so the suffix is not sanitized away.
5. Recheck complete reread occupancy inside `IntakeSession::ingest`, after lock-taking
   and pre-sync, before any move/task/note mutation. The chosen policy is the parent's
   permitted conservative fallback: if another host took the name, refuse filing with a
   clear collision/nothing-filed report and preserve the queued source. Do not silently
   re-use the occupant or leave a marker id different from its stem. Ignore only the
   queued unit's own files during this recheck. Keep the queued item available for
   recovery, and retain ordinary intake rollback behavior.

Use temp-vault fixtures to exercise both inline intake and `scan --intake-only`/full
scan drain. Verify birth `earlier:`, exact note/embed/task addresses, a second sync's
byte-identical task line and lineage, long-stem collisions, and no bytes changed in
earlier notes/PDFs/sidecars/audio/tasks. Test a fabricated collision appearing between
planning and locked filing, including an id-only or other-type collision. Test a
version-looking PDF with no matching predecessor stays an ordinary birth. Existing sync,
task insertion, migration, and family doctor tests must still pass.

## Phase single_sources

Enable the single-source reread behavior using the completed plan and birth path.

1. Resolve duplicates and family defaults before requiring `-P`, doing network work,
   invoking adapters/listen/pandoc, or installing files. Use the shared preflight result
   throughout a route instead of running independent checks that disagree. Guard again
   when redirected/fetched metadata discovers another recorded identity.
2. Wire Markdown, original local PDF, PDF URL, arXiv, web article/`-H`, and same-day
   vault-note snapshots to the allocated target. Stamp `id` equal to its final stem on
   every reread, including routes where `-i` is otherwise optional. Preserve source
   metadata for queued birth derivation. Add an optional plan/context input to
   `clip.rs::capture_article` instead of defeating dedupe or using AlwaysDifferent.
3. With no force, preserve the common refusal before fetch or missing-parent errors.
   Bare force with no captured match is an ordinary create. Legacy-only matches remain
   warn-and-create-fresh. New-day note snapshots remain ordinary dated creates; same-day
   duplicates become `<dated-root>__vN`.
4. When TARGET is the actual library PDF, refuse a reread even with force: copying
   highlights into the next reading is forbidden. Hint to use its recorded `source_url`
   or the original file. `-L` without force still attaches to that PDF.
5. `-L` without force keeps the attach path. With reread it narrates the new reading; do
   not accidentally return early through old attach logic. With no `-L`/`-a`, earlier
   audio is not inherited. Print the parent's hint when an earlier reading is narrated,
   identifying its reading number and audio path. Existing explicit audio replacement
   permissions still apply.
6. Emit the `-f` refusal remedy with the computed next stem, and a `reread:` output line
   at the existing `renamed:` position, for both real runs and dry runs:

   ```text
   reread: attention_is_all_you_need → attention_is_all_you_need__v2 (v1 read · 41 highlights)
   ```

   Use shared formatting/Styler. Dry run ends in `writes: none` and needs no fetch or
   renderer. Filed overwrite now uses the complete interim exit-2 message:
   `-f overwrite is not available yet; -f reads it again as <stem>__vN`. Fresh overwrite
   still refuses with exit 1; queued-only and existing exact-output replacement retain
   their old force permission.

7. Update the single-source Recapture documentation and help to describe the behavior
   that now exists. Keep the temporary collection limitation explicit until the next
   phase and do not claim overwrite already works. Use `feat(ref)!` for the contract
   change when finalization asks for its commit description; do not edit versions.

Flip the parent's named tests for same-title library occupancy, folded-URL dedupe, and
scanned-library collisions: no force refuses; force rereads from an original source.
Keep the actual-library-PDF-as-TARGET refusal test. Cover every route, hidden clip,
omission and overrides of `-P`/`-t`/`-s`, URL `-N` rejection/member selection,
title-route free `-N`, `-o`, `-k` then scan, inline filing, no-ANSI output, and
fake-tool logs proving preflight ordering. Compare snapshots of old bundles and existing
task lines. Add producer retry tests so `ingest_url`, jobs, and Keep never create
`__vN`. No real network or real vault writes are part of acceptance.

## Phase collections_acceptance

Finish collection behavior and the externally documented contract.

1. Teach `collection.rs` to use the shared preflight and family plan. An
   order-insensitive exact normalized URL-set match rereads the selected family, even
   when several readings match that set. Preserve argument order for the newly captured
   content. Default parent/type, naming/id, queued intake, output, and audio semantics
   match single-source rereads. Preserve all-or-nothing collection capture.
2. Any partial overlap, including a single URL in a larger collection, refuses under
   every force mode before adapters/fetch. Name the overlapping members and use
   `hint: -f recaptures only the exact URL set of <stem>`. Remove the stale suggestion
   to delete old references. Do not mistake unrelated duplicate families for one exact
   family; member selection uses the shared `-N` rule.
3. Test exact sets in reversed order, three-generation families, partial subsets and
   supersets, one URL versus collection, different unrelated families with the same set,
   queued-only replacement under both modes, failed-part atomicity, no-parent defaults,
   `-f -L` versus attach, and parse-adopted `URL1 URL2 -f overwrite`.
4. Finish `docs/highlights-create.md` Dedupe/Output/Audio and a `## Recapture` section;
   update `docs/ref.md` Identity for `__vN`; update `docs/highlights-ref-sync.md` only
   where needed to explain birth and task behavior. Final clap help covers both mode
   spellings, bare-force reread, queued replacement, `-N` as the later overwrite
   selector, escapes, unsupported `-foverwrite`, and cases that need no force. Clearly
   state overwrite is parsed but unavailable for filed references in this increment.
   Remove temporary reread limitations. Regenerate, do not hand-edit,
   `docs/reference/cli/ref.md` and other generated help artifacts.
5. Run the complete acceptance checks below and resolve integration defects. Leave a
   concise evidence note on this phase for the land agent, including known base
   failures, exact commands, and temporary symbols removed or still owned elsewhere.

## Validation and land-agent completion

Each worker runs focused behavior tests for its changed paths, formatting, and lint
checks. The final phase runs `just check`, `just docs-gen`, and `just docs-check` once
the integration changes are complete. Use `/sase_monitor` for long commands and wait for
the monitor handoff itself to exit. Use existing `bob_command`, `write_highlights_pdf`,
`configure_test_git_identity`, fake clip/curl/pandoc/listen helpers, and temp-vault sync
fixtures. Do not run the application against `~/bob`.

The completed result must satisfy:

- The entire parser matrix and hidden-alias behavior; completion offers both modes.
- All supported source routes create a new complete bundle/task on explicit reread, with
  `id == stem`, lineage, family defaults, stable aliases, and safe naming.
- Queued reread followed by scan has the same lineage and task semantics as inline
  intake. Locked races file nothing over an occupant and report recovery context.
- Old notes, PDFs, highlights sidecars/assets/audio, and reading task lines retain their
  bytes. The new family causes no `duplicate_identity` or new doctor warnings.
- Second sync is stable; rename still rewrites family links using existing machinery.
- Exact-set collections reread and partial overlaps refuse before any adapter work.
- Filed overwrite remains refused, fresh overwrite fails, old queue/output/audio force
  permissions remain, and automatic producers never mint a new reading.
- Help and generated docs reflect the actual interim overwrite limitation and all reread
  behavior. No changes to Cargo.toml's version, CHANGELOG.md, or memory.

If a check fails, establish whether the identical failure occurs on the clean base. Use
an allowed isolated base checkout via `/sase_repo` if another checkout is needed. For a
confirmed base failure, append `PROPOSED FOLLOW-UP:` to the worker's phase bead, citing
an existing tracking bead if known, record evidence, and close the completed phase. Do
not create task beads or leave finished phases open over unrelated failures. Temporary
unused symbols must be removed/consumed or keyed to a still-open owning bead; never
loosen the project's lint gate to hide them.

Every child worker runs `sase bead epic-symbols <its-phase>` before closing only its
assigned child phase. The child epic's land agent verifies the whole acceptance list,
records the implementation and check evidence on `bob-cli-c9.3`, then runs:

```sh
sase bead epic-symbols bob-cli-c9.3
sase bead close bob-cli-c9.3 --note "<implemented behavior and verification evidence>"
```

Resolve every remaining symbol or re-key its justfile entry to an actually open later
owner before closing. Finish this child epic through its normal land lifecycle. The
explicit delegated completion is for `bob-cli-c9.3` only: never close `bob-cli-c9` or
another ancestor plan. The original overwrite, annotated-overwrite, and record phases
remain the original epic's responsibility.
