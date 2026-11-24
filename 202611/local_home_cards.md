---
tier: epic
title: Local Area and Project cards
goal:
  Complete bob-cli-pl.4 with locally painted Area and Project title cards, native
  Properties still visible, unchanged project-task interaction, and verified
  real-Obsidian behavior.
phases:
  - id: presentation
    title: Local home-card model, species, and presentation
    size: medium
    depends_on: []
    description:
      "presentation: implement the pure local Area/Project card model, field and pennant
      species, DOM presentation, and deterministic model/DOM tests described below. Wire
      source fragments and exports, but leave production mounting to mount-foot."
  - id: mount-foot
    title: Home-card host, title adoption, foot, and lifecycle integration
    size: medium
    depends_on:
      - presentation
    description:
      "mount-foot: mount the presentation in desktop Live Preview and Reading through
      the shared host and title kit, integrate the property foot and local invalidation,
      and test cleanup, exclusions, collapse, truthful inventory marks, and no writes or
      card-triggered subprocesses. Keep native Properties visible."
  - id: acceptance
    title: Real-Obsidian acceptance, screenshots, and card documentation
    size: medium
    depends_on:
      - mount-foot
    description:
      "acceptance: add the disposable-vault home-card e2e and offline wrapper, run its
      real Obsidian matrix and inspect screenshots, fix phase regressions, document the
      cards, and complete plugin build, test, docs, and deployment checks."
proposed_by: bbugyi200.apollo.bob-cli-pl.4--1
proposed_with: codex/gpt-6-astra
parent_bead: bob-cli-pl.4
create_time: 2026-11-24 09:51:06
status: wip
---

- **PROMPT:**
  [prompts/202611/local_home_cards.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/local_home_cards.md)

# Local Area and Project cards for bob-cli-pl.4

Deliver the `home-cards` phase of `plan:202611/area_project_cards.md`. Eligible area and
project notes gain a compact title card built entirely from local data. The card owns
the visible title, preserves the native Properties block, and leaves the project's
`^prj` task as its existing editable control below the card.

This is a child epic of assigned phase **bob-cli-pl.4**, not a replacement for the
surrounding `bob-cli-pl` epic. Three sequential medium phases separate the pure
presentation, the stateful mount integration, and real-app acceptance. Each can be
implemented directly by one coding agent. No earlier implementation of this phase needs
recovery: its only previous note records waiting for `bob-cli-pf`.

## Prerequisite, decisions, and scope

The launch prerequisite was checked with an audited read on 2026-11-24: **bob-cli-pf is
closed with resolution `done`**, closed at 14:32:16 UTC. Its landing note records
shared-foot, editor, and native-ownership acceptance, with bob-plugins at `b1413ba9`.
Each worker must re-read its assigned bead and `bob-cli-pf`, then use `/sase_repo` to
open bob-plugins and read its `AGENTS.md`. Inspect current code before editing, since
sibling phases may have landed.

Read `plan:202611/area_project_cards.md` through `sase artifact read` for the governing
design and any needed rationale. Read bob-cli `docs/props.md`, the linked repo's
`docs/property-editor.md` and `docs/develop/architecture.md`, and
`glossary:property-card` through `/sase_memory_read`. The design below repeats the
local-card behavior so implementation does not depend on this conversation.

The parent decisions are final:

- `card_structure_counts=no`: no sub-area, project, or sub-project counts.
- The three authorized glossary changes belong to the parent's later `takeover` phase.
  **This child epic edits no memory**, including those three strands.

The landed foundation uses bare **`-`** for the Properties drawer and **`+`** for the
Property Editor. `==` is still the species-editor dispatcher. Use those current bindings
in code, tests, and docs; do not restore the older `=-`/`=+` bindings referenced in the
parent design.

All product code lives in bob-plugins `plugins/bob-ledger-tools`; tests and docs live in
that repository's existing trees. No Rust or vault migration is needed. Do not implement
report-backed blocked reasons, open/today counts, report lint, goal-line styling,
settings groups/stages, `==` registration for these species, bank arbitration, or native
Properties takeover. Those belong to parent phases `card-live`, `editor-settings`, and
`takeover`. Do not add a YAML writer or any `processFrontMatter` call. Do not copy the
contents of a live vault into fixtures.

## Landed integration points

Paths in this section are relative to the opened bob-plugins repository.

