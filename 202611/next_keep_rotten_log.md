---
tier: tale
title: Next keeps ask for an optional Rotten Log reason
goal:
  Marking a Next task fresh asks why it stays in Next and logs a typed reason under its
  Rotten Log, like the Pending prompt.
size: medium
decisions:
  blank_reason:
    ask: What should a blank Enter write when a Next task is marked fresh?
    choices:
      skip:
        Nothing extra; the task is stamped and only a typed reason adds a Rotten Log
        entry
      shrug:
        A `🤷 no reason given · kept in Next` entry on every Next keep, like counted
        keeps
    default: skip
    why:
      You asked for an optional entry; daily Next reviews would bury real reasons under
      🤷 lines
    answer: skip
  memory_decision_records:
    ask:
      Should a new Next-keep decision record replace the Next exclusion in
      rotten-keeps-are-logged?
    memory:
      - decisions:rotten-keeps-are-logged
      - decisions:next-keeps-ask-an-optional-rotten-reason
    default: true
    corrects:
      NEW, early/FRESH, Pending, Next, PROJECTS, recurring, checklist, closed, and
      unblock-tickler keeps never prompt and never write it
    answer: true
  memory_rotten_log_glossary:
    ask:
      Should the Rotten Log glossary term be rewritten to cover optional Next-keep
      reasons?
    memory:
      - glossary:rotten-log
    default: true
    corrects:
      NEW, early, Pending, Next, PROJECTS, and unblock-tickler keeps never write it
    answer: true
proposed_by: bbugyi200.apollo.bt
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-01 10:28:40
status: wip
---

# Next keeps ask for an optional Rotten Log reason

## Goal

When Bryan marks a Next (`[*]`) task fresh with Alt+F or Ctrl+Alt+F, ask once "Why keep
it in Next?" and record a typed answer as a dated entry under the task's
`🥀 **ROTTEN LOG**`. This mirrors the existing Pending (`[/]`) refresh prompt: the
answer is optional, Enter always keeps the task, and Esc cancels the refresh.

Note on the existing behavior: the Pending prompt writes a **Work Log** summary
(`🛠️ **WORK LOG**`, "What did you get done?"), not a Rotten Log entry. Pending stays
exactly as it is. A Next task has not started, so it gets a keep reason ("why is this
still Next?") in the Rotten Log instead of a work summary.

## Current state (read these first)

bob-plugins, `plugins/bob-navigation-hotkeys/src/`:

- `520-plugin-lane-links-and-review.js` `refreshTaskFreshnessOnTasks` handles single and
  counted `N<Alt+F>` keeps. After the checklist, recurring, decision, and
  `classifyFreshStampTarget` filters, it builds `eligibleTargets` (Pending, through
  `isPendingWorkLogTargetRawLine`) and `rottenTargets` (counted Ready ROTTEN/TICKLER
  keeps). An if/else ladder then chooses one of four prompts through
  `requestFreshnessRefreshAnswers`.
- `532-plugin-freshness-link-refresh.js` repeats the same ladder for Task Link sessions.
  It always uses the batch-form prompt.
- `192-freshness-refresh-summary-modal.js` `FreshnessRefreshSummaryModal` has three
  forms: the single `keep` form (it hosts `RottenKeepReasonStage`), the batch form
  (`keepTargets` plus an optional Work summary field), and the plain Work summary form.
- `195-rotten-keep.js`: `classifyRottenKeepTarget`, `planFreshStampBatchWithLogs` (stamp
  plus Work and Rotten Log entries in one plan), `buildRottenKeepPromptModel`,
  `RottenKeepReasonStage`.
- `030-bullet-blocks-and-logs.js` (974 lines; fragments are capped at 1000):
  `formatRottenLogEntryText`, `parseRottenLogEntryBullet`, `readRottenLog`,
  `planManagedTaskLogEntry`.
- `010-requires-and-config.js`: `ROTTEN_LOG_ENTRY_RE`, which requires
  `· kept (\d{1,3})×`.
- `220-property-targets.js`: `isPendingWorkLogTargetRawLine`.
- `480-review-jump-and-nav-api.js`: the Fresh toast already appends
  `· Rotten Log updated` / `· N Rotten Logs updated` from `rottenLogWrittenCount`.

bob-plugins, `plugins/bob-ledger-tools/src/135-freshness-rotten-log.js` mirrors the
entry grammar byte-for-byte in two places: `FRESHNESS_ROTTEN_ENTRY_RE` for source lines
and the DOM-text regex in `freshnessMarkRottenLogSummaryFromListItem`. Both feed the
`🥀 N keeps logged · last <date> — <reason>` freshness-mark tooltip line.

bob-cli Rust (`src/native/capture_pomodoros_agenda.rs` `tag_log_subtrees`,
`capture/sub_bullet.rs`) and Bob Mac Capture's `🥀 N` chip recognize only the marker and
tag every line beneath it. They never parse the `kept N×` tail, so **no Rust or Swift
change is needed**.

## Behavior

### 1. Eligibility

Add `isNextRottenLogTargetRawLine(rawLine)` next to `isPendingWorkLogTargetRawLine`:
`getObsidianTaskCheckboxStatus(line) === "*"` and `isObsidianTaskLine(line)`.
Eligibility depends only on status, just as for Pending. It covers any accepted `[*]`
stamp target, whether the task is due in the NEXT tier or still early. It also covers
rows reached by `]w` and Task Link targets.

Apply it to the same accepted-target list that Pending eligibility uses (`keepTargets`
in 520, `stampTargetsByGroup` in 532). That list has already dropped checklist,
recurring, at-limit, and refused rows. A Next target is never a counted keep, because
counting requires `lane === "ready"`. Defensively, a line that is in the rotten set is
never also in the lane set.

Out of scope, with no prompt: the Task Card picker's refresh row, the Alt+P and
scheduling Work Log prompt, the Alt+[ / Alt+] Task Link lane toggle, Pending (which
keeps its Work summary only), and every automation path.

### 2. Entry grammar

A Next keep entry reads `*YYYY-MM-DD* — <reason> · kept in Next`. It uses the local
review date and sits newest-first under the single `🥀 **ROTTEN LOG**` direct child.
That child is created as the task's last child when missing, and reused otherwise (same
placement as counted keeps through `planManagedTaskLogEntry`).

