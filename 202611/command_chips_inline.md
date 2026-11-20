---
tier: epic
title: Complete inline command chips in Live Preview and Reading view
goal: Finish bob-cli-nu.4 with safe source-aware activation, persistent live feedback,
  both inline renderers, and verified visual and Obsidian evidence.
phases:
- id: engine-contract
  title: Complete the engine contract consumed by inline chips
  size: medium
  depends_on: []
  description: 'engine-contract: complete additive resolver metadata, signature lookup,
    per-run state callbacks, loading behavior, and catalog invalidation. Preserve
    existing Command Mode consumers and guards; add focused contract regressions.'
- id: runtime
  title: Shared command-chip state and activation
  size: medium
  depends_on:
  - engine-contract
  description: 'runtime: add source policy, cached resolution, CommandChipRunStore,
    and commandChipActivate with stale checks, a synchronous latch, output deduplication,
    copy-only behavior, and cleanup. Publish shared hooks for renderers and the outer
    fence/keymap phases.'
- id: live
  title: Shared command-chip face and Live Preview rendering
  size: medium
  depends_on:
  - runtime
  description: 'live: add the production inline DOM face and state styling, extend
    the existing viewport scan to both chip kinds, wire gestures and subscriptions,
    preserve copy behavior, and verify mixed-chip performance and rebuild continuity.'
- id: reading
  title: Reading, embeds, hover previews, and query source attribution
  size: medium
  depends_on:
  - live
  description: 'reading: extend the single Markdown processor with accessible command
    buttons and render-child cleanup. Verify Tasks and Dataview source hooks, prevent
    dashboard paths from authorizing imported content, and test stale source and protected
    folders across reading surfaces.'
- id: evidence
  title: Visual previews, isolated Obsidian acceptance, and documentation
  size: medium
  depends_on:
  - reading
  description: 'evidence: add production-backed light/dark previews and an isolated
    stub-bob Obsidian harness with offline self-tests; finish reference docs and run
    required gates. Record deploy and acceptance evidence for the child epic land
    agent; do not close bob-cli-nu.4 yourself.'
proposed_by: bbugyi200.apollo.bob-cli-nu.4
proposed_with: codex/gpt-6-astra
parent_bead: bob-cli-nu.4
create_time: 2026-11-20 07:36:58
status: wip
bead_id: bob-cli-nu.4.1
---

- **PROMPT:** [prompts/202611/command_chips_inline.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/command_chips_inline.md)
- **PARENT:** [202611/command_chips.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/command_chips.md)
- **BEAD:** [bob-cli-nu.4.1](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-nu/bob-cli-nu.4.1.md)

# Inline command chips

This implements phase `inline` of `plan:202611/command_chips.md`, assigned bead
`bob-cli-nu.4`. Required research is
`research:202611/obsidian_command_chips/obsidian_command_chips.md`.

## Outcome and boundary

An authored inline span such as `` `run: bob task reconcile` `` becomes a quiet play
pill in Live Preview and a real button in Reading view. A deliberate click runs through
Command Mode, Shift+Click opens its guarded sheet path, and live feedback survives a
note rewrite. Notes in protected folders, mobile, an absent Command Mode, and query
results with unknown origins copy the payload instead. Invalid or host-unavailable
commands show a reason and never run.

This child epic implements the entire `inline` phase of the approved parent. It does not
implement the parent's `bob-run` card, cursor dispatcher/keycap, authoring card, or
vault adoption phases. It supplies their shared activation, store, and armed-state
hooks. No bob-cli code, CLI options, vault task content, memory files, release version,
or changelog changes belong here.

The parent's decisions are final: Ack becomes a copy chip in adoption; only the
command-chip and copy-chip glossary edits are authorized there; the proposed decision
record was rejected. This child epic makes no memory edits and adds no new reviewer
choices. Parent design wins wherever its research differs.

## Evidence and sizing

The assigned phase was `in_progress` with no prior implementation notes. Its
dependencies are closed, and their work is present in the linked bob-plugins checkout:

- `bf13f43`: rename to Bob Chips.
- `591998e`: Command Mode engine and provenance; manifest version 0.3.0.
- `c8da83f`: command grammar and cursor rule; Bob Chips version 0.7.0.

