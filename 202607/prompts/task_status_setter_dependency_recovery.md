---
plan: 202607/task_status_setter_dependency_recovery.md
---
 #fork:ad Can you now help me make a few related improvements?

- Make sure that the `bob mark-next-tasks` command sets a task's status to open (or in-progress or next, depending on the command's other rules) if it finds that the task has no dependencies or that its dependencies are all closed.
- Make sure that the `<ctrl+enter>` keymap re-opens any tasks that are dependent on the task it closes (if necessary).
- Also, can we rename the `bob mark-next-tasks` command to `bob task-status-setter`, which probably better reflects its functionality at this point?

Think this through thoroughly and create a plan using your `/sase_plan` skill. Choose and author the appropriate
tier, validate and revalidate until it passes, then submit it with `sase plan propose` (as the skill instructs)
before making any file changes.
