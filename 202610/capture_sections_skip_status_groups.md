---
tier: tale
title: Keep generated status-group headings out of @route# section capture
goal: '`@note#` section completion, `bob capture-sections`, shell `--section` completion,
  and `@note#prefix` / `--section` bullet placement all skip the `Next & In Progress`,
  `Blocked`, and `Done & Canceled` headings that `bob task reconcile` generates under
  a Tasks heading, while authored sections keep working.'
size: small
proposed_by: bbugyi200.apollo.93
proposed_with: claude/opus
status: done
---

# Plan: Keep reconcile's generated status-group headings out of `@route#` section capture

## Problem

In Bob Mac Capture, typing `… @sase_blog#` opens the section completion menu, and its
first three rows are the H3 headings `Next & In Progress`, `Blocked`, and
`Done & Canceled`. `bob task reconcile` generates those headings under `## Tasks` (see
`docs/task-status-hooks.md` § "Status Grouping"). They belong to the Tasks section and
should never be offered as bullet-capture sections. The real authored sections
(`Requirements`, `Screenshots`, …) come after them.

Mac Capture is a thin client (decision `mac-capture-is-a-thin-client`). It shows the
`Candidates::Section` rows that `bob capture-complete` returns, so the fix belongs
entirely in bob-cli. No bob-mac-capture change is needed.

## Root cause

`src/native/capture/sections.rs` drops only the heading titled exactly `Tasks`:

- `non_tasks_section_headings()` keeps every ATX heading except `Tasks`. It is the
  single source for three surfaces:
  - `capture-complete` section candidates
    (`capture_complete/candidates.rs::section_candidates`).
  - `bob capture-sections` (`capture_sections.rs::list_capture_sections`).
  - Shell completion for `capture --section` (`completion/providers.rs::sections`).
- `target_bullet_section()` (used by `insert_bullet_line()` for both typed
  `@route#prefix` and forced `--route … --section TITLE`) uses the same
  `heading.title != "Tasks"` test.

So placement has the same bug as completion. A dry run against a copy of the live
`sase_blog.md` (layout: `## Tasks` at line 12, `### Next & In Progress` at 20,
`### Blocked` at 25, `### Done & Canceled` at 44, `## Requirements` at 51) shows:

- `bob capture --dry-run 'hello @sase_blog#'` inserts at line 22, inside
  `### Next & In Progress`. A bare `#` picks the first non-H1 non-Tasks heading, and
  that is currently the generated group.
- `@sase_blog#Bl` inserts at line 27 (`### Blocked`), and `@sase_blog#Done` inserts at
  line 46 (`### Done & Canceled`).

A plain bullet inside a managed group also breaks the next reconcile. Group bodies must
hold only task blocks, so reconcile reports `ownership_collision` and leaves that whole
Tasks container ungrouped. The fix must therefore cover completion and placement
together, so that a picked row and a typed selector still agree.

## Design

### Which headings are reserved

`docs/task-status-hooks.md` defines the reconciler's ownership. A heading directly under
a `Tasks` container, or under an authored topic inside one, is a status group when its
title is exactly `Next & In Progress`, `Blocked`, or `Done & Canceled` (`TITLE_ACTIVE` /
`TITLE_BLOCKED` / `TITLE_CLOSED` in `src/native/task_status_groups/mod.rs`).

For capture, treat a heading as a **reserved status-group heading** when both of these
hold:

- Its title equals one of those three constants (exact match, as in
  `GroupKind::from_title`).
- It has an ancestor heading whose title passes the reconciler's `is_tasks_title`
  (case-insensitive `Tasks`).

The rule deliberately does **not** check that the body holds only movable tasks. A group
that already has a stray bullet in it (the corruption this bug causes) must stay hidden
and untargetable, not come back as a target.

Authored headings stay as they are:

- Authored topic headings under Tasks (for example `### Ideas` or `### Backend`) stay
  listed. The existing tests assert this, and the user only asked about the generated
  groups.
- A `## Blocked` heading outside any Tasks subtree is an ordinary authored section and
  stays listed and targetable.

### Where the rule lives

Put the rule next to the reconciler's own heading model, so that it uses the same title
constants, Tasks predicate, frontmatter/fence/HTML-comment masking, and heading tree. Do
not copy the titles into capture code.

In `src/native/task_status_groups/mod.rs`, add a crate-visible helper, for example:

```rust
/// Zero-based line indexes of the status-group headings the reconciler
/// reserves: a `Next & In Progress`, `Blocked`, or `Done & Canceled` heading
/// anywhere below a `Tasks` heading. Capture never offers or targets them as
/// bullet sections.
pub(crate) fn reserved_group_heading_lines(contents: &str) -> BTreeSet<usize>
```

To implement it:

1. Build the tree the way `transform` does: `source_lines` → `heading_mask` →
   `discover_headings` → `build_tree`.
2. Walk the tree recursively, carrying an `under_tasks` flag. When
   `under_tasks && GroupKind::from_title(&node.title).is_some()`, insert
   `node.heading_line - 1`. `heading_line` is 1-based.
3. Recurse into children with `under_tasks || is_tasks_title(&node.title)`.

Indexes from `source_lines` match capture's `line_spans` indexes, because both split on
`\n` and count a trailing unterminated segment. That lets capture filter its own
`MarkdownHeading.line_index` against the set. Where the two heading scanners disagree
(setext headings, HTML comments), the intersection is safe: a heading is hidden only if
both scanners see a reserved heading on that line.

### Capture changes (`src/native/capture/sections.rs`)

- Add one private predicate shared by listing and placement, for example
  `fn is_bullet_section(heading: &MarkdownHeading<'_>, reserved: &BTreeSet<usize>) -> bool`,
  which returns `heading.title != "Tasks" && !reserved.contains(&heading.line_index)`.
  Keep the existing exact-`Tasks` check unchanged.
