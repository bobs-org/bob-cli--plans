---
tier: tale
title: Repair relation-successor core gaps and finish its landing
goal: Correct core identity, recovery, ordering, placement and fixture parity, verify
  the result, then close bob-cli-fs.2.1 and parent phase bob-cli-fs.2.
size: medium
proposed_by: bbugyi200.athena.bob-cli-fs.2.1.land
proposed_with: codex/gpt-6-astra
bead: bob-cli-fs.2.1
status: done
---

- **PARENT:**
  [202611/relation_successor_core.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/relation_successor_core.md)
- **BEAD:**
  [bob-cli-fs.2.1](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-fs/bob-cli-fs.2.1.md)

# Finish the relation-successor core and land bob-cli-fs.2.1

Repair only the gaps found while landing `bob-cli-fs.2.1`, then finish its landing and
close its parent phase `bob-cli-fs.2` in this same coding turn. This is one bounded
implementation across the existing classifier/planner seam, its tests, and shared
fixture parity; it needs one coder, not another phased epic.

## Context and constraints

Read `bob-cli-fs.2.1` and its three children with `sase bead read`, and read
`plan:202611/relation_successor_core.md` and `plan:202611/starts_after_relation.md` with
`sase artifact read`. The latter's DECISIONS are final: no decision record, and the
authorized glossary work was completed by the contract phase. This repair changes no
memory or generated agent instructions. Read required reference memory through
`sase_memory_read`.

The reviewed bob-cli tree is `cb729a76`, equal to freshly fetched `origin/master`. The
core commits are `c3a2e62a` (policy/copy), `ace86f8e` (classifier), and `cb729a76`
(integration). Those commits retain all capture callers at `relation: None`; preserve
that boundary. Do not implement capture event detection, disk scanning, batching, new
JSON fields, CLI grammar, or the downstream parent phases. Do not change Cargo versions
or CHANGELOG. The only linked-repository repair in this tale is the navigation
shared-vector/copy parity described below; open bob-plugins through `sase_repo` and obey
its AGENTS.md. No vault note mutations are needed.

Canonical behavior is in `docs/task-relations.md` §20 and `docs/task-dependencies.md`
§§12.18–12.19 plus the core plan. Recovery is automatic; linking, ID minting, and fresh
stamps require explicit selection. Relation-only unpicked rows write nothing.
Cancellation skips relations. Disabled/nontriggering calls perform no relation lookup.
Keep these boundaries and existing group behavior.

The lander read every child note and inventoried non-core commits since the first core
commit: `311b1514` (blocked-mark docs), `b483c93b` (section completion plus the stale
keep-moving expectation), and `fa1118c1` (gcal link ordering). The earlier `16f3a28f`
documentation registration and `a8f61375` ID-cap change were also checked. Gcal sync is
not a managed successor gesture; section completion and blocked-mark presentation need
no new relation wiring. Preserve the current shared ID suggester and its 20-byte first
suggestion. Recheck subsequent drift before closeout.

## Verified failures and acceptance gaps

The lander temporarily appended four tests to
`src/native/task_complete/tests/relation_successor_tests.rs`, ran them, and restored the
tracked file byte-for-byte. Audited artifacts preserve both the probes and output:

- `file:explicit:02f513efcf6d714094b172c0`: Rust probes, suitable for adding to the
  existing test module and formatting normally.
- `file:explicit:ed01e90996e17bd9acfc75b6`: execution on `cb729a76`; all four probes
  fail. Seven unrelated tests matching the broad `land_audit_` filter pass.

The failures are concrete:

1. Classifier rows in one note return `a-second, z-first` although line order is
   `z-first, a-second`. `classify.rs` sorts by block ID. Its group tie-breakers also use
   block IDs where the contract requires target line order.
2. The integrated planner gives a Ready source in `agents.md` number 1 and a Next source
   in `later.md` number 2 in the same relation group. Its old
   `OfferOrderKey = (group, anchor, source_path, lane, line)` cannot express target
   group order followed by lane-first source order, and placement/routing must not
   determine the triggering-group order.
3. A real `[?]` direct dependent recovered by Done becomes a hand-off but loses its
   relation second reason. `relation_recovered_map` marks booleans while
   `classify_source` still tests the original `?` at R7. The current SA14 test uses an
   already-Ready source, and SA15 uses a Ready resident, avoiding this failure.
4. With two identical-text tasks lacking block IDs, only the second having LINKS,
   selecting its offer rewrites/mints the first task.
   `find_file_task_for_relation_source` discards the provisional identity and returns
   the first matching description.

Source review additionally found:

- `RelationEvent.slot` has no production read anywhere. The purported explicit-slot test
  checks only row count, and its deleted-slot case supplies no slot at all.
- `integrate_relation_offers` ignores its `_index`, builds known paths only from held
  files, and wraps those same texts in `MapTargetReader`. An unread target excluded from
  those files cannot resolve, even if in `unavailable_targets`; ambiguity from another
  known note can also disappear. There is no integrated narrow target-read seam as
  required by the core plan.
