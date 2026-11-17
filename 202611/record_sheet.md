---
tier: tale
title: Read-only record sheet under the H1
goal:
  Complete bob-cli-ms.3 with a cached read-only record sheet, complete property
  coverage, subscription docking, tests, and deployment while native Properties stays
  visible.
size: medium
proposed_by: bbugyi200.apollo.bob-cli-ms.3
proposed_with: codex/gpt-6-astra
bead: bob-cli-ms.3
create_time: 2026-11-17 11:39:01
status: wip
---

- **PARENT:**
  [202611/record_card.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/record_card.md)
- **BEAD:**
  [bob-cli-ms.3](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-ms/bob-cli-ms.3.md)

# Read-only record sheet for bob-cli-ms.3

## Goal and completion owner

Complete only phase `bob-cli-ms.3` (`record-sheet`) of the Record Card epic. Canonical
record notes gain a read-only value sheet directly below their first H1 in Live Preview
and Reading view, using cached album definitions and Obsidian metadata. It shows every
authored frontmatter key in one defined place, docks subscription facts, and leaves
native Properties and the filename-derived inline title visible. The later
`record-entry` phase enables editing and activates their hide class.

This is one medium implementation: one plugin, three cohesive source fragments,
subscription ownership integration, a focused regression suite, and plugin docs. There
are no earlier unfinished-work notes and no record-sheet implementation in the inspected
tree, so a child epic is unnecessary. Re-read the assigned bead and inspect history at
implementation start; reuse any work that has since landed.

The implementing agent closes only `bob-cli-ms.3` after verification and deployment. Do
not create beads, change its status manually, or close the parent epic. Append any
discovered work as `PROPOSED FOLLOW-UP:` notes on this phase. Before closing, run
`sase bead epic-symbols bob-cli-ms.3`, resolve its entries or re-key them to a
still-open bead, then use
`sase bead close bob-cli-ms.3 --note "<verification evidence>"`.

## Governing context and fixed decisions

Read these through audited artifact commands before implementation:

- `plan:202611/record_card.md`: settled design, shared sheet contract, and
  `record-sheet` delivery. This epic overrides the earlier research where they differ.
- `research:202611/record_card/record_card.md`: normative design and visual examples.
- `plan:202611/double_equals_schema_keymap.md`: existing shortcut behavior, which stays
  unchanged in this phase.

The parent decisions are final: `inline_title = hide` and `decision_record = no`.
Prepare the dormant inline-title selector together with the dormant Properties selector,
but do not activate either. Do not edit any memory note or generated agent instructions.
There are no new reviewer choices in this plan.

All application changes belong in linked `bob-plugins`. Open it with
`sase repo open bob-plugins -r "Implement bob-cli-ms.3 read-only record sheet"` and read
its `AGENTS.md`; use only that returned checkout. Read `docs.md` through
`sase memory read` before documentation changes. No Rust, CLI contract, Cargo version,
or changelog changes are required.

## Inspected implementation and integration constraints

Planning inspected bob-cli `f6924787` and bob-plugins `b0a9c89`; the latter includes the
existing album/bank `==` feature (`2b61e76`). Ledger Tools was `1.128.0`; read the
current version again before making its required minor bump.

Relevant paths below are relative to the opened bob-plugins checkout. Ledger source
paths abbreviate `plugins/bob-ledger-tools/src/`.

- `241v-plugin-album-header.js` supplies the Reading surface/ownership pattern, a
  priority-49 postprocessor, a `Prec.highest` CM6 widget, Source-mode exclusion,
  H1/frontmatter anchors, scoped classes, and cleanup. Its current builder creates DOM
  before comparing widget signatures: improve that order in the new sheet rather than
  copying its allocation behavior.
- `241l-plugin-album-data.js` normalizes types and preserves additive report fields
  through `albumIndexReport({...report})`. `241m-plugin-album-data-service.js` owns the
  shared in-memory report. Subscribe once during setup; mounting, repainting, folding,
  and switching notes must not request a report or spawn a subprocess.
- `242d-plugin-album-entrypoints.js` has `albumResolveSchemaTargetForFile`, but its
  underlying kind classifier accepts unknown types as records. Enforce the strict
  canonical-type gate before calling this resolver, then verify direct folder membership
  and the resolved parent's actual album identity.
- `241hg-plugin-bank-subscription-header.js` owns a second H1 widget and consumes
  `subscriptionSurfacesReport`. `241he` builds its model, `241hf` paints and cleans its
  DOM, and `241hc` has the money formatter. Existing `albumCleanSubscriptionHeader`
  removes every matching descendant: docking needs explicit ownership so standalone
  cleanup cannot delete a sheet-owned band.
