---
tier: epic
parent_bead: bob-cli-a0.10
title: Publish the generated Obsidian plugin reference
goal:
  Complete bob-cli-a0.10 with source-owned documentation for all seven plugins,
  generated command and version facts, a concise README, and verified reference and
  development mounts in the Bob site.
phases:
  - id: mount-hubs
    title: Support source-owned mounted documentation hubs
    size: medium
    depends_on: []
    description:
      "mount-hubs: add compatible support for a mounted repository's index.md while
      preserving authored landing pages, collision guards, provenance, and navigation."
  - id: inventory
    title: Generate plugin facts and registered commands
    size: medium
    depends_on: []
    description:
      "inventory: implement the dependency-free static inventory scanner, generated
      documentation regions, drift checks, and focused scanner tests in bob-plugins."
  - id: plugin-pages
    title: Write the seven plugin reference pages
    size: medium
    depends_on:
      - inventory
    description:
      "plugin-pages: move the plugin-specific README details into seven usable reference
      pages with settings, keyboard-handler distinctions, and authoritative contract
      links."
  - id: topic-migration
    title: Preserve README topics and create the plugin front door
    size: medium
    depends_on:
      - plugin-pages
    description:
      "topic-migration: migrate the remaining README topics and development material,
      add both source navigation files, audit coverage, and shorten the README only
      after every detail has a home."
  - id: site-integration
    title: Mount the plugin reference and verify its complete workflow
    size: medium
    depends_on:
      - mount-hubs
      - topic-migration
    description:
      "site-integration: add the two mounts, site entry links, generated-content gate
      integration, the sibling preview wrapper, and end-to-end verification evidence for
      the land agent."
proposed_by: bbugyi200.apollo.bob-cli-a0.10
proposed_with: codex/gpt-6-astra
create_time: 2026-10-19 23:15:59
status: wip
---

- **PROMPT:**
  [prompts/202610/plugin_reference.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202610/plugin_reference.md)
- **PARENT:**
  [202610/bob_public_docs_site.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/bob_public_docs_site.md)

# Publish the generated Obsidian plugin reference

## Objective and authority

Complete the `plugins-ref` scope of `bob-cli-a0.10`. Plugin documentation remains owned
by `bob-plugins`; Bob's site publishes it at `/reference/plugins/` and
`/develop/plugins/`. Readers can find every plugin, its registered commands and
defaults, additional keyboard behavior, settings, and detailed topics without reading
the old monolithic README.

This is a child epic because the static JavaScript scanner, documentation migration, and
safe mount integration are separate substantial tasks. Each phase above fits one coding
agent. `mount-hubs` and `inventory` are independent; documentation phases are ordered to
avoid conflicting edits to the same source files. The integration phase prepares
evidence; the child epic's land agent owns final acceptance and completion of the
enclosing phase.

Read these through the audited artifact commands:

- `plan:202610/bob_public_docs_site.md`, especially `plugins-ref`, the mount contract,
  and the rules every phase follows.
- `research:202610/bob_public_docs_site/bob_public_docs_site.md`.

The research artifact was initially missing locally and became readable after opening
`bob-cli--research` with `/sase_repo`. Retry through that workflow if needed. Do not
read sidecar files directly.

Inherited decisions are final: MIT, search indexing enabled, generated public design
decisions enabled, and memory authorization limited to `docs.md`. This child epic needs
no memory changes. Do not edit generated bob-cli `AGENTS.md` or any memory. The
requested one-line addition to bob-plugins' ordinary `AGENTS.md` is in scope. No plugin
runtime changes, version bumps, new CI, private-vault edits, new images, or publication
infrastructure are required.

Use `/sase_repo` before accessing bob-plugins; use only the checkout it prints. Read its
`AGENTS.md`. Its rule requires `bob plugins sync` after **any** repository change,
including documentation. Run it without force and record the result. Do not edit
installed vault copies. Each worker follows `/sase_final` for its own changes; the final
declaration must include every repository it modified.

## Evidence and implementation boundaries

Planning inspected bob-cli `f8ef1628` and bob-plugins `bd6ff0a`; refresh the evidence
against the actual implementation tree rather than freezing these versions:

- The phase is in progress, has no prior remaining-work notes, and has no
  `--epic-symbol` entries. Neither repository had working-tree changes.
- bob-plugins has no docs tree or docs scripts. Its README is about 10,400 words,
  including a roughly 2,600-word Navigation Hotkeys table cell, substantial Ledger Tools
  and Task Status Cycler behavior, topic sections, and shared API descriptions.
