---
tier: tale
size: small
title: Save the approved v2 Review of 2026-W40
goal:
  2026/2026-W40.md carries the approved work-first v2 Review, saved through `bob periods
  draft` without `-F` over its `drafted` state and committed alone in the vault.
bead: bob-cli-j4.5
proposed_by: bbugyi200.athena.bob-cli-j4.5
proposed_with: claude/opus
create_time: 2026-11-09 11:48:47
status: wip
---

- **PARENT:**
  [202611/q4_weekly_reviews_v2_redraft.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/q4_weekly_reviews_v2_redraft.md)
- **BEAD:**
  [bob-cli-j4.5](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-j4/bob-cli-j4.5.md)

# Plan: Save the approved v2 Review of 2026-W40 (bead `bob-cli-j4.5`)

This is phase `redraft-w40` of epic `bob-cli-j4` (plan
`202611/q4_weekly_reviews_v2_redraft.md`). It is the epic's last phase. Approving this
plan approves the exact Review text below. The coder saves it through
`bob periods draft` **without `-F`**, commits only `2026/2026-W40.md` in the vault, and
replies. No bob-cli file changes.

## Findings from planning (steps 1–8 of the epic procedure)

- **Vault state:** `git -C ~/bob status --short` was clean. The newest vault commits are
  sync commits, plus `ae317accc`, the W41 v2 redraft from phase `redraft-w41`.
- **Review state:** `bob periods list -k week -f json` shows `2026-W40` as `drafted`,
  with the v1 Headline. So the save uses **no `-F`**.
  `git -C ~/bob log -- 2026/2026-W40.md` lists only `cf0e7a67d` (the v2 callout
  refresh), `8091ae2bd` (the v1 draft), and `ff499d867` (the period-note tree adoption).
- **Brief:** `bob periods brief 2026-W40 -f md` (brief v2) reports the week as closed
  and drafted. Its Facts:
  - Worked 35h 55m; Reviewed "–" (no typical week yet); Done 90; Created 103; Review
    closed 0 of 5.
  - Two `^prj` completions, listed in full: `sase_decks#^prj` and
    `sase_gate_shells#^prj`.
  - "Kept 3+ times" is None. Blocked projects are "Omitted — blocked state has no
    history."
  - "Review closed: 0 of 5" and the missing typical Reviewed week are treated as data,
    not as a question prompt.
- **Neighbours:**
  - This week's `## Intentions` is empty, and so is W39's `## Reflection`.
    `2026/2026-Q4`'s `## Outcomes` is empty too.
  - W39 (Q3) is still v1 and stays v1. Its Review has no Still open or Carried forward
    part, so nothing here gets an "also carried in W39" mark. W39 is used only as
    context and is never edited.
  - W39's three Questions (unlinked hours on Mon Sep 21, the dropped daily reviews, the
    agent relaunches) are not repeated.
  - Nor are the v1 W40 Questions: the Done/Created close-out, Review closed 0 of 5 with
    gtd at 47% of Sun 4, and `sase_sites` plus six reads dropped.
- **Claims verified in notes:**
  - **The two `^prj` completion dates:** `sase_decks.md` closed Oct 4 and
    `sase_gate_shells.md` closed Oct 3.
  - **Every other Shipped completion date falls in Sep 28 – Oct 4:**
    - In `done/sase_done.md`: `axe-restart` Sep 28; `clean-prompt-history` and `linker`
      Sep 30; `e2e-listen` and `memory-file-versions` Oct 3; and `listen`,
      `memory-history-tui`, and `support-3-pane` Oct 4.
    - In `done/bob_done.md`: `gkeep` and `randomize` Sep 28; `fuzzy-capture-task` Sep
      30; and `cli-completion` Oct 4.
    - In `bob_gtd.md`: `hide-rotten-tasks`, `scheduled-are-stale`, and
      `wip-next-refresh` Oct 1.
  - **Carried forward:** in the vault snapshot `d720970c5` (Oct 4 18:30, the last commit
    before Oct 5):
    - `bob_gtd.md`'s `^prj` is `[/]`, created Sep 30; it closed Oct 6. The brief lists
      it under "Started".
    - `^epic-roadmap` is `[*]` in `sase_goals.md`. It has since moved to
      `sase_post_blog.md`, where the brief's link resolves.
    - `^release-v18` is `[/]` in `sase.md`; it closed Oct 19.
- **Every duration** below is copied exactly as the brief displays it. The Review states
  no counts other than durations.
