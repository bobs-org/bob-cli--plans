---
tier: tale
title: Wave-gated draft-or-cut tasks for every SASE Blog Zettel record
goal:
  sase_blog_drafts.md holds one draft-or-cut task per sase_blog_zettel record, in waves
  of 3-5 tasks chained through one wave task per wave, so each record task has at most
  one prerequisite and blocks only its own wave task.
size: small
decisions:
  wave_split:
    ask:
      Should the waves follow the album's sections, even if a few waves hold only 3
      tasks?
    choices:
      sections:
        7 waves of 3-5 tasks in album section order; no section is split across waves
      even:
        6 waves of 4-5 tasks in walk order; five sections each split across two waves
    default: sections
    why:
      each wave review rereads whole sections, and no wave goes over the 5-task Ready
      cap
    answer: sections
  gate_role:
    ask:
      Should each wave task be a short review of that wave's drafts, rather than a bare
      milestone?
    choices:
      review:
        Each wave task is a short reread of the wave that also routes new records into
        the next wave
      milestone:
        Each wave task is a bare "Finish Wave N" marker, closed as soon as it unblocks
    default: review
    why:
      it follows the earlier Wave 1 review and gives new records from the section-fill
      task a place to go
    answer: review
  close_planning_task:
    ask: Should the agent complete ^create-draft-tasks after the tasks exist?
    choices:
      leave_open: Leave ^create-draft-tasks In Progress so Bryan closes it from Obsidian
      close:
        Complete it with a `!sase_blog_drafts:create-draft-tasks` capture after the wave
        tasks are verified
    default: leave_open
    why: Bryan closes his own WIP tasks, and the close logs the Pomodoro work there
    answer: leave_open
proposed_by: bbugyi200.apollo.fq
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-23 11:18:18
status: wip
---

# Create wave-gated draft tasks for every SASE Blog Zettel record

## Goal

