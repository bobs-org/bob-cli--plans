---
tier: epic
title: Bob Ledger Tools projects catalog card
goal:
  Complete bob-cli-ld.9 with a live projects card, reliable auto-sync, pane-local
  interactions, real Obsidian evidence, and verified deployment.
phases:
  - id: projects-data
    title: Shared projects report service and debounced auto-sync
    size: medium
    depends_on: []
    description:
      "projects-data: implement projects JSON decoders, the shared report/sync service,
      lifecycle fingerprints, and deterministic data-service tests in bob-plugins."
  - id: projects-card
    title: Projects catalog card in Reading view and Live Preview
    size: medium
    depends_on:
      - projects-data
    description:
      "projects-card: render the hued projects card, live chips, per-pane folds, nested
      children, recovery actions, and responsive layout with focused UI tests."
  - id: projects-acceptance
    title: Real Obsidian acceptance, documentation, and deployment
    size: medium
    depends_on:
      - projects-card
    description:
      "projects-acceptance: add the isolated real-Obsidian runner and offline wrapper,
      inspect screenshots, document the card, run full gates, deploy, and record closure
      evidence for the land agent."
proposed_by: bbugyi200.athena.bob-cli-ld.9
proposed_with: codex/gpt-6-astra
parent_bead: bob-cli-ld.9
create_time: 2026-11-13 15:29:23
status: wip
---

- **PROMPT:**
  [prompts/202611/projects_catalog_card.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/projects_catalog_card.md)
- **PARENT:**
  [202611/subprojects_catalog_section.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/subprojects_catalog_section.md)

# Projects catalog card for Bob Ledger Tools

Implement the remaining work of **bob-cli-ld.9**. This is a child epic of that assigned
phase, not a replacement for the enclosing catalog epic. The child epic's land agent
owns closing bob-cli-ld.9 after its phases and integration checks pass. Child phase
workers close only their own assigned beads. Nobody in this child epic closes bob-cli-ld
or its other phases.

The approved source is `plan:202611/subprojects_catalog_section.md`, especially “Phase
projects-catalog-card” and the shared contract. Read it with `sase artifact read`;
inspect the assigned bead with an audited `sase bead read`. Its decisions remain final:
`sync_lifecycle = locked`, `memory_decision_record = no`, and
`memory_capture_glossary = yes`. The permitted glossary edit belongs to the enclosing
rollout phase; **this child epic edits no memory** and adds no reviewer decisions.

## Evidence and size

The planning checkout is clean. bob-cli-ld.9 has no earlier-work or remainder notes. The
JSON report dependency landed as `4570a7ea` and the sync writer as `7cd57603`; both
dependency beads are closed. The current bob-plugins checkout contains the notebook and
album cards, and the dual-format SUBOPEN reader (`7b27f05`), but no projects data
service or `[!projects]` card. Recheck the current tree before coding so any work landed
since this proposal is reused.

Three sequential medium phases keep each coding agent's scope bounded. The service has
independent concurrency and event-ordering work; the renderer has two Obsidian
lifecycles and pane-local interaction state; acceptance requires a real Electron
process, screenshot inspection, and deployment. Combining those into one tale would put
both implementation and acceptance at risk. The existing album service alone spans two
substantial fragments, and its real UI runner is over 2,000 lines. Reuse the existing
helpers and harness instead of copying that whole feature.

## Repository and rollout boundaries

- Open `bob-plugins` through `/sase_repo` and use only the checkout it prints. Read its
  `AGENTS.md`. All feature code belongs in `plugins/bob-ledger-tools/src/` and its
  source stylesheet, with generated `main.js` rebuilt by `npm run build`. Register
  fragments in `src/fragments.json`; no hand-edited fragment exceeds 1,000 lines. Wire
  mixins through `310-install-methods.js`, setup/teardown through
  `170-plugin-lifecycle.js`, and test exports through a small export fragment.
