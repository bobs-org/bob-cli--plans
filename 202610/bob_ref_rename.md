---
tier: epic
title: bob ref rename — rename a reference's files safely, plus a Bob Refs rename card
goal: "`bob ref rename <REF> <NAME>` renames the file stem of a modern, PDF-backed,
  unannotated reference (note, PDF, sidecar, audio) and rewrites every inbound link in
  one verified, scoped commit, while keeping its id, ^ref block ID, title, and PDF
  bytes. Bob Refs gets a ⌘⇧R / ⌘K → Rename… card that calls it. The URL namer stops
  minting opaque hex stems, and Obsidian's rename-current-file command refuses managed
  ref notes. The misnamed 442a6bf555914893e9891c11519de94f gist ends up as
  karpathy_llm_wiki.

  "
decisions:
  highlights_fallback:
    ask:
      If Highlights recreates the old PDF path after a rename, how should the card guard
      against it?
    choices:
      confirm: Show a warning naming the file and require a second Return to submit
      hold:
        Hide ⌘⇧R and the ⌘K item behind a default-off setting until Highlights is fixed
    default: confirm
    why:
      Renames are rare, Bryan knows if the PDF is open, and bob ref doctor flags
      duplicates
    answer: confirm
  rename_gist:
    ask:
      Should the acceptance phase rename the 442a6bf… gist to karpathy_llm_wiki in the
      live vault?
    default: true
    why:
      Bryan named this file as the motivating case, and the result is one revertable
      commit
    answer: true
  decision_record:
    ask:
      Should rename-write add a decisions strand saying a ref rename moves the stem but
      keeps id and ^ref?
    memory:
      - decisions
    default: true
    requested:
      I agree with all of the requirements recommended in that research file (but keep
      in mind that you are responsible for planning and splitting up this work
      appropriately).
    answer: true
phases:
  - id: namer
    title: Stop minting opaque URL stems
    depends_on: []
    size: small
    description:
      "namer: in bob-cli, make stem_from_url return None when the last non-skipped
      segment is opaque (hex ID or UUID) and strip trailing opaque words, sharing
      collection_name's is_hex_id_word; strip a trailing .md/.txt from titles before
      title-derived stems; unit tests and docs."
  - id: rename-links
    title: Vault link rewrite engine for a renamed bundle
    depends_on: []
    size: medium
    description:
      "rename-links: in bob-cli, a pure module that, given old→new vault paths for a
      reference bundle, finds and rewrites every inbound wikilink, embed, and Markdown
      link across vault .md files (done/ and daily notes included), honoring the
      bare-stem uniqueness rule and skipping code, and reports ^h- fragments and [h::]
      follow-ups that cite the note."
  - id: rename-plan
    title: bob ref rename command, refusals, and dry run
    depends_on:
      - rename-links
    size: medium
    description:
      "rename-plan: in bob-cli, register `bob ref rename` in the Library help group with
      help, completion, and docs; resolve REF exactly; compute the full move, link, and
      frontmatter plan; implement every scope, name, highlight-state, and precondition
      refusal; print -d plans as human or JSON (schema 1); live mode still refuses until
      rename-write."
  - id: rename-write
    title: bob ref rename live write flow
    depends_on:
      - rename-plan
    size: medium
    description:
      "rename-write: in bob-cli, take bob_sync.lock then the scan lock, pre-sync,
      re-plan, snapshot, move, rewrite, verify (no-op dry-run sync plus a clean index
      row), roll back on failure, commit exactly the touched paths, post-sync;
      --offline, already_renamed re-runs, docs runbook, and the decision record."
  - id: obsidian-guard
    title: Guard Obsidian rename-current-file for managed ref notes
    depends_on:
      - rename-write
    size: small
    description:
      "obsidian-guard: in bob-plugins bob-navigation-hotkeys, refuse rename-current-file
      (modal open and submit) when the active file is a managed ref note under ref/ with
      source_pdf frontmatter, with a Notice pointing at `bob ref rename`; first tests
      for the rename path; build, test, and bob plugins sync."
  - id: acceptance
    title: Highlights-open acceptance test and the gist rename
    depends_on:
      - rename-write
    size: small
    description:
      "acceptance: prepare a scratch-vault script that renames a PDF while it is open in
      Highlights, have Bryan run it on the Mac through /sase_questions, record the
      verified behavior in docs/ref.md, then (if decided) rename the 442a6bf… gist to
      karpathy_llm_wiki in the live vault."
  - id: mac-rename-core
    title: RefsCore rename contract, fetcher, stores, and card state
    depends_on:
      - rename-plan
    size: medium
    description:
      "mac-rename-core: in bob-mac-capture RefsCore, decode the ref rename report, add
      RefsFetching.rename on a non-cancelling refs-rename lane, map older-bob and error
      envelopes, add RefsOpenLogStore.rename(from:to:), pure rename-card state, key map,
      and presentation strings; fake-bob branch and portable tests."
  - id: mac-rename-card
    title: Bob Refs ⌘⇧R rename card
    depends_on:
      - mac-rename-core
      - rename-write
      - acceptance
    size: medium
    description:
      "mac-rename-card: in bob-mac-capture, route ⌘⇧R and a ⌘K Rename… item to a rename
      card with an editable stem field, pass text-editing keys through, run the rename
      without cancelling on hide, select to.path and migrate history on success, render
      errors inline, apply the acceptance outcome, README, tests."
proposed_by: bbugyi200.apollo.research.1k.linker.w0
decided_by: auto
create_time: 2026-10-15 15:25:58
status: wip
---

- **PROMPT:**
  [prompts/202610/bob_ref_rename.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202610/bob_ref_rename.md)

# Plan: `bob ref rename` and a Bob Refs rename card

## Background

