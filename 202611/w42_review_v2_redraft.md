---
tier: tale
size: small
title: Save the approved v2 Review of 2026-W42
goal:
  2026/2026-W42.md carries the approved work-first v2 Review, saved through `bob periods
  draft` without -F and committed alone in the vault.
bead: bob-cli-j4.3
proposed_by: bbugyi200.athena.bob-cli-j4.3
proposed_with: claude/opus
create_time: 2026-11-09 11:22:33
status: wip
---

- **PARENT:**
  [202611/q4_weekly_reviews_v2_redraft.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/q4_weekly_reviews_v2_redraft.md)
- **BEAD:**
  [bob-cli-j4.3](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-j4/bob-cli-j4.3.md)

# Plan: Save the approved v2 Review of 2026-W42 (bead `bob-cli-j4.3`)

Phase `redraft-w42` of epic `bob-cli-j4` (plan
`202611/q4_weekly_reviews_v2_redraft.md`). Approving this plan approves the exact Review
text below. The coder saves it through `bob periods draft`, commits only
`2026/2026-W42.md` in the vault, and replies. No bob-cli file changes.

## Findings from planning (steps 1–8 of the epic procedure)

- **Vault state:** `git -C ~/bob status --short` was clean. The last W42 commits are
  `cf0e7a67d` (v2 callout refresh) and `ead3ef9d8` (the v1 draft).
- **Review state:** `bob periods list -k week -f json` shows `2026-W42` as `drafted`. So
  the save runs **without `-F`**.
- **Brief:** `bob periods brief 2026-W42 -f md` (brief v2) reports the week closed and
  drafted. Facts: Worked 33h 40m (the same as typical), Done 179, Created 244, Review
  closed 7 of 7. Three `^prj` completions: `job_apply_to_databricks`,
  `sase_agent_history`, `job_update_cv`. "Kept 3+ times" is None. Blocked projects are
  "Omitted — blocked state has no history."
- **Neighbours:**
  - This week's `## Intentions` is empty, and W41's `## Reflection` is empty.
  - W41's Review is still v1 and has no Carried forward or Still open part, so nothing
    here gets an "also carried in W41" mark. W41's Questions (Done vs typical, the
    gtd_daily share of Tue 6 / Wed 7, `sase_art_agents` dropped) are not repeated.
  - The v1 W42 Questions (Created vs Done, what the root-access check found,
    `sase_bug_bash`/`sase_message_boards` dropped vs `sase_memory_plan` started) are not
    repeated either.
- **Claims verified in notes:**
  - `^prj` completions: `job_update_cv.md` and `job_apply_to_databricks.md` show
    `[completion:: 2026-10-12]`; `sase_agent_history.md` shows
    `[completion:: 2026-10-18]`.
  - Applications: Tower and Tailscale, Figma, and Two Sigma closed Oct 14–15 in
    `done/job_2026_apply_29_done.md`; Warp closed Oct 13 in `done/job_done.md`. Bryan's
    "Applied to Ramp, Clay, and LangChain!" is a log line in `2026/20261013.md` (under
    the Open Roles reading Pomodoro); it has no block id, so the link goes to the note.
  - Plex hack: `dev_plex_hack_patch.md` shows `sudo`, the credentials checklist, and the
    root-access check closed Oct 17; the Plex update and the postmortem closed Oct 18.
  - bob: `capture-templates` and `star-block-capture` closed Oct 15, and
    `migrate-depends-scheduled` closed Oct 13, in `done/bob_done.md`.
  - `ref-sase-launch-post-outline` closed Oct 18 in `done/sase_blog_done.md`;
    `fix-core-dump` closed Oct 14 in `done/sase_done.md`.
  - Carried forward items, each open on Oct 18 (checked in the last vault snapshot
    before Oct 19, `67dc4f288`):
    - `write-introduction` was `[?]` in `sase_blog.md`; it was cancelled only on Oct 26.
    - `sase-project-stats` was `[/]` in `sase.md`; it closed Oct 20. Bryan's Oct 18
      work-log line is quoted verbatim.
    - `dev_plex_hack_patch#^prj` was `[ ]`, and it is still open.
  - The Question's fact: in the Oct 18 snapshot (`58362d297`), `^no-plain-secrets` and
    `^router` are both `[?]` with `dependsOn` on the incident-report ref, which was
    still open (`[*]`, in `bob.md` then). That report closed only on Nov 1.
