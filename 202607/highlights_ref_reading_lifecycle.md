---
create_time: 2026-07-14 07:21:36
status: done
prompt: 202607/prompts/highlights_ref_reading_lifecycle.md
tier: tale
---
# Plan: `^ref` task statuses drive the reference-note reading lifecycle

## 1. Problem

`bob highlights scan -w` aborts a PDF with:

> generated PDF task line on line 1 is malformed; expected a generated task such as `- [ ] … ^ref`, `- [x] … ^ref`, or
> `- [-] … ^ref` …

The reference note it chokes on looks like this (real example, `ref/chat/backcompat_lifecycle_governance.md`):

```markdown
---
status: done
…
---

- [*] #task #ref [[lib/chat/backcompat_lifecycle_governance.pdf]] #hide ^ref
```

The `^ref` "reference lifecycle task" had been promoted to Obsidian Tasks **Next** (`[*]`) — in this case automatically,
because the ref is referenced as a dependency from a project note and `bob mark-next-tasks` promoted it. The highlights
parser only understands three checkbox marks (`[ ]`, `[x]/[X]`, `[-]`), so any note the user (or another Bob command)
moves to **in‑progress** (`[/]`) or **next** (`[*]`) becomes unscannable.

### Root cause

`parse_markdown_task_checkbox` (in `src/native/highlights_ref/mod.rs`) accepts only `' '`, `'x'/'X'`, `'-'`. `/` and `*`
fall through to `None`, which `parse_pdf_task_line` reports as a malformed generated task.

## 2. Product vision

The `^ref` task is already a first‑class citizen of Bryan's Obsidian Tasks + Pomodoro + `mark-next-tasks` workflow. This
feature makes that the single, intuitive source of truth for a reference's **reading status**: the checkbox you see in
the note _is_ the status. One lifecycle, no second mental model.

