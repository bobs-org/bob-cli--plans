---
tier: epic
title: Finish the title-block proof, visual matrix, and guarded album rollout
goal:
  The shipped title-block family is proven in real Obsidian and the full visual matrix,
  live albums are tidied only after every sync host proves compatible, and the verified
  merged plugin build is deployed with durable evidence for o6.6.
phases:
  - id: title-walkthrough
    title: Real-Obsidian title ownership and editing walkthrough
    size: medium
    depends_on: []
    description:
      "title-walkthrough: add the shared isolated obsidian-title-block-e2e runner and
      offline wrapper, proving the shipped album, record, day, and period title blocks
      in Live Preview and Reading, scoped ownership, real
      Vim/search/Outline/heading-link reveal, edit/undo, and fixture byte neutrality.
      Run on athena and repair defects in the owning fragments."
  - id: fail-open-proof
    title: Real-Obsidian fallback and teardown proof
    size: medium
    depends_on:
      - title-walkthrough
    description:
      "fail-open-proof: extend the shared title-block runner with native-rendering
      checks for Source, embeds, hover popovers, canvas, failed mounts and repaints,
      plugin disable, file/mode changes, and leaf teardown. Measure restoration at the
      first paint, preserve byte neutrality, and fix failures without weakening
      assertions."
  - id: visual-matrix
    title: Complete and inspect the family preview and accessibility matrix
    size: medium
    depends_on:
      - fail-open-proof
    description:
      "visual-matrix: compose the existing real card builders into the full eleven-case
      light/dark by 375px/wide preview matrix, add grayscale and forced-colors species
      proofs, inspect every image, and fix the owning DOM or CSS. Attach a durable
      indexed screenshot bundle and inspection record, and rerun the shared Obsidian
      proof after product changes."
  - id: album-rollout
    title: Verify every host and perform the gated live album tidy
    size: medium
    depends_on:
      - visual-matrix
    description:
      "album-rollout: recheck athena, apollo, and mac with their actual installed bob
      binaries; behavior-probe album title precedence in isolated fixtures. Review the
      live tidy dry run, apply only if all hosts are compatible, sync, and prove title,
      record-walk, index, and catalog stability. Otherwise attach the dry run and record
      the blocking host/version as PROPOSED FOLLOW-UP."
  - id: final-gates-deploy
    title: Run final gates and deploy the verified merged tree
    size: medium
    depends_on:
      - title-walkthrough
      - fail-open-proof
      - visual-matrix
      - album-rollout
    description:
      "final-gates-deploy: integrate the completed phases, repair the epic-caused
      vault-format word-budget regression if still present, run both repositories'
      required gates and the final combined proof, deploy through bob plugins sync, and
      run deployed-vault smoke with matching managed-file hashes. Attach the final
      evidence manifest for the child land agent; close only this worker's own phase."
proposed_by: bbugyi200.athena.bob-cli-o6.6
proposed_with: codex/gpt-6-astra
parent_bead: bob-cli-o6.6
create_time: 2026-11-21 08:22:45
status: wip
---

- **PROMPT:**
  [prompts/202611/title_block_final_proof.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/title_block_final_proof.md)
- **PARENT:**
  [202611/property_card_title_block.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/property_card_title_block.md)

# Finish the title-block proof and rollout

## Scope, sizing, and completion ownership

This is a child epic for **bob-cli-o6.6**, the `obsidian-proof` phase of **bob-cli-o6**.
The assigned phase had no prior notes or incomplete implementation when inspected. The
product work has landed; the remainder combines new real-editor test automation,
lifecycle failure proof, visual inspection, a three-host data migration, and final
integration. Five bounded coding units are appropriate; a single tale would combine too
many independent acceptance surfaces.

The phases are intentionally sequential. They share the title-block harness, generated
plugin bundle, CSS, and live deployment. Each phase can reuse the preceding phase's
runner and evidence. Dependencies above are the complete phase graph.

**Workers close only their own beads.** The child epic's land agent settles this child
epic and completes the delegated **bob-cli-o6.6** scope through the host's supported
child-epic closeout. No worker closes bob-cli-o6.6, bob-cli-o6, bob-cli-o6.7, or another
ancestor/sibling. The outer epic remains for its own land agent. The original task's
prohibition on manually creating task beads applies throughout: record discoveries as
`PROPOSED FOLLOW-UP:` on the worker's assigned bead, with duplicate IDs where known.

