---
tier: tale
title: Make open-PDF writes actually safe
goal:
  Marker writes on the Mac coordinate with Highlights, and a stale Highlights save can
  never demote or reopen a reading task.
size: medium
decisions:
  enable_now:
    ask:
      Turn coordinated PDF writes on in your managed bob config now, before the
      acceptance run?
    choices:
      now:
        Every Mac scan coordinates from its next run; a failed acceptance run can turn
        it back off
      after_acceptance:
        The key stays off until you run the acceptance kit; only the guard and doctor
        ship
    default: now
    why:
      The loss came from the uncoordinated path; a coordinated write's worst case is a
      hold
    answer: now
  decision_record:
    ask:
      Record a decisions strand that a marker reverted to bob's replaced copy is a stale
      save to restore?
    memory:
      - decisions
    default: false
    answer: false
proposed_by: bbugyi200.athena.13i
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-10-23 18:10:44
status: wip
---

# Plan: Make open-PDF writes actually safe

## Diagnosis

**Your suspicion is right.** The `bob-cli-at` epic built coordinated marker writes, but
nothing in your setup ever ran them.

- `execute_pdf_sync` (`src/native/highlights_ref/marker_write.rs`) takes the direct,
  uncoordinated path whenever `highlights.coordinate_pdf_writes` is false.
- False is the shipped default (`HighlightsConfig::coordinate_pdf_writes()`). Your
  managed config (chezmoi `home/dot_config/bob/config.yml`, byte-identical on the Mac)
  never sets the key.
- Enabling was deliberately gated on the real-Highlights acceptance run (`bob-cli-ay`),
  which has not happened.
- With the key off, `-v` output never prints `pdf_write_coordination`, so nothing showed
  that the feature was dark.
- The Mac's `bob` already contains the epic. Its scan receipt
  (`~/.local/state/bob-cli/ref/last-scan.json`) exists and carries `held: []`. So the
  missing piece is the key, not the build.

**What happened to `lib/chat/sase_blog_writers_block_task_plan.pdf`.** The source is the
vault commits, with the page-1 marker read out of every PDF revision. bob's writes are
single-revision lopdf rewrites; Highlights' saves are the original file plus one
appended revision.

| Time (EDT) | Writer         | Marker       | What it did                                                                                                                                                                                            |
| ---------- | -------------- | ------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| 16:58:37   | Highlights     | ready        | You annotated and saved                                                                                                                                                                                |
| 17:00:21   | cron `scan -w` | ready → next | Direct rewrite + rename under the open document (research case D)                                                                                                                                      |
| 17:00:55   | Highlights     | **ready**    | Highlights saved its stale in-memory copy over bob's file                                                                                                                                              |
| 17:04:27   | scan           | ready        | Read the stale save as **your** marker edit: note `status: ready`, reading task **`[*]` → `[ ]`**                                                                                                      |
| 17:07:18   | scan           | ready → next | The task was back at `[*]`; another direct rewrite under the still-stale document                                                                                                                      |
| 17:17:54   | scan           | next → wip   | `=x` close set `[/]`; a third direct rewrite. Highlights raised "changed by another application", and **Revert** dropped the annotation edit made after its 17:06:51 save (the 17:15 sidecar shows it) |

**Root cause.**

1. **The uncoordinated write.** Every marker write ran the pre-epic lopdf temp-file
   rename. An open `NSDocument` gets only a change notice and keeps its stale copy.
2. **An amplifier bug.** A stale save reverts the marker to exactly the copy bob
   replaced, and the next scan adopts that as a deliberate marker edit:
   - `resolve_sync_projection` sees marker changed, frontmatter unchanged, so the marker
     wins.
   - `v2_task_status_signal` turns a marker-only status change into a checkbox edit.
   - That demoted a Next task, against `decisions:task-lanes-are-sticky`. The demotion
     triggered two more rewrites into the open document, and the last one cost the
     annotation.
   - The same path would **reopen** a finished task. You mark `[x]`, bob writes
     `status: read`, and a stale Highlights save of `wip` reads as a reopen.
   - The docs' current advice (Save Anyway, then fix `status:` by hand) races the next
     scan. That race is exactly what happened at 17:04.

No defect turned up in the coordinated path itself. Its primitive passed the Swift
`NSDocument` fixture on the Mac during `bob-cli-at.1`. It has simply never run in
production.

## What changes for you

- From the next Mac scan on, a marker write asks Highlights to save first, writes on top
  of the saved bytes, and Highlights reloads in place.
- If any writer ever leaves Highlights stale again and you press **Save Anyway**, bob no
  longer believes the old marker. The next `-w` scan keeps your task's lane, restores
  the marker, and says so:
  `ok  <title>  updated note + marker  … · marker restored after a stale save`.
- `bob ref doctor` says whether PDF write coordination is on, and on macOS it warns when
  it is off.

