---
tier: tale
title: Closing a Pomodoro leaves prose bullets behind, with no 🍅
goal:
  Pomodoro closes (bob capture x / x= and Obsidian Ctrl+Enter) carry only dedicated Task
  Link bullets. 🍅 markers are written and kept only on those bullets, by close,
  reconcile, and closed-Pomodoro Ctrl+Enter alike.
size: medium
proposed_by: bbugyi200.athena.1fc
proposed_with: claude/opus
create_time: 2026-11-22 18:21:59
status: wip
---

# Plan: Closing a Pomodoro leaves prose bullets behind, with no 🍅

## What Bryan saw

Screencast `~/tmp/screencasts/20261122_181053.mp4` (vault task `bob#^fix-tomato-issue`,
"Fix tomato issue and stop copying plain bullets!"). Today's ledger had a PLAN session
with one prose child:

```markdown
- [ ] (**1750-1755** [t:: 5m]) — PLAN
  - Planned [[20261122#^rahway-station-ucpac]]!
- [ ] () — PLAN
  - [[job#^calendar-event-dan]]
```

Closing it produced two bugs that Bryan then fixed by hand (`dd`, then `dw`):

1. **The prose bullet was copied.** `- Planned [[20261122#^rahway-station-ucpac]]!`
   showed up in the open `() — PLAN` continuation. It is a note that mentions a task,
   not a Task Link.
2. **A 🍅 landed mid-sentence.** The closed entry's line became
   `- Planned 🍅 [[20261122#^rahway-station-ucpac]]!`.

## Root cause

The close and the marker repair both treat any bullet holding an unstruck block link as
a Task Link. The glossary and `docs/plan.md` say otherwise: a Task Link is a sub-bullet
whose **only** content is the link.

bob-cli (`bob capture x`, the record close `x=`, and `bob task reconcile`):

- `src/native/capture_pomodoro_close/ledger.rs` `classify_sub_bullets` puts every bullet
  with an unstruck plain link into `BulletKind::WorkedOn`. Only the `startable` subset
  (`bare_plain_link`) is a real Task Link. The rest get the `Mentioned` role, yet
  `collect_links_from_bullet` reports them `carried: true`, and both carry loops copy
  every `WorkedOn` line:
  - `plan_record_ledger_close`
  - `plan_ledger_close_with_parked`
- `src/native/capture_pomodoro_close/links.rs` `rewrite_completed_markers`
  (`MarkerPolicy::Completed`) puts `🍅 ` before every non-embedded, non-struck link on
  every line in the closed range. Mixed text is included.
- `src/native/task_status_hooks/pomodoro.rs`:
  - `marker_expected_for_occurrence` and `completed_pomodoro_marker_expected` expect a
    🍅 on every live, non-embedded link under a completed entry, per link and with no
    view of the bullet.
  - So reconcile (`structure.rs`, the `marker_expected` loop) would put the 🍅 back
    after Bryan's manual fix. The same structural planner also runs inside capture-side
    retirement (`!note:id`, `x!N`).

bob-plugins (`task-status-cycler`, the Obsidian Ctrl+Enter close):

- `plugins/task-status-cycler/src/060-pomodoro.js`:
  - `classifyPomodoroSubBullets` pushes every bullet with an unstruck non-embedded link
    into `copyableTaskLinkBullets`.
  - `buildPomodoroCompletionPlan` copies all of them into `copiedBulletLines`.
  - The close rewrites every line with `completedPomodoroMarkerPolicy`, which marks
    every live link.
- `175-plugin-pomodoro-names.js` feeds `copyableTaskLinkBullets` into the
  continuation-name `in_progress` group. Mixed lines therefore steer R6 names in the
  plugin but not in Rust. Rust names come from numbered Task Links only.
- `070-block-links.js`, used when Ctrl+Enter is pressed on a link under a closed
  Pomodoro:
  - `strikeSelectedTaskBlockLinkInLine` with `keepPomodoroMarker` (stale-Done retire)
    adds a 🍅 to any line.
  - `buildRestoredExactStrikeEdit` with `closedPomodoro` (reopen) also adds a 🍅 to any
    line.

## The rule (same in both repos)

A **dedicated Task Link bullet** is a list item, at any depth inside a Pomodoro entry's
sub-bullet range, whose body is exactly one `[[path#^id]]` or `![[path#^id]]` block link
(alias allowed). Before that check, strip in order: the 🍅 markers, one trailing `#`,
and one enclosing `~~…~~` pair. The predicate already exists in both repos for Work Log
routing:

- Rust: `work_log_task_link_target` in `ledger.rs`.
- JS: `getPomodoroWorkLogTaskLinkTarget` in `060-pomodoro.js`.

Any other bullet that contains a block link is a **mention** (the existing "mentioned"
role): prose plus a link, or two links on one line.

- **Carry.** Only dedicated bullets carry: worked-on bare plain links at their original
  indentation, plus `[[…]]#` deferrals as today. A mention stays in the closed session
  as history and is never copied. A continuation placeholder is created iff something
  real carries or no later entry follows (the existing rule), so a close whose only live
  child is a mention creates no continuation when a later entry exists.
