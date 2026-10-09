---
tier: tale
title: Locate ref reading tasks anywhere in the vault and expose them read-side
goal: '`bob ref list/show/find/doctor` and `bob capture-complete` read the reading
  state, parent, finished date, and task identity of every reference from its located
  `#task #ref` line (live or archived in `done/`), with v1 in-note `^ref` trackers
  unchanged. Phase bead bob-cli-5y.5 (`ref-locator`) is then closed so `ref-sync-v2`
  (bob-cli-5y.7) and `mac-refs-v2` (bob-cli-5y.8) can start.'
size: medium
proposed_by: bbugyi200.athena.0z5
status: done
---

# Plan: the done/-aware ref-task locator and read-side contracts (bob-cli-5y.5)

## Context

This tale implements phase `ref-locator` (bead **bob-cli-5y.5**) of the epic
`plan:202610/ref_tasks_live_with_parent.md` (bead bob-cli-5y). Read its "Design
specification" §1 (the v2 ref task line) and §4 (the locator) with
`sase artifact read plan:202610/ref_tasks_live_with_parent.md "<why>"`. This plan
restates everything you need and makes the open design calls. Where this plan is more
specific than the epic, follow this plan.

The phase is **read-only**. It never writes the vault. `ref-sync-v2` (scan births, line
edits, embed healing) and `migrate-tasks` come later and build on the API defined here.

Background facts, all checked on 2026-10-09:

- Today every open reference has a **v1 tracker**: one task line inside its own ref
  note, for example
  `- [ ] #task #ref [[lib/blogs/x.pdf]] [fresh:: 2026-10-07] #hide ^ref`.
  `src/native/ref_library/status.rs` `find_trackers`/`parse_tracker_line` reads it, and
  `decide_status(body, front)` applies the status rules.
- A **v2 task** lives in a root area, project, or inbox note, normally under `## Tasks`:
  `- [*] #task #ref [[ref/papers/harness_engineering|Harness Engineering…]] [created::2026-10-09] ^ref-harness-engineering`.
  Its first wikilink targets the ref **note**, not the PDF. Nightly `bob task archive`
  may move a closed line to `done/<stem>_done.md`. That archive file's frontmatter
  carries `parent: "[[<source>]]"` and `type: "[[done]]"`.
- The live vault has no v2 lines yet and no `#ref` task lines outside `ref/`. It has 29
  open v1 trackers.
- Baseline timing, already recorded on the bead: `bob ref list -R all -A -f json` took
  52–60 ms on athena (installed bob, 953 ref notes).

## Scope

In scope:

1. A new module `src/native/ref_tasks/`.
2. `ref_library` integration: status, parent, finished date, the `task` object, and
   diagnostics.
3. `bob ref list -P` resolution.
4. `bob ref show` (human, markdown, and JSON).
5. Two new `bob ref doctor` rows.
6. `capture-complete` `task_kind` and the `#ref` text cleanup.
7. Docs and tests.
8. A performance measurement.
9. Closing the bead.

Out of scope, owned by later phases:

- Any vault write.
- `render_ref_task_line`, `allocate_ref_block_id`, `managed_embed_line`, and
  `insert_ref_task`.
- Generalizing `close_date_insert_position`.
- `region.rs` anatomy for the embed.
- The Mac app.
- Making `-P` required on `bob ref create`.
- The `#hide` bypass.
- Memory notes. Per the epic's DECISIONS, do **not** edit any memory note; see the
  closeout step.

## Design

### 1. Module layout and shared names

Create `src/native/ref_tasks/` and declare `pub(crate) mod ref_tasks;` in
`src/native.rs`. Suggested files: `mod.rs` (types and `build`), `walk.rs` (walk and
prefilter), `line.rs` (candidate and follow-up parsing), `select.rs` (selection and
residence), `v1.rs`, `embed.rs`, `dates.rs`, and `tests.rs`. Keep each file under about
600 lines.

Later phases depend on these names, from the epic's "Shared names". Keep them:

