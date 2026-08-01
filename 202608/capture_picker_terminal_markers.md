---
tier: tale
title: Make Hammerspoon capture pickers compose with clipboard markers
goal:
  Interactive Bob capture routes preserve terminal clipboard and schedule markers while opening every applicable picker
  flow.
proposed_by: bbugyi200.athena.qz
create_time: 2026-08-01 07:18:27
status: wip
---

- **PROMPT:** [202608/prompts/capture_picker_terminal_markers.md](prompts/capture_picker_terminal_markers.md)

# Make Hammerspoon capture pickers compose with clipboard markers

## Goal

Make the `cmd+shift+ctrl+i` Hammerspoon capture panel honor Bob's terminal clipboard markers when they appear on either
side of every interactive `@<route><suffix>` form. In particular, submitting text such as `Review this @sase# %` must
open the section picker for `sase.md`, preserve `%` for the eventual capture, and attach the live clipboard beneath the
bullet chosen by the user. The same composition must work for bare/target-selection `@` forms, section `#` forms,
Pomodoro `:` forms, and sub-bullet `^` forms, including `%<positive integer>` and `%<header>` markers.

## Diagnosis and scope

- `bob-cli/src/native/capture.rs` already treats clipboard and schedule tokens as orthogonal terminal markers. It can
  find a trailing route token on either side of those markers, preserves its one-marker-per-kind stopping rules, and has
  Rust unit/integration coverage for ordinary, section, Pomodoro, and sub-bullet routes. No native capture-parser change
  is currently indicated.
- The linked `chezmoi` repo owns the Hammerspoon panel. Its pure request model in
  `home/dot_hammerspoon/task_capture.lua` only examines the final whitespace-delimited token. Consequently,
  `Body % @sase#` enters section selection, while `Body @sase# %` falls through to an immediate CLI invocation. The
  latter does capture successfully at the CLI layer, but the panel never presents the requested section UI. The same
  asymmetry applies to incomplete `:`, `^`, and bare `@` picker forms.
- Keep Bob's native parser authoritative for actually interpreting and rendering clipboard/schedule markers. The
  Hammerspoon model should recognize just enough of the terminal-marker region to locate an interactive `@` token,
  remove only that UI token, and forward the other tokens unchanged to `bob capture` after picker completion.
- This is a `tale`: the parser adjustment, Hammerspoon flow verification, regression tests, and documentation are one
  tightly coupled change that one coding agent can implement and validate without phase handoffs.

## Implementation

1. In the linked `chezmoi` repo, update `home/dot_hammerspoon/task_capture.lua` so `M.parse` can locate the candidate
   interactive `@` token immediately before a valid trailing terminal-marker sequence, rather than requiring it to be
   the final token.
   - Model Bob's supported clipboard spellings: bare `%`, positive numeric history counts (including accepted leading
     zeroes), and nonnumeric headers containing only letters, digits, `_`, or `-`. Do not skip over `%0`, overflowing or
     otherwise invalid numeric counts, punctuation-bearing headers, mid-body `%` text, or a duplicate marker that Bob
     would treat as the stopping boundary.
   - Include valid lowercase `s:<N>` schedule tokens in the terminal region because Bob permits schedule and clipboard
     markers together in either order. Track marker kinds independently so at most one clipboard marker and one schedule
     marker are crossed while looking for the interactive route token.
   - Preserve every crossed token, in its original spelling and relative order, in the request's `text`. Only the
     interactive `@...` token is consumed by Hammerspoon. Existing picker/finalizer paths will then canonicalize the
     selected route or task while the native command remains responsible for extracting `%...` and `s:<N>`.
   - Apply the behavior uniformly to `@`, `@#`, `@#prefix`, `@route#`, the four canonical/legacy Pomodoro picker forms,
     and the four sub-bullet picker forms. Continue leaving plain `@route`, explicit prefix-bearing `@route#prefix`,
     mid-text route-looking tokens, malformed special markers, and unsupported terminal regions to `bob capture` as they
     are today.