- `src/240f-plugin-property-card-species.js`: `PROPERTY_CARD_KINDS`,
  `PROPERTY_CARD_SPECIES`, identity attributes and accessible names.
- `240g` title DOM, `240h` source ranges, `240i` ownership, `240k` adoption, and
  `240c`/`240d` host: `createPropertyCardHost`, `propertyCardBuildTitleBlock`,
  `propertyCardAdoptTitle`, `propertyCardAdoptPatch`, and path-bound release. Live
  Preview adoption is deferred to a microtask; DOM mutations/dispatch must not run
  inside CM updates.
- `241u`/`241v`/`241va` album header: local identity, modifier-aware links,
  source-cache/foot lifecycle, and in-place title-slot refresh examples. Do not copy
  album report subscriptions, embed rendering, or frontmatter takeover.
- `241e` notebook model: ancestry traversal and `notebookHeaderCollapse` are useful, but
  the generic note-title helper accepts late H1s and is unsuitable unchanged for the
  home card. Reuse traversal primitives with the rules below.
- `240m`/`240n`/`240o`/`240p`/`240r`: created formatter, source-backed inventory, foot
  DOM, per-note drawer state, and shared adapter helpers. The current missing stamp
  defaults to `no created date` for non-temporal cards; area needs a narrow species
  exception. The current managed-key predicate only knows generated digest/drafted/Zorg
  fields; home-specific grouping needs an additive seam or adapter projection, not a
  global reclassification of ordinary notes.
- `api.noteReady.version=1`: `snapshot()` and `forNote(path)` are synchronous. Rows
  expose `count`, `cap`, `cap_source`, and `state` (empty/room/full/crowded/ exempt);
  `make_up.ready` is not the lane count. Use availability from snapshot. The
  implementation is `190-plugin-note-ready.js` and `330-note-ready-model.js`.
- `pomodoroHomeResolve` supplies effective home hue. The navigation plugin's
  `api.inboxRoute.isInboxNote(path)` supplies the inbox classification. Read APIs
  defensively and never write or launch bob to obtain a missing answer.
- The record sheet's `recordSheetCollapsed` Map is the existing per-note session fold
  pattern. Whole-card collapse and the foot drawer are independent state.
- Add mixins to `310-install-methods.js`, setup/disposal to `170-plugin-lifecycle.js`
  and its teardown counterpart, fragments in `src/fragments.json`, and test exports
  alongside the existing `374*` exports.

Use new `249a`–`249d` home-card fragments if those names remain free; split a
host/editor fragment further rather than exceeding the 1000-line limit. The model, DOM
builder, host, and foot adapter remain separate. Source order comes from the manifest,
not filename sorting. Keep shared-file edits additive.

## Required local presentation

### Identity, title, and ancestry

Use the existing area/project type classifiers. Exclude `area.md`, `project.md`,
`bank.md`, `_templates/`, `done/`, and `_conflicts/`. Only typed home notes qualify; no
filename heuristic upgrades an untyped note. Handle malformed or ambiguous metadata
conservatively and preserve native chrome when a safe model cannot be built. The same
eligibility decision must drive both mount surfaces.

Add Area (`word: Area`, `icon: layers`, tile `field`) and Project (`word: Project`,
`icon: target`, tile `pennant`) to the species registry. CSS uppercases the words; icons
are monochrome and aria-hidden, and accessible names start `Area:` or `Project:`. The
field is a soft rounded hue swatch, and the pennant has a shallow swallowtail trailing
notch. Terminal projects use its outline. Keep all seven existing species unchanged.
Inspect the silhouettes in screenshots; only if necessary use the parent's allowed
fallback icons `compass`/`flag` and record why.

Title order is the eligible first-body H1 before any bullet or H2, then the first
nonblank supported alias, then **stem verbatim**, including underscores and case. Never
use a `title:`/`emoji:` setting or the project's criterion as the title. Use
source/cache positions to reject H1s following `^prj`, another bullet, H2, or a fenced
example. Adopt only the kit-recognized title range; reveal it on Live Preview
cursor/selection contact. Its range must never consume `^prj` or other task text. If
safe title ownership fails, restore the native title/H1.

