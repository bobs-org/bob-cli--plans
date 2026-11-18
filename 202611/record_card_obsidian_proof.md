---
tier: epic
title: Prove the Record Card in real Obsidian and verify vault coverage
goal: Finish bob-cli-ms.8.3 with a repeatable CDP walkthrough, DOM-backed hide repairs,
  guarded keyboard-edit proof, read-only vault coverage, and verified deployment evidence.
parent_bead: bob-cli-ms.8.3
phases:
- id: sheet-dom
  title: Prove record sheet rendering and fix Properties hiding in real Obsidian
  size: medium
  depends_on: []
  description: 'sheet-dom: build the isolated Record Card CDP runner and offline wrapper,
    prove the sheet and native Properties lifecycle on athena, and fix defects using
    captured DOM evidence and regression tests.'
- id: card-workflows
  title: Prove Record Card entry points and guarded keyboard edits
  size: medium
  depends_on:
  - sheet-dom
  description: 'card-workflows: extend the same CDP runner with all Record Card entry
    points, typed keyboard saves, Undo, refusal cases, YAML return, Schema Card navigation,
    and byte-neutrality checks; fix failures and publish inspected evidence.'
- id: coverage-release
  title: Verify live record coverage and the final deployed build
  size: medium
  depends_on:
  - card-workflows
  description: 'coverage-release: add and run the read-only per-record coverage audit,
    run the complete walkthrough and repository gates against the final tree, deploy
    without force, and publish the final acceptance and deployment evidence.'
proposed_by: bbugyi200.apollo.bob-cli-ms.8.3
proposed_with: codex/gpt-6-astra
create_time: 2026-11-18 08:59:22
status: wip
bead_id: bob-cli-ms.8.3.1
---

- **PROMPT:** [prompts/202611/record_card_obsidian_proof.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/record_card_obsidian_proof.md)
- **PARENT:** [202611/record_card_title_and_proof.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/record_card_title_and_proof.md)
- **BEAD:** [bob-cli-ms.8.3.1](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-ms/bob-cli-ms.8.3.1.md)

# Finish bob-cli-ms.8.3: real Obsidian Record Card proof

## Scope and why this is an epic

The assigned phase requires a new real-Obsidian harness, a DOM-backed Properties-hide
repair, a large keyboard/write acceptance matrix, and a read-only live-vault audit with
deployment evidence. Three sequential, bounded coding phases separate those concerns.
They share one harness, so there is no parallel editing of its files. Each phase below
is medium direct implementation work, not another planning assignment.

This epic's parent is **bob-cli-ms.8.3**, not bob-cli-ms.8 or bob-cli-ms. The child
epic's land agent completes and closes bob-cli-ms.8.3 after auditing all three phases.
It must leave bob-cli-ms.8 and bob-cli-ms open for their own land agents. Phase workers
close only their own assigned beads. Do not create beads manually; plan approval owns
the child epic/phase creation.

## Checked starting state

Planning inspection on 2026-11-18 found:

- bob-cli-ms.8.3 is in_progress with no prior work notes. Its two prerequisites are
  closed. `sase bead epic-symbols bob-cli-ms.8.3` reports no entries.
- bob-cli HEAD is `2f80442b`, which writes H1-less records and extends tidy. The
  prerequisite's evidence records all 192 live record H1s removed in vault commit
  `70dc35cb6`, followed by a clean no-op tidy. Do not repeat that migration.
- The linked bob-plugins checkout is clean at `04ef18f`. Commit `9ad471d` already
  implements the sheet title and H1-less mounts; bob-ledger-tools 1.134.0 was deployed
  without force by bob-cli-ms.8.1. Do not rebuild those features from scratch.
- `scripts/obsidian-record-card-e2e.cjs` and its offline wrapper do not exist.
- `plugins/bob-ledger-tools/styles.css` still uses direct-child selectors for
  `.metadata-container` and `.inline-title`, while `recordSheetSetEditorScope` marks
  `.markdown-source-view` and the Reading path uses `recordSheetReadingScope`. Whether
  those selectors miss the real wrappers must be established by DOM evidence.
