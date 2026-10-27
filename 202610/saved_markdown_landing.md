---
tier: tale
size: medium
title: Repair saved-source verification and finish bob-cli-d2 landing
goal:
  Correct verified-source false negatives and catalog/JSON integration gaps, complete
  the two-week backfill, and close bob-cli-d2 with its plan marked done.
proposed_by: bbugyi200.athena.bob-cli-d2.land
proposed_with: codex/gpt-6-astra
bead: bob-cli-d2
create_time: 2026-10-27 09:24:49
status: wip
---

- **PARENT:**
  [202610/research_ref_saved_markdown.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/research_ref_saved_markdown.md)
- **BEAD:**
  [bob-cli-d2](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-d2/README.md)

# Finish the saved-Markdown epic

Complete the remaining work in **bob-cli-d2**, then close that epic in this coding turn.
This is a single-agent implementation tale with its own explicit closeout; there is no
later lander to finish it. Do not wait for this turn's eventual commit, push, SHA, or CI
run before closing the epic. Build, test, and perform the required operations from the
working tree first; the host commits the code after the turn.

## Context already verified

Read `sase bead read bob-cli-d2 -r "Need landing findings and final decisions"`, the
three children d2.1/d2.2/d2.3 and their notes, and these audited artifacts:

- `plan:202610/research_ref_saved_markdown.md`: original scope and operating procedure.
- `file:explicit:c61a26370b705cae1446a6ed`: lander's source review, reproducible
  failures, integration review, and completed follow-up triage.
- `file:explicit:a6b747946294df16b787c942`: backfill summary and all unresolved stems.
- `file:explicit:f20b1594f834db3006f4bb8c`: structured inventory and vault commit IDs.
- `file:explicit:825f29b45683e48216bdbe70`: original 72-candidate dry-run verdicts.
- `file:explicit:40c8bdcd27614b2afecc819f`: exact machine-load report whose hash starts
  66d9d991ad1e; useful for reproducing the verifier failure.

The lander reviewed the implementation at bob-cli d38bef59, including pointer-fix
3a13ec6f, adoption d38bef59, prior pointer safeguards 017b6927, candidate-selection
69b7d962, source-store/read surfaces, sync executors, intake and overwrite paths, help,
doctor, and tests. All 19 existing attach CLI tests pass with render tools. The only
non-epic bob-cli commit since the first epic commit was 79b2859d (CI helper and its
docs/tests); it requires no source-feature integration. Refresh origin and review any
newer overlapping commits before editing.

The approved `hook_timeout=raise` branch is already implemented: chezmoi 62806ca3 sets
300s on the existing `bob_research_capture` wrapper. Live file-hook lists on athena and
apollo both reported that command, `sidecars=[research]`, and `timeout_seconds=300.0`.
Preserve the newer wrapper from chezmoi 10e9f1d2; it adds stable naming and explicit
overwrite/reread selection. Do not restore the old direct `bob ref create` hook command.

The backfill repaired 2 orphan references (vault commit 57afc6b74) and attached 17
sources (85d7924b2). All 19 pointers were independently confirmed. Of 143 hook-window
entries, 87 now have pointers, 55 lack sources, and one remains an orphan:
`machine_load_accounting_transparency`. No references were missing. The 55/72 attachment
skip rate was 76.4%, exceeding the original 10% stop gate; the phase nevertheless wrote
the 17 independently verified candidates. Preserve those valid repairs and investigate
the systematic failure before further writes.

There is no parent_bead on bob-cli-d2. All original phases are closed. The lander's
follow-up triage is already recorded on d2, co, and the relevant batch-17 epic:

- d2.3 #2: the list of 17 repaired stems is supplementary evidence on **bob-cli-co**,
  not a new task. Membership in co's original 47 ambiguous/35 title-mismatch sets
  remains unproved; co stays open with its batch-17 owner.
- d2.3 #3: the 55 low-match skips and orphan remain this epic's acceptance work. The
  inherited matcher needs repair to complete this backfill; do not defer these known
  failures into a new task merely to close d2.
- **bob-cli-cy** is already closed with its named muse-commentary orphan repaired. Do
  not reopen it for the different machine-load orphan.
- The JSON-output defect below is also recorded on active epic
  **bob-cli-an.6.6.6.5.6.5.5.6.6.6.6.6.6.6.6.6**, whose phase `.2` introduced the JSON
  format. That note says this tale owns the shared helper repair.

No separate task beads were created. Include these outcomes in the final close note.

## Constraints

- Follow AGENTS.md and audited memory reads. Before changing the relevant behavior, read
  `decisions:reference-sources-are-preserved-write-once` and
  `decisions:code-copy-reads-render-time-catalog`. Read `docs.md` before docs/help
  changes, `cli_rules.md` if adding options (none are expected), and `obsidian.md` and
  `tailnet.md` before live operations.
- Use `/sase_repo` for every other repository and `/sase_memory_read` and
  `sase artifact read` for their canonical contexts. Obtain original artifact bytes
  through the audited reference/path; the pretty artifact reader strips frontmatter.
- Bob is the only live vault writer. Do not manually add pointers, alter source
  Markdown, regenerate old render-time catalogs, or rewrite existing library PDFs.
