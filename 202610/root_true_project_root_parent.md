---
tier: tale
title: "root: true makes a project note its own root parent"
goal:
  "A project note with root: true groups its own and its sub-projects' working time as
  its own root parent in bob stats, the Day strip, and the stats dash, starting with
  sase_blog."
size: small
decisions:
  glossary_strands:
    ask:
      "Update the Root Parent and Top-Level Project glossary strands to name the root:
      true override?"
    memory:
      - glossary:Root Parent
      - glossary:Top-Level Project
    corrects:
      The [[area-note]] a task lives in, else the [[top-level-project]] above its
      project note.
    default: true
    answer: true
  decision_record:
    ask: "Add a decision record that root: true makes a project its own root parent?"
    memory:
      - decisions:root-flag-makes-a-root-parent
    default: false
    answer: false
proposed_by: bbugyi200.athena.14f
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-10-26 05:28:35
status: wip
---

# Plan: `root: true` makes a project note its own root parent

## Context

Bryan wants to mark a project note as a **root parent** with a new frontmatter field,
`root: true`. The Day strip bar and the stats dash's default "Where the time went" view
would then show that project as its own root, instead of folding it into the top-level
project above it. The first user is `~/bob/sase_blog.md`.

### How root parents work today

- **Rule.** `docs/pomodoro-homes.md` R2 defines the root parent as the area note a task
  lives in, else the top-level project above its project note.
- **Code.** `HomeResolver::resolve_note` in `src/native/pomodoro_homes/resolve.rs` walks
  the `parent:` chain. When the nearest home `chain[h]` is a project, it takes the
  **last** project in `chain[h..a]`, where `a` is the first area index above `h` (or the
  chain's end). `Home.root_parent` carries the result.
- **Facts.** `NoteFacts` in `src/native/pomodoro_homes/facts.rs` parses `type`,
  `parent`, and `color` from frontmatter, using the line-based helpers in
  `src/native/projects/scan.rs` (`frontmatter_value`, `trim_yaml_scalar`).
- **Stats.** `src/native/stats/homes.rs` groups per-task shares by `home.root_parent`.
  It fills `parts.root_parent`, `parts_summary.root_parent`, `apportioned.root_parent`,
  the count-mode lead root, and the catalog's `homes[<key>].root_parent` (`ensure_root`
  and `catalog_entry`). The global inspect pass already parses every `[[project]]` note
  (`needs_global_inspect`), so a new project frontmatter key is seen.

### Consumers need no code change

- **Day strip and stats dash** live in bob-plugins (`bob-ledger-tools`). They only
  render bob's `root_parent` parts and the catalog's `root_parent`
  (`239-plugin-stats-panel-models.js`: `statsPanelHomeSessions` and
  `statsPanelRootSubRows`). A sub-project marked `root: true` will appear as its own
  root row under its area group, with its own sub-project disclosure. It also drops out
  of the old root's disclosure. The plugin's distinct-hue logic gives the new bar
  segment its own colour. No plugin edit or `bob plugins sync` is needed.
- **Bob Mac Capture** never reads `root_parent`; I checked its Swift sources.
- **Shared home vectors.** `tests/fixtures/pomodoro_homes/vectors.json` and its
  byte-identical bob-plugins copy carry no `root_parent` expectations. Leave both files
  untouched.

### History

- `0cf45b8c` retired `rollup: subprojects`, which grouped descendant projects **under**
  an ancestor.
- `5078e8f8` then made top-level projects the default grain.
- `root: true` is the inverse direction: it opts one sub-project, and everything below
  it, **out** of its top-level project's root. It does not revive rollup.
- No vault note uses a `root:` key today.

### Live evidence (read-only `bob stats show worked -r 30d -f json`, 2026-10-26)

- `sase.md` is a 2630m root.
- The catalog maps `sase_blog.md` (272m of leaf time) and its sub-project
  `sase_pre_blog.md` (60m) to `sase.md`.
- After this change, both should map to `sase_blog.md`, and the totals stay the same.

## Contract

### Grammar

- Write `root: true` in the frontmatter of a project note (`type: "[[project]]"`).
- Read the value with `frontmatter_value(fm, "root")`, then `trim_yaml_scalar`. It turns
  the flag on only when it equals `true` ASCII-case-insensitively, so `"true"` and
  `True` count.
