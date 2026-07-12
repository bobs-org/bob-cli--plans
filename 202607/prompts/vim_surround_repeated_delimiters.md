---
plan: 202607/vim_surround_repeated_delimiters.md
---
 #fork:6u This worked, but I found a bug. Namely, when text is surrounded by multiple of the same character, the `cs` and `ds` keymaps don't work. For example, pressing `ds~` while the following block link is selected does not work: `~~[[sase#^fix-sdd-error]]~~`. Can you help me diagnose the root cause of this issue and fix it? Think this through thoroughly and create a plan using your `/sase_plan` skill. Submit your plan with the
`sase plan propose` command (as the skill instructs) before making any file changes.
 