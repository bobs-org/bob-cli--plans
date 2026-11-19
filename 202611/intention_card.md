---
tier: epic
title: Complete the intention card and editor
goal:
  Daily and period notes show the intention property card, all approved card-editor
  gestures write through guarded bob updates, and the plugins are deployed with real
  Obsidian acceptance evidence for bob-cli-nh.7.
phases:
  - id: display
    title: Intention card model, property-card host, and visual previews
    size: medium
    depends_on: []
    description:
      "display: implement the canonical subject and metadata models, digest parity,
      placement anchors, and intention display on the landed property-card host; include
      parent and recent-closed lines, fail-open ownership, unit coverage, and reviewed
      light/dark/narrow previews. Build, validate, and sync plugins."
  - id: editor
    title: Guarded intention card editor and field stages
    size: medium
    depends_on:
      - display
    description:
      "editor: adapt createGuardedUpdateSession to bob periods update, build the browse
      modal and intention/reflection/summary stages, register the intentions card kind,
      and implement add/edit/reorder/remove/Undo with preview/apply, draft retention,
      and overlays. Test real writer round trips, build, and sync."
  - id: navigation
    title: Period navigation, YAML excursion, command entry points, and docs
    size: medium
    depends_on:
      - editor
    description:
      "navigation: add in-place parent and sibling navigation, missing-note states, the
      intentions YAML excursion, Edit intentions and Reflect on yesterday
      palette/Command Mode entries, and the non-Vim fallback. Document the complete card
      in docs/intention-card.md and nav; test, regenerate, and sync plugins."
  - id: acceptance
    title: Real Obsidian acceptance and final plugin deployment
    size: medium
    depends_on:
      - navigation
    description:
      "acceptance: add the isolated intention-card CDP runner and offline wrapper, prove
      mounting, both yesterday gestures, field edits/Undo, navigation/YAML, stale
      drafts, and fail-open behavior with a current real bob binary. Run regression and
      docs gates, inspect screenshots, deploy and verify provenance, and provide
      closeout evidence for bob-cli-nh.7 without closing ancestors."
proposed_by: bbugyi200.athena.bob-cli-nh.7
proposed_with: codex/gpt-6-astra
parent_bead: bob-cli-nh.7
create_time: 2026-11-19 18:17:35
status: wip
---

- **PROMPT:**
  [prompts/202611/intention_card.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/intention_card.md)
- **PARENT:**
  [202611/period_intentions.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/period_intentions.md)

# Complete the intention card and editor

This is a child implementation epic for **bob-cli-nh.7**, the `card` phase of
`plan:202611/period_intentions.md`. Its outcome is a deployed intention card on daily,
weekly, quarterly, and yearly notes, with the complete keyboard editor and the approved
acceptance evidence. The child epic's land agent owns the delegated closeout of
bob-cli-nh.7 after its phases pass. Phase workers close only their own assigned phase.
Nobody here closes bob-cli-nh or any other ancestor.

## Already landed and scope boundaries

Read the assigned bead and its final DECISIONS through `sase bead read`, and the parent
plan through `sase artifact read`, before implementation. Supporting design:
`research:202611/period_intentions_card/period_intentions_card.md`.

Planning inspected bob-cli `eaa52d06` and bob-plugins `db89aec`. The assigned bead is in
progress and has no prior implementation/remainder notes. Its dependencies are closed:

- bob-cli-nh.2 delivered `bob periods update` (implementation `a0b91a8d`), including
  preview, owned-field fingerprints, inverse operations, and day subjects.
- bob-cli-nh.6 delivered the instance-owned registry, selective property ownership,
  shared host lifecycle, and shared chrome. Its child epic is closed. The phase note
  documents deployment of ledger-tools 1.140.0 and the acceptance baseline.
- No intention-card implementation was found in the plugin source. Do not redo either
  dependency. At planning time `sase bead epic-symbols bob-cli-nh.7` reports no entries;
  recheck at completion.

Four bounded implementation phases are appropriate: the metadata display, the
writer-backed modal, the navigation/entry integration, and the real application proof
each have separate acceptance criteria. They are sequential because they share plugin
lifecycle, fragments, CSS, generated bundles, and deployment files. Each phase is
`medium` direct implementation work; no phase needs to redesign the kit or the Rust
writer.

Implementation is primarily in **bob-plugins**. Open it from the assigned bob-cli
workspace with `/sase_repo`, use only the returned checkout, and read its `AGENTS.md`,
`docs/develop/architecture.md`, `validation.md`, and `deployment.md`. Paths below are
relative to that plugin checkout unless explicitly marked bob-cli. Use the bob-cli
checkout for contract fixtures and the real test binary. Never edit installed plugin
source in the vault.