The outer DECISIONS are final: the authorized three glossary updates belong exclusively
to **bob-cli-o6.7**, and a decisions-memory entry was rejected. **This child epic
changes no memory files and asks no new design questions.** Do not change daily or
period title, navigation, template, or scaffold bytes; do not add mobile title
ownership, a bank-month card, or new title ownership for bank Setup. Preserve existing
keymaps and CLI/JSON contracts. No Cargo version or CHANGELOG edits.

## Verified starting point

Recheck these facts against the current tree before editing; ongoing work may have moved
the base. Read artifacts through `sase artifact read`, and open bob-plugins only through
`/sase_repo`. Paths below are repository-relative, never workstation-specific.

- Binding design: `plan:202611/property_card_title_block.md`, especially
  `obsidian-proof`, the title ownership contract, and the per-species byte policy.
  Accepted research:
  `research:202611/property_card_title_block/property_card_title_block.md`. If the
  latter initially resolves missing, refresh its research sidecar through `/sase_repo`
  and retry the audited read; the file exists.
- Read the current assigned bead and parent notes before making changes. bob-cli
  **d27041cd** includes **bb91c370**, which implemented title-first album reads, H1-less
  creation, and dry-run-first `bob albums tidy`.
- The inspected bob-plugins **1a95b29** includes album adoption **a68e29b**, record
  adoption **b8b7baf**, day/period adoption **eed318e**, and the trailing-title-blank
  and shared editor-badge fixes. bob-ledger-tools is **1.149.0**. Do not redo those
  features.
- `bob-cli-o6.1` records **TITLE SLOT: GO** for both Live Preview and Reading. Its
  primitive spike is `scripts/obsidian-title-slot-spike.cjs` with its library and
  offline wrapper. It already demonstrates placement, settings, geometry, and denied
  surfaces with a probe; the remaining proof must exercise actual cards and actual
  ownership, not merely repeat the probe.
- `bob-cli-o6.4` note 1 records the latest collection closeout: album 129/130 (sole
  known failure `bob-cli-nq`), record 180/180, record walk 18/18, intention 82/82,
  previews 28/28, deploy hashes and deployed-vault smoke. Evidence archive:
  `file:explicit:3f4377e85c60d8c949c94f82`; companion evidence:
  `file:explicit:3828c89b1d6161027caa83b4`.
- `bob-cli-o6.5` records 82/82 real-Obsidian checks and 52 intention-preview captures.
  Its report is `file:explicit:8446f75d3c5fc1b954fdc8e7` and preview report is
  `file:explicit:526c6d191f83b84ea35eb8c1`. These are precedent, not fresh evidence for
  a subsequently changed bundle.
- `plan:202611/collection_cards_title_block.md` and `bob-cli-o6.4.1.3` settle the bank
  exception: Setup keeps its existing heading, placement, and `Bank setup` accessible
  name; it receives a BANK species badge. Do not impose the album/day title-ownership
  assertion on bank or move its H1 as a way to make the test pass.
- Existing previews are `scripts/preview-property-cards.cjs` (actual album, record, bank
  builders), `scripts/preview-intention-card.cjs` (actual day/period builders), and
  `scripts/preview-title-block.cjs` (kit, contrast, grayscale and forced colors). The
  last is a kit demonstration with simplified bodies, not the full adopter matrix.
- On 2026-11-21 all three SSH aliases were reachable. Each installed binary reported
  `bob 0.5.0` and supported `bob albums tidy --help` describing title-first reads and
  the cross-host warning. Version strings alone do not identify these local builds;
  refresh this evidence and prove behavior before live apply.
- `bob-cli-oc` tracks the **6016/6000** `docs/reference/vault-format.md` lint failure
  introduced by the album-title phase. Outer epic notes explicitly assign its repair to
  this epic. Preserve the page URL and contract facts; a small wording trim suffices.

## Shared implementation and evidence rules

In bob-plugins read `AGENTS.md`, `docs/develop/architecture.md`, and
`docs/develop/deployment.md`. Edit source fragments, regenerate `main.js`, keep
hand-edited fragments within 1000 lines, preserve all concurrent manifest/test entries,
and use the next available plugin version when product behavior changes. Changes to the
shared kit API must remain additive. After changes, run `bob plugins sync` using the
opened source checkout and the existing refusal procedure; never force another
checkout's deploy. Fetch and integrate landed work before syncing.

Useful implementation seams in `plugins/bob-ledger-tools/src/`:

- `240e` title-slot primitive; `240f` species; `240g` title/rail DOM; `240h` range;
  `240i` ownership; `240j` badge; `240k` adoption; `374*` helper exports.
