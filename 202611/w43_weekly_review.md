---
tier: tale
size: small
goal:
  2026/2026-W43.md has a `## Review` drafted through `bob periods draft` per its weekly
  v1 callout (closed · drafted), committed alone, with brief defects recorded as
  follow-ups and phase bob-cli-gr.1 closed.
title: Draft, save, and commit the 2026-W43 weekly Review (phase bob-cli-gr.1)
proposed_by: bbugyi200.athena.bob-cli-gr.1
proposed_with: claude/opus
bead: bob-cli-gr.1
create_time: 2026-11-03 19:18:01
status: wip
---

- **PARENT:**
  [202611/weekly_reviews_w43_to_w27.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/weekly_reviews_w43_to_w27.md)
- **BEAD:**
  [bob-cli-gr.1](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-gr/bob-cli-gr.1.md)

# Plan: Draft the 2026-W43 Review (Mon Oct 19 – Sun Oct 25, 2026)

## Context

This is phase `bob-cli-gr.1` (slug `w43`) of epic `bob-cli-gr`
(`plan:202611/weekly_reviews_w43_to_w27.md`). It is the first link in Bryan's
newest-first chain W43 → W27. The phase writes exactly one thing: the `## Review`
section of the Bob vault note `~/bob/2026/2026-W43.md`. It goes through
`bob periods draft` and is committed alone. No file in the bob-cli repository changes.
The epic has no DECISIONS block, and no memory note changes.

The authority is the `> [!agent]- Agent instructions · weekly v1` callout inside the
note. The epic's "Shared drafting contract" (steps 1–8) restates it. Read both before
acting. If they disagree, follow the callout and report the difference.

### Verified while planning (2026-11-03)

- `bob periods brief 2026-W43 -f md` opens with
  `2026-W43 · Oct 19 – 25 (closed · empty)`, and `git -C ~/bob status` was clean.
  Nothing from an earlier run had landed.
- Facts (copy verbatim): Worked 25h 40m (9h 08m below the typical week of 34h 48m);
  Reviewed 362; Done 115 (20 below the typical week of 135); Created 133 (9 above the
  typical week of 124); Review closed 6 of 7.
- Root parents at ≥10% of Worked: `gtd_daily` 8h 48m (34%), `sase` 8h 31m (33%), and
  `bob` 5h 20m (21%). `sase_blog` at 9% and the `PLAN/READ/BUILD (type)` rows are out.
- Parts: every day reads `not reviewed`, meaning the daily note has no Review. That is a
  different measure from `Review closed`, which counts the daily GTD closeout: six days
  `met` and Sat 24 `missed`, with Sat at `gtd_daily 52%` of 4h 50m. Use
  `Review closed 6 of 7` and the Sat closeout. Do not use the "not reviewed" caveat as a
  contradiction.
- **The `bob` root lists no tasks in the brief, which is a brief defect.** Its Pomodoros
  link `[[done/bob_done#^…]]`, but `linked_minutes` in `src/native/periods/brief.rs`
  matches a link target only against `IndexedTask.source_path` (`bob`). It ignores
  `archive_path` (`done/bob_done.md`). So archived tasks get 0 linked minutes in both
  Time and Finished. The week's Pomodoro sections (2026/20261019–25) were opened only to
  name what bob's time bought. All of these tasks are in `done/bob_done.md` with
  completion dates inside the week: `^split-highlights` (Oct 19), `^today-marker` (Oct
  19), ref-panel and Highlights tasks (`^full-ref-panel`, `^just-added-ref`,
  `^previous-ref-panel-loc`, `^cmd-t`, `^section-highlights`), `^rotten-log` (Oct 25),
  `^cmd-p` (Oct 25), `^add-daily-statistics` (Oct 25), and `^root-parent` (Oct 25). No
  bob minutes appear in the Review, because the brief has none.
- **The brief's "Kept 3+ times" items fail the week-end check.** In
  `sase_memory_plan.md`, both `^ref-docs-backed-memory-and-agent-maintained` and
  `^ref-from-agent-behaviour-to-agent-friendly` reached `kept 2×` on 2026-10-30 and
  `kept 3×` on 2026-11-02, both after W43. Leave them out.
