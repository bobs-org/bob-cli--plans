---
tier: tale
title: Draft the 2026-W45 weekly Review
goal:
  2026-W45's Review is drafted through bob periods draft from the brief, committed
  alone, and the brief gap is filed.
size: small
proposed_by: bbugyi200.apollo.d2
proposed_with: claude/opus
create_time: 2026-11-09 05:41:27
status: wip
---

# Draft the 2026-W45 weekly Review in the Bob vault

## Goal

Fill the empty `## Review` section of `~/bob/2026/2026-W45.md` (Mon Nov 2 – Sun Nov 8,
2026). Follow the note's `Agent instructions · weekly v1` callout exactly. Save only
through `bob periods draft`, commit only that note, and reply as the callout requires.
Change nothing else in the note: not the callout, the frontmatter (beyond what `draft`
itself sets), `## Intentions`, `## Reflection`, or any other note.

## What planning already established

- `bob periods brief 2026-W45 -f md` reports the week **closed** with review state
  **empty** (`This week is closed: ready to draft.`), so no `--force` is needed.
- The brief's facts: Worked 24h 45m (5h 28m below the typical week of 30h 13m); Reviewed
  344; Done 185 (33 above the typical week of 152); Created 194 (13 above typical);
  Review closed 6 of 7. Wed 4 closeout is `missed`. Every day reads "not reviewed".
- Root parents at ≥10% of Worked: bob 11h 11m (45%), gtd 6h 58m (28%), sase 2h 59m
  (12%). dev is 2h 07m (9%), so it gets no bullet.
- `## Intentions` of this note, the W44 `## Reflection`, and the Q4 `## Outcomes` are
  all empty in the brief. So the **Intentions** part is left out, and nothing is quoted
  from them.
- Finished: no `^prj` completions. The top 7 by linked minutes are 2 bob tasks plus 5
  `gtd_daily` rows, then `+240 more`. That `+240` is `finished_total` (247) minus the 7
  shown. It is a different count from the strip's Done fact (185), so the draft never
  puts the two numbers side by side.
- Still open: the brief's **Kept 3+** list has 4 tasks; that list is used.
  - Its **No linked minutes** list is left out. For a closed week,
    `src/native/periods/brief.rs` (`still_open_section`, past branch) does not filter
    that list to Next/WIP lanes. It sorts every open task with a block id and no linked
    minutes by source path, then cuts it to 7 rows with no `+N more` tail. So the 7 rows
    are just the first `bob.md` tasks alphabetically, not "stalled Next or WIP tasks".
  - Blocked projects are omitted by the brief ("blocked state has no history").
- Facts checked against notes:
  - These tasks were completed inside the week, per their `[completion::]` dates in
    `done/*_done.md`:
    - `done/bob_done#^auto-decay-wip-next` (11-03)
    - `done/bob_done#^wqy-stat-strips` (11-05)
    - `done/bob_done#^tilde-x-equals` (11-06)
    - `done/sase_done#^fix-agent-sidecars` (11-05)
  - Wed's `2026/20261104#^gtd` row is `[-]`, cancelled 2026-11-05. This matches Wed's
    `missed` closeout.
  - The root-parent notes `bob.md`, `gtd.md` (area, root of `gtd_daily`), and `sase.md`
    exist.
- The draft text below passed `bob periods draft 2026-W45 -i <file> -n -f json`:
  `ok: true`, **239 words** (cap 250). The Headline is 23 words (cap 25).

## Steps

1. **Re-check.** In `~/bob`, run `git status --short 2026/2026-W45.md`. If the note has
   uncommitted changes, stop and report; never overwrite them. Then run
   `bob periods brief 2026-W45 -f md` and read all of it.
   - Stop and report if the brief no longer says `closed` with review state `empty`.
   - Compare every number used in the draft with the new brief (facts, root shares and
     task minutes, Finished rows and `+N more`, Kept 3+ rows and reasons). If any number
     changed, edit only the affected text so it copies the new brief exactly. Recheck
     the rules in step 2 after any edit.
2. **Write the draft** to a scratch file in a fresh `mktemp -d` directory, never inside
   the vault. Use exactly the text in "Draft text" below, plus any step-1 edits. Rules
   the text must keep:
   - Headline first.
   - Every bullet links.
   - Numbers are copied as the brief displays them.
   - No scores, praise, blame, advice, predictions, or "because" claims.
   - Bryan is quoted only verbatim, with a link.
   - No headings, `> [!agent]` callouts, or `---` fences.
3. **Dry run:** `bob periods draft 2026-W45 -i <file> -n -f json`. It must report
   `ok: true` with `words` ≤ 250. If it fails, fix the text (shorten aliases first) and
   rerun.
4. **Save:** `bob periods draft 2026-W45 -i <file>`. Expect
   `Drafted 2026-W45 Review · <N> words · state drafted`. Exit code 3 means a refusal:
   report it and do not use `-F`.
5. **Verify:** `git -C ~/bob diff 2026/2026-W45.md` changes only:
   - the `## Review` body;
   - the frontmatter `review_digest` and `review_drafted` lines.

   `bob periods list -k week` should show 2026-W45 as `drafted`.

