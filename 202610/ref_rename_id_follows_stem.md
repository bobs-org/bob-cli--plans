---
tier: tale
title: bob ref rename keeps a reference's id equal to its file stem
goal:
  A renamed reference records its new stem as its id in the note and PDF marker
  (suffixed only to avoid a duplicate), a same-stem rename repairs mismatched ids, and
  the live vault's existing mismatches are audited and repaired.
size: medium
decisions:
  idless:
    ask: Should a rename add an id to a reference that has none?
    choices:
      keep:
        Id-less references stay id-less; renaming them leaves the PDF bytes untouched
      add:
        Renames write id = new stem into the note and PDF marker, so every renamed ref
        gains an id
    default: keep
    why: 77 PDF-backed refs were created without --include-id on purpose
    answer: keep
  vault_repair:
    ask:
      Should the coder re-key the live vault's mismatched references once the fix passes
      its tests?
    default: true
    why:
      The eight live violations are the visible symptom; each repair is one revertable
      commit
    answer: true
  decisions_memory:
    ask:
      Should a new decisions record replace the rename-keeps-id claim, marking the old
      record superseded in part?
    memory:
      - decisions:reference-renames-keep-identity
      - decisions:reference-ids-follow-file-stems
    default: true
    corrects:
      It keeps the reference's `id`, `^ref-…` block ID, title, `source_url`, dates, PDF
      marker, and PDF bytes.
    answer: true
proposed_by: bbugyi200.athena.10v
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-10-17 06:41:29
status: wip
---

# Plan: Make `bob ref rename` Keep A Reference's Id Equal To Its File Stem

## Background

Bryan's rule: **a reference's id, which is recorded in the PDF's page-1 marker and in
the note's frontmatter `id`, must equal the reference note's basename** (`foo` for
`~/bob/ref/chat/foo.md`). The only allowed exception is avoiding a duplicate id. Ids
must be distinct across the whole `ref/` directory.

`bob ref rename` breaks this rule on purpose today. Epic `bob-cli-7x`
(`plan:202610/bob_ref_rename.md`) and the decision record
`decisions:reference-renames-keep-identity` made a rename keep `id`, the PDF marker, and
the PDF bytes. As a result:

- `src/native/highlights_ref/rename/plan.rs` copies `row.marker_id` into
  `kept.marker_id`. `verify_renamed` then _fails_ if the id changed ("reference id
  changed during rename").
- `src/native/highlights_ref/rename/cli.rs` after-help, `docs/ref.md`
  (`## bob ref rename` and the task table near the top), and the tests in
  `tests/cli/ref_library/rename.rs` all promise that the id stays, and that
  `bob ref find <old id>` keeps resolving.

On 2026-10-17 the live vault had these violations: `ref/blogs/karpathy_llm_wiki.md` (id
`442a6bf555914893e9891c11519de94f`, left behind by the motivating rename);
`ref/papers/BryanBugyi_CV_extended_v{2,3,4,5}.md` (all four have id
`BryanBugyi_CV_extended`, which duplicates `ref/papers/BryanBugyi_CV_extended.md`);
`ref/chat/obsidian_sync_footprint_audit_bob-cli-1l.md` (id
`obsidian_sync_footprint_audit_bob-cli-1l.1_20260827`);
`ref/chat/sase_task_bead_48h_impact_rating.md` (id `task_bead_48h_impact_ranking`, which
duplicates `ref/chat/task_bead_48h_impact_ranking.md`); and
`ref/blogs/harness_engineering.md` and `ref/papers/harness_engineering.md`, which share
both a stem and the id `harness_engineering`.

### Where the id lives

- Note frontmatter `id:`, mirrored from the PDF's page-1 marker (`- id: …`). The marker
  is `FIELD_ID` in `src/native/highlights_ref/mod.rs` and is part of
  `COMMON_USER_FIELDS`, so the sync engine merges it three ways.
- The sync bookkeeping fields `highlights_marker_base` (JSON that contains `"id"`),
  `highlights_marker_hash`, and `source_pdf_sha256`. The sha changes whenever the marker
  is rewritten.
- The Highlights app's sidecar export (`lib/<t>/<stem>.md`) mirrors the marker text. Bob
  never writes that file, and a stale id there is harmless:
  `region.rs::is_marker_mirror_text` needs only `status` and `parent`.

### What the id is used for

The id is used only for lookup and display: `ref_library/resolve.rs::resolve_name` (stem
or id), `find.rs` `MatchKind::Id`, and `marker_id` in the `ref list/show/today` JSON.
URL deduplication keys come from `source_url`/`url` (`ref_library/identity.rs`,
`clip_url::dedupe_key_for`), never from `id`. Re-keying the id therefore breaks no
identity join. The old decision record's claim that "the `id` is the URL deduplication
key" does not match the code.

The reading task's block ID (`^ref-<slug>`) is a separate value.
`ref_tasks::line::allocate_ref_block_id` minted it from the stem at birth. Today, Task
Links, Depends-On, and `[id:: …]` join on it. **The block ID stays unchanged** (see
Non-goals).

## Target Behavior

1. **Id allocation.** The planned id is the new stem. If another reference already
   records that id, the planned id is the first free `<stem>_2`, `<stem>_3`, …, the same
   suffix shape `bob ref create` uses for stems. The rules:
   - "Another reference" means every other row of the reference index (`build_index`
     over the whole configured ref dir, legacy rows included). The reference being
     renamed is excluded.
   - Compare case-insensitively, because `resolve_name` matches ids with
     `eq_ignore_ascii_case`.
   - Check the bare stem only against other rows' **ids**. The rename's existing
     `name_taken` check already guarantees no other `<stem>.md` exists. Checking the
     bare stem against other stems would wrongly bump one of two same-stem refs in
     different type directories.
   - Check each suffixed candidate against other rows' ids **and** stems, so a suffixed
     id never makes `bob ref find` ambiguous.
   - The rule is stable: a reference whose id is already its correct allocation is left
     unchanged. For example, `foo_2` stays `foo_2` while `foo` is still taken.

   Put this in one small pure function (for example
   `allocate_reference_id(stem, rows, self_index)`) with unit tests.

2. **Full rename (stem changes).** A reference with an id ends with `id == planned id`
   in both the note frontmatter and the PDF marker. The commit subject stays
   `bob ref rename: <old> → <new>`, so `already_renamed` detection is unchanged. The v1
   scope rules stay: annotated, legacy, and intake references are refused, as are
   out-of-sync notes.
3. **Same-stem repair.** `bob ref rename <ref> <its current stem>` re-keys a reference
   whose id differs from its planned id. It is an id-only change:
   - No moves and no inbound-link rewrites.
   - `outcome: "id_updated"`, `changed: true`.
   - Commit subject `bob ref rename: <stem> (id <old> → <new>)`.

   An id-only change touches neither the PDF path nor the note path. Highlight `^h-` ids
   hash the PDF path (`sidecar_render.rs`) and follow-up `[h::]` digests hash the note
   path (`annotation_tasks.rs`), so neither changes. The id-only path therefore **skips
   the `has_highlights` and `listen_card_link` refusals**. It keeps the
   `legacy_reference`, `intake_reference`, `not_in_sync`, and `dirty_paths` refusals.
   When the id is already correct, the result stays `outcome: "unchanged"` with no
   commit.

4. **Id-less references.**

   > [!decision] idless = keep A reference with no `id` stays id-less. A rename moves
   > its files exactly as today: the PDF is byte-identical and is not marker-rewritten,
   > and the report's `id` object is `{from: null, to: null, changed: false}`. The
   > same-stem path returns `unchanged`.

   > [!decision] idless = add A reference with no `id` gets `id: <planned id>` through
   > the same note-edit plus marker-write path, both on a full rename and on a same-stem
   > repair. The planned id follows the allocation rule above. Insert the `id:` line
   > directly after `title:` when one exists; otherwise insert it as the first
   > frontmatter field.

5. After an id change, `bob ref find <old id>` no longer resolves; the new id and the
   new stem do. Say so in the docs and the after-help.

## Implementation

All code changes live in `src/native/highlights_ref/rename/` (`plan.rs`, `report.rs`,
`cli.rs`). Reuse the sync engine; do not re-implement marker hashing.

### Planning (`prepare`)

- Compute the id plan for every outcome: `from = row.marker_id`, `to = allocated id` (or
  `None` per the `idless` decision), and `changed = from != to`.
- If the stem changes, add a frontmatter rewrite of the note's top-level `id:` line. It
  sets the value to `render_frontmatter_string(<new id>)` (`highlights_ref/io.rs`), so a
  numeric-looking stem such as `2024` is quoted. This is a whole-value line replacement.
  It is not the substring replacement that `rewrite_own_frontmatter` does for path
  fields; either extend that function with a dedicated `id` rule or add a sibling
  helper. Merge the result into the same note rewrite that already carries `source_pdf`,
  `highlights_sidecar`, and `audio`, and add `"id"` to the report's `frontmatter` list.
  Never touch `source_url`, `title`, or the `^ref` embed.
- If `name == old_stem`: return the existing `unchanged_plan` when `!id.changed`.
  Otherwise build an id-only plan:
  - no moves and no link plan;
  - the frontmatter `id` rewrite;
  - `touched_paths` = the note and the PDF;
  - the same in-sync and dirty-path checks as a full rename.

  Do not call `build_bundle`, the highlight refusal, or `pdf_has_listen_card_link` on
  this path.

- Keep every existing refusal and its order for full renames.

### Live write: propagate the id to the PDF marker

The planned frontmatter edit is a note-side change to a synced marker field. The sync
engine already turns that into a marker write plus refreshed `highlights_marker_base`,
`highlights_marker_hash`, `source_pdf_sha256`, and `highlights_synced_at`. This is the
documented `bob ref sync -w` path. So, when `id.changed`, the live writer does this
inside the transaction after the moves and Markdown rewrites, and before verification:

1. Snapshot the PDF's bytes as they are before the marker write (in memory is fine).
2. Build `ScanContext::build(config)`. Call
   `plan_pdf_sync(config, <new pdf>, SyncOptions { dry_run: false, write_pdf: true, prefer: None }, &ctx)`,
   then run `finalize_annotation_task_plans` and `reserve_preview_ids`, as
   `sync.rs::sync_pdf` does.
3. **Guard the side effects.** Fail (with rollback) unless all of these hold:
   - `marker_write_needed` is true;
   - the reading-task plan is `ReadingTaskAction::NoWrite`;
   - `annotation_tasks_created == 0`;
   - `routed_intents` is empty;
   - every entry in `routed_task_note_writes` has action `"none"`;
   - `image_assets` is empty.

   The rename commits only `touched_paths`, so any other write would be left out of the
   commit.

4. Call `execute_pdf_sync(config, &plan, &mut RunWriteState::default())` directly. **Do
   not call `ensure_safe_to_write`**: it vetoes dirty targets, and the rename's own
   moves and rewrites are deliberately uncommitted at this point. Rename has already run
   its own dirty-path checks under the locks.
5. Record the note's post-sync bytes as its after-image in the transaction. Otherwise
   `restore_before_images` treats the sync's rewrite as a concurrent change and leaves
   it in place on rollback.

Restructure `apply_transaction`/`apply_and_verify` as needed, for example with a
post-write step that receives `&mut AppliedRename`. Keep it unit-testable with an
injected failure.

### Rollback

`rollback_applied` must also restore the PDF's original bytes. Moving the files back
takes the marker-rewritten PDF to its old path. Restore the snapshot atomically
(`atomic_write` or the existing PDF temp-and-rename pattern), and report a rollback
issue rather than clobbering if the bytes changed concurrently. The note is covered by
its updated after-image.

### Verification (`verify_renamed`)

- Expect `row.marker_id == planned id` (the old id when unchanged) instead of "unchanged
  id".
- Also read the new PDF's marker (`read_pdf_marker` + `parse_marker_with_normalization`)
  and require its `id` to equal the planned id. Require no `id` when the plan is
  id-less.
- Keep the other checks: dry-run sync plans no work, the old row is gone, the task has
  the same `(path, block_id)`, there is no `duplicate_identity` or
  `open_ref_without_task`, and no new orphan.
- For the id-only path, "new row" and "old row" are the same path. Skip the old-row-gone
  check there.

### Report and JSON

- Add a top-level `id` object to `RenameEnvelope`:
  `{"from": <string|null>, "to": <string|null>, "changed": <bool>}`. Fill it in every
  outcome: renamed, id_updated, unchanged, and already_renamed (where
  `from == to == the row's current id`).
- Remove `kept.marker_id`; `kept` keeps only `task`. Bob Mac Capture's
  `RefsRenameResponse` (`Sources/RefsCore/RefsRename.swift` in `bob-mac-capture`) does
  not decode `kept`, and bob-plugins never parse rename JSON, so neither client changes.
  Unknown `outcome` values fall through to the Mac card's generic renamed status. Keep
  `schema_version` 1.
- Human output (`report.rs`): when the id changes, print
  `  id: <old> → <new> (note and PDF marker)` and `  keeps: task <path>#^<block>`.
  Otherwise keep today's `  keeps: id <id> · task …` line. For `id_updated`, print
  `  outcome: id_updated` and still show the commit and sync lines. Keep the existing
  colors and style helpers.

### Help text and docs

- `cli.rs` after-help: replace the "frontmatter id … stay" sentence. The new text says:
  - the reference id (frontmatter and PDF marker) follows the new stem, suffixed `_2`,
    `_3`, … only when another reference already records that id;
  - renaming to the current stem repairs a mismatched id, even on annotated references;
  - the reading task's `^ref` block ID stays, so Today, Task Links, and dependencies
    keep working, but `bob ref find <old id>` stops resolving;
  - v1 refusals and the out-of-vault-link caveat stay.

  Keep both examples, and keep the options list unchanged and alphabetical.

- `docs/ref.md`:
  - Update the task-table row ("keeps the id and reading task").
  - Update the `## bob ref rename` scope paragraph and add the id rule, the same-stem
    repair, `outcome: "id_updated"`, and the `id` JSON object (drop `kept.marker_id`).
  - Note that an id change rewrites the PDF marker, so the PDF should be closed in
    Highlights first.
  - In "Live write flow", describe the in-transaction `bob ref sync -w` step and the PDF
    byte rollback.
  - Leave the dated 2026-10-15 acceptance paragraph as history.

### Tests

Update `tests/cli/ref_library/rename.rs`, and add unit tests in `plan.rs`:

- **Gist dry-run.** Assert `id.from == "442a6bf…"`, `id.to == "karpathy_llm_wiki"`,
  `id.changed == true`, `frontmatter == ["id", "source_pdf"]`, and that `kept` has no
  `marker_id` while `kept.task` is unchanged. Vault bytes stay unchanged.
- **Offline gist write.** Assert:
  - the note frontmatter `id: karpathy_llm_wiki`;
  - the PDF marker has `- id: karpathy_llm_wiki`;
  - `source_url` is unchanged and the `^ref` block ID is unchanged;
  - a dry-run `ref sync` reports `note_action: none` and `pdf_marker_action: none`;
  - the commit records the PDF as a rename (relax the `R100` assertion to any `R` score
    for that path pair);
  - `bob ref show karpathy_llm_wiki` resolves;
  - the old-stem rerun still reports `already_renamed`, and the same-stem run reports
    `unchanged`.

  Replace the "find old stable id" assertion.

- **Collision.** Use a fixture ref with stem `other` and id `target`, then rename a
  second ref to `target`; its id becomes `target_2`. Unit-test the allocator for:
  - case-insensitive matches;
  - suffixed candidates that skip another row's stem;
  - same-stem refs in two type directories, where only the second repaired one gets
    `_2`;
  - stability (`foo_2` stays `foo_2`).
- **Same-stem repair.** Use an in-sync ref with stem `foo` and id `stable-id`.
  `rename foo foo -o` reports `outcome: "id_updated"` with moves `[]` and the commit
  subject `bob ref rename: foo (id stable-id → foo)`. The marker and frontmatter are
  updated and the sync is clean; a rerun reports `unchanged`.
- **Annotated same-stem repair.** Build an in-sync annotated reference by syncing a
  fixture with a Highlights Markdown sidecar, as the existing highlights sync tests in
  `tests/cli/` do. The id-only repair succeeds and keeps the `^h-` blocks
  byte-identical. A full rename of the same reference is still refused with
  `has_highlights`.
- **Id-less ref.** Follow the `idless` decision. With `keep`: the rename succeeds, the
  PDF bytes are identical (`R100`), and `id` is `{null, null, false}`. With `add`: the
  id equals the new stem in both places.
- **Rollback.** Extend `forced_verify_failure_reverses_moves_and_restores_rewrites` (or
  add a sibling). Force a failure after a marker-rewriting step and assert that the
  original PDF bytes and the note bytes are restored at the old paths.
- **Help.** If the help test is extended, assert the new after-help mentions the id
  following the stem.

Run `just fmt`, `just lint`, and `cargo test --test cli ref_library` while iterating,
then `just check` (or `just all`) before finishing.

### Memory

> [!decision] decisions_memory Write the new record and mark the old one superseded in
> part, as below.

Before editing memory, use `/sase_memory_write`, and stay inside the `decisions_memory`
decision above.

- Add a new decisions strand, `decisions:reference-ids-follow-file-stems` (slug
  `reference-ids-follow-file-stems`, keyword "Reference Ids Follow File Stems"). Mirror
  the frontmatter shape of recent strands: keyword, aliases, a summary that is a rule,
  `metadata.status`. The body:
  - **Applies to:** bob-cli, Bob Mac Capture's rename card, and the Bob vault.
  - **Claim:** a reference's recorded id equals its note stem. The only exception is a
    `_N` suffix that avoids a duplicate across `ref/`. `bob ref rename` re-keys the id
    with the stem, and a same-stem rename repairs a mismatch, while the `^ref-…` block
    ID, title, `source_url`, and dates stay.
  - **Why:** Bryan's rule, quoted from this request. The id is only a lookup key (URL
    dedupe uses `source_url`), so re-keying it breaks no join. The sync engine already
    propagates a note-side id edit to the marker.
  - **Rejected alternatives:**
    - keeping the stale id (the previous record);
    - re-keying the `^ref` block ID too, which would churn Task Links, `[id::]`, and
      Depends-On;
    - refusing on an id collision instead of suffixing.
  - **Cost:** the PDF bytes change on an id change; `bob ref find <old id>` stops
    resolving; a PDF open in Highlights could later write back a stale marker.
  - **Reopens when:** ids gain another consumer, or the block ID must follow the stem.
  - **Evidence:** this plan, `docs/ref.md`, and `tests/cli/ref_library/rename.rs`.
