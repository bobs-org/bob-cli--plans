---
tier: tale
size: medium
title: Repair task-theme pill contract gaps and finish the ledger landing
goal:
  Finish the remaining correctness and integration work in bob-cli-dm.5.1, then close
  that epic and its parent phase bob-cli-dm.5.
proposed_by: bbugyi200.apollo.bob-cli-dm.5.1.land
proposed_with: codex/gpt-6-astra
bead: bob-cli-dm.5.1
create_time: 2026-10-28 19:25:19
status: wip
---

- **PARENT:**
  [202610/ledger_task_theme_mirror.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/ledger_task_theme_mirror.md)
- **BEAD:**
  [bob-cli-dm.5.1](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-dm/bob-cli-dm.5.1.md)

# Task-theme pill landing repairs

## Scope and verified baseline

This is the remaining work from the landing audit of **bob-cli-dm.5.1**,
`plan:202610/ledger_task_theme_mirror.md`. Its parent is the **phase** **bob-cli-dm.5**
in `plan:202610/task_theme_suggestions.md`. Finish the code, verification, and both
permitted closes in this tale's coding turn. There is no later land agent for this tale.
Do not wait for this work's own commit SHA, push, or CI: the host commits after the turn
ends.

Use `/sase_repo` to open **bob-plugins**, then read its `AGENTS.md`. Production changes
belong there, in source fragments; regenerate `main.js`. Keep each handwritten fragment
at most 1000 lines (the current editor and Reading mixins are close to the limit;
extract small helpers where needed). Read the plans with `sase artifact read` and bead
context with audited `sase bead read`.

The lander read all three child beads and every note. Their completed commits in
bob-plugins are `25d864e` (semantic API), `4721a74` (editor/ledger colors), and
`ae40495` (Reading/query rows). Six focused test suites and `npm run build:check` pass
at `ae40495`; `npm run docs:check` also passed all implemented gates (447 documentation
tests passed, one optional font-dependency test skipped). The gaps below were reproduced
separately using the existing test harness against the generated bundle. Existing tests
sometimes encode the wrong behavior, especially the test that expects a pill with no
query provenance. Keep the passing behavior while correcting those assertions.

Final inherited decisions remain: `ledger_theme_pills = yes`, both memory decisions =
no; the optional seed belongs to another phase. Preserve top-level API v3 and `themes`
v1, all old namespace methods, synchronous pure API helpers, byte-preserving string
setters, hidden-theme eligibility, and existing hue precedence. Do not edit memory,
versions, CHANGELOG, real task fixtures, or implement later picker/Card/capture work.
Task fields never change status, freshness, links, history, or Stats.

## 1. Fix the semantic and field-rendering mismatches

The relevant fragments are under `plugins/bob-ledger-tools/src/`:
`279-plugin-task-theme-model.js`, `279-plugin-task-theme-suggest.js`,
`279-plugin-task-theme-marks.js`, and `279-plugin-task-theme-editor.js`.

- `pomodoroThemeInlineFields` calculates value offsets using `match[0].indexOf(raw)`,
  which can find the key instead of the value. In `- [ ] A [theme:: theme]`,
  `valueStart` is 9 rather than 17. Derive the actual capture position and retain UTF-16
  offsets, delimiter/spacing rules, and exact setter bytes. Test repeated key/value
  text, empty values, Unicode titles, both delimiters, and whitespace.
- `pomodoroThemeCanonicalDailyPath` validates month/day against the directory year
  without comparing the filename year. `2025/20261028.md` currently gives a review
  suggestion for `^gtd`. Match Rust's `task_status_hooks::canonical_daily_date`,
  including valid leap dates and directory/filename year equality.
- Live Preview's `line.text.indexOf("theme::")` prefilter skips accepted
  `[theme :: FIX]` and `(theme<TAB>:: FIX)` fields. Use the canonical scanner without
  the stricter literal gate. Render all recognized explicit spans: duplicate fields must
  each be repair, including mixed bracket/paren forms. Preserve per-field
  selection/caret reveal via `dateMarkSelectionFold`, source/code exclusions, and
  closed-task historical marks. Avoid a second parser or duplicated scanner rules.
