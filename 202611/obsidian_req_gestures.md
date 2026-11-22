---
tier: epic
title: Obsidian requirement guards, toggle, drop, and typing assist
goal: Complete bob-cli-on.10 (the obsidian-gestures phase of plan:202611/task_requirements.md).
  The cycler, nav, and Block ID Prompt recognize requirement lines through byte-identical
  JS twins that pass bob-cli's RQ/RL/RW vectors. Every task gesture refuses or redirects
  on a req line with the contract copy. Ctrl+] toggles note and req with zone placement
  and the first-req scoop, Alt+[ drops with a reason and Alt+] restores, Ctrl+Shift+P
  opens the owner's Task Card, the offer-close scope note never counts reqs, and typing
  `- [` then space expands to `- [ ] ` with overtype absorption, atomic Backspace,
  Vim insert support, and a clean Undo. Each write is one undo step.
phases:
- id: req-read-twin
  title: Recognizer and layout twins in the cycler
  depends_on: []
  size: medium
  description: 'req-read-twin: port bob-cli''s task_reqs recognizer (RQ) and task_block_layout
    zones, slots, violations, note homes, tidy, scoop, and file-into-NOTES (RL) into
    new plugin-agnostic cycler fragments, copy the RQ/RL fixtures, and pass every
    RQ and RL vector.'
- id: req-write-twin
  title: Writer twin, cross-plugin copies, and the offer-close scope fix
  depends_on:
  - req-read-twin
  size: medium
  description: 'req-write-twin: port the drop/restore/toggle/insert writers (RW) into
    the cycler and pass every RW vector, copy the twin fragments byte for byte into
    nav and Block ID Prompt with a drift test, re-copy offer_close vectors, and make
    successorOfferCloseOpenScopeCount count only real #task subtasks.'
- id: cycler-guards-and-drop
  title: Cycler requirement guards, Alt+[ drop, and Alt+] restore
  depends_on:
  - req-write-twin
  size: medium
  description: 'cycler-guards-and-drop: add the req-line guard beside the bank-row
    guard and consult it at every cycler task gesture, link-target, transclusion,
    and status-writer site with the contract copy, then implement Alt+[ drop (reason
    prompt plus nav Log Recall) and Alt+] restore through the RW twin as one undo
    step each.'
- id: cycler-toggle-and-typing
  title: Ctrl+] note/req toggle, `- [` typing assist, and o/O continuation
  depends_on:
  - cycler-guards-and-drop
  size: medium
  description: 'cycler-toggle-and-typing: route Ctrl+] (command and Vim) through the
    RW toggle with zone placement, the first-req scoop notice, and refusals; add the
    cycler''s CodeMirror typing extension (expansion, autopair, overtype absorption,
    atomic Backspace, Vim insert, isolated Undo) with a real-Obsidian e2e; pin o/O
    req continuation.'
- id: nav-guards-and-owner-card
  title: Nav and Block ID Prompt requirement guards and the owner Task Card
  depends_on:
  - req-write-twin
  size: medium
  description: 'nav-guards-and-owner-card: refuse Alt+N, Alt+F, Ctrl+Alt+F, their
    counted forms, and the other nav task gestures on req lines with the lane copy,
    redirect Ctrl+Shift+P (and the Task Card stage shortcuts) to the owner''s Task
    Card, and guard Block ID Prompt''s Ctrl+Shift+Enter so a req line or carried stub
    is never linked or unlinked.'
- id: gestures-contract-and-e2e
  title: Contract section, cross-plugin e2e, and ship
  depends_on:
  - cycler-toggle-and-typing
  - nav-guards-and-owner-card
  size: small
  description: 'gestures-contract-and-e2e: document the shipped Obsidian gestures
    in bob-cli docs/task-requirements.md, extend the real-Obsidian e2e to every gesture
    across the three plugins, run the full plugin and docs gates, and sync the three
    plugins.'
proposed_by: bbugyi200.athena.bob-cli-on.10
proposed_with: claude/opus
parent_bead: bob-cli-on.10
create_time: 2026-11-21 14:10:14
status: done
bead_id: bob-cli-on.10.1
---

- **PROMPT:** [prompts/202611/obsidian_req_gestures.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/obsidian_req_gestures.md)
- **PARENT:** [202611/task_requirements.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/task_requirements.md)
- **BEAD:** [bob-cli-on.10.1](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-on/bob-cli-on.10.1.md)

# Scope and authority