- Preserve the existing 12-character minimum, 85% per-unit similarity, 80% coverage,
  document ordering, title checks/explicit-pair semantics, and Mermaid exclusion. No
  digest-only verification shortcut, broad Unicode deletion, or threshold decrease.
- Keep saved source bytes exact and render-time code.json byte-identical on adoption.
- Do not change Cargo.toml's version or CHANGELOG.md.
- Verification is `just check`, plus the original plan's `just docs-check` gate. **Do
  not run `just check-full`.**

## 1. Correct the verifier's demonstrated false negatives

Work primarily in `src/native/ref_library/attach.rs` and its existing test modules.

`match_lines` currently requires each entire normalized Markdown line to occur exactly
in the concatenated PDF text, then awards that whole line's byte count. The promised 85%
unit similarity therefore behaves like an all-or-nothing test per source line. A single
dropped glyph in a long line loses all of its credit.

The lander reproduced this with a new, isolated render from just the three `text` fences
in the audited machine-load report. `bob ref create ... -k` saved a matching store. A
scratch pointerless note was seeded against that same newly rendered PDF. Adoption still
returned `low match 2/3 blocks`. The PDF text loses only the U+27F3 glyph in the first
block's line:

```text
stale:    ⌂athena cap 5/8 ⟳41s              snapshot older than ~2 refresh cycles
```

The PDF contains `5/8 41s`, otherwise the same words. Improve the matching
implementation so local extraction differences contribute their actual character cost to
similarity, instead of throwing away an entire line. Use bounded local alignment/windows
with ordered matches; do not treat a scattered subsequence of words anywhere in the
document as a matching block. Preserve page spans and avoid letting a failed unit
advance the cursor past otherwise valid later units. Retain an exact-match fast path and
bound the cost on long reports.

Add positive/negative regressions for this exact lost-glyph case, genuinely altered
content below threshold, out-of-order content, repeated generic text, and page
boundaries. Verify code blocks and paragraph fallback. Use the real render toolchain for
at least the new exact-render adoption regression.

Inspect representative paragraph and block failures from the 55-reference list with the
unchanged thresholds. Correct proven parsing/extraction false negatives using the same
matching authority; do not assume all reported failures are edits. Keep any true
changed-source mismatch explicit in the inventory. Avoid general renderer or
source-store redesign.

## 2. Preserve orphan catalogs through candidate fallback

An explicit source pair currently bypasses the title check only after ordinary adoption
rejects it. If the candidate bytes equal those already in the orphan store,
`attach_verified` calls `write_store`, whose legitimate same-source refresh behavior
replaces the render-time catalog with a new attached/text-match catalog.

The lander reproduced this in a scratch Git vault: source title JSON Verification,
pointerless note title Renamed Fixture, same PDF/source digests, render catalog
route=markdown and block page_source=anchor. `attach-source report -s report.md`
reported adoption title mismatch, then attached the same bytes and changed code.json to
route=attached and page_source=text_match. That violates this epic's catalog rule.

Make candidate selection distinguish a new store from an existing store target. For a
verified explicit pair that identifies the same unclaimed, correctly paired render
store, reuse the catalog-preserving adoption path. Refuse a candidate whose destination
collides with a claimed, wrong-PDF, invalid, or otherwise ineligible render store rather
than refreshing that catalog. Preserve the existing multiple-qualifying-store refusal. A
different verified source with a distinct content-addressed destination can still follow
the normal fallback path.

Keep this protection in attachment/adoption policy; do not globally disable catalog
refresh in `source_store::write_store`, which other authorized writers use. Add
regressions asserting complete code.json byte equality, preserved anchors, correct
scoped commit contents, no write on refusal/dry run, and explicit-pair title handling.
Adjust the existing wrong-PDF fallback test if its same-content fixture currently
depends on overwriting the old render catalog.

## 3. Integrate JSON output with real writes

`commit_batch` unconditionally prints `committed <sha> (N references)` to stdout.
Consequently a successful write-mode adoption or attachment with `-f json` cannot be
parsed as JSON, although dry-run JSON tests pass. Keep stdout a single JSON envelope in
JSON mode. Preserve useful commit diagnostics on stderr or in the existing human output
path; do not break the versioned envelope unnecessarily.

Test a real committing adoption and a real committing search attachment with
`serde_json::from_slice` over the entire stdout. Keep the human-mode commit message and
verify multi-batch behavior uses the same format-aware reporting policy. Record the
completed shared repair on the batch-17 epic named above.

## 4. Verify and deploy without waiting for this turn's commit

Run the focused attach/source-store/sync/intake regressions, then `just check` and
`just docs-check`. Failures introduced here must be repaired. For unrelated failures,
establish unchanged-base evidence and use the required task workflow. Do not substitute
check-full or invent further CI gates.

Build/install the tested working tree on athena. Deploy that tested build to apollo
through the normal supported install path, or transfer the compatible Linux binary after
checking host architecture, without waiting for the host's post-turn commit or a release
SHA. Use `/sase_monitor` if a command needs a long wait, carrying these remaining
closeout instructions in the successor context.

