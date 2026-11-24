---
tier: tale
title: Add a "Why I left Google?" section to the job_2026 note
goal:
  ~/bob/job_2026.md ends with a short, interview-ready "Why I left Google?" H2 built on
  the voluntary-exit research.
size: small
decisions:
  extras:
    ask:
      Should the section include the probe follow-ups and the one-line rules strip below
      the answer?
    choices:
      full:
        Answer, one-liner, four probe follow-ups, a rules strip, and sources (about 400
        words)
      answer_only:
        Only the answer, the one-liner, and sources (about 225 words); the playbook
        keeps everything else
    default: full
    why:
      The "laid off?" probe is where a live answer most often goes wrong, so it should
      be one glance away.
    answer: full
  sase_after_hire:
    ask:
      Should the follow-ups commit to "sase stays my personal toolkit on my own time;
      the job gets my working hours"?
    choices:
      include:
        Keep that follow-up; it states a real boundary you will repeat word for word in
        every interview
      omit:
        Drop that follow-up until you have decided what sase's future is after a hire
    default: include
    why:
      Research ranks it the top flight-risk counter; it sets a boundary, not
      abandonment.
    answer: include
proposed_by: bbugyi200.apollo.fv
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-24 08:27:39
status: wip
---

# Add a "Why I left Google?" section to `~/bob/job_2026.md`

## Goal

Add one short H2 section, `## Why I left Google?`, to Bryan's Bob vault project note
`~/bob/job_2026.md`. It is the quick reference he glances at before or during an
interview. It holds the answer to say, a one-line version, and only the guardrails that
matter live.