- **Three of the brief's four "Blocked projects" are newly Blocked in W43.** Vault git
  pickaxe (`git log -S'- [?] #task #prj …'`) shows when each `^prj` line became `[?]`:
  `sase_projects` on 2026-10-20 and `sase_better_plans` and `sase_remote` on 2026-10-21.
  Each line has stayed `[?]` since. `sase_goals` turned Blocked on 2026-10-27, which is
  W44, so leave it out.
- The "No linked minutes" items are Next/WIP **as of today**. One is a 2026-11-03
  calendar event. Their state on Oct 25 cannot be verified, so leave them all out.
- Done (115) excludes recurring tasks and daily `^gtd` Task Links (stats registry). The
  brief's Finished list includes three `^gtd` rows. So `+N more` cannot be derived from
  brief figures without inference. The draft therefore writes `+more of 115 Done`
  instead of an invented N.
- The candidate draft below passed `bob periods draft 2026-W43 -i <file> -n -f json`:
  `ok: true`, `words: 233`, `review_state: drafted`. The vault stayed untouched.

## Steps

1. **Check.** Run `bob periods brief 2026-W43 -f md` and read all of it. If the header
   does not say `closed · empty`, stop: do not draft, and report the state. Never pass
   `-F/--force`. Compare every number you will copy with the Recommended draft below. If
   any brief figure changed, update the draft to match the brief exactly. Run
   `git -C ~/bob status --short` and note any pre-existing dirty paths. Leave them
   alone.
2. **Compose.** Make a fresh scratch directory with `mktemp -d`, outside the vault, and
   write the Recommended draft into `<dir>/review.md`. Keep the callout's order and its
   rules: link every bullet, no scores/praise/blame/advice/predictions/"because", and no
   retelling of days. Small wording edits are fine. Stay ≤250 words and keep the
   Headline ≤25 words.
3. **Dry-run.** Run `bob periods draft 2026-W43 -i <dir>/review.md -n` and fix anything
   it reports. Run it again with `-f json` and record `words`.
4. **Save.** Run `bob periods draft 2026-W43 -i <dir>/review.md`. It replaces only
   `## Review` and sets the `review_digest` and `review_drafted` frontmatter. Edit
   nothing else by hand.
5. **Verify.**
   - `bob periods brief 2026-W43 -f md | head -1` must show `closed · drafted`.
   - `git -C ~/bob diff -- 2026/2026-W43.md` must show changes only to the lines under
     `## Review`, plus the two frontmatter keys `review_digest` and `review_drafted`
     that `draft` writes by design.
6. **Commit only this note** with `/sase_git_commit`, run from inside `~/bob`. Use a
   conventional header such as `docs(periods): draft 2026-W43 review`. The wrapper
   commits every dirty path, so pass `-x` for any unrelated pre-existing path from step
   1. Never stage, revert, or commit them. If vault auto-sync committed the note first
      (`git status` clean and `git log -1 -p -- 2026/2026-W43.md` shows the new Review),
      do not create an empty commit. Report that sync committed it.
7. **Record follow-ups** on the phase bead. Do not create beads. Add one
   `sase bead note bob-cli-gr.1 'PROPOSED FOLLOW-UP: …'` entry for each:
   - `bob periods brief` drops linked minutes for tasks archived into `done/` notes.
     `linked_minutes` in `src/native/periods/brief.rs` compares the Pomodoro link target
     with `IndexedTask.source_path` but never with `archive_path`. W43's `bob` root (5h
     20m, 21%) therefore lists no tasks, and Finished omits bob completions such as
     `done/bob_done#^root-parent`.
   - The brief's "Kept 3+ times" uses today's `[keeps:: N]` with no period gate, so
     past-week briefs list tasks whose third keep came later. W43 listed two
     `sase_memory_plan` refs kept 3× on 2026-11-02.
   - The brief gives no basis for the callout's `+N more`. The Done fact excludes daily
     `^gtd` rows, while Finished includes them. The brief should print the done-task
     total behind Finished, or the `+N` figure itself.
   - "Still open → No linked minutes" in past-week briefs includes today's calendar
     Event Tasks (W43 listed 2026-11-03 "Disney Movie Night").
