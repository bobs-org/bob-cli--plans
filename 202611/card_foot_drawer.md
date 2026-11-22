---
tier: epic
title: Card foot row, properties drawer, and equals-minus entry points
goal:
  Every property card exposes its complete frontmatter and created stamp through a
  read-only foot and drawer, with preserved card editing and verified real-Obsidian
  behavior, completing bob-cli-pf.1.
phases:
  - id: models
    title: Shared created stamp and source-ordered property inventory
    size: medium
    depends_on: []
    description:
      "models: build and test the pure created formatter and complete inventory model,
      including source order, duplicate diagnostics, grouping, and typed empty values."
  - id: drawer-record
    title: Shared foot runtime and Record Card migration
    size: medium
    depends_on:
      - models
    description:
      "drawer-record: build the lazy accessible foot and drawer, per-note session state
      and mount lifecycle, then replace the record-only fold and created footer in
      Reading and Live Preview."
  - id: adapters
    title: Album, temporal, and bank foot adoption
    size: medium
    depends_on:
      - drawer-record
    description:
      "adapters: adopt the shared foot after each rail on album, day, week, quarter,
      year, and bank setup cards, including identity-only and collapsed states, with
      focused tests and species documentation."
  - id: entrypoints
    title: Equals arm table and drawer commands
    size: medium
    depends_on:
      - adapters
    description:
      "entrypoints: generalize the existing equals gate, add the mounted-view minus arm
      and palette and Command Mode toggles, retain equals behavior, and publish the
      tested kit integration contract."
  - id: acceptance
    title: Real-Obsidian drawer acceptance and deployment
    size: medium
    depends_on:
      - entrypoints
    description:
      "acceptance: add and run the isolated real-Obsidian foot walkthrough, inspect the
      complete screenshot matrix, verify no-write and no-spawn behavior, run repository
      gates, and deploy the verified plugins."
proposed_by: bbugyi200.apollo.bob-cli-pf.1
proposed_with: codex/gpt-6-astra
parent_bead: bob-cli-pf.1
create_time: 2026-11-22 14:45:05
status: wip
---

- **PROMPT:**
  [prompts/202611/card_foot_drawer.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/card_foot_drawer.md)
- **PARENT:**
  [202611/card_owned_frontmatter.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/card_owned_frontmatter.md)

# Card foot row and properties drawer

Implement the complete `foot-drawer` phase assigned to `bob-cli-pf.1` as this child
epic. Its source design is `plan:202611/card_owned_frontmatter.md`, especially
experience sections 1–4 and 7 and phase `foot-drawer`. Read that artifact through
`sase artifact read` for context; this plan supplies the implementation boundaries and
acceptance requirements. This does not implement the parent epic's property writer, `=+`
editor, or whole-container Properties takeover.

## Existing work and scope

At inspection, `bob-cli-pf.1` is in progress and has no prior implementation notes. The
opened bob-plugins checkout at `2b3b6b60` still has the old record-only fold,
`recordSheetFolds`, `recordSheetCreatedFooter`, and equals-only gate. The new shared
foot, drawer, and command are not present. Recheck landed work and bead notes before
implementing each phase; retain any equivalent work that lands in the meantime.

The phase spans pure models, a shared mount lifecycle, four adapter families, Vim input
dispatch, documentation, and real-application verification. Five serial, medium phases
keep each coding assignment bounded. The dependencies also avoid concurrent changes to
bundle manifests, lifecycle wiring, CSS, and the explicit test list. The child epic's
land agent owns final integration and completion evidence.

All implementation is in the linked **bob-plugins** repository. Open it with
`/sase_repo`, use only the returned checkout, and read its `AGENTS.md`. Paths below are
relative to bob-plugins unless expressly identified otherwise. Edit fragments under
`plugins/*/src/`; generated `main.js` files come only from `npm run build`. Keep each
hand-edited fragment at or below 1000 lines. Register new tests in `package.json`'s
explicit `test:unit` list and test helpers in the existing export fragments. Do not edit
the deployed vault copies.

Approved parent decisions are final:

- `bank_foot = yes`: bank.md's setup header gets the foot, drawer, and stamp; native
  bank Properties keep their existing visibility and `bank` row ownership.
- `drawer_memory = note`: session-only state is keyed by normalized note path and shared
  by mounts of that note. A different note starts closed.
- Glossary corrections are authorized only for Property Card and Record Card, but belong
  to the parent's takeover phase. This child epic edits no memory and adds no decision
  record.

Preserve existing native Properties hiding by key, title ownership, report caching, and
card editing. There is no new whole-container hiding or `props_v1` dependency here.
Painting and toggling are read-only: no bob subprocess and no vault write.

## Shared implementation contract

Create kit fragments near `240m`–`240p` for created formatting, inventory, foot DOM, and
runtime state; split further if required by the fragment line limit. Use `374*` exports
for tests and maintain source manifest order. Another parent phase owns the prospective
`240q` frontmatter-ownership primitive; avoid that namespace.

The inventory accepts confirmed source text, metadata-cache frontmatter, and adapter
declarations `{ where: {key: mark}, nested: {key: opener} }`. Source determines
top-level membership, order, and key-line locations; the cache supplies values. Do not
use object enumeration as authoritative order when source exists: it sorts integer-like
keys. Cache order is only the fallback while source is unavailable. Exclude Obsidian's
cache bookkeeping fields. Distinguish pending source/cache confirmation from genuinely
malformed YAML, and refresh after confirmation without reusing a previous file's source
or values.

Return stable occurrence identities and line locations so duplicate keys remain two rows
and `y` can visit either occurrence. Handle quoted keys, comments, nested maps, flow
values, block scalars, BOM, and CRLF without mistaking nested content for top-level
properties. Reuse Obsidian's YAML parser for validity rather than writing a second YAML
value parser. Duplicate-key diagnostics must not accidentally hide the duplicate rows
behind the generic parse-failure state. Values for duplicate occurrences must not claim
independent source values when only a cache value exists.

Group the inventory in this order, preserving source order within each group:

1. Loose keys, without a heading; retain album-declared legacy chips.
2. **On this card**, with the adapter's faint where-mark.
3. **Managed · N**, initially folded, for `*_digest`, `*_drafted`, `zorg_*`, and
   `generated_from_zorg*`. An explicit where-mark wins, including `summary_drafted` →
   agent. This display grouping is not the future writer role table and must not enable
   writes.

Invalid or unclosed frontmatter shows a recovery line and no value rows: “Frontmatter
doesn't parse — y opens YAML”. The lint chip explains invalid created values, parse
failures, unclosed fences, and duplicate keys without opening the drawer. The disclosure
count reflects the complete top-level inventory, including all groups and duplicate
occurrences.

The shared runtime must expose `propertyCardToggleProperties(view)` returning whether it
acted. Resolve a live, path-matching mount in the supplied view; never toggle a hidden
or stale mount left by a previous file, another pane, or an embed. Per-path open state
survives refresh, card-body collapse, and revisiting the note, but does not persist
through plugin reload. Release DOM registrations/listeners on replacement, path change,
removal, failed paint, and unload. Rename/delete events must not leak old-path state.
Keep per-path state separate from per-mount disclosure IDs, which must be unique even
for two panes showing one note.

Expose the future editor hook `onOpenEditor(path, {key, mode})`, unset here. Key labels
and Enter invoke it only when registered; value links continue navigating. Do not
advertise Add property or Edit all before the parent's editor-core registers the hook.
Nested album `record` and hub `bank` values use their existing species editor opener
immediately. This phase adds no JS YAML writer or processFrontMatter call.

## Phase models

Add the pure created formatter and inventory model with unit exports and tests. Inject
the clock and parser where needed for deterministic tests. Keep the helpers independent
of DOM, subprocesses, and vault access; no card adoption in this phase.

The formatter returns `{text, tooltip, state, dateOnly}` with state
`ok | missing | invalid`. The verb is `added` for records and `created` otherwise.

