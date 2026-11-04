---
tier: tale
size: small
title: Draft, save, and commit the 2026-W29 Review (Jul 13 – 19)
goal:
  '`~/bob/2026/2026-W29.md` has a `## Review` drafted through `bob periods draft` that
  follows the note''s "weekly v1" instructions. The brief reads `closed · drafted`, only
  that note is committed, and phase bead bob-cli-gr.15 is closed.'
proposed_by: bbugyi200.athena.bob-cli-gr.15
proposed_with: claude/opus
bead: bob-cli-gr.15
create_time: 2026-11-03 21:51:36
status: wip
---

- **PARENT:**
  [202611/weekly_reviews_w43_to_w27.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/weekly_reviews_w43_to_w27.md)
- **BEAD:**
  [bob-cli-gr.15](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-gr/bob-cli-gr.15.md)

# Plan: Draft the 2026-W29 Review

## Context

This is phase `w29` (bead `bob-cli-gr.15`) of epic `bob-cli-gr`, whose plan
(`plan:202611/weekly_reviews_w43_to_w27.md`) holds the **shared drafting contract**. The
contract and the `> [!agent]- Agent instructions · weekly v1` callout inside
`~/bob/2026/2026-W29.md` are the authority. Only `2026/2026-W29.md` in the `~/bob` vault
changes. The bob-cli repository does not change.

What planning verified on 2026-11-03:

- `bob periods brief 2026-W29 -f md` opens with `(closed · empty)`. `## Intentions` and
  the brief's Intentions block are empty, so the draft has no **Intentions** part.
- Brief facts: Worked 27h 55m (1h 40m below the typical week of 29h 35m), Done 60 (9
  below typical 69), Created 80 (13 below typical 93), Reviewed `–`, and Review closed
  `–`.
- Time: sase 23h 29m (84%), bob 2h 16m (8%), and gtd_daily 1h 58m (7%). Only sase
  reaches 10%. `(unnamed) (type)` is a type row, not a root parent.
- Finished (capped at 7): `sase_onboard#^prj` (completed 2026-07-16),
  `sase_gate#^sase-gate` · 4h 55m (completed 2026-07-18), gtd rows for Jul 16, 18, and
  15, and `sase_better_plans#^custom-family-members` · 35m and `^validate` · 30m (both
  completed 2026-07-16).
- The source code (`src/native/periods/brief.rs`, `still_open_section`) confirms that
  every `## Still open, as of today` row reflects the vault today. That covers today's
  Next/WIP lanes, blocked `^prj` lines, and `[keeps:: N]`. None of it describes the end
  of W29, so the draft leaves **Still open** out. This matches the other drafted Q3
  weeks.
- These link targets exist in the vault: `sase_gate#^sase-gate`,
  `sase_better_chops#^read`, `sase_better_macros#^better-name-ow`, `sase_onboard#^prj`,
  `sase_better_plans#^custom-family-members`, `sase_better_plans#^validate`, and
  `2026/20260715#^gtd`, `2026/20260716#^gtd`, `2026/20260718#^gtd`. The quotes "Started
  `research.d` swarm!" and "Started `research.e` swarm!" appear verbatim in
  `2026/20260716.md`.
- `bob periods draft 2026-W29 -i <file> -n -f json` accepted the draft below: `ok: true`
  and `words: 155`, under the 250 cap. The Headline has 22 words, under the cap of 25.

Brief inconsistencies that planning left out and did not reconcile. Report them in the
reply:

1. Under `## Time`, gtd_daily's root total of 1h 58m is smaller than its listed rows (2h
   00m, 1h 50m, and 1h 00m). The draft therefore lists the gtd rows without minutes.
2. `## Caveats` lists all seven days as "Not reviewed" and as review-journal gaps, but
   gtd_daily rows were completed Wed 15, Thu 16, and Sat 18. The draft makes no claim
   about review.
3. The Dropped links `[[sase#^cmd-steps]]`, `[[sase#^fix-fix-just]]`, and
   `[[sase#^nested-clone-ctx]]` point to blocks that now live in `done/sase_done.md`.
   They are stale targets, so the draft does not use the Dropped rows.
4. The Tue 14 gtd row (2h 00m under `## Time`) does not appear under `## Finished`. The
   draft does not mention it.

## Steps