- `formatRottenLogEntryText({ date, reason, keeps, lane })`: when `lane === "next"`,
  emit the lane tail instead of `· kept N×`.
- Parsers accept either tail. Parsed entries carry
  `{ date, reason, keeps: N | null, lane: null | "next" }`. The lane token is exactly
  `Next`; anything else (such as `kept in next` or `kept in Pending`) is not an entry.
- Update both nav `ROTTEN_LOG_ENTRY_RE` / `parseRottenLogEntryBullet` and both
  ledger-tools regexes in the same change. The ledger-tools file header says these must
  not diverge.
- Every display that prints `kept ${keeps}×` must print `kept in Next` for lane entries.
  That means the `buildRottenKeepPromptModel` history rows (also used by the decay
  card's EARLIER KEEPS) and the batch preview rows.
- The "N keeps logged" count includes lane entries. It stays display-only: nothing reads
  it to change decay, `[keeps::]`, lanes, tiers, or freshness.
- `[keeps::]` is unchanged. Next stamps stay uncounted through `keepLine`, so the streak
  is preserved exactly as today.

> [!decision] blank_reason = skip Only a nonblank reason writes an entry.
> `formatRottenLogEntryText` returns `""` for a blank lane reason, and the writer skips
> that target.

> [!decision] blank_reason = shrug A blank reason writes
> `*YYYY-MM-DD* — 🤷 no reason given · kept in Next`
> (`SCHEDULE_LOG_SKIPPED_REASON_TEXT`), exactly like counted keeps. Then every Next keep
> writes an entry, ambiguous markers disable submit as they do for counted keeps, and
> the hints and placeholder below use the counted-keep wording (`↵ logs 🤷`,
> `Keep & log`).

### 3. Single prompt (one requested `[*]` target, no count)

Reuse the single `keep` form of `FreshnessRefreshSummaryModal` /
`RottenKeepReasonStage`, and add a lane mode to `buildRottenKeepPromptModel`
(`options.lane === "next"`):

- Header 🥀 with the title `Keep in Next?`.
- Subtitle line 1: `<task title> · NEXT · <age>d`. The age comes from the exact queue
  entry when one matches, otherwise from `[fresh::]`, and it is omitted when unknown.
- Subtitle line 2: the logged count only (`none logged yet` / `N keeps logged` /
  `log unreadable`). There is no streak text, because Next keeps don't count.
- Placeholder: `Why keep it in Next? (optional · ↵ to skip)`.
- Blank preview: a `No entry` row and `Refresh only; no Rotten Log entry`.
- Nonblank preview: the exact entry text, the existing destination line, and the
  existing `::` warning.
- EARLIER KEEPS: up to three of the newest entries in either form.
- Footer and button labels: `Keep` / `Keep & next` while blank, and `Keep & log` /
  `Keep, log & next` with a reason. The `& next` variants apply when `advance`
  (Ctrl+Alt+F) is set.
- Ambiguous or unknown history (several markers): a blank Enter still keeps and writes
  no entry. With a nonblank reason, submit is disabled and a meta row reads
  `Rotten Log unreadable; clear the reason to keep without logging`.
- Esc writes nothing: no stamp, no entry, no walk advance. The existing note-changed and
  day-changed guards and `rottenKeepPromptPending` apply, as does swallowing Alt+F /
  Ctrl+Alt+F inside the input.

### 4. Counted `N<Alt+F>` batches and Task Link sessions

Next targets join the existing batch `Keep reason` field. One prompt and one Enter
submit everything. Work summary stays first.

- Add `kind: "counted" | "lane"` to each `keepTargets` item. Lane items carry `raw`, the
  exact queue `entry` when one matched, and `history` from `readRottenLog`.
- The Keep reason field is shown when any counted or lane target exists. The Work
  summary field is shown when any Pending target exists.
- Placeholder: when any counted target exists, keep the current text
  (`Why keep them as written? (optional · ↵ logs 🤷)`). For lane targets only, use
  `Why keep them in Next? (optional · ↵ to skip)`.
- Preview rows: counted rows are unchanged. A lane row reads
  `<title> · Next · N logged`, with meta set to the entry text or
  `no entry (blank reason)`. The destination line counts only the targets that will get
  an entry with the current reason.
- Submit is disabled when any counted target's history is ambiguous (as today), or when
  the reason is nonblank and any lane target's history is ambiguous.
- The single prompt (section 3) applies only when 520 sees exactly one requested `[*]`
  target outside a counted session. Task Link sessions keep using the batch form, as
  they already do for counted keeps.

Replace the duplicated if/else ladders in 520 and 532 with one pure helper. For example,
`planFreshnessRefreshPrompt({ singleKeep, rottenTargets, workTargets, laneTargets })`
returns
`{ kind: "none" | "single-rotten" | "single-lane" | "work-only" | "batch", showWorkField, keepTargets }`.
Put it in a new fragment such as `src/197-freshness-refresh-prompt-plan.js`, registered
in `src/fragments.json` after `195-rotten-keep.js`. This keeps 520 (825 lines) and 532
under the 1000-line cap, and both call sites stay in step.

### 5. Writer

Extend `planFreshStampBatchWithLogs` with `options.laneLines`. For each entry in
`plan.stamped` (bottom-up, deduplicated, as today) whose line is in `laneLines`, is not
in `rottenLines`, and whose `entry.before` passes `isNextRottenLogTargetRawLine`:

- Build the entry with
  `formatRottenLogEntryText({ date, reason: keepReason, lane: "next" })`. With
  `blank_reason = skip`, a blank result skips the target.
- Plan it with `planManagedTaskLogEntry` (`kind: MANAGED_TASK_LOG_KIND_ROTTEN`). On
  `ambiguous-markers`, refuse the whole plan with `rotten-log-ambiguous` and write
  nothing, matching counted keeps.
- Increment `rottenLogWrittenCount`, so the existing Fresh toast tail reports it.

The lane entry is written in the same editor or vault transaction as the stamp, with one
undo. 520 and 532 pass `laneLines` per group, the same way they pass `rottenLines`
today. Whether an already-fresh-today Next task logs follows the same `plan.stamped`
membership as the Pending Work Log insertion. Don't add a separate same-day rule.

## Steps

### bob-plugins (open it with `/sase_repo` and read its `AGENTS.md`)

1. Grammar: edit `ROTTEN_LOG_ENTRY_RE` in 010. Put the lane-aware
   `formatRottenLogEntryText` / `parseRottenLogEntryBullet` changes where 030 stays at
   or under 1000 lines; move the two helpers into `195-rotten-keep.js` if needed. Mirror
   the change in ledger-tools `135-freshness-rotten-log.js` (both regexes and the
   returned shape).
2. `isNextRottenLogTargetRawLine` in 220, and export it through `700-exports.js`.
3. Lane mode in `buildRottenKeepPromptModel` / `RottenKeepReasonStage`. Lane rows and
   labels in the batch form of `FreshnessRefreshSummaryModal`. Make the single `keep`
   form honor the lane model's disabled rule (it reads
   `buildRottenKeepPromptModel(...).disabled` today).