| Input/context                             | Display                                                       |
| ----------------------------------------- | ------------------------------------------------------------- |
| Day note, created on its represented date | created 6:43 AM                                               |
| Day/week/quarter/year, other date         | absolute date and stored time if present; no relative word    |
| Non-temporal, today or yesterday          | created today, 7:55 AM / created yesterday, 9:41 PM           |
| Non-temporal, other date this year        | created Jun 11                                                |
| Non-temporal, earlier year                | created Jun 11, 2025                                          |
| Date-only                                 | calendar date without an invented midnight time               |
| Missing on temporal card                  | no stamp                                                      |
| Missing on album/record/bank              | faint no created date                                         |
| Invalid                                   | amber created warning, with raw value retained for the drawer |

For a same-date daily date-only value, show an absolute date rather than an empty verb
or invented time. Preserve the temporal absolute-date rule, and include the year where
needed to disambiguate an earlier year. Use the record verb consistently for record
states. Tooltip text retains the raw stored timestamp and offset plus age; use confirmed
source scalar text when the cache has converted it to a Date.

Accept `T` and space separators, `-0500`, `-05:00`, `+0000`, `Z`, no offset, and valid
YAML Date objects. Render times in the viewer's local zone. Date-only strings remain
calendar dates; source syntax also disambiguates date-only cache Date objects. Reject
impossible dates instead of letting JavaScript normalize them. Never infer created from
file times, git history, or the date represented by a note.

Inventory tests cover file order including numeric keys, duplicates and line locations,
managed override precedence, nested maps, parse failures/unclosed fences, and
absent/null/empty string/empty list/false/zero distinctions. Created tests cover each
table row, midnight timestamps with a space separator, all offsets, Date objects,
invalid dates, and at least two explicit time zones including a date-boundary
conversion. Reuse established typed value/link helpers where possible.

Verify the focused model tests, `npm run build:check`, and repository-required checks.
After changes run `bob plugins sync` per repository instructions. Provide the final
helper shapes and test evidence in the phase note for the runtime worker.

## Phase drawer-record

Implement foot DOM and runtime state against the preceding model contract, then migrate
only the Record Card. Relevant existing code is in `240c/240d` shared hosts, `242i`
sheet model, `242j` sheet DOM, `242k` sheet host, `242ka/242kb` properties/title, and
`242v` Reading mounting. Reuse `recordSheetBindYamlJump` and `recordSheetOpenSource`
behavior as shared helpers, preserving other callers.

The foot follows the nav rail, outside the collapsible body, including identity-only,
broken-schema, and collapsed record returns. Remove the duplicate old fold and created
footer and migrate `recordSheetFolds` to the kit's session map. Update record sheet
coverage bookkeeping and tests so removed fold/footer implementation details no longer
drive native property ownership incorrectly. Preserve schema editing, field ghosts,
links, walk positioning, title identity, and body collapse.

Declare record marks: parent→kicker, type→tile, title→title when actually covered,
schema fields/ghosts→field, created→foot. An absent ghost is not a fabricated stored
property; the drawer inventories actual top-level source keys.

Foot layout: 1.6 em line, hairline top border using the rail ink, disclosure on the
left, optional amber lint chip, muted tabular stamp on the right. Disclosure uses a real
button with `aria-expanded`, unique `aria-controls`, `Properties · N`, and a “Toggle
properties =-” tooltip. The drawer sits below it and builds its body only when opened.
Use semantic `<dl>` rows and a grid `minmax(84px, max-content) 1fr auto`; below about
420 px stack key above value. Chevron and 0fr↔1fr drawer transitions last 160 ms and
disable under reduced motion. Use existing card colors and scope CSS to card roots.

Render wikilinks with normal/Mod-click navigation and page preview, external URLs,
searchable tag chips, lists with six chips then +N, booleans as check/cross, null as
muted dash, explicit empty/empty list labels, and two-line clamped scalars with full
tooltip. Nested maps get a summary, never a YAML dump, and use a registered species
opener when supplied. Focusable rows support `y` to the exact source key line; the
parse-error teaching line supports the YAML recovery jump. Keep view navigation distinct
from vault mutation.