Run every vault command against `~/bob`. Never pass `-F/--force`.

1. **Check.** Run `bob periods brief 2026-W29 -f md` and read all of it. If the header
   no longer says `(closed · empty)`, stop without drafting. Record the state with
   `sase bead note bob-cli-gr.15 '<state found>'`, close the bead (step 8), and report.
   If any figure used below has changed since planning, update the draft to match the
   brief exactly before saving.
2. **Write the draft** to a scratch file in a fresh `mktemp -d` directory outside the
   vault, with this exact text:

   ```markdown
   **Headline:** A 27h 55m week, 1h 40m below typical, went 84% to sase and finished the
   sase onboarding project and the `/sase_gate` command.

   **Where the time went:**

   - [[sase]] · 23h 29m (84%): [[sase_gate#^sase-gate|the `/sase_gate` command]] (4h
     55m), [[sase_better_chops#^read|reading the lumberjack chop config improvements]]
     (3h 00m), and [[sase_better_macros#^better-name-ow|better agent-name overwrites]]
     (1h 40m).

   **Finished:**

   - [[sase_onboard#^prj|The sase onboarding project]]
   - [[sase_gate#^sase-gate|The `/sase_gate` command]] · 4h 55m
   - [[sase_better_plans#^custom-family-members|Custom `improve_plan` and `tester` family members removed]]
     · 35m
   - [[sase_better_plans#^validate|The `sase plan validate` command]] · 30m
   - [[gtd_daily]] rows for [[2026/20260715#^gtd|Wed 15]],
     [[2026/20260716#^gtd|Thu 16]], and [[2026/20260718#^gtd|Sat 18]]
   - +more of 60 Done

   **Questions:**

   - Done was 60 and Created 80, both below typical, while
     [[sase_gate#^sase-gate|`/sase_gate`]] took 4h 55m — did fewer, longer tasks shape
     the week?
   - [[sase_better_chops#^read|Reading the lumberjack chop config improvements]] took 3h
     00m — what did it settle?
   - [[2026/20260716|Thu 16]] logged “Started `research.d` swarm!” and “Started
     `research.e` swarm!” — what did those swarms produce?
   ```

   The file must start with the `**Headline:**` line. No line may start with `#` or
   `---`.

3. **Dry-run.** Run `bob periods draft 2026-W29 -i <file> -n -f json`, expect
   `"ok": true` and about 155 words, and fix anything it reports.
4. **Save.** Run `bob periods draft 2026-W29 -i <file>`. It should print
   `Drafted 2026-W29 Review · <N> words · state drafted`. Edit nothing by hand: not the
   callout, other sections, frontmatter, tasks, or other notes.
5. **Verify.** Re-run `bob periods brief 2026-W29 -f md`. Its header must now read
   `(closed · drafted)`. Run `git -C ~/bob diff -- 2026/2026-W29.md` and confirm that
   only lines under `## Review` changed.
6. **Commit only this note**, following `~/bob/AGENTS.md`. Run `git -C ~/bob status`
   first and leave unrelated changes alone: don't stage, revert, or commit them. Other
   epic phases, such as W28, may be writing nearby notes. Commit `2026/2026-W29.md`
   alone with `/sase_git_commit`. If vault auto-sync has already committed it
   (`vault(<host>): 1 file - 2026/2026-W29.md`), the note shows clean, and
   `git -C ~/bob log -1 -p -- 2026/2026-W29.md` contains the new Review, don't create an
   empty commit. Report that sync committed it.
7. **Epic symbols.** Run `sase bead epic-symbols bob-cli-gr.15`. At planning time there
   were none. Resolve any entry that has appeared.
8. **Close only this phase.** Run
   `sase bead close bob-cli-gr.15 --note "<what you verified: closed · drafted header, word count, diff limited to ## Review, commit hash>"`.
   Do not close epic `bob-cli-gr` or any other bead.
9. **Reply** with the Headline, the word count from `bob periods draft`, the left-out
   **Still open** section and why, and inconsistencies 1–4 above.

## Verification

- `bob periods brief 2026-W29 -f md | head -1` ends with `(closed · drafted)`.
- `git -C ~/bob log --oneline -2 -- 2026/2026-W29.md` shows the new Review commit, and
  the commit touched only `2026/2026-W29.md`.
- No other vault file changed through this phase.
