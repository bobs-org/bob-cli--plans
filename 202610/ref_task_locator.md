---
tier: tale
title: The done-aware ref-task locator and read-side contracts
goal:
  bob ref list, show, find, and doctor derive status, parent, and dates from the located
  reading task, and capture-complete reports task_kind for
size: medium
proposed_by: bbugyi200.athena.bob-cli-5y.5
bead: bob-cli-5y.5
create_time: 2026-10-09 13:35:45
status: wip
---

- **PARENT:**
  [202610/ref_tasks_live_with_parent.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/ref_tasks_live_with_parent.md)
- **BEAD:**
  [bob-cli-5y.5](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-5y/bob-cli-5y.5.md)

# Plan: The done-aware ref-task locator and read-side contracts

Implement phase `ref-locator` of epic `bob-cli-5y` (bead `bob-cli-5y.5`) in bob-cli
only. The work is read-only: one vault walk builds `RefTaskIndex`, and `bob ref list`,
`show`, `find`, `doctor`, and `capture-complete` read it. Do not write the vault, do not
install, and do not edit memory.

Epic decisions are final. Do not cancel wrapper tasks (`cancel_dropped_wrapper_refs` is
no). Do not edit `decisions:ref-tasks-live-with-their-parent`,
`glossary:reference-task`, `glossary:reference-note`, or `glossary:area-note`. No other
memory note may be edited. Record the two skipped memory changes as
`PROPOSED FOLLOW-UP:` notes on `bob-cli-5y.5` before closing (wording in Done when).

Keep these names. Later phases depend on them. `RefTaskIndex::build` takes the config
struct below, not `LibraryConfig`, because `ref_library` must call into `ref_tasks` and
a `LibraryConfig` parameter would cycle.

- `src/native/ref_tasks/`
- `RefTaskIndex::build`
- `LocatedRefTask` fields `path`, `line_index`, `line`, `mark`, `block_id`, `archived`,
  `residence`
- `RefTaskDiagnostic`

There is no `task_status_hooks::NoteIndex`. Link resolution uses
`vault_links::NoteIndex`. Do not call `task_status_hooks::markdown_files`: that walk
skips `done/`.

## Outcome

`bob ref list`, `show`, and `find` derive status, parent, and the finished date from the
located reading task. Every row gains a `task` object (`null` when no task).
`schema_version` stays 1. `bob ref list -P` resolves aliases and still matches frozen
literal parents. `bob ref show` prints the reading-task line. `bob ref doctor` gains
`ref tasks` and `parents` rows and still exits 0 on warnings. Capture-complete task
candidates whose body starts with `#task #ref` gain `task_kind: "ref"`, omitted
otherwise, and picker text drops that `#ref`.

## Locator

Add `src/native/ref_tasks/mod.rs` and `mod ref_tasks;` in `src/native.rs`.

```rust
pub(crate) struct RefTaskConfig<'a> {
    pub bob_dir: &'a Path,
    pub ref_dir: &'a Path,
}

pub(crate) struct RefTaskIndex { /* selections keyed by ref-note path, plus orphans */ }

impl RefTaskIndex {
    pub(crate) fn build(config: &RefTaskConfig<'_>) -> Self
}

pub(crate) struct LocatedRefTask {
    pub path: String,            // vault-relative, forward slashes, with .md
    pub line_index: usize,       // 0-based
    pub line: String,            // trimmed raw line
    pub mark: char,
    pub block_id: Option<String>,
    pub archived: bool,          // file is under done/
    pub residence: Option<String>, // parent route, not the file path
}

pub(crate) struct RefTaskDiagnostic {
    pub code: String,
    pub detail: String,
    pub path: String,
}
```

`build` never returns an error for an unreadable file or a missing directory other than
a missing vault root. Skip a file that cannot be read. One index per `build_index` call.

### Walk

Walk every Markdown file under `bob_dir`, including `done/` and `ref/`. Skip a directory
whose name starts with `.` or is `_generated`, `_templates`, `_conflicts`, `lib`, or
`xlib`. Skip conflict copies the same way `ref_library::collect_member_files` does: the
file name contains ` (conflict`, ` (Conflicted copy`, or `.sync-conflict-`. Do not
follow symlinks. Do not skip `*.assets` or any other directory the list above does not
name.