4. The shared prompt-plan helper fragment, wired into 520 and 532.
   `requestFreshnessRefreshAnswers` passes `lane` through and returns the lane reason as
   `keepReason`.
5. `laneLines` in `planFreshStampBatchWithLogs`, passed from 520 and 532.
6. Tests (Node `--test`, `scripts/`). Extend the existing files and keep each new file
   registered in `package.json` `test`:
   - `test-navigation-rotten-log.cjs`: lane entry round trip; rejects `kept in next`,
     `kept in Pending`, and `kept 0×`; blank lane reason per `blank_reason`; history
     rows print `kept in Next`.
   - `test-navigation-rotten-keep.cjs`: writer with `laneLines`. Blank and nonblank
     reasons. Created versus reused marker. Ambiguous markers refuse only when an entry
     would be written. `[keeps::]` unchanged. A mixed Pending + Next + counted batch
     routes each answer only to its own log.
   - `test-navigation-refresh-modal.cjs`: the lane single-prompt model (title,
     placeholder, hints with and without `advance`, blank and nonblank previews, the
     disabled rule), batch lane preview rows, and the destination counts.
   - The prompt-plan helper: every combination of counted, Pending, and Next targets for
     single, counted, and Task Link shapes.
   - Gesture coverage, in whichever of `test-navigation-review-advance-gestures.cjs` /
     `test-navigation-freshness.cjs` already drives Alt+F: Ctrl+Alt+F on a landed NEXT
     row prompts, then stamps, logs, and advances. Esc writes nothing and does not
     advance. Alt+F on an early `[*]` task prompts. Pending still gets only the Work
     summary.
   - Ledger-tools `test-ledger-tools-freshness-mark*.cjs`: the tooltip line counts lane
     entries on both the source-line and DOM paths.
   - Update every existing assertion that says Next never prompts or never writes a
     Rotten Log.
