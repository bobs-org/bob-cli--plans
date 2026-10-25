---
tier: tale
title: Constrain relation cache pruning and finish bob-cli-bj.12 landing
goal: Preserve unrelated and future-version cache files, verify the repair, close
  bob-cli-bj.12, and recheck parent readiness.
size: small
proposed_by: bbugyi200.apollo.bob-cli-bj.12.land
proposed_with: codex/gpt-6-astra
bead: bob-cli-bj.12
status: done
---

- **PARENT:**
  [202610/relation_label_vocabulary_landing_repairs.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/relation_label_vocabulary_landing_repairs.md)
- **BEAD:**
  [bob-cli-bj.12](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-bj/bob-cli-bj.12.md)

# Constrain relation cache pruning and finish bob-cli-bj.12 landing

## Outcome

Finish the only remaining implementation issue in repair epic **bob-cli-bj.12**: the
relation catalog sweep must delete only its own obsolete cache files. Then close that
epic and mark its plan done in this same turn. Recheck its parent, **bob-cli-bj**, and
stop there if the parent parity issues described below remain. This tale has no land
agent; its coder owns this closeout. Do not wait for this tale's commit, SHA, push, or
CI before closing the repaired epic.

## Context already verified by the land agent

- Audited `bob-cli-bj.12`, every note on its three closed phases, the repair plan
  `plan:202610/relation_label_vocabulary_landing_repairs.md`, the parent plan
  `plan:202610/relation_label_vocabulary.md`, and every note on parent phases
  `bob-cli-bj.1` through `.11`.
- bob-cli **d8a8f1f8** contains the cache repair and isolated catalog/completion tests.
  Read its diff and current `src/native/relation_labels/scan.rs` and `tests.rs` before
  changing them. The lander's `just check` passed: formatting, clippy, and 5,641 tests
  across 18 result records (including empty doctests), with no test failures. The named
  `sase tool run check` was unavailable, so the lander used the documented
  `SASE_TOOL_BYPASS` fallback to run `just check`.
- bob-mac-capture **a2e230e** fixes the rename state mutability/refusal fields and the
  three failing relation tests. **569cb36** fixes the remaining `RelationLabelField`
  self capture. Completed macOS CI
  <https://github.com/bobs-org/bob-mac-capture/actions/runs/40146325983>, on
  `569cb364b021cd3e5ae611baed1be9aebb0cbb20`, passes lint, app build, test compilation,
  and all 47 `CaptureRelationPanelTests`, 33 `CaptureSeparatorToggleTests`, 13
  `CaptureRelationshipPresentationTests`, and 29 `DependencyDecodingTests`. Of 3,087
  tests, only `bob-cli-bv` and `bob-cli-bw` fail; bundle/signature/smoke/install steps
  are skipped because of those failures. They are outside this epic.
- No unrelated commit landed after the repair epic started in bob-cli, bob-mac-capture,
  or bob-plugins at the audited tips. Parent-period drift includes inbox routing, theme
  cache cleanup, Highlights work, and the Day strip. The inbox route write still uses
  `commit_capture_batch`, including relation snapshot invalidation and after-success
  one-off hides. No integration edit was needed there.
- bob-plugins is **21e9923**, unchanged since the parent's prior landing. Its relation
  vectors are byte-identical to bob-cli; 55 focused relation/index/stage tests passed in
  this audit. The parent lander already ran all 3,648 tests and deployed this tip. No
  plugin change is required for this repair tale.
- `sase bead epic-symbols bob-cli-bj.12` and `sase bead epic-symbols bob-cli-bj` listed
  no exemptions. bob-cli's current justfile has no `symvision` recipe. Recheck both
  facts at closeout.

## Remaining defect and reproduction

`sweep_catalog_cache` in `src/native/relation_labels/scan.rs` removes every file with
extension `tmp` after ten minutes, including unrelated files. Its `sweep_catalog_file`
removes every `version != CACHE_VERSION`, including a cache from a newer Bob version.
The approved repair plan says to remove obsolete versions and stale atomic-write files,
while preserving unrelated files.

The lander reproduced this with the exact d8a8f1f8 workspace binary and an isolated
`XDG_CACHE_HOME`: create a scratch vault with a LINKS entry, put an aged `unrelated.tmp`
and `catalog-v999-<64 hexadecimal characters>.json` sentinel in
`$XDG_CACHE_HOME/bob-cli/relations/`, then run
`bob relations list -b <scratch-vault> -f json`. Both sentinels disappear. Neither file
is eligible for cleanup. No live-vault or real-cache mutation was used.

## Constraints

- Keep the inherited DECISIONS: `hidden_key=ctrl_h`, `obsidian_manager=yes`,
  decision-memory-only authorization, and `memory_glossary=no`. Make no memory edits.
- Do not change theme behavior, the shared preference journal, vault rewriting, command
  contracts, versions, or CHANGELOG.md.
- Use scratch vaults and isolated cache directories. Do not write to the live vault or
  manually clean the real cache.
- Verification is **`just check`**. Never run `just check-full`.
- Use `/sase_repo` before accessing any linked repo or the plans sidecar; use
  `sase artifact read` for plan context. Use `/sase_final` at normal turn end.

## Implementation and verification

