---
tier: epic
title: Ref tasks live with the work they serve
goal: "Every open reference has exactly one ordinary reading task, `#task #ref` with a
  unique `^ref-<slug>` block ID, in the `## Tasks` section of a real area, project, or
  inbox note, and that note is the reference's parent. Capture paths ask for, or
  receive, the parent. A done/-aware locator keeps ref-note status in sync wherever the
  task moves or is archived. The `#hide` and residence special cases are gone, and the
  29 open refs are migrated without losing a field, link, or dependency.

  "
decisions:
  cancel_dropped_wrapper_refs:
    ask:
      During the live migration, cancel the 4 ref tasks whose hand-written wrapper tasks
      you already cancelled?
    default: false
    why:
      Migration stays structural; you triage with Alt+N or cancel once they sit in their
      notes
    answer: false
  memory_ref_parent_decision:
    ask: Add a decisions strand recording that ref tasks live with their parent note?
    memory:
      - decisions:ref-tasks-live-with-their-parent
    default: false
    answer: false
  memory_glossary_ref_terms:
    ask:
      Update the reference-task, reference-note, and area-note glossary strands for the
      new model?
    memory:
      - glossary:reference-task
      - glossary:reference-note
      - glossary:area-note
    default: false
    answer: false
phases:
  - id: sase-hook-env
    title: "sase: file hooks export SASE_FILE_HOOK_PROJECT"
    depends_on: []
    size: small
    description:
      "sase-hook-env: the sase file-hook runner exports the producing project's name to
      every hook command as an environment variable, with docs and dispatch tests."
  - id: parent-resolver
    title: One strict parent resolver and project_name_aliases
    depends_on: []
    size: medium
    description:
      "parent-resolver: add the shared area/project/inbox resolver with
      project_name_aliases, expose aliases in capture-targets, and resolve an explicit
      bob ref create -P."
  - id: freshness-rekey
    title: "Freshness keys refs on the #ref tag, with the lane split"
    depends_on: []
    size: medium
    description:
      "freshness-rekey: re-key ref review identity from the ^ref block ID to the #ref
      tag in Rust and bob-ledger-tools; Ready refs keep REFERENCES while Next/Pending
      refs walk their lanes."
  - id: hook-config
    title: Live alias, install, and the hook passes -P
    depends_on:
      - sase-hook-env
      - parent-resolver
    size: small
    description:
      "hook-config: install bob, add project_name_aliases to bob.md, confirm the live
      sase exports the variable, then switch the chezmoi hook command to pass -P."
  - id: ref-locator
    title: The done/-aware ref-task locator and read-side contracts
    depends_on:
      - parent-resolver
    size: large
    description:
      "ref-locator: build the read-only locator, derive status/parent/dates from the
      located task, add the task object, diagnostics, ref list -P aliases, and
      capture-complete task_kind."
  - id: ref-glyph
    title: The open-book identity glyph and picker text
    depends_on:
      - freshness-rekey
    size: medium
    description:
      "ref-glyph: specify and implement the #task #ref open-book glyph with conformance
      vectors, and show reading tasks cleanly in plugin pickers."
  - id: ref-sync-v2
    title: Scan writes reading tasks into parent notes
    depends_on:
      - ref-locator
    size: large
    description:
      "ref-sync-v2: births insert the v2 line into the parent's Tasks section, ref notes
      carry the managed embed, status syncs to the located line across files, and
      annotation follow-ups go to the parent."
  - id: mac-refs-v2
    title: Bob Mac Capture reads located ref tasks
    depends_on:
      - ref-locator
    size: medium
    description:
      "mac-refs-v2: decode the task object, join Today on path and block ID, refresh on
      root-note changes, and show the book symbol for ref tasks in pickers and the
      inspector."
  - id: ref-create-parent
    title: bob ref create requires -P; ingest, jobs, and fallbacks carry the parent
    depends_on:
      - hook-config
      - ref-sync-v2
    size: medium
    description:
      "ref-create-parent: make -P required with no default, thread the resolved parent
      through typed ingest, ref jobs, and clip-failure fallbacks, and delete the
      obsidian_ref defaults."
  - id: capture-gkeep-parent
    title: Capture URL @route and gkeep pull choose the parent
    depends_on:
      - ref-create-parent
    size: large
    description:
      "capture-gkeep-parent: a bare URL plus one @route becomes a ref filed there,
      previews and parse report the parent, and gkeep pull honors note routes, -P, and a
      TTY prompt."
  - id: migrate-tasks
    title: bob ref migrate-tasks
    depends_on:
      - ref-sync-v2
    size: large
    description:
      "migrate-tasks: add the dry-run-first, reversible command that moves open v1 ref
      tasks into parent notes and rewrites every link and dependency ID that pointed at
      them."
  - id: mac-file-under
    title: Bob Mac Capture asks where a captured link belongs
    depends_on:
      - capture-gkeep-parent
      - mac-refs-v2
    size: medium
    description:
      "mac-file-under: open a File under picker for a bare URL, insert the chosen
      @route, preview the destination, and match project_name_aliases."
  - id: live-migration
    title: Migrate the live vault
    depends_on:
      - hook-config
      - freshness-rekey
      - ref-glyph
      - mac-refs-v2
      - capture-gkeep-parent
      - migrate-tasks
    size: medium
    description:
      "live-migration: confirm the Mac runs the new bob, run migrate-tasks against the
      live vault with the confirmed parent map, reconcile, and verify every invariant."
  - id: closeout
    title: Retire the transitional bypass, docs coherence, memory, final report
    depends_on:
      - mac-file-under
      - live-migration
    size: medium
    description:
      "closeout: remove the transitional hidden ^ref review bypass, read the docs end to
      end, apply the accepted memory decisions, and leave Bryan the verification
      checklist."
proposed_by: bbugyi200.athena.0z0
decided_by: auto
create_time: 2026-10-09 12:29:33
status: wip
---

- **PROMPT:**
  [prompts/202610/ref_tasks_live_with_parent.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202610/ref_tasks_live_with_parent.md)

# Ref tasks live with the work they serve

## Why

Bryan already treats reading as ordinary work. He links reading tasks from Pomodoros
(`🍅 [[databricks_omnigent_job_fit#^ref]]`), uses them as Depends-On prerequisites
(`sase_blog_0.md`), and writes wrapper tasks for them by hand in project notes
(`- [-] #task Read [[ref/chat/agent_history_in_agents_tab]]!`). The wrappers drift: for
4 of the open refs the wrapper is closed while the ref task is still open.

The implementation fights this. Each ref task lives inside its ref note under `ref/`,
carries `#hide`, and is identified by the exact block ID `^ref`. Those three facts hide
it from lanes, caps, Today, the `^` picker, status grouping, and project lifecycle, and
make `^ref` collide the moment two reading tasks share a note.