- Sources without a legacy recovery verdict fall back to raw `depends_on.len()`; inspect
  and fix this so already-closed prerequisites do not become open blockers. Project
  recovery must project effective status and hide/tag removals, not merely bypass one
  condition. Keep incomplete dependency data conservative.
- Moving an existing offer into Relation changes `offer.group` and clears its project
  field without reattaching relation placement. Preserve original reasons and recovery
  separately; use the winning relation's event anchor for placement. Keep-moving folds
  are collected before this merge and are not filtered afterward.
- The integration tests do not execute all skipped fixture expectations. The
  `seen == expected_ids` check counts skipped IDs as covered. SA15's synthetic test does
  not assert all fixture fields, and `run_planner` always disables keep-moving and uses
  `bob_dir: None`, so it proves neither that dedup nor typed routing. The capture
  compatibility test calls the identical planner closure twice; it does not load three
  configs or compare capture JSON/text/writes.

## Implementation

### 1. Preserve identity and post-recovery facts across the pure boundary

Add the four regression probes as permanent, appropriately named/formatted tests. Use an
explicit source identity carrying its note and exact task location/provisional key
across extraction, classification, merge, group filtering, and write selection. Keep
real block IDs distinct from provisional keys. Do not resolve missing-ID sources by
text, or key all missing-ID recovery entries in a note by the bare path. Test two
identical id-less sources, one and then both with relations; selection by number/key
must change only the selected row, mint distinct IDs, and retain preview keys.

Project effective recovery state with one pure helper, reusing the existing dependency
and project analysis. Do not run the writer to manufacture a preview. Carry enough facts
for R1–R7 and ordering: effective status/lane, remaining prerequisite state, project
block, and planned hide/tag removal. Preserve original state for final writes and
reporting. Test actual blocked direct dependents, blocked project residents, and hidden
project-finished pull-ins with both no picks and explicit picks. Verify relation
metadata survives hand-off precedence and that lower-priority recovery writes survive a
Relation winner. Test a non-dependent relation source whose own prerequisite was already
closed before the gesture, plus unavailable dependency data.

Make the integration consume the complete caller-supplied note index, independent
relation source texts, and a narrow cached target reader or equivalent explicit
in-memory target data. Keep relation and dependency completeness separate. Staged texts,
including deleted LINKS, override older snapshots. Never enumerate the vault or read a
label cache. Preserve shared reconcile resolution for same-note, qualified, basename,
ambiguous, archived, missing/non-task, and self-edge cases. A resolved but unread target
must hold R6, not silently disappear. Add integrated tests where an unavailable target
or an ambiguity-causing note is in the index but absent from source texts, and count
reads across repeated edges. Cheap gates must do zero reads.

### 2. Correct ordering, deduplication, and placement

Replace or extend the order key with a typed representation that can express the
existing groups unchanged and relations as:

`group precedence → triggering target baseline anchor → target path → target line → normalized label key → source lane (Next, Ready) → source path → source line`.

Use one consistent ordering through classifier groups, merged rows, offer numbering, and
number/key selection. Do not reorder relation groups by destination after type routing,
or by target/source block ID. Test same-note adversarial IDs, groups with no baseline
anchor, several labels and targets, cross-note Next/Ready, and routing to different
destinations. Preserve stable old-group keys and relative order when relations are
absent.

Maintain all applicable reasons and recovery effects separately from the winning display
group. Hand-off retains its placement and gains relation metadata. For a lower-priority
winner changed to Relation, rebuild placement using the relation's trigger rather than
retaining the previous group's unrelated anchor. Recompute surviving groups/folds after
dedup; preserve the specified legacy empty-project-group behavior. Test real project,
project-finished, and keep-moving overlaps.

Actually consume explicit parked-event slots. Define their line-coordinate semantics
clearly, validate the named day/entry against the post-gesture ledger, use the removed
link's slot when the entry survives, and fall back to the implicit destination when
cleanup removed it. No preview or unpicked offer creates a placeholder. Done and closing
gestures retain struck-slot/inheritance and continuation/next-same-name/new placeholder
behavior; type routing remains the final placement authority. Assert exact day bytes and
reported entry/line, with at least two open entries so a wrong implicit fallback cannot
pass. Test picked and unpicked cases and deleted-slot fallback with a real supplied
slot.

### 3. Finish meaningful vector and capture compatibility verification

Replace the integration skip accounting with actual production assertions for every
integration-owned vector/case/output, including SA14, SA15, SA18, SA20, and SA21. The
pure SA20/SA21 tests cover event-time and final-state helpers only; capture batching
remains the later parent phase's job. Unknown fixture inputs/expectations must fail
coverage, not be marked visited while ignored. Do not weaken approved vectors.

Replace the repeated planner-call compatibility test with real temporary-vault capture
tests. Compare representative JSON, human output, and resulting files under absent,
empty, and populated nonoverlapping successor-label configs for managed Done, Pomodoro
close, complete-label, and cancel recovery. The populated config still passes
`relation: None` to the core and emits no new relation fields. Use deterministic clocks
and paths. Cover selected/unselected writes, fresh/unblocked/keeps cleanup, Ready/Next
status, missing-ID minting, repeated park then done, and typed routing. Reuse existing
SO/PP/KM/LP/recovery/close fixtures and tests instead of duplicating their engines.

