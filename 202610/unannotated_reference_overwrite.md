---
tier: epic
title: In-place overwrite of unannotated references
goal:
  Complete bob-cli-c9.4 with task-preserving overwrite on every capture route, inert
  prose archives, durable crash recovery, and verified code and documentation gates.
phases:
  - id: shared
    title: Share reference mutation safety primitives with rename
    size: medium
    description:
      "shared: extract the narrow lock, precondition, scoped-git, and rollback helpers
      needed by overwrite, retaining rename behavior and its tests."
    depends_on: []
  - id: prepare
    title: Plan overwrite selection and inert prose archives
    size: medium
    description:
      "prepare: implement strict read-only overwrite preflight, task and bundle
      fingerprints, annotation and open-task refusals, fresh skeletons, and frozen-note
      rendering with lineage preservation."
    depends_on:
      - shared
  - id: recovery
    title: Recover interrupted overwrites from host-local intents
    size: medium
    description:
      "recovery: implement durable overwrite intents and scoped HEAD recovery before
      create or scan can sync; preserve dry-run and JSON contracts and test interrupted
      and committed states."
    depends_on:
      - shared
  - id: transaction
    title: Apply and verify a staged overwrite in one scoped commit
    size: medium
    description:
      "transaction: consume the prepared overwrite and private staged bundle under both
      locks, recheck preimages, publish and sync the replacement, preserve task
      identity, verify, commit, and roll back failures."
    depends_on:
      - prepare
      - recovery
  - id: local
    title: Enable local-file and vault-snapshot overwrite
    size: medium
    description:
      "local: connect local PDF, Markdown, and vault-note rendering to the overwrite
      transaction, including explicit status handling and the shared preserved or
      replaced audio policy, with offline CLI tests."
    depends_on:
      - transaction
  - id: web
    title: Enable single-URL overwrite across web capture routes
    size: medium
    description:
      "web: connect article, saved-HTML, PDF-URL, and arXiv capture to the same staged
      overwrite transaction and preserve redirect, duplicate-before-fetch, and listen
      semantics."
    depends_on:
      - local
  - id: collections
    title: Enable exact-set collection overwrite and complete remedies
    size: medium
    description:
      "collections: integrate exact-URL-set collection overwrite, collection narration,
      shared refusal and success rendering, and finish user-facing help and contract
      documentation for unannotated overwrite."
    depends_on:
      - web
  - id: acceptance
    title: Verify overwrite preservation and crash recovery end to end
    size: medium
    description:
      "acceptance: close cross-route acceptance gaps, prove archive inertness and
      recovery through CLI fixtures, remove temporary symbols, and run the full code and
      documentation gates for the child epic land agent."
    depends_on:
      - collections
proposed_by: bbugyi200.athena.bob-cli-c9.4
proposed_with: codex/gpt-6-astra
parent_bead: bob-cli-c9.4
create_time: 2026-10-25 22:34:43
status: wip
---

- **PROMPT:**
  [prompts/202610/unannotated_reference_overwrite.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202610/unannotated_reference_overwrite.md)
- **PARENT:**
  [202610/ref_create_force_recapture.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/ref_create_force_recapture.md)

# In-place overwrite of unannotated references

This child epic implements **bob-cli-c9.4**, the `overwrite` phase of
`plan:202610/ref_create_force_recapture.md`. The result is a working
`bob ref create TARGET -f overwrite` that replaces a filed reference's material while
retaining its note identity and reading task. It freezes authored prose into inert
`ref_history/` notes, never queues an overwrite, and recovers a crash without losing the
earlier capture.

The child epic's **land agent**, after all child phases and their verification are
complete, owns `sase bead epic-symbols bob-cli-c9.4` and
`sase bead close bob-cli-c9.4 --note "<implementation and verification evidence>"`. No
child phase worker may close bob-cli-c9.4, this child epic, bob-cli-c9, or any other
ancestor. The land agent must not close the outer epic bob-cli-c9. The remaining outer
`annotated` phase bob-cli-c9.5 depends on this work and owns annotated overwrite.

## Authority, sizing, and observed starting point

The audited phase read reports `in_progress`, with no prior-work notes and no
`--epic-symbol` entries. The inspected checkout at `516d7b98` already contains the
family model, reread parser and routes, lineage births, and refusal renderer.
`force.rs::overwrite_filed_unavailable*` and `recapture.rs::finish_selection` still
refuse filed overwrite. There is no overwrite transaction to extend. Recheck the
implementation and bead notes before starting a phase; preserve later landed work.