The consolidated research report
`research:202610/ref_tasks_move_into_parent_notes/ref_tasks_move_into_parent_notes.md`
(2026-10-09) recommends moving every open ref task into a real area or project note and
deriving the parent from where the task lives. Bryan agreed with every requirement it
recommends (J1–J11). This epic builds that design.
[Design calls this plan adds](#design-calls-this-plan-adds) lists where it refines the
research.

Live numbers measured on 2026-10-09 while writing this plan:

- **Open modern ref tasks:** 29 (`ready` 12, `next` 9, `wip` 8; one Next row is Blocked
  `[?]`). All 29 carry `#hide` and `parent: "[[obsidian_ref]]"`. Two arrived after the
  research census: `ref_tasks_move_into_parent_notes` and
  `unblocked_successor_links_on_close`.
- **Links into open trackers:** daily notes 2026-10-06 to 10-08 and `sase_blog_0.md` (a
  Depends-On line plus path-derived `[dependsOn::]`/`[id::]` values such as
  `ref__chat__sase_launch_post_introduction__ref`). Project notes also embed ref
  trackers as child bullets (`- ![[ref/chat/x#^ref]]`), some by bare stem
  (`![[xprompt_role_binding#^ref]]`).
- **Vault:** 6,211 Markdown files; a `#ref` substring sweep takes about 0.1 s on athena.
- **SASE projects:** `bob-cli` (must map to `bob.md`), `sase` (maps by stem), `actstat`
  (no note today).
- **Scan runs on the Mac.** athena's `bob ref create` (and the SASE research hook)
  writes intake PDFs into `xlib/`; the Mac's scheduled `bob highlights scan` pulls them,
  moves them into `lib/`, and writes the ref notes. The Mac's `bob` is therefore the one
  that writes ref tasks.

## The experience (north star)

1. **Capture.** Bryan pastes `https://example.com/essay` into Bob Mac Capture. The link
   underlines, and a **File under** list opens on its own, his last-used parent first.
   He presses ↵ on `sase`; the draft now reads `https://example.com/essay @sase` and the
   preview says `📖 Queue · example.com/essay → sase`. Esc instead keeps the default:
   `→ mac_inbox · file it later`.
2. **Birth.** On the next scan a line appears in `sase.md`'s `## Tasks`:

   ```markdown
   - [ ] #task #ref [[ref/blogs/essay|The Essay Title]] [created::2026-10-12] ^ref-essay
   ```

   In Obsidian, `#task #ref` renders as one teal **open-book** glyph, so the line reads
   `☐ 📖 The Essay Title`. The ref note shows the same live task through an embed where
   the old tracker sat.

3. **Work.** The task behaves like any task: Alt+N commits it to Next, a Pomodoro Task
   Link (`[[sase#^ref-essay]]`) puts it in Today, `^` in Bob Mac Capture lists it with a
   book symbol, Ctrl+Shift+M re-files it (and the parent follows), dependencies point at
   it, and the per-note Ready cap counts it.
4. **Review.** A Ready ref keeps the weekly REFERENCES walk Bryan configured; once he
   commits to it (Next or Pending) it gets the same daily lane review as any lane task.
5. **Finish.** Checking it `[x]` marks the reference read: `bob ref list` and the Refs
   panel see it at once, and the ref note's `status` follows on the next scan. Nightly
   `bob task archive` later moves the closed line into `done/sase_done.md`; the locator
   still finds it there, so the reference stays read and its parent stays `sase`.
6. **Research reports.** A SASE research report from the `bob-cli` project becomes a
   reading task in `bob.md` (`project_name_aliases: ["bob-cli"]`); one from `sase` lands
   in `sase.md`.

## Decisions already made (research J1–J11, accepted by Bryan)

| #   | Rule                                                                                                                                                                                                                                                        |
| --- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| J1  | Keep the existing `-P\|--parent` on `bob ref create`, make it required, drop the `obsidian_ref` default (`-p` stays `--published`).                                                                                                                         |
| J2  | The task's residence defines the parent. Frontmatter `parent` is a projection of where the ref task lives; for an archived task it is the `done/` file's own `parent:`.                                                                                     |
| J3  | Prompt where a human is present (Mac picker, `gkeep pull` on a TTY). Unattended paths default to the inbox area note the equivalent task would use (`mac_inbox`, `gkeep_inbox`). `bob ref create` itself has no default.                                    |
| J4  | Remove ref special-casing except: Ready refs keep the weekly REFERENCES walk on `reference_interval` (never NEW, never decide), and the marker ↔ frontmatter ↔ checkbox status policy stays. Next/Pending refs become ordinary lane rows with daily review. |
| J5  | `^ref-<slug>` is only an address. Identity is the `#ref` tag plus a path-qualified link to the ref note.                                                                                                                                                    |
| J6  | Ref tasks live in `## Tasks` and archive normally into `done/`.                                                                                                                                                                                             |
| J7  | Migrate only open modern refs (29 today). Closed trackers (~320) and zorg-era legacy notes (602) stay frozen.                                                                                                                                               |
| J8  | New Highlights annotation follow-up tasks default to the ref's parent note.                                                                                                                                                                                 |
| J9  | `project_name_aliases` is honored on area and project notes and by every parent lookup.                                                                                                                                                                     |
| J10 | sase exports the project as `SASE_FILE_HOOK_PROJECT`; no `{project}` placeholder.                                                                                                                                                                           |
| J11 | `URL @route` becomes a ref filed under that route instead of a plain URL task.                                                                                                                                                                              |

The research's own open-question recommendations are taken as accepted: the J4 split,
archive normally, inbox defaults, `URL @route`, and the open book in the teal identity
ink.

## Design calls this plan adds

These refine the research where implementation detail forced a choice. Each phase
implements them as written.

1. **`parent` leaves marker sync for v2 notes.** For a ref note whose task lives outside
   it, `parent` is excluded from the marker/frontmatter projection, hash, and base, and
   frontmatter `parent` is rendered from residence like the command-managed `type`.
   Re-filing a ref (Ctrl+Shift+M, inbox triage) therefore never needs a PDF write and
   never fails a scheduled scan. The PDF marker's `parent` is the birth hint; a
   `--write-pdfs` run refreshes a stale one. The migration writes no PDFs.
2. **No birth journal.** Scan writes the parent note first, then the ref note. On a
   rerun the locator finds an orphaned line by its link target (even when the ref note
   does not exist yet) and adopts it instead of inserting a second task. The same
   adoption makes `migrate-tasks` re-runnable after a crash.
3. **v1 notes stay byte-for-byte.** Notes that still hold an in-note `^ref` tracker, and
   legacy notes without one, keep today's behavior exactly, so ~900 frozen notes see no
   churn.
4. **Born-closed refs also get a task.** A ref created `-s read` is born as a closed
   line in its parent and archives normally, so every modern ref has one task.
5. **`-r/--route` is the flag spelling of `@route`** for a bare URL, and an unresolvable
   `URL @route` is an error with near-miss hints. It never silently creates a new note
   holding a URL task.
6. **`capture-parse` reports `ref_parent` as its own additive object,** not inside
   `needs`, so a bare URL stays submittable while the client knows to offer a parent.
7. **`gkeep pull` asks once per URL with a sticky default** (each answer becomes the
   next default), instead of a batch table with per-item override syntax.
8. **Only `SASE_FILE_HOOK_PROJECT`** is exported; no sibling variables until needed.
9. **No new `finished` frontmatter projection and no `bob projects doctor`.**
   `bob ref list` derives finished dates from the located task; alias problems surface
   in `bob ref doctor` and `bob capture-targets -v`.
10. **The transitional `#hide` review bypass** stays only for exact `^ref` rows until
    the live migration proves none remain open; `closeout` deletes it.

## Mental model and invariants

> **A reference is a note in `ref/`. Reading it is one ordinary task, and that task
> lives with the work it serves. Wherever the task lives is the reference's parent.**

| #   | Invariant                                                                                                                                                                                                 |
| --- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| R1  | An open ref task lives in a root area, project, or inbox note, normally under `## Tasks`. A closed one may live in `done/`.                                                                               |
| R2  | A ref note has at most one live ref task. Several open candidates is a diagnostic, never a guess.                                                                                                         |
| R3  | A new or migrated ref note holds no open task lines; it shows its task through the managed embed.                                                                                                         |
| R4  | Frontmatter `parent` equals the task's residence (a `done/X_done.md` residence maps to that file's `parent:`).                                                                                            |
| R5  | The marker ↔ frontmatter ↔ checkbox status policy keeps today's rules, including the `[?]` overlay, `pending` status sync, conflicts, and explicit `--write-pdfs`. Only where the checkbox lives changes. |
| R6  | Ref tasks carry no `#hide`. They count in lanes, caps, Today, pickers, project lifecycle, and archiving like any task.                                                                                    |
| R7  | The one review exception: due Ready `#ref` tasks walk REFERENCES on `reference_interval`, never NEW, and never decide.                                                                                    |
| R8  | bob-cli is the only authority for grammar and parent resolution; Bob Mac Capture renders what `bob` returns.                                                                                              |

## Design specification (decided; implement exactly)

### 1. The ref task line (v2)

```markdown
- [*] #task #ref
  [[ref/papers/harness_engineering|Harness Engineering: Anatomy, Architecture, and Evolution of Agent Harnesses]]
  [created::2026-10-09] ^ref-harness-engineering
```

- **Tags.** `#task #ref` adjacent, in that order, at the start of the body. Both stay in
  stored text so Dataview, Tasks, `bob query`, and grep keep working. No verb: the glyph
  reads as "read".
- **Link.** The first wikilink targets the **ref note** (not the PDF), always
  vault-relative and path-qualified (`ref/<type>/<stem>`), because stems collide
  (`ref/blogs/harness_engineering` and `ref/papers/harness_engineering` both exist). The
  alias is the title (frontmatter `title`, else H1, else humanized stem) with `[`, `]`,
  `|`, `#`, `^`, and backticks removed, whitespace collapsed, and cut at a word boundary
  to at most 100 characters plus `…`.
- **Created.** `[created::YYYY-MM-DD]` in capture's exact format (`format_task_line`),
  from the birth invocation's local date (`BOB_NOW` honored); the migration uses the ref
  note's `created` date part when the v1 line has none. Creation never writes
  `[fresh::]`.
- **Block ID.** `ref-` + slug of the ref-note stem: lowercase ASCII; every run of
  characters outside `[a-z0-9]` becomes one `-`; trim `-`; cut at a `-` boundary so the
  whole ID is at most 44 characters; an empty slug becomes `ref-reading`. It must be
  unique against the destination note, the destination's `done_tasks` archive file, and
  IDs reserved earlier in the same run; collisions append `-2`, `-3`, …. Bryan may
  rename it freely: identity never depends on it.
- **Closed at birth.** A born-closed line carries `[completion:: YYYY-MM-DD]` or
  `[cancelled:: YYYY-MM-DD]` immediately before the block ID, exactly as the v1 stamp
  does today.

### 2. The ref note after the move

```markdown
---
status: next
parent: "[[sase]]"
type: "[[ref]]"
…
---

# Harness Engineering: Anatomy, Architecture, and Evolution of Agent Harnesses

![[sase#^ref-harness-engineering]]

![[lib/papers/harness_engineering.mp3]]

## Highlights

…
```

- **Managed embed.** One line, `![[<residence>#^<block-id>]]`, one blank line below the
  H1, exactly where the tracker sat. `<residence>` is the route for a root note (`sase`)
  and the vault-relative path without `.md` otherwise (`done/sase_done`). Ctrl+Shift+M
  and `bob task archive` already rewrite `[[note#^id]]` links and embeds vault-wide, so
  it follows moves with no new code. Scan heals it (re-inserts a missing embed,
  re-points a stale one) whenever it locates the task; it never creates a task. It is a
  view, never identity, and since it is not under a Pomodoro or a task it is neither a
  Task Link nor a dependency edge.
- **v2 note.** A ref note is v2 when the locator finds its task outside the note (live
  or archived) or its body holds the managed embed. v2 rules (residence parent, embed,
  cross-file sync) apply only to v2 notes; see design call 3.
- `region.rs` treats the managed embed as anatomy (like the old tracker line), not
  content, and the audio embed anchors after it.

### 3. Parent resolution and `project_name_aliases`

`resolve_parent(bob_dir, input)` is the single resolver for `bob ref create -P`, ingest,
ref jobs, capture `URL @route`/`@@route`/`-r`, `gkeep pull`, `bob ref list -P`, births,
and `migrate-tasks`.

- **Candidates:** the capture-target set: root area notes, non-terminal root project
  notes, and the inboxes (`inbox`, `mac_inbox`, `gkeep_inbox` are area notes). Topic
  hubs such as `obsidian_ref` are not candidates.
- **Input forms:** `sase`, `sase.md`, `[[sase]]`, surrounding whitespace trimmed.
- **Order:** exact stem, then `project_name_aliases`. Case-insensitive; `-` and `_` are
  different characters; never fuzzy when mutating.
- **Aliases:** frontmatter `project_name_aliases` on area or project notes, a YAML list
  in flow (`["bob-cli"]`) or block form. Entries are strings; empty or non-string
  entries are ignored with a warning. Two notes claiming one alias is an ambiguity error
  naming both. An alias equal to another candidate's stem is shadowed (the stem wins)
  and reported.