8. **Close the phase.** Run `sase bead epic-symbols bob-cli-gr.1` and resolve or re-key
   any leftovers. Then run
   `sase bead close bob-cli-gr.1 --note "<brief header after save, word count, commit hash or sync commit, diff scope verified>"`.
   Do not close epic `bob-cli-gr` or any other bead.
9. **Reply** with the Headline, the word count `bob periods draft` reported, and every
   claim left out as unverifiable: the kept-3× refs, the `sase_goals` block, the "No
   linked minutes" items, and an exact `+N`. Also report that the bob bullet's contents
   were verified from daily notes, because the brief listed no bob tasks.

## Recommended draft (233 words, dry-run passed)

```markdown
**Headline:** A 25h 40m week, 9h 08m below typical, went to the daily loop, sase
reference reading, and bob review and stats tooling.

**Where the time went:**

- [[gtd_daily]] · 8h 48m (34%): daily review rows, led by
  [[2026/20261019#^gtd|Mon Oct 19]] at 3h 50m and [[2026/20261024#^gtd|Sat 24]] at 2h
  30m.
- [[sase]] · 8h 31m (33%): reference reads in [[sase_pypi_releases]] on source control,
  CI test impact analysis, and code review.
- [[bob]] · 5h 20m (21%): ref-panel and Highlights work, a
  [[done/bob_done#^today-marker|today marker]], the
  [[done/bob_done#^rotten-log|Rotten Log]], a
  [[done/bob_done#^add-daily-statistics|daily statistics dash]], and
  [[done/bob_done#^root-parent|root-parent grouping]].

**Finished:**

- [[sase_pypi_releases#^ref-redesigning-source-control|Redesigning source control at Anthropic]]
  · 3h 40m
- [[sase_pypi_releases#^ref-agentic-coding-is-straining-ci-heres|Agentic coding is straining CI]]
  · 1h 35m
- [[sase_pypi_releases#^ref-solving-code-review-at-anthropic|How the Claude Code team does code review]]
  · 1h 20m
- [[sase_pypi_releases#^ref-github-native-merge-queue-primer|How GitHub's native merge queue works]]
  · 1h 00m
- Daily [[gtd_daily]] rows; Review closed 6 of 7, [[2026/20261024|Sat 24]] missed
- +more of 115 Done, including bob's
  [[done/bob_done#^split-highlights|split highlights]] and
  [[done/bob_done#^cmd-p|`<cmd+p>` auto-roll]]

**Still open:**

- [[sase_projects#^prj]] · newly Blocked; still blocked today
- [[sase_better_plans#^prj]] · newly Blocked; still blocked today
- [[sase_remote#^prj]] · newly Blocked; still blocked today

**Questions:**

- Created ran 9 above typical while Done ran 20 below — did intake outpace closes?
- [[2026/20261024|Sat 24]] missed its closeout while [[gtd_daily]] took 52% of it — what
  happened to that review?
- [[sase_back_policy#^prj]] was dropped and [[sase_pre_blog#^prj]] started this week —
  what set those priorities?
```

## Out of scope

- Any other vault file: W42–W27, W44–W46, `2026-Q4.md`, daily notes, `done/` notes, and
  task lines.
- Fixing the brief defects in bob-cli. Record them as follow-ups (step 7). The epic
  forbids repo changes in this phase.

## Verification

- `bob periods brief 2026-W43 -f md | head -1` shows `(closed · drafted)`.
- `git -C ~/bob log -1 --stat -- 2026/2026-W43.md` shows a commit touching only
  `2026/2026-W43.md`, made by `/sase_git_commit` or by vault auto-sync.
- `git -C ~/bob status --short` shows no new dirty paths from this phase.
- `sase bead read bob-cli-gr.1` shows the four PROPOSED FOLLOW-UP notes and a closed
  status.