- The existing album harness writes `showInlineTitle: false`; the new runner must set it
  true and enable native Properties, otherwise its hide assertions can pass for the
  wrong reason. Its ordinary fixtures also use legacy noun types: construct canonical
  `type: "[[record]]"` fixtures for positive Record Card cases.
- A read-only SSH preflight succeeded on athena: `/usr/bin/node`, `/usr/bin/xvfb-run`,
  and executable `/snap/obsidian/current/app/obsidian` exist; architecture is x86_64.
  This is prerequisite evidence, not an Obsidian walkthrough pass.
- Commit `8a0c2f1` changed **Schema Card** field stepping to Ctrl+J/K. Current
  `docs/albums.md` and `241y-plugin-album-schema-card.js` agree. Record Card field
  stepping remains Shift+J/K. The older parent plan's Schema Card Shift+J acceptance is
  stale: prove the shipped Ctrl+J/K plus digit navigation, and verify Shift+J does not
  write; do not restore a superseded shortcut.

## Binding sources and boundaries

Read through the audited interfaces before implementation:

1. `sase bead read bob-cli-ms.8.3 -r "Need proof scope and prior evidence"` and the new
   child epic/assigned phase. Re-check landed work before editing.
2. `sase artifact read plan:202611/record_card_title_and_proof.md "Need parent proof contract"`.
3. `sase artifact read plan:202611/record_card.md "Need Record Card acceptance and write contract"`.
4. `sase artifact read plan:202611/record_notes_on_demand.md "Preserve shipped Notes semantics"`
   if a discovered defect touches record bodies.

The ancestors' final decisions remain binding: `inline_title = hide`,
`decision_record = no`, and `first_cleanup = now` (already completed). **No memory file
may be edited**, and these decisions are not reopened here. The requested result is
verification and repairs of the existing contract, not new product behavior. Title/stem
rename, in-sheet editing, quick-pick menus, and provenance cleanup remain outside scope
(already tracked by bob-cli-n1/n2/n3/n4).

Open bob-plugins with `/sase_repo` and use only the returned path; read its AGENTS.md.
Do not refer to or reuse another agent's numbered checkout. Edit source fragments,
rebuild generated main.js, keep hand-edited fragments at most 1000 lines, and register
new offline tests explicitly in package.json's test:unit. Split harness helpers by
fixture, DOM probe, and workflow when that keeps each piece understandable; avoid a
general harness refactor. Use the existing startup and album/project CDP mechanisms.

No changes to Cargo.toml versions or CHANGELOG.md. Read cli_rules.md before any new bob
CLI option and docs.md before docs/help changes. Prefer proof tooling in bob-plugins;
touch bob-cli only for a reproduced implementation defect. JSON contracts remain schema
version 1 with additive changes only.

All longer builds, tests, remote runs, and waits use `/sase_monitor`. Wait for the
monitor-start command itself to exit and report Staged, then end through `/sase_final`;
do not wait for the staged successor. Every phase preserves exact commands, source SHAs,
binary hashes, results, and remaining obligations in its bead notes.

## Common harness and evidence contract

Use `scripts/obsidian-album-ui-e2e.cjs`, its `-lib` and offline wrapper for patterns,
`scripts/obsidian-projects-ui-e2e.cjs` for binary/source provenance, and
`scripts/obsidian-record-writer-e2e.cjs` for real-writer fixtures and byte assertions.
The last script is offline writer coverage and cannot substitute for UI acceptance.

Create `scripts/obsidian-record-card-e2e.cjs`, importable helpers as needed, and
`scripts/test-obsidian-record-card-e2e.cjs`. Support the established executable, report,
screenshot output, help, and self-test arguments. Offline tests must run without
Obsidian or a bob installation and verify fixture/probe assertions, including negative
cases that make the assertions fail. Do not let a skipped real run masquerade as a
passing UI check. A partial sheet-only mode can support phase one, but the default final
run must require the full acceptance inventory.