7. Versions and docs. Bump bob-navigation-hotkeys 2.63.0 → 2.64.0 and bob-ledger-tools
   1.80.1 → 1.81.0. Update both manifest descriptions:
   - nav: "Alt+F and Ctrl+Alt+F ask Next tasks why they stay in Next and log a typed
     reason in the Rotten Log", next to the Pending sentence, and the combined-prompt
     sentence.
   - ledger-tools: the tooltip counts `kept in Next` entries.

   Rewrite the keep section of `docs/bob-navigation-hotkeys.md`, roughly the paragraphs
   starting "`Alt+F` and `Ctrl+Alt+F` stamp…" through the counted/Task Link paragraph.
   Remove Next from the never-ask list, and add the Next prompt and the batch routing.
   Run `npm run docs:gen`.

8. Run `npm run build`, `npm test`, and `npm run docs:check`, then `bob plugins sync`.

### bob-cli

9. Docs:
   - `docs/freshness.md`:
     - §2a **Rotten Log**: add the Next lane-keep entry and its optional reason, and
       that it never touches `[keeps::]`.
     - §5: add a paragraph next to the Pending Work Log paragraph. Extend "Only
       bob-navigation-hotkeys writes the Rotten Log (…)" to name Next keeps.
     - §6 step 3 lanes: "Pending asks for an optional Work Log summary; Next asks why it
       stays in Next…".
     - §8 surfaces table row for the keep-reason prompt.
     - §11 rotten-log tooltip bullet: lane entries count.
     - §13 rollout log line for 2026-11 and the nav/ledger-tools versions.
   - `docs/reference/vault-format.md`: the entry grammar gains the `· kept in Next`
     form.
   - `docs/concepts/freshness-and-the-review-walk.md` line ~54.
   - `docs/task-dependencies.md` managed-log table, `🥀 **ROTTEN LOG**` writer cell: add
     "nav Next keep reason".

   Plugin reference pages under `docs/reference/plugins/` are mounted from bob-plugins
   and are not tracked here, so don't edit them in bob-cli. Do not touch `Cargo.toml`'s
   version or `CHANGELOG.md` (release train).

