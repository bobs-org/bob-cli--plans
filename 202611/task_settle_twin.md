---
tier: epic
title: Vector-pinned taskSettle provider for ledger-tools
goal:
  Complete bob-cli-q4.8 with a Rust-parity pure JS settle transform, guarded editor
  application, and a shared idle scheduler behind the versioned taskSettle API.
phases:
  - id: pure-twin
    title: Port the pure settle transform and pin Rust parity
    size: medium
    description:
      "pure-twin: port the Rust schedule, grouping, badge, and surviving-line-map
      behavior into pure bob-ledger-tools fragments. Run all canonical settle vectors
      through the built bundle and enforce byte-identical fixture copies. Export
      testable helpers without advertising an incomplete taskSettle v1 namespace."
    depends_on: []
  - id: editor-apply
    title: Add the guarded editor adapter and cursor policy
    size: medium
    description:
      "editor-apply: add Tasks classification and daily-path adapters plus the
      synchronous editor application layer. Apply minimal changes atomically, preserve
      the approved cursor-stays policy, return mapped jump targets, and verify immediate
      undo and path safety. Keep the public namespace unpublished until scheduler
      completion."
    depends_on:
      - pure-twin
  - id: pending-api
    title: Publish taskSettle v1 with the pending scheduler
    size: medium
    description:
      "pending-api: implement the shared 500 ms pending scheduler, early flush and
      teardown lifecycle, publish the complete taskSettle v1 namespace, document its
      consumer contract, verify skew and race behavior, and deploy bob-ledger-tools."
    depends_on:
      - editor-apply
proposed_by: bbugyi200.athena.bob-cli-q4.8
proposed_with: codex/gpt-6-astra
parent_bead: bob-cli-q4.8
create_time: 2026-11-25 10:27:06
status: wip
---

- **PROMPT:**
  [prompts/202611/task_settle_twin.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/task_settle_twin.md)
- **PARENT:**
  [202611/better_o_gesture_settle.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/better_o_gesture_settle.md)

# Complete the ledger-tools settle provider for bob-cli-q4.8

## Scope and evidence

This is a child epic implementing only `bob-cli-q4.8` (settle-twin) from
`plan:202611/better_o_gesture_settle.md`. Its deliverable is a complete
`bob-ledger-tools.api.taskSettle` v1 provider: a pure note-local transform, a guarded
editor adapter, and a shared pending scheduler. The parent epic's cycler-settle,
nav-settle, and link-settle phases remain responsible for wiring their gestures.

Read the assigned phase and parent design with audited `sase bead read` and
`sase artifact read` calls. The consolidated research is
`research:202611/better_o_gesture_settle/better_o_gesture_settle.md`.

At planning time, q4.8 has no previous work notes. The required Rust dependency has
landed in bob-cli commit `93956838` (`feat(task): add pure note settle helper`).
`src/native/task_settle.rs` contains the pure transform and surviving-line mapping;
`src/native/task_status_groups/{mod,parse,emit}.rs` contains the grouping parser and
emitter; `src/native/task_status_hooks/compose.rs` supplies eligibility. The shared
`tests/fixtures/task_settle/vectors.json` is version 1 and has 18 cases. The inspected
bob-plugins tree at `eac9d744` has no taskSettle provider or pending-settle scheduler.
Recheck those facts before editing so landed work is reused.

The port spans about 2,400 lines of Rust source, including conservative Markdown
parsing, plus editor/history and timer integrations. Three sequential medium phases
separate deterministic byte parity, editor behavior, and scheduler lifecycle. They all
modify ledger-tools, so no phase runs in parallel with another here.

## Fixed decisions and boundaries

The ancestor epic's reviewer choices are final: `o_in_group = intake_end`,
`settle_cursor = stay`, `wasm_spike = yes`, and `memory_settle_record = no`. This
provider implements the stay policy. It does not change open-line placement or file the
WASM research task. No memory file may be edited.

Rust is authoritative. Settle applies the local future-schedule Blocked rule, then
eligible status groups and badges. It never unblocks, lowers a sticky lane, tidies
children, completes requirements, resolves dependencies, stamps freshness, repairs
mirrors, or performs clock-driven or vault-wide reconciliation. Requirements, notes,
drawers and logs travel with their existing blocks; their internal order stays intact.

