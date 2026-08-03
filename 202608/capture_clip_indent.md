---
tier: tale
title: Indent bob capture clipboard children like other Obsidian sub-bullets
goal:
  Clipboard child bullets produced by `bob capture` `%`/`--clip` markers use the target note's own sub-bullet whitespace
  prefix (falling back to a tab) instead of a hard-coded two spaces.
proposed_by: bbugyi200.athena.sq
create_time: 2026-08-03 08:39:26
status: wip
---

# Plan: Indent `bob capture` clipboard children like other Obsidian sub-bullets

## Problem

`bob capture` renders clipboard (`%`, `%N`, `%header`, `--clip`) values as child bullets with a hard-coded two-space
prefix, and the nested header form with a hard-coded four-space prefix. The single source of that decision is
`rendered_lines` in `src/native/capture_clip.rs:915-926`:

```rust
fn rendered_lines(header: Option<&str>, items: &[String]) -> Vec<String> {
    if let Some(header) = header {
        if items.len() == 1 {
            return vec![format!("  - **{header}:** {}", items[0])];
        }
        let mut lines = vec![format!("  - **{header}:**")];
        lines.extend(items.iter().map(|item| format!("    - {item}")));
        return lines;
    }

    items.iter().map(|item| format!("  - {item}")).collect()
}
```

Every other sub-bullet `bob capture` writes already resolves its whitespace from the note it is writing into.
`README.md:244-245` documents that contract for `@route^block-id` sub-bullet capture: existing child indentation is
copied; otherwise capture uses the note's dominant tab-or-two-space indentation and falls back to a tab.
`plan_sub_bullet_capture` implements it at `src/native/capture.rs:871-881` via `first_child_indentation`,
`dominant_indent_unit` (`src/native/capture.rs:931-946`), and a `\t` fallback.

The clipboard renderer ignores that contract, which produces two concrete defects:

1. **Mixed indentation on sub-bullet captures.** `plan_sub_bullet_capture` (`src/native/capture.rs:882-886`) prefixes
   every line of the capture block with the resolved parent-child indentation, so a tab-indented note gets a
   tab-plus-two-spaces clipboard child. `tests/cli.rs:4682`
   (`capture_sub_bullet_task_ref_recovers_shift_and_nests_clipboard`) asserts exactly this today:

   ```text
   - [?] #task Parent without ID
   \t- new note
   \t  - first
   \t  - second
   ```

2. **Wrong prefix on tab-indented notes generally.** Bryan's vault is mixed. Obsidian's default editor indentation is a
   tab (`~/bob/.obsidian/app.json` sets no `useTab`/`tabSize` override), and hand-authored sub-bullets in notes such as
   `~/bob/cash.md` use tabs, while migrated notes such as `~/bob/mac_inbox.md` are uniformly two-space. A fixed
   two-space prefix is wrong for the first group and right for the second, so no single hard-coded literal is correct.

## Design decision

Resolve the clipboard child indentation unit from the capture's target note and thread it into the clipboard renderer,
reusing the already-documented and already-tested `dominant_indent_unit` → `\t` chain.

This was chosen over simply swapping the `"  "` literal for `"\t"`. A fixed tab would be correct for tab-indented notes
but would newly break two-space notes such as `mac_inbox.md` — the default capture target — mixing a tab child under
two-space siblings. Deriving the unit per note is what "the same whitespace prefix used by other Obsidian sub-bullets"
means everywhere else in this command, keeps `mac_inbox.md` byte-identical to today's behavior, and fixes the tab notes.

Two deliberate scope limits:

- The unit is resolved once per capture from the **route target note** and used for every clipboard line, including all
  entries of a counted `%N` history capture. Per-entry or per-parent resolution buys nothing: all clipboard children of
  one capture hang off one parent line.
- For `@route^block-id` sub-bullet capture the clipboard step is the note-dominant unit applied on top of the base
  indentation `plan_sub_bullet_capture` already computes. Do **not** try to derive the step by subtracting the parent's
  indentation from its first child's indentation; the base indentation is resolved after the clipboard plan is built,
  and note-dominant resolution already yields the right answer for real notes.

## Implementation

### 1. `src/native/capture.rs` — resolve the unit and pass it down

- Add a helper next to `dominant_indent_unit`:

  ```rust
  fn clip_indent_unit(target: &Path) -> String {
      let Ok(contents) = fs::read_to_string(target) else {
          return "\t".to_string();
      };
      dominant_indent_unit(&line_spans(&contents))
          .unwrap_or("\t")
          .to_string()
  }
  ```

  `line_spans` (`src/native/capture.rs:2255`) borrows `contents`, so keep both inside the function and return an owned
  `String`.

  Use `fs::read_to_string(...)` with `let ... else`, **not** `read_target` (`src/native/capture.rs:1272`). A missing or
  unreadable target must silently fall back to `\t` here; the real read with real error reporting still happens later in
  `plan_capture_to_target` / `plan_capture_with_pomodoro_link` / `plan_sub_bullet_capture`. Introducing a new fallible
  read at this point would reorder clipboard-vs-note error precedence, which
  `capture_clip_failures_leave_vault_untouched` (`tests/cli.rs:4019`) covers.