Create a unique scratch Git vault with a local disposable remote and an owned profile,
process, debugging port and independent Xvfb display. Set BOB_DIR and a harmless test
config explicitly; isolate sync and writer commands to that scratch vault. Copy the
authorized source tree and a verified current bob-cli binary to an owned disposable
directory on athena. Use the exact supplied BOB_BIN, record its source SHA and SHA256,
version and record_values_v1 capability, and refuse incompatible/stale binaries rather
than silently falling back to an installed one. Do not open Bryan's live vault in the
test app, use his application profile, modify his checkouts, or kill unrelated apps.
Release only owned resources in success/failure cleanup and preserve reports on error.

Reports distinguish pass, fail and skipped per named assertion, record prerequisite
failures, console exceptions, versions and hashes, and retain DOM paths/computed styles
that support selector decisions. A successful real invocation requires all mandatory
assertions in its selected scope to pass. Missing assertions are failures. Record
Markdown byte hashes separately from application settings and expected scratch writes.
Drive user input through CDP or connected DOM controls; helper method calls may inspect
state or arrange fixtures but must not replace the entry point or save being tested.

Use `/sase_artifact` after reading sase_artifacts.md to register JSON reports, coverage
counts, and key screenshots. Inspect **every** screenshot, record the visual verdict,
and link canonical artifact references in phase notes. Raw live-vault values need not be
published: coverage evidence should contain paths, key names, counts and findings.

If athena becomes unavailable or cannot launch Obsidian, preserve the exact failed
preflight/launch evidence and mark UI acceptance skipped, keep offline tests green, and
record a `PROPOSED FOLLOW-UP:` on the assigned phase. The parent explicitly allows this
fallback, but a hide-selector change still needs a recorded real DOM source; never guess
it from a fake unit-test tree. Product assertion failures with a working host require
fixes, not an infrastructure skip.

## Phase sheet-dom

Build the runner and fixtures, then prove sheet rendering and hide behavior in real
Obsidian on athena. Scope production fixes to `242i`/`242j`/`242k` record-sheet
fragments, styles and their lifecycle collaborators as demonstrated by the run.

Fixtures include an H1-less zettel with Notes; a sparse movie; a canonical restaurant
with legacy keys and wikilink location; a subscription with docked strip and list match;
a missing-required record; a broken-schema album record; a bank/2026-11.md lookalike; a
frontmatter-only record; and a legacy-H1 canonical record. Add explicit legacy noun
type, outside-folder, ordinary note, embed, and hover contexts for negative cases.
Provide schemas covering every field kind used by phase two. Generate album reports
through real bob, not fabricated success payloads.

Capture the actual LP and Reading ancestry of the view's own Properties and inline
title, including embed/popover ancestry. Repair selectors using bounded ownership paths,
or mark exact owned elements if real DOM requires it. Never use a broad descendant hide
that catches nested content. Preserve hideEligible as the authority. Pin the recorded
DOM structure and nested exclusions in a regression test that would fail with the old
selectors, alongside existing gate/lifecycle tests.

Acceptance:

- In Live Preview and Reading, one sheet leads the note body and uses frontmatter title
  as page title, with the established heading role and theme title styling. H1-less
  positive cases show no native inline title or H1 when eligible. The deliberate
  legacy-H1 fixture retains its authored H1; do not delete it through UI rendering or
  fail its transition behavior for that expected extra heading.
- Frontmatter-only and Notes/Source cases mount without duplicates; the editor can reach
  the end of frontmatter and first body line. Field rows, fold and identity collectively
  account for each frontmatter key exactly once, using bob's vocabulary. Subscription
  strip docking occurs once. Broken schema warnings retain their title.
- Light/dark, 375px/wide, long titles, expanded/collapsed sheet and fold are inspected.
  Toolbar wrapping is usable and reduced motion disables motion.