- **Every duration** below is copied exactly as the brief displays it.
- **Dry run:** `bob periods draft 2026-W42 -i <file> -n -f json` returned
  `{"id":"2026-W42","ok":true,"review_state":"drafted","schema_version":1,"words":193}`.

## Headlines

- **Old (v1, joined onto one line):** A 33h 40m week, the same as the typical week,
  closed the CV and DataBricks projects while bob and job applications led the time.
- **New (v2):** Finished the CV and applied to DataBricks, Tower, Tailscale, and others;
  patched athena after its Plex hack; gave bob capture templates.

## Exact Review text to save (193 words per bob)

Save this byte for byte, from the `**Headline:**` line to the end of the Question. In
the source, the Headline and every bullet or numbered item are **one physical line
each**. If this plan's rendering wrapped any of them, join each continuation line back
onto its bullet with a single space before saving: every non-blank line of the saved
file starts with `**`, `- `, or `1. `.

```markdown
**Headline:** Finished the CV and applied to DataBricks, Tower, Tailscale, and others;
patched athena after its Plex hack; gave bob capture templates.

**Shipped:**

- Closed [[job_update_cv#^prj|Update CV]] and
  [[job_apply_to_databricks#^prj|Apply to DataBricks NYC]] Oct 12, and sase's
  [[sase_agent_history#^prj|Agents tab migration]] Oct 18.
- Applications:
  [[done/job_2026_apply_29_done#^apply-tower-tailscale|Tower and Tailscale]] · 1h 15m,
  [[done/job_done#^apply-warp|Warp]] · 20m,
  [[done/job_2026_apply_29_done#^apply-figma|Figma]] · 20m, and
  [[done/job_2026_apply_29_done#^apply-two-sigma|Two Sigma]]; Bryan,
  [[2026/20261013|Oct 13]]: "Applied to Ramp, Clay, and LangChain!"
- Plex hack, Oct 17–18: [[dev_plex_hack_patch#^sudo|`sudo` passwords changed]],
  [[dev_plex_hack_patch#^complete-incidents-athena|exposed-credentials checklist]] ·
  23m, [[dev_plex_hack_patch#^update-plex|Plex updated and set to auto-update]] · 12m,
  and a [[dev_plex_hack_patch#^postmortem|postmortem]].
- [[bob]]: [[done/bob_done#^capture-templates|capture templates]] · 18m and
  [[done/bob_done#^star-block-capture|`*` to block a task]] · 10m;
  [[done/bob_done#^migrate-depends-scheduled|`depends_on` and `scheduled` moved back to the project task]]
  · 50m.
- [[sase_blog]]:
  [[done/sase_blog_done#^ref-sase-launch-post-outline|launch-post outline]] read · 1h
  56m; [[sase]]: [[done/sase_done#^fix-core-dump|TUI core-dump crash fixed]] · 30m.

**Carried forward:**

- [[done/sase_blog_done#^write-introduction|The blog post's introduction]] · 35m.
- [[sase_projects#^sase-project-stats|`sase project stats`]] · 25m; Bryan, Oct 18: "The
  epic is still running on apollo (just restarted grok agent using astra)"
- [[dev_plex_hack_patch#^prj|Plex hack patch and postmortem]] · 25m.

**Questions:**

1. [[dev_plex_hack_patch#^no-plain-secrets|Removing plain-text secrets]] and
   [[dev_plex_hack_patch#^router|disabling router port-forwarding]] ended the week
   waiting on
   [[dev_plex_hack_patch#^ref-athena-plex-compromise-incident-report|the incident report]];
   which should come first once it lands?
```

## Self-check (epic step 7), already passed

- The Headline is 21 words on one line. It names work, not Worked or typical, and does
  not start with `A {Nh Nm} week`.
- Shipped has 5 bullets, `^prj` completions first, and none is a `gtd_daily` or other
  routine row. No item appears in two parts: Shipped names the finished Plex tasks, and
  Carried forward names only the still-open Plex `^prj`.
