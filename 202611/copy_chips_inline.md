---
tier: tale
title: Inline copy chips in Live Preview and Reading view
goal:
  Render and copy inline chips with exact payloads, stable feedback, a shared cursor
  API, and verified deployment for bob-cli-ni.2.
size: medium
proposed_by: bbugyi200.apollo.bob-cli-ni.2
proposed_with: codex/gpt-6-astra
bead: bob-cli-ni.2
create_time: 2026-11-19 11:17:05
status: wip
---

- **PARENT:**
  [202611/copy_chips.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/copy_chips.md)
- **BEAD:**
  [bob-cli-ni.2](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-ni/bob-cli-ni.2.md)

# Inline copy chips in Live Preview and Reading view

Complete assigned phase **bob-cli-ni.2** of **bob-cli-ni**, implementing the `inline`
phase of `plan:202611/copy_chips.md`. A valid authored code span becomes a quiet copy
chip; clicking it copies its literal payload without moving the caret or modifying the
note. The cursor command and the versioned API resolve the same chip and use the same
clipboard and feedback path.

## Starting evidence and boundaries

- Read the phase and parent design through
  `sase bead read bob-cli-ni.2 -r "Need current phase scope and notes"` and
  `sase artifact read plan:202611/copy_chips.md "Need the accepted inline design"`. Also
  read `research:202611/obsidian_copy_chips/obsidian_copy_chips.md` with
  `sase artifact read`; its A1–A10 and R1–R14 requirements are accepted. The epic's more
  specific API signatures and phase boundaries take precedence over the earlier research
  sketches.
- The phase is in progress with no notes recording a previous implementation attempt.
  Its dependency bob-cli-ni.1 is closed. In bob-plugins, commit `0b38f84` supplies
  plugin version 0.1.0, the pure grammar, frozen API shell, shared JSON vectors, and a
  5,000-case round-trip test. Both grammar scripts passed during planning: 9 tests, no
  failures. Recheck the current tree before editing, and preserve any newer landed work.
- `plugins/bob-copy-chips/src/fragments.json` currently contains only `010-load.js`,
  `020-grammar.js`, `090-plugin.js`, and `099-exports.js`. There is no renderer,
  feedback implementation, cursor resolver, or command. Registration in chezmoi and the
  vault was completed by the grammar phase; do not repeat that work.
- This is one medium tale because it extends one existing plugin and one shared runtime
  contract, with established CM6, DOM-test, CPU-test, and CDP patterns to adapt. No
  child epic or helper agents are needed.
- Open **bob-plugins** with `/sase_repo`, use only the path it prints, and read its
  `AGENTS.md`. All authored source, CSS, tests, plugin versioning, and docs changes
  belong there. Do not edit generated `main.js` by hand; keep each hand-edited fragment
  at or below 1,000 lines.
- Leave the fence card and block resolution to bob-cli-ni.3, the Copy card `C` row to
  bob-cli-ni.4, and the make-or-edit card, Alt+Click edit behavior, vimrc mappings, and
  vault adoption to their later epic phases. Keep APIs extensible for those consumers
  without implementing their features here.
- No bob-cli code or mounted-doc edits, no CLI or capture grammar changes, and no Mac
  Capture changes are needed. No memory edits in this phase: the final epic decisions
  prohibit the copy-chip decision record and authorize only `glossary:copy-chip`; its
  addition belongs to adoption.

## 1. Shared runtime, DOM, and feedback

Extend the guarded imports in `010-load.js` for the Obsidian and CM6 APIs actually used.
Follow bob-emoji's guarded imports and ledger-tools' `010-load-and-constants.js`.
Missing editor APIs may disable Live Preview without breaking Reading view or the pure
API.

Add `030-feedback.js` and `040-chip-dom.js`, and list them in source order.

- `copyChipWrite(text)` calls and awaits `navigator.clipboard.writeText` directly within
  the activating gesture and returns `{ok: true}` or `{ok: false, reason}`. Missing
  clipboard access, synchronous exceptions, and rejected promises become a useful
  failure reason. Empty/whitespace-only input never overwrites the clipboard. Do not
  trim, interpolate, HTML-decode, or otherwise change a valid payload.
- `copyChipBuildInline(doc, chip, {interactive})` builds the shared face for Live
  Preview, Reading view, and the future inert author preview. Use `bob-copy-chip`,
  `is-labeled`/`is-literal`, and a fixed-width `bob-copy-chip-glyph` slot with Obsidian
  `setIcon` Lucide glyphs.
- Label and face text use `textContent` only. Retain the clipboard payload in closures
  or a WeakMap, never `innerHTML`, `innerText`, `dataset`, or payload-bearing
  attributes. A labeled tooltip contains `Copy: ` plus a maximum 120-character preview
  with newlines shown as `⏎`; an unlabeled tooltip says `Click to copy`. Construct
  tooltip text safely as DOM text, so payload `::` cannot become an attribute that
  Dataview rescans.
- Reading controls are real `button[type=button]` elements with the accessible name
  `Copy <face>` and native Enter/Space activation. To satisfy both this accessible-name
  requirement and the ban on payload attributes, use `aria-labelledby` referencing a
  visually hidden `Copy` prefix and the visible face when necessary; do not duplicate a
  literal payload into `aria-label`. Verify the resulting accessible name in the real
  app.
- Successful fulfillment sets `is-copied`, swaps the glyph to a green `check` for about
  1.2 seconds, pulses once, and announces `Copied` through one polite live region per
  owner document. Success has no Notice. Failure sets `is-failed`, shows
  `alert-triangle` in `--text-error` for about two seconds, and emits exactly one
  `Couldn't copy: <reason>` Notice.
- Repeated activation resets feedback safely. A stale timer or late promise must not
  overwrite newer feedback or mutate a destroyed widget. Dispose timers, event
  listeners, tooltip nodes, and live regions when their widget, render child, or plugin
  unloads. Use ownerDocument for pop-out windows.
- Inert preview construction must not register clipboard gestures or write anything.
  Neither rendering nor copying writes the vault.

Implement the epic's look in `styles.css`, using the dependency-chip tokens in
`plugins/bob-ledger-tools/styles.css` as the local reference: theme variables, hairline
border, padding `0 .45em`, font size `.92em`, inherited line height, faint copy glyph,
accent hover/focus, `cursor: copy`, and a visible keyboard focus outline. Unlabeled text
is monospace, labeled text uses the text font. Use inline wrapping with
`box-decoration-break: clone`, no face truncation, and no feedback width changes. The
pulse uses `--bob-yank-flash-bg` with the approved `rgba(80, 200, 120, .35)` fallback.
Reduced motion retains the glyph change and removes the pulse. Print/PDF styles show the
face as inline code without interactive chrome.

## 2. Live Preview and Reading view

Add `050-live-preview.js` with a guarded `ViewPlugin` and `WidgetType`. Use
`registerEditorExtension` only when the required APIs are available.

1. Gate decoration on `editorLivePreviewField`; Source mode is raw Markdown. Inspect
   actual Obsidian syntax nodes early in the real-app harness rather than assuming
   upstream Markdown node names match Obsidian's parser.
2. Visit inline-code syntax nodes only within `view.visibleRanges`. Recover the whole
   code-span range including equal-length delimiters, normalize its raw source with
   `copyChipNormalizeSpan`, then parse with the existing grammar. Infer table context
   from the document syntax. Never replace the runtime parser with a regex scan over the
   whole document.
3. Exclude links and wikilinks, fenced/indented code, math, comments, frontmatter, and
   physical multiline spans. Confirm container support for lists, tabs, blockquotes,
   callouts, and tables. Do not rescan or decorate the same span twice when visible
   ranges overlap.
4. Replace the whole eligible span unless **any** selection touches its closed boundary
   interval `[from, to]`; reveal that span's source then. Test empty cursors, both
   boundary positions, nonempty selections, and secondary cursors. Rebuild on document,
   viewport, selection, syntax-tree, and Live Preview mode changes, without cross-mode
   DOM state.
5. Widget equality compares label, text, and position. Widgets own their DOM; all editor
   changes go through CM6 transactions. `ignoreEvent()` returns true and primary
   `mousedown` prevents caret relocation.
6. A primary-button release inside the same chip copies only when total movement stayed
   below 4 px. Track the gesture rather than treating every click event as permission: a
   drag that returns to its start, cancellation, release outside, or a secondary button
   must not copy. Support touch taps and native keyboard activation without a duplicate
   clipboard write.

Add `060-reading.js` and register one Markdown post-processor. Process the root code
element as well as code descendants, skip anything within `pre`, `a`, an internal link,
or an existing chip, parse renderer-normalized `code.textContent` once, and replace
valid code with the shared button. Do not perform CommonMark normalization twice. Make
repeat processing idempotent and scope listeners through render-child/plugin lifecycles.
Embeds, hover previews, and Markdown rendered in Tasks/Dataview containers should use
this same path; no document-wide MutationObserver. Leave `=` and `$=` Dataview spans and
neighboring Bob marks untouched.

## 3. Cursor resolver, public API, and command

Add a small pure cursor/context fragment before `070-commands.js` if needed, keeping
editor adaptation separate from the grammar. Implement `chipAt({lines, line, ch})` from
live buffer lines, not metadata cache.

- Coordinates are zero-based lines and UTF-16 character offsets. `from`/`to` in an
  inline chip are line-relative offsets including the delimiters, with `to` exclusive,
  matching `scanLine`. A cursor contained in `[from, to)` wins. Otherwise the only
  eligible chip on that line wins; multiple eligible chips yield `ambiguous`; zero yield
  `none`.
- Return frozen results `{state, chip?, count}`. A ready chip is frozen
  `{form: "inline", label, text, line, from, to}`; count is the number of eligible chips
  on the line. Invalid inputs safely yield none/count 0.
- Skip fenced lines, including their opening and closing lines and nested backtick/tilde
  fences. Match `copyCardLineIsInFence` and the related helpers in
  `bob-navigation-hotkeys/src/101-copy-card-model.js` without depending on that plugin
  being enabled. Apply the same inert-context and table rules as the renderer. A
  document-context pass is acceptable for an explicit command; it must never run in
  every visible-range rebuild.
- At this phase boundary, all fence positions return none. Phase ni.3 adds the block
  fallback after the inline rule. Keep this extension point clear in code and API docs.

Extend the existing frozen `api.copyChips` namespace without breaking its already
shipped helpers:

| Member                           | Contract                                                |
| -------------------------------- | ------------------------------------------------------- |
| `version`                        | `1`                                                     |
| `parse`, `scanLine`, `serialize` | Existing grammar behavior                               |
| `parseFenceInfo`, `fencePayload` | Preserve the existing additive helpers                  |
| `chipAt({lines, line, ch})`      | Frozen none/ambiguous/ready result above                |
| `copy(text)`                     | Promise of `{ok, reason?}`, never throws/rejects        |
| `copyChip(chip, {view?})`        | One clipboard write plus success/failure feedback       |
| `flash(chip, {view?})`           | Boolean indicating success feedback was shown; no write |

Guard every public method, including optional/missing views and unloaded plugins. Avoid
matching widgets by text alone: two identical payloads at different positions remain
independent. `copyChip` and `flash` locate the matching widget in the supplied editor
view; a revealed source span uses a short-lived `StateEffect`/mark-decoration flash in
`--bob-yank-flash-bg`. Map or clear transient ranges on edits, and clean their timers on
unload. Do not make a failed visual lookup turn a successful clipboard write into a
false failure.

Register `copy-on-cursor-line`, named `Copy chip on cursor line`, with an
`editorCallback` and no default hotkey. Use the current editor buffer and cursor,
resolve through `chipAt`, then call the shared copy/feedback path. Notice
`No copy chip on this line` for none and `N chips on this line · move onto one` for
ambiguity. The command works in Source mode and with revealed source as well as rendered
Live Preview.

## 4. Verification and documentation

Extend `scripts/copy-chips-test-helpers.cjs` with isolated Obsidian/CM6/DOM stubs and
fake timers, following existing ledger-tools and Copy card tests. Add focused scripts
for cursor/API, feedback/DOM, Live Preview, and Reading view behavior; add each to
`test:unit`. Reuse `scripts/fixtures/copy-chips-vectors.json` as the shared payload
oracle.

Cover independently observable behavior:

- All vector payloads, labels, independent chips, escaped pipes, backticks, Unicode,
  leading/trailing spaces, ampersands, HTML-like strings, and `::`.
- Inert contexts, invalid/empty payloads, mode transitions, overlapping ranges,
  selection reveal, rebuild triggers, and no note mutations.
- Cursor precedence and ambiguity, UTF-16 positions, table inference,
  nested/mismatched/unclosed fences, API immutability, and safe failures.
- Clipboard promise fulfillment/rejection and unavailable clipboard, one-write gesture
  handling, timers/repeated feedback, unload disposal, no source payload in attributes,
  tooltip truncation without payload truncation, accessible naming, and Enter/Space
  behavior.
- Source-range flashes and exact widget targeting; `flash` performs no clipboard write.
  Inert preview performs none either.

Create `scripts/test-copy-chips-live-preview-performance.cjs` in `test:perf`, using
`scripts/cpu-work-perf-support.cjs`. Exercise a 2,000-line daily note with 200 chips,
assert correct decorations and bounded traversal of visible syntax ranges, and use a 35
ms CPU-work budget following the existing 200-link off-clock chip benchmark. Include a
narrow viewport assertion that would fail a whole-document scan.

