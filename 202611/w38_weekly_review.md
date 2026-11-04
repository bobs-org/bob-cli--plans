---
tier: tale
size: small
title: Draft, save, and commit the 2026-W38 Review (Sep 14 – 20)
goal:
  "`2026/2026-W38.md` holds a `## Review` drafted through `bob periods draft` under its
  own weekly v1 callout, the brief reports `closed · drafted`, the note is committed
  alone, and phase bob-cli-gr.6 is closed. Nothing else in the vault changes."
proposed_by: bbugyi200.athena.bob-cli-gr.6
proposed_with: claude/opus
bead: bob-cli-gr.6
create_time: 2026-11-03 20:12:40
status: wip
---

- **PARENT:**
  [202611/weekly_reviews_w43_to_w27.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/weekly_reviews_w43_to_w27.md)
- **BEAD:**
  [bob-cli-gr.6](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-gr/bob-cli-gr.6.md)

# Plan: Draft, save, and commit the 2026-W38 Review

## Context

This tale completes phase `w38` of epic `bob-cli-gr` (draft the weekly Reviews for
2026-W43 back through 2026-W27). It follows that epic's **shared drafting contract** and
the `> [!agent]- Agent instructions · weekly v1` callout inside
`~/bob/2026/2026-W38.md`. The callout is the authority. If anything below disagrees with
it, follow the callout and report the difference.

- **Note:** `~/bob/2026/2026-W38.md` · week id `2026-W38` · Mon Sep 14 – Sun Sep 20,
  2026 · previous `2026/2026-W37` · parent `2026/2026-Q3`.
- **The only file that may change:** `2026/2026-W38.md`. Nothing changes in the bob-cli
  repository.
- **State on 2026-11-03 (planning):** `bob periods brief 2026-W38 -f md` opened with
  `(closed · empty)`, and `git -C ~/bob status` was clean. `## Intentions` is empty and
  the brief shows no W37 Reflection and no Q3 Outcomes, so the draft leaves out
  **Intentions**.

What was verified while planning:

- The brief's sase root (22h 37m, 72%) lists only one task
  (`[[sase_clean#^work-all-beads]]` · 50m). The Sep 14–20 Pomodoro ledgers in
  `2026/20260914.md` … `2026/20260920.md` show where the rest went. These are
  `done/sase_done` tasks: `^services` (the Services tab), `^tool-use-cases` and
  `^read-tool-epic-roadmap` (`sase tool` research), `^tui-screenshot`, `^sudo`, and the
  muse rows (`^muse-usage`, `^fix-muse-reply`, `^add-muse-to-pools`). The draft names
  them with no minutes, because the brief prints no per-task numbers for them.
- The brief prints Finished and Dropped links as `[[bob#^…]]` / `[[sase#^…]]`, but those
  blocks now live only in `done/bob_done.md` and `done/sase_done.md`. The printed
  targets are dead, so the draft links to the `done/` notes, as the W39–W43 Reviews did.
  This brief bug is already recorded in bob-cli-gr.5 note #1 (and bob-cli-gr.1 note #1);
  cite it rather than filing a duplicate.
- The gtd_daily task rows (2h 15m + 1h 40m + 1h 15m) add up to more than the 3h 47m
  root. The brief counts merged Pomodoros in full for each task, but splits them for
  roots. That mismatch is already recorded in bob-cli-gr.3 note #1. The draft quotes
  only one task figure, exactly as the brief shows it, and does not reconcile the two.
- **Still open is left out entirely.** The two `kept 3×` `sase_memory_plan` refs were
  created on 2026-10-15, after this week. The four Blocked projects and the "No linked
  minutes" rows reflect the vault today, and nothing shows they were newly Blocked or
  stalled at the end of W38. The contract allows leaving out items whose week-end state
  cannot be verified.
- A dry run (`bob periods draft 2026-W38 -i - -n -f json`, which writes nothing) of the
  proposed draft below returned `{"ok":true,"review_state":"drafted","words":195}`.
- The W40 phase found that `bob periods draft` also writes `review_digest` /
  `review_drafted` frontmatter keys. That is bob's own write, and it is expected in the
  diff.

## Proposed draft (≤250 words; dry-run verified at 195)

```markdown
**Headline:** A 31h 15m week, 2h 53m below typical, went 72% to sase while Done ran 53
below and Created 62 below typical.

**Where the time went:**

- [[sase]] · 22h 37m (72%): the [[done/sase_done#^services|Services tab]],
  [[done/sase_done#^tool-use-cases|`sase tool` use-case research]] and the
  [[done/sase_done#^read-tool-epic-roadmap|tool epic roadmap]],
  [[done/sase_done#^tui-screenshot|`sase tui screenshot`]],
  [[done/sase_done#^sudo|`sudo` prompt support]], and
  [[done/sase_done#^muse-usage|muse code]] work, plus
  [[sase_clean#^work-all-beads|bead review]] at 50m.
- [[gtd_daily]] · 3h 47m (12%): daily review rows, led by the
  [[2026/20260920#^gtd|Sun 20 row]] at 2h 15m; all seven days show not reviewed.

**Finished:**

- [[done/bob_done#^equal-split|`=pom` split syntax]]
- [[done/bob_done#^copy-merge-poms|Copying and merging pomodoros]]
- [[done/bob_done#^mac-capture-next|Mac capture task activation]]
- [[done/sase_done#^enter-piw-prompt|PIW submit y/n prompt]]
- [[ref/chat/sase_tool_control_plane#^ref|sase tool control plane reference]]
- [[gtd_daily]] habits: Brush teeth, Check weather
- +more of 20 Done, including
  [[done/sase_done#^tool-use-cases|`sase tool` use-case research]]

**Questions:**

- Worked ran 2h 53m below typical while Done ran 53 below and Created 62 below — what
  kept both intake and closes low?
- All seven days show not reviewed, and [[gtd_daily]] rows such as
  [[2026/20260914#^gtd|Mon 14]] were dropped after 1h 40m — what happened to the daily
  review?
- [[done/sase_done#^tool-critique|The architecture-critique read]] and
  [[done/sase_done#^mac-plans-on-athena|launching mac plans on athena]] were dropped —
  what set those calls?
```