- Any other value is off: absent, `false`, `yes`, empty, or a list. It is off silently,
  with no lint and no warning, the same way `parent:` silently ignores list forms.
  Obsidian's checkbox property writes a bare `true`.
- The key is ignored on every non-project note: areas, sub-areas, `done/` archives,
  `ref/` notes, and plain notes.

### Resolution (R2, project branch)

- Let `span = chain[h..a]`. The root is the **first** project in `span` whose facts set
  `root`. If none does, the root is the **last** project in `span`, which is today's
  rule.
- The home itself (`chain[h]`) is in the span, so a flagged project is its own root.
- The nearest flag wins, so a flagged sub-project of a flagged project is its own root.
- A flag on a top-level project is a no-op.
- A flag above the first area is never consulted, because the span stops there.
- Area-less walks use the same span, up to the chain's end.

### Examples (Bryan's vault after this change)

| Task lives in                          | Walk                                     | Root parent |
| -------------------------------------- | ---------------------------------------- | ----------- |
| `sase_goals.md` (project)              | `sase_goals → sase → dev`                | `sase`      |
| `sase_blog.md` (project, `root: true`) | `sase_blog → sase → dev`                 | `sase_blog` |
| `sase_pre_blog.md` (project)           | `sase_pre_blog → sase_blog → sase → dev` | `sase_blog` |
| `done/sase_blog_done.md` (archive)     | `→ sase_blog → sase → dev`               | `sase_blog` |

### What stays the same

- The home `key`, `label`, and `kind`.
- `area`, `subarea`, and `hue`. The root keeps its top-level area's hue (R8).
- `residence_chain`, so `in:` and `exclude_in:` filters still match the full chain:
  `in: [sase]` still includes blog time.
- The `project` dimension and R5 allocation.
- The R9 session `homes` object.
- The JSON shape. This is a value change only: no schema bump, no new field, and no `!`.

### What changes

- The count-mode lead root follows the new roots.

### Scope: reporting only

The flag changes nothing else:

- blocking and `#prj_is_blocked` inheritance;
- Sub-projects lines;
- Ready caps;
- review tiers;
- capture routing;
- the plan budget;
- `bob projects` output.

## Steps

1. **Facts.** In `src/native/pomodoro_homes/facts.rs`, add `pub(crate) root: bool` to
   `NoteFacts` and parse it per the Grammar above. Store the raw flag. The resolver
   applies the project-only rule.

2. **Resolver.** In `resolve_note`'s `NoteKind::Project` arm in
   `src/native/pomodoro_homes/resolve.rs`:
   - prefer the first `kind == Project && facts.root` entry in
     `chain[nearest_home..end]`;
   - otherwise fall back to the existing `.last()` project.

   Update the `Home.root_parent` doc comment to name the flag. Change nothing else in
   the resolver.

3. **Resolver unit test.** Add `root_flag_makes_a_project_its_own_root_parent` to
   `src/native/pomodoro_homes/tests.rs`, in the style of
   `root_parent_follows_the_examples_table`. Cover:
   - a flagged sub-project is its own root;
   - its sub-project, a `done/` archive, and a `ref` note under it root there;
   - an unflagged sibling still folds to the top-level project;
   - nested flags: the nearest flag wins;
   - a flagged top-level project is unchanged;
   - a flag on an area, ref, or archive note is ignored;
   - `root: "true"` and `root: True` count; `root: false` and `root: yes` do not;
   - the flagged home's `key`, `area`, `hue`, and `residence_chain` match the unflagged
     resolution.

4. **Stats test.** Add a test in the `src/native/stats/homes.rs` test module:
   - Copy `tests/fixtures/stats/homes_vault` into a `TempDir`. Reuse or mirror
     `copy_tree` from `src/native/stats/compute.rs` tests.
   - Insert `root: true` into the copy's `sase_blog.md` frontmatter.
   - Compute with the existing `homes_request` / `write_config` helpers.
   - Assert that worked `root_parent` is `sase.md` 30 and `sase_blog.md` 50, with every
     other H16 root unchanged and the sum still 167.
   - Assert that the `area` and `project` parts equal H2 and H3.
   - Assert that the `sase_in` metric is still 80 and `sase_blog_in` still 50.
   - Assert that catalog `homes["sase_blog.md"].root_parent == "sase_blog.md"`.
   - Recompute the count-mode `sessions` root split by hand and assert it; the H5 lead
     homes suggest 1 `sase.md` and 1 `sase_blog.md`.
   - Do **not** edit the shared fixture or the existing H-vector tests.