### 4. Resolve shared-vector parity from phase .1

Use `sase repo open bob-plugins -r "Finish relation-successor shared-vector parity"` and
read its AGENTS.md. At reviewed bob-plugins `5f1c23c`, the cycler's
`plugins/task-status-cycler/scripts/fixtures/relation_successor/vectors.json` already
matches bob-cli byte-for-byte (SHA256
`08ffccedd68489dbfc06c73faf7a1be65df7b49cb48fdee1804f98d748c11cc0`). The navigation copy
at `plugins/bob-navigation-hotkeys/scripts/fixtures/relation_successor/vectors.json`
still has the old unquoted SA23 YAML and 49-character SU-P22 expectation.

Copy the canonical bob-cli fixture into the navigation location and verify all current
copies are byte-identical. In navigation `src/206b-successor-relation-copy.js`, repair
`relationCopyTruncate`: it currently takes 48 UTF-16 code units then appends an
ellipsis, returning 49 characters and sometimes a lone high surrogate. Match the
Rust/cycler 48-Unicode-scalar inclusive cap, with tests for the corrected SU-P22 and
astral text. Run the navigation/cycler relation-successor vector and presentation
suites, build and build:check, then deploy with `bob plugins sync` as that repo
requires. Recheck concurrent changes and preserve them. Do not force through another
checkout's protected deploy; record any concrete refusal and use the prescribed safe
workflow.

This is epic integration, not a new task. The lander already added the finding to
`bob-cli-fs` and the triage outcome to `bob-cli-fs.2.1`.

### 5. Verify and complete this landing in the same turn

Run `just check` and `just docs-check` on the repaired tree, plus the focused plugin
checks above. Do not run `just check-full`. Use the SASE named-tool/monitor workflow for
long verification and wait for the handoff command itself to exit if it yields. If a
test failure is claimed unrelated, establish its cause and record an existing tracking
outcome through `sase_new_task`; epic-caused failures stay in this repair. No check may
be waived by replacing meaningful expected values with current output.

Re-read `bob-cli-fs.2.1` and all three descendants/notes, its linked plan, and
post-audit commits. Verify every original core acceptance item and every repair above.
The existing phase statuses can remain closed; this tale owns the discovered residual
work. There is no later lander for this tale, and nothing resumes this landing
automatically.

Record all follow-up dispositions in the close note:

- `bob-cli-fs.2.1.1` notes #1/#2: one parity issue; cycler was already synchronized,
  navigation fixture and Unicode truncation now corrected and verified here.
- `bob-cli-fs.2.1.3` note #1: declined as already resolved by unrelated `b483c93b`. The
  lander reran
  `cargo test --test cli capture::keep_moving::human_pick_suffix_and_not_pulled_in -- --exact`
  on `cb729a76`: 1 passed. It expects `^write-changelog`, consistent with the shared ID
  cap. Do not reopen or refile it.
- No new unrelated task resulted from the lander's all-status search, last-week sweep,
  and active-epic review. Add outcomes for anything genuinely new found during repair.

Then finish the following concrete closeout, without waiting for this turn's commit,
SHA, push, or CI. The host commits this tale only after the turn ends:

1. Run `sase bead epic-symbols bob-cli-fs.2.1` and
   `sase bead epic-symbols bob-cli-fs.2`. For every result, wire up, privatize, delete,
   or add a valid non-test pragma under Symvision policy. Re-key an exemption only to a
   verified still-open downstream bead that needs it. Both were empty at audit time.
2. Close the epic normally:
   `sase bead close bob-cli-fs.2.1 --note "<source and commit verification, repaired regressions, integration review, check results, and all triage outcomes>"`.
   If stale symbols reject it, fix them and retry. If unfinished descendants reject it,
   complete or reopen them deliberately. Never use `--force` to make a successful nested
   landing advance.
3. Run `just symvision` if the recipe is available; otherwise record that this Rust repo
   has no recipe (none existed at audit time). Open the plans repo via `sase_repo`, read
   the plan using `sase artifact read`, and set `status: done` in the frontmatter of the
   epic's actual linked plan `202611/relation_successor_core.md`. Use the PLAN path
   returned by the fresh bead read and the repo path printed by `sase repo open`.
4. Run `sase bead read bob-cli-fs.2.1 -r "Need the parent link"`. Its parent is the
   phase `bob-cli-fs.2`. Re-read that phase, verify this completed child supplies its
   entire policy/classifier/copy/planner scope, check its symbols again, and close only
   that phase normally:
   `sase bead close bob-cli-fs.2 --note "<core completion, child landing, integration and verification evidence>"`.
   Leave outer epic `bob-cli-fs` and its plan to the already-waiting outer land agent.
   Do not mark `starts_after_relation.md` done. If the parent is incomplete or its
   linkage has unexpectedly changed, record the blocker there and report it.
5. Finish through `sase_final`, declaring commits for every modified repository,
   including bob-plugins and plans. Keep the declaration as the final action before the
   response. Report which beads closed and the actual checks performed.