- **Output:** the canonical route (`bob`, never `bob-cli`), its kind, label (`bob.md`),
  and how it matched (`stem` or `alias:bob-cli`).
- **Errors** name the problem and the fix:

  ```text
  bob ref create: error: no area or project named 'bob-cli'
  hint: did you mean bob (project · bob.md)?
  hint: to accept this name, add `project_name_aliases: ["bob-cli"]` to bob.md
  ```

  A terminal project reports `project 'x' is done` (or canceled); a non-parent note
  reports `obsidian_ref.md is not an area or project note`.

- **Non-goal:** ordinary task `@route` capture does not learn aliases in this epic.

### 4. The locator

One index per command (`RefTaskIndex`), never one walk per PDF.

1. **Walk** vault Markdown including `done/`, excluding hidden directories,
   `_generated/`, `_templates/`, `_conflicts/`, `lib/`, `xlib/`, and conflict copies.
   Prefilter files on a case-insensitive `#ref` or `^ref` substring before parsing. Skip
   fenced code and embed lines.
2. **v2 candidate:** a real task line (`- [m] …`, `>` quote prefixes allowed) carrying
   the whole-token tag `#ref` (case-insensitive), whose **first wikilink** resolves to a
   note under the configured ref dir. Resolve path-qualified targets by normalized path
   (keyed even when the note does not exist yet, for adoption); resolve bare stems only
   when exactly one ref note has that stem. A plain link without `#ref` never counts, so
   wrappers, follow-ups, and prose are safe.
3. **v1 tracker:** today's rule (exact `^ref` token inside the ref note itself).
4. **Selection per ref note:**
   1. the unique open v2 candidate outside `done/` is the live task;
   2. otherwise the newest closed v2 candidate (by completion/cancel date, preferring
      one outside `done/`) sets the terminal state;
   3. otherwise the v1 in-note tracker;
   4. otherwise frontmatter `status` (today's fallback).

   If an open v2 candidate and an open v1 tracker both exist, the v2 candidate wins and
   both diagnostics below fire.

5. **Residence parent:** a root note → its route; `done/X.md` → the stem of that file's
   `parent:` frontmatter link; anything else → none (diagnostic).

| Diagnostic (row `diagnostics[]`, rolled up by `bob ref doctor`) | Meaning                                                | Write behavior                                                        |
| --------------------------------------------------------------- | ------------------------------------------------------ | --------------------------------------------------------------------- |
| `multiple_open_ref_tasks`                                       | Several open tasks claim one ref                       | Status falls back to frontmatter; status/parent writes refused        |
| `open_ref_task_in_done`                                         | An open line inside `done/`                            | Report only; scan never writes into `done/`                           |
| `open_ref_without_task`                                         | Open status, v2 note, no task anywhere                 | Never recreated; hint: restore it from git or set `status: abandoned` |
| `orphan_ref_task`                                               | A `#ref` task whose first link resolves to no ref note | Report only                                                           |
| `ref_task_outside_area`                                         | Live task in a non-area, non-project, non-inbox file   | Report; suggest Ctrl+Shift+M                                          |
| `parent_mismatch`                                               | Frontmatter `parent` disagrees with residence          | Residence wins on the next scan                                       |
| `open_v1_tracker`                                               | An open in-note `^ref` remains                         | Suggest `bob ref migrate-tasks`                                       |

### 5. Status sync across files

| Event                                | Behavior                                                                                                                                                                                                                                                                                                                                                                                                                         |
| ------------------------------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| New ref (scan creates the note)      | **Birth:** resolve the marker's `parent` hint; insert the v2 line into that note's `## Tasks` through capture's insertion path (`insert_task_line` + `write_staged_files`, the bytes `bob capture @sase` writes); then write the ref note with the managed embed. An unresolvable hint files the task under `mac_inbox` with one child bullet `⚠️ parent '<hint>' is not an open area or project · refile me with Ctrl+Shift+M`. |
| Crash between the two writes         | The rerun adopts the orphaned line (locator keyed by link target) and only writes the ref note.                                                                                                                                                                                                                                                                                                                                  |
| Bryan changes the mark               | The same signal as today, read from the located line.                                                                                                                                                                                                                                                                                                                                                                            |
| Marker or frontmatter status changes | A one-line checkbox edit on the located line. At write time re-read the file; replace the line if it still equals the planned original at its index, else the unique equal line elsewhere in the file, else fail this PDF with `reading task changed during sync; rerun`. No git-dirty guard on parent notes (capture has none); the ref-note guard is unchanged.                                                                |
| Sync closes the task                 | `[completion::]`/`[cancelled::]` before the trailing block ID (generalize `close_date_insert_position` from `^ref` to any trailing `^id`). Existing dates are never touched.                                                                                                                                                                                                                                                     |
| Task archived to `done/`             | Nothing to do: the locator finds it. Scan never writes into `done/`.                                                                                                                                                                                                                                                                                                                                                             |
| Reopen signal for an archived ref    | Insert a fresh open line through the birth path into the archive's source note (its `parent:`), reusing the block ID when free. If that note is not an open candidate, use `mac_inbox` plus the ⚠️ child. The archived line is untouched.                                                                                                                                                                                        |
| Parent                               | v2 notes: residence → frontmatter `parent` (design call 1). `--write-pdfs` also refreshes a stale marker `parent`. v1 notes: unchanged.                                                                                                                                                                                                                                                                                          |
| Annotation follow-ups (J8)           | v2 default target is the live task's residence (else `mac_inbox`), inserted through the same path, with full-path `[[ref/<type>/<stem>#^h-…\|🔖]]` back-links. An explicit `@name` suffix keeps today's behavior.                                                                                                                                                                                                                |

Births and cross-file edits execute **sequentially** after parallel planning; each
re-reads its destination immediately before writing, so two refs born into `sase.md` in
one scan never lose each other. Block IDs are allocated at execute time against the
fresh destination contents.

Unchanged and worth stating: task-driven status changes still reach the PDF marker only
on `--write-pdfs` (R5); until then `bob ref list` reports `status_sync: pending` and
derives status from the task.

### 6. Freshness review (the J4 split)

- **Identity.** A row is a ref when its task line carries the whole-token tag `#ref`
  (case-insensitive), or the exact block ID `^ref` (v1). `^prj` keeps exact block-ID
  identity.
- **Ready refs** (`[ ]`): unchanged REFERENCES behavior: `reference_interval` (else the
  Ready chain), never-confirmed first, never NEW, never decide, never count keeps. Their
  Ready `state`/`bucket` are unchanged.
- **Lane refs** (`[*]`/`[/]`): ordinary lane rows: `next_interval`/`pending_interval`,
  PENDING/NEXT tiers, and the `false` off-switch behave exactly as for any lane task.
- **Visibility.** Ref tasks are visible like any task. The `#hide` bypass survives only
  for exact `^ref` rows until `closeout` (design call 10). A hand-written `#hide` on a
  v2 line hides it like any task.
- **Contract.** Rust JSON `schema_version` 12 for `bob freshness` (history line: "ref
  identity is the `#ref` tag; lane refs walk PENDING/NEXT"); bob-ledger-tools
  `api.freshness.version` 10. Rust and JavaScript change together under the shared
  vectors.

### 7. The open-book identity glyph

- **One identity slot.** On a task line, an exact `#task` immediately followed (one
  whitespace run) by `#ref` (case-insensitive) renders as **one** open-book glyph in the
  slot and teal ink `#task`'s hash uses. Shape says the kind; ink still says "tracked
  task". `#ref` anywhere else (prose, headings, code, non-task lines, `#ref #task`
  order) never renders the book.
- **Glyph:** a 16×16 stroked open-book mask in the hash's stroke language (stroke width
  1.8, round caps and joins), defined once as `--bob-ref-task-glyph` beside
  `--bob-task-tag-glyph`.
- **Behavior** matches `#task` marks: closed tasks rest at the same 40 % mix toward
  `--text-faint`; cursor or click reveals both raw tags; the session toggle, keyboard
  access, and PDF export behave as for `#task`; tooltip
  `#task #ref · reference reading task`; accessible label "Reference reading task".
- **Reading view and Tasks results:** the `#task` anchor becomes the book and the
  adjacent `#ref` anchor is hidden; in Tasks query results, where `#task` is already
  hidden by CSS, the `#ref` pill renders as the book.
- **Why a book:** `🔖` already means "jump to this highlight" on annotation follow-ups,
  which now share the same `## Tasks` lists. One symbol, one meaning.
- **Elsewhere:** Mac pickers and the Refs inspector use the SF Symbol `book`; CLI human
  output uses `📖`; plugin and Mac pickers drop `#ref` from display text as they drop
  `#task`.

### 8. Capture, Keep, create, and the SASE hook

- **`bob ref create TARGET -P PARENT`.** Required, no default, resolved before pandoc,
  the browser, the network, or any write. The marker stores the canonical route. The dry
  run prints `parent    bob  (project · bob.md · via alias bob-cli)`. Help: "Area or
  project note that owns the reading task (required)". A missing `-P` prints a one-line
  error plus a hint naming `bob capture-targets`.
- **Typed ingest and ref jobs.** `IngestRequest` gains `parent`; `INGEST_PARENT` and
  `DEFAULT_PARENT` are deleted. The job JSON gains an optional `parent` (schema stays 1;
  jobs written by an older `bob` fall back to their source inbox). A failed clip writes
  the fallback task into the **parent** note with the ⚠️ child and a retry command that
  includes `-P <parent>`.
- **Capture grammar (J11).** A whole item that is one admitted bare URL plus exactly one
  route token (`URL @sase` or `@sase URL`), or a bare URL under an `@@sase` declaration,
  or a bare URL with `-r sase`, becomes a ref whose parent is that route (aliases
  allowed). No route: parent `mac_inbox`. Anything more (an extra word, `#tag`, `s:`,
  `p:`, an operator, a child line, `-s/-t/-S/-c`, `--task-ref`) or `-R` keeps it a task;
  excluded hosts stay tasks. An unresolvable route is an error with the resolver's
  hints.