## Work

### 1. Turn coordination on (chezmoi)

> [!decision] enable_now = now Do this whole step. With `after_acceptance`, skip it and
> say in the final summary that the key stays off until the acceptance kit passes.

- Open the `chezmoi` linked repo with `/sase_repo`. In `home/dot_config/bob/config.yml`,
  under `highlights:`, add the key with a short comment in the file's style:

  ```yaml
  # Ask Highlights (macOS file coordination) to save an open PDF before bob
  # rewrites its page-1 marker, then reload it in place. Inert off macOS. See
  # "PDFs open in Highlights" in bob-cli docs/highlights-ref-sync.md.
  coordinate_pdf_writes: true
  ```

- The file is shared by athena, apollo, and the Mac. It is not a template, and the key
  is accepted and inert on Linux.
- Apply it on the Mac now, best effort (`ssh mac` works only while the lid is open). The
  installed binary already supports the key.
  1. Back up `~/.config/bob/config.yml`.
  2. Copy the new source file byte-for-byte over it.
  3. Verify read-only with
     `bob ref scan -d -v 2>&1 | grep -E '^(pdf_write_coordination|reading_hold):'`.
     Expect `pdf_write_coordination: on`.
  4. Do not run `chezmoi apply` on the Mac before the chezmoi commit lands there. The
     Mac's next `chezmoi update` reconciles the file.
  5. If the Mac is offline, say so in the final summary and give the one-line enable
     step instead.
- Add a note to `bob-cli-ay` (read `sase_beads` memory first). It should say the managed
  config enabled the key on 2026-10-23 after this incident, and that the acceptance run
  still gates flipping the shipped default.

### 2. Stale-save guard (bob-cli)

**New pipeline field.** Add `highlights_marker_replaced_hash`:

- Declare a `FIELD_MARKER_REPLACED_HASH` constant in `src/native/highlights_ref/mod.rs`
  and add it to `PIPELINE_FIELDS`.
- Add a `ParsedNote` accessor next to `marker_hash()` in `model.rs`.
- Render it after `highlights_marker_base` in the metadata block.
- Its value is the parent-free projection hash of the marker that bob's **latest marker
  write overwrote**. Use the same `projection_hash(without_parent(..))` family as
  `highlights_marker_hash`.

**Writing the field.**

- Whenever a write-mode run writes the marker, the note it writes stores the pre-write
  marker's hash. This covers v1 and v2, coordinated or direct, and `ref scan -w`,
  `ref sync -w`, and create-time ingest.
- The plan already computes that hash (`marker_hash` from `marker_pf` in
  `plan_pdf_sync_v2`, and the v1 equivalent). In the coordinated saved-first branch the
  fresh marker contents equal the planned ones, so the hash still holds.
