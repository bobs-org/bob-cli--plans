---
tier: tale
title: Capture Alias glossary term
goal:
  Define Bob Mac Capture's single-character typing substitutions as a new Capture Alias
  glossary strand, and republish the roster and public glossary page.
size: small
decisions:
  memory_capture_alias_strand:
    ask:
      Should a new glossary:capture-alias strand define Capture Alias and republish the
      roster?
    memory:
      - glossary:capture-alias
      - glossary
    requested:
      Can you help me add an excellent new "capture alias" glossary term that describes
      the various single-character substitutions that the bob-mac-capture app makes?
    default: true
    answer: true
proposed_by: bbugyi200.athena.1ag
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-09 10:38:42
status: wip
---

# Plan: Add the Capture Alias glossary term

## Goal

Add one glossary strand, **Capture Alias** (alias: `sigil alias`). It defines the
single-character substitutions Bob Mac Capture makes while you type, such as `.` → `^`,
`,` → `+`, `\` → `#`, `=` → `~`, and `x` → `!`. Then regenerate everything derived from
the glossary web: the always-loaded roster in `AGENTS.md` / `CLAUDE.md`, the memory
README, and the public glossary page `docs/reference/glossary.md`.

No code changes. Nothing in `bob-mac-capture` changes; that repo is a read-only source.

## Background (verified while planning)

- **bob owns every alias.** Following `decisions:mac-capture-is-a-thin-client`, the app
  forwards a candidate key (`: ^ + | , . ; \ / - = ~ x s`), the post-key draft, and the
  caret to `bob capture-rewrite --cursor N --format json`. bob answers with rule
  `expand_sigil_alias` and summary `Typed <sigil> for <key>`, and the app applies that
  one edit natively. The rule lives in bob-cli at
  `src/native/capture_language/rewrite.rs` (`expand_sigil_alias` plus its helpers:
  `expand_dot_block_id_alias`, `expand_dot_route_alias`,
  `expand_parent_task_comma_alias`, `expand_hash_alias`,
  `expand_past_segment_dash_alias`, `expand_relation_equals_alias`,
  `expand_relation_flip_alias`, `expand_close_outcome_alias`). The contract is
  documented in `docs/capture.md` under `## bob capture-rewrite`, paragraph **Sigil
  alias.**, and in `docs/reference/capture-syntax.md` under "Overloaded markers".
- **The same rewrite lane runs other rules that are not aliases**, because they edit an
  existing marker instead of standing in for a sigil: `switch_block_id_separator`,
  `toggle_relation_direction` (a glued `~`/`=` flip), `attach_relation_label` (`|`), and
  `@@` absorption.