Bryan wants to rename a misnamed reference —
`lib/blogs/442a6bf555914893e9891c11519de94f.pdf` and its note
`ref/blogs/442a6bf555914893e9891c11519de94f.md` — from a new `bob ref rename` command
and from the Bob Refs panel in Bob Mac Capture. The research report
`research:202610/bob_ref_rename_command/bob_ref_rename_command.md` (read it with
`sase artifact read research:202610/bob_ref_rename_command/bob_ref_rename_command.md "<why>"`)
merged four independent reports, reproduced every naive route in scratch vaults, and
recommended this epic. **Bryan accepted every requirement in it (R1–R11).** This plan
turns that recommendation into phases. Where the plan settles something the report left
open, it says so explicitly ("Plan choice").

### Why a plain rename is unsafe

A reference's identity is spread across paths, hashes, and links:

- Scan pairs PDF and note purely by path: `lib/<t>/<s>.pdf` ↔ `ref/<t>/<s>.md`
  (`src/native/highlights_ref/io.rs`, `ref_note_path`, `pdf_path_metadata`). Moving one
  side makes the next scan create a fresh note and a second reading task.
- Highlight block IDs (`^h-…`) hash the vault-relative PDF path
  (`sidecar_render.rs:115-148` via `source_pdf_value`), and follow-up `[h:: …]` digests
  hash the note path (`annotation_tasks.rs:227-242`). Renaming an annotated reference
  re-keys every highlight (research experiment E: 8 minted, 8 tombstoned).
- The reading task in `sase.md` links the note path:
  `- [ ] #task #ref [[ref/blogs/442a6bf…|karpathy/llm-wiki.md]] [created::2026-10-15] ^ref-442a6bf…`.
- Bundle members are found by sitting beside the PDF: sidecar `<stem>.md` or
  `<stem>.textbundle/` (`sidecar.rs:88-116`), audio `<stem>.{mp3,m4a,ogg,opus}`
  (`audio.rs:5-41`), and `ref/<t>/<stem>.assets/` (`sidecar.rs:595`).
- Markdown-rendered PDFs with narration bake an
  `obsidian://open?…&file=lib%2F<t>%2F<stem>.mp3` listen-card link into the PDF
  (`create.rs:2159-2203`).

**Research experiment A is the design's foundation.** Moving the note and the PDF,
rewriting the `sase.md` link, and updating `source_pdf` — while keeping `id` and
`^ref-…` — makes `bob ref sync -d <new pdf>` plan no work at all (`note_action: none`,
`reading_task: unchanged`, `pdf_marker_action: none`), and `bob ref find <old id>` still
resolves. **No sync-engine change is needed.**

Experiment D shows the most likely way this damage happens today: the vault's own
Obsidian rename command (`bob-navigation-hotkeys` `rename-current-file`) renames only
the note, and the next scan recreates the old note and inserts a duplicate `^ref-…-2`
reading task.

### The motivating reference

- Note `ref/blogs/442a6bf555914893e9891c11519de94f.md`; PDF
  `lib/blogs/442a6bf555914893e9891c11519de94f.pdf` (git-tracked). No sidecar, audio, or
  `.assets/`. 0 annotations, 0 follow-ups.
- Title `karpathy/llm-wiki.md`; `source_url`
  `https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f`; frontmatter `id`
  `442a6bf555914893e9891c11519de94f`.
- Inbound links: only the `sase.md` reading task and the note's own `![[sase#^ref-…]]`
  embed.
- **The hash is the gist's real ID.** A rename must never replace every occurrence of
  the hash: that would corrupt `source_url`, the URL dedupe key.
- It happened because `stem_from_url` (`src/native/highlights_ref/clip_url.rs:250-260`)
  walks path segments from the end and `stem_from_segment` accepts a 32-character hex
  segment. About 1 in 425 library PDFs has an opaque stem.

### Invariants every phase keeps

1. **"Rename" means the file stem only.** The stem is the PDF basename, the note
   basename, and their companions. The frontmatter `id`, the `^ref-<slug>` block ID,
   `title`, `source_url`, dates, the PDF marker, and the PDF bytes never change. Today,
   Task Links, and Depends-On all join on `(task path, block_id)`, so they keep working.
2. **v1 renames only modern, PDF-backed, unannotated references, in place** (same
   `ref_type` directory). Annotated renames, folder refiling, retitling, bulk renames,
   and intake (`xlib/`) renames are later work.
3. **Bob Mac Capture is a thin client** (`decisions:mac-capture-is-a-thin-client`): the
   app never moves files, computes stems, or infers the result; it runs bob and renders
   the JSON bob returns, using bob's explicit `from.path` → `to.path` mapping.
4. **CLI rules** (`sase/memory/cli_rules.md`): excellent `-h`, alphabetical options, a
   short alias for every public long option, colored human output.

## The `bob ref rename` contract

Phases `rename-plan` and `rename-write` implement this contract, and the Mac phases
decode it. Treat it as the source of truth; change it only additively.

### Command surface

```text
bob ref rename [OPTIONS] <REF> <NAME>

Rename a reference's files and update every link to it

Arguments:
  <REF>   Reference: note path, stem, id, URL, arXiv ID, DOI, or exact title
  <NAME>  New file stem; a trailing .pdf is ignored ([A-Za-z0-9][A-Za-z0-9_.-]*)

Options:
  -b, --bob-dir <PATH>   Bob vault root; defaults to BOB_DIR or ~/bob
  -d, --dry-run          Print the plan without moving, rewriting, committing, or syncing
  -f, --format <FORMAT>  Output format [default: human] [possible values: human, json]
  -l, --lib-dir <PATH>   Highlights PDF library; defaults to BOB_HIGHLIGHTS_LIB_DIR or lib
  -o, --offline          Commit locally; skip the vault sync before and after
  -r, --ref-dir <PATH>   Reference note directory; defaults to BOB_HIGHLIGHTS_REF_DIR or ref
  -x, --xlib-dir <PATH>  Highlights intake directory (checked for name collisions)
  -h, --help             Print help
```