- **🍅.** A marker may appear only on a dedicated Task Link bullet. The existing
  per-link rules still apply on those bullets:
  - Live gets a marker.
  - Struck keeps whatever marker it already has.
  - Embedded never gets one.

  On every other line under a completed entry, the expected marker count is zero. The
  close and reconcile therefore **strip** any 🍅 they find there. Reconcile only writes
  the current ledger, so this cleans today's note and never rewrites history. Marker
  repair stays decoration-only.

- Unchanged:
  - x-close numbering ("lines that mix a link with other text … are never numbered").
  - Task-start semantics: mentions are still never started.
  - Nested dedicated links: still numbered, carried at their indentation, and marked, as
    in `nested_worked_on_links_keep_their_indent_when_carried`.
  - Today / `list_queued_links`.
  - Work Log routing.
  - gcal, settle, and coalesce writers. They only write dedicated lines.

## bob-cli changes

1. **Share the predicate.**
   - Move `work_log_task_link_target` into `src/native/capture_pomodoro_close/links.rs`
     as `pub(crate) fn dedicated_task_link(line: &str) -> Option<WikiToken>`.
   - Re-export it from `mod.rs`, next to `bare_plain_link`.
   - Keep `collect_work_log_note_groups` calling it; its behaviour must not change.
2. **Stop carrying mentions** (`ledger.rs`).
   - In both carry loops, push a `WorkedOn` line only when `bullet.startable`.
   - In `collect_links_from_bullet`, report `Mentioned` links with `carried: false`.
     Then `tasks[].carried` in the close JSON and the "carries N link(s)" caption both
     stop counting mentions.
   - Keep `Mentioned` registration as it is today, meaning `role: "mentioned"` rows
     still appear in the report.
3. **Gate the close-time 🍅.** In `rewrite_completed_markers`, keep the current
   per-token policy when `dedicated_task_link(line)` is `Some`. Otherwise apply
   `MarkerPolicy::Strip`.
4. **Gate reconcile's 🍅.** In `src/native/task_status_hooks`:
   - Add `dedicated: bool` to `LinkBullet` (`model.rs`).
   - Compute it in `scan_pomodoros` with the shared predicate on the raw line.
   - Make `marker_expected_for_occurrence` take the bullet (or the flag):
     `entry.completed && bullet.dedicated && completed_pomodoro_marker_expected(link)`.
   - Update the caller in `structure.rs`. Bullets that move to the completed fallback
     follow the same rule, since dedicatedness does not change under retirement or
     marker edits.
   - Stripped mixed-line markers show up in the existing `marker_removed` report.
5. **Tests.**
   - Add the screencast fixture above as a close regression in
     `src/native/capture_pomodoro_close/tests.rs`:
     - The closed entry keeps `\t- Planned [[20261122#^rahway-station-ucpac]]!` byte for
       byte.
     - `carried_lines` is empty, and the later `() — PLAN` block is unchanged.
     - No continuation is created.
     - The mention is still reported as `(Mentioned, …, carried: false)`.
   - Add a variant where the session also has a dedicated `\t- [[bob#^x]]`: only that
     line carries and only it gains `🍅 `.
   - Cover the record policy too.
   - Add a reconcile regression in `src/native/task_status_hooks/tests/`:
     - Under a completed entry, `\t- Planned 🍅 [[20261122#^rahway-station-ucpac]]!`
       becomes the unmarked line with a `marker_removed` item.
     - The unmarked mixed line stays unchanged (idempotent).
     - A dedicated `\t- [[dev#^live]]` still gains `🍅 `.
     - A dedicated `🍅 ~~[[dev#^done]]~~` keeps its marker.
   - Update the existing tests that encode the old mixed-line behaviour. Known ones (let
     `just check` find the rest):
     - `struck_markers_collapse_and_embedded_drop` in `capture_pomodoro_close/tests.rs`.
       The glued `🍅🍅 [[bob#^nospace]]` line is now a mention: not carried, and its
       spaced marker is stripped.
     - The mixed-content marker cases in
       `src/native/task_status_hooks/tests/structure.rs`, around the
       `parses_embedded_alias_and_mixed_block_links`,
       `moves_completed_mixed_bullet_subtree_…`, and `canceled_reference_removal_…`
       fixtures.
     - `tests/cli/task_status_hooks/structure.rs` (`Review 🍅 ~~…~~`,
       `history 🍅 ~~…~~ and …`).
     - Possibly `tests/cli/task_status_hooks/consume.rs`, `sync.rs`, and
       `src/native/task_complete/tests/retirement_tests.rs`.
   - When updating these tests, keep each test's original intent, such as retirement or
     relocation. Change only the marker expectation.
