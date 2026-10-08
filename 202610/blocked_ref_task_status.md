---
tier: tale
title: Let bob ref scan accept a Blocked [?] ^ref task
goal:
  A ^ref task that bob task-status-hooks marked Blocked [?] syncs cleanly through bob
  ref scan/sync as a status-neutral overlay, with no planning failure, no PDF marker
  churn, and its Depends-On child line kept attached.
size: medium
decisions:
  keep_dep_child_attached:
    ask:
      "Insert the ## Tasks heading and audio embeds after the ^ref task's child lines?"
    default: true
    why:
      Blocked ^ref tasks carry a DEPENDS ON child line that these writers would split
      off.
    answer: true
  ref_library_blocked:
    ask:
      Also make bob ref list/show/doctor accept [?] trackers without an unknown_ref_mark
      warning?
    default: true
    why: Otherwise every blocked reference shows up as a bob ref doctor warning.
    answer: true
proposed_by: bbugyi200.athena.0y9
decided_by: auto
create_time: 2026-10-08 09:52:35
status: wip
---

# Let `bob ref scan` accept a Blocked `[?]` `^ref` task

## Problem

`bob task-status-hooks` treats the generated reference task
(`- [ ] #task #ref [[lib/...pdf]] #hide ^ref`) as an ordinary task. When that task has
an open dependency (a `⛓️ **DEPENDS ON:**` first-child line plus a derived
`[dependsOn:: ...]` field) or a future `scheduled` date, the hooks correctly set it to
Blocked `[?]`. The highlights sync then rejects the note, and `bob ref scan -w` records
a per-PDF planning failure and exits non-zero:

```
error  agent instructions budgeted router  generated PDF task line on line 4 is malformed;
expected a generated task with one of [ ], [*], [/], [x], [X], or [-] ...
```

The real vault line is:

```md
- [?] #task #ref [[lib/chat/agent_instructions_budgeted_router.pdf]] [fresh::
  2026-10-08] #hide [dependsOn:: sase__dynamic-agents-md] ^ref
  - ⛓️ **DEPENDS ON:** [[sase#^dynamic-agents-md]]
```

The frontmatter has `status: next`. Only the checkbox mark is rejected. The `[fresh::]`
and `[dependsOn::]` fields already pass the parser.

## Design decision: Blocked is a status-neutral overlay, not a new reference status

Treat `[?]` on the `^ref` task as the derived Blocked overlay that
`bob task-status-hooks` owns. It does **not** add a `blocked` value to the reference
`status` vocabulary (`ready`, `next`, `wip`, `read`, `abandoned`, `legacy`), so
frontmatter and the PDF marker never store "blocked".

Why:

- The decision records say Blocked is _derived_ state and explicitly reject an
  _authored_ Blocked status (`decisions:task-status-is-derived`, reaffirmed by
  `decisions:task-lanes-are-sticky`: "Blocked stays derived, overrides every lane"). A
  `status: blocked` value that could be typed into frontmatter or the Highlights PDF
  marker would be exactly that authored status. The hooks would then undo it, causing
  churn.
- Reference `status` describes reading progress. Blocked describes actionability.
  `bob ref list` reading states (queued/started/finished/ dropped) have no blocked slot.
- With a first-class status, every block and unblock transition would change the PDF
  marker. Each one would then need `--write-pdfs` or fail the scan.
- On recovery the hooks write the derived rank (`[ ]`, `[*]`, or `[/]`). That checkbox
  then flows through sync as a normal lifecycle signal. So the stored status is never
  used to "restore" a pre-block lane, which the decision records also reject.

Rejected alternative: adding `blocked` to `ALLOWED_STATUS_VALUES` with a bidirectional
`[?]` ↔ `status: blocked` mapping. Reasons are above. Revisit only if Bryan wants
Dataview or `bob ref list --status blocked` filtering badly enough to accept an
authored, marker-synced Blocked value.

### Precise semantics

Call `ready`, `next`, `wip`, and `legacy` the **open** statuses. Call `read` and
`abandoned` the **terminal** ones. "Selected status" means the marker/frontmatter status
after `resolve_sync_projection`, before the task signal is applied.