- `scripts/test-ledger-tools-album-header.cjs` has usable Obsidian, CM6, DOM, and timer
  fakes. Subscription-surface and album-entrypoint tests are regression peers.
- Source order is `src/fragments.json`; lifecycle is `170-plugin-lifecycle.js` and
  `170-plugin-lifecycle-teardown.js`; mixins install in `310-install-methods.js`.
  Exports currently end with `369-exports-schema-double-equals.js`.

## 1. Pure gate, field presentation, and complete key coverage

Add `242i-plugin-record-sheet-model.js`. Keep each edited fragment at most 1000 lines;
split formatters into an adjacent cohesive fragment if necessary and register its order
explicitly.

Expose a pure
`recordSheetGate({ path, frontmatter, report, desktop, metadataCache, vault })`
returning at least `{ mount, hideEligible, editable, state, reason }`. Keep capability
eligibility distinct from actual availability of edit handlers.

- `albumTypeNames(frontmatter.type)` must identify exactly the canonical record type,
  not a legacy noun, mixed type, unknown type, or bank month.
- Require a real metadata snapshot, a cached report, and desktop. Resolve the one parent
  to a root album in that report, verify its album type, and require the record path to
  be an immediate Markdown child of `<album-stem>/`. Reject missing, ambiguous,
  mismatched, cyclic/non-album parents and nested/outside-folder records.
- A broken album schema still mounts a warning sheet with its diagnostic and an album
  YAML jump; it is neither editable nor eligible to hide. A healthy schema has a
  non-null `schema_fingerprint` and no album error issue.
- Missing `record_values_v1` still permits a healthy read-only sheet, with
  `editable: false` and `hideEligible: false`. Malformed/missing capabilities must never
  enable either. Mobile/no report and failed identity resolution mount nothing.
- Source-mode exclusion belongs to the surface host as well as tests. Keep native
  Properties visible in every branch in this phase.

Build `recordSheetModel(...)` from the gated album entry, the record's cached
frontmatter, its first real H1, and an injected clock. Accept an optional overlay
argument as an integration seam; no optimistic update store or writer is implemented
here. Expose reusable value models/render inputs for the later Record Card.

Preserve declaration order and bob's field/choice labels. Every schema field gets
exactly one row or ghost entry. Model valid, invalid, empty optional, missing required,
and broken-schema states separately; never turn malformed shapes into empty values. Lint
is presentation only and never authorizes writes.

| Value            | Presentation                                                                                 |
| ---------------- | -------------------------------------------------------------------------------------------- |
| Choice           | Only selected emoji/label chips; an unknown key stays raw with a textual lint mark.          |
| Text             | Literal text, with complete wikilink spans rendered as internal links.                       |
| Date             | Local-date-aware `today`, `yesterday`, or a friendly date plus relative age.                 |
| URL              | Safe real anchor, middle-truncated label, complete value in its title.                       |
| Link             | Resolved internal-link display text and hover behavior.                                      |
| Number           | Tabular numeral; subscription amount uses existing bank money formatting with correct units. |
| List             | One chip per item; preserve commas within items and visibly retain invalid items.            |
| Empty optional   | One shared row of `＋ <Label>` ghost chips, without edit controls yet.                       |
| Missing required | Full field row and amber `missing` pill.                                                     |

Kind/shape failures retain raw text in monospace plus a reason, including unknown
choice, invalid date/URL, out-of-bounds number, scalar/list mismatch. Do not coerce
numeric strings or invalid arrays into valid values.

Account for every authored frontmatter key exactly once:

- `parent`: breadcrumb; `type`: implicit identity; matching `title`: the H1.
- A differing `title` appears in the read-only fold with `differs from heading`. When
  the H1 is missing, retain title visibly in the fold rather than claiming an absent
  heading represents it.
- Valid `created`: footer (`added today, 7:55 AM`, `added yesterday`, or
  `added Aug 8, 2025`). Omit it when absent; retain an invalid authored value visibly
  with lint rather than dropping it.
- Schema keys: rows/ghosts; everything else: `N other properties` fold with compact
  values, readable wikilinks, `null` as `—`, and lists joined with `, `. Honor the
  album's declared legacy patterns and label matching keys `legacy`.
- Exclude Obsidian's synthetic metadata positions from authored-key coverage.

The default is expanded. Collapsed presentation contains the breadcrumb, the selected
choice chips with the group field first, then `· N fields`. Export a stable signature
over visible model content, state, heading, collapse/fold state, hue, overlay, and
subscription band; ordinary body text must not affect it.

