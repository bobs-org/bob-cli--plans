---
tier: epic
title: Coordinate remaining reference PDF writes and moves
goal: With PDF coordination enabled, reference renames, id repairs, and capture replacements
  preserve presenter saves, notify moves, and hold without partial publication; Bob
  Refs and the scratch acceptance kit reflect the contract.
phases:
- id: move-coordinator
  title: Add bounded save-and-move coordination
  size: medium
  depends_on: []
  description: 'move-coordinator: add a move-capable coordination scope with save-before-write,
    explicit forward and reverse move notifications, the existing cancellation gate,
    and portable fake plus macOS NSDocument tests. Extend the explicit-write session
    seam without changing existing scan or overwrite behavior.'
- id: rename-transaction
  title: Coordinate the complete rename and id-repair transaction
  size: medium
  depends_on:
  - move-coordinator
  description: 'rename-transaction: put ref rename, same-stem id repair, verification,
    scoped commit, and rollback inside one explicit coordination scope. Revalidate
    after presenter saves, preserve the post-save rollback baseline, notify PDF moves,
    and return truthful retryable errors without claiming success on a hold.'
- id: staged-install
  title: Build a guarded stamped-PDF publication helper
  size: medium
  depends_on:
  - move-coordinator
  description: 'staged-install: separate private PDF stamping from public installation
    and add one guarded install boundary, used by install_stamped_pdf. Recheck destination
    identity and marker after access, preserve presenter changes, and keep companion
    audio inside the granted callback. Cover the helper and PDF publication path with
    injected-coordinator tests.'
- id: capture-writers
  title: Route capture replacement writers through guarded publication
  size: medium
  depends_on:
  - staged-install
  description: 'capture-writers: migrate stamp_and_install and remaining direct public
    PDF copies from create, clip, and collection to the shared guarded install boundary,
    including companion audio ordering. Preserve existing filed-overwrite coordination
    and all force-mode, collision, source-store, and ingest contracts.'
- id: presentation-acceptance
  title: Update rename presentation, contracts, and acceptance coverage
  size: medium
  depends_on:
  - rename-transaction
  - capture-writers
  description: 'presentation-acceptance: remove the Bob Refs close-the-PDF rename
    hint, verify retryable errors retain the draft, document the coordination contracts,
    and extend the scratch-only acceptance kit for rename, same-stem id repair, and
    forced PDF publication. Run portable checks and compile the Mac app and tests
    in a disposable copy; record real Highlights acceptance separately if it cannot
    be run.'
proposed_by: bbugyi200.athena.bob-cli-an.6.6.6.5.6.5.5.6.6.6.6.6.6.6.2
proposed_with: codex/gpt-6-astra
parent_bead: bob-cli-an.6.6.6.5.6.5.5.6.6.6.6.6.6.6.2
create_time: 2026-10-26 22:51:26
status: done
bead_id: bob-cli-an.6.6.6.5.6.5.5.6.6.6.6.6.6.6.2.1
---

- **PROMPT:** [prompts/202610/remaining_pdf_write_coordination.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202610/remaining_pdf_write_coordination.md)
- **PARENT:** [202610/backlog_batch15_vault_safety_pdf_moves_and_red_gates.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/backlog_batch15_vault_safety_pdf_moves_and_red_gates.md)
- **BEAD:** [bob-cli-an.6.6.6.5.6.5.5.6.6.6.6.6.6.6.2.1](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-an/bob-cli-an.6.6.6.5.6.5.5.6.6.6.6.6.6.6.2.1.md)

# Coordinate the remaining reference PDF writers

Finish assigned phase `bob-cli-an.6.6.6.5.6.5.5.6.6.6.6.6.6.6.2` from
`plan:202610/backlog_batch15_vault_safety_pdf_moves_and_red_gates.md`. This is a bounded
child implementation epic for that phase, not the next backlog batch. The parent batch's
land agent retains ownership of `bob-cli-az` and the backlog continuation campaign. Do
not enumerate or implement unrelated backlog work here.

The outcome is that, when `highlights.coordinate_pdf_writes` is enabled on macOS,
reference rename/move, same-stem id repair, and capture replacements ask open PDF
documents to save before bob changes them. A successful move informs the document of its
new path. A hold makes no bob-owned publication changes and invites an explicit retry.
No branch falls back to an uncoordinated write after a failed or held coordination
request.

## Evidence and boundaries

The phase was in progress with no earlier-worker notes when inspected. The tree was
clean at bob-cli `dc040b02`. Its prerequisite's wrong-vault refusal is present in
`rename/plan.rs`; preserve it. No `--epic-symbol` entries were associated with the
assigned phase at planning time; check again at closure.