The after-help explains that the frontmatter id and the reading task's `^ref` block ID
stay, so Today, Task Links, dependencies, and `bob ref find <old id>` keep working; that
v1 refuses annotated, legacy, and intake references; and that links outside the vault
(browser bookmarks, copied `obsidian://` or `highlights://` URIs) cannot be updated. Its
examples are `bob ref rename 442a6bf555914893e9891c11519de94f karpathy_llm_wiki -d` and
`bob ref rename ref/chat/README.md research_prototype_cdx`. Reuse the shared
`bob_dir_arg`/`lib_dir_arg`/`ref_dir_arg`/`xlib_dir_arg` builders in
`src/native/highlights_ref/cli.rs`. Live mode is the default and `-d` previews (like
`create`, `sync`, and `task reroll`; unlike the dry-run-by-default `migrate-*` verbs).

### Resolution and scope

- Resolve REF with `ref show`'s exact resolver (`ref_library::show::resolve_show`, which
  uses `resolve.rs`). It accepts a note path (`.md` optional), stem, frontmatter id,
  URL, arXiv ID, DOI, or a unique exact title. Fuzzy matches are never mutated. Map
  `ShowFailure::NotFound` to `unknown_reference` and `Ambiguous` to
  `ambiguous_reference`, carrying the candidates in the hint. The app always passes the
  exact note path after `--`.
- `legacy_reference`: the row's `era` is `legacy`, or the note has no `source_pdf`, or
  the note sits under a legacy directory (`ref/ai`, `ref/zorg`). Hint: legacy notes have
  no PDF pairing, so Obsidian's rename is safe for them.
- `intake_reference`: REF is a path under the intake directory (`xlib/`). Hint: intake
  PDFs have no note yet; rename the file before `bob ref scan`.

### Names (R6)

- Validate NAME with `clip_url::validate_name` (strips one trailing `.pdf`; requires
  `[A-Za-z0-9][A-Za-z0-9_.-]*`). On failure, refuse `invalid_name`, and put a
  `snake_case` suggestion in the hint.
- Same stem as now → success with `changed: false`, `outcome: "unchanged"`.
- A case-only change (equal ignoring ASCII case, but different) → `invalid_name`; APFS
  is case-insensitive.
- `name_taken`, compared case-insensitively, when any of these exist:
  - any planned bundle destination: `ref/<t>/<NAME>.md`, `lib/<t>/<NAME>.pdf`,
    `lib/<t>/<NAME>.md`, `lib/<t>/<NAME>.textbundle`,
    `lib/<t>/<NAME>.{mp3,m4a,ogg,opus}`, `ref/<t>/<NAME>.assets`;
  - `xlib/<t>/<NAME>.pdf`;
  - **any `.md` file anywhere in the vault named `<NAME>.md`**, other than the bundle's
    own members (the `migrate-zorg` rule, `migrate_zorg/plan.rs:409-454`, which skips
    hidden directories). This protects bare `[[NAME]]` links.
- Never auto-suffix.

### Highlight state (R3, R4)

- `has_highlights` when any of these hold:
  - the note's managed region (`region.rs::parse_managed_region`) has a `^h-` block,
    live or tombstoned under "Removed highlights";
  - `ref/<t>/<stem>.assets/` exists;
  - a vault line holds an `[h:: …]` property and links this note;
  - any inbound link to this note has a `#^h-…` fragment.

  A sidecar alone is not a refusal; it moves with the bundle.

- `listen_card_link` when the PDF has a Link annotation whose URI starts with
  `obsidian://` and whose percent-decoded `file` parameter names one of this bundle's
  audio paths. Build the reader on `marker::annotation_ids_for_page`'s `/Annots` walk
  (`marker.rs:111`). Moving the audio and rewriting `audio:` and the player embed is
  fine; rewriting the PDF is not.

### Preconditions (R5)

- `not_in_sync`: a dry-run sync of the current PDF plans work
  (`stable_note_action != "none"`, `marker_write_needed`, or a reading-task plan other
  than unchanged/adopt), or `source_pdf` is not exactly `lib/<t>/<stem>.pdf` for the
  note's own `<t>/<stem>`. Hint: run `bob ref scan`. Programmatic dry-run sync,
  mirroring `sync_pdf` (`sync.rs:6-30`): `ScanContext::build`,
  `plan_pdf_sync(…, SyncOptions{dry_run: true, write_pdf: false, prefer: None}, …)`,
  `finalize_annotation_task_plans`, `reserve_preview_ids`, then read
  `stable_note_action`, `marker_write_needed`, and `reading_task_plan`
  (`report::reading_plan_report_lines` names the reading-task outcome).
- `dirty_paths`: any touched path (every move source, plus every file the plan rewrites)
  differs from `HEAD` or is untracked and not ignored. **Plan choice:** before the
  pre-sync (that is, in `-d` mode, and in live mode before vault-sync runs) dirty paths
  are a **warning**, because vault-sync commits the vault's dirty files first; with
  `-o`, and in live mode after the pre-sync, they are a refusal. Parse
  `git status --porcelain=v1 -z` so rename entries are handled, unlike
  `migrate_tasks/write.rs:456-488`.
- Ignored paths (for example an untracked audio format) move on disk but are never
  passed to `git add`.

### The plan

Compute the whole plan before writing anything:

1. **Moves** (no-clobber renames), for whichever exist: the note, the PDF, the sidecar
   (`<stem>.md` and/or `<stem>.textbundle/` — inventory both so a stale export is not
   left behind), and the audio (all four extensions). There is no `.assets/` in v1.
   Build paths by joining the stem and extension strings; never use
   `Path::with_extension`, which clobbers a dotted stem's final component.
2. **Inbound link rewrites** across vault `.md` files (phase `rename-links`).
3. **Own frontmatter**: update `source_pdf`, `highlights_sidecar`, and `audio`, plus the
   audio player embed line if present. Edit those lines in place; never re-render the
   note from a template. Leave `id`, `title`, the marker hash/base fields, the `^ref`
   embed, and all other text untouched.

### Write flow (live mode)