1. **Parse.** `[?]` is an accepted generated-task mark, alongside `[ ]`, `[*]`, `[/]`,
   `[x]`, `[X]`, and `[-]`. Every other custom mark (for example `[>]`) is still
   malformed.
2. **Read signal.** A Blocked task agrees with any open selected status. It contributes
   nothing, so the stored lane stays as is (for example `status: next`). If the selected
   status is terminal, Blocked acts like a reopen and targets `ready`. It uses the
   existing conflict check unchanged. If the base was already terminal (the checkbox
   reopened a closed reference to `[?]`), it contributes `ready`. If marker or
   frontmatter moved status to terminal away from the stored base while the task shows
   `[?]`, it reports the existing "PDF task conflicts with marker/frontmatter status
   edit" error, labelled `blocked`. This matches the current contract: to finish a
   blocked reference, check the `^ref` task `[x]`; the hooks leave done tasks alone.
3. **Write-back.** If the current mark is `[?]` and the final projected status is open,
   sync keeps `[?]`. It never rewrites a Blocked task to `[ ]`, `[*]`, or `[/]`, because
   the hooks own that recovery. If the final status is terminal (unreachable after step
   2, but deterministic), the existing rewrite to `[x]` or `[-]` and its close-date
   stamp apply.
4. **Dirty-target guard.** Toggling only the checkbox to or from `[?]` (for example
   `[*]` → `[?]` or `[?]` → `[ ]`) still counts as a checkbox-only body change.
5. **Recovery needs no special code.** When the hooks unblock the task to `[ ]`, `[*]`,
   or `[/]`, the next sync applies that mark as today. For example `[?]` → `[ ]` moves
   `status: next` to `ready`.
6. **Child lines stay attached** (decision `keep_dep_child_attached`). Blocked `^ref`
   tasks almost always carry an indented `⛓️ **DEPENDS ON:**` first-child line. Two
   writers currently insert lines _directly_ below the `^ref` line, which would split
   that child from its task:
   - `ensure_tasks_section` (the new `## Tasks` heading for annotation-task intake)
   - `insert_audio_embed` (the companion-audio player)

   Both must insert after the end of the `^ref` task's block instead: the task line plus
   following indented lines, the same rule as the existing `task_block_end_line_index`.
   Notes whose `^ref` task has no children keep byte-identical output.

## Implementation

All paths are relative to the repo root.

### `src/native/highlights_ref/annotation_tasks.rs`

- `parse_markdown_task_checkbox`: accept `'?'` as an unchecked mark
  (`checked == false`).
- `malformed_pdf_task_line_error`: list `[?]` among the accepted marks, in the order
  `[ ], [*], [/], [?], [x], [X], or [-]`.
- `apply_pdf_task_status_signal`: get the target from a status-aware helper (see
  `model.rs`) instead of the fixed `target_status()`, so Blocked can agree with open
  statuses and reopen terminal ones (semantics 2). The early return when the projection
  already matches the target must stay, so the open case is a no-op.
- `projection_pdf_task_mark` stays a pure status → mark mapping. Put the "keep `[?]`"
  rule in the write-back path below, not here: new notes generated from a projection
  never start Blocked.

### `src/native/highlights_ref/model.rs`

- `PdfTaskLine::status`: map `'?'` to a new `PdfTaskStatus::Blocked`.
- `PdfTaskStatus::Blocked`:
  - `target_status()` returns `None`. This keeps the `ref_task_mark_status` seam saying
    `[?]` has no fixed lifecycle status.
  - `label()` returns `"blocked"` (shown as `pdf_task: blocked` in scan/sync reports).
  - `contribution_reason()` returns something like
    `"blocked PDF task reopened status ready"`.
  - `conflict_action()` returns `"change"`.
- Add one helper, e.g.
  `PdfTaskStatus::target_status_given(current: Option<&str>) -> Option<&'static str>`:
  - Lifecycle variants return their fixed target.
  - `Blocked` returns the matching static constant for an open current status,
    `STATUS_READY` for a terminal one, and `None` when there is no status.

  Use this single helper for both the sync signal and the ref-library seam below, so the
  two cannot drift.

### `src/native/highlights_ref/note.rs`

