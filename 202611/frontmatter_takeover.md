---
tier: epic
title: Complete card frontmatter takeover
goal: Eligible mounted cards replace native in-document Properties without losing
  keyboard editing, native recovery, sidebar access, or source fidelity.
phases:
- id: focus-proof
  title: Prove whole-container ownership in real Obsidian
  size: medium
  depends_on: []
  description: 'focus-proof: run and strengthen the existing S1 probe, fix bounded
    ownership/focus defects, and retain the observed Add file property command contract
    and mode verdicts. Keep production adapters unwired.'
- id: capability-routing
  title: Add cached capability gating and native property entry routing
  size: medium
  depends_on:
  - focus-proof
  description: 'capability-routing: add the session-and-bob-path capability cache,
    shared actual-ownership query, reversible Add file property command wrapper, and
    owned-scope focus-helper routing. Test lifecycle and stale-result behavior before
    enabling adapters.'
- id: adapter-takeover
  title: Enable fail-open ownership on album, record, and temporal cards
  size: medium
  depends_on:
  - capability-routing
  description: 'adapter-takeover: wire successful card mounts to generation-bound
    whole-container ownership, preserving row hiding as fallback and promptly releasing
    on invalidation, failure, Source, navigation, or teardown. Include identity-only
    day cards.'
- id: acceptance
  title: Verify takeover restoration, keyboard access, and visual behavior
  size: medium
  depends_on:
  - adapter-takeover
  description: 'acceptance: build and run the disposable-vault real-Obsidian takeover
    matrix and offline wrapper, fix integration regressions, and retain inspected
    screenshots and reports tied to the exact CLI binary and plugin builds.'
- id: docs-memory
  title: Document verified ownership and correct the two approved glossary strands
  size: small
  depends_on:
  - acceptance
  description: 'docs-memory: update plugin behavior and validation references and
    the inherited-approved Property Card and Record Card glossary strands, regenerate
    memory, and prepare completion evidence for the child epic''s land agent.'
proposed_by: bbugyi200.apollo.bob-cli-pf.6
proposed_with: codex/gpt-6-astra
parent_bead: bob-cli-pf.6
create_time: 2026-11-23 20:38:13
status: wip
bead_id: bob-cli-pf.6.1
---

- **PROMPT:** [prompts/202611/frontmatter_takeover.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/frontmatter_takeover.md)
- **PARENT:** [202611/card_owned_frontmatter.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/card_owned_frontmatter.md)
- **BEAD:** [bob-cli-pf.6.1](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-pf/bob-cli-pf.6.1.md)

# Finish card frontmatter takeover for bob-cli-pf.6

Eligible album, record, day, week, quarter, and year cards will replace the note's
native Properties block in Live Preview and Reading. The existing foot row and `=-`
drawer show every property; `=+` edits properties; `==` opens the species editor. Native
Properties returns whenever ownership cannot be justified. Bank keeps its native block.
Native Add file property and property-focus helpers remain useful.

This is a child epic for **bob-cli-pf.6**, implementing the approved `takeover` phase of
`plan:202611/card_owned_frontmatter.md`. All five phases are bounded direct coding work.
The dependency chain is intentional: do not deploy adapter takeover before the real
focus verdict and native entry routing exist. The child epic's land agent settles
completion of bob-cli-pf.6 after verification; child phase workers close only their own
assigned phases. Do not close bob-cli-pf or any other ancestor.

## Evidence and scope

Inspection on 2026-11-24 found bob-cli-pf.6 in progress, with no earlier worker notes or
child implementation to finish. Its prerequisites pf.1, pf.3, and pf.5 are closed.
`sase bead epic-symbols bob-cli-pf.6` reported no entries. Recheck before completion.