This child epic implements only the `obsidian-gestures` phase of
`plan:202611/task_requirements.md`, assigned bead `bob-cli-on.10` under the epic
`bob-cli-on`. The parent plan's "Design of record" §1–§5 and its "Ground rules for every
phase" are authoritative; this plan only splits the phase into agent-sized work and pins
the details the parent left to the implementer. Read the parent plan (§1, §2, §3, §5,
and the `obsidian-gestures` phase) and `sase bead read bob-cli-on.10` before starting.
The two research reports behind the parent
(`research:202611/task_reqs_design/task_reqs_design.md` and its amendment
`research:202611/task_reqs_nested_and_notes_impact.md`, which wins on conflict) are
background only; read them through `sase artifact read` if a detail here seems
ambiguous.

The parent epic's decisions are final and are not re-asked:
`notes_drawer_birth = with_first_note` (`NOTES` is created only when a note lands in it;
the RW vectors pin this), `seed_review = yes` (vault-migration's concern, not this
epic's). The memory decisions belong to the parent's `memory` phase: **this child epic
edits no memory note.**

Out of scope (later parent phases): Ctrl+Shift+T carry-now, `api.taskSplit`, picker v6
requirement groups and the cycler close-card verdicts (`obsidian-close-card`,
`bob-cli-on.11`); promotion (`promotion`); anything in bob-cli Rust, Bob Mac Capture, or
the live vault; bob-ledger-tools (its rendering and freshness guards already landed in
`bob-cli-on.9`). Do not edit Cargo.toml's version or CHANGELOG.md.

Workers close only their own child phase bead. The child epic's land agent verifies the
combined scope and closes `bob-cli-on.10`; nobody closes `bob-cli-on` or any other
ancestor. Do not create beads: record discovered work as `PROPOSED FOLLOW-UP:` notes on
your own bead; the land agent copies the ones that still matter onto `bob-cli-on.10` for
the parent land agent.

## Repositories and ground rules

- **bob-plugins** (linked repo): open it with `/sase_repo`
  (`sase repo open bob-plugins -r "<why>"`), read its `AGENTS.md`, and work only at the
  printed path.
  - Edit `src/` fragments, never `main.js`. Every hand-edited fragment stays ≤ 1000
    lines; check `wc -l` before editing. These were ≥ 985 lines at planning time and
    must not grow: cycler `085`, `087`, `089i`, `137`, `160`, `180`; nav `080`, `160`,
    `218`, `280`, `285`, `350`, `426`, `428`, `470`, `480`, `550`, `570`, `677b`, `700`.
    Add new fragments (and list them in that plugin's `src/fragments.json`) instead, and
    touch a full fragment only with a net-zero one-line hook.
  - Install new plugin-method mixins the way `192-bank-row-guard.js` is installed
    (cycler `200-install-methods.js`); export test helpers from a new exports fragment
    (the cycler's `210-exports.js` has room; nav's `700-exports.js` is full, so follow
    nav's `7xx-*-exports.js` pattern, and bob-ledger-tools' `351-exports-task-reqs.js`
    shows the shape).
  - Run `npm run build`, `npm test` (which includes `build:check` and the docs inventory
    check), and `npm run docs:check`. Add every new `scripts/test-*.cjs` to the
    `test:unit` list in `package.json`.
  - Deploy only the plugins you changed with `bob plugins sync -p <id>`; never pass `-F`
    unprompted (follow `AGENTS.md` if sync refuses). Follow the manifest-version
    convention in `docs/develop/deployment.md`.
  - Generated files (`main.js`, the docs inventory) are rebuilt, never hand-merged: if a
    rebase conflicts in them, regenerate with `npm run build` / `npm run docs:gen`.
- **bob-cli** (this workspace): only the last phase edits it (docs). Gate with
  `just docs-check` and follow the `docs.md` reference memory.
- Rust is the authority. The JS twins mirror bob-cli and pass bob-cli-owned vectors
  copied byte for byte; a behavior difference is a twin bug, never a vector edit. If a
  vector looks wrong, record a `PROPOSED FOLLOW-UP:` and match bob-cli anyway.
- No silent vault rewrite: code changes only what a gesture writes.

# Current evidence (planning snapshot)

Planning inspected bob-cli `532df825` and bob-plugins `cdd9df7`. Recheck both before
coding; sibling phases land meanwhile. `sase bead epic-symbols bob-cli-on.10` reports no
entries.

bob-cli (the authority to mirror):

- `src/native/task_reqs.rs`: `req_forest`, `note_req_forest`, `owner_of_line`,
  `is_req_line`, `req_path`, `req_row_key`, `checkbox_mark_without_filter`,
  `MAX_REQ_DEPTH = 16`.
- `src/native/task_block_layout.rs` plus `task_block_layout/{edit,tidy,writers}.rs`:
  zones, `ChildKind`, `insertion_offset`, `TaskBlock::read`, `plan_tidy`,
  `scoop_loose_notes`, `file_into_notes`, and the writers `drop_req`, `restore_req`,
  `toggle_note_req`, `insert_reqs` with their refusal codes and copy.
- Helpers those call:
  `capture::{line_spans, list_item_body, nearest_shallower_list_item_parent, dominant_indent_unit, first_child_indentation, leading_spaces_or_tabs_len, document_line_ending}`,
  `capture_drawers::is_drawer_title`, `task_children::classify_task_child_with_filter`,
  `task_dependencies::blocks::is_cancel_log`.
- Fixtures:
  `tests/fixtures/task_reqs/{README.md, vectors.json (RQ1–RQ21), layout.json (RL1–RL52), writes.json (RW1–RW53)}`
  and `tests/fixtures/offer_close/vectors.json` (now with SV25: open reqs are never open
  subtasks). Contract: `docs/task-requirements.md` §1–§5, §10.

bob-plugins:

- **Cycler** (`plugins/task-status-cycler/src/`): the bank-row guard
  `192-bank-row-guard.js` (`cyclerBankInfoAtCursor`, `cyclerBankTransclusionIsBank`,
  `cyclerBankLineKind`) is consulted at `150:14` (cycle), `140:137` (Vim Ctrl+Enter),
  `140:326` (Vim Ctrl+]), `140:340` (Vim Ctrl+Shift+]), `160:140` (Ctrl+Enter),
  `160:172` (Ctrl+]), `160:196` (Ctrl+Shift+]), `160:304` (`completeTaskAtCursor`),
  `160:451`, `160:828` and `190:263` (transclusion targets), `180:11`, `180:434`,
  `180:481` (status writers), `180:594` (`getActiveObsidianTaskToggle`), and `150:643`.
  `TASK_LINE_RE` / `getTaskStatusForLine` (`010`) match any checkbox.
  `getTaskCheckboxMarkerToggle` (`090:410`) and `getActiveTaskCheckboxMarkerToggle`
  (`180:550`) are today's Ctrl+]. `getOpenLineBelowPrefix` (`010:96`) already continues
  every checkbox line with `- [ ] `. `applyDocumentLineReplacement` (`180:277`) writes a
  whole-note change as one `replaceRange` (one undo step).
  `successorOfferCloseOpenScopeCount` (`089h:40`) counts every open checkbox under the
  task. Command IDs: `cycle-task-status-backward/forward` (Alt+[ / Alt+]),
  `toggle-task-checkbox-marker` (Ctrl+]), `toggle-task-open-done` (Ctrl+Enter, plus the
  Vim mapping), `toggle-obsidian-task` (Ctrl+Shift+]); counted cycles come from
  `registerCountedTaskCycleInputListeners`. The cycler has no CodeMirror input extension
  yet; it already requires `@codemirror/view`. bob-chips' `010-load.js` shows the
  defensive `@codemirror/commands` / `@codemirror/history` `isolateHistory` load.
