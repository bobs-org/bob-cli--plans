---
tier: tale
title: Define project and reference notes and their status tasks
goal:
  Add four concise glossary strands with the requested aliases, accurate lifecycle
  meanings, resolving links, and regenerated memory indexes.
size: small
proposed_by: bbugyi200.athena.0vs
create_time: 2026-10-03 14:50:30
status: wip
---

# Define project and reference notes and their status tasks

## Outcome and scope

Add four concise, linked terms to bob-cli's existing `glossary` memory web: Project Note
(`prj note`), Project Task (`prj task`), Reference Note (`ref note`), and Reference Task
(`ref task`). A note is the vault file; its designated task represents the lifecycle of
the whole project or source material. The user's phrase "a `^ref` reference note" means
the reference task inside that note.

This is a `tale` with implementation size `small`: one agent can add four well-specified
strands, regenerate the memory indexes, and verify resolution. Planning is complete
here; implementation needs no code or behavior changes. The user explicitly authorized
these memory additions, subject to plan approval.

## Evidence and definition boundaries

The existing glossary has seven terms and none of these four names or aliases. Its
descriptor already indexes strands and enables implicit term references. Use explicit
same-web links for each note/task pair, preserving the descriptor's existing
configuration and all existing definitions.

Definitions were checked against:

- `docs/projects.md`, "Project Notes", "The `^prj` Task", "Sync Rules", and "Warnings":
  project identity comes from `type: "[[project]]"`; one `^prj` task carries completion
  criteria. Checked/canceled tasks close the project; reopening a terminal project sets
  `wip`, while open nonterminal projects keep their existing status. Terminal notes may
  lack an archived lifecycle task.
- `docs/highlights-ref-sync.md`, "Required Parent, Type, and Ref Type" and "Generated
  Body Contract": reference notes use `type: "[[ref]]"`; generated `^ref` tasks link to
  PDFs and track reading status. Legacy notes may lack the task. The material
  represented can be a book, article, captured conversation, or other external source;
  PDF/Highlights generation is an implementation of the convention, not the definition
  of all reference material.
- `docs/freshness.md`, "Tracking review (projects and references)": task identity uses
  an exact trailing block ID, not a tag, a block link, or a near-match. Legacy task
  lines without the additive `#prj`/`#ref` tag remain recognized. `#hide` controls
  visibility rather than note/task identity.
- `src/native/projects/scan.rs` and `src/native/highlights_ref/tests/tasks.rs`:
  validation of task shapes and rejection of duplicate lifecycle tasks; the latter also
  covers legacy lines without `#ref` and the generated task's PDF link requirement.
- `src/native/highlights_ref/tests/status.rs`: reference terminal statuses and reopening
  to `ready`, distinct from project reopening to `wip`.

Keep detailed scheduling, freshness rules, keymaps, PDF write permissions, and sync
conflict handling in their existing documentation. These glossary additions must not
imply that completing every ordinary task completes the project, that finishing a
reference completes its follow-up tasks, or that a sync operation silently overwrites
conflicting status edits. They introduce terminology only.

## Implementation

### 1. Add the four strands

Use `/sase_memory_write` and record its use before editing memory. Consult memory only
through `/sase_memory_read`; a useful batched check of current strand metadata is
`sase memory read glossary -f json -r "Add approved note/task glossary terms"`. The
current tree has no initialization drift: `sase memory init --check` passed during
planning.

Create these files with the existing glossary strand frontmatter convention, setting
each canonical keyword and alias exactly as listed. These are strands, so do not add
flat-note `type: core` or `type: reference` metadata.

| File under `sase/memory/glossary/` | Keyword          | Aliases    |
| ---------------------------------- | ---------------- | ---------- |
| `project-note.md`                  | `Project Note`   | `prj note` |
| `project-task.md`                  | `Project Task`   | `prj task` |
| `reference-note.md`                | `Reference Note` | `ref note` |
| `reference-task.md`                | `Reference Task` | `ref task` |