The kicker contains AREA/PROJECT and clickable parent ancestry from actual local links.
Omit the `area` root sentinel and self, show at most four crumb slots with a middle
ellipsis, and put a 6 px hue pip on the top-level ancestor. Walk with a bounded cycle
guard and never invent a link for unresolved ancestry. Use modifier-aware navigation
(`albumHeaderLinkPane`): Mod opens a tab, Mod+Alt a split, and middle-click a tab.
Include a useful full-path tooltip. `INBOX` comes from the inbox route API, including
`inbox.md`; feature-detect the API rather than maintain a second list. `ROOT` appears
only for project `root: true`, with tooltip "Pomodoro time here reports under this
project".

### Subtitle and lifecycle

Area subtitle is its authored `summary`, clamped to two lines with full tooltip; absence
removes that row. Areas use about 8% hue wash, projects 5%, and terminal projects 2%
plus muted ink. Grey/unresolved hue uses neutral paper.

For projects, inspect the recognized `^prj` source task, excluding frontmatter, fences,
nested examples, and ambiguous duplicates. Reuse safe scan/date helpers where they fit;
`projectsLifecycleFingerprint` alone is insufficient because it does not include inline
dates or title-range boundaries. Keep source offsets for tests, without rendering or
relocating the source task. Local state is:

| Source facts                                  | Lifecycle subtitle                                              |
| --------------------------------------------- | --------------------------------------------------------------- |
| `[ ]`, `[*]`, `[/]`                           | WIP                                                             |
| `[?]` and future inline `[scheduled::]`       | SCHEDULED · starts Dec 2 · in 10 days (using the actual date)   |
| `[?]` without future schedule                 | BLOCKED                                                         |
| `[x]` / `[-]`                                 | DONE / CANCELED, adding the line's completion date when present |
| No task, terminal `status: done` / `canceled` | Preserve that terminal identity                                 |
| Active project with no recognized task        | Amber NO ^prj TASK with the first-line repair instruction       |

Use Lucide glyphs plus words. The scheduled decision uses the local calendar date, with
injected time for unit tests, and refreshes at the existing date invalidation boundary.
Do not briefly show WIP while source bytes are pending: defer unknown lifecycle content
until the local source is available. Frontmatter `scheduled` is legacy inventory, never
this line's schedule. No dependency reason, inferred ancestor state, or fallback close
date is fabricated; parent `card-live` adds those from reports. A terminal project has
no chip/readout row.

### Authored chips and Ready

Show only explicitly authored keys from confirmed source bytes, in this order:

1. `task_type`, or legacy `theme` when not superseded: `#BUILD` using the existing type
   hue resolver, or `type off`. Do not migrate legacy keys when painting.
2. `offclock_types` wins over `offclock`; render `off-clock · MEET, CALL`, `off-clock`,
   or explicit `on-clock`. Both keys add the shadowing warning and tooltip explaining
   the actual ignored value and the Settings cleanup route.
3. `task_refresh`: `review 3d` using the actual authored value.
4. `project_name_aliases`: `aka tailnet` plus `+N` for additional names.

Do not invent inherited/default chips or silently coerce an invalid value into a valid
policy. Preserve false, zero, empty, and absent distinctions; leave unsupported source
shapes visible in the drawer with the kit's recovery path.

The noninteractive right readout uses `noteReady.forNote(path).count` and cap:
`2/5 ready`, or `14 ready · no cap` for the exempt state. Reuse existing
room/full/crowded ink and tabular numerals. Tooltip states residence scope and the
actual source, e.g. "Ready tasks living in this note: 2 of 5 · cap from
plan.max_ready_per_note". Missing/unavailable local Tasks data is unknown, never a
fabricated zero: omit the count until available. Do not count locally again, use
freshness-only Ready totals, or introduce open/today/structure counts.

`⊞ Settings` and policy chips use one feature-detected adapter callback to the landed
`openPropertyEditor({ filePath, key?, mode: 'browse' })`. Select the corresponding
authored key (including legacy theme or the deciding clock key). Settings opens browse
without pretending a not-yet-landed settings group exists. When the API is absent,
render plain inert text, not a disabled control that claims an action. Preserve a small
callback seam for parent `editor-settings` to route by setting id later. Do not register
`==` or author a second editor.

### Foot and collapse

Append the shared foot after content, outside the collapsible body. Reuse the lazy
inventory, disclosures, editor hook, links, source-line recovery, created formatting,
and per-path drawer memory. An area missing `created` is silent; a project missing it
says `no created date`. Invalid stamps retain kit lint.

