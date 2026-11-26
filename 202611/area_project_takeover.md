---
tier: epic
title: Complete area and project Properties takeover
goal:
  Area and project cards safely own native Properties, bank has one coordinated title
  and foot, and the verified rollout completes bob-cli-pl.9.
phases:
  - id: home-ownership
    title: Gate home-card Properties ownership and route native entry points
    size: medium
    depends_on: []
    description:
      "home-ownership: extend the shared capability and mount coordinator for
      area/project cards, preserve fail-open behavior, and route summary/Add actions to
      the existing Property Editor."
  - id: bank-arbitration
    title: Integrate the Area Card with the landed bank title and setup card
    size: medium
    depends_on:
      - home-ownership
    description:
      "bank-arbitration: prove the bank anchor in real Obsidian, implement scope-local
      title/foot/frontmatter arbitration with the approved fallback ladder, and preserve
      the bank editor."
  - id: takeover-acceptance
    title: Verify home and bank takeover through production Obsidian adapters
    size: medium
    depends_on:
      - bank-arbitration
    description:
      "takeover-acceptance: extend and run the real-app fail-open, focus, capability,
      and visual matrix with retained evidence and no passive writes or paint
      subprocesses."
  - id: cleanup-docs-memory
    title: Repeat retired-count cleanup and publish the verified card contract
    size: medium
    depends_on:
      - takeover-acceptance
    description:
      "cleanup-docs-memory: repeat the guarded retired-count sweep, finish source-owned
      documentation, apply the three inherited glossary approvals, deploy, and prepare
      completion evidence for bob-cli-pl.9."
proposed_by: bbugyi200.apollo.bob-cli-pl.9
proposed_with: codex/gpt-6-astra
parent_bead: bob-cli-pl.9
create_time: 2026-11-25 19:49:44
status: wip
---

- **PROMPT:**
  [prompts/202611/area_project_takeover.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/area_project_takeover.md)
- **PARENT:**
  [202611/area_project_cards.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/area_project_cards.md)

# Area and project Properties takeover

Complete phase `bob-cli-pl.9` of `plan:202611/area_project_cards.md`. Eligible desktop
Area and Project cards will own in-document Properties only when the validated settings
editor is available. Bank will have one note identity and one Properties foot, with
banking controls retained. Uncertain evidence restores native access immediately.

This is a child epic because shared capability/lifecycle changes, bank host integration,
real-app acceptance, and vault/documentation completion each warrant a bounded worker.
The four phases are deliberately serial: the later phases verify and document the
implementation that actually landed. The original phase has no prior implementation
notes; do not rebuild its completed dependencies.

## Inherited requirements and launch checks

Read `bob-cli-pl.9` and `bob-cli-pf` with `sase bead read -r` before work. The
foundation `bob-cli-pf` was verified closed on 2026-11-26; it must remain closed. Read
the parent plan through `sase artifact read`, then inspect the current source. Open
`bob-plugins` through `/sase_repo` and read its `AGENTS.md`. Use only the path that
command returns; source paths below are relative to that repository unless explicitly
prefixed `bob-cli`.

The parent decisions are final:

- `card_structure_counts = no`: no sub-area or sub-project counts on cards.
- `memory_area_card = yes`: author only `glossary:area-card` for that decision.
- `memory_project_card = yes`: author only `glossary:project-card` for that decision.
- `memory_property_card = yes`: update only `glossary:property-card` for that decision.

These approvals are inherited; do not submit them for another choice. The final phase
uses `/sase_memory_write` and republishes generated instructions after editing those
three strands. Other memory corrections discovered during this work become
`PROPOSED FOLLOW-UP:` notes on the assigned phase, not memory edits or new beads.

Governing decisions, read during planning:

- `decisions:q-and-ctrl-bracket-mirror-esc`: preserve the editor's shared close ladder.
- `decisions:projects-block-from-prj-line`, as amended by
  `decisions:subprojects-inherit-project-blocking`: the existing `^prj` task remains the
  project control; do not move, hide, duplicate, or write it during takeover.