- `rewrite_pdf_task_checkbox_for_projection`: if the parsed task's current mark is `'?'`
  and `projection_pdf_task_mark(projection)` is `' '`, `'*'`, or `'/'`, return the body
  unchanged. Otherwise keep today's behavior. `replace_pdf_task_checkbox_mark` itself
  stays generic, because the dirty guard uses it to apply the current mark, which can be
  `'?'`. `stamp_close_date` already ignores `'?'`.
- `ensure_tasks_section` (decision `keep_dep_child_attached`): anchor the inserted blank
  line and `## Tasks` heading after
  `task_block_end_line_index(lines, pdf_task_line_index, lines.len())` instead of the
  `^ref` line itself. Return the matching heading index.

### `src/native/highlights_ref/audio.rs`

> [!decision] keep_dep_child_attached Approved: change both insertion points
> (`ensure_tasks_section` above and `insert_audio_embed` here) plus their tests and the
> matching doc sentence. Declined: leave both writers and their docs as they are, and
> skip the child-line tests. Record the split-child hazard as a `bug` follow-up through
> `/sase_new_task` instead.

- `insert_audio_embed`: insert the blank line and audio embed after the same `^ref`
  task-block end instead of directly after the task line.

### `src/native/highlights_ref/marker.rs` and `mod.rs`

- Keep `ref_task_mark_status(mark)` as is; `'?'` still returns `None`.
- Add a narrow `pub(crate)` seam for `ref_library`, e.g.
  `ref_task_mark_target_status(mark: char, current: Option<&str>) -> Option<&'static str>`,
  plus whatever "is this a known tracker mark" predicate the library needs. Build both
  on `PdfTaskLine::status()` and `target_status_given`. Re-export them next to
  `ref_task_mark_status` in `mod.rs`.

### `src/native/ref_library/status.rs` (`bob ref list` / `show` / `find` / `doctor`)

> [!decision] ref_library_blocked Approved: make this change, add the
> `ref_task_mark_target_status` seam, and include the ref-library fixtures, tests, and
> `docs/ref.md` update below. Declined: leave `ref_library` and `docs/ref.md` unchanged,
> keep the `unknown_mark.md` fixture on `[?]`, and skip the new seam.

Today `[?]` produces an `unknown_ref_mark` diagnostic, which `bob ref doctor` rolls up
as a library warning. Make `decide_status` treat a single `[?]` tracker as usable:

- Get the tracker status from the new seam, passing the normalized frontmatter status.
- Open frontmatter status: tracker status equals frontmatter, so the result is
  `status_sync: ok`, the frontmatter status (for example `next` → reading state
  `queued`), and `reading_state_source: ref_task:[?]`.
- Terminal frontmatter status: tracker targets `ready`, and the existing
  pending/conflict comparison against `highlights_marker_base` applies unchanged.
- No usable frontmatter status: fall through to the existing no-tracker path, without an
  `unknown_ref_mark` diagnostic.
- Other unknown marks (for example `[>]`) still get `unknown_ref_mark`.
- Update the module doc comment.

## Tests

Use the existing patterns (`src/native/highlights_ref/tests/*.rs`,
`tests/cli/highlights/tasks.rs`, `src/native/ref_library/tests.rs`).

Unit (`src/native/highlights_ref/tests/`):

- `tasks.rs`:
  - `parse_pdf_task_line` accepts
    `- [?] #task #ref [[lib/x.pdf]] [dependsOn:: a] #hide ^ref` as `Present` with mark
    `'?'`.
  - The malformed-message assertion loop includes `"[?]"`.
  - `[>]` is still rejected.
- `status.rs`:
  - Blocked plus each open projection (`ready`, `next`, `wip`, `legacy`): no
    contribution, projection unchanged, `frontmatter_contributed == false`.
  - Blocked plus terminal base, marker, and frontmatter (`read`, `abandoned`):
    contributes `ready`.
  - Blocked plus a marker that moved `next` → `read` from base: conflict error whose
    message starts with `blocked PDF task conflicts`.
  - `rewrite_pdf_task_checkbox_for_projection` keeps `[?]` for `ready`/`next`/`wip`, and
    rewrites to `[x]` with a completion stamp for `read`.
  - `bodies_differ_only_by_pdf_task_checkbox` accepts `[*]`→`[?]` and `[?]`→`[ ]`.
