---
tier: epic
title: Mac Capture reference, media, and documentation front door
goal: Complete bob-cli-a0.11 with preserved Mac Capture reference, actionable recovery
  guides, CI-rendered images, a useful README, and verified shared-site mounts.
phases:
- id: capture-reference
  title: Extract capture, keyboard, and protocol reference and mount it
  size: medium
  depends_on: []
  description: 'capture-reference: Create the Mac reference and developer hubs, extract
    the capture-side README content into bounded reference pages and an architecture
    page, consolidate capture protocol compatibility, and add both bob-cli mounts.
    Leave the README intact. Follow the capture-reference section and shared contracts
    below.'
- id: application-reference
  title: Extract installation, settings, permissions, and Bob Refs reference
  size: medium
  depends_on:
  - capture-reference
  description: 'application-reference: Extract the remaining application reference
    into install, settings, permissions, diagnostics, refs, refs-inspector, and developer
    build/CI pages. Extend nav and the protocol compatibility table. Keep troubleshooting
    and the original README available until the final migration phase. Follow application-reference
    below.'
- id: rendered-media
  title: Import CI panel images and add Mac documentation preview tooling
  size: medium
  depends_on:
  - application-reference
  description: 'rendered-media: Add Scripts/docs-media.sh to import a curated set
    of real synthetic CI renders with provenance, Scripts/docs-check.sh and just recipes,
    focused script tests, render-fixture contributor instructions, and the Mac AGENTS.md
    docs instruction. Use real panel images in the mounted reference and bob-cli homepage.'
- id: guides-front-door
  title: Migrate troubleshooting, finish the README front door, and verify coverage
  size: medium
  depends_on:
  - rendered-media
  description: 'guides-front-door: Move both troubleshooting sections into goal-titled
    bob-cli Fix problems guides, finish all cross-links and search cases, audit every
    original README section, then replace the README with a useful front door retaining
    an exact Signing heading. Validate both builders and the combined docs gate. Close
    only the assigned child phase; leave bob-cli-a0.11 for this child epic''s land
    agent.'
proposed_by: bbugyi200.apollo.bob-cli-a0.11
proposed_with: codex/gpt-6-astra
parent_bead: bob-cli-a0.11
create_time: 2026-10-20 03:05:20
status: wip
bead_id: bob-cli-a0.11.1
---

- **PROMPT:** [prompts/202610/mac_capture_reference.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202610/mac_capture_reference.md)
- **PARENT:** [202610/bob_public_docs_site.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/bob_public_docs_site.md)
- **BEAD:** [bob-cli-a0.11.1](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-a0/bob-cli-a0.11.1.md)

# Mac Capture reference, media, and documentation front door

## Outcome and scope

Complete the `mac-ref` phase of `plan:202610/bob_public_docs_site.md`, tracked by
`bob-cli-a0.11`. Mac reference stays beside its code in `bob-mac-capture/docs/`, mounted
at `/reference/mac-capture/`, with contributor pages mounted separately at
`/develop/mac-capture/`. Troubleshooting belongs in bob-cli's Guides → Fix problems. The
Mac README becomes a short front door, and the homepage displays an actual synthetic
panel render.

This is a child epic because the inspected README has 2,810 lines and about 29,366
words: Runtime Contract alone has 10,528 words, Keyboard 5,270, and Bob Refs 5,977.
Preserving that material while adding media import and cross-repository integration is
better divided into four bounded implementation agents. There were no previous
remaining-work notes on `bob-cli-a0.11` at planning time.

No Swift, application behavior, installer behavior, Rust runtime behavior, new public
Bob commands, publishing infrastructure, or memory changes belong here. Do not modify
bob-plugins. The parent decisions are final: MIT, search indexing, and published design
decisions remain selected. Its authorization for `docs.md` memory edits is not needed
for this work; no memory edit is planned.

## Required context and verified starting points

Before implementation, use `/sase_repo` to open bob-mac-capture and read its
`AGENTS.md`. Use repository-relative paths throughout this plan. Open bob-plugins only
if consulting its existing `scripts/docs-check.sh` pattern. Read memory via
`/sase_memory_read`, including `docs.md` and the relevant glossary/decision strands.
Read both artifacts with `sase artifact read`:

- `plan:202610/bob_public_docs_site.md`, especially mac-ref and the common rules.
- `research:202610/bob_public_docs_site/bob_public_docs_site.md`.

