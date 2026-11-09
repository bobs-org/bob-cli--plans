---
tier: tale
size: medium
title: Finish and land Record Notes (bob-cli-ir)
goal: 'Fix the last defects the Record Notes epic left behind: a missing blank line
  before a new section, a broken Mac build, and two Mac reference pages it pushed
  over the word cap. If the decision approves it, add the record-note glossary term
  Bryan asked for. Then close epic bob-cli-ir.

  '
decisions:
  memory_glossary_record_note:
    ask: Add the Record Note glossary term (aliases record, record type) that Bryan's
      bob-cli-ir note asks for?
    memory:
    - glossary:record-note
    default: false
    answer: false
proposed_by: bbugyi200.athena.bob-cli-ir.land
proposed_with: claude/opus
bead: bob-cli-ir
decided_by: auto
status: done
---

- **PARENT:**
  [202611/record_notes.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/record_notes.md)
- **BEAD:**
  [bob-cli-ir](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-ir/README.md)

# Finish and land Record Notes (bob-cli-ir)

## Context

Epic `bob-cli-ir` (plan `plan:202611/record_notes.md`) has all 13 phases closed. The
land agent `bob-cli-ir.land` checked every phase note and the epic's commits, plus
everything that landed since the epic started. The work is complete except for the four
items below. Follow-up triage is done and recorded on the epic (`sase bead note` "LANDER
TRIAGE"). Do not file or +1 anything again for those proposals.

Things on the tree that are **not** this tale's job (leave them alone):

- `just check` currently fails four lib tests from the notebooks commit `0a74731b`
  (j8.1): `native::catalog::tests::document_shape`,
  `native::completion::kinds::tests::every_value_arg_has_a_decision`,
  `runner::tests::every_native_command_is_on_exactly_one_canonical_path`, and
  `runner::tests::subcommands_are_contiguous_in_section_order_and_alphabetical`. These
  are recorded on active epic `bob-cli-j8`.
- `just docs-check` also reports `docs/reference/json-outputs.md` (j8.1), and Mac
  `capture-panel.md` / `keyboard.md` (pre-existing, task `bob-cli-h9`).
- The spaced `.`/`;` capture-rewrite aliases fire inside record field values. This is
  recorded on active epic `bob-cli-j6`. Do not change alias code here.

Epic `bob-cli-j6` has a landing tale in flight in `bob-mac-capture` too. Before each
repo edit, fetch and fast-forward to `origin/master`. If a change below is already
there, skip it and say so in the close note.

## 1. Blank line before a new record-update section (bob-cli)

Epic note #4. A record update that adds a **new** ALL-CAPS section (for example
`@eat/ninja_sushi` with `- MUST ORDER` / `  - Dragon roll`) writes `## Must Order`
straight after the last body line, with no blank line in between. The last body line can
be the closing fence of `## Source` on migrated notes, or the last `## Notes` bullet on
bob-created notes. Create already gets this right.

Root cause: `src/native/records/render.rs` `append_section_bullets`, in the `else`
branch (no matching heading). It pushes the blank separator only when the last body line
is non-blank **and** its line ending is empty. A normal file that ends in a newline
therefore gets no separator.

Fix:

- When the body's last line is non-blank, push one blank separator line before the
  heading. If that last line's ending is empty, first set it to `newline`.
- Keep today's behavior when the body is empty or already ends in a blank line. Never
  write two blank lines.

Add unit tests in the `render.rs` `tests` module, next to
`named_sections_merge_instead_of_duplicating`. Use `apply_record_update` with
`RecordNoteInput::section(...)` and assert on the exact bytes for:

1. A body ending in a `## Notes` bullet plus `\n`. Expect
   `- old\n\n## Must Order\n\n- Dragon roll\n`.
2. A body ending in a closing code fence plus `\n`.
3. A body with no trailing newline. Keep the existing separator behavior.
4. A body already ending in a blank line. Expect exactly one blank line before the
   heading.
5. CRLF input, so the separator uses `\r\n`.

Grep `tests/cli` for record-update goldens that encoded the old bytes, and update them
if any exist.

Verify by hand with the debug binary (`$CARGO_TARGET_DIR/debug/bob`). Use a throwaway
vault with `git init` and a `BOB_CONFIG_FILE` restaurant config (`dir: eat`, required
`status` with `to_try`/`liked` and `category`), and a note `eat/ninja_sushi.md` whose
body ends `## Notes\n- 2025-01-02: First visit.\n`. Running
`bob capture -b <vault> -- "$(printf '@eat/ninja_sushi\n- MUST ORDER\n  - Dragon roll')"`
must write a blank line before `## Must Order`.

## 2. Mac `.addTo` exhaustive switches (bob-mac-capture)

Epic note #6. Exact-SHA Mac CI run 42673185238 (for `ef5dc4a`) fails at **Build**. ir.12
`a5152aa` added `CaptureRelatedAction.addTo`, but three switches were never updated:

- `Sources/BobMacCapture/CapturePanelModel.swift`, `completeSimilarTaskIDAssignment`
  (`switch action`, near line 15612).
- `Sources/BobMacCapture/CapturePanelView.swift`, the `.similarTask` branch of the
  "Inserts …" string helper (near line 2096).
- `Sources/BobMacCapture/CapturePanelView.swift`, `similarTaskInsertsText` (near line
  2151).