Read each file once. Continue only when the lowercase text contains `#ref` or `^ref`.
Parse with `markdown::fenced_lines` and `markdown::strictly_closed_frontmatter_end`.
Skip frontmatter lines, fenced lines, and embed lines. An embed line, after the
blockquote strip below, trims to a token that starts with `![[`.

Strip blockquote prefixes with the same loop as Dataview (`0–3` spaces, `>`, optional
following space, repeated) before recognizing a task. A task line is then optional
indentation, a `-`, `*`, or `+` list marker, whitespace, a single-character checkbox,
and whitespace or end of line. Numbered lists are not tasks.

### Candidates

A line is a v2 candidate when it is a task line, a whitespace token equals `#ref`
case-insensitively (`#REF` counts, `#references` does not, `[[x#^ref]]` does not), and
its first wikilink resolves to a ref note. The first wikilink is the first `[[...]]` not
preceded by `!`. The target is the inner text before `|`. The note path is the target
before `#`.

Normalize a target by trimming, turning `\` into `/`, dropping one leading `/`, and
removing a trailing `.md` ignoring case. It is path-qualified when it contains `/`. The
ref prefix is `ref_dir` relative to `bob_dir` using forward slashes (normally `ref`).

- A path-qualified target under that prefix keys `{normalized}.md` even when the file
  does not exist yet. That is an adoption key, not an orphan.
- A path-qualified target outside the ref prefix does not resolve.
- A bare stem resolves only when `vault_links::NoteIndex`, built from Markdown files
  under `ref_dir`, returns exactly one note. Ambiguous and missing bare stems do not
  resolve.

A candidate whose file is the target ref note is not a v2 candidate. The in-note tracker
stays v1.

A v1 tracker is today's rule, only inside a file under `ref_dir`: `parse` the same loose
checkbox as `ref_library::status::parse_tracker_line` (unordered marker,
single-character mark, whitespace token exactly `^ref`). Keep that helper where it is
and call it, or duplicate its match exactly. Do not treat `^ref-slug` as v1.

A follow-up is a task line, not a v2 candidate match, that contains a wikilink whose
alias is exactly `🔖`, whose target resolves to a ref note under the same rules (a
missing path-qualified ref note still keys), and whose block id after `#^` starts with
`h-`. Store follow-ups on that ref note. They do not affect status or parent.

A `#task` line with no `#ref` token is neither a candidate nor an orphan. That is a
wrapper.

### Selection, per resolved ref-note key

Closed marks are `x`, `X`, and `-`. Every other mark is open.

1. The unique open v2 candidate outside `done/` is the live task.
2. Otherwise the newest closed v2 candidate sets the terminal state. Date is the first
   of `[completion:: D]`, `[cancelled:: D]`, `✅`/`✔` plus a token, or `❌`/`✖` plus a
   token, using the same token rules as `finished_date` in
   `src/native/ref_library/mod.rs`. Extract one `pub(crate)` helper and use it from
   both. Newest date wins. A missing date loses to any dated line. Equal dates prefer a
   file outside `done/`, then path, then line index.
3. Otherwise the v1 in-note tracker, for a note that exists.
4. Otherwise no task. Frontmatter status remains today's fallback.

Two or more open v2 candidates outside `done/` are `multiple_open_ref_tasks`. Do not
pick a winner. Status and parent fall back to frontmatter.

An open v2 candidate and an open v1 tracker together: the v2 candidate wins. Emit
`open_v1_tracker`. Do not emit `multiple_open_ref_tasks` unless there are also two or
more open v2 candidates outside `done/`. Emit `open_ref_task_in_done` as well when an
open line is inside `done/`.

`residence` for a selected v2 task:

- A vault-root file `{stem}.md` → route `stem`.
- A file `done/X.md` → the stem of that file's frontmatter `parent:` link, through
  `bare_parent_name`. Missing or empty parent → `None`.
- Anything else → `None`.

### Diagnostics

Attach note-scoped diagnostics to that ref note. Orphans have no note; keep them on the
index. Use these codes and details.

