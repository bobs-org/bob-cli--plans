---
tier: tale
title: Preserve duplicate Work Log entries in the Mac close card
goal: Close bob-cli-2z with every typed and hand-written Work Log entry visible.
size: small
proposed_by: bbugyi200.athena.bob-cli-2z.land
bead: bob-cli-2z
status: done
---

- **PARENT:**
  [202609/close_work_log_entries.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202609/close_work_log_entries.md)
- **BEAD:**
  [bob-cli-2z](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-2z/README.md)
- **AGENTS:**
  - [bbugyi200.athena.bob-cli-2z.land](https://github.com/bobs-org/bob-cli--agents/blob/main/sessions/bbugyi200.athena.bob-cli-2z.land.md)
- **COMMITS:**
  - [0f5def1](https://github.com/bobs-org/bob-mac-capture/commit/0f5def1ab90e9279dd7075294c55c3569a28ef7a)
    — fix(mac-capture): preserve duplicate Work Log rows in close card by occurrence
    count

# Preserve duplicate Work Log rows in the Mac close card, then land bob-cli-2z

## Context

Epic `bob-cli-2z` implements typed Work Log entries in `=x` closes. Its phases
`bob-cli-2z.1` through `.4` are closed. The bob-cli engine and grammar are on master in
`8653c67` and `c7ce096`; Bob Mac Capture presentation is on master in `6a0263d`. The
approved epic plan is `plan:202609/close_work_log_entries.md` (read with
`sase artifact read`).

The Mac close card promises typed entries first, followed by up to two other entries
from `work_log`. In `Sources/CaptureCore/CapturePomodoroClosePresentation.swift`,
`taskRow` currently builds `Set(task.typedWorkLog)` and filters every matching `workLog`
string. If a user already hand-wrote the same dated text as one typed entry, both copies
disappear from the remainder. For example,
`workLog = ["*2026-09-28* — Same", "*2026-09-28* — Same"]` and
`typedWorkLog = ["*2026-09-28* — Same"]` should present one typed row and one other row,
but currently presents only the typed row. This gap was found in the epic landing audit
and is caused by the Mac phase.

The two phase proposals about the unrelated parallel `BOB_DAY_FILE` test race
(`bob-cli-2z.1` #1 and `bob-cli-2z.2` #1) were triaged before this plan: existing task
`bob-cli-2e` received a +1 and the epic records the outcome. The MacBook install was
explicitly best effort; phase `.4` contains Bryan's checklist. No `--epic-symbol`
entries were present at audit time, but check again immediately before close.

## Work

1. Open the linked `bob-mac-capture` repo through `/sase_repo` and read its
   instructions. In `CapturePomodoroClosePresentation.swift`, subtract typed entries by
   occurrence count, preserving the order of the remaining `workLog` entries. Keep the
   existing defensive fallback for formatting differences by matching one normalized
   entry per still-unmatched typed entry. If an entry has no match, leave other entries
   visible. Preserve the two-row cap on non-typed entries and the uncapped typed rows.
2. Add a focused `CapturePomodoroClosePresentationTests.swift` case with one typed entry
   and one hand-written entry having identical dated text. Assert one typed and one
   other preview. Also cover repeated typed entries and the existing formatting fallback
   if the chosen implementation changes it.
3. Run the Mac repository's relevant Swift tests and its documented format/lint check.
   Run `just check` in bob-cli for file-change verification; do not run
   `just check-full`. Confirm the later freshness commits `32d7007` and `3cd4d44` still
   use the close planner correctly.
4. Finish this landing in the same coding turn. Run `sase bead epic-symbols bob-cli-2z`
   and resolve every listed entry or re-key it only to a still-open bead that genuinely
   needs it. Close with
   `sase bead close bob-cli-2z --note "Verified the engine, grammar, Mac presentation including duplicate Work Log rows, rollout notes, and post-start freshness integration; triaged both BOB_DAY_FILE proposals to bob-cli-2e."`.
   Do not use `--force` merely to make close succeed. Run `just symvision` afterward.
   Set `status: done` in the frontmatter of the epic plan
   `plan:202609/close_work_log_entries.md` using `/sase_repo` to open the plans
   repository before editing. `bob-cli-2z` has no parent bead in the audit; re-read it
   after close and handle any parent link according to the land instructions if one
   appears. Declare commits for every modified repository through `/sase_final`.

## Acceptance

- Identical typed and hand-written dated entries both appear in the Mac close card, one
  in each presentation group.
- Other typed entries remain uncapped; other Work Log entries remain capped at two, in
  source order, and older Bob JSON still decodes with empty fields.
- `bob-cli-2z` is closed without force, `just symvision` passes, and its linked plan
  frontmatter says `status: done`.