- `decisions:note-ready-cap-counts-the-lane`: preserve residence-based Ready readouts
  and absent-not-zero behavior.
- `decisions:off-clock-follows-tag-then-note-default`, as amended by
  `decisions:off-clock-note-defaults-filter-by-task-type`: existing authored policy
  chips and report-backed settings retain their current semantics.

Use the landed key grammar: bare `+` opens Property Editor and bare `-` toggles
Properties in Live Preview Vim normal mode; `==` opens the species editor. The parent
plan's older `=+`/`=-` spelling is obsolete. Preserve current Reading-mode key behavior.
No new Rust/JSON contract, CLI option, release version, or changelog change is needed.

## Verified starting point

The inspected bob-cli HEAD was `19e7fe54`; linked bob-plugins was `691182f4`. Reconcile
later commits before editing rather than assuming these stay current.

- The home model/DOM/host/foot/live fragments are `249b` through `249f` under
  `plugins/bob-ledger-tools/src/`. `249d-plugin-home-card-host.js` explicitly leaves
  native Properties visible; `249b-plugin-home-card-model.js` excludes `bank.md`. Cards,
  reports, settings stages, and parent changes have already landed.
- `240q-plugin-property-card-frontmatter-ownership.js` contains the evidence gate,
  generation-bound mounted owner, lifecycle coordinator, and restoration primitive. It
  recognizes album, record, temporal, and bank mounts, but not area/project.
- `298v-plugin-property-capability-routing.js` confirms `props_v1` per resolved bob path
  and wraps `markdown:add-metadata-property`. Its probes run outside painting.
  `298z-plugin-property-editor-settings-entrypoints.js` and `openPropertyEditor` already
  support `setting` and `focusSettings` requests.
- Bank has advanced beyond the original design's assumptions: `248cb` owns a title slot,
  `248cc` acquires whole frontmatter, and `248ca` appends a shared foot. Preserve that
  work while adding explicit arbitration; do not simply remove the `bank.md` exclusion
  and let two title/frontmatter owners compete.
- `241d-plugin-notebook-catalog.js` routes summary focus on an owned scope to Property
  Editor, but area settings need the logical `summary` setting, including when it is
  absent. The Areas catalog's affordance is in `241ai`.
- Existing acceptance runners are `scripts/obsidian-home-cards-e2e.cjs` and
  `scripts/obsidian-frontmatter-takeover-e2e.cjs`, with offline wrappers. Extend their
  production-adapter tests rather than manufacturing ownership in tests.
- `scripts/retire-project-count-keys.cjs` already provides dry-run/apply cleanup through
  guarded `bob props update`. `bob-cli-pl.1` verified removal of 212 keys from 106
  notes, then recorded recurrence after sync from a stale Mac writer. Its final receipt
  is `file:explicit:7e981553181fd23c81d6b221`; read that artifact through
  `sase artifact read` if needed. Do not treat the live vault as count-free.

## Phase home-ownership

Limit implementation to shared ownership/capability plumbing, home adapters, native
entry routing, and focused tests. Bank remains excluded until the next phase.

1. Extend the existing session/bob-path capability controller with explicit
   `props_contract_v1` evidence. Use an off-paint lifecycle probe of
   `bob __catalog --protocol 1` (its `features` contains this capability), or an already
   obtained valid typed-note report as positive evidence. An ordinary note's
   `props show` deliberately omits the settings capability: that report must never
   establish global absence or downgrade a confirmed contract. Keep generic `props_v1`
   ownership working on existing species. Deduplicate probes, bound retries, discard
   late replies after path/generation/unload changes, and refresh home mounts when
   current evidence settles. Never fetch or spawn from paint, model building, collapse,
   resize, or the drawer.