- Repair model construction currently passes the whole `[theme:: ???]` span as its
  value, so its tooltip contains raw inline-field syntax. Carry the parsed raw value
  into the model and make all tooltip/aria copy safe even for invalid values containing
  colons or delimiters. Preserve original bytes in source/native content. Match PT13's
  exact shared phrases: `reading tasks suggest READ`, `daily review suggests PLAN`, and
  `no default here`.
- Use catalog spelling for known names and the existing PT6 muted classes for
  `hue_source: exempt` or `unconfigured`; configured/pinned hues still win. Do not
  implement another hue algorithm. Unknown and repair remain neutral with their
  respective dotted/dashed treatments. Keep the color toggle independent from field
  visibility.

## 2. Repair rendered provenance, duplicate state, and live refresh

Work in the task-theme Reading, reading-dom, tasks, and editor fragments, with small
lifecycle hooks in existing theme infrastructure as necessary.

- Recover the actual source path and task line from supported Reading, Tasks, and
  Dataview context. Missing `data-line` currently becomes line 1 because
  `Number(null) === 0`; distinguish absence from real line zero. Section start is not
  automatically the task's own line, particularly in a multi-task section or query.
  Normalize backlink paths using existing vault/link helpers and never borrow a
  dashboard's path or line.
- Verify explicit source fields before replacing Dataview inline-field markup. A
  page/default/computed `task.theme` value is insufficient. If query provenance or the
  explicit source cannot be established, leave the native row intact as the approved
  plan requires. Do not manufacture a reliable-looking pill with null origin. Support
  known explicit query rows even when their source note is not open in an editor: use
  available renderer source text or a bounded, deduplicated source-text cache/read,
  preferring matching live editor text. Keep `api.themes.suggestFor` synchronous; any
  asynchronous adaptation belongs in the rendered layer. No subprocess or vault-wide
  read per hover/decoration.
- Determine duplicate/invalid state once for the whole task, across both raw text spans
  and native Dataview fields. Currently two native fields each render `kind=pill`, even
  though the same raw line is duplicate repair. Use the common scanner/model for all
  explicit spans and retain original nodes, links, emphasis, code, nested rows, and
  unrelated fields. Keep split text-node replacement safe and idempotent, with native
  restoration.
- Rendered hover currently invokes editor-only `suggestFor({path,line})` without source
  text. Supply trustworthy source text so unopened source notes get correct inherited
  reasons; unavailable text must not be cached as a false `no default here`. Keep lazy
  hover and clone-safe delegated events, including popouts. Update stale reasons when
  source defaults, catalog/config, or source text change.
- `refreshTaskThemeMarksForCache` has no production caller. Wire task marks to existing
  catalog/journal, Today, metadata, and applicable editor refresh events, with
  coalescing and teardown. A probe calling setup followed by `themesNotifyChanged`
  currently produces zero task refreshes. Test events through production setup, not by
  directly calling the orphan method. Refresh model attributes/classes and invalidate
  enriched tooltip caches; widget equality must not retain an old inherited reason after
  a refresh.
- Track and remove editor hover listeners as well as rendered-root listeners on editor
  destruction/plugin unload; currently editor handlers set an untracked
  `__bobTaskThemeHover` marker. Ensure disabling/unloading cancels queued work, repeated
  enable/render/reload does not duplicate handlers, and native output is accessible when
  the gate is off (the current native wrapper retains `aria-hidden=true` while CSS
  reveals it). Keep retries bounded, and do not permanently suppress a row that later
  rerenders.

Regression coverage must exercise real entry points with realistic source context, more
than one task in a section, an unopened source note, duplicate native fields and mixed
native/raw fields, unavailable provenance, refresh events, hover after a default change,
and disable/unload/reload. Reuse the existing fake DOM and editor harness; improve it
where needed so tests do not hide these failures behind a stubbed suggestion result.

## 3. Integrate current fixtures and verify

The integration audit covered bob-cli through `74874cb5` and bob-plugins through
`ae40495`. Plugin changes since the planning baseline `7f42bb8` included `5b20917`
(relation shortcuts) and `1da19e9` (same-note relation guards); they do not need
task-theme integration. After the first child commit, plugin history contained only this
epic's commits. bob-cli `13b24303` extends theme rename over authored fields without
changing the pure setter boundary; `74874cb5` changes Stats summaries without feeding
the theme API. Preserve those changes. Recheck any newer commits before the final close.

