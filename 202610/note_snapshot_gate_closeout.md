---
tier: tale
title: Repair the snapshot corpus gate and finish bob-cli-87.13 landing
goal:
  The corpus gate detects existing-file false warnings and reports each warning in its
  source context, bob-cli-87.13 is verified and closed, and parent readiness is handled
  explicitly.
size: medium
proposed_by: bbugyi200.athena.bob-cli-87.13.land
proposed_with: codex/gpt-6-astra
bead: bob-cli-87.13
create_time: 2026-10-17 19:48:57
status: done
---

- **PARENT:**
  [202610/note_snapshot_landing_repairs.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/note_snapshot_landing_repairs.md)
- **BEAD:**
  [bob-cli-87.13](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-87/bob-cli-87.13.md)

# Repair the snapshot corpus gate and finish bob-cli-87.13 landing

## Scope and evidence

This is the remaining work from the land audit of **bob-cli-87.13**, whose accepted plan
is `plan:202610/note_snapshot_landing_repairs.md`. All four phases are closed. The
implementation at `860f0be` includes the projection, task-row, image-frame,
earlier-snapshot, text-layer, and documentation repairs. The final phase recorded 102
athena renders, 5,077 dry runs, zero residue, and a passing `just check`. The lander
independently ran
`cargo test --lib pandoc_maps_every_contract_construct_to_tex -- --nocapture`: one
passed, so the epic's reported lifted-link failure does not reproduce here.

Two defects in `scripts/dev/note_snapshot_corpus_gate` mean the gate still cannot
establish its promised existing-file check:

1. `find_vault_file_match` treats directory-qualified Markdown targets as exact-only.
   The same epic's `860f0be` changed `VaultLinkResolver`, `NoteIndex`, and
   `AttachmentIndex` to use the final basename after a path miss. With `notes/Target.md`
   the unique basename, both `old/Target` and `old/Target.md` are wrongly reported
   missing by the gate; bare `Target` works. The lander's synthetic full gate, with a
   stub renderer returning a missing-file warning for `old/Target.md`, exited **0**,
   reported **0 false unresolved file warnings**, and described it as genuinely missing.
   Expected: failure, because the file exists by unique basename.
2. The final `Dry-run warning details` loop calls
   `dry_warning_explanation "$line" "$rel"` using the last dry-run note's `rel` for
   every earlier warning. A warning from `notes/Host.md` about `img/chart.png` matches
   `notes/img/chart.png` with its correct origin, but is labeled genuinely missing when
   the last note was `elsewhere/Last.md` and `chart.png` has multiple vault-wide
   candidates. The count and the report can therefore disagree.

These are epic-caused defects, not follow-up tasks. Fix only this remaining gate work;
do not redo the completed renderer phases.

## Constraints and already completed triage

- Preserve the accepted choices: `mac_rerun = no`; do not contact the Mac. The parent
  selected `default_text_layer = yes`, `obsidian_command = yes`, and
  `decision_record = no`. Edit no memory. Add no TeX packages or fonts.
- Keep vault data out of source, fixtures, notes, and public artifacts. Synthetic
  fixtures only. Live checks use dry runs and scratch-output rendering outside the
  vault, with before/after vault git-status checks.
- Use `just check`, never `just check-full`. Use `/sase_monitor` for commands that need
  a long wait; follow its named-tool escalation rules.
- Use audited artifact reads for plans and `/sase_repo` before opening linked
  repositories. Use the returned plans checkout for status edits.
- The lander already completed `/sase_new_task` triage and recorded it on bob-cli-87.13.
  Phase .13.2 proposal #1: corroborated **bob-cli-8q** for the bash PTY flake and
  recorded both bash/zsh failures on causally responsible active epic **bob-cli-3j**.
  Proposal #2: corroborated **bob-cli-5l** for the curl executable-stub ETXTBSY race. No
  new tasks were created; duplicate proposals were declined. Carry these outcomes into
  the close note.
- The lander reviewed all four repair commits and six intervening commits: `c9a5063`
  (Highlights hotkey docs), `b59af42`/`b5adef5` (Mac keychain install reporting),
  `8b6ca21`/`4052bd7` (capture open_notes), and `30bdfb8` (reverse Blocks contract).
  None needs snapshot integration: capture locations are independent, and Blocks
  preserves the generated dependency grammar. `git fetch origin` found
  `HEAD == origin/master == 860f0be` at audit time. Recheck any newer drift before
  landing.

## Work

1. **Make file-warning classification agree with the resolver.** Read the current
   resolver and gate before editing. Keep exact and source-relative hits first. After
   those miss, check the last component of a path-qualified target with the same
   case-insensitive basename rules as a bare target: Markdown note names with or without
   `.md`, non-Markdown files with their extension, and unique non-dot-directory
   candidates. Keep ambiguity and true absence distinguishable from an existing match.
   Remove the stale exact-only comment. Do not weaken the existing-file check to make
   the corpus pass, or change product resolution merely to match the old checker.