No gesture or provider method spawns bob, reconcile, or tidy. The transform reads no
files and no clock. A missing, too-old, or throwing provider leaves a consumer's
existing write intact; reconcile heals later. No global command monkey patch is
introduced: consumer phases call the shared `flushPending()` at their command entry
points, while this epic owns provider lifecycle events and ledger-tools' own entry
points. Repeated cycle taps must be able to reset the pending timer without flushing
their own preceding tap.

Open bob-plugins through `/sase_repo`, and obey its AGENTS.md/CLAUDE.md. Edit source
fragments and register them in `src/fragments.json`; keep every hand-edited fragment at
or below 1,000 lines. Rebuild generated main.js after changes and rebases, never
hand-merge it. Use existing mixin installation and export conventions.

Before documentation changes read `docs.md` through `/sase_memory_read`. Before relying
on task terminology or changing governed behavior, read the applicable glossary strands
and decisions, including task-status-is-derived, task-lanes-are-sticky, and
task-reqs-are-met-with-their-task. New follow-up work is recorded as
`PROPOSED FOLLOW-UP:` notes on the assigned phase, never a new task bead.

## Contract shared by the three phases

The final runtime lookup is `app.plugins.plugins["bob-ledger-tools"]?.api?.taskSettle`.
The namespace is frozen, has `version: 1`, and offers the parent's four methods:

- `settleNote({path, contents, today})` synchronously returns at least
  `{contents, changed, lineMap}` without I/O or writes.
- `applyToEditor(editor, path, options)` synchronously applies the settle, returning the
  line map, changed/applied outcome, and optional jump target. Guard failures return a
  non-writing outcome; they never try another editor or write the vault.
- `schedulePending({editor, path, identity})` arms or resets a pending settle.
- `flushPending()` synchronously drains all currently pending eligible work, so a caller
  can flush before reading task positions for its next command.

Define `lineMap` as a plain object mapping zero-based surviving input line indexes to
zero-based output indexes; omit removed lines. Compare it to the Rust fixture's ordered
pairs after normalizing representation. This is a semantic move map, not just the editor
transaction's monotonic offset mapping.

Keep the core helper fully explicit in its inputs: path, contents, today,
classification, and dailyPath. The runtime adapter supplies a synchronous snapshot of
Tasks status settings/global filter and the configured daily path. Dates are explicit in
pure calls. At editor/scheduler boundaries use the existing local-date/daily-path
helpers, capture context once per apply, and do not read or create daily files. Do not
substitute a Tasks cache task list or filesystem metadata for the passed contents;
unsaved frontmatter is part of eligibility.

Use task identity as `{blockId, line, text}` with zero-based `line` a hint and exact
live task text the fallback when no block id exists. Resolve block id first; otherwise
use the hinted text match or an unambiguous exact-text match in the same path. Never
assign an id merely to settle. Missing or ambiguous identities drop pending work. The
cycler passes the identity after each immediate checkbox edit, refreshing the text
fallback on every tap.

`applyToEditor` must support settling a caller's staged post-gesture contents in the
same editor transaction, with an expected preimage guard, as well as settling the
editor's current text after another API has already written. Specify the exact options
and coordinate systems in the API docs. In the staged case, map the caller's cursor and
target identity into the staged input before applying settle's lineMap; do not mistake
pre-gesture line indexes for settle-input indexes.

## Phase pure-twin

Port the actual Rust algorithm, not only the happy-path examples. Put new pure fragments
under `plugins/bob-ledger-tools/src/` and expose helpers through the built bundle's test
exports. Split parser, emission, and settle/map functions as needed. The existing
requirements/layout twins may inform helpers, but do not import another plugin's runtime
bundle or force a new cross-plugin dependency.

Match these details explicitly:

- `TaskClassification::from_status_types`, including defaults, custom/unknown statuses,
  global-filter recognition, and open versus terminal status semantics.
- The local scheduled-date rule and stale `[unblocked::]` pruning, including already
  Blocked rows. Use the Rust parsing boundaries even in notes ineligible for groups.