Where each claim comes from:

- **Headline / Q1:** brief `## Facts`: Worked 31h 15m (2h 53m below 34h 08m); Done 20
  (53 below); Created 83 (62 below); and `## Time`: sase 72%.
- **Where the time went:** brief `## Time` roots at ≥10% are sase and gtd_daily; bob
  (9%) and unlinked (6%) fall below 10%, and type rows are left out. The sase task names
  come from the W38 ledgers and the `done/sase_done.md` task lines. The gtd_daily figure
  comes from the brief's top task row. "All seven days show not reviewed" comes from
  `## Parts`.
- **Finished:** the brief's seven `## Finished` rows (no `^prj` among them), relinked to
  their `done/` homes. `^tool-use-cases` was completed 2026-09-17, inside the week.
- **Q2:** brief `## Parts` plus `## Started and dropped` (`2026/20260914#^gtd` · 1h 40m;
  that row was cancelled 2026-09-15).
- **Q3:** brief `## Started and dropped`. `^tool-critique` was cancelled 2026-09-15 and
  `^mac-plans-on-athena` on 2026-09-20.

## Steps

1. **Check.** Run `bob periods brief 2026-W38 -f md` and read all of it. If the header
   does not say `closed · empty`, stop: report the state, do not draft, and never pass
   `-F/--force`. Otherwise, compare it with the facts above (Worked, Done, Created, root
   shares, the Finished and Dropped rows). If any number the draft quotes has changed,
   use the new value as the brief shows it. Then run
   `grep -rlE '\^<block>$' ~/bob --include='*.md'` for each block id the draft links to,
   and confirm each one still resolves in the note the link names.
2. **Compose.** Write the proposed draft, with any corrections from step 1, to a scratch
   file in a fresh directory outside the vault (`mktemp -d`). Do not add retelling of
   the days, scores, praise, blame, advice, predictions, or "because" claims. Keep it at
   no more than 250 words, with a single-sentence Headline of no more than 25 words.
3. **Dry-run, then save.** Run `bob periods draft 2026-W38 -i <file> -n` and fix
   anything it reports. Then run it again without `-n`. Edit nothing by hand.
4. **Verify.** Re-run `bob periods brief 2026-W38 -f md`; the header must read
   `closed · drafted`. Run `git -C ~/bob diff -- 2026/2026-W38.md` and confirm that only
   the `## Review` body and bob's `review_digest` / `review_drafted` frontmatter keys
   changed.
5. **Commit only this note.** Run `git -C ~/bob status` first. Leave any unrelated
   pre-existing changes alone. Commit `2026/2026-W38.md` alone with `/sase_git_commit`.
   If vault auto-sync (`vault(<host>): 1 file - 2026/2026-W38.md`) already committed it,
   the note is clean, and `git -C ~/bob log -1 -p -- 2026/2026-W38.md` shows the new
   Review, then make no empty commit and report that sync committed it.
6. **Close the phase.** Run `sase bead epic-symbols bob-cli-gr.6` and resolve any
   leftovers. Then run
   `sase bead close bob-cli-gr.6 --note "<brief header after draft, word count, commit SHA, diff scope, omitted claims>"`.
   Do not close the epic `bob-cli-gr`. Record discovered work only as
   `sase bead note bob-cli-gr.6 'PROPOSED FOLLOW-UP: …'`, and only when the earlier
   phases' notes do not already cover it.
7. **Reply** with the Headline, the word count `bob periods draft` reported, and every
   claim left out because it could not be verified:
   - all of **Still open**: the `kept 3×` refs were created 2026-10-15, and the Blocked
     projects' and zero-minute rows' week-end state can't be verified;
   - per-task minutes for the sase work, which the brief does not show;
   - the gtd_daily task-row sum versus the root total, which is not reconciled (see
     bob-cli-gr.3 note #1);
   - the Finished and Dropped links moved to `done/` targets (see bob-cli-gr.5 note #1).

## Verification

- `bob periods brief 2026-W38 -f md | head -1` ends with `(closed · drafted)`.
- `git -C ~/bob log --oneline -1 -- 2026/2026-W38.md` is the Review commit (from
  `/sase_git_commit` or vault sync). `git -C ~/bob status --short` lists nothing this
  phase caused.
- No other vault file changed, and the bob-cli repository has no diff.
