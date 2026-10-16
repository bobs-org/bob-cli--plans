---
tier: tale
title: Complete vault-note snapshot creation and provenance
goal:
  Wire the existing note projection and print renderer into bob ref create, preserve
  provenance through scan and sync, verify the CLI contracts, and close bob-cli-87.5.
size: medium
proposed_by: bbugyi200.athena.bob-cli-87.5
bead: bob-cli-87.5
create_time: 2026-10-16 14:37:32
status: wip
---

- **PARENT:**
  [202610/note_snapshots.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/note_snapshots.md)
- **BEAD:**
  [bob-cli-87.5](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-87/bob-cli-87.5.md)

# Complete note snapshot integration for bob-cli-87.5

## Scope and authority

Implement and close only phase `bob-cli-87.5`, “bob ref create integration, provenance,
help, and docs,” under epic `bob-cli-87`. Read the phase again with
`sase bead read bob-cli-87.5 -r "Need assigned integration scope and current notes"` and
read `plan:202610/note_snapshots.md` with `sase artifact read` before coding. This tale
specifies the integration work within that approved design.

The initial phase read contained no earlier worker notes or recorded unfinished
implementation. The working tree was clean. The view, Bob projection, fonts, and
two-step print renderer are already present, so this is bounded integration for one
coding agent. Do not create a second projection engine or reproduce their content rules
in create.rs.

The epic's final decisions remain final: `default_text_layer=yes`,
`obsidian_command=yes`, and `decision_record=no`. This phase makes no memory changes.
Image conversion, scheduled-lane splitting, `-S` section filtering, earlier-snapshot
hints, plugin work, cross-host corpus checks, and the human Highlights round trip belong
to other phases. Preserve the existing default Markdown text-layer changes. Do not close
the parent epic or any ancestor plan.

All new fixtures must be synthetic. No live vault content enters this repository; tests
use temporary vaults. Projection and dry run never mutate source notes or run reconcile.
Use existing create/scan behavior for the eventual intake and reference-note writes.

## Outcome

`bob ref create harbor_cranes` resolves a vault note and, on a fixed test date of
2026-10-16, creates `xlib/notes/harbor_cranes_261016.pdf`. It uses the existing snapshot
projection and Bob print profile, defaults the parent to the nearest eligible
area/project, titles the PDF and marker `Harbor Cranes · 2026-10-16`, and records
`source_note: harbor_cranes` in the marker. Scan produces the normal ref note with
`source_note: "[[harbor_cranes]]"`; subsequent syncs are stable.

`-d` reports the selected identity, parent, target, projection summary, and warnings
without invoking external tools. `-C` restores the existing closed-task
appendix/full-log projection. `-M` selects the plain Markdown renderer for a vault note.
It changes the rendering profile; vault-note identity, parent defaults, and source
provenance still apply. Existing non-vault Markdown, PDF, URL, collection, audio, and
explicit-output behavior must remain intact.

## Existing interfaces and integration points

- `src/native/highlights_ref/create.rs` owns CLI construction, early parent validation,
  `CreateOptions`, `CreatePlan`, route dispatch, marker composition, filters, audio,
  target planning, and reporting. Today it requires `-P` before local classification and
  only dispatches through `target::resolve_local_target`. `create_markdown_route` and
  `render_temp_pdf` implement the legacy Markdown path.
- `note_snapshot/classify.rs` already exposes `SourceStore`, `classify_target`,
  `ClassifiedNote`, `SnapshotTarget`, canonical membership and exclusions, and cached
  bytes. `default_parent` is an intentional deferred stub; replace it and its stub test.
  Keep the source cache alive through classification and planning.
- `note_snapshot/transclusion.rs` exposes `project_snapshot` / `project_bob_snapshot`
  with `SnapshotProjectionOptions`: masthead, snapshot date, include_closed, and the
  exact consumed H1 anchor. The result includes parsed source bytes, the view,
  summaries, warnings, and a `TitleCandidate`.
- `note_snapshot/obsidian.rs` already recognizes the original first H1 outside
  fences/comments and exposes its plain display text and collision-safe anchor. Use that
  parser rather than create.rs's simple legacy H1 extractor.
- `serialize_projected_markdown` is the only view serializer.
  `note_snapshot/profile.rs::render_note_pdf` takes projected Markdown, scratch,
  `create::RenderFilters`, and `NoteMeta`; it returns PDF path, typed TeX warnings, and
  the return-links report. It already writes projected.md, fonts, glyph table,
  Lua/header files, note.tex, and note.log, and runs the bounded xelatex passes.
