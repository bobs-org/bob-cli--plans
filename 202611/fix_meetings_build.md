---
tier: tale
title: Fix the master build broken by the meetings notes layer
goal:
  "`just install` and the `just check` gates (fmt, lint, test) pass again on master, the
  meetings recap marker matches the epic contract, and its golden files are committed."
size: small
proposed_by: bbugyi200.athena.1fq
proposed_with: claude/opus
create_time: 2026-11-24 11:54:03
status: wip
---

# Plan: Fix the master build broken by the meetings notes layer (commit dbf405e3)

## Diagnosis

`just install` fails because commit `dbf405e3` ("feat(meetings): add notes normalization
and rendering", bead `bob-cli-pw.1`) landed a new `src/native/meetings/` module that was
never compiled. The master CI run for that commit (45434238183) fails for the same
reason, in both the `lint` and `package` jobs. There are two compile errors:

1. **E0106 in `src/native/meetings/dossier.rs` (`fn class_fence`, about line 348).**
   `class_fence(line: &str, class_name: &str) -> Option<&str>` has two reference inputs,
   so lifetime elision cannot pick one for the return value. The returned fence is a
   sub-slice of `line` (`line.trim().split_once(' ')`), so the return must borrow from
   `line` only. The compiler's suggestion, which ties `class_name` to `'a` as well, is
   wrong.
2. **E0308 in `src/native/meetings/normalize.rs` (`push_wispr_item`, about line 242).**
   `parse_next_step(text: String)` takes `String` so it can be used point-free as
   `markdown_items(&content).into_iter().map(parse_next_step)` (about line 145). The
   Wispr path passes the `&str` it gets from `paragraph.lines().map(str::trim)`.

Once those compile, the same commit also fails the other `just check` gates. A throwaway
copy of the tree confirmed each problem, and each fix below:

3. **`just lint`, unused imports.** `rustc_gate` includes `-D unused`. The
   `pub(crate) use` re-exports in `src/native/meetings/mod.rs` (lines 18–27) have no
   non-test callers yet. Some (`DossierDocument`, `DossierError`, `NotesError`,
   `NotesInput`, `ReplaceRecapError`) are not used by the tests either. `src/native.rs`
   already has `#[allow(dead_code)] pub(crate) mod meetings;`, with a comment saying
   later meeting phases wire the production callers. But `dead_code` does not cover
   `unused_imports`.
4. **`just lint`, three denied Clippy classes in the new code:**
   - `clippy::collapsible_if` in `dossier.rs` at the
     `if let Some((level, heading)) = atx_heading(line) { if (2..=3).contains(&level) {`
     nest (about line 241).
   - `clippy::manual_pattern_char_comparison` in `intake.rs` (about line 107) and
     `normalize.rs` (`canonical_text`, about line 35):
     `line.trim_end_matches(|ch| ch == ' ' || ch == '\t')`.
5. **`just test`, missing goldens.** `src/native/meetings/tests.rs`
   (`update_or_assert_golden`) reads seven files from `tests/fixtures/meetings/golden/`,
   and none were committed: `model_wispr.json`, `model_markdown.json`,
   `model_adversarial.json`, `model_unrecognized.json`, `dossier_wispr.md`,
   `recap_wispr.md`, `recap_adversarial.md`. So `synthetic_models_match_goldens` and
   `dossier_and_recap_match_goldens` panic. The other seven meetings tests pass once the
   code compiles.
6. **Recap marker bug that the new goldens would lock in.** `recap.rs` has
   `const BEGIN_PREFIX: &str = "<!-- bob:meeting-recap:begin v1 sha256:";`, and
   `render_recap` appends `crate::native::intentions::digest(&inner)`, which already
   returns `sha256:<16 hex>`. Every rendered marker therefore reads
   `<!-- bob:meeting-recap:begin v1 sha256:sha256:<16 hex> -->`. The epic plan contract
   (`plan:202611/meeting_notes_event_refs.md`, "The recap region") specifies
   `<!-- bob:meeting-recap:begin v1 sha256:<16 hex> -->`. `recap_region` reads whatever
   follows `BEGIN_PREFIX` and requires it to be exactly `sha256:` plus 16 lowercase hex.
   Shortening `BEGIN_PREFIX` therefore fixes the writer and the parser together. This
   must land before goldens are committed and before a later phase writes recap regions
   into vault notes. After that, a format change would make existing markers parse as
   edited (never rewritten).

## Changes

### 0. Check origin first

Run `git fetch` and look at `origin/master`. Another agent may already have pushed some
of these fixes; meeting phases `bob-cli-pw.3` and `bob-cli-pw.4` are open and start from
this code. Rebase onto the latest `origin/master` and skip any item below that is
already fixed there. Do not change any bead's status.

### 1. `src/native/meetings/dossier.rs`

- Give `class_fence` an explicit lifetime tied only to `line`:
  `fn class_fence<'a>(line: &'a str, class_name: &str) -> Option<&'a str>`.
  `tldr_fence(line: &str) -> Option<&str>` has a single reference input, so it stays as
  is.
- Collapse the heading nest into a let-chain (the crate already uses edition-2024
  let-chains, for example in `parse_next_step`). The arms stay the same; they just lose
  one level of indentation:

  ```rust
  if let Some((level, heading)) = atx_heading(line)
      && (2..=3).contains(&level)
  {
      if heading.eq_ignore_ascii_case("transcript") {
          ...
      } else if is_next_steps_heading(heading) {
          ...
      } else {
          ...
      }
      index += 1;
      continue;
  }
  ```

### 2. `src/native/meetings/normalize.rs`

- In `push_wispr_item`, call `parse_next_step(line.to_string())`. Keep the `String`
  signature so the point-free `.map(parse_next_step)` call at about line 145 stays
  valid.
- In `canonical_text`, use `line.trim_end_matches([' ', '\t'])`.

### 3. `src/native/meetings/intake.rs`

- Use `line.trim_end_matches([' ', '\t'])` (about line 107).

### 4. `src/native.rs`

- Change the attribute on the meetings module to `#[allow(dead_code, unused_imports)]`.
  Keep the existing comment, which already explains that it is temporary until later
  meeting phases wire production callers. Do not delete the re-exports in
  `meetings/mod.rs`: they are the module's intended `pub(crate)` API, and the tests
  reach them through `use super::*`.

### 5. `src/native/meetings/recap.rs`

- Change `BEGIN_PREFIX` to `"<!-- bob:meeting-recap:begin v1 "` so that the
  `intentions::digest` output (`sha256:<16 hex>`) completes the marker. Leave
  `recap_region` alone. It already slices from the end of `BEGIN_PREFIX` to ` -->` and
  checks for a 23-character `sha256:` plus lowercase hex digest, and `recap_state` /
  `replace_recap` use `BEGIN_PREFIX` the same way.

### 6. Generate and review the goldens

- Generate them with: `BOB_UPDATE_MEETING_GOLDENS=1 cargo test --lib native::meetings`.
  Then run `cargo test --lib native::meetings` again without the variable; all 9 tests
  must pass.
- Review every generated file against its input fixture before keeping it. A trial run
  with the fixes above produced files that meet each of these checks:
  - `model_wispr.json`: `shape` is `wispr_plain_text` and `warnings` is empty. The TL;DR
    is the first paragraph, verbatim. There are two sections, "Migration risks" with 2
    items and "Deployment" with 1. Next steps are owned by "Avery Rowan" and "Morgan
    Hale" (the other party). `content_order` is section 0, then `next_steps`, then
    section 1. "CloudCove", "sounded like 'Mistral'", "18–24", and "18–42" all appear
    verbatim.
  - `model_markdown.json`: `own_title` is "Project Seaglass planning" and `shape` is
    `markdown`. There is one section, "Decisions", with 2 items. The checkbox next steps
    are parsed with owner "Morgan Vale" and owner `null`. The transcript holds both
    speaker lines.
  - `model_adversarial.json` and `model_unrecognized.json`: `shape` is
    `plain_paragraphs`, with the single warning "unrecognized notes shape; filed as
    plain paragraphs" and the whole text as the TL;DR.
  - `dossier_wispr.md`: YAML front matter with `bob_meeting_dossier: 1` and the
    `test_meta(false)` fields, then a `::: {.meeting-tldr}` fenced div, then the
    sections and Next steps in their original order.
  - `recap_wispr.md` and `recap_adversarial.md`: the first line matches
    `<!-- bob:meeting-recap:begin v1 sha256:[0-9a-f]{16} -->` exactly, with a single
    `sha256:`, and the last line is `<!-- bob:meeting-recap:end -->`. In the adversarial
    recap every payload token (`#task`, `- [ ]`, `[[Link]]`, `key::`, `^id`, `%%c%%`,
    the literal end marker, `<b>`) is entity-escaped.
- Commit the seven files under `tests/fixtures/meetings/golden/`. Do not hand-edit them.

### 7. Recap layout differs from the epic contract

`render_recap` does not produce the epic contract's layout. The contract has a
`## Notes` heading, a `> [!summary] TL;DR` callout, and `###` sub-headings. The current
output is the meta line, a plain TL;DR paragraph, and `##` headings that copy the notes'
own levels. Fixing this is outside this build fix. Record it with `/sase_new_task`: it
will most likely be filed as a note on the active epic `bob-cli-pw` rather than as a new
bead. Then leave the current layout as it is in the goldens.

## Verification

Run each of these in the foreground and give long runs a large timeout. Each must
succeed:

1. `cargo fmt --all -- --check` (`rustfmt.toml` uses `max_width = 80`; after the
   collapse, rustfmt joins
   `notes.content_order.push(ContentRef::Section(section_index));` onto one line).
2. `just lint`. The trial run passed with the changes above. Advisory Clippy warnings
   elsewhere in the crate are expected and are not errors.
3. `cargo test --lib native::meetings`, which must show 9 passed.
4. `just check`, the canonical gate.
5. `just install`, which reproduces the original failure and must now succeed.

## Out of scope

- The `test (macos-latest)` failures on earlier master commits (`props::lock_wait_*`,
  `props::absolute_note_with_relative_bob_dir_stays_confined`) are separate. They
  predate `dbf405e3` and only appear on macOS.
- The existing `noop_method_call` warning in `src/native/records/render.rs` is a
  warning, not an error, and is unrelated.
- No CLI, docs, or memory changes: the meetings module still has no production callers.
