---
tier: tale
title: Finish drawer capture composition and land bob-cli-fc
goal:
  Multi-note relationships and parent reporting agree with staged writes, reserved
  drawer diagnostics agree with capture, and bob-cli-fc closes with verified evidence.
size: medium
proposed_by: bbugyi200.apollo.bob-cli-fc.land
proposed_with: codex/gpt-6-astra
bead: bob-cli-fc
status: done
---

- **PARENT:**
  [202611/capture_drawer_bullets.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/capture_drawer_bullets.md)
- **BEAD:**
  [bob-cli-fc](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-fc/README.md)
- **AGENTS:**
  - [bbugyi200.apollo.bob-cli-fc.land](https://github.com/bobs-org/bob-cli--agents/blob/main/sessions/bbugyi200.apollo.bob-cli-fc.land.md)
- **COMMITS:**
  - [50a3922](https://github.com/bobs-org/bob-cli/commit/50a39221291b7a08b64efd0082582f2e4e9cf9f2)
    — fix(capture): compose multi-note captures with relation closes and report the
    final parent

# Finish drawer capture composition and land bob-cli-fc

## Outcome

Finish the three bounded gaps found in the landing audit of
`plan:202611/capture_drawer_bullets.md`, verify the final tree, and close **bob-cli-fc
in this coder's turn**. This tale has no separate land agent. Its own SHA, push, and
post-commit CI are not prerequisites for closing the epic.

The accepted decisions stay in force: the drawer menu inserts the bare title and opens a
nested ` -` continuation when appropriate; the already-written decision and Task Drawer
glossary changes remain. This repair needs no memory edits, new CLI options, or
grammar-version bump. Preserve the current grammar revision (5 at audit time), including
the subsequently landed `=/` syntax.

## Audit already completed

The lander read the epic, every note on fc.1 through fc.4, the linked plan, the
implementation and tests, and epic commits `bbd26d97`, `f490d88f`, `5ac2fbc2`, and
`971d864f`. The Mac phase is `bob-mac-capture` commit `1b196f4`. The audited Rust head
was `ccd1ada1`. No parent bead exists above bob-cli-fc.

The grammar, drawer merge/write paths, completion's literal-title agreement,
continuation decision, additive notes/drawers JSON, and Mac decode/presentation are
implemented. The authorized memory edits are present. Intervening commits include event
agenda/start warnings, stamped Work Logs, keep-moving offers, merged-Pomodoro
routing/close names, and split syntax. The new multi-note early return duplicates the
ordinary item post-processing and omits parts of it; this is the remaining integration
work. Restoring shared relation close processing also brings multi-note completion
through the current successor/keep-moving engine.

The lander built the current Rust head with `cargo build --bin bob`. Determine the
actual build binary from `cargo metadata --no-deps --format-version 1` and its
`target_directory`; a stale local `target/debug/bob` can exist when Cargo uses a
different target directory. Only the metadata-selected build produced the findings
below. No `just check` pass is claimed by this audit.

## 1. Compose multi-note relationships through the shared finalization path

In `src/native/capture/plan.rs`, `plan_multi_note_sub_bullet_item` calls
`plan_relation_writes` but never calls `plan_relation_close_effects`. Its
`relation_cancels` and `relation_completes` are permanently empty. The ordinary item
path calls the shared close helper and absorbs its task/Pomodoro references and day
file. Reuse that behavior for multi-note items, preferably extracting a small common
helper where that avoids maintaining two inconsistent copies.

Reproduction on the audited head, in an isolated vault:

```text
foo.md:   - [ ] #task Parent ^parent
other.md: - [ ] #task Other ^other
config: relations.complete_labels: [implemented by]

@foo+parent ~other:other|"implemented by"
- first
- second
```

The dry run currently succeeds but leaves Parent open, adds the relation, and omits
`relation_completes` and `complete_effect`. The same-line-note variant
`@foo+parent a note ~other:other|"implemented by"` completes Parent correctly. With
`relations.cancel_labels: [duplicate of]`, the analogous multi-note cancel also silently
stays open. This is not deferred follow-up work.

Carry the shared helper's complete/cancel effects, warnings, day file, task-block roles,
and Pomodoro refs into the multi-note result. Keep existing conflict and recurring-task
refusals and batch rollback semantics. Exercise `~` and `~~` so the actual relation
source receives the effect. Ensure same-note relation mirrors cannot leave stale line
positions in returned refs.

Add meaningful CLI regression coverage in `tests/cli/capture/note_bullets.rs` and/or the
relation-close suites: configured completion and cancellation, inverse source, conflict
refusal with byte-identical rollback, and a completed task with ledger/successor or
keep-moving effects. Compare dry-run and commit results and the resulting files. Retain
ordinary unconfigured relation behavior.

## 2. Report the final multi-note parent state

The multi-note path saves the parent's line/status before `&`, `*`, and relation
effects. It never refreshes those fields. With valid Tasks settings (reuse
`write_blocked_tasks_settings`), this draft demonstrates the discrepancy:

```text
@foo+parent &other:other
- first
- second
```

Current JSON says `parent_status_symbol: " "` and `parent_status_name: "Todo"`, while
`dependency_update.dependent_status` and `task_blocks` correctly say `?` / Blocked.
Human output and Mac's header consequently describe the old state.

After all staged effects, resolve the target by stable identity and refresh parent line,
text, status symbol/name, and task-block refs from the final staged content. Use the
existing lookup helpers; do not duplicate dependency or status rules. Test `&` blocking,
applicable `*` commitment changes, and relation close results from step 1. Assert
agreement among parent metadata, human output, task blocks, and file bytes, including a
same-note mirror that shifts the parent line.

## 3. Restore the planned reserved-drawer editor diagnostics

`capture_language/editor_parse.rs::attach_drawer_bullet_metadata` currently emits drawer
spans with no `reserved_drawer` diagnostic for a reserved key. For both:

```text
Ship it
- #tasks
  - note
```

and the literal `- TASKS` variant, `capture-parse -f json` reports `mode: task`, no
needs and no diagnostics, while `capture --dry-run -f json` refuses `reserved_drawer`.
The epic explicitly requires lexical reserved-name diagnostics.

Use the existing drawer resolver/key rules in `capture_drawers.rs` to produce the same
code and appropriate copy for known reserved names at under-task bullet positions. Keep
the distinction between a task drawer, an existing `^prj` section, and a newly created
project note's structural headings; do not blanket ban valid new-project `TASKS`
sections. Do not make the editor guess vault-dependent ambiguity or near-duplicate
results. Preserve `drawer_notes` for childless shortcuts, childless literal ALL-CAPS
plain-note behavior, escaping, UTF-8 byte ranges, and nested task-type markers. Add
parse/preview/commit agreement tests covering reserved shortcuts and literals, legal
controls, and project scope.

## 4. Verify and finish the landing

1. Re-read `sase bead read bob-cli-fc -r "Recheck epic closeout and triage"` and its
   child notes for any new evidence. Review changes landed since `ccd1ada1` and
   integrate any that touch these paths. Do not reimplement the completed phases. Update
   the epic note with this tale's fixes and verification.
2. Run focused regressions, then **`just check`** and **`just docs-check`**. Never run
   `just check-full`. Fix failures caused by this work; independently existing failures
   use `/sase_new_task` and must be reported accurately. The known `bob-cli-fh` Clippy
   report concerns an unrelated named-capture change; check its current status before
   attributing any failure. If docs/help change, read the docs memory first and run the
   repository's generation gates.
3. Repeat the original worked example in a scratch vault, including completion
   acceptance into existing task, existing `^prj`, new task, and new project note. Cover
   the original refusal cases and the repaired compositions. Confirm no repository or
   live-vault changes came from those scratch checks.
4. Follow-up triage has already been completed by the lander and recorded on bob-cli-fc.
   Preserve all outcomes in the close note: fc.1 #1 / fc.3 #2 / fc.4 #2 became
   **bob-cli-fi**, a large feature task for third authored nesting; fc.2 #1 was declined
   as already fixed by `39804896`; fc.3 #1 was declined as already covered by bob-cli-fe
   and fixed by `d712168f`'s accepted merge expectations; fc.4 #3 was declined as fixed
   by Mac `6f13e48`'s writable `_colorSchemeContrast` key path. A separate new Calendar
   banner compile error became small CI task **bob-cli-fj**: Mac runs 41377358537
   (`6f13e48`, clean base) and 41378013621 (`1b196f4`) fail identically at
   `CapturePanelView.swift:4688` because `eventWarningBanner` omits `return` after local
   setup. It is unrelated to the drawer phase. Read these tasks if their status changes;
   do not create duplicates or turn them into epic work.
5. Mac verification is accurately bounded: fc.4's portable-check passed 1,922 tests; its
   earlier snapshot Build passed, but Test was blocked by the old contrast-key compile
   failure. Current master CI is blocked at Build by bob-cli-fj. Inspect whether that
   independent repair has landed and record the current evidence. This tale is
   Rust-only; no Mac mutation is required. If a necessary Swift change is discovered,
   open the linked repo with `/sase_repo`, follow its AGENTS.md, run portable-check and
   compile macOS targets through its scratch-host/CI workflow before considering that
   change verified. Do not wait for this tale's own future commit, push, or CI to
   perform closeout.
6. Before closing, run `sase bead epic-symbols bob-cli-fc`. Resolve every listed entry
   by wiring, privatizing, adding a justified non-test pragma, or deleting it under the
   Symvision policy; re-key only to a still-open bead that actually needs the exemption.
   The audit found no entries, but recheck. Complete and close this tale's own plan bead
   normally first if descendant readiness requires it; use `sase bead close`, never
   hand-edit bead status.
7. Close the epic **in this turn** with
   `sase bead close bob-cli-fc --note "<phase and drift audit, three fixes, actual verification results, all follow-up outcomes, and Mac CI limitation>"`.
   If refused for leftover symbols or unfinished named phases, fix the blocker and
   retry. Never force to make successful nested landing advance. Cancellation or
   supersession requires a deliberate non-done resolution and explicit reason.
8. After the close, run `just symvision` if the recipe exists (the audited bob-cli
   justfile has none). Use `/sase_repo` to open `plans` and the audited artifact path
   for `plan:202611/capture_drawer_bullets.md` (the file is
   `202611/capture_drawer_bullets.md` under the opened plans repo), then set only its
   frontmatter `status: done`. Re-read bob-cli-fc's parent link; it was null at audit
   time, so there is no ancestor to close. If that has changed, follow the land prompt's
   parent-phase/parent-plan readiness rules before closing any ancestor. Declare all
   changed repositories through `/sase_final`; no manual commit is needed before the
   epic closes.
