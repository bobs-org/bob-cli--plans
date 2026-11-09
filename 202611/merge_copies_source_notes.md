---
tier: epic
title: Merge copies source notes instead of moving them
goal: 'Every task merge, including the `merged into` relation label, copies the source
  task''s notes to the destination while the cancelled source tombstone keeps its
  original notes, so links into those notes keep resolving where they always did.

  '
decisions:
  glossary_merge_terms:
    ask: Correct the Task Merge and Task Relation glossary strands, which say a merge
      cuts or moves source notes?
    memory:
    - glossary:task-merge
    - glossary:task-relation
    default: true
    corrects: It cuts the selected source blocks and appends their notes to the destination,
    answer: true
  decision_record:
    ask: Add a decisions memory record stating that a merge copies source notes and
      never moves them?
    memory:
    - decisions:merge-copies-source-notes
    default: false
    answer: false
phases:
- id: engine
  title: Merge engine copies source notes instead of cutting them
  depends_on: []
  size: medium
  description: 'engine: stop cutting source bodies, strip identity from copies, keep
    in-copy links on S''s originals, update wording, docs, memory, and tests.'
- id: nav-copy
  title: Nav Links-stage merge strip says copies
  depends_on:
  - engine
  size: small
  description: 'nav-copy: change the Links-stage consequence strip to copies N notes,
    update tests and docs, rebuild, and sync plugins.'
- id: mac-copy
  title: Bob Mac Capture merge source row says copied
  depends_on:
  - engine
  size: small
  description: 'mac-copy: change the transfer card merge source detail to N notes
    copied, update tests, compile, and watch macOS CI.'
proposed_by: bbugyi200.athena.1an
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-09 18:00:11
status: wip
bead_id: bob-cli-je
---

- **PROMPT:** [prompts/202611/merge_copies_source_notes.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/merge_copies_source_notes.md)
- **BEAD:** [bob-cli-je](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-je/README.md)

# Merge copies source notes; it never moves them

## Goal

When a task is merged into another task, drawer, or section, its notes should be copied
to the destination. They should not be moved. Today the merge engine
(`src/native/task_merge/`) **cuts** the source task's authored body out of the source
note, pastes it under a title bullet in the container, and relocates every descendant
anchor with vault-wide link repair. After this epic:

- **T (the container)** still gets the same title bullet with S's authored body beneath
  it. That body is now a **copy**.
- **S (the source)** keeps its whole body byte for byte. As before, it is cancelled with
  `[cancelled:: today]`, gets `🪢 LINKS: merged into [[T]]`, and has its Cancel Log
  entry prepended when one exists. The tombstone therefore keeps the full history.
- Every link that pointed into S's notes keeps resolving to the **original** on S.
  Nothing is relocated, so the merge does no vault-wide anchor repair.

Everything else in the contract stays the same: refusals, dependent re-pointing, pruning
today's live links, the hand-off offer, provenance, write order, idempotency, and the
JSON envelope.

### Scope: every merge gesture, not only the `~` label

Every merge gesture runs one engine and writes the same `merged into` label: capture
`~`/`~~` with a merge label, the nav Links stage (which commits through
`bob task merge`), capture `>` to a task, drawer, or section, capture `<`, and
`bob task merge`. ML6 in `docs/task-relations.md` pins these gestures as byte-equivalent
(`@a+x ~db:t|"merged into"` ≡ `@a+x > @db+t`). So copy semantics become the engine's
single rule and the gestures get no mode flag. `>` to a bare **note** is a task **move**
(`task_move`), not a merge, and does not change.

## Design rules (all phases cite these)

1. **Copy, never cut.** The source's authored body is copied: continuation paragraphs,
   wrapper-free tables and fences, and authored children with their nested drawers and
   closed nested tasks. Direct managed children (Depends-On, BLOCKS, LINKS, LINKED FROM,
   and the Schedule, Work, Rotten, and Cancel logs) still stay only on S, as they do
   today.
2. **A copy never carries identity.** Before insertion, every live block anchor (`^id`)
   in the copied lines is stripped, together with its leading space. Any `[id:: …]`
   dependency-id field on a copied task line is stripped too, because it names S's
   original task. Identity stays only on S. As a result, a same-note merge can no longer
   produce an anchor collision or a rename. Fenced and inline-code text is never
   touched.