`021-run-grammar.js` provides `chipScanLine` with copy/run kinds, run parsing and
serialization. `066-command-cursor.js` publishes the pure command namespace. However,
`050-live-preview.js` still calls the copy-only scanner and constructs only copy
widgets; `060-reading.js` parses and replaces only copy code spans. No shared command
activation/store or command face exists. There is no `--epic-symbol` entry for
`bob-cli-nu.4` as of planning; check again before close.

The landed engine needs a bounded compatibility completion before rendering can use the
parent contract. In `057-activation-planner.js`, resolve returns
`{ok, steps, signature}` with steps containing argv, entry_id, and title, but no
original step text, state discriminator, policy, or presentation metadata. `080-api.js`
ignores `runText`'s `onState`, and `activeRun` takes payload text rather than a
signature. It already exposes global start/state/finish events. `090-plugin.js` emits
catalog-changed only when catalog JSON differs, not on every binary fingerprint change.
Do not recreate the runner or reopen nu.2.

An epic is appropriate because the missing engine contract, shared runtime, editor
integration, query-origin integration, and real-app acceptance each have distinct
failure modes. Each phase below is bounded direct implementation work for one medium
coding agent. The dependencies are deliberately serial: they share fragment manifests,
generated entrypoints, package scripts, and docs. No child phase may close nu.4 or nu;
the child epic's land agent owns nu.4's completion after all children finish. It must
never close `bob-cli-nu`.

## Repository and execution rules

All product changes are in the linked `bob-plugins` repo. Each worker opens it with
`sase repo open bob-plugins -r "<specific reason>"` and reads its AGENTS.md. Paths below
are relative to that checkout, so this plan works in any workspace. Read the parent plan
and report through `sase artifact read`, not sidecar files. Before docs edits read the
project's `docs.md` reference memory through the memory skill. Honor the parent's
task/status/walk and panel-dismissal decisions.

Edit fragments listed in `src/fragments.json`, never generated `main.js`; keep
hand-edited fragments at or below 1,000 lines. In particular the current Live Preview
fragment is 777 lines and the engine runner is already near the limit, so put additions
in small cohesive fragments instead of growing them past the limit. Rebuild generated
files after integration. Bump each changed plugin's minor manifest version relative to
the version present when it lands, then regenerate docs inventory. Keep Bob Chips
mobile-capable.

Register new Node test scripts in `package.json` under `test:unit` or `test:perf`. Use
the existing fake Obsidian/CM6 and Command Mode harnesses. Each phase runs relevant
focused tests and the required repo checks: `npm run build`, `npm test`,
`npm run validate`, `npm run docs:gen`, and `npm run docs:check`. Regenerate docs before
drift checks when necessary. Use SASE monitors for long verification commands; wait for
staging commands to exit and then finish through `sase_final`. Commit through host
finalization, never raw git commit. Every edited repo receives a commit declaration.

After changing bob-plugins, run `bob plugins sync` as its AGENTS.md requires. On a
protected-deployment refusal follow that repo's deployment recovery instructions,
inspect upstream, and never force an overwrite. Record exactly what was deployed.
Implementation agents append follow-ups to their own bead; they do not create task
beads. The land agent triages those proposals.

## Contracts shared by every phase

1. Only Command Mode tokenizes argv, consults bob's catalog, or spawns bob. The renderer
   never shells out, expands variables, or reconstructs arguments. Every step is
   validated before the first starts. Authored text cannot supply execution options, a
   program path, policy, or an entry id.
2. Run-now skips previews, keeps the confirm floor, and preserves save-all, writer
   serialization, in-flight deduplication, history, and outcome cards. Guarded opening
   uses `openText`. Accepted command sequences stay fixed even if a step rewrites their
   source note. There is no automatic retry.
3. Run eligibility follows the actual source note. Root folder components `mail`, `ref`,
   `ref_history`, `.bob`, and `_conflicts` are copy-only. Unknown, ambiguous, or invalid
   source paths are copy-only. Reject absolute paths and parent traversal; resolve links
   with vault APIs before applying the rule. A dashboard's own path must never authorize
   a result copied from `ref/`.
