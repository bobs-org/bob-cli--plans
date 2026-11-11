---
tier: tale
size: medium
title: Finish album index write safety and preview parity
goal:
  Repair the remaining album-index defects and complete the landing of bob-cli-k3.5.1
  and parent phase bob-cli-k3.5.
proposed_by: bbugyi200.athena.bob-cli-k3.5.1.land
proposed_with: codex/gpt-6-astra
bead: bob-cli-k3.5.1
status: done
---

- **PARENT:**
  [202611/album_records_index.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/album_records_index.md)
- **BEAD:**
  [bob-cli-k3.5.1](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-k3/bob-cli-k3.5.1.md)
- **AGENTS:**
  - [bbugyi200.athena.bob-cli-k3.5.1.land](https://github.com/bobs-org/bob-cli--agents/blob/main/sessions/bbugyi200.athena.bob-cli-k3.5.1.land.md)
- **COMMITS:**
  - [300a6e8](https://github.com/bobs-org/bob-cli/commit/300a6e81005263311408fac024038a12a50fd62d)
    — feat(albums): finish index write safety and preview parity

# Finish the album index landing

Implement this tale directly, then perform the concrete closeout below in the same turn
as the final code. There is no separate tale land agent. Do not wait for this work's own
commit SHA, push, or CI before closing the completed epic.

## Scope and inherited decisions

This is only the remaining work from the landing audit of `bob-cli-k3.5.1`, whose
approved design is `plan:202611/album_records_index.md`. Its parent is phase
`bob-cli-k3.5` in `plan:202611/album_and_record_notes.md`, under epic `bob-cli-k3`. Read
those artifacts with `sase artifact read`, and read the current epic and parent phase
with `sase bead read` before changing code. The current epic's three phases are closed;
the lander reviewed every note and their implementation commits: `1f4c9457`
(render/report), `b79ea376` (record writes), and `6352824b` (sync/nightly).

The audited source baseline is `bc3983d3`, also `origin/master` after a fetch. Preserve
the shipped shared renderer, route-local capture lookup, one folder scan per batch,
notebook region bytes, JSON schema version 1, and guarded writer. This tale adds no
commands, owner catalogs, plugins, config retirement, migrations, or memory changes.
Parent decisions concerning ledger mining and glossary work belong to later phases; the
proposed albums decision memory was explicitly declined.

The lander's executable reproducer is archived as
`file:explicit:061f2ab2ef1ce5bfc4660e3b`. Read it with `sase artifact read` if useful;
replace its machine-specific binary locator with the current checkout's Cargo target
when running it. Its scratch data is synthetic. It confirmed success exits while
skipping a damaged-folder index, success while indexing unreadable members, an
intermediate batch preimage/state, a phantom net-zero album edit, and raw newline
injection from both a plural and a choice label. The normal temporary-vault walkthrough
passed; the edge cases are the remaining work. Its captured edge-case results are
`file:explicit:5353c7f0e5bab132ebba1cc7`. The successful walkthrough transcript is
`file:explicit:653de3c666f7c5264ea02ada`. The baseline `just check` passed in tool run
`6a224703ed20d7ef538696dabc0c3928` (including all seven historical vault-sync failures);
these defects escaped the existing tests.

## 1. Refuse an unsafe index plan before writing a record

`src/native/albums/write.rs::plan_album_write` drops the index planner's findings and
passes only `folder.diagnostics().is_empty()` as its completeness check. It misses
`NoteRead` diagnostics on individual catalog members. Both
`capture/record_note.rs::stage_album_index` and
`records/cli.rs::plan_album_write_for_record` then accept `IndexState::Damaged` with no
edit. Thus an unsafe index can be silently skipped while the record is written, or an
unreadable member can be rendered as if it were a readable malformed record.
`albums/report.rs::project_album_from_folder` already distinguishes these cases.

Carry enough diagnostic information through the shared album-write planner for both
callers to refuse a damaged plan with a useful path and reason, before any directory
creation or real write. Share the folder/member completeness rule with the report path
rather than allowing two rules to drift. Check the final overlay: readable invalid YAML
remains a counted member with its fallback title/status; actual I/O or invalid UTF-8
reading failures must not become success. Preserve config-backed behavior and later-item
batch atomicity. Do not invoke public sync from a record writer.

Add focused regressions for capture and records new/update, including dry-run:

- A direct `.md` occupant that is a directory makes the album slice damaged. A
  create/update of another record must write neither record nor index and report that
  affected path, rather than silently omitting the index edit.
- A direct record with unreadable frontmatter (invalid UTF-8 is deterministic even under
  privileged test execution) prevents publishing an index from incomplete inputs. A
  readable malformed YAML member still appears and does not block a healthy record
  write.
- Put the failing record item after a valid item in a batch and assert the whole batch
  writes nothing. Preserve the existing ordinary-task laziness tests.

## 2. Derive batch album reports and guards from the final staged files

`capture_record.rs::record_album_edit` replaces the entire saved edit on every item. The
next item's `before` is the previous in-memory postimage, so `album_files[].before` and
its state describe an intermediate edit rather than the original disk-to-final change. A
batch that changes a grouping field and changes it back still reports an album edit even
though `into_staged_files` omits that unchanged album. Because `stage_album_index`
currently adds a read guard only when the per-item plan has no edit, an album whose
_final_ bytes revert to the original can also lose its required guard while a record
body still changes.

Use the batch planner's original/current bytes as the authority for the final
`album_files` collection. Keep one entry per actually changed album, the original disk
preimage, the final postimage, and the original missing/stale state. Drop net-zero
entries. Guard the original album preimage even when the final index is unchanged,
without adding reads or whole-vault discovery. Preserve typed `NoteChanged` failures,
existing rollback, and original-byte guards across all items. Avoid a second
independently maintained final-file model if the existing planner can supply this
information.

Honor the approved human dry-run contract as well: both capture's `print_album_files`
and records' `print_album_dry_run` currently print the entire album postimage. Show the
actual unified original-to-final album diff alongside the existing record output,
reusing the existing diff convention. Keep additive JSON before/after fields and
record-oriented item cards stable. Assert the human preview's changed lines and file
headers, not just the presence of a title.

Test create/create/update against an initially missing index: the reported before must
equal the exact original album, the state must remain missing, and the after must equal
the actual written bytes. Test changes that restore the original index while adding a
note to a record: no album edit is reported or written, but a concurrent album/schema
change after planning must still refuse the entire write. Also cover repeated updates,
an unchanged last item, two albums, and CRLF parity where existing tests can be extended
rather than duplicated.

## 3. Make the tally header inert and repeatable

`albums/index.rs::tally_text` / `tally_fragment` interpolate authored plural and
choice-label text directly into `RegionSpec.head_title`; the region engine writes that
value verbatim. Group headings and facets already use `notebooks::catalog::inert_text`,
but the tally does not. A valid YAML string with an embedded newline and `## Injected`
can therefore escape the owned callout, and Markdown/link syntax in the title remains
active.

Apply the shared inert-text handling at the renderer boundary to every authored piece of
the tally, including the empty-album plural. Preserve normal icon, count, choice order,
plural override, and acronym grammar. Do not change the generic region engine's
semantics for notebooks. Add byte-exact hostile plural and choice-label tests with
newlines, links, brackets, and Markdown markers; the generated header must stay one
physical line. Plan/install/replan must be a no-op on its second pass, and authored
surrounding text must remain unchanged.

## 4. Verify the combined feature and intervening changes

The lander inspected all non-epic commits from `1f4c9457^` through `bc3983d3`:
`bffbf36b`, `4771047e`, `8134d173`, `6e3d8837`, `03a0aba2`, `6a88d810`, `ee2d2cbf`, and
`bc3983d3`. The partial-registry refusal from `4771047e` is already honored by sync;
preserve it rather than publishing from incomplete discovery. The later vault
transaction rollback fixes coexist with the shared capture writer and host-catalog
rollback. Bank import, plugin deployment, task marks, and keep-moving changes require no
new album consumers. Command-catalog counts and nightly ordering already incorporate
albums sync. Review subsequent drift on the starting checkout and integrate any actual
conflict with these repairs.

Run the focused album renderer, shared writer, capture-record, records CLI, sync, and
nightly regressions. Run `just check`, never `just check-full`. Run `just docs-check`;
five known mounted-page word-budget errors at the audit baseline are unrelated and
tracked below. Do not hide newly introduced failures among those known failures. If a
long tool run escalates, join that exact run through `sase_monitor`; follow its
finalization instructions.

Finish the approved temporary-vault walkthrough: create an album fixture, capture two
records and update one in the same batch, exercise records new and update, compare the
rendered index with albums list/check, hand-edit the generated index, preview and
execute sync, and verify the next sync is a no-op. Verify the owner and unrelated notes
remain byte-identical and sync commits only its changed album paths. Use isolated vault,
config, clock, lock, cache, and Git fixtures; never write the real vault. Keep normal
scoped sync and nightly failure/continue behavior intact. Record concrete results on
`bob-cli-k3.5.1`.

## Follow-up triage already performed by the lander

Carry these outcomes into the close note; do not file duplicates:

- `bob-cli-k3.5.1.1` note 1, `.2` note 3, and `.3` note 1: the reproduced mounted Mac
  capture-panel/previews/keyboard and plugin navigation word-budget failures were
  corroborated on `bob-cli-jz`; protocol was corroborated on `bob-cli-h9`.
  `bob-cli-kj.2` is implementing those existing remediations. Baseline counts are 6265,
  6244, 6083, 6200, and 6294 respectively, against the 6000 cap.
- The same proposals' ledger-tools overage is no longer reported by docs-check. A
  resolution-evidence note was added to existing headroom task `bob-cli-il`; no
  stale-failure +1 or duplicate was filed.
- The capture-syntax overage in `.3` note 1 is fixed by `bc3983d3` (now 6000 words).
  JSON output docs are now 5965 words. Decline new tasks for these resolved reports.
- `.2` note 2, `items_after_test_module` in vault_git, is fixed by `6e3d8837`.
- `.2` note 1, seven vault-sync tests, has its git-pathspec root cause fixed by
  `4771047e`; consult the lander's recorded check result for current verification. Do
  not reopen it on historical evidence alone.

The three repair groups above are caused by this epic and remain epic work. They must be
finished before close, not filed as unrelated follow-ups.

## 5. Complete this landing, then close only its parent phase

This is the tale's final step, after the implementation and checks; it is not ordered
after a commit, push, SHA, or CI gate for this tale.

1. Re-read `bob-cli-k3.5.1` and all three children (`.1`, `.2`, `.3`) with reasons,
   review every note including this audit, and verify all descendants remain completed
   with resolution done. Re-read the linked index plan and confirm all original
   acceptance requirements and these repairs are satisfied.
2. Run `sase bead epic-symbols bob-cli-k3.5.1`. Resolve every listed symbol by wiring,
   privatizing, deleting, or the permitted non-test pragma; only re-key an exemption
   when a concrete still-open later bead genuinely needs it. None existed at the audit
   baseline. Do not leave any keyed to a closing bead.
3. Run
   `sase bead close bob-cli-k3.5.1 --note "<concrete source/commit review, integration, repaired defects, test/walkthrough evidence, and every follow-up outcome above>"`.
   Never force a successful nested landing. If normal close rejects unfinished work or
   symbols, finish those first and retry normally.
4. Run `just symvision` if that recipe is available (it is absent at this baseline), and
   record that result or absence. Open the plans repo using
   `sase repo open plans -r "Mark the verified album index epic plan done"`, use only
   its printed path, and set `status: done` in `202611/album_records_index.md`. Read
   plan context through audited artifact reads, not direct sidecar file reads. Include
   the modified plans repo in the final declaration.
5. Read `sase bead read bob-cli-k3.5.1 -r "Need the parent link"`. The concrete parent
   is phase `bob-cli-k3.5`. Verify this completed child plan satisfies that phase's
   renderer, atomic record writes, sync, and nightly scope; run
   `sase bead epic-symbols bob-cli-k3.5` and resolve/re-key any entries before
   `sase bead close bob-cli-k3.5 --note "<verified child completion and phase scope>"`.
   Close only this parent phase normally. Leave epic `bob-cli-k3`, its plan
   `202611/album_and_record_notes.md`, and later phase `bob-cli-k3.6` to their existing
   workers. If the parent scope has become incomplete or ambiguous, record the precise
   blocker on it and report that instead of forcing closure.
6. Record that `albums::sync::plan_indexes` / `IndexSyncPlan` is the shared transaction
   extension point for the owner-catalog phase. Finish through `sase_final`, declaring
   every modified repository for the host commit.
