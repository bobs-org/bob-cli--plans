---
tier: epic
title: Bank views and safe editing in Obsidian
goal:
  Finish bob-cli-kd.6 with month and queue cards, a daily chip, safe bank-row editing,
  styled rows in both Obsidian views, and verified plugin deployment.
phases:
  - id: bank-ui-data
    title: Bank JSON data service and shared presentation models
    size: medium
    depends_on: []
    description:
      "bank-ui-data: implement the bank JSON boundary, cached reads, invalidation,
      source-location helpers, and synthetic fixtures in bob-plugins. Follow the
      bank-ui-data section."
  - id: bank-ui-gestures
    title: Safe bank gestures, commands, and typing helpers
    size: medium
    depends_on:
      - bank-ui-data
    description:
      "bank-ui-gestures: implement exact-selection acknowledgment, open-today and
      acknowledge-day commands, Enter continuation, category suggestions, and bank-row
      guards in ledger-tools and task-status-cycler. Follow the bank-ui-gestures
      section."
  - id: bank-ui-cards
    title: Month and queue cards with the daily bank chip
    size: medium
    depends_on:
      - bank-ui-gestures
    description:
      "bank-ui-cards: render bob-bank month and queue fences and the daily bob-plan chip
      using the shared data and action services. Follow the bank-ui-cards section."
  - id: bank-ui-marks
    title: Bank row and day-heading rendering in both Obsidian views
    size: medium
    depends_on:
      - bank-ui-cards
    description:
      "bank-ui-marks: add Live Preview and Reading view bank decorations, mark gating,
      bank-property hiding, and dollar-sign math protection. Follow the bank-ui-marks
      section."
  - id: bank-ui-proof
    title: Fixture walkthrough, documentation, and deployment proof
    size: medium
    depends_on:
      - bank-ui-marks
    description:
      "bank-ui-proof: verify the combined bank UI in a synthetic vault, finish plugin
      documentation, run the required gates, deploy, and publish evidence for the child
      epic land agent. Follow the bank-ui-proof section; do not close any ancestor bead."
proposed_by: bbugyi200.athena.bob-cli-kd.6
proposed_with: codex/gpt-6-astra
parent_bead: bob-cli-kd.6
create_time: 2026-11-11 14:27:32
status: wip
---

- **PROMPT:**
  [prompts/202611/bank_obsidian.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/bank_obsidian.md)
- **PARENT:**
  [202611/bank_daily_acknowledgment.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/bank_daily_acknowledgment.md)

# Bank views and safe editing in Obsidian

## Objective and delegation

Finish the complete `bank-obsidian` scope of **bob-cli-kd.6** from
`plan:202611/bank_daily_acknowledgment.md`. Opening a bank month note should show a
statement-like month card, readable rows, and useful day summaries. The hub should show
exactly what is waiting, and daily notes should provide a quiet route into that day's
bank note. Bank rows must remain ordinary acknowledgment checkboxes through every
affected task gesture.

This is a child epic, not another single-agent tale. The original phase has no prior
work notes. Inspection found the CLI dependency shipped at `49218b38`, with subsequent
bank fixes at `757e36bb`; bob-plugins at `99ccca1` has no bank fragments or tests. The
scope combines asynchronous data services, two cards and a daily host, two editor
renderers, and cross-plugin mutation guards. Each phase below is bounded to one coding
agent. Dependencies are intentionally serial because these changes share the fragment
manifest, lifecycle registration, generated entrypoints, and stylesheet.

Each child phase closes only its own assigned bead. The child epic's land agent verifies
the whole delegated scope and closes **bob-cli-kd.6**, as authorized by the original
request, after resolving its epic symbols. It must not close **bob-cli-kd** or
**bob-cli-kd.7**. The parent epic's guide, real-vault bank audit, subscriptions
follow-up, and memory work remain with bank-wrapup.

## Accepted boundaries

- The parent decisions are final: `accounts=checking`,
  `subscriptions_scope=discretionary`, `memory_bank_glossary=yes`, and
  `memory_bank_decision=no`. This child epic edits **no memory**. It introduces no new
  account, subscriptions work, or reviewer decision.
- Read the parent plan and the UI sections of
  `research:202611/bank_daily_acknowledgment/bank_daily_acknowledgment.md` with
  `sase artifact read`. Open bob-plugins with `/sase_repo` and read its `AGENTS.md`. If
  the research artifact is missing in a fresh workspace, first open the configured
  research sidecar with `/sase_repo`, then retry the audited artifact read.
- Work in bob-plugins source fragments; never edit generated `main.js` directly. Keep
  each hand-edited fragment at or below 1000 lines, add it to `fragments.json`, and
  install its mixin/export through the existing mechanism.
- Read current history before implementation and preserve any landed albums or other
  concurrent plugin work. Resolve source conflicts, then regenerate `main.js`.
