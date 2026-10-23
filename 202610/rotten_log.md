---
tier: epic
title: 'Rotten Log: a dated, reasoned history of every rotten keep'
goal: 'Every counted keep of a rotten task (Alt+F or Ctrl+Alt+F on an exact due Ready
  ROTTEN or scheduled TICKLER row, and Keep on the approved-decay card) asks "Why
  keep it as written?" and, in the same transaction as the stamp, writes one dated
  entry under a `🥀 **ROTTEN LOG**` child. Each task then shows how many times, when,
  and why it was kept: in the note, in the prompt, on the decay card, on the freshness
  mark tooltip, in note snapshots, and in the Mac agenda.

  '
decisions:
  memory_decision_record:
    ask: Add the decisions-web record rotten-keeps-are-logged that governs this feature?
    memory:
    - decisions:rotten-keeps-are-logged
    default: true
    requested: Review the rotten_log_keep_history.md file in the research sidecar
      repo for context and inspiration before planning. I agree with all of the requirements
      recommended in that research file
    answer: true
  memory_supersede_decay:
    ask: Mark rotten-keeps-use-priority-decay superseded in part by the new record?
    memory:
    - decisions:rotten-keeps-use-priority-decay
    default: true
    corrects: Keep anyway leaves only the line increment.
    answer: true
  memory_glossary:
    ask: Add the Rotten Log glossary term?
    memory:
    - glossary:rotten-log
    default: true
    requested: I would like to add support for a new "rotten log" (a good glossary
      entry should be added for this new term). Can you help me implement this?
    answer: true
phases:
- id: record
  title: Decision record, supersession mark, and glossary term
  depends_on: []
  size: small
  description: 'record: write decisions:rotten-keeps-are-logged, mark rotten-keeps-use-priority-decay
    superseded in part, add the Rotten Log glossary strand, republish memory, and
    regenerate the decisions and glossary docs pages.'
- id: registry
  title: bob-cli managed-log registry, agenda JSON, and note snapshots
  depends_on:
  - record
  size: medium
  description: 'registry: add ManagedTaskLogKind::Rotten with the 🥀 marker grammar,
    fix the Schedule Log inserter that files entries under any managed log, emit log
    "rotten" in capture-pomodoros agenda JSON, render KEPT rows in note snapshots,
    and document the registry row and MC vectors.'
- id: nav-grammar
  title: Navigation plugin Rotten Log grammar, reader, and inserter
  depends_on:
  - record
  size: small
  description: 'nav-grammar: teach bob-navigation-hotkeys the rotten managed-log kind,
    a history reader, an entry formatter, and a kind-generic log inserter, so project
    conversion carries the log and later phases share one grammar.'
- id: keep-single
  title: Keep-reason prompt and combined write for a single keep
  depends_on:
  - nav-grammar
  size: medium
  description: 'keep-single: build the shared keep-reason stage and prompt model,
    the counted-keep predicate, and the stamp-plus-log planner, then wire single-target
    Alt+F and Ctrl+Alt+F (including the card-unavailable fallback) through them.'
- id: card-keep
  title: Decay card Keep asks why and logs
  depends_on:
  - keep-single
  size: medium
  description: 'card-keep: turn the approved-decay card''s Keep into an in-card reason
    stage that commits stamp, increment, and Rotten Log entry as one transaction,
    and show the logged count and last reason on the card.'
- id: keep-batches
  title: Counted and Task Link keep sessions with one combined prompt
  depends_on:
  - card-keep
  size: medium
  description: 'keep-batches: give counted N<Alt+F> and Task Link sessions one prompt
    that shows a Keep reason field and/or a Work summary field, and write one entry
    per counted target across notes with honest refusals.'
- id: mark-tooltip
  title: Freshness mark tooltip shows the logged keeps
  depends_on:
  - keep-batches
  size: small
  description: 'mark-tooltip: add a "🥀 N keeps logged · last <date> — <reason>" line
    to the bob-ledger-tools freshness mark tooltip in source-note views, never in
    Tasks query rows.'
- id: mac-agenda
  title: Mac agenda renders Rotten Logs with a 🥀 chip
  depends_on:
  - registry
  size: small
  description: 'mac-agenda: decode the agenda "rotten" log kind in Bob Mac Capture,
    render its header and entries like the other logs, and add a 🥀 N count chip.'
- id: docs
  title: Behavior contracts, guide, and rollout record in bob-cli docs
  depends_on:
  - registry
  - mark-tooltip
  - mac-agenda
  size: small
  description: 'docs: update docs/freshness.md, docs/projects.md, the morning-review
    guide, and the freshness concept page to the shipped behavior, record the rollout,
    and pass the docs gate.'
proposed_by: bbugyi200.athena.research.55.linker.w2
proposed_with: claude/opus
decided_by: reviewer
decided_via: telegram
create_time: 2026-10-23 08:24:11
status: done
bead_id: bob-cli-as
---

- **PROMPT:** [prompts/202610/rotten_log.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202610/rotten_log.md)
- **BEAD:** [bob-cli-as](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-as/README.md)

# Plan: Rotten Log

## Why

A bare keep is the only morning-review answer that leaves no trail. Defer, Less often,
and Reword write Schedule Log entries. Drop writes a Cancel Log entry. A keep only
rewrites the task line, and `[keeps:: N]` is a _streak_ that the next generic stamp
erases. Nobody can later tell why a stale task survived three reviews.

The research report `research:202610/rotten_log_keep_history/rotten_log_keep_history.md`
recommends a Work-Log-shaped child under the task, written by the counted keep itself.
Bryan agreed with all of its requirements (R1–R8) and its recommended answers to the
open questions:

- include scheduled ticklers, exclude unblock ticklers;
- a blank Enter always logs `🤷`;
- prompt on every counted keep;
- keep the `· kept N×` tail.

This plan turns that report into a design and splits the work. Where the report left
choices open (the optional surfaces, how Esc behaves inside the decay card, IME, and day
rollover), this plan decides them.

The measured load is small, about 0.6 prompts per morning. Most of the cost is the
closed managed-log registry, which spans Rust and four JavaScript plugins. An
unregistered marker corrupts data. For example, nav project conversion turns an unknown
direct child into a task. Registration therefore comes before any writer.