The original task describes some paths before recent refactoring. Plan from the current
call graph instead of wrapping every `atomic_save_pdf`:

- `src/native/file_coordination.rs` and `file_coordination/macos.rs` implement a bounded
  `.forMerging` writer. `attempt.rs` already prevents a late callback from writing after
  timeout and never interrupts a running accessor.
- `highlights_ref/marker_write.rs` has `for_explicit_write`, a fake coordinator,
  `coordinate_explicit_pdf_write_with`, and `for_external_coordination` to avoid nested
  coordination. The latter does not independently grant access.
- `highlights_ref/rename/plan.rs` currently moves the bundle before writing its id with
  `MarkerWriteSession::uncoordinated()`. Commit and commit-error rollback occur outside
  `apply_and_verify`. Both transaction-error and commit-error rollback must be covered
  by the new outer scope.
- `stamp.rs::stamp_and_install` installs at the public destination directly.
  `pdf_target.rs::stamp_pdf_to_scratch` writes only a private staged PDF; its
  `atomic_save_pdf` is not itself the unsafe public replacement.
  `pdf_target.rs::install_stamped_pdf` publishes through `atomic_copy`.
- `create.rs` has direct public copies as well as the shared stamp helpers; `clip.rs`
  and `collection.rs` also publish via these paths. Several routes install audio before
  the PDF. Merely wrapping `atomic_save_pdf` would miss copies and leave partial
  companion changes on a hold.
- `overwrite/execute.rs` already coordinates the filed-overwrite transaction, rechecks
  its manifest inside access, commits there, and uses externally coordinated
  sync/recovery. Preserve this implementation and its annotated archive and rollback
  contracts; do not duplicate coordination around it.
- The obsolete hint appears in bob-mac-capture `Sources/RefsCore/RefsRename.swift`, its
  test, and `Sources/BobMacCapture/Refs/RefsRenameCardView.swift`. The model already
  keeps the rename draft when bob returns an error.

Read `plan:202610/highlights_coordinated_marker_writes.md` through `sase artifact read`
and these memories through `/sase_memory_read` before implementation:
`decisions:open-pdf-writes-are-coordinated`,
`decisions:reference-ids-follow-file-stems`,
`decisions:reference-renames-keep-identity`, and, for the Mac UI,
`decisions:mac-capture-is-a-thin-client`. The current docs allow same-stem id repair on
annotated references, while a full filename rename still refuses them. Keep that
distinction.

No memory changes are authorized. The parent's final `deploy_guard = refuse` decision is
unchanged and unrelated to this work. Leave the shipped coordination default off. Do not
add flags, version changes, or CHANGELOG entries. Do not change full-rename eligibility,
force-mode selection, reference ids, reading-task identity, scan receipts, or the
thin-client boundary beyond the changes stated here. Keep the existing scan/sync hold
exit semantics.

## Shared transaction contract

1. Dry runs stay read-only: no coordinator, presenter save, live file installation, or
   reading probe. Scratch rendering/stamping is separate from publication.
2. Load the existing configuration through `MarkerWriteSession`. These explicit commands
   are exempt from the frontmost-reading hold, but still coordinate and time out. Retain
   existing budgets and environment overrides. A compound move uses one bounded request,
   not a fresh timeout at every internal step.
3. On non-macOS or when the key is off, preserve current direct behavior. On an enabled
   Mac, timeout/cancellation means no accessor ran; a platform error also means no
   accessor ran. Never turn either into permission to run directly.
4. Read and validate current bytes after the presenter has saved. A changed marker gives
   the established `marker_changed` code; another changed destination or bundle preimage
   gives `preimage_changed` or the existing relevant refusal. Distinguish an editor's
   save from bob's writes in output and rollback evidence. A hold may leave the editor's
   newly saved bytes, but no bob-authored note, PDF, audio, source pointer, queue entry,
   move, or commit.
5. For explicit rename/create, use the existing command error surface with a nonzero
   exit and a concrete manual retry hint. Reuse `coordination_timeout` and
   `marker_changed`; keep unexpected coordinator failures distinct. Do not promise that
   the next scan will retry a rename or create. Rename JSON stays schema 1 with
   `ok: false` and `error: {code, message, hint}` on these refusals; successful
   envelopes stay compatible. A hold must never masquerade as `unchanged`, `id_updated`,
   or a successful move.
6. All writes, verification, scoped commit, and error rollback for a rename remain
   within access. Pre-sync and post-sync stay outside OS coordination but under their
   existing vault locks. Preserve committed-despite-error handling and post-sync failure
   reporting. A Git failure cannot cause a later uncoordinated reverse move or
   restoration of pre-save bytes.