- **Dry run:** `bob periods draft 2026-W40 -i <file> -n -f json` returned
  `{"id":"2026-W40","ok":true,"review_state":"drafted","schema_version":1,"words":177}`,
  and the vault stayed clean.

## Headlines

- **Old (v1, already one line):** A 35h 55m week, 3h 55m above typical, finished two
  projects while bob took 48% and sase 37% of the time.
- **New (v2):** Closed the AMP-to-decks/cards and gate-shell projects; built sase-listen
  and tracked memory-file history; added `bob gkeep`, `bob randomize`, and CLI
  completion.

## Exact Review text to save (177 words per bob)

Save this byte for byte, from the `**Headline:**` line to the last Carried forward
bullet. The Headline and every bullet are one physical line each. Do not rewrap them.

```markdown
**Headline:** Closed the AMP-to-decks/cards and gate-shell projects; built sase-listen
and tracked memory-file history; added `bob gkeep`, `bob randomize`, and CLI completion.

**Shipped:**

- Closed [[sase_decks#^prj|AMP to decks/cards]] Oct 4 and
  [[sase_gate_shells#^prj|sase gate shells]] Oct 3.
- [[sase]]: [[done/sase_done#^listen|sase-listen]] · 28m and its
  [[done/sase_done#^e2e-listen|E2E test]] · 1h 12m;
  [[done/sase_done#^memory-file-versions|tracked memory-file history]] · 26m and
  [[done/sase_done#^memory-history-tui|memory history in the TUI]] · 31m.
- [[done/sase_done#^axe-restart|`sase axe restart`]] · 1h 05m,
  [[done/sase_done#^clean-prompt-history|non-user prompts kept out of prompt history]] ·
  1h 11m, a [[done/sase_done#^linker|`linker` research-swarm agent]] · 1h 03m, and
  [[done/sase_done#^support-3-pane|3-pane splits in pager and deck panels]] · 33m.
- [[bob]]: [[done/bob_done#^gkeep|`bob gkeep`]] · 1h 09m,
  [[done/bob_done#^randomize|`bob randomize`]] · 1h 06m,
  [[done/bob_done#^cli-completion|CLI completion and `just install`]] · 1h 03m, and
  [[done/bob_done#^fuzzy-capture-task|`@:` fuzzy task search in capture]] · 51m.
- [[bob_gtd]]: [[bob_gtd#^hide-rotten-tasks|rotten tasks hidden from the dash]] · 50m,
  [[bob_gtd#^scheduled-are-stale|scheduled tasks treated as stale]] · 27m, and
  [[bob_gtd#^wip-next-refresh|a 1-day refresh for WIP and Next]] · 25m.

**Carried forward:**

- [[bob_gtd#^prj|The GTD morning-review project]] · 1h 48m, started Sep 30.
- [[sase_post_blog#^epic-roadmap|The epic roadmap for goals]] · 50m.
- [[done/sase_done#^release-v18|The v0.18.0 release]] · 25m.
```

## Self-check (epic step 7), already passed

- **Headline:** 20 words on one physical line. It names work, not Worked or typical, and
  does not start with `A {Nh Nm} week`. It names the two projects instead of writing
  "two", so it reads whole in a parent note.
- **Shipped:** 5 bullets, the `^prj` completions first, then finished work grouped by
  project. No `gtd_daily` or other routine row. The `bob_gtd` tasks are project work,
  not routine rows, and they are distinct from the still-open `bob_gtd#^prj` under
  Carried forward. No item appears in two parts.
- **Carried forward:** 3 items, all worked this week and open on Oct 4, chosen by
  significance: the week's started project, the goals roadmap, and the release. No "also
  carried in W39" marks, because W39's v1 Review has no carried part.
- **Questions:** none. No fact checked this week supports a question the records cannot
  answer that also differs from the W39 and v1 W40 Questions. The `sase_sites` drop is
  the v1 Question, and the gate-shells blog topic is already recorded in
  `sase_blog.md`'s "Gates → Plans → Epics" section.
- **Left-out parts:**
  - Intentions, because this note's Intentions is empty.
  - Where the time went, because it would add nothing to Shipped. gtd's 15% is routine
    daily review, and the strip already shows bob at 48% and sase at 37%.

## Claims left out

- **v1 claims dropped:**
  - The `gtd_daily` time clause and "Review closed 0 of 5", because they are routine.
  - "+more of 90 Done", because v2 has no Done tail. The `decision-web`,
    `bob_gtd#^diagnostics`, and `bob_gtd#^keymaps` completions it named are left out for
    length; all are verifiable.
  - The three v1 Questions.