- **Nav** (`plugins/bob-navigation-hotkeys/src/`): `toggleTaskLane` (`510:310`, Alt+N),
  `refreshTaskFreshness` (`520:309`, Alt+F / Ctrl+Alt+F), the counted capture-phase
  fallback (`540:304`), `openBulletPropertyPicker` (`550:258`, Ctrl+Shift+P; the
  managed-line branch already moves the cursor to the owning task with
  `setEditorCursorSafely` before reopening), `getNavigationLogRecallApi` and
  `api.logRecall.attach({inputEl, kind, bandHost, …})` (`184`). Nav task predicates
  (`isObsidianTaskLine`, `isObsidianTaskAtLine`) already require `#task`, so req lines
  refuse today with generic "not a task" copy.
- **Block ID Prompt** (`plugins/block-id-prompt/src/`): Ctrl+Shift+Enter is its
  `link-task-to-pomodoro` command, `openPomodoroTaskLink` (`120:12`). On a non-task line
  holding a Task Link it runs task-link mode, which deletes that link and reopens the
  linked task — on a carried stub `- [>] [[#^id|Alias]]` that would destroy the stub's
  link.
- **bob-ledger-tools** `139-task-reqs.js` is a display-only RQ subset (no `path`,
  `subtree_end`, note ranges, or row keys); it is not changed here.
- Vault hotkeys (`~/bob/.obsidian/hotkeys.json`, read-only reference): Ctrl+] →
  `toggle-task-checkbox-marker`, Alt+[ / Alt+] → cycle backward/forward, Ctrl+Shift+] →
  `toggle-obsidian-task`, Ctrl+Shift+P → nav `set-bullet-property`.

# Shared design

## The twins

One plugin-agnostic twin set, written once and copied byte for byte:

| Twin | Mirrors                                      | Copies                       |
| ---- | -------------------------------------------- | ---------------------------- |
| RQ   | `task_reqs.rs`                               | cycler, nav, Block ID Prompt |
| RL   | `task_block_layout.rs`, `edit.rs`, `tidy.rs` | cycler, nav                  |
| RW   | `writers.rs`                                 | cycler, nav                  |

