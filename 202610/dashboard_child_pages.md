---
tier: tale
title: Dashboard child pages and grouped navigation
goal:
  Give Projects and References their own Dashboard child pages with reliable live count
  badges in a clear Work, Review, and Browse navigation layout.
size: medium
proposed_by: bbugyi200.athena.0vq
create_time: 2026-10-03 14:40:24
status: wip
---

# Dashboard child pages and grouped navigation

## Outcome and scope

Make the Dashboard a calm place to choose work, review its health, and open the two
larger collections. Projects and References become real Markdown children of `dash`,
reachable through live count badges at the top of `dash.md`. Their existing Obsidian
Bases remain the interactive views and the source notes stay where they are.

This is one medium implementation tale: a bounded addition to Bob Ledger Tools, three
vault notes, targeted tests, documentation, and deployment. One coding agent should
deliver the whole feature. No new CLI command, task status, review tier, configuration
setting, or note migration is needed.

Only this scratch plan is authored during planning. Implementation and deployment begin
after plan approval.

## Evidence and a source-location discrepancy

Read the current sources again before editing; this plan was researched on 2026-10-03
against the vault checkout at `29cff793` and Bob Ledger Tools 1.24.0.

- `dash.md` currently contains its inline DataviewJS badge renderer and the five TODAY /
  NEW / PENDING / NEXT / READY Tasks sections. Its eight badges currently form one
  wrapping strip under `## Tasks`.
- Vault commit `70ef12e3` moved the old `## Projects ([[projects.base]])` and
  `## Reading List ([[refs.base]])` embeds from `dash.md` to `type.md` on September 30.
  The inspected synced checkout consequently has no Projects or References section left
  in `dash.md`. Treat the user's request as establishing the new Dashboard destinations,
  including safely handling a newer version that has the sections back in `dash.md`; do
  not assume text is present and blindly remove it.
- `projects.md` already exists as an older index under `org`. Do not overwrite it or use
  an ambiguous `Projects` wikilink as the new destination.
- `projects.base` has a default `🚀 Active & Waiting` view. Its global filters are
  `note.type == link("project")` and `file.inFolder("_templates") == false`; the view
  filter is `status.containsAny("wip", "waiting")`.
- `refs.base` has a default `🔖 Reading Queue` view. Its global filters are
  `file.path.startsWith("ref/")` and `file.ext == "md"`; the view matches exact statuses
  `next`, `wip`, or `ready`. Its misleadingly named `📚 All Refs` view is also filtered;
  do not infer membership from view names.
- Existing shared renderers live in `bob-plugins/plugins/bob-ledger-tools/main.js`:
  `renderDashboardLaneBadge`, `renderReadyBadge`, `renderReviewChip`, and
  `noteReady.renderCrowdedChip`. The dashboard currently uses a mixture of shared
  renderers and inline fallbacks. Preserve their behavior when changing hosts.
- `docs/plan.md` documents section-count versus whole-lane-cap semantics;
  `docs/freshness.md` documents the task partition. The accepted memories
  `decisions:ready-is-freshness-gated` and `decisions:note-ready-cap-counts-the-lane`
  also record the old badge order. The user's explicit request authorizes the visual
  regrouping here, while all count, lane, freshness, cap, and review-walk rules remain
  intact. This plan does not edit SASE memory or generated instruction files.

Obsidian supports selecting an embedded Base's default view with `![[File.base#View]]`;
global and selected-view filters combine with AND. Sources:
[embedding Bases](https://obsidian.md/help/bases/create-base) and
[Bases syntax](https://obsidian.md/help/bases/syntax). Its string `containsAny` uses
substring matching, while list `containsAny` matches list elements; link equality uses
resolved file identity. Preserve these distinctions when counting. Sources:
[functions](https://obsidian.md/help/bases/functions) and the syntax page.

## Interaction and visual design

Place navigation immediately after `# Dash`, above `## Tasks`. Use three labeled rows,
in this exact reading and keyboard order:

| Group  | Badges, left to right            | Purpose                                             |
| ------ | -------------------------------- | --------------------------------------------------- |
| Work   | TODAY · PENDING · NEXT · READY   | Today's plan and the actionable lanes               |
| Review | NEW · ROTTEN · BLOCKED · CROWDED | Intake, aging work, dependencies, and crowded notes |
| Browse | PROJECTS · REFERENCES            | Open the two collection pages                       |

Schematic only; `n` is a live value, not a proposed literal count:

```text
Dash

Work     [TODAY n/cap · n/cap] [PENDING n/cap] [NEXT n/cap] [READY n/cap]
Review   [NEW n] [ROTTEN n] [BLOCKED n ↗] [CROWDED n ↗]
Browse   [PROJECTS n ↗] [REFERENCES n ↗]

Tasks
  TODAY Tasks
  NEW Tasks
  PENDING Tasks
  NEXT Tasks
  READY Tasks
```

The existing NEW/ROTTEN renderers retain any richer text they currently display. The
grouping is navigation, not a prescribed review sequence: do not change the freshness
walk or interpret the Projects collection badge as its PROJECTS tier.

Use the existing pill vocabulary: muted compact labels, tabular-number values, soft
theme-native backgrounds, and the familiar accent edge. Align the quiet group labels in
a roughly 5rem column; each row's badges form their own wrapping flex container with
about 0.4rem gaps. Keep rows close enough to read as one navigation component. No heavy
cards, oversized icons, new rainbow, animation, or totals that add unlike units. Use a
restrained shared accent for Browse; collection size is informational and never turns
red for being large or empty. Preserve existing warning colors and threshold behavior on
all task badges.

At narrow note-pane widths, put the group label above its wrapping badges; use
component/container-responsive layout so split panes work as well as phones. Badges must
not split internally or create horizontal scrolling. Scope all new CSS to the Dashboard
navigation root. Respect reduced motion, light/dark themes, visible keyboard focus, and
adequate tap targets. Do not restyle daily `bob-plan` blocks or shared badges in other
notes.

Use a navigation landmark named `Dashboard`, with labeled groups. Keep real links,
descriptive accessible names and tooltips, keyboard activation, Cmd/Ctrl opening in
another leaf, and Obsidian hover previews. New page arrows mean another note, consistent
with existing Blocked/Crowded links. Count unavailability must not disable navigation.

## Child-page and count contracts

Create these root-level files; hierarchy comes from frontmatter, as in the vault's
existing note convention, rather than from a new filesystem folder:

| File                 | Frontmatter                                                                                  | Heading and body        |
| -------------------- | -------------------------------------------------------------------------------------------- | ----------------------- | ---------------------------------------------------------------------------------------------------------- |
| `dash_projects.md`   | `parent: "[[dash]]"`; alias `Dashboard Projects`; created timestamp in existing convention   | `# Projects`, `[[dash   | ← Dashboard]]`, one short description of active/waiting projects, `![[projects.base#🚀 Active & Waiting]]` |
| `dash_references.md` | `parent: "[[dash]]"`; alias `Dashboard References`; created timestamp in existing convention | `# References`, `[[dash | ← Dashboard]]`, one short description of the reading queue, `![[refs.base#🔖 Reading Queue]]`              |

Do not give these index pages `type: [[project]]` or move/reparent any projects or
reference notes. The two Base files keep all views, filters, columns, formulas,
grouping, and sorting. Users can still switch views or open the original Base. The badge
always describes the child's **default view**, even if someone switches that embedded
table to an archive view during a visit.

- PROJECTS links to `dash_projects`; its unit is project notes in the selected Active &
  Waiting view, counted once per path. Tooltip:
  `N projects in Active & Waiting. Open Projects.` It is not open tasks, `^prj`
  trackers, due project reminders, or all historic projects.
- REFERENCES links to `dash_references`; its unit is Markdown reference notes in the
  Reading Queue, counted once per path. Tooltip:
  `N references in Reading Queue (next, wip, ready). Open References.` It is not PDFs,
  `lib/` files, all of `ref/`, or reference tasks.
- A successfully evaluated empty view is `0`; incomplete metadata, a failed source read,
  or an unrecognized view contract is `–`, with a short reason in the
  tooltip/accessibility label. One unavailable collection does not hide the other
  collection or any task badge.
- Match Base membership exactly. In particular, project type must be the link equal to
  `project.md`, using the resolver from the source note; a list or arbitrary string must
  not be silently widened into link equality. Match the actual `containsAny` semantics
  for project status, including scalar/list distinctions. References use exact,
  case-sensitive status equality. Folder matching must respect `_templates/` boundaries
  and the exact `ref/` prefix. Do not silently add task-only Today, hide, freshness,
  schedule, or dependency filters to either collection.

## Implementation steps

### 1. Reconcile current sources and prepare the scoped edits

Use `/sase_repo` to open `bob-plugins` and `gh:bobs-org/bob`; use only the paths it
returns for repository work. Read their instructions and inspect status before editing.
Read relevant vault memory through `/sase_memory_read`; the current git-sync runbook
supersedes the vault instruction file's stale Obsidian Sync description. Preserve all
unrelated user/sync changes.

Inspect the latest `dash.md`, `type.md`, the two Base files, existing destination paths,
and inbound links. Capture the original task frontmatter, Tasks blocks, and section
headings for comparison. If the destination files already exist, incorporate the change
into their current contents rather than replace them.

Move only the collection sections actually found in `dash.md`, retaining any user prose
on their appropriate child page. In the researched state, create the children from the
Base embeds already located in `type.md`, leaving `type.md` and its existing navigation
intact. Do not opportunistically clean up the type taxonomy or old `projects.md`. Check
legacy `dash#Projects`, `dash#References`, and `dash#Reading List` heading links,
including headings containing Base links; retarget actual active navigation links to the
new pages without rewriting historical transcripts or inserting duplicate tables back
into Dash.

### 2. Add one small, live collection-badge capability

In the linked plugins source, add an additive, versioned `api.dashboardCollections`
namespace to Bob Ledger Tools, following the existing `noteReady`/shared-badge patterns.
Keep the top-level API version compatible. Use a synchronous, never-throwing
`snapshot()` and `renderChip(host, { kind, sourcePath, component })` for `projects` and
`references`. Each snapshot entry carries `count` (number or null), availability,
reason, destination, and the named view. Keep pure membership/model helpers testable; do
not depend on the Tasks plugin, freshness state, or Dataview's excluded-folder index to
count the collections.

Read Markdown file identities and frontmatter from Obsidian's vault and metadata cache;
resolve type links with Obsidian's resolver. An absent frontmatter field on a cached
note is different from an unavailable file cache. Compute both counts from a shared
cached pass after metadata is ready; do not read every note body or scan separately per
badge. Pending relevant metadata yields an unavailable entry until metadata events
complete it, never a partial count presented as final.

Prevent future filter drift without building a general Bases interpreter. Parse the two
Base files using the existing Obsidian `parseYaml` dependency, and validate each named
view's global/view membership filters against the explicit small contracts above.
Compare only membership-affecting structure, including any result limit; tolerate
presentation-only changes. If filters or a view name change beyond the recognized
contract, show that collection as unavailable rather than continue advertising a stale
count. Never evaluate Base strings with JavaScript `eval`, scrape a table's DOM, or use
private Bases query APIs. Document that predicate and contract support must be extended
together for a future filter change.

Load/validate Base definitions asynchronously outside the synchronous snapshot
interface. Coalesce metadata changed/resolved and vault create/delete/rename events, and
invalidate on relevant `.base` changes, including renames/deletions. Refresh open badges
after a normal debounce without requiring navigation away. Discard stale asynchronous
completions. Reuse the plugin's lifecycle patterns: one shared refresh, widgets keyed by
owning component and kind, cleanup on component/plugin unload, no listener or widget
growth on repeated renders, and no per-badge polling. Use cached results when nothing
relevant changed.

Render through the existing chip style family with labels PROJECTS and REFERENCES,
correct note units, and the accessible/link behaviors above. Add only scoped collection
styles. Bump the plugin manifest according to repository conventions and describe the
additive API in its README.

### 3. Compose the grouped Dashboard and create its children

Move the existing DataviewJS navigation block above `## Tasks`. Refactor its composition
so generic chips and existing shared renderers accept the appropriate row host; do not
copy their count logic into new implementations. Add the two Browse chips through
`dashboardCollections.renderChip` with `dv.component`. Guard feature detection and
calls: an older/unloaded/throwing ledger plugin gets clickable `PROJECTS –` and
`REFERENCES –` inline fallbacks. Keep collection rendering independent of task-data
failures.

Preserve exactly one instance of every existing badge, each existing destination, the
section/cap versus whole-lane tooltip distinction, warning rules, and current fallback
behavior. In particular TODAY still opens today's daily note and shows theme/link
budgets; it is not relabeled as a task count. READY remains gated; CROWDED still counts
notes. Do not change the five Tasks queries or their order.

Implement the three-row layout and scoped responsive CSS in the Dashboard's existing
inline-style mechanism so grouping also works with older plugin versions. Shared
plugin-rendered and generic chips must align consistently within the same rows. Create
the two child pages with the exact links and selected-view embeds specified above. Keep
each child's framing minimal so its Base gets the available width.

### 4. Verify behavior, appearance, and source compatibility

Add focused Node tests in the plugin repository, following its existing `.cjs` test
harnesses, and include them in `npm test`. Test independent expected note identities and
counts rather than merely calling the same predicate twice:

- Projects: scalar resolved type links (including alias/path spellings), wrong type,
  list-vs-scalar type, template boundaries, active/waiting/closed/missing status, and
  string/list `containsAny` edge cases.
- References: nested `ref/` Markdown paths, `ref.md`, similarly named folders,
  non-Markdown attachments, other status values, and list-vs-scalar status.
- True zero, cold/incomplete metadata, unreadable/missing/changed Base definitions,
  renamed views, presentation-only Base edits, and failure isolation between the two
  collections. Source contract changes must never produce plausible stale numbers.
- Create, delete, rename, folder move, and status/type changes refresh counts; repeated
  render, two Dashboard panes, component unload, and plugin unload leave correct
  widgets/listeners and ignore late asynchronous results.
- Links, title/accessible unit text, keyboard/modifier navigation, and fallback
  rendering with absent/old/throwing APIs.

Run `npm test` and `npm run validate` in the linked plugins repo. Existing
dashboard-parity, ready-badge, and note-ready tests must remain green. Use a
parameterized/disposable harness on the actual changed `dash.md` to verify the three
groups and exact-once badge composition, feature-detection fallback paths, and unchanged
task frontmatter/queries. Do not hardcode an agent's checkout or copy private vault
contents into test fixtures.

In Obsidian, compare each badge to the row count of its pinned default Base view and
verify the children appear beneath Dash through `parent`. Open both badges and return
links. Check Reading view and Live Preview at normal width and a 320–375px pane,
light/dark themes, large counts, unavailable states, keyboard focus, and two
simultaneous Dashboard panes. Ensure no clipping, accidental restyling of daily notes,
duplicate chips, or lost task anchors. If desktop UI access is unavailable, explicitly
report that visual/Base parity acceptance is pending rather than claim a headless check
proves it.

### 5. Document and deliver to the actual vault

Update `docs/plan.md`'s Dashboard ownership/ordering description and the relevant Dash
row in `docs/freshness.md`, plus a concise `docs/dashboard.md` explaining the three
groups, two children, exact count scopes, source/view contract guard, and unavailable
behavior. Update the plugins README/manifest and test script registration. No Rust
behavior change or broad Rust test run is required for documentation-only changes in
bob-cli.

Deploy Bob Ledger Tools from the **opened linked source checkout** using
`bob plugins sync --no-pull --repo <opened-plugin-repo> --plugin bob-ledger-tools`,
first with `--dry-run`, then for real. Target the configured live Bob vault and do not
use `--force` to overwrite unrelated drift. Never edit the installed `main.js` or CSS as
the source. Verify that the plugin deployed and its new API is available after the
normal reload path.

Deliver the scoped vault-note patch through the established SASE commit/finalizer and
Bob git-sync workflow; recheck current versions before integration. The external
repository opened during implementation is an isolated checkout, not proof that `~/bob`
changed. Verify that the intended note versions actually reach the live vault and that
sync has not quarantined them as conflicts. Follow `docs/vault-git-sync.md`; avoid
blanket sync commits of unrelated dirty changes, force-pushes, or direct edits through
an unapproved repository path. Any manual commit must use the authorized SASE commit
skill, never raw `git commit`.

Report the changed child paths, final grouping, test results, deployment status, and any
remaining manual visual acceptance. Keep rollback scoped: restore the previous Dashboard
navigation and plugin version, preserving any subsequently authored content in the child
pages and all user source notes.

## Acceptance criteria

1. `dash_projects.md` and `dash_references.md` are children of Dash, have working
   Dashboard return links, and embed the existing Bases with the intended views.
2. All ten badges appear once above `## Tasks`, in the exact Work / Review / Browse
   grouping, with usable wrapping and consistent visual alignment.
3. PROJECTS and REFERENCES show live counts equal to their default views, remain
   clickable at zero/unavailable, and never count tasks or archived collections by
   accident. Changed definitions cannot silently invalidate their counts.
4. All existing task badge destinations, values, warnings, shared-renderer lifecycles,
   fallbacks, and the five Tasks blocks retain their behavior.
5. The plugin's automated checks pass, note changes are verified in the actual target
   vault, and UI verification is completed or precisely reported as pending.