- All seven manifests currently report `minAppVersion: 1.8.7`. Five plugins use
  `src/fragments.json`; `bob-project-tasks` and `bob-vim-surround` use handwritten
  `main.js`. Inventory discovery must not hard-code either list.
- Current registration sites number 2, 2, 15, 41, 3, 0, and 8 for Block ID Prompt,
  Articles, Ledger Tools, Navigation Hotkeys, Project Tasks, Vim Surround, and Task
  Status Cycler respectively. Navigation's last site expands the six entries in
  `YANK_PATH_COMMANDS`, so the expected current command total is 76, not 71. Treat this
  as a source-audit checkpoint, not a permanently fixed command count.
- `scripts/build-plugins.mjs` already validates fragment order and build freshness;
  `npm test` explicitly names its Node test files. No inspected tests pin the README.
- `tools/docs/bob_docs/mounts.py` supports nested same-repo source partitioning.
  However, it rejects a source `index.md` and requires a bob-cli-owned landing page.
  `.gitignore` exempts each mount's `index.md` from its ignored directory. The epic
  requires the plugin inventory hub to live in bob-plugins, so that mismatch must be
  resolved as part of this work.
- Existing `nav.yml` accepts a flat ordered list of `{title, path}` entries and inserts
  the landing page itself. Keep this schema and omit `index.md` from nav.
- `bob_docs/check.py` assembles once, then checks generation and builds.
  `gen_contracts.py` currently dispatches contract and memory-reference generation.
  Extend the existing workflow instead of creating a separate site builder.

Before writing prose, read `docs.md` through `/sase_memory_read` and follow
`docs/develop/writing-docs.md`. Read glossary terms and decision strands through that
skill when relying on their meaning. Preserve current behavior from source and tests; do
not reinterpret an accepted decision while moving documentation.

## Shared content contract

- Each typed page has a title and a description of at most 160 characters.
- Hubs have at most 400 handwritten words; reference pages at most 6,000. Split
  supporting material when necessary; do not raise budgets to accommodate a dump.
- Generated regions use `<!-- BEGIN GENERATED: name -->` and matching END markers.
  Preserve handwritten content outside these regions. Generated facts do not count
  against prose budgets; do not mark handwritten behavior as generated.
- Keep normal Markdown tables and GitHub callouts. Shell commands use `bash` fences
  without prompts, and output uses the documented no-copy `text` convention.
- Links within one mounted source tree are relative. Links between mounts or into
  bob-cli use absolute `https://bobs-org.github.io/.../` URLs. README links are
  absolute, including source/license links. Do not invent future-page URLs that the
  current build cannot resolve; use an existing appropriate destination and record a
  later retargeting need if another epic phase has not landed.
- Reference prose contains no bead IDs. Replace personal paths and examples with
  synthetic ones, retain useful technical detail, and preserve stable public URLs.
- Distinguish registered default hotkeys from custom `.obsidian/hotkeys.json` bindings
  and editor/Vim handlers. An absent `hotkeys` field means no registered default, not an
  absence of all keyboard behavior. Manifest desktop flags alone do not justify a claim
  that every feature works on mobile.
- Omit screenshots; no media manifest is needed unless an image is actually added.

## Phase: mount-hubs

Work in bob-cli only. Add a narrow compatibility extension to `mounts.py` and its tests
so a source-owned `index.md` can be copied and stamped like any other page:

1. When the selected source supplies `index.md`, it is the hub. Its destination must be
   ignored and untracked; a tracked or unowned destination is still a collision. A
   previous copied hub may be refreshed and cleaned up through the existing ownership
   manifest. Never silently replace an authored landing page.
2. When the source has no index, retain the existing authored bob-cli landing-page path.
   Keep its validation and current behavior for other mounts. No new mount configuration
   field or link rewriting is necessary.
3. Include either hub exactly once, first, when expanding the unchanged placeholder and
   `nav.yml` schema. Copy/stamp source hubs before reading their titles. Preserve
   most-specific-source partitioning for `docs/` plus `docs/develop/`, so developer
   content never appears twice.
4. Preserve all existing guardrails: preflight before writes, ignored destinations,
   symlink confinement, spoofed-provenance rejection, stale-output protection,
   production SHA/cleanliness verification, and source discovery.