1. Take `bob_sync.lock` with `ob::acquire_lock_waiting` (60 s; `lock_busy` after), then
   the scan writer lock. **Plan choice:** wait at most 30 s for the scan lock
   (`scan_busy` after) — shorter than scan's 120 s default — so bob's own worst case
   stays well inside the app's timeout. Scan never takes `bob_sync.lock`, and the
   configured pre-scan hook is `bob_xlib_pull`, not vault-sync, so this order inverts
   nothing. Never run the pre-scan hook or a scan as a side effect.
2. Require a git worktree (`ob::detect_git_worktree`; `not_a_worktree`). Pre-sync with
   `vault_sync::run_cycle_with_existing_lock_report` unless `-o`; on failure refuse
   `sync_failed` with the hint "re-run with --offline".
3. Re-plan from disk under the locks and re-run every refusal.
4. Snapshot before-images of every file to be rewritten
   (`migrate_tasks::write::snapshot_before_images` / `restore_before_images` are
   `pub(crate)`). Move the files, then rewrite links and frontmatter with atomic writes
   (`highlights_ref::io::atomic_write`).
5. **Verify; on any failure undo every move and restore every before-image, then exit
   `verify_failed`.**
   - Idempotence: the dry-run sync of the new PDF plans `note_action: none`, no marker
     write, and an unchanged reading task.
   - Index: a rebuilt index has a row at the new note path with the same `marker_id`, a
     located task with the same `(path, block_id)`, and no `open_ref_without_task` or
     `duplicate_identity` diagnostic, with no new `orphan_ref_task`; no row remains at
     the old path.