Repeat the original throwaway-vault create smoke with installed bob on both hosts,
proving the note's pointer equals the create report and `ref code` is available.
Exercise the current bob_research_capture wrapper in isolated vaults so its stable-name
path is covered too. Recheck the 300-second hook timeout on both hosts. If configuration
needs correction, open chezmoi through `/sase_repo` and obey its instructions; do not
regress the stable recapture wrapper.

## 5. Complete the bounded hook-window backfill

Rebuild the inventory with `sase file-hook history -j -n 0` on both hosts, filtering
research-highlights audits since 2026-10-12T00:00:00-04:00 using timezone-aware
comparison and deduping `(project, rel_path, commit_sha)`. Follow the original plan's
batch/log mapping to references, including newer wrapper stable stems, queued runs, and
explicit missing-reference classification. Reuse prior durable evidence but do not
assume no hooks ran since the previous phase.

Recover exact hook-time source bytes from the audited commit in each opened research
sidecar (bob-cli, sase, job, tailnet); use SASE's audited artifact/repository workflow.
Stage candidates outside the live vault, named for the reference stems. If needed, try
the same sidecar's HEAD only through the original verified fallback procedure. Do not
rely on the prior worker's ephemeral workspace or temporary files.

First dry-run machine_load_accounting_transparency adoption. It must now verify and
adopt without changing source.md or code.json; confirm the saved code/links after
repair. Recheck the other reported orphan repairs and new orphans, if any.

For attachments, evaluate the complete original 72-candidate cohort, including
rechecking saved sources of the already-attached 17 in an isolated fixture if needed,
plus newly discovered eligible entries. Report counts with an explicit denominator.
Honor the original gate: if more than 10% skip, or there is a repeated unexplained
systematic failure, investigate and stop before more writes. Do not claim a lower skip
rate by excluding failing references. Use only verified candidates in the write pass,
with at most 100 references per scoped, revertable Bob commit.

Run `bob ref doctor`: `ref sources` must be ok, with no remaining orphans, dangling
pointers, or digest mismatches. Reclassify the entire window. Every eligible reference
must have a pointer apart from individually documented true source/PDF mismatches
allowed by the original plan; no systematic matcher defect or unresolved orphan may be
relabeled as an exception. Do not create previously missing references or rerender
annotated PDFs. If an acceptance criterion still cannot be met, retain the epic as open
and record the precise blocker.

Verify `bob ref code` and `bob ref links` for every newly adopted reference and at least
10 attachments. Compare representative raw code blocks with the source and PDF and check
byte preservation. Register the refreshed inventory, dry-run results, and verification
summary as artifacts attached to d2; record vault commit IDs, per-host smoke results,
counts, exceptions, and all follow-up outcomes there.

## 6. Close bob-cli-d2 in this turn

This is the tale's final step, after the repairs and acceptance checks above.

1. Re-read `bob-cli-d2`, all descendants and their notes, the linked plan, and any
   source changes that landed during this tale. Confirm no outstanding epic work.
   Follow-up dispositions are already triaged above; carry them into the close note.
2. Run `sase bead epic-symbols bob-cli-d2`. Resolve each listed symbol by wiring,
   privatizing, a justified non-test pragma, or deletion under the Symvision policy.
   Re-key a Justfile entry only to a still-open later bead that truly needs it. The
   lander found no entries, but recheck immediately before closing.
3. Close normally:

   ```sh
   sase bead close bob-cli-d2 --note "Verified all three phases and their notes; birth pointers, catalog-preserving adoption including explicit fallback, pure write-mode JSON, integrated intervening changes, both host installs/hooks/smokes, and the completed hook-window backfill. Record actual check results, inventory counts, exceptions, artifact refs, vault commits, and follow-up dispositions here."
   ```

   Replace the illustrative note with the concrete final evidence. Never use `--force`
   just to succeed or to advance a successful nested landing. If symbols or incomplete
   descendants reject the close, finish the cleanup/work and retry normally. Only
   deliberate canceled/superseded work permits the user's explicit
   `--force --reason ... --resolution canceled|superseded` path.

4. Run `just symvision` if available. No recipe existed during the audit; if it is still
   absent, record that and the clean epic-symbols result rather than adding an unrelated
   tool. If present, it must pass after closure.
5. Open the plans repository through `/sase_repo`, then set only `status: done` in the
   frontmatter of the epic's linked plan file,
   `sase/repos/plans/202610/research_ref_saved_markdown.md` (resolve the current PLAN
   path from the bead instead of assuming any workspace path).
6. Run `sase bead read bob-cli-d2 -r "Need the parent link"`. The audited value is no
   parent, so finish normally. If concurrent work has added a parent, follow the user's
   parent rules: verify and close only a directly parented phase, or fully re-audit a
   plan parent's prior landing note, descendants, linked-plan readiness and drift before
   normal closure, symbol cleanup, symvision, and plan status update. Stop at the first
   incomplete/ambiguous parent and note its blocker.
7. Use `/sase_final` as the last action before the final response. Declare this tale's
   repository changes for the host to commit; no dependency on that commit belongs
   before the epic close above.
