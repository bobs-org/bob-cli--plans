---
tier: tale
title: Reverse relation-label renames
goal:
  Complete bob-cli-du.4 with previewed, atomic edge reversal, accurate mirrors, and
  verified undo.
size: medium
proposed_by: bbugyi200.apollo.bob-cli-du.4
proposed_with: codex/gpt-6-astra
bead: bob-cli-du.4
status: done
---

- **PARENT:**
  [202610/relation_complete_labels.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/relation_complete_labels.md)
- **BEAD:**
  [bob-cli-du.4](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-du/bob-cli-du.4.md)
- **AGENTS:**
  - [bbugyi200.apollo.bob-cli-du.4](https://github.com/bobs-org/bob-cli--agents/blob/main/sessions/bbugyi200.apollo.bob-cli-du.4.md)
- **COMMITS:**
  - [cba7c47](https://github.com/bobs-org/bob-cli/commit/cba7c47dada6b0f409fc2b9c6ad72ecf05e97bb2)
    — feat(relations): reverse relation-label renames with previewed edge reversal

# Reverse relation-label renames for bob-cli-du.4

## Outcome and scope

Complete only phase `bob-cli-du.4` of `bob-cli-du`:
`bob relations rename OLD NEW -r/--reverse` transforms every accepted `S —OLD→ T`
relation into `T —NEW→ S`, with an accurate dry-run, maintained mirrors, strict
refusals, and one revertable transaction. Ordinary rename keeps its existing behavior
and JSON shape.

The governing design is `plan:202610/relation_complete_labels.md`, especially RV10 and
phase `reverse-rename`; read it through `sase artifact read`. Also read the assigned
bead through `sase bead read bob-cli-du.4 -r "Need current phase scope and notes"`, and
`docs/task-relations.md` §§3–7, §16 RV8/RV10, and §17. The older
`plan:202610/relation_cancel_labels.md` supplies the shared closing-label rules.

At planning time, HEAD was `387a4fc2`. The working tree was clean, this bead was
`in_progress`, it had no previous-work notes, and `sase bead epic-symbols bob-cli-du.4`
reported no entries. Commit `847a3c53` shipped the reverse-rename _contract_, not its
implementation; the command still has no reverse flag. Phase `bob-cli-du.3` has landed
complete-label warnings and CLI surfaces. Inspect the current tree again before coding
and preserve any newer work.

This is one medium tale because it is confined to bob-cli, the behavior is already
decided, and existing parsers, edit planners, identity helpers, and the vault rewrite
transaction provide the implementation foundation. No client changes or live rollout are
required here.

## Fixed decisions and boundaries

- The epic's decisions are final: `offer_successors=yes`,
  `memory_glossary_relation=yes`, `memory_decision_record=no`. Reversal never invokes a
  closing gesture, so it neither closes tasks nor offers successors. This phase needs no
  memory edits; the authorized glossary rollout belongs to `bob-cli-du.9`. Do not edit
  any memory note or decision record.
- Work only in bob-cli. Do not migrate the real vault, deploy clients, edit chezmoi, or
  enable configuration. All fixtures use isolated scratch vaults.
- Preserve task statuses, dates, inline fields, tags, descriptions, and unrelated text.
  The only task-line addition is the required block ID on an eligible writable original
  source. Do not mint `[id::]` or freshness fields.
- Preserve the v1 relation-label journal schema and hidden-state fold. A reverse rename
  emits one ordinary `relation_label.rename` event.
- Do not edit release versions or `CHANGELOG.md`.
- Do not set bead status by hand or create beads. Record out-of-scope findings as
  `PROPOSED FOLLOW-UP:` notes on `bob-cli-du.4`.

## Implementation

### 1. Separate reverse planning from ordinary label-span replacement

`src/native/relation_labels/rename.rs` currently owns clap, report types, `plan_rename`,
`run_live`, `verify_renamed`, and rendering. It is already about 1,250 lines. Add a
focused `reverse.rs` module (split scan/edit/verification or rendering helpers further
if needed), register it in `relation_labels/mod.rs`, and expose only the small internal
types needed for integration. Keep ordinary `plan_note` and its label-span preservation
semantics intact.

Add the public `-r/--reverse` flag after `-o/--offline` in help. Explain the
transformation and give an example:

```text
bob relations rename implements "implemented by" --reverse --dry-run
```

Thread the mode through preparation and dispatch to the new planner. Keep existing
validation, catalog lookup, hidden-state transfer, inverse information, and
cancel/complete-label configuration warnings. Exact OLD==NEW remains `unchanged` for
ordinary rename; reverse permits same-key/same-spelling reversal. RV10's
existing-NEW-key merge requirement still applies, including the same-key case; test
`old old -r -m` explicitly.

`src/native/completion/tree.rs` already uses `build_cli_rename`, so the flag should flow
through that descriptor. Test that instead of introducing another option list.

### 2. Build the entire move set from one immutable snapshot

Use the existing RV2 note discovery (`capture_dependency_tasks::vault_note_paths`) and
read every relevant note strictly, including `done/` and canonical daily notes. Refuse
unreadable notes or incomplete discovery as ordinary rename does. Keep before-images and
use deterministic path/line/entry order for output and newly appended entries.

Useful existing components:

- `task_relations::parse::{links_children_of, linked_from_children_of}` and the
  accepted/malformed verdicts; skip fenced examples and non-`#task` owners.
- `note_tasks::scan` and `BlockIdLookup` for unique task identity, including closed
  tasks, duplicate IDs, and IDs belonging to non-tasks.
- `vault_links::NoteIndex` for note resolution; match archive resolution rules in
  `task_status_hooks/relations.rs` rather than accidentally resolving an archive through
  a basename fallback.
- `task_dependencies::format::canonical_link`, `note_line_ending`, and
  `has_final_newline`; `capture_block_ids::mint_block_id` and
  `collect_done::block_ids_in_markdown`.
- The previous-daily selection in `capture_task_id::previous_daily_note` and the shared
  daily-date helpers. Respect the configured clock/day file, not a guessed yesterday
  path.

Collect original OLD-key edges first; do not scan newly inserted NEW edges again.
Resolve by `(vault-relative path, block ID)`, not raw link spelling. A target must be
one unambiguous task; reject missing, ambiguous, non-task, and self endpoints before any
writes, with the existing applicable refusal or RV10's `unresolved_target` and
actionable locations. Matching malformed LINKS/LINKED FROM lines keep the existing
`malformed_links_line` refusal; malformed affected destination lines also refuse instead
of losing information.

For each original source without an ID, plan one deterministic, collision-free mint if
its note is writable. Reuse it across all of that source's moves. Refuse
`source_needs_block_id` in `done/` or the selected previous daily note. This mint is
necessary even when mirrors are disabled: the original source becomes the target of the
new authored link. Dry-run reports the prospective ID without persisting it.

Preflight reverse collisions against resolved endpoint identities. If the opposite edge
already has NEW's key, preserve that edge and classify the move as `merged`; if it has
another label, refuse `reverse_conflict`, identifying every conflicting pair. Treat the
whole set as a simultaneous graph transformation so cycles, reciprocal edges, and tasks
that are both source and target cannot depend on iteration order or silently lose an
edge. Pin these cases in planner tests.

### 3. Stage LINKS and mirror changes without unrelated rewrites

Aggregate edits per task, then use `plan_links_edit` for removal from S and addition to
T. Remove empty LINKS lines; retain the order of surviving entries; append new entries
in deterministic original scan order. Create missing lines in the established relation
slot. Canonicalize new links from their new owning note; archive references must retain
explicit `[[done/…#^id]]` spelling, including when both endpoints are archived. Check
the helper's same-note fast path and handle the RV10 archive requirement locally if
necessary.

Use stable task identities/original coordinates and a staged edit map or explicit
reparse between edit stages. Inserts/removals must not shift later edits onto the wrong
task. Preserve CRLF, final-newline state, indentation conventions, and all bytes outside
the affected relation lines and required ID insertions.

Project LINKED FROM for affected endpoints from the _final_ graph via
`plan_linked_from_line`, using the §6 ordering and uncertainty rules. Preserve unrelated
incoming links and uncertain existing mirrors; do not invoke full task reconciliation,
which could also change statuses and fields. Only writable notes receive mirror changes.
Archived/previous-daily mirrors stay untouched under RV10. With
`linked_from_lines: false`, write no mirrors; do not perform a global cleanup as part of
this command. Keep mirror additions/removals and merged-edge counts available for
presentation.

### 4. Reuse the transaction, with reversal-aware verification

Feed the prepared rewrites through `vault_rewrite::run`: preserve online vault identity
checks before sync, bounded maintenance lock, pre-sync, stale checks, offline dirty-path
refusal, atomic writes, scoped commit, and post-sync handling. The commit subject is
exactly:

```text
bob relations rename: OLD → NEW (reversed)
```

Keep one journal append and existing hidden-state precedence. Ensure its before state is
covered by rollback too, including when the host journal did not exist. The current
`ReadyPlan.post_write` returns extra commit paths only after writing them;
`snapshot_before_images` currently covers `ReadyPlan.files` only. Add the smallest
necessary transaction/caller guard for the journal before append, retain the existing
append API and schema, and test verification/commit failure without leaving a rename
event behind. Preserve concurrent external edits when rolling back rather than
overwriting bytes that no longer match this transaction's write. Avoid unrelated
transaction refactoring.

Do not reuse ordinary rename's verifier unchanged: it assumes constant LINKS entry count
and absence of OLD labels even in all mirrors. Reversal can merge edges, retain
OLD==NEW, and leave read-only historical mirrors intact. Verify exact planned file
bytes, the expected final directed graph/labels, collision deduplication, required
minted IDs, and affected writable mirror projections by rescan. Fail closed on
missing/unreadable scan inputs. Expected entry count is the original count minus actual
merges. A mismatch must restore the transaction's before-images and produce
`verify_failed`, without a successful commit.

### 5. Make preview and JSON explain the move

Extend the success report additively, keeping `schema_version: 1` and omitting
reverse-only fields for ordinary rename:

```text
reversed: true
moves[]: {
  source: {path, line, block_id, text},
  target: {path, line, block_id, text},
  from_label, to_label, action: "moved" | "merged"
}
minted_block_ids[]: {path, line, block_id}
```

Use original one-based task lines for move/mint locations and document that convention;
include planned minted IDs in endpoint identities. Retain existing files, state,
inverse, replacements, warnings, sync, and commit fields. Keep file/line previews
truthful for inserted and removed lines, not only replacements. Dry-run and apply must
describe the same planned move set on an unchanged vault.

Render the RV10 preview grouped by original source note: original task/location, the new
`T —NEW→ S` sentence, links in `done/`, merged links, minted IDs, mirrors added/removed,
and `statuses none change`. Reuse the relation catalog's mint accent and the existing
Styler/color policy on a TTY, with readable `NO_COLOR` output. Include commit/undo
information and the existing state/inverse changes. When NEW is a configured cancel or
complete label, list affected open new sources beneath:

```text
stay open  {k} new sources keep their status — a rename never closes
```

All output must reflect the planner, never independently reconstruct effects in the
renderer. Ordinary rename output remains unchanged.

### 6. Update the shipped contract

Read `cli_rules.md` and `docs.md` through `/sase_memory_read` before editing help or
docs. Mark RV10 implemented and remove only this phase's planned markers from RV7/LR7/§6
in `docs/task-relations.md`. Clarify the reverse exception in the
`docs/task-status-hooks.md` archive/read-only paragraph, while keeping reconcile
read-only there. Update `docs/reference/json-outputs.md` and document the report
fields/count/location meanings. Run `just docs-gen` for the generated CLI page; inspect
the generated diff so unrelated active-epic documentation is preserved.

## Verification and acceptance

Add focused planner tests and scratch-Git-vault CLI tests, preferably in a new
`tests/cli/relations_reverse.rs` registered by `tests/cli/main.rs`. Reuse the existing
rename test helpers where practical. Pin `BOB_DIR`, `BOB_CONFIG_FILE`, `BOB_DAY_FILE`,
`BOB_NOW`, host, cache, and lock paths; never depend on the real vault or user
configuration.

The main fixture reproduces the approved migration shape: 14 OLD edges, comprising 5
active-to-active, 2 active-to-archive, and 7 archive-to-archive; multiple entries on one
LINKS line; active mirrors; existing LINKS/LINKED FROM on destinations; and extra
unrelated relations/prose. Assert the actual fixture-derived counts.

Required assertions:

1. Dry-run writes no note, journal, or commit. Apply moves exactly the previewed edges,
   preserves unrelated bytes/statuses, appends one v1 event, and creates one correctly
   named commit. `git revert` restores original note and journal bytes, including
   absence of a newly created journal. A second run after a non-symmetric applied rename
   refuses `unknown_label` without writing.
2. Same-note and cross-note moves; explicit archive links; previous-daily LINKS movement
   with existing IDs; deletion of an emptied line; canonical slot/order; multiple moves
   touching the same note/task; CRLF and no-final-newline files.
3. Writable source minting, collision-free multiple mints, no mint on preview,
   archive/previous-daily mint refusal, and minting necessary for authored links even
   with mirrors disabled.
4. Existing-NEW merge arming; successful opposite-edge merge; different-label conflicts
   (including differently spelled links to the same resolved task); same-key reversal;
   cycles/reciprocal inputs; no self-link or duplicate identity writes. Cover malformed
   matching/affected lines and unresolved/ambiguous or non-task targets. Every refusal
   leaves bytes, journal, and HEAD unchanged.
5. Closing-label configs never change any status, completion/cancellation field,
   dependency, or ledger; the preview names the open sources that remain open. With
   mirrors disabled, no mirror writes; with mirrors enabled, affected active projections
   are correct and archived mirrors stay untouched.
6. Run reconcile on an already otherwise reconciled scratch fixture after apply; it must
   make no additional relation changes. Configure/seed it so unrelated derived-status or
   layout work does not obscure the assertion.
7. Exercise stale/verify failure and journal rollback using deterministic internal
   hooks/closures, not timing races or new public debug options. Existing
   `vault_rewrite` tests cover lock, dirty paths, non-worktree, and pre/post sync
   failures; retain them and add reverse-path coverage for vault mismatch/offline
   behavior and any changed transaction integration.
8. Test `-r`/`--reverse`, help ordering, inherited completion, JSON-only stdout, human
   move grouping/counts/undo, and `NO_COLOR`. Existing normal rename, complete-label
   warning, and relation tests must still pass.

Run focused suites first (`cargo test --lib relation_labels`,
`cargo test --lib vault_rewrite`, `cargo test --test cli relations`), then required
repository gates `just check` and `just docs-check` after docs generation. Follow
`/sase_monitor` for commands needing a long handoff and wait for handoff commands
themselves to exit. Record the concrete test results; do not claim a full gate passed
from a filtered run.

Compare any suspected unrelated failure against an unchanged clean base. If it
reproduces identically, record `PROPOSED FOLLOW-UP: ...` on `bob-cli-du.4`, with the
evidence and any existing tracking bead; do not leave the phase open for that baseline
failure. Fix regressions introduced by this work.

## Completion protocol

Before closing, run `sase bead epic-symbols bob-cli-du.4` again and resolve every
remaining symbol or re-key its Justfile line to a still-open bead. Close only
`bob-cli-du.4` with
`sase bead close bob-cli-du.4 --note "<implementation and verification evidence>"`.
Never close `bob-cli-du` or an ancestor plan bead; rollout and parent acceptance belong
to their land agents. Submit `/sase_final` as the last action before a normal final
response, as required by the host.