| Code                      | When                                                                                                                                                                                                    | Detail                                                                                        | `path`        |
| ------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------- | ------------- |
| `multiple_open_ref_tasks` | Two or more open v2 candidates outside `done/`                                                                                                                                                          | `{n} open reading tasks claim this ref; using frontmatter status`                             | one task file |
| `open_ref_task_in_done`   | Any open v2 candidate under `done/`                                                                                                                                                                     | `open reading task inside done/`                                                              | that file     |
| `open_ref_without_task`   | The note exists, its body has a managed embed, no v2 task and no v1 tracker were selected, and the decided status is `ready`, `next`, or `wip`                                                          | `open status on a v2 note with no reading task; restore it from git or set status: abandoned` | the ref note  |
| `orphan_ref_task`         | A `#ref` task line is not an in-note exact `^ref` v1 tracker, and its first wikilink does not resolve to a ref note (no link, a PDF link, a bare stem with zero or many matches, a path outside `ref/`) | `reading task link does not resolve to a ref note`                                            | the task file |
| `ref_task_outside_area`   | The selected task is the live open v2 task and its file is not `{route}.md` for any `parent_candidates` route                                                                                           | `live reading task is outside an area, project, or inbox note; refile it with Ctrl+Shift+M`   | the task file |
| `parent_mismatch`         | A v2 task is selected and the note's bare frontmatter parent differs from `residence`                                                                                                                   | `frontmatter parent {front} disagrees with residence {residence}`                             | the ref note  |
| `open_v1_tracker`         | An open in-note exact `^ref` exists                                                                                                                                                                     | `open in-note ^ref remains; run bob ref migrate-tasks`                                        | the ref note  |

Compare parent strings case-insensitively. Use `missing` when frontmatter parent or
residence is absent. A case-only difference is not a mismatch; the stored parent is
still the canonical residence route.

A path-qualified link to a missing note under `ref/` is not an orphan. An in-note v1
line that links to a PDF is not an orphan. A managed embed is one non-fenced line that,
after the blockquote strip and trim, is only `![[...#^...]]`.

`parent_candidates` is the capture-target set already built by
`parent_notes::parent_candidates` (root areas, non-terminal root projects, and the
inboxes). A root note that is not in that set is outside an area, project, or inbox.

## Read-side contracts

### Status, parent, finished, `task`

`ref_library::build_index` builds one `RefTaskIndex` from `config.bob_dir` and
`config.ref_dir` before the row loop, and passes each note's selection into `build_row`.
Store the index on `RefIndex` so doctor does not walk again. It is not serialized.

Change `decide_status` so the caller supplies the tracker hits. Today's
`find_trackers(body)` stays the default for notes with no locator selection. A selected
v2 task supplies one `TrackerHit` whose `mark` and `line` are the located line, and does
not scan the ref note, so an in-note `^ref` cannot become `multiple_ref_trackers`.
`multiple_open_ref_tasks` supplies no hits, so status is frontmatter even when the note
still holds a v1 tracker. v1 selection and "no task" keep today's in-note scan. Pending,
conflict, and `[?]` rules stay as they are.

Parent:

- Selected v2 task: `residence` (`None` when residence is none). Do not copy
  frontmatter.
- `multiple_open_ref_tasks`, v1, or no task: today's frontmatter parent.

Finished date: when the reading state is `finished` or `dropped` and the tracker hit is
the located line, keep using `finished_date`. For a v2 selection, do not let
`fill_git_dates` fill `added` or `finished`. v1 rows and rows with no task keep today's
`-g` git backfill. A `#[serde(skip)]` flag on `RefRow` is the signal. Do not infer it
from JSON.

Add this object to `RefRow`, always present, `null` when no task. Field order inside the
object is `path`, `block_id`, `link`, `mark`, `archived`. `mark` serializes as a
one-character string. `block_id` is `null` when missing and `"ref"` for a v1 tracker.
`link` is `[[{path without .md}#^{block_id}]]`, or `[[{path without .md}]]` when there
is no block id. Put `task` after `parent` in the row. `schema_version` stays 1 on list,
show, and find. Update the three `RefRow { ... }` literals in `output.rs`, `show.rs`,
and `coverage.rs`.

### `bob ref list -P`

Read `sase memory read cli_rules.md -r "Updating bob ref list -P help"` before editing
the option. Do not add an option, do not change `-P` / `--parent`, and do not reorder
help. `list_selection` currently copies the raw string and `matches_attributes` compares
it to `row.parent`. Pass `config.bob_dir` in. When `-P` is set, call
`parent_notes::resolve_parent`. On `Ok`, filter against `resolved.route`. On `Err`,
filter against the trimmed input and do not print the parent error or change the exit
code. `-P bob-cli` then matches parent `bob` when that alias resolves. `-P obsidian_ref`
still matches a frozen row whose stored parent is `obsidian_ref`.