2. Review the call paths in `home/dot_hammerspoon/init.lua` and adjust only where necessary to keep the preserved marker
   text intact through every asynchronous stage:
   - target selection and forced ordinary-task capture;
   - target selection followed by exact section selection, including the fewer-than-two-sections fallback;
   - Pomodoro block-ID prompting/finalization;
   - sub-bullet task selection/finalization. Ensure cancellations still refocus the original prompt, failures retain
     staged values, and the final shell command continues passing user text through positional parameters rather than
     interpolation. Update the adjacent grammar comments so they describe terminal `%...`/`s:<N>` composition and the
     fact that the route token need not be last.

3. Extend `chezmoi/tests/hammerspoon/task_capture_spec.lua` with table-driven regression coverage for the pure request
   model.
   - For each interactive route family (`@`, `@#`, `@#prefix`, `@route#`, complete/incomplete `:`, and
     complete/incomplete `^`), cover a clipboard marker both before and after the route token.
   - Exercise bare, positive-count, and named-header clipboard forms, plus representative combinations with `s:<N>` in
     both marker orders. Assert the correct picker mode/needs flags and that `request.text` retains the non-route
     terminal markers exactly once.
   - Assert `finalize` and `finalize_sub_bullet` produce canonical leading route syntax while retaining the clipboard
     and schedule tokens for Bob to parse.
   - Add negative cases for `%0`, invalid `%...` tokens, duplicates/stopping boundaries, mid-text markers, plain
     `@route`, and explicit `@route#prefix` so the new scan does not broaden picker activation beyond Bob's terminal
     grammar or the panel's intentional interactive shorthands.

4. In `bob-cli/README.md`, update the Hammerspoon panel section to document that supported `%`, `%N`, `%header`, and
   `s:<N>` terminal markers may surround the interactive `@...` token and survive target/section/task selection. Include
   `@sase# %` as the concrete section-picker example and clarify that clipboard interpretation still belongs to
   `bob capture`.

## Validation

Run the focused checks first, then repository-level checks appropriate to the touched files:

1. In `chezmoi`, run `busted ./tests/hammerspoon` (or `just test-hammerspoon`) and confirm the complete table-driven
   marker/route matrix passes.
2. In `chezmoi`, run
   `stylua --check home/dot_hammerspoon/task_capture.lua home/dot_hammerspoon/init.lua tests/hammerspoon/task_capture_spec.lua`
   if supported by the installed formatter; otherwise run the repository's Lua formatter and verify only the intended
   Lua files changed. Run the repository's relevant Lua/static checks when they cover Hammerspoon sources.
3. In `bob-cli`, run `cargo test native::capture` (or the exact focused capture test filter supported by the suite) to
   re-confirm native terminal-marker composition, then run the capture-related CLI integration tests that cover
   clipboard markers and picker discovery commands.
4. Run `just check` in each touched repository when practical. If a broad unrelated check fails, record the exact
   command and failure without weakening the focused regression coverage.
5. Inspect both worktrees and verify the change set is limited to the Hammerspoon request model/integration comments,
   its tests, and Bob's capture documentation; do not modify vault notes or generated/deployed Hammerspoon files.

## Acceptance criteria

- `Body @sase# %` opens the section flow for `sase.md`; selecting a heading submits an exact-section bullet capture and
  the live clipboard is attached to that bullet.
- Moving `%`, `%N`, or `%header` to either side of any supported interactive `@` suffix form does not change which
  picker/prompt stages appear, and the marker reaches `bob capture` exactly once.
- Clipboard markers continue composing with `s:<N>` in either order without changing scheduled-capture behavior.
- Literal/invalid/duplicate marker cases retain the established native and Hammerspoon stopping behavior, and plain or
  noninteractive route tokens do not unexpectedly open a picker.
- Existing cancellation, retry-state retention, safe shell argument passing, and native capture tests remain green.
