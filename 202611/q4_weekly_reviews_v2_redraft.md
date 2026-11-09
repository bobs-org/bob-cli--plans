---
tier: epic
title: Redraft 2026Q4's v1 weekly Reviews under the v2 instructions
goal: 'Every closed 2026-Q4 week still carrying a v1 Review (W40–W44) has a v2 Review
  saved through `bob periods draft`, so the first quarterly review on 2027-01-04 reads
  work-first, one-line Headlines instead of stats-first ones.

  '
phases:
- id: redraft-w44
  title: Redraft the 2026-W44 Review under v2
  depends_on: []
  size: xlarge
  description: 'redraft-w44: prove the false `edited` state comes only from bob''s
    archive link moves, then force-redraft W44 (Oct 26 – Nov 1) under its v2 callout,
    commit only that note, and check W45''s "also carried in W44" mark against the
    new list.'
- id: redraft-w43
  title: Redraft the 2026-W43 Review under v2
  depends_on:
  - redraft-w44
  size: xlarge
  description: 'redraft-w43: redraft the drafted W43 (Oct 19 – 25) under its v2 callout
    without -F, commit only that note, and check the new W44 Review''s "also carried
    in W43" marks.'
- id: redraft-w42
  title: Redraft the 2026-W42 Review under v2
  depends_on:
  - redraft-w43
  size: xlarge
  description: 'redraft-w42: redraft the drafted W42 (Oct 12 – 18) under its v2 callout
    without -F, commit only that note, and check the new W43 Review''s "also carried
    in W42" marks.'
- id: redraft-w41
  title: Redraft the 2026-W41 Review under v2
  depends_on:
  - redraft-w42
  size: xlarge
  description: 'redraft-w41: prove the false `edited` state comes only from bob''s
    archive link move, then force-redraft W41 (Oct 5 – 11) under its v2 callout, commit
    only that note, and check the new W42 Review''s "also carried in W41" marks.'
- id: redraft-w40
  title: Redraft the 2026-W40 Review under v2
  depends_on:
  - redraft-w41
  size: xlarge
  description: 'redraft-w40: redraft the drafted W40 (Sep 28 – Oct 4) under its v2
    callout without -F, commit only that note, and check the new W41 Review''s "also
    carried in W40" marks; W39 (Q3) stays v1.'
proposed_by: bbugyi200.athena.1ac.f0
proposed_with: claude/opus
create_time: 2026-11-09 10:47:00
status: done
bead_id: bob-cli-j4
---

- **PROMPT:** [prompts/202611/q4_weekly_reviews_v2_redraft.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/q4_weekly_reviews_v2_redraft.md)
- **BEAD:** [bob-cli-j4](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-j4/README.md)

# Plan: Redraft 2026Q4's v1 weekly Reviews under the v2 instructions

## Context

The period-note agent instructions and the agent brief are now v2:

- bob-cli commit `4c09fac4` ("feat(periods)!: v2 period review instructions and brief
  v2") is installed at `~/.cargo/bin/bob`.
- Vault commit `cf0e7a67d` ("periods: refresh agent instructions to v2") refreshed every
  period note's folded callout to v2. It did not rewrite any existing Review.
- W45 was redrafted under v2 already, in vault commit `d58e4a193`. Read its `## Review`
  as one worked example of the v2 shape. Do not copy its wording or selection.

The research report behind v2 recommends redrafting Q4's weeks before the first
quarterly review, because the Q4 brief reads each week's Headline
(`research:202611/period_review_instructions_v2/period_review_instructions_v2.md`,
"Redraft old weeks" row and Rollout step 8). Bryan has now asked for it. These are the
closed Q4 weeks whose Reviews are still v1 (each opens with a stats-first Headline and
uses the v1 Finished / Still open parts):

| Phase         | Note               | Week                    | State            | v1 draft commit | `-F`                       |
| ------------- | ------------------ | ----------------------- | ---------------- | --------------- | -------------------------- |
| `redraft-w44` | `2026/2026-W44.md` | Mon Oct 26 – Sun Nov 1  | `edited` (false) | `982dc74ed`     | yes, after the check below |
| `redraft-w43` | `2026/2026-W43.md` | Mon Oct 19 – Sun Oct 25 | `drafted`        | `9d359b4f2`     | no                         |
| `redraft-w42` | `2026/2026-W42.md` | Mon Oct 12 – Sun Oct 18 | `drafted`        | `ead3ef9d8`     | no                         |
| `redraft-w41` | `2026/2026-W41.md` | Mon Oct 5 – Sun Oct 11  | `edited` (false) | `6088517df`     | yes, after the check below |
| `redraft-w40` | `2026/2026-W40.md` | Mon Sep 28 – Sun Oct 4  | `drafted`        | `8091ae2bd`     | no                         |

W41 and W44 read `edited` only because `bob task archive` moved links in their v1
Reviews into `done/` before the archive digest fix landed (vault commits `7b414a272` and
`f84e23596`). Nobody hand-edited either Review.

The phases run one at a time, newest week first, as Bryan asked. So each week's brief
carries the previous week's still-v1 Review (its Headline, Still open, and Questions) as
context. Brief v2 reads both "Carried forward" and "Still open", so the callout's "mark
any item the previous week also carried" rule still works.

## Procedure every phase follows

The target note's own callout, `Agent instructions · weekly v2` at the bottom of the
note, is the source of truth. Read it in full and follow its steps, part order, caps,
and rules exactly. Where this plan and the callout disagree, the callout wins. The only
exceptions are the `-F` permission in step 2 and the planning shape below. Bryan said
"redo" for these weeks; that is the redo the callout's step 1 names.

1. **Read the vault conventions.** Read `~/bob/AGENTS.md`, then run
   `git -C ~/bob status --short`. Leave every pre-existing change alone.
2. **Check the review state.** Find the week's row in
   `bob periods list -k week -f json`.
   - `drafted`: proceed without `-F`.
   - `edited` (expected only for W41 and W44): first prove that only bob changed the
     Review. Run `git -C ~/bob log --format='%h %ad %an %s' --date=short -- <note>` and
     `git -C ~/bob diff <v1 draft commit> HEAD -- <note>`. Inside `## Review`, the only
     changes may be the archive commit's `[[note#^block]]` →
     `[[done/<note>_done#^block]]` link moves. Outside it, the only changes may be the
     `cf0e7a67d` callout refresh and bob's own frontmatter fields. If that holds, use
     `-F`. If anything else changed, Bryan edited the Review: stop without writing and
     report the diff.
   - `reflected` (Bryan wrote this week's `## Reflection`): stop without writing and
     report it. Never force a redraft over a Review Bryan has reflected on.
   - `empty`, or the brief says the week is open: stop and report. This should not
     happen for these weeks.
3. **Read the brief.** Run `bob periods brief 2026-W<nn> -f md` and read all of it. It
   is the only source of numbers.
4. **Verify claims.** Open other notes only to check a claim: `done/*_done.md`, project
   and area notes, and daily notes (`2026/2026MMDD.md`). Check completion dates and
   every sentence of Bryan's you quote.
   - Name nothing hidden in a "+N more" tail unless a note confirms it.
   - The brief keeps no history of blocked state. Do not carry a `^prj` as "Blocked at
     week end" unless a note proves it was blocked on that Sunday.
   - Unknown is not zero.
5. **Record the old Headline.** Copy the v1 Review's whole first paragraph, joined onto
   one line, for the reply. Use the old Review only to avoid repeating its questions and
   to find evidence links. Never reuse one of its numbers without matching it to the
   brief.
6. **Draft** to a file in a fresh `mktemp -d` directory, outside `~/bob` and outside the
   bob-cli workspace.
7. **Self-check the draft** (the report's acceptance list):
   - The Headline is one physical line of at most 25 words. It does not start with
     `A {Nh Nm} week`, and it names Worked or typical only if they are the news.
   - Shipped has no `gtd_daily` or other routine row, and no item appears in two parts.
   - There are 0–2 numbered Questions. None only restates a typical-delta, and none
     repeats the previous week's Questions.
   - Every count is displayed in the brief for exactly that set.
   - The word cap of 250 is a ceiling; 120–180 words is usually enough.
8. **Dry-run** with `bob periods draft 2026-W<nn> -i <file> -n -f json`, adding `-F`
   only when step 2 allows it. Fix every reported problem.
9. **Save** with the same command without `-n`. Then confirm three things:
   - `bob periods list -k week -f json` shows the week as `drafted`;
   - its Headline reads whole;
   - `git -C ~/bob diff -- <note>` touches only `## Review` and bob's `review_digest` /
     `review_drafted` frontmatter.
10. **Commit only that note**, as `~/bob/AGENTS.md` says:
    `git -C ~/bob add <note> && git -C ~/bob commit -m "periods: redraft 2026-W<nn> review under v2"`.
    Never use `/sase_git_commit` in the vault. Edit nothing else: not the callout, other
    sections, tasks, or other notes. Change no file in the bob-cli workspace.
11. **Check the next week, and report only.** The next week's Review is already v2. For
    W44 that is W45; for the others it is the note the previous phase redrafted. It may
    mark Carried forward items "also carried in W<nn>". If such an item is not in this
    week's new Carried forward, report the mismatch. Do not edit the other note.
12. **Reply** with what the callout's step 6 asks:
    - the new Headline;
    - the word count;
    - every claim left out because it could not be verified;
    - whether this week's Intentions or the previous week's Reflection is empty.

    Also give the old one-line Headline next to the new one, the vault commit hash, and
    any mismatch from step 11.

## Planning shape for each phase

Each phase is `xlarge` because Bryan asked for that, so its worker plans before writing.
Keep that plan small: one `tale` of size `small` that carries everything Bryan needs to
approve the exact text:

- the result of the state check, plus the git evidence for W41 and W44 that justifies
  `-F`;
- the full proposed Review text, already clean under the step 8 dry run, with its word
  count;
- the old and new Headlines;
- the claims left out.

The tale's coder then repeats steps 1–2 and the dry run, saves exactly the approved
text, commits, and replies (steps 9–12). If the dry run now fails, the coder changes the
text only as much as the fix needs and says what changed. Approving the plan therefore
means approving the exact text.

## Phases

### `redraft-w44` · 2026-W44 (Mon Oct 26 – Sun Nov 1)

- The state is a false `edited`: archive commit `f84e23596` moved
  `[[job#^ref-stripe-industry-screen-prep]]` and `[[job#^stripe-assessment]]` to
  `done/job_done`. Verify that this is the only Review change since `982dc74ed`, then
  use `-F`.
- The v1 Review is the only work-first Headline of the quarter, and the v2 callout's
  "Good" example happens to be W44's own content. Still check every item against the
  brief, and drop the hours unless they are the news.
- The v1 Questions (Created vs Done; Reviewed below typical) are the typical-delta kind
  that v2 forbids. Do not carry them over.
- For step 11, W45's v2 Review marks
  `[[sase_memory_plan#^ref-docs-backed-memory-and-agent-maintained]]` (Docs as Memory)
  "also carried in W44". Report it if the new W44 Carried forward leaves it out.
- The v1 Still open lists `sase_better_plans#^prj` and `sase_goals#^prj` as blocked.
  Carry them only if a note proves they were blocked on Nov 1 (see step 4).

### `redraft-w43` · 2026-W43 (Mon Oct 19 – Sun Oct 25)

- The state is `drafted`, so no `-F`. Its previous week, W42, is still v1, and its next
  week is the W44 just redrafted.
- The v1 Headline wraps across two lines and opens with hours. Do not reuse its daily
  loop lead: `gtd_daily` is routine and never goes in Shipped.

### `redraft-w42` · 2026-W42 (Mon Oct 12 – Sun Oct 18)

- The state is `drafted`, so no `-F`. Its previous week, W41, is still v1, and its next
  week is the W43 just redrafted.
- The v1 Headline names the CV and DataBricks projects as closed. Confirm each `^prj`
  completion date in the done notes before you name it.

### `redraft-w41` · 2026-W41 (Mon Oct 5 – Sun Oct 11)

- The state is a false `edited`: archive commit `7b414a272` moved
  `[[cash#^pair-new-headphones]]` to `done/cash_done`. Verify that this is the only
  Review change since `6088517df`, then use `-F`.
- The v1 Headline says four projects closed. Name only the `^prj` completions the brief
  shows inside Oct 5–11, each confirmed in a done note.

### `redraft-w40` · 2026-W40 (Mon Sep 28 – Sun Oct 4)

- The state is `drafted`, so no `-F`. The note's parent is `2026/2026-Q4` even though
  three of its days fall in September.
- Its previous week, W39, belongs to Q3 and stays v1. Use W39's Review only as context,
  and never edit it.
- The brief reports "Review closed: 0 of 5" and no typical Reviewed week. Treat that as
  data, not as a question prompt.

## Out of scope

- W45, which is already v2; W46 and W47, which are open or future; the `2026-Q4`
  quarterly note, which stays open until 2027-01-03; W38 and W39, which belong to Q3;
  and the `2026` yearly note.
- bob-cli code, the callouts, frontmatter beyond bob's own draft fields, tasks, and
  every other vault note.
- New beads, unless a bob defect blocks a draft. In that case, file it through
  `/sase_new_task` and report it.
