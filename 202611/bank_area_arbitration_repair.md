---
tier: epic
title: Finish Bank and Area card arbitration from retained failures
goal:
  Repair the remaining bank.md ownership and editor failures, verify the production
  adapters in isolated Obsidian, and complete bob-cli-pl.9.1.2.
parent_bead: bob-cli-pl.9.1.2
phases:
  - id: bank-probe-contracts
    title: Make Bank acceptance probes match the production contract
    size: medium
    depends_on: []
    description:
      "bank-probe-contracts: isolate a complete bank acceptance slice, repair
      report/modal assumptions, and retain independent failure evidence without masking
      ownership defects."
  - id: bank-mount-lifecycle
    title: Repair pane-local title and Properties foot arbitration
    size: medium
    depends_on:
      - bank-probe-contracts
    description:
      "bank-mount-lifecycle: fix the duplicate-foot and second-pane Area-loss failures
      through production lifecycle paths while preserving fail-open native access."
  - id: bank-editor-routing
    title: Complete Bank and Area editor gestures
    size: medium
    depends_on:
      - bank-mount-lifecycle
    description:
      "bank-editor-routing: verify and repair nested Bank Setup, bare + Area Settings,
      Settings/chip focus, bank-first ==, and truthful bank where-marks."
  - id: bank-proof-rollout
    title: Retain real Bank acceptance and deploy the verified bundle
    size: medium
    depends_on:
      - bank-editor-routing
    description:
      "bank-proof-rollout: finish the scoped real-app matrix, classify regression
      failures against clean base, retain inspected evidence, sync without force, and
      run post-sync smoke."
proposed_by: bbugyi200.apollo.bob-cli-pl.9.1.2
proposed_with: codex/gpt-6-astra
create_time: 2026-11-26 03:28:33
status: wip
---

- **PROMPT:**
  [prompts/202611/bank_area_arbitration_repair.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/bank_area_arbitration_repair.md)

# Finish the remaining Bank arbitration work

This child epic implements only the unfinished bank-arbitration phase `bob-cli-pl.9.1.2`
of `plan:202611/area_project_takeover.md`. Multiple earlier implementation/monitor turns
landed repairs but left the real Bank checks failing. The assigned-phase prompt requires
a child epic when that remainder no longer fits one coder. Do not repeat the original
implementation or create another single-agent acceptance loop.

Read the assigned phase, `bob-cli-pl.9.1`, and `bob-cli-pl.9` through
`sase bead read -r`; read both `plan:202611/area_project_takeover.md` and
`plan:202611/area_project_cards.md` through `sase artifact read`. Confirm foundation
`bob-cli-pf` remains closed. Open bob-plugins through `/sase_repo`, read its
`AGENTS.md`, and use only its returned checkout. Paths below are relative to that
checkout unless prefixed bob-cli. Reconcile newer source before editing.

The outer phases remain authoritative: `bob-cli-pl.9.1.3` owns the full home-card
acceptance matrix, and `bob-cli-pl.9.1.4` owns cleanup, docs, and the three approved
memory changes. This child epic neither duplicates those jobs nor changes their scope.
No memory edits, CLI/JSON changes, version bumps, or changelog edits belong here.
Preserve the final decisions: no structure counts; the three glossary approvals remain
for the existing cleanup phase and must not be asked again.

## Governing decisions