The research initially failed to resolve, then succeeded after
`sase repo open bob-cli--research`; it is available. Never open sidecar documents
directly. Runtime facts should be checked in the current source, not inferred from the
age of the original design.

The inspected Mac baseline was `065908f89c1646dcbf77dad90e4de0f4cbdb4607`. There was no
`docs/` tree. `SettingsView.swift` orders sections as Bob, Hotkey, References, Agenda,
Canceled Draft Stash, Saved Templates, Launch, Notifications, Diagnostics.
`BundleSigningInspectorTests` asserts that the signing advice mentions README, so retain
a literal `## Signing` in the front door.

At planning time the latest successful master CI run was `39330761089`, SHA
`c81e00a3a4c6da272ac7591341fd85355162ce53`, created 2026-10-19. Its unexpired
`render-fixtures` artifact was about 107 MB. Query afresh when importing; this
observation is evidence of availability, not a hard-coded run selection. CI already
renders Swift views using `BOB_MAC_CAPTURE_RENDER_DIR` and uploads the artifact with a
14-day retention period. No renderer changes are needed.

The bob-cli assembler already handles nested source mounts, excludes the nested source
from its parent, stamps source provenance, and stages output under `target/docs/`. Local
mount discovery recognizes SASE linked repos. `nav.yml` is a list of `{title, path}`
entries; `index.md` is inserted automatically and must not be listed again. `load_media`
currently permits only `source` and `date`, so put run URL/ID, full SHA, fixture
identity, and synthetic origin in the `source` string rather than inventing manifest
keys.

## Shared implementation contracts

1. Preserve observable behavior and meaningful caveats. Extract existing material before
   shortening the README. Move protocol details out of reader walkthroughs, consolidate
   repetition, and replace duplicate explanations with links. Do not silently discard
   unique keyboard rules, compatibility fallbacks, failure behavior, or privacy details.
2. The CLI owns capture grammar, completion data, previews, and vault mutation. The app
   owns presentation, caching, process orchestration, settings, and packaging. The
   agenda's facts come from `capture-pomodoros --tasks`.
3. Every page has allowed front matter with `title` and a description of at most 160
   characters. Reference pages stay at or below 6,000 counted words; hubs stay
   below 400. Split at useful headings instead of increasing budgets. Guides have goal
   titles, `macOS`/`CLI` component tags as applicable, prerequisites, confirmed steps, a
   failure section, and related links. Preserve portable Markdown.
4. Same-mount links are relative. Cross-mount and cross-repository links, including all
   README documentation links, use `https://bobs-org.github.io/...`. Do not use relative
   links from `docs/develop/` into its source parent's pages: these publish under
   different mount roots.
5. Do not introduce private examples or weaken privacy lint. Import only inspected
   synthetic render fixtures; all image files must have manifest entries. Keep source
   paths and implementation details in developer/reference pages where useful, and bead
   IDs out of published reference and reader pages.
6. All phases append a concise README-section-to-destination mapping to their bead notes
   for the final audit. Include the relevant mapping in the repository's eventual
   finalizer commit description; manual git commits are not required. Re-read README
   changes since the recorded baseline before deleting content.
7. Do not add exception symbols merely to defer documentation defects. Before closing
   each assigned child phase, run `sase bead epic-symbols <assigned-id>`, resolve or
   re-key leftovers to an actually open bead, and close only that phase. Record
   out-of-scope findings on the assigned phase as `PROPOSED FOLLOW-UP: ...`; do not
   create task beads.

## Phase: capture-reference

Work in bob-mac-capture and bob-cli. Keep the original README untouched.

Create `docs/index.md` and these reference pages, splitting further only where a
coherent topic exceeds the budget:

- `capture-panel.md`: panel lifecycle, editor, layout, submission/error behavior, live
  preview, clipboard semantics, and links to its detailed pickers.
- `pickers.md`: Task Link, Go To, Parent Task, Dependency, Blocks, and Complete pickers,
  including cancellation, selection, ID assignment, and action continuation. If
  necessary split into `task-pickers.md` and `relationship-pickers.md`, keeping
  `pickers.md` a useful landing page.
- `capture-previews.md`: preview cards, aggregate task/Pomodoro blocks, session
  controls, dependencies, completion outcomes, and successor-selection behavior as
  actually documented in the current README and implemented source.