The opened bob-plugins checkout at `305b637c` contains the foot/drawer implementation,
the generic editor, and its native parity work. Reuse these; do not recreate them. The
whole-container primitive in
`plugins/bob-ledger-tools/src/240q-plugin-property-card-frontmatter-ownership.js` still
says it is unwired. It already supplies CodeMirror ownership, Reading classes, scoped
CSS, inert/aria-hidden, a focus guard, generation-bound release, and teardown. The
existing `propertyCapability` in `298d-plugin-property-update-service.js` is a single
value learned from show requests, not the required cache keyed by bob path. No
production adapter calls `propertyCardOwnFrontmatter` yet.

pf.3's completion note explicitly deferred the real S1 run because apollo lacked
Obsidian and Xvfb; no native command ID was recorded. Its runnable probe is
`scripts/obsidian-frontmatter-ownership-spike.cjs`. It currently permits the first
property-related command as a fallback and does not require observed Mod+; dispatch to
match the chosen command. Tighten that evidence before using it as a go/no-go gate.
pf.5's completion reports 263 passing real-Obsidian checks and supplies a proven
runner/provenance pattern, not proof of whole-container ownership.

Relevant code in bob-plugins:

- Shared host/adoption: `240c-plugin-property-card-host.js`,
  `240d-plugin-property-card-host-editor.js`,
  `240k-plugin-property-card-title-adopt.js`.
- Album: `241v-plugin-album-header.js` and `241va-plugin-album-header-foot.js`.
- Record: `242k-plugin-record-sheet.js`, `242ka-plugin-record-sheet-properties.js`,
  `242kb-plugin-record-sheet-title.js`, `242kc-plugin-record-sheet-foot.js`.
- Day/period: `296c-plugin-intention-card-host.js`,
  `296d-plugin-intention-card-title-slot.js`, `296da-plugin-intention-card-foot.js`.
- Property service/opener: `298d-plugin-property-update-service.js`,
  `298m-plugin-property-editor-entrypoints.js`, and their existing tests/exports.
- Native focus: `notebookFocusSummaryProperty` in `241d-plugin-notebook-catalog.js`.
- Existing real runners: ownership spike, card-foot e2e, and property-editor e2e.

These are fragment filenames under `plugins/bob-ledger-tools/src/` unless a full
repo-relative path is given. Read source before modifying it and accommodate new landed
changes rather than assuming these exact revisions remain current.

## Inherited final decisions

The parent's reviewed decisions apply unchanged:

- `bank_foot = yes`: retain bank's foot, drawer, and created stamp; exclude bank from
  whole-container ownership.
- `drawer_memory = note`: retain the existing per-path session state.
- `memory_glossary_cards = yes`: authorization covers only `glossary:property-card` and
  `glossary:record-card`, plus their generated outputs.
- `memory_decision_record = no`: no decision-record work is authorized.

Do not re-ask these decisions. There are no new reviewer choices in this child plan. No
CLI grammar, property writer, card editor parity, created-date backfill, Reading chord,
or new card species is part of this work.

## Required behavior

Ownership requires a successfully mounted supported card, a parsed cache mapping
confirmed against current source with a closed frontmatter block, desktop Live Preview
or Reading, an allowed scope, confirmed `props_v1` for the current bob path, and working
property-editor/native-command routing. Unknown or stale evidence must leave native
Properties available. Embeds, hover popovers, canvas, print, mobile, Source, bank, and
non-card notes never acquire whole-container ownership.

Keep the native metadata node in the DOM as the title-slot anchor. Keep CSS local to
owned scopes; never hide the sidebar or add a global `.metadata-container` rule. Title
ownership, individual-row ownership, and whole-container ownership remain independent.
Losing whole ownership restores the preexisting row-hiding behavior; losing the card or
entering a denied context restores native rendering entirely. Identity-only temporal
cards qualify without requiring authored intentions.

Painting reads local cache/source state and never starts bob or writes vault bytes.
Capability discovery is separate lifecycle/event work. All edits still use existing
`bob props` or species editor paths. Capability absence is not invalid YAML, and an I/O
failure or missing note is not proof of an obsolete binary.

## Phase focus-proof

1. Open bob-plugins using `/sase_repo` and read its AGENTS.md. Read the parent artifact
   and pf.3's notes through audited commands. Inspect the existing ownership tests,
   spike library, and spike runner; run the focused unit/offline checks first.
