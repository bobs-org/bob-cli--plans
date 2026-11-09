---
tier: tale
title: Repair guarded archive landing gaps and close the writer epic
goal:
  Fix four reproduced archive regressions, verify integration, and close bob-cli-j5.2.1
  and parent phase bob-cli-j5.2 normally.
size: medium
proposed_by: bbugyi200.athena.bob-cli-j5.2.1.land
proposed_with: codex/gpt-6-astra
bead: bob-cli-j5.2.1
create_time: 2026-11-09 15:27:17
status: wip
---

- **PARENT:**
  [202611/guarded_archive_writer.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/guarded_archive_writer.md)
- **BEAD:**
  [bob-cli-j5.2.1](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-j5/bob-cli-j5.2.1.md)

# Finish guarded archive verification and land bob-cli-j5.2.1

## Assignment and boundary

Finish the four defects reproduced by the land audit of `bob-cli-j5.2.1`, then perform
this epic's closeout **in this same coding turn**. This is one bounded implementation
assignment, not another epic. A tale has no land agent to resume this landing. Do not
wait for the final code's commit SHA, push, or CI before closing; host finalization
commits the verified work after the turn.

Read these using their audited commands:

- `sase bead read bob-cli-j5.2.1 -r "Need landing findings and follow-up outcomes"`
- `sase bead read bob-cli-j5.2 -r "Need the parent phase completion boundary"`
- `sase artifact read plan:202611/guarded_archive_writer.md "Need the accepted writer contract"`
- `sase artifact read plan:202611/done_notes_archiving_fixes.md "Need the parent phase scope"`

The parent is **phase bob-cli-j5.2**, inside epic **bob-cli-j5**. Close only the current
epic and that phase after verification; leave bob-cli-j5 and its plan status to its
waiting land agent.

Preserve the final decisions: `keep_window=week_cap`, `finished_projects=frozen`,
`rename_following=rename`, `plugin_reopen=restore`, `memory_archive_decision=no`. This
work retains the current threshold and pairing; the later parent phases implement keep
windows, pairing, restore, and link-repair policy. Edit no memory files, Cargo version
fields, or CHANGELOG.md. Never run a writing archive/nightly command against the live
vault. Use isolated temporary vaults, lock files, state/cache/config directories, and
local remotes for tests.

## Audit already completed

The lander read all three child beads and every note, both plans, and the implementation
in commits `2b5b8328` (guarded creates), `37df0e52` (captured planning), and `e802071f`
(command/report). Children are closed, but the four findings below leave the epic
incomplete. `cargo build --bin bob --quiet` passed; the four CLI fixtures used the built
binary from `cargo metadata`'s `target_directory`, not a stale `target/debug/bob` or
installed command.

The isolated reproduction script is `file:explicit:32c5dd2c69f2b0a7b394a3d8`; its four
results are `file:explicit:822ae2f99cc5cbfc67be94e9`. Read either with
`sase artifact read <ref> "Need exact landing reproduction evidence"`. No final
`just check` was run by the lander after finding these defects; run it after the repairs
as required below.

Integration review covered every non-epic commit from `2b5b8328` through `e802071f`:
`4b59ae81`, `4321fd70`, `0a74731b`, `93d67df4`, `d7b339d0`, `55468bff`. Fetch showed no
additional origin/master commits at audit time. The notebook catalog writer deliberately
uses the shared capture transaction with rollback; do not replace it with the archive
engine, whose partial writes and recovery are different. Nightly invokes both writers
under its existing maintenance lock. The new catalog is an integration case for grouping
tests. Alias tips, record rendering, and install output have no archive integration
changes. Recheck any commits added after `e802071f` before closing.

## 1. Regroup repair-only eligible sources

In `src/native/collect_done/plan.rs`, `plan_from_captured` calls
`apply_same_write_grouping` only on `files`. An area/project changed solely by link
repair is instead in `link_repairs`; `LinkRepairPlan` has no structural flag, and
`write_plan_for` hardcodes `false` for these outputs.

Reproduction: `work.md` is an area containing `- [x] finished #task ^gone`.
`observer.md` is an area whose `## Tasks` section contains
`- [ ] See [[work#^gone]] #task ^live` followed by an empty `### Done & Canceled`.
Archive with threshold 1 succeeds and repairs the link to `[[done/work_done#^gone]]`,
but leaves the empty Done heading.