- `keyboard.md`: the full Keyboard section, including shelf keys and Control-\\. Keep
  exact shortcut behavior; consolidate repeated narrative into links to the relevant
  reference rather than dropping it.
- `saved-templates.md`, `wikilink-completion.md`, and `idle-agenda.md`: their dedicated
  README sections, linked from the panel page.
- `protocol.md`: subprocess commands, JSON/schema boundaries, capability requirements,
  unsupported-response behavior, and one compatibility table. Move the long capability
  list from Requirements here. Consolidate scattered older-Bob notes using columns for
  feature/field, expectation, absent-field behavior, and reader action. Distinguish
  required commands from optional fields; do not invent minimum release numbers.

Create `docs/develop/index.md` and `architecture.md` for source layout and process,
presentation, cache, and failure boundaries extracted from Runtime Contract. Keep exact
externally observable rules in reference pages. Create both `nav.yml` files covering
only real pages present in this phase, and an empty `docs/media.yml` until the media
phase adds assets.

Add two mount entries in bob-cli `docs/_meta/mounts.yml`, both for `bob-mac-capture`,
remote `https://github.com/bobs-org/bob-mac-capture`, branch `master`: `docs` →
`reference/mac-capture` (Mac Capture), and `docs/develop` → `develop/mac-capture` (Mac
Capture development). Add matching `mkdocs.yml` placeholders under Reference and
Develop. Ensure both destinations are gitignored. Link the new hubs from the
corresponding bob-cli hubs and point the homepage Mac card at the reference hub, leaving
its image replacement for the media phase. Add a small search smoke case targeting a
distinctive Mac keyboard/agenda topic.

Verification: assemble with explicit paths to the SASE-opened sibling checkouts, inspect
nav/source provenance, and run bob-cli `just docs-check`. Check the extracted sections
against the original README, including fenced examples. Run meaningful existing mount
tests only if mount machinery needs modification; the existing machinery should suffice.

## Phase: application-reference

Extend the now-mounted source tree, leaving the README intact.

- `install.md`: actual platform/toolchain requirements, build/install commands, signing
  identities and keychain cases, updates, reinstall behavior, rollback, uninstall, and
  permission effects. Link developer details without losing instructions needed to
  install. Keep source builds and signed installed bundles distinct.
- `settings.md`: current SettingsView section order above, actual controls and defaults
  verified against `AppSettings` and related source. Do not invent an in-app control for
  CLI-owned configuration. Link underlying reference pages.
- `permissions.md`: Accessibility for Highlights tabs, notification authorization,
  launch at login, and both Privacy sections. Preserve what is stored on disk, what
  stays in memory, and what diagnostic logs omit.
- `refs.md`: Data, Updates, Sorting, NEW, tuning, panel interactions, Copy/Rename/ Link
  to Today cards, opening the panel, Tabs, and all their current constraints. Move
  Inspector to `refs-inspector.md`; split additional sections if necessary rather than
  crowding the 6,000-word ceiling.
- `diagnostics.md`: the Diagnostics and Signposts section, Recent Activity, `refs-rank`
  invocation/tuning material, command timeouts, and diagnostic copy behavior. Include
  the Menu-Bar Icon state meanings here; place rendering implementation notes in
  developer material.
- `docs/develop/build.md` and `ci.md`: Development, toolchain selection, portable
  checks, bundle and test recipes, and the CI section with links to render-fixture
  coverage added by the next phase.

Extend `protocol.md` with Bob Refs command/field compatibility and consolidate any
remaining scattered fallback notes. Add all pages to their source nav and hub. Link the
read-only troubleshooting sections in the existing README until their guide destinations
exist in the last phase; avoid unresolved future links.

Verification: inspect settings against source, read pages as mounted, check budgets and
section mappings, and run bob-cli `just docs-check`. Instructions that write state must
be checked in isolated scratch state; do not install into the user's running Mac or
manipulate their permission grants.

## Phase: rendered-media

Add `Scripts/docs-media.sh` and a `just docs-media` recipe in bob-mac-capture. The
script queries successful `ci.yml` runs on master, chooses the latest, verifies artifact
availability, then uses `gh run download --name render-fixtures` into a temporary
directory. Fail clearly if no successful run, an expired/missing artifact,
authentication failure, or a required curated image is missing. Leave the previously
committed images and manifest intact on these failures.