- `decisions:q-and-ctrl-bracket-mirror-esc`: Bank Setup and Property Editor retain the
  existing shared one-layer close behavior for Esc, q, and Ctrl+[.
- `decisions:projects-block-from-prj-line`, as amended by
  `decisions:subprojects-inherit-project-blocking`: preserve the literal ^prj control
  and task behavior when shared home plumbing changes.
- `decisions:note-ready-cap-counts-the-lane`: preserve residence-based Ready readouts
  and unavailable-versus-zero semantics.
- `decisions:off-clock-follows-tag-then-note-default`, as amended by
  `decisions:off-clock-note-defaults-filter-by-task-type`: policy chips/settings remain
  read-only projections until an explicit validated editor write.

Planning attempted the mandated `sase memory read roster:decisions`; this installed SASE
rejects that selector as an unknown web. The available index command
`sase memory web show decisions` supplied the 78-strand roster, followed by audited
reads of the governing strands. This compatibility issue does not authorize memory
changes.

## Starting evidence and useful distinctions

Recovery evidence: file:explicit:e85d653a52d90836c21d7b53. Read it with
`sase artifact read`, then use `sase artifact path` for extraction into local scratch.
It contains the last raw Athena report, full matrix, inspected failure PNG, transport
logs/checksums, CLI and plugin provenance, previous full-suite logs, exact launch
scripts, and the fresh focused-test log. Historical paths in raw evidence are
provenance, not instructions to reuse another checkout.

Audit HEADs: bob-cli `1c23bce7`; bob-plugins `e82bd932`. The latest real run used plugin
`a0ebe63eca90f814a7503a87ee570836f9876716` and CLI
`926c92589bf68dc4df851e28a26999f67f2074cd`, Obsidian 1.15.1. At audit time there was no
subsequent diff in the bank/home/editor fragments or takeover runner. Existing repairs
include `8e401c6e`, `b185579f`, `5addf8ca`, and `a0ebe63e`; preserve their intent and
regression coverage. Fresh host/frontmatter suites pass 48/48.

The latest real runner exited 1: **38/64 pass**, NO-GO. Transport/wrapper exit 0 is not
acceptance. The report ends in a day screenshot timeout; final byte-preservation,
paint-process, screenshots and console checks were not executed. Its one 720x1000 PNG
shows the day card drawer and native Properties simultaneously. It does not prove Bank
appearance. Apollo lacks Obsidian/Xvfb; Athena is the documented isolated execution
route, so requiring a second local GUI installation adds no useful gate.

Specific remaining observations:

1. Initial Bank probes in both modes have Area title ownership and whole-frontmatter
   ownership, with Bank Setup body/Edit intact. **Both feet remain connected**. Title
   adoption is promising but not yet accepted across lifecycle transitions.
2. Both arbitration probes pass the nested-opener availability wait and then time out
   waiting for the Bank Setup modal. Investigate event delivery, returned opener
   results, and actual modal owner document. Do not equate a document selector miss with
   an absent control or blindly change a working opener.
3. A second Live Preview Bank pane loses the Area card and shows the bank title card
   plus native Properties. Check title-owner replacement, mount order and cleanup, not
   only the initial single-pane happy path.
4. Bare + yields `note_kind: bank`, `selectedId: setting::summary`, an AREA SETTINGS
   heading and `focusSettings: false`. The runner incorrectly requires
   `note_kind: area`; bob-cli `src/native/props/roles.rs` intentionally keeps the Bank
   wire kind. Preserve that contract. Determine the requested Settings focus separately.
5. Contractless Bank restores native Properties but Reading still shows both feet. The
   old Bank adapter must not use props_v1 alone to reacquire whole ownership.
6. The broad runner has additional non-Bank failures. Classify them through evidence;
   neither treating all as baseline nor making this phase fix all of them is valid.

## Phase bank-probe-contracts

Keep implementation confined to acceptance harness/fixture tests unless a minimal
instrumentation export is necessary. Extend the existing takeover runner/library, not an
unrelated test application.

- Add an internal Bank-only execution seam/profile using the same production adapters
  and fixtures. It must run to its final invariants and captures even when an unrelated
  day/album case fails. Preserve the default full-species matrix and offline wrapper
  registration in `package.json`.
- Match bank.md's stable `note_kind: bank`; use the report's area contract and actual
  settings group/selected row to assert AREA SETTINGS. A corrected assertion must still
  fail on missing settings, wrong setting, refused opener, or wrong mode.
- Locate modals through the owning Obsidian document/root and exposed modal lifecycle,
  with diagnostics of document identity, connection, class, and opener outcome. Keep
  real click/keyboard gestures; do not call ownership primitives or inject mounts to
  manufacture success.
- Make each mutable probe restore its leaves, editor/modal state, source, cache,
  capability shim and activity flags in finally. Retain the first failure and actual
  pre/post state; a later timeout must not erase earlier arbitration evidence.
- Include bank regression identity/contract coverage in offline tests. Keep expected
  control presence separate from whether a closed drawer has rendered its children.

Run build/check and focused runner tests. Use `/sase_monitor` for an isolated Athena
Bank slice on current built CLI/plugin bytes, recording initial failures. This phase may
close with truthful failing product probes because its deliverable is the correct
reproducible harness and diagnostics; it must not claim Bank acceptance. Publish raw
results for the following two repair phases.

## Phase bank-mount-lifecycle

Own the Bank/Area mount, title and foot lifecycle only. Start in
`249db-plugin-bank-area-arbitration.js`, `249d`/`249da`/`249e`/`249f`, `248ca`/
`248cb`/`248cc`, and the shared `240p`/`240r` lifecycle. Determine concrete causes from
phase-one probes before patching.

- Repair the duplicate-foot handoff with an exact scope/path/mount/generation owner.
  `propertyCardFootAdapterTrack` currently releases all scope/path registrations; audit
  every call and cleanup, not only the guarded Bank no-foot branch. Each adapter must
  release only what its lifecycle owns. A newly connected and registered functional Area
  foot is required before Bank yields.
- Fix both mount orders, second-pane initialization, metadata/container replacement,
  repeated refreshes, navigation, Reading reattachment, Source transitions and either
  unload order. Unmounting one pane must not delete another pane's registrations.
- Area loss restores native Properties and the Bank foot/body at the transaction or next
  frame boundary. A surviving Bank body must not conceal that loss by claiming whole
  frontmatter. Stale callbacks cannot release a replacement owner's state.
- Missing props_contract_v1 leaves native access visible even though Bank historically
  supports props_v1. Preserve row fallback and standalone Bank setup/editor.
- Exercise connected real foot registration, not only hand-built DOM flags, in unit
  regressions. Include the refresh order that defeated `a0ebe63e`.

Keep the approved fallback order: first Area owns title/single foot/frontmatter and Bank
stays in the body; if measured title adoption breaks the Bank anchor, use the kit
no-title Area fallback and preserve literal H1; only if that also conflicts keep Area
excluded and use Bank's existing ancestry/summary. Record failed real probes before
choosing a fallback. Do not abandon the preferred branch merely to silence a test. No
paint operation writes/moves H1, edits YAML or starts a subprocess.

Run focused Bank/home/foot/ownership suites, build/check and the Bank lifecycle slice in
real Obsidian. Record the selected branch and actual title/foot owner. Finish this phase
once lifecycle cases pass; editor-specific failures remain explicitly assigned to the
next phase.

## Phase bank-editor-routing

Own editor gestures and their regression coverage; use the prior lifecycle result. Start
with `249e`, `298m`, `298z`, `248l`, the registry and existing foot handlers.

- Repair any confirmed nested `bank` handoff issue using the existing Bank Setup modal
  API. Keep a truthful where-mark only while setup content is painted; folded state,
  refreshes, both modes and split panes must agree.
- Bare + on the Bank Area card opens AREA SETTINGS with its first setting selected.
  Scope any focus routing to the actual eligible home context, preserve ordinary notes,
  Add mode and File Properties sidebar routes, and leave the Rust note kind unchanged.
  The shared current + registration only passes filePath; test the behavior, not an
  incidental implementation flag.
- Area Settings and policy chips focus the requested setting. == stays bank-first; Bank
  Edit still opens the categories/accounts/rules editor. Preserve setup collapse, YAML
  recovery, existing byte-preserving writes and close-key behavior.
- Probe actual modal connection and focus and handle rejected openers fail-open. Restore
  all state after each fixture gesture. Do not weaken assertions to accept a missing or
  nonfunctional control.

Run focused editor/bank/home suites and build/check, then real Bank gestures in both
modes. Use disposable write fixtures for editor/YAML checks, separate from the passively
protected notes. No changes to the broader editor design or writer contract.

## Phase bank-proof-rollout

This is a bounded integration/evidence phase after the two source repairs. Use the fixed
Bank profile with a CLI built from the current primary checkout; follow
BOB_BIN/BOB_CLI_DIR and transported provenance requirements. Run on Athena through the
documented isolated profile/process and disposable Git vault; read tailnet memory first.
Use `/sase_monitor` for builds, GUI acceptance and full suites.

Required Bank proof in both Live Preview and Reading:

- chosen title/H1 branch, one functional foot, Bank setup/Edit retained;
- both mount orders, two panes, refresh loops, reattachment, navigation and Source;
- Area/Bank loss and restoration, teardown orders, missing/replaced metadata/foot,
  malformed or unclosed YAML and source/cache skew;
- props_v1-only/pending/failed/stale contract evidence restores native access;
- nested Bank setup, bare +, bank-first ==, Settings/chips, native Add/sidebar and focus
  protection; categories/accounts/rules/collapse/YAML remain functional;
- protected note hashes unchanged, zero paint/drawer/collapse/resize-triggered
  processes, and no runtime exceptions. Separate lifecycle capability probes from
  forbidden painting subprocesses.

Capture light/dark x narrow/wide x both modes for Bank, plus restoration/Source. Inspect
every PNG individually and retain verdict/dimensions/digest. Keep complete reports,
exact commands, source/binary/managed-plugin hashes, separate runner and transport
exits/logs, baseline comparisons and post-sync smoke in explicit artifacts attached to
the phase and cited on bob-cli-pl.9.1.2. A skip or wrapper success is not acceptance.
The broader area/inbox/project matrix belongs to bob-cli-pl.9.1.3.

Run npm test, build:check, plugin validation and required bob-cli just check via the
named tool workflow. If an unrelated check fails, reproduce the exact identity and
failure on a clean base using the opened checkout's archive, record SHA/command and any
existing tracker as a PROPOSED FOLLOW-UP on the assigned phase, and continue. Same
clean-base failures do not keep the bead open. Existing phase note #1 covers the two
date assertions; no duplicate beads. Do not fix unrelated fixture/timing failures or
repeatedly rerun unchanged suites seeking a lucky green. A real Bank failure or a
phase-caused regression remains blocking and is repaired before closure.

Follow `docs/develop/deployment.md`: preview, then
`bob plugins sync --repo "$PWD" --no-pull` from the opened plugin root, never -F.
Resolve protected landed builds per the documented fetch/ancestor procedure. Verify
post-sync smoke against a faithful copy of deployed managed bytes on Athena if this host
lacks Obsidian, with before/after deployed match checks and hashes proving the copy.
Never open the live vault in automated Obsidian. Read obsidian memory and open the vault
through `/sase_repo` before inspecting/copying its files. Report drift or missing
prerequisites honestly; do not call unrun smoke a success.

## Shared boundaries and completion

Edit source fragments and rebuild generated main.js; never edit generated output by hand
or exceed 1000 lines per fragment. Every plugin-changing phase follows the parent
build/test/sync requirements, using monitors for long commands and waiting for the
staging command itself to exit. Record source and test evidence once per changed
revision. Each worker closes only its own phase after checking its epic symbols; record
discovered follow-ups on that phase, never create task beads.

The child epic's land agent reviews all four completed phases and retained Bank
acceptance/deployment evidence, verifies that remaining non-Bank failures are either
clean-base exceptions or owned by the existing outer acceptance phase, and checks
`sase bead epic-symbols bob-cli-pl.9.1.2`. Resolve/remove every allowance for that phase
or re-key it to a still-open genuine owner. Then close only the original phase with
`sase bead close bob-cli-pl.9.1.2 --note "<verified branch, gates, artifact refs>"` when
descendant-close ordering permits, alongside normal completion of this child epic. This
is completion work delegated by the assigned-phase prompt. Child phase workers must
never close that ancestor themselves. Do not close bob-cli-pl.9.1, bob-cli-pl.9 or
bob-cli-pl. Their existing land agents retain those responsibilities.

Leave the original assigned phase in progress during this planning turn. Finish every
root turn through `/sase_final`; declare all repositories changed by that turn.