Apply the same settings-based pure grouping transform to every changed eligible source,
including link-only and dependency-metadata-only repairs. Read eligibility and original
bytes from captured inputs. Preserve the existing exclusion of daily, notebook,
reference, and untyped grandfathered notes from grouping. Keep one final postimage per
changed path, preserve report repair counts, and carry a structural flag into
`planned_write` whenever grouping changes the postimage. Do not invoke reconcile or
alter task status/log/priority/schedule inputs.

Add planner/apply regressions for repair-only grouping, removal of the empty Done
heading and associated chip, the structural flag/quiet-period behavior, excluded note
kinds, and second-run idempotence. Include an area with the generated `## Notebooks`
catalog from `55468bff`; grouping must preserve its region bytes while link repair and
task grouping finish correctly.

## 2. Validate original and destination identities separately

`verify_moved_identities` currently searches for each **final destination ID** in the
source and refuses if it finds one. Destination IDs are note-qualified, so this can
mistake a different surviving source task for the moved occurrence.

Reproduction: source area `work.md` contains `- [x] new body #task ^a1` and
`- [ ] active #task ^a1-1`; its paired archive contains `- [x] old body #task ^a1`.
Preview with threshold 1 should move the new body to the archive as `^a1-1` while
preserving the active source task. Instead it refuses:
`moved block id ^a1-1 still occurs at its source`.

Use the captured moved-root record's original identity/occurrence when proving removal
from the source, and its final note-qualified identity when checking the destination. Do
not solve this by disabling source-removal validation, arbitrarily renaming a legal
source ID, or making bare IDs globally unique. Retain refusal of genuine
missing/duplicate destination anchors. Validate resolution for contained real anchors as
well as roots, with replay coverage. Parent phase bob-cli-j5.3 still owns ambiguous-ID
partial-success holds and renamed self-link repair.

Add regressions for the example, nested anchors, replay, repeated legal bare IDs across
notes, and a genuinely invalid postimage that still refuses before writes.

## 3. Use content-aware anchors throughout the verification records

The destination census `anchor_id_lines` masks fenced code/comments/frontmatter, but
`partition_selected_roots` and `contained_block_ids` build records from raw line scans.
This causes the new validator to demand a nonexistent anchor.

Reproduction: an area has the closed task `- [x] finished #task ^gone` with an indented
fenced `text` block containing `example ^fake`. Preview refuses with
`moved block id ^fake occurs 0 times at its destination, expected exactly once`.

Make moved identity collection and final-ID alignment use the same content-aware anchor
semantics as validation. Preserve exact block bytes, CRLF, child notes, and exact replay
behavior. Code/comment text must not become an identity merely because it ends in caret
text. Preserve offset alignment for actual renamed IDs. This is a verification/anchor
fix, not the separate link-repair rewrite policy owned by bob-cli-j5.3. Cover fenced
code, multiline comments, inline code where relevant to anchor scanning, nested real
anchors, collisions, and exact replays.

## 4. Preserve valid vault-root aliases without allowing unsafe descendants

The new `ensure_destination_parent` in `src/native/task_status_hooks_write/staging.rs`
lexically compares an output path to `WritePlan.vault_canonical`. A valid vault path
reached through a symlink has a different prefix, so it is rejected before any write.
This affects the shared engine's existing replacement users as well as new archive
creates.

Reproduction: create a real temporary vault and a sibling symlink `alias` to it; set
`BOB_DIR` to that alias. Archiving one closed task reports `write_failed` /
`destination is outside the canonical vault`, despite the destination being inside the
real vault. This also matters on platforms whose temporary-root prefix has a canonical
alias.

Normalize the verified vault root and output paths consistently, or otherwise retain
enough root identity to accept a verified alias for the vault itself. Apply the solution
to the shared engine and its existing callers, not just the archive CLI. Keep snapshots,
membership paths, applied/deferred paths, and recovery manifests consistent; the JSON
paths must remain vault-relative. Do not blindly canonicalize descendants and thereby
accept an in-vault symlink or an escape. Preserve refusals for symlink/non-directory
descendants, outside destinations, hard-linked replacement files, and races. Keep
new-file atomic no-clobber installation and `original_missing` recovery semantics.