- Mark `sase/memory/decisions/reference-renames-keep-identity.md`
  `metadata.status: superseded-in-part` with
  `superseded_by: decisions:reference-ids-follow-file-stems`. Add one back-link
  sentence: "Superseded in part: the claim that a rename keeps the reference `id`, PDF
  marker, and PDF bytes is replaced by [[decisions:reference-ids-follow-file-stems]];
  the `^ref-…` block ID, title, `source_url`, and dates still stay." Do not otherwise
  edit its accepted body.
- Run `sase memory init` to regenerate the roster and `AGENTS.md`.

### Live vault repair

> [!decision] vault_repair After the code passes `just check`, build
> `cargo build --release` and use `target/release/bob`; do not install it. First re-run
> the audit below against `~/bob`, because the vault may have changed since 2026-10-17.
> Then, for each reported reference, run
> `target/release/bob ref rename <note path> <its stem> -d`, review the planned `id`,
> and run it again without `-d`. Use normal mode, with vault-sync before and after, so
> each repair is one commit.
>
> For the `harness_engineering` pair, repair only the reference whose `created:` is
> later. It becomes `harness_engineering_2`, and the older one keeps
> `harness_engineering`.
>
> If a repair refuses (for example `not_in_sync` or `dirty_paths`), record the reference
> and the refusal in the final report rather than forcing it. Re-run the audit at the
> end and report what remains.