6. **Commit only this note,** as `~/bob/AGENTS.md` describes. Never use
   `/sase_git_commit` in the vault, and do not stage any other pre-existing change.

   ```bash
   git -C ~/bob add 2026/2026-W45.md
   git -C ~/bob commit -m "periods: draft 2026-W45 Review" \
     -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
   ```

   Do not push by hand; `bob vault-sync` pushes vault commits.

7. **File the discovered brief gap.** Use `/sase_new_task` (it checks for duplicates
   first) to record a `bug` task bead in bob-cli. The bug: for closed periods, the
   brief's Still open → "No linked minutes" list is not lane-filtered and is cut to the
   first 7 tasks by source path with no `+N more` tail. That makes it unusable for the
   weekly callout's "stalled Next or WIP tasks". The code is in
   `src/native/periods/brief.rs`, past branch of `still_open_section`, and
   `docs/periods.md` §Agent briefs item 7 documents the behaviour. Let the skill choose
   the size; the fix (keep a usable lane snapshot vs. change the list or callout
   wording) is a design choice, so `large` is the likely default.
8. **Reply** with:
   - the Headline;
   - the final word count;
   - what was left out as unverifiable or inapplicable: the No-linked-minutes rows
     (reason above), Blocked projects (no history in the brief), and Intentions (empty);
   - the bead id from step 7.

## Draft text

```markdown
**Headline:** A 24h 45m week, 5h 28m below typical, put 45% into bob, led by WIP/Next
auto-decay and weekly/quarterly/yearly note templates.

**Where the time went:**

- [[bob]] · 11h 11m (45%):
  [[done/bob_done#^auto-decay-wip-next|WIP and Next auto-decay]] (55m),
  [[done/bob_done#^wqy-stat-strips|weekly/quarterly/yearly templates]] (45m), and
  [[done/bob_done#^tilde-x-equals|`~N` for `x=`]] (32m).
- [[gtd]] · 6h 58m (28%): daily review rows, led by [[2026/20261102#^gtd|Mon Nov 2]] at
  1h 30m and [[2026/20261104#^gtd|Wed 4]] at 1h 18m.
- [[sase]] · 2h 59m (12%):
  [[sase#^ref-the-ai-native-sdlc-playbook|The AI-native SDLC playbook]] (35m),
  [[sase#^agent-hijack-turns|agent hijack turns]] (30m), and
  [[done/sase_done#^fix-agent-sidecars|sidecar publishing]] (20m).

**Finished:**

- [[done/bob_done#^auto-decay-wip-next|Auto-decay WIP to Next and Next to Ready]] · 55m
- [[done/bob_done#^wqy-stat-strips|Weekly/quarterly/yearly file templates]] · 45m
- Daily [[gtd_daily]] rows: [[2026/20261102#^gtd|Mon]] 1h 30m,
  [[2026/20261103#^gtd|Tue]] 1h 00m, [[2026/20261105#^gtd|Thu]] 45m,
  [[2026/20261107#^gtd|Sat]] 43m, [[2026/20261106#^gtd|Fri]] 40m

+240 more.

**Still open:**

- [[bob#^successor-deps|Dependencies from successor offers]] · kept 3×, no reason given
- [[sase|`/sase_plan` agents may critique requirements]] · kept 3×, no reason given
- [[sase_completion#^e2e-gate-finalizers|Gates and monitors to finalizers]] · kept 3×,
  no reason given
- [[sase_memory_plan#^ref-docs-backed-memory-and-agent-maintained|Docs as Memory]] ·
  kept 3×: "I still really need to clean up sase's memory."

**Questions:**

- Done ran 33 above typical while Worked ran 5h 28m below and [[bob]] took 45% — what
  kinds of tasks made up those closes?
- [[2026/20261104|Wed 4]] missed its closeout although its
  [[2026/20261104#^gtd|gtd row]] logged 1h 18m — what happened to that review?
- Three of four 3× keeps, like [[bob#^successor-deps|successor dependencies]], carry no
  reason — what keeps them on the list?
```

### Why the draft looks this way

- **Finished** keeps the brief's own top 7, folding the 5 `gtd_daily` rows into one
  bullet the way W44 did, so `+240 more` stays exact. Swapping in other tasks would mean
  computing a new `+N`, which the callout forbids. `+240 more.` is a separate paragraph,
  not a bullet, so the link-every-bullet rule holds.
- The `[[sase|…]]` link has no `^block`: that kept task has no block id in `sase.md`.
- Each question rests on one brief fact: the Done/Worked facts, Wed's `missed` closeout
  with its 1h 18m gtd row, and three `🤷 no reason given` keeps.

## Done when

- `bob periods draft` reported `state drafted` with ≤ 250 words.
- The vault diff touches only the Review body and the two `review_*` frontmatter lines.
- One vault commit contains only `2026/2026-W45.md`.
- The bug bead is filed, or the existing duplicate is corroborated.
- The reply gives the Headline, the word count, and what was left out.