Finish the setup work behind `^create-draft-tasks` in `~/bob/sase_blog_drafts.md`.
Create one Obsidian task for each record in the `sase_blog_zettel` album ("draft or
cut"). Group the tasks into waves of about 5. Each wave has one **wave task** that
depends on every task in its wave and blocks every task in the next wave. Each record
task has at most one prerequisite (the previous wave task) and blocks only one task (its
own wave task):

```text
W1 records ──▶ ^wave-1 ──▶ W2 records ──▶ ^wave-2 ──▶ … ──▶ W7 records ──▶ ^wave-7
 (Ready)       (Blocked)    (Blocked)      (Blocked)          (Blocked)      (Blocked)
```

This is vault work only: no bob-cli code, docs, or memory changes.

## Facts checked while planning (2026-11-23)

- `~/bob/sase_blog_zettel/` holds **28** record notes, all `status: seed`. The album's
  `section` field groups them as follows: Intro 1, What 3, Macros 4, Agent CLIs 3, Turn
  3, Artifacts 5, VCS 2, Not ready 2, Future 5. The Gates, Plugins, Services, How I use
  SASE, Demos, and Conclusion sections have no records yet.
  (`sase_blog.md ^at-least-one-rec-in-sect` is an open task to fill or remove them.)
- `bob ready` reports **cap 5 per note**. Wave 1 tasks start Ready. Each later wave
  becomes Ready only after the previous wave's tasks are closed. So a wave of 5 or fewer
  keeps `sase_blog_drafts` at or under the cap.
- The `^prj` in `sase_blog_drafts.md` is not Blocked: its only prerequisite,
  `sase_blog_annotations__prj`, is done. **Never** add any of the new tasks to `^prj`'s
  Depends-On line. Under `decisions:projects-block-from-prj-line`, an open prerequisite
  there would block every resident, including the new tasks themselves.
- The new tasks inherit type WRITE from `sase_blog` (`theme: WRITE`). Do not write
  `[type::]` fields.
- One `bob capture` list draft creates every task in a single atomic write. A later
  bullet's `&` can name a task an earlier bullet creates, because items plan in order
  against staged content (`docs/capture.md`, "Lists: several tasks or notes at once").
  The exact draft below passed `bob capture --dry-run --no-clip` against the live vault:
  35 items, exit 0. Wave 1 tasks plan as `[ ]`, and every other task plans as `[?]`,
  with derived `[id::]`/`[dependsOn::]` fields and BLOCKS lines.
- `--no-clip` refuses only drafts containing `%c`, so it is safe on the real run.

## Conventions

- **Record task text:** `Draft or cut [[sase_blog_zettel/<stem>|<Title>]]!`. The title
  comes from the record's `title:` frontmatter, which is the album index text.
- **Record block ID:** the stem with `_` → `-`. The one exception is
  `microsoft_ceo_thinks_model_agnostic_architecture_is_good`, which gets
  `^microsoft-ceo-model-agnostic`. Use `^id` (an ordinary Ready task), never `:id`:
  `:id` would raise the task to Next and link it into today's Pomodoro.
- **Wave task block ID:** `^wave-N`.
- **No child bullets** on any new task. Capture and reconcile own the Depends-On and
  BLOCKS lines.

## Waves (default `wave_split = sections`)

| Wave | Sections                         | Records (stem)                                                                                | Tasks |
| ---- | -------------------------------- | --------------------------------------------------------------------------------------------- | ----- |
| 1    | 🪝 Intro, 🧱 What is SASE?       | what_came_before, recursive_sase_improvement, sase_is_many_abstractions, why_a_tui            | 4     |
| 2    | 📜 Macros                        | agent_control_flow, layers_of_prompt_management, model_alias_pools, the_prompt_composer       | 4     |
| 3    | 🤖 Agent CLIs                    | memory, skills_should_be_procedural_memories, usage_limits_and_api_costs                      | 3     |
| 4    | 🔁 Turn < Session < Clan < Tribe | finalizers, microsoft_ceo_thinks_model_agnostic_architecture_is_good, multi_provider_swarms   | 3     |
| 5    | 🎨 Artifacts                     | artifact_links, audited_artifact_reads, bead_lifecycle, beads_and_birds, repos_vs_directories | 5     |
| 6    | 💾 VCS, 🛑 Not ready             | host_owned_completion, monorepo_tools_are_trending, ai_slop, sase_is_too_complex              | 4     |
| 7    | 🔮 Future Work                   | bob_task_events, evals, files_not_tracked_by_a_vcs, goals, memory_and_docs_are_the_same       | 5     |

> [!decision] wave_split = even Use 6 waves in album walk order (the index order above),
> with no section alignment. Gates become `^wave-1` … `^wave-6`, and the last wave task
> uses the "final" wording below.
>
> | Wave | Records (stem)                                                                                                                          |
> | ---- | --------------------------------------------------------------------------------------------------------------------------------------- |
> | 1    | what_came_before, recursive_sase_improvement, sase_is_many_abstractions, why_a_tui, agent_control_flow                                  |
> | 2    | layers_of_prompt_management, model_alias_pools, the_prompt_composer, memory, skills_should_be_procedural_memories                       |
> | 3    | usage_limits_and_api_costs, finalizers, microsoft_ceo_thinks_model_agnostic_architecture_is_good, multi_provider_swarms, artifact_links |
> | 4    | audited_artifact_reads, bead_lifecycle, beads_and_birds, repos_vs_directories, host_owned_completion                                    |
> | 5    | monorepo_tools_are_trending, ai_slop, sase_is_too_complex, bob_task_events                                                              |
> | 6    | evals, files_not_tracked_by_a_vcs, goals, memory_and_docs_are_the_same                                                                  |
>
> Build the draft with the same per-record lines and the same gate pattern as the
> default draft below. Rerun the dry run: expect 34 items.

## Wave task wording (default `gate_role = review`)

- Waves 1 to N−1:
  `Wave N review: reread the Wave N [[sase_blog_zettel]] drafts together and add any new records to Wave N+1!`
- Last wave:
  `Wave N review: reread the Wave N [[sase_blog_zettel]] drafts together and confirm no record is still a seed!`

> [!decision] gate_role = milestone Every wave task, including the last, reads
> `Finish Wave N of the [[sase_blog_zettel]] drafts!`. The `^wave-N` IDs and every `&`
> prerequisite stay the same.

## The capture draft (default decisions)

Save this exact text to a scratch file **outside** the vault, for example
`"$(mktemp -d)/draft.txt"`. Keep the header line `@sase_blog_drafts` alone on its first
line, with no blank lines between bullets:

```text
@sase_blog_drafts
- Draft or cut [[sase_blog_zettel/what_came_before|What Came Before]]! ^what-came-before
- Draft or cut [[sase_blog_zettel/recursive_sase_improvement|Recursive SASE Improvement]]! ^recursive-sase-improvement
- Draft or cut [[sase_blog_zettel/sase_is_many_abstractions|sase is many abstractions]]! ^sase-is-many-abstractions
- Draft or cut [[sase_blog_zettel/why_a_tui|Why a TUI?]]! ^why-a-tui
- Wave 1 review: reread the Wave 1 [[sase_blog_zettel]] drafts together and add any new records to Wave 2! ^wave-1 &sase_blog_drafts:what-came-before &sase_blog_drafts:recursive-sase-improvement &sase_blog_drafts:sase-is-many-abstractions &sase_blog_drafts:why-a-tui
- Draft or cut [[sase_blog_zettel/agent_control_flow|Agent Control Flow]]! ^agent-control-flow &sase_blog_drafts:wave-1
- Draft or cut [[sase_blog_zettel/layers_of_prompt_management|Layers of Prompt Management]]! ^layers-of-prompt-management &sase_blog_drafts:wave-1
- Draft or cut [[sase_blog_zettel/model_alias_pools|Model Alias Pools]]! ^model-alias-pools &sase_blog_drafts:wave-1
- Draft or cut [[sase_blog_zettel/the_prompt_composer|The Prompt Composer]]! ^the-prompt-composer &sase_blog_drafts:wave-1
- Wave 2 review: reread the Wave 2 [[sase_blog_zettel]] drafts together and add any new records to Wave 3! ^wave-2 &sase_blog_drafts:agent-control-flow &sase_blog_drafts:layers-of-prompt-management &sase_blog_drafts:model-alias-pools &sase_blog_drafts:the-prompt-composer
- Draft or cut [[sase_blog_zettel/memory|Memory]]! ^memory &sase_blog_drafts:wave-2
- Draft or cut [[sase_blog_zettel/skills_should_be_procedural_memories|Skills should be Procedural Memories]]! ^skills-should-be-procedural-memories &sase_blog_drafts:wave-2
- Draft or cut [[sase_blog_zettel/usage_limits_and_api_costs|Usage Limits and API Costs]]! ^usage-limits-and-api-costs &sase_blog_drafts:wave-2
- Wave 3 review: reread the Wave 3 [[sase_blog_zettel]] drafts together and add any new records to Wave 4! ^wave-3 &sase_blog_drafts:memory &sase_blog_drafts:skills-should-be-procedural-memories &sase_blog_drafts:usage-limits-and-api-costs
- Draft or cut [[sase_blog_zettel/finalizers|Finalizers]]! ^finalizers &sase_blog_drafts:wave-3
- Draft or cut [[sase_blog_zettel/microsoft_ceo_thinks_model_agnostic_architecture_is_good|Microsoft CEO thinks model-agnostic architecture is good]]! ^microsoft-ceo-model-agnostic &sase_blog_drafts:wave-3
- Draft or cut [[sase_blog_zettel/multi_provider_swarms|Multi-provider Swarms]]! ^multi-provider-swarms &sase_blog_drafts:wave-3
- Wave 4 review: reread the Wave 4 [[sase_blog_zettel]] drafts together and add any new records to Wave 5! ^wave-4 &sase_blog_drafts:finalizers &sase_blog_drafts:microsoft-ceo-model-agnostic &sase_blog_drafts:multi-provider-swarms
- Draft or cut [[sase_blog_zettel/artifact_links|Artifact Links]]! ^artifact-links &sase_blog_drafts:wave-4
- Draft or cut [[sase_blog_zettel/audited_artifact_reads|Audited Artifact Reads]]! ^audited-artifact-reads &sase_blog_drafts:wave-4
- Draft or cut [[sase_blog_zettel/bead_lifecycle|Bead Lifecycle]]! ^bead-lifecycle &sase_blog_drafts:wave-4
- Draft or cut [[sase_blog_zettel/beads_and_birds|Beads and Birds]]! ^beads-and-birds &sase_blog_drafts:wave-4
- Draft or cut [[sase_blog_zettel/repos_vs_directories|Repos vs Directories]]! ^repos-vs-directories &sase_blog_drafts:wave-4
- Wave 5 review: reread the Wave 5 [[sase_blog_zettel]] drafts together and add any new records to Wave 6! ^wave-5 &sase_blog_drafts:artifact-links &sase_blog_drafts:audited-artifact-reads &sase_blog_drafts:bead-lifecycle &sase_blog_drafts:beads-and-birds &sase_blog_drafts:repos-vs-directories
- Draft or cut [[sase_blog_zettel/host_owned_completion|Host-owned Completion]]! ^host-owned-completion &sase_blog_drafts:wave-5
- Draft or cut [[sase_blog_zettel/monorepo_tools_are_trending|Monorepo Tools are Trending]]! ^monorepo-tools-are-trending &sase_blog_drafts:wave-5
- Draft or cut [[sase_blog_zettel/ai_slop|AI Slop]]! ^ai-slop &sase_blog_drafts:wave-5
- Draft or cut [[sase_blog_zettel/sase_is_too_complex|SASE is Too Complex]]! ^sase-is-too-complex &sase_blog_drafts:wave-5
- Wave 6 review: reread the Wave 6 [[sase_blog_zettel]] drafts together and add any new records to Wave 7! ^wave-6 &sase_blog_drafts:host-owned-completion &sase_blog_drafts:monorepo-tools-are-trending &sase_blog_drafts:ai-slop &sase_blog_drafts:sase-is-too-complex
- Draft or cut [[sase_blog_zettel/bob_task_events|Bob Task Events]]! ^bob-task-events &sase_blog_drafts:wave-6
- Draft or cut [[sase_blog_zettel/evals|Evals]]! ^evals &sase_blog_drafts:wave-6
- Draft or cut [[sase_blog_zettel/files_not_tracked_by_a_vcs|Files Not Tracked by a VCS]]! ^files-not-tracked-by-a-vcs &sase_blog_drafts:wave-6
- Draft or cut [[sase_blog_zettel/goals|Goals]]! ^goals &sase_blog_drafts:wave-6
- Draft or cut [[sase_blog_zettel/memory_and_docs_are_the_same|Memory and Docs are the Same]]! ^memory-and-docs-are-the-same &sase_blog_drafts:wave-6
- Wave 7 review: reread the Wave 7 [[sase_blog_zettel]] drafts together and confirm no record is still a seed! ^wave-7 &sase_blog_drafts:bob-task-events &sase_blog_drafts:evals &sase_blog_drafts:files-not-tracked-by-a-vcs &sase_blog_drafts:goals &sase_blog_drafts:memory-and-docs-are-the-same
```

## Steps

1. **Preflight (read-only).** Run all of these from `~/bob`.
   - Re-list `sase_blog_zettel/*.md` and read each record's `status:` and `section:`.
     The draft assumes the 28 seed records above.
     - A listed record that is gone, or no longer `seed`, already left the seed state:
       drop its line, drop its `&` from its wave task, and report it.
     - A record not in the table: add a line for it to the wave that holds its section,
       as long as that wave stays at 5 or fewer tasks (also add its `&` to that wave's
       task). Otherwise, leave it out and name it in the final report. The wave reviews
       exist to pick up newcomers.
   - Confirm that none of the planned block IDs already exist in `sase_blog_drafts.md`.
     Today it holds only `^prj` and `^create-draft-tasks`. Capture refuses a duplicate
     ID anyway.
2. **Dry run.** Run `bob capture --dry-run --no-clip < "$draft"`. Expect exit 0 and `35`
   `would capture` lines with the default decisions. The 4 Wave 1 tasks must plan as
   `[ ]`. Every other task must plan as `[?]`, with `[dependsOn:: sase_blog_drafts__…]`.
   Fix any refusal in the draft and rerun. Never hand-edit the note to work around a
   refusal.
3. **Capture.** Run `bob capture --no-clip < "$draft"` once. It writes all or nothing.
   If it fails, read the error, fix the draft, dry-run again, then retry.
4. **Status grouping.** Capture puts new tasks in the unheaded intake under `## Tasks`.
   `bob task reconcile` moves Blocked tasks under `### Blocked` and refreshes the badge
   row. Run `bob task reconcile --dry-run` first. In `sase_blog_drafts.md` it should
   plan only grouping and BLOCKS-line work, with no `unresolved_dependency_link`,
   `dependency_cycle`, or `malformed_dependency_line` warnings for these tasks. Then run
   `bob task reconcile`. Changes it plans in other notes are the same routine work
   `bob tick` applies. If the run is deferred by the two-second quiet interval, wait a
   few seconds and rerun.
5. **Verify.** Run all of these from `~/bob`.
   - `bob ready sase_blog_drafts` lists exactly the 4 Wave 1 tasks.
   - In `sase_blog_drafts.md`, check every task:
     - Each Wave 1 task has no Depends-On line and one `🧱 **BLOCKS:** [[#^wave-1]]`.
     - Each later record task has exactly `⛓️ **DEPENDS ON:** [[#^wave-(N-1)]]` and
       `🧱 **BLOCKS:** [[#^wave-N]]`.
     - Each `^wave-N` depends on exactly its own wave's tasks. Each wave task except the
       last has one BLOCKS line listing exactly the next wave's tasks.
   - A short script over the note, or careful `grep`, is enough. Count 28 `Draft or cut`
     tasks and 7 wave tasks.
   - `^prj` and `^create-draft-tasks` are unchanged. The album, records, `sase_blog.md`,
     and today's daily note are untouched: `git -C ~/bob status` should show only
     `sase_blog_drafts.md`, plus any notes the reconcile run legitimately touched.
6. **Planning task.** With the default `close_planning_task = leave_open`, leave
   `^create-draft-tasks` alone.

   > [!decision] close_planning_task = close After step 5, run
   > `printf '%s\n' '!sase_blog_drafts:create-draft-tasks' | bob capture --dry-run --no-clip`.
   > Check that the dry run completes only that task, then run it without `--dry-run`.

7. **Report.** Give the final wave table, with task counts and the block IDs of the wave
   tasks. Name any record you skipped or added in step 1, and anything reconcile changed
   outside `sase_blog_drafts.md`.

## Guardrails

- Write through `bob capture` and `bob task reconcile` only. Never hand-write
  Depends-On, BLOCKS, `[id::]`, or `[dependsOn::]` text. Never edit the `bob-*` plugins.
- Do not commit or push the vault. `bob-vault-sync` handles it. Make no changes in the
  bob-cli repo.
- Do not stamp `[fresh::]`, link tasks into today's Pomodoros, or raise any task to
  Next. Wave 1 stays as plain Ready intake for Bryan's own review walk.
- If the real capture lands but verification shows a wrong edge, fix it with
  `bob capture`:
  - Add a missing prerequisite with `&`, for example
    `bob capture '&sase_blog_drafts:wave-1 @sase_blog_drafts+agent-control-flow'`.
  - Removing a wrong edge would need a Depends-On line edit, so stop and report it
    rather than reverting the note through git.
