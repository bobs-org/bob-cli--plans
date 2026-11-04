---
tier: tale
size: small
title: Draft, save, and commit the 2026-W27 Review (Jun 29 – Jul 5)
goal:
  2026/2026-W27.md has a `## Review` drafted through `bob periods draft` per its weekly
  v1 callout, verified as closed · drafted, committed alone, and phase bead
  bob-cli-gr.17 is closed with that evidence.
bead: bob-cli-gr.17
proposed_by: bbugyi200.athena.bob-cli-gr.17
proposed_with: claude/opus
create_time: 2026-11-03 22:09:04
status: wip
---

- **PARENT:**
  [202611/weekly_reviews_w43_to_w27.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/weekly_reviews_w43_to_w27.md)
- **BEAD:**
  [bob-cli-gr.17](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-gr/bob-cli-gr.17.md)

# Plan: Draft the 2026-W27 Review (bead bob-cli-gr.17)

## Context

This is phase `w27` of epic `bob-cli-gr`. The epic is
`plan:202611/weekly_reviews_w43_to_w27.md`, "Draft the weekly Reviews for 2026-W43 back
through 2026-W27". W27 is the last phase in the chain, and its predecessor w28
(`bob-cli-gr.16`) is closed. This phase writes the `## Review` of
`~/bob/2026/2026-W27.md` (week id `2026-W27`, Mon Jun 29 – Sun Jul 5, 2026, parent
`2026/2026-Q3`) through `bob periods draft`, then commits that one note. **No file in
the bob-cli repository changes.** No other vault file changes either.

Two sources set the rules, and both are binding. One is the epic's **Shared drafting
contract** (steps 1–8). The other is the `> [!agent]- Agent instructions · weekly v1`
callout below `## Reflection` in the note. If they disagree, follow the callout and
report the difference. The epic lists no DECISIONS. This plan restates the steps and
adds what was verified while planning on 2026-11-03.

### State verified while planning

- `bob periods brief 2026-W27 -f md` opens with `(closed · empty)`. Nothing has been
  drafted yet: `git -C ~/bob log -- 2026/2026-W27.md` shows only
  `ff499d867 periods: adopt the period-note tree for ISO 2026`, and
  `git -C ~/bob status --short` was clean.
- `## Intentions` is empty, and so are the W26 `## Reflection` and the Q3 `## Outcomes`.
  The brief's neighbours section shows only an empty `### Intentions`. So the draft
  **omits the Intentions part**.
- `sase bead epic-symbols bob-cli-gr.17` reports no `--epic-symbol` entries.
- The phase bead carries no earlier notes, so no earlier attempt left remaining work.
- The candidate draft below passed `bob periods draft 2026-W27 -i - -n -f json` (fed on
  stdin, so no file was written) with
  `{"ok":true,"review_state":"drafted","words":238}`.

### What the brief shows (re-read it; do not trust this summary over the live brief)

- Facts: Worked `23h 00m` (`6h 13m below the typical week of 29h 13m`). Done `60`
  (`5 below the typical week of 65`). Created `54` (`19 below the typical week of 73`).
  Reviewed `–`. Review closed `–`.
- Root parents in Time:
  - `sase · 15h 11m (66%)`. Children: `[[sase_config#^nice-config-edits]]` 3h 45m,
    `[[sase_dyn_agent_fam#^read-critique]]` 3h 00m, and
    `[[sase_actstat#^fix-existing-gha-fails]]` 2h 05m.
  - `gtd_daily · 6h 51m (30%)`.
  - No other root parent reaches 10%. Parts shows bob only on Mon (12% of 4h 35m) and
    sase_blog only on Tue (16% of 2h 35m), both far below 10% of the week.
- The row `(unnamed) (type) · 23h 00m (100%)` is a type row, not a root parent. Exclude
  it from "Where the time went".
- Finished lists six `^prj` completions: `sase_cleanup`, `sase_update` (· 50m),
  `sase_install`, `vacation_sc_26`, `sase_version`, and `sase_model_panel`. It then
  lists one task, `sase_config#^nice-config-edits` · 3h 45m. Every completion date in
  those notes falls between 2026-06-30 and 2026-07-05.
- Caveats: all seven days appear under both "Review journal gaps" and "Not reviewed
  days".

### Verified while planning (link targets, labels, quotes)

- Every Time and Finished link target resolves to a matching block in the vault, with
  one exception: `[[sase_install#^prj]]` does not. `~/bob/sase_install.md` has no `^prj`
  line (its frontmatter says `status: done` and
  `done_tasks: "[[done/sase_install_done]]"`). The `^prj` line now lives in
  `~/bob/done/sase_install_done.md` line 6, completed 2026-07-05. The draft therefore
  links **`[[done/sase_install_done#^prj]]`**, and the reply reports the brief's stale
  target.
- The block labels used as link aliases below are copied from the task lines. The
  `#hide` tag is dropped from the `sase_model_panel` label.