- In `capture()`, move the target resolution above the clipboard planning block. Today `src/native/capture.rs:468-469`
  computes:

  ```rust
  let relative_target = relative_target(parsed.route.as_deref());
  let target = request.bob_dir.join(&relative_target);
  ```

  after the `let clip_plan = match parsed.clip.as_ref() { ... }` block at `src/native/capture.rs:430-461`. Both lines
  depend only on `parsed` and `request`, so hoisting them to just before `let clip_plan = ...` is side-effect free. Then
  compute `let clip_indent = clip_indent_unit(&target);` and pass `&clip_indent` to the three `capture_clip::plan(...)`
  / `capture_clip::plan_history(...)` call sites at `src/native/capture.rs:435`, `:448`, and `:456`.

  Resolve `clip_indent` lazily or unconditionally as preferred, but do not read the target when no clipboard capture was
  requested if that is easy to arrange — a non-clip capture should not gain a filesystem read.

### 2. `src/native/capture_clip.rs` — thread the unit through rendering

Add an `indent: &str` parameter to each of these and forward it; no other logic changes:

- `plan` (`:643`) and `plan_history` (`:658`) — the public entry points.
- `plan_entry` (`:695`).
- `plan_attachments` (`:954`) and `plan_snippet` (`:1158`).
- `inline_output` (`:928`) and `lines_output` (`:939`).
- `rendered_lines` (`:915`) becomes:

  ```rust
  fn rendered_lines(
      header: Option<&str>,
      items: &[String],
      indent: &str,
  ) -> Vec<String> {
      if let Some(header) = header {
          if items.len() == 1 {
              return vec![format!("{indent}- **{header}:** {}", items[0])];
          }
          let mut lines = vec![format!("{indent}- **{header}:**")];
          lines.extend(
              items.iter().map(|item| format!("{indent}{indent}- {item}")),
          );
          return lines;
      }

      items.iter().map(|item| format!("{indent}- {item}")).collect()
  }
  ```

  Note the nested level is `{indent}{indent}`, i.e. one repeat of the resolved unit, not a fixed four spaces.

`ClipOutput.lines` stays `Vec<String>` holding final Markdown, so `capture_block` assembly
(`src/native/capture.rs:462-467`), the human printer (`src/native/capture.rs:2379-2381`), and the JSON `clip` object
(`src/native/capture.rs:541`, `clip: clip_plan.map(|plan| plan.output)`) all keep their current shapes. `plan_history`
keeps flattening `entry.lines` (`:674-677`) unchanged, so every history entry inherits the same unit.

### 3. Tests

All existing assertions stay meaningful; expected strings change according to each fixture's dominant indentation. There
are roughly 29 affected assertions in the `capture_clip.rs` unit tests and about 10 in `tests/cli.rs`.

- **`src/native/capture_clip.rs` unit tests** call `plan` / `rendered_lines` (and friends) directly. Pass an explicit
  unit at each call site. Keep most existing cases on `"  "` so their expectations are unchanged and continue to guard
  the two-space path, and add focused cases that pass `"\t"` to prove `rendered_lines` honors the unit for the
  headerless, single-item-header, multi-item-header, attachment, and snippet shapes. The multi-item-header tab case must
  assert `"\t\t- ..."` for the nested items.