Use the following exact definition bodies. Wrapping lines is fine; preserve the meaning,
inline code, and links. These bodies keep each entry under 110 words.

#### Project Note

```markdown
A Bob vault Markdown file with frontmatter `type: "[[project]]"` that holds a project's
context and work. Its [[project-task]] (`^prj`) states the completion criteria and
represents the project's overall status; ordinary tasks describe work toward that
outcome. A closed project remains a project note if its `^prj` task is archived. See
`docs/projects.md`.
```

#### Project Task

```markdown
The single project-level `#task` checkbox in a [[project-note]], identified by the exact
trailing block ID `^prj`. Its description states what must be true for the project to be
complete. `bob projects sync` maps checked to `status: done`, canceled to
`status: canceled`, and reopening a terminal project to `status: wip`; an open
nonterminal project's status is preserved. New lines include `#prj`; legacy lines
without that tag remain recognized. See `docs/projects.md`.
```

#### Reference Note

```markdown
A Bob vault Markdown file with frontmatter `type: "[[ref]]"` for a particular piece of
external reference material, such as a book, article, or captured conversation. It
gathers source links, notes or highlights, and any follow-up work. Its
[[reference-task]] (`^ref`) represents progress through that material. Highlights sync
generates this form; legacy notes may lack the task. See `docs/highlights-ref-sync.md`.
```

#### Reference Task

```markdown
The single source-level `#task` checkbox in a [[reference-note]], identified by the
exact trailing block ID `^ref`, that tracks progress through the external material. In
Highlights-generated notes it links to the PDF: `[ ]` means `ready`, `[*]` means `next`,
`[/]` means `wip`, `[x]`/`[X]` means `read`, and `[-]` means `abandoned`. Highlights
sync reconciles this with note and PDF status. New lines include `#ref`; legacy lines
without that tag remain recognized. Follow-up tasks track separate work. See
`docs/highlights-ref-sync.md`.
```

### 2. Regenerate the published memory indexes

Run `sase memory init` as required by `/sase_memory_write`. Let the generator update the
glossary roster, memory README, `AGENTS.md`, and any applicable provider instruction
shims. Never hand-edit generated output. Review the resulting changes, preserving
unrelated user changes if any appeared since planning. Do not edit vault notes, CLI
code, docs, decision records, existing glossary terms, or other repositories for this
task.

### 3. Verify the definitions and discoverability

- Run `sase memory init --check`; it must report no remaining drift.
- Use `sase memory web show glossary` to confirm all four canonical names and aliases
  appear once, alongside the existing seven terms.
- Read the four canonical selectors together with `sase memory read` and a specific
  reason. Confirm complete, nonduplicated bodies and resolving links in each pair
  (`project-note` ↔ `project-task`, `reference-note` ↔ `reference-task`), including
  termination of their mutual reference cycles.
- Repeat using quoted alias selectors `"glossary:prj note"`, `"glossary:prj task"`,
  `"glossary:ref note"`, and `"glossary:ref task"`; inspect JSON output if necessary to
  verify they resolve to the intended canonical slugs. Ensure the literal frontmatter
  examples in inline code do not produce unresolved memory links to `project` or `ref`.
- Check the generated glossary roster includes all four requested aliases and contains
  no full definition bodies. Confirm existing entries are preserved.
- Review the new text against the evidence above: the file/task distinction, completion
  criteria versus reading progress, exact IDs, optional additive tags, archival/legacy
  exceptions, and different lifecycle status vocabularies must all remain accurate. Each
  body must stay below 110 words.
- Run `git diff --check` and inspect the final change set. No new software tests or full
  Rust test suite are needed for this memory-only change; the actual memory renderer and
  lookup commands provide the relevant verification.

Acceptance is four concise, correctly linked and aliased definitions that can be read on
demand, plus regenerated indexes with no initialization drift.