- `seams.rs`:
  - Keep `'?'` in the `ref_task_mark_status == None` set, updating the comment so it no
    longer reads as unsupported.
  - Add a test that the new target seam agrees with `PdfTaskStatus::target_status_given`
    for every mark, including Blocked with open, terminal, and missing current statuses.
- `tasks.rs` or `audio.rs`: for a body whose `^ref` line has a tab-indented
  `⛓️ **DEPENDS ON:**` child:
  - Annotation-task insertion creates `## Tasks` after the child line.
  - Audio-embed insertion places the embed after the child line.
  - Both outputs stay byte-identical for a childless `^ref` line.

CLI (`tests/cli/highlights/tasks.rs`):

- Initial `highlights sync` of a PDF with marker `status: next`. Then edit the note so
  the `^ref` line becomes `[?]` with a `[dependsOn:: dep]` field and a tab-indented
  `⛓️ **DEPENDS ON:** [[other#^dep]]` child. Run `ref scan` (writing, no
  `--write-pdfs`):
  - Exit 0.
  - The note keeps `status: next`, the `[?]` line, and the child line unchanged.
  - The PDF marker is untouched.
  - A second scan reports the PDF unchanged.
- Recovery: change `[?]` → `[ ]` and run `ref scan --write-pdfs`. Frontmatter and marker
  move to `ready`.
- Blocked plus a marker edit to `read` from base `next`: the scan reports a per-PDF
  failure containing `blocked PDF task conflicts` and writes nothing.

Ref library:

- Switch `tests/fixtures/ref_library/vault/ref/papers/unknown_mark.md` from `[?]` to
  `[>]`, so `unknown_ref_mark` coverage stays (including
  `every_diagnostic_code_is_represented`).
- Add a `blocked_mark.md` fixture: `status: next` plus a `[?]` tracker. Assert
  `status == "next"`, `status_sync == "ok"`, `reading_state == "queued"`,
  `reading_state_source == "ref_task:[?]"`, and no diagnostics.
- Adding a fixture shifts fixture-wide totals (note counts, list output,
  counts/coverage, and `tests/cli/ref_library/*` or
  `tests/cli/highlights/doctor_library.rs` expectations). Update every such assertion.

## Documentation

- `docs/highlights-ref-sync.md`:
  - Add a `[?]` row to the lifecycle checkbox table: Blocked | (status unchanged) |
    "Waiting on an open dependency or future schedule; derived by
    `bob task-status-hooks`".
  - State the overlay semantics (1–6 above) next to the bidirectional-mapping paragraph
    and in the "Implemented conflict policy" bullets, including "to finish a blocked
    reference, check the `^ref` task".
  - Note that `## Tasks` and the audio embed are inserted after the `^ref` task's
    indented child lines.
- `docs/task-status-hooks.md`, in the paragraph on the machine-managed
  `#task #ref ... ^ref` task: blocking it to `[?]` is status-neutral for highlights sync
  (the reference keeps its stored lane and needs no PDF marker write); unblocking flows
  through like any other checkbox change.
- `docs/ref.md`, "status precedence": a single `[?]` tracker defers to the frontmatter
  status as described above, with source `ref_task:[?]` and no diagnostic.
  `unknown_ref_mark` now covers other custom marks.
- `README.md`, in the `bob ref` section near "The generated `^ref` task is the visible
  lifecycle control": one sentence saying a Blocked `[?]` `^ref` task is accepted and
  status-neutral. The marker status list stays the same.

## Out of scope

- No new CLI subcommands or options.
- No change to `bob task-status-hooks`, freshness/plan tiers, bob-plugins, or Bob Mac
  Capture. Their handling of `[?]` already exists or is unaffected.
- No PDF marker schema change.
- No SASE memory edits. If Bryan wants this overlay policy recorded as a decision, file
  it separately through `/sase_memory_write`.

## Verification

- `just check` (cargo fmt --check, clippy with all targets and features, and the full
  `cargo test --no-fail-fast`) passes.
- Optional, read-only: if a Bob vault is available at `~/bob`, run
  `BOB_DIR=~/bob cargo run --quiet -- ref scan --dry-run -n`. It must no longer report a
  planning failure for `lib/chat/agent_instructions_budgeted_router.pdf`, and must plan
  no note or marker change for it. Do not run a writing scan against the real vault.