- **`RefTaskIndex::build`.** Takes `(bob_dir: &Path, ref_dir: &Path)` and returns
  `RefTaskIndex`. It never errors; problems become `warnings`.
- **`LocatedRefTask`.** Has at least these fields:
  - `path: String`: vault-relative file, for example `sase.md` or `done/sase_done.md`.
  - `line_index: usize`: 0-based, counted over the whole file.
  - `line: String`: the raw line without its newline.
  - `mark: char`
  - `block_id: Option<String>`: the trailing `^id`, via
    `collect_done::trailing_block_id_in_line`.
  - `archived: bool`: the path is under `done/`.
  - `residence: Option<String>`

  Add `closed_on: Option<String>` (the completion or cancel date) and
  `in_capture_target: bool`.

- **`RefTaskDiagnostic`.** Fields `{ code: &'static str, detail: String }`, converted to
  `ref_library::Diagnostic` on rows.

Mark the fields and methods that only later phases use (for example `line_index` and
`line`) with `#[allow(dead_code)]` and a comment such as
`// Reserved for phase ref-sync-v2.`. This mirrors the existing
`// Reserved for the planner phase` pattern in `ref_library/mod.rs`.

Move the v1 tracker parser into `ref_tasks/v1.rs` as `pub(crate)`, verbatim, so
`ref_tasks` never depends on `ref_library`:

- `TrackerHit { mark, line }`
- `find_trackers(body)`
- `parse_tracker_line`
- `managed_region_line_range`

`ref_library::status` then imports them. Also move the date helpers from
`ref_library/mod.rs` (`bracket_task_date`, `emoji_task_date`) into `ref_tasks/dates.rs`
as `pub(crate)`. Expose them as `fn close_date(line: &str) -> Option<String>`, which
accepts `[completion:: D]`, `[cancelled:: D]`, `✅ D`/`✔ D`, and `❌ D`/`✖ D`.
`ref_library`'s `finished_date` keeps using them.

### 2. The walk (`RefTaskIndex::build`)

1. **Paths.** Use
   `capture_dependency_tasks::vault_note_paths(bob_dir) -> (Vec<PathBuf>, Vec<String>)`.
   It returns sorted vault-relative `.md` paths, includes `done/`, excludes
   dot-directories and `_conflicts/_generated/_templates`, and puts walk problems in its
   warnings.
   - Additionally drop paths whose first component is `lib` or `xlib`, and conflict
     copies. A conflict copy's file name contains ` (conflict`, ` (Conflicted copy`, or
     `.sync-conflict-`.
   - Extract that file-name test from `ref_library/mod.rs` `collect_member_files` into
     one shared `pub(crate)` helper that both use.
2. **Ref-note set.** `ref_rel` is `ref_dir` relative to `bob_dir`.
   - If `ref_dir` is not under `bob_dir`, return an empty index with the warning
     `ref dir is outside the vault; reading tasks were not located`.
   - Ref notes are the walked paths under `<ref_rel>/`. Skip any path with a `*.assets`
     directory component, matching `ref_library`'s membership rules.
   - Build a case-insensitive stem map from ref-note stems to paths.
3. **Read and prefilter.** Read files in parallel the way
   `capture/dependencies.rs::read_prefiltered_parallel` does: scoped workers, an atomic
   index, and a merge back into path order. Keep a file only if its bytes contain any
   of:
   - `#ref` or `^ref`, ASCII case-insensitive, or
   - `🔖` (follow-up back-links carry neither `#ref` nor `^ref`).

   Scan bytes with `memchr`/`memmem` or a manual loop; do not allocate a lowercased copy
   of each file. An unreadable file becomes a warning.