- **`tests/cli.rs` integration tests.** These go through real fixture vaults, so recompute the expected prefix per
  fixture:
  - `capture_clip_marker_composes_with_schedule_routes_bullets_and_pomodoro` (`:3197`) writes `work.md` as
    `"# Work\n## Tasks\n- [ ] #task Existing\n"`, `notes.md` as `"# Notes\n## Ideas\n"`, and `dev.md` as
    `"# Dev\n## Tasks\n"`. None contain an indented line, so `dominant_indent_unit` returns `None` and the unit falls
    back to `\t`. Every `"  - ..."` clipboard expectation in this test — including the `json["clip"]["lines"]` array at
    `:3231-3234` — becomes `"\t- ..."`. The day-file assertion `"## Pomodoros\n- [ ] Current\n  - [[dev#^ship-id]]\n"`
    at `:3295-3296` is the Pomodoro ledger link, **not** a clipboard line, and must stay two spaces.
  - `capture_headerless_clip_marker_renders_under_tasks_and_pomodoros` (`:3299`),
    `capture_flat_clipboard_list_routes_normalized_children` (`:3366`),
    `capture_percent_one_is_an_exact_single_clip_alias` (`:3422`),
    `capture_clip_options_force_or_disable_marker_parsing` (`:3716`), and
    `capture_clip_saves_attachments_snippets_and_reports_dry_run` (`:3893`): inspect each fixture note and update
    expectations the same way.
  - `capture_sub_bullet_task_ref_recovers_shift_and_nests_clipboard` (`:4682`) is the headline fix. Its fixture is
    `"shifted line\n## Tasks\n- [?] #task Parent without ID\nTail\n"` — no indented lines, so the unit is `\t`, and the
    base indentation the sub-bullet planner picks is also `\t`. The expected note becomes:

    ```text
    shifted line
    ## Tasks
    - [?] #task Parent without ID
    \t- new note
    \t\t- first
    \t\t- second
    Tail
    ```

  - `capture_clip_failures_leave_vault_untouched` (`:4019`) must keep passing untouched; if it needs edits, the
    `clip_indent_unit` read was made fallible somewhere it should not have been.

- **New regression coverage.** Add one integration test proving per-note resolution, since that is the behavior this
  plan introduces and nothing else pins it: capture the same clipboard value into two fixture notes in one vault — one
  whose existing content is tab-indented and one whose existing content is two-space-indented — and assert the clipboard
  child matches each note's own prefix. Also add a case for a target note that does not exist yet (fresh
  `mac_inbox.md`), asserting the `\t` fallback.

### 4. Documentation

- `README.md:146-185`: the clipboard rendering examples are fenced `markdown` blocks using two-space children. Update
  them to the tab prefix and add a sentence stating the rule — clipboard children use the target note's dominant
  tab-or-two-space indentation and fall back to a tab, matching the sub-bullet rule already documented at
  `README.md:244-245`. Keep the header example's nested list one further unit deep.
- `src/native/capture.rs:83-87`: extend the `long_about` sentence that currently ends "...become child bullets, with
  source list markers removed" to state the indentation rule, so `bob capture --help` matches the README. Keep the
  existing wrapping style and re-check the rendered help width.
- No new subcommands or options are added, so `sase/memory/cli_rules.md` needs no change beyond keeping the help text
  clear and scannable.

## Verification

```bash
cargo fmt --check
cargo clippy --all-targets --all-features
cargo test
# or, all three:
just all
```

Then confirm the real-world shape by hand from the workspace checkout, in a scratch vault rather than `~/bob`:

```bash
tmp=$(mktemp -d)
printf '# Cash\n## Tasks\n- [ ] #task Parent\n\t- existing child\n' > "$tmp/cash.md"
printf '# Inbox\n- [ ] #task Parent\n  - existing child\n' > "$tmp/mac_inbox.md"
printf '#!/bin/sh\nprintf "copied value\\n"\n' > "$tmp/clip"; chmod +x "$tmp/clip"
BOB_CLIPBOARD_CMD="$tmp/clip" cargo run -q -- capture -b "$tmp" -- 'tab note %' '@cash'
BOB_CLIPBOARD_CMD="$tmp/clip" cargo run -q -- capture -b "$tmp" -- 'space note %'
cat -A "$tmp/cash.md" "$tmp/mac_inbox.md"
```

`cash.md` must gain a `^I`-prefixed clipboard child and `mac_inbox.md` a two-space one.

## Risks and edge cases

- **Behavior change on notes with no indentation.** A target note that contains no indented line at all now yields `\t`
  instead of `  `. This is intentional and matches the documented sub-bullet fallback, but it changes output for
  brand-new notes and for flat notes. Call it out in the commit message.
- **`mac_inbox.md` regression risk.** The default capture target is space-dominant in the real vault, so its output must
  remain two-space. The new per-note regression test is the guard.
- **Ledger links are out of scope.** `plan_pomodoro_link` keeps its own resolution chain (`child_bullet_indentation` →
  `nearby_child_bullet_indentation` → `"  "`, `src/native/capture.rs:1055-1064`). Do not change it in this plan; it
  writes into the daily note, not under the captured task, and it already inspects that note.
- **`dominant_indent_unit` counts every indented line**, including continuation lines and fenced code, not just list
  items. That is the existing documented heuristic; reuse it rather than inventing a second one, so clipboard children
  and `@route^block-id` sub-bullets never disagree.
- **Human output.** `src/native/capture.rs:2379-2381` prints each clip line behind a two-space display indent, so a
  tab-indented line renders as two spaces then a literal tab. That is the true file content and is acceptable; do not
  expand tabs for display.
