---
tier: tale
title: Finish capture type-marker agreement, previews, and landing
goal:
  Repair the remaining t:type contract and integration defects, verify performance and
  consumers, then close bob-cli-ee and mark its plan done.
size: medium
proposed_by: bbugyi200.apollo.bob-cli-ee.land
proposed_with: codex/gpt-6-astra
bead: bob-cli-ee
status: done
---

- **PARENT:**
  [202610/capture_type_marker.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/capture_type_marker.md)
- **BEAD:**
  [bob-cli-ee](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-ee/README.md)
- **AGENTS:**
  - [bbugyi200.apollo.bob-cli-ee.land](https://github.com/bobs-org/bob-cli--agents/blob/main/sessions/bbugyi200.apollo.bob-cli-ee.land.md)
- **COMMITS:**
  - [f6a2e73](https://github.com/bobs-org/bob-mac-capture/commit/f6a2e732ef0a8136885488d5bb9b068303614a8a)
    — test(capture): refresh t:type fixtures for truthful relocation previews

# Finish capture type markers and land bob-cli-ee

This is the remaining work from the landing audit of **bob-cli-ee**, not a new feature
or a replay of its completed phases. One coder owns the bounded Rust fixes, consumer
fixture refresh, verification, and **the epic's final closeout in that same turn**. A
tale has no land agent afterward. Do not wait for this work's own commit, push, or CI
SHA before closing the epic: the host commits after the coder's turn.

## Scope and fixed decisions

Read `sase bead read bob-cli-ee -r "Need landing audit and follow-up dispositions"` and
`sase artifact read plan:202610/capture_type_marker.md "Need approved marker contract"`.
The epic's final decisions remain **menu_order=config**, **unknown_types=warn**, and
**memory_record=no**. Do not edit any memory file, Cargo package version, CHANGELOG.md,
or `tests/fixtures/pomodoro_themes/vectors.json`. Keep schema version 1 and optional
Swift decoding. Bob owns grammar, completion, preview facts, and writes.

The lander read all notes on ee.1–ee.4, nested epic ee.1.1, and ee.1.1.1–ee.1.1.3; all
descendants are closed. Relevant CLI commits are 59d5a093, 23ba4805, f40a625a, 41e89597,
0e178061, and 03a3497f. The audited CLI HEAD and fetched origin/master were 7414d3bc;
the Mac feature is 68aa63a. Recheck post-audit drift before editing.

Open `bob-mac-capture` through `/sase_repo`, and read its AGENTS.md before working
there. To update the epic's plan at closeout, open the configured plans sidecar through
`/sase_repo`; use its returned path. Read sidecar artifacts only through
`sase artifact read`. Never reuse a path from another agent's workspace.

## 1. Make D8 relocation produce truthful capture previews

`CaptureTaskThemeContext::relocate_for_retype` in `capture/task_theme.rs` stages the
daily-note edit but returns only `TypeRelocationJson`. Its callers in `capture/plan.rs`
retain the earlier gesture's Pomodoro refs. The shared applier in
`pomodoro_themes/relocation_apply.rs` knows which entry was renamed or created but
discards the identity/line information the capture tracker needs.

Consequences reported by ee.4 and corroborated in source:

- `rename_entry`, `append_component`, and `rename_placeholder` can hit
  `pomodoro_blocks.rs`'s "changed or created pomodoro needs an explicit before" guard.
- `create_below` can hit "rewritten pomodoro headline has no ref".
- Release builds avoid the panic by reporting a misleading block diff. An installed bob
  0.4.0 dry run renaming running BUILD to FIX reported `created: true`, every line
  `unchanged`, and top-level `pomodoro_name: BUILD` beside relocation `to: FIX`.

Return enough internal relocation metadata to compose explicit
`PomodoroBlockRef::at/created` references into the **whole capture item's** pre/post
snapshots. Rebase existing gesture refs if the relocation shifts their positions. Report
both source and destination where appropriate, with correct cumulative before/after line
changes across multi-item batches. Do not guess identity from name equality, weaken the
tracker's guards, disable debug assertions, or special-case test builds. Preserve the
shared applier's `bob task move` behavior.

Refresh result fields that describe the link's final destination after relocation
(`pomodoro_name`, line/placement data and related preview facts where present), so the
main preview, `type_relocation`, and `pomodoro_blocks` agree. Preserve omission when
relocation does nothing.

Reproduce each outcome in CLI tests with normal debug assertions. A compact scratch
fixture has canonical daily note `2026/20261030.md`, `BOB_NOW='2026-10-30 09:15:00'`,
configured BUILD/FIX/WRITE types, `plan.max_themes: 30`, and a project/area note
`dev.md` with `task_type: WRITE` and `- [ ] #task Fix parser ^fix-x`. Capture
`^dev:fix-x t:fix` against these ledger contents beneath `## Pomodoros`:

| Outcome            | Ledger setup                                                           |
| ------------------ | ---------------------------------------------------------------------- |
| rename_entry       | Running `- [ ] (**0900-1000**) — BUILD` with only `[[dev#^fix-x]]`     |
| append_component   | Same running entry with a second dedicated link to another real task   |
| move_to            | Planned BUILD containing the link, followed by an existing planned FIX |
| rename_placeholder | Planned BUILD containing only the link; no FIX entry                   |
| create_below       | Planned BUILD with two dedicated links; no FIX entry                   |

Test dry-run and actual capture against equivalent fresh fixtures. Require the exact
outcome, final note bytes, accurate block diffs and destination metadata. Exercise
whole-item links, relationship-only owners, and the applicable toggle/Ensure Next paths,
including a batch where an earlier item changes line numbers.

Strengthen `tests/cli/capture/type_marker_existing.rs`: its current strict-cap test
accepts either success or failure, and its relocation test accepts any outcome. Use
fixtures whose D8 result is unambiguous; assert `strict_cap` and byte-for-byte rollback
on a real attempted write, plus rollback when a later batch item fails. Keep the
existing task-move tests unchanged and passing.

## 2. Validate existing-task markers against the resolved named destination

`apply_type_marker_to_existing` validates raw `pomodoro_name` before the link/toggle
planner resolves it. With configured BUILD/FIX, raw `bui` looks like a container. This
regressed the resolved-name rule already fixed for new tasks in 41e89597.

Independent installed-binary checks in an isolated fixture:

- `^dev:fix-x#bui t:fix` succeeds and renames BUILD to FIX; it must refuse.
- With the task linked under WRITE and another planned BUILD entry,
  `@dev+fix-x#bui t:fix` succeeds and leaves the FIX task linked under BUILD; it must
  refuse. Full `#build` correctly returns `type_marker_conflict` in both forms.

Keep the marker's field staged before placement, then validate the actual resolved
destination before D8 repair or final writes, using the established new-task agreement
helper or a shared equivalent. A typed component, completed-name "again" match,
numeric/name selector, literal, or newly created destination follows the approved
container/component/conflict table. Do not reinterpret a named conflict as permission to
rename or move away from the user's explicit choice.

Add exact/prefix/literal/component/container coverage for existing links, Ensure Next,
and supported named toggles/start forms. Require structured `type_marker_conflict`, no
writes, and atomic batch rollback. The name supplied by `t:` itself still never resolves
by prefix; unknown marker names warn and are used literally. Preserve strict-cap refusal
and closed-task protection.

## 3. Integrate type completion with Task Wikilink spans

Post-start commit 4b4a4a34 adds Task Wikilink parse spans via the shared scanner and
`fill_span_gaps`; its Mac counterpart is ef07ce2. For `Follow up ((foo t:` at byte 18,
parsing reports the type marker at bytes 16..18, but `capture-complete` returns
`context: wikilink_task`, query `foo t:`. The engine checks Task Wikilink completion
before the capture marker field. This makes paint, menu selection, and accepted
replacement disagree.

Use Bob's existing lexer/field ownership to let a valid edge `task_type` token own its
caret and replacement range. Keep ordinary and closed `((...)` queries working at their
own positions, preserve protected markup and mid-line prose, and keep
`wikilink_task_spans` and `completion_field_at` on their shared scanner. Do not add
Swift grammar or duplicate the type-marker lexer.

Add parse/completion coexistence tests at every relevant UTF-8 boundary (including a
Unicode prefix), before/inside/after the type marker, and neighboring `@route`
completion. Accepting FIX must replace only `t:`, preserving the preceding query bytes.
Include a capture mention with `t:fix` to verify post-start mention staging retains the
typed task and correct source/mirror identities. The earlier mention commits
22a8c84f/b081097c already stage mirrors after routing; reuse that boundary. The e7 epic
has a coordination note that this tale owns the t:-specific integration.

## 4. Complete the feature's performance acceptance

ee.3 recorded task-type zsh completion at p50 81.7 ms/p95 107.4 ms on apollo, against a
75 ms vault target and 150 ms deadline. This is an unresolved feature requirement, not
an unrelated task. The approved plan also requires PT3's warm +5 ms p95 budget for
capture completion/parse. `docs/completion.md` contains the measurement; its other rows
date from October 2, so do not infer a controlled delta from that table.

Build the current source and repeat warm measurements with documented argv, build
profile, date, sample count, and comparable baseline requests on the same host and
vault. Profile the new path if it remains over budget. Reuse the catalog's completion
view and lazy home resolution; avoid duplicate catalog/config scans or unnecessary
vault-wide work. Keep configured order, hidden exact matches, typed rows, inherited
effects, and current data correct. Do not drop annotations, cache stale answers, or
raise deadlines to make the measurement pass. Bound optimization to the task-type
completion path and its shared cache use. Record both baseline and final p50/p95 in
`docs/completion.md`, replacing the unresolved-follow-up sentence only when supported by
the results. Read docs memory before editing; regenerate help only if it changes.

## 5. Refresh the Mac evidence and run verification

The Mac implementation already decodes optional fields and renders type pills,
configured-order rows, the consequence footer, Ctrl+D ghost state, Type/ledger previews,
and notifications. Its fixture generator currently documents the debug panic workaround,
and ee.4 generated relocation fixtures with `CARGO_PROFILE_DEV_DEBUG_ASSERTIONS=false`.
Regenerate using the repaired normal debug binary, remove the workaround commentary, and
verify correct before/after block presentation. Extend the existing fixture tests to
reject the inaccurate `created`/unchanged relocation output and cover the completion
coexistence case. Do not rewrite the UI unless the corrected contract exposes a concrete
defect.

At audit time Mac CI run 41098433632 for 68aa63a was still running. Inspect its final
Build/Test/Bundle results, and triage failures with the normal evidence rules. Existing
ee.4 evidence is athena portable-check (1800 tests), Mac scratch typecheck-tests,
docs-test, and docs-check; no recorded manual smoke/screenshots were in the phase notes.
Inspect the existing render fixtures for the task-type menu and typed/ledger preview in
light and dark appearances. When the Mac graphical session is available, use a scratch
app and an isolated fixture vault for the approved smoke cases: choose FIX, leading
`t:`, marker-only child line, linked existing task, unknown name, and Ctrl+D hide/show.
Register the menu and preview images with `/sase_artifact`. Report unavailable manual
evidence precisely; never present generated renders as manual smoke. Follow the existing
Mac plan's best-effort/offline rules.

Run focused regressions, then **`just check`** in bob-cli and `just docs-check` for
changed docs. Do **not** run `just check-full`. `just check` already runs formatting,
Clippy and workspace tests; a separate `just all` repeats those gates unnecessarily. A
check-full-only failure is test infrastructure, not remaining epic work.

In the Mac checkout run `just portable-check` (on a disposable athena copy if Swift is
absent locally), and `just docs-check` with `BOB_CLI_DIR` explicitly set to the
authorized bob-cli checkout. If Swift source or hosted tests change, follow its
AGENTS.md: compile a scratch Mac copy with `Scripts/bundle.sh --identity -` and
`just typecheck-tests`; never use Bryan's checkout or `just install`. If the Mac is
offline, explicitly report macOS targets unverified. Use `/sase_monitor` for long
commands; make every handoff carry this tale's remaining closeout, and wait for the
monitor-start command itself to exit. Never end a turn with an unmonitored command still
running. Do not make closeout depend on CI for this tale's future commit.

## 6. Close bob-cli-ee in this coder's turn

First verify the fixes, current descendants, linked plans and post-audit commits. The
completed nested grammar plan is `plan:202610/capture_type_grammar.md`; the outer plan
to finish is **`plan:202610/capture_type_marker.md`**. Preserve all final decisions. No
parent_bead existed on bob-cli-ee at this audit.

The lander completed follow-up triage and recorded it on bob-cli-ee. Carry these
outcomes into the close note:

- ee.1 #1, ee.2 #1, ee.3 #1, ee.4 #1: one skipped decision-memory proposal; declined
  because memory_record=no is final. No memory edit or task.
- ee.1.1.1 #1: clone_on_copy fixed by a0a6b497 before the first grammar commit; declined
  a new task/corroboration because it is resolved.
- ee.3 #2: completion latency retained and resolved/verified as epic work here.
- ee.4 #2: four relocation debug panics retained and repaired as epic work here.
  bob-cli-dq is a distinct in-section task-status rewrite issue; no unsupported +1 or
  duplicate task. No other child proposals require triage.

Then perform these actions, without waiting for your own future commit:

1. Run `sase bead epic-symbols bob-cli-ee`. Resolve every listed symbol by wiring it,
   privatizing it, adding an appropriate non-test pragma, or deleting it under the
   Symvision policy. Re-key an exemption only to a still-open later bead that truly
   needs it. The audit found no entries; recheck immediately before close.
2. Close normally with `sase bead close bob-cli-ee --note "<verification>"`. Write a
   concrete note covering all child/nested notes and decisions, D8 preview and
   named-destination fixes, wikilink/mention integration and later drift, performance,
   exact check/docs/Mac results and any permitted limitations, and every follow-up
   disposition above. If symbols reject close, clean them and retry. If descendants
   reject close, finish or reopen them and verify their scope. Never use `--force`
   merely to make close succeed or to advance a successful nested landing; only a
   deliberate canceled/superseded outcome permits force with a reason.
3. Run `just symvision` if the recipe is available. Neither audited repository had that
   recipe; if still absent, record the absence instead of inventing a pass.
4. In the authorized plans sidecar, set **`status: done`** in the frontmatter of
   **`202610/capture_type_marker.md`**, preserving the other metadata. Resolve the
   current PLAN path from the bead/artifact rather than assuming a local directory.
5. Run `sase bead read bob-cli-ee -r "Need the parent link"`. If it still has no parent,
   finish normally. If a parent was added, follow the landing user's rules: verify and
   close only an immediate parent phase normally; for an immediate plan parent, re-audit
   its previous landing note, all descendants and notes, linked plan, and post-child
   drift, retire its symbols and close/mark done only if fully ready. Stop at the first
   incomplete/ambiguous parent and note the blocker there. Never force a successful
   ancestor close.
6. Finish through `/sase_final`, declaring every modified repository, including the Mac
   and plans checkouts. Report whether bob-cli-ee closed and any remaining verification
   limitation. No separate land agent will resume this tale.