- Keep the existing legacy Sub-projects recognizers. Notebook and album 🧩 group
  recognizers remain scoped to their own callouts. Do not change SUBOPEN semantics,
  lifecycle derivation, project membership, Rust report contracts, or Mac Capture.
- The plugin never writes catalog bytes. Both automatic and manual writes call
  `bob projects sync -f json`, which owns the approved locked transaction.
- Never run a real projects sync, nightly, or capture against Bryan's live vault during
  this work. Exercise mutations in isolated, Git-initialized scratch vaults with
  isolated Bob configuration and no live remotes. `bob plugins sync` deployment is
  required by repository instructions and is authorized; do not use live-vault project
  edits to test the newly deployed feature.
- For every changed plugin phase, bump Bob Ledger Tools from the then-current manifest
  version according to repository convention, rebuild generated assets, regenerate the
  documentation inventory when necessary, run the required checks, and run
  `bob plugins sync`. Never force an overwrite: follow the repository's deployment
  conflict runbook. Use `/sase_monitor` for long commands and wait for the staging
  command to exit before finalizing. Include every edited repository in `/sase_final`.
- Do not create follow-up beads. Append `PROPOSED FOLLOW-UP:` entries to the assigned
  phase; the child land agent relays unresolved findings to bob-cli-ld.9 for the
  enclosing land agent. A failure reproduced identically on clean base is recorded with
  evidence, not treated as a reason to leave a completed coding phase open.

## Shared data and UI contract

Use the **landed** JSON shapes, not a copy of the album decoder or just the example in
the parent design. See bob-cli `src/native/projects/output.rs`,
`src/native/projects/sync_engine.rs`, `tests/cli/projects/list.rs`, and the sync tests.
Add fixtures reflecting those shapes.

The list envelope has `schema_version: 1`, `command: "list"`, `ok`, `today`, `projects`,
`catalogs`, `findings`, and `counts`:

- A project has `path`, `stem`, `status`, nullable `state`, nullable `criterion`,
  nullable `parent`, `children` paths, nullable `closed_on`, `prj`, `blocked`, and
  `tasks`. `prj` is an object with nullable `line`, `mark`, `hidden`, and `scheduled`.
  An unknown value must never become zero, false, or ready through coercion.
- States are `open`, `scheduled`, `blocked`, `warning`, `done`, and `canceled`.
  `blocked` is null, scheduled with nullable `date`, dependency with `waits_on`, or
  inherited. `tasks.open`, `tasks.shown`, and `tasks.today` are nullable counts.
- A catalog has `host`, `path`, `kind`, `state`, nested `rows` counts (`open`,
  `blocked`, `warning`, `closed`), and `issues`. Catalog issues do not have the album
  issue schema's required path/suggestions fields. A valid `ok: false` report can
  contain useful rows and must remain displayable with its findings.
- The **sync** report has `schema_version: 1`, `command: "sync"`, `ok`, `dry_run`,
  `changed`, `changes`, `events`, `catalogs`, `findings`, `commit`, `sync`, `warnings`,
  and optional `error`. It has no list `projects`, `today`, or `counts` fields; sync
  catalog counts are top-level `open/blocked/warning/closed`, not `rows`. Decode it
  separately, surface a well-formed refusal, and revalidate list data after sync instead
  of installing sync output as a list report.

Index projects and catalogs by normalized full vault-relative path. Resolve link targets
through the metadata cache and retain identity when stems collide; do not join by
display text. Membership and nested children come from Bob's report, never from a second
JS parent-resolution or Blocked engine. Area and terminal project hosts are supported.
Open rows sort by stem case-insensitively, path tie-break; closed rows sort newest
closure first, then undated A–Z. Preserve this contract when grouping report rows for
area hosts; project `children` are already ordered.