Inherited decisions are final: daily summaries use the sixty-word branch; glossary
changes belong to bob-cli-nh.9; the two proposed decision records were declined. **This
child epic edits no memory.** It does not migrate the vault, change GTD rows or chezmoi,
write intentions/reflections for Bryan, alter summary drafting, change release versions
in bob-cli, or implement Mac capture UI. Those are different phases or explicit
follow-ups. Plugin manifest versions follow the plugin repository's normal deployment
convention.

## Existing extension points and contract details

- `240b-plugin-property-card-ownership.js`, `240c-plugin-property-card-host.js`,
  `240d-plugin-property-card-host-editor.js`, and `248m-plugin-schema-double-equals.js`
  provide the finished platform. Register `{kind: "intentions", match, open}`; preserve
  bank → album → record → extensions and every existing Vim refusal. A host adapter
  supplies model, DOM, anchors, refresh, and ownership slots. Its `allowEmbedDisplay`
  must be false. Do not refactor the three existing hosts.
- Reuse `dayStripDayForPath` in `239-plugin-day-strip-hosts.js` and
  `periodNotesPeriodForPath` plus ISO-week helpers in `292b-plugin-period-notes.js`.
  Compare canonical paths against bob-cli `src/native/intentions/subject.rs`: do not
  accidentally admit arbitrary nested day-like paths that the writer cannot resolve.
  Bob's quarters are groups of ISO weeks; its year is an ISO week-year.
- `242aa-plugin-guarded-update-session.js` is the session kernel;
  `242l-plugin-record-update-service.js` and `242m-plugin-record-update-session.js` are
  service/adapter examples, not report schemas to copy unchanged.
- `241xc-plugin-card-close-keys.js`, `241xa`/`241xb` arrange helpers,
  `242e-plugin-album-yaml-excursion.js`, `albumSchemaSaveOpenEditors`, and
  `recordSheetPushOverlay` supply the interaction precedents. Lifecycle setup is in
  `170-plugin-lifecycle.js`; register new mixins and exports through the current build
  installation pattern and `src/fragments.json`.

The Rust contract is already documented in bob-cli `docs/periods.md` and implemented in
`src/native/periods/update.rs` and `src/native/intentions/protocol.rs`:

```text
bob periods update <SUBJECT> -i - -f json [-n]
```

Initial read is a dry run with `{"schema_version":1,"ops":[]}` and **no expect**. Do not
use the record writer's `-d` dry-run flag; periods uses `-n`. All mutations carry the
current `expect`. Reports contain `ok`, `schema_version`, `subject` (`id`, `kind`,
`label`, `path`), `dry_run`, `fingerprint_before`, `fingerprint_after`, `before`,
`after`, boolean `changed`, `inverse_ops`, and `warnings`. Field objects contain
intentions, summary, summary state, and reflection. Errors have
`error.code/message/hint` and optional operation information; decode the actual periods
shape and process exit status.

Operations are `intentions.set/insert/replace/move/remove`, `reflection.set/unset`, and
`summary.set/unset`. Empty strings use unset/remove, not invalid `.set` operations.
Store inverse operations as opaque JSON: Undo can include `summary.restore`, which
restores the original stamps. Summary edits remove both stamps. The writer does not
commit. Never use `processFrontMatter`, text replacement, or a plugin YAML serializer
for field writes.

## Phase display: model, rendered card, and previews

Add appropriately split intention model/render/host fragments, CSS under
`.bob-property-card` plus a species class, exports for tests, and explicit unit test
membership in `package.json`. Keep each handwritten fragment below 1000 lines.

1. Model canonical day/week/quarter/year subjects, labels, parent, siblings, inclusive
   span boundaries, and open/closed state using the existing date conventions. Inject
   today's date into pure helpers so DST, ISO W53, quarter rollover, and
   December/January boundaries are deterministic in tests. `all_time` and noncanonical
   notes never qualify. Year has no editable parent.
2. Read the three fields and stamps through metadataCache. Display must never spawn bob.
   Absent/empty values are empty; nonempty values must satisfy their types (intentions
   is an array of nonempty one-line strings; summary is a one-line string; reflection is
   a string). Do not silently coerce malformed authored values to a clean card.
   Unavailable/malformed frontmatter keeps native Properties visible. Where
   metadataCache alone cannot distinguish malformed YAML or stale successful parsing,
   use the host's current source/parse evidence to fail open, without a bob subprocess.
   Preserve Markdown and reflection paragraphs.