- All monetary totals, category ordering, waiting counts, and account states come from
  `bob bank list|report -f json`. Do not duplicate accounting rules in JavaScript.
  Syntax recognition for decoration, typing, and locating a source checkbox is allowed.
  Never write derived amounts, summaries, charts, or links into notes.
- No new CLI option or schema is required. No release version or changelog edits. Schema
  1 account fields may initially be null/unverified; render that honestly and accept the
  balance phase's populated fields when present.
- Use synthetic data in fixtures, captures, logs, and bead notes. Do not run bank
  commands against Bryan's real transactions for UI evidence. `bob plugins sync`
  deployment remains required by bob-plugins instructions after source changes.
- Discovered unrelated work goes in a `PROPOSED FOLLOW-UP:` note on the assigned child
  phase, with evidence for the land agent. Do not create task beads.

## Implementation contracts

### bank-ui-data

Use `241-plugin-reading-board-data.js`, notebook data fragments, and the stats error
model as patterns. Add focused bank fragments and
`scripts/test-ledger-tools-bank-*.cjs`.

1. Decode `schema_version: 1`, `ok`, `teaching_empty`, and errors before reading fields.
   Support list `days[].rows`, `counts`, `categories`, `accounts`, and report `period`,
   `totals`, `breakdown`, `days`, `waiting`, and `flagged`. Preserve integer cents and
   null fields; malformed JSON or a schema mismatch never becomes a zero-dollar success.
   Validate finite safe integer amounts; use text nodes for payees and labels.
2. Read through `bobSpawnJson`, using the existing bob-path resolver and explicit vault
   path. Cache by command and exact argv; coalesce same-key requests, retain last-good
   results during refresh, and prevent a late response from overwriting a newer
   generation. A stale/error state must remain visible rather than claiming freshness.
3. Expose one bank service for list/report reads, refresh subscription, and teardown.
   Mounts subscribe with their date/month requirements. Unmount cancels listeners;
   unload must prevent queued work from repainting dead views.
4. Invalidate on `bank.md` and canonical `bank/YYYY-MM.md` metadata changes, creation,
   deletion, or rename; focus, visibility, and local date rollover; successful ensure or
   ack; and the `bank` tag in `283-plugin-command-mode-refresh.js`. Unrelated vault
   edits do not trigger bank subprocesses. Debounce bursts and bound the cache.
5. Add a pure source recognizer that requires a canonical bank month path, a valid day
   section, and a top-level checkbox amount beginning with required `$` (optional
   bold/sign). Respect fences, indented children, non-day sections, and empty stubs.
   Preserve source spans rather than serializing the whole row. Reuse the Rust grammar
   fixtures as evidence; do not treat an arbitrary dollar sign as a bank row.
6. Discover existing canonical month paths through the vault file catalogue; request
   their monetary values from report JSON. Never infer existence from report's full
   calendar of days: absent headings can have fallback line 1. Navigation must resolve a
   real heading in the current source before moving the cursor.

Tests cover success/teaching-empty/error shapes, unavailable bob, timeout, schema
mismatch, refresh deduplication, multiple keys, out-of-order completion, filtered
invalidation, local midnight, unload, and exact cent formatting. Synthetic fixtures
include empty notes, two months, transfers, cash, inflows, and account caveats. Hook
tests into the root npm test command and keep generated docs inventory current.

### bank-ui-gestures

Implement the shared action service before rendering clickable cards.

1. `bank-open-today` / **Bank: open today** invokes `bob bank ensure -D today -f json`,
   opens its returned path, and places the cursor at the returned next-row position
   (JSON is 1-based; editor positions are 0-based). Reuse the same action with an
   explicit date for daily-chip clicks. Do not pass `-o` and independently open a second
   leaf. Handle pending editor saves and reused views with the existing
   external-write/reload conventions; preserve undo history.
2. `bank-ack-day` / **Bank: acknowledge this day** locates the day section under the
   cursor and snapshots the waiting IDs it contains. Cards pass their own rendered
   snapshot. Invoke `bob bank ack <IDs...> -f json`, never `ack -D`, never recompute a
   wider set at click time, and never submit an empty selection. Deduplicate IDs, lock
   repeat clicks while pending, show failures, and revalidate after success.
3. ID-less typed rows are valid. An individual queue checkbox may use the ordinary
   source-checkbox edit, changing only `[ ]` to `[x]` after verifying the exact
   displayed row and current day context through the editor/vault guarded-write API.
   Refuse ambiguous, moved, or changed snapshots and refresh; do not invent IDs. Bulk
   acknowledgment includes only the displayed IDs and reports any remaining ID-less rows
   as needing their individual checkbox. It never reports an entire day acknowledged
   while such rows remain. No fallback to `ack -D` is permitted.