- Fragments contain only pure functions and constants: no `this`, no Obsidian or
  CodeMirror imports, no clock (dates are parameters), no plugin helpers. Split them so
  each stays ≤ 1000 lines (for example RQ, RL core, RL tidy/edit, RW). Use the
  `taskReq*` prefix for RQ (and keep bob-ledger-tools' row field names — `line`,
  `lineIndex`, `depth`, `parent`, `owner`, `ownerLine`, `ownerText`, `ownerName`,
  `mark`, `markChar`, `state`, `text`, `anchor`, `depth_exceeded`, `excludedBy`,
  `excludedBecause`, `markerFrom`, `markerTo` — as a strict subset, so ledger-tools
  could adopt the full twin later), `taskBlock*` for RL, and `taskReqWrite*` or
  `taskReq{Drop,Restore,Toggle,Insert}` for RW. Grep each target plugin first; none of
  these names exist in the cycler, nav, or Block ID Prompt today.
- The global filter defaults to `#task` (whole-token match, as `note_tasks` does); an
  optional `{ globalFilter }` option exists for tests.
- Writers take the whole note, a target `{ lineIndex, preimage }`, and (for drop) a
  `YYYY-MM-DD` date and a reason. They return `{ ok: true, updated, … }` with the RW
  fields (`lineIndex`, `log`, `descendants`, `direction`, `depth`, `scooped`,
  `notesCreated`, `firstLine`, `lineCount`) or `{ ok: false, code, message }` with
  bob-cli's exact codes and copy. They preserve per-line endings and the final-newline
  state (the CRLF vectors). The toggle additionally reports an additive `firstReq: true`
  when the owner owned no req before the write.
- Row keys need SHA-256: `taskReqRowKey(ownerKey, row, digest8)` takes an injected
  synchronous 8-hex digest function so the fragment stays pure. Tests inject Node's
  `crypto`; runtime use belongs to `obsidian-close-card`.
- Fixture copies, each with bob-cli's `README.md` stating "update bob-cli first, then
  copy": `plugins/task-status-cycler/scripts/fixtures/task_reqs/` and
  `plugins/bob-navigation-hotkeys/scripts/fixtures/task_reqs/` get all four files;
  `plugins/block-id-prompt/scripts/fixtures/task_reqs/` gets `README.md` and
  `vectors.json`.

## Which lines are reqs, and which gestures do what

The guard asks the RQ twin about the cursor line in the editor's current text. A line is
a **req line** exactly when it is a node of `taskReqForest` (so fenced, quoted,
frontmatter, drawer, note-nested, and managed-line checkboxes never are). Cache the
forest per document text (reuse it while `editor.getValue()` returns the same string)
and prefilter cheaply: only a checkbox list item without `#task` can be a req, so most
calls never build a forest. Guards fail open to today's behavior only on an internal
error, never on a recognized req.

Copy (exact strings; `{Owner}` is the owner's description as the writers render it,
`{T}` the req's clean text):

| Situation                                                                                              | Copy                                                                                                                                |
| ------------------------------------------------------------------------------------------------------ | ----------------------------------------------------------------------------------------------------------------------------------- |
| Ctrl+Enter on a req; Alt+] on a pending req                                                            | `Requirements are met when “{Owner}” closes — Ctrl+Enter on the task`                                                               |
| Any refused gesture on a carried `[>]` stub                                                            | `Carried — open the linked task`                                                                                                    |
| Alt+N, Alt+F, Ctrl+Alt+F, Ctrl+Shift+], Ctrl+Shift+Enter, counted cycles, other lane/schedule gestures | `Requirements have no lane or schedule — Ctrl+Shift+T makes it a task`                                                              |
| Alt+[ prompt placeholder                                                                               | `Why drop it? (optional)`                                                                                                           |
| Drop succeeded                                                                                         | `⊘ Dropped “{T}”`, plus ` · {k} nested go with it` when `k > 0`                                                                     |
| Restore succeeded                                                                                      | `↺ Restored “{T}”`                                                                                                                  |
| Ctrl+] gave a task its first req                                                                       | `⌗⌗ First requirement · filed {n} note(s) under NOTES` (`1 note` / `{n} notes`); just `⌗⌗ First requirement` when nothing was filed |
| Any writer refusal                                                                                     | the writer's `message` verbatim                                                                                                     |

The Ctrl+Shift+T in the lane copy is the contract's final wording; until
`obsidian-close-card` lands, Ctrl+Shift+T on a req runs today's split, which also makes
it a task.