- `stamp.rs` already owns title-based identity/collision walking and exact output.
  `pdf_target::stamp_pdf_to_scratch` provides verified marker stamping.
  `stamp_listen_and_install_pdf` already handles stamp-before-listen, collision recheck,
  audio installation, cleanup, and post-listen recovery reporting.
- Marker normalization is in `sidecar.rs`, `marker.rs`, `model.rs`, and
  `frontmatter.rs`. Canonical parent normalization currently runs when parsing markers,
  extracting frontmatter, and reading stored projection bases. V2 sync removes only
  parent from its hash/base because residence owns parent.
- CLI tests live under `tests/cli/highlights/` and `tests/cli/help_options.rs`; existing
  fake tools and temporary-vault helpers can be extended. `just check` runs fmt, clippy,
  and all test binaries.

## Implementation

### 1. Add a small snapshot planning facade

Add `note_snapshot::plan_snapshot` and a `SnapshotPlan` in a focused module under
note_snapshot, exported through mod.rs. Keep projection planning separate from target
collision/install policy. The facade should carry the view, serialized Markdown or data
needed to serialize it, source identity/digest, selected titles, dated base stem, source
route, default-parent explanation, summary, and warnings. The outer CreatePlan can hold
an optional snapshot payload rather than spreading snapshot-only fields across every
existing route.

Use `env::current_datetime()` once per invocation for all snapshot dates, honoring
`BOB_NOW`. Select display title in order: explicit `-T`, frontmatter title, original
first H1 rendered to plain text, then humanized note stem. Identity title is
`<display> · <YYYY-MM-DD>` unless `-T` was supplied, in which case it is exactly the
override. Consume only the exact H1 selected for the title; preserve other H1s, and
preserve the H1 when a higher-precedence title won. Apply this consistently to ordinary
notes as well as Bob notes. The current consumed-title plumbing is Bob specific, so
extend only that necessary integration seam for ordinary notes.

Avoid a second independent parse/read for title selection. Factor the syntax projection
handoff if necessary so title selection precedes final assembly while using the same
cached bytes and existing TitleCandidate. Repeated access through SourceStore must not
reopen root or embed sources. The root digest, projection, title, metadata, and audio
discovery must all describe the same loaded content.

Reuse `ref_library::migrate_zorg::render::titleize_id`, moving the helper into a shared
module if needed. Add SASE acronym handling; Bob must remain title case. Test
`sase_blog` → `SASE Blog` and `bob_git` → `Bob Git` without regressing the existing
humanizer tests.

Implement the default parent as a cycle-safe, depth-bounded walk: test the note itself
for the same eligibility as `parent_notes::resolve_parent`, then follow its frontmatter
parent links, up to eight ancestor hops. A terminal project is a valid snapshot source
but is skipped as a parent. Ordinary/subfolder notes follow their parents. Missing
links, cycles, and exhaustion yield no default, resulting in the existing
`--parent is required` error and capture-targets hint. Explicit `-P` always uses the
existing resolver and its errors and wins over inference.

Keep the traversal pure over cached frontmatter and resolution results. Reuse or extract
the existing parent eligibility checks rather than introducing competing rules or
broadening the set of routable parents. Parent discovery must not reload root/embed
bytes outside the cache; a small loader-aware helper is acceptable. Return why the
default was chosen so dry run can distinguish the note itself, an ancestor, and an
explicit parent.

Construct SnapshotMasthead and SnapshotProjectionOptions using the selected titles,
date, `-C`, `glyphs.lua`, and the source's `bob-return-links` setting. Let the existing
Bob layer fill its masthead and content. Supply sensible NOTE masthead/snapshot/footer
metadata for ordinary notes. Normalize terminal-project classification to the
user-visible project kind where appropriate.

### 2. Route targets and expose the two options

Add `-C, --closed` and `-M, --plain` under a `Vault notes` help heading, in alphabetical
order. Use the approved descriptions:

- closed: “Also print closed tasks (a Closed tasks appendix) and full task logs”.
- plain: “Render a vault note as plain Markdown, without the snapshot view”.

Update every CreateOptions initializer: create.rs, collection.rs, and test helpers.
Reject `-C` plus `-M` through clap. Reject either option on non-vault targets with exit
2 and a one-line hint, before tool invocation or writes; also check the early
multi-target branch so collection processing cannot silently ignore these flags. Several
TARGETs remain the existing web-article collection path, and a local/note target among
them retains its current refusal. Do not add `-S` in this phase.