- With native Properties and inline title initially enabled, hide occurs only when
  eligible. Old bob/no capability, broken schema, legacy type, outside folder, Source
  mode and embeds fail open; hover Properties remain unaffected. Test both the native
  element's presence and effective visibility, not the hide class alone.
- File switch, parent/type changes, mode switch, view destruction and plugin disable
  clear the owned hide state and restore applicable native UI. Re-enable and healthy
  return recover without duplicate sheets or leaked classes.
- Render, fold, collapse and navigation leave all scratch Markdown bytes unchanged.

Update docs/develop/validation.md with the runner, isolated host setup, explicit binary
provenance and offline-vs-real distinction. Run focused record-sheet and offline wrapper
tests, real sheet acceptance, the common plugin gates, and deploy under the shared rules
below. Publish the DOM evidence, report and inspected screenshots. Leave interaction
checks explicitly pending for phase two, never marked as complete.

## Phase card-workflows

Extend the same runner with workflows below, using the actual Record Card modal and
writer. Existing code lives in `242l`–`242q` record-update/card fragments, `242e` YAML
excursion, `248m` double-equals, navigation's `551` fallback and Command Mode's surface
catalog. Extend their focused unit suites only where a defect or missing contract case
warrants it. Keep every phase-one assertion running.

- `==` opens the Record Card without indentation changes or an undo-history entry.
  Counted `3==`, Source mode and ordinary/non-record notes retain Vim reindent. Use
  intentionally misindented fixtures so the negative branch is observable.
- Ctrl+Shift+P, palette Edit record, the actual Command Mode row, Edit toolbar,
  row/value/ghost/missing controls and keyboard activation open the expected record,
  field and stage. Observe the resulting selection, not just the presence of a modal.
- A digit status pick performs one logical guarded update: only that status span and the
  album's generated records callout change; all unrelated bytes stay identical. Count
  real writer apply calls independently of preview reads, observe regrouping, then prove
  Undo restores value and grouping.
- Exercise `t`, optional Ctrl+D, and Tab/Shift+Tab save-and-advance across text,
  suggestions, date, number, URL, link and list. Include comma-containing list values.
  Assertions must check persisted bytes and current sheet values, not only notices.
- Make a genuine stale conflict through another pane in Source mode after opening a
  session. Stale, invalid value and required-empty attempts refuse with no additional
  file changes. Compare against the bytes immediately before each attempted write, so
  the deliberate external edit is not misclassified. Verify draft retention.
- `y` enters the YAML excursion and Done returns to the Record Card. `=` opens the album
  Schema Card on the same field; current Ctrl+J/K and digit 3 navigate without writing.
  Record Card Shift+J/K retain their independent behavior, and it has no Schema Card
  keycaps. Do not regress the newer shortcut change described above.
- Hash checks prove opening, navigation, cancelling and folding are byte-neutral. Sample
  sheet values after apply and across metadata-cache refresh: no flash back to old
  values and no duplicate automatic album sync after the guarded write.

Run all scenarios with real BOB_BIN on athena, fix discovered defects with focused
regressions, and rerun affected scenarios until green. Publish the complete report,
writer evidence and inspected card/stage screenshots, document usage changes, run plugin
gates, and deploy. Explicit skipped prerequisite cases follow the common fallback
policy; assertions that fail on a working host remain implementation work.

## Phase coverage-release

Add a reusable **read-only** coverage audit in bob-plugins scripts (for example
`record-card-vault-coverage.cjs`) and a focused offline test. It must never update,
tidy, sync, initialize or repair the live vault. Enumerate canonical records from the
vault independently of successful `records show` calls; cross-check album/folder
enumeration so rejected or misplaced records are counted rather than disappearing. Reuse
an available proper YAML parser; do not treat line-based regexes as a complete
frontmatter parser. Handle quoted and nested values, malformed inputs and duplicate keys
explicitly. Bound process timeouts and report errors per record.

