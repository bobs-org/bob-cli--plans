---
tier: tale
title: Finish and land task requirements (bob-cli-on)
goal:
  Fix the epic-caused gaps found at landing. A req-owning ^prj task files its notes into
  a real `## Notes` section. Capture reports implicit NOTES filing. The Mac reveals a
  requirement marker's raw bytes when the caret touches it. User docs stop teaching the
  retired REQUIREMENTS drawer. Then close epic bob-cli-on.
size: medium
bead: bob-cli-on
proposed_by: bbugyi200.athena.bob-cli-on.land
proposed_with: claude/opus
status: done
---

- **PARENT:**
  [202611/task_requirements.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/task_requirements.md)
- **BEAD:**
  [bob-cli-on](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-on/README.md)
- **AGENTS:**
  - [bbugyi200.athena.bob-cli-on.land](https://github.com/bobs-org/bob-cli--agents/blob/main/sessions/bbugyi200.athena.bob-cli-on.land.md)
- **COMMITS:**
  - [eb236b4](https://github.com/bobs-org/bob-plugins/commit/eb236b40ef8ac32cf8dbf290218592a46a24cf53)
    — fix(task-block): port ^prj \#\# Notes home into cycler and nav twins

# Finish and land task requirements (bob-cli-on)

## Context

Epic **bob-cli-on** ("Task requirements (reqs)", plan
`plan:202611/task_requirements.md`) has all 17 phases closed. Its land agent verified
every phase, child epic, and note (see the `LAND TRIAGE` note on `bob-cli-on`). Four
gaps remain. All four were caused by the epic, so they are epic work and must be fixed
before it closes. Follow-ups that the epic did not cause are already filed
(`bob-cli-pn`, `bob-cli-po`, `bob-cli-pp`, +1 on `bob-cli-gd`). Do not redo them.

Repositories:

- bob-cli: your workspace.
- **bob-plugins** and **bob-mac-capture**: open each with `/sase_repo`
  (`sase repo open bob-plugins -r "<why>"`,
  `sase repo open bob-mac-capture -r "<why>"`). Use only the printed path, and read each
  repo's `AGENTS.md` first.

Ground rules from the epic still apply:

- Rust owns the semantics. Change bob-cli first, then copy the fixtures byte for byte
  into the plugins.
- bob-plugins: edit `src/` fragments only, keep each at 1000 lines or fewer, run
  `npm run build` and `npm test`, and deploy only the changed plugins with
  `bob plugins sync -p <id>`. Never pass `-F` unprompted.
- Never edit the Cargo.toml version or CHANGELOG.md.
- Run `just check`, not `just check-full`.

Interaction with the next epic: **bob-cli-ph** (true-case drawers,
`plan:202611/true_case_drawers.md`) waits on `bob-cli-on` closing. Its first phase fixes
the `^prj` NOTES bug only "if that bug survived the epic". Fixing it here, with the
design that plan already names, lets `bob-cli-ph.1` skip that step. Do not start any
true-case drawer work: no bold headers, no recognizer changes.

To build bob for manual checks, use `cargo build`. The binary is
`$CARGO_TARGET_DIR/debug/bob`, not `target/debug/bob`. Scratch vaults need no config:
`BOB_NOW=2026-06-15 bob capture -b <vault> -- '<draft>'`.

## 1. A req-owning `^prj` task keeps its notes in `## Notes` (epic notes #2 and #4)

**Defect** (reproduced at master 2b626d01). Note #2 traces the root cause to
bob-cli-on.3 (ca070941). `task_block_layout::note_home`, `ensure_notes_drawer`, and
`tidy::scoop_loose_notes` treat a `^prj` task like any other task:

- `note_home` looks for `NOTES` only among task-child drawers
  (`capture_drawers::drawers`). It never finds a project `##` section.
- `ensure_notes_drawer` goes through `capture_drawers::ensure_drawer`. Its
  `is_prj_parent` branch appends an ALL-CAPS `## NOTES` at the end of the note.
- The caller then inserts tab-indented bullets under it, which render as an indented
  code block.

Repro:

```text
pk.md:
---
type: "[[project]]"
---
- [ ] #task #prj Pk root ^prj
	- [ ] hand req

## Tasks
- [ ] #task Row a
```

- `bob capture -b V -- $'@pk+prj\n- first note'`, then again with `second note`. The
  note now ends with `## NOTES\n\t- first note\n\n\t- second note\n## NOTES\n`:
  tab-indented bullets and one extra empty heading per capture.
- First-req scoop: on a `^prj` with a loose `\t- loose` child,
  `$'@pm+prj\n- [ ] only req'` writes `## NOTES\n\t- loose`.
- `bob task tidy` on a `^prj` with a req and a loose note refuses: "it would add the
  line `## NOTES`". It refuses even when a `## Notes` section already exists.
- For contrast, the explicit `@pk+prj#notes` route already does the right thing through
  `sub_bullet.rs::plan_prj_section_capture`. It creates a Title Case `## Notes` once,
  with a blank line, column-zero bullets, and appends on later captures.

**Design** (matches `bob-cli-ph`'s drawer-contract step 0 and the plan's promotion and
migration, which both use `## Notes`). On a `^prj` owner
(`capture_drawers::is_prj_parent`), the NOTES drawer is the project note's `## Notes` H2
section:

- Find it through `capture_drawers::project_h2_drawers`, by drawer key `NOTES`. Any
  existing `## Notes` or `## NOTES` heading matches; keep its authored spelling.
- Create it with the explicit route's rules: Title Case `## Notes`, placed and separated
  exactly as `plan_prj_section_capture` / `ensure_drawer` create a project section.
  Create it only when a note lands in it (`notes_drawer_birth = with_first_note`).
- Write bullets at column zero, inserted at `capture_drawers::h2_insert_position`. Never
  write a duplicate heading.
- The scoop and tidy move each loose direct note of the `^prj` task (whole subtree) out
  of the task block into `## Notes`. Each moved line is dedented by exactly the task's
  child indent unit, so the note bullet lands at column zero with its nesting kept. The
  bytes are otherwise preserved.

**Work (bob-cli):**

1. `src/native/task_block_layout.rs`: make `note_home` and `ensure_notes_drawer`
   `^prj`-aware per the design. `drawer_header_offset` stays task-child only.
2. `src/native/task_block_layout/tidy.rs` (`scoop_loose_notes`, `plan_tidy`) and
   `writers.rs` (`insert_reqs` scoop, `toggle_note_req` req→note at depth 1 and the
   first-req scoop): apply the same home. Tidy's content-preservation check must accept
   the created `## Notes` heading, plus any blank separator line it reports, and the
   one-unit dedent. Tidy must report nothing on an already-canonical `^prj` note.
3. `src/native/capture/sub_bullet.rs` (commit) and `src/native/capture/plan.rs` (~3609,
   the dry-run report) must use the H2 insertion (`h2_insert_position`, column-zero
   indentation) whenever the home is a project section. Reuse
   `plan_prj_section_capture`'s writer rather than copying it.
4. Vectors: add `^prj` cases to `tests/fixtures/task_reqs/layout.json` (RL: slots,
   note-home, tidy before/after with and without an existing `## Notes`) and
   `writes.json` (RW: first-req scoop, insert with a later note, req→note toggle at
   depth 1, the CRLF case). Update the fixture README index.
5. Tests: a CLI test in `tests/cli/capture/` covering the repro, where two note captures
   give one `## Notes` with column-zero bullets, plus the scoop case and a
   `bob task tidy` CLI case. Existing non-`^prj` vectors must stay byte-identical.
6. Docs: `docs/task-requirements.md` §3 (NOTES rule) gains one short paragraph: on a
   project note's `^prj` task, `NOTES` is the note's `## Notes` section, born with the
   first note, holding column-zero bullets. Link it from §3.4 tidy if that section lists
   homes.

**Work (bob-plugins), after the bob-cli vectors are final:**

7. Port the same home into the cycler twins:
   `plugins/task-status-cycler/src/196-task-block-layout-twin.js`
   (`taskBlockEnsureNotesDrawer`, `taskBlockNoteHome`, `taskBlockProjectNotesDrawer`),
   `197-task-block-tidy-twin.js`, and `198-task-reqs-writers-twin.js`. Mirror the
   bob-cli helpers by name in comments, as the fragments already do.
8. Re-copy the twins byte for byte into nav (`696`, `697`, `698`). Copy `layout.json`,
   `writes.json`, and `README.md` into
   `plugins/task-status-cycler/scripts/fixtures/task_reqs/` and
   `plugins/bob-navigation-hotkeys/scripts/fixtures/task_reqs/`. Block ID Prompt and
   bob-ledger-tools carry only the RQ `vectors.json`, which is unchanged. If
   `vectors.json` changes anyway, copy it everywhere.
   `scripts/test-task-reqs-twin-copies.cjs` enforces all of this.
9. Run `npm run build`, `npm test` (the RQ/RL/RW vector suites and the twin-copy test
   must pass), and `npm run docs:check` if docs changed. Then run
   `bob plugins sync -p task-status-cycler` and
   `bob plugins sync -p bob-navigation-hotkeys`. Known timing flakes (`bob-cli-io`,
   `bob-cli-km`) may fail under full load: rerun them alone and cite the bead.

## 2. Capture reports implicit NOTES filing (bob-cli-on.3 PROPOSED FOLLOW-UP #3)

**Defect.** On a task with `- [ ] a req` and no NOTES drawer, the note-only capture
`$'@work+ex\n- first note'` correctly writes `\t- NOTES\n\t\t- first note`. But
`-f json --dry-run` and the commit report `"drawer": null` and `notes[0].drawer: null`,
and the human preview never mentions NOTES. capture-reqs step 4 required the dry run and
preview to say where notes file. The explicit `#notes` route already reports
`· Notes · 1 note  new drawer`.

**Work.** Find where the `DrawerJson` computed by the `NoteHome::Notes` / `NewNotes`
branches (`sub_bullet.rs` ~116 and `plan.rs` ~3609) is dropped before the JSON `drawer`
/ `notes[].drawer` fields and the human "under … · <drawer>" line. Report it the way the
explicit drawer route does: `created: true` when NOTES is born, and the `new drawer`
marker in human output. Do this for task-child NOTES and for the `^prj` `## Notes`
section from step 1. Add CLI assertions to `tests/cli/capture/reqs_existing.rs` (dry run
equals commit, as `normalize_dry_run` already checks). If
`docs/reference/json-outputs.md` or `docs/capture.md` says the implicit route reports
`null`, fix that wording.

## 3. Mac: caret adjacency reveals a requirement marker (epic note #6)

**Defect.** Plan §4 ("Caret adjacency reveals the raw bytes") and mac-authoring step 3
are unimplemented. In bob-mac-capture `Sources/BobMacCapture/CapturePanelModel.swift`,
`paintHighlighting` (~23260) paints a depth-bearing `req_marker` span's `[ ] ` clear
whenever `concealRouteStartGlyphs` is set. `requirementMarkerSpans()` (~1943) feeds
`RequirementMarkerGlyphOverlay` (CapturePanelView.swift ~1691) with no caret gate. The
caret therefore walks invisibly across four transparent cells, and only Backspace
(`consumeRequirementMarkerBackspace`) looks at it.

**Work.**

1. A span is caret-adjacent when the collapsed caret's UTF-8 offset lies in
   `[span.start, span.end]` (inclusive at both ends) or a non-empty selection intersects
   it. Skip concealment for such spans in `paintHighlighting` (it already receives
   `cursorUTF8`). Exclude them from `requirementMarkerSpans()` so the overlay does not
   draw over the revealed bytes. Use one shared predicate for both.
2. Repaint when the caret enters or leaves a `req_marker` span without a reparse,
   following the existing `refreshRecordRouteConcealmentIfNeeded` lockstep pattern.
   Never mutate the draft.
3. Tests: hosted or model tests showing that a caret at the span start, inside, and at
   the end reveals the bytes and drops the overlay span, and that a caret elsewhere
   conceals them. Atomic Backspace and overtype absorption must behave as before.
4. Docs: one line in `docs/capture-panel.md` (or wherever the hash run is described)
   saying the caret reveals the raw `[ ] `.
5. Gate per that repo's `AGENTS.md`:
   - compile a scratch copy on the Mac and run `just typecheck-tests`;
   - run `just portable-check`;
   - run `Scripts/macos-ci-check` in the foreground with a generous timeout, as the
     pre-push gate.

   Watching the exact-SHA macOS run after the push is the host's post-commit concern. Do
   not order this epic's close after it (see §6).

## 4. User docs stop teaching the retired `REQUIREMENTS` drawer

This epic renamed every `REQUIREMENTS` drawer to `NOTES` vault-wide and made
"requirement" a first-class term. Several user-facing examples still create or name a
`REQUIREMENTS` drawer. The worst is `docs/capture.md:229`: "`@a+x#req @b+y note` puts
the note in ^x's REQUIREMENTS". It now reads like a requirement, and bob refuses
requirements in drawers.

**Work.** In examples and prose only, replace the `REQUIREMENTS` drawer with
`FUTURE WORK`, or with `NOTES` where the example is about notes. Change `#req` /
`#requirements` selectors to match (`#future`, `#notes`). Keep every example internally
consistent: its draft, output, JSON, and line counts.

- bob-cli:
  - `docs/capture.md` (≈229, 290–291, 7188–7226, 8524, 10778);
  - `docs/completion.md:487`;
  - `docs/reference/capture-syntax.md:1362`;
  - the capture CLI help example `--drawer REQUIREMENTS`. Edit the help source in
    `src/`, update `tests/fixtures/help`, then run `just docs-gen` to regenerate
    `docs/reference/cli/capture.md`.
- bob-mac-capture: `docs/capture-panel.md` (≈458–483) and `docs/capture-previews.md`
  (≈154). `docs/reference/mac-capture/` in bob-cli is a gitignored mount of these.
- Leave alone: the migration and history text that names `REQUIREMENTS` on purpose
  (rename-drawer docs, `docs/task-requirements.md` §9, `docs/develop/decisions.md`,
  `docs/reference/glossary.md`), and the Rust tests that use `REQUIREMENTS` as an
  arbitrary drawer name.

Run `grep -rn REQUIREMENTS docs` afterwards and justify each survivor.

## 5. Verify

- bob-cli:
  - `cargo fmt --check`, `just lint`, and `just check` all pass;
  - `just docs-gen` leaves no unexpected drift, and `just docs-check` passes;
  - `uv run --project tools/docs python -m pytest -q tools/docs/tests/test_contracts.py`
    stays 8/8.
- Re-run step 1's repro on a scratch vault with the fresh binary. Expected after two
  note captures:

  ```text
  - [ ] #task #prj Pk root ^prj
  	- [ ] hand req

  ## Tasks
  - [ ] #task Row a

  ## Notes

  - first note
  - second note
  ```

  Also check: the scoop case files `- loose` at column zero under `## Notes`, and
  `bob task tidy` then reports 0 changes.

- bob-plugins: `npm run build:check` and `npm test` pass (or only the known flakes fail,
  rerun alone), and both syncs report up to date.
- bob-mac-capture: the step 3 gate passes.
- Read-only on the live vault: run `bob task tidy` (dry run, fresh binary, no `-a`)
  against `~/bob`. Report any new `^prj` moves in the close note, and do not apply them
  here.

## 6. Closeout (final step; do it in this same turn)

Do not wait for, or order this after, anything that needs this work's own commit (its
SHA, push, or post-push CI). Closing the epic in the same turn as the final code is the
normal landing.

1. Run `sase bead epic-symbols bob-cli-on`. If it lists any `--epic-symbol` entry,
   resolve it (wire it up, privatize it, add a non-test pragma, or delete it per the
   Symvision epic-whitelist policy), or re-key it to a still-open later bead that needs
   it. None existed at landing.
2. Run `sase bead close bob-cli-on --note "<verification>"`. The note should summarize:
   - the four fixes, with commits/SHAs if known and the new vectors (RL/RW `^prj` ids);
   - the gates run and their results;
   - the plugin versions deployed;
   - the live-vault tidy dry-run result;
   - a pointer to the `LAND TRIAGE` note for follow-up dispositions (bob-cli-pn,
     bob-cli-po, bob-cli-pp, +1 bob-cli-gd).

   Never use `--force` merely to make the close succeed. If the close is rejected for
   leftover epic-symbol entries, clean them up and close again.

3. Run `just symvision` if the recipe exists. At landing the justfile had none; say so
   in your final report.
4. In `sase/repos/plans/202611/task_requirements.md` (the PLAN path
   `sase bead read bob-cli-on` shows), change the frontmatter `status: wip` to
   `status: done`.
5. `bob-cli-on` has no `parent_bead`, so nothing further cascades. Closing it releases
   `bob-cli-ph`'s wait. Mention that its drawer-contract step 0 `^prj` fix is now done.
