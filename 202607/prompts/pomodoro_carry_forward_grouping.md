- **PLAN:** [../202607/pomodoro_carry_forward_grouping.md](../pomodoro_carry_forward_grouping.md)

 When a pomodoro task block link sub-bullet has `#` appended to it, the `<ctrl+enter>` Obsidian keymap just copies those sub-bullets to the new pomodoro note that it creates. Can you help me also start sorting these sub-bullets below that did not have `#` appended to them? Think this through thoroughly and create a plan using your `/sase_plan` skill. Choose and author the appropriate
tier, validate and revalidate until it passes, then submit it with `sase plan propose` (as the skill instructs)
before making any file changes.
 

%xprompts_enabled:false
### Questions and Answers

#### Q1: Sort order

> Ctrl+Enter currently carries both kinds of task-link sub-bullets into the new `- [ ] ()` Pomodoro in plain source-line order (interleaved): unmarked links are copied (leaving a 🍅 history entry behind) and `#`-marked links are moved (source bullet removed). How should the new Pomodoro order them?

- [x] **#-marked last** — Unmarked (worked-on, 🍅) links first, then `#`-marked (deferred/moved) links; source order preserved inside each group.
- [ ] **#-marked first** — `#`-marked (deferred/moved) links on top, then the unmarked worked-on links; source order preserved inside each group.
- [ ] **Other rule** — A different sort (e.g. alphabetical, by target task text/priority, or grouping I will describe).

%xprompts_enabled:true