- Bryan's-words quotes are verbatim, and each sits in its daily note:
  - `2026/20260629.md` line 40: `Knock out some [[sase]] tasks (too many in that file)!`
  - `2026/20260703.md` line 24: `Set up [[sase]] on apollo!`
  - `2026/20260705.md` line 48: `Asked Fable to critique what we have so far!`. It is
    under `[[sase_dyn_agent_fam#^reqs]]` in the 1805-1855 Pomodoro. The `^read-critique`
    task follows in the 1920-2015 (55m) and 2015-2220 (125m) Pomodoros, which add up to
    the brief's 3h 00m.
- **Still open: omit the whole part.** Every item the brief lists was created after W27,
  so none of them was stalled or blocked during the week:
  - The seven bob "No linked minutes" tasks have `[created::]` dates from 2026-10-27 to
    2026-11-03.
  - The blocked projects were created later: `sase_better_plans` on 2026-07-08,
    `sase_goals` on 2026-09-26 (frontmatter), `sase_projects` on 2026-09-01, and
    `sase_remote` on 2026-09-04.
  - The two "Kept 3+ times" `sase_memory_plan` refs were created on 2026-10-15. Their
    Rotten Log entries are dated 2026-10-30 and 2026-11-02.

### Brief figures that disagree (do not reconcile; leave out and report)

1. **gtd_daily per-row minutes do not match Parts.** The row `20260703#^gtd` shows 2h
   15m, but Parts gives Fri 3 gtd_daily 45% of 3h 40m (≈1h 39m). The row `20260704#^gtd`
   shows 1h 05m, but Parts gives Sat 4 29% of 1h 55m (≈33m). Also, `2026/20260703.md`
   line 19 shows the Fri 3 gtd row as cancelled (`[-]`, `[cancelled:: 2026-07-04]`), yet
   Time still lists it. Do not print any per-row gtd_daily minutes. Use only the root
   total `6h 51m (30%)`.
2. **"Not reviewed" versus gtd_daily time.** Caveats list every day as not reviewed,
   while gtd_daily took 6h 51m. Make no claim about review days closing.
3. **Stale link target.** Finished prints `[[sase_install#^prj]]`, but the block lives
   in `done/sase_install_done.md`, as noted above.
4. **Bryan's-words parsing artifact.** The brief prints a second 2026-06-29 line,
   `[[2026/20260629|sase]]]] · sase]]`. It looks like a fragment split off the nested
   `[[sase]]` link in the "Knock out some…" line. Do not quote it.

## Steps

### 1. Check

Run `bob periods brief 2026-W27 -f md` and read all of it. If the header does not say
`closed · empty`, stop without drafting. That includes `drafted`, `edited`, and
`reflected`. Report the state, then go to step 7 and close the bead with that report.
Never pass `-F/--force`. Compare the live brief with the summary above. If any figure,
link, or list item that the candidate draft uses has changed, rebuild that line from the
live brief before saving.

### 2. Compose (at most 250 words, scratch file outside the vault)

Create a fresh scratch directory with `SCRATCH=$(mktemp -d)` and write
`$SCRATCH/review.md` with the candidate below. The candidate follows the callout's
order: Headline, Where the time went, Finished, and Questions. It omits Intentions,
because Intentions is empty, and Still open, because every listed item postdates the
week. It uses the house format of `2026/2026-W44.md`'s Review: bold part labels, `- `
bullets, and `[[note#^block|label]]` links. It takes no facts from another week's
Review.

```markdown
**Headline:** A 23h 00m week, 6h 13m below typical, gave sase 66% and closed five sase
projects plus the July 4th SC trip.

**Where the time went:**

- [[sase]] · 15h 11m (66%):
  [[sase_config#^nice-config-edits|Make it WAY nicer to edit settings from config panel!]]
  · 3h 45m, [[sase_dyn_agent_fam#^read-critique|Read dynamic agent family critique!]] ·
  3h 00m, and
  [[sase_actstat#^fix-existing-gha-fails|Fix all existing GitHub Actions failures!]] ·
  2h 05m.
- [[gtd_daily]] · 6h 51m (30%): the daily loop.

**Finished:**

- [[sase_cleanup#^prj|Cleanup unused / not useful features!]]
- [[sase_update#^prj|Create new `sase update` command!]] · 50m
- [[done/sase_install_done#^prj|Installing sase using the published versions works end-to-end on a fresh machine!]]
- [[vacation_sc_26#^prj|July 4th, 2026 vacation to SC!]]
- [[sase_version#^prj|Release stable versions of all packages and thoroughly test user install instructions and the new `sase version` command!]]
- [[sase_model_panel#^prj|Remove `default_model` config in favor of model management using `,m` keymap!]]
- [[sase_config#^nice-config-edits|Make it WAY nicer to edit settings from config panel!]]
  · 3h 45m
- +more of 60 Done

**Questions:**

- [[2026/20260705|Sun 5]] logged “Asked Fable to critique what we have so far!”, then
  [[sase_dyn_agent_fam#^read-critique|Read dynamic agent family critique!]] took 3h 00m
  — what did the critique change?
- [[2026/20260629|Mon Jun 29]] logged “Knock out some [[sase]] tasks (too many in that
  file)!”, and [[sase_cleanup#^prj|Cleanup unused / not useful features!]] finished —
  how much did [[sase]] shrink?
- [[2026/20260703|Fri 3]] logged “Set up [[sase]] on apollo!”, and
  [[done/sase_install_done#^prj|the fresh-machine install project]] finished — was
  apollo that fresh machine?
```