This is an epic because read-only planning, durable crash recovery, task-preserving
mutation, and several independently implemented render/install pipelines are separate
coding work units. Every phase below is bounded direct implementation (`medium`), not
another planning handoff. `prepare` and `recovery` have disjoint new-module ownership
and may run together; all route phases are sequential because they share `create.rs`,
option plumbing, and test support. The dependency graph is:

`shared → {prepare, recovery} → transaction → local → web → collections → acceptance`

The parent plan is the authoritative detailed design. Its cited research artifact
`research:202610/ref_create_recapture_modes/ref_create_recapture_modes.md` did not
resolve during this planning turn; this plan relies on the accepted parent contract, not
inferred research findings. No additional product decision is requested.

The outer epic's final decisions apply unchanged:

- `legacy_force = no`: PDF-less legacy-only matches retain warn-and-fresh-capture
  behavior; they do not become reading-family roots. Preserve already shipped routing
  for queued PDFs, fresh sources, exact external outputs, and legacy-only sources.
- `keep_created = yes`: preserve the raw existing `created` value; write note-local
  `recaptured: YYYY-MM-DD`; refresh `captured` for the new material.
- `memory_decision_record = no`: do not create or edit decision records.
- `memory_glossary_ref_task = yes`: only `glossary:reference-task` is authorized for
  memory changes, and the outer `record` phase owns that correction. This child epic
  changes no memory. Modern task handling must follow code: the located task is normally
  in the parent note, referenced by a managed embed, not necessarily the stale
  glossary's in-note `^ref` tracker.

## Non-negotiable behavior

### Selection and refusal

Reuse `recapture.rs` and `ref_library::lineage`; do not infer reading numbers from
filenames or duplicate URL-family resolution. Overwrite chooses the newest reading
unless `-N` selects a member. It does not allocate a new `__vN`, and must work even when
reread allocation reaches `__v999`. Ambiguous unrelated families still require an
unambiguous selector and must never be silently merged. A nonmember `-N` on a URL match
exits 2. Exact-set collections match regardless of argument order; a partial overlap,
including one URL belonging to a collection, refuses under every force mode before
fetching.

For a filed overwrite, reject `-o`, `-k`, a differing explicit `-t`, or a differing
explicit `-P` with exit 2 and useful hints; the `-P` hint points to `bob task move`.
Compare resolved parent identity, not raw spelling. Default parent/type come from the
**selected** member. A matching explicit parent/type is valid. A truly fresh source
still gives `nothing to overwrite` with exit 1. A free name on a title route must not
become an overwrite selector for a different reference. Existing queued PDF and
exact-output replacement exceptions retain their force behavior.

Require a modern PDF-backed reference outside intake, a uniquely located task,
synchronized note/PDF/task, a Git worktree, available intake, and clean touched paths.
Keep the established `legacy_reference`, `intake_reference`, `not_in_sync`, and
`dirty_paths` codes, and add `open_note_tasks` for open checkboxes outside the managed
reading embed/tracker. List offending tasks. The not-in-sync hint is
`run bob ref scan -w on the Mac first`. Missing/ambiguous task identity is a refusal,
never permission to manufacture a replacement task. Disabled ingest, unavailable vault,
or unavailable locks refuse; overwrite never falls back to `xlib/`.

Until bob-cli-c9.5, refuse highlights, image selections, existing `.assets/`, or inbound
highlight-fragment/follow-up links with:
`overwriting an annotated reference is not available yet; -f starts a new reading`.
Inspect actual PDF/sidecar/note evidence, not just a possibly stale count. Ordinary PDF
navigation or audio-link annotations alone are not user highlights. A library PDF used
as TARGET must retain the existing refusal directing the user to its recorded source or
original file; do not render the old annotated library bytes as new source material.

### Preserved identity and rebuilt material

Keep the note path/stem/id and the located reading task's path, block id, checkbox,
children, dependencies, relations, log history, and freshness. Keep note/heading links
pointed at the live note. Title changes update only the existing reading-task link
alias, retaining a family's `(vN)` suffix and all other task content.

Clap supplies a default status, so capture whether `status` came from
`clap::parser::ValueSource::CommandLine`. No explicit `-s` means preserve the located
task status, even for a closed or archived task; explicit `-s` changes that same task's
status and stamps through a targeted edit. Do not accidentally select sync's
reopen-by-insertion branch. Preserve the same task address and exactly one reading task;
do not globally change scan's policy against editing archived tasks.