The card enhances only real `[data-callout="projects"]` elements, with class
`bob-projects-catalog`. It uses `notebookHostHueName`, `data-root-hue`, existing catalog
tint/border/radius tokens, and shared DOM/fold utilities. A redundant native callout
title is hidden. The header has the lucide puzzle icon, “Projects”, and
`N open · N blocked · N closed`; blocked includes scheduled and dependency/inherited
blocked rows and is a subset of open, never added twice.

Rows show a hue ring for ready, then calendar-clock, chain, alert-triangle, check, or x
for the other states; a bold stem link; and the muted criterion clamped to one line,
with full text on hover and keyboard focus. Look up an H1 tooltip through the resolved
child's cached headings when available: the JSON report has no title field. Do not add
one to the CLI for this card. Render untrusted text as text. State and counts are
report-driven; preserve readable stored links/criterion when live data is unavailable.
Catalog rendering never rewrites note content.

Right-side chips are `N tasks` for known `tasks.open`, `● today` only when its count is
positive, `empty` only when `prj.hidden === false`, and the reported blocked reason
(`Dec 8 · in 3w`, `waits on <project>`, or `parent blocked`). Unknown fields omit their
chip. Reuse date/age helpers with the report's date. A `▸ N` button reveals a read-only
list of that child's direct `children`; never recurse without bounds or write nested
Markdown.

The Closed group defaults folded and peeks at its two newest rows. Its button says
`Closed · N`, supports Space/Enter, and exposes `aria-expanded`. The all-closed case
shows the header and this group. With no children, no empty card remains. Store both
Closed and nested-child expansion in a WeakMap keyed by the stable pane view, with
host/child path keys inside. DOM replacement and background refresh must not reset
state, share it with another pane, or steal focus. Reconcile stable DOM nodes where
possible, or restore the same action's focus without scrolling.

`↻ sync` appears for a missing/stale host catalog; a missing callout needs an actionable
anchor at its Projects heading, H1, or safe body position as in the album card. A
damaged catalog is a visible diagnostic, never a guessed empty set or an automatic
repair. `＋ project` copies exactly `@<host>^` using the catalog host identity and the
existing copy-affordance pattern; it does not submit capture. Report failures keep
last-good data and show Retry, and first-load failure keeps the source readable without
fake zeros. At roughly 375 px, chips wrap below the criterion and the tally abbreviates
while retaining an accessible full label.

## Phase projects-data

Implement small project-specific model/decoder and service fragments adjacent to the
album/notebook data family. Reuse transport, timers, and basic helpers where their
contracts fit; do not refactor the entire catalog family.

1. Provide `projectsReportState()`, `apiProjectsReport(options)`,
   `apiProjectsSubscribe(listener)`, `apiProjectsInvalidate(reason)`, and
   `apiProjectsSync({automatic})`, following `241l-plugin-album-data.js` and
   `241m-plugin-album-data-service.js`. One list flight is shared across panes;
   generation and lifecycle epoch checks discard stale results. Subscribers get loading,
   stale, success, error, and sync-busy changes. Keep the last-good report on
   transport/schema failure, queue a newer-generation refresh, and resolve
   waiters/unsubscribe/cancel timers on teardown. Preserve `bobSpawnJson`'s desktop,
   timeout, buffer, executable-path, and nonzero-exit-with-JSON handling.
2. Sync is single-flight with at most one trailing operation for a genuine change while
   syncing. Always revalidate list data afterward; a refusal remains visible and must
   not cause a tight automatic retry loop. Calls after unload do nothing.
3. Fingerprint project identity plus frontmatter `type`, scalar `parent`, `status`, and
   the real lifecycle `^prj` checkbox. Include old identity for delete/rename and
   project-to-nonproject transitions. Ignore frontmatter/fenced example checkboxes;
   handle missing/duplicate lifecycle tasks deterministically without deriving lifecycle
   state in JS. Read metadata event text when available; if cached reads are needed, use
   per-path generation guards so late reads cannot undo a newer fingerprint. Seed
   baseline fingerprints before enabling automatic sync, using the existing
   startup/metadata-readiness path (`171-relation-index-lifecycle.js` already seeds
   notebook/album data). Startup indexing is not a structural change.
