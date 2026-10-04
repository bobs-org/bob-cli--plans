---
tier: tale
size: small
title: Finish canonical nightly documentation and land bob-cli-46
goal:
  Correct the last current-use old command name, recheck readiness, and close bob-cli-46
  with its plan marked done.
proposed_by: bbugyi200.apollo.bob-cli-46.land
bead: bob-cli-46
create_time: 2026-10-04 09:53:30
status: wip
---

- **PARENT:**
  [202610/bob_command_tree.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/bob_command_tree.md)
- **BEAD:**
  [bob-cli-46](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-46/README.md)

# Remaining work

The land audit found one remaining item from the approved command-tree epic:
`docs/obsidian-sync-exclusions.md`, Procedure step 2, currently says that `bob nightly`
runs `vault-sync`, `move-done-tasks`, `vault-sync`. The middle step must say
`task archive`, matching `src/native/nightly.rs` and the new canonical tree. This is a
current procedure, not permitted historical narrative. The rest of the epic
implementation is complete. This tale has no separate land agent: **its coder must
finish the epic closeout in this same turn**. Do not wait for this turn's own commit
SHA, push, or CI result before closing.

## Verified starting point

Read `sase bead read bob-cli-46 -r "Need the land audit and follow-up outcomes"` and its
land-audit/triage notes. All four phases are closed: `bob-cli-46.1`, `.2`, `.3`, `.4`.
The approved epic plan is `plan:202610/bob_command_tree.md`; use an audited artifact
read to consult it. There is no `parent_bead` on bob-cli-46 at the time of this audit.

The lander read every child note, actual source, main epic commits `660c171`, `b13f96c`,
`192e8b5`, chezmoi commit `53960fc0`, and plugin commit `0e6620a`. Verified: workflow
help sections, canonical task/Pomodoro groups, permanent silent aliases, OsString and
`--` preservation, help path validation, capture grammar guards, group defaults,
help/completion parity, hidden freshness seed, default labels, canonical native messages
and Git subjects, retained persisted identifiers, caller migration, and rollout
guard/shim help.

Main non-epic changes since epic creation are `e09c11b` (Task Card docs), `b188144`
(decay decision metadata), and `354b5ae` (remove freshness trial docs). They are
integrated and must remain. `origin/master` was fetched and matched HEAD `192e8b5`. In
linked repos, chezmoi `06eb7965` only regenerates the memory index; bob-plugins
`c8ec83f` already mirrors the canonical navigation comments and notice into source
fragments and generated output; `5b476ad` noteReady performance changes do not affect
command callers. Recheck later drift rather than redoing this completed work. Open any
other repo with `/sase_repo` before reading it, and do not edit generated plugin main.js
directly.

Current verification: `just fmt` passed; `cargo test --lib runner::` passed 5/5;
`cargo test --test cli aliases -- --test-threads=4` passed 13/13. Installed
`bob task --help`, `bob pomodoro tmux --help`, `bob pomodoro notify --help`,
`bob_notify --help`, and `tmux_bob_pomodoro --help` passed. No epic symbols were listed.
The linked bob-plugins `npm run validate` also passed: generated sources match and all
six manifests are valid. The checkout lacks `just check` and `just symvision` recipes.
Clippy's only error is the pre-existing `|| true` assertion in
`tests/cli/capture/pomodoro_name.rs:808`, owned by active epic bob-cli-28. Do not fix
those unrelated infrastructure issues in this tale and do not run `just check-full`.

## Follow-up triage already completed

Carry **every outcome** into the epic close note; do not create duplicate tasks. The
lander used `/sase_new_task`, all-status and same-type searches, the recent task sweep,
and active-epic checks. Ten proposal entries reduce to eight issues:

- `bob-cli-46.1` note #1, `bob-cli-46.2` #2, and `bob-cli-46.3` #6/#8: clippy deny was
  independently reproduced on 192e8b5 and recorded as a `DISCOVERED ISSUE` on its causal
  active epic bob-cli-28; no new task.
- `bob-cli-46.2` #3: BOB_DAY_FILE parallel flake corroborated existing root bug
  bob-cli-2e and flake bob-cli-40 with phase attribution and source evidence.
- `bob-cli-46.4` #1: stage-ranker timing flake corroborated existing bob-cli-3w with the
  phase's two failures, 19.75/29.20 ms, and isolated pass.
- `bob-cli-46.3` #1: new ready bug bob-cli-49, **large**, for vault-sync's no-argument
  notifier and ignored exit status. No-arg notify exits 2 on the current binary and
  identical caller code existed on pre-epic fc438bc. Merely supplying sleeps would hang
  synchronous sync in an endless Pomodoro loop; this is separately planned work,
  explicitly deferred by the epic.