- A note write with no marker write keeps the existing value, with one exception. When
  that run observes the PDF marker hash equal to `highlights_marker_hash` (the editor
  has bob's write), drop the field. Never write a note just to drop it.

**Detecting a stale save.** In v1 `plan_pdf_sync` and v2 `plan_pdf_sync_v2`, before
`resolve_sync_projection`, use one shared helper. A stale save means all of these hold:

- the note has a replaced hash `R`, a stored hash `L`, and a stored base `B`;
- the PDF marker hash `M == R`;
- `M != L`.

When it holds:

- Resolve as if the marker were unchanged. Pass `B` (v2: `normalize_v2_base(B)`; v1: the
  base as stored) as the marker projection and `L` as the marker hash. A real task
  gesture or frontmatter change still resolves normally.
- Feed `B`'s status as `marker_status` to `plan_reading_task`. The stale status then
  never edits a checkbox, never reopens, and never conflicts with a genuine task
  gesture.
- Force a marker restore. Add a `stale_restore` term next to
  `decision.frontmatter_contributed` in v2's `semantic_needed` and v1's
  `marker_write_needed`, so the rendered synced marker replaces the stale contents.
  - Without `-w` (and not a dry run), do not raise `needs_write_pdfs` for a restore
    alone. Skip the marker write and leave it for the next `-w` run.
  - Dry runs preview the restore.
- Leave everything else alone: sidecar import, note body, the coordinated executor and
  its in-accessor re-check against `plan.marker.contents`, and holds. A held restore
  retries on the next scan, and the guard fires again until it writes.
- A marker that changed to anything other than `R` is still a genuine edit, handled
  exactly as today.

**Reporting.**

- Append `marker reverted to the copy bob replaced (stale editor save); restoring it` to
  the decision reason (verbose `sync_reason`, and `ref sync`).
- Give the concise scan line the dim detail `marker restored after a stale save`.
- Add a JSON field: `notes[].marker_write.restored: true`, present only for a restore.
  It is additive (`schema_version` stays 1), so the cron's scan receipt records it. Bob
  Mac Capture's decoders ignore unknown keys; confirm that, but make no app change.

**Tests** (Linux, existing fixtures and helpers):

- v2 replay of 17:04: task `[*]`, base and note `next`, `R` = hash of the `ready`
  marker, PDF marker `ready`. Expect: task stays `[*]`, note stays `next`, and the
  marker is rewritten to `status: next` under `-w`. The written note keeps `R`.
- v2 finished task: task `[x]`, base `read`, `R` = `wip`, PDF marker `wip`. Expect: no
  reopen, and the marker is restored to `read`.
- v2 gesture during a stale state: task `[/]`, base `next`, `R` = `ready`, PDF marker
  `ready`. Expect: status `wip`, no conflict error, marker `wip`.
- A genuine marker edit (`M` is neither `R` nor `L`) is still adopted, and the existing
  merge tests stay green.
- Field lifecycle:
  - it is written on every marker write;
  - it is dropped on a note write that observes agreement;
  - a missing field (legacy notes) disables the guard.
- No `-w`: no refusal and no marker write. A dry run previews the restore.
- The v1 equivalent of the first case.
- A coordinated fake (saved-first bytes) restores inside the accessor.
- A JSON golden for `marker_write.restored`.

### 3. Make "off" visible (bob-cli)

- `bob ref doctor` gains a `pdf_write_coordination:` line:
  - macOS: `on` or `off`. When off, add a doctor **warning**:
    `highlights.coordinate_pdf_writes is off: marker writes replace PDFs open in Highlights without asking it to save first`.
  - Elsewhere: `n/a (no OS file coordination)`.
- Leave scan and sync output byte-identical with the key off. The epic promised that,
  and doctor is the right place for the signal.
- Test both branches with `cfg!(target_os = "macos")` kept compile-visible on Linux.

### 4. Docs (bob-cli; read `docs.md` memory first)

`docs/highlights-ref-sync.md`:

- Add a short "Stale saves" subsection under "PDFs open in Highlights" stating the
  guard's rule, its report line, and its one limit. That limit: deliberately setting the
  marker back to the exact copy bob replaced is undone. Change status through the
  reading task instead.
- Replace the "Until then … set the page-1 `status:` back" paragraph in "Enabling" and
  the troubleshooting row for "The file has been changed by another application". The
  new advice: **Save Anyway** keeps your annotations, and the next `-w` scan restores
  the marker with no hand edit. Keep the acceptance-kit guidance for the shipped
  default.
- Document `highlights_marker_replaced_hash` where `highlights_marker_hash` is
  described.
- Add `marker_write.restored` to the JSON table, and to
  `docs/reference/json-outputs.md`.
- Add the doctor line where `doctor` checks are listed.

> [!decision] decision_record Use `/sase_memory_write` to add
> `sase/memory/decisions/stale-marker-saves-are-restored.md`, then run
> `sase memory init`. It covers:
>
> - **Applies to:** bob-cli `ref scan`, `ref sync`, and ingest.
> - **Claim:** a PDF marker that matches the copy bob's latest marker write replaced is
>   a stale editor save. bob keeps the note and the task lane and restores the marker;
>   it never treats that copy as a status edit.
> - **Why:** the 2026-10-23 incident, `decisions:task-lanes-are-sticky`, and the reopen
>   risk.
> - **Rejected:** adopting the stale marker; failing the PDF until a hand fix; PDF-byte
>   heuristics.
> - **Cost:** a deliberate revert to that exact copy is undone.
> - **Reopens when:** the marker no longer carries lifecycle status (`bob-cli-b0`).
> - **Evidence:** this plan and its commits.
>
> Link `[[decisions:open-pdf-writes-are-coordinated]]` and
> `[[decisions:task-lanes-are-sticky]]`.

## Verification

- bob-cli: `just check`, `just fmt`, `just lint`, and `just docs-check`, all exit 0.
- The guard is platform-neutral, so macOS CI is the backstop. Optionally compile on the
  Mac only if `doctor`'s cfg branch changes macOS-only code.
- Mac (step 1): the dry-run verbose check shows `pdf_write_coordination: on`. Report the
  result, or that the Mac was offline.
- Final summary for Bryan:
  - The guard and doctor line reach the Mac on his next `just install-all` there (or the
    release train).
  - Running `scripts/dev/highlights_coordination_acceptance` on the Mac is still the
    real-Highlights proof, and it gates `bob-cli-ay`.

## Out of scope

- Coordinating rename/move, id stamp, and `create --force` (`bob-cli-az`).
- Whether lifecycle `status` belongs in the marker at all, which would remove most
  writes during reading (`bob-cli-b0`).
- Flipping the shipped default (`bob-cli-ay`).
- Per-PDF dirty-target refusal (`bob-cli-b1`).
- Any Bob Mac Capture change.
