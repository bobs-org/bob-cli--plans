---
plan: 202607/mark_next_hidden_transclusions.md
---
 We recently made the `!` Obsidian keymap not add tasks corresponding with transcluded block links in sub-bullets as dependencies of the the current task if the linked to task has the `#hide` tag. This is correct, but the `bob mark-next-tasks` command should still mark those transcluded tasks with the "next" status when the parent task should also have the "next" status (ex: when that task is linked to from an open pomodoro in today's daily file). For example, the `- [ ] #task #ref [[lib/chat/sase_beads_full_potential_consolidated.pdf]] #hide ^ref` task in the ~/bob/ref/chat/sase_beads_full_potential_consolidated.md file should be marked as next (i.e. `[ ]` should be changed to `[*]`) since it corresponds with a trancluded block link in a `[*]` task that lives in the ~/bob/sase.md file. Can you help me fix this?

Think this through thoroughly and create a plan using your `/sase_plan` skill. Submit your plan with the
`sase plan propose` command (as the skill instructs) before making any file changes.
 