- **Not named:**
  - The `sase_sites#^prj` cancellation and the dropped `#ref` reads. Neither has a
    recorded reason, the v1 already asked about them, and the brief hides 23 drops in a
    "+23 more" tail.
  - The `+3 more` still-open tasks.
  - Blocked `^prj` lines. The brief keeps no blocked history.
- **Left out for length or the 3-item cap, all verifiable:**
  - Still open on Oct 4 with time: `bob#^better-roadmaps` · 46m and
    `sase_agent_history#^read-research` · 46m.
  - The "sasos" rename, cancelled Oct 4 with Bryan's reason "See the sasos research
    recommendation."
  - The decks-related `p` keymap and card blocks (Oct 3, no time shown), and the smaller
    sase and bob completions.
  - Bryan's Sep 28 release log line, "Launched `sase-1c1`!".
- **Checked and not a W40 claim:** Bryan's Oct 3 "Need to verify this still!" belongs to
  `done/sase_done#^hold`, which closed Oct 11, in W41.

## Coder steps

1. Read `~/bob/AGENTS.md`, then run `git -C ~/bob status --short`. Leave every
   pre-existing change alone.
2. Re-check the state:
   - Confirm `2026-W40` is still `drafted` in `bob periods list -k week -f json`, and
     that its Headline is still the old v1 one.
   - Confirm `git -C ~/bob log --format='%h %s' -- 2026/2026-W40.md` still lists only
     `cf0e7a67d`, `8091ae2bd`, and `ff499d867`.

   If the state is `edited`, `reflected`, `empty`, or open, or the note has any other
   commit, stop without writing and report what changed. Never use `-F` for this week.

3. In a fresh `mktemp -d` directory, outside `~/bob` and outside the bob-cli workspace,
   write the exact Review text above to a file. Dry-run it with
   `bob periods draft 2026-W40 -i <file> -n -f json`. Expect `ok: true` and 177 words.
   If it now fails, change the text only as much as the fix needs, and say what changed.
4. Save with `bob periods draft 2026-W40 -i <file> -f json`. Then confirm three things:
   - `bob periods list -k week -f json` shows `2026-W40` as `drafted`, with the new
     Headline whole;
   - `git -C ~/bob diff -- 2026/2026-W40.md` touches only `## Review` and bob's
     `review_digest` / `review_drafted` frontmatter;
   - nothing else in the vault changed.
5. Commit only that note:
   `git -C ~/bob add 2026/2026-W40.md && git -C ~/bob commit -m "periods: redraft 2026-W40 review under v2"`.
   If `bob vault-sync` already auto-committed the note, do not make an empty commit.
   Instead, report the sync commit's hash, and confirm with `git -C ~/bob show --stat`
   that it holds only `2026/2026-W40.md`. Never use `/sase_git_commit` in the vault.
6. Step 11 check (report only): read the `**Carried forward:**` part of
   `2026/2026-W41.md`'s `## Review`. At planning time it carried three items, none
   marked "also carried in W40", so there was no mismatch:
   - `done/sase_done#^release-v18`;
   - `sase_pre_blog#^dynamic-agents-md`;
   - `done/job_done#^update-linkedin`.

   Re-read it, and report any "also carried in W40" mark whose item is not in W40's new
   Carried forward. Do not edit W41.

   Also report one gap in the other direction. W40's new Carried forward now includes
   `^release-v18`, which W41 also carries without an "also carried in W40" mark. W41 was
   drafted while W40 was still v1, so no mark was possible then. Record it on the bead
   as
   `sase bead note bob-cli-j4.5 'PROPOSED FOLLOW-UP: Optionally mark the v0.18.0 release item in the W41 Review Carried forward as "also carried in W40" — the W41 v2 draft predates the W40 v2 draft that carries ^release-v18; Bryan decides whether to hand-edit or leave it'`.

7. Close only the phase bead:
   - Run `sase bead epic-symbols bob-cli-j4.5` and resolve any leftovers.
   - Run `sase bead close bob-cli-j4.5 --note "<what you verified>"`. Put the commit
     hash, the word count, the no-`-F` state check, and the step 6 result in the note.
   - Do not close epic `bob-cli-j4` or open new beads.
8. Reply with:
   - the new Headline next to the old one;
   - the word count;
   - the claims left out (above);
   - that this week's Intentions and W39's Reflection are both empty;
   - the vault commit hash;
   - the step 6 result.

## Out of scope

- Every other vault note: the W40 callout, `## Intentions`, `## Reflection`, W39, W41,
  and tasks.
- All bob-cli files.