4. Debounce structural sync like notebooks (about three seconds); a promoted project,
   changed parent/status/type, lifecycle checkbox, rename, or deletion updates catalogs
   within seconds. Generated callout writes, task counters, metadata position changes,
   and ordinary prose edits never cause a sync loop. They may invalidate live list data.
   Invalidate on relevant task/ledger edits and day rollover so count/today/reason chips
   remain live; revalidate on show/focus. Retain existing relevant-path and archive
   exclusions. Area/project ancestry changes refresh hue and host data without inventing
   new membership rules.
5. Add `scripts/test-ledger-tools-projects-data.cjs` to the explicit `test:unit` list.
   Test both actual JSON envelopes, nullable fields, malformed/schema failure, usable
   `ok: false`, one shared fetch, stale result races, last-good errors, trailing
   invalidation, sync refusal/revalidation, startup seeding, all structural event types,
   lifecycle-read races, body/callout no-op fingerprints, day rollover, and unload. Use
   controlled timers and transport stubs, not wall-clock sleeps.

Acceptance: focused tests and full plugin build/tests pass, generated output is current,
the API is wired and usable by the next phase, and no painter exists that can call an
undefined service. Bump/deploy the changed plugin per the shared rules.

## Phase projects-card

Implement model, DOM, and lifecycle fragments using `241o/p/q-plugin-album-catalog*`,
`241b/c/d-plugin-notebook-catalog*`, and `241r/s/t-plugin-album-records*` as the
reference. Reuse actual helpers; if a helper is narrowly hardcoded to albums, extract
only the shared primitive needed and retain coverage for both callers.

1. Build host, row, state-chip, tally, blocked-reason, health/action, and fold models
   from the service contract above. Use path keys for duplicate stems and identity
   across reorders. Preserve native internal-link navigation and modifier behavior.
2. Implement the complete card, Closed peek, nested children, tooltip/focus behavior,
   Retry/sync/copy controls, missing-catalog anchor, and narrow layout. Ensure unknown
   data does not manufacture healthy/empty states. Keep report errors and damaged
   catalog issues actionable without hiding the stored source.
3. Register a Reading postprocessor and Live Preview ViewPlugin. Respect embed source
   identities, ignore fences/examples and foreign callout types, and do not let
   postprocessor cleanup remove CodeMirror-owned chrome. Use MarkdownRenderChild
   cleanup, editor destruction, timer cancellation, and unsubscribe. Preserve fold state
   and focused controls across revalidation and DOM remounts within the same pane;
   separate panes stay independent. Unload removes only generated chrome.
4. Add `scripts/test-ledger-tools-projects-catalog.cjs` with the existing Ledger Tools
   DOM harness. Cover all six states, unknowns, catalog statuses, zero/only closed/mixed
   hosts, duplicate stems and path links, two-row peek, nested children, keyboard/ARIA
   behavior, two-pane isolation, focus/folds across refresh and remount, missing-catalog
   anchors, retry/sync/copy, cleanup, and no vault mutations. Add regressions proving
   notebook/album 🧩 groups are unaffected and not painted as projects. Wire the tests
   in `package.json`.

Acceptance: the complete approved UI works in both view modes under focused tests,
existing catalog tests remain green, and the plugin passes build/full tests before
version bump/deployment. Do not defer card features to the acceptance phase.

## Phase projects-acceptance

1. Add `scripts/obsidian-projects-ui-e2e.cjs`, supported by small reusable fixture and
   probe helpers if needed, following the isolated vault/profile, owned-process, CDP,
   real-Bob logging shim, and cleanup patterns in the existing notebook/album runners
   and `obsidian-startup-smoke-lib.cjs`. Add `scripts/test-obsidian-projects-ui-e2e.cjs`
   to `npm test`; its offline self-test validates fixtures/probes without launching
   Obsidian. Offline success is not a substitute for the real acceptance run.
