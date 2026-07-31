- **PLAN:** [../202607/capture_sub_bullets.md](../capture_sub_bullets.md)

 Can you help me add support for a new `@<file>^<id>` syntax to the
`bob capture` command's task text argument (we already support the `@<file>`
syntax--with support for the `#` suffix)?

- When this syntax is found, it indicates that we should not use the provided
  text to create a new Obsidian task, but should instead use it to append a new
  sub-bullet to the existing (fail with a good error if the task doesn't exist)
  Obsidian task in the `~/bob/<file>.md` file that has a block ID of `<id>`.
- We should also add support for just providing `@<file>^` as text to the input
  prompt that is triggered by the Hammerspoon `<ctrl+shift+alt+i>` keymap
  (configured in my chezmoi repo).
- If only `@<file>^` is provided (i.e. with no `<id>` suffix) to the
  `<ctrl+shift+alt+i>` keymap's input prompt, the user should then be prompted
  to select one of the open tasks that exists in that file currently. Make sure
  we support all obsidian task types except for canceled and done and make sure
  we clearly visually indicate each task status in this menu. The selected
  task's block ID should be used to infer `<id>` (when constructing the
  `bob capture` command).
- I want you to lead the design on this one. Make sure you design this feature so it is intuitive, reliable, and (last but not least) beautiful!

Think this through thoroughly and create a plan using your `/sase_plan` skill. Choose and author the appropriate
tier, validate and revalidate until it passes, then submit it with `sase plan propose` (as the skill instructs)
before making any file changes.
 