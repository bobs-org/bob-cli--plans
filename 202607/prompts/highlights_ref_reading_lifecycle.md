---
plan: 202607/highlights_ref_reading_lifecycle.md
---
 The `bob highlights scan -w` command fails when the `^ref` note has an in-progress (`[/]`) or next (`[*]`) status (see the command output below). These statuses should be allowed. Can you help me fix this? Think this through thoroughly and create a plan using your `/sase_plan` skill. Submit your plan with the
`sase plan propose` command (as the skill instructs) before making any file changes.
   

- In fact, this command should use these statuses to determine what value to give the `status` frontmatter property. Namely:
  - We should use the `ready` ref note status when the `^ref` note task is open (`[ ]`).
  - We should use the `wip` ref note status when the `^ref` note task is in-progress (`[/]`).
  - We should use the `next` ref note status when the `^ref` note task is "next" (`[*]`).
- Make sure the ~/bob/refs.base file has great support for all of these statuses and make sure that we write the appropriate changes to the main Highlights note (the one that triggered the reference note file's creation--we should keep the `status:` field in that note in sync with the Obsidian reference note file's `status` frontmatter property).
- I want you to lead the design on this one. Make sure you design this feature so it is intuitive, reliable, and (last but not least) beautiful!

```
❯ bob highlights scan -w
Scanning 46 PDFs in lib

  error  backcompat lifecycle governance  generated PDF task line on line 1 is malformed; expected a generated task such as '- [ ] #task #ref [[...pdf]] #hide ^ref', '- [x] #task #ref [[...pdf]] #hide ^ref', or '- [-] #task #ref [[...pdf]] #hide ^ref'; legacy generated lines without #ref, with [p::2], or without #hide are still accepted
46 pdfs · 0 created · 0 updated · 45 unchanged · 0 markers · 0 tasks · 1 image · 1 failure · writes: none
bob highlights: scan completed with 1 per-PDF failure(s)
planning failures:
  /Users/bbugyi/bob/lib/chat/backcompat_lifecycle_governance.pdf: generated PDF task line on line 1 is malformed; expected a generated task such as '- [ ] #task #ref [[...pdf]] #hide ^ref', '- [x] #task #ref [[...pdf]] #hide ^ref', or '- [-] #task #ref [[...pdf]] #hide ^ref'; legacy generated lines without #ref, with [p::2], or without #hide are still accepted
```