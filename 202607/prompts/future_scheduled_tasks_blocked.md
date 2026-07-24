- **PLAN:** [../202607/future_scheduled_tasks_blocked.md](../future_scheduled_tasks_blocked.md)

 Can you help me start always marking Obsidian tasks that have the
`[scheduled::<YYYY-mm-dd>]` property, where `<YYYY-mm-dd>` is some future date
(i.e. some date later than or equal to tomorrow) as blocked (i.e. change the
task's status to use the custom `[?]` status)?

- We should add support to the `<ctrl+shift+p>` Obsidian keymap when `scheduled`
  is selected and the user selects a future date.
- We should also have the `bob task-status-hooks` command start marking any
  tasks it finds that should be blocked (i.e. are scheduled for a future date)
  as blocked.
- Make sure that these tasks show up in the ~/bob/blocked.md file (this might not require any changes).

Think this through thoroughly and create a plan using your `/sase_plan` skill. Choose and author the appropriate
tier, validate and revalidate until it passes, then submit it with `sase plan propose` (as the skill instructs)
before making any file changes.