Render a new PDF with the same id and effective status, rebuild the live body from the
actual template plus the same managed task embed, reset stale pipeline/annotation
metadata, and remove the old `.md` or `.textbundle` sidecar, including tracked
descendants. Preserve note-local keys, `created`, and all prior `earlier:` entries.
Write `recaptured` once as a note-local date; it must stay out of marker projection,
hashes, and opt-in marker fields just as `created` and `earlier` do.

Preserve companion audio unless `-L` or explicit `-a` requests replacement. Implicit
Markdown audio discovery must not replace the existing episode during overwrite.
`--no-audio` does not delete an existing episode. Stage requested replacement audio
before vault mutation and include every changed/deleted companion in the same
transaction. A render/listen failure leaves the vault intact; retain useful scratch
audio diagnostics from existing create behavior. `-L` without force still attaches;
`-f overwrite -L` performs overwrite and narrates the replacement.

### Frozen notes

Detect authored prose by comparing the existing body and the generated skeleton for the
**old** projection after removing their managed regions. Use the existing v1/v2 template
and task-embed helpers; a normal generated title, audio embed, tracker, Highlights
heading, or managed wrapper alone is not prose. Conservative differences count as
authored material; do not erase formatting or closed user tasks.

When prose exists, allocate `ref_history/<type>/<stem>__<YYYY-MM-DD>.md`, then `_2`,
`_3`, etc. on collisions. Keep the old body verbatim except for replacing its managed
reading-task embed with an information banner and neutralizing a recognized legacy
tracker (`#task`, `#ref`, and its `^ref` address cannot remain a task source). The
banner records replacement date, original captured date, status, highlight count, and a
link to the current reference. Open authored tasks must have refused before this point.

The frozen frontmatter allowlist is `title`, `source_url`, `created`,
`parent: "[[ref/<type>/<stem>]]"`, `replaced`, `status_when_replaced`, and
`highlights_count`. Do not copy live-reference identity/projection/task metadata.
Preserve YAML value types, including collection `source_url` lists. Prepend a
block-style `earlier:` link labelled `replaced <date> · 0 highlights` to the live note,
carrying all older entries forward in order. Two same-day overwrites must leave two
independently addressable archives. If there is no prose, create no archive, preserve
existing lineage, and report
`earlier: nothing to archive (0 highlights, no notes); git keeps the old capture`.

### Transaction and crash recovery

The operation has a strict stage boundary: preflight and render in private scratch
first; lock/recheck/apply/verify/commit second. No renderer may install into the live
bundle or intake queue before the transaction owns publication.

1. Read and validate the target and compute a manifest of touched paths, absent
   destinations, preimage hashes, selected task address/subtree, and expected identity.
   Stage PDF and optional audio outside the vault. `-d` stops before fetch, adapters,
   pandoc, or listen and ends with `writes: none`.
2. Acquire `bob_sync.lock → scan.lock`. Recover any earlier intent before any pre-sync
   can commit half-applied files. Pre-sync, then repeat selection, eligibility,
   clean-path, annotation, task, and absence/hash checks. Abort on changed preimages
   rather than silently retargeting. The rollback base is the clean locked HEAD after
   pre-sync, not an earlier pre-render HEAD.
3. Persist and fsync an intent in `bob_cli_state_dir()/ref/`, never in the vault.
   Identify the canonical vault and transaction, base HEAD, stem/type, touched and newly
   created paths, and enough expected postimage/commit evidence to distinguish this
   overwrite from a previous same-stem overwrite. Use atomic updates. Expand directory
   operations to the actual file manifest; untracked contents must not be accidentally
   deleted with a sidecar directory.
4. Write the optional frozen note, remove stale sidecar data, and install the staged PDF
   without silently clobbering a changed destination. Use the existing atomic
   write/no-replace helpers. Write the fresh skeleton carrying the task embed and
   note-local state, then run single-PDF sync to render managed regions and refresh
   metadata. Permit no task birth or unexpected writes outside the manifest.
5. Verify a second dry-run single-PDF sync is a no-op; the task address and effective
   status are correct, there is exactly one reading task, untouched task content is
   byte-identical, no new `duplicate_identity` exists, and every newly emitted
   lineage/embed link resolves. All failures restore only the transaction's paths from
   the recorded base, remove its creations, unstage its paths, and exit 1 with
   `verify_failed`. Leave unrelated working-tree/index content untouched.