Every write gesture: compute the whole-note result with the twin, refuse on any
`{ ok: false }` with its copy and no write, otherwise apply through one
`applyDocumentLineReplacement` (one undo step), place the cursor on the result line,
then show the notice.

# Phase details

## req-read-twin (cycler, medium)

1. Port RQ: list-item parsing, checkbox marks, whole-token filter, fence (including
   fences inside list items), quote, and frontmatter exclusion, structural containers
   (drawer titles, managed links, logs, Cancel Log), nearest-shallower-parent ownership,
   depth and the 16 bound (`depth_exceeded`), the derived state table and Excluded rule,
   clean text and trailing `^anchor`, `path`, `subtree_end`, direct note ranges,
   `owner_of_line`, `is_req_line`, `req_path`, and `req_row_key`.
2. Port RL: zones, child kinds and extents, `insertionSlot` / `zoneOf`,
   `layoutViolations`, `noteHome`, `planTidy` (whole subtrees, authored order kept,
   longest-increasing-run move count, the NOTES rule, `null` when canonical or not
   open), and the shared `scoopLooseNotes` / `fileIntoNotes` and line-preserving edit
   helpers (`physical_lines`, endings, `rebase_indent`, `insert_lines`, `pull_ranges`).
   Mirror `is_drawer_title` and the `task_children` managed predicates exactly.
3. Copy `vectors.json`, `layout.json`, and `README.md` into the cycler's
   `scripts/fixtures/task_reqs/`. Add
   `scripts/test-task-status-cycler-task-reqs-vectors.cjs` that runs every RQ case (all
   `expect.nodes` fields, `probes`, and `assert_distinct_keys` with Node `crypto`) and
   every RL `slots`, `violations`, `tidy` (including "tidy of its own output is
   `null`"), and `note_homes` case through the built plugin's exported helpers.
4. No gesture changes. Gate: `npm run build`, `npm test`.

## req-write-twin (cycler, nav, Block ID Prompt; medium)

1. Port RW (`drop`, `restore`, `toggle`, `insert`) with the preimage check
   (`req_stale`), every refusal code and message, Cancel Log creation/prepend with the
   `🤷 no reason given` rule, the depth-1 move-to-reqs-zone with the first-req scoop,
   req→note in place at depth ≥ 2 and into `NOTES` at depth 1, the nested-req refusal,
   and `insert` with the depth bound. Copy `writes.json`; extend the vector test to run
   every RW case.
2. Copy the twin fragments byte for byte into nav and (RQ only) Block ID Prompt, add
   them to each `fragments.json`, export their helpers, and copy the fixtures there. Add
   `scripts/test-task-reqs-twin-copies.cjs`: it asserts the nav and Block ID Prompt
   fragments are byte-identical to the cycler's, every plugin copy of each fixture is
   byte-identical (including bob-ledger-tools'
   `scripts/fixtures/task_reqs/vectors.json`), and that the built nav and Block ID
   Prompt bundles expose working twins (a few RQ/RW smoke cases through each plugin's
   existing test harness or a `vm` load of its `main.js`).
3. Offer-close scope: re-copy bob-cli's `tests/fixtures/offer_close/vectors.json` (and
   its README if it changed) into the cycler's `scripts/fixtures/offer_close/`. Fix
   `successorOfferCloseOpenScopeCount` so the subtask path counts only open lines that
   carry the global filter (matching bob-cli's `open_scope_count`, which counts parsed
   tasks only); confirm the project path already counts only real tasks. SV25 and every
   existing SV case must pass in the existing offer-close tests.
4. Gate: `npm run build`, `npm test`. Sync `task-status-cycler`,
   `bob-navigation-hotkeys`, and `block-id-prompt` (the twins are inert until later
   phases call them, but the deployed bundles must match the repo).

## cycler-guards-and-drop (cycler, medium)

1. **Guard mixin.** New fragment after `192` (for example `193-req-line-guard.js`) with
   `cyclerReqInfoForText(text, lineIndex)`, `cyclerReqInfoAtCursor(editor)` (shaped like
   `cyclerBankInfoAtCursor`, returning the RQ row plus owner name or `null`),
   `cyclerReqTargetIsReq(noteText, lineIndex)` for link and transclusion targets, and
   `cyclerReqRefuse(info, gesture)` that picks the copy above. Bank checks stay first
   wherever both apply.