5. Update `tools/docs/README.md` to describe both supported hub ownership forms. Remove
   only the plugin-index ignore exceptions when integration activates those source-owned
   hubs; leave Mac behavior intact.

Tests must demonstrate a source hub and nested developer hub with provenance and one nav
entry each, repeated assembly and cleanup, an authored-hub mount still working, and
collision/ignored-path/symlink failures leaving outputs untouched. Adapt tests that
assume _every_ source index is illegal without weakening real collision tests. Include
edit-link provenance and lint coverage for copied hubs.

Acceptance: targeted mount/assembly/lint tests and `just docs-check` pass with fixture
source hubs while the committed mount table remains unchanged. This phase does not
activate incomplete plugin documentation.

## Phase: inventory

Work in bob-plugins. Add `scripts/docs-inventory.mjs` with no external dependencies:

1. Read all `plugins/*/manifest.json` deterministically. For built plugins, read only
   the ordered fragments listed by `src/fragments.json`; otherwise read `main.js`. Never
   scan both generated output and its sources.
2. Use a small token-aware static reader that can skip comments and strings and balance
   delimiters. Do not execute plugin source or evaluate arbitrary JS. Support literal
   command registrations across lines or on one line, guarded registrations, and the
   current constant-array `.forEach` pattern with literal `command.id` and
   `command.name` values such as `YANK_PATH_COMMANDS`.
3. Account for every actual `addCommand` registration. Unknown argument forms,
   unresolved id/name/hotkey expressions, malformed tables, and duplicate command IDs
   must fail with file/line context. Ignore misleading text in comments or string
   literals. Do not infer command metadata from callbacks or duplicate registration text
   using broad regex matches.
4. Generate a concise inventory in `docs/index.md`, facts and command/default-hotkey
   tables in each `docs/<plugin-id>.md`, and a minimum-version requirements block in
   README. Use manifest name/id/version/description/minimum version/desktop declaration
   appropriately, with table escaping and stable sorting. Calculate the required
   Obsidian minimum as the maximum semantic version, not a lexicographic comparison.
   Keep all existing README detail for later migration.
5. Start each plugin page with a real purpose sentence and typed frontmatter, then
   generated regions for facts and commands. Zero-command plugins explicitly say they
   register no palette commands. This phase owns the generator and scaffold; the next
   phase owns the complete prose.
6. Provide `-c/--check` and `-h/--help` with clear, alphabetized option help. `--check`
   reports all stale/missing outputs and exits nonzero without writing. Normal
   generation updates regions deterministically; compute/validate the output set before
   writing so a parse failure cannot leave half-regenerated docs.
7. Add `docs:gen` and a local `docs:inventory:check` npm script. Include inventory drift
   checking in both `npm test` and `npm run validate`, retaining build checks and the
   explicit existing test list. Add the scanner test file explicitly.

Use `node:test` fixtures covering each real registration form, data-table expansion,
no-command plugins, nested callback syntax, misleading comments/strings, escaped
metadata, unsupported dynamic forms, duplicate IDs, and nonwriting drift checks. Test
that generation preserves prose and a second generation changes no bytes. Audit the real
seven-plugin scan against source, including all six yank commands.

Acceptance: focused scanner tests, generator/check/idempotence checks,
`npm test && npm run validate`, and required plugin sync. No plugin runtime file needs
to change.

## Phase: plugin-pages

Work in bob-plugins after `inventory`. Complete the seven existing generated pages:

| Page                        | Handwritten material to preserve and explain                                                                                            |
| --------------------------- | --------------------------------------------------------------------------------------------------------------------------------------- |
| `block-id-prompt.md`        | Block ID prompts, task selection, link/unlink behavior, context and dependency-line guards, picker behavior, and relevant configuration |
| `bob-articles.md`           | Format/adopt commands, save/leave triggers, actual settings-tab keys/defaults, and article appearance controls                          |
| `bob-ledger-tools.md`       | Snippets, supported rendered blocks, budgets and badges, review/freshness displays, session toggles, and configuration keys             |
| `bob-navigation-hotkeys.md` | Navigation and counts, task and Pomodoro gestures, review controls, note/tab behavior, custom/Vim bindings, and configuration           |
| `bob-project-tasks.md`      | What project-task synchronization adds, its commands and settings/absence of settings                                                   |
| `bob-vim-surround.md`       | Supported Vim operators and delimiters, padding, symmetric-run behavior, rejected keys, and settings/absence of settings                |
| `task-status-cycler.md`     | Task/Pomodoro completion and recovery, successor offers, lane controls, summaries/logs, reference lifecycle, and configuration          |