3. **Links inside the copy keep their targets.** A same-note block link in the copied
   text (`[[#^x]]`, or the markdown `(#^x)` form) is rewritten to name S's note
   explicitly (`[[S-note#^x]]`), so it still resolves to the original on S. Links that
   already name another note are left unchanged. These rewrites are the only "repairs" a
   merge payload makes now.
4. **No vault-wide repair for the payload.** No anchor moves, so the merge rewrites no
   other note's links. Dependent re-pointing (Depends-On rewrites, BLOCKS mirrors) is a
   separate step and is unchanged.
5. **Open nested tasks are still refused** (`merge_source_open_subtasks`). A copy would
   duplicate a live task.
6. **Wire compatibility.** The JSON field names `moved_lines` and `moved_notes` stay as
   they are and now count the copied lines and notes. Every `schema_version` stays 1,
   and no client decoder changes. Only the human and Bob-authored copy changes wording.
7. **Canonical copy** (`docs/task-merge.md` §10 is the contract; Rust, JS, and the Mac
   app use it verbatim):
   - source row: `⊘ S  path:line · Prev → Cancelled · N notes copied` (singular
     `1 note copied`, unchanged `· title only`);
   - nav merge strip and ML7 consequence strip:
     `⤵ Merges “S” into “T” · copies N notes · cancels “S” · …` (singular
     `copies 1 note`);
   - Mac transfer card source detail: `N notes copied` / `1 note copied` / `title only`.

## Phase `engine`: merge engine copies source notes (bob-cli)

Before you touch docs/ or CLI help, read the docs memory with
`sase memory read docs.md -r "<why>"`. Read `docs/task-merge.md` in full: it is the
contract, and it says to fix the doc first in the same change.

### Engine (`src/native/task_merge/payload.rs`, `prepare_task_merge`)

- Keep `source_payload`. It still computes the authored-body ranges and fragments,
  enforces the open-subtask refusal, and yields `moved_lines`/`moved_notes`, which now
  mean copied counts. Update its comments, which talk about moving and cutting.
- Remove the source cut. Do not call `transplant::remove_line_ranges`, so the staged
  source notes keep their original bytes. The destination is then resolved against
  unchanged bytes, and the "source cuts may have shifted" re-resolution logic can
  simplify. Keep the `created` drawer/section flag behavior and keep
  `insertion_origins`, because capture's `MergeLineOrigin` tracking and the Mac origin
  folds use them.
- Replace the anchor-relocation pipeline (`rename_fragments` with
  `IdCollisionPolicy::Rename`, `source_id_maps`, `moved_targets`, and
  `repair_vault_postimages` over moved targets) with a copy-sanitize step for each built
  payload:
  1. Strip live block anchors (rule 2). Find them with the existing occurrence scanner
     (`transplant::fragment_occurrences` / `live_block_ids`), which already skips hidden
     and fenced lines. Confirm that before relying on it.
  2. Strip `[id:: …]` from copied task lines (rule 2). Prefer an existing field helper,
     for example `task_status_hooks::reconcile::fields::set_task_fields` or the
     dependency-field formatter, over a new regex.
  3. Run `transplant::repair_moved_fragment` with an empty `MovedBlockTargets`, an empty
     id map, and `stayed_source_ids` set to every live id in S's (unchanged) note. That
     rewrites same-note block links to name S's note (rule 3). Count those rewrites in
     the destination's `repaired_files` entry so the `↻ repaired N link(s)` row stays
     honest. With no rewrites, report no repair row.
- `PreparedMerge.postimages` now holds only the destination postimage. Source notes
  change only through retirement. `renamed_block_ids` is always empty. Keep the field if
  removing it would ripple; otherwise delete the dead path.
- Check `retirement::apply_retirement` and the plan's offer/prune staging against the
  new staged bytes. The key case is a same-note merge where T sits **above** S, so the
  payload insertion shifts S's line. S must be re-resolved by block ID, not by its saved
  line.