Use an explicit, small curated filename list. Inspect the downloaded files and their
render-test fixtures before selecting them. Suitable starting families are the current
idle agenda, a task/preview card, and the basic Refs browse panel. Choose standard-width
files and retain only useful images, not the full artifact. Stage and validate the
complete selection before replacing docs assets. Record original artifact filename, run
URL/ID, full SHA, synthetic-fixture provenance, and run ISO date in `docs/media.yml`
using the existing two-key schema. Use stable descriptive destination filenames under
`docs/images/`.

Prefer light/dark pairs with `#only-light`/`#only-dark` only after testing actual
computed visibility in both strict Zensical and Material output. If either builder does
not honor the pair, commit and show only the light image. This is the parent's approved
fallback, not a new decision. Inspect image readability and privacy at normal desktop
and narrow viewport sizes. Keep accurate alt text and natural aspect ratio. Replace the
placeholder mark and phase-related sentence in bob-cli `docs/index.md` with the real
panel image; use the mounted image rather than a second copied asset. Add relevant
images to the Mac reference pages.

Add `Scripts/docs-check.sh` and `just docs-check`, based on the inspected bob-plugins
wrapper: resolve `BOB_CLI_DIR`, then `../bob-cli`, then a managed cache, and invoke
bob-cli's `just docs-check` with `BOB_DOCS_BOB_MAC_CAPTURE_DIR` set to the Mac source.
Explicit invalid/empty overrides fail; user checkouts are never reset. Preserve
downstream exit status and quote paths containing spaces. SASE agents must use explicit
`BOB_CLI_DIR` pointing at their authorized checkout, so the script's end-user cache path
is not a substitute for `/sase_repo`.

Add focused script tests, with stub `gh`/`just` and local fixtures: selection of a
successful master run, missing/failed download, missing selected image, preservation of
previous outputs, provenance consistency, override validation, source forwarding, paths
containing spaces, and failure exit propagation. Keep dependencies minimal. These tests
exercise the import/preview behavior; do not snapshot documentation prose.

Create `docs/develop/render-fixtures.md` explaining source tests, artifact retrieval,
curation, provenance, refresh commands, and the builder appearance check. Update its
nav/hub. Add one line to the Mac repo's hand-authored `AGENTS.md` that docs are mounted
into Bob's site and changes require `just docs-check`; preserve all Swift requirements.

Verification: shell syntax, focused script tests, an actual successful import,
manifest/image parity, both builder image checks, and both the Mac wrapper and bob-cli
docs gate. Follow Mac AGENTS.md's portable-check instruction on a host without Xcode; if
Swift is missing, use `/sase_repo` and the tailnet instructions for the prescribed
Swift-capable host. No Swift source edits or app installation.

## Phase: guides-front-door

Inventory every item in README's two Troubleshooting sections and their adjacent
recovery instructions. Put user recovery steps into a small set of goal-titled bob-cli
guides under Fix problems:

- `fix-mac-capture-bob-resolution.md`: resolve the executable, stale targets, timeouts,
  and command/completion diagnostics.
- `resolve-mac-capture-hotkey-conflict.md`: active bindings, registry conflicts, and
  conflict feedback.
- `restore-mac-capture-permissions.md`: notifications, stable signing, Accessibility,
  and launch-at-login recovery.
- `recover-bob-refs.md`: library load/scan failures, Highlights app selection, stale
  grants, tab-close failures, history reset, and bounded retry behavior.
- `recover-mac-capture.md`: retained drafts, block-ID errors/focus, clipboard delays,
  restart failures, and a missing menu-bar item.

Build/XCTest diagnosis belongs in the mounted developer build page; link there from
recovery pages. Reuse existing guides for shared vault/ref problems instead of
duplicating their procedures. Adjust these guide boundaries only to preserve one reader
goal and the existing guide budgets; account for every migrated item. Use diagnostic
instructions safe to copy, distinguish observation from writes, and preserve the
existing confirmation behavior for destructive user actions.

Update `docs/guides/index.md`, Fix problems navigation, Mac page links, and
`docs/_meta/search_smoke.yml`. Retarget existing `start/mac-capture.md`,
`guides/capture-from-your-mac.md`, and other Mac README-anchor references where the new
page owns the answer. Keep the Signing anchor valid. Run the component generator through
the existing docs generation workflow; do not hand-edit its block.

