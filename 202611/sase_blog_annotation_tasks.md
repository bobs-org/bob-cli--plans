---
tier: tale
title: Create one sase_blog task per sase_blog_261116 annotation from tmux_ai_window on
goal:
  ~/bob/sase_blog.md gains 17 open, unlinked, block-ID'd tasks, one per annotation of
  ref/notes/sase_blog_261116.md from tmux_ai_window through the last, ready to be pulled
  into the project Bryan promotes from his review task.
size: small
decisions:
  relation:
    ask:
      Should each new task link back to the SASE Blog · 2026-11-16 reading task with an
      `inspired by` relation?
    choices:
      link:
        Each task gets an `inspired by` LINKS child; the reading task's LINKED FROM
        lists all 17
      plain:
        One-line tasks with only the 🔖 backlink; the reading task is left untouched
    default: link
    why:
      Matches the configured annotation follow-up convention (annotation_task_relation)
    answer: link
proposed_by: bbugyi200.apollo.fc
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-20 18:40:58
status: wip
---

# Create one `sase_blog.md` task per `sase_blog_261116` annotation, starting at `tmux_ai_window`

## Goal

Add 17 new open Obsidian tasks to `~/bob/sase_blog.md`, one for each annotation in
`~/bob/ref/notes/sase_blog_261116.md`'s managed `## Highlights` region. Start with the
`tmux_ai_window` annotation (`^h-10e35ebe0f8b`) and go through the last live annotation
(`^h-1f846c42620b`). Each task is the unit of work "turn this annotation into a
[[sase_blog_zettel]] record (or fold it into one)". Bryan's WIP task
`^sase-blog-annotations` ("Review … annotations and convert to [[sase_blog_zettel]]!")
logged the purpose at 2026-11-20 18:33: _"apollo agent to create tasks for each
annotation (so I can pull them into promoted project)"_. The new tasks therefore have to
be ordinary, open, unhidden, unlinked, top-level `## Tasks` rows with block IDs, so he
can later promote that review task and pull these rows into the new project.

This is a vault-only change. Nothing changes in the bob-cli repo, nothing is committed
there, and no code or docs are touched.

## Source annotations (in scope)

The ref note has 21 live annotations. The first four (gastown/beads, AI-slop definition,
PIW goals, AI-slop categories) come before `tmux_ai_window` and are **out of scope**.
The `### Removed highlights` entry `^h-48aa90d5cf5d` is a tombstone, not an annotation,
so it is also excluded. The 17 annotations in scope, in document order, are below. The
"Comment" column is Bryan's handwritten comment on each annotation, which is his section
hint for the zettel album.

| #   | Block             | Annotation (abridged)                                               | Comment                   | Task block ID                   |
| --- | ----------------- | ------------------------------------------------------------------- | ------------------------- | ------------------------------- |
| 1   | `^h-10e35ebe0f8b` | tmux_ai_window                                                      | turns                     | `zettel-tmux-ai-window`         |
| 2   | `^h-1e186c040833` | Some good examples of using stashed prompts!                        | prompts                   | `zettel-stashed-prompts`        |
| 3   | `^h-b9500948b43c` | Agent control flow (forks/waits/queues)                             | prompts                   | `zettel-agent-control-flow`     |
| 4   | `^h-d32a08558083` | Joke: the disable-Vim-mode promise is an ulterior motive            | prompts                   | `zettel-vim-mode-joke`          |
| 5   | `^h-48a087f971ef` | Include a "Why a TUI?" section in blog post!                        | what                      | `zettel-why-a-tui`              |
| 6   | `^h-7a1a64062df9` | I'd implement skills as (procedural) memory                         | memory                    | `zettel-skills-as-memory`       |
| 7   | `^h-fc0d6bdfdcc9` | Mention reads and snips as potential future sidecar repos!          | artifacts                 | `zettel-reads-and-snips`        |
| 8   | `^h-d1dd0a662709` | Example of how useful `sase artifact read` / the plans repo is      | artifacts                 | `zettel-artifact-read`          |
| 9   | `^h-2d5c8cf4204d` | Quote from where_do_hub_and_struct_notes_go                         | future (related to goals) | `zettel-hub-note-quote`         |
| 10  | `^h-35c3838ad36e` | FUTURE: merge agent project memory with project docs (diataxis?)    | future/memory             | `zettel-memory-and-docs`        |
| 11  | `^h-781ea1441011` | FUTURE: goals much smaller and much larger; a goal outlives an epic | future                    | `zettel-goal-sizes`             |
| 12  | `^h-06abbf6c7ff7` | Project memory implemented every wrong way possible                 | memory                    | `zettel-memory-every-wrong-way` |
| 13  | `^h-29f8d53b63a8` | FUTURE: Mention finalizer ideas!                                    | future                    | `zettel-finalizer-ideas`        |
| 14  | `^h-710a900c8dd3` | QUOTE: competing labs' frontier models swarm one task               | turns                     | `zettel-swarm-quote`            |
| 15  | `^h-7cc49634a570` | SASE's purpose: build the next SASE from scratch (first "myth")     | what                      | `zettel-first-myth`             |
| 16  | `^h-f9fb3f8a55fa` | Funny diagram: stick figure hiding models in the closet             | intro                     | `zettel-closet-diagram`         |
| 17  | `^h-1f846c42620b` | Start with a timeline of software engineering transformation        | what (instead of intro?)  | `zettel-swe-timeline`           |