## Design

### One event, two records

A **counted keep** is the one event that increments `[keeps:: N]`. Every counted keep
now also writes exactly one Rotten Log entry, in the same editor transaction, with one
undo. Nothing else ever writes the log.

| Route                                                                                                                | Today                                       | After                                            |
| -------------------------------------------------------------------------------------------------------------------- | ------------------------------------------- | ------------------------------------------------ |
| Alt+F / Ctrl+Alt+F on an exact due Ready ROTTEN or scheduled TICKLER row, below the decay limit                      | stamp + increment                           | prompt → stamp + increment + entry               |
| Same row at the limit (single press)                                                                                 | opens the decay card                        | opens the decay card (unchanged)                 |
| Decay card **Keep** (⌥F)                                                                                             | stamp + increment, one line                 | in-card reason stage → stamp + increment + entry |
| Decay card Not now / Less often / Reword / Drop                                                                      | own Schedule/Cancel entries                 | unchanged; never a Rotten Log entry              |
| Card cannot render (mixed versions)                                                                                  | fallback stamp + increment                  | the same prompt → stamp + increment + entry      |
| Counted `N<Alt+F>` and Task Link sessions                                                                            | stamp + increment per counted target        | one prompt → one entry per counted target        |
| Unblock TICKLER (`returnedBy: "unblocked"`), NEW, early/FRESH, Pending, Next, PROJECTS, recurring, checklist, closed | uncounted stamp / refusal / Work Log prompt | unchanged; never prompts for a keep reason       |
| `bob capture`, Rust, hooks, randomize, seed                                                                          | never increment                             | never write the log                              |

### Vocabulary: two numbers, never mixed

- **Streak**: `[keeps:: N]`, which resets. UI copy says "kept N in a row" (title and
  subtitle lines) or the established `kept N×` tail on log entries and notices.
- **Logged count**: the number of valid entries in the task's Rotten Log, which only
  grows from rollout onward. UI copy always says "N keeps logged" (or `🥀 N` in a chip).
- Never add the two numbers or show one as the other. On rollout day, `keeps:: 2` with
  an empty log is expected. Surfaces say "none logged yet" or omit the logged count, and
  nothing backfills.

### Event rule: the counted-keep predicate

Compute the predicate before any prompt, per resolved target. A target is a counted keep
when all of these hold:

1. `matchFreshStampExactEntry(queueBefore, { path, line, raw }).ok` is true. That means
   an exact pre-write queue row with `lane === "ready"` and tier `rotten` or `tickler`.
2. `match.entry.returnedBy !== "unblocked"`.
3. The raw line does not already carry today's stamp (`freshStampLineHasToday`).
4. A **dry run** of the same stamper, `keepLine(raw, dateText, { counted: true })`,
   yields `keeps === Math.min(keepsBefore + 1, 999)` with a changed line.

`keepLine` is pure, so step 4 asks the sole increment helper itself whether it would
count. That includes its prior-`fresh` guard, saturation, and the unblock rule. The
Rotten Log entry therefore cannot disagree with the streak. The predicate requires
`freshnessSupportsKeeps(api)`. Pre-v5 namespaces never count, so they never prompt.

After the prompt, the planner re-stamps from the revalidated content. Every target
predicted as counted must show the increment in its actual before/after pair, using
`countFreshStampKept` semantics. If any target disagrees, the whole write is refused.
Never log a guessed keep.

### Storage grammar

```markdown
- [ ] #task Read "Agentic software engineering" #ref [fresh:: 2026-10-23] [refresh:: 7]
      [keeps:: 3] ^ref-ase
  - 🥀 **ROTTEN LOG**
    - _2026-10-23_ — still the primer for the memory epic; read it this weekend · kept
      3×
    - _2026-10-16_ — 🤷 no reason given · kept 2×
    - _2026-10-09_ — waiting for the weekend reading block · kept 1×
```

- **Marker.** Writers emit `🥀 **ROTTEN LOG**` (U+1F940, one code point, no VS16).
  Parsers accept an optional U+FE0F after the emoji, the emoji-less `**ROTTEN LOG**`,
  and the title-case `**Rotten log**`, each with an optional colon and only trailing
  spaces or tabs, the same rule as the Schedule and Work Logs. A present emoji must
  agree with the label, so `🛠️ **ROTTEN LOG**` is not a marker. Any list marker is
  accepted (`-`, `*`, `+`, `1.`, `1)`).
- **Entry.** `*YYYY-MM-DD* — <reason> · kept N×`. The separator is an em dash with
  spaces (U+2014) and the tail is a middle dot (U+00B7) plus `×` (U+00D7). N is the
  streak after the write, 1–999. The reason is whitespace-normalized like a Work summary
  (`normalizeLaneWorkSummary`). A blank reason becomes `🤷 no reason given`, reusing
  `SCHEDULE_LOG_SKIPPED_REASON_TEXT`. The date is the local review date, `dateText`.
- **Order.** Newest first: a new entry is inserted directly under the marker.
- **Placement.** On the first entry, the marker plus the entry are appended as the
  task's **last direct child**, as the Work Log is. After that the marker is reused in
  place and never moved or duplicated. This never displaces a first-child Cancel Log or
  Depends-On line. Entry indentation and list marker follow the marker's first existing
  entry, else marker indent + one tab and `-`.
- **Counting.** A logged keep is a direct child bullet of the marker whose text matches
  `*YYYY-MM-DD* — …`. Other children (hand notes, nested bullets) are kept but not
  counted. When a task has two or more direct-child Rotten Log markers, writers refuse
  with a notice and readers report "unknown" (no count), never a wrong number.
- **Lifetime.** Automation never clears the log. It travels with the block through
  Ctrl+Shift+M, `bob task move`, archive, `bob ref` migration, and nav project
  conversion (once registered).

### The keep-reason prompt (bob-navigation-hotkeys)

The keep-reason prompt is one shared stage, `RottenKeepReasonStage`, driven by a pure
model, `buildRottenKeepPromptModel`. Two hosts mount it: `FreshnessRefreshSummaryModal`
gains a keep mode, and `FreshnessDecayCardModal` gains a Keep stage. Do not build a new
modal family. Reuse the `bob-cnp-*` layout classes. Add only the small `bob-rotten-*`
styles below.