4. In bank month notes only, Enter at a complete row's end inserts `- [x] $`; Enter on
   an empty `- [x] $` or `- [ ] $` stub clears it. Preserve ordinary Enter in code,
   nested notes, non-row checklists, nonempty selections, and mid-line edits. `[cat:: `
   suggestions use JSON categories in declared order with emoji/label and insert only
   the key. Do not give either command a default hotkey.
5. Audit `task-status-cycler` commands, counted/Vim paths, completion handlers, and
   lower-level editor writers, plus ledger-tools snippet/task gesture entry points.
   Recognize bank rows before task promotion, freshness stamping, completion fields, or
   status cycling. Complete toggles only `[ ]` and `[x]`; other task-only gestures are
   handled without mutating bank rows. Protect amount-like invalid-status rows and stubs
   from further task mutations without silently repairing them. Preserve normal tasks
   and checklists outside bank rows. The cycler's guard must remain safe when
   ledger-tools is disabled; use a small standalone recognizer with the same fixture
   vectors and parity tests if shared build code is impractical.

Test late-arriving IDs, unknown IDs, duplicate clicks, exact byte preservation, edited
and ambiguous ID-less rows, cursor-day selection, suggestions, and Enter behavior.
Exercise actual command and writer paths with spies proving that completion, task tags,
freshness, and task recovery APIs are never invoked for bank rows. Audit any delegated
navigation path encountered by those commands and guard it at the boundary as needed.

### bank-ui-cards

Register `bob-bank` with `view: month` or `view: queue` through the normal Markdown
render-child lifecycle. Invalid fence options show a helpful local error. Use existing
area hues (`data-root-hue`, `--bob-area-<hue>`), tabular numerals, small labels, a 2px
accent start border, and a 14% accent background. Stack secondary chips at narrow
widths; use native accessible buttons/links, visible keyboard focus, and text labels for
states.

**Month card:** select the month from the month note's frontmatter, validate it against
the canonical filename, and show the period title plus `so far` when supplied. Show
account status, Out/In/Net, waiting and flagged counts, category bars in report order
with amount/share, and the cash-out `destination unknown` label. Display unclassified
amounts explicitly. The waiting pill jumps to the first waiting row. The Monday-first
heat strip contains every day, uses quantiles of that month's positive daily Out (zero
separate, equal values equal intensity), outlines today, marks waiting days, and
navigates to existing day headings. Test all-zero and tied distributions. No click may
silently navigate to frontmatter via a fallback line 1. Render an empty-month teaching
state. Footer links are previous existing month, Bank hub, and next existing month;
missing adjacent months are not dangling links.

**Queue card:** show waiting rows by oldest day first, with line links and individual
checkboxes wired to the action service. Each day's acknowledge button carries exactly
its rendered waiting IDs. Show `nothing to acknowledge` only with the account caveats
still visible; zero waiting is not proof of complete import coverage. Footer includes
account statuses and last import, the weekly import command, and up to six most recent
existing months with Out values fetched through report. Do not scan the entire vault or
launch unbounded subprocesses to assemble this footer.

**Daily chip:** mount once inside a canonical daily note's rendered `bob-plan`, after
the day strip. Model the lifecycle on `239-plugin-day-strip-plan-host.js`; it must work
when the day strip is disabled and must not duplicate across rerenders. Show that date's
`out $… · in +$…`; only today's note appends the all-month waiting count. Clicking
ensures that exact date and jumps there. Hide the chip when `bank.md` is absent. Add a
persistent default-on bank-chip setting using the plugin's existing configuration
mechanism; cache it outside paint paths. Error data is not displayed as zero. Keep the
chip subordinate to the day strip, and independent of row-mark visibility.

Tests cover month models, quantiles, navigation/existence, account caveats, queue
snapshot actions, six-month bounds, missing hub, historical/today chips, configuration,
remount/unload, and narrow-pane layout. Capture fixture renders for later integration.

### bank-ui-marks

Use `279-plugin-root-marks.js` and the mirror in `266-plugin-mark-gate-mirror.js`.
Separate pure tokenization from CodeMirror and Reading-view application.

- Decorate canonical bank rows only. Recognize amount variants including bold, thousands
  commas, one/two decimal places, `+`, `-`, and Unicode minus; parse tail spans in any
  supported order. Test both `⚑`/`!!` and `—`/`--`, links, custom block IDs, and
  imported IDs. Never consume indented children, code, or ordinary prose.
- Fixed-width right-aligned tabular amount column; positive amounts green with `+`;
  category label chips; muted account chips; flag chip; faint `bank ✓` replacing only
  imported `^bank-…`; 2px area-hue bar for waiting rows; slight dimming for
  acknowledged. Retain functional native checkboxes and links. Unknown
  categories/accounts stay visible as literal labels instead of disappearing.