For a single target, retain URL dispatch and existing filesystem precedence. Classify
local/note inputs early enough to infer parent before the missing-parent guard. Reuse
classify_target's exact relative path, unique case-insensitive basename, wikilink,
ambiguity-list, near-stem, canonical-membership, and exclusion behavior. Do not
lossy-convert OsString paths before existing non-UTF-8 file validation. External
Markdown/PDF routes must not newly require a usable vault source cache just to classify
an existing file. Preserve the current explicit-P preflight behavior for non-note
routes.

Vault notes default to ref type `notes` through default_ref_type_for_kind; this is a
free-form shelf, not a new registry. Default stem is `<source-stem>_<YYMMDD>`; `-N`
wins. Use existing target helpers, exact `-o` validation, final-stem `-i` handling,
title identity, suffix walk, and refusal rules. A same-day repeat with the same title
refuses as already queued/captured; `-f` can replace only a queued copy under the
existing rules, and `-N` permits another deliberate snapshot. `-o` remains an exact
ad-hoc output and keeps its existing conflicts with -N/-t.

### 3. Carry source_note through marker, scan, and sync

Add an optional standard synced `source_note` field alongside source_url in the common
user-field ordering and marker-extra ordering. Compose it from the actual vault-relative
source route with no .md suffix, independent of parent and -N. Never attach it to
non-vault input. It is present for vault-note plain rendering as well as snapshots.

Mirror parent note-link normalization: the internal synced projection and generated
frontmatter contain a quoted wikilink; the rendered marker uses the bare route. Apply
canonicalization at all three entry points (marker, frontmatter, stored base JSON).
Absence is valid and leaves legacy notes alone; invalid present values should produce
field-specific diagnostics. Reuse a parameterized note-link helper if that keeps parent
behavior byte-compatible.

Unlike parent, source_note remains in v2 projections, hashes, and stored bases. It must
not become a pipeline-owned or command-managed field or require
highlights_marker_fields. Existing source_note-free refs must not churn.

Ensure `stamp_marker_for_outcome` preserves source_note when it adds
`return_links: true`; recomposing a marker after rendering must not drop provenance.
Prove that scan emits `source_note: "[[folder/note]]"`, that a second scan/sync has no
projection or file churn, and that PDF writeback retains the bare marker value. Include
a generated-ref fixture that proves the existing self-snapshot filter recognizes this
provenance.

### 4. Render and install through the existing lifecycle

For snapshot payloads, copy each resolved asset to its serializer-assigned path under
scratch/assets, serialize the view, and call render_note_pdf using the existing
code-break and return-links filters in the prescribed order. Plain copies are sufficient
here; conversion/downscaling belongs to the images phase. Report copy failures clearly
and install nothing on failure. Keep all generated files under ScratchDir ownership and
its KEEP_WORKDIR behavior.

Use the returned PDF and return-link report to compose and verify the final marker.
Follow the PDF routes' reload/page-count/marker-reparse validation before atomic
installation, preferably through stamp_pdf_to_scratch. Set Info title to the identity
title and preserve note profile metadata. Retain author/published, status, audio
discovery, no-audio, and force semantics.

For `-L` snapshots, `{target}` is scratch/projected.md, `{pdf}` is the rendered scratch
PDF, and `{title}` is the identity title. Render and validate the stamped PDF before
invoking listen; preserve its preflight, audio binding, collision recheck, failure
cleanup, and scratch recovery behavior. Reuse the existing stamp/listen/install helper
where possible. A failed render, stamp, or listen must leave the destination and source
vault notes unchanged. Plain-mode and non-note narration retain their existing input
behavior.

Dry run computes the plan without pandoc, xelatex, listen execution, or generated render
assets. KEEP_WORKDIR on a real snapshot retains projected.md, note.tex, note.log, fonts,
and assets through the existing ScratchDir mechanism.

### 5. Report the projection and document the command

Preserve the existing useful print_plan fields, using `would create note snapshot` and
`created note snapshot` for the new profile. The dry run adds source kind and short
SHA-256, parent default reason, surviving section names, per-status shown task counts,
closed/self-snapshot omission counts, condensed schedule/work-log counts, images/missing
images, transclusions, resolved/unresolved links, and glyph coverage. Compute glyph
coverage from visible projected content rather than raw hidden bookkeeping. Use the
typed summary and warnings rather than reparsing printed text. End dry run with
`writes: none`.

Success retains source/pdf/title/status/parent/pages/next lines and adds concise
`shown:` and `omitted:` lines. Emit each projection/glyph/render problem as a `warning:`
line, including unresolved links, missing images, guarded embeds, uncovered glyphs,
and >20pt overfull boxes. Preserve return-link reporting without duplicating its
diagnostics. Use deterministic ordering and normal no-ANSI output when piped. Do not
label `-M` output as a projected snapshot.