For each record on apollo, invoke the current `bob records show <album> <stem> -f json`
and compare parsed frontmatter keys with keys in record.properties, record.identity and
record.other. Require each authored key to have exactly one home. Optional unset schema
rows are not authored keys and do not count as unexpected duplicates. Record
enumerated/attempted/succeeded/failed totals, albums, key counts, missing/duplicate
coverage, and findings by code. Expected legacy findings require an explicit rationale;
do not turn every warning into an allowlist. Unexplained findings must be investigated
and attributed. Tests cover an omitted key, duplicate home, show failure, malformed
frontmatter, optional absent field and expected/unexpected findings.

Hash inspected Markdown before and after, recording concurrent change as an unstable
read and rerunning the affected read instead of claiming neutrality. Do not copy the
live vault into a synthetic passing fixture. Record the observed count (192 was the
prerequisite's observation, not a constant). Publish aggregate/path/key evidence as
artifacts and put refs in the phase note.

Refresh upstream, rebuild, and run the complete CDP acceptance on the final combined
source with exact bob/plugin provenance. Reinspect all final screenshots. Run the shared
repository gates and deploy; verify the mandatory startup smoke and managed file hashes.
Record expected and observed plugin versions, source SHAs, deployment status and every
exception in one final acceptance report. If any final change is required, rerun the
checks it invalidates before declaring completion.

## Gates, deployment, and completion

After changes to bob-plugins, run npm run build, npm run docs:gen, npm test, npm run
validate, and BOB_CLI_DIR=<current bob-cli checkout> npm run docs:check. Use monitors
for long runs. Do not raise documentation budgets. If bob-cli changes, run focused
regressions, just fix/docs-gen as appropriate and just check through a monitor. The
final integration gate includes bob-cli just check; never check-full.

Known unrelated failures listed by the parent are the full-suite timing budgets
bob-cli-km/io/ki/ju, album schema-writer held-lock test bob-cli-n0, and old contract
count doc tests attributed to bob-cli-mx. Do not fix or re-report those simply for
appearing again. A newly discovered failure that reproduces identically on a clean base
is recorded as `PROPOSED FOLLOW-UP:` on the worker's own bead with evidence and any
existing tracker, and does not prevent closure. A failure caused by this work must be
fixed. Do not create follow-up beads; this child lander relays proposals to
bob-cli-ms.8.3 for the parent epic lander to triage.

For production plugin changes, take the next free minor version at implementation time
and regenerate the inventory. Just before syncing, fetch/merge origin/master and rebuild
so no already-landed work is dropped. Every plugin-repo editing phase must run bob
plugins sync under its AGENTS.md. Preview and deploy with the current linked repo passed
explicitly and --no-pull; follow docs/develop/deployment.md if a protected deploy
refuses. **Never pass -F on your own.** The older co-landing force permission does not
cover this epic. Check landed inclusion first and use its non-force recovery procedure.
Preserve and report any actual blocker.

Complete the required post-sync startup smoke in an owned isolated environment,
including deployed-vault version/hash checks. Distinguish a skipped host check from a
success. Verify all three Record Card participating plugins (ledger-tools,
navigation-hotkeys and command-mode) against the final source/deploy, even if only
ledger-tools changed. Let SASE finalizers commit every modified repository.

The child land agent audits each acceptance item against artifacts and the final
deployed bytes, not just closed child statuses. Once the epic's phases are complete,
close this child epic through the usual assigned-bead completion path, then run
`sase bead epic-symbols bob-cli-ms.8.3`, resolve or re-key any leftovers to a still-open
owner, and close **only bob-cli-ms.8.3** with an evidence note. Closing that explicitly
assigned parent phase is this child lander's authorized completion duty; it is not
permission to close bob-cli-ms.8 or bob-cli-ms, mark their plans done, or do their
remaining landing work. Record the allowed infrastructure skips honestly if used.

The final note/report states exact checks, coverage totals, DOM repair, artifact refs
and deployment outcome. Carry forward the parent lander's Mac checklist: install the
current bob with just install-all before editing is enabled there, then try == on
bead_lifecycle, a status digit pick, claim edit and Undo, and inspect the title in its
sheet. That human check is not a substitute for the automated athena run.