Declare where-marks only for fields actually represented by this model: `type`→tile,
`parent`→kicker, `summary`→subtitle, `color`→tint, authored policy keys→chip,
`ready_cap`→readout, `created`→foot; projects add `root`→kicker and `status`→state. Do
not claim a hidden terminal policy, unresolved crumb, pending readout, or ignored
project summary is painted. Group home `done_tasks`, `template`, legacy frontmatter
`scheduled`, and leftover `task_count` / `open_task_count` in the managed fold.
`done_tasks` is a working archive link; retired counts remain inspectable/deletable
through the generic editor, not newly protected. Limit any new grouping option to
explicit adapter callers.

Collapsed cards keep kicker, title, project lifecycle, and foot; area summary and the
entire chip/readout row disappear. Use the record disclosure interaction and a
per-normalized-path session Map, shared across panes, independent of drawer state. Keep
focus stable and respect reduced motion. Use the parent's compact anatomy (kicker,
title, subtitle, optional row, foot); its explicitly allowed two-line summary can wrap
within the subtitle slot. Avoid extra rows, clipped narrow controls, and horizontal
scrolling.

## Phase presentation

Implement species, pure models, and the DOM builder with injected callbacks,
frontmatter/source/ancestry inputs, Ready snapshots, and clock. Build accessible DOM
with existing kit builders and CSS tokens. Do not mount a new production card until the
next phase supplies teardown and fail-open behavior.

Add explicit unit suites for eligibility/title/ancestry/lifecycle/chips/Ready and
presentation. Cover top-level and deep/cyclic/broken ancestry, aliases and underscored
stems, an H1 after `^prj`, every lifecycle row, future/today/past schedule boundaries,
false clock/default-type-off/legacy theme, chip ordering, unknown Ready data, cap
provenance and ink, terminal state, missing stamps, compact collapse composition, and
reduced motion. Reuse DOM fakes and exports; do not duplicate the production parser in
tests. Add every test to `test:unit`.

Acceptance: focused tests and the plugin gates below pass; production mounts are
unchanged, and the model/DOM contract is ready for the next phase. Record the exported
model/builder signatures in the phase note.

## Phase mount-foot

Implement one shared home adapter and its foot mixin, installed once. Use
`createPropertyCardHost` with title ownership, explicit desktop gates, and
`allowEmbedDisplay: false`. Register lifecycle setup/teardown alongside existing card
hosts. Use metadata and confirmed source caches, invalidating by path and guarding
asynchronous flights against stale generations or navigation. Read the live CM document
where appropriate; do not treat stale cache as a confirmed postimage. Observe the
existing local Tasks/query-refresh and metadata events for Ready, ancestor, hue, date,
and body changes; coalesce refreshes.

Never subscribe in a way that starts a per-card projects/areas report request. Do not
call `apiAlbumSubscribe`, projects/areas refresh/sync, capability probes, or any bob
process from the paint, resize, fold, or drawer path. Later report patches can use the
exported model and `propertyCardAdoptPatch` seam.

Adopt title slots and patch mounted nodes in place. Preserve path/generation identity,
split-pane state, editor selection, disclosure focus, and unique per-mount DOM IDs.
Failed builder/adoption, path change, Source toggle, and unload release the title/H1 in
the same frame, without global title/CSS effects. Denied contexts (embed, popover,
canvas, print, mobile, Source) never show this card. Do not copy the album's fallback
embed display behavior. No call to
`propertyCardFrontmatterOwnMounted`/`propertyCardOwnFrontmatter`, no whole-block or
per-key native Properties hiding: native Properties remains wholly visible.

Integrate the foot behavior above. Test injected mount/build failures, lost title slots,
mismatched path/source, rename/delete/navigation, split panes, teardown and remount,
eligibility changes, forbidden contexts, collapse state, where-marks/managed fold,
links, and absent editor API. Spy on writes/spawns around painting, resize, drawer
toggles, and folds. Assert title restoration and untouched `^prj` offsets, not just a
successful return value.

Acceptance: all eligible desktop surfaces mount and update locally; native Properties
stays visible; new unit suites and regression gates pass. Record host/foot entry points
for `card-live` and `editor-settings` on bob-cli-pl.4.

## Phase acceptance

Add `scripts/obsidian-home-cards-e2e.cjs` with an offline `--self-test`, fixture helpers
as needed, and `scripts/test-obsidian-home-cards-e2e.cjs` in `test:unit`. Follow the
existing card-foot/title-block runners: isolated disposable Git vault, owned Obsidian
profile/process and CDP port, deterministic fixture time, explicit binary provenance,
JSON assertions, captured exceptions, and screenshot output. Never open or mutate
Bryan's personal vault to prove card behavior.