- Re-check every `prepare_task_merge` / `plan_task_merge` caller for assumptions that
  source lines disappear: `capture/transfer.rs`, `capture/merge_in.rs`,
  `capture/relations_merge.rs`, `capture/task_blocks.rs` (merge-origin forwarding),
  `capture/pull_in_copy.rs`, `capture/output.rs`, and
  `capture_complete/transfer_assist.rs`. Capture's destination-before-source staging
  must still produce one consistent postimage when S and T share a note.

### Wording (Rust)

- `plan.rs` record message and `command.rs` human source row: `N notes moved` →
  `N notes copied` (rule 7).
- `command.rs` `bob task merge` `long_about`: replace "the destination owns the moved
  notes" with wording like "Sources retire as cancelled tombstones with a merged-into
  link and keep their notes; the destination receives a copy." Regenerate
  `docs/reference/cli/task.md` with `just docs-gen`.
- Grep `src/` for any other merge-specific "move/moved/cut" copy that a user sees, and
  update it. Leave `task_move` and Pomodoro-merge wording alone.
- `capture_complete/transfer_assist.rs` (`⤵ merge in · N notes`) is already neutral.
  Rename local identifiers only if that helps clarity.

### Docs (bob-cli)

- `docs/task-merge.md`:
  - §1 rule and the move/merge/split vocabulary paragraph;
  - the §2 fate table: S's authored body is copied, S keeps the originals, copies carry
    no anchors or `[id::]`, and same-note links in the copy name S's note;
  - §4 payload shape: relative-indent wording is unchanged;
  - §9 human sample (`2 notes copied`) and a note that `moved_lines` and `moved_notes`
    count copied lines and notes;
  - §10 canonical copy (rule 7);
  - §11 property 1, now worded for merges: every block link resolves to the same block
    as before;
  - the §12 MG2, MG3, and MG4 rows and the test-name table, matching the renamed tests;
  - the §13 MI4 row (same-note copy, no collisions).