4. Rendering, resolution, hover, scroll, catalog refresh notification handling, and
   export are inert. Only a deliberate activation may execute or copy. Command Mode's
   normal catalog load is not initiated by chip rendering.
5. Running a chip never completes/freshens its task or calls walk advancement. Keep
   `api.copyChips` v2, existing copy vectors, copy gestures, and `gy` behavior intact.
   Plain `bob` code spans and unrelated code remain inert.
6. Payloads stay in closures/WeakMaps and text nodes, never `innerHTML`, `innerText`,
   dataset values, or payload-bearing attributes. ARIA names use references to visible
   text nodes. Cleanup is scoped to each mount and plugin.

## Phase engine-contract

Complete the existing additive `api.commandMode` members in the Command Mode fragments
(`057-activation-planner.js`, `080-api.js`, runner/sequence wiring, and catalog
publication), without replacing the engine or changing unrelated sheet behavior.

- Enrich and freeze `resolve(text)` with `state: ready|invalid|unavailable|loading`,
  signature, first failing step/reason, and immutable steps containing index, original
  trimmed step text, argv, entryId, title, effect, policy, presentation, hasPreview, and
  confirm. Retain `ok` and `entry_id` compatibility where existing callers use them. Do
  not flatten original quoted text for faces.
- Complete the parent's command-text cases in the existing splitter: quote-aware `&&`
  and newline steps, empty-step refusal, mandatory literal `bob`, aliases, exclusions,
  missing arguments, host restrictions, and unquoted shell/variable syntax refusal. Do
  not create a second tokenizer. `~` and globs stay literal.
- `runText(text, {source, onState})` re-resolves at activation, waits at most five
  seconds only for an already in-flight catalog load, and returns a clear refusal on
  failure. Render-time resolve remains synchronous and side-effect free. Whitelist
  source/onState; note text or callers cannot override execution policy.
- Deliver guarded `onState` callbacks for queued, running, step, and finished, carrying
  runId, step, total, outcome and enough error detail for failed ink. Forward through
  runner and sequence callbacks; do not infer queued/running from a resolved Promise. A
  throwing callback cannot affect execution. One sequence finish is distinct from each
  constituent run finishing.
- Normalize stage outcomes for confirm/review/output without manufacturing a running
  state for a card that has only opened. Keep existing consumers' result fields
  compatible. Preserve report streaming and guarded previews.
- Add `activeRun(signature)` support while retaining the shipped text-input form. Return
  state/kind and sequence progress for signature lookup. Search actual immutable
  sequence steps/signatures, not display-only snapshots, so matching still works between
  steps. Repeated activation opens live output.
- Notify `onCatalogChanged` on the initial usable catalog and fingerprint changes even
  if catalog JSON is identical. Include a stable invalidation token (fingerprint) and
  keep unsubscribe behavior. Existing cached startup catalogs must be usable without
  waiting for an event.

Tests pin the new metadata and frozen values, callback order and isolation, catalog
wait/timeout, the identical-catalog/new-fingerprint case, active sequence lookup between
steps, and no duplicate execution. Exercise report, run, preview-first, confirm, and
multi-step outcomes. Existing sheet and sequence suites must pass; preserve their
existing semantics. Extend `docs/develop/plugin-apis.md` with actual result/callback
shapes.

## Phase runtime

Add cohesive Bob Chips fragments for source policy, cached command resolution,
`CommandChipRunStore`, and `commandChipActivate(chip, ctx)`. Wire plugin setup and
unload in `090-plugin.js` and export test helpers through `099-exports.js`.

Use one plugin-owned store keyed by the engine's canonical signature. Each entry holds
state, runId, sequenceId, step, total, outcome/reason, and timer/latch data. Expose
snapshot, subscribe/unsubscribe, state update, and armed-target operations as shared
internal helpers for the parent fence and keymap phases. Armed identity also includes
source path and span identity: equal commands in different notes share run state but
must not all appear armed. Document these hooks and their coordinate conventions; do not
add key bindings or advertise an unimplemented keycap. Keep the existing frozen pure
`api.commandChips` v1 members working.

