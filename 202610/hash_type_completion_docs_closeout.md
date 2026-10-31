---
tier: tale
size: small
title: Fix stale `@route+id#` completion docs and help, then close epic bob-cli-ep
goal:
  Every capture-complete doc and help line describes the no-body `@route+id#` and
  `@route^id#` as task-type completion, and epic bob-cli-ep is closed with its plan
  marked done.
proposed_by: bbugyi200.apollo.bob-cli-ep.land
proposed_with: claude/opus
bead: bob-cli-ep
create_time: 2026-10-31 10:05:24
status: wip
---

- **PARENT:**
  [202610/capture_hash_type_marker.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/capture_hash_type_marker.md)
- **BEAD:**
  [bob-cli-ep](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-ep/README.md)

# Plan: Fix stale `@route+id#` completion docs and help, then close `bob-cli-ep`

## Context

Epic `bob-cli-ep` (plan `plan:202610/capture_hash_type_marker.md`) replaced the
`t:<type>` capture marker with `#<type>`. Both phases are closed and verified:

- bob-cli commits `4a95e5b7` and `d1dd23e2`;
- bob-mac-capture commit `9d023bb`.

The epic changed what a `#` means on a marker-only `@route+block-id` (no body text, no
authored children). That `#` is now always the task type, and the `task_type` completion
context owns it. With body text, `@route+id#…` is still the task-section selector.

`@route^block-id#` also became a `task_type` completion context, including the empty
`#`.

The land review found the grammar, writers, completion engine, and most docs correct. A
few completion descriptions still say the no-body `@route+id#` opens Pomodoro-name
completion, or never mention the attached forms. Fixing them is the only remaining epic
work. After the fix, this tale closes the epic.

The epic's reviewer decisions still bind:

- `bare_hash_menu = letter`: a bare, unattached `#` never opens the type menu;
- `retire_t = prose`: `t:` is ordinary text;
- `memory_record = no`: edit no `sase/memory/` note.

Do not change any grammar or runtime behavior.

## Before editing

Read the docs rules:
`sase memory read docs.md -r "Editing docs/ and clap help for stale #<type> completion text"`.
Follow its rules for generated CLI reference pages.

## Changes

### 1. `docs/capture.md` capture-complete prose

**Lines ~6460–6470.** The Pomodoro-name sentence currently ends with "`@route+id#prefix`
or a bare `@route+id#` while the item has no body text".

- Remove that clause.
- Add a sentence saying a marker-only `@route+id#` (no body text, no authored children)
  and `@route^id#` complete `task_type` instead.
- Keep the task-section sentence. It already says "once the item has body text".

**Lines ~6528–6535.** The `pomodoro_name` task-aware suggestion list names
"`@route:block-id#`, `^route:block-id#`, or `@route+id#`". Drop `@route+id#` from that
list, because it no longer opens `pomodoro_name`.

**Line ~3161.** The literal-positions list "named starts, link suffixes
(`@route:id#"…"`, `@route+id#"…"`, `^route:id#"…"`), project-note bullets, toggles, and
close positions" still lists the retired named-toggle Pomodoro form. Fix it:

- remove `@route+id#"…"` and "toggles" from this Pomodoro-name list;
- if the sentence needs it, note that `#"NAME"` type literals follow the same quoting
  rule (see `### Typing tasks with #<type>`).

Grep the rest of `docs/capture.md` for any other claim that a no-body `@route+id#` opens
Pomodoro-name completion or names a Pomodoro, and fix it the same way.

### 2. `src/native/capture_complete/cli.rs` long help

**Around line 118.** The sentence "Task-section completion covers '@route+id#prefix' and
a bare '@route+id#'" needs the qualifier "once the item has body text".

**Around line 214.** The paragraph "A marker-position `#<type>` token at either line
edge completes `task_type` …" must also say:

- the attached `@route^block-id#…` suffix completes `task_type` (subject `new`);
- a marker-only `@route+block-id#…` suffix completes `task_type` (subject `existing`),
  including an empty `#`;
- an attached suffix replacement never adds a space;
- a bare unattached `#` returns no `task_type` context.

Then run `just docs-gen` so `docs/reference/cli/capture-complete.md` regenerates.

### 3. `docs/completion.md` (~line 456)

The `#` completes task types bullet mentions marker-only `@route+block-id#`. Add that
`@route^block-id#` (the new-task attached form, including an empty `#`) uses the same
context.

### 4. `src/native/capture_parse.rs` help (optional check)

Check the help near line 174 for the same staleness. The incomplete list near line 197
already treats `'@route+id#'` generically. Change it only if it is actually wrong. If
you change it, rerun `just docs-gen`.

## Verify

- Run `just docs-gen`, then `just docs-check`, then `just check`. All must pass.
- `git grep -n "bare \`@route+id#\` while the item" docs src` finds nothing.

## Commit

- Subject: `docs(capture): fix stale @route+id# completion help after #<type>`.
- Do not edit the `Cargo.toml` version or `CHANGELOG.md`.

## Final step: close out epic `bob-cli-ep` (same turn)

Do these steps in order, after the edits and checks above. None of them waits for this
tale's commit, push, or CI.

1. Run `sase bead epic-symbols bob-cli-ep`.
   - At land review it reported no entries.
   - If any entries appear now, resolve each one (wire it up, privatize it, add a
     non-test pragma, or delete it per the Symvision epic-whitelist policy). Re-key one
     only when a still-open later bead needs it.
2. Close the epic:

   ```bash
   sase bead close bob-cli-ep --note "Verified phases bob-cli-ep.1 (4a95e5b7, d1dd23e2) and bob-cli-ep.2 (bob-mac-capture 9d023bb) against the plan and DECISIONS (bare_hash_menu=letter, retire_t=prose, memory_record=no); just check passed; capture/capture-parse/capture-complete spot checks matched the contract. No PROPOSED FOLLOW-UP notes, so nothing filed. Post-epic commits needed no integration. Lander tale fixed the stale @route+id# / @route^id# completion docs and help; just docs-check and just check passed."
   ```

   - Never use `--force` to make the close succeed.
   - If the close refuses because `--epic-symbol` entries remain, clean them up and
     close again.

3. Run `just symvision` and confirm the whitelist is clean.
4. Set `status: done` in the YAML frontmatter of the epic's plan file. That file is the
   PLAN path printed by `sase bead read bob-cli-ep -r "Need the plan path for closeout"`
   (`plan:202610/capture_hash_type_marker.md`).
5. `bob-cli-ep` has no `parent_bead`, so there is no parent to handle.