Fixture matrix: top-level area, sub-area, inbox, open project, blocked project,
future-scheduled project, done and canceled projects (including archived `^prj`),
`root: true`, legacy theme, and active missing-`^prj`. Include deep ancestry, explicit
false clock, shadowed clock, cap off, and missing/invalid created. Exercise Live Preview
and Reading, body collapse and `-` drawer, Settings/chip handoff, links with modifiers,
narrow/wide widths, reduced motion, and the denied contexts. Inject a mount failure and
verify native title/H1 restoration and native Properties visibility. Check existing
species for integration regressions.

Prove actual cursor/selection behavior: a project whose first body line is `^prj` still
edits that task at that position; an H1 after it is still visible. Open its existing
Task Card, cancel through existing close keys, then exercise an existing task close
gesture on a separate disposable fixture. Verify the expected task result and that the
card never rewrites or relocates the goal. Separate these intentional interaction writes
from zero-write render assertions.

Hash fixture bytes and record subprocess calls before/after passive operations. After
app startup services settle, navigation/render/fold/resize must introduce zero home-card
bob calls and zero note writes; distinguish unrelated existing service traffic by call
provenance rather than hiding it. Capture and inspect light/dark × narrow/wide
screenshots, including nonterminal/terminal forms, long summary, collapsed state, and
open drawer. Offline self-test alone is not real-Obsidian acceptance. Fix failures
caused by this work and rerun affected acceptance. Retain reports/screenshots as audited
artifacts using `/sase_artifact`.

Add `docs/area-and-project-cards.md`: anatomy, title/lifecycle rules, authored chips,
local data/unknown states, collapse, foot, shortcut handoffs, exclusions, and visible
native Properties. Link it in `docs/nav.yml`, `docs/albums.md`, and the species list in
`docs/develop/architecture.md`. Document the runner in `docs/develop/validation.md`. Use
current bare `-`/`+` bindings. Read `docs.md` through `/sase_memory_read` first. Do not
claim later-phase features are shipped.

Acceptance: the real-app report and inspected screenshots prove the complete home-cards
phase, all required gates pass or have clean-base evidence, and the deployment is
verified. Leave enough evidence for the child land agent to close the assigned outer
phase without another implementation pass.

## Verification, deployment, and landing

For each phase that changes the plugin repository:

1. `npm run build`; never hand-edit generated `main.js`.
2. Run focused unit suites for the changed behavior; include identity/title/ host/foot
   regressions when shared kit code changes.
3. `npm run docs:gen` if inventory changes, then `npm test` and `npm run validate`.
4. For docs changes run `BOB_CLI_DIR=<current bob-cli checkout> npm run docs:check`, so
   the gate uses the approved workspace instead of cloning another repository.
5. `bob plugins sync`, obeying the opened repo's overwrite-refusal workflow; never force
   another checkout's deploy. Verify the plugin is synced.

Use `/sase_monitor` before starting long suites or real-app runs, with a concrete
continuation; finish the staging command and finalize the turn. Follow current
validation docs and the `tailnet` memory if athena is needed for Obsidian. Open
repositories only via `/sase_repo`; transport test inputs using the supported
isolated-run workflow, with explicit source/binary provenance. Record commands,
versions, source SHA, results, and inspected screenshot refs.

Any unrelated failure reproduced identically on the clean base is recorded as
`PROPOSED FOLLOW-UP:` via `sase bead note bob-cli-pl.4`, with existing tracking ID when
known. It does not keep this phase open. Workers do not create task beads, hand-set
reservation/status, or reopen bob-cli-pl.4. Resolve epic-symbol entries for each
assigned child before closing that child.

The child epic's land agent checks all three child results and verifies the complete
original `home-cards` acceptance. Before completing **bob-cli-pl.4**, run
`sase bead epic-symbols bob-cli-pl.4`; resolve each symbol or re-key its Justfile line
to an appropriate still-open bead. Record the API/keymap differences and verification
evidence on bob-cli-pl.4, then close it with
`sase bead close bob-cli-pl.4 --note "<verified results>"` after the child epic's normal
completion bookkeeping is satisfied. Child phase workers close only their own assigned
child. **Never close bob-cli-pl or any ancestor above bob-cli-pl.4.** Do not close
bob-cli-pl.4 merely because this plan was approved.
