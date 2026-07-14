---
plan: 202607/pomodoro_move_only_hash_keymap.md
---
 We recently added support to the `<ctrl+enter>` keymap for treating pomodoro sub-bullet block links that end with a trailing `#` special (we just delete the `#` and copy those bullets to the next pomodoro, without leaving a tomatoe preficed block link--which would indicate I worked on that task in that pomodoro). Can you now help me add a new `#` Obsidian keymap that automatically appends a `#` to the current line if it is a pomodoro sub-bullet block link to an Obsidian task? This keymap should accept an optional count. For example, assume the following pomodoro and assume the cursor is on the first sub-bullet (the `^gtd` line):

```
- [ ] (**0620-0645** [t:: 25m]) 
	- [[#^gtd]]
	- [[sase#^no-sdd]]
    - foo bar baz
```

Now assume that the user then presses `1#`. The result should be the following:

```
- [ ] (**0620-0645** [t:: 25m]) 
	- [[#^gtd]]#
	- [[sase#^no-sdd]]#
    - foo bar baz
```

Think this through thoroughly and create a plan using your `/sase_plan` skill. Submit your plan with the
`sase plan propose` command (as the skill instructs) before making any file changes.
 