Source acquisition must follow the existing confirmed-source lifecycle. Use shared
bounded per-path caches/flights if adapters need them; paint synchronously from
available evidence and schedule source reads off the paint path. On source/cache changes
refresh only relevant mounts. Reuse existing render lifecycle components for hover
previews and dispose them with the mount. Opening must not trigger an album report
refresh. Multiple panes showing the same note agree on open state.

Expand scrolls an off-screen foot into view with `block: nearest`, smooth unless reduced
motion is requested; collapse never scrolls. Preserve editor selection and cursor for
keyboard/API toggles. Keep the rail's position stable by appending the foot and drawer
below it.

Test DOM semantics, all value shapes, lazy construction, malformed/duplicate views,
two-pane state, path transitions and cleanup, collapse persistence, source refresh, zero
writes/spawns, and scrolling. Update `scripts/test-ledger-tools-record-sheet.cjs` and
record-related DOM/coverage tests. Update `docs/record-card.md` for the migrated fold
and collapsed stamp. Run focused tests and required gates, build, and sync.

## Phase adapters

Adopt the completed shared foot on these additional surfaces in both Reading and Live
Preview. Do not add it to bank fence/category cards or modal editors.

| Adapter                               | Placement and marks                                                                                                                                                 |
| ------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Album header (`241u`, `241v`, `241x`) | After schema/header content and any rail/mobile hint; type→tile, parent→kicker, emoji→tile, summary→subtitle, record→schema with nested schema opener, created→foot |
| Day/week/quarter/year (`296a`–`296d`) | After existing rail; parent→kicker, type→tile, intentions→intentions, summary→summary, reflection→reflection, summary_drafted→agent, created→foot                   |
| Bank setup header (`248b`, `248c`)    | After setup/header content and outside collapse; bank→card with nested setup opener, created→foot                                                                   |

Each adapter must pass current note metadata, confirmed source, temporal context, and
only truthful rendered marks. Identity-only temporal cards still get the foot; missing
created remains quiet on temporal notes. A bank/album/record with no created gets the
faint missing state. Bank keeps native Properties behavior. Match the current platform
support of each adapter; add no desktop-only interaction to a mobile hint that cannot
safely handle it.

Avoid duplicating source reads and foot fragments during refresh. Keep property
ownership acquisition tied to successful existing mounts. Test each species,
identity-only temporal notes, record/bank collapse, multiple modes, cleanup and failed
rebuilds, and unchanged native Properties visibility. Update `docs/albums.md`,
`docs/intention-card.md`, and `docs/bank.md`, preserving published paths. Run the
affected suites, build and sync; regenerate documentation inventory when needed.

## Phase entrypoints

Generalize `248m-plugin-schema-double-equals.js` to a registered second-key arm table.
Preserve the `registerPropertyCard` API, species ordering, matching behavior, legacy
test helpers, and all existing equals tests. The `=` arm dispatches through the species
registry. The `-` arm calls `propertyCardToggleProperties(view)` only for a current
mounted foot in that view. Expose registration for future `+`; do not register a
production plus arm or implement the property editor in this child epic.

All arms share the existing clean-pending-`indentAuto` checks: normal mode, Live
Preview, empty key buffer, no count/register, no repeat/IME, no Ctrl/Alt/Meta, not
already prevented, editor focus, readable Vim state, and usable handleKey. Match
`event.key` and do not reject Shift. Consume only an actionable arm, stop propagation
once, cancel the Vim operator with `Vim.handleKey(cm, "<Esc>", "mapping")`, then execute
the arm. Preserve input fall-through for unknown arms, unmatched species, and views
without a foot.

Add `toggle-card-properties` with palette label “Toggle card properties”, and its
feature-detected `CM_SHEET_SURFACE_ROWS` entry in
`bob-command-mode/src/160-sheet-rank.js`. Palette and Command Mode use the same
current-view runtime and work in Reading/non-Vim use; no Reading prefix timer. Unload
removes arm registrations and command-related state safely.