Single keep (Alt+F; Ctrl+Alt+F differs only in the Enter hint):

```text
╭───────────────────────────────────────────────────────────────────────╮
│ 🥀  Keep as written?                                                  │
│     Read "Agentic software engineering" · TICKLER · 9d                │
│     kept 2 in a row · 2 keeps logged                                  │
│ ┌───────────────────────────────────────────────────────────────────┐ │
│ │ ✎  Why keep it as written?  (optional · ↵ logs 🤷)                 │ │
│ └───────────────────────────────────────────────────────────────────┘ │
│   ✓  *2026-10-23* — read it this weekend · kept 3×                    │
│      Prepends under 🥀 ROTTEN LOG                                     │
│   EARLIER KEEPS                                                       │
│      Oct 16   🤷 no reason given                            kept 2×   │
│      Oct 09   waiting for the weekend reading block         kept 1×   │
│                                                                       │
│   ↵ Keep & log        esc Cancel                                      │
╰───────────────────────────────────────────────────────────────────────╯
```

Copy and behavior:

- **Title.** "Keep as written?". A batch reads "Keep 3 tasks as written?". The header
  icon is the 🥀 glyph in a soft round tint:
  `color-mix(in srgb, var(--color-red) 14%, transparent)` with `var(--text-normal)`
  text. Use Obsidian variables only, so light and dark themes both work.
- **Subtitle.** Task title (cleaned with `cleanTaskDisplayText`), tier label (`ROTTEN`
  or `TICKLER`), and age. The second line holds the streak and the logged count. Omit
  "kept N in a row" when the streak is 0. Say "none logged yet" when the log is absent
  or empty, and "log unreadable" when markers are ambiguous. In that last case the
  submit is disabled, because the planner would refuse anyway.
- **Field.** `aria-label="Keep reason"`, placeholder "Why keep it as written? (optional
  · ↵ logs 🤷)". Focus starts in the field. Never prefill a previous reason. Use neutral
  copy, never "why haven't you done this?".
- **Live preview row.** Shows the exact entry text the write will insert, from the same
  formatter the planner uses. Blank shows the `🤷 no reason given` entry in a muted
  `is-empty` style. That is not an error. A reason containing `::` shows the existing
  warning: "`::` creates a Dataview inline field on this bullet".
- **Destination line.** "Prepends under 🥀 ROTTEN LOG" or "Adds 🥀 ROTTEN LOG as the
  task's last child".
- **Earlier keeps.** Up to three newest entries plus "+ N older". Each row is a
  two-column grid: the date as `Oct 16` (the same fixed English short date the freshness
  tooltip uses, `font-variant-numeric: tabular-nums`), then the reason on one line with
  CSS ellipsis and the full text in `title`. A quiet right-aligned `kept N×` pill sits
  at the end. `🤷` rows are muted (`--text-faint`). Omit the section when the log has no
  entries. History comes from content the handler already holds; never read the vault
  again for it. The list has `role="list"`, and the preview has `aria-live="polite"`.
- **Keys.**
  - Enter submits.
  - Ignore Enter while `event.isComposing` (IME).
  - Esc cancels: no stamp, no entry, no advance, and the walk anchor is retained.
  - Alt+F and Ctrl+Alt+F keydowns inside the field are swallowed (`preventDefault`), so
    the held or repeated chord never types `ƒ` and never submits.
  - A double submit is a no-op (`completed` guard).
- **Hints.** `↵ Keep & log` for Alt+F; `↵ Keep, log & next` for Ctrl+Alt+F;
  `esc Cancel`. The buttons are Cancel and Keep & log.
- **Notice.** The Fresh notice gains ` · Rotten Log updated` (one entry) or
  ` · N Rotten Logs updated` (several), after the existing `kept N×` and Work Log tails.

### The decay card's Keep stage

The card stays a single-key card, and nothing about opening it changes. A single
at-limit press opens it before any reason prompt, so no prompt ever stands in front of a
card that might defer or drop.

- **⌥F Keep** swaps the card's rows for `RottenKeepReasonStage` _inside the same modal_.
  There is no close/reopen flash, and focus moves into the field. The stage title is
  "Keep as written?". The subtitle is "Kept 3 reviews in a row · this keep makes 4" plus
  the logged-count line.
- **Enter** revalidates the card context again (the stage is asynchronous), then commits
  the stamp, the increment, and the entry in **one** `applyEditorContentTransaction`.
  This replaces today's one-line `replaceEditorLine`. Ctrl+Alt+F advances exactly once
  through `maybeAdvanceFreshnessDecayWalk`. The notice reads
  `Kept 4× · Rotten Log updated · next review asks`.
- **Esc in the stage returns to the card's choices.** This is the intended nudge: no
  reason comes to mind, so pick Not now, Less often, or Drop. A second Esc closes the
  card and writes nothing, as today.
- The card header gains a display-only line,
  `🥀 2 keeps logged · last Oct 16 — 🤷 no reason given`, omitted when nothing is
  logged. The Keep row detail becomes `say why · logs 🥀 · asks again next review`.
- The other four outcomes keep their writers and logs and never write a Rotten Log
  entry.

### Batches

Counted `N<Alt+F>` and Task Link sessions open **one** prompt:

- It has a Keep reason field when at least one counted-keep target is present. The field
  shows per-task preview rows (`title · kept N× · 3 logged`) and no history panel.
- It has the existing Work summary field when Pending targets are present.
- Mixed sessions show both fields, Work summary first. Tab and Shift+Tab move between
  them, and one Enter submits both.
- A target can never qualify for both fields: a Pending task is not in the Ready lane.
- The shared reason gives each counted target its own entry with its own `kept N×`. A
  work summary never goes into a Rotten Log, and a reason never goes into a Work Log.
- At-limit targets still skip ("N needs a decision"), unstamped and unlogged.
- Duplicate links to one source task are deduplicated before planning, as today.

### The visual indication, surface by surface