Cache resolve results in a bounded map keyed by engine instance, catalog
fingerprint/generation, and exact payload. Seed from the current catalog and invalidate
on catalog change or engine replacement/disable. Cache neither a stale loading verdict
forever nor a source-policy verdict across note moves. Subscribe once per plugin rather
than once per scan or rebuild.

Activation context contains the actual source path, zero-based source line, origin kind,
current view/element, source snapshot/range, and a fresh source reader/checker supplied
by its renderer. The common activation path:

1. Re-evaluate source eligibility, mobile state, and engine capability. A copy-only
   activation copies the exact payload with existing copy feedback; it never calls
   runText/openText. Alt+Click is inert in this phase.
2. Recheck the displayed source against current source. A changed/disappeared span
   refreshes and requires a fresh gesture. Live Preview compares the exact document
   slice including delimiters; Reading supplies section/source evidence where available.
   Do not guess a match from identical text elsewhere.
3. Resolve again through the engine for activation (not just cached paint data).
   Invalid/unavailable yields one Notice with the reason and no execution. Loading may
   use the engine's bounded in-flight wait, never render-time spawn.
4. Take a synchronous per-signature latch before any asynchronous execution boundary,
   held at least 400 ms and through pending acceptance. Coalesce concurrent stale-source
   checks and fast gestures; recheck catalog/source before dispatch after an
   asynchronous read. If already active, open its output; if acceptance is pending
   without a run id, swallow the duplicate.
5. Call runText with source and onState, or openText for Shift+Click/Shift+Enter. Update
   the store from real engine events/results. Confirm/review/output handoff never leaves
   a fake spinner. A rejected or throwing API settles the latch/store and reports one
   useful failure without retrying.

Completed is green for about 1.6 s then Ready; attention and failed persist until the
next run or plugin reload. Queued/running clicks open output. A stale timer from an
earlier run cannot reset a newer run. A rebuilt mount immediately sees the current store
state. Dispose subscriptions/timers on unload; unmounting one widget cannot cancel the
shared run or erase another widget's feedback.

Use deterministic tests for all transitions, duplicate signatures across mounts,
simultaneous activation during async source reads, a sub-400-ms completion, shift
opening, exceptions, rebuilding mid-run, timer races, and source/mobile/ engine policy.
Document the internal contract consumed by live/reading and the outer fence/keymap
phases. Neither task nor walk API is called.

## Phase live

Add the shared production `.bob-run-chip` DOM builder and painter (new fragments beside
`040-chip-dom.js`), then integrate it into `050-live-preview.js`. The production
discovery pass calls existing `chipScanLine` once and dispatches by kind; preserve
existing copy-only helpers where tests/API consumers need their old shape. Retain
viewport limits, syntax-tree shielding, source reveal when selection touches the span,
table normalization, and fence/frontmatter/math/ comment exclusions. Do not register a
second scanner/decoration pass for run.

Use the shared builder for both later Reading buttons and editor widgets. Resolve source
path from the widget's own editor info, not the globally active leaf. Store exact source
ranges and bytes. Mount subscribes to run state and updates glyph/classes in place;
destroy unsubscribes. No re-created spinner for each step or global periodic editor
refresh. A document rebuild mounts from the same store and reflects the in-progress run
immediately.

Appearance follows the parent exactly:

- Copy-pill geometry: padding `0 .45em`, `.92em` font, inherited line height,
  `--radius-s`, cloned box decoration, wrapped untruncated text, fixed glyph slot.
  Resting background mixes accent at 7% with secondary background; border mixes accent
  at 25% with border. Theme variables, no fixed colors or per-chip colors.
- Label faces use the text font and a faint tabular `· N` for multiple steps. Literal
  faces use monospace, original resolved step text, and a faint `›` separator replacing
  only later steps' leading bob words. Quoted `&&` is text, not a step boundary.
  Loading/copy-only fallback displays the verbatim payload.
- Ready play; armed accent play/border; queued hourglass/dashed accent; running loader
  and 2px segmented done/current/upcoming underline; completed green check and one
  yank-flash pulse; attention orange play; failed error play; unavailable ban/dashed
  muted; copy-only copy glyph. Loading uses a faint play. No state changes line width or
  height. Failed tooltip includes the first error line.