6. Commit once through `ob::commit_paths` with subject
   `bob ref create: overwrite <type>/<stem>`. Handle an error reported after Git
   actually committed without rolling back committed data. Clear the intent only after
   the committed state is proved; post-sync failure reports
   `overwritten; push pending (run bob vault-sync)`, never advice to rerun overwrite.

At the start of real `ref create` and all writing `ref scan` variants (including
intake-only and JSON), detect pending host-local intents under the same lock order. If
the recorded overwrite commit is present, retain the new state and clear the intent;
otherwise restore the touched paths from the recorded clean HEAD and remove only
recorded creations. A generic matching commit subject alone is insufficient. Print
`recovered: rolled back an interrupted overwrite of <stem>` after successful rollback.
Recovery must be idempotent and preserve its marker if repair fails. Validate stored
paths against the intended vault; malformed intents fail without performing broad
restore/clean operations.

Dry runs remain genuinely read-only: with a pending intent they report that recovery is
required by a non-dry-run command and stop without restoring files or clearing the
marker. JSON scans keep machine-readable stdout; recovery notices belong on stderr or in
an existing appropriate report field. Avoid recursive/reentrant locking when recovery
runs within an already-held mutation session.

## Implementation phases

### `shared`: narrow reusable safety code

Inspect `highlights_ref/rename/plan.rs` first. Extract, rather than copy, the needed
lock-session pieces, worktree checks, clean-path inspection, legacy/intake checks,
managed-highlight inspection, and single-PDF consistency check. Reuse
`ref_library::migrate_tasks::{snapshot_before_images, restore_before_images}`,
`highlights_ref/io.rs::rename_no_replace`, atomic writes, scoped Git helpers, and
rename's committed-despite-error logic where their semantics fit. Keep operation
specific messages and policy at the callers: rename currently has offline and dirty path
behavior that overwrite must not inherit wholesale.

Suggested new module: `highlights_ref/reference_mutation.rs`. Migrate rename to its
shared pieces in the same phase. Do not build a generic transaction framework or change
rename's public output. Clean-directory checks must cover descendants for overwrite;
retain exact path behavior for rename where needed.

Validate existing rename CLI/unit suites, including rollback, dirty-path refusals,
family rename, and no-op cases, plus formatting and lint. No create behavior changes.

### `prepare`: strict plan and note transforms

Own new `highlights_ref/overwrite/plan.rs` and `overwrite/archive.rs`, with focused
module tests. Reuse `SelectedFamily::selected_member`, `disambiguate`, the library task
locator, `ParsedNote`, `note.rs`, the managed-embed helpers in `reading_plan.rs`, and
`ref_library/lineage.rs` parsing/rendering. Separate family classification from reread
allocation in `recapture.rs` so overwrite does not need a free version stem. An index
read failure must be a hard overwrite error, not the best-effort empty result currently
returned by `recapture::index_rows`.

Produce a typed `OverwritePlan` containing the selected reference, kept task and
frontmatter, archive decision/content, affected bundle paths, and preimage/absence
conditions. Keep command-line provenance in a small overwrite request type so a
defaulted status cannot be mistaken for an explicit reopen. The planner should be
reusable before render and after locked pre-sync.

Tests cover all refusal codes and incompatible flags, newer versus explicitly selected
older reading, multiple unrelated matches, `__v999`, prose versus skeleton, open
checkbox detection outside code fences/comments, legacy tracker neutralization,
block/inline lineage input, repeated archives, audio skeletons, and kept local keys.
Keep ordinary reread/fresh/queued matching behavior intact. Do not enable live writes.

### `recovery`: durable intent lifecycle

Own new `highlights_ref/overwrite/recovery.rs` and entry hooks. Implement the intent
schema, atomic durable writing, checked path manifests, scoped restore/unstage,
committed-state proof, clear-on-success, and idempotent retry. Hook it before ordinary
create dispatch and before all writing scan branches, using shared locking without
altering unrelated no-intent runs. Coordinate `overwrite/mod.rs` exports with `prepare`;
own entry-hook changes in `create.rs`, `mod.rs`, `scan_json.rs`, and `intake.rs` only
where necessary.

Use isolated Git fixture vaults and state roots. Test half-written PDF/note/sidecar, new
archive removal, committed-but-uncleared intent, same-stem previous commits, repeated
recovery, failed recovery retaining its intent, unrelated files/index preservation,
invalid paths, dry-run immutability, and parseable JSON. Include a CLI fixture with a
pending intent; no real vault or network service is required.

### `transaction`: staged bundle to verified commit