Update TARGET help, the Targets summary, parent/type/name descriptions, a `Vault notes:`
after-help block, and at least three note examples. Explain detection/exclusions, parent
fallback, dated names, title precedence, omissions, -C, -M, dry run, and KEEP_WORKDIR.
Keep options alphabetical within their help sections and hidden clip aliases
byte-identical to canonical create help.

Add a Vault notes section and update the targets table in docs/highlights-create.md.
Include defaults, short content policy, status glyph and chip legend, flag conflicts,
identity/same-day behavior, -o, projected listen input, text-layer guarantees, and
troubleshooting. Briefly explain the approved xelatex choice with the research reference
identified by the epic plan. Mention that the old lib/chat/sase_blog_v1 artifact stays
where it is. Describe only implemented flags; later phases own -S and cross-host
verification claims. Document source_note's marker/frontmatter representation and sync
membership in docs/highlights-ref-sync.md. Update docs/ref.md only where its existing
summary would otherwise be incorrect. Do not claim real-vault or Mac validation here.

## Verification and completion

Use focused unit and CLI coverage, extending existing helpers instead of duplicating
test infrastructure:

1. A synthetic vault with Tasks settings exercises route, .md, absolute path, subfolder
   path, and wikilink targets; exclusions and filesystem precedence; missing-name hints
   and all ambiguous candidates; explicit parent, self parent, terminal-project ancestor
   fallback, subfolder-note ancestry, cycles, missing ancestors, depth bound, and no
   eligible parent. Add a cache/read-count test spanning classification, title
   selection, and projection.
2. Freeze BOB_NOW and assert dated stem, title precedence/plain H1 text and exact H1
   consumption, humanizer results, notes shelf, -N/-T/-t overrides, -i final stem
   behavior, -o output and conflicts, queued/captured refusal, -f behavior, and the
   exact normalized dry-run text. Verify snapshot flags fail with exit 2 for external
   Markdown, PDFs, URLs, and collections, and -C conflicts with -M.
3. Fake pandoc/xelatex scripts must assert the two-step invocation and produce a valid
   synthetic PDF, with controlled return-link reports and log warnings. Check successful
   stamp round trip, source_note retention with paired return links, asset staging,
   closed/full-log projection, -M's legacy render branch, KEEP_WORKDIR files, and
   unchanged source bytes. Rendering failure or invalid PDF must install nothing.
4. Extend listen tests to assert `{target}` is projected.md and contains the visible
   projection, while `{pdf}` is the PDF. Verify audio/install cleanup on failure,
   recovery after post-listen collision, and no tool execution in dry run. Keep existing
   non-note create/listen tests passing.
5. Exercise create → scan → repeat sync with the synthetic marker/ref. Assert linked
   source_note frontmatter, bare PDF marker, stable hashes/base, no unnecessary
   unknown-field bookkeeping, and no churn on refs lacking the new field. Unit-test
   source_note canonicalization and validation on all entry points and the existing
   self-snapshot filtering integration.
6. Update help order assertions and byte-identical alias tests. Add one real
   pandoc/xelatex-gated synthetic project-note create test using the repository's
   existing skip-with-eprintln pattern. Verify a nonempty stamped PDF and its
   title/provenance; the print-profile tests already own typography internals.
7. Run focused tests while iterating, then `just check` before closing. Use
   `/sase_monitor` for a long-running check as required by the single-turn host; wait
   for the handoff command itself to exit. Fix failures caused by this work. For a
   failure that reproduces identically on the clean base, record the base revision,
   exact command/failure, and any existing tracking bead in a `PROPOSED FOLLOW-UP:` note
   on bob-cli-87.5, and continue to close the phase.

Record out-of-scope discoveries only through
`sase bead note bob-cli-87.5 'PROPOSED FOLLOW-UP: <summary — evidence/detail>'`. Do not
create task beads or manually change the assigned phase's status.

Immediately before closure run `sase bead epic-symbols bob-cli-87.5`. Planning found no
entries, but recheck the live result. Resolve every remaining entry or re-key its
justfile line to a still-open owning bead before closing; do not leave stale exceptions.
Then run
`sase bead close bob-cli-87.5 --note "<implemented integration and actual verification results>"`.
Do not close bob-cli-87 or any ancestor. Finish with the required `/sase_final`
declaration; no manual git commit unless separately authorized.