| Surface                                                                   | What it shows                                                                                          | Phase        |
| ------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------ | ------------ |
| The note                                                                  | the log itself, right under the landed task (the walk lands in the source note)                        | keep-single  |
| Keep prompt                                                               | logged count, last three reasons, exact entry preview                                                  | keep-single  |
| Decay card                                                                | "N keeps logged · last …" header line; the Keep stage                                                  | card-keep    |
| Freshness mark tooltip (Live Preview and reading view of the source note) | `🥀 3 keeps logged · last Oct 16 — waiting for the weekend…` under "Kept N reviews in a row"           | mark-tooltip |
| Note snapshots (`bob` highlights/ref note snapshots)                      | open tasks: one condensed `KEPT` row (newest entry, `+N older`); include-closed snapshots: every entry | registry     |
| Mac idle agenda                                                           | `🥀 Rotten log` header, entries folded with the other logs, a `🥀 N` chip after `⚒ N` / `🗓 N`         | mac-agenda   |
| Task line                                                                 | unchanged: the pips stay the only mark, and nothing new goes on the line                               | —            |

### Write safety

- Capture the preimage (`content` or each group's content), `dateText`, the queue match,
  and the predicate **before** the prompt. After the prompt, refuse with a notice and
  write nothing in any of these cases:
  - the editor or any linked note changed: "Current note changed; no tasks were updated"
    or "A linked note changed; no tasks were updated";
  - the local day changed: "Day changed while the prompt was open; nothing was written";
  - the post-plan increment check failed: `REVIEW_QUEUE_CHANGED_NOTICE`.
- Plan stamps first (all-or-nothing, as `planFreshStampBatch` does). Then insert Work
  and Rotten entries bottom-up (descending line), so earlier insertions never shift
  later targets.
- If any counted target's entry cannot be planned (ambiguous markers, an unparseable
  child block), refuse the whole keep. Never write a stamp without its entry.
- Same-note writes are one editor transaction and one undo step. Linked notes go through
  the existing grouped `commitLinkPickerNoteWrites` with preimage checks. Do not claim
  atomicity across notes. A partial cross-note failure reports honestly, and a retry
  never duplicates an entry, because a same-day retry is not a counted keep.
- Child writes stay out of `api.freshness.keepLine`, which remains a pure single-line
  helper shared with the counting vectors.
- Inserts land below the answered row, so the insert-above settle class of bugs
  (`bob-cli-94`) does not apply. Ctrl+Alt+F after an insert must still land on the right
  successor. Cover that in tests.

### What does not change

These stay as they are:

- the `[keeps::]` contract and `tests/fixtures/freshness_keeps/vectors.json`;
- `decide` and the card's choices and their writers;
- buckets, tiers, chips, and the `NEW ∪ TICKLER ∪ ROTTEN ∪ READY` partition;
- walk advance and lock rules, with Alt+F as the one stay answer;
- the review journal's `keep` outcome;
- `bob capture`, which stays a generic stamp that never writes this log;
- Rust's no-increment rule;
- the Mac thin-client contract;
- the plan-budget and Task Link scanners, because a Rotten Log is never a Task Link.

### Rejected alternatives

- **A Rotten Log history line in the `]s` landing toast.** The log is already on screen
  under the landed row, and the toast stays short (Alt+P previews already extend it).
- **Esc in the card's Keep stage closes the card.** Returning to the choices costs
  nothing and turns "no reason" into a decision.
- **Alt+F inside the reason field submits.** A double press would silently log `🤷`.
  Enter is explicit.
- **A `freshness.rotten_log: false` switch.** At about 0.6 prompts a morning, a plugin
  rollback is enough.
- Everything the research rejected stays rejected: nonblank-only entries, logging every
  Alt+F, a lifetime field, a numeral by the mark, Schedule- or Work-Log storage, the
  journal as the store, a second-keep gate, and prompting only at the card.

## Coordination and baselines

- Epic `bob-cli-aq` (Alt+P) is still in progress. Phases aq.3 (review footer) and aq.4
  (bob-cli docs and memory) touch `bob-ledger-tools` 120 and `docs/freshness.md`. Its
  nav landing work (aq.1 and aq.2) has already landed. Before starting, every
  bob-plugins phase rebases onto the latest `master`, and the `docs` phase merges with
  aq.4's `docs/freshness.md` edits.
- The bob-plugins phases form a serial chain (`nav-grammar` → `keep-single` →
  `card-keep` → `keep-batches` → `mark-tooltip`). They share `src/` fragments, generated
  `main.js`, `docs/index.md`, `package.json`, and plugin versions, so they must not run
  concurrently. `mark-tooltip` depends on `keep-batches` for this ordering only.
- Plugin test baseline: `bob-cli-6w` is a known pre-existing failure, a decision-card
  Keep notice assertion that sees the review-journal missing-API notice. `card-keep`
  rewrites that notice and its tests. Record on the phase bead whether the failure
  persists. Never change `bob-cli-6w`'s status.
- bob-cli rides the daily release train. Never edit `Cargo.toml`'s version or
  `CHANGELOG.md`. The agenda `log: "rotten"` value is an additive JSON change, not a `!`
  break, because the Mac decodes unknown kinds to `.other`.
- Every bob-plugins phase ends with `npm run build`, `npm test`, `npm run docs:check`,
  the touched plugins' `manifest.json` version bumps (minor for new behavior), and
  `bob plugins sync`. Open the linked repos with `/sase_repo` and read each repo's
  `AGENTS.md` first.

## Phases

### record: Decision record, supersession mark, and glossary term

All memory edits go through `/sase_memory_write`, authorized by this plan's three memory
decisions. Write nothing else in memory.

> [!decision] memory_decision_record Item 1 writes the new record. Later phases and docs
> link to it as `decisions:rotten-keeps-are-logged` /
> `docs/develop/decisions.md#rotten-keeps-are-logged`.

> [!decision] memory_decision_record = no Skip item 1. Later phases and docs cite
> `research:202610/rotten_log_keep_history/rotten_log_keep_history.md` instead of the
> record. The glossary strand ends with that research ref in place of the record link,
> and item 2 is skipped too.