6. **Docs (contracts).**
   - `docs/capture.md`, "Sub-bullet classification": a body that is exactly one plain
     block link is worked-on (kept, one `🍅 `, carried, startable). A line mixing a link
     with other text is a mention: kept as history in the closed entry, never marked,
     never carried, never started.
   - `docs/capture.md`, the "Numbering" paragraph's "mentioned" sentence: add "never
     carried, never marked".
   - `docs/task-status-hooks.md`:
     - In the intro paragraph ("Live non-transcluded links beneath completed Pomodoros
       carry the machine-owned `🍅` marker"), scope the marker to dedicated Task Link
       bullets.
     - In `## Pomodoro Marker`, state the bullet-level rule and its stripping.
     - Change the `mixed content` table row to `Work on [[a#^x]] and ~~[[b#^y]]~~`.
     - Scope the completed-fallback sentence (`🍅 ~~[[...]]~~`) to dedicated bullets.
   - Do not touch `docs/reference/plugins/*`. Those are mounted copies owned by
     bob-plugins.
   - Do not edit `Cargo.toml` version or `CHANGELOG.md` (release train).
   - Run `just docs-check`.

## bob-plugins changes

Open the linked repo with `/sase_repo` (`sase repo open bob-plugins -r …`) and read its
`AGENTS.md`. Edit `src/` fragments only, never `main.js`. Keep each fragment at or below
1000 lines. `060-pomodoro.js` is at 957, so put any helper that would push it over into
another fragment.

1. **`classifyPomodoroSubBullets`.**
   - Put a bullet in `copyableTaskLinkBullets` only when
     `getBareNonEmbeddedBlockLinkTargetFromListItem` matches, which is the same set as
     `startableNonTranscludedTaskLinkBullets`.
   - Classify mixed-text link bullets as `noteBullets`. `pomodoroStruckTaskLinkTarget`
     only matches dedicated struck lines, so names are safe.
   - Result: `copiedBulletLines` drops mentions, and R6 continuation names stop counting
     them, matching Rust's numbered-link names.
2. **Close-time 🍅.**
   - Add a line-level twin of Rust's `rewrite_completed_markers`: if
     `getPomodoroWorkLogTaskLinkTarget(line)` is non-null, run
     `rewritePomodoroMarkersInLine(line, completedPomodoroMarkerPolicy)`; otherwise run
     `stripPomodoroMarkersFromLine(line)`.
   - Use it in `buildPomodoroCompletionPlan`'s rewrite loop.
   - Export it from `210-exports.js`.
   - Existing per-occurrence helpers and exports may stay.
3. **Ctrl+Enter under a closed Pomodoro** (`070-block-links.js`). On a line that is not
   a dedicated Task Link bullet, never add a 🍅:
   - The stale-Done retire (`keepPomodoroMarker`) writes `~~[[X]]~~` with no marker.
   - The restore (`buildRestoredExactStrikeEdit` with `closedPomodoro`) writes `[[X]]`
     with no added marker.
   - Dedicated lines keep today's behaviour.
4. **Tests** in `scripts/test-task-status-cycler-pomodoro-completion.cjs`:
   - Add the screencast fixture through `buildPomodoroCompletionPlan`:
     - `copiedBulletLines` is `[]`.
     - There is no `replaceLine` edit for the `Planned …` line.
     - `createdPomodoro` is `false`.
   - Add a mixed-plus-dedicated variant.
   - Update "Pomodoro completion marks originals and carries clean live copies": its
     `🍅 [[Tasks#^live|Live]] and [[Tasks#^other]]` line now becomes unmarked and is not
     copied.
   - Update "completed Pomodoro markers are normalized per occurrence" only if it goes
     through the new line-level helper.
   - Add a `070` case for the mixed-line strike and restore.
   - Run the closed-Pomodoro-link and names tests
     (`test-task-status-cycler-closed-pomodoro-links.cjs`,
     `test-task-status-cycler-relations.cjs`,
     `test-task-status-cycler-blocks-guards.cjs`).
5. **Docs and release.**
   - `docs/task-status-cycler.md`, "Completing Pomodoros and tasks": only dedicated Task
     Link bullets carry or wear 🍅, and prose bullets that mention a task stay behind as
     history.
   - In the closed-Pomodoro Ctrl+Enter paragraph, scope "adding the marker when it was
     missing" to dedicated bullets.
   - Bump `plugins/task-status-cycler/manifest.json` 1.58.0 → 1.58.1.
6. **Build and deploy.**
   - Run `npm run build`, `npm run docs:gen`, `npm test`, and `npm run docs:check`.
   - Then run `bob plugins sync`. If sync refuses, follow that repo's `AGENTS.md`.

## Verification

- bob-cli: `just check` (fmt, clippy gate, every test binary) and `just docs-check`.
- bob-plugins: `npm run build:check`, `npm test`, `npm run docs:check`, then a
  successful `bob plugins sync`.
- Manual dry run against a scratch copy of the fixture with the locally built binary.
  Never use the live vault.
  - `bob capture --dry-run 'x'` reports no carried link for the mention, and the planned
    day text has no `🍅` on the `Planned …` line.

## Out of scope

- No new CLI flags or JSON fields.
- No change to x-close numbering or selection grammar.
- No change to Today, Event Task clock outcomes, or nested dedicated carry.
- No backfill of older daily notes. Reconcile only writes the current ledger.
- No memory-note changes. No existing memory statement becomes false.
