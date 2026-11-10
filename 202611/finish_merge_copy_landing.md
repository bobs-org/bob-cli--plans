---
tier: tale
size: medium
title: Preserve code text in merge copies and finish bob-cli-je
goal:
  Preserve literal code in copied task notes, verify archive integration, correct the
  landing-card docs, and close bob-cli-je.
proposed_by: bbugyi200.athena.bob-cli-je.land
proposed_with: codex/gpt-6-astra
bead: bob-cli-je
status: done
---

- **PARENT:**
  [202611/merge_copies_source_notes.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/merge_copies_source_notes.md)
- **BEAD:**
  [bob-cli-je](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-je/README.md)
- **AGENTS:**
  - [bbugyi200.athena.bob-cli-je.land](https://github.com/bobs-org/bob-cli--agents/blob/main/sessions/bbugyi200.athena.bob-cli-je.land.md)
- **COMMITS:**
  - [2914728](https://github.com/bobs-org/bob-mac-capture/commit/29147289ca584953ce7f8cfd29b5ed60a5380f76)
    — docs(capture-transfers): merged sources keep notes, moves keep removed-run fold

# Preserve code text in merge copies and finish bob-cli-je

Finish the remaining work in epic `bob-cli-je`, then complete its landing in this tale's
coding turn. There is no separate land agent for this tale. Do not wait for this work's
commit SHA, push, or CI result before closing the epic: verified code and the epic close
normally land in the same turn through `/sase_final`.

## Context and completed verification

The approved plan is `plan:202611/merge_copies_source_notes.md`. Read it through
`sase artifact read`, and read `bob-cli-je` and its three closed children through
`sase bead read`. The initial lander reviewed every child note, the actual source, and
these commits:

- bob-cli `076fdc48` (`bob-cli-je.1`): destination copies authored bodies; originals
  stay on cancelled sources; copies strip identities; local links name the source.
- bob-plugins `f06c7e8` (`bob-cli-je.2`): navigation strip says `copies N notes`; built
  entrypoint and docs agree. Build check and 96 navigation merge/Links-stage tests
  passed during landing. The phase already synced the plugin.
- bob-mac-capture `a5a0ffb` (`bob-cli-je.3`): source details and accessibility text say
  `title only`, `1 note copied`, or `N notes copied`. The phase reports 2395 portable
  tests passing. Existing macOS watchdog failures on clean base and phase head are
  tracked separately, not claimed green.

The epic's DECISIONS authorized only `glossary:task-merge` and `glossary:task-relation`
memory corrections; they are complete. The decision-record proposal was declined. This
tale needs no memory changes. Preserve schema version 1 and JSON keys
`moved_lines`/`moved_notes`, dependent repointing, offers, retirement, idempotency, and
task move/split semantics. Do not change versions or CHANGELOG.md.

Integration was reviewed through bob-cli `8728fe28` (also current origin/master),
bob-plugins `f06c7e8`, and bob-mac-capture `a5a0ffb`. Changes since the epic began
include notebook-route composition `3f8c49d4`, alias-tip documentation `b5196709`,
archive-reader migration `0996111c`, metadata-bound archive repair `2327590f`, and dot
separator rewrite `8728fe28`. Plugin notebook catalog/header commits `56b3604`/`32a8545`
and Mac alias lifecycle/dot commits `542a37d`/`5a28892` were also reviewed. No new merge
mode is needed: notebook transfer refusals and dot transfer guards remain valid. Archive
integration needs the specific retained-note regression below. Recheck any newer drift
before close.

## Remaining defects

1. `src/native/task_merge/payload.rs::strip_copied_id_fields` calls
   `projects::edits::remove_all_inline_fields` on every line with `#task`. That helper
   uses `projects/tags.rs::inline_field_spans`, which does not skip inline code. Running
   the exact extracted production helpers on `- [x] #task Nested \`[id:: literal]\`
   [id:: src__nested]`produces`- [x] #task Nested
   \`\``. The live identity must disappear, but the literal code must survive. A plain authored note containing `#task`
   must not be treated as a task row merely because of that token.
2. The new `copied_fence_marker`/`copied_fence_closes` scanner remembers the fence
   character but loses its length. Its extracted helpers report that a triple backtick
   closes a four-backtick fence. The same character-only mechanism is used by
   `task_move/transplant.rs::fragment_hidden_line_indexes`, introduced by this epic for
   link repair. Nested/indented code after the short inner delimiter can therefore be
   treated as live text. The canonical `markdown::MarkdownFence` helpers already retain
   opening length; raw fragment indentation requires care. The extraction proves these
   helper defects, not a full engine test result. Its source is
   `file:explicit:daae807d6d6751409db0db84` and output is
   `file:explicit:8f948c7f6873781d14e00111`; read them with `sase artifact read`.
3. Both the Mac source `docs/capture-transfers.md` and its bob-cli mounted page still
   explain merge landing cards as removed source runs. The source row copy was
   corrected, but that paragraph was missed. The mounted page also predates the new
   copied-note wording. New merge sources retain notes; the removed-run fold remains for
   move cards and older Bob responses.

## Implementation

1. Read the docs reference memory and `docs/task-merge.md`. Make its copy rule
   explicitly protect inline and fenced code and distinguish live task identity from
   literal examples. Correct the vocabulary sentence that currently says “A task holds
   notes, so `>` to a bare note moves them”: a bare note holds tasks. Keep the contract
   aligned with the approved epic rather than changing behavior to fit existing tests.

2. Fix copy sanitization in `task_merge/payload.rs` using code-aware spans and real
   task-row recognition. Remove only live identity metadata and anchors, preserving code
   delimiters/content, indentation, and line endings. Prefer the existing task/Markdown
   scanners and field machinery; do not globally change unrelated project edits merely
   to make merge safe. Share or reuse a fence scanner that honors opening length and
   properly normalizes nested fragment indentation. Apply the same protection to wiki
   and Markdown link repair in the shared transplant path without changing move/split
   relocation behavior. Preserve the source body and exact destination-only repair
   counts.

3. Add meaningful regression coverage in `src/native/task_merge/tests.rs` and the
   appropriate CLI capture/merge tests. A closed nested task with both a live `[id::]`
   and inline-code `[id:: literal]` must lose only the former in the copy; the original
   remains byte-exact apart from the specified retirement edits. Cover multi-backtick
   inline code, ordinary non-task prose containing `#task`, and nested backtick/tilde
   fences with an inner shorter delimiter followed by literal anchors, fields, wiki
   links, and Markdown links. A real anchor/link immediately after the actual closing
   fence must still be sanitized/rebased. Exercise both same-note target-above-source
   and cross-note destinations, preserving `MergeLineOrigin` tracking, repaired-link
   counts, reconcile no-op, repeatability, and the existing gesture-equivalence tests.
   Run the move/split regressions when touching their shared helper.

4. Extend MG18 or adjacent production archive-planner coverage to merge a source that
   has an authored descendant anchor and intra-copy links, then archive its tombstone
   through a pre-existing metadata-bound noncanonical archive. Check that original
   identity moves only at archive time, external references and the copied links follow
   that bound archive, and destination copies never acquire duplicate identities. Use
   the production pairing resolver added by `2327590f`, not inferred `_done` filenames.
   Repair an actual incompatibility if this test exposes one; do not broaden into the
   active archive epic's work.

5. Open `bob-mac-capture` with `/sase_repo`, follow its AGENTS.md, and correct its
   `docs/capture-transfers.md` landing-card paragraph. Keep the legacy removed-run fold
   implementation and existing Swift wording; no Swift change is presently needed.
   Regenerate bob-cli's mounted docs through the documented docs tooling, never
   hand-edit the mounted page. Run the appropriate docs checks in both repos. Inspect
   any incidental regenerated pages before finalizing them.

## Verification and already-triaged follow-ups

Run `just check` for file-change verification and `just docs-check` for the docs gate.
Do not run `just check-full`. Follow `/sase_monitor` if a named tool run escalates; join
the same run rather than restarting it. Do not skip compilation. If Swift changes prove
necessary, follow the Mac repo's portable compilation and exact-SHA macOS validation
rules. The intended repair changes Rust plus docs only.

Use a fresh scratch vault for a built-binary dry-run/commit check of code text,
copied-note wording, retained source identity, and unchanged inbound links. Never test
writes against Bryan's vault. The installed `bob` binary on the landing host still
advertised old move semantics, so build/use this checkout's executable.

The lander completed `/sase_new_task` duplicate searches, the last-week sweep, and
review of all 17 active epic scopes. Do not file duplicate tasks for these:

- `bob-cli-je.2` note #1, task-wikilink CPU budgets: corroborated `bob-cli-f8`. Fresh
  isolated control at `f06c7e8` failed cold chunk 16.1ms/16ms, snapshot/order
  74.7ms/50ms, and rank 15.1ms/12ms; evidence `file:explicit:072da7bba40ac9881322bb16`.
- The same proposal's Live Preview/viewport timing: corroborated `bob-cli-ej`. Both
  freshness large-child and viewport tests passed in the isolated control. The phase's
  completed chat describes three index timing checks, two stylesheet checks, and one
  Live Preview timing check. No distinct new freshness defect is established; no
  duplicate or speculative task was filed.
- The same proposal's two stylesheet contracts: recorded a DISCOVERED ISSUE on active
  epic `bob-cli-j8.6.1`, whose phase .2 introduced the `#c98500` notebook fallback. Its
  own landing already owns that repair; no separate task.
- `bob-cli-je.3` note #2, macOS watchdog: corroborated `bob-cli-h7` with clean base
  `5a28892` run `42725629824` and phase head `a5a0ffb` runs `42725854554` and
  `42725895948`. This is phase-reported evidence; the lander launched no new CI.

## Final step: finish the epic landing

After the above passes, reread `bob-cli-je`, all descendants and every note, the
approved plan, and new commits on the current/base branch and affected linked repos.
Confirm the repair and existing phase claims against source and evidence. Record the
follow-up dispositions above and any additional triage in the close note.

Run `sase bead epic-symbols bob-cli-je`. Resolve each listed symbol (wire it up,
privatize it, add an appropriate non-test pragma, or delete it), or re-key its Justfile
entry only to a still-open later bead that actually needs it. The initial landing found
no entries; check again. Resolve any unfinished descendants normally. Then close with
`sase bead close bob-cli-je --note "<concrete scope, decisions, source/commit verification, integration and test evidence, and every follow-up disposition>"`.
Never use `--force` merely to succeed or advance a successful nested landing. A rejected
close means finish the named remaining work.

After closing, run `just symvision` when the recipe is available (the initial landing
found none; record absence if still unavailable). Open the plans sidecar with
`/sase_repo` and set `status: done` in `plan:202611/merge_copies_source_notes.md`, at
the PLAN path printed by the bead. Read
`sase bead read bob-cli-je -r "Need the parent link"` again. It currently has no parent,
so no ancestor close is required; do not close unrelated plans. If a parent link has
genuinely changed, follow the original landing prompt's normal phase-versus-plan
ancestor checks, never force.

Finish through `/sase_final`, declaring all changed repositories. This closeout is part
of this tale and must not be postponed until this work is committed or pushed.