4. **Lines.** For each kept file:
   - Skip frontmatter (`markdown::strictly_closed_frontmatter_end`) and fenced lines
     (`markdown::fenced_lines`).
   - Strip blockquote prefixes: up to 3 spaces, `>`, an optional space, repeated. Mirror
     the private `freshness/placement.rs` `strip_blockquote_prefix`; promote it or copy
     it.
   - Accept only list task lines: indentation, then `-`, `*`, or `+`, a space, `[m]`,
     and then end-of-line or whitespace.
   - Lines that are not tasks, including embed lines such as `- ![[ref/chat/x#^ref]]`
     and the managed embed, are ignored.
5. **v2 candidate.** A task line qualifies when all of these hold:
   - Its whitespace tokens include `#ref`, ASCII case-insensitive. `#references` and
     `#ref/x` do not count.
   - It does **not** carry the exact whitespace token `^ref`. Such lines are v1
     trackers, handled by `v1.rs` from the ref note body, never as candidates or
     orphans.

   Resolve its **first** wikilink (`task_dependencies::raw_wikilink_spans(line)[0]`).
   Take the target as the text before `|`, then before `#`, trimmed.
   - **Path-qualified** (contains `/`). Normalize with
     `vault_links::target_to_markdown_path`.
     - Under `<ref_rel>/`: key by the exact path if it is a ref note, else by a
       case-insensitive match against the ref-note set, else by the normalized path
       itself. The last case covers a note that does not exist yet; `ref-sync-v2` adopts
       it.
     - Not under `<ref_rel>/`: orphan, reason `not_a_ref_link`.
   - **Bare stem.** Look up the stem map. Exactly one ref note means that key. Zero
     means orphan (`no_ref_note`). Several means orphan (`ambiguous_stem`, naming the
     candidates). So `[[harness_engineering]]` never resolves when `ref/blogs/` and
     `ref/papers/` both have that stem.
   - **No wikilink:** orphan (`no_link`).
   - A candidate keyed to a path outside the ref-note set is stored under that key for
     adoption **and** listed as an orphan (`no_ref_note`).

6. **Follow-up tasks.** On any task line, look at every wikilink whose alias is exactly
   `🔖` (`highlights_ref` `SOURCE_LINK_ALIAS`; expose it `pub(crate)`) and whose
   fragment is `#^h-…`.
   - Resolve the target part the same way as candidates.
   - When it resolves to an existing ref note and the line's file is not that ref note,
     record `RefFollowUp { path, line_index, mark, text, block_id }` under that ref
     note.
   - `text` and `block_id` come from the existing `## Tasks` parser in
     `highlights_ref/region.rs` (`parse_region_task`). Expose it as
     `pub(crate) fn parse_follow_up_task(line) -> Option<RegionTask>`, so both paths
     strip the `🔖` link and inline fields identically.
   - Same-note links (`[[#^h-…|🔖]]`) are never attributed to another file.
7. **Residence**, stored on each candidate:
   - A root file (no `/`): `residence = Some(stem)`.
     `in_capture_target = stem ∈ routes`, where `routes` comes from
     `capture_targets::scan_capture_targets(bob_dir).targets`. Compute it **lazily**,
     only if some candidate lives in a root file, so today's vault (no v2 lines) pays
     nothing.
   - `done/…`: `archived = true`. `residence` is the stem (last path segment, without
     `.md`) of the file's frontmatter `parent:` link. Read it from the already-loaded
     contents with `collect_done`'s `archive_parent_target`; promote it from
     `pub(super)` to `pub(crate)` with a re-export.
   - Anything else: `residence = None`.
8. `closed_on = dates::close_date(line)`. `RefTaskIndex` derives `Debug, Clone`.

Accessors:

- `candidates(ref_path) -> &[LocatedRefTask]`
- `follow_ups(ref_path) -> &[RefFollowUp]`
- `orphans() -> &[OrphanRefTask]`
- `warnings()`

`OrphanRefTask` has `{ path, line_index, reason, target }`.

### 3. Selection (`RefTaskIndex::select`)

`select(&self, ref_path: &str, v1_hits: &[TrackerHit]) -> RefTaskSelection` is pure and
reusable by `ref-sync-v2`. Terminal marks are `x`, `X`, and `-`. Every other mark,
including `?` and unknown marks, is open.