- Area/project frontmatter eligibility, exclusion of canonical daily notes and the
  configured current daily path, and conservative Markdown section ownership.
- All ten grouping skip codes, legacy ownership markers, empty groups, badge
  replacement, complete subtree movement, and exact newline/final-newline behavior.
- Rust's surviving-line map: raw-line Myers matches first, nearest unused identical line
  next, then the same stable task-line key for status/prunable-field rewrites. Duplicate
  blank or authored lines and tie breaking must agree with Rust. The navigation editor
  diff normalizes more fields and must not replace this algorithm.

Copy the canonical fixture directory into
`plugins/bob-ledger-tools/scripts/fixtures/task_settle/` byte for byte. The README must
say “update bob-cli first, then copy”; add that instruction to the canonical fixture
README first and copy it too. If new parity cases are necessary, add and verify them in
bob-cli first, then recopy. Do not weaken expected maps or warnings to make the twin
pass.

Add `scripts/test-ledger-tools-task-settle-vectors.cjs` and a fixture-copy test using
`scripts/test-task-reqs-twin-copies.cjs` / `scripts/task-reqs-vectors.cjs` as the local
patterns. Run every vector through the built ledger-tools bundle, asserting bytes,
changed, maps, warnings, and a second-pass no-op. Compare every copied fixture file to
the canonical bob-cli file when it is available; ensure that comparison actually runs in
this linked-checkout workflow. A standalone plugin checkout may explicitly skip only the
canonical comparison, never its own vector suite.

Add focused regressions for custom classification, duplicate-line maps, Unicode and CRLF
coordinates, and input immutability where the canonical suite lacks them. Keep
byte-contract regressions in the authoritative vectors when applicable. Register every
new test file in `package.json`'s unit list. Do not publish a partial v1 namespace.

## Phase editor-apply

Build the context adapter and editor layer over the verified pure helpers. Follow
`174-plugin-task-merge-api.js` for versioned-namespace structure and
`170-plugin-lifecycle.js` / `310-install-methods.js` for integration. Keep the future
public API unpublished until all four methods are implemented by pending-api.

Resolve the Markdown view that owns the supplied editor, including inactive leaves, and
check its file path against the supplied path immediately before mutation. A
missing/mismatched view or stale staged preimage yields no changes, cursor movement,
jump history, scroll, or vault writes. Avoid basing the guard only on the active file.

Plan ascending non-overlapping minimal editor edits. The algorithm in navigation's
`181-editor-line-diff.js` is an existing reference for preserving stable content and
viewport; keep settle's semantic move map separate. Apply all ranges and the intended
selection in one CM6/editor transaction. Do not replace the whole document with
setValue. Recheck unchanged/no-op cases and preserve UTF-16 editor column semantics.

Implement the chosen cursor policy when the affected task's block moves: if the cursor
belongs to that task, retain its physical line index (clamped if necessary) and return a
jump target for the task's new location. Other cursors follow their surviving content
through lineMap. Return enough identity information for a caller to record the jump with
navigation's existing jump API and to resolve its own focus contract afterward; do not
add duplicate jumps. Selections and removed lines use deterministic document-bounded
fallbacks. Preserve the viewport unless the cursor would be offscreen.

Test with the ledger-tools harness and actual CM6 history support where available: one
transaction for staged gesture plus settle, one undo/redo restoring both, no-op does not
add history, moved own task versus unrelated cursor, subtree cursor, selection mapping,
deleted headings, path switch, inactive editor, stale preimage, and no whole-document
replacement. For an external already-applied write, use the supported history join
mechanism and verify its behavior instead of assuming event annotations alone guarantee
it. Establish a scratch-vault real-Obsidian check if the unit harness cannot prove the
immediate undo contract; never use the live vault for these tests. Document that
separate editor fallback's observed behavior precisely.

## Phase pending-api

Implement a provider-owned pending scheduler with fake-timer-friendly internals. Pending
entries carry the editor instance, exact path, task identity, and generation; they never
retain a full document to overwrite later. Repeated taps on the same task replace its
identity snapshot and restart the roughly 500 ms timer. A stale timer cannot clear or
apply a newer generation. Flush any prior different-task work before arming its
replacement, after applying the same safety checks.