5. **Contract docs**
   - **`docs/pomodoro-homes.md` R2:**
     - Add the flag to the opening definition.
     - Add the "first flagged project in the span, else the last project" rule to the
       project bullet.
     - Add a short "Root projects (`root: true`)" paragraph with the Grammar and Scope
       above.
     - In the examples table, change the plain sub-project row to `sase_goals.md`. Add
       the `sase_blog` (`root: true`), `sase_pre_blog`, and archive rows from the
       Examples table above.
     - Keep the `in: [sase_blog]` sentence.
   - **`docs/projects.md`**, "Project Notes" section: add one paragraph on optional
     `root: true` (reporting only) that links to pomodoro-homes R2.
   - **`docs/reference/vault-format.md`**, "Project notes": add one sentence with the
     same link.
   - **`docs/stats.md`**, the H table: add an H17 row for the step-4 test (`root: true`
     on sase_blog: root_parent sase 30, sase_blog 50; area, project, and `in:` values
     unchanged). Then state H17's setup in one sentence after the fixture-day prose.
   - Change no other docs. The bob-plugins mounted reference stays correct.

6. **Memory**, only for accepted memory decisions; follow `/sase_memory_write`.
   - `glossary_strands`, Root Parent strand: rewrite the first sentence to read, in
     substance, "The area note a task lives in, else the nearest project marked
     `root: true` on the walk up to that area, else the top-level project above its
     project note." Keep the rest of the strand.
   - `glossary_strands`, Top-Level Project strand: change "belongs to the top-level
     project above it" to say that it reports under that project unless it, or a project
     between them, sets `root: true`.
   - `decision_record`: add a new strand to the `decisions` memory web with slug
     `root-flag-makes-a-root-parent`, using the decisions-web shape (Applies to, Claim,
     Why, Cost, Reopens when; `metadata.status: accepted`, `decided: 2026-10-26`).
     - **Claim:** the Grammar, Resolution, and Scope above.
     - **Why:** Bryan asked for a `root: true` frontmatter field so that sase_blog reads
       on its own in the Day strip and the dash. Cite this plan.
     - **Rejected alternatives:**
       - Revive `rollup:`. It groups the other way and was retired in `0cf45b8c`.
       - A `stats:` config list of root notes. It goes stale on renames, and the fact
         belongs to the note.
       - Leaf-project grouping by default. It brings back about 82 `sase_*` rows.
   - Then run `sase memory init` and `just docs-gen`. The generator rewrites
     `docs/reference/glossary.md`, and `docs/develop/decisions.md` if the record was
     added. With no memory decision accepted, edit no memory and skip both commands.

7. **Vault.** In `~/bob/sase_blog.md`, add the line `root: true` directly after
   `type: "[[project]]"` in the frontmatter, and change nothing else.
   - Do not run git in `~/bob`; `bob vault-sync` commits and pushes it.
   - The `bob-project-tasks` plugin rewrites `task_count` / `open_task_count` through
     Obsidian's `processFrontMatter`, which keeps unknown keys.
   - An older installed `bob` silently ignores the key until it updates.

8. **Gates and live check**
   - Run `just check` and `just docs-check`.
   - Then run a read-only check from the repo:

     ```bash
     cargo run --quiet -- stats show worked -r 30d -f json
     ```

     Confirm all of the following:
     - `parts_summary.root_parent` has a `sase_blog.md` row of about 332m;
     - `sase.md` dropped by the same amount;
     - `area` and `project` totals and `worked`'s total are unchanged;
     - catalog `sase_pre_blog.md` and `sase_blog.md` both have
       `root_parent: "sase_blog.md"`.

     Report the numbers.

   - Commit as `feat(homes): let root: true make a project its own root parent`. Never
     edit `Cargo.toml`'s version or `CHANGELOG.md`; the daily release train ships it.
     The Day strip and stats dash pick it up once the installed `bob` includes this
     commit.

## Risks

- **Silent no-op on typos.** A value such as `root: yes` is ignored without a warning.
  The Day strip makes the miss visible at once, and Obsidian's checkbox writes `true`.
  Revisit with a lint if typos turn up.
- **Past minutes regroup.** Per R10, past minutes regroup under the new root the moment
  the flag lands. Totals never change. This matches how moving a project already
  behaves.