Move and organize the seven README table descriptions and the plugin prose embedded
under the misleading `Task Card` heading. Keep the README until the next phase confirms
coverage. Preserve the generated regions untouched.

Use the actual registration and handler sources for binding claims, and current
defaults/settings readers for configuration. In particular, label the Highlights
`Meta+Enter` binding as custom vault configuration, not a registered plugin default. Do
not open the private vault to discover bindings; use tracked source, tests, and the
README's documented configuration.

Link the behavior contracts as absolute site URLs, particularly `/plan/`, `/freshness/`,
`/task-dependencies/`, `/task-move/`, `/task-status-hooks/`, `/projects/`, `/articles/`,
`/ref/`, and relevant mark/display contracts. Keep exact command facts generated and
user behavior in readable sections. Shared developer API material will move into Develop
in the next phase.

Record a compact coverage mapping of source README rows/paragraph ranges to their new
pages in the phase note. If a detail is obsolete, verify against source/tests and
explain the correction in that mapping instead of silently dropping it.

Acceptance: all seven pages are complete and within budgets; registered/default versus
handler/custom bindings are distinguishable; `docs:inventory:check`, `npm test`, and
`npm run validate` pass; sync is recorded.

## Phase: topic-migration

Work in bob-plugins after `plugin-pages`. Move the rest of the README before shortening
it:

| README material                                           | Destination                            |
| --------------------------------------------------------- | -------------------------------------- |
| Block ID prompt defaults                                  | `docs/block-id-defaults.md`            |
| BLOCKS dependency mirrors                                 | `docs/blocks-mirrors.md`               |
| Actual Task Card interaction reference                    | `docs/task-card.md`                    |
| Go to task URI, grammar, re-find order, notices           | `docs/goto-task-uri.md`                |
| Open notes URI, grammar, re-find behavior, notices        | `docs/open-notes-uri.md`               |
| Open reference PDF in Highlights                          | `docs/highlights.md`                   |
| Templater new-note trigger                                | `docs/templater-trigger.md`            |
| Development model, generated sources, layout              | `docs/develop/architecture.md`         |
| Validation and test guidance                              | `docs/develop/validation.md`           |
| Versioned cross-plugin API details buried in plugin prose | `docs/develop/plugin-apis.md`          |
| Depends-On line migration                                 | `docs/develop/dependency-migration.md` |
| Deploying and distribution scope                          | `docs/develop/deployment.md`           |

Create `docs/develop/index.md` as a concise contributor hub. Preserve URI encoding,
parameters, failure notices, read-only semantics, keyboard caveats, API shapes, and
deployment safety details. Replace obsolete/private examples with synthetic ones. Do not
flatten every Task Card subsection into that page: Ledger Tools snippets, budgets, and
plugin APIs belong to the destinations above.

Create `docs/nav.yml` and `docs/develop/nav.yml`, flat ordered `{title, path}` lists for
their respective mounts, excluding each index and excluding developer pages from the
parent nav. Link the plugin pages to the new topics and developer APIs. There must be no
temporary broken topic links when this phase lands.

Audit **every** original README section and plugin table row against a destination,
including unheaded paragraphs. Record the completed mapping in the phase note and commit
description where supported by finalization. Then replace README with a useful front
door: one-sentence purpose, generated minimum requirements, install snippet using
`bob plugins sync`, brief prerequisites, docs links for Get started, Reference, and
Develop, source navigation links that work without a live site, and MIT license.
Preserve the generator's requirements markers.

Verify documented commands with temporary HOME/XDG/BOB_DIR and a synthetic fixture where
they mutate data. GUI URI activation remains a documented manual step; do not claim it
was run on this server. Inspect `bob plugins sync --help` and source for the correct
local source override before describing local developer deployment.

Acceptance: migration coverage is recorded; no important detail remains orphaned; all
generated checks and plugin tests/validation pass, and sync is recorded. The site
integration phase runs the full mounted docs gates.

## Phase: site-integration

Work in both repositories after `mount-hubs` and `topic-migration`.

1. In bob-cli append two entries to `docs/_meta/mounts.yml`, preserving mounts added by
   other phases. Both use repo `bob-plugins`, remote
   `https://github.com/bobs-org/bob-plugins`, and branch `master`: `source: docs` to
   `target: reference/plugins`, and `source: docs/develop` to `target: develop/plugins`.
   Use sensible titles `Obsidian plugins` and `Plugin development`. Add exactly one nav
   placeholder per mount under the existing Reference and Develop tabs.