- `bob-cli-46.3` #2: new ready feature bob-cli-4a, **large**, JSON option policy.
- `bob-cli-46.3` #3: new ready feature bob-cli-4b, **large**, noun policy.
- `bob-cli-46.3` #4: new ready feature bob-cli-4c, **medium**, native hand-help style.
- `bob-cli-46.3` #5: new ready memory task bob-cli-4d, **small**, proposed command-tree
  decision record; explicit memory authorization remains required.

No proposal was declined. Additional infrastructure gap `just check` was corroborated on
existing bob-cli-3c. The exact chezmoi stale-skill removal list was recorded as
implementation evidence on bob-cli-45; that task stays open because all hosts were not
independently reverified. No duplicate was filed.

## Steps

1. **Correct the remaining wording.** In Procedure step 2 of
   `docs/obsidian-sync-exclusions.md`, replace the present-tense nightly middle step
   `move-done-tasks` with `task archive`. Check README and docs for any other
   current-use command spelling missed by the original residual sweep:
   `task-status-hooks|task-status-setter|mark-next-tasks|move-done-tasks|bob randomize|tmux-pomodoro|bob notify`.
   Preserve aliases/parity coverage, persisted IDs, internal filenames/modules,
   historical narrative, single "formerly" notes, fallback binary names, and
   `vault_sync::run_notify`, which the approved plan explicitly leaves alone. Do not add
   a test that merely pins this sentence.

2. **Revalidate the completed scope and any new drift.** Audited-read each child if its
   notes changed; confirm all named phases and descendants are complete. Review commits
   after 192e8b5 and any newly fetched base changes, integrating only changes that need
   this epic's canonical command tree. Verify the linked approved epic plan's
   requirements are satisfied. Run `just check` if it exists; if it still fails solely
   because the recipe is absent, record bob-cli-3c and use `cargo fmt --check` plus
   `cargo test --test cli help:: -- --test-threads=4` for the small documentation
   change. Do not rerun the full suite without a concrete new concern. If a pre-existing
   clippy or environment flake is encountered, use the owners above, keep it distinct
   from epic work, and do not claim a green unavailable gate. Do not run
   `just check-full`.

3. **Finish bob-cli-46's landing yourself.** Run `sase bead epic-symbols bob-cli-46`
   immediately before closing. For every listed entry, resolve it under the Symvision
   policy (wire up, privatize, non-test pragma, or delete), or re-key its Justfile line
   only to a verified still-open later bead that needs it. Do not leave a stale
   exemption or this judgment for another agent. Close normally with
   `sase bead close bob-cli-46 --note "<verification and complete triage outcomes>"`.
   The note must state the source/commit/child-note and drift checks above, the
   corrected exclusion procedure, actual validation results and known limitations, all
   eight proposal outcomes, bob-cli-3c and bob-cli-45 evidence, and whether any
   additional proposals were declined with reasons. If close names leftover symbols,
   clean them and retry. If close names an unfinished phase/descendant, finish or reopen
   it and complete it normally; never force merely to succeed and never force a
   successful nested landing. Only an intentional canceled/superseded outcome may use
   `--force --reason ... --resolution canceled|superseded`. After successful close, run
   `just symvision` when available; otherwise record that the recipe is absent and
   recheck the epic-symbol list is empty. Open the plans repo via
   `sase repo open plans -r "Mark the verified command-tree epic plan done"` and set
   **`status: done`** in the frontmatter of the PLAN file shown by the bead read
   (`plan:202610/bob_command_tree.md`, file `202610/bob_command_tree.md` under the
   printed plans checkout). Do not alter its accepted scope. Then run
   `sase bead read bob-cli-46 -r "Need the parent link"`. No parent is expected. If a
   parent was added, inspect its concrete ID: for a phase, verify this child satisfies
   its work, close only that phase normally, and leave its containing epic to its
   waiting lander; for a plan, review the earlier land note, all descendants and notes,
   linked plan, and post-child drift, rerun readiness checks, retire its epic symbols,
   close normally, run symvision if present, mark its plan done, and repeat only through
   fully complete directly parented plan ancestors. Stop at the first ambiguous or
   incomplete parent, note its blocker, and report it. Submit `/sase_final` for all
   changed repos as the last action before ending the turn.

## Acceptance

The current exclusion procedure teaches `task archive`; no disallowed current old
spellings remain; validation evidence is honest about unrelated failures; bob-cli-46 is
normally closed, its epic-symbol list is clear, its linked plan is `status: done`, and
every proposed follow-up has its recorded outcome.