2. Execute the real S1 runner against the current built plugin in a disposable vault and
   owned profile/process. Follow `docs/develop/validation.md`'s existing athena
   execution route if the current host has no Obsidian/Xvfb. Read the tailnet memory
   before remote access and use `/sase_repo` for repository access. Do not inspect or
   modify sibling checkouts, deploy into the live vault for a test, or treat an offline
   self-test as real acceptance.
3. Strengthen command discovery to require the actual Add file property command,
   recording its ID, callback/check-callback shape, Obsidian version, effective hotkeys,
   and observed invocation. Verify the command through its registered dispatch and Mod+;
   (bind the normal shortcut explicitly in the scratch profile if needed). Do not
   silently accept another metadata command or an empty event log.
4. Verify in both modes: ownership variables, hidden-but-present anchor, title-slot
   geometry, no hidden-input focus from Vim k/gg, Up at line zero, Tab/Shift+Tab, and
   body typing after navigation. Verify Source and file-change release. Fix bounded
   primitive defects and add regressions; preserve generation protection and avoid
   dispatching inside CodeMirror updates. No production adapter is enabled here.
5. Retain the report and useful screenshots through `/sase_artifact`; write their refs,
   exact observed command ID, and per-mode go/no-go outcomes in the phase note. If a
   mode cannot be made focus-safe, record the precise mitigation: that mode must retain
   per-row hiding in the remaining phases. Missing runtime evidence is not a passing
   mode verdict; arrange the runnable host route before closing.

Acceptance: a reproducible real S1 result supports the eligible modes and command
contract, or explicitly disables only an identified unsafe mode. Tests and build checks
pass, adapters remain unwired, and later phases can consume the exact result.

## Phase capability-routing

1. Implement a small capability controller alongside the property service, using
   `apiStatsBobPath()` / the existing `bobSpawnJson` resolver. Cache positive, negative,
   and unresolved outcomes per plugin session and resolved bob path; deduplicate
   in-flight probes. Use an existing Markdown note and `props show` as the read-only
   probe. Schedule from setup/workspace/configuration events, never from paint/build
   functions. Wait for a usable note without manufacturing one.
2. Bind each result to path and lifecycle generation. On a configured executable change,
   use that path's cache or unknown state immediately; discard obsolete pending
   responses after a path change/unload. Unknown-subcommand means absent; malformed
   reports and transient note/I/O failures remain unconfirmed and native. Define a
   bounded retry on a subsequent relevant lifecycle event, with no repaint retry loop.
   Successful confirmation schedules card refreshes. Keep the editor's capability
   display consistent with this controller and do not let unrelated per-note errors
   revoke a confirmed capability.
3. Add one actual-ownership query keyed by originating view/scope and current note path.
   It must observe active ownership, not merely card eligibility, a global Boolean, or a
   stale class. Make it available to the native wrapper and helpers.
4. Wrap the S1-observed Add file property command at its registered dispatch point so
   default/custom hotkeys and the palette use the same route. On a currently owned
   document scope call `openPropertyEditor({filePath, mode: "add"})` using the opener's
   verified option shape. Preserve callback context, arguments, check semantics, and
   return values on untouched scopes. Restore only this wrapper on unload, preserving
   wrappers subsequently installed by another plugin. Failure to establish routing must
   prevent ownership, rather than strand a hidden input.
5. Audit native-property focus across all plugin source and hand-edited plugins. Route
   `notebookFocusSummaryProperty` through the common opener with `key: "summary"` only
   for an actually owned scope; keep ordinary notebook behavior. Native sidebar focus
   must retain native behavior even if the main note is owned. If an opener fails after
   a scope was acquired, release that scope before any native fallback. Avoid recursion
   and duplicate modal creation.
6. Test deduplication, path-switch races, teardown races, transient failures, unknown
   capability, ordinary-note and sidebar passthrough, command check invocations, custom
   bindings, callback restoration, and focus-helper routing. Keep adapters unwired until
   the next phase.