| `^ref` checkbox | Obsidian Tasks status | ref `status` | Meaning                                          |
| --------------- | --------------------- | ------------ | ------------------------------------------------ |
| `[ ]`           | Todo                  | `ready`      | In the reading queue, not started                |
| `[*]`           | Next                  | `next`       | Queued for action (today's Pomodoros / promoted) |
| `[/]`           | In Progress           | `wip`        | Actively reading                                 |
| `[x]` / `[X]`   | Done                  | `read`       | Finished                                         |
| `[-]`           | Cancelled             | `abandoned`  | Dropped                                          |

Why this is the right shape:

- **Intuitive** — the reading lifecycle _is_ the task lifecycle. Cycling the checkbox in Obsidian (task‑status‑cycler)
  moves the ref through the pipeline and the frontmatter follows.
- **Reliable / automatic** — `mark-next-tasks` already flips open `^ref` tasks between `[ ]`↔`[*]` based on today's
  Pomodoro ledger, so the reading queue self‑organizes around what's actually queued, and leaves `[/]`/`[x]`/`[-]`
  untouched (its documented transition table).
- **Beautiful** — `refs.base` renders a clean, color‑coded pipeline: what's **Next**, what's **In Progress**, what's
  **Ready**, then the archive.

## 3. Design decisions (I'm making the calls; flagging the notable ones)

**D1 — Five‑state canonical mapping (bidirectional).** The table above is a total mapping in both directions: checkbox →
`status` (on read) and `status` → checkbox (on write‑back / note generation). No mark is ever "malformed" again.

**D2 — `unread` becomes a deprecated alias of `ready`.** The vault currently has **zero** `unread` notes, and `unread`
("haven't read it") is semantically the same queue slot as `ready` ("ready to read") — and the `[ ]` checkbox can't
distinguish them. So `unread` is normalized to `ready` on input exactly like the existing `done → read` alias, and
`ready` becomes the canonical "not started" status. New‑PDF marker examples in the docs switch from `- status: wip` to
`- status: ready` so a freshly added book starts life in the queue.

**D3 — `legacy` is untouched.** The 282 `legacy` notes are bulk imports; almost none carry a `^ref` task, so the
checkbox path never touches them. `legacy` stays a valid, non‑lifecycle status.

**D4 — Migration of existing data — NOTABLE.** 16 notes are currently `status: wip` but carry `[ ]` (15) or `[*]` (1) —
i.e. `wip` was being used as a catch‑all default, not literally "in progress." Because the checkbox is now
authoritative, the **first** `scan` will reclassify them from their checkbox: **15 → `ready`, 1 → `next`**, and with
`-w` it rewrites those 16 PDF markers to match. This is a one‑time correction that aligns the labels with reality
(nothing is silently "downgraded" — they were never really in progress). I recommend accepting it. _(Conservative
alternative if you'd rather preserve the existing `wip` labels and instead auto‑correct their checkboxes to `[/]`: gate
the checkbox‑as‑signal behind a `PIPELINE_VERSION` bump so pre‑existing notes let their stored status win once; I can
switch to this in review.)_

**D5 — `next` and the PDF marker — NOTABLE, needs your blessing.** Because `mark-next-tasks` churns open refs between
`ready`↔`next` daily, this is the one real tension with "keep the marker `status:` in sync with the note."

- **D5‑a (default in this plan): treat all five statuses uniformly.** `next` syncs to the PDF marker like any status.
  This is literally "keep them in sync." Consequence: a promotion/demotion needs a marker write, so it's covered by the
  `-w` you already run (and by `--dry-run`, which never fails). A _bare_ `bob highlights scan` (no `-w`, no `--dry-run`)
  will report a per‑PDF failure for a ref whose Next state changed — but that's the **existing** behavior for any status
  change today (checking `[x]` without `-w` already fails), just more frequent. Simplest model, fewest edge cases.
- **D5‑b (refinement, if the churn bites): make `next` an ephemeral overlay.** `next` normalizes to `ready` for the
  durable projection/marker, and is re‑applied to the _note_ frontmatter purely from the live `[*]` mark. The PDF marker
  then stays `ready` for a queued ref, so Next churn never needs `-w` and never rewrites a PDF. Cleaner semantically
  (`next` = "ready + queued today", which isn't a durable property of the PDF), at the cost of the note's `status`
  intentionally differing from the marker's for `[*]` tasks, plus a little more code.

  I'll build **D5‑a** unless you say otherwise; tell me if you'd prefer the ephemeral‑`next` behavior and I'll fold in
  D5‑b.

## 4. Implementation — `bob-cli`

All in `src/native/highlights_ref/mod.rs` unless noted.

1. **Status vocabulary.** Add `STATUS_READY = "ready"` and `STATUS_NEXT = "next"`. `ALLOWED_STATUS_VALUES` becomes
   `ready, next, wip, read, abandoned, legacy`. Keep `STATUS_UNREAD` only as the deprecated‑alias constant (like
   `DEPRECATED_STATUS_DONE`), not in the allowed set.

2. **Checkbox parser.** In `parse_markdown_task_checkbox`, accept `/` and `*` (both `checked = false`) alongside the
   existing marks.

3. **`PdfTaskStatus`.** Replace the `Unchecked` variant with the explicit `Ready`, `Next`, `Wip` (keep `Read`,
   `Abandoned`, `Missing`). Update the mark→status map in `PdfTaskLine::status()` and everything that matches on the
   enum: `target_status()` (now total: every non‑`Missing` status maps to itself), `label()`, `contribution_reason()`,
   `conflict_action()`.

4. **Status → mark.** `projection_pdf_task_mark` becomes the total inverse: `ready`→`' '`, `next`→`'*'`, `wip`→`'/'`,
   `read`→`'x'`, `abandoned`→`'-'`, (`legacy`/anything else → `' '`). This drives both new‑note generation
   (`default_note_body`) and write‑back (`rewrite_pdf_task_checkbox_for_projection`).

5. **Signal application.** `pdf_task_target_status` simplifies to the total mapping and the old "unchecked reopens a
   terminal ref to `wip`" special case is dropped — unchecking a `read`/`abandoned` ref to `[ ]` now cleanly means
   `ready` (choose `[/]` if you mean resume). `apply_pdf_task_status_signal` keeps its existing early‑return (no
   contribution when the resolved projection already equals the checkbox target) and its marker/frontmatter **conflict
   detection** unchanged — so "you moved the checkbox to X but also edited the marker/frontmatter status to Y≠X since
   the last sync" still errors instead of guessing.

6. **Normalization.** Extend `normalize_deprecated_status` to also map `unread → ready` (in addition to `done → read`),
   so old inputs canonicalize and the `status_normalized` bookkeeping/marker write‑back already in place persists the
   canonical value.

7. **Error message.** Update `malformed_pdf_task_line_error` to list the valid marks including `[/]` and `[*]`.

8. **No `PIPELINE_VERSION` bump** (under D4 default): keeps the first scan's diff minimal — only notes whose
   status/checkbox actually changed are rewritten. (Bump only if we choose the conservative D4 alternative.)

The mark‑agnostic helpers (`replace_pdf_task_checkbox_mark`, `bodies_differ_only_by_pdf_task_checkbox`,
`dirty_note_allowed_change`) need no changes — once the parser accepts `[/]`/`[*]`, the git dirty‑target allowance that
lets a scan proceed when the _only_ change is the `^ref` checkbox toggle starts working for the new marks for free. This
is what lets `scan` run cleanly right after `mark-next-tasks` flips a ref to `[*]`.

## 5. `refs.base` (the "beautiful" deliverable — edited in `~/bob`)

`~/bob/refs.base` is a plain, git‑tracked vault file (not chezmoi‑ or plugin‑deployed), so it's edited in place and
committed with the vault. Redesign so the full pipeline reads at a glance:

- **`status_badge` formula** — add lifecycle badges and put them in pipeline order, e.g. `⭐ Next`, `🛠️ In Progress`,
  `📥 Ready`, `✅ Read`, `⚫ Abandoned`, plus `🗄️ Legacy`; keep the existing `unread` and `collect/review` fleeting
  badges so nothing regresses.
- **`status_order` formula** — a numeric key (`next=0, wip=1, ready=2, read=3, abandoned=4, legacy=5, …`) so views sort
  in reading‑pipeline order instead of alphabetically.
- **Views**:
  - **🔖 Reading Queue** — the active pipeline `next → wip → ready`, grouped by status, `next` on top.
  - **📚 All Refs** — broaden today's `wip/unread` filter to the active set (`next, wip, ready`).
  - **📊 By Status** — group by status, ordered by `status_order`.
  - (Optional) **✅ Archive** — `read`/`abandoned`.

Exact operators will match Obsidian Bases syntax already used in the file.

## 6. Docs

- `docs/highlights-ref-sync.md`: the status list (add `ready`, `next`; note `unread → ready` alias), the "generated task
  line" section (document the full five‑mark mapping, replacing the `[x]`/`[-]`‑only text and the reopen‑to‑`wip`
  paragraph), the conflict‑policy bullets, the marker examples/starter (`status: ready`), and the "unsupported status"
  rows in the failures table.
- `docs/mark-next-tasks.md`: a short note that `^ref` reference tasks are ordinary scanned tasks, so promotion to `[*]`
  flows into the ref's `next` reading status.
- Leave `memory/` canonical files alone (they need explicit approval to edit).

## 7. Tests

- Parser: `[/]` and `[*]` parse to `Wip`/`Next` (not malformed).
- Mapping round‑trips: mark→status and `projection_pdf_task_mark` status→mark for all five, plus `unread → ready` and
  legacy `done → read` normalization.
- Signal: promoting `ready`→`next` (and back) contributes cleanly with no conflict; a real
  marker/frontmatter‑vs‑checkbox divergence still errors.
- Dirty‑target: a note whose only change is a `[ ]`→`[*]`/`[/]` toggle is allowed through the preflight.
- Update/replace existing tests that assert `PdfTaskStatus::Unchecked` and the reopen‑to‑`wip` behavior.
- Full `cargo test` + `cargo clippy`; targeted `cargo fmt`.

## 8. Rollout & verification

1. `cargo test` / `clippy` / `fmt` green.
2. `bob highlights scan --dry-run` over the real library — confirm zero `plan_failures` (the `[*]` note now parses) and
   preview the reclassification of the 16 `wip` notes.
3. `bob highlights scan -w` — verify the 16 markers/notes update as expected, the failing note lands on `status: next`,
   and a second run is a no‑op (`writes: none`).
4. Reload `refs.base` in Obsidian and eyeball the Reading Queue.
5. Vault (`~/bob`) changes to `refs.base` and any rewritten notes are committed through the normal vault flow.

## 9. Out of scope / risks

- Not adding a `sync: false` mechanism or new CLI flags.
- Obsidian Tasks `statusSettings` already define `[/]`/`[*]` (per `mark-next-tasks`), so no vault‑config change is
  needed for the marks to render and cycle.
- Main risk is the D4 one‑time reclassification and the D5 `next`/marker/`-w` behavior — both surfaced above for your
  call before any code changes.