- **Capture output.** `capture --dry-run`/JSON gain
  `ref.parent: {route, label, kind, source: "explicit"|"global"|"default", alias}`. The
  human headline becomes `would queue example.com/essay → sase` with the detail
  `new to your library · reading task lands in sase.md`, or `→ mac_inbox` with
  `… · file it later`. `capture-parse` gains `ref_parent: {token, source}` on `ref`
  items, and the route token carries the ordinary `route` span so completion works on
  it. Route completion also matches `project_name_aliases`.
- **`bob gkeep pull`.** Per URL item, the parent is the first of: a trailing `@route`
  token in the Keep note (the item still counts as URL-only); `-P/--parent ROUTE`; on a
  TTY, one prompt per new URL before any clipping
  (`File example.com/essay under [gkeep_inbox]: `), each answer becoming the next
  default, unknown names re-prompting with hints; otherwise `gkeep_inbox`. The Keep
  journal stores the chosen parent so a re-pull never re-asks. `gkeep list` shows
  `🔗 ref → sase` (or `→ asks · gkeep_inbox`).
- **SASE research hook.** sase exports `SASE_FILE_HOOK_PROJECT` (the run's project
  display name; unset when missing or `unknown`). The chezmoi config becomes
  `command: bob ref create --include-id -P "$SASE_FILE_HOOK_PROJECT"`. An unknown
  project fails through the existing `❌ research-highlights failed` notification; it
  never falls back to `obsidian_ref`. The fix is an alias on the right note.

### 9. The special-case ledger

| Special case today                                                   | Fate                                                                                                  |
| -------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------- |
| `#hide` on generated ref lines                                       | **Removed** from the writer and by the migration                                                      |
| Glossary residence exception "a reference note's own reference task" | **Removed** (memory decision)                                                                         |
| `DEFAULT_PARENT` / `INGEST_PARENT` = `obsidian_ref`                  | **Removed**; parents are required and resolved                                                        |
| "`URL @route` stays a task"                                          | **Changed**: it becomes a ref under that route                                                        |
| `parse_pdf_task_line` / `find_trackers` (one exact `^ref` per note)  | **Kept read-only for v1**; v2 uses the locator                                                        |
| Dirty guard's in-note checkbox allowance                             | **Kept for v1**; v2 uses the line-level preimage guard                                                |
| Annotation follow-ups into the ref note's `## Tasks`                 | **Changed**: default to the parent (J8)                                                               |
| Audio embed anchored after `^ref`                                    | **Changed**: after the managed embed on v2 notes                                                      |
| `git log -G\^ref` date backfill (`ref list -g`)                      | **v1 only**; v2 reads the located line's fields                                                       |
| Freshness `TrackerKind::Ref` from the exact block ID (Rust and JS)   | **Re-keyed** to the `#ref` tag                                                                        |
| `#hide` bypass for ref trackers, "hidden references still count"     | **Transitional**, then **removed** in `closeout`                                                      |
| Lane `^ref` rows walking REFERENCES                                  | **Removed**: lane refs walk PENDING/NEXT                                                              |
| Ready refs walk REFERENCES, never NEW, never decide                  | **Kept**, re-keyed                                                                                    |
| "`#^ref` embeds are not dependency edges"                            | **Kept** (already generic: unmanaged sole embeds are content)                                         |
| `^prj` exemptions (move, cap, reroll)                                | **Kept**; not extended to refs                                                                        |
| Per-note Ready cap, Next/Pending caps                                | **Count refs** (`decisions:note-ready-cap-counts-the-lane`)                                           |
| `bob task archive`                                                   | **No exemption**                                                                                      |
| Mac Refs Today join on note path                                     | **Changed** to `(task.path, task.block_id)`                                                           |
| Mac Refs watcher on `ref/` and `lib/` only                           | **Changed**: also root notes and `done/`                                                              |
| Mac `^` picker                                                       | **No filter existed**; refs appear once they live in root notes. Adds the book symbol via `task_kind` |
| PDF marker as authority for `parent`                                 | **Changed** for v2: residence is the authority (design call 1)                                        |

### 10. Invariants Bryan relies on, and what changes

| You rely on…                                                        | After                                                                         | Mitigation                                                                                                |
| ------------------------------------------------------------------- | ----------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------- |
| Reading never consumes Next/Pending slots                           | **It does.** Migrating as-is: Next 13 → 21 (cap 15), Pending 9 → 17 (cap 10). | Caps warn and never refuse; triage with Alt+N or cancel. Optional decision `cancel_dropped_wrapper_refs`. |
| Reading never crowds a note's Ready cap                             | Research refs (about 6/day) count in `sase.md`/`bob.md`                       | Honest pressure: read, release, re-file into sub-projects, or set `ready_cap:`                            |
| Weekly ref review, never NEW                                        | Unchanged for Ready refs; lane refs get daily review                          | —                                                                                                         |
| `^` picker excludes reading                                         | It shows `[*]`/`[/]` refs with the book symbol                                | This is the request                                                                                       |
| Ref status visible in `refs.base`                                   | Same scan latency; CLI and Refs panel derive status live                      | —                                                                                                         |
| Closed refs stay in ref notes                                       | v2 refs archive with their parent note                                        | The locator reads `done/`                                                                                 |
| `[[x#^ref]]` links                                                  | Rewritten for migrated refs; links to closed v1 refs untouched                | —                                                                                                         |
| A project with only reading left looks empty to `bob projects sync` | It now has open work, so `^prj` stays hidden                                  | Correct: unread prerequisites are open work                                                               |
| `bob ref list -P obsidian_ref`                                      | Values become area/project routes (v1 frozen rows keep theirs)                | Release note                                                                                              |
| Scan never clobbers edits                                           | Preserved, per line instead of per file                                       | Line preimages; parent-first plus adoption                                                                |
| PDFs written only on `--write-pdfs`                                 | Unchanged (and moves never need one, design call 1)                           | —                                                                                                         |
| Task-driven status reaches frontmatter after a `--write-pdfs` scan  | Unchanged (R5)                                                                | `bob ref list` shows `pending` and the live status meanwhile                                              |
| Annotation follow-ups appear in the ref note                        | They appear in the parent, with the `🔖` back-link                            | J8                                                                                                        |

## Ground rules (every phase)