1. **New record** `sase/memory/decisions/rotten-keeps-are-logged.md`, in the format of
   its siblings: frontmatter `keyword`, `aliases`, `summary`, and
   `metadata.status: accepted` with `decided: 2026-10-23`.
   - Keyword: "Rotten Keeps Are Logged". Aliases: `rotten log`, `keep log`.
   - Summary (the roster rule): "Every counted keep — Alt+F/Ctrl+Alt+F on an exact due
     Ready ROTTEN or scheduled TICKLER row, or the decay card's Keep — asks an optional
     reason and writes one dated 🥀 ROTTEN LOG entry with the stamp; the entry count is
     display-only history, never a field or a decay input."
   - **Applies to.** bob-plugins (bob-navigation-hotkeys, bob-ledger-tools), bob-cli
     (managed-log registry, agenda JSON, note snapshots, docs), Bob Mac Capture (agenda
     chip), and the vault.
   - **Claim.** State the event rule (the counted-keep predicate in plain words), the
     always-an-entry and optional-reason rule, Esc writing nothing, the grammar, N as
     the streak after the keep, and that the log is audit and display only. Its count
     ("N keeps logged") never resets, is never stored as a field, and nothing reads it
     to change decay, buckets, tiers, lanes, Today, schedule, priority, or freshness.
     Also state the excluded keeps, that `bob capture` and Rust never write it, and that
     there is no backfill.
   - **Rejected alternatives**, one line each, from the research report's "Alternatives
     considered": nonblank-only entries, logging every Alt+F, ROTTEN tier only, a
     lifetime field or numeral by the mark, Schedule- or Work-Log storage, the journal
     as the store, and a second-keep gate.
   - **Evidence**: the research ref, this epic's plan ref (as `sase bead read` shows
     it), `docs/freshness.md` §2a, and `docs/task-dependencies.md` §2.4.
   - **Cost**: one more kind in the closed registry across Rust and every plugin parser;
     a two-key keep (Alt+F, then Enter); log growth of about four lines a month on a
     weekly-kept reading task; two numbers to keep apart; editable, not tamper-proof
     Markdown.
   - **Reopens when**: a month of entries is more than 80% `🤷`, or counted keeps exceed
     about five per morning (consider a from-the-second-keep gate), or the log goes
     unread.
   - End with a `[[decisions/rotten-keeps-use-priority-decay]]` link that names what it
     supersedes in part.
     > [!decision] memory_supersede_decay Item 2 marks the old record, so the roster
     > shows it as superseded in part by the new one.

   > [!decision] memory_supersede_decay = no Skip item 2. The new record still links
   > back to the old one. On the phase bead, note that the old record's roster tag is
   > now stale.

2. **Supersession mark** on `decisions/rotten-keeps-use-priority-decay.md`. Append
   `decisions/rotten-keeps-are-logged` to `metadata.superseded_by` (the status is
   already `superseded-in-part`). Append one back-link paragraph in the style of the
   existing ones: "Superseded in part: the rejection of a new REVIEW LOG structure and
   'Keep anyway leaves only the line increment' only — Keep now asks why and writes one
   Rotten Log entry; see [[decisions/rotten-keeps-are-logged]]. Counting, explicit
   consent, the priority ladder, and the other card outcomes stand." Do not reword the
   body.

   > [!decision] memory_glossary Item 3 adds the glossary strand, which `just docs-gen`
   > mirrors into `docs/reference/glossary.md`.

   > [!decision] memory_glossary = no Skip item 3. The `docs` phase defines the term
   > inline in `docs/freshness.md` §2a instead.

3. **Glossary strand** `sase/memory/glossary/rotten-log.md`:

   ```markdown
   ---
   keyword: Rotten Log
   aliases:
     - "keep log"
   ---

   A task-local history of rotten keeps, stored as a `🥀 **ROTTEN LOG**` direct child
   with newest-first `*YYYY-MM-DD* — <reason> · kept N×` entries. Alt+F or Ctrl+Alt+F on
   an exact due Ready ROTTEN or scheduled TICKLER row (the same keep that increments the
   [[keep-streak]]) and Keep on the approved-decay card first ask "Why keep it as
   written?": the reason is optional (blank logs `🤷 no reason given`) but the entry is
   not, and Esc writes nothing. N is the streak after that keep. The entry count is the
   number of keeps logged since rollout: unlike `[keeps:: N]` it never resets, and
   nothing reads it to change decay, lanes, Today, schedule, priority, or
   [[task-freshness]]. NEW, early, Pending, Next, PROJECTS, and unblock-tickler keeps
   never write it; Pending summaries stay in the [[work-log]] and deferrals in the
   [[schedule-log]]. See [[decisions/rotten-keeps-are-logged]].
   ```

4. Run `sase memory init`, then `just docs-gen`, which regenerates
   `docs/develop/decisions.md` and `docs/reference/glossary.md`. Then run
   `just docs-check`. Confirm that `sase memory read glossary:rotten-log` resolves every
   link and that the decisions roster shows the new record and the updated supersession
   tag.

### registry: bob-cli managed-log registry, agenda JSON, and note snapshots

Rust learns the kind. A registered marker is never a task, a section, a Task Link, or a
place to file another log's entries.

1. **Registry** in `src/native/capture/sub_bullet.rs`:
   - add `ROTTEN_LOG_EMOJI = "🥀"`;
   - add the labels `("ROTTEN LOG", Rotten)` and `("Rotten log", Rotten)` to
     `MANAGED_TASK_LOG_LABELS`;
   - add `ManagedTaskLogKind::Rotten`;
   - in `parse_managed_task_log_marker`, add an emoji branch that accepts an optional
     U+FE0F after 🥀 and filters to `Rotten`.

   Update the parity comment, which today claims only Schedule and Work are anchors.
   Extend the vectors in `src/native/capture/tests/assembly.rs`: every list-marker form,
   emoji/no-emoji, colon, legacy case, VS16, mismatched emoji (`🗓️ **ROTTEN LOG**` is
   not a marker), and trailing text (not a marker).

2. **Fix the Schedule Log inserter.** `capture_schedule_log::plan_entry_insertion` (used
   by `bob randomize` and `bob task reroll` via `randomize_plan.rs`) currently prepends
   under `first_direct_managed_log_start`, which is the first direct managed log of
   **any** kind. On a task with only a Work Log, it already files the schedule entry
   under the Work Log. With Rotten registered, it would file schedule entries under a
   Rotten Log, and Rotten-kept Ready tasks are exactly randomize's targets. Make it find
   the first direct-child **Schedule** marker, in the style of
   `capture_work_log::find_work_marker`. Otherwise it creates the marker as the last
   direct child. First add failing tests for a Work-Log-only and a Rotten-Log-only task.
   Keep the existing tests green.
3. **Resolve every exhaustive match** the compiler flags:
   - `capture_pomodoros_agenda.rs`: `AgendaLog::Rotten` serializes as `"rotten"`. Marker
     and subtree lines carry it, and the existing kind-stack logic is reused. Add tests
     next to the Work and Schedule ones.
   - `highlights_ref/note_snapshot/visibility.rs`: add `LogKind::Rotten`.
     - Open tasks get one condensed `KEPT` log row (newest entry, `older_count` for the
       rest), in the style of `condensed_work_row`.
     - Include-closed snapshots get `full_log_rows("KEPT", …)`.
     - Closed tasks in default snapshots render nothing for it.
     - Add a `rotten_logs` counter beside `schedule_logs` and `work_logs`.
   - `transclusion.rs` `count_log_blocks`: count rotten entries and `KEPT` rows into the
     new counter. Plumb `rotten_logs` through `view.rs`'s summary, and print
     `· N rotten logs` on the human `logs:` line in `highlights_ref/create.rs`.
4. **Confirm the automatic consequences** with tests:
   - captured sub-bullets (`capture_task_toggle/task_update.rs`) and task sections
     (`capture_task_sections.rs`) insert above an existing Rotten Log;
   - a Rotten Log is never a task section;
   - `capture_work_log` still finds only Work markers.
5. **Docs** (behavior and contract pages this phase makes true):
   - `docs/task-dependencies.md` §2.4: add a registry row for `🥀 **ROTTEN LOG**` and
     its dated entries. Owner: any `#task` with a keep history. Writer: nav counted keep
     and decay-card Keep. Slot: appended as the last direct child on creation, then
     reused. Rejecting recognisers: Task Link and Pomodoro scanners; section titles;
     Depends-On, BLOCKS, Sub-projects, LINKS/LINKED FROM, Schedule Log, Work Log, and
     Cancel Log parsers; R8 legacy-child. Add "Rotten Log" to the other rows' recogniser
     columns.
   - `docs/task-dependencies.md` §11.8: add MC vectors. A Rotten Log is never a Task
     Link or section. No other parser claims it, and it claims no other line.
     `bob randomize` / reroll never files a schedule entry under it. Nav project
     conversion carries it with the `^prj` task.
   - `docs/reference/vault-format.md`: add the grammar to the managed-log section.
   - `docs/capture.md`: the agenda `log` tag is `work`, `schedule`, or `rotten`, and
     captured notes land above the Rotten Log like the other logs.
   - Run `just docs-check`.