10. Memory, gated on the memory decisions; use `/sase_memory_write`:
    - `memory_decision_records` (1 of 2): new accepted record
      `sase/memory/decisions/next-keeps-ask-an-optional-rotten-reason.md`, in the same
      shape as `decisions/alt-n-answers-from-due-rows.md` (frontmatter `keyword`,
      `aliases`, `summary`, `metadata.status: accepted`, `decided` on the landing date).
      - Applies to: bob-plugins nav and ledger-tools, bob-cli docs, the vault.
      - Claim: §1–§5 condensed, including the chosen `blank_reason`.
      - Why: Bryan's request ("start prompting for an optional rotten log entry when
        next Obsidian tasks are marked fresh").
      - Rejected alternatives: a Work Log summary for Next ("What did you get done?"
        does not fit unstarted work); counting Next keeps toward `[keeps::]`; the
        non-chosen `blank_reason` option; a separate Next log.
      - Evidence: this plan and nav 2.64.0 / ledger-tools 1.81.0.
      - Cost: up to about 15 extra Enter presses per morning at the Next cap. Rotten Log
        entries are no longer only counted keeps.
      - Reopens when: Next reasons go mostly blank for a month, or Bryan wants Pending
        in the Rotten Log too.
      - Links: `[[decisions/rotten-keeps-are-logged]]`,
        `[[decisions/task-lanes-are-sticky]]`, `[[decisions/review-walk-is-tiered]]`.
    - `memory_decision_records` (2 of 2): in `decisions/rotten-keeps-are-logged.md`, set
      `metadata.status: superseded-in-part` and
      `superseded_by: [decisions/next-keeps-ask-an-optional-rotten-reason]`, and append
      "Superseded in part: Next keeps now ask for an optional reason and may write a
      `· kept in Next` entry — see
      [[decisions/next-keeps-ask-an-optional-rotten-reason]]." Do not reword the body.
    - `memory_rotten_log_glossary`: rewrite the `glossary/rotten-log.md` body to add the
      `· kept in Next` entry form and the optional Next reason, and narrow the exclusion
      to "NEW, early Ready, Pending, PROJECTS, and unblock-tickler keeps never write
      it".
    - Run `sase memory init`.
11. Run `just docs-gen` (regenerates `docs/develop/decisions.md` and
    `docs/reference/glossary.md` from memory, plus the mounted plugin inventory) and
    `just docs-check`.

## Verification

- In bob-plugins, `npm test`, `npm run build:check`, and `npm run docs:check` all pass,
  and `bob plugins sync` deploys.
- In bob-cli, `just docs-gen` leaves a clean, committed tree and `just docs-check`
  passes. No Rust sources change.
- For Bryan to check in Obsidian:
  - Alt+F on a Next task opens "Keep in Next?". A blank Enter stamps only. A typed
    reason stamps and adds `*2026-11-0N* — <reason> · kept in Next` under
    `🥀 **ROTTEN LOG**`, and one undo reverts both.
  - Ctrl+Alt+F on a landed NEXT row advances after Enter. Esc leaves the row due and in
    place.
  - A mixed `3<Alt+F>` over Pending, Next, and ROTTEN rows shows one prompt with both
    fields.
  - The freshness-mark tooltip counts the new entry.

## Out of scope

Pending stays Work-Log-only. There are no changes to Ready counted keeps, the decay
card's decisions, streak math, the Task Card or scheduling prompts, the Rust agenda, or
the Mac app, and there is no backfill.