- Tooltips contain full payload and policy text as DOM text. Describe run-now, skipped
  preview, confirm, or output honestly, plus Shift+Click opening. Preserve the parent
  hotkey wording design but only advertise a working keyboard binding after the outer
  keymap phase provides it. No Alt+Click edit promise yet.
- Shared polite announcements, visible keyboard focus, reduced motion without
  spin/pulse, and plain inline-code print/PDF presentation. Reading accessibility uses
  action text plus face via aria-labelledby, with action matching Run/Copy.

Primary clicks/taps, Enter, and Space activate; Shift+Click/Shift+Enter open. Suppress
secondary buttons, cancelled gestures, releases outside, drags at or above the existing
4px threshold, and duplicate native keyboard click events. Do not move the editor caret
on activation. Alt gestures remain inert for run chips until the parent's author phase.
Existing copy gestures remain identical.

DOM tests cover every face, text safety, accessibility, in-place updates, both kinds on
one line, stale source, gestures, and rebuild continuity. Extend
`scripts/test-copy-chips-live-preview-performance.cjs` with a 2,000-line mixed fixture
under the existing 35ms CPU budget, viewport-only discovery, and assertions that
identical text resolves once per catalog generation. Measure the actual shared build
path, not a parallel toy scanner; do not relax budgets. Update manifest description to
name both kinds and add inline behavior docs.

## Phase reading

Extend `060-reading.js`'s existing processor to dispatch copy/run without letting the
old processed marker consume a run span first. Preserve raw code exclusions,
idempotence, table handling, and the existing copy path. Command controls are
`button type=button` instances from the shared builder. Native Enter/Space and
Shift+Enter must activate once. Subscribe/update in place and tie listeners, feedback,
and subscriptions to `MarkdownRenderChild` teardown; never add a document-wide
MutationObserver.

Resolve provenance separately from payload parsing:

- Normal Reading, embeds, and hover previews use the actual source supplied by their
  render context; never substitute the active note. Retain section info and source
  offsets for stale checks when available. Missing section metadata alone need not
  disable a known ordinary source note, but a known mismatch always refuses and
  refreshes.
- Detect Tasks/Dataview result context before trusting ctx.sourcePath, which can refer
  to the dashboard. Resolve the nearest result's owning source from verified plugin
  metadata/backlink hooks, not an arbitrary link in its text. Link aliases, headings,
  and relative paths go through vault resolution.
- Inspect the installed Tasks and Dataview versions and real rendered DOM before
  selecting adapter selectors. Follow repo/vault access instructions and open any
  additional source repository with `sase_repo`; never fetch repo files outside that
  workflow. Capture small sanitized DOM fixtures and version notes. Verify each
  known-source hook against those versions in an isolated scratch vault. Unknown,
  ambiguous, missing-backlink, or unresolvable results copy only with reason "source
  note unknown". Do not treat a nearby dashboard link as proof.
- Recheck origin and available fresh source at activation; an embedded/protected source
  never inherits an allowed dashboard's privileges. Copies preserve payload bytes and
  reuse existing success/failure feedback.

Tests form a source matrix over ordinary note, embed, hover, Tasks, and Dataview: each
protected folder; allowed source; unknown source; source renamed into a protected
folder; stale/disappeared content; mobile; Command Mode absent, disabled, loading, and
unavailable. Assert both runText and openText call counts, byte-exact copy, one refusal
Notice, and zero spawn/copy while rendering. Record actual query-hook evidence. Update
the reference's run/copy/refuse table.

## Phase evidence

Create `scripts/preview-command-chips.cjs` following `scripts/preview-command-mode.cjs`.
Load production builder/painter fragments and styles into Chromium, with deterministic
store snapshots. Cover every state, labeled and literal sequences,
copy-only/loading/refusal, mixed copy/run, wrapping, narrow widths, reduced motion,
focus, and print. Capture and visually inspect light/dark PNGs; register screenshots
with `sase_artifact`. Screenshots demonstrate actual production rendering, not separate
mock markup.