`addTo` is a record splice that the Similar band applies directly. It must never make up
a task block-ID replacement. Add `.addTo` to the existing
`case .open, .discard, .unknown:` arm in all three places (nil replacement, empty text).
Change nothing else.

Before editing, grep `Sources` and `Tests` for any other exhaustive
`CaptureRelatedAction` switch and fix it the same way. Then follow
`bob-mac-capture/AGENTS.md`:

- Run `just portable-check` on athena.
- Compile a scratch copy on the mac (see `sase memory read tailnet.md`): rsync without
  `.build`/`.git`, run `Scripts/bundle.sh` and `just typecheck-tests`, then delete the
  copy. Never use Bryan's checkout or `just install`.
- Run `Scripts/macos-ci-check` from the repo **in the foreground with a long timeout**
  (at least 60 minutes; rerun with a larger timeout if it is killed). Its snapshot run
  must pass Build and Test. That snapshot run is this tale's Mac gate.

Do not order the epic close after the post-push exact-SHA run of your own commit. If the
mac host is offline, say so in the close note and the final report.

## 3. Bring the two record-grown Mac reference pages under 6000 words

Epic note #5. `bob-mac-capture/docs/protocol.md` is mounted as
`docs/reference/mac-capture/protocol.md`; it went from 5996 to 6062 words with ir.11
`fe9ce15` and now counts 6048. `docs/capture-previews.md` went from 6174 to 6424 words
with ir.11/ir.12.

Tighten both pages to at most 6000 words by bob-docs `count_words`, without dropping
contract facts. Prefer removing duplicated prose and merging record paragraphs that
repeat the compatibility table. Keep the pages' existing anchors so inbound links still
resolve.

Check the result with bob-mac-capture `just docs-check` and bob-cli `just docs-check`,
which mounts the sibling repo. Neither page may appear in the lint budget errors. The
other three pages listed above may still fail; they are not this tale's job.

## 4. Record Note glossary term (memory, decision `memory_glossary_record_note`)

> [!decision] memory_glossary_record_note Add the strand. Bryan's bob-cli-ir note #1
> says "The epic lander agent should add an excellent new glossary term for 'record
> note' before landing." The original Record Notes plan's reviewer had answered no to
> the same question, so this decision lets the reviewer settle it. When it is answered
> no, skip this step entirely and file nothing.
>
> Use `/sase_memory_write` first. Create `sase/memory/glossary/record-note.md` with
> frontmatter `keyword: Record Note` and aliases `record` and `record type`. Model it on
> `glossary:notebook`, and run `sase memory read glossary:notebook glossary:capture`
> first. Define it from `docs/records.md` and the shipped behavior, in about 120–180
> words:
>
> - A Bob vault note of a type declared under `records:` in config. Each type has an id,
>   `dir`, `label`, optional `emoji`, and an ordered `fields` schema.
> - It lives at `<dir>/<stem>.md`, with YAML frontmatter `type: "[[<type>]]"` and the
>   typed fields, plus an H1 title and a `## Notes` body of dated bullets.
> - Capture creates one with `<title> @<dir>/` plus `- key:: value` bullets, and updates
>   one with `@<dir>/<stem>`.
> - `bob records list|show|check|new|update|init` manages records for agents.
> - Every write is planned in full, never replaces a file, and refuses duplicates.
> - Say what it is not: not a [[project-note]], a [[reference-note]], or a [[notebook]],
>   and never a task residence.
>
> Link existing strands with `[[...]]` where the prose names them. Edit no other memory
> note. Then run `sase memory init` and `just docs-gen`, so the generated
> `docs/reference/glossary.md` and the instruction shims update.

## 5. Verify

- `cargo fmt --all -- --check`, plus the clippy project gate (`just lint`, or the clippy
  line of `just check`).
- `just check`. Only the four j8.1 lib failures listed above may remain, if they are
  still on `origin/master`. Every records, capture, and help test must pass.
- `just docs-gen`, then `just docs-check`. Only the j8/h9-owned pages may remain in the
  lint output.
- Mac gate from step 2.
- `git diff --check` in both repos.

## 6. Close out epic bob-cli-ir (final step, same turn)

1. Run `sase bead epic-symbols bob-cli-ir`. It was empty at landing review. Resolve or
   re-key any entry that appears (wire up, privatize, pragma, or delete, per the
   Symvision epic-whitelist policy).
2. Close the epic normally, never with `--force`:
   `sase bead close bob-cli-ir --note "<verification>"`. The note must cover:
   - All 13 phases verified.
   - Discovered issues #2/#3 fixed by `cb27f8bc`.
   - #4 fixed here, with the new tests.
   - #5: protocol and previews pages trimmed (give the word counts); json-outputs now
     owned by j8; capture-panel and keyboard by h9.
   - #6: `.addTo` switches, with the Mac compile and `macos-ci-check` evidence.
   - #1: the glossary term, or that the decision was answered no.
   - The triage outcomes from the LANDER TRIAGE note.
   - Integration: the aliases issue routed to j6, the notebooks failures to j8.
3. Run `just symvision` if the recipe exists. It is absent today, so record that.
4. Set `status: done` (currently `status: wip`) in the frontmatter of the epic's plan
   file: the PLAN path that `sase bead read bob-cli-ir` prints for
   `plan:202611/record_notes.md`.
5. `bob-cli-ir` has no `parent_bead`, so nothing further to close.