- `non_tasks_section_headings(contents)`: compute
  `task_status_groups::reserved_group_heading_lines(contents)` once and filter with the
  predicate. Keep the function name and signature, so the three callers are fixed with
  no edits. Update its doc comment to say generated status groups are excluded.
- `insert_bullet_line`: compute the reserved set from `contents` and pass it into
  `target_bullet_section` (new parameter). Replace the three `heading.title != "Tasks"`
  checks in `target_bullet_section` (the `matches`, `raw`, and `identity` closures) with
  the predicate. Leave the section-end computation (`headings.get(pos + 1)`) on the full
  heading list. The section that follows a target still ends at the next heading of any
  kind.
- The documented no-match fallback is unchanged. A selector that matches only a
  generated group (for example `@note#Blocked` when no authored `Blocked` heading
  exists) now falls back to the pre-heading section, like any other unmatched selector.
  A bare `@note#` lands in the first authored non-H1 section (`## Requirements` for
  `sase_blog.md`).
- `capture/sections.rs` already reaches `task_status_groups` (it calls
  `task_status_groups::is_badge_row`), so no new module wiring is needed beyond
  importing `BTreeSet` if it is not already in scope.

## Tests

Add focused tests. Each must fail before the fix and pass after.

1. **Helper unit tests** in `src/native/task_status_groups/tests.rs`:
   - A note with `## Tasks`, a badge row, intake tasks, and the three generated H3
     groups reports exactly those three line indexes.
   - Not reserved:
     - A `## Blocked` outside Tasks.
     - An authored `### Backend` under Tasks.
   - Reserved: a nested `#### Blocked` under an authored `### Backend` inside
     `## Tasks`.
   - Not reserved: a group title inside a fenced block or YAML frontmatter.
   - A lowercase `## tasks` container still reserves its groups.
2. **Capture unit tests** in `src/native/capture/tests/started.rs`, next to
   `non_tasks_section_headings_match_bullet_heading_scan`: `non_tasks_section_headings`
   on a reconciled note returns the authored headings only. It keeps an authored
   `### Ideas` under Tasks and a top-level `## Blocked` outside Tasks.
3. **Placement tests** in `src/native/capture/tests/placement.rs`:
   - A bare-marker capture (`insert_bullet_line(contents, BULLET, None)`) on a note
     whose first non-H1 heading is a generated `### Next & In Progress` lands in the
     next authored section.
   - `Some("Bl")` skips the generated `### Blocked` and lands in a later authored
     `## Blog Ideas`.
   - `Some("Done")` with no authored match falls back to the zeroth section.
   - The exact/forced path
     (`super::super::insert_bullet_line(.., Some("Blocked"), true)`) does not target the
     generated group.
4. **CLI integration tests**:
   - `tests/cli/capture/complete_query.rs`: `capture-complete` on `"Idea @notes#"`
     against a reconciled-shaped `notes.md` returns `context == "section"` with
     candidate titles excluding the three group headings.
   - `tests/cli/capture/sections.rs`: `capture-sections --format json` reports the right
     `count` and `sections` for the same shape.

Use a realistic reconciled fixture copied from the example in
`docs/task-status-hooks.md` § "Status Grouping": badge row, intake tasks, and the groups
holding `- [/] #task …`, `- [?] #task …`, `- [x] #task …` lines.

## Docs

Bring the docs in line with the new behavior:

- `docs/capture.md` § "Section bullets": after "or any non-`Tasks` section when the
  marker is a bare `#`", add that the `Next & In Progress`, `Blocked`, and
  `Done & Canceled` status-group headings `bob task reconcile` generates under a `Tasks`
  heading belong to that Tasks section. They are never bullet targets, so `@note#` and
  `@note#Bl` skip them. Mention the same exclusion in the `--section` paragraph ("place
  the bullet in a non-`Tasks` heading …").
- `docs/capture.md` § "Discovery commands": change "except a heading titled exactly
  `Tasks`" to also exclude the generated status-group headings below a `Tasks` heading.
- `docs/task-status-hooks.md` § "Status Grouping": add one sentence saying that
  `bob capture` section completion, `bob capture-sections`, and `@route#` placement skip
  these headings.
- `src/native/capture/cli.rs` long help: add a short clause after "or any non-Tasks
  section for a bare '#'" and in the `--section` paragraph. `bob capture --help` is not
  snapshot-tested.
- `src/native/capture_sections.rs` `long_about`: add one sentence saying that generated
  status-group headings under Tasks are omitted. Leave the one-line `about` text
  unchanged so `tests/fixtures/help/root-long.txt` and `src/runner.rs` stay in sync
  without edits.

No SASE memory note describes section completion, so no memory change is needed.

## Verification

1. Run `just check`: fmt, clippy with the repo's rustc gate, and every test binary.
2. Manual regression against a scratch copy of the vault note, never the live vault:
   - Copy `~/bob/sase_blog.md` into a temp dir and run
     `bob capture-sections -b <tmp> -r sase_blog`. It should list only `Requirements`,
     `Screenshots`, `Contents`, `Outline`, and `Later Posts`.
   - `bob capture -b <tmp> --dry-run --format json 'hello @sase_blog#'` should report an
     `open_notes[0].line` inside `## Requirements` (after line 51), not line 22.
   - `bob capture-complete -b <tmp> -c <len> -f json -- 'x @sase_blog#'` should return
     no group rows.

   Use the freshly built binary (`cargo run --` or `target/debug/bob`).

3. Run `just install` so the local `bob` picks up the fix. The Mac host's `bob` needs
   the same update (for example `just install-all` there) before Mac Capture shows the
   change.