bob-cli `ac13ab54` added eight `preselect` vectors after the plugin copied the fixture.
Copy the **current entire** `tests/fixtures/pomodoro_themes/vectors.json` byte-for-byte
to bob-plugins' `plugins/bob-ledger-tools/scripts/fixtures/pomodoro_themes/vectors.json`
and update provenance README. Preserve every section added since this audit. Keep
executing all `suggest`, `task_field`, and `write_rule` vectors. The preselection
implementation belongs to the outer picker phase; this task only synchronizes its
fixture data.

Run focused theme/config/editor/rendered/ref-display suites, `npm run build`,
`npm run docs:gen`, `npm run build:check`, and full `npm test` in bob-plugins. Update
only affected shipped API/mark docs if needed, following `docs.md` memory. Run
`npm run docs:check` with **BOB_CLI_DIR explicitly pointing to the active bob-cli
checkout** so its script does not create a separate cache clone. Use `/sase_monitor` for
a long command before starting it and place the remaining repairs/closeout in its
continuation; never end with a yielded command still running. Use `sase tool run check`
for the primary `just check` gate and join its monitor if it escalates. Never run
`just check-full`. Independent infrastructure failures are triage, not unfinished epic
work; fix regressions caused by these changes and document exact evidence.

Run `bob plugins sync --repo <opened bob-plugins path>` after changes and record its
result. Manually inspect source, Live Preview, Reading, Tasks, Dataview, and a popout if
an Obsidian instance is available; otherwise report that none of those manual checks was
performed. Do not modify real tasks to create fixtures. The lander performed no manual
Obsidian inspection.

## 4. Finish this epic and the parent phase in the same coding turn

The lander completed follow-up triage in a note on **bob-cli-dm.5.1** before this
proposal: no child `PROPOSED FOLLOW-UP:` entries and no independent actionable issue
were found. The historical load-sensitive performance failure in the dm.5.1.2 close note
was not independently reproduced; the next phase recorded a full passing suite. No task
or +1 was filed on that history. All reproduced defects above remain epic work. Include
this disposition in the close note; use `/sase_new_task` for any genuinely independent
new finding.

After implementation and verification, re-read **bob-cli-dm.5.1** and every
child/descendant with `sase bead read ... -r "Verify landing readiness"` and review all
new notes. Confirm named phases are completed and linked plans are ready. If this tale
is itself an open descendant bead, close the completed tale normally first so it cannot
prevent the epic close. This is same-turn completion, not a dependency on its future
commit.

1. Run `sase bead epic-symbols bob-cli-dm.5.1`. For **every** entry resolve it (wire up,
   privatize, add a legitimate non-test pragma, or delete per Symvision policy), or
   re-key only to a still-open later bead that actually needs the exemption. Check the
   parent phase too with `sase bead epic-symbols bob-cli-dm.5`. Both were empty during
   the audit; repeat rather than relying on that snapshot.
2. Close normally with
   `sase bead close bob-cli-dm.5.1 --note "<verified child implementation, repaired contract gaps, integration drift, checks/deployment/manual limitations, and follow-up dispositions>"`.
   Retry only after resolving any reported stale symbols or unfinished work. Never force
   a successful nested landing, or force merely to pass a guard.
3. Run `just symvision` if the recipe is available, recording if unavailable. Open the
   plans sidecar with `/sase_repo` for writing, use the linked PLAN location from
   `sase bead read` (relative file `202610/ledger_task_theme_mirror.md`), and set its
   frontmatter `status: done`.
4. Run `sase bead read bob-cli-dm.5.1 -r "Need the parent link"`, then re-read parent
   **phase bob-cli-dm.5**. Verify its original ledger-mirror requirements are satisfied:
   semantic mirror/writers/vectors, readingTheme, explicit pills across all promised
   surfaces, and enabled ledger colors. Resolve any remaining phase symbols and close
   **only that phase** normally:
   `sase bead close bob-cli-dm.5 --note "<child plan and verification evidence completing the original ledger-mirror phase>"`.
   Leave containing epic **bob-cli-dm** and its plan status to its waiting land agent.
   Do not close that epic or unrelated sibling phases.

If a real readiness blocker remains, finish it within this plan where bounded; otherwise
record the precise blocker on the affected parent and report it without pretending this
landing succeeded. Submit `/sase_final` for changed repositories after the permitted
closeout, without manually committing or waiting for this turn's SHA/push/CI.
