---
plan: 202607/capture_clipboard_subbullets.md
---
 Can you help me improve the `bob capture` by adding support for capturing my clipboard contents, which should be added to the note as a sub-bullet?

- This behavior should be triggered by placing a word using the following syntax at the end of the line: `%[<bullet_header>]` where `<bullet_header>` is optional.
- `<bullet_header>` should default to `clip` and should be preficed to the task sub-bullet text by capitalizing the text, appending `: ` to the text, bolding the text, and replacing any `_` in the text with a ` `. For example, the default `clip` should become `**CLIP:** `. As another example, if `%foo_bar_baz` were found at the end of the capture input text, we would prepend `**FOO BAR BAZ:** ` to the sub-bullet text (before the clip reference).
- The "clip reference" is the reference to the contents that were found in the clipboard. This should depend on what those contents look like:
  - A file path should be converted into a valid obsidian link to that file path.
  - A file path that points to an existing image on the system should be treated as a special case of the former and we should render an appropriately sized image for this clip reference.
  - Any file path (i.e. either of the previous two cases) should be copied into either ~/bob/img/ (if an image file) or ~/bob/file/ (otherwise) and referenced using that file path.
  - If the clipboard contents are over 10 lines long or contain any blank lines, we should write those lines to a new file in the ~/bob/file/ directory (use a good naming system for this) and link to that file.
  - Any other contents are pasted as is (I _think_ this is right, but try to be smart about this and catch any edge cases I missed).
- I want you to lead the design on this one. Make sure you design this feature so it is intuitive, reliable, and (last but not least) beautiful!

Think this through thoroughly and create a plan using your `/sase_plan` skill. Choose and author the appropriate
tier, validate and revalidate until it passes, then submit it with `sase plan propose` (as the skill instructs)
before making any file changes.
 