Create `scripts/obsidian-copy-chips-e2e.cjs`, its `-lib.cjs`, and an offline self-test
wrapper in `test:unit`, adapting the existing Copy card CDP harness. Use an isolated
scratch vault/profile and an owned Obsidian process under `xvfb-run -a`; never open the
live vault in the app. Read the actual clipboard over CDP and assert source fixture
bytes remain unchanged. Keep machine-readable reports and useful screenshots outside the
repo.

The real-app gate must exercise:

1. Live Preview clicks without caret movement; drags and cancelled/nonprimary gestures
   without copying; cursor-in-span reveal; Source mode; normal → insert → normal Vim
   transitions.
2. Reading buttons by click, Enter, and Space; an embedded note; a hover preview or
   equivalent Markdown-rendered child; repeated post-processing.
3. Every inline conformance-vector line in LP and Reading, with exact clipboard
   agreement. Construct a real table header/delimiter for the table vector. Test each
   chip on a multi-chip line independently.
4. Empty and whitespace payloads, forbidden contexts, Dataview `=`/`$=`, and clipboard
   rejection with the alert glyph and exactly one Notice.
5. The cursor command, ambiguous refusal, widget and source feedback, reduced motion,
   print styling, long wrapping faces, and clean unload.

This planning host has Node but no `xvfb-run` or Obsidian on PATH. A read-only SSH probe
confirmed **athena** has `/usr/bin/node`, `/usr/bin/xvfb-run`, and `/snap/bin/obsidian`.
Read `tailnet.md` with `/sase_memory_read` before remote work. Stage the already-opened
repo's built plugin and harness dependencies into a fresh temporary test bundle there,
with matching source hashes; do not read or alter another checkout or the live vault to
obtain fixtures. Execute the isolated harness against those exact bytes and collect the
report. Use `/sase_monitor` for long commands, including remote e2e, and wait for its
staging command to exit. An offline self-test is not evidence that the real-app gate
passed.

Update bob-plugins `docs/bob-copy-chips.md` with the implemented inline gestures, cursor
rules, look/feedback, syntax fallback, exclusions, exactness contract, and non-goals (no
secret store, variables, or protocol handler). Avoid promising the fence card or author
card before their phases land. Document the namespace and coordinate/return conventions
in `docs/develop/plugin-apis.md`. Read the project docs memory before editing docs. Bump
only bob-copy-chips from 0.1.0 to 0.2.0 (or the next minor if a newer version has
landed) and regenerate inventories.

Run `npm run build`, `npm run docs:gen`, the focused tests, `npm test` (including the
added CPU case), `npm run validate`, and `npm run docs:check`. Build and regenerate
before checking generated files. Use the real-app test above and the deployment smoke
gate below as separate acceptance evidence. Do not run bob-cli's Rust suite solely for a
linked plugin change.

## 5. Deployment and closing the assigned phase

Honor bob-plugins' required deployment after changes and the epic's commit-before-deploy
order. Repository commits belong to the host through `/sase_final`; do not manually
commit. If the order needs a turn boundary, stage the deployment/verification through
`/sase_monitor`, declare the linked repo commit with the assigned bead kept open, and
let the monitored continuation finish deployment and closure after the finalizers
settle. Do not end with an unmanaged command still running.

From the opened bob-plugins checkout, preview and deploy only this plugin with
`bob plugins sync --repo "$PWD" --no-pull -p bob-copy-chips` (add `--dry-run` for the
preview). Follow `docs/develop/deployment.md`, including its mandatory isolated
post-sync smoke check against the deployed source hashes and versions. Honor the
overwrite guard: fetch and use the documented landed-deploy recovery, wait through
`/sase_monitor` for an active owner, and never pass `-F` without explicit approval.
Report the actual deploy and smoke verdicts and tell Bryan to reload Obsidian after
successful sync.

Record completion evidence on **bob-cli-ni.2**: touched plugin version, API
availability, focused/full/performance tests, real-app report and source hashes, docs
checks, and deployment/smoke results. Any discovered unrelated work is a
`PROPOSED FOLLOW-UP:` note on this bead, never a newly created bead. A check failure
reproduced identically on a clean base gets that note (including any already-tracking
task ID) and does not keep this phase open; retain precise base and changed-tree
evidence.

Immediately before closing, run `sase bead epic-symbols bob-cli-ni.2`; planning found no
entries, but recheck. Resolve each remaining symbol or re-key it to a still-open
parent/later phase. Once the phase is implemented and verified, close **only**
`bob-cli-ni.2` with
`sase bead close bob-cli-ni.2 --note "<what was implemented and verified>"`. Never
manually set its status, reopen it, or close bob-cli-ni or another ancestor. Finish the
root turn through `/sase_final` with every changed repository accounted for.