2. Extend `240q` mount-kind/source-path handling for area/project. Add a
   species-specific contract requirement to the shared eligibility predicate, not a
   separate CSS-only ownership system. Require the connected card and foot, matching
   current path and mount, desktop Live Preview/Reading, closed valid YAML with matching
   source/cache values, current `props_v1` and contract capability, and working
   editor/native-command routing.
3. Wire `249d` through `propertyCardFrontmatterOwnMounted` after successful mounting,
   with current-source callbacks and generation-bound release handles. Include setup,
   deferred Reading attachment, source edits, source/cache invalidation, navigation,
   container replacement, failure, and teardown. Preserve the settled-cache behavior
   during autosave and the separate title ownership mechanism. Split fragments rather
   than exceeding the 1000-line cap.
4. Preserve applicable row-level ownership as the fallback when only whole ownership is
   unavailable; home fallback rows may claim only values actually represented by the
   connected card. On loss of the card or entry into a denied context, remove all its
   ownership. Unknown/loose keys and invalid policy values remain available natively. Do
   not add a global `.metadata-container` hide.
5. Route Areas "add summary" and remaining native-focus helpers on these homes through
   `openPropertyEditor({filePath, setting: "summary"})` or the matching setting.
   Mod+`;`, a custom shortcut, and command dispatch continue using the existing Add-mode
   wrapper based on actual originating-scope ownership. An unavailable/rejected opener
   releases ownership before native fallback. The File Properties sidebar and unrelated
   notebook routes remain functional.

Focused tests cover confirmed/pending/absent/failed contract probes, ordinary-note
reports, bob-path switches and late replies, source/cache skew, malformed and duplicate
YAML, unclosed frontmatter, missing foot, mount replacement, split panes, and release
timing. Extend home host, frontmatter ownership, capability routing, summary entrypoint,
and editor suites; register new suites in `package.json`. Existing
album/record/temporal/bank behavior must pass regression tests. Build the generated
entrypoint, run targeted tests and `npm test`, and sync per the shared completion rules.
Record evidence for the real-app phase.

## Phase bank-arbitration

Use the new contract gate and home ownership, plus existing bank host helpers. Start
with a disposable-vault real-Obsidian spike in Live Preview and Reading: record whether
adopting `# 🏦 Bank` affects the bank setup anchor, title slot, setup body, or editor
gestures. Extend the existing runner/spike seam; retain the result. Static DOM tests
alone do not settle this choice.

Implement the parent plan's ordered fallback ladder without a new reviewer decision:

1. Preferred: Area owns the title, single foot, and whole frontmatter; bank stays in the
   body with its species identity, setup content, and Edit control.
2. If title adoption breaks the bank anchor, mount Area without title ownership using
   the kit fallback and retain the H1. Keep one functional Properties foot.
3. Only if that still conflicts, leave the Area adapter excluded on `bank.md` and
   provide area ancestry/summary through the bank header. Its landed identity model
   already supplies these; reuse it and record the concrete failed probes.

Make arbitration local to the exact pane/scope/path and mount generation. Suppress the
bank foot only after the replacement Area foot is connected and functional; never use a
vault-wide boolean. While Area is active, bank must not reacquire the title slot or
frontmatter through its independent refreshes. Area loss restores the bank foot and
native Properties in the same frame; a surviving bank body must not conceal an Area
failure by reacquiring whole ownership. Repeated callbacks, two panes, reattachment,
unload order, and Source transitions must be idempotent. On the bank hub, missing
`props_contract_v1` restores native Properties even though the preexisting bank adapter
can operate with `props_v1` alone. Preserve the bank row fallback and standalone
setup/editor behavior. For the final fallback branch, the bank adapter may take over
only under the same complete contract/evidence gate.

The Area drawer's `bank` field has a nested bank-editor handoff and truthful where-mark.
`==` remains bank-first in the registry, bare `+` opens AREA SETTINGS, and Area
Settings/chips focus their respective setting. Preserve categories, accounts, rules,
setup collapse, YAML recovery, and byte-preserving bank writes. Painting does not write
any note or move the literal H1.