2. **Preserve warning origins through report generation.** Compute and store each
   explanation while its originating `rel` is in scope, or store the origin alongside
   the raw warning. Reuse the same classification for the failure count and report so
   they cannot disagree. Do not recover origins by splitting a user path on an arbitrary
   colon.

3. **Add synthetic regression coverage for the gate's behavior.** Cover
   directory-qualified `old/Target` and `old/Target.md` resolving by unique basename,
   case-insensitive matching, ambiguous and genuinely absent names, exact and
   source-relative attachment matches, and two warnings from different source
   directories. Exercise the complete gate's exit status and generated summary, not just
   a copied implementation of its helper. A small fake `BOB_BIN` plus fake PDF
   inspection tools can supply retained scratch workdirs, minimal projected/TeX files,
   and controlled warnings; the synthetic vault can be an initialized temporary git
   repo. Verify the existing false-warning case fails and the legitimate missing case
   passes with the correct explanation. Wire the regression into an appropriate existing
   test/check path without adding public CLI options. Keep `just check-scripts` covering
   the gate.

4. **Verify the repaired gate and update the verification record.** Run the focused
   regressions, `just check-scripts`, and `just check`. Build bob from the current
   checkout and rerun the athena corpus gate with `BOB_BIN` set explicitly, output
   outside the vault. Require 100% render/dry-run success, zero missing glyphs, Type 3
   fonts, overfull warnings, residue and false unresolved-file warnings, and unchanged
   vault git status. Investigate any newly exposed false warning rather than bypassing
   it. Check representative warning explanations against their actual origin. Rendering
   code is not intended to change; reuse phase .13.4's documented visual review if so.
   If rendering must change, repeat the affected visual checks. Update
   `docs/highlights-create.md` with the corrected gate's numbers/date and honest
   provenance (base commit plus working-tree changes is sufficient). Keep the Mac
   `df4e120` verification explicitly pre-fix and the live round-trip limitations
   explicit. No new Mac run is authorized by this tale.

5. **Finish the landing in this coding turn.** There is no later land agent for this
   tale. Do not wait for this turn's own commit SHA, push, or CI result; the host
   commits after the turn ends. Read `bob-cli-87.13`, its four children and any new
   notes, and `plan:202610/note_snapshot_landing_repairs.md` through audited commands.
   Verify completed scope and DECISIONS, and review post-audit non-epic commits for
   integration. Carry the triage outcomes above into the close note, including why no
   new tasks were filed.

   Run `sase bead epic-symbols bob-cli-87.13`. Resolve every entry by wiring,
   privatizing, adding an appropriate non-test pragma, or deleting the symbol; re-key an
   exemption only to a still-open later bead that really needs it. The audit found no
   entries, but check again. Then close normally:

   ```sh
   sase bead close bob-cli-87.13 --note "<scope and notes verified, integration review, corrected corpus and just check results, follow-up outcomes>"
   ```

   Run `just symvision` if available (the audited justfile has no recipe). Open the
   plans repo with `/sase_repo` and set `status: done` in the repair plan,
   archive-relative path **202610/note_snapshot_landing_repairs.md** (the bead's PLAN
   path is `sase/repos/plans/202610/note_snapshot_landing_repairs.md` relative to the
   current workspace). Do not hand-edit bead status. If close rejects remaining symbols,
   resolve them and retry. If named phases are incomplete, finish/reopen them; use
   `--force` only for an actual canceled/superseded outcome with a reason, never for
   successful nested landing.

   **Parent bob-cli-87:** after .13 closes, run
   `sase bead read bob-cli-87.13 -r "Need the parent link"`. The recorded parent is plan
   bead **bob-cli-87**, not a phase. Read the parent's earlier LAND AUDIT/triage notes,
   every descendant and note, and `plan:202610/note_snapshots.md`; recheck descendant
   and linked-plan readiness plus post-child drift. Do not assume a closed phase proves
   its acceptance. There is a recorded ambiguity: parent acceptance #4 requires the live
   Highlights `#task` follow-up to land, while current docs and the earlier LAND AUDIT
   say that path and an image highlight were not exercised. Phase .12 notes #20–#24
   contain failed empty attempts; the child `mac_rerun=no` decision does not explicitly
   waive that parent requirement. Look for newer completion evidence or an explicit
   waiver. If none resolves it, leave bob-cli-87 open, append a concrete blocker note,
   and report it; do not contact the Mac or claim a full parent close.

   Only if the parent is fully complete: retire its own
   `sase bead epic-symbols bob-cli-87` entries, close it normally with a note stating
   what was rechecked, run `just symvision` if available, and mark
   **202610/note_snapshots.md** done in the opened plans checkout. Repeat through any
   directly parented plan ancestors only while fully complete; stop and note the first
   incomplete/ambiguous parent. A phase parent is closed by itself after verifying its
   required work, leaving its containing epic for its waiting land agent. Finish with
   `/sase_final` so all modified repositories are included in the host commit
   declaration.