> [!decision] vault_repair = no Make no live-vault changes. In the final report, list
> the audit command and the per-reference repair commands for Bryan to run.

Audit (one line per PDF-backed reference whose id differs from its stem, plus duplicate
ids):

```bash
cd ~/bob && for f in $(find ref -name '*.md' -not -path '*/.*'); do
  id=$(awk '/^---$/{n++; next} n==1 && /^id:/{sub(/^id: */,""); gsub(/"/,""); print; exit}' "$f")
  [ -n "$id" ] && [ "$id" != "$(basename "$f" .md)" ] && echo "$f id=$id"
done; grep -h '^id:' $(find ref -name '*.md' -not -path '*/.*') | sort | uniq -d
```

Expected after the repair: no mismatch lines except an intentional `_N` id, and no
duplicate ids.

### Follow-up bead

`bob ref create` can still mint ids that break the rule. With `--output`, the id stays
the base stem while the file name differs. The default-target collision walk also
appears to be per type directory, which is how `harness_engineering` was duplicated
across `blogs` and `papers`. Use `/sase_new_task` to file one `bug` task bead (size
`large`) describing both paths. Optionally the bead can also cover a `bob ref doctor`
check for id/stem mismatches and duplicate ids. Do not fix these in this tale.

## Non-goals

- Re-keying the reading task's `^ref-<slug>` block ID, `[id:: …]`, Task Links, or
  Depends-On links. They join on the block ID, which stays.
- Full renames of annotated references, refiling, retitling, bulk renames, and intake
  renames. The v1 scope is unchanged, except that the id-only repair is allowed on
  annotated references.
- Changing `bob ref create`, scan, or sync id minting (see the follow-up bead).
- Changing Bob Mac Capture or bob-plugins. Their decoders and copy remain correct.

## Acceptance

- `bob ref rename <ref> <new>` leaves `id == <new>` (or `<new>_N` when another reference
  already records `<new>`) in both the note and the PDF marker. The sync is clean
  afterward, and the reading task's block ID and links are untouched.
- `bob ref rename <ref> <current stem>` repairs a mismatched id (`id_updated`),
  including on annotated references, and is a no-op when the id is already correct.
- A failure at any point after the moves restores the files, the Markdown, and the
  original PDF bytes.
- The help text, `docs/ref.md`, the tests, and the decisions memory all describe the new
  rule. `just check` passes.
- Per `vault_repair`: either the live mismatches are repaired and audited, or the repair
  commands are reported.