Add an isolated stub-bob harness following `obsidian-copy-chips-e2e.cjs`, its
`-lib.cjs`, offline test wrapper, and `obsidian-command-mode-tab-e2e.cjs`. Use a
temporary profile/vault/config and a fake executable selected explicitly for Command
Mode; prevent fallback to the real bob binary. The stub returns a fixture catalog for
`__catalog`, records argv and source/run evidence, provides successful/attention/failing
outcomes, and can rewrite only its scratch note. Fixture setup/self-tests must prove all
filesystem and execution paths are isolated. Do not run this harness against the live
vault.

Real-app acceptance covers Live Preview and Reading, embeds/hover, and the
installed-version query adapters:

1. A click invokes exact expected argv once. Double-click and a second mounted instance
   cannot create duplicate runs. No checkbox/freshness/walk changes.
2. A three-step inline sequence reaches one summary; attention continues; failure at
   step two prevents step three and leaves persistent failed ink.
3. A stub rewrite rebuilds the note widget during a pending run; feedback and
   click-to-output survive. Stale-source activation performs no run.
4. Shift opens the guarded path; confirm commands do not run before acceptance; reports
   open output. Cards do not leave a chip pretending to run.
5. A copy span beside a bob-copy card still copies. A ref/ chip copies its exact
   payload; protected embeds/query results do likewise from an allowed dashboard.
   Unknown query origins copy; unavailable commands refuse.
6. Render, scroll, hover, and export produce zero stub command invocations beyond
   Command Mode's explicit catalog handshake.

Run with `xvfb-run -a` locally when possible. Planning found no Obsidian/xvfb/ Chromium
command on PATH on apollo; implementation must recheck availability, then follow tailnet
memory and the repo's harness runbook to use athena if needed. Read tailnet/obsidian
memory before remote/vault work. If neither host can execute the real-app harness,
record exact attempts and the limitation; the parent explicitly allows that limitation,
not a fabricated pass. Run and report offline tests and available visual/DOM evidence
regardless.

Finish `docs/bob-chips.md`: command syntax, labels/sequences, what click and Shift+Click
do, source policy, all states, accessibility/print, and the explicit attack-surface
bounds (prefix, catalog, confirm floor, no shell, protected source folders, desktop-only
execution). Preserve grammar already documented; remove claims that command spans remain
inert when they now render. Do not claim the outer keymap, fence card, or authoring
feature is shipped ahead of its phase. Keep the exact API/internal contracts in
`docs/develop/plugin-apis.md` and observe docs word budgets. Generate inventory, run all
required checks, deploy, and attach a concise acceptance matrix with screenshots and
harness results.

## Landing and completion evidence

The child epic land agent verifies all five phases are closed and their linked repo work
is integrated. Rebuild generated files after any merge and verify the final shared APIs
are available for `bob-cli-nu.5` (fence) and `bob-cli-nu.6` (keymap). Their work must
not be implemented or closed here. Verify the final Bob Chips/Command Mode versions with
`bob plugins list` and sync without force. Report the reload requirement to Bryan.

Acceptance is the parent's complete inline matrix, including mixed-kind perf, actual
source attribution, rebuild continuity, and honest real-app evidence. Run
`sase bead epic-symbols bob-cli-nu.4` immediately before close. Resolve any remaining
symbols or re-key their Justfile lines to a genuinely still-open owner.

Known dependency-phase notes report pre-existing docs published-URL failures and
load-sensitive 25ms ledger tests (tracked by bob-cli-io and bob-cli-km). Those notes are
leads, not proof that a current failure is unrelated. If a required check fails,
reproduce the same failure on the clean base; record the precise evidence and existing
tracker in a `PROPOSED FOLLOW-UP:` note on the assigned phase. A verified identical
baseline failure does not leave a finished phase open. Do not weaken checks or silently
omit feature failures.

Only after all descendants and this inline scope are complete, the child epic's land
agent closes its own child epic, then closes **only `bob-cli-nu.4`** among the original
epic's beads with
`sase bead close bob-cli-nu.4 --note "<verified implementation, checks, artifacts, deployment, and limitations>"`.
This is the delegated completion of nu.4 requested by the user. Child phase workers only
close their own beads; they never close ancestors. Leave `bob-cli-nu` and every other
original phase to their designated agents. Finish each turn through `sase_final`, with
`keep` for plans/intermediate work and `close` only for fully verified assigned scope.