3. Implement summary provenance using Node crypto and the exact normalization recipe in
   bob-cli `docs/periods.md`: CRLF/lone CR to LF, trailing whitespace stripped per line,
   joined text trimmed, SHA-256 UTF-8, first 16 lowercase hex characters prefixed
   `sha256:`. Pin all vectors in `tests/fixtures/intentions/digest-vectors.json` from
   bob-cli in plugin tests, retaining their source and parity check. Never use the whole
   Review digest for a frontmatter summary. Matching stamp means agent; absent/mismatch
   means edited.
4. Gate on any nonempty intention, summary, or reflection, valid values, desktop, and
   Reading/Live Preview. No card on empty notes, Source, embeds, popovers, or mobile.
   Register the host's cleanup and refresh, including parent-cache changes and day
   rollover. Acquire owned-key handles only after a successful mount and release them on
   failure, invalidation, mode change, file change, and unload.
5. Own only `intentions`, `summary`, `summary_digest`, `summary_drafted`, `reflection`,
   `review_digest`, and `review_drafted`. Other Properties remain. Place the panel after
   the actual H1, navigation and child-link lines and before the first unrelated block:
   above daily `^gtd` and period strip fences. Test both source anchors and Reading
   placement; do not treat all initial paragraphs or arbitrary links as navigation.
6. Render the approved kicker; numbered accent chips ①–⑨; hero weight for a single
   intention; MarkdownRenderer links and hover previews; SUMMARY with `✦ agent · Nov 19`
   or `✎ edited`; REFLECTION with a quote rule. Unresolved links are struck in
   `--text-error` with `bob periods list -p` in their tooltip; cover missing files and
   missing heading/block targets with the existing link resolution helpers. The muted
   parent line shows the parent's intentions, when present, and opens it.
7. A closed subject with intentions, closed within the last seven days, shows the quiet
   missing-band line (`✦ summary pending · ＋ reflection r`, omitting parts already
   present). Never show it on open/future or older subjects. An entirely empty note
   still has no display card. The editor supplies its own Add ghost.
8. Use a labelled group, theme variables, glyphs/words for provenance, content-sized
   heights, wrapping at 375 px, and reduced motion. Add
   `scripts/preview-intention-card.cjs` using the actual painter, with light/dark
   desktop and narrow cases: recent-closed ghost, single intention, three intentions
   plus both bands, edited summary, summary-only and reflection-only. Inspect every
   rendered screenshot and repair clipping/contrast/spacing issues.

Tests must cover invalid/missing fields, canonical gates, boundary dates, digest parity,
unresolved links, anchors, parent updates, ownership release and recovery, and no
subprocess dependency for painting. Run the shared host/ownership/registry tests as
regression coverage. Build, generate inventory if needed, validate, and deploy this
phase's plugin changes using the common rules below.

## Phase editor: guarded writes and all field editing

Build a thin periods service and session adapter, a modal with browse selection, field
stages, and a per-path optimistic overlay. Register the intentions kind with the
existing registry so `==` opens on every canonical supported subject, including empty or
malformed ones. A malformed field can expose a refusal and YAML recovery; it must not
produce a hidden, misleading display card.

The modal initially focuses browse controls, never an input: first intention/Add for
open subjects, Reflection for closed ones. The complete field-key behavior is:

| Gesture                    | Result                                                                             |
| -------------------------- | ---------------------------------------------------------------------------------- |
| j/k, arrows, Tab/Shift+Tab | Move selection                                                                     |
| 1–9                        | Select that intention                                                              |
| Enter                      | Edit selected row                                                                  |
| i                          | Edit first intention, or Add if empty                                              |
| o/O                        | Insert below/above selected intention; sensible end/start with a non-intention row |
| r                          | Open reflection textarea with caret at end                                         |
| s                          | Select Summary; Enter explains ownership before opening its edit stage             |
| Alt+J/K                    | Reorder the selected intention with the shared arrange animation                   |
| Ctrl+D                     | Remove selected intention, undoable                                                |
| u                          | Apply guarded inverse-operation Undo                                               |
| Esc/q/Ctrl+[               | Existing shared one-layer back/close behavior                                      |

Read `decisions:q-and-ctrl-bracket-mirror-esc` through `/sase_memory_read` before wiring
close keys. q types in text controls, Ctrl+[ never does, and q is never a badge. The
next phase supplies `=`, `[`, `]`, and `y`; expose one path-pinned open method with
initial-stage and return-selection options for it.

Intention input is one line; Enter saves, Esc cancels. Pasted newlines collapse to
spaces with a quiet note. Reflection auto-grows; Enter saves, Shift+Enter adds a
newline, Ctrl/Cmd+Enter saves and closes. Show the existing summary read-only above its
textarea. At four or more intentions show “Three is plenty — what can go?” without
refusing. Summary editing explains “agent drafted this; your edit makes it yours” before
the intentional edit and uses the guarded summary ops.

All writes preview then apply through `createGuardedUpdateSession`; await
`albumSchemaSaveOpenEditors` for the pinned path first. Carry the server's field
fingerprint rather than whole-file hashes or a recomputed client fingerprint. On stale
refusal, fetch a fresh empty-op preview, retain the exact draft and selection, rebase,
and require a reviewed retry rather than silently applying to a new list position.
Preserve retained drafts on transport, timeout, lock, validation, and unavailable-binary
errors. Never map every exit 2 to an old bob: distinguish unsupported-command output
from real validation failures. An unsupported update command shows “Update bob to edit
intentions”; painting continues normally.

Ignore superseded async responses after close or target change. Keep Undo and overlays
path-scoped; clear an overlay when matching metadata arrives, on later conflicting
metadata, or on bounded expiry. No optimistic success before apply succeeds.
Intentions/summary changes from another pane must not be overwritten silently, while
unrelated body edits must remain safe.

Add unit tests for decoding, key dispatch, stage semantics, reordered indexes,
stale/refusal retention, before-save pane flushing, superseded replies, overlay
settlement, old-bob degradation, and stamped-summary Undo. Include a real-bob
scratch-fixture test for preview/apply/Undo and body-edit survival with `BOB_BIN`;
mock-only tests cannot establish writer compatibility. Build, validate, and sync.

## Phase navigation: hierarchy, YAML, entry points, and documentation

1. `[`/`]` change the subject inside the same modal; `=` climbs day → week → quarter →
   year. Reuse the subject helpers and preserve ISO-week quarters and W53. Do not switch
   the underlying active note. Missing dailies show “no note for Fri Nov 20 yet” and
   disable edits; never create them implicitly. Missing periods provide the existing
   `bob periods ensure` hint. No `all_time` editor. Browse navigation cannot discard an
   active stage: stage keys remain text, and only explicit cancel/save leaves it. Pin
   all requests, Undo, pending drafts, and async completions to the correct target.
2. Extend the shared YAML excursion with `kind: intentions`, correct target path,
   owned-field line focus, saved selection and prior view. `y` opens Source on the
   selected field. Done awaits saves, validates with a fresh periods empty-op preview,
   and restores the prior view plus editor selection only on success. Errors remain
   visible in Source with recovery available. Cover daily and period paths and
   regression-test album/bank/record excursions. Split the shared file if needed to
   honor the fragment cap.
3. Register **Edit intentions** (`edit-intentions`) and **Reflect on yesterday**
   (`reflect-on-yesterday`) in ledger-tools. The first uses the active canonical note.
   The second uses existing local daily path/date logic (`todayDailyPath` and daily
   settings), pins yesterday, and opens Reflection without navigating. Compute date
   subtraction with calendar arithmetic, not milliseconds or UTC string parsing. Missing
   notes show the same disabled state.
4. Add both commands to `CM_SHEET_SURFACE_ROWS` in
   `plugins/bob-command-mode/src/160-sheet-rank.js`, next to card rows, preserving
   command feature detection. Extend the Ctrl+Shift+P fallback in
   `plugins/bob-navigation-hotkeys/src/551-plugin-album-schema-fallback.js` only after
   existing task, Task Link, relation/dependency, and direct-stage routes have priority.
   Keep album/record behavior and the `==` gate unchanged.
5. `[n==r` composes existing previous-note navigation, the registry, and direct
   reflection opening. `==[r` changes to yesterday inside the modal then opens
   Reflection. Keep early focus in browse mode; do not invent a global keystroke queue.
   Test fast stage selection during data loading against the pinned target.
6. Add `docs/intention-card.md`, `docs/nav.yml` entry, and links from the ledger
   reference/index as appropriate. Document display gates, ownership/provenance, every
   key, missing/old-bob/stale recovery, and both yesterday routes. Use “card” for the
   panel and “card editor” for the modal. Generate inventories with `npm run docs:gen`,
   not manual edits to generated regions.

Add navigation boundary, missing-note, entrypoint priority, Command Mode feature
detection, YAML return/refusal, and focus/keymap tests. Build, validate, run mounted
docs checks with `BOB_CLI_DIR` pointing to the current bob-cli checkout, and sync.

## Phase acceptance: isolated application proof and rollout

Add `scripts/obsidian-intention-card-e2e.cjs` and its offline wrapper in the explicit
unit list. Reuse existing scratch-vault/profile/CDP helpers and real input patterns from
record/album suites; never open Bryan's live vault in the test application. Use an owned
Obsidian process under `xvfb-run -a`. Build bob from the current bob-cli source
containing the closed dependencies and pass `BOB_BIN` and `BOB_CLI_DIR`. Record source
revision, binary hash/version, plugin revisions and hashes, test clock, writer
stdin/results, screenshots, and exact failed checks. No missing binary or skipped UI
test counts as acceptance.

The real suite must prove:

- Reading and Live Preview placement, exactly seven owned keys hidden, unrelated rows
  visible, restore on unmount/mode change/failure, and recovery after valid metadata
  returns. Empty, malformed, Source and embed cases fail open.
- Empty-note `==`; open/closed initial selection; `[n==r` and `==[r` end with the caret
  in yesterday's textarea, with no literal r inserted or wrong-note writes.
- Add/replace/reorder/remove/Undo with real bob, one-line paste normalization, soft-cap
  hint, multiline reflection/save-and-close, and summary provenance plus stamp-restoring
  Undo. Verify note bytes and unaffected body/unowned frontmatter.
- Parent and sibling navigation across boundaries, missing-day editing disabled, both
  palette commands, Command Mode rows, and the Ctrl+Shift+P task-priority fallback. YAML
  Done returns to the exact subject/selection; invalid YAML stays recoverable. Close
  aliases work from browse and stages with q's text exception.
- Two real panes: hold a draft, edit owned fields in the other Source pane, observe
  stale refusal with the draft intact and no unintended write, then explicitly retry
  successfully. A body-only change survives a save without a field-stale error. Old-bob
  refusal keeps the rendered card and native recovery available.

Review every final preview/UI screenshot, including light/dark and 375 px. Fix new
failures in this feature before closeout. Run `npm test`, `npm run validate`, and
`BOB_CLI_DIR=<current-bob-cli-checkout> npm run docs:check`. Run existing
album/record/bank acceptance where shared entrypoints, YAML, focus, or chrome are
affected. Do not weaken assertions to manufacture parity. The kit phase recorded known
failures in bob-cli-nq, bob-cli-nr, bob-cli-ns, bob-cli-m6, bob-cli-m7 and intervening
bank queue drift; these are context, not a blanket allowlist. A failure is unrelated
only when it reproduces identically on the clean base. Record that comparison and any
already-tracking bead in a `PROPOSED FOLLOW-UP:` note on the assigned phase; it does not
keep completed work open. The earlier docs word-budget failure likewise needs current
evidence if it recurs.

After all fixes and builds, deploy using the checkout-explicit sync below and run the
documented deployed-vault startup smoke gate. Verify `.bob-deploy.json` and managed-file
hashes match the source and the new commands are present. Preserve reports/screenshots
as audited explicit artifacts and put their references plus the final test verdicts in
the bead notes. This is deployment of managed plugin files, not the separate vault
adoption phase.

## Shared execution and closeout rules

All phases edit source fragments, run `npm run build`, and include generated `main.js`
with their source changes. Keep inventory current with `npm run docs:gen` and validate
with `npm run build:check` and `npm run validate`. Run each phase's focused tests, then
the relevant full gates. Every plugin-editing phase must sync:

```bash
bob plugins sync --repo "$PWD" --no-pull --dry-run
bob plugins sync --repo "$PWD" --no-pull
```

Run from the linked plugin checkout. Follow its deployment guard if another checkout's
build is newer; fetch and integrate landed work using the documented recipe, never force
an overwrite. The final phase runs the deployed-vault smoke gate described in
`docs/develop/deployment.md`. Long builds, suites, or waits go through `/sase_monitor`;
wait for the staging command itself to exit and finalize the turn as instructed. Do not
leave subprocesses running at turn end.

Phase workers append discovered work with
`sase bead note <assigned-phase> 'PROPOSED FOLLOW-UP: ...'`; they create no task beads
and make no memory edits. Before any phase close, run
`sase bead epic-symbols <assigned-phase>` and resolve or re-key its entries to a
still-open owner. Close only the assigned phase with verified evidence. Finalize every
root turn with `/sase_final`, declaring all repositories changed in that turn.

The child land agent verifies all four phases landed and acceptance covers the complete
original bob-cli-nh.7 scope, checks its epic symbols and
`sase bead epic-symbols bob-cli-nh.7`, and supplies the delegated completion note for
bob-cli-nh.7. Use the host's child-epic completion path; if it has already closed the
delegated phase, append evidence rather than reopening. Leave bob-cli-nh, its ancestor
plan, and the adopt/vocabulary phases to their own agents.