Apply the first rule that matches:

1. **Exactly one open candidate outside `done/`:** it is the live task (`Selected::V2`).
2. **Two or more open candidates outside `done/`:** no selection. Emit
   `multiple_open_ref_tasks` (the detail lists `path:line` for each), and status falls
   back to frontmatter.
3. **Otherwise, closed candidates:** the newest by `closed_on`. Undated lines sort
   oldest. Break ties by preferring outside `done/`, then path, then line index.
4. **Otherwise, v1:** pass `v1_hits` (from `v1::find_trackers` on the ref note body)
   through unchanged. That includes today's `multiple_ref_trackers` behavior inside
   `decide_status`.
5. **Otherwise:** nothing, so frontmatter decides.

Also apply these, whatever rule matched:

- Each open candidate under `done/` emits `open_ref_task_in_done` (report-only; it is
  never selected).
- An open v1 tracker emits `open_v1_tracker`, even when a v2 task wins. The detail is
  `open in-note ^ref tracker; run bob ref migrate-tasks to move it into its parent note`,
  plus `(a v2 reading task also claims this ref)` when rule 1 or 3 matched. This is how
  this plan reads the epic's "both diagnostics fire".
- A selected **live** task with `!in_capture_target` (non-root, or a root note that is
  not an area, non-terminal project, or inbox) emits `ref_task_outside_area`:
  `reading task lives in <path>, not an area, project, or inbox note; refile it with Ctrl+Shift+M`.

`RefTaskSelection` carries:

- `task`: `Option<Selected>`, where `Selected` is `V2(LocatedRefTask) | V1(TrackerHit)`.
- `status_hits: Vec<TrackerHit>`: what `decide_status` receives. For v2 that is
  `[TrackerHit { mark, line: line.trim() }]`; empty for rule 2.
- `v2: bool`: true when any candidate exists for this ref. The caller ORs in the managed
  embed (§5).
- `residence: Option<String>`, from the selected v2 task.
- `diagnostics`.

`pub(crate) const REF_TASK_DIAGNOSTIC_CODES` lists every code in this plan:

- `multiple_open_ref_tasks`
- `open_ref_task_in_done`
- `open_ref_without_task`
- `orphan_ref_task`
- `ref_task_outside_area`
- `parent_mismatch`
- `open_v1_tracker`

### 4. `ref_library` integration

- **`decide_status`.** It becomes `decide_status(hits: &[TrackerHit], front)`. The body
  of the function is otherwise unchanged, so the `[?]`, pending, conflict, and
  `unknown_ref_mark` rules apply unchanged to v2. Update its unit tests, which pass
  `&find_trackers(body)` or `&[]`.
- **`build_index`.** Build one `RefTaskIndex::build(&config.bob_dir, &config.ref_dir)`
  and keep it on the index as `pub ref_tasks: RefTaskIndex`. Push its warnings to stderr
  only from the commands that already print warnings (doctor); `list`/`show` stay quiet.