2. Remove only the two plugin `index.md` negations from `.gitignore`, so all copied
   plugin pages are ignored. Do not commit mount copies or duplicate the generated
   inventory into bob-cli. Retarget `docs/reference/index.md`, the homepage's Obsidian
   card, and the Develop hub to the mounted pages.
3. Wire the plugin generator into the existing `just docs-gen` / generated-drift
   workflow when the plugin mount is configured. Use the already resolved source
   directory in docs-check, especially production's explicitly pinned sources; never
   rediscover from environment or cache during a production check. Normal generation
   updates bob-plugins' canonical files, not mount copies. `--check` never changes
   tracked source. Missing Node or an invalid generator fails clearly rather than
   silently skipping a required gate.
4. In bob-plugins add `npm run docs:check` invoking `scripts/docs-check.sh`. Resolve
   bob-cli from `BOB_CLI_DIR`, then `../bob-cli`, then an explicitly managed cache clone
   of bob-cli origin master. Validate the chosen checkout, preserve user-owned trees,
   quote paths, print useful failures, and propagate nonzero status. Run its
   `just docs-check` with `BOB_DOCS_BOB_PLUGINS_DIR` set to the canonical plugin
   checkout. The script must not call itself through npm test or recurse through the
   site gate.
5. During SASE work use `/sase_repo` for any repository inspection and an explicit
   `BOB_CLI_DIR` pointing at the active bob-cli checkout. Exercise cache discovery
   through temporary fixture repositories and stubbed commands, rather than reading
   unmanaged checkouts. Add focused wrapper tests for precedence, spaces in paths, error
   propagation, and the exact source override forwarded to `just`.
6. Add the requested bob-plugins `AGENTS.md` sentence: its docs are mounted into the Bob
   site, inventory must be generated, and contributors run `npm run docs:check`.
   Document the regeneration/preview commands in its developer pages.
7. Extend `docs/_meta/search_smoke.yml` with `install plugins` targeting a real
   install/deployment section and a verified command name such as
   `Yank absolute path with tilde`. Use the actual rendered search location.

Add focused site tests proving the canonical source generator is used by both generation
and check, a stale plugin output fails the gate, check mode does not write, and
production checks cannot fall back to another checkout. Avoid tests that pin handbook
prose; test the workflow and its failures.

Final acceptance:

- In bob-plugins: generation is deterministic; `npm test && npm run validate` pass;
  `BOB_CLI_DIR=<active bob-cli checkout> npm run docs:check` passes; sync succeeds or
  its exact environmental blocker is recorded.
- In bob-cli: `just docs-gen` followed by `just docs-check` passes with both plugin
  mounts, without disturbing other mounts. `just docs-build-material` also succeeds.
- All seven inventories match manifests and all current registrations are covered.
  Source hubs appear once, developer pages are not duplicated under Reference, and Edit
  this page targets bob-plugins for mounted pages including hubs.
- README links resolve; search finds the added cases; generated copies stay ignored;
  both repositories have only intended tracked changes.
- Record commands, results, source SHAs, migration coverage, and any remaining
  limitations on the assigned phase for the child land agent.

No Rust changes are planned. Run `just check` if Rust is changed to resolve a real
integration issue; do not change Rust merely to make documentation tests easier.

## Completion and follow-up ownership

Workers close only their assigned child phase after `sase bead epic-symbols` shows no
leftovers (or each entry has been resolved/re-keyed to a still-open owner). They never
close this child epic, `bob-cli-a0.10`, or `bob-cli-a0`.

The child epic's land agent verifies all phase work together, checks any changes that
landed meanwhile, and supplies completion evidence for `bob-cli-a0.10`. After all
required descendants are complete, that land agent closes `bob-cli-a0.10` with
`sase bead close bob-cli-a0.10 --note "<verified results>"` as authorized by this
assignment, checking its epic symbols first. It must not close the outer epic
`bob-cli-a0`.

Do not create task beads. Record discovered out-of-scope work using
`sase bead note <assigned-phase> 'PROPOSED FOLLOW-UP: <summary — evidence>'` and carry
those notes to `bob-cli-a0.10` for the outer land agent. A check failure reproduced
identically on a clean base is recorded this way with any existing task reference and
does not leave an otherwise completed phase open. A regression from this work must be
fixed before completion.