At fire/flush time, validate the editor path and resolve the live task identity, then
settle the current buffer. Drop work after file switches, task deletion, identity
ambiguity, or unavailable context. Clear ownership before applying to avoid reentrant
editor events double-applying; cancel timers when drained. Multiple views/editors must
not overwrite one another or cause duplicate writes for a shared document.

Install early flush on cursor leaving the pending task line, active-leaf-change,
file-open, and blur. If a switch event arrives after an editor has changed buffers, drop
its pending work; never load the old path through that editor. Dispose all
timers/listeners and pending state on unload/reload. Add the command-entry flush to
ledger-tools commands; the parent consumer phases own cycler/nav/block-id command
wiring. Do not flush merely on every document update, which would break rapid cycling.

Publish the complete frozen `taskSettle` v1 namespace in the root API only now. Add
tests for the exact runtime lookup and call-time feature detection (absent, old,
throwing provider) so downstream consumers can copy a safe usage pattern. The normal
pure/apply paths remain synchronous and filesystem-free on desktop and mobile.

Test rapid double and counted cycling with an identity updated after each flip; 500 ms
rearming; global flush across editors; cursor-leave, blur, leaf/file events; same editor
switching A to B before timer fire; same path in multiple leaves; unload; reentrancy;
exceptions; and timers firing after explicit flush. Assert both resulting bytes and that
the wrong note never receives a write. Verify the delayed undo behavior: prefer cycle
plus regroup in one undo event, but the parent permits two events when the editor cannot
join them. If so, document and test the exact order (regroup undo, then cycle undo) and
ensure unrelated edits are never swallowed into that event.

Update `docs/bob-ledger-tools.md` and `docs/develop/plugin-apis.md` with the complete
method signatures, line/column bases, identity rules, staged apply/preimage options,
stay/jump semantics, fallback, 500 ms/early-flush triggers, and measured undo behavior.
Link bob-cli's `docs/task-status-hooks.md` settle contract through the docs site's
established link convention. Explain that parent consumer phases provide gesture wiring;
do not claim every keymap already settles. Bump the ledger-tools manifest version
according to repository practice, rebuild, validate, and deploy it.

## Verification, deployment, and closure

Each phase runs its targeted tests, `npm run build`, `npm test`, and
`npm run docs:check` for documentation changes. Run bob-cli's `just check` when it
changes; the child epic's land agent runs the final bob-cli gate regardless. Run
`just docs-gen` / `just docs-check` when bob-cli documentation is touched. The pure
phase's canonical fixture change also runs the Rust task_settle tests. Do not edit
Cargo.toml's version or CHANGELOG.md. Mark an actual CLI/JSON/vault-byte contract break
with `!`; adding this API does not authorize changing the canonical Rust bytes. Each
phase that changes plugin code also bumps that plugin's manifest version using the
repository convention before rebuilding and deploying its changes.

After changes to bob-plugins, run `bob plugins sync -p bob-ledger-tools`; do not pass
`-F`. On refusal, follow AGENTS.md and `docs/develop/deployment.md`, fetch/check the
named deployment, and wait through `/sase_monitor` for active unlanded work when needed.
Rebuild after rebases. Do not deploy other plugins. Long verification or wait commands
use `/sase_monitor` and the prescribed finalization flow.

Record failures reproduced identically on the clean base as `PROPOSED FOLLOW-UP:` notes
with evidence and any existing tracking bead; do not strand a completed phase on such
failures. Never hand-set phase status. Before each close run
`sase bead epic-symbols <assigned-phase>` and resolve its entries or re-key them to a
still-open owner; close only the assigned child phase.

The child epic land agent verifies all three phases, shared-vector copy parity, complete
API publication, scheduler and editor tests, documentation, and the deployed
ledger-tools version. It records the API signatures and verification evidence on
`bob-cli-q4.8` for its downstream consumers. Finish this child epic's own closure
lifecycle, then run `sase bead epic-symbols bob-cli-q4.8`, clear/re-key remaining
symbols, and close **bob-cli-q4.8** with the verified results. This is the explicit
delegation to complete the original phase. Never close `bob-cli-q4` or any ancestor plan
bead. The original phase remains open until this complete provider is verified.
