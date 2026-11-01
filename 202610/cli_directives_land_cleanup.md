---
tier: tale
size: medium
title: 'Finish cli-directives landing: dead clip-control plumbing, stale %l/%c prose,
  %n detail parity, then close bob-cli-ey.1.1 and bob-cli-ey.1'
goal: 'The cli-directives epic lands clean: no dead parse_clip_markers or _with_clip_control
  plumbing survives, every doc and doc comment teaches `URL %l` and `%c` instead of
  retired listen and clip spellings, the `%n` completion detail names the Pomodoro
  that `%n` execution writes to, and epic bob-cli-ey.1.1 plus its parent phase bob-cli-ey.1
  are closed with their verification recorded.'
proposed_by: bbugyi200.apollo.bob-cli-ey.1.1.land
proposed_with: claude/opus
bead: bob-cli-ey.1.1
status: done
---

- **PARENT:**
  [202610/cli_directives_split.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/cli_directives_split.md)
- **BEAD:**
  [bob-cli-ey.1.1](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-ey/bob-cli-ey.1.1.md)

# Plan: finish the `cli-directives` landing

## Context

Epic bead `bob-cli-ey.1.1` (plan `plan:202610/cli_directives_split.md`) split phase
`cli-directives` (bead `bob-cli-ey.1`) of epic `bob-cli-ey` ("Capture drawers and `%`
directives", `plan:202610/capture_drawers_and_directives.md`) into six phases. All six
are closed and landed on master:

| Phase              | Commit     | Subject                                                                                      |
| ------------------ | ---------- | -------------------------------------------------------------------------------------------- |
| `bob-cli-ey.1.1.1` | `36c48075` | `feat(capture)!: remove @@route+block-id`                                                    |
| `bob-cli-ey.1.1.2` | `6ae5df00` | `feat(capture)!: add %c/%c<N> directive lexer and split clipboard reading`                   |
| `bob-cli-ey.1.1.3` | `4c011cd7` | `feat(capture)!: teach %n and %l directives, retire trailing # and URL %/@`                  |
| `bob-cli-ey.1.1.4` | `d7f683a1` | `feat(capture)!: treat a bare edge # as an incomplete type marker`                           |
| `bob-cli-ey.1.1.5` | `95278934` | `feat(capture)!: frame %t templates, retire @@template:, @@@, and the @@ template candidate` |
| `bob-cli-ey.1.1.6` | `b290d833` | `feat(capture): add directive completion and grammar revision`                               |

The land agent verified the epic at `b290d833` and found it functionally complete:

- `just check` passed (4249 lib and 2595 CLI tests), and so did `just docs-check`.
- The parent plan's done-when `git grep` leaves only allowed hits.
- The scratch-vault smokes all behave to contract: `jot idea %n`, `Build output %c3`
  through `BOB_CLIPBOARD_CMD` and `BOB_CLIPBOARD_HISTORY_CMD`, the `--no-clip`
  `not_read` dry run and the `clip_not_read` refusal, `{0} @job+search %t` plus rows,
  `https://example.com/essay %l`, bare `#` and bare `%` with
  `initial_selection: "none"`, `grammar: 2`, zsh `%<TAB>` and `#<TAB>`, and every lexer
  edge case.
- The glossary strands and the shelf-bead notes are done.
- No commit edited the `Cargo.toml` version or `CHANGELOG.md`.

Three leftovers remain. Each was introduced by this epic and is in scope for its
landing:

1. Phase 2 deleted `CaptureParseOptions.parse_clip_markers`, but not the internal
   plumbing behind it. The plan required deleting that too.
2. Phase 3 left stale `URL %` / `URL @` listen prose in docs and doc comments. That
   includes the `ref_listen` doc comment, which the plan explicitly required to say
   `%l`. One docs example no longer even runs.
3. Phase 6's `%n` completion detail can name a different Pomodoro from the one that `%n`
   execution actually writes to.

This tale fixes all three, then closes `bob-cli-ey.1.1` and its parent phase
`bob-cli-ey.1`.

Ground rules from the parent plan still apply:

- **No compatibility.** Delete retired code and prose outright, and never write copy
  that names a retired spelling.
- **Release train.** Never edit the `Cargo.toml` version or `CHANGELOG.md`.
- **Memory.** Edit no memory.
- **CLI rules.** Run `sase memory read cli_rules.md` before touching CLI help.
- **Docs rules.** Run `sase memory read docs.md` before touching docs.

## 1. Delete the dead clip-control plumbing (no behavior change)

Every caller passes `true` for the `parse_clip_markers` / `parse_clip` bool, and the
leaf uses already ignore it (`markers.rs` `_parse_clip_markers`, `dependencies.rs`
`is_trailing_transparent(text, _parse_clip)`).

**Remove the bool parameter** from every function in `src/native/capture_language/` that
takes it, along with every argument that passes it:

- `draft.rs`: `split_capture_draft_with_clip_control`,
  `split_items_from_item_lines_with_clip_control`, `push_capture_item`,
  `session_chain_tokens`, and the helpers around lines 270–390.
- `line.rs`: `resolve_line` and `resolve_line_inner`.
- `markers.rs`: `extract_terminal_markers`.
- `type_marker.rs`: `scan_line_with_positions`, `marker_only_task_toggle`,
  `collect_item_with_clip_control`, `scan_line_with_clip_control`,
  `masked_line_with_clip_control`, and `masked_tokens_with_clip_control`.
- `dependencies.rs`: `is_trailing_transparent` and every `parse_clip: bool` parameter,
  about 24 sites.
- Every caller in `item.rs`, `editor_parse.rs`, `completion.rs`, `directive.rs`, and the
  tests.

**Collapse the wrapper pairs.** Each pair is a plain function that only forwards `true`
to its `_with_clip_control` twin. Merge the twin's body into the plain name and delete
the twin:

- `split_capture_draft` / `split_capture_draft_with_clip_control`
- `split_items_from_item_lines` / `split_items_from_item_lines_with_clip_control`
- `type_marker::scan_line` / `scan_line_with_clip_control`
- `type_marker::masked_line` / `masked_line_with_clip_control`

Give the remaining twins a plain name too: `collect_item_with_clip_control` becomes
`type_marker::collect_item`, and `masked_tokens_with_clip_control` becomes
`type_marker::masked_tokens`.

**Rename the option-taking parsers** to say what they take. The plain names
`parse_capture_text` / `parse_capture_draft` already exist as test helpers in
`capture/sections.rs` and `capture/tests/mod.rs`, so use these names:

- `parse_capture_text_with_clip_control` → `parse_capture_text_with_options`
- `parse_capture_draft_with_clip_control` → `parse_capture_draft_with_options`
- `parse_capture_draft_with_clip_control_coded` →
  `parse_capture_draft_with_options_coded`

Apply the renames everywhere these names appear:

- the language crate and the `mod.rs` re-exports;
- the `capture/sections.rs` wrappers and `capture/plan.rs`;
- every test module
  (`capture_language/tests/{chain,draft,editor_modes,editor_spans,globals,goto,grammar,mod,ref_grammar,type_marker}.rs`
  and `capture/tests`);
- the doc comments that cite the old names (`editor_classify.rs` ~33, `editor_model.rs`
  ~541, `editor_parse.rs` ~237).

**Done when** this returns nothing:

```bash
git grep -n -w -e parse_clip_markers -e _parse_clip_markers -e parse_clip -e _parse_clip -- src
git grep -n clip_control -- src
```

Also update comments that still describe the deleted control. For example,
`dependencies.rs` ~595 says "clipboard markers": reword it to say `%` directives where
it means them.

## 2. Finish the `%l` listen and `%c` prose sweep

The code is already correct: only exact `URL %l` listens, a bare `%` is always an
incomplete directive, and `URL @` stays a task. Fix the prose to match. Line numbers are
as of `b290d833`.

**Doc comments.** Each one must say that `%l` requested companion audio, and must drop
the "standalone `%`" and "compatible `@` alias" wording:

- `src/native/capture_language/model.rs` ~66–69 (the `ref_listen` / `CaptureKind::Ref`
  comment; the parent plan explicitly requires `%l` here);
- `src/native/capture/output.rs` ~22–26;
- `src/native/gkeep/plan.rs` ~114, the `url_only_intent` doc ~310–322, the body-shape
  comment ~341, and the `url_only_body_tokens` doc ~360. The accepted shapes are `URL`,
  `URL @route`, `URL %l`, and `URL %l @route`;
- `src/native/highlights_ref/ingest.rs` ~48;
- `src/native/ref_jobs/spool.rs` ~61;
- `src/native/capture_parse.rs` ~1299: "a `URL %` row" becomes "a `URL %l` row".

**Docs:**

- `docs/capture.md`:
  - ~154: the `%` row. A bare `%` is always incomplete, so drop "except on a bare URL
    where `%l` requests companion audio".
  - ~84–85: drop "there is no `@` alias" and "(`%l` is the only listen spelling)". Those
    phrases name retired spellings.
  - ~980–992, the reading-queue listen section. Rewrite it for `%l`:
    - Teach "append `%l` immediately after the URL", with the example
      `https://example.blog.com/somepost %l`.
    - Delete "Exact standalone `@` is still a compatible listen spelling".
    - Keep the rest of the contract as written: `URL %l @route`, `@route URL %l`, the
      placement rules, the `-R` / `--clip` notes, and the route-completion sentence.
    - Link the "Directives (`%`)" section.
  - ~4529–4531: the quoted error must match the code:
    `project-note capture cannot be combined with %c`.
  - ~6116: "A bare `%` span that is not listen" becomes "A bare `%` span".
- `docs/reference/capture-syntax.md` ~322–329, the `URL %l` section:
  - Fix the broken sentence "Exact there is no `@` alias".
  - Change the example to
    `bob capture --dry-run 'https://example.com/article %l @work'`. Today it uses a bare
    `%`, which now fails as an incomplete directive.
  - Rerun the example against a scratch vault, and update the Output block if it
    differs.
- `docs/getting-started.md` ~371: "a `%` listen marker" becomes "a `%l` listen
  directive".
- `docs/gkeep.md` ~224–234: say "an optional `%l` listen directive". Delete "with `@` as
  a compatible alias".
- `docs/highlights-create.md` ~840–842: "The `URL %l` directive in `bob capture` and
  `bob gkeep pull` clips like …". Delete the `URL @` alias clause.
- `docs/ref-jobs.md` ~106: "`listen` is true when `%l` requested companion audio …".

**Then sweep for anything this list missed** and fix every real hit the same way:

```bash
git grep -n -I -i -E "compatible (listen|alias)|standalone .?%|standalone .?@ |listen marker|URL %([^l0-9]|$)|% clipboard|clipboard marker" -- docs src README.md ':!docs/develop/decisions.md'
```

Leave these alone:

- generic mentions of `%` as the directive sigil;
- the unrelated Task Link "trailing `#` move-only marker" in `docs/plan.md`,
  `docs/today-marks.md`, and `docs/getting-started.md` ~152;
- narrow negative tests;
- the generated `docs/develop/decisions.md`, which comes from an immutable record.

If any clap help text changed, run `just docs-gen`.

## 3. Make the `%n` completion detail name the Pomodoro execution would pick

**The bug.** `src/native/capture_complete/directives.rs` `pomodoro_note_target` picks
its entry from `capture_pomodoros::scan`. The scan sets `time_range` only for a
_leading_ range (`(**HHMM-HHMM**) — NAME`).

Execution works differently. `%n` runs through
`PomodoroSelection::CurrentOrLastCompleted` in `src/native/capture/pomodoro_insert.rs`
`insert_pomodoro_child_block`. That path classifies column-0, non-fenced ledger lines
with `pomodoro::open_ledger_task`, `pomodoro::completed_ledger_task`, and
`pomodoro::task_time_range`, which finds a range anywhere in the line.

**Reproduction.** Use this day note:

```text
## Pomodoros

- [x] (**1330-1355** [t:: 25m]) — PLAN
- [ ] BUILD (1400-1425)
```

`bob capture --dry-run 'jot %n'` files under `BUILD (1400-1425)`, but the `%n` row says
`→ 13:30–13:55 PLAN · last completed`. With only the BUILD line, the row says
`→ unnamed · planned`.

**The fix.** Extract the line-selection step for the default selections out of
`insert_pomodoro_child_block` into one `pub(crate)` helper that both callers use. The
default selections are `CurrentOrFuture` and `CurrentOrLastCompleted`. The helper:

- returns the selected 0-based line index plus which branch chose it (timed open, last
  completed, or first open);
- returns the same errors as today: no Pomodoros section, "multiple open timed
  Pomodoros", and `no_entry_message`.

Then:

- `insert_pomodoro_child_block` calls the helper. Its behavior and messages must not
  change.
- `pomodoro_note_target` calls the helper too, and maps each error to today's disabled
  reasons.
- Label the row from the `capture_pomodoros::scan` entry at that line when the entry
  parsed a time range or name, using the existing `pomodoro_target_label`.
- Otherwise, label it `→ <ledger task text> · <running|last completed|planned>`, taking
  the state from the selection branch.

**Tests** (in `capture_complete/tests/directives.rs`):

- The reproduction above names `BUILD (1400-1425)` as running.
- A mixed-format ledger with two open timed entries disables `%n` with the multiple-open
  reason.
- The existing canonical-format test still yields `→ 08:30–08:55 PLAN · last completed`.

## 4. Verify

- Run `just check`. Do not run `just check-full`.
- Run `just docs-check`.
- Smoke against a scratch vault: `bob capture-parse` / `capture-complete` on
  `https://example.com/essay %l`, `jot idea %` (the `%n` detail), bare `#`, and bare
  `%`. Run the corrected `docs/reference/capture-syntax.md` example with
  `bob capture --dry-run`.
- Rerun the parent's done-when grep, outside `CHANGELOG.md` and `sase/`. It should leave
  only the allowed hits listed in step 5's close note:

  ```bash
  git grep -n -e '@@template' -e '@@@' -e start_template -e template_header -e TemplateHeader -e global_sub_bullet -e GlobalSubBullet -e parse_clip_token -e is_pomodoro_note_marker -e is_standalone_listen_marker -e 'clip=HEADER'
  ```

- Commit as `refactor(capture): finish cli-directives cleanup`. It is not breaking: no
  grammar, JSON, or CLI contract changes. Do not edit the `Cargo.toml` version or
  `CHANGELOG.md`.

## 5. Close out epic `bob-cli-ey.1.1`, then its parent phase `bob-cli-ey.1`

Do this in the same turn as the code. Do not wait for this tale's commit, push, or CI.
Never use `--force` to make a close succeed.

1. **Retire epic-symbol entries.** Run `sase bead epic-symbols bob-cli-ey.1.1`. It
   listed none at planning time. For each entry it now lists, either resolve the symbol
   (wire it up, privatize it, add a non-test pragma, or delete it) or re-key it to a
   still-open bead that needs the exemption.

2. **Close the epic:**

   ```bash
   sase bead close bob-cli-ey.1.1 --note "<verification>"
   ```

   The note states:
   - All six phases (`bob-cli-ey.1.1.1`–`.6`) are closed. Their commits are `36c48075`,
     `6ae5df00`, `4c011cd7`, `d7f683a1`, `95278934` (all `feat(capture)!:` with
     `BREAKING CHANGE:` footers), and `b290d833` (`feat(capture):`).
   - There are no `Cargo.toml` version or `CHANGELOG.md` edits.
   - `just check` and `just docs-check` pass, giving this tale's results.
   - In the done-when grep, the only remaining hits are allowed:
     - the generated `docs/develop/decisions.md` (immutable record);
     - Mermaid `@@@` in `scripts/web_clip/web_clip_adapter.py`;
     - `-p @@@` in `tests/cli/ref_library/today.rs`;
     - narrow negative tests.
   - The scratch-vault smokes (listed in Context) pass.
   - The Capture and Bulk Capture glossary strands match parent §7.
   - `%t` notes are on `bob-cli-82`, `83`, `8h`, `8i`, and `8b`.
   - Integration: the commits that landed in parallel left no retired forms and need
     none of the new machinery. Those are the project-note offers (`9a30ec4c`,
     `e9b9cd1d`, `e8dd3b65`, `42dd92bc`, `86a63e84`), gkeep status (`efe2ebe0`,
     `17702dca`), and the glossary update (`e2838ac3`).
   - This tale deleted the dead `parse_clip_markers` / `_with_clip_control` plumbing,
     finished the `%l` / `%c` prose sweep, and aligned the `%n` detail with execution.
   - Follow-up triage:
     - The `bob-cli-ey.1.1.2` Mac-docs proposal went to epic `bob-cli-ey` as a
       DISCOVERED ISSUE for `bob-cli-ey.3`.
     - The `bob-cli-ey.1.1.6` grep-review proposal was declined: all hits are allowed.

3. **Run `just symvision`** if the recipe exists. bob-cli had none at planning time; if
   it is still missing, say so in your final response.

4. **Mark the plan done.** Set `status: done` in the frontmatter of
   `plan:202610/cli_directives_split.md`. That is the PLAN path
   `sase bead read bob-cli-ey.1.1` shows.

5. **Close the parent phase.** `bob-cli-ey.1` is a **phase** bead of epic `bob-cli-ey`.
   Confirm that its scope (the parent plan's §1 contract and its §8 "Phase
   `cli-directives`" list) is met, which the steps above verify. Then:
   - Run `sase bead epic-symbols bob-cli-ey.1`, which listed none at planning time, and
     resolve or re-key anything it lists.
   - Close it:

     ```bash
     sase bead close bob-cli-ey.1 --note "<what you verified>"
     ```

   - Add a note:

     ```bash
     sase bead note bob-cli-ey.1 "mac-directives (bob-cli-ey.3) may now start: bob emits grammar: 2 in capture-parse and capture-complete."
     ```

   Do **not** close epic `bob-cli-ey`, and do not mark its plan file done. Its own land
   agent owns both.