Replace the help with:
`Only notes filed under this area or project. A resolvable name matches that note's route (stem or project_name_aliases; sase.md and [[sase]] count). Any other name is compared literally, so -P obsidian_ref still finds frozen rows.`

`tests/cli/help_options.rs` does not snapshot this sentence. No shell-completion change.

### `bob ref show`

When `task` is `Some`, insert this line after the header and before the metadata rows.
Glyphs are plain text so they survive `NO_COLOR`.

Open marks:

| Mark  | Fragment        |
| ----- | --------------- |
| space | `○ Ready`       |
| `*`   | `⭐ Next`       |
| `/`   | `▸ In Progress` |
| `?`   | `? Blocked`     |

Open line, parent present: `📖 Reading task  {parent} · {fragment} → {link}` (two spaces
after `task`). Parent missing: `📖 Reading task  — · {fragment} → {link}`.

Closed marks use `Read` for `x`/`X` and `Abandoned` for `-`. With a done date and a
parent: `📖 Read · 2026-10-12 · sase`. Append ` (archived)` when `archived` is true.
Omit the date segment when the date is missing, and omit the parent segment when the
parent is missing.

The Markdown digest gets one bullet with the same sentence. JSON already shows `task`
through the flattened row.

`tasks[]` keeps today's in-note `## Tasks` entries and does not add `path` on them
(`skip_serializing_if` absent). Append follow-ups whose file is not the ref note. Each
extra entry uses the same `checked`, `mark`, `text`, and `block_id` fields, plus `path`.
`checked` is true only for `x` and `X`. `text` is the body with the trailing block id
and inline fields removed and with a leading `#task` plus a directly following `#ref`
removed. `block_id` is `""` when the line has none. Human TASKS lines for those entries
gain a dim `  · {path}` suffix. Markdown task lines for them gain the path in
parentheses.

### `bob ref doctor`

In `append_library_doctor_rows`, after the library row, print `ref tasks` and `parents`.
Warnings append to the existing warning list. They never add a failure. `result: ok`
stays when nothing else fails.

Counts over existing ref notes only:

- live: selected task is the unique open v2 candidate outside `done/`
- archived: selected task has `archived == true`
- open v1: selected task is the in-note v1 tracker and its mark is open

No locator diagnostics: `ref tasks: ok (N live · M archived · K open v1)`.

Any note diagnostic or orphan:
`ref tasks: warn (N live · M archived · K open v1 · D diagnostics: {counts} · e.g. {paths})`.
Counts are `{n} {code}` in code order, joined by `, `, same shape as the
library-diagnostics row. At most three example paths, sorted by code then path. Also
push one warning `{D} ref-task diagnostics`.

Parents: take `capture_targets::scan_capture_targets` warnings whose message contains
`project_name_aliases`. Ignore ordinary non-routable-note warnings. None: `parents: ok`.
Some: `parents: warn (N alias problems · e.g. {up to 3 ScanNote display strings})`, and
push `{N} project_name_aliases problems`.

### Capture-complete `task_kind`

On `note_tasks::NoteTask`, set `task_kind: Option<&'static str>` while scanning. It is
`Some("ref")` when the first two whitespace tokens of the checkbox body are `#task` and
`#ref`, case-insensitively, and `None` otherwise. `#task #references`, `#ref #task`, and
`#task read #ref` do not match.

`clean_description` already drops the global-filter token. Also drop a `#ref` token that
is the immediate next token after a dropped global-filter token, case-insensitively. An
empty global filter drops nothing extra. `#task #ref #ref` with filter `#task` keeps the
second `#ref`.

Copy `task_kind` onto every capture-complete candidate built from a `NoteTask`:
`TaskCandidate` (task and link contexts), `ActiveTask` then `ActiveTaskCandidate`,
`LinkTask` then `TaskLinkCandidate` and `TaskParentCandidate`, the dependency-task
struct then `DependencyCandidate`, and the completable-task struct then
`TaskCompleteCandidate`. Serialize with `skip_serializing_if` so non-ref rows stay
byte-identical. `schema_version` stays 1. Do not add the field to section, route,
wikilink, or pomodoro-name candidates.

## Docs

`docs/ref.md`:

- In Reading state, say the effective status comes from the located task: the unique
  open v2 line outside `done/`, else the newest closed v2 line, else the in-note `^ref`,
  else frontmatter. v2 does not scan the ref note for a second tracker. `-g` git dates
  stay v1-only.