- **Mac-side behavior is documented in the Mac repo**: `docs/keyboard.md` (rows for `,`,
  `.`, `..`, `;`, `-`, `x / s`, `=`, `\ or /`, `\`), `docs/capture-panel.md` (the
  rewrite-lane paragraph: hold/timeout, fast-double-tap escapes), `docs/task-pickers.md`
  and `docs/capture.md` (`action_continuation_aliases`), and
  `docs/relationship-pickers.md`. The trigger key set is in `noteSeparatorToggleIntent`
  in `Sources/BobMacCapture/CapturePanelModel.swift`. The only app-local alias is a
  blank draft's `\` → `/`, which opens Go To.
- **Name collision to defuse.** The Mac app already uses "Capture alias" for the global
  `Command-\` hotkey: `CaptureAliasHotkeyRegistration`, Settings › Hotkey › "Also open
  Capture with Command-\", and the `Capture alias hotkey conflict` diagnostic. "Alias"
  also means frontmatter/`project_name_aliases` route aliases, wikilink `|alias`,
  deprecated tag spellings (`#offclock`), and the bare owner alias `^route:id`. The
  strand must say it is none of these.
- **The glossary web uses `closure: mentions`.** `sase memory read glossary:<term>`
  prints in full every other term whose keyword or alias appears in the body's prose.
  `[[...]]` links only become numbered Linked References. Mentioning "capture" already
  pulls in Capture and its closure (Project Note, Task Type, Pomodoro, Task Link, Task
  Relation, Task Freshness, Task Successor), so mentions of those cost nothing extra.
  Avoid incidental prose matches for unrelated terms, for example "drawer", "keeps",
  "mention", "decay", "Work Log", "batch"/"multi-item capture", "project-note capture",
  and "command sheet". Use a `[[link]]` when you need to name one of them.
- **Pre-existing docs drift.** `uv run --project tools/docs bob-docs gen --check`
  already reports `out of date: docs/reference/glossary.md`, because the Project Note
  Capture strand (commit `186c5c1d`) was added without regenerating it. The regeneration
  in step 4 fixes that too; that is expected and in scope.

## Steps

### 1. Re-verify the catalog against current sources

Before writing, use your `/sase_memory_write` skill. This plan's accepted
`memory_capture_alias_strand` decision authorizes this one new strand plus the
`sase memory init` regeneration, and nothing else.

Check every claim in the draft below against the bob-cli sources named above. Open
`bob-mac-capture` with your `/sase_repo` skill (never by path guessing) and check the
Mac docs and the trigger set. If a source disagrees with the draft, the source wins: fix
the wording, keep the structure. Do not edit either repo's docs or code.

### 2. Create `sase/memory/glossary/capture-alias.md`

Match the existing strands (`sase/memory/glossary/project-note-capture.md`,
`task-mention.md`): YAML frontmatter with `keyword:` and `aliases:`, then Markdown prose
wrapped near 88 columns. Write no bead IDs: `bob-docs gen` refuses generated pages that
name one. Keep `Command-\` inside a code span, because a bare `\"` in Markdown swallows
the backslash. Start from this draft (the content is verified; tighten wording if
useful):

```markdown
---
keyword: Capture Alias
aliases:
  - "sigil alias"
---

A keystroke substitution in Bob Mac Capture's editor: an easy-to-reach key that the app
swaps for the [[capture]] sigil it stands for, only where that sigil would act at the
caret by opening its picker or filling a close slot. bob decides every one
([[decisions:mac-capture-is-a-thin-client]]). The app sends the post-key draft and caret
to `bob capture-rewrite --cursor`, which answers with rule `expand_sigil_alias` (summary
`Typed ^ for .`), and the app applies that one edit natively, so Undo restores the typed
key. The aliases:

- `.` → `^` right after `@route` (`@errands.` → `@errands^`); `;` → `:` there, and as a
  whole item (Task Link picker). In a [[project-note-capture]], a first-level `- .` or
  `- ;` opens the Move picker and `- ^.` gives `- ^^`.
- `.` → `@` after body text and a space or tab, wherever a bare `@` opens route
  completion (`Buy milk .` → `Buy milk @`).
- `,` → `+` wherever the Parent Task picker would open (`@cash,` → `@cash+`); glued to a
  complete `@route+id` target, it inserts ` +` for the next pick.
- `\` or `/` → `#` wherever a `#` menu would open (task type, section, Pomodoro name,
  wikilink heading, and the rest). Never at a line start, glued to a word, or after
  another `\`/`/`; `/` stays literal where it is already field text (`=/`, `[[folder/`,
  `@@` routes).
- `-` → `~` inside an `x=` past token (`x=2-` → `x=2~`); a spaced `-` stays literal.
- `=` → `~` after a space or tab where the Links To relation picker would open, while
  `~=` / `~~=` flip to `~~` / `~` ([[decisions:relation-equals-alias-and-flips]]).
- `x` / `s` → `!` / `*` (complete / park) when the lowercase letter ends a close
  selection: `xx` → `x!`, `+2 xs` → `+2 x*`, `2xx` → `2x!`, `@r:id=xx` → `@r:id=x!`.

Open pickers honour bob's `action_continuation_aliases`: `,` acts as `+` in the lone-`+`
Parent Task picker, and `.` acts as `^` in the `- ^` Move picker. In the relation picker
and label card, `=` flips direction like `~`. The only app-local alias is a blank
draft's `\` → `/`, which opens Go To. Each alias can be escaped: by Undo; by a fast
second `=` or spaced `.` (350 ms), which types one literal (`===` gives `==`); by a
second `\`/`/` at any speed; or by Space or Escape on the still-empty picker the alias
opened. Aliases that change meaning (`x`/`s`, and `-` in `x=`) hold later edits and
Return until bob answers. On a timeout the key stays literal and Return is dropped ("Bob
did not answer; press Return again"). An older bob leaves every alias literal.

Aliases exist only while typing. `bob capture`, `capture-parse`, paste, and piped stdin
keep the text, so a pasted `Buy milk .` captures a dot and `xx` closes nothing. Template
drafts and `#"…"` exact-name literals are never rewritten. Rewrites in the same lane
that edit an existing marker are not aliases: the block-ID separator toggle
(`@route:id^` → `@route^id`), the glued `~`/`=` relation direction flip, the `|` label
attach, and `@@` absorption. None of these is a capture alias either: the Mac's "Capture
alias" hotkey (Settings › Hotkey's global `Command-\`), a frontmatter or
`project_name_aliases` route alias, a wikilink `|alias`, a deprecated tag spelling
(`#offclock`), the bare owner alias `^route:id`, Tab's `--` → `—` snippet, or the
close-list comma (`x1` then `2` → `x1,2`). See bob-cli `docs/capture.md`
(`bob capture-rewrite`, "Sigil alias") and Bob Mac Capture's `docs/keyboard.md` and
`docs/capture-panel.md`.
```

### 3. Republish memory

Run `sase memory init`. It regenerates the glossary roster in `sase/memory/glossary.md`
(between the `sase:strands` markers), `AGENTS.md`, `CLAUDE.md`, and
`sase/memory/README.md`. Never hand-edit those files. Then verify:

- `sase memory web show glossary` lists **Capture Alias**, and `AGENTS.md`'s roster
  reads `Capture Alias (sigil alias)`, placed alphabetically between Capture and Command
  Mode.
- `sase memory show glossary:capture-alias` renders the body, and its Linked References
  resolve `glossary:capture`, `glossary:project-note-capture`, and both decisions with
  no unresolved-link warning. Look at its `#` headings to see which terms the closure
  inlined. Anything beyond the Capture closure listed in Background means an incidental
  mention: reword it or turn it into a `[[link]]`.
- `git diff --stat` touches only the new strand plus the generated files listed above.

### 4. Regenerate the public glossary page

Run `uv run --project tools/docs bob-docs gen` to rewrite `docs/reference/glossary.md`.
It adds Capture Alias, and the missing Project Note Capture term from the drift noted
above. Then run `uv run --project tools/docs bob-docs gen --check`, which must report
everything up to date, and `just docs-check`, which is the full documentation gate. If a
docs lint rejects strand wording, fix the strand (not the generated page), then rerun
`sase memory init` and `bob-docs gen`.

## Acceptance

- `sase/memory/glossary/capture-alias.md` exists, matches current bob-cli and Mac
  sources, and names no bead IDs.
- The generated roster, memory README, `AGENTS.md`, `CLAUDE.md`, and
  `docs/reference/glossary.md` are regenerated, not hand-edited.
- `bob-docs gen --check` and `just docs-check` pass.
- No other memory note, bob-cli source, or `bob-mac-capture` file changes.