Audit the mapping below and every level-three heading against the original README at the
baseline and any subsequent additions. Only after all destinations exist, replace README
with: one-sentence description, requirements and a usable install snippet, absolute site
links to Get started/Reference/Develop and major topics, a short `## Signing` that
explains stable identity and links to the full signing reference, and MIT license
information. Keep it useful from GitHub without a live site. Include the complete
section-to-page mapping in the finalizer commit description and record coverage evidence
on the assigned child bead.

Final verification: full docs gate from bob-cli and through the Mac wrapper, strict
Material fallback, URL/anchor/search and privacy gates, image-manifest coverage, page
budgets, and manual checks of the README links, Mac settings page, keyboard lookup, Refs
recovery, and homepage in light/dark/narrow views. Use the existing checks; add
automated tests only for new script behavior or a real tooling change. Do not rewrite
runtime tests to pin guide prose.

## Required migration coverage

| Original README section                                       | Destination                                                                             |
| ------------------------------------------------------------- | --------------------------------------------------------------------------------------- |
| Introduction                                                  | README and reference/developer hubs                                                     |
| Requirements                                                  | install.md for platform; protocol.md for capabilities/fallbacks                         |
| Development and Signing                                       | develop/build.md and install.md; README retains Signing                                 |
| Menu-Bar Icon                                                 | diagnostics.md; render details in develop/render-fixtures.md                            |
| Runtime Contract, including all picker subheadings            | capture-panel.md, pickers.md, capture-previews.md, protocol.md, develop/architecture.md |
| Completing tasks with `!`; Saving links to your reading queue | capture-previews.md / capture-panel.md with canonical CLI contract links                |
| Bob Refs: Data, Updates, Sorting, NEW, Tuning                 | refs.md, diagnostics.md, protocol.md                                                    |
| Bob Refs: panel, Copy/Rename/Link to Today, Opening, Tabs     | refs.md and keyboard.md cross-links                                                     |
| Bob Refs: Inspector                                           | refs-inspector.md                                                                       |
| Bob Refs: Privacy and Troubleshooting                         | permissions.md and recover-bob-refs guide                                               |
| Idle agenda                                                   | idle-agenda.md                                                                          |
| Keyboard and its shelf/Control-\\ subheadings                 | keyboard.md                                                                             |
| Saved templates                                               | saved-templates.md and settings.md cross-link                                           |
| Wikilink Completion                                           | wikilink-completion.md                                                                  |
| Live Preview and Clipboard Semantics                          | capture-panel.md                                                                        |
| Notifications                                                 | permissions.md, settings.md, recovery guide                                             |
| Hotkey Registry, Conflicts, and Launch at Login               | keyboard.md, settings.md, permissions.md, hotkey guide                                  |
| Updating, Reinstalling, and Rollback; Uninstalling            | install.md                                                                              |
| Privacy                                                       | permissions.md                                                                          |
| Troubleshooting                                               | Fix problems guides and develop/build.md                                                |
| Diagnostics and Signposts                                     | diagnostics.md                                                                          |
| CI                                                            | develop/ci.md and develop/render-fixtures.md                                            |

## Land-agent completion and failures

Child workers close only their own assigned phases. After all four phases are closed,
this child epic's land agent verifies the combined changes and the full mac-ref
acceptance criteria, records the coverage mapping and validation evidence, and completes
the child epic through its normal land flow. It then runs
`sase bead epic-symbols bob-cli-a0.11`, resolves or re-keys any leftovers, and closes
**only the assigned outer phase** with
`sase bead close bob-cli-a0.11 --note "<coverage, images, and checks verified>"`. The
user authorized completing this phase through a child epic. Do not close `bob-cli-a0` or
any other ancestor plan bead.

No symbol exemptions existed for `bob-cli-a0.11` at planning time; recheck at land. If a
check fails identically on an untouched base with the same source inputs, record
`PROPOSED FOLLOW-UP:` on the responsible assigned phase with the command, diagnostic,
clean-base evidence, and any existing tracker ID. Such an unrelated failure does not
keep the completed phase open. Regressions introduced here must be fixed. Do not
silently skip required checks or report unavailable macOS verification as completed.
Record unresolved external prerequisites explicitly. Use `/sase_monitor` for genuinely
long commands and `/sase_final` for normal turn completion; every modified repository is
part of the final declaration.