6. Run `just check` (or the repo's fmt, clippy, and test targets) and `cargo test`.

### nav-grammar: Navigation plugin Rotten Log grammar, reader, and inserter

In `plugins/bob-navigation-hotkeys/src` (bob-plugins):

1. In `010-requires-and-config.js`, add `MANAGED_TASK_LOG_KIND_ROTTEN = "rotten"` and
   the `ROTTEN_LOG_EMOJI`, `ROTTEN_LOG_LABEL`, `LEGACY_ROTTEN_LOG_LABELS`
   (`Rotten log`), `ROTTEN_LOG_MARKER_TEXT`, and `ROTTEN_LOG_PARENT_RE` constants. Build
   the regex with `buildManagedTaskLogParentRe(`${ROTTEN_LOG_EMOJI}️?`, …)`. Add a
   byte-level comment like the other logs'.
2. In `030-bullet-blocks-and-logs.js`, add the `rotten` branch to
   `parseManagedTaskLogParentBullet`. Update its "mirrors bob-cli" comment, because Rust
   now has Rotten too. Then add:
   - `formatRottenLogEntryText(reason, dateText, keepsAfter)`;
   - `parseRottenLogEntryBullet(line)` → `{ date, reason, kept }` or null;
   - `readRottenLogHistory(lines, taskLine)` →
     `{ status: "none" | "ok" | "ambiguous", markerLine, count, entries }`. Entries are
     newest first, as `{ date, reason, kept }`, and only valid dated direct children
     count.
3. In `220-property-targets.js`, generalize `insertLaneWorkLogEntry` into
   `insertManagedTaskLogEntry(workingLines, taskLine, { kind, entryText })`, with a
   kind-specific marker finder and refusal on ambiguous Rotten markers. Keep
   `insertLaneWorkLogEntry` as a thin wrapper so existing callers and tests are
   untouched. Its output must stay byte-identical.
4. In `110-project-seed.js` / `665-plugin-project-conversion.js`, conversion now carries
   the Rotten Log with the `^prj` task automatically through the parser. Add the
   `"rotten log moved"` notice detail next to the schedule, work, and cancel ones.
5. Export the new constants and helpers in `700-exports.js`.
6. **Audit** block-id-prompt, task-status-cycler, and bob-ledger-tools for closed
   managed-log recogniser lists. They use kind-specific Work and Schedule finders today,
   so expect no change. Record each finding in the phase bead. Change only a recogniser
   that would claim a Rotten Log marker or entry as prose, a section, or a Task Link.
7. **Tests** (new `scripts/test-navigation-rotten-log.cjs`, added to `npm test`):
   - parser vectors identical to the Rust ones;
   - reader counts and the ambiguity case;
   - inserter create, prepend, and mixed-indent cases, CRLF, a missing final newline,
     and children with nested tasks;
   - an inserter call on a task with a Cancel Log or Depends-On first child;
   - project conversion keeps the log as a log, not a task, with the notice detail;
   - the `insertLaneWorkLogEntry` byte-identity regression.
8. Update `docs/bob-navigation-hotkeys.md` (grammar mention only), run the checks, bump
   the version, and run `bob plugins sync`.

### keep-single: Keep-reason prompt and combined write for a single keep

1. **Predicate.** `classifyRottenKeepTarget({ match, raw, stamper, dateText })` returns
   `{ counted, keepsBefore, keepsAfter }`, exactly as in "Event rule".
2. **Prompt model and stage.**
   - `buildRottenKeepPromptModel(...)` is pure: it returns title, subtitle lines,
     placeholder, preview entry text for a given input, destination text, history rows,
     older count, hint labels, and disabled state.
   - `RottenKeepReasonStage` mounts the model into a container, with `value()`,
     `focus()`, and `destroy()`. Its keys are Enter (except while composing), the
     swallowed Alt+F / Ctrl+Alt+F, and an Esc callback.
   - Give `FreshnessRefreshSummaryModal` a keep mode that hosts the stage. The existing
     Work-summary mode must render and behave exactly as before.
   - Put the styles in `styles.css` under `bob-rotten-*`.
3. **Prompt request.** Extend `requestFreshnessRefreshSummary` into
   `requestFreshnessRefreshAnswers(...)`, which returns
   `{ cancelled, failed, summary, keepReason }`. Keep the test bypass: string
   `options.summary` / `options.keepReason` skip the modal.
4. **Planner.**
   `planFreshStampBatchWithLogs(content, targets, stamper, dateText, { workSummary, keepReason, rottenLines })`:
   - stamp all-or-nothing;
   - verify each predicted counted keep's increment (refusal `rotten-keep-mismatch`);
   - insert Work and Rotten entries bottom-up, with each tail taken from that target's
     post-stamp `keeps`;
   - refuse on an ambiguous marker (`rotten-log-ambiguous`);
   - return `rottenLogWrittenCount` alongside `workLogWrittenCount`.

   `planFreshStampBatchWithWorkLogs` becomes a wrapper. Add refusal notices to
   `freshStampRefusalNotice`.

5. **Wire the single-target source-note path** in `refreshTaskFreshnessOnTasks`, for
   Alt+F and Ctrl+Alt+F, inside or outside the walk:
   - classify;
   - prompt when counted, using history from `content`;
   - revalidate the content, the guarded write context, **and the local day**;
   - plan, then make one `applyEditorContentTransaction`;
   - `finishFreshStamp` gets the new tail, and Ctrl+Alt+F advances once through the
     existing tail.
6. **Fallback reroute.** When `maybeOpenFreshnessDecayCard` returns false, delete the
   duplicate fallback write block. Continue into the shared path with decision skipping
   disabled for that target, so the fallback prompts and logs like any counted keep.
7. Counted and Task Link sessions are out of scope here. They keep today's behavior
   until `keep-batches`, and they must still pass their tests unchanged.
8. **Tests** (`scripts/test-navigation-rotten-keep.cjs`, plus extensions to
   `test-navigation-refresh-modal.cjs`, `test-navigation-keep-counting.cjs`, and
   `test-navigation-review-advance*.cjs`):
   - a reason writes one entry; a blank writes `🤷`; Esc leaves the document and walk
     untouched;
   - a same-day repeat or a double submit writes nothing new;
   - eligibility: rotten and scheduled tickler log; unblock tickler, NEW, Pending (still
     gets its Work Log prompt), Next, PROJECTS, recurring, checklist, and early rows
     never prompt; a `#ref` task that evaluates ROTTEN logs; a saturated 999 streak
     still logs `kept 999×`;
   - the fallback cannot bypass the planner;
   - a document change, day rollover, or prediction mismatch during the prompt commits
     nothing;
   - one undo restores line, streak, and log;
   - Ctrl+Alt+F after an insert lands on the correct successor;
   - a `::` warning; IME Enter; Alt+F swallowed in the field;
   - the prompt model's copy for zero, one, and many logged keeps, and for the ambiguous
     log.
9. Update `docs/bob-navigation-hotkeys.md` (Alt+F section, manifest description
   sentence) and `docs/develop/plugin-apis.md` if any api surface changes. Then run the
   checks, bump the version, and run `bob plugins sync`. Do a live Obsidian check of
   focus, Enter/Esc, folding, and the walk after an insert if possible. Otherwise state
   on the phase bead that it was not live-verified.

### card-keep: Decay card Keep asks why and logs

1. `FreshnessDecayCardModal` gains a Keep stage. Choosing `keep` no longer closes the
   card. It swaps the row list for `RottenKeepReasonStage`, with no flash and focus in
   the field.
   - Enter in the stage invokes a new `onKeepReason(reason)` callback.
   - Esc returns to the rows and re-focuses the Keep row.
   - Esc on the rows closes the card, writing nothing, as today.
   - `handleKey` must route keys by stage, so Enter means Keep & log in the stage and
     Not now on the rows.
   - The opening ⌥F, and any repeat, must not submit the stage.
2. `openFreshnessDecayCard` passes the stage model. That is the history from the card
   context content (add it to `buildFreshnessDecayCardCtx`: `rottenHistory`, plus the
   preview `keepsAfter = min(keeps + 1, 999)`) and the header line
   `🥀 N keeps logged · last <date> — <reason>`.
3. `applyFreshnessDecayCardKeep(cardCtx, live, reason)`:
   - revalidate with `revalidateFreshnessDecayCard` again at submit;
   - plan the stamp and entry with the shared planner on `live.content`;
   - commit through one `applyEditorContentTransaction`;
   - send the notice `Kept N× · Rotten Log updated · next review asks`;
   - advance through `maybeAdvanceFreshnessDecayWalk`;
   - a stale check keeps today's behavior: `REVIEW_QUEUE_CHANGED_NOTICE` and rebuild the
     card.
4. In `050-decay-planner.js`, change the Keep row detail to
   `say why · logs 🥀 · asks again next review`.
5. **Tests** (extend `test-navigation-decision-card.cjs` and
   `test-navigation-decision-card-handlers.cjs`):
   - opening or Esc-ing the card writes nothing;
   - Keep → stage → Enter logs exactly once, with the correct `kept N×`;
   - stage Esc returns to the rows, and a second Esc writes nothing;
   - the other outcomes write only their own logs;
   - a stale edit during the stage refuses and rebuilds;
   - one undo restores everything;
   - Ctrl+Alt+F advance after the Keep stage.
   - Note the `bob-cli-6w` baseline (see Coordination).
6. Update `docs/bob-navigation-hotkeys.md` and `docs/task-card.md` if the card is
   described there. Then run the checks, bump the version, and run `bob plugins sync`.

### keep-batches: Counted and Task Link keep sessions with one combined prompt

1. **Source-note sessions.** In `refreshTaskFreshnessOnTasks` (counted `N<Alt+F>`),
   classify every stamp target and collect `rottenTargets` and `workEligible`. Open
   **one** prompt when either is non-empty: keep mode, work mode, or a two-field mixed
   mode. Then plan once with `planFreshStampBatchWithLogs` and commit one transaction.
2. **Task Link sessions.** In `refreshTaskFreshnessOnLinks`, classify per group from
   each group's content, after `deduplicateFreshStampTargets`. Show one prompt. Re-read
   every group's live content whenever _any_ prompt was shown (today that happens only
   for the Work prompt). Plan each group with the shared planner, and commit through
   `commitLinkPickerNoteWrites`. Sum `rottenLogWrittenCount` into the notice.