- **Repositories.**
  - bob-cli (this project) is the primary checkout.
  - Open other repos only with `/sase_repo`: `sase repo open bob-plugins -r "<why>"`,
    `sase repo open bob-mac-capture -r "<why>"`, `sase repo open chezmoi -r "<why>"`,
    and `sase repo open sase -r "<why>"` (a different SASE project's primary repo). Work
    only in the printed path and read that repo's `AGENTS.md` (or README rules) first.
  - Read research and plans with `sase artifact read`, never directly.
- **Live vault.** Only `hook-config` (the `bob.md` alias) and `live-migration` write the
  live `~/bob` vault, and only as their sections say. Every other phase tests against
  temporary fixture vaults. Run `bob vault-sync` after a live vault write.
- **Installs.** This plan authorizes `hook-config` and `live-migration` to run
  `just install` from their bob-cli checkout (master with the landed phases). No phase
  installs on the Mac; Bryan runs `just install-all` there (see
  [Rollout](#rollout-and-manual-verification-for-bryan)).
- **Thin client (`decisions:mac-capture-is-a-thin-client`).** Grammar, resolution,
  status, and parents live in bob-cli. Bob Mac Capture decodes, ranks, and presents.
- **JSON contracts grow additively.** Keep `schema_version` 1 for `ref list/show/find`,
  `capture-*`, and ref jobs, and 2 for `plan`: the Mac app rejects any other version.
  New fields are optional for clients (`decodeIfPresent`). Only `bob freshness` bumps
  (to 12), after confirming no other repo pins 11.
- **CLI rules.** Read `sase memory read cli_rules.md -r "<why>"` before adding options
  or subcommands: excellent help, alphabetical options, a short alias for every public
  long option, no hard renames. Update shell-completion specs and
  `tests/cli/help_options.rs` expectations when options change.
- **Shared names.** Later phases depend on these names; keep them, or record
  `INTERFACE CHANGE:` on the phase bead:
  - `src/native/parent_notes.rs`: `resolve_parent`, `ResolvedParent`, `ParentError`,
    `parent_candidates`.
  - `src/native/ref_tasks/`: `RefTaskIndex::build`, `LocatedRefTask` (`path`,
    `line_index`, `line`, `mark`, `block_id`, `archived`, `residence`),
    `RefTaskDiagnostic`.
  - `src/native/ref_tasks/line.rs` (from `ref-sync-v2`): `render_ref_task_line`,
    `allocate_ref_block_id`, `managed_embed_line`; `src/native/ref_tasks/insert.rs`:
    `insert_ref_task` (the birth inserter `migrate-tasks` reuses).
- **bob-plugins phases.** Edit `src/` fragments and run `npm run build`; never hand-edit
  generated `main.js`; keep fragments ≤ 1000 lines; add new test files to the
  `package.json` list; bump each changed plugin's manifest version; `npm test` must
  pass; then run `bob plugins sync`.
- **Bob Mac Capture phases** (no Swift toolchain on agent hosts). Write conservatively:
  Swift 5 mode, `@MainActor` on AppKit types, `@available(macOS 26.0, *)` on views,
  long-stable APIs, swift-format default style, no long `+` chains. This plan authorizes
  each Mac phase to commit to `master` with `/sase_git_commit` from the bob-mac-capture
  checkout (conventional headers), find the CI run with
  `gh run list -R bobs-org/bob-mac-capture --workflow CI --commit <sha> --json databaseId`,
  watch it through `/sase_monitor`
  (`gh run watch <id> -R bobs-org/bob-mac-capture --exit-status`), and fix forward until
  green (`gh run view <id> --log-failed`). A Mac phase is not done until CI is green;
  its bead note records the run URL and SHA. Download `render-fixtures` and look at
  every PNG the phase touched.
- **Docs.** Each phase documents its user-facing contract in the doc its section names.
- **Verification.** bob-cli: `just check` passes. A pre-existing unrelated failure is
  noted and left alone.
- **Follow-ups.** Record out-of-scope discoveries as `PROPOSED FOLLOW-UP:` notes on the
  phase bead; do not create beads.

## Phase `sase-hook-env`: sase file hooks export SASE_FILE_HOOK_PROJECT

**Repo:** sase
(`sase repo open sase -r "Export SASE_FILE_HOOK_PROJECT to file hook commands"`). Follow
its `AGENTS.md` (agents never commit by hand; `sase tool run check` after `just fix`).

1. **`src/sase/file_hooks/runner.py` `_execute_run`.** Build the child environment as
   `os.environ.copy()` plus `SASE_FILE_HOOK_PROJECT` = `run.get("project")`. Remove any
   inherited value first, and set none when the project is missing, empty, or
   `"unknown"`. Use `.get` so batches written by an older sase still run.
2. **Run log.** `_write_run_log` gains a `project:` line (the exported value or `-`).
3. **Docs.** `docs/configuration.md` `file_hooks` (the `command` field and the
   "Execution" bullets): name the variable, its value (the project display name, as
   `sase project list` shows it), when it is unset, and the recommended quoting
   (`-P "$SASE_FILE_HOOK_PROJECT"`). Mention it where `docs/plugins.md` shows a
   `research-highlights` template.
4. **Tests** (`tests/file_hook_engine/test_dispatch.py`, helpers in `helpers.py`): a
   hook whose command prints the variable logs the event's project; a project containing
   spaces and shell punctuation reaches a quoted `"$SASE_FILE_HOOK_PROJECT"` argument as
   one argument; `"unknown"` and a pre-set inherited value leave it unset.

**Done when** `sase tool run check` passes in the sase checkout.

## Phase `parent-resolver`: one strict parent resolver and project_name_aliases

**Repo:** bob-cli. Spec: [§3](#3-parent-resolution-and-project_name_aliases).

1. **`src/native/parent_notes.rs`** (new): `parent_candidates(bob_dir)` reuses
   `capture_targets::scan_capture_targets` (one root scan) and parses
   `project_name_aliases`;
   `resolve_parent(bob_dir, input) -> Result<ResolvedParent, ParentError>` implements
   the order, errors, and hints (near misses via `note_tasks::bounded_levenshtein`).
   `ParentError` renders the multi-line error/hint text.
2. **Alias parsing.** Flow and block YAML lists in the line-based frontmatter reader
   (`projects/scan.rs` helpers); warnings for malformed entries, duplicate claims, and
   shadowed aliases.
3. **`bob capture-targets`.** Every target gains `project_name_aliases: []` in JSON
   (additive; `schema_version` stays 1); human output shows `aka bob-cli`; `-v` prints
   alias warnings. Update the JSON-shape test.
4. **`bob ref create -P`.** When `-P` is given, resolve it before any work and store the
   canonical route in the marker; the dry run prints the parent line from §8. The
   `obsidian_ref` default stays for now (`ref-create-parent` removes it).
5. **Docs.** `docs/projects.md` gains "External names (`project_name_aliases`)";
   `docs/capture.md` documents the new `capture-targets` field;
   `docs/highlights-ref-sync.md` documents `-P` resolution.
6. **Tests.** Resolver unit tests: stem, `.md` and `[[…]]` inputs, case, alias, alias
   ambiguity, shadowed alias, near-miss hints, terminal project, non-parent hub, each
   inbox. CLI tests for `capture-targets` JSON and `ref create -P … -d`.

## Phase `freshness-rekey`: freshness keys refs on the #ref tag, with the lane split

**Repos:** bob-cli and bob-plugins. Spec: [§6](#6-freshness-review-the-j4-split).

1. **Rust** (`src/native/freshness/state.rs`, `scan.rs`): replace
   `TrackerKind::from_block_id` with identity from the row's tags and block ID (`#ref`
   tag or exact `^ref` → `Ref`; exact `^prj` → `Prj`). In the evaluator, `is_ref`
   applies the tracker cadence and REFERENCES tier only in the Ready lane; lane refs
   take the ordinary lane branches (`lane_days`, `walk_scope`, PENDING/NEXT). Keep the
   `#hide`-allowed visibility only for exact `^ref` rows (`RowCtx::freshness_row`
   callers). `^prj` behavior is unchanged.
2. **Docs** (`docs/freshness.md`): §2 (`reference_interval` applies to Ready `#ref`
   rows), §4 tier definitions, the "Tracking review" bullets, the PR/RF conformance text
   and vectors (lane refs walk PENDING/NEXT; tag-only `#ref` rows qualify; the
   transitional exact-`^ref` hide bypass), and schema 12 in the history paragraph and
   §7.
3. **JSON.** `schema_version` 12 for `bob freshness list/seed`. First grep bob-plugins
   and bob-mac-capture for a pinned freshness schema 11 and update any consumer.
4. **bob-ledger-tools** (`src/100-freshness-evaluate.js`, `160-widgets-and-row.js`,
   `110-freshness-queue.js`, `120-freshness-footer.js`, `130-freshness-marks.js`,
   `170-plugin-lifecycle.js`, `230-plugin-freshness-api.js`): replace
   `freshnessTrackerFromBlockId` with a line-based identity mirroring Rust; route lane
   refs through the ordinary lane branches (`freshnessIntervalForLine`,
   `freshnessEvaluateWithoutRecurring`); keep the hide bypass only for exact `^ref`;
   `api.freshness.version` 10 with a `refTagIdentity: true` capability; README version
   line fixed. **bob-navigation-hotkeys:** confirm the REFERENCES tier notices still
   read correctly and that nothing requires version 9 exactly.
5. **Tests.** Rust state tests and vectors; update
   `test-ledger-tools-freshness-tracking.cjs`, `…-recurring.cjs`,
   `test-navigation-freshness.cjs`, and the namespace test. New vectors: a v2 Ready ref
   walks REFERENCES (never NEW), a v2 `[*]` ref walks NEXT on `next_interval`, a v2
   `[/]` ref with `pending_interval: false` falls back like an ordinary task, a `#ref`
   line with `#hide` is hidden, an exact-`^ref` hidden v1 tracker still reviews.
6. `npm test`, `bob plugins sync`, `just check`.

## Phase `hook-config`: live alias, install, and the hook passes -P

**Repos:** bob-cli (install only), the live vault, chezmoi.

1. `just install` from bob-cli (master includes `parent-resolver`).
2. **Alias.** Add `project_name_aliases: ["bob-cli"]` to `~/bob/bob.md` frontmatter,
   leaving every other byte unchanged; run `bob vault-sync`.
3. **Verify bob.** `bob capture-targets -f json` shows the alias on `bob`;
   `bob ref create --include-id -P bob-cli -d <any research .md>` prints
   `parent    bob  (project · bob.md · via alias bob-cli)`; `-P sase` resolves by stem.
4. **Verify the live sase.** The installed sase is an editable install of the sase
   primary checkout. Confirm that it exports the variable, for example by running a
   scratch `file_hooks` dispatch or by checking the installed module source with the
   sase tool's Python. If it does not yet, ask Bryan with `/sase_questions` to update
   that checkout (or to authorize a clean `git pull --ff-only` there), and do not
   continue until it does.
5. **Hook config.** In chezmoi `home/dot_config/sase/sase.yml`, change the
   `research-highlights` hook to
   `command: bob ref create --include-id -P "$SASE_FILE_HOOK_PROJECT"`. Apply it
   (`chezmoi apply` on that target) and confirm `sase file-hook list` (or the config
   loader) shows the new command. Follow chezmoi's `AGENTS.md`: after its commit,
   `chezmoi update -a --force` runs.

**Done when** the live hook command passes `-P "$SASE_FILE_HOOK_PROJECT"`, the live sase
exports it, and `-P bob-cli` resolves to `bob`. `ref-create-parent` must not start
otherwise: once `-P` is required, an unconfigured hook would fail every research report.

## Phase `ref-locator`: the done/-aware locator and read-side contracts

**Repo:** bob-cli. Spec: [§1](#1-the-ref-task-line-v2), [§4](#4-the-locator). Read-only.

1. **`src/native/ref_tasks/`** (new): `RefTaskIndex::build(config)` implements the walk,
   candidates, selection, residence, and diagnostics. Reuse the existing note index
   (`task_status_hooks` `NoteIndex`) for link resolution, `markdown` fence helpers, and
   `collect_done` trailing-block-ID parsing. Also collect **follow-up tasks**: task
   lines anywhere whose `🔖` back-link targets a ref note (`[[ref/…#^h-…|🔖]]`).
2. **`ref_library`.** `build_index` builds one `RefTaskIndex` and feeds each row:
   - `decide_status` takes the located `TrackerHit` (v2 or v1), so the status,
     `pending`, `conflict`, and `[?]` rules apply unchanged;
   - `parent`: residence for v2, frontmatter for everything else;
   - `finished`: the located line's completion/cancel date (v2); `-g` git backfill stays
     v1-only;
   - an additive `task` object on every `list`/`show`/`find` row:
     `{path, block_id, link, mark, archived}` (`null` when no task). v1 rows report
     their in-note tracker (`block_id: "ref"`), so one join works for both eras;
   - the §4 diagnostics.
3. **`bob ref list -P NAME`.** Resolve through `resolve_parent` when it resolves (so
   `-P bob-cli` matches `bob`), else compare literally (so `-P obsidian_ref` still finds
   frozen rows). Update the option help.
4. **`bob ref show`.** Human output adds
   `📖 Reading task  sase · ⭐ Next → [[sase#^ref-harness-engineering]]`, or
   `📖 Read · 2026-10-12 · sase (archived)`. JSON `tasks[]` also lists follow-up tasks
   found elsewhere, with an additive `path`.
5. **`bob ref doctor`.** A `ref tasks` row (`ok (N live · M archived · K open v1)`, or a
   warning rolling up diagnostics with up to 3 example paths) and a `parents` row for
   alias problems. Warnings never fail the command.
6. **`capture-complete` and pickers.** Every task candidate whose line starts with the
   `#task #ref` pair gains `task_kind: "ref"` (omitted otherwise), and
   `note_tasks::clean_description` drops a `#ref` token that directly follows the global
   filter.
7. **Performance.** Measure `bob ref list -R all -A -f json` on athena before and after;
   the locator should add at most about 150 ms. Report both numbers on the bead.
8. **Docs.** `docs/ref.md` (reading state from the located task, the `task` object,
   diagnostics, `-P`, show); `docs/capture.md` (`task_kind`).
9. **Tests.** A fixture vault with: v1 open and closed trackers; v2 lines in root area
   and project notes; an archived v2 line in `done/x_done.md` with `parent:`; two open
   candidates for one ref; an orphan; the `papers`/`blogs` stem collision with bare and
   path-qualified links; a managed embed line and a fenced code block (both ignored); a
   `#ref` line in a daily note (`ref_task_outside_area`); a wrapper task without `#ref`
   (not a candidate). Snapshot `list`/`show` JSON for the new fields.

## Phase `ref-glyph`: the open-book identity glyph and picker text

**Repos:** bob-cli (docs) and bob-plugins. Spec: [§7](#7-the-open-book-identity-glyph).

1. **`docs/task-tag-marks.md`** (bob-cli) gains a "Reference reading tasks" section and
   new TT/TR conformance rows: the pair on open and closed lines (one range covering
   both tokens), case-insensitive `#REF`, extra whitespace between the tokens,
   `#task #references` (hash only), `#ref #task` (hash only on `#task`), `#ref` alone, a
   non-task line, inline code, and reading-view anchors.
2. **bob-ledger-tools.** `138-task-tag-marks.js`: detect the pair and return a `ref`
   kind range spanning both tokens; widget and element builders draw the book.
   `264-plugin-task-tag-marks.js`: Live Preview decorations replace the whole pair; the
   reading-view post-processor turns the `#task` anchor into the book and hides the
   adjacent `#ref` anchor; reveal-on-cursor exposes both tags; the session toggle and
   teardown cover the new class. `styles.css`: `--bob-ref-task-glyph`, the ref variant
   rules, the closed-task faint rule, and a Tasks-results rule that draws the book on
   the `#ref` pill. Copy the new vectors into
   `scripts/test-ledger-tools-task-tag-marks.cjs` and extend the CSS contract check.
3. **Picker text.** block-id-prompt `stripInternalTaskTags` and navigation-hotkeys
   `stripTaskTag`/`cleanTaskDisplayText` drop the `#ref` that follows `#task` and prefix
   `📖 `, so pickers, the Task Card header, and inbox-route subtitles read
   `📖 Harness Engineering…`. Add tests.
4. Bump versions, `npm test`, `bob plugins sync`.

## Phase `ref-sync-v2`: scan writes reading tasks into parent notes

**Repo:** bob-cli. Spec: [§1](#1-the-ref-task-line-v2),
[§2](#2-the-ref-note-after-the-move), [§5](#5-status-sync-across-files), design calls
1–4.

1. **`ref_tasks/line.rs`:** `render_ref_task_line` (mark, ref-note path, title, created
   date, block ID, optional close stamp), title-alias sanitizing,
   `allocate_ref_block_id` (destination contents, its `done_tasks` file, reserved set),
   and `managed_embed_line`.
2. **`ref_tasks/insert.rs`:** `insert_ref_task(bob_dir, destination, line, children)`
   through capture's `insert_task_line` + `write_staged_files` with preimage retries,
   returning the final block ID and placement. `migrate-tasks` reuses it.
3. **Planning** (`highlights_ref/sync.rs`, `note.rs`, `model.rs`): `scan_library` and
   `sync_pdf` build one `RefTaskIndex`. For each PDF decide v1 or v2 (§2).
   - **v1** keeps today's path untouched.
   - **v2 existing:** the status signal comes from the located task; a needed mark
     change is planned as a line edit on the located file (refused for `done/` and for
     `multiple_open_ref_tasks`); `parent` is excluded from the projection, hash, and
     base and rendered from residence; the embed is healed; annotation candidates target
     the residence.
   - **Birth:** a missing note is born v2: resolve the marker's parent hint, plan the
     insertion (or adopt a located orphan), and render the note body with the embed
     instead of a tracker.
4. **Execution.** Parallel planning stays; births, line edits, routed annotation groups,
   and reopen-from-archive insertions execute sequentially, each re-reading its
   destination. Write order per PDF: destination note(s), then the PDF marker (only with
   `--write-pdfs`), then the ref note. `--write-pdfs` also refreshes a marker `parent`
   that differs from residence.
5. **Guards.** The ref-note git-dirty guard is unchanged for v1 and allows, for v2, a
   tracked ref note whose only body difference from `HEAD` is the managed embed line
   (plus frontmatter). Destination notes get preimage checks only.
6. **Anatomy.** `region.rs` recognizes the managed embed; `audio.rs` anchors the audio
   embed after it; `annotation_tasks.rs` writes full-path `🔖` links for v2 targets.
7. **Output.** Concise scan output adds
   `📖 N reading tasks created · sase.md (1) · bob.md (1)`, `↻ N reading tasks updated`,
   and, when any remain, `N open v1 ref tasks · run bob ref migrate-tasks`. Verbose and
   dry-run output name each destination, block ID, and line edit. `bob ref sync <pdf>`
   reports the same.
8. **Docs.** `docs/highlights-ref-sync.md`: "What it does", the generated-note shape,
   the task-line status rules (now "the located reading task"), `## Tasks` follow-up
   placement, parent projection, the guard, and births/adoption.
9. **Tests** (temp vaults, `BOB_NOW` fixed): birth into a note with and without
   `## Tasks`; two births into one note in one scan (both lines, unique IDs); the
   `harness-engineering` / `harness-engineering-2` collision against a `done_tasks`
   file; unresolvable hint → `mac_inbox` plus the ⚠️ child; crash simulation (parent
   written, ref note missing) → rerun adopts with no duplicate; marker status change →
   one-line edit plus stamp in a dirty parent note; a concurrent edit to the located
   line → refused, nothing else written; archived task → status read, no write into
   `done/`; reopen signal on an archived ref → fresh line in the archive's parent; move
   between notes → frontmatter `parent` follows with no marker write and no
   `--write-pdfs` error; embed heal after a manual embed deletion; v2 annotation
   follow-up lands in the parent with a full-path back-link; a v1 note's bytes are
   unchanged by a scan.

## Phase `mac-refs-v2`: Bob Mac Capture reads located ref tasks

**Repo:** bob-mac-capture. Spec: [§4](#4-the-locator),
[§7](#7-the-open-book-identity-glyph).

1. **Decoding** (`Sources/RefsCore/RefsDecoding.swift`): `RefRecord.task: RefTask?`
   (`path`, `blockID`, `link`, `mark`, `archived`) with `decodeIfPresent`, plus the
   hand-written `encode(to:)` used by the snapshot cache. `RefsPlanTodayTask` decodes
   `block_id`.
2. **Today join** (`RefsToday.swift`): key Today entries by `(path, block_id)`; a ref is
   in Today when its `task` matches. Rows without `task` (an older `bob`) fall back to
   today's note-path join.
3. **Refresh** (`Sources/BobMacCapture/Refs/RefsLibrary.swift`): FSEvents also triggers
   on vault-root `*.md` and `done/` changes, with the same 0.5 s debounce.
4. **Presentation.** The inspector shows a Reading task row: SF Symbol `book`, parent
   label, and lane label (`sase · Next`), or `Archived · done/sase_done`. Parent
   captions use the real route; drop the `_ref`-stripping heuristic for rows whose
   `task` is v2. Search keeps matching the parent.
5. **Pickers.** `CaptureCompletionCandidate` decodes `task_kind`; the `^`, `:`, `+`,
   `&`, and `!` pickers draw SF Symbol `book` before the text when it is `"ref"` (text
   already arrives without `#ref`).
6. **Fixtures and tests.** v2 `refs-list` rows with `task`, a `refs-plan` fixture with
   `block_id`s, a Today-join test for two refs sharing `sase.md`, an older-bob fixture
   without `task`, and picker fixtures with `task_kind`. README `## Bob Refs` updated.
7. CI loop until green; check the Refs and picker PNGs.

## Phase `ref-create-parent`: bob ref create requires -P; ingest, jobs, and fallbacks carry the parent

**Repo:** bob-cli. Spec: [§8](#8-capture-keep-create-and-the-sase-hook). Requires
`hook-config` to have switched the live hook.

1. **`bob ref create`** (`highlights_ref/create.rs`, `cli.rs`): `-P/--parent` required
   with no default (custom missing-parent error plus hint); delete `DEFAULT_PARENT`;
   update help, `docs/highlights-ref-sync.md`, `docs/highlights-create.md`, and
   `docs/highlights-clip.md`. The hidden `bob highlights create` alias behaves
   identically.
2. **Typed ingest** (`highlights_ref/ingest.rs`): `IngestRequest.parent` (a resolved
   route); delete `INGEST_PARENT`; update the module doc.
3. **Ref jobs** (`ref_jobs/spool.rs`, `worker.rs`, `fallback.rs`, `output.rs`): optional
   `parent` on `JobFile`/`NewJob` (serde default; schema stays 1); the worker passes it
   to ingest; a job without one uses its source's inbox (`capture` → `mac_inbox`); the
   fallback task goes to the parent note with the ⚠️ child and
   `bob ref create '<url>' -P <parent>`; `bob ref jobs` shows `→ <parent>`.
   `docs/ref-jobs.md`.
4. **Interim callers.** Capture's `plan_ref_item` stages `parent: mac_inbox` and
   `gkeep pull` passes `gkeep_inbox` until `capture-gkeep-parent` replaces them.
5. **Tests.** Missing `-P` error text; alias resolution end to end in a dry run; job
   round trip with and without `parent`; a fallback written into the parent with the
   retry command; help and completion expectations.

## Phase `capture-gkeep-parent`: capture URL @route and gkeep pull choose the parent

**Repo:** bob-cli. Spec: [§8](#8-capture-keep-create-and-the-sase-hook).

1. **Grammar** (`capture_language/item.rs` `claim_reference_item`, `draft.rs`): claim a
   bare URL plus exactly one leading or trailing route token, a bare URL under an `@@`
   declaration (instead of re-parsing it as a task), and a bare URL with a forced `-r`
   route. Resolve the parent with `resolve_parent`; an unresolvable route is an item
   error carrying the resolver's hints. Every other marker keeps today's task behavior.
2. **Plan and output** (`capture/plan.rs`, `output.rs`, `batch.rs`): stage the job with
   the resolved parent; report `ref.parent`; the human headline and detail from §8; the
   fallback preview names the parent note.
3. **`capture-parse`:** additive `ref_parent: {token, source}` on `ref` items; the route
   token gets the `route` span. **`capture-complete`:** route candidates match
   `project_name_aliases` (`match_kind: "alias"`, ranked after prefix matches).
4. **`bob gkeep pull`** (`gkeep/plan.rs`, `pull.rs`, `cli.rs`, `ledger.rs`, `list.rs`,
   `ui.rs`): the URL-only rule admits one trailing `@route` token; `-P/--parent ROUTE`;
   the TTY prompt (stdin and stderr are terminals, not `--quiet`, not JSON) asks per new
   URL before clipping; the journal's `ref_created` event stores `parent`; `gkeep list`
   shows the planned parent. Unresolvable note routes warn and fall through to the next
   source.
5. **Docs.** `docs/capture.md` ("Saving links to your reading queue", the grammar table,
   `capture-parse`, the dry-run verdict table; drop `@route` from "anything more keeps
   it a task"), and `docs/gkeep.md`.
6. **Tests.** `URL @sase`, `@sase URL`, `URL @bob-cli` (alias), `URL @nope` (error with
   hint), `URL @sase extra` (task), `@@sase` with three URLs, `-r sase URL`, `-R`, an
   excluded host with a route (task); parse spans and `ref_parent`; route completion by
   alias; gkeep URL-only with a trailing route, `-P`, a scripted TTY prompt (sticky
   default, unknown name re-prompts), non-TTY default, and a journal replay that does
   not re-ask.

## Phase `migrate-tasks`: bob ref migrate-tasks

**Repo:** bob-cli. Model the command on `bob ref migrate-zorg`
(`ref_library/migrate_zorg/`) and reuse `ref_tasks::insert_ref_task`,
`render_ref_task_line`, and the `collect_done/link_repair.rs` machinery.

```text
bob ref migrate-tasks [-b|--bob-dir DIR] [-f|--format human|json|tsv] [-m|--map FILE]
                      [-o|--offline] [-r|--ref-dir PATH] [-w|--write]
```

1. **Scope.** Every ref note with an open v1 in-note tracker (`[ ]`, `[*]`, `[/]`,
   `[?]`). Closed trackers and legacy notes are never touched.
2. **Parents.** For each ref: the map row (`-m`, TSV `ref_note<TAB>parent`, `#` comments
   allowed) wins; else the frontmatter/marker parent when it resolves to a candidate;
   else the ref is **unmapped** and `--write` refuses the whole run, listing them. Never
   guess from titles. `-f tsv` prints the editable map
   (`ref_note  parent  status  title`, parent blank when unmapped).
3. **Per ref** (a pure plan first):
   - render the v2 line from the v1 line: keep the mark, `[fresh::]`, `[refresh::]`,
     `[keeps::]`, priority, schedule, close fields, `[id::]`, `[dependsOn::]`, and other
     tags; drop `#hide` and the PDF link; add the path-qualified ref-note link with the
     title alias, `[created::]` when absent, and `^ref-<slug>`;
   - move every child line with it (Depends-On lines, work logs) and any **open**
     follow-up task blocks from the ref note's `## Tasks` (rewriting compact
     `[[#^h-…|🔖]]` links to full paths);
   - insert into the parent's `## Tasks` through `insert_ref_task`, or adopt an existing
     v2 line for that ref (re-run after a crash);
   - replace the v1 block with the managed embed and set frontmatter `parent`, rewriting
     the hash and base under the v2 rules. No PDF writes.
4. **Graph rewrite** in one pass over the vault: every link or embed (`[[…]]`, `![[…]]`,
   `~~[[…]]~~`, `[…](…)`) whose target resolves to a migrated tracker (`x#^ref`,
   path-qualified or a unique bare stem) becomes `<parent>#^ref-<slug>`, keeping `!`,
   `~~`, and aliases; every `[id::]`/`[dependsOn::]` value equal to the old
   `dependency_id(ref note, "ref")` becomes the new `dependency_id(parent, block ID)`.
   Ambiguous bare-stem links are reported and left alone.
5. **Never** stamp `[fresh::]`, never touch Today membership beyond rewriting links, and
   never change a mark.
6. **Report.** Headline `29 open ref tasks → 12 notes`, a table (stem, mark, parent, new
   block ID, source of the parent), links and dependency IDs to rewrite by file,
   `possible_wrapper` (open tasks that link or embed a migrated ref, excluding Pomodoro
   Task Links and Depends-On lines), ambiguous links, unmapped refs, and a lane-cap
   preview (`Next 13 → 21 / 15`).
7. **Write flow** (as migrate-zorg): take `bob_sync.lock`, require a git worktree,
   pre-sync unless `--offline`, re-plan under the lock (an empty plan prints
   `nothing to migrate`), keep before-images, write, then **verify** through rebuilt
   indexes: each migrated ref resolves to exactly one live v2 task with unchanged
   status, reading state, `blocked`, and fields; each rewritten link resolves; a second
   plan is empty. On failure restore before-images only where the after-image still
   matches, and exit 1 without committing. Commit exactly the written paths as
   `bob ref migrate-tasks: <N> ref tasks into <M> notes`, then post-sync unless
   `--offline`. JSON carries `mode` and `commit`.
8. **Docs.** `docs/ref.md` "Migrating open ref tasks (`bob ref migrate-tasks`)" with the
   rollback runbook (one `git revert`).
9. **Tests.** A fixture vault reproducing the live shapes: `[ ]`/`[*]`/`[/]`/`[?]`
   trackers with `#hide`, `[fresh::]`, `[id::]`, `[dependsOn::]`; a Depends-On child
   line pointing at two trackers and the dependent's path-derived `[dependsOn::]`;
   daily-note Task Links; project-note child embeds by path and by bare stem; the stem
   collision; an unmapped ref (refusal); a TSV map round trip; crash recovery via
   adoption; verify failure restoring before-images; a second run reporting
   `nothing to migrate`.

## Phase `mac-file-under`: Bob Mac Capture asks where a captured link belongs

**Repo:** bob-mac-capture. Spec: [§8](#8-capture-keep-create-and-the-sase-hook).

1. **Decoding.** `CaptureRef.parent` (`route`, `label`, `kind`, `source`, `alias`);
   capture-parse `ref_parent`; `CaptureTarget.projectNameAliases` as an optional
   property, so targets from an older `bob` still decode.
2. **File under.** When analysis reports mode `ref` with
   `ref_parent.source == "default"` for a single-item draft, and the dry-run verdict
   will queue (`not_found`, `legacy`, `unknown`), open the route completion list on its
   own with the header **File under**: cached inbox, area, and project targets, the
   last-used parent first, then `mac_inbox`, then the usual ranking; typing filters,
   aliases included (`bob · aka bob-cli`). Accepting inserts ` @<route>` at the end of
   the draft through the existing `setPlainDraft` + `scheduleAnalysis` pattern. Esc
   dismisses and keeps the default; the list does not reopen until the URL changes.
   Library hits (`in_library`, `in_intake`, `clipping`, `duplicate`) never open it.
   Follow the existing completion key semantics for ↵ and ⇥.
3. **Preview and confirmation.** The ref preview reads
   `📖 Queue · example.com/essay → sase`, or `→ mac_inbox · file it later`, and an
   explicit `@route` shows the parent's kind. The post-submit confirmation and
   notification read `Queued for reading → sase`. Remember the last-used parent (app
   settings) after a successful submit.
4. **Route completion** everywhere ranks alias matches after prefix route matches.
5. **Tests and fixtures.** Parse and dry-run fixtures with `ref_parent`/`ref.parent`,
   targets with and without aliases, the auto-open rules (opens, never for library hits,
   not again after Esc), and a render fixture of the File under state. README updates.
6. CI loop until green; check the capture PNGs.

## Phase `live-migration`: migrate the live vault

**Repos:** bob-cli (install), the live vault.

1. **Install.** `just install` from bob-cli; `bob plugins sync`.
2. **Mac readiness (hard gate).** The Mac's scheduled scan must run the new `bob` (it
   writes ref tasks), or it will create v1 notes and fight v2 parents. Try
   `ssh -o ConnectTimeout=5 mac 'zsh -lc "bob ref migrate-tasks --help >/dev/null"'`. If
   that fails or the Mac is offline, ask Bryan with `/sase_questions` to run
   `just install-all` on the Mac (bob, plugins, and the app) and confirm before
   continuing.
3. **Census and map.** `bob ref migrate-tasks -f tsv`; fill parents from
   [the parent table](#migration-parent-table). A ref missing from the table whose
   frontmatter or marker parent resolves keeps it. For any other unmapped ref, ask Bryan
   with `/sase_questions` (stem, title, a suggested parent); never guess.
4. **Dry run** with `-m`. Read the whole report: `possible_wrapper`, ambiguous links,
   the cap preview. Stop and ask Bryan if anything contradicts this plan.
5. **Write.** `bob ref migrate-tasks -m <map> -w`, then `bob task reconcile` and
   `bob vault-sync`.
6. **Optional cancellations.**

   > [!decision] cancel_dropped_wrapper_refs Cancel the migrated reading tasks for
   > `agent_history_in_agents_tab`, `agent_instructions_budgeted_router`,
   > `toobig_split_beyond_python`, and `research_swarm_improvement_roadmap`: set each
   > line to `[-]` with `[cancelled:: <today>]` before its block ID (the bytes
   > Obsidian's cancel writes), then `bob task reconcile` and `bob vault-sync`.

   > [!decision] cancel_dropped_wrapper_refs = no Leave all marks exactly as migrated.

7. **Verify and record on the bead:**
   - `bob ref list -R all -A -f json`: every migrated ref has a v2 `task`, the same
     status/reading state/`blocked` as before (except accepted cancellations), and
     `parent` equal to its map row;
   - `bob ref doctor`: zero `open_v1_tracker`, no new diagnostics;
   - `bob ref migrate-tasks`: `nothing to migrate`;
   - `bob ref scan --dry-run`: no planned writes to migrated notes;
   - `bob plan`: refs in Next/Pending/Ready with lane counts and cap warnings;
   - `bob freshness list`: Ready refs in REFERENCES, lane refs in PENDING/NEXT;
   - `bob task reconcile`: no new unresolved references;
   - `grep` the vault for `#^ref]]` links to migrated notes: none.
8. Report the lanes over cap and suggest triage; do not release or cancel anything
   beyond the decision above.

## Phase `closeout`: retire the transitional bypass, docs coherence, memory, final report

1. **Bypass.** If `bob ref doctor` reports zero open v1 trackers, delete the
   exact-`^ref` `#hide` review bypass in Rust (`freshness`) and bob-ledger-tools
   (including the "hidden references still count" status-bar path), update
   `docs/freshness.md` and the vectors, bump the plugin version, `npm test`,
   `bob plugins sync`. If any remain, keep it and record a follow-up.
2. **Docs coherence.** Read `docs/ref.md`, `docs/highlights-ref-sync.md`,
   `docs/capture.md`, `docs/gkeep.md`, `docs/ref-jobs.md`, `docs/freshness.md`,
   `docs/task-tag-marks.md`, `docs/projects.md`, and the docs index end to end; fix
   contradictions and stale `^ref`/`#hide`/`obsidian_ref` statements (the frozen-v1
   descriptions stay, labeled as such).
3. **Memory.**

   > [!decision] memory_ref_parent_decision Use `/sase_memory_write` to add the
   > `decisions` strand `ref-tasks-live-with-their-parent` ("Ref Tasks Live With Their
   > Parent Note"): Applies to (bob-cli, bob-plugins, Bob Mac Capture, vault), Claim
   > (R1–R8 in short: residence is the parent, `#ref` plus a path-qualified link is the
   > identity, `^ref-<slug>` is only an address, only Ready refs keep REFERENCES), Why
   > with the rejected alternatives from the research, Cost (lane and cap pressure,
   > vault-wide locator walk, Mac install coupling), Reopens when (`orphan_ref_task`
   > appears in practice, or the daily lane review of refs is rubber-stamped), and
   > Evidence (the research report, this plan, the phase commits). Link
   > `[[task-lanes-are-sticky]]`, `[[review-walk-is-tiered]]`,
   > `[[note-ready-cap-counts-the-lane]]`, and `[[mac-capture-is-a-thin-client]]`.

   > [!decision] memory_glossary_ref_terms Update three glossary strands:
   > `glossary:reference-task` becomes the single `#task #ref` reading task for a
   > reference note, living in an area/project/inbox note's `## Tasks` with a
   > `^ref-<slug>` block ID, identified by the tag plus its path-qualified link, with
   > the mark mapping and the frozen in-note `^ref` form for closed legacy refs;
   > `glossary:reference-note` says the note holds no open tasks, embeds its reading
   > task, and projects `parent` from where that task lives; `glossary:area-note` drops
   > "a reference note's own [[reference-task]]" from its residence exceptions.

   After any accepted memory edit run `sase memory init`. For a memory decision `%auto`
   left off, record a `PROPOSED FOLLOW-UP:` note on this phase's bead instead.

4. **Final checks.** `just check` in bob-cli; `npm test` in bob-plugins; the last Mac CI
   run green; `bob ref doctor` clean.
5. **Proposed follow-ups** (bead notes only): making the task the sole status authority
   so task-driven changes never wait for `--write-pdfs`; alias-aware ordinary `@route`
   capture; a "reading task created in sase" notification after the Mac scan; anything
   the phases recorded.
6. **Leave Bryan the checklist below** in the final report, stating plainly what was and
   was not verified.

## Migration parent table

Bryan reviews this table during plan approval (edit it in feedback). `live-migration`
uses it as the map. Every target exists and is an open capture target (checked
2026-10-09).

| Ref (`ref/…`)                                                                   | Mark  | Parent                  | Note                                                    |
| ------------------------------------------------------------------------------- | ----- | ----------------------- | ------------------------------------------------------- |
| `chat/agent_history_in_agents_tab`                                              | `[*]` | `sase_agent_history`    | wrapper cancelled 10-06                                 |
| `chat/agent_image_generation_toolset`                                           | `[*]` | `sase_art`              |                                                         |
| `chat/agent_instructions_budgeted_router`                                       | `[?]` | `sase_memory`           | depends on `sase#^dynamic-agents-md`; wrapper cancelled |
| `chat/auto_directive_autonomy_policy`                                           | `[*]` | `sase`                  |                                                         |
| `chat/goals_redesign_recent_reading_list`                                       | `[*]` | `sase_goals`            |                                                         |
| `chat/memory_and_instruction_file_inspiration_reading_list`                     | `[*]` | `sase_memory`           |                                                         |
| `chat/memory_built_instruction_migration_epics`                                 | `[*]` | `sase_memory`           |                                                         |
| `chat/research_swarm_improvement_roadmap`                                       | `[*]` | `sase`                  | wrapper cancelled 10-08                                 |
| `chat/toobig_split_beyond_python`                                               | `[*]` | `sase_toobig_symvision` | wrapper cancelled 10-06                                 |
| `chat/athena_cpu_saturation_orphaned_load_loops`                                | `[/]` | `sase`                  |                                                         |
| `chat/auto_autonomy_epic_roadmap`                                               | `[/]` | `sase`                  |                                                         |
| `chat/databricks_omnigent_job_fit`                                              | `[/]` | `job`                   |                                                         |
| `chat/sase_launch_post_introduction`                                            | `[/]` | `sase_blog_0`           | prerequisite of `sase_blog_0`'s review task             |
| `chat/sase_launch_post_outline`                                                 | `[/]` | `sase_blog_0`           | same                                                    |
| `chat/sase_listen_plugin_commands`                                              | `[/]` | `sase`                  |                                                         |
| `chat/sase_task_bead_48h_impact_rating`                                         | `[/]` | `sase_better_tasks`     |                                                         |
| `chat/tools_bg_split_and_tool_run_visibility`                                   | `[/]` | `sase`                  |                                                         |
| `chat/global_just_recipe_completion`                                            | `[ ]` | `dev`                   |                                                         |
| `chat/goals_inspiration_round_two_reading_list`                                 | `[ ]` | `sase_goals`            |                                                         |
| `chat/sase_md_instruction_delivery`                                             | `[ ]` | `sase_memory`           |                                                         |
| `chat/sase_next_direction_reading_list`                                         | `[ ]` | `sase`                  |                                                         |
| `chat/ref_tasks_move_into_parent_notes`                                         | `[ ]` | `bob`                   | bob-cli research (this epic)                            |
| `chat/unblocked_successor_links_on_close`                                       | `[ ]` | `bob`                   | bob-cli research                                        |
| `papers/harness_engineering`                                                    | `[ ]` | `sase`                  | the papers ref, not the read blogs ref                  |
| `papers/agentic_software_engineering`                                           | `[ ]` | `sase`                  |                                                         |
| `papers/what_does_a_harness_buy_tokens`                                         | `[ ]` | `sase`                  |                                                         |
| `blogs/introducing_omnigent_meta_harness_combine_control_and_share_your_agents` | `[ ]` | `sase`                  |                                                         |
| `blogs/the_dot_and_the_swarm`                                                   | `[ ]` | `sase`                  |                                                         |
| `blogs/understanding_is_the_new_bottleneck`                                     | `[ ]` | `dev`                   |                                                         |

## Rollout and manual verification for Bryan

**Order that matters.** `sase-hook-env` → `hook-config` → `ref-create-parent` (or
research reports fail once `-P` is required). `ref-sync-v2` → `migrate-tasks` →
`live-migration` (the migration uses the shipped writer). `mac-refs-v2` before
`live-migration` (or the Refs panel's Today section empties for migrated refs).

**On the Mac,** run `just install-all` before `live-migration` (it updates the `bob`
that scans, the plugins, and the app), and again after `mac-file-under` lands.

**Checklist:**

1. Paste a new article URL into Bob Mac Capture: File under opens, ↵ inserts
   ` @<parent>`, the preview names it, and the confirmation says
   `Queued for reading → <parent>`. Esc on another URL files it under `mac_inbox`.
2. After the next scan the reading task sits in that note's `## Tasks` and renders as
   the teal open book; the ref note shows it through the embed.
3. `^` in Bob Mac Capture lists Next/In Progress reading tasks with the book symbol.
4. Link a ref from a Pomodoro: it is in Today in `bob plan` and in the Refs panel's
   Today section.
5. Ctrl+Shift+M a ref task to another project: `bob ref show` reports the new parent at
   once; the ref note's `parent` follows after the next scan with no `--write-pdfs`
   error.
6. Check one `[x]`: `bob ref list` shows it read immediately; frontmatter follows after
   `bob ref scan --write-pdfs`, as before.
7. The next SASE research report from `bob-cli` lands in `bob.md`, one from `sase` in
   `sase.md`.
8. `bob gkeep pull` with a URL-only Keep note asks where to file it.
9. The morning `]s` walk shows Ready refs in REFERENCES and Next/Pending refs in their
   lanes.

## Out of scope

- Activating zorg-era legacy notes or closed v1 trackers (J7).
- A `## References` section or any non-archiving home for reading tasks (J6).
- Making the task the sole status authority (R5 is kept; recorded as a follow-up).
- Alias-aware ordinary task `@route` capture.
- A `bob projects doctor` command, a `finished` frontmatter projection, and
  `SASE_FILE_HOOK_*` siblings beyond the project.
- Renaming `refs.base` views or other vault dashboards.