## 2. Read-only DOM and styling

Add `242j-plugin-record-sheet-dom.js` with reusable value rendering and a sheet builder.
Use text nodes/textContent for all supplied strings. Never interpolate frontmatter or
report data into HTML. Validate external-link protocols, and use existing internal link
conventions (`internal-link`, `data-href`, navigation, hover preview) for wikilinks,
including links mixed with literal text.

Render `role="group"`, labelled `<Title> record details`, album breadcrumb/noun, a
semantic `dl` for fields, ghost strip, optional created footer, and a read-only
disclosure for other properties. Collapse and disclosure controls have accessible names,
`aria-expanded`, keyboard activation, and visible focus. Required markers stay quiet
until missing; lint always has text alongside color.

Until record-entry supplies handlers, render no Edit button, dialog claims, change
labels, or dead clickable value/ghost controls. Rows can take focus for `y`. That
gesture jumps to the record's own YAML key; the broken-schema jump targets the album's
issue line. Reuse safe Source navigation but do not accidentally register an album
Schema Card excursion for a record. Source bytes may be read lazily on an explicit YAML
jump to locate its key; this is not a per-note bob request or a YAML writer. Missing key
locations fall back to the frontmatter rather than an unrelated line.

In `plugins/bob-ledger-tools/styles.css`, use the existing root-hue convention, 8px
corners, 2px start border, 1px border at roughly 30% hue mix, and a quiet 4% tint.
Labels use small interface text; values use the note text font. Stack labels over values
at 375px, wrap long text/links/chips, support light/dark and reduced motion. Prepare the
600ms changed-row accent style for the later overlay, without generating fake save
events in this phase.

Define dormant `.bob-record-sheet-hides-properties` selectors for the whole native
Properties block and the view's own `.inline-title` (the accepted decision). Scope with
child combinators so an ancestor cannot hide a nested embed's Properties. Nothing in
this delivery adds that class, even for `hideEligible: true`. The regular
`.bob-record-sheet-surface` class is styling-only. Real Obsidian selector acceptance
remains owned by the epic land agent before hide activation is accepted.

## 3. Surface lifecycle and subscription docking

Add `242k-plugin-record-sheet.js` with `BobLedgerToolsRecordSheetMixin` and wire setup,
teardown, source order, mixin installation, and a new final
`37x-exports-record-sheet.js`.

- Live Preview: a separate `Prec.highest` ViewPlugin mounts at the end of the first
  non-code H1, falling back to after frontmatter. Resolve path and mode each update.
  Compute model, signature, and anchor before creating any DOM. If unchanged, return the
  same decoration/widget; moving an anchor may rebuild the decoration without rebuilding
  unchanged content. Body typing must not allocate a replacement sheet.
- Reading: reuse the album ownership pattern and priority-49 postprocessor, inserting
  after the first owned non-code H1 (top-of-content fallback without one). Treat each
  pane, embed, and hover surface independently. Ignore nested headings belonging to a
  different embed and never paint Reading chrome inside CM6 chunks.
- Store collapse and fold state in session maps keyed by canonical record path. Preserve
  them across report refreshes and concurrent panes; never write them to frontmatter or
  persist them in plugin settings.
- Subscribe once to the shared album service at setup. Coalesce refreshes on album
  report changes, metadata `changed` for mounted records and their parent identities,
  subscription-report changes, and an explicit overlay-refresh seam for record-session.
  Use existing cached subscription data rather than installing another fetcher. Relative
  date labels should refresh across local midnight with a cleaned-up timer.
- Remove owned DOM/classes on path/type/parent changes, mode switch, view destruction,
  unload, and disable. Clear timers, subscriptions, maps, and pending work; late
  callbacks after teardown must do nothing. Never remove another surface's nodes.

Expose `recordSheetOwnsPath(path)` using the same active/mount gate. A subscription
record's sheet appends `bankSubscriptionPaintHeader` as a trailing band, using
`bankSubscriptionHeaderModel(subscriptionSurfacesReport, path)` and including its
signature in the sheet signature. Show the band only when that model is ready.

Update `241hg` so its standalone Live Preview and Reading headers yield for paths the
sheet owns, and restore them when ownership is lost. Ownership changes refresh both
surfaces so no stale duplicate widget survives. Add an ownership marker or scoped
cleanup in `241hf`: standalone refresh, render-child unload, and teardown must not
delete docked bands. Keep existing album summary and index behavior unchanged.