2. **Consult it at every site in "Current evidence"**, re-found through the calls if
   they drifted:
   - Ctrl+Enter (command `160:140` and Vim `140:137`, before
     `claimReviewWalkCtrlEnter`): owner copy (carried copy on stubs), no write; in
     checking mode return `true` so the hotkey is consumed.
   - Ctrl+Shift+] (`160:196`, Vim `140:340`, `getActiveObsidianTaskToggle` `180:594`,
     and the counted form): lane copy. A counted range that includes any req refuses
     whole, before any prompt.
   - Counted Alt+[ / Alt+] (`registerCountedTaskCycleInputListeners`): lane copy when
     any target is a req.
   - Link and transclusion targets (`160:828`, `190:263`, and the Pomodoro Task Link
     close path in `170`): a Task Link or embed whose target line is a req never writes
     it; show the owner copy (carried copy for stubs) using the target note text the
     resolver already holds. The recursive transcluded tree close
     (`completeResolvedTranscludedTaskTargetTree`) never rewrites a checkbox on a line
     without the global filter, matching bob-cli's reader-hardening fix to
     `finish_close_node`.
   - Status writers as a backstop (`setActiveCheckboxStatus`, the open/done toggle, and
     the `180:434` / `180:481` paths): refuse a req line with no write.
     `completeTaskAtCursor` returns `{ ok: false, reason: "requirement" }` on a req.
   - Keep `getTaskStatusForLine` / `TASK_LINE_RE` unchanged (single-line, no document
     context); make `getActiveTaskStatus` callers req-aware through these explicit
     guards rather than by returning `null`, because a `null` status would send a req
     line into the non-checkbox bullet-formatting cycle.
   - Leave Ctrl+] (`160:172`, `140:326`) to the next phase.