- `241u`/`241v` album model and DOM; `242i*`/`242j`/`242k*` record model, DOM and title
  host; `242w` record recognizer.
- `296a`/`296ab` model/identity, `296b` DOM, `296c` host, `296d` title slot for
  day/period cards; `248j`/`248k*` bank Setup DOM and card; `297cc` editor identity.

Use the existing owned-process, isolated-profile, temporary-vault CDP pattern and
existing utilities from the album/record/intention runners. Use `/snap/bin/obsidian`
explicitly on athena; older album/bank runs found that the internal snap executable can
exit before opening the fixture. Run under an independent `xvfb-run -a`. Do not attach
test automation to the user's live Obsidian session. Use a short scratch path when
Electron socket limits require it. The test subprocess must receive an explicit scratch
vault/config and must not accidentally invoke a writer against the live vault.

The shared runner reports source SHAs, dirty paths, managed plugin hashes and versions,
bob executable/version/hash, Obsidian version, fixture clock, fixture hashes, per-check
observations, screenshots, and console exceptions. Mandatory checks are a declared
inventory: missing, skipped, or unrun checks cannot turn into a passing verdict.
Incremental phases must state which portion they prove; only the complete final run can
claim the whole phase passed. The offline wrapper tests inventory completeness and
failure propagation, fixture generation, and argument handling without launching
Obsidian or masquerading as an integration pass. Register it in `npm test`.

Long commands and waits use `/sase_monitor`, with enough command context and explicit
next steps to recover after a failure. Await a handoff command's exit and finish through
`/sase_final` after it reports Staged. Attach evidence using `sase artifact create`;
archive screenshots with their index and hashes so relative image links remain useful.
Keep raw personal vault data out of public fixture reports.

## Phase title-walkthrough

Add `scripts/obsidian-title-block-e2e.cjs`, support modules as needed, and
`scripts/test-obsidian-title-block-e2e.cjs`. Reuse existing helpers without copying
whole runners or broadly refactoring their proven workflows.

Create minimal but real album/record fixtures, canonical today/past/future/empty days,
2024 and 2025 H1-less legacy daily shapes, the recognized legacy 2026 shape, and
week/quarter/year notes with navigation and child links. Include an album before tidy
and one with `title:` and no H1, an unrecognized authored H1 that must remain, and
source-heading links. Freeze the fixture clock consistently for bob and renderer.

Prove in Live Preview and Reading:

1. Each title-owning species mounts one card before Properties with the correct local
   title, tile and kicker. Exactly one visible accessible level-one title exists in the
   normal owned state. Exclude embeds from pane counts, and treat authored H1s and
   intentional source reveal as separate states, not accidental duplicate-title errors.
2. The inline title and only the recognized redundant H1/nav/children/blanks hide. A
   second pane of the same note and an unrelated note demonstrate per-view scope.
   Frontmatter selection or a restored cursor outside the range does not reveal it.
3. Send real CDP keys for Vim `gg`, `j`, and `k` across the hidden boundaries and `/`
   search into a hidden title. Record cursor positions, visible source text, and the
   title card remaining mounted. Moving away hides the redundant range again.
4. Exercise the real Outline UI and real `[[note#heading]]` navigation. Check resolved
   destination and visible scroll landing in both modes, and selection reveal where
   applicable. Do not substitute a direct `setCursor` call for the interaction under
   test; use it only to establish a precondition or native control comparison.
5. Edit a revealed source H1, verify authored nonredundant text remains visible, then
   perform native undo and verify exact original bytes. No writer may restore the file
   before this equality check. Separate this intentional-edit case from passive hashes.
6. Hash every fixture before and after passive open, fold, reveal, navigation and close.
   Day/period H1/nav bytes and all unrelated content stay identical.

Run the offline wrapper and real walkthrough on athena. Fix defects in the owning
fragment with a focused behavioral regression check. Attach the report and screenshots;
record the covered inventory and bundle hashes for the next phase.

## Phase fail-open-proof

Extend that runner, keeping the first phase's checks. For each distinct adapter family
(album, record, day/period), prove native rendering and removal of ownership classes,
source-hiding decorations/classes and owned-Properties hiding when relevant:

- switch to Source and back, switch files in one leaf, close a leaf, disable/re-enable
  the plugin, and force a repaint failure;
- render the same fixture inside an embed, hover preview, and canvas file node while a
  separate eligible pane owns its title;