Add unit/regression coverage for arbitration and the selected fallback, including loss
and restoration of each owner, both mount orders, refresh loops, and bank settings
entrypoints. Run focused bank/home/ownership suites, build, `npm test`, and sync. Record
the chosen branch and actual mount/foot ownership for later docs.

## Phase takeover-acceptance

Extend the existing production-adapter runner and fixture library, with its offline
wrapper in `npm test`. Keep earlier species tests. Use a bob built from the current
primary checkout via `BOB_BIN` and the runner's `BOB_CLI_DIR`/provenance contract. Use
an isolated Obsidian profile/process and disposable Git vault. If execution uses athena,
first read tailnet memory and follow the documented isolated route; repository access
still goes through `/sase_repo`. Never open the live vault in the automated UI suite.
Use `/sase_monitor` for long builds and real-app runs.

Required matrix:

| Concern             | Evidence required                                                                                                                                                                                     |
| ------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Normal takeover     | Live Preview and Reading for top-level/sub-area, inbox, active/terminal project, and the selected bank integration                                                                                    |
| Native restoration  | Source, embed, hover, canvas, print, mobile emulation, bad/duplicate YAML, unclosed frontmatter fence, source/cache mismatch, path change, failed mount, unload                                       |
| Capability          | Pending, absent, failure, props_v1-only bob, ordinary-note initial probe, configured bob-path change, stale/late completion                                                                           |
| Ownership stability | Autosave/reparse, replaced metadata containers, Reading reattachment, split panes, bank refresh and foot handoff; no duplicate title/foot                                                             |
| Focus and routing   | Vim k/gg, Up at body start, Tab/Shift+Tab, body typing after navigation, Mod+`;`, custom binding, command dispatch, summary ghost, Settings and chips, opener failure, native File Properties sidebar |
| Project integrity   | Cursor/selection/IME on `^prj`, Task Card and close gestures, source edits still target the actual line                                                                                               |
| Exclusions          | Untyped area.md/project.md, templates, done/ and _conflicts/, ordinary notes unchanged                                                                                                               |
| Passive behavior    | Protected fixture hashes unchanged; no paint/drawer/collapse/resize-triggered subprocesses                                                                                                            |

Check restoration at the transaction/frame boundary rather than after a long settling
delay. Separate allowed lifecycle capability probes from forbidden paint-triggered
processes. Keep destructive gesture probes on disposable notes outside the passively
protected fixture set.

Capture light/dark by narrow/wide in both modes for area, inbox, project/terminal, and
bank, plus representative native recovery/Source states. Inspect every image and record
an individual visual verdict, size, and digest. Retain raw and inspected reports,
screenshots, console logs, source/binary/plugin hashes, exact commands, and bank spike
evidence through `/sase_artifact`. Offline self-tests or a skipped real-app run are not
acceptance. Repair phase-related defects and rerun affected checks before handing off;
preserve evidence for unrelated baseline failures.

## Phase cleanup-docs-memory

Read `obsidian.md` before vault work and use `/sase_repo` for the vault repository
before examining or mutating its contents. Reuse the landed cleanup tool rather than
writing another YAML mutator:

```sh
BOB_BIN=<current-built-bob> node scripts/retire-project-count-keys.cjs --dry-run --bob-dir <opened-vault>
BOB_BIN=<current-built-bob> node scripts/retire-project-count-keys.cjs --apply --bob-dir <opened-vault>
```

Run dry-run first. If keys returned, identify the responsible stale host from available
deployment/sync evidence (distinguish current attribution from the previously proved Mac
recurrence) and report it. Apply only the two retired keys through guarded props
transactions. Retain fresh preimages/receipts, prove that dry-run changes nothing and
each successful apply removes only those keys, preserve concurrent edits, and verify a
repeat no-op at a named snapshot. Handle stale fingerprints/lock-busy by re-reading and
retrying boundedly, never overwriting. Do not promise permanent absence while another
host can recreate them. Deployment of retired writers on stale hosts uses the existing
sync workflow; no force overwrite or service reconfiguration is assumed. Use monitored
execution for the potentially long sweep. Follow the vault's sync/commit ownership
conventions.