6. Commit exactly the touched paths, old and new (`ob::commit_paths`; git records the
   PDF as a rename), with subject `bob ref rename: <old> → <new>`. Then post-sync unless
   `-o`. **Plan choice:** a post-sync failure after a successful commit is still
   `ok: true` (exit 0) with `sync.post: "failed"` and a warning ("the rename is
   committed locally; run bob vault-sync"), because the rename is durable and callers
   need `to.path`.
7. **Re-runs:** when REF names a note path or stem that no longer exists,
   `ref/<t>/<NAME>.md` exists, and the latest commit touching it has the subject
   `bob ref rename: <REF stem> → <NAME>`, report success with `changed: false`,
   `outcome: "already_renamed"`. When both the old and the new paths exist, refuse
   `name_taken`. **Plan choice:** `already_renamed` is a success outcome rather than an
   error code, so the app still gets `to.path`.

### JSON output (`schema_version: 1`)

Success, on stdout, exit 0:

```json
{
  "ok": true,
  "schema_version": 1,
  "command": "ref rename",
  "generated_at": "…",
  "mode": "write",
  "changed": true,
  "outcome": "renamed",
  "from": {
    "path": "ref/blogs/442a6bf….md",
    "pdf": "lib/blogs/442a6bf….pdf",
    "stem": "442a6bf…"
  },
  "to": {
    "path": "ref/blogs/karpathy_llm_wiki.md",
    "pdf": "lib/blogs/karpathy_llm_wiki.pdf",
    "stem": "karpathy_llm_wiki"
  },
  "moves": [
    { "kind": "note", "from": "…", "to": "…" },
    { "kind": "pdf", "from": "…", "to": "…" }
  ],
  "links": { "rewritten": 1, "files": ["sase.md"], "skipped_ambiguous": [] },
  "frontmatter": ["source_pdf"],
  "kept": {
    "marker_id": "442a6bf…",
    "task": { "path": "sase.md", "block_id": "ref-442a6bf…" }
  },
  "ref": { "…": "the index row at the new path, serialized like ref show's row" },
  "commit": { "sha": "…", "subject": "bob ref rename: 442a6bf… → karpathy_llm_wiki" },
  "sync": { "pre": "ok", "post": "ok" },
  "warnings": []
}
```

- `mode` is `"write"` or `"dry_run"`. `outcome` is `"renamed"`, `"unchanged"`, or
  `"already_renamed"` (dry runs report `"renamed"` with `changed: true` for what they
  would do). Move `kind` is `note`, `pdf`, `sidecar`, or `audio`.
- In `-d` mode: `commit: null`, `sync: {"pre":"skipped","post":"skipped"}`, and `ref` is
  the current row. With `-o`, both sync values are `"skipped"`.
- `sync` values are `"ok"`, `"skipped"`, or `"failed"`.

Errors, on stdout with `-f json` (human mode prints `bob ref rename: error: …` and a
`hint:` line on stderr), exit 1:
`{"ok":false,"schema_version":1,"command":"ref rename","error":{"code":"…","message":"…","hint":"…"}}`.
Clap usage errors exit 2. Codes:

| Category        | Codes                                          |
| --------------- | ---------------------------------------------- |
| Resolution      | `unknown_reference`, `ambiguous_reference`     |
| Scope           | `legacy_reference`, `intake_reference`         |
| Name            | `invalid_name`, `name_taken`                   |
| Highlight state | `has_highlights`, `listen_card_link`           |
| Preconditions   | `not_in_sync`, `dirty_paths`, `not_a_worktree` |
| Locks and sync  | `lock_busy`, `scan_busy`, `sync_failed`        |
| Outcome         | `verify_failed`                                |

Human output is colored and scannable: a header (`old → new`, with `(dry run)` when
`-d`), one line per move, the link count with the files, the kept id and task, then the
commit and sync status.

## Phase details

### Phase `namer` — stop minting opaque URL stems (bob-cli)

Change `stem_from_url`, **not** `stem_from_segment`: the collection namer imports
`stem_from_segment` (`collection_name.rs:10-13`), and making `stem_from_segment` reject
hex would make the reverse walk fall back to the owner segment (`karpathy`).

1. Move `is_hex_id_word` (`collection_name.rs:552-558`: at least 6 characters, at least
   one digit, lowercase hex) into `clip_url.rs` as a shared `pub(super)` helper, and
   import it back into `collection_name.rs` unchanged. Add a UUID check (8-4-4-4-12 hex
   groups, case-insensitive).
2. In `stem_from_url`: find the last segment that `stem_from_segment` does not skip. If
   that raw segment (after page-extension stripping) is a UUID, or every one of its
   words is an opaque hex word, return `None` — do not keep walking to earlier segments.
   Otherwise drop trailing opaque words from its stem (`my-post-title-1a2b3c4d5e6f` →
   `my_post_title`).
3. Title-derived stems (the `snake_case(title)` fallbacks in `clip.rs:351-363` and
   `ingest.rs:1040-1041`) go through one new helper that first strips a trailing `.md`,
   `.markdown`, or `.txt` (case-insensitive) from the title, so `karpathy/llm-wiki.md`
   yields `karpathy_llm_wiki`. The title itself is unchanged.
4. Unit tests (extend `stem_derivation_skips_noise_segments` in `clip_url.rs`):
   - `https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f` → `None`;
   - the Medium-style slug above → `my_post_title`;
   - a UUID last segment → `None`;
   - all existing expectations unchanged, and every collection-namer test unchanged;
   - the title helper on `karpathy/llm-wiki.md` → `karpathy_llm_wiki`. Where a
     `clip`/`ingest` test seam exists, assert that an explicit `-N` still wins and that
     a missing title still falls back to `<host>_<YYYYMMDD>`.
5. Update the stem rules in `docs/highlights-create.md` (the target table) and
   `docs/highlights-clip.md` if they describe URL slugs.

Existing stems never change; dedupe stays URL-based. The gist's bad title and its author
`262588213843476` come from host metadata — a separate bug: record it as a
`PROPOSED FOLLOW-UP:` note on this phase bead.

### Phase `rename-links` — vault link rewrite engine (bob-cli)

A pure, unit-tested module (suggested home `src/native/highlights_ref/rename/links.rs`,
inside the new `rename` module so later phases can use `highlights_ref` internals). The
existing `migrate_tasks/rewrite.rs` is hard-coded to `#^ref` block links and cannot be
reused as-is; build on these instead:

- `task_dependencies::raw_wikilink_spans` (`task_dependencies/mod.rs:70`, skips inline
  code) and `parse_block_link_inside` (`:102`);
- `vault_links::NoteIndex` / `markdown_basename` / `target_to_markdown_path`
  (`vault_links.rs`);
- `ref_tasks::is_conflict_copy_name` (`ref_tasks/walk.rs:43`).

**Input:** the vault root and a bundle map of old → new vault-relative paths (note, PDF,
sidecar `.md` or `.textbundle` text file, audio files). **Output:** a deterministic list
of per-file edits, each with its line numbers and before/after text; a list of skipped
ambiguous bare links; and highlight-state findings. The same engine must be able to
produce the plan (dry run) and the rewritten file contents (write).

Rules:

1. **Files:** every `.md` under the vault except hidden directories, `_generated/`,
   `_templates/`, `_conflicts/`, `lib/`, `xlib/`, and conflict-copy names (the
   `migrate_tasks/rewrite.rs::is_excluded` rule). **`done/` and daily notes are
   included.** Scan whole files, including frontmatter values (for example
   `parent: "[[ref/blogs/old]]"`), but skip fenced code blocks and inline code spans.
2. **Forms to rewrite** — rewrite the target, never the alias, fragment, or embed
   marker:
   - `[[ref/<t>/<old>]]` and `[[ref/<t>/<old>.md]]`, with `|alias`, `#heading`,
     `#^block`, and `!` embed variants;
   - a bare `[[<old>…]]` only when it resolves to this note: no `.md` outside the bundle
     has basename `<old>` (case-insensitive). Otherwise leave it and report it in
     `skipped_ambiguous`;
   - `[[lib/<t>/<old>.pdf]]` and `![[…]]` (the v1 `^ref` tracker form);
   - links to the bundle's audio files and sidecar text file;
   - Markdown links `](ref/<t>/<old>.md)` and `](lib/<t>/<old>.pdf)`, comparing
     percent-decoded targets and keeping the original encoding style in the output.
     Preserve the link's original spelling style: keep an explicit `.md` if it had one,
     and leave a bare link bare when the new stem stays unique.
3. **Never** do a string replace of the old stem: the hash also appears in `source_url`,
   `id`, and `^ref-…`.
4. **Highlight findings:** report every inbound link with a `#^h-` fragment, and every
   line that has an `[h:: …]` property and links this note, so `rename-plan` can refuse
   `has_highlights`.
5. **Unit tests:** every form above; aliases and fragments preserved; code spans and
   fences untouched; frontmatter links; `done/` and daily-note files; an ambiguous bare
   stem; dotted stems; a hash-named stem whose hash also appears in a URL on the same
   line; percent-encoded Markdown links; `^h-` and `[h::]` findings; idempotence (a
   second pass over rewritten text finds nothing).

### Phase `rename-plan` — command, refusals, dry run (bob-cli)

1. **Module:** `src/native/highlights_ref/rename/` (`mod.rs`, `cli.rs`, `plan.rs`,
   `report.rs`, plus `links.rs` from the previous phase). It lives in `highlights_ref`
   because it needs that module's private sync, sidecar, audio, region, marker, and
   scan-lock internals. It uses the `pub(crate)` `ref_library` APIs (`build_index`,
   `resolve_show`, `output::{Format, generated_at, print_find_error}`, the `ref show`
   row serializer). Add `pub(crate)` re-exports only where needed.
2. **Registration:**
   - Clap command per the contract, registered in `all_subcommands` and the
     `HELP_GROUPS` "Library" list, alphabetically between `migrate-zorg` and `show`
     (`highlights_ref/cli.rs`). Dispatch it in `highlights_ref/mod.rs`
     (`matches.subcommand()`).
   - Update the about lines: `bob ref` (`highlights_ref/cli.rs:62`) becomes "Find, list,
     read, and rename Bob reference notes, and sync Highlights PDFs into them"; the root
     `ref` entry (`src/runner.rs:407`) and `README.md:274` become "Find, list, read, and
     rename references; sync Highlights PDFs into them". Refresh the help fixtures
     (`tests/fixtures/help/ref-short.txt`, `root-short.txt`, `root-long.txt`).
   - Add a completion entry (`src/native/completion/kinds.rs`): `ref rename` arg `ref`
     is `Kind::VaultNote`.
   - Add a `bob ref rename -d` example to `REF_AFTER_HELP`.
3. **Plan computation and refusals:** everything in the contract's Resolution, Names,
   Highlight state, Preconditions, and Plan sections, in that order, so the cheapest
   refusal wins. `-d` prints the plan (human or JSON) and exits 0; it takes no locks and
   writes nothing.
4. **Live mode in this phase** refuses with code `not_implemented` and the hint "preview
   with -d; the write flow lands in a follow-up phase". Phase `rename-write` removes
   that code.
5. **Docs** (`docs/ref.md`): an intro mention; a "Choose a command" row ("Fix a
   reference's file name | `bob ref rename <REF> <NAME> -d`, then without `-d` | Move
   the note, PDF, and companions and update links; keeps the id and reading task"); and
   a `## bob ref rename` section covering scope, what is kept, refusal codes, the JSON
   envelope, and the gist example.
6. **CLI integration tests** (`tests/cli/ref_library/rename.rs` or
   `tests/cli/highlights_ref/…`, using the temp-git-vault helpers from
   `tests/cli/ref_library/migrate_zorg.rs` and the PDF helpers in
   `tests/cli/support.rs`):
   - a gist-shaped fixture: the `-d` JSON plan (moves, one `sase.md` link, `source_pdf`
     edit, kept id and task), as a snapshot;
   - a bundle with a sidecar and audio;
   - bare-stem links, including the ambiguous case;
   - links in `done/` and a daily note;
   - a dotted stem;
   - every refusal code reachable without locks, each with its JSON envelope;
   - `-d` writes nothing (the tree is byte-identical);
   - `bob ref rename --help` lists options alphabetically and every long option has a
     short alias (mirror `migrate_tasks.rs:561-580`).

### Phase `rename-write` — live write flow (bob-cli)

1. Replace `not_implemented` with the contract's Write flow: locks, worktree, pre-sync,
   re-plan, before-images, moves, rewrites, verify, rollback, commit, post-sync, and
   `already_renamed` re-runs. Model it on `migrate_zorg/write.rs` and
   `migrate_tasks/write.rs` (lock budget, wait message on stderr in human mode,
   `ob::child_env`, `--offline`).
2. Make the verifier injectable inside the module (a function parameter or trait) so a
   unit test can force a verify failure and assert full rollback: moves undone,
   before-images restored, no commit. Do not add a fault-injection environment variable
   to release builds.
3. **Integration tests** (temp git vaults, `--offline`):
   - experiment A as a test: after the gist-shaped rename, `bob ref sync -d <new pdf>`
     plans no work, `bob ref find <old id>` resolves to the new path, and the `sase.md`
     task is located with the same block ID;
   - sidecar and audio moves, with the commit recording the PDF as a rename;
   - the commit's exact path set and subject (`git show --name-status`), with unrelated
     dirty files left uncommitted;
   - dirty touched paths refused with `-o`;
   - `already_renamed` and `unchanged` outcomes;
   - `not_in_sync` after a stale `source_pdf`;
   - JSON snapshots for write mode.
4. **Docs:** finish `## bob ref rename` in `docs/ref.md` with the numbered write flow,
   the commit fields, the lock order, the post-sync-failure behavior, and a rollback
   runbook (`git revert <sha>`, then `bob vault-sync`).

   > [!decision] decision_record Add the decision strand (step 5 below).

5. **Memory** (decision `decision_record`): use `/sase_memory_write` to add the strand
   `sase/memory/decisions/ref-rename-keeps-identity.md`, then run `sase memory init`.
   - Applies to: bob-cli, bob-mac-capture, bob-plugins.
   - Claim: `bob ref rename` changes only the file stem (note, PDF, sidecar, audio, and
     links to them); it keeps the frontmatter `id`, `^ref` block ID, title,
     `source_url`, marker, and PDF bytes; v1 refuses annotated references; a future
     annotated rename uses a pipeline-owned identity-origin frontmatter field rather
     than re-keying `^h-` IDs.
   - Why and rejected alternatives: changing `id`/`^ref` needs a PDF marker write and
     churns Today, Task Links, and Depends-On joins; re-keying highlights must rewrite
     anchors, assets, inbound fragments, and archived `[h::]` digests; content matching
     puts fuzzy matching into every scan.
   - Cost: `id` and stem can differ (6 notes already do).
   - Reopens when: an annotated rename is actually needed.
   - Evidence: the research ref above and this phase's commit.

### Phase `obsidian-guard` — guard `rename-current-file` (bob-plugins)

Open the linked repo with `/sase_repo` (`bob-plugins`) and read its `AGENTS.md`.

1. Add a pure helper, for example `isManagedRefNote(path, frontmatter)`: the path is
   under `ref/`, ends in `.md`, and the frontmatter has a non-empty `source_pdf`. Read
   frontmatter defensively (`metadataCache.getFileCache(file)?.frontmatter`, as in
   `190-summary-modals.js:476-482`). Export it through `src/700-exports.js` like the
   other rename helpers.
2. Guard both `openRenameCurrentFileModal` (`src/670-plugin-project-files.js:465`), so
   the modal never opens, and `renameCurrentFileToName` (`:474`), as defense in depth.
   Each returns without renaming and shows a `Notice` such as "Managed reference note:
   rename it with `bob ref rename ref/blogs/x.md <new-name>` so its PDF, reading task,
   and links move together." Fragment `670` is at 943 of 1000 lines: if the guard does
   not fit, put the helper in a smaller fragment.
3. Tests (the first for this command): a new
   `scripts/test-navigation-hotkeys-rename.cjs` on `navigation-hotkeys-harness.cjs`,
   added to the root `package.json` test script:
   - a managed ref note is refused, with no `renameFile` call and the notice text;
   - a legacy ref note (no `source_pdf`) and a non-ref note still rename;
   - the helper's edge cases.
4. `npm run build`, `npm test`, then `bob plugins sync`. Add a short clause to the
   plugin's README row if one fits.

`delete-current-file` and Obsidian's built-in file-explorer rename are out of scope.

### Phase `acceptance` — Highlights-open test and the gist rename

**Why:** Return in Bob Refs opens the selected PDF in Highlights, so the file being
renamed is often open. A plain `rename(2)` is not an `NSFileCoordinator` move, so
Highlights' open document may never learn of it; a later autosave or sidecar export
could recreate `<old>.pdf` or `<old>.md`, and the next scan would duplicate the note.
Only a manual GUI test can settle this.

1. Write a self-contained acceptance script that builds a throwaway scratch vault under
   `/tmp` (git repo, minimal `ref/` and `lib/blogs/` layout, a small PDF, then
   `bob ref scan -w --no-hooks` with `BOB_DIR` pointed at it, using
   `BOB_VAULT_SYNC_LOCK_FILE` and `XDG_STATE_HOME` scratch values so it never touches
   the live vault or its locks). The script then:
   1. pauses while Bryan opens the scratch PDF in Highlights;
   2. runs `bob ref rename <old> <new> -o` and shows the result;
   3. pauses while Bryan adds a highlight, waits for autosave, and quits Highlights;
   4. lists the directory and prints PASS when no `<old>.pdf` or `<old>.md` exists, or
      FAIL with what reappeared, plus `lsof` output for the PDF captured while it was
      open (to show whether Highlights holds a descriptor — a possible detector). Copy
      it to the Mac over SSH when the Mac is reachable (`ssh mac`, best effort).
2. Use `/sase_questions` to ask Bryan to: update bob on the Mac (`just install-all` in
   the bob-cli checkout), run the script, and report PASS/FAIL with its output. In the
   same round, when `rename_gist` is `true`, ask him to confirm that
   `442a6bf555914893e9891c11519de94f.pdf` is closed in Highlights on every machine.
3. Record the verified behavior and the date in `docs/ref.md` under `## bob ref rename`
   ("When Highlights has the PDF open"), and commit. Phase `mac-rename-card` reads it.

   > [!decision] rename_gist Rename the gist in the live vault (step 4 below).

4. If `rename_gist` is `true` and Bryan confirmed: on the live vault, run
   `bob ref rename 442a6bf555914893e9891c11519de94f karpathy_llm_wiki -d`, check the
   plan (two moves, one `sase.md` link, `source_pdf` only), then run it without `-d`.
   Report the commit SHA. Then check `bob ref show karpathy_llm_wiki` and
   `bob ref doctor`. Leave the title alone; the docs say how to retitle (`title:` edit
   plus `bob ref sync lib/blogs/karpathy_llm_wiki.pdf -p frontmatter -w`). If the gist
   rename is refused, report the refusal and do not work around it.

### Phase `mac-rename-core` — RefsCore contract and pure state (bob-mac-capture)

Open the linked repo with `/sase_repo` (`bob-mac-capture`) and read its `AGENTS.md`.
Everything here is in the portable `RefsCore` target and its Linux-runnable tests.

1. **Response types** (new `Sources/RefsCore/RefsRename.swift`): `RefsRenameResponse`
   decodes the success envelope (`schema_version` with `decode`, everything else with
   `decodeIfPresent`; reject other schema versions through `SchemaVersioned`), including
   `from`, `to`, `moves`, `links`, `outcome`, `changed`, `commit`, `sync`, and
   `warnings`. Error envelopes decode to a typed failure carrying `code`, `message`, and
   `hint`.
2. **Fetcher:** add `rename(notePath:name:)` to `RefsFetching` and `BobRefsFetcher`
   (`RefsFetching.swift`). It runs `ref rename -f json -- <note path> <name>` through
   `decodeReport` (which reads stdout whatever the exit code) on a dedicated
   `refs-rename` lane with `cancelsPreviousInLane: false` and a `renameTimeout` of **180
   s** (above bob's own 60 s + 30 s lock budgets plus two sync cycles). Give other
   conformers a default or update them: `FakeRefsFetcher`
   (`RefsInspectorTests.swift:762`), `TimedOutRefsFetcher`, and `ScriptedRefsFetcher`
   (`RefsLibraryTests.swift:820,907`).
3. **Problem mapping**, like `RefsLibrary.scanProblem`: exit 2 with no JSON →
   `bob_too_old` ("This bob can't rename references yet — update bob on this Mac");
   timeout → `timed_out`; bob error envelope → its `message` and `hint`; anything else →
   `transport`.
4. **Stores:** `RefsOpenLogStore.rename(from:to:)` rewrites matching open-log entries
   while keeping their timestamps (make the private `save` usable for it).
5. **Pure card state and keys** (mirroring `RefsCopy.swift`): a `RefsRenameCard` value
   (frozen item ID and note path, read-only title, draft stem prefilled with the current
   stem, a `.pdf` suffix, phase idle/running/failed(message, hint)) and a key map:
   Return submits (ignored while running or while IME text is marked), Esc and Ctrl-[
   cancel (ignored while running), and every other key returns nil so the text field
   edits natively.
6. **Presentation strings:** the card caption "Also renames the note and updates links.
   Keeps the reading task."; the quiet hint "Close the PDF in Highlights first."; the
   footer status "Renamed → karpathy_llm_wiki.pdf · 1 link updated" (pluralized, with a
   no-link variant); the VoiceOver announcement; and the hidden-panel notification text.
7. **fake-bob:** add a `ref rename` branch to `Tests/Fixtures/fake-bob` driven by
   `FAKE_BOB_REFS_RENAME_FIXTURE` and `FAKE_BOB_REFS_RENAME_EXIT`, recording argv as the
   other branches do. Base the success fixture on real `bob ref rename -d -f json`
   output from a scratch vault, with the write-mode fields from the contract.
8. **Tests** (`Tests/RefsCoreTests`): decoding (success, unchanged, already_renamed,
   error envelope, unknown fields, wrong schema), argv and lane, problem mapping
   including older bob, store rename, card key map and state transitions, and
   presentation strings. Run `just portable-check` (on athena if this host lacks Swift).

### Phase `mac-rename-card` — the ⌘⇧R rename card (bob-mac-capture)

Read `docs/ref.md` "When Highlights has the PDF open" first. If the acceptance test
**passed**, ship the card with the quiet hint. If it **failed**, apply decision
`highlights_fallback`.

> [!decision] highlights_fallback = confirm Only if the acceptance test failed: the
> card's hint becomes a visible warning naming the file ("Close <stem>.pdf in Highlights
> first — Highlights may recreate the old file"), and the first Return turns into "Press
> Return again to rename". Only the second Return submits.

> [!decision] highlights_fallback = hold Only if the acceptance test failed: land
> everything below, but put ⌘⇧R and the ⌘K item behind a default-off setting (documented
> in the README as experimental), and record a `PROPOSED FOLLOW-UP:` note for a real
> Highlights fix.

1. **Keys:** in `RefsKeyRouter` (`case KeyCode.r`), `.command` stays `.refresh` and
   `[.command, .shift]` routes to a new `RefsCommand.showRenameCard`. Do not use bare or
   ⌥ letters (they type into search), Return or Tab (they open Highlights), or F2.
   Update the router test that asserts ⌘⇧ combinations return nil only as far as R
   requires.
2. **⌘K menu:** add `RefsAction.rename` ("Rename…", ⌘⇧R) to the per-row section in
   `RefsActionsMenu.swift`, disabled while a rename runs or with no selection.
3. **Card view:** shown where the Copy card renders (`RefsPanelView.swift:137-225`) and
   mutually exclusive with it. It shows the read-only title, a "File name" field
   prefilled with the current stem and fully selected, the fixed `.pdf` suffix, the
   caption, and the hint. Build the field on the `PomodoroNameField` pattern
   (`PomodoroNameField.swift`), with first-responder repair.
4. **Key handling:** add a rename branch in `RefsPanel.processKeyEvent` ahead of the
   router, like the Copy-card branch (`RefsPanel.swift:334-341`), but return the event
   (pass through) for every key the rename key map does not consume. IME checks look at
   the rename field. Put the card first on the Esc ladder
   (`RefsPanelModel.perform(.escape)`). Close the card on selection change, hide, or
   present, as the Copy card does, except while a rename is running.
5. **Running:** submitting calls the fetcher through a new `RefsLibrary.rename` that
   defers watcher refreshes while it runs (like `handleWatcherChange` during a scan) and
   refreshes once afterwards. Disable resubmission while it runs. Hiding the panel must
   not cancel it; when it completes while hidden, post a notification through
   `NotificationService`, as scan does.
6. **Success:** build a fresh listing that selects `to.path` (mirror
   `buildFreshListingForScan`), so the old row is never shown as "No longer in your
   library". Migrate open history with `RefsOpenLogStore.rename(from:to:)`, drop
   inspector-loader cache entries for the old ID (content, thumbnails, show cache, and
   intrinsics), remap other path-keyed panel state (`selectedID`, `knownItems`,
   `unavailableIDs`, `revealedPageID`), set the footer status, and post the VoiceOver
   announcement. Do not re-key app identity to marker IDs. For `unchanged` and
   `already_renamed`, select `to.path` and show a matching status.
7. **Errors** render inline in the card from bob's `message` and `hint`, and the draft
   stays editable. Older bob shows the `bob_too_old` text.
8. **README:** a key-table row (⌘⇧R Rename…), a short "Rename card" section next to
   "Copy card", and the ⌘K menu paragraph. Leave the footer key hints unchanged, since
   footer tests pin the exact strings.
9. **Tests** (`Tests/BobMacCaptureTests`, using the fake-bob harness): router (⌘⇧R →
   card, ⌘R still refresh), ⌘K section contents, footer hints unchanged, card prefill
   and focus, `processKeyEvent` passing text keys through, Esc ladder order, argv and
   lane, selection following `to.path`, history migration, inline errors, older bob, and
   completion while hidden.
10. **Compile:** run `just portable-check` (athena if needed), then — per the repo's
    `AGENTS.md` — compile a scratch copy on the Mac (rsync without `.build`/`.git`,
    `bundle.sh`, `just typecheck-tests`, then delete the copy; never Bryan's checkout or
    `just install`). If the Mac is offline, say in the final report that the macOS
    targets are unverified.

## Out of scope (record as `PROPOSED FOLLOW-UP:` notes where relevant)

- Renaming annotated references (v2: the identity-origin field), folder refiling
  (`blogs` → `papers`), retitling (`--title` or `bob ref retitle`), bulk renames, and
  `xlib/` renames.
- A bob-provided title-derived stem suggestion for the card (it must come from bob as a
  JSON field, never from Swift heuristics).
- The gist's bad title and author host metadata.
- Guarding `delete-current-file` and Obsidian's built-in file-explorer rename.
- A durable crash journal for the sub-second mutating window (before-images plus verify,
  rollback, and re-run detection cover v1).
- Repairing a `sase.md` link rewrite lost to a vault-sync "remote wins" conflict
  (`bob ref doctor` reports `open_ref_without_task`).

## Verification

- bob-cli phases: `just` lint and test recipes (as the repo's justfile defines them)
  plus the new tests. Check `bob ref --help`, `bob ref rename --help`, and the root help
  fixtures by eye for alignment and alphabetical order.
- bob-plugins: `npm run build:check`, `npm test`, `bob plugins sync`.
- bob-mac-capture: `just portable-check`, plus the Mac scratch compile and
  `just typecheck-tests` (best effort); macOS CI runs the full tests.
- End to end: the acceptance script on the Mac, then (if decided) the live gist rename:
  `bob ref show karpathy_llm_wiki` shows the same id and reading task, and
  `bob ref doctor` reports no new diagnostics.