Acceptance: probing stays off the paint path, capability results cannot cross binary or
lifecycle boundaries, and native commands work with both owned and native scopes.

## Phase adapter-takeover

1. Add a shared mount ownership coordinator/eligibility helper around the existing
   `propertyCardOwnFrontmatter` handles. Reuse source-validation and confirmed-source
   mechanisms already used by card models/feet instead of creating another YAML writer
   or loose parser. Treat duplicate/invalid/unclosed or cache/source-skewed frontmatter
   conservatively. Do not confuse a stale optimistic overlay with confirmed source, or
   hiding eligibility for individual rows with whole ownership.
2. Wire successful album and record mounts/adoption and temporal title-slot mounts in
   both modes. Review the host's actual production mount/remove hooks: its generic
   `commitMount` is not sufficient by itself. Register the field before acquiring; keep
   a generation/path-bound handle per mounted scope. Handle reused DOM, replaced
   headers, two panes of one note, rapid path changes, and old async paint callbacks.
   Never release a newer mount's ownership from an old callback.
3. Own only after the supported card and foot mount succeed and native routing is ready.
   Include identity-only day/week/quarter/year cards. Bank retains only its current key
   ownership; no new species adopts the whole-container primitive.
4. Release when current source/cache invalidates, the mount disappears or throws,
   capability becomes unconfirmed for a new executable, the scope becomes denied, the
   pane navigates, Source activates, or the plugin disables/unloads. Source and file
   changes must remove ownership in the same CodeMirror transaction; other invalidations
   must release before stale UI is painted, rather than wait for the ordinary 100ms
   refresh. Retain observers that reapply ownership to replacement containers while the
   same valid mount exists. Teardown must not leave CSS, inert/aria-hidden, focus
   listeners, or late callbacks behind.
5. Add adapter integration tests for the full eligibility truth table, mount success and
   failure, identity-only temporal cards, stale source/cache, multiple panes,
   replacement nodes, stale release, and unload. Assert native title placement and
   existing `==`, `=-`, foot, created stamp, and per-row fallback still work. Use
   spawn/write spies to prove painting is read-only and launches no subprocess.

Acceptance: verified supported modes own only eligible successful mounts, with prompt
restoration in every failure state and zero change to bank/sidebar behavior.

## Phase acceptance

Create `scripts/obsidian-frontmatter-takeover-e2e.cjs`, focused reusable helpers, and an
offline wrapper registered in `package.json`'s unit suite. Reuse the existing ownership,
property-editor, and card-foot runner infrastructure. Avoid a monolithic new runner or a
test that manually claims ownership instead of testing production adapters. Preserve
existing editor e2e coverage; adjust assertions made obsolete by intentional
whole-container hiding without weakening native fallback assertions.

Use a bob built from the current primary checkout, provided through `BOB_BIN` and
`BOB_CLI_DIR`; follow the documented transported-build provenance contract when needed.
The real run records source SHAs, binary/plugin hashes, capabilities, Obsidian version,
skips, console exceptions, and command invocations. Run long verification through
`/sase_monitor` rather than ending with a live process.

Required assertions:

- Automatic ownership in Live Preview and Reading for day, week, record, and album; add
  quarter/year and a day without intentions as focused coverage.
- Source toggle, embedded notes, hover popovers, canvas/print denial, mobile emulation,
  malformed YAML, unclosed fence, stale cache/source, same-pane file changes, injected
  mount failures, removed/replaced containers, and disable/unload. Assert both visual
  restoration and removal of inert/aria-hidden/focus guards; use
  transaction/animation-frame probes so settling delays cannot mask stale hides.
- Pending capability, absent/old bob, failed probe, switching configured bob paths, and
  a late result after unload; preserve native editing throughout uncertainty.
- Mod+;, the palette command, and a custom shortcut open the editor in Add mode on owned
  notes. Non-card/bank notes and the File Properties sidebar retain native behavior.
  Cover field-focused helpers and a failed opener's fallback.