3. **Mixed modal.** The Work summary field comes first and the Keep reason field second.
   Tab and Shift+Tab move between them, and one Enter submits both. Each field has its
   own preview and destination text. The keep field lists per-target rows
   (`title · kept N× · M logged`) instead of the history panel.
4. Skipped at-limit and checklist targets keep their tails and are never logged.
5. **Tests**:
   - mixed Pending + rotten fields route text to the right logs;
   - Task Link duplicates write one entry;
   - skipped decision rows stay unlogged;
   - a linked-note change during the prompt writes nothing;
   - a partial cross-note commit failure reports honestly, and a retry the same day
     writes no duplicate;
   - existing counted and link tests stay green.
6. Update the docs sentence, then run the checks, bump the version, and run
   `bob plugins sync`.

### mark-tooltip: Freshness mark tooltip shows the logged keeps

In `plugins/bob-ledger-tools/src` (an independent parser copy, because plugins never
import each other):

1. Add a Rotten Log marker and entry recogniser byte-compatible with nav's (a comment
   names the source of truth), plus `freshnessMarkRottenLogLine(history)`:
   - `🥀 3 keeps logged · last Oct 16 — waiting for the weekend reading block`;
   - `🥀 1 keep logged · …` for one entry;
   - the reason truncated to 48 characters with `…`, and `🤷 no reason given` shown as
     is;
   - the same fixed English short date the tooltip already uses;
   - null when the log is absent, empty, or ambiguous.
2. Show the line after the "Kept N reviews in a row" line and before the action line,
   **only** where the task's child block is available from the same source document:
   - Live Preview: `view.state.doc`. Scan only the task's direct child block, bounded by
     indentation.
   - Reading view of the source note: the section source, when `getSectionInfo` provides
     it.

   Tasks query rows, the dashboard, and `rotten.md` query results never show it, and
   their marks must be byte-identical to today's. Keep the line out of the model fields
   that feed `freshnessMarkConsensus`, so query consensus is unaffected. Include the
   line in the mark's `aria-label`.

3. **Tests** (extend `test-ledger-tools-freshness-mark.cjs` and
   `test-ledger-tools-freshness-mark-surfaces.cjs`):
   - wording for 0, 1, and many entries;
   - truncation and `🤷`;
   - the ambiguous log;
   - Live Preview and reading-view surfaces;
   - query rows unchanged;
   - a performance bound for a long child block.
4. Update `docs/bob-ledger-tools.md` (tooltip description) and the manifest description.
   Then run the checks, bump the version, and run `bob plugins sync`.

### mac-agenda: Mac agenda renders Rotten Logs with a 🥀 chip

In bob-mac-capture (thin client: render what bob returns, never parse the vault):

1. In `CaptureAgendaModels.swift`, `CaptureAgendaLog` gains `.rotten` (decoded from
   `"rotten"`), and unknown values still decode to `.other`.
2. In `CaptureAgendaPresentation.swift`:
   - `childRowsForLines` maps `(.logMarker, .rotten)` to a `.logHeader` row "🥀 Rotten
     log" (accessibility "Rotten log") and rotten entries to `.logEntry`, so they fold
     with the other logs;
   - `logChips` adds `🥀 N` after `⚒ N` and `🗓 N`;
   - `accessibilityChip` reads "Show rotten log, N entries".
3. Tests in `CaptureAgendaModelsTests` and the presentation tests: decoding, header,
   entry rows, chip order and count, fold behavior, and the `.other` fallback.
4. Update `docs/idle-agenda.md` (the chip list). Follow the repo `AGENTS.md`: run
   `just portable-check` (on athena if Swift is missing), compile any
   `Sources/BobMacCapture*` change on the Mac (best effort), and report unverified macOS
   targets honestly. Run `just docs-check`.

### docs: Behavior contracts, guide, and rollout record in bob-cli docs

Follow the `docs.md` routing: behavior goes in contracts and references, and guides
change only where steps became wrong.

1. `docs/freshness.md`:
   - §2a: add a "Rotten Log" paragraph. It covers the event rule, a grammar pointer to
     `docs/reference/vault-format.md`, and that the log is history, not a counter field.
     Reconcile the sentence "Do not introduce a second lifetime counter or alias": the
     entry count is display-only and never stored. Add a link to the decision record.
   - §5: only nav writes the log; capture, Rust, and automation never do.
   - §6: the ritual's keep step with the prompt, and the card's Keep stage with "no
     reason? Esc back to the choices".
   - §8 and §11: the new surfaces and the tooltip line.
   - §13: a rollout-log entry with the shipped plugin versions.
   - §14: rollback. Logs remain as plain bullets. An older nav would convert the marker
     into a task during project conversion, so roll forward rather than back.
2. `docs/projects.md` ("Approved-decay decision planner"): replace "Keep and dismissals
   write no log" with the Keep stage and its single Rotten Log entry. Dismissals still
   write nothing.
3. `docs/guides/run-morning-review.md` and
   `docs/concepts/freshness-and-the-review-walk.md`: one short passage each on what a
   keep now asks and where the history shows.
4. Run `just docs-gen` if any generated page changed, then `just docs-check`.

## Verification across the epic

- Rust: `cargo test` plus fmt and clippy per the repo's `justfile`.
- Plugins: `npm run build`, `npm test`, `npm run docs:check`, `bob plugins sync`.
- Mac: `just portable-check`.
- Docs: `just docs-check` in bob-cli.
- A final live check in Obsidian on athena's vault, recorded on the last plugin phase's
  bead, covers one rotten keep with a reason, one blank keep, one Esc, and the
  card-limit Keep stage, and confirms that the log renders under the task.
