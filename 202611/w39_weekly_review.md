---
tier: tale
title: Draft, save, and commit the 2026-W39 Review (Sep 21 – 27)
goal:
  ~/bob/2026/2026-W39.md has a `## Review` drafted through `bob periods draft` that
  follows the note's weekly v1 callout, the brief reports `closed · drafted`, the note
  is committed with no other vault change, and bead bob-cli-gr.5 is closed with the
  verification evidence.
size: small
proposed_by: bbugyi200.athena.bob-cli-gr.5
proposed_with: claude/opus
bead: bob-cli-gr.5
create_time: 2026-11-03 20:03:10
status: wip
---

- **PARENT:**
  [202611/weekly_reviews_w43_to_w27.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/weekly_reviews_w43_to_w27.md)
- **BEAD:**
  [bob-cli-gr.5](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-gr/bob-cli-gr.5.md)

# Plan: Draft, save, and commit the 2026-W39 Review (Sep 21 – 27)

## Context

This is phase **w39** (bead `bob-cli-gr.5`) of the epic
`plan:202611/weekly_reviews_w43_to_w27.md` ("Draft the weekly Reviews for 2026-W43 back
through 2026-W27"). The epic's **Shared drafting contract** governs this work, and the
`> [!agent]- Agent instructions · weekly v1` callout inside `~/bob/2026/2026-W39.md` is
the final authority. The only vault file this phase may change is
`~/bob/2026/2026-W39.md`, and only its `## Review` section, which `bob periods draft`
writes. No bob-cli repository file changes.

State verified while planning (2026-11-03):

- `bob periods brief 2026-W39 -f md` opens with
  `# brief v1 · 2026-W39 · 2026-W39 · Sep 21 – 27 (closed · empty)`, and the vault work
  tree was clean.
- Facts: Worked 37h 20m (5h 20m above the typical week of 32h 00m); Done 22 (30 below
  the typical 52); Created 84 (28 below 112); Review closed `–`. Time:
  `sase · 30h 26m (82%)` is the only root parent at ≥10%. `unlinked · 5h 05m (14%)` is
  not a root parent. The `(type)` rows are not root parents. No `^prj` completed this
  week.
- The candidate Review below passed `bob periods draft 2026-W39 -n -f json -i -` with
  `{"ok":true,"review_state":"drafted","words":192}`. It was a dry run through stdin, so
  nothing was written.

### Facts checked against the vault (keep these; don't re-derive them unless the brief changed)

- **Done = 22 is exact.** Tasks with `[completion:: 2026-09-21..27]` total 28. Six of
  them are recurring `done/gtd_daily_done.md` rows, which leaves 22. All seven brief
  Finished rows are among those 22. The brief prints no `+N` figure, so the draft uses
  the sibling phases' form, `+more of 22 Done`.
- **Archived link targets.** The brief prints `[[sase#^add-muse-to-pools]]`,
  `[[sase#^muse-usage]]`, and `[[bob#^capture-and-start]]`, but those block ids now live
  only in `done/sase_done.md` (lines ~824–825) and `done/bob_done.md` (~271). The
  brief's links are dead, so the draft uses the `done/...` targets, which resolve.
- Every other link target was confirmed to exist: `sase_goals#^research`,
  `sase_decks#^research`, `sase_decks#^read-research`,
  `ref/chat/sase_vs_hermes_agent#^ref`, `sase_better_tasks#^bead-triage-job`,
  `done/sase_done#^axe-ace-rename` / `^tools-e2` / `^fix-plan-approve` /
  `^fix-agents-tab-jitter`, `2026/20260925#^gtd`, `gtd_daily`, `sase`, `2026/20260921`,
  `2026/20260926`.
- The two quotes are verbatim. `2026/20260925.md` line 24 reads
  `Relaunch [[sase]] agents that failed over night.` and `2026/20260926.md` line 24
  reads `Re-launch failed agents!`.
- **Still open is omitted entirely.** None of its items can be shown to hold at the end
  of the week:
  - The six `No linked minutes` bob tasks were created between 2026-10-27 and
    2026-11-03, after W39. `Disney Movie Night` is a 2026-11-03 calendar event.
  - At the last vault commit before the week ended (`b54afe777`, 2026-09-27 18:15), none
    of the four `Blocked projects` (`sase_better_plans`, `sase_goals`, `sase_projects`,
    `sase_remote`) had `#prj_is_blocked`, a `dependsOn`, or a future `scheduled` date on
    its `^prj` line.
  - `sase_memory_plan.md`, home of both `kept 3×` refs, did not exist at that commit.
- Review state agrees with itself: `Review closed: –`, every Parts row says
  `not reviewed`, and both caveats list all seven days. There is no figure conflict to
  report.
- Task minutes in Time and Finished are copied as the brief displays them. The brief
  credits each linked task with the whole session, so these figures are unsplit; that is
  the existing `bob-cli-gr.3` follow-up #1.

## Implementation

Run everything in the foreground. Write nothing inside `~/bob` by hand.

1. **Check.** Run `bob periods brief 2026-W39 -f md` and read all of it.
   - If the header does not say `(closed · empty)`, stop. Report the state and do not
     draft. Never pass `-F/--force`.
   - If the Facts, Time `sase` row, Finished list, or Parts percentages differ from the
     figures above, re-derive the affected lines from the new brief before step 2. Never
     paste stale numbers. Otherwise use the text below verbatim.
2. **Compose.** Write the Review text below to a file in a fresh directory outside the
   vault, for example `d=$(mktemp -d); $d/w39_review.md`:

   ```markdown
   **Headline:** A 37h 20m week, 5h 20m above typical, went 82% to sase, led by research
   reading, while Done ran 30 below typical.

   **Where the time went:**

   - [[sase]] · 30h 26m (82%): [[sase_goals#^research|research reading]] at 4h 10m, a
     [[sase_decks#^research|research-agent launch]] at 1h 20m, and the
     [[sase_better_tasks#^bead-triage-job|bead-triage job]] at 35m, plus the
     [[done/sase_done#^axe-ace-rename|schedule/routine/job/tui rename]], the
     [[done/sase_done#^tools-e2|E2 tools epic launch]], and a
     [[done/sase_done#^fix-plan-approve|`sase plan approve` fix]].

   **Finished:**

   - [[sase_goals#^research|Read research!]] · 4h 10m
   - [[sase_decks#^research|Launch research agent!]] · 1h 20m
   - [[sase_decks#^read-research|Read agents_tab_decks_and_cards]] · 25m
   - [[ref/chat/sase_vs_hermes_agent#^ref|sase vs Hermes Agent reference]]
   - [[done/sase_done#^add-muse-to-pools|muse code in `@medium` alias pools]]
   - [[done/sase_done#^muse-usage|muse code usage windows]]
   - [[done/bob_done#^capture-and-start|`@file+id=[<M>-<N>]` capture syntax]]
   - +more of 22 Done, including an
     [[done/sase_done#^fix-agents-tab-jitter|Agents-tab jitter fix]]

   **Questions:**

   - Worked ran 5h 20m above typical and Done 30 below, while 5h 05m stayed unlinked,
     38% of [[2026/20260921|Mon Sep 21]] — what filled those hours?
   - All seven days show not reviewed, and five [[gtd_daily]] rows were dropped, such as
     [[2026/20260925#^gtd|Fri 25]] after 25m — what happened to the daily review?
   - [[2026/20260925|Fri 25]] logged "Relaunch [[sase]] agents that failed over night."
     and [[2026/20260926|Sat 26]] "Re-launch failed agents!" — what set off the
     relaunches?
   ```

3. **Dry-run, then save.** Run `bob periods draft 2026-W39 -i "$d/w39_review.md" -n`.
   Fix anything it reports, then run it again without `-n`.
4. **Verify.**
   - `bob periods brief 2026-W39 -f md | head -1` now ends `(closed · drafted)`.
   - `git -C ~/bob diff -- 2026/2026-W39.md`, or
     `git -C ~/bob show <sync commit> -- 2026/2026-W39.md` if sync already committed,
     touches only the lines under `## Review`, plus the `review_digest` /
     `review_drafted` frontmatter keys that `bob periods draft` writes itself. Earlier
     phases saw the same keys.
5. **Commit only this note, without leaking a plan archive.**
   - Phases w43 and w42 ran `/sase_git_commit` in `~/bob`, and that committed this
     tale's plan archive (`sdd/plans/202611/wNN_weekly_review.md`) into the vault.
     Phases w41 and w40 let vault auto-sync commit the note within seconds and avoided
     the leak.
   - So: run `git -C ~/bob status --short` and
     `git -C ~/bob log -1 --format='%h %s' -- 2026/2026-W39.md`. If the note is clean
     and the latest commit is a `vault(<host>): … 2026/2026-W39.md` sync commit carrying
     the new Review, do not commit anything. Record the sync commit hash.
   - If the note is still dirty, wait in the foreground (`sleep 30`, then check again,
     at most 6 times).
   - Only if it is still uncommitted after that, commit `2026/2026-W39.md` alone with
     `/sase_git_commit`, as `~/bob/AGENTS.md` describes. Pass `-x <path>` for every
     other pending path, and leave unrelated pre-existing changes alone. If that commit
     adds an `sdd/plans/...` archive, don't delete it. Record it in step 6 for the land
     agent.
6. **Bead bookkeeping (bead `bob-cli-gr.5`).**
   - Record this new follow-up:
     `sase bead note bob-cli-gr.5 'PROPOSED FOLLOW-UP: bob periods brief Finished prints source-note link targets for tasks archived into done/ notes — the W39 brief printed [[sase#^add-muse-to-pools]], [[sase#^muse-usage]], and [[bob#^capture-and-start]], whose blocks live only in done/sase_done.md and done/bob_done.md, so the printed links are dead; the draft used the done/ targets. Sibling of bob-cli-gr.1 follow-up #1 (linked_minutes ignores archive_path).'`
   - Add a second note with only corroborating evidence (not a new follow-up):
     - The W39 `No linked minutes` list held six tasks created 2026-10-27 or later, plus
       a 2026-11-03 event. This corroborates gr.1 #4 and gr.2 #1.
     - None of the four Blocked projects was blocked at week-end commit `b54afe777`
       (gr.2 #1).
     - The `kept 3×` refs' note did not exist at that commit (gr.1 #2).
   - Add any other new follow-up only if execution uncovers one.
   - Run `sase bead epic-symbols bob-cli-gr.5`. Resolve or re-key any leftover entries.
   - Close with
     `sase bead close bob-cli-gr.5 --note "<brief header line>; <words> words; <commit hash and whether sync or sase_git_commit made it>; diff touches only ## Review plus review_digest/review_drafted; Still open omitted (unverifiable at week end)"`.
   - Do not close the parent epic `bob-cli-gr` or any other bead.
7. **Reply.** Give the Headline, the word count `bob periods draft` reported, and every
   omission or deviation:
   - Still open was omitted, for the reasons above.
   - `unlinked` 14% is not a root parent, so it appears in a question rather than a
     bullet.
   - Three Finished links use `done/` targets instead of the brief's dead links.
   - The form is `+more of 22 Done` because the brief prints no `+N` figure.
   - Task minutes are unsplit, as displayed.

## Verification

- `bob periods brief 2026-W39 -f md | head -1` reports `closed · drafted`.
- `git -C ~/bob log --oneline -3 -- 2026/2026-W39.md` shows exactly one new commit
  carrying the Review. `git -C ~/bob status --short` shows no W39 change left. No other
  vault path changed through this phase, except an archive leak recorded in step 6.
- W38, W40, W44, and the `2026-Q3` note are unchanged by this phase.
- Bead `bob-cli-gr.5` is closed with the evidence note. `bob-cli-gr` and later phases
  stay open.