- **`build_row`.** It gains a selection argument. `migrate_zorg/plan.rs` also calls
  `build_row`; give it the empty selection that `select` returns for a path with no
  candidates and no hits.
  - Status: `decide_status(&selection.status_hits, &front)`.
  - **parent:** for a v2 selection with `residence = Some(r)`, the parent is `r`.
    Otherwise it stays the frontmatter parent (today's `bare_parent_name`).
  - **parent_mismatch:** a selected v2 task with residence `r`, and a frontmatter parent
    that is missing or not ASCII-case-equal to `r`. Detail:
    `frontmatter parent <x|missing> ≠ residence <r>; the next scan sets it`.
  - **open_ref_without_task:** the note is v2, meaning a candidate exists or the body
    holds the managed embed (§5). It applies when there are no candidates and no v1
    hits, and the decided status is `ready`, `next`, or `wip`. Detail:
    `status <s> but no reading task found; restore it from git or set status: abandoned`.
  - **finished:** unchanged code. It reads the selected hit's line, which is now the v2
    line when one is selected.
- **New `RefRow` fields.**
  - `task: Option<RefTaskView>`, serialized right after `parent` and always present
    (`null` when there is no task). `RefTaskView` is
    `{ path, block_id: Option<String>, link: String, mark: char, archived: bool }`:
    - **v2:** `path` is the residence file. `link` is
      `[[<path without .md>#^<block_id>]]`, or `[[<path without .md>]]` when the line
      has no block ID. A root note therefore reads `[[sase#^ref-x]]`, an archive
      `[[done/sase_done#^ref-x]]`.
    - **v1:** `path` is the ref note, `block_id` is `"ref"`, `link = [[ref/…/x#^ref]]`,
      and `archived` is false. One `(path, block_id)` join then works for both eras.
    - Rule 2 (ambiguous) and "no task" both give `null`.
  - `#[serde(skip_serializing)] pub v2: bool`.

  Update every `RefRow { … }` literal: `ref_library/mod.rs`, and the tests in
  `output.rs`, `show.rs`, and `coverage.rs`.

- **`fill_git_dates` (`-g`).** Stays v1-only: skip rows with `v2 == true` when choosing
  the rows to backfill.
- **`library_counts`** is unchanged.

### 5. Managed-embed detection (`ref_tasks/embed.rs`)

`pub(crate) fn find_managed_embed(body: &str) -> Option<ManagedEmbed { line_index, target, block_id }>`
looks between the H1 and the first `##` heading (or the managed region begin), skipping
fenced lines. It returns the first line whose trimmed text is exactly one block embed
`![[<target>#^<id>]]`: no alias, nothing else on the line, and `parse_block_link_inside`
accepts the inside. No ref note has such a line today. `ref-sync-v2` will reuse this for
healing.

### 6. `bob ref list -P NAME`

In `cli.rs` `list_selection`:

1. If `parent_notes::resolve_parent(bob_dir, NAME)` succeeds, match rows whose `parent`
   ASCII-case-equals either the canonical route or the normalized input
   (`normalize_parent_input`). So `-P bob-cli` matches `bob`.
2. On any `ParentError`, compare the normalized input literally, ASCII case-insensitive.
   So `-P obsidian_ref` and `-P sase_ref` still find frozen rows.

`-P` never errors. `filters.parent` in the JSON envelope echoes the canonical route when
the input resolved, else the normalized input.

New help text:
`Only notes whose parent is NOTE: an area or project route or a project_name_aliases entry (bob-cli matches bob); other names match the frontmatter parent literally`.
Keep options alphabetical. Read `sase memory read cli_rules.md -r "<why>"` first. Update
the `README.md` `bob ref list` synopsis if it describes `-P`.

### 7. `bob ref show`

- **JSON.** The flattened row now carries `task`. `tasks[]` also lists the follow-ups
  found elsewhere (`index.ref_tasks.follow_ups(&row.path)`), after the in-note ones.
  `ShowTask` gains
  `#[serde(skip_serializing_if = "Option::is_none")] path: Option<String>`. It is set
  only on those rows, so in-note rows stay byte-identical.
- **Human.** Print one line right after `header_line`, before `metadata_rows`:

  | Case             | Line                                                                  |
  | ---------------- | --------------------------------------------------------------------- |
  | Open v2          | `📖 Reading task  sase · ⭐ Next → [[sase#^ref-harness-engineering]]` |
  | Closed v2        | `📖 Read · 2026-10-12 · sase (archived)`                              |
  | v1               | `📖 Reading task  in this note (v1) · Ready`                          |
  | Rule 2 ambiguity | `📖 Reading task  ⚠ 2 open tasks claim this ref · run bob ref doctor` |
  - Lane labels by mark: ` ` → `Ready`, `*` → `⭐ Next`, `/` → `▸ In Progress`, `?` →
    `Blocked`; any other mark → `[m]`.
  - Closed lines: `x`/`X` → `Read`, `-` → `Dropped`. Omit the date segment when
    `closed_on` is unknown. Append ` (archived)` for `done/` residences.
  - Use the existing `Styler` (dim for the link) so piped output stays plain.
  - The `TASKS` section lists elsewhere-follow-ups with a dim ` · <path>` suffix.

- **Markdown.** `render_one_markdown` adds a `- **Reading task:** …` bullet with the
  same text.

### 8. `bob ref doctor` (`highlights_ref/doctor.rs`)

- **`library diagnostics`** skips every `REF_TASK_DIAGNOSTIC_CODES` code, the same way
  it already skips `marker_mirror_excluded`, so nothing is double-counted.
- **`ref tasks` row**, printed right after `library diagnostics`:
  - Counts, over all rows: _live_ is rows whose selected task is v2 and not archived;
    _archived_ is v2 archived; _open v1_ is rows carrying `open_v1_tracker`.
  - `ref tasks: ok (3 live · 1 archived · 29 open v1)` when no warn-level code exists.
    `open_v1_tracker` is count-only and expected until migration; it never warns.
  - Otherwise:
    `ref tasks: warn (3 live · 1 archived · 29 open v1 · 1 multiple_open_ref_tasks, 2 orphan_ref_task · e.g. <path>, <path>, <path>)`.
    Counts appear per code in code order. Up to 3 example paths: row paths for row
    codes, the containing file for orphans.
  - In the warn case, push the warning `N reading-task problems`.
- **`parents` row**, next:
  - `parents: ok (N parent notes · M aliases)`, from `scan_capture_targets`.
  - Or `parents: warn (K alias problems: <first>; <second>; <third>; …)`, plus the
    warning `K project_name_aliases problems`.
  - Collect alias problems structurally. Make `capture_targets` record the warnings from
    `push_alias_warnings` and `alias_conflict_warnings` in a separate
    `alias_warnings: Vec<ScanNote>`, also still pushed to `warnings`, rather than
    matching on message text.
- Warnings never fail the command. Document both rows in `docs/highlights-ref-sync.md`,
  where the library doctor rows are listed.

### 9. `capture-complete` `task_kind` and display text

- **`note_tasks::clean_description`** also drops a `#ref` token (ASCII case-insensitive)
  that directly follows a dropped global-filter token in the token sequence. This
  matches the landed `docs/task-tag-marks.md` "Reference reading tasks" rule and
  bob-plugins `REF_AFTER_TASK_RE`:
  - `#task #ref X` → `X`
  - `#task #references X` keeps `#references`
  - `#ref #task X` keeps `#ref`

  Rust adds no `📖` prefix; clients draw the symbol.

- **`NoteTask`** gains `task_kind: Option<&'static str>`, set to `Some("ref")` when the
  parsed body contains the global filter token immediately followed by a `#ref` token
  (the same predicate). Using the same predicate as the glyph and picker rule refines
  the epic's "starts with" wording.
- **Propagation.** Pass it through every intermediate task struct
  (`capture_active_tasks`, `capture_link_tasks`, `capture_dependency_tasks`
  `DependencyTask`, `capture_completable_tasks` `CompletableTask`) into all six
  candidate structs in `capture_complete/model.rs`: `TaskCandidate`,
  `ActiveTaskCandidate`, `TaskLinkCandidate`, `TaskParentCandidate`,
  `DependencyCandidate`, and `TaskCompleteCandidate`. Each gets
  `#[serde(skip_serializing_if = "Option::is_none")] task_kind`, so existing JSON is
  byte-identical.
- **Check other `clean_description` callers.** It also feeds `bob query` `text`,
  task-complete display text, and block-ID minting. Confirm no pinned vector contains
  `#task #ref`; the successor vectors in `capture_block_ids.rs` do not. Mention the
  `bob query` `text` effect in `docs/capture.md`.

## Implementation steps

1. Create `ref_tasks`: the v1 move, dates, walk, line parsing, selection, embed, and
   follow-ups. Add unit tests in `ref_tasks/tests.rs` using `tempfile` vaults.
2. Integrate `ref_library` (§4), then `list -P` (§6), `show` (§7), and doctor (§8).
3. Add capture `task_kind` and `clean_description` (§9).
4. Update docs:
   - `docs/ref.md`:
     - "Reading state": state comes from the located task, the v2/v1 selection order,
       and residence parent.
     - `bob ref list`: `-P` resolution.
     - `bob ref show`: the reading-task line and `tasks[].path`.
     - "JSON envelope": the `task` object, the `link`/`block_id` join, and the
       diagnostics table from §3/§4 with meanings.
   - `docs/capture.md`, `## bob capture-complete` per-candidate fields: `task_kind`.
   - `docs/highlights-ref-sync.md`: the doctor rows.
5. Tests (§Tests), then `just check`.
6. Measure performance, then close out the bead.

## Tests

Use `BOB_NOW="2026-10-06 12:00:00"`. There is no snapshot crate: assert exact JSON
values.

### Unit tests (`ref_tasks/tests.rs`)

Each case is its own test.

**Candidates and resolution**

- Path-qualified resolution, including a case-insensitive fallback.
- A bare unique stem resolves; a bare stem colliding between
  `ref/papers/harness_engineering.md` and `ref/blogs/harness_engineering.md` becomes an
  `ambiguous_stem` orphan.
- A missing path-qualified target is keyed for adoption **and** reported as an orphan.
- A `#ref` task with no link is an orphan.
- Path-qualified targets under the ref dir resolve.
- Blockquote `> - [*] #task #ref …` counts.
- Fenced code, frontmatter, embed lines, and the managed embed are ignored.
- A wrapper `- [-] #task Read [[ref/chat/x]]` without `#ref` is not a candidate.
- `#references` does not count; `#REF` does.
- A v1 line carrying `^ref` is never a candidate or orphan, including the vault-root
  `hub_ref.md` style.

**Selection and residence**

- Selection rules 1–5 and `open_ref_task_in_done`.
- `open_v1_tracker`, with and without a winning v2 task.
- Residence for root, `done/` (via `parent:`), and other paths, and lazy capture-target
  classification: `ref_task_outside_area` for a daily note (`2026/20261008.md`) and for
  a root non-candidate hub (`obsidian_ref.md`).

**Follow-ups and the walk**

- A full-path `[[ref/papers/x#^h-abc|🔖]]` follow-up in `sase.md` is attributed to
  `ref/papers/x.md`; a same-note `[[#^h-abc|🔖]]` is not.
- `lib/`, `xlib/`, hidden directories, `_generated/`, and conflict copies are never
  read.
- `find_managed_embed` matches positive and negative cases.
- Existing `ref_library` status unit tests still pass after the move.

### CLI tests (`tests/cli/ref_library/tasks.rs`)

Register the file in `tests/cli/ref_library/mod.rs`. Add a dedicated fixture vault,
`tests/fixtures/ref_tasks/vault/`. Do **not** add notes to
`tests/fixtures/ref_library/vault/`: many tests hard-code its counts.

Fixture contents:

- Root area notes `sase.md` and `mac_inbox.md` (`type: "[[area]]"`).
- Project `bob.md` (`type: "[[project]]"`, `status: wip`,
  `project_name_aliases: ["bob-cli"]`).
- Hub `obsidian_ref.md` (not a parent).
- `done/sase_done.md` with `parent: "[[sase]]"`.
- Daily note `2026/20261008.md`.
- Ref notes covering:
  - v1 open, and v1 closed;
  - v2 live in `sase.md` (path-qualified, `[*]`);
  - v2 live in `bob.md` (bare unique stem);
  - v2 archived closed in `done/sase_done.md` with `[completion:: 2026-10-12]`;
  - two open candidates for one ref;
  - the papers/blogs collision;
  - a ref note holding only a managed embed with `status: next` (no task);
  - a frontmatter parent mismatch;
  - a `#ref` line in the daily note;
  - an orphan;
  - a fenced `#task #ref` line;
  - a wrapper task;
  - a full-path follow-up.

Assert on that fixture:

- **`ref list -R all -A -f json`:**
  - each row's `task` object (v1 `block_id: "ref"`, v2 `link`, `archived`);
  - `parent` from residence (including the archived row reading `sase`);
  - `status`/`reading_state`;
  - `finished` from the archived line;
  - every diagnostic code.
- **`ref list -P bob-cli`** matches the `bob.md` row; `-P obsidian_ref` matches
  literally; `filters.parent` echoes as specified.
- **`ref show`:** human output for the open v2, closed archived, v1, and ambiguous
  cases. JSON `tasks[]` carries `path` on the follow-up.
- **`ref doctor`:** the `ref tasks` warn row with counts and examples, the `parents`
  row, and the `library diagnostics` row excluding ref-task codes. Also update
  `tests/cli/highlights/doctor_library.rs` for the two new rows.
- **`-g`** never backfills v2 rows.

### Other tests

- Existing `ref_library` tests: update `every_diagnostic_code_is_represented` in
  `src/native/ref_library/tests.rs` only if new codes now appear in the old fixture.
  Open v1 trackers there will now carry `open_v1_tracker`, so fix any
  exact-`diagnostics` assertions that this legitimately changes. Do not weaken unrelated
  assertions.
- `clean_description` unit cases (`#task #ref X`, `#task #REF`, `#task #references`,
  `#ref #task`).
- `NoteTask.task_kind`.
- One `capture-complete` JSON test per picker family (`^`, `:`/`+`, `&`, `!`) proving
  `task_kind: "ref"` appears on a `#task #ref` row and is absent elsewhere. Reuse the
  existing capture-complete harness and fixtures.

## Verification and closeout

1. **`just check`** passes (`cargo fmt --check`, clippy with all targets, and
   `cargo test --no-fail-fast`). Note a pre-existing unrelated failure on the bead and
   leave it alone.
2. **Performance.**
   - Build `cargo build --release`. Run
     `BOB_DIR=/home/bryan/bob target/release/bob ref list -R all -A -f json >/dev/null`
     three times under `time`. This is read-only against the live vault.
   - Compare with the same command from a release build of the base commit, or cite the
     52–60 ms baseline already on the bead.
   - The locator should add at most about 150 ms. If it adds more, profile the prefilter
     and read path before closing.
3. **Bead notes**, with `sase bead note bob-cli-5y.5 "<text>"`:
   - Before and after timings.
   - The final `ref_tasks` API for downstream phases:
     - the `RefTaskIndex::build(bob_dir, ref_dir)` signature;
     - `select`;
     - `LocatedRefTask` fields;
     - `find_managed_embed`;
     - `REF_TASK_DIAGNOSTIC_CODES`.

     Prefix the note with `INTERFACE CHANGE:` because `build` takes paths rather than a
     config.

   - One `PROPOSED FOLLOW-UP:` note per out-of-scope discovery. Create no beads.
   - Per the epic DECISIONS, these two notes:
     - `PROPOSED FOLLOW-UP: Add decisions strand ref-tasks-live-with-their-parent recording that ref tasks live with their parent note (skipped per epic memory_ref_parent_decision=no)`
     - `PROPOSED FOLLOW-UP: Update glossary strands reference-task, reference-note, and area-note for the ref-lives-with-parent model (skipped per epic memory_glossary_ref_terms=no)`

4. **Epic symbols.** Run `sase bead epic-symbols bob-cli-5y.5`; it must list nothing.
5. **Close.** Run
   `sase bead close bob-cli-5y.5 --note "<what you verified: just check, perf numbers, CLI/JSON contracts>"`.
   Never close the parent epic bob-cli-5y.