## 4. Regression evidence

Add `scripts/test-ledger-tools-record-sheet.cjs` to the explicit `package.json`
`test:unit` list. Reuse the existing fake environment pattern and freeze the clock. Use
several focused test files only if needed for readability; register each explicitly.

Required behavioral evidence:

1. Gate matrix: canonical record; bank month; legacy/mixed/other type; absent
   metadata/report; mobile; missing/ambiguous/cyclic/non-album/mismatched parent;
   nested/outside folder; broken schema; absent/malformed/present capability. Healthy
   capability changes eligibility without adding a hide class or editing control.
2. Fixtures shaped like `bead_lifecycle`, a sparse movie, legacy-keyed `eat` record with
   wikilink location, and subscription with list `match` and `recur_id`. Assert each
   authored key has exactly one home, all fields retain declaration order, all kinds
   render, invalid shapes remain visible, and missing differs from invalid.
3. Deterministic footer dates, absent/invalid `created`, H1/title agreement and
   mismatch, missing H1, legacy patterns, nulls and unknown compound values.
4. Safe literal DOM for HTML-looking labels/values, dangerous URL schemes remaining
   inert, valid internal links and hover metadata, meaningful focus/ARIA, and working
   YAML destinations. No writer or subprocess call during note mount/render/folding.
5. Expanded/default and per-path collapse/fold retention; collapsed group-choice
   ordering; body edits reuse the actual decoration and do not call the DOM builder;
   metadata, report, heading, docked report, and overlay changes invalidate it.
6. One header per Reading/Live Preview surface; owned H1/fallback placement; two panes,
   nested embeds, Source transitions, detached Reading attachment, and late callbacks.
   Unload removes all owned chrome, scoped classes, listeners, and timers.
7. Subscription takeover/release, exactly one timeline, and standalone cleanup unable to
   delete the docked timeline. Existing subscription surfaces still work when the sheet
   is unavailable.
8. Active style-class lifecycle and unconditional absence of the dormant hide class on
   every phase-3 path. Do not claim DOM fakes prove real CSS selector behavior.

Run the new focused tests plus the existing album-header, album-entrypoint,
album-records, and bank-subscription-surfaces suites first. Full repository gates below
provide the remaining regression coverage. For light/dark and 375px, inspect a rendered
fixture using the available UI harness/browser tooling and record the result; never
present a source/CSS assertion as a screenshot check. The epic land agent owns the full
real-Obsidian Record Card walkthrough and exact hide-selector DOM probe.

## 5. Docs, validation, deployment, and completion

Add `docs/record-card.md` in bob-plugins with reference frontmatter and the shipped
sheet behavior, fold/collapse/YAML access, fail-open cases, and subscription docking.
State clearly that native Properties remains visible at this phase and editing arrives
separately. Add its entry to `docs/nav.yml` and cross-link from `docs/albums.md`. Leave
`==`, palette, Command Mode, and Ctrl+Shift+P behavior unchanged. Read the current
Ledger Tools manifest version, bump its minor, rebuild `main.js`, and generate docs;
never hand-edit generated bundles/inventory.

Run from the linked repo, with `BOB_CLI_DIR` pointing to the implementing agent's own
bob-cli checkout:

```sh
npm run build
npm run docs:gen
npm test
npm run validate
BOB_CLI_DIR=<current-bob-cli-checkout> npm run docs:check
```

Use `/sase_monitor` for long gates, including the epic-required bob-cli `just check`;
follow its staged-handoff/finalizer flow instead of leaving a process running when
ending a turn. Inspect intended generated diffs and verify fragment sizes. If a check
fails, establish whether it reproduces identically on clean base; a demonstrated base
failure is a `PROPOSED FOLLOW-UP:` with evidence and any existing tracking bead, not a
reason to leave this completed phase open.

Deploy only the edited plugin from its opened checkout with
`bob plugins sync --repo "$PWD" --no-pull -p bob-ledger-tools`. Verify the sync result.
If overwrite protection refuses, follow `docs/develop/deployment.md`: inspect upstream,
incorporate the landed protected build, or wait for its live author via the appropriate
SASE workflow. Do not force a deploy. Ask only for an unlanded, abandoned protected
build, identifying what would be removed.

Record the implementation, tests, visual evidence/limitations, and deployment outcome on
`bob-cli-ms.3`; close only that phase after its epic-symbol check. Finish through
`/sase_final`, declaring every changed repository for host commit. Editing services,
modal/stages, new shortcuts/commands, active hiding, and memory decisions remain with
the later phases and epic land agent.
