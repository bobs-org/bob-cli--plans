---
tier: tale
title: Define area notes and the area-or-project task containment rule
goal:
  The glossary web defines Area Note, including the rule that every task lives in
  exactly one area or project note and its machine-owned exceptions, and the term
  resolves from published memory.
size: small
proposed_by: bbugyi200.apollo.5c
status: done
---

- **AGENTS:**
  - [bbugyi200.apollo.5c](https://github.com/bobs-org/bob-cli--agents/blob/main/sessions/bbugyi200.apollo.5c.md)
- **COMMITS:**
  - [ea92b38](https://github.com/bobs-org/bob-cli/commit/ea92b38c73e0bd0a942d785348e75e5c655aa20d)
    — docs(memory): define Area Note glossary strand with area-or-project containment
    rule

# Define area notes and the area-or-project task containment rule

## Outcome and scope

Add one new term, Area Note, to bob-cli's existing `glossary` memory web. It defines the
Bob vault's `type: "[[area]]"` notes and records Bryan's containment rule: every task
belongs in exactly one area note or project note. The definition also names the
machine-owned exceptions that the vault and tooling already rely on, so the glossary
does not contradict its existing Reference Task term or Bob's archive behavior.

This is a `tale` with implementation size `small`. One agent adds one fully specified
strand, regenerates memory indexes, verifies resolution, and files one follow-up memory
bead. No code, docs, vault notes, decision records, existing glossary strands, or other
repositories change. Bryan explicitly requested this memory addition, subject to plan
approval.

## Evidence the definition rests on

Planning checked these sources. The implementer does not need to re-derive them, but
must not contradict them.

- **Bryan's own type note `area.md` in the vault:** "ongoing spheres of responsibility
  with no end state, such as [[job]]. Area notes can contain ongoing task lists
  directly. [[project]]s and sub-areas use their `parent` property to point at the area
  they belong to." The vault's `project.md` type note: projects are "finite outcomes
  that can eventually be marked `done` or `canceled`. Ongoing responsibilities use
  [[area]] instead", with `parent: "[[area-or-project]]"`.
- **Live vault (read-only `bob query`, 2026-10-06):** there are 13 area notes, all at
  the vault root: `body`, `cash`, `dev`, `fun`, `gkeep_inbox`, `gtd`, `gtd_daily`,
  `inbox`, `job`, `love`, `mac_inbox`, `recur`, `vacation`. Top-level areas use
  `parent: "[[area]]"`. Sub-areas name their parent area: `recur`→`cash`, `fun` and
  `vacation`→`love`, `inbox` and `gtd_daily`→`gtd`, `mac_inbox` and
  `gkeep_inbox`→`inbox`. Areas carry no meaningful status. Projects name an area as
  `parent` in 5 cases and another project in 78.
- **Where open `#task` items live:** project notes hold 415 and area notes 126. The
  remainder is 4 `^ref` reference tasks in reference notes, 2 templated `^gtd` tasks in
  daily notes, and the placeholder `^prj` line inside the `project.md` type note. The
  Tasks global filter is `#task`, so Pomodoro checkboxes are not tasks. Closed tasks
  also sit in `done/` archive notes (`type: "[[done]]"`), which `bob task archive`
  creates. Each daily note gets `- [*] #task [[gtd_daily]] … ^gtd` from
  `_templates/daily.md`.
- **bob-cli treats areas as always-open homes for tasks:**
  - `bob capture-targets` and `@route` capture list root-level area notes as kind `area`
    (`docs/capture.md`).
  - The per-note Ready cap counts tasks by residence: `area ∨ status ∉ {done,canceled}`
    (`docs/plan.md` "Ready cap per note", decision `note-ready-cap-counts-the-lane`).
  - `bob task reconcile` groups area/project `## Tasks` sections
    (`docs/task-status-hooks.md`).
  - Project-note capture parents must be an area or a non-terminal project
    (`docs/capture.md`).
- **bob-plugins agrees:** the Ctrl+Shift+M task-move destination picker accepts only
  area notes or open project notes (`collectTaskMoveDestinations` /
  `parseTaskMoveDestinationFrontmatter`). Creating a project note from a task requires
  an area or project source note.

## Implementation

### 1. Add the strand

Use `/sase_memory_write` and record its use before editing memory. Read existing
glossary terms only through `/sase_memory_read`, for example
`sase memory read "glossary:project note" "glossary:reference task" -r "<why>"`.
`sase memory init --check` reported no drift during planning.

Create `sase/memory/glossary/area-note.md`. Follow the frontmatter convention of the
existing strands, such as `project-note.md`: set `keyword: Area Note` and no `aliases`
key. Do not add flat-note `type:` metadata, because this is a strand.

Use this exact body. Rewrapping lines is fine; keep the wording, inline code, and links.

```markdown
A Bob vault Markdown file with frontmatter `type: "[[area]]"` for an ongoing sphere of
responsibility with no end state, such as `cash`, `job`, or `dev`. Unlike a
[[project-note]], it has no `^prj` task and no done/canceled lifecycle, so Bob always
treats it as open. Its `parent` names the area it sits under (`"[[area]]"` for a
top-level area); sub-areas and projects point `parent` at the area (or, for a
sub-project, the project) they belong to. Every `#task` belongs in exactly one area or
project note by residence, the file holding its checkbox line; a [[task-link]] or embed
elsewhere never moves it. Area notes hold the ongoing tasks, usually under `## Tasks`,
that belong to no finite project; the inboxes (`inbox`, `mac_inbox`, `gkeep_inbox`) are
area notes whose tasks await triage. The only exceptions are a reference note's own
[[reference-task]], each daily note's templated `^gtd` task, and closed tasks that
`bob task archive` moved into `done/`; move any other stray task with Ctrl+Shift+M.
Root-level area notes are `@route` capture targets, and every area note gets the
per-note Ready cap and `## Tasks` status grouping.
```

Why it is shaped this way:

- **"belongs in exactly one … by residence."** This states Bryan's rule as a norm. It
  reuses the residence concept from the Ready-cap contract, so Task Links in Pomodoros,
  Depends-On links, and transclusions are never mistaken for misplaced tasks.
- **Exceptions.** These are limited to rows that Bob or the vault templates own. An
  unqualified "every task" would contradict the existing Reference Task strand, the
  daily template, and `bob task archive`.
- **"so Bob always treats it as open."** This is the practical difference from a project
  note that agents need. It explains why areas are always capture, move, Ready-cap, and
  project-parent targets.
- **Links.** These are `[[project-note]]`, `[[task-link]]`, and `[[reference-task]]`.
  Through the glossary's mentions closure they pull in project-task and reference-note,
  so one read gives the whole note-type family. The literal `type: "[[area]]"` and
  `"[[area]]"` examples sit inside inline code, so they must not become memory links.

### 2. Regenerate the published memory indexes

Run `sase memory init`. It updates the glossary roster, `sase/memory/README.md`,
`AGENTS.md`, and the provider instruction shims. Never hand-edit generated output.
Review the diff: the only semantic change should be "Area Note" added to the roster,
sorted before "Keep Streak".

### 3. Verify

- `sase memory init --check` reports no remaining drift.
- `sase memory web show glossary` lists Area Note (slug `area-note`) next to the
  existing twelve terms, with no aliases.
- `sase memory read "glossary:area note" -r "<why>"` prints the full body once. Its
  closure includes Project Note, Project Task, Task Link, Task Dependency Link,
  Reference Task, and Reference Note. There are no unresolved-link warnings and no
  memory link to a nonexistent `area` strand.
- The generated `GLOSSARY TERMS` roster in `AGENTS.md` and `CLAUDE.md` contains "Area
  Note" and no definition text. Every other roster entry is unchanged.

### 4. File the follow-up memory bead

Use `/sase_new_task` to file one `memory` task bead, size `small`. The Reference Note
strand says a reference note "gathers source links, notes or highlights, and any
follow-up work." Under the new containment rule, follow-up tasks belong in an area or
project note, and the reference note keeps only its `^ref` task. The bead proposes
rewording that phrase, and optionally adding an `[[area-note]]` back-link from
`project-note`. It must not edit those strands in this plan: Bryan asked only for the
new term.

## Decisions for the reviewer

Each choice below is already applied in the body above. Override any of them at the
approval gate.

1. **Exceptions are explicit.** The recommended exceptions are a reference note's `^ref`
   task, the daily `^gtd` task, and `done/` archives. The alternative is an absolute
   "every task" rule, which would contradict Bob's own tooling and the existing
   Reference Task term.
2. **Highlights annotation tasks count as strays.** `bob highlights` sync still writes
   annotation-derived tasks into a reference note's `## Tasks` section unless they end
   with an `@name` route (`docs/highlights-ref-sync.md`). Under this definition they are
   misfiled and should be routed or moved. The vault currently has no open ones; only
   three closed ones exist, all in `sase_AGENTS_v2`. This plan does not change that
   tooling. Request a feature bead at the gate if intake should default to an area such
   as `mac_inbox`.
3. **No alias.** "area" alone is too common a word to act as a selector or mention
   alias. Add one at the gate if wanted.
4. **No edits to existing strands.** Reconciling the Reference Note wording goes to the
   step 4 bead rather than into this change.