- `docs/task-relations.md`: §2 (line ~113, "moves the source's notes"), the §21 intro,
  the ML2 YAML comment ("S's notes move under T"), ML4 step 1 ("cuts each source's
  notes"), the ML5 `merged` effect, and the ML7 consequence strip.
- `docs/capture.md` ("when no notes would move" → copied) and any `>`/`<` merge prose in
  `docs/reference/capture-syntax.md` that says notes move.
- `docs/reference/mac-capture/capture-transfers.md`: merged sources no longer lose
  lines. Their landing card shows the cancelled headline and the new LINKS line, and the
  source row shows the copied-note count. Keep the removed-run fold sentence only for
  move cards and for older Bob responses.
- Run `just docs-gen` and `just docs-check`.

### Memory

Apply only what the accepted memory decisions allow (use `/sase_memory_write`). Then run
`sase memory init`, and `just docs-gen` so that `docs/reference/glossary.md`
regenerates.

> [!decision] glossary_merge_terms Edit the `glossary:task-merge` strand. A merge copies
> the source's notes to the destination and retires the source as a cancelled tombstone
> that keeps them. Drop "rather than copying and deleting it". Edit the
> `glossary:task-relation` strand, which says a merge label "merges it into the target
> (moving its notes". It should say that merge labels merge the source into the target
> by copying its notes and cancelling it.

> [!decision] decision_record Add the `decisions:merge-copies-source-notes` strand with
> the decisions-web shape:
>
> - **Applies to:** the bob-cli merge engine and every merge gesture.
> - **Claim.**
> - **Why:** Bryan's 2026-11-09 request; the tombstone keeps the history, and links into
>   it keep resolving.
> - **Cost:** duplicated note text, and closed nested tasks appear twice.
> - **Reopens when.**

### Tests

Update the byte-exact expectations in every suite that asserts cut semantics:
`src/native/task_merge/tests.rs`, `tests/cli/task_merge.rs`,
`tests/cli/capture/{merge_in,transfer,relations_merge,complete_editor}.rs`,
`src/native/task_relations/tests.rs`, and any merge vectors under `tests/fixtures/`. Add
or rename tests so that these hold:

- **Source intact (all MG/MI fixtures):** S's postimage equals its preimage except for
  the task-line cancel edit, the LINKS line, and the Cancel Log entry. Add a shared
  assertion helper.
- **MG2:** nested drawers are copied without flattening, and the originals stay on S.
- **MG3:** a daily-note link to an anchor under S is byte-unchanged and resolves to S's
  note. The copy has no `^` anchors, and there is no collision rename, including when S
  and T share a note (MI4).
- **Intra-copy links:** `[[#^x]]` inside the copied body becomes `[[S-note#^x]]`, links
  in fences and inline code are untouched, and the `↻` count matches.
- **Closed nested task:** a closed nested task carrying `^id`, `[id::]`, and a
  Depends-On line is copied without `^id`/`[id::]`, and `bob task reconcile` is a no-op
  afterwards (extend MG17). If reconcile still rewrites the copy, sanitize whatever
  derived field it touches in the copier. Do not special-case reconcile.
- **Unchanged guarantees:** MI3 byte equivalence (`<` vs two-item `>`), dry run equals
  commit, a re-run is `unchanged` with zero writes, offers (MG10–MG12), dependents
  (MG5–MG9), and the JSON wire shape test (field names unchanged).

### Verification and commit

- Run `just fmt`, `just lint`, `just test` (or `just check`), and `just docs-check`. All
  must pass.
- A scratch-vault smoke test with the release binary (`bob task merge -d` on a temp
  vault fixture, never Bryan's vault) shows `N notes copied` and leaves S's notes in
  place.
- Commit subject: `feat(task-merge)!: copy merged source notes instead of moving them`.
  The documented vault effect of every merge gesture changes. Never edit the
  `Cargo.toml` version or `CHANGELOG.md`.

## Phase `nav-copy`: Links-stage strip says "copies" (bob-plugins)

Open the repo with `/sase_repo` (`bob-plugins`) and follow its `AGENTS.md`.

- `plugins/bob-navigation-hotkeys/src/305-relation-stage-copy.js`: in
  `formatTaskMergeConsequenceStrip`, change `moves 1 note` / `moves N notes` to
  `copies 1 note` / `copies N notes` (rule 7) and update the comment that quotes the
  canonical copy. It keeps decoding `record.moved_notes` (rule 6). Renaming the internal
  `movedNotes` summary field to `copiedNotes` is optional.
- Grep the nav plugin and the bob-ledger-tools `api.taskMerge` code for other merge text
  that says notes move (preview captions, Undo-merge notice), and align it with rule 7.
- Tests: `scripts/test-navigation-relation-merge.cjs` (expected strip) and
  `scripts/test-navigation-relation-links-stage.cjs` if it asserts the strip.
- Docs: `docs/bob-navigation-hotkeys.md` (the strip sample near line 588).
- Run `npm run build` (never hand-edit generated `main.js`), `npm test`, and
  `npm run docs:check`, then `bob plugins sync` to deploy to the vault.

## Phase `mac-copy`: transfer card says "copied" (bob-mac-capture)

Open the repo with `/sase_repo` (`bob-mac-capture`) and follow its `AGENTS.md`. It
requires compiling every Swift change and watching CI after each push.

- `Sources/CaptureCore/CaptureTransferPresentation.swift` (~line 433): change the merge
  source `detailText` from `N notes move` / `1 note moves` to `N notes copied` /
  `1 note copied`. Keep `title only` (rule 7). It keeps decoding `moved_notes` (rule 6).
- Leave the removed-run `.movedNotes` fold (`⤵ N lines moved to the target`) in
  `CaptureTaskBlockPresentation.swift`. It stays correct for move cards and for older
  Bob responses, and merges from the new Bob simply have no removed source run. The
  merge-origin fold (`⤵ T · N notes ▸`) and `CaptureMergeInPresentation` (`N notes`) are
  already neutral.
- Update the tests and VoiceOver expectations that assert the old detail text. Fixture
  JSON keeps `moved_notes`.
- Update the repo's `docs/` if it quotes the old text (`just docs-check`).
- Run `just portable-check` (on athena if `swift` is missing locally) and
  `Scripts/macos-ci-check` before landing. After the push, watch the exact-SHA macOS run
  to green with `/sase_monitor`. If the Mac is offline for the type-check, say that the
  macOS targets are unverified.

## Out of scope

- Task **move** (`bob task move`, `>` to a bare note, Ctrl+Shift+M) and **split**: they
  still relocate blocks with link repair.
- Pomodoro merges (`=*`).
- Rewriting existing tombstones in the vault. Past merges stay as they are.