3. **Alt+[ drop / Alt+] restore.** In `handleCycleCommand`'s req branch (one hook line
   in `150`, logic in a new fragment):
   - Alt+[: on a carried stub show the carried copy. Otherwise dry-run the drop with an
     empty reason; on refusal show its message (`Already dropped`, `Already met`,
     `Its parent requirement was dropped — restore it first`, …). On success open a
     small cycler modal (new fragment; `Modal` is already imported) titled with the
     req's text, input placeholder `Why drop it? (optional)`, with nav's Log Recall band
     attached via `api.logRecall.attach({ kind: "cancel", … })` when nav exposes
     `logRecall` v ≥ 1 (plain input otherwise). Enter submits (an empty reason is fine);
     Esc and Ctrl+[ close without writing, while `q` types into the reason field
     (`decisions:q-and-ctrl-bracket-mirror-esc`: q types only in text controls). On
     submit re-read the editor, run `taskReqDrop` with the preimage captured when the
     gesture began and today's local date, apply it as one undo step, keep the cursor on
     the req, show the drop notice, and call `logRecall.noteUsed("cancel", reason)` for
     a nonempty reason. A stale line shows `The line changed — try again`.
   - Alt+]: run `taskReqRestore`; success shows the restore notice; a pending req shows
     the owner copy (the writer's `req_not_dropped` message).
   - Both work under open, done, and cancelled owners exactly as the writers allow.
4. **Tests.** `scripts/test-task-status-cycler-req-guards.cjs` (model-level, through the
   harness, one case per guarded gesture and site: req lines stay byte-identical, the
   right copy shows, `#task` lines, plain checklists outside tasks, checklists in
   `NOTES`, fenced lookalikes, and bank rows keep today's behavior) and
   `scripts/test-task-status-cycler-req-drop.cjs` (drop with and without a reason, with
   an existing log, nested descendants count, excluded and carried refusals, restore,
   staleness between prompt and submit, Esc writes nothing, single `replaceRange` per
   write, Log Recall attach and `noteUsed` with a fake nav).
5. **Docs and ship.** Add a "Requirement lines" section to `docs/task-status-cycler.md`
   and a short manifest-description clause; `npm run docs:gen`, `npm test`,
   `npm run docs:check`; sync `task-status-cycler`.

## cycler-toggle-and-typing (cycler, medium)

1. **Ctrl+] routing** (`handleToggleCheckboxMarkerCommand`, Vim
   `handleVimToggleCheckboxMarker`; logic in a new fragment, one-line hooks):
   - The cursor line is a req → `taskReqToggle` (req → note).
   - Else the cursor line is a list item without a checkbox, outside fences, quotes, and
     frontmatter, whose structural ancestor chain reaches a real `#task` line →
     `taskReqToggle` (note → req). Its refusals (`req_in_drawer`, `req_parent`,
     `not_a_note`, `req_depth`, `req_has_nested`, `req_dropped`, `req_carried`) show
     their messages with no write.
   - Everything else keeps today's `getTaskCheckboxMarkerToggle` behavior unchanged:
     list items outside any task block, `#task` lines (including nested subtasks), and
     checkboxes that are not reqs (pasted checklists).
   - Apply as one undo step; land the cursor on the writer's result line at the same
     offset into the item body (clamped). Show the first-req notice when `firstReq`.
     Depth-1 moves (into the reqs zone, or into `NOTES`) follow the twin exactly.
2. **Typing assist** — the cycler's first CodeMirror extension, registered in `onload`
   through `registerEditorExtension` and built in new fragments with a pure planner
   (unit-testable) plus a thin CM adapter. Load `Prec`, `StateField`, `StateEffect` from
   `@codemirror/state`, `keymap` from `@codemirror/view`, and `isolateHistory`
   defensively (bob-chips' pattern); if any is missing, skip the extension silently.
   - **Expansion.** On a typed single space with one empty selection, when the text
     before the caret is a list-item start ending in a lone `[`
     (`^[ \t]*(?:[-+*]|\d+[.)])[ \t]+\[$`) and the text after the caret is empty or a
     lone auto-paired `]`, the line becomes `…- [ ] ` with the caret after it (`- [|]` +
     space → `- [ ] |`). It never fires on `- [x`, `- [[`, `- [link`, in quotes, fenced
     code, or frontmatter, or with other text after the caret. The rule handles both
     `autoPairBrackets` settings without reading the setting.
   - **Undo.** Let the space land as its own ordinary input transaction, then dispatch
     the expansion as a separate transaction annotated with `isolateHistory`, so one
     Undo returns exactly the bytes a plain space would have produced (`- [ ` or, with
     autopair, `- [ ]` with the caret before `]`).
   - **Overtype absorption.** The expansion arms a state field at the caret. A `]` typed
     immediately there is swallowed, and a space typed immediately after that `]` is
     swallowed too; any other keystroke, edit, or caret move disarms it. Typing `- [ `
     and typing `- [ ] ` in full both give `- [ ] `.
   - **Atomic Backspace.** A `Prec.high` Backspace binding: with one empty selection
     right after a list-item-start `[ ] ` (open mark only), delete the four bytes and
     leave `- `, on any such line, outside fences and frontmatter. Raise to
     `Prec.highest` only if the e2e shows Obsidian's list keymap winning.
   - **Vim.** Insert-mode typing and Backspace go through the same extension; normal
     mode is untouched.
   - Paste, templates, and programmatic edits never trigger (only user input events).
3. **o / O continuation.** `getOpenLineBelowPrefix` already emits `- [ ] ` at the same
   indentation for every checkbox line; add unit tests that `o`/`O` on `[ ]`, `[-]`,
   `[>]`, `[x]`, and unknown-mark req lines continue with `- [ ] ` and never copy `[-]`
   or `[>]`. Change code only if a test fails.
4. **Tests.** `scripts/test-task-status-cycler-req-toggle.cjs` (both directions at depth
   1 and ≥ 2, the scoop with and without notes and its notice, `NOTES` birth only with a
   note, every refusal, the legacy paths unchanged, one `replaceRange`, cursor landing)
   and `scripts/test-task-status-cycler-req-typing.cjs` (planner cases for every fire
   and no-fire input, absorption sequences, Backspace, and an adapter test with a fake
   `EditorView` proving two transactions with the second isolated). Add a real-Obsidian
   e2e `scripts/obsidian-task-reqs-e2e.cjs` (isolated scratch vault and profile, owned
   Obsidian process, CDP — follow `obsidian-vim-yank-flash-e2e.cjs` and its
   `test-obsidian-*-e2e.cjs` wrapper convention) covering expansion with
   `autoPairBrackets` on and off, absorption, atomic Backspace, Vim insert mode, and one
   Undo. Run it under `xvfb-run`; if this host cannot run Obsidian, say so in your bead
   note instead of skipping silently.
5. **Docs and ship.** Extend the cycler docs section (Ctrl+], typing, o/O), the manifest
   clause, `npm run docs:gen`, `npm test`, `npm run docs:check`; sync
   `task-status-cycler`.

## nav-guards-and-owner-card (nav, Block ID Prompt; medium)

1. **Guard helper.** A new nav fragment (for example `258-req-line-guard.js`) with
   `navReqInfoAtLine(content, lineIndex)` over the RQ twin and the lane/carried copy.
2. **Refusals with the lane copy, no write** (carried stubs get the carried copy): Alt+N
   `toggleTaskLane`, Alt+F / Ctrl+Alt+F `refreshTaskFreshness`, and their counted forms
   (a counted batch with any req target refuses whole, with that copy). Then audit nav's
   other cursor-line task gestures (for example Alt+P quick roll and Ctrl+Shift+M move):
   wherever one refuses a req line as "not a task" today, show the lane copy instead,
   and if any would write to a req line, refuse it. Never change behavior on `#task`
   lines or non-req lines. Leave Ctrl+Shift+T (split) untouched; it belongs to
   `obsidian-close-card`.
3. **Owner Task Card.** In `openBulletPropertyPicker` (one net-zero hook line in the
   full `550`, logic in a new fragment), when there is no `taskSession` and the cursor
   line is a req, move the cursor to the owner line with `setEditorCursorSafely` and
   reopen with the same options, exactly like the managed-line redirect. This covers
   Ctrl+Shift+P and the Task Card stage shortcuts that route through it (Ctrl+Shift+B,
   Ctrl+Shift+Alt+B, Ctrl+Shift+L, Ctrl+Shift+Alt+L); verify each in `427` and route any
   that bypass it the same way. This runs before the Task Link branch, so a carried stub
   also opens its owner's card, as the parent table says for every req line.
4. **Block ID Prompt Ctrl+Shift+Enter.** At the top of `openPomodoroTaskLink`, before
   the review-walk capture and before `resolvePomodoroLinkTaskFromEditor` /
   `startTaskLinkOpen`, a req line (including a carried stub) refuses with the lane copy
   (carried copy for stubs) and writes nothing, so task-link mode can never delete a
   stub's link.
5. **Tests.** `scripts/test-navigation-req-guards.cjs` (each gesture refuses with the
   right copy and leaves the note byte-identical; `#task` and Task Link behavior
   unchanged; owner redirect lands on the owner line and opens the owner's card for a
   depth-1, a depth-3, and an excluded req) and
   `scripts/test-block-id-prompt-req-guard.cjs` (req and carried-stub refusal, no write,
   no task-link mode; `#task` lines unchanged).
6. **Docs and ship.** Update `docs/bob-navigation-hotkeys.md`, `docs/task-card.md`,
   `docs/block-id-prompt.md`, and the manifest clauses; `npm run docs:gen`, `npm test`,
   `npm run docs:check`; sync `bob-navigation-hotkeys` and `block-id-prompt`.

## gestures-contract-and-e2e (bob-cli docs, bob-plugins e2e; small)

1. **Contract.** In bob-cli `docs/task-requirements.md` §5, add an "Obsidian gestures"
   subsection with the shipped gesture table (Ctrl+], Alt+[, Alt+], Ctrl+Enter,
   Ctrl+Shift+P, the lane-copy refusals, Ctrl+Shift+Enter, o/O, and the typing assist
   with its Obsidian-specific scope: any list-item start, autopair, Vim insert, Undo),
   marked "Shipped in the `obsidian-gestures` phase", leaving Ctrl+Shift+T to
   `obsidian-close-card`. In §10, note that the cycler, nav, and Block ID Prompt run the
   RQ/RL/RW vectors through byte-identical twins. Link the plugin reference pages. Gate:
   `just docs-check` (and `just docs-gen` if the docs tooling requires it).
2. **Cross-plugin e2e.** Extend `scripts/obsidian-task-reqs-e2e.cjs` with one scenario
   per gesture on a task block with reqs at three depths, notes between branches, and a
   `NOTES` drawer: Ctrl+] first-req scoop then Undo, Alt+[ drop with a reason then Undo,
   Alt+] restore, Ctrl+Enter refusal leaving bytes unchanged, Ctrl+Shift+P opening the
   owner's card, Alt+N / Alt+F refusals, and Ctrl+Shift+Enter on a carried stub writing
   nothing. Run it under `xvfb-run` (report honestly if the host cannot).