The section closes the loop on the completed `^why-leave-goog` task in that note ("Plan
answer to 'Why did you leave Google?'!"). Three linked research reports back that task.
All three are Bob reference notes under `~/bob/ref/chat/`, with their source Markdown
preserved under `~/bob/.bob/ref-sources/chat/`.

## Source of truth (already resolved; do not re-research)

- **`google_exit_framing_verdict`** (the newest report, captured 2026-11-23 14:44) wins.
  Bryan **left Google voluntarily in May 2026** and was not let go. Lead with "I left".
  Never say "Google and I parted ways", and never script "I was let go".
- **`why_leave_google_answer_playbook`** agrees with the verdict. It supplies the
  one-liner, the follow-up bank, and the phrasing rules.
- **`why_leave_google_spoken_answer`** is **superseded**. It was built on a stale
  involuntary premise ("parted ways", "I was let go"). Do not use or link any of its
  scripts.

The verified facts used below come from those reports and nowhere else:

- Ad Manager, Jun 2022 – May 2026.
- sase's first commit was on 2026-02-14 (nights and weekends) and it went full-time in
  May.
- Bloomberg reported in Apr 2026 that most Google engineers could not use Claude Code or
  Codex on work code.
- In Sep 2026, Google opened Claude to engineers through Antigravity.

Do not add other facts, numbers, or claims. In particular, add no commit or line counts,
no Gemini quality verdict, and no MCP-server claim.

## Placement

- Append the section at the **end of the file**, after the `## Tasks` section and its
  `### Done & Canceled` subsection. Bryan's own earlier `## Cover Letter` H2 lived in
  that same spot, which he removed on 2026-11-24. Vault history shows Bob's task writers
  correctly add `###` subsections under `## Tasks` before a trailing custom H2. Do not
  place the new section between the `^prj` line, `## Projects`, and `## Tasks`.
- Use the heading text exactly as Bryan wrote it: `## Why I left Google?`.
- Keep exactly one blank line before the heading. End the file with a single trailing
  newline.
- Do not touch the frontmatter, the `^prj` line, the `## Projects` callout, the task
  counts line, or any task or relation line. Bob owns those.
- Do not link the `^why-leave-goog` task block. `bob task archive` will move that done
  task to `done/job_2026_done`. Link the two reference note files instead.

## Exact content (`extras = full`, `sase_after_hire = include`)

Insert this text verbatim:

```markdown
## Why I left Google?

> [!quote] The answer (~45 seconds, then stop; swap the last clause per company) I left
> Google in May to go full-time on a question I couldn't pursue there. I'd had a good
> four years on Ad Manager — publisher-facing launches, shard migrations, API release
> on-call. But I'd become convinced that coding agents would soon do most of the typing,
> and that the engineer's job was shifting to planning, reviewing, and landing work they
> didn't type. That was hard to explore inside Google: at the time, most engineers
> couldn't use outside agents like Claude Code or Codex on work code, and on a
> revenue-critical ads system that caution was reasonable. So I started sase nights and
> weekends in February and went all-in in May — an open-source harness that runs coding
> agents from different providers as one team, with reviewable plans and human approval
> gates. I learned what I set out to learn solo. Now I want to do this on a product team
> with real users, for the long haul, and `<COMPANY>` is the production version of
> exactly that problem.

**One-liner** (forms, recruiters): "I left Google voluntarily in May to work full-time
on agentic software engineering, and since then I've built sase, an open-source harness
for running teams of coding agents."

**If they probe** (one breath each, then back to the work):

- _Laid off or let go?_ "No, I resigned. I wanted to work on this full-time."
- _Did you build it while at Google?_ "Nights and weekends from February, on my own
  time, on public GitHub. By May it deserved my full attention."
- _Google lets engineers use Claude now._ "Yes, since September, through Antigravity,
  and I think that was the right call. In May I didn't know if or when that would
  happen."
- _What happens to sase if you join?_ "It was a way to answer a question, not a company.
  It stays my personal toolkit on my own time; the job gets my working hours."

**Rules:** ~20% Google, ~80% forward · end on _this_ team · never "parted ways" or "took
some time off" · never volunteer "I wasn't fired" · no Gemini dunk; Ad Manager's caution
was reasonable · agents write _most_ of the code, never "all" · one plain sase sentence:
no jargon, no MCP-server claim, lessons over line counts

**Sources:** [[ref/chat/google_exit_framing_verdict|Exit framing verdict]] ·
[[ref/chat/why_leave_google_answer_playbook|Answer playbook]]
```

> [!decision] extras = answer_only Keep the heading, the `[!quote]` callout, the
> **One-liner** paragraph, and the **Sources** line. Delete the **If they probe** block
> and the **Rules** line. Keep one blank line between the remaining blocks. The
> `sase_after_hire` decision then has no effect.

> [!decision] sase_after_hire = omit Delete only the
> `*What happens to sase if you join?*` bullet. The other three probe bullets stay
> unchanged.

## Steps

1. Re-read `~/bob/job_2026.md` immediately before editing. Bryan edits this note from
   his Mac, and vault sync may have pulled a newer copy. If a `## Why I left Google?`
   heading already exists, stop and report back rather than adding a duplicate.
2. Append the section chosen by the decisions at end of file, following the placement
   rules above. Do not reflow, rewrap, or otherwise edit any other line.
3. Verify:
   - `grep -n '^## ' ~/bob/job_2026.md` lists exactly `## Projects`, `## Tasks`, and
     `## Why I left Google?`, in that order.
   - `git -C ~/bob diff -- job_2026.md` shows only added lines at the end of the file.
     There must be no removed lines, apart from the old final line regaining its
     newline.
   - `tail -c1 ~/bob/job_2026.md | xxd` shows `0a`.
   - Both wikilink targets exist: `~/bob/ref/chat/google_exit_framing_verdict.md` and
     `~/bob/ref/chat/why_leave_google_answer_playbook.md`.
   - The inserted text contains no "parted ways", no "let go" except inside the probe
     question, and no "all of the coding".
4. Run `bob vault-sync` once so the edit reaches the Mac. Confirm that
   `bob vault-sync status --json` shows `"last_error": null` and an empty `conflicts`
   list. Do not hand-commit in `~/bob`; vault sync owns those commits.

## Out of scope

- Make no changes to the bob-cli repository. This plan edits only the vault note, so the
  implementing workspace is expected to end with no diff.
- Do not edit the three reference notes or the `^why-leave-goog` / `^ref-*` task lines.
- Two older artifacts still script "parted ways": the October `google_exit_story`
  research report and the Yogi intro-call prep report. Correcting or archiving them is a
  separate cleanup for Bryan to decide on, so leave them alone.
- Restoring the cover letter is out of scope. Bryan removed it from this note on
  2026-11-24.