## Task shape

Every task is written by `bob capture`, the vault's canonical task writer, as one
17-item list capture routed to `@sase_blog`. Each row:

- is a Ready `- [ ] #task` row with `[created::2026-11-20]` (capture stamps the run
  date; if this runs on a later day, that later date is correct);
- reads `Convert the "<short label>" annotation to a zettel (<comment>)!` ("zettel" is
  the album's record noun). The comment is carried verbatim, except that "future
  (related to goals)" and "what (instead of intro?)" are written as
  `future, related to goals` and `what, instead of intro?` to avoid nested parentheses;
- carries the 🔖 annotation backlink `[[ref/notes/sase_blog_261116#^h-…|🔖]]`, the same
  form Highlights sync uses for annotation follow-ups (see
  `docs/highlights-ref-sync.md`);
- ends with a stable `^zettel-<slug>` block ID, so Bryan can pull it into the promoted
  project with a capture `- ^<id>` / `- :<id>` bullet or the `- ^^` Move card;
- lands in the unheaded Ready intake directly under `## Tasks`: below the status-count
  badge row and above `### Next & In Progress`, in annotation order.

> [!decision] relation = link Each row gets one child,
> `- 🪢 **LINKS:** inspired by [[#^ref-sase-blog-261116]]`. Capture also rewrites the
> closed reading task's existing
> `🪢 **LINKED FROM:** [[#^sase-blog-annotations]] (💬 mentions)` line to append the 17
> `[[#^zettel-…]] (inspired by)` entries. `inspired by` is not a cancel, complete, or
> successor label in `~/.config/bob/config.yml`, so the relation stays inert.

> [!decision] relation = plain Rows have no children. Drop the trailing
> ` ~sase_blog:ref-sase-blog-261116|"inspired by"` token from every bullet in the draft
> below. Nothing else changes.

Deliberately **not** done:

- **No Depends-On links between the new tasks and `^sase-blog-annotations`.** That task
  is the one Bryan will promote. Its Depends-On line would move onto the new `^prj`, and
  under [[decisions:projects-block-from-prj-line]] the project would then be Blocked by
  its own residents, tagging every one `#prj_is_blocked`.
- No `:id` / Pomodoro Task Links (they would raise the tasks to Next and plan them
  today), no `[type::]`, priority, or schedule fields (the note's WRITE default
  applies).
- No `[h:: …]` processed IDs. Those are Highlights-sync dedupe markers for `#task`
  bullets typed in annotation comments, and these tasks did not come from such bullets.
- No edits to the ref note, its sidecar (`lib/notes/sase_blog_261116.md`), the PDF, the
  zettel album or its records, or the `^sase-blog-annotations` task. No zettel records
  are created.

## Implementation steps

1. **Pull the latest vault.** Bryan edits `sase_blog.md` from the Mac. Run
   `bob vault-sync` once (it shares `bob_sync.lock` with the background service), then
   confirm `bob vault-sync status --json` shows `"last_error": null` and empty
   `conflicts`.
2. **Preconditions** (stop and report if any fails):
   - The 17 new IDs are not present yet. Note that the existing `^zettel-require-type`
     task also starts with `zettel-`, so a bare `\^zettel-` grep is wrong. Use the exact
     set:
     `grep -cE ' \^zettel-(tmux-ai-window|stashed-prompts|agent-control-flow|vim-mode-joke|why-a-tui|skills-as-memory|reads-and-snips|artifact-read|hub-note-quote|memory-and-docs|goal-sizes|memory-every-wrong-way|finalizer-ideas|swarm-quote|first-myth|closet-diagram|swe-timeline)$' ~/bob/sase_blog.md`
     must print `0`. If it prints anything else, some tasks already exist: do not
     re-run.
   - `grep -n '\^ref-sase-blog-261116$' ~/bob/sase_blog.md` finds the reading task.
   - All 17 `^h-…` IDs in the table still appear as standalone block lines in
     `~/bob/ref/notes/sase_blog_261116.md`, above `### Removed highlights`.
3. **Write the draft** to a file in a fresh `mktemp -d` directory. Write it with a
   quoted heredoc (`<<'EOF'`) so the shell expands nothing. Use exactly this content
   (relation = link):

   ```text
   @sase_blog
   - Convert the `tmux_ai_window` annotation to a zettel (turns)! [[ref/notes/sase_blog_261116#^h-10e35ebe0f8b|🔖]] ^zettel-tmux-ai-window ~sase_blog:ref-sase-blog-261116|"inspired by"
   - Convert the "good examples of using stashed prompts" annotation to a zettel (prompts)! [[ref/notes/sase_blog_261116#^h-1e186c040833|🔖]] ^zettel-stashed-prompts ~sase_blog:ref-sase-blog-261116|"inspired by"
   - Convert the "Agent control flow (forks/waits/queues)" annotation to a zettel (prompts)! [[ref/notes/sase_blog_261116#^h-b9500948b43c|🔖]] ^zettel-agent-control-flow ~sase_blog:ref-sase-blog-261116|"inspired by"
   - Convert the "never let them disable Vim mode" joke annotation to a zettel (prompts)! [[ref/notes/sase_blog_261116#^h-d32a08558083|🔖]] ^zettel-vim-mode-joke ~sase_blog:ref-sase-blog-261116|"inspired by"
   - Convert the "Why a TUI?" section annotation to a zettel (what)! [[ref/notes/sase_blog_261116#^h-48a087f971ef|🔖]] ^zettel-why-a-tui ~sase_blog:ref-sase-blog-261116|"inspired by"
   - Convert the "skills as (procedural) memory" annotation to a zettel (memory)! [[ref/notes/sase_blog_261116#^h-7a1a64062df9|🔖]] ^zettel-skills-as-memory ~sase_blog:ref-sase-blog-261116|"inspired by"
   - Convert the "`reads` and `snips` as future sidecar repos" annotation to a zettel (artifacts)! [[ref/notes/sase_blog_261116#^h-fc0d6bdfdcc9|🔖]] ^zettel-reads-and-snips ~sase_blog:ref-sase-blog-261116|"inspired by"
   - Convert the "`sase artifact read` and the plans artifact repo" annotation to a zettel (artifacts)! [[ref/notes/sase_blog_261116#^h-d1dd0a662709|🔖]] ^zettel-artifact-read ~sase_blog:ref-sase-blog-261116|"inspired by"
   - Convert the "hub note vs. structure note" quote annotation to a zettel (future, related to goals)! [[ref/notes/sase_blog_261116#^h-2d5c8cf4204d|🔖]] ^zettel-hub-note-quote ~sase_blog:ref-sase-blog-261116|"inspired by"
   - Convert the "merge agent project memory with project docs (diataxis?)" annotation to a zettel (future/memory)! [[ref/notes/sase_blog_261116#^h-35c3838ad36e|🔖]] ^zettel-memory-and-docs ~sase_blog:ref-sase-blog-261116|"inspired by"
   - Convert the "SASE goals are much smaller and much larger; a goal outlives an epic" annotation to a zettel (future)! [[ref/notes/sase_blog_261116#^h-781ea1441011|🔖]] ^zettel-goal-sizes ~sase_blog:ref-sase-blog-261116|"inspired by"
   - Convert the "project memory implemented every wrong way possible" annotation to a zettel (memory)! [[ref/notes/sase_blog_261116#^h-06abbf6c7ff7|🔖]] ^zettel-memory-every-wrong-way ~sase_blog:ref-sase-blog-261116|"inspired by"
   - Convert the "finalizer ideas" annotation to a zettel (future)! [[ref/notes/sase_blog_261116#^h-29f8d53b63a8|🔖]] ^zettel-finalizer-ideas ~sase_blog:ref-sase-blog-261116|"inspired by"
   - Convert the "frontier models from competing labs swarm one task" quote annotation to a zettel (turns)! [[ref/notes/sase_blog_261116#^h-710a900c8dd3|🔖]] ^zettel-swarm-quote ~sase_blog:ref-sase-blog-261116|"inspired by"
   - Convert the "SASE's purpose: build the next SASE from scratch (the first myth)" annotation to a zettel (what)! [[ref/notes/sase_blog_261116#^h-7cc49634a570|🔖]] ^zettel-first-myth ~sase_blog:ref-sase-blog-261116|"inspired by"
   - Convert the "stick figure hiding frontier models in the closet" diagram annotation to a zettel (intro)! [[ref/notes/sase_blog_261116#^h-f9fb3f8a55fa|🔖]] ^zettel-closet-diagram ~sase_blog:ref-sase-blog-261116|"inspired by"
   - Convert the "timeline of software engineering transformation" annotation to a zettel (what, instead of intro?)! [[ref/notes/sase_blog_261116#^h-1f846c42620b|🔖]] ^zettel-swe-timeline ~sase_blog:ref-sase-blog-261116|"inspired by"
   ```

   Grammar notes, verified with `bob capture --dry-run` and a sandboxed real capture
   while planning:
   - In list form, the `~` relation must be the **last** token, after `^id`. If `~`
     comes before `^id`, it is not treated as a directive and leaks into the task text
     verbatim.
   - `[[…]]` links and backtick spans are never scanned for markers, so the `#^h-…`
     inside the 🔖 link is safe.
   - Do not put `<`, `>`, `&`, `*`, `=`, `+`, or `%` into any label.

4. **Dry run:** `bob capture --dry-run --no-clip < "$draft"`. Expect
   `list  sase_blog.md · 17 tasks`, 17 `would capture` items, and (relation = link) 17
   `🪢 would link to … · inspired by` lines. Also run it with `-f json` and check:
   `"ok": true`, 17 entries in `captures`, every `block_id` from the table in order, and
   no `task_line` containing `~` or `|"`. If capture refuses (for example a duplicate
   block ID), stop and report. Never fall back to hand-editing the note.
5. **Capture for real:** `bob capture --no-clip < "$draft"`. Expect
   `✓ captured  17/17  sase_blog.md`. The write is all-or-nothing and preimage-checked.
   If Bryan's Mac edits race it and capture reports a changed note, rerun steps 2–5
   once. The duplicate-ID check makes a rerun after a successful write refuse instead of
   duplicating tasks.
6. **Verify** in `~/bob`:
   - `git -C ~/bob status --short` / `git -C ~/bob diff --stat` show only `sase_blog.md`
     changed. Relation = link gives 36 insertions and 1 deletion (the rewritten LINKED
     FROM line); relation = plain gives 17 insertions.
   - The exact-ID grep from step 2, run with `-nE` instead of `-cE`, lists 17 rows in
     table order. All of them sit between the `## Tasks` badge row and
     `### Next & In Progress`, and each is a `- [ ] #task Convert the …` row. The
     pre-existing `^zettel-require-type` row is unchanged.
   - Relation = link: each row is followed by exactly
     `\t- 🪢 **LINKS:** inspired by [[#^ref-sase-blog-261116]]`, and the reading task's
     LINKED FROM line still starts with `[[#^sase-blog-annotations]] (💬 mentions)` and
     now also lists the 17 `[[#^zettel-…]] (inspired by)` entries.
   - The badge row may still read `⚪ 0 open` until the next `bob task reconcile` run
     (Obsidian plugins / tick). That is expected; do not run a vault-wide reconcile just
     for this.
7. **Publish:** run `bob vault-sync` again, and confirm `bob vault-sync status --json`
   reports `"last_error": null` with no `conflicts` or `task_losses`.
8. Remove the temporary draft directory.

## Expected result (excerpt, relation = link)

```markdown
## Tasks

[`⚪ 0 open`](#Tasks) · [`🔵 1 next/wip`](#Tasks#Next%20&%20In%20Progress) · …

- [ ] #task Convert the `tmux_ai_window` annotation to a zettel (turns)!
      [[ref/notes/sase_blog_261116#^h-10e35ebe0f8b|🔖]] [created::2026-11-20]
      ^zettel-tmux-ai-window
  - 🪢 **LINKS:** inspired by [[#^ref-sase-blog-261116]]
- [ ] #task Convert the "good examples of using stashed prompts" annotation to a zettel
      (prompts)! [[ref/notes/sase_blog_261116#^h-1e186c040833|🔖]] [created::2026-11-20]
      ^zettel-stashed-prompts - 🪢 **LINKS:** inspired by [[#^ref-sase-blog-261116]] …
- [ ] #task Convert the "timeline of software engineering transformation" annotation to
      a zettel (what, instead of intro?)!
      [[ref/notes/sase_blog_261116#^h-1f846c42620b|🔖]] [created::2026-11-20]
      ^zettel-swe-timeline
  - 🪢 **LINKS:** inspired by [[#^ref-sase-blog-261116]]

### Next & In Progress
```

## Known consequences to report back

- New capture rows carry no `fresh` stamp, so until Bryan triages or pulls them, the 17
  rows show up in the `]s` walk's NEW tier.
- 17 Ready rows exceed the default per-note Ready soft cap
  ([[decisions:note-ready-cap-counts-the-lane]]), so the Mac `w` mark may show lane
  excess for `sase_blog` (and later for the promoted project) until the rows are worked
  or moved.
- The final report should list the 17 created block IDs and the exact `bob capture`
  summary line.