Notes on the candidate:

- The Headline has 22 words, all copied from Facts, Time, and Finished. "Five sase
  projects" counts `sase_cleanup`, `sase_update`, `sase_install`, `sase_version`, and
  `sase_model_panel`. The sixth `^prj`, `vacation_sc_26`, is named separately.
- `+more of 60 Done` follows the w28 precedent. The brief displays no `+N` remainder, so
  do not compute one.
- Each Question rests on a fact stated earlier in the draft (the 3h 00m critique read,
  the `sase_cleanup` completion, the `sase_install` completion), plus one verbatim,
  linked quote. None asserts a cause.
- Rules (callout step 4): link every bullet. Copy numbers exactly as displayed. No
  scores, grades, praise, blame, advice, predictions, or "because" claims. Quote Bryan
  only verbatim and with a link. Write about the week, never as Bryan. Don't retell the
  days.

### 3. Dry-run, then save

```bash
bob periods draft 2026-W27 -i "$SCRATCH/review.md" -n -f json
```

Fix every reported problem (Headline shape, word cap) until it reports `"ok":true`.
During planning it reported 238 words. Then run it again without `-n`:

```bash
bob periods draft 2026-W27 -i "$SCRATCH/review.md" -f json
```

Edit nothing else by hand: not the callout, other sections, frontmatter, tasks, or other
notes.

### 4. Verify

- `bob periods brief 2026-W27 -f md | head -1` now ends with `(closed · drafted)`.
- `git -C ~/bob diff -- 2026/2026-W27.md` shows changes only between `## Review` and
  `## Reflection`, plus any `review_digest`/`review_drafted` frontmatter that
  `bob periods draft` itself writes. W28's sync commit had the same shape. If vault
  auto-sync already committed the file, inspect the newest commit that touches it with
  `git -C ~/bob log -1 -p -- 2026/2026-W27.md`.
- Every `[[target#^block]]` in the saved Review resolves. For each block id, run
  `grep -n '\^<block>' ~/bob/<target>.md` and confirm it matches. Include
  `done/sase_install_done.md` for `^prj`.

### 5. Commit only this note

Run `git -C ~/bob status --short` first. The vault's auto-sync can commit the change
before you do, as a `vault(athena): 1 file - 2026/2026-W27.md` commit; that is what
happened for W28 and earlier weeks. If the note is clean and
`git -C ~/bob log -1 -p -- 2026/2026-W27.md` contains the new Review, do not create an
empty commit. Report that sync committed it. Otherwise commit `2026/2026-W27.md` alone
with `/sase_git_commit`, which is the vault's required path (see `~/bob/AGENTS.md`). Do
not stage, revert, or commit any other file.

### 6. Record follow-ups

Do not create beads. Record each of these with
`sase bead note bob-cli-gr.17 'PROPOSED FOLLOW-UP: …'`:

- The `bob periods brief` Finished list links `[[sase_install#^prj]]`, but the `^prj`
  line moved to `done/sase_install_done.md` when the project closed. The brief should
  print the resolving target for completed projects whose task line moved to `done/`.
- The `bob periods brief` "Bryan's words" section splits a line that contains a nested
  `[[sase]]` link and prints a stray fragment, `[[2026/20260629|sase]]]] · sase]]`. The
  first line's alias also nests a wiki link inside `[[2026/20260629|…]]`.
- The `bob periods brief` Time section shows gtd_daily per-row minutes that disagree
  with Parts' per-day shares (Fri 3: 2h 15m against ≈1h 39m; Sat 4: 1h 05m against
  ≈33m). It also lists a cancelled gtd row (`20260703#^gtd`). This is the same family as
  the w28 follow-up, so cite that note rather than filing a duplicate.

### 7. Close the phase bead

Run `sase bead epic-symbols bob-cli-gr.17` and confirm there are no entries. Then:

```bash
sase bead close bob-cli-gr.17 --note "<header now closed · drafted; word count from bob periods draft; diff limited to ## Review (+ draft frontmatter); commit hash or sync commit; omitted claims>"
```

Close **only** `bob-cli-gr.17`. Never close the epic `bob-cli-gr`, which belongs to its
land agent, or any other bead.

### 8. Reply

Give the Headline, the word count `bob periods draft` reported, and the commit (yours or
the sync commit). List every claim left out as unverifiable or out of scope:

- the omitted Still open part (every listed item postdates W27)
- the four brief disagreements above
- the replaced `sase_install` link target

## Verification (this phase)

- `bob periods brief 2026-W27 -f md | head -1` ends with `(closed · drafted)`.
- `git -C ~/bob log --oneline -2 -- 2026/2026-W27.md` shows exactly one new commit for
  this Review, and that commit touches only `2026/2026-W27.md`.
- `git -C ~/bob status --short` shows no change made by this phase to any other file.
- `sase bead read bob-cli-gr.17 -r "confirm close"` shows the bead closed with the
  verification note.
