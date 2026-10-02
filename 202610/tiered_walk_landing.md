---
tier: tale
size: small
title: Integrate the tiered walk into crowded.md and plan surfaces, then land bob-cli-3g
goal: "Update the two surfaces committed while epic bob-cli-3g was open so they match
  the landed tiered walk, then close bob-cli-3g in this same coding turn.

  "
proposed_by: bbugyi200.athena.bob-cli-3g.land
bead: bob-cli-3g
status: done
---

- **PARENT:**
  [202610/tiered_morning_review_walk.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/tiered_morning_review_walk.md)
- **BEAD:**
  [bob-cli-3g](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-3g/README.md)
- **AGENTS:**
  - [bbugyi200.athena.bob-cli-3g.land](https://github.com/bobs-org/bob-cli--agents/blob/main/sessions/bbugyi200.athena.bob-cli-3g.land.md)
- **COMMITS:**
  - [284522c](https://github.com/bobs-org/bob/commit/284522c4b6a619b7393876db27329820277b4a9d)
    — docs(crowded): align intro with tiered walk Commitments-done ritual

# Integrate the tiered walk, then land bob-cli-3g

This is the remaining work of epic bob-cli-3g. One coding agent finishes it and lands
the epic in the same turn. This tale has no land agent of its own. Nothing resumes this
landing after you. Do not wait for this turn's own commit SHA, push, or CI result before
closing the epic. The host commits after the turn ends.

Do not run `just check-full`. File-change verification is `just check`.

## Already verified — do not redo

The land agent read the epic, all four closed children and every child note, the plan
`plan:202610/tiered_morning_review_walk.md`, the Rust evaluator, the plugin evaluator,
and the commits.

- bob-cli-3g.1 `a9c47d6`: schema 3, lane intervals, tiered queue and counts, vectors Q1
  Q2 L1–L5 R1 R2 S14 S15 B1 and missing-created in
  `src/native/freshness/state_tests.rs`. `Snapshot.pending` and `Snapshot.next` exist in
  `src/native/freshness/scan.rs`.
- bob-cli-3g.2 plugin `cdadcde`: bob-ledger-tools 1.17.0, `api.freshness` namespace v4,
  `intervalForLine`, upkeep meter, lane marks. Top-level api stays v3.
- bob-cli-3g.3 plugin `ede89d3`: bob-navigation-hotkeys 1.50.0, tier notices,
  Commitments-done boundary, walk anchor, lane-aware refresh row.
- bob-cli-3g.4 `9e548bb` plus vault `36bda225` and `24fbd255` plus chezmoi `9a3e13f7`:
  freshness config 7/1/1, ritual, decision
  `sase/memory/decisions/review-walk-is-tiered.md`, glossary, `^wip-next-refresh` closed
  `[x]`, trial window 2026-10-05 through 2026-10-18.
- bob-cli-3f landed during the epic and is already integrated where it had to be:
  `docs/freshness.md` §6 step 4 clears CROWDED and does not depend on ROTTEN progress;
  `gtd_daily.md` keeps the `[[crowded|CROWDED]]` step and the weekly
  `` `bob ready -a` `` check; `f83b250` counts NEXT/PENDING from `Snapshot.next` /
  `Snapshot.pending` in `src/native/note_ready/scan.rs`.
- No `--epic-symbol` entries for bob-cli-3g at triage time. `parent_id` is null.
- Follow-up triage is already recorded on bob-cli-3g. Do not file another task for the
  bob-cli-3g.4 GUI verification gate, and do not add a second walk-went-live bullet to
  `rotten.md`.

## 1. crowded.md intro

Open the vault with
`sase repo open gh:bobs-org/bob -r "Align crowded.md with the tiered morning ritual"`.
Read that checkout's `AGENTS.md` and run `git status` before editing. Use only the path
the open command prints. Read
`sase memory read obsidian.md -r "Need vault edit rules before changing crowded.md"`.

`crowded.md` already has `parent` frontmatter. Change only the intro sentence under
`# Crowded Notes`. It currently says the pre-walk order:

```text
Back to the [[dash|Dashboard]]. After NEW and ROTTEN, clear CROWDED to 0 by splitting, sequencing, deferring, or dropping work.
```

Replace that sentence with:

```text
Back to the [[dash|Dashboard]]. After Commitments done (NEW → PENDING → NEXT → RETURNED), clear CROWDED to 0 by splitting, sequencing, deferring, or dropping work. This does not depend on how far ROTTEN review got.
```

Leave the `bob-ready-notes` block, the remedies table, and the Tasks query unchanged. Do
not edit `~/bob` and do not run `bob vault-sync` in this turn. Vault-sync reconciles
`~/bob` and would need this turn's vault commit to already be pushed. The host commits
the opened checkout after the turn. The athena `bob-vault-sync` service pulls that
commit into `~/bob` afterward. `~/bob` is still at `24fbd255` until then. Say that in
the close note.

## 2. docs/plan.md namespace

In this bob-cli workspace, `docs/plan.md` has two surface rows written by `447e97d`
(bob-cli-3f.5) while this epic was open. Both say `freshness namespace v3`. The landed
namespace is v4. Change only those two phrases from `freshness namespace v3` to
`freshness namespace v4`:

- the `Daily note with a bob-plan code block` row
- the `` `dash.md` `` row

Leave `api v3` as `api v3`. Leave the method lists, the CROWDED / noteReady text, and
every other row unchanged. Do not edit generated `AGENTS.md` shims or the accepted body
of `decisions/ready-is-freshness-gated`.

## 3. Verify the doc edit

Run `just check` in this bob-cli workspace and wait until it exits. Do not run
`just check-full`. A `just check` pass is enough. If `just check` fails on a
pre-existing test-infrastructure problem, say so in the close note and do not treat it
as remaining epic work. If it fails because of this edit, fix the edit and rerun
`just check`.

## 4. Close bob-cli-3g

Recheck symbols:

```bash
sase bead epic-symbols bob-cli-3g
```

There were none at triage. If the command prints any `--epic-symbol` entry, resolve it
before closing: wire it up, privatize it, add a non-test pragma, or delete it per the
Symvision epic-whitelist policy. Re-key a Justfile line to a different bead only when a
still-open later bead still needs the exemption. Do not leave that judgment for a later
agent. `sase bead close` refuses while any of these entries remain. Do not use `--force`
to get past them.

Then close the epic. The note must cover what you changed in steps 1–2 and the
verification already recorded above. Use this note, adjusted only if a symbol or a
`just check` result differed:

```bash
sase bead close bob-cli-3g --note "Verified bob-cli-3g.1 through .4 against the plan, the Rust and plugin evaluators, and commits a9c47d6, cdadcde, ede89d3, 9e548bb, vault 36bda225/24fbd255, and chezmoi 9a3e13f7. Schema 3, namespace v4, nav 1.50.0, lane intervals, ritual, decision record, and closed ^wip-next-refresh are in tree. 3f's CROWDED ritual step, gtd_daily wikilink, weekly bob ready -a check, and Snapshot.next/.pending reuse were already integrated. This turn aligned crowded.md with §6 (CROWDED after Commitments done, independent of ROTTEN) and updated the two docs/plan.md surface rows from freshness namespace v3 to v4. just check passed. No PROPOSED FOLLOW-UP entries. The 3g.4 GUI verification gate stays the recorded headless residue and was not filed as a new task. No epic-symbol entries. ~/bob remains at 24fbd255 until the next vault-sync after this vault commit is pushed. No parent bead."
```

If the close is rejected because `--epic-symbol` entries remain, finish that cleanup and
close again with the same note. If it is rejected because named phases were never
completed, finish or reopen them. Do not use `--force` merely to make the close succeed,
and do not use `--force` to advance a successful nested landing.

After a successful close, run `just symvision`.

Then set `status: done` in the frontmatter of the epic plan file. The path is the PLAN
path from `sase bead read bob-cli-3g -r "Need the plan path after close"`:
`sase/repos/plans/202610/tiered_morning_review_walk.md` (the plans repo, not this tale
file). Change only the `status:` field from `wip` to `done`.

## 5. Parent

Re-read `sase bead read bob-cli-3g -r "Need the parent link after close"`. `parent_id`
was null at triage. If it is still null, stop. Do not close any other bead.

If a parent appeared and it is a phase bead, verify this child plan completed that
phase's work, close only that phase with
`sase bead close <parent-bead> --note "<what you verified>"`, and leave the containing
epic to its land agent.

If the parent is a plan bead, review its previous landing note, descendants, notes,
linked plan, and post-child drift, and rerun descendant and linked-plan readiness checks
before closing it. When that parent plan is still complete, retire leftover
`--epic-symbol` entries first (`sase bead epic-symbols <parent-bead>`), close it
normally, run `just symvision`, mark its linked plan file done, and repeat through
directly parented plan ancestors while each remains fully complete. Stop at the first
incomplete or ambiguous parent, record a note on that parent describing the blocker, and
report it. Never use `--force` to advance a successful nested landing.