- There is one numbered Question. It is not a typical-delta question, it repeats no W41
  or v1 W42 Question, and it contains no advice or cause.
- No count appears; only durations the brief displays for exactly those rows.
- These parts are left out:
  - Intentions, because the note's Intentions is empty.
  - Where the time went, because it would add nothing to Shipped: bob, job_2026, sase,
    and sase_blog already appear there, and gtd's 11% is routine daily review.

## Claims left out

- **v1 numbers that do not match the brief:** the 1h 50m goals reading (brief: 52m), the
  `tailnet` project at 1h 20m (brief: 32m), and the credentials checklist at 1h 05m
  (brief: 23m).
- **Blocked `^prj` lines:** the brief keeps no blocked history, and the three carried
  items are work that got time at week end.
- **Hidden tails:** nothing from the brief's `+32 more`, `+23 more`, `+17 more`, or
  `+8 more` tails is named.
- **The root-access check's result:** neither its task row nor the Oct 17 daily log
  records what it found, so the Review does not say.
- **Verified but cut for length:** the `job_2026_apply_29` project (open at week end),
  OpenRouter and DataBricks Agent Quality applications, Bryan's Oct 13 "Applied to last
  tier A role (BrainTrust)", saved capture templates, `/` capture, successor prompts,
  project task dependency fixes, the `sase_blog_0` merge, and the `dash_stats` fix.

## Coder steps

1. Read `~/bob/AGENTS.md`, then run `git -C ~/bob status --short`. Leave every
   pre-existing change alone.
2. Confirm `2026-W42` is still `drafted` in `bob periods list -k week -f json`. If it is
   anything else, stop without writing and report it. Never use `-F` for this week.
3. In a fresh `mktemp -d` directory, outside `~/bob` and outside the bob-cli workspace,
   write the exact Review text above to a file (one physical line per Headline, bullet,
   and numbered item). Dry-run it with
   `bob periods draft 2026-W42 -i <file> -n -f json`. Expect `ok: true` and 193 words.
   If it now fails, change the text only as much as the fix needs, and say what changed.
4. Save with `bob periods draft 2026-W42 -i <file> -f json`. Then confirm three things:
   - `bob periods list -k week -f json` shows `2026-W42` as `drafted`, with the new
     Headline whole;
   - `git -C ~/bob diff -- 2026/2026-W42.md` touches only `## Review` and bob's
     `review_digest` / `review_drafted` frontmatter;
   - nothing else in the vault changed.
5. Commit only that note:
   `git -C ~/bob add 2026/2026-W42.md && git -C ~/bob commit -m "periods: redraft 2026-W42 review under v2"`.
   If `bob vault-sync` already auto-committed the note, do not make an empty commit.
   Instead, report the sync commit's hash, and confirm with `git -C ~/bob show --stat`
   that it holds only `2026/2026-W42.md`. Never use `/sase_git_commit` in the vault.
6. Step 11 check (report only): read the `**Carried forward:**` part of
   `2026/2026-W43.md`'s `## Review`. At planning time it carried only
   `sase_blog_done#^ref-sase-blog-writers-block-task-plan` and
   `sase_pre_blog#^ref-pypi-release-reliability-design-and`, with no "also carried in
   W42" mark, so there is no mismatch. Re-read it; report any "also carried in W42" mark
   whose item is not in W42's new Carried forward. Do not edit W43.
7. Close only the phase bead. Run `sase bead epic-symbols bob-cli-j4.3` and resolve any
   leftovers. Then run `sase bead close bob-cli-j4.3 --note "<what you verified>"`. Put
   the commit hash, the word count, and the step 6 result in the note. Record any
   mismatch as `sase bead note bob-cli-j4.3 'PROPOSED FOLLOW-UP: ...'`. Do not close
   epic `bob-cli-j4` or open new beads.
8. Reply with:
   - the new Headline next to the old one;
   - the word count;
   - the claims left out (above);
   - that both this week's Intentions and W41's Reflection are empty;
   - the vault commit hash;
   - the step 6 result.

## Out of scope

- Every other vault note, including the W42 callout, `## Intentions`, `## Reflection`,
  W41, W43, and tasks.
- All bob-cli files.