Own `highlights_ref/overwrite/execute.rs` and the typed staged-input/result boundary.
Define one `StagedOverwrite` owning private PDF/audio paths and final marker metadata,
and one executor consumed by every route. It receives the plan and request, performs the
full transaction above, and returns structured report data instead of route-specific
printing. Build focused test staging fixtures before connecting external renderers.

Reuse `sync::{plan_single_pdf_sync, write_single_pdf_sync}`. Provide a narrow
overwrite-specific task-status/title update where needed; do not route through
`ReadingTaskAction::Insert` or weaken ordinary scan safeguards. Preserve blocked
checkbox overlays and closed-task stamps without normalization when no status was
requested. Check archived-task behavior explicitly. Include all predicted sync/task
paths in the transaction and refuse unexpected ones.

Exercise failpoints after rendering, after intent persistence, after each mutation,
during verification, before commit, after a successful commit reported as error, and at
post-sync. Implement failpoints through test seams, not a public CLI flag. Each outcome
must be either the complete old state or the verified committed new state, with durable
recovery available for simulated crashes. Prove sidecar removal prevents stale
annotations from returning on a later scan. No route is enabled until the executor's
safety tests pass.

### `local`: local renderers, status, and audio

Own `create.rs` option plumbing and local Markdown, original local PDF, and same-day
vault-note snapshot routes. Introduce an explicit typed capture outcome for ordinary
create/reread/overwrite instead of inferring overwrite from error strings or stacking
booleans. Keep selected-member defaults separate from reread's newest-member defaults.
Connect dry-run planning before render and connect the existing renderers to private
staging before any ordinary install/`finish_create_intake` call.

Implement the shared audio policy here, including no implicit inheritance changes,
explicit `-a`, and private `-L` rendering. Existing ordinary create and reread paths
must still use their current intake and collision behavior. Keep unavailable routes
truthfully refused until their own phase enables them. Factor small staging adapters
from `stamp_listen_and_install_pdf`, Markdown rendering, and snapshot code rather than
recursing into the CLI with hidden force overrides.

CLI tests use existing fake rendering/listen helpers and Git fixture vaults: a normal
overwrite; no archive; prose archive; selected older reading; same task with read, wip,
ready, blocked, and archived status; explicit `-s ready`; changed title retaining the vN
alias; kept/replaced audio; fetch/render/listen failure leaving bytes unchanged; no
writes or adapter calls on `-d`; library-PDF source refusal; unchanged ordinary
create/reread. Update help/docs only to describe routes actually enabled at this point.

### `web`: single URL renderers

Own the PDF-URL/arXiv paths in `create.rs` and article capture in `clip.rs`. Feed
existing network/adapter results into the same staging/execution interface. Saved HTML,
source metadata overrides, id/status stamping, title changes, and listening must reach
overwrite without selecting old attach behavior. In particular, current
`force != Reread` attach predicates must become correct for both explicit modes.

Retain duplicate and eligibility refusal before any fetch/adapter/listen call. After
fetch, validate redirected source identity against the selected family before
publication; do not change which family gets overwritten as a side effect of a redirect.
Locked revalidation still applies after long rendering. Hidden clip aliases must share
the same semantics. Automatic producers continue to pass no force mode.

Tests use fake curl, fake clip, and fake listen only: each URL kind succeeds on a filed
unannotated reference, wrong-family redirects and changed preimages refuse,
annotated/incompatible targets do no network work, and both `-L` meanings remain
correct. Update contract/help text as these routes become supported.

### `collections`: exact sets and complete user reporting

Own `collection.rs`, remaining cross-route report integration, and final behavior docs.
Adapt its existing exact-family preflight and all-or-nothing render to the shared
overwrite executor. A matching unordered URL set selects one family; argument order
still governs rendered content. Partial overlap must refuse with the established
exact-set hint under every mode, including listen. A failed member render leaves the
entire old bundle intact. Use the selected member for defaults, metadata, status, audio
preservation, and `-N` handling.

Add shared overwrite refusal remedies in `recapture.rs`/`duplicate.rs`: `-f overwrite`
names the actual selected target and says notes move to `ref_history/`. Keep computed
reread names and only applicable `-L` hints. Bare reread of a ready zero-highlight
reference prints the parent plan's hint that overwrite would replace it instead. Keep
plain piped/NO_COLOR output and collision substrings used by ingest.