- Document the `task` object and `null`.
- Document the seven diagnostic codes and that they do not fail `doctor`.
- In `bob ref list`, replace the `-P` sentence with the alias-or-literal rule.
- In `bob ref show`, document the reading-task line and that `tasks[]` can include
  follow-ups from other files with `path`.
- In the JSON envelope field list, add `task`.

`docs/capture.md`, in the capture-complete JSON section: a task candidate whose body
starts with the `#task #ref` pair includes `task_kind` `"ref"`. The field is omitted
otherwise. The candidate `text` drops that `#ref` when it follows the global filter.

## Tests

Unit tests on `RefTaskIndex` for the selection order, the seven diagnostics, stem
collision, the missing path-qualified adoption key, a fenced line, an embed line, a
wrapper, and a `done/` parent stem.

One CLI fixture vault under `tests/cli/ref_library/` covering the phase list: v1 open
and closed trackers; v2 lines in a root area note and a root project note; an archived
v2 line in `done/x_done.md` with `parent:`; two open candidates for one ref; an orphan;
`ref/papers/harness.md` and `ref/blogs/harness.md` with a bare link and a path-qualified
link; a managed embed and a fenced copy; a `#ref` line in a daily note; a wrapper
without `#ref`. Snapshot `list` and `show` JSON for `task` and for a follow-up `path`.
Assert the human reading-task line for the Next example shape
`📖 Reading task  sase · ⭐ Next → [[sase#^ref-harness-engineering]]` and the archived
shape `📖 Read · 2026-10-12 · sase (archived)`.

`-P`: an area `bob.md` with `project_name_aliases: ["bob-cli"]` and a v2 row whose
parent is `bob` matches `-P bob-cli` and `-P bob`. A frozen row with parent
`obsidian_ref` matches `-P obsidian_ref` and does not match `-P bob-cli`.

Doctor: the ok line on a vault whose only tasks are closed v1 trackers with no alias
problems; a warn line that names `open_v1_tracker` and still prints `result: ok`; a
parents warn when two notes claim one alias.

`clean_description` and one `capture-complete` JSON test: a `#task #ref` line has
`task_kind` `"ref"` and text without those two tokens; a normal `#task` line omits
`task_kind`.

Existing v1 status tests must keep today's status, pending, conflict, and `[?]` results.
Update fixtures only where the new `task` field is asserted.

## Performance

Before editing, if `~/bob` exists, time `bob ref list -R all -A -f json` with
`BOB_DIR=~/bob` (read-only) and record the elapsed time. After the locator lands, time
the same command with the new binary. The walk should add at most about 150 ms. Record
both numbers, the host, and the binary in a bead note. If `~/bob` is absent, say so and
time a multi-file fixture instead of claiming an athena measurement. If the delta is far
above 150 ms, the prefilter is not doing its job: non-matching files must not be parsed,
and the walk must read each file once.

## Out of scope

Do not implement scan writes, block-id allocation, the open-book glyph, migration,
capture `URL @route`, or `gkeep` prompts. Do not bump `schema_version`. Do not close
`bob-cli-5y` or any ancestor. Do not create beads. Do not edit memory files.

## Done when

1. `just fix` if formatting is dirty, then `just check` passes. A failure that
   reproduces on the clean base tree is a `PROPOSED FOLLOW-UP:` citing any task bead
   that already tracks it, and this phase still closes.
2. Notes on `bob-cli-5y.5`:
   - `PROPOSED FOLLOW-UP: Add decisions strand ref-tasks-live-with-their-parent — epic decision memory_ref_parent_decision left it off; residence is the parent, #ref plus a path-qualified link is the identity, ^ref-<slug> is only an address, and only Ready refs keep REFERENCES.`
   - `PROPOSED FOLLOW-UP: Update glossary reference-task, reference-note, and area-note — epic decision memory_glossary_ref_terms left it off; the reading task is the #task #ref line in the parent note, the ref note embeds it and projects parent from residence, and area notes no longer except a reference note's own reference task.`
   - The before/after timing note.
   - Any other discovery as `PROPOSED FOLLOW-UP: <summary — detail>`.
3. `sase bead epic-symbols bob-cli-5y.5` reports no entries (that was true when this
   plan was written; re-run it). If a symbol appears, re-key that Justfile line to a
   still-open bead before closing.
4. `sase bead close bob-cli-5y.5 --note "<what you verified, including the timing numbers and that just check passed or which pre-existing failure you recorded>"`.