1. **Constrain ownership and version checks.** Keep the sweep private and best effort.
   Recognize the generated catalog basename (`catalog-v<version>-<64 hex>.json`) before
   deleting catalog files. Only delete a version older than `CACHE_VERSION`; preserve
   future versions. For the current version, retain the existing parsed vanished-vault
   test, keep-vault exception, and malformed-content preservation. For temporary files,
   recognize the actual name from `collect_done::temporary_write_path` in
   `src/native/collect_done/archive.rs`: `.catalog-v<version>-<64 hex>.json.<pid>.tmp`.
   Require a numeric PID and a supported current/older version; preserve future-version,
   malformed, and unrelated names. Apply the ten-minute age check only to eligible
   temporary files. Do not add a new sweep to the snapshot-hit path or make cleanup
   failure fail a scan.

2. **Extend the existing regression and run the gate.** Extend
   `catalog_scan_prunes_vanished_vaults_and_sweeps_orphan_files` (or split it if
   clearer) so it verifies the positive cleanup cases already covered and preserves an
   aged unrelated `.tmp`, a future-version catalog, a future-version atomic temp, and
   malformed catalog/temp names. Continue preserving the current live-vault cache,
   current malformed JSON, unrelated text file, and fresh atomic temp. Exercise real
   `Catalog::load` in a scratch cache, without sleeping or exposing a new public API.
   Run the focused test, then `just check`; use `/sase_monitor` if necessary for a long
   run. Retain the phase's XDG isolation. No doc change is expected: the existing
   pruning sentence already describes the intended behavior. If docs do change, read
   `docs.md` through `/sase_memory_read` and run `just docs-check`.

3. **Finish the landing in this turn.** Read `bob-cli-bj.12` again, including the
   land-audit and follow-up triage notes, and confirm its three phases remain closed.
   Review drift from the audited tips above; integrate any new changes that affect this
   repair. The parent follow-up outcomes are already triaged: glossary proposals remain
   in **bob-cli-bu**; the stage-ranker report remains in **bob-cli-3w**; snapshot
   invalidation and prior theme/help gate failures were resolved; the previously
   declined ref-rename transaction proposal is now ready task **bob-cli-bx**, and the
   macOS precommit verification proposal is now ready task **bob-cli-by**. Non-epic Mac
   failures remain **bob-cli-bv** and **bob-cli-bw**. Carry these outcomes and their
   reasons from the epic notes into the close note.

   Run `sase bead epic-symbols bob-cli-bj.12`. Resolve every entry by wiring,
   privatizing, adding a justified non-test pragma, or deleting it; re-key only when an
   identified still-open later bead really needs the exemption. Then close normally with
   `sase bead close bob-cli-bj.12 --note "<cache regression, just check result, Mac CI evidence, drift review, and all follow-up outcomes>"`.
   Run `just symvision` if the recipe exists (record explicitly if unavailable), then
   open the plans sidecar through `/sase_repo` and set only the frontmatter
   `status: done` in the PLAN file resolved by `sase bead read`:
   **`plan:202610/relation_label_vocabulary_landing_repairs.md`**.

   Read `sase bead read bob-cli-bj.12 -r "Need the parent link"`, then recheck
   **bob-cli-bj**, its previous landing notes, all descendants and notes, the linked
   parent plan, and post-child drift. There are two newly recorded parent blockers:
   - Rust `Catalog::add_journal_only_keys` retains named explicit preferences even when
     unhidden and unused. Obsidian's `relationLabelCatalogRows` in
     `plugins/bob-ledger-tools/src/147-relation-label-model.js` skips every journal-only
     preference unless `hidden===true`. A hide/unhide of an unused label consequently
     leaves a visible zero-count Rust row but no Obsidian row, violating the parent's
     cross-client catalog agreement. The desired resolution must retain the current bob
     contract and add parity coverage.
   - Rust `collapse_label_text` trims, collapses whitespace, and strips one trailing
     colon. The Mac model's string `relationLabelKey` at
     `Sources/BobMacCapture/CapturePanelModel.swift` only collapses whitespace and
     lowercases. With a `see` catalog row, query `see:` matches in bob and Obsidian but
     not the Mac card. It also affects exact typed-key eligibility for rename. This
     requires a compiled Mac fix and regression coverage, outside this cache repair
     tale.

   If either parent issue remains, **leave bob-cli-bj open**, append a parent note
   naming the remaining blocker and confirming bj.12 landed, and report that status. Do
   not mark the parent plan done. This is the user-required stop at the first incomplete
   parent; do not turn this tale into another feature repair epic. If concurrent work
   has resolved both, verify it, rerun descendant/linked-plan readiness checks, retire
   `sase bead epic-symbols bob-cli-bj` entries, close with
   `sase bead close bob-cli-bj --note "<parent scope, decisions, descendants, drift, parity fixes, and triage rechecked>"`,
   run `just symvision` if available, and set `status: done` in
   **`plan:202610/relation_label_vocabulary.md`**. The audited parent has no parent; if
   that changes, apply the original landing prompt's ancestor rules.

   Never use `--force` merely to make a close succeed or to advance a successful nested
   landing. Resolve symbol or descendant blockers deliberately. If this tale is
   represented by its own descendant plan bead, complete that bead normally after its
   implementation and verification are complete, before the epic close; do not wait for
   a finalizer commit and do not force-close it with the parent.