- force the title-slot mount to return unavailable and force a real insertion failure,
  using reversible test instrumentation around the actual mount path. Verify the body
  fallback/native content remains visible, then remove the fault and verify recovery.

Install observation before the lifecycle action, and capture ownership state plus
computed visibility before the next paint/at the first animation frame. A late
screenshot after a long settle does not prove the same-frame restoration contract.
Exercise fresh and already-mounted cases so stale ownership and zombie observers are
detected. Keep a second unaffected pane as a scope control.

Retain mobile and print/export exclusions: use existing pure eligibility tests for
mobile, clearly labeled as such rather than claiming a physical mobile run, and CDP
print media for the rendering rule. Do not require new product behavior on those
surfaces. Unsupported desktop boundary rendering is a visible failure/limitation, never
a silent skip.

Check native inline title, recognized source heading/nav, Properties and absence of
stray card nodes where they apply. H1-less fixture expectations follow their actual
bytes. Maintain the fixture-hash guard and console exception checks. Run the extended
suite on athena, fix regressions, and publish the lifecycle evidence with source hashes.

## Phase visual-matrix

Compose or extend the existing previews using actual adopter builders and committed CSS.
Avoid a second hand-built approximation of the card body. Produce one index and
machine-readable manifest with the complete cross product:

| Dimension     | Required values                                                                                               |
| ------------- | ------------------------------------------------------------------------------------------------------------- |
| Theme         | light, dark                                                                                                   |
| Content width | 375px, wide (record the actual width)                                                                         |
| Case          | album, record, today, past day, empty day, legacy daily, week, quarter, year, bank Setup, broken album report |

That is **44 required captures**, plus any expanded/collapsed, pending-report,
long-title, modal, or accessibility variants. Ensure the measured content/card width
matches the scenario; setting a 375px viewport with undocumented padding is not the same
claim. Preserve useful existing baseline captures and inspect changes.

Add grayscale and `forced-colors: active` captures covering every species, with
identical display titles in a clearly labeled visual stress fixture. That fixture may
override presentation data solely for comparison; it must not change canonical
path-derived title behavior or substitute for real-card ownership proof. Use actual OS
media emulation for forced colors. Carry these checks into the shared runner's final
evidence index.

Inspect **every image**, recording filename, verdict and any repair. Contact sheets aid
review but must be legible, with full-size inspection for uncertain details. Verify
species word and tile shape, wrapping titles/actions/chips, compact rails, collapsed
identity, pending/broken report retaining local titles, visible keyboard focus, text
contrast >=4.5:1, stripe contrast >=3:1, CanvasText borders in forced colors, and
reduced motion. Bank remains a Setup badge, not a title-slot adopter. Confirm
album/record share area meaning and calendar cards use the temporal accent.

Fix actual failures in the owning DOM/CSS, regenerate the build, run affected tests, and
rerun the combined walkthrough when the bundle changes. Publish screenshots, index,
complete coverage manifest, inspection notes, and hashes as durable artifacts.

## Phase album-rollout

Read `tailnet.md` and `obsidian.md` through `/sase_memory_read` before host/vault work.
Use Bob's commands for live vault reads and writes; any direct repository inspection
must first use `/sase_repo`. This phase has authorization for the specified tidy and
sync once the compatibility condition holds; no extra permission gate is invented.

1. On **athena**, **apollo**, and **mac**, record the actual installed bob executable,
   version, binary digest, and tidy availability. In an isolated disposable vault on
   each host, make an album whose valid frontmatter title differs from the H1 and stem.
   Prove `bob albums list -b <fixture> -f json` selects the title. Also use an H1-less
   variant and `albums sync` against the scratch vault to prove generated labels use
   that title. Supply an isolated config and verify all command targets are scratch.
   Distinguish a failed probe from an unreachable host. Recheck compatibility directly
   before applying; the earlier planning-time `0.5.0` strings are insufficient.
2. Capture `bob albums tidy -f json` and the human dry-run diff from the live vault.
   Review every album's title additions/H1 removals and all refusals or kept headings.
   Capture current album titles and managed walk/index/catalog state. If tidy already
   landed, verify idempotence and label stability instead of redoing it.
3. If every host proves compatible, ensure the latest compatible plugin build has been
   deployed through the guarded sync procedure, then run **`bob albums tidy -a`**. Use
   its maintenance lock, full-preimage validation, scoped commit and sync lifecycle; do
   not manually edit albums, invoke manual git commits, or bypass refusals. Record the
   actual changed paths and transaction commit. Run `bob albums sync`, compare generated
   walk lines, indexes and catalogs with the preimage, and verify a second tidy dry run
   is empty. Record unrelated pre-existing stale blocks separately; do not attribute all
   pre-existing churn to migration or hide migration-caused churn.