2. Use a verified Bob binary built from the current bob-cli checkout, supplied by
   `BOB_BIN`; record its path/version and actual invocation results. Use a scratch Git
   vault and isolated config. Keep all live-vault paths out of the child process
   configuration. Seed an area host and a project host with 60+ direct children, all six
   states, long criteria, a nested child, duplicate stems, and dated/undated closures.
   Also cover all-closed and empty hosts.
3. Run real Reading and Live Preview scenarios: light/dark at desktop and about 375 px,
   keyboard Closed and nested toggles, independent panes, focus/fold persistence after
   refresh, missing/stale sync, report failure/last-good/Retry, copy starter, and
   creation/reparenting/lifecycle-change auto-sync. Show that settled automatic writes
   do not repeat. Separate user fixture edits and Bob's expected sync writes from
   painting; hash note bytes before/after passive paint, folds, navigation, and report
   refresh to prove those operations write nothing.
4. Save the JSON acceptance report and screenshots, inspect each screenshot, and fix
   failures or clipping before declaring completion. Preserve the report and image
   evidence with audited SASE artifact tooling; record artifact refs and commands on the
   assigned bead and bob-cli-ld.9. Do not claim a mocked, skipped, or unavailable real
   run as passing. Resolve an environment failure or use a monitored continuation;
   report the concrete blocker if truly unavailable.
5. Add “Projects catalog card” to bob-plugins `docs/bob-ledger-tools.md`, and document
   the offline and real runners in `docs/develop/validation.md`. Cover chips, Closed
   peek and keyboard, nested children, sync/capture, errors, auto-sync, and display-only
   behavior. Read `docs.md` through `/sase_memory_read` first. Update the source docs
   and generated inventory; refresh the bob-cli mount with its tools, never hand-edit
   the mounted copy. Use the opened checkout in `BOB_DOCS_BOB_PLUGINS_DIR` /
   `BOB_CLI_DIR` as appropriate.
6. Run `npm run build`, `npm test`, `npm run validate`, and
   `BOB_CLI_DIR=<current-bob-cli-checkout> npm run docs:check` in bob-plugins. Run the
   enclosing epic's `just check` in bob-cli and the docs gate with the current linked
   source. Use `/sase_monitor` for commands too long for this turn. Run
   `bob plugins sync` after the verified final plugin build and inspect the result;
   follow deployment protection instructions on refusal. Record final plugin version and
   deployment evidence.

Acceptance: actual screenshots and both view modes pass; reports establish no unintended
note writes; docs/build/full tests/deployment are verified; any proven clean-base
failure has a `PROPOSED FOLLOW-UP:` with reproduction evidence.

## Child epic landing and closing bob-cli-ld.9

The child land agent re-reads this plan and bob-cli-ld.9, checks that all three child
phases landed, reviews their real acceptance evidence and deployment result, and
verifies integration without redoing completed work. It closes its own child epic only
through the host-authorized landing workflow. It then closes the containing **phase**
bob-cli-ld.9 when the whole scope is complete; this is not authorization to close the
enclosing bob-cli-ld epic.

Before closing any assigned phase, run `sase bead epic-symbols <phase-id>` and resolve
or re-key each remaining Justfile entry to a still-open appropriate bead. The planning
inspection found none for bob-cli-ld.9, but landing must recheck with
`sase bead epic-symbols bob-cli-ld.9`. Once complete, use
`sase bead close bob-cli-ld.9 --note "<implemented UI, test and screenshot artifact refs, deployment version, and verification evidence>"`.
Do not change status by hand or reopen an in-progress bead. Record remaining
out-of-scope work only as `PROPOSED FOLLOW-UP:` notes; preserve the enclosing epic's
other phases and decisions.