Read `docs.md` and edit source-owned documentation:

- bob-plugins `docs/area-and-project-cards.md`: capability gate, exclusions, native
  fallback, selected bank branch, single-foot behavior, actual shortcuts.
- `docs/property-editor.md`, `docs/develop/architecture.md`, and
  `docs/develop/validation.md`: species, dual capability requirements, arbitration,
  summary/Add routing, exact tested matrix and retained evidence.
- Relevant `docs/bob-ledger-tools.md` and `docs/bob-project-tasks.md` wording if it
  still describes the old bank foot or future native-focus takeover.
- bob-cli `docs/bank.md`: final Area/bank responsibility split and recovery. Remove
  superseded blanket "native Properties stays visible" and bank exclusion statements
  only where this implementation changes them. Mounted plugin docs are generated from
  their source repository, not hand-edited in bob-cli.

Apply the inherited memory approvals with `/sase_memory_write`:

- `glossary:area-card`: concise definition linked to Property Card/Area Note/Inbox File;
  layers/AREA identity, crumbs, H1→alias→verbatim stem, summary, authored policies,
  Ready readout, Settings, inherited ghosts, read-only paint and native recovery, and
  the actual bank relationship.
- `glossary:project-card`: target/PROJECT/ROOT identity, lifecycle from Project Task
  with report-backed reason/date, muted terminal state, policy and live readouts,
  unchanged `^prj` control, PROJECT SETTINGS and read-only lifecycle.
- `glossary:property-card`: include area/project species, registry order after existing
  species and before extensions, bare -/+, their stronger capability gate, and actual
  bank ownership. Link the two new strands.

Link the published `docs/reference/plugins/area-and-project-cards.md` reference from the
new terms. Describe the measured bank fallback honestly if one was needed. Run
`sase memory init`; generated instruction/README updates are covered by these approvals.
If another memory such as Bank Card becomes misleading, record only a
`PROPOSED FOLLOW-UP:` on this phase for the outer land agent to triage.

Run plugin docs validation and bob-cli `just docs-check`, inspect generated diffs, and
complete the plugin sync and required isolated post-sync startup smoke check following
`docs/develop/deployment.md`. Record source/deployment provenance, cleanup
results/limitations, artifact references, and completed gates on the phase.

## Shared verification and completion

Edit fragment sources, never generated `main.js`; run `npm run build` and
`npm run build:check`. Each plugin-changing phase runs meaningful focused tests,
`npm test`, and `bob plugins sync --repo <opened-plugin-root> --no-pull`. Follow
AGENTS/deployment instructions on protected deploys; do not use `-F`. Use
`/sase_monitor` for long commands, wait for the staging command to exit, and finish
through `/sase_final`. Every changed repository belongs in the finalizer.

Run required project checks (`just check`, docs gates, plugin validation) against the
completed change. If a failure reproduces identically on the clean base, record the
exact reproduction, source SHA, and any existing tracker in a `PROPOSED FOLLOW-UP:`
note; it does not prevent closing the assigned phase. Do not create task beads. Run
`sase bead epic-symbols <assigned-phase>` before each close and remove or re-key any
allowances to a still-open owning bead.

Each worker closes only its assigned child phase. The child epic's land agent reviews
all four results, verifies the whole original scope and final deployment, resolves its
own epic symbols, and completes the child epic through its host workflow. It then checks
`sase bead epic-symbols bob-cli-pl.9` and closes `bob-cli-pl.9` with a note citing the
verified implementation, bank branch, real-app evidence, cleanup result, and docs/memory
gates. This is the expressly requested completion of the original phase; never close
`bob-cli-pl` or another ancestor. The present planning turn leaves `bob-cli-pl.9` in
progress.