4. Verify every changed album's title with Bob's report and an isolated Obsidian copy of
   the relevant migrated shapes, preserving parent/schema context. Do not open the live
   vault in the automation. Card titles must match their pre-migration display titles,
   with no native H1 required for the migrated form.
5. If any host cannot be verified or is old, **do not apply**. Attach the dry-run and
   host evidence, and add
   `PROPOSED FOLLOW-UP: Live album tidy awaits <host/version> — <failure and required compatibility check>`.
   This is an explicitly accepted terminal outcome for this phase. Do not install or
   upgrade remote hosts merely to bypass the condition. Similarly, record any
   preimage/schema refusal precisely rather than forcing a destructive migration.

Publish a concise migration report with a host table, command outcomes, stable-title
comparison and either the transaction evidence or the explicit blocked-host outcome.

## Phase final-gates-deploy

Integrate current upstreams and all prior phase changes using the opened checkouts.
Preserve other active work, especially the newer calendar navigation-group API already
present in bob-plugins. Regenerate generated entrypoints after conflict resolution.

Read `docs.md` before docs work. If the epic-caused `bob-cli-oc` overflow persists, trim
`docs/reference/vault-format.md` below its actual lint word cap without removing
contract facts, raising budgets, or moving its published URL. Record the repair's
evidence for the outer land agent; do not close that separate task bead.

Run all required gates on the final tree and retain exit codes and logs:

- bob-plugins: `npm run build:check`, `npm test`, `npm run docs:check` with
  `BOB_CLI_DIR` pointing to the current opened bob-cli checkout.
- bob-cli: `just check`, `just docs-check`, with the current linked plugin docs mounted
  through the supported docs tooling.
- The complete shared real-Obsidian title-block walkthrough on athena and the complete
  preview manifest/inspection against the final managed-file hashes. Reuse unchanged
  evidence by exact hashes; rerun affected evidence after changes.

Do not turn unrelated base failures into an endless blocker. Reproduce any such failure
on a clean base from the authorized checkout, record exact named assertions, base/final
revisions, logs and duplicate bead IDs in a `PROPOSED FOLLOW-UP:` entry, and close the
phase when its scope is done. Known examples are album Ctrl+J `bob-cli-nq` and
load-sensitive test budgets cited by the predecessor phases. A known-failure label alone
is insufficient; confirm the actual failure matches. Fix newly introduced title-block
failures, and do not waive missing new acceptance checks. Never describe known-failure
gates as green.

Deploy the final integrated tree with
`bob plugins sync --repo <opened-plugins> --no-pull`, after its dry run, following
`docs/develop/deployment.md` if protected. Never pass `-F` without the runbook's
explicit authorization. Run the required
`npm run smoke:obsidian -- --deployed-vault <vault> --report <report>` under Xvfb and
verify JSON deployment state plus `manifest.json`, `main.js`, and `styles.css` hashes
for each touched plugin; command exit alone is insufficient to prove sync.

Publish one final evidence manifest linking the walkthrough report, preview archive and
inspection record, migration report, gate logs, deployed smoke, source SHAs, hashes,
known failures and remaining follow-ups. Run `sase bead epic-symbols <own-phase>` and
resolve/re-key any entry before closing only the worker's phase.

## Child land-agent acceptance

The land agent verifies that all five phases are closed and their artifacts collectively
cover the original bob-cli-o6.6 description. Check that final deployed bytes match the
landed bob-plugins tree; if finalizers landed a newer relevant bundle, reconcile and
rerun affected proof rather than claiming a pre-merge screenshot represents it. An
unchanged bundle does not require another expensive run merely because the commit SHA
changed. The live tidy must be verified applied/idempotent or explicitly deferred by the
allowed host-compatibility condition with its follow-up recorded.

Run `sase bead epic-symbols` for this child epic and **bob-cli-o6.6** before delegated
closeout. Resolve each symbol or re-key it to genuinely remaining work on an open bead.
Only the child land agent completes bob-cli-o6.6 through the host's delegated closeout;
it never closes bob-cli-o6, bob-cli-o6.7, or unrelated task beads. Forward the evidence
manifest and all proposed follow-ups to the outer epic's land agent for final triage.