- Repeat the S1 focus and body-typing checks after real adapter mounts. Confirm the
  title slot stays correctly positioned and drawers still expose every key.
- Hash scratch notes before/after passive rendering and record invocations to prove no
  unintended writes or paint-triggered subprocesses. Intentional editor actions may
  write only through the existing guarded writer.
- Capture light/dark × narrow/wide screenshots for day, week, record, album, bank, and
  an ordinary note's `=+` editor. Include both rendering modes across the card shots and
  the important fallback/owned states. Inspect every PNG with image tools, record
  per-image hashes and visual verdicts, and retain the report/images through
  `/sase_artifact`. Geometry probes alone do not establish visual acceptance.

Fix regressions caused by this work, with focused tests. Preserve per-row hiding in any
explicitly S1-denied mode and make that limitation prominent in the report. An offline
pass or skipped real suite does not complete this phase.

Acceptance: the matrix passes on the exact delivered builds, every required view is
accounted for, screenshots are inspected, and the report is ready for land review.

## Phase docs-memory

Read `docs.md` memory before documentation changes and `/sase_memory_write` before the
already-authorized glossary edits. Use the verified acceptance results:

- Update bob-plugins `docs/record-card.md`, `docs/intention-card.md`, `docs/albums.md`,
  `docs/property-editor.md`, `docs/develop/architecture.md`, and
  `docs/develop/validation.md`; update any now-inaccurate hotkey reference. Describe the
  third ownership bit, capability cache, restoration rules, native command routing,
  sidebar escape hatch, and tested mode limitations. Replace outdated unconditional
  claims that native Properties/Add property remain visible. Mention that Metadata
  Menu's native Properties context menu is unavailable on owned notes.
- In bob-cli `sase/memory/glossary/property-card.md`, replace the row-only ownership
  description with concise gated whole-frontmatter ownership, its bank exception,
  foot/created/drawer, and the `==`, `=-`, `=+` family. Preserve read-only painting,
  title semantics, and restoration; link the property-editor reference.
- In `sase/memory/glossary/record-card.md`, replace "above native Properties" and the
  claim that identity/legacy/unrelated keys stay in Properties. State eligible
  replacement and complete inventory in `=-`/`=+`, preserving species-editor and Source
  behavior. These two edits inherit `memory_glossary_cards = yes`; no new permission is
  needed. Run `sase memory init` rather than editing generated files.
- Run plugin docs/build checks and the primary repo's `just docs-check`; inspect
  regeneration diffs. Add concise evidence notes for the land agent.

## Shared verification, deployment, and completion

All phases honor bob-plugins AGENTS.md: edit fragments, keep each hand-edited fragment
at or below 1000 lines, update `fragments.json`/mixins/exports when needed, and generate
`main.js` using `npm run build`. Run meaningful focused tests while working, then
`npm test`, `npm run build:check`, and `npm run docs:check` as applicable. After plugin
changes run `bob plugins sync` and follow its documented protected-build recovery
without forcing another agent's deployment. Use `/sase_monitor` for long commands. Host
finalization commits changes through `/sase_final`; do not manually commit or bump
release versions.

Child phase workers append discovered out-of-scope work as `PROPOSED FOLLOW-UP:` notes
on their assigned phase, never create task beads or close ancestors. For a check
failure, establish whether it reproduces identically on the clean base. If so, record
the precise command/evidence and existing task ID if known, then close the completed
phase rather than strand it. This exception does not turn missing takeover acceptance
evidence into a passing result.

The child epic's land agent reviews all phase artifacts, integrated tests, deployment,
and the two memory changes against the final inherited decisions. Run
`sase bead epic-symbols` for its assigned bead and bob-cli-pf.6, resolving or re-keying
every exception to a still-open appropriate bead before closure. Confirm no child work
remains and record tested modes plus any retained fallback. Use the supported delegated
completion path to close only this child epic and bob-cli-pf.6 with evidence; leave
bob-cli-pf and all other ancestors to their own land agents.