Test old `==`, minus, and an injected test plus arm with shifted and unshifted `+`, all
rejected states above, cross-pane/path targeting, event deduplication, and no mounted
foot/unregistered plus fall-through. Include Command Mode availability and dispatch
tests. Update kit API and arm table in `docs/develop/architecture.md`,
`docs/bob-command-mode.md`, and the applicable hotkey reference found in this repo.
Document future editor hook and registration names for parent editor-core without
claiming that `=+` works yet. Run focused and full plugin tests, build, and sync.

## Phase acceptance

Add `scripts/obsidian-card-foot-e2e.cjs` and an offline `--self-test` wrapper in
`test:unit`. Reuse the isolated vault/profile/CDP runner conventions in
`scripts/obsidian-record-card-e2e.cjs`, `scripts/obsidian-intention-card-e2e.cjs`, their
helper files, and `docs/develop/validation.md`. Use an owned Obsidian process, scratch
vault, explicit bob provenance where a runner requires it, deterministic clock fixtures,
and invocation recording. Never exercise the personal vault. Consult tailnet memory
before using a remote host, and open other repos only through `/sase_repo`. Use
`/sase_monitor` for long-running verification or waits.

Required real-app evidence:

- Album, record, day, week, quarter, year, and bank setup feet in Reading and Live
  Preview, including identity-only notes and collapsed record/bank states.
- Disclosure and actual `=-` keys toggle the same drawer; normal `==` still opens the
  appropriate editor. Palette/Command Mode work without a Vim prefix.
- Real editor cursor/selection stays unchanged on `=-`; opening an off-screen foot
  scrolls it into view and collapsing does not scroll. Check reduced-motion behavior.
- `y` reaches the correct YAML line, including duplicate occurrences and invalid-YAML
  recovery. Nested schema/setup links reach existing editors; value links navigate.
- Per-note session behavior when walking to another note and returning; two panes, mode
  changes, metadata refresh, unload, and failed mounts do not leak old content.
- Source note bytes stay identical for paint/fold/toggle/collapse; the invocation log
  proves those gestures did not spawn bob. Establish baselines after fixture/report
  initialization to distinguish existing background activity from paint requests.
- Light/dark and narrow/wide screenshots of all adopted species, plus expanded,
  collapsed, duplicate/invalid, and missing-created cases. Inspect every screenshot and
  retain reports identifying build inputs, outcomes, and screenshot paths.

Add the runner invocation and coverage limits to `docs/develop/validation.md`. An
offline wrapper or screenshot preview alone is not real-app acceptance. Resolve
regressions, rerun affected tests, and then run `npm run build:check`, `npm test`,
`npm run validate`, and `npm run docs:check` with `BOB_CLI_DIR` set to the current
opened primary checkout. Run bob-cli's required checks if its tracked files changed.
Deploy with `bob plugins sync`; on a protected deployment refusal follow
`docs/develop/deployment.md` and `AGENTS.md` instead of forcing.

Retain real-app reports/screenshots through `/sase_artifact` and audited artifact
references, and attach concise verification evidence to this phase. Record unrelated
failures that reproduce identically on the clean base as `PROPOSED FOLLOW-UP:` notes on
the assigned phase, with reproduction evidence and an existing task ref if known. Do not
create new task beads or leave a finished phase open for a clean-base failure.

## Child epic landing and parent coordination

The land agent verifies every child phase and the complete acceptance matrix, checks for
stale epic-symbol allowances, resolves integration issues, and confirms the documented
APIs are ready for parent `editor-core` and `takeover`. Run
`sase bead epic-symbols bob-cli-pf.1`; remove resolved symbols or re-key justified
future symbols to a still-open owner before closing.

The user's explicit request authorizes this child epic's land agent to complete **only
`bob-cli-pf.1`** after all scope is verified, using
`sase bead close bob-cli-pf.1 --note "<verified behavior and evidence>"`. Child phase
workers close only their own assigned phase. Do not close `bob-cli-pf` or any other
ancestor. Leave `bob-cli-pf.1` in progress during planning and intermediate work; never
reopen it or hand-edit its status. Use `/sase_final` for each root turn and include
every changed linked repository in the host's declaration.