Success starts `ok replaced Highlights-ready PDF`, then `overwrite:`, the archive or
no-archive `earlier:` line, `task: kept <path> ^<id> [<mark>] <status>`, optional
kept-audio warning, and the scoped commit/push result. Mention `pass -s ready to reopen`
for a kept closed task. Do not print annotated-PDF or moved-highlight counts as if that
later phase had shipped. Dry runs describe the same plan and end with `writes: none`.

Remove obsolete filed-overwrite-unavailable code and tests now that every route is
integrated, while retaining the annotated interim refusal. Update
`docs/highlights-create.md` Recapture/Dedupe/Output/Audio and
`docs/highlights-ref-sync.md` for `recaptured`, frozen notes, and recovery. Update clap
help and run `just docs-gen` and `just docs-check`. Preserve the parser matrix and the
shipped glued-short-option wording; this epic changes no parser syntax.

### `acceptance`: cross-route preservation evidence

Extend behavior-focused integration tests, preferably in a focused
`tests/cli/highlights/overwrite.rs` registered from the existing test module. Reuse
`bob_command`, `write_highlights_pdf`, `configure_test_git_identity`, existing fake
clip/curl/listen helpers, and the synced-vault pattern in
`tests/cli/ref_library/rename.rs`. Avoid copying an entire existing test module or
writing tests that only mirror helper implementations.

The acceptance matrix must demonstrate:

- Two same-day prose overwrites create distinct frozen notes, keep both `earlier:`
  entries newest-first and the same `created`, update `recaptured`, and retain one
  reading task at the same address. Existing family lineage survives both overwrites.
- No-prose overwrite creates no archive, removes each supported sidecar shape, keeps the
  no-archive message, and remains stable after sync/scan.
- Task children, dependencies, relations, keep/freshness fields, and inbound
  note/heading links are preserved; explicit status and title are the only permitted
  requested task changes. An archived reference does not create a second task.
- `ref_history/` adds no library rows, dedupe hits, reading tasks, or annotation tasks
  across list/find/doctor/create and the relevant vault task walkers. Inspect actual
  walker behavior; add an exclusion only if required, without changing the frozen format
  into a new reference type.
- Every refusal and option combination above leaves vault and intent state unchanged;
  every route has an end-to-end success case and read-only dry run. Collection order,
  partial overlap, automatic producers, queued force, and ordinary reread regressions
  remain covered.
- Recovery is exercised via both create and scan, including JSON/intake-only variants,
  crash before commit, commit before marker cleanup, and a second overwrite of the same
  stem. Commit/post-sync failures never advise repeating an already committed overwrite.
  Unrelated dirty/staged files survive scoped mutation and recovery.
- Rename behavior, family health, normal scan status semantics, and stable sync round
  trips remain intact. No temporary unavailable path remains for eligible unannotated
  references on any host.

Run targeted suites while implementing; at acceptance run `just check`, `just docs-gen`,
and `just docs-check`. Use `/sase_monitor` for commands that need a long-running
handoff, and wait for the monitor handoff command itself to finish. Record
commands/results and any baseline failures for the land agent. A failure reproducing
identically on a clean base is a `PROPOSED FOLLOW-UP:` note, not a reason to leave
completed phase work open.

## Phase hygiene and land responsibilities

Each worker reads its own bead and this design, makes only its bounded changes, runs
relevant checks, and closes only its assigned child phase. No worker creates task beads;
discovered work is a `PROPOSED FOLLOW-UP:` note on that worker's phase. Read required
memory through `/sase_memory_read`; read artifacts through `sase artifact read`. No
linked repository edits, live-vault operations, client changes, release version edits,
CHANGELOG edits, or memory edits belong here.

Keep intermediate commits buildable and the shipped command truthful. Shared code must
have an actual caller or appropriately scoped test/build exposure; do not add blanket
lint suppression. If temporary `--epic-symbol` entries are necessary, key them to the
still-open phase that will consume the symbol and remove or re-key them before closing
their owner. Every phase runs `sase bead epic-symbols <its-id>` before close. The final
child land agent also checks bob-cli-c9.4 and the child epic for leftovers; the initial
check found none for bob-cli-c9.4.

The child land agent verifies all acceptance evidence on the combined tree, reruns
checks only where integration changes or unresolved failures justify it, then closes
**bob-cli-c9.4 only among the outer epic's beads** with a verification note. It records
the transaction/staging extension points and annotated refusal for bob-cli-c9.5.
Annotated PDF movement/coordination, inbound highlight-link retargeting, late-save
guards, and glossary/decision-record work remain with their already-defined outer
phases.