## Phase 1: move-coordinator

Add a small move-capable variant beside `coordinate_merging_write`, sharing
`attempt::run` rather than duplicating its watchdog. Its scoped context lets the
transaction perform a no-clobber move and notify its result, and then perform a reverse
move and notification if rollback is necessary, before the callback returns. Keep
Foundation objects private to the macOS implementation. A context/token must not escape
the synchronous accessor or be usable as evidence of coordination later. A
direct-platform context performs the same filesystem operation without OS notifications.

Request source access with `ForMerging | ForMoving`: saving before mutation and move
notification are separate requirements. Notify `itemAtURL:didMoveToURL:` after each
successful physical move, including its successful inverse. Balance `willMoveToURL:`
where used; do not issue a success notification when the move fails. Preserve
`rename_no_replace` and destination checks; never use `ForReplacing`, option-less
writes, or an overwrite rename for the destination. Validate the accessor URL before any
mutation as the current merging primitive does. The source and destination must remain
the planned paths; an unexpected presenter relocation is a refusal, not silent
retargeting.

This combined-option design is an implementation choice to verify with the Mac fixture,
not a claim that real Highlights acceptance has already passed. Apple documents the
distinct
[move option](https://developer.apple.com/documentation/foundation/nsfilecoordinator/writingoptions/formoving),
[move notification](<https://developer.apple.com/documentation/foundation/nsfilecoordinator/item(at:didmoveto:)>),
and
[restriction on nested coordinated writes](<https://developer.apple.com/documentation/foundation/nsfilecoordinator/coordinate(writingitemat:options:error:byaccessor:)>).
Do not nest an inner merging writer inside a moving writer. Expose a matching test fake
through the explicit session so later phases can assert ordering and inject saves,
timeouts, and failures on Linux.

Test no presenter, grant, blocked presenter, late callback, changed actual URL, accessor
error, destination collision, forward move, and inverse move. Extend
`tests/fixtures/file_coordination/presenter.swift` and `macos_tests.rs` to observe the
document URL and a later save: it must save to the new path without recreating the old
one; rollback must restore its URL as well. Verify dirty contents save before access and
that timeout never runs the accessor. Run the Mac-only code on a disposable Mac checkout
when available, not just Linux cfg tests. Record any unavailable Mac checks explicitly
for the child lander.

## Phase 2: rename-transaction

Refactor `rename/plan.rs` around a prepared transaction and one explicit session. The
existing clean/dirty eligibility checks still run before requesting access. For a real
move, request the move-capable scope on `from.pdf` even if the id is absent or
unchanged. For same-stem id repair, use ordinary merging access.
Unchanged/already-renamed requests need no write scope.

Capture the selected reference identity, expected marker, and touched-path preimages
before access. Revalidate before moving or rewriting anything. A full rename whose PDF
changed during the save holds conservatively, even if its marker is unchanged: the saved
annotations can invalidate v1 rename eligibility. Do not silently refresh the plan into
an annotated rename. Return a retry/sync hint and leave the saved source in place. Other
changed paths also refuse before writes.

Same-stem id repair can retain ordinary newly saved annotations when the marker still
matches. Re-read the saved PDF and locate the marker annotation again, because object
ids may change. Rebuild the narrow sync plan against those bytes and allow it only if
its existing side-effect safety predicates hold; otherwise hold with no bob writes. This
must not import new annotation tasks or broaden rename's mutation set. Use the saved
bytes as the PDF rollback preimage and retain the last digest guard. Never restore the
bytes from before the presenter saved.

Run existing marker sync under the proven outer scope using the external-scope session
instead of `uncoordinated()`, without a second OS request. Separate the PDF movement
from generic `rollback_moves` so both directions go through the move context. Keep all
affected Markdown restores and PDF restores in the same scope and retain
concurrent-change guards. Report an incomplete rollback truthfully rather than claiming
every path was restored.

Tests in rename's unit tests and `tests/cli/ref_library/rename.rs` must cover: one grant
for the entire operation; timeout/platform failure before writes; saved-first full
rename hold; changed marker; same-stem annotated id repair with renumbered PDF objects;
id-less move; absent target becoming occupied; verification failure; commit failure with
forward and reverse notifications; post-save bytes surviving rollback; and a commit
reported as an error remaining committed. Retain the wrong-vault, offline, collision,
link rewrite, id suffix, and already-renamed coverage. Exercise human and JSON refusal
output.

## Phase 3: staged-install

Build the shared publication boundary in a focused module under `highlights_ref`, with
unit tests and a small explicit API for phase 4. Adopt it in
`pdf_target.rs::install_stamped_pdf`, which already owns the PDF copy and its companion
audio installation. This provides a production consumer without an unused-code
exemption. Phase 4, which depends on this phase, migrates the other public writers and
fixes routes that install audio outside this helper.

Separate rendering/marker stamping into private scratch from installation into the
requested public path. Before publication, capture destination absence or its identity,
bytes/digest, and marker if present. Carry that expectation across the bounded request;
recheck it inside access, including path type and symlink guards. A previously absent
target becoming occupied must refuse, not bypass coordination. Keep the existing force
permission checks as an independent requirement: coordination never grants permission to
overwrite a collision.

For an existing destination, request ordinary merging access. If a presenter save
changes its bytes, refuse this replacement without publishing; use `marker_changed` for
a different marker and `preimage_changed` for other saved changes. A force flag
authorizes the planned replacement, not throwing away a save that arrived while waiting.
Do not merge old annotations into a newly rendered capture. For a new destination, use
atomic no-clobber publication so an intervening occupant cannot be overwritten.

The callback boundary must allow companion publication to occur only after the grant and
final destination checks. Preserve existing failure cleanup. Keep
source-store/ingest/queue actions from running on a held install; private staging is
allowed. Document which helpers require an already-granted scope so phase 4 does not
wrap the existing filed-overwrite executor again.

Tests inject a save with unchanged marker, a changed marker, hold, coordinator failure,
and a late destination collision. Assert byte-for-byte destination and companion
preservation, no publication callback on refusal, and exactly one callback on success.
Test missing destination, key-off/Linux pass-through, and original replacement errors.
Add a local-PDF route regression through the migrated `install_stamped_pdf` and
integrate without introducing warning exemptions.

## Phase 4: capture-writers

Audit and migrate the remaining public replacement call sites in `stamp.rs`,
`pdf_target.rs`, `create.rs`, `clip.rs`, and `collection.rs`. The baseline includes both
direct `atomic_copy` calls and `install_stamped_pdf`/`stamp_and_install`; search afresh
because line numbers and callers can change. Keep private scratch writes direct. Apply
the guard to existing public output paths, including queued/intake and exact `-o`
outputs, while retaining existing force mode selection and restrictions. Do not create a
second coordinator inside `overwrite/execute.rs` or its sync path.

Move companion-audio installation that currently precedes publication into the granted
boundary, including `--listen` output. Rendering, fetching, and narration may still
occur beforehand in private scratch. On a hold, do not install new audio, erase an old
audio file, publish source-store pointers, enqueue ingest, or print a success line.
Preserve the existing post-listen retry information and scratch retention where
appropriate. Keep successful create-time ingest outside this install scope if it
requires its own write: do not nest coordinators.

Add route-level regressions for Markdown, local PDF, web/collection shared publication,
exact existing output, queued replacement, audio/listen publication, fresh output, dry
run, and key-off behavior. Use unit seams to simulate Mac coordination on Linux without
production environment switches that fake grants. Retain existing filed/annotated
overwrite and recovery suites as regression coverage. Document a call-site inventory
showing every public writer is guarded or already protected by the filed-overwrite
transaction.

## Phase 5: presentation-acceptance

Use `/sase_repo` to open `bob-mac-capture`; read its `AGENTS.md`. Remove
`RefsRenamePresentation.quietHint` and its view, and update the assertion in
`Tests/RefsCoreTests/RefsRenameTests.swift`. Keep the ordinary rename caption. Add
behavior coverage that a `coordination_timeout` or `marker_changed` response retains the
entered stem, displays bob's message/hint, and never moves selection or announces
success. Reuse the current error model; the app does not infer file state or read the
vault.

Read `docs.md` through `/sase_memory_read`. Update `docs/ref.md` and
`docs/highlights-ref-sync.md` for enabled/disabled behavior, the explicit-command retry
contract, and the distinction between full rename and same-stem id repair. The
historical 2026-10-15 rename acceptance predates id rewriting and must not be presented
as verification of this change. Preserve that historical fact with a clear scope label.
Do not claim the off-by-default configuration guarantees safe open-document writes. No
help changes or new CLI switches are expected; if any are necessary, read `cli_rules.md`
and regenerate docs with `just docs-gen`.

Extend `scripts/dev/highlights_coordination_acceptance` and its `--self-test`:

- Clean open-document rename changes the filename, marker id, and inbound links; a
  subsequent edit/save goes to the destination, and the old PDF is not recreated.
- Unsaved annotations during full rename are saved and cause the conservative
  no-mutation hold. Sync then reports the existing annotated-rename refusal.
- Same-stem id repair on an annotated PDF keeps unsaved annotations, writes the id, and
  allows a later save without reverting the id.
- Existing-output forced installation coordinates; a newly saved change holds before
  PDF/audio publication. An unchanged target replaces as requested and reloads. A
  blocked save times out with no bob changes.
- Record page position, same-tab behavior, conflict dialogs, and later save results. A
  failure never enables the setting or asks an agent to automate the GUI.

Use only isolated scratch vaults/config/state with `BOB_DIR` matching `-b` and offline
rename fixtures. Keep this new checklist's verdict separate from the original
marker-write enablement verdict and annotated-overwrite checklist. Run `bash -n` and the
kit's self-test with the just-built bob. Real Highlights requires Bryan's interactive
Mac session: do not claim it ran, do not silently enable the key, and record it as an
outstanding acceptance limitation if absent.

Run `just portable-check` in the Mac repo. Because the view changes a macOS target, also
rsync to a disposable Mac scratch copy, excluding `.git` and `.build`, run
`Scripts/bundle.sh` and `just typecheck-tests`, then remove the scratch copy. Use the
tailnet memory before accessing that host. Never use Bryan's checkout or `just install`.
No local XCTest is available on the Command Line Tools Mac; typechecking is not test
execution. Run/inspect the real macOS CI tests when available; explicitly report any
unverified native targets when the Mac is offline.

## Verification, handoff, and closure

Each child phase is direct, bounded `medium` implementation work. Dependencies are
explicit: the native move/session contract precedes rename and publication; the first
guarded install route precedes remaining route migration; the UI and acceptance work
follows both. Do not create further planning layers just to defer these specified
changes. Check current history before implementing anything already landed.

Build bob using `$CARGO_TARGET_DIR`. Run focused tests during development and the full
`just check` on the final changed bob-cli tree, never `just check-full`. Run
`just docs-check` for documentation edits, opening mounted repositories through
`/sase_repo` when the gate needs them. No bob-plugins edits or deployment are planned.
The native move phase must compile the macOS Rust implementation and run its fixture
when available; Linux success is not native verification. Use `/sase_monitor` for long
checks and wait for handoff commands themselves to exit. Record host, revision, command,
result, and any skipped native coverage.

When a check fails, establish whether it reproduces identically on a clean base before
declaring it unrelated. Record an unrelated baseline failure as `PROPOSED FOLLOW-UP:` on
the assigned phase (including any existing tracking bead) and close completed phase work
anyway, as the user's launch contract requires. Do not create task beads. Report memory
corrections only as such follow-ups; memory edits remain forbidden. Do not use a
pre-existing failure to hide a new failure in these paths.

Before closing any child phase, run its `sase bead epic-symbols` check and resolve or
re-key leftovers to a still-open dependent phase or this child epic. Do not leave stale
warning exemptions. Use normal SASE final declarations for repository obligations; no
manual Git commit is requested by this plan. Read other repos only through `/sase_repo`
and sidecar artifacts only through audited artifact reads. No live vault edits, app
installs, credential handling, or live GUI automation.

The child epic's land agent consolidates evidence and checks the integrated result; it
does not expand into the parent backlog campaign. It verifies:

1. Existing scan/sync/ingest and filed-overwrite coordination still works, the remaining
   public writers have the documented guard, and dry runs do not save open documents.
   All native and portable test limitations are explicit.
2. A hold never moves a bundle, commits a partial rename, loses a presenter save, or
   publishes replacement audio. Forward and inverse moves notify the document.
3. The rename hint is removed and contracts and the executable scratch checklist agree.
   No real-Highlights acceptance is claimed without a recorded run.
4. Every child phase is complete; repository changes and check evidence are recorded
   through the SASE finalization flow.

The user explicitly assigned this child lander completion of the original phase. After
the above, run:

```bash
sase bead epic-symbols bob-cli-an.6.6.6.5.6.5.5.6.6.6.6.6.6.6.2
sase bead close bob-cli-an.6.6.6.5.6.5.5.6.6.6.6.6.6.6.2 --note "<integrated verification, revisions, and native acceptance limitations>"
```

Resolve or re-key any original-phase symbol entries before that close. Leave
`bob-cli-az`, the batch-15 epic, and every higher ancestor open for their own land
agent. Supply evidence for their closure, not instructions to close them. The batch-15
lander still owns its mandatory five-task successor campaign and all parent decisions;
this child epic neither completes nor replaces that work.