Add Unix fixtures for a root symlink and an aliased ancestor, each with an existing
replacement and a nested new archive. Include an archive CLI regression and retain the
unsafe-parent/no-clobber tests. Use injected hooks for races, not sleeps.

## 5. Verification, drift review, and epic closeout

Run focused guarded-engine and archive planner/execution tests plus CLI `move_done`,
nightly, and notebook catalog cases as appropriate. Run **`just check`** for final
file-change verification. Do not run `just check-full`. Use `sase tool run check` inline
according to `/sase_monitor`; if it escalates, join that exact run with a monitor and
give the continuation this entire remaining closeout. Do not start a second check or
finish with a live shell subprocess. No closeout condition depends on this turn's future
commit/push/CI. If docs/help change, read docs memory, regenerate generated help as
required, and run `just docs-check`, distinguishing the tracked baseline failures below.

Before closing, re-read the epic and children, their notes and linked-plan readiness,
and inspect post-audit drift. Verify these four gaps are fixed and that the accepted
archive contract still holds. Existing out-of-scope follow-up triage was completed by
the lander and recorded on bob-cli-j5.2.1:

- bob-cli-j5.2.1.3 note #1: the four notebooks registration/count/order/completion
  failures are fixed by `55468bff`; no new task because the proposal is obsolete.
- That phase's note #2, mounted Mac pages: independently measured capture-panel 6006 and
  keyboard 6028 against cap 6000; corroborated existing **bob-cli-h9** with +1 (six
  independent reports). No duplicate task.
- Note #2, JSON reference: `4321fd70` was 6000 words, notebook `0a74731b` made it 6117,
  and archive `e802071f` added 26 to 6143. Evidence was attached as a `DISCOVERED ISSUE`
  to causal active epic **bob-cli-j8**, which already tracks the initial overage.
  Preserve the archive contract when that page is trimmed.
- Other child notes only report verification; there are no other proposed follow-ups.
  Include these outcomes in the epic close note. Triage any genuinely new unrelated
  issue through `/sase_new_task`; epic-caused defects stay here.

Perform these concrete final operations after the code and checks are complete:

1. Run `sase bead epic-symbols bob-cli-j5.2.1`. Resolve every exemption belonging to
   this epic or its phases, or re-key it only to a still-open later bead that needs it.
   The land audit found none; check again. Never leave one keyed to a bead you are
   closing.
2. Close normally with `sase bead close bob-cli-j5.2.1 --note "<verification>"`. The
   note must summarize the three original phases, the four repaired gaps,
   post-start/post-audit integration, actual test results, and the follow-up
   dispositions above. If closure refuses an exemption or unfinished descendant, finish
   that work and retry. Never force to make a successful nested landing advance. Do not
   cancel or supersede completed work to bypass readiness.
3. Run `just symvision` when the recipe is available. It is absent in the audited
   justfile; if still absent, record that explicitly instead of inventing a pass.
4. Set `status: done` in the frontmatter of the epic's linked plan
   **plan:202611/guarded_archive_writer.md**. Use `/sase_repo` to open `plans`, use its
   printed checkout path, and audit any artifact read with `sase artifact read`. The
   relative plan file is `202611/guarded_archive_writer.md`. Do not hardcode a numbered
   workspace path.
5. Re-read `bob-cli-j5.2.1` for its parent link and `bob-cli-j5.2` for readiness. Verify
   this child plan completes that phase's guarded-writer scope. Run
   `sase bead epic-symbols bob-cli-j5.2`, resolve or legitimately re-key any entries,
   and close **only phase bob-cli-j5.2** normally with
   `sase bead close bob-cli-j5.2 --note "<what was verified>"`. Leave containing epic
   **bob-cli-j5** and `plan:202611/done_notes_archiving_fixes.md` open for its own
   waiting land agent. If the parent is unexpectedly incomplete or ambiguous, record the
   blocker on bob-cli-j5.2 and report it rather than forcing closure.
6. Finish through `/sase_final`, including every changed repository in its required
   declaration. The finalizer commits the code and plan-status changes; do not wait for
   that commit before the preceding close operations.