3. **Ship.** Full `npm test` and `npm run docs:check`; sync `task-status-cycler`,
   `bob-navigation-hotkeys`, and `block-id-prompt`; record any follow-ups (for example
   bob-ledger-tools adopting the full RQ twin) as `PROPOSED FOLLOW-UP:` notes.

# Acceptance (the land agent checks these)

- The cycler, nav, and Block ID Prompt twins are byte-identical, and every RQ, RL, and
  RW vector passes in `npm test`; SV25 passes and the offer-close scope note never
  counts reqs.
- On a req line no cycler, nav, or Block ID Prompt gesture writes a status, a lane, a
  freshness stamp, a Task Link, or a completion; each refusal shows its contract copy.
  `#task` lines, plain and pasted checklists, bank rows, managed lines, and Task Links
  behave exactly as before.
- Ctrl+], Alt+[, and Alt+] each write exactly the twin's bytes as one undo step, honor
  `with_first_note`, and refuse stale lines.
- Typing `- [` then space (autopair on or off, Vim insert included) gives `- [ ] `,
  absorption and atomic Backspace work, and one Undo gives back the plain-space bytes.
- Ctrl+Shift+P on a req opens the owner's Task Card.
- bob-cli `docs/task-requirements.md` documents the shipped gestures; `just docs-check`,
  `npm test`, and `npm run docs:check` pass; the three plugins are synced.
- `sase bead epic-symbols bob-cli-on.10` has no leftovers. The land agent then closes
  `bob-cli-on.10` only, copying still-relevant child `PROPOSED FOLLOW-UP:` notes onto
  it.