- In Live Preview, use `Decoration.replace`/marks, skip code, and reveal raw syntax on
  the cursor line. Do not change Source mode or note bytes. In Reading view, use source
  positions and safe DOM construction, preserve links/nested content, and make repeated
  processing idempotent.
- Each real day heading receives report-provided Out/In and a waiting badge. A real
  empty day heading shows muted `nothing posted`; a missing day is not an empty heading.
- `toggle-bank-marks` and a persistent default-on bank-marks setting control row and
  heading decorations together, mirrored into popouts. Disable/unload restores raw
  content. Cards and the daily chip remain independently available.
- Hide only the `bank` frontmatter property row on the root `bank.md`. Do not hide the
  property in any other note, and restore it when leaving that file or unloading.
- Prevent bank-row dollar amounts from being interpreted as math in both views,
  including two dollar amounts across nearby rows. Verify the actual rendering pipeline,
  not just a tokenizer. Any protection must be scoped to recognized bank rows, preserve
  source bytes, and leave ordinary math elsewhere intact. Keep this protection effective
  when decorative marks are toggled off.

Tests cover all grammar vectors, selection/cursor reveal, repeated Reading processing,
code exclusion, custom IDs, heading locations, settings and popouts, and note-byte
preservation. Add a real Obsidian fixture case demonstrating no math rendering.

### bank-ui-proof

Finish integration, not a new product phase. Rebase or integrate concurrent plugin
changes before regenerating entrypoints. Verify all contracts above using a disposable
synthetic vault and the repository's existing Obsidian smoke/E2E harness; never use
production bank contents as fixtures. If native UI execution is unavailable, retain the
runnable harness and explicitly record the unverified cases rather than claiming them
passed; resolve that limitation before claiming complete UI proof.

The walkthrough must demonstrate:

1. Month card and queue card in both views, including a narrow pane and account caveat.
2. Today's and a historical daily chip, absence without a hub, and correct ensure jump.
3. Row and day-heading styling, cursor reveal, mark toggle, popout, and intact links.
4. Enter continuation and stub removal, category suggestions, no inline math, and
   unchanged behavior for ordinary tasks and math outside the bank month note.
5. Single-row and bulk acknowledgments; a new row arriving after draw stays waiting; an
   ID-less checkbox edits only its original row; failed/stale actions stay honest.
6. Cycler, complete, counted/Vim, and ledger-tools task gestures never attach task
   metadata or non-bank statuses. Confirm the guard also works without ledger-tools.
7. Metadata refresh, local day rollover, focus, schema mismatch, timeout, missing bob,
   stale data, and view disposal produce no false success or duplicate chip.

Update `bob-plugins/docs/bob-ledger-tools.md` and the cycler reference as needed:
fences, commands, no default hotkeys, settings, ID-less acknowledgment semantics, and
guards. Generate the command inventory; do not hand-edit mounted copies in bob-cli. Read
`docs.md` memory first. Update relevant API documentation only for APIs actually added.

Run `npm run build`, `npm run build:check`, `npm run docs:gen`, `npm test`,
`npm run validate`, and `npm run docs:check` in bob-plugins. Follow repository
validation instructions and include `just docs-check` in bob-cli when validating the
mounted docs. Each earlier phase runs its targeted tests plus required repository checks
and syncs its changes; this phase supplies the complete integrated gate evidence.
Long-running commands use `/sase_monitor`; wait for the staging command to exit and
finish through `/sase_final` rather than leaving a live subprocess behind.

Deploy with `bob plugins sync`. On protected-deploy refusal, follow the linked repo's
`AGENTS.md` and `docs/develop/deployment.md`: fetch and reconcile landed work; never
force over another agent's deployment. Publish screenshot or text captures for every
view, test and deployment outputs, and known limitations as an audited artifact, with no
real financial data. Record the artifact ref on the assigned phase bead and make it
available to the child epic's land agent and parent bank-wrapup.

## Land acceptance and close protocol

The land agent reviews every requirement above, integrates all phase commits, verifies
the final built plugin deployment and required checks, and gives the parent wrap-up a
concise evidence reference. Reproduce unrelated check failures on a clean base;
identical pre-existing failures become `PROPOSED FOLLOW-UP:` evidence and do not prevent
closure. New failures or missing required behavior must be fixed before closing.

Run `sase bead epic-symbols bob-cli-kd.6` before closing it (the planning inspection
found none). Resolve remaining symbols or re-key their Justfile entries to an actually
open bead. Each child phase follows the same protocol for its own ID. Close bob-cli-kd.6
only after all child work and the integrated verification are complete, with a note
naming the report and what was verified. Do not close the original parent epic or
bank-wrapup. Every root turn finishes through `/sase_final`.
