---
tier: epic
title: 'Task Decay: one ladder, two stays per rung, one card'
goal: 'Repeated freshness keeps on WIP, Next, Ready, and P-level tasks follow one
  consent-based decay ladder (WIP → Next → Ready → P1 → … → P4): each rung allows
  a configured number of due Alt+F keeps (Bryan: 2), then the next due keep opens
  one rung-aware decay card whose Enter steps the task exactly one rung down and whose
  Alt+F stays and restarts the count; every keep, stay, step, and drop writes one
  rung-qualified Rotten Log entry; WIP keeps log to the Rotten Log instead of the
  Work Log; and the Ready card''s "no priority ladder is configured" bug is fixed.

  '
decisions:
  early_keeps:
    ask: How should Alt+F on a WIP or Next task that is not due (early or linked today)
      behave?
    choices:
      quiet: Stamp quietly with an "early · not counted" notice; no prompt, no log
        entry
      prompt: Prompt and always log an uncounted "kept early in Next" entry; 🤷 when
        blank
    default: quiet
    why: Every entry then marks a real due review, so its count is truthful
    answer: quiet
  memory_task_decay_term:
    ask: Add the new glossary:task-decay strand that defines the unified decay ladder?
    memory:
    - glossary:task-decay
    default: true
    requested: We should add a new glossary term for "task decay" that does a good
      job of explaining this new / improved policy.
    answer: true
  memory_decay_decision:
    ask: Record decisions:task-decay-is-one-ladder and mark the four records it supersedes
      in part?
    memory:
    - decisions:task-decay-is-one-ladder
    - decisions:rotten-keeps-use-priority-decay
    - decisions:rotten-keeps-are-logged
    - decisions:next-keeps-ask-an-optional-rotten-reason
    - decisions:task-lanes-are-sticky
    default: true
    corrects: Next keeps never change `[keeps::]` and never count toward the streak.
    answer: true
  memory_freshness_strands:
    ask: Correct the keep-streak, rotten-log, and task-freshness glossary strands
      for the ladder?
    memory:
    - glossary:keep-streak
    - glossary:rotten-log
    - glossary:task-freshness
    default: true
    corrects: human stamp clears it, and automation never touches it.
    answer: true
phases:
- id: card-hotfix
  title: Decay card hotfix and interim config
  depends_on: []
  size: small
  description: 'card-hotfix: make today''s Ready decay card load the priority ladder
    (never this.config), enter P1 via a >= entry rule, put Enter and the focus ring
    on the same recommended row, give distinct config error copy, add a no-injection
    regression test, and set the interim chezmoi config (keeps 2, enter P1, rolls
    2).'
- id: rust-contract
  title: Rust config, decide, and reconcile contract
  depends_on: []
  size: medium
  description: 'rust-contract: parse per-rung allowances (pending_keeps, next_keeps,
    ready_keeps with keeps fallback), widen decide to the PENDING and NEXT tiers,
    make reconcile lane raises clear [keeps::], pin with a test that a stepped-down
    former prerequisite is not re-raised, bump freshness JSON to schema 15, extend
    the shared vectors, and update the bob-cli contract docs.'
- id: ledger-mirror
  title: bob-ledger-tools config, decide, and mark parity
  depends_on:
  - rust-contract
  size: medium
  description: 'ledger-mirror: mirror the per-rung config and lane decide in bob-ledger-tools
    against the new vectors, show the decay mark on lane rows only when navigation''s
    card capability is at least 3, count lane decisions in the footer, and bump the
    freshness namespace.'
- id: nav-keeps
  title: Lane keep counting and the always-logged Rotten Log
  depends_on:
  - card-hotfix
  - ledger-mirror
  size: medium
  description: 'nav-keeps: count due WIP and Next keeps, ask "Why keep it in <rung>?"
    with the allowance shown, always write a rung-qualified Rotten Log entry (🤷 when
    blank), stop the Work Log prompt on every freshness gesture, ship the shared Rotten
    Log grammar and per-rung tallies in both plugins, and leave lane decisions count-only
    until the card supports them.'
- id: nav-card
  title: One rung-aware decay card
  depends_on:
  - nav-keeps
  size: medium
  description: 'nav-card: generalize the decay card to every rung: Enter steps exactly
    one rung (WIP to Next, Next to Ready, Ready to the entry level, P-level to the
    next, P4 rolls again) while bypassing the roll streak, Alt+F stays and resets
    the count, each answer is one composed and undoable write with its Rotten Log
    entry, batches skip at-limit lane targets, and the card capability rises to 3.'
- id: live-e2e
  title: Real-Obsidian decay ladder end-to-end check
  depends_on:
  - nav-card
  size: medium
  description: 'live-e2e: add an isolated real-Obsidian e2e that walks a scratch vault
    through every rung (keeps, card, step, stay, drop, undo, batch skip), asserts
    the exact task lines and Rotten Log entries, captures the card in light and dark
    themes, and records the result in a live-verification doc.'
- id: memory-config
  title: Glossary, decision record, and final config
  depends_on:
  - nav-card
  size: small
  description: 'memory-config: add the task-decay glossary strand and the task-decay-is-one-ladder
    decision record, correct the keep-streak, rotten-log, and task-freshness strands,
    mark four superseded records, regenerate the docs pages, and replace the interim
    chezmoi keeps with explicit per-rung allowances.'
proposed_by: bbugyi200.apollo.research.32.linker.w0
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-02 15:55:40
status: done
bead_id: bob-cli-g9
---

- **PROMPT:** [prompts/202611/task_decay_ladder.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/task_decay_ladder.md)
- **BEAD:** [bob-cli-g9](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-g9/README.md)

# Plan: Task Decay — one ladder, two stays per rung, one card

## Context

Bob already asks a real question about a due Ready task after repeated Alt+F keeps (the
approved-decay card). WIP (`[/]`) and Next (`[*]`) tasks never get that question. Over
the last 22 review days, Next rows were kept 88 times (five tasks 11–12 times each),
there are zero Next Rotten Log entries because every Next reason was blank, and WIP
holds 14 tasks against a soft cap of 8. The research report
`research:202611/task_decay_ladder/task_decay_ladder.md` (read it with
`sase artifact read` before starting any phase) verified the current code and
recommended the design below. Bryan agreed with all of its requirements. This plan
adopts them, with two verified adjustments called out in
[Adjustments to the research](#adjustments-to-the-research).

Bryan's Ready decay card is broken today (it says "no priority ladder is configured",
and Enter actually runs Keep). Three stacked defects cause this, and the code confirms
each one:

- **A — the card never loads the priority config.**
  `530-plugin-freshness-refresh-and-decay.js` calls
  `findFreshnessDecayPriorityProperty(this.config)` at lines 157, 365, 558, and 589, and
  `540-plugin-decay-picker-and-cancel.js:24` passes `this.config` to the Less-often
  picker. The plugin never assigns `this.config`; only `BulletPropertyPickerModal` does
  (`340-picker-core.js:9`). The planner therefore always takes the
  `notNow = unavailable("no priority ladder is configured")` branch. The handler test
  suite hides this by injecting `plugin.config` by hand.
- **B — P0 entry would skip P1.** `resolveFreshnessDecayP0EntryLevel`
  (`040-priority-roll.js`) enters the first level with `minDays > interval`. With
  `interval: 3` and P1 `min_days: 3`, it picks P2. Task Card Ctrl+Enter on an
  unprioritized task picks P1.
- **C — Enter runs Keep.** `FreshnessDecayCardModal.onOpen` calls
  `focusKeepRowOrFirst()`. Each focused row's `keydown` catches Enter and calls
  `stopPropagation()`, so the content-level `Enter → notNow` handler never fires. The
  chip, badge, and focus ring disagree about what Enter does.

"Always use P1, right?" — yes for Bryan, and the code will agree without inventing
levels. Pin `enter: P1` in config, and relax the automatic entry rule to
`min_days >= interval`. The relaxed rule still yields P2 for the documented interval-7
example, yields P1 for Bryan, and never enters a level that returns sooner than a plain
keep. Never hardcode a P1 write when no ladder loads, because that would invent a Tasks
value the Task Card cannot roll.

## The design

### Vocabulary

- **Rung**: where a task sits on the ladder.
  - **WIP** is a `[/]` task (machine lane `pending`; R12 still calls it "In Progress").
  - **Next** is a `[*]` task.
  - **Ready** is a `[ ]` task with no priority (implicit P0).
  - A `[ ]` task with `[priority::]` sits on that **P-level** (P1–P4 for Bryan).
  - Blocked is an overlay, never a rung.
- **Keep**: a due Alt+F / Ctrl+Alt+F confirmation that counts in `[keeps:: N]`.
- **Allowance**: how many keeps in a row a rung permits (Bryan: 2 on every rung).
- **Step**: move exactly one rung down.
- **Stay**: answer the card by keeping the task on its rung. This restarts the count.

User-facing copy says **WIP**. Config keys and JSON say `pending`, matching
`pending_interval` and `plan.max_pending`.

### The ladder

```
WIP ──↵──▶ Next ──↵──▶ Ready ──↵──▶ P1 ──↵──▶ P2 ──↵──▶ P3 ──↵──▶ P4 ──↵──▶ P4 (rolls again)
 each rung: 2 keeps, then the next due Alt+F opens the card · Alt+F on the card stays · only D drops
```

Each rung allows `N` keeps. A rung's `N+1`th due Alt+F writes nothing and opens the
decay card. **Enter** steps one rung down, **Alt+F** stays (with an optional reason) and
restarts the count, **E** rewords, and **D** drops. Enter never drops a task, and
nothing moves without a keypress. Nothing decays on a timer. This is what separates the
ladder from the age-based decay that the sticky-lanes decision rejected.

The Ctrl+Enter roll streak on P-levels (`properties.priority.decay.rolls`, Schedule Log
derived) keeps governing Ctrl+Enter on the Task Card. Bryan sets `rolls: 2`, which means
the same thing: two same-level rolls, then the next recommended roll decays. The decay
card **does not stack** that threshold on top of the keep threshold. Its Enter steps one
level directly, reusing the ladder, the window maths, and the existing writers, but
skipping the roll-streak recommendation that today's "Not now" delegates to.

### Counting rules

- **A keep counts** only when the pre-write queue holds exactly this row (path, line,
  raw text) on its rung's due tier:
  - PENDING for WIP;
  - NEXT for Next;
  - ROTTEN or scheduled TICKLER for Ready and P-levels.

  `api.freshness.keepLine` must also see a valid prior `fresh < today`. A counted keep
  increments once per review day and saturates at 999.

- **No count, streak preserved.** These keeps stamp through the existing uncounted path:
  - NEW rows;
  - early or FRESH rows;
  - same-day repeats (byte-identical);
  - Today-linked rows;
  - PROJECTS trackers;
  - recurring rows;
  - checklist rows;
  - unblock ticklers (`returned_by: unblocked`);
  - unresolved cache matches;
  - Pomodoro ledger lines.

  > [!decision] early_keeps = quiet Early and Today-linked WIP/Next keeps stamp quietly
  > with an "early · not counted" notice: no prompt and no Rotten Log entry. (Today,
  > every Next keep prompts, even an early one.)

  > [!decision] early_keeps = prompt Early and Today-linked WIP/Next keeps still prompt
  > "Why keep it in <rung>?" and always log an uncounted
  > `<reason> · kept early in <Rung>` entry (🤷 when blank). The streak is unchanged.

- **Every rung change clears the streak, whoever makes it.**
  - Human lane and priority writers already stamp, which clears `[keeps::]`. nav-keeps
    verifies each one.
  - Reconcile's lane raises (`MarkNext`, `MarkInProgress`, and `Unblock` into Next or
    WIP) must also strip `[keeps::]` in the same line write. Otherwise a Ready streak
    leaks into Next after an automatic raise and opens a false "Kept 2× in Next" card.
  - `Unblock` back to Ready preserves the streak, consistent with `unblock_keeps`.
  - Hand edits remain unobserved.
- **At the allowance**, a single due Alt+F / Ctrl+Alt+F opens the card and writes
  nothing. Counted `N<Alt+F>` batches and Task Link sessions skip at-limit targets ("N
  need a decision"), including lane targets once the card supports lanes.

### One card for every rung

The Enter row always names the destination, always holds the focus, and is the only row
with the "recommended" badge. Mockups:

```text
🍂  Kept 2× in WIP
    Write migration notes · project.md · captured Oct 28
    🥀 WIP 2 · last Nov 3 — waiting on the agent epic

 ↵      Move to Next                              recommended
        [/] → [*] · still planned, no longer in progress
 Alt+F  Stay in WIP
        say why · logs 🥀 · count starts over
 E      Reword       edit the task · count starts over
 D      Drop         cancel · dropped after 2 keeps in WIP

 Esc changes nothing · ↵ never drops · Ctrl+Z undoes
```

```text
🍂  Kept 2× in Ready
    foo bar baz · sase.md · captured Oct 30
    🥀 Next 2 · Ready 2 · last Nov 2 — 🤷 no reason given

 ↵      Defer to P1                               recommended
        Ready → P1 · back in 5 days (3–8)
 Alt+F  Stay in Ready
        say why · logs 🥀 · count starts over
 L      Less often   review every 7 days instead of 3
 E      Reword       edit the task · count starts over
 D      Drop         cancel · dropped after 2 keeps in Ready
 1–4    Pick a level P1 6d · P2 20d · P3 61d · P4 190d

 Esc changes nothing · ↵ never drops · Ctrl+Z undoes
```

The other cards follow the same pattern:

- **Next** cards read "Move to Ready · [*] → [ ] · back in the Ready backlog".
- **P-level** cards are titled "Kept 2× at P2" and read "Defer to P3 · P2 → P3 · back in
  41 days (32–100)".
- **P4** cards read "Roll again at P4 · bottom of the ladder · back in N days".
- **Stay replaces today's Keep on every card, Ready included.** Today's Keep increments
  and asks again at the next review, which contradicts the requested reset. Stay reuses
  the in-card reason stage, titled "Stay in <rung>?". Blank Enter logs 🤷, and Esc
  returns to the rows.
- **Lane cards omit Less often and 1–4.** Lane cadence overrides `[refresh::]`, and
  jumping from Next straight to a P-level would skip rungs.
- **Header line 3** shows per-rung keep tallies derived at read time from the Rotten Log
  (legacy entries included). It replaces "N keeps logged" and answers "how many times
  was it kept in each status" without adding a field.
- **When a step is unavailable** (for example, an invalid ladder), its row explains why
  ("priority config is invalid", "no `priority` property in bob/config.yml", "config
  file missing", "config unreadable"). Stay becomes the recommended, focused Enter row.
- **The footer is built from the rows that exist.** "1–4 pick a P-level" appears only
  when level rows exist.
- **Opening the card ignores Enter** until the opening chord's keys are released, so a
  held Alt or key repeat never answers it.
- **Every committed answer shows a notice** naming the move plus "Ctrl+Z undoes", for
  example "Next → Ready · Ctrl+Z undoes".
- **Walk rules are unchanged.** Alt+F stays on the row; Ctrl+Alt+F advances exactly once
  after a committing answer; Esc and failed validation never advance. The review journal
  records `decay:step` and `decay:stay` outcomes (Rust stats accept any outcome string).

### Rotten Log grammar

The Rotten Log records every **rung event**: a counted keep, a Stay, a step, or a drop.
Reword and Less often do not change the rung. They keep their existing Schedule Log
review-decision entries, whose `kept N×` tails gain the rung (`kept 2× in Ready`). Those
entries intentionally break the Ctrl+Enter roll streak, and moving them would silently
change that behavior.

Entries stay `*YYYY-MM-DD* — <body>` lines under the `🥀 **ROTTEN LOG**` direct child,
newest first. `<Rung>` is `WIP`, `Next`, `Ready`, or the configured level label.
P-levels use "at", and lanes and Ready use "in". The count uses `×` (U+00D7) and "after
1 keep" is singular.

| Event                      | Entry body                                                                                       |
| -------------------------- | ------------------------------------------------------------------------------------------------ |
| Counted keep               | `waiting on Alice · kept 1× in Next` · `kept 1× at P2`                                           |
| Counted keep, blank reason | `🤷 no reason given · kept 2× in WIP`                                                            |
| Stay (count restarts)      | `still the next thing · stayed in Next after 2 keeps`                                            |
| Lane or priority step      | `🍂 WIP → Next · after 2 keeps` · `🍂 Ready → P1 · after 2 keeps` · `🍂 P2 → P3 · after 2 keeps` |
| P4 re-roll                 | `🍂 rolled again at P4 · after 2 keeps`                                                          |
| Drop from the card         | `🍂 Next → dropped · after 2 keeps` (plus the existing Cancel Log entry)                         |
| Legacy, parsed forever     | `<reason> · kept N×` reads as Ready; `<reason> · kept in Next` reads as an uncounted Next keep   |

A Ready → P1 or P-level step also writes its existing Schedule Log `🎲` decay entry,
with a rung-qualified `kept N× in Ready` / `kept N× at P2` tail, in the same
transaction.

Example timeline (newest first):

```markdown
- 🥀 **ROTTEN LOG**
  - _2026-11-09_ — 🍂 Next → Ready · after 2 keeps
  - _2026-11-08_ — 🤷 no reason given · kept 2× in Next
  - _2026-11-07_ — waiting on Alice's review · kept 1× in Next
  - _2026-11-06_ — 🍂 WIP → Next · after 2 keeps
  - _2026-11-05_ — 🤷 no reason given · kept 2× in WIP
  - _2026-11-04_ — still mid-refactor · kept 1× in WIP
```

The grammar lives in one shared fixture inside bob-plugins, mirrored byte-for-byte by:

- navigation's `ROTTEN_LOG_ENTRY_RE` (`010-requires-and-config.js`) and
  `formatRottenLogEntryText` / `parseRottenLogEntryBullet` (`195-rotten-keep.js`);
- ledger-tools' `FRESHNESS_ROTTEN_ENTRY_RE` (`135-freshness-rotten-log.js`).

bob-cli's `docs/reference/vault-format.md` owns the prose grammar. Rust only treats the
marker as a drawer and never parses entries, so it needs no parser change.

**Always log.** If a counted keep, Stay, step, or drop cannot write its entry (duplicate
or ambiguous 🥀 markers, unreadable history), the whole answer writes nothing and shows
a repair message: "Repair the duplicate 🥀 ROTTEN LOG markers first; nothing was
written". This replaces today's "blank keep with no entry" fallback.

### Prompts by surface

- **Single due counted keep, any rung.** The prompt asks "Why keep it in WIP?" / "in
  Next?" / "in Ready?" / "at P2?".
  - A subtitle states the allowance: "keep 1 of 2 in Next", or "keep 2 of 2 in Next ·
    the next due keep opens the decay card". When the rung's allowance is `false`, it
    reads "keep 3 in Next · decay off for Next".
  - Enter writes the entry; blank writes 🤷.
  - Esc writes nothing: no stamp, no entry, no advance.
- **Counted `N<Alt+F>` batch and Task Link session refresh.** One optional Rotten reason
  applies to every counted target, each logged with its own rung tail. At-limit targets
  are listed for individual decisions. The Work summary field is removed from these
  freshness prompts.
- **WIP no longer asks for a Work Log summary on any freshness gesture.** The Work Log
  keeps its real writers: Ctrl+Shift+Enter unlink, Alt+N release, the `=x` close, and
  Task Link pause.

### Configuration

Flat per-rung keys inside `freshness.decay` are forward-compatible: `parse_decay_config`
ignores unknown keys, so an older bob on another host keeps working, while an int-or-map
`keeps` would make an older bob exit 2.

```yaml
freshness:
  decay: # absent/null/true/{} = on; false = count and log, never ask
    keeps: 3 # default allowance for every rung (0–999); code default stays 3
    pending_keeps: 2 # WIP [/] override: 0–999 or false
    next_keeps: 2 # Next [*] override: 0–999 or false
    ready_keeps: 2 # Ready and every P-level override: 0–999 or false
    enter: P1 # unchanged
```

- **Resolution.** A rung's effective allowance is its `*_keeps` key, then `keeps`,
  then 3.
  - `false` on one rung (or `decay: false` overall) means count and log, but never ask.
  - `0` asks on every due keep for that rung, never on NEW.
- **Errors.** Invalid values follow the existing freshness failure contract: Rust exits
  2, and plugins fall back to defaults with `invalid` diagnostics.
- **Per-level `rolls`.** The priority property's per-level `rolls` override is
  unaffected.

Bryan's final chezmoi block in `home/dot_config/bob/config.yml` merges into the existing
`freshness:` block:

```yaml
# Task decay (bob-cli docs/freshness.md §2a): each rung allows N due Alt+F keeps, and
# the next due keep opens the decay card. Enter steps one rung down (WIP → Next →
# Ready → P1 → … → P4; never drops); Alt+F stays and restarts the count. Every keep,
# stay, and step writes a 🥀 ROTTEN LOG entry. false = count and log, never ask.
decay:
  pending_keeps: 2 # WIP [/] → Next
  next_keeps: 2 # Next [*] → Ready
  ready_keeps: 2 # Ready → P1, and each P-level → the next
  enter: P1 # unprioritized Ready tasks enter the ladder at P1
```

On the priority property, replace the commented `# decay: / #   rolls: 1` lines (keep
the explanation above them) with:

```yaml
decay:
  rolls: 2 # two same-level Ctrl+Enter rolls per level, then the next one decays
```

`rolls: 2` is a real behavior change, not just an explicit default: each level gets a
third window. That is exactly the requested "2 per priority, then roll on". The code
default for `rolls` stays 1, so the shared roll vectors are not rewritten.

### Reliability

- **One composed write per answer.** The stamp, the rung change, the Rotten Log entry,
  and any Schedule Log or Cancel Log entry land in one editor transaction with one undo.
- **Revalidate before writing.** Every answer first rechecks:
  - the live line;
  - the local day;
  - the reloaded decay config and priority ladder (compare serialized snapshots, never
    two `null`s);
  - eligibility and Today membership;
  - the child-block preimages.

  Stale input writes nothing and rebuilds the card. Never commit a priority change and
  then append the log in a second write.

- **Version skew.** The ledger shows lane decay marks only when navigation's card
  capability is 3 or higher. An older navigation running on a newer ledger stamps lane
  rows without a card, so tasks under-roll instead of moving by surprise.
- **No backfill.** Lane streaks start at 0, and legacy log entries parse as they are.

## Adjustments to the research

1. **No `held` flag for dependency-promoted lane rows.** The research worried that
   reconcile would re-raise a Next task stepped down to Ready when today's linked work
   depends on it. The code rules out a steady-state path:
   - a linked task with an open prerequisite is Blocked (`has_derived_block`);
   - `consume_blocked_links_after_compose` (`src/native/task_status_hooks/sync.rs`)
     consumes that Blocked task's Task Link in the same run that raises its
     prerequisites;
   - so `desired_statuses` no longer reaches the prerequisite on the next run, and the
     sticky lane is the only thing keeping it at Next.

   Mirroring `desired_statuses` into three codebases would buy nothing. rust-contract
   adds a regression test that pins this behavior instead. If the worker finds a real
   steady-state re-raise path, they record a `PROPOSED FOLLOW-UP:` note rather than
   widening scope.

2. **Reword and Less often stay in the Schedule Log** (see
   [Rotten Log grammar](#rotten-log-grammar)). The Rotten Log owns rung events. Those
   two answers change text or cadence, and their entries load-bear on the roll streak.

## Phase details

Every phase that edits bob-plugins or chezmoi opens it with `/sase_repo`
(`sase repo open bob-plugins …`, `sase repo open chezmoi …`) and reads that repo's
`AGENTS.md` first. bob-plugins workflow:

- edit `src/` fragments (each at most 1000 lines);
- run `npm run build`, `npm test` (it includes `build:check`), and `npm run docs:check`;
- bump each touched plugin's manifest version the way recent commits do;
- run `bob plugins sync` after the changes.

In bob-cli, read `sase/memory/docs.md` (via `/sase_memory_read`) before touching
`docs/`, and run `just docs-check`. Never edit `Cargo.toml`'s version or `CHANGELOG.md`.

### card-hotfix

This phase touches bob-plugins (navigation) and chezmoi, and ships alone.

- **Defect A.** When the card context is built, load the normalized config through
  `loadBulletPropertyConfig()` and keep a snapshot on the card context (property plus
  its serialization via `serializeFreshnessDecayPriorityProperty`). Use the snapshot, or
  a re-load compared against it, at every former `this.config` site in `530` and `540`.
  Never read `this.config` on the plugin.
- **Distinct error copy.** Give each failure its own message instead of the single "no
  priority ladder is configured": config file missing, config unreadable, config
  invalid, and no `priority` property.
- **Defect B.** Change `resolveFreshnessDecayP0EntryLevel` to `minDays >= interval`.
  Update its tests and the entry-rule prose in bob-cli `docs/projects.md`
  ("Approved-decay decision planner") and `docs/freshness.md` §2a.
- **Defect C.** Focus the row that Enter runs (the recommended row; today that is Not
  now when available) so the focus ring, the chip, and Enter agree. Build the footer
  from the rows that exist.
- **Regression tests.** Add a handler test that enters through the real card handler
  with no injected `plugin.config` and a real config loader stub, and asserts that P1 is
  offered for Bryan's ladder (`interval: 3`, P1 3–8). Also cover the changed-mid-card,
  malformed, and missing config cases. Remove reliance on the
  `plugin.config = decayConfig()` injection.
- **Interim chezmoi config.** Add `decay: {keeps: 2, enter: P1}` to `freshness:` with a
  short comment, and set the priority property's `decay: {rolls: 2}`. Both are valid
  today, so Bryan immediately gets Ready decay at 2 keeps with P1 entry. memory-config
  later replaces `keeps: 2` with the per-rung keys.

### rust-contract

This phase touches bob-cli.

- **Config.** In `src/native/config/freshness.rs`:
  - extend `DecayConfig` with per-rung allowances (`pending`, `next`, `ready`), each
    "ask at N" or "never ask" (`false`), resolved through the `keeps` fallback;
  - parse `pending_keeps`, `next_keeps`, and `ready_keeps` (integer 0–999 or `false`,
    with the same error contract as `keeps`);
  - keep the `keeps` default at 3;
  - add parser tests for 0, 2, 999, `false`, the fallback, and invalid values.
- **Decide.** `decide_for` in `src/native/freshness/state.rs` becomes
  `enabled && limit(lane) is "ask at L" && keeps >= L && due tier for that lane`, where
  the due tier is PENDING for pending, NEXT for next, and ROTTEN/TICKLER for ready.
  Unblock ticklers, PROJECTS, recurring, and checklist overlays still never decide.
  Today-linked lane rows never decide either: if they can carry a due tier, exclude
  `is_today` explicitly. Pin this with a test.
- **Reconcile.** In `src/native/task_status_hooks/sync.rs`, the `MarkNext`,
  `MarkInProgress`, and `Unblock(Next | InProgress)` line writes strip every `[keeps::]`
  occurrence in the same write. Reuse the existing keeps-clearing helper so the Tasks
  suffix and placement rules hold. `Unblock(Ready)` preserves the streak.
  - Add a test: a Ready task with `[keeps:: 2]` linked today is raised to Next without
    `keeps`.
  - Add the [Adjustment 1](#adjustments-to-the-research) regression test: a prerequisite
    raised by a Blocked dependent whose link was consumed, then hand-set to `[ ]`, stays
    `[ ]` on the next reconcile.
- **JSON.** Bump `bob freshness` JSON to schema 15. The `decay` object gains
  `limits: {pending, next, ready}` (effective integer or `false`), and `decide` can now
  be true on lane rows. Document the bump in the `SCHEMA_VERSION` comment, and mark the
  commit as a contract break with `!`.
- **Human header.** The meter reads `keeps 2` when all rungs are equal, and otherwise
  `keeps WIP 2 · Next 3 · Ready 2`. A `false` rung shows `off`, and the overall
  `· decay off` suffix stays.
- **Vectors.** Extend `tests/fixtures/freshness_keeps/vectors.json` with per-rung config
  cases and lane decide cases (PENDING and NEXT tiers at, below, and above the limit;
  `false`; Today-linked; fallback). Keep the existing cases byte-stable.
- **Docs.**
  - `docs/freshness.md`: rewrite §2a as "Keep streaks and task decay" (storage, counting
    rules, the configuration block, the decide predicate, reconcile clearing; leave the
    card and Rotten Log details to the later phases, with a forward note); update §4 and
    §7 (JSON and header) and §10a (vectors).
  - `docs/reference/vault-format.md`: the `[keeps::]` semantics.

### ledger-mirror

This phase touches bob-plugins (`bob-ledger-tools`).

- **Config.** `090-freshness-config.js` parses the per-rung keys exactly like Rust, with
  invalid values falling back to defaults marked invalid.
- **Decide.** `100-freshness-evaluate.js` (`freshnessDecideFor`) mirrors the Rust
  predicate.
- **Vector mirrors.** Update `scripts/test-ledger-tools-freshness-keeps.cjs` with the
  new config and decide vectors from bob-cli's fixture.
- **Footer.** The footer's "need a decision" counts include lane rows.
- **Marks.** The 🍂 decide mark renders on lane rows only when navigation's exported
  `FRESHNESS_DECAY_CARD_VERSION` is 3 or higher. Read it through the same capability
  probe used today for Ready pips. Below 3, lane rows show their pips without the decide
  promise.
- **Namespace.** Bump the freshness namespace (v8 → v9) so navigation can detect
  per-rung support.
- **Docs.** Update `docs/bob-ledger-tools.md` and bob-cli `docs/freshness.md` §11/§12
  for lane marks.

### nav-keeps

This phase touches bob-plugins (navigation and ledger-tools' Rotten Log parser) and
bob-cli docs.

- **Counting.** `classifyRottenKeepTarget` (`195-rotten-keep.js`) accepts exact due lane
  rows (pending/PENDING, next/NEXT) in addition to Ready ROTTEN/TICKLER, and returns the
  rung (WIP, Next, Ready, or the P-level label) plus the before and after keeps. All
  [exclusions](#counting-rules) hold. Early and Today-linked lane keeps follow
  `early_keeps`.
- **Prompts.** Implement the [prompts by surface](#prompts-by-surface): rung-aware
  titles, the allowance subtitle, always-log with 🤷, and Esc writes nothing.
- **Work Log removal.** Remove the Work summary field and the WIP Work Log path from
  every freshness gesture: the single keep, `planFreshnessRefreshPrompt` (`197`), the
  batch prompt (`192`), `520-plugin-lane-links-and-review.js`, and the Task Link session
  refresh (`532`). Leave the other Work Log writers untouched.
- **Grammar.**
  - Add the shared grammar fixture (for example
    `scripts/fixtures/rotten-log-grammar.json`) with format and parse vectors for every
    row of the [grammar table](#rotten-log-grammar), including singular and plural
    forms, P-level "at", legacy forms, and rejects (plus `kept early in <Rung>` if
    `early_keeps = prompt`).
  - Update `formatRottenLogEntryText`, `parseRottenLogEntryBullet`,
    `ROTTEN_LOG_ENTRY_RE`, and ledger-tools' `FRESHNESS_ROTTEN_ENTRY_RE` together, with
    tests in both plugins that read the fixture.
  - Add a per-rung tally helper shared in spirit by both plugins. Ledger tooltips and
    the card header show `🥀 WIP 2 · Next 1 · Ready 3`.
- **Always-log contract.** Implement the duplicate-marker refusal from
  [Rotten Log grammar](#rotten-log-grammar).
- **Lane writers clear the streak.** Verify that every human lane writer (Alt+N commit
  and release, the Task Link lane raise, the Ctrl+Shift+P lane row) stamps and so clears
  `[keeps::]`. Fix any that do not.
- **Interim guard.** Until nav-card raises the card capability, lane rows with
  `decide: true` count and log like any keep: no card and no batch skip. Gate this on
  `FRESHNESS_DECAY_CARD_VERSION < 3` so nav-card only has to bump it.
- **Docs.** Update `docs/bob-navigation-hotkeys.md` and `docs/bob-ledger-tools.md`, plus
  bob-cli `docs/freshness.md` §2a (the Rotten Log paragraph and prompts) and
  `docs/reference/vault-format.md` (the full grammar).

### nav-card

This phase touches bob-plugins (navigation) and bob-cli docs.

- **Planner.** Generalize `planFreshnessDecayCard` (`050-decay-planner.js`) so it takes
  the rung and returns the rows of [One card for every rung](#one-card-for-every-rung):
  `step` (Enter, recommended), `stay` (Alt+F), `lessOften` and 1–4 (Ready lane only),
  `reword`, and `drop`. It also returns the title, the destination copy, and the header
  tallies. The P0 step uses the hotfixed entry resolution. A P1–P3 step targets the next
  level with a fresh date in that level's window, bypassing
  `planPriorityRollRecommendation`. A P4 step is a same-level roll at the last level and
  never cancels.
- **Modal.** Rework `FreshnessDecayCardModal` (`060-decay-card-modal.js`):
  - rename Keep to Stay, with a "Stay in <rung>?" reason stage;
  - focus the recommended row, so Enter follows focus;
  - build the footer from the rows;
  - add the Enter-after-chord-release guard;
  - label the dialog and input for accessibility.
- **Writers** (`530` / `540`), each a single composed transaction behind `revalidate`:
  - **WIP → Next:** `[/]` → `[*]` through the existing lane planner (`planTaskLaneBatch`
    or the Task Link lane planner), without a Work Log, plus the stamp and the 🍂 entry.
  - **Next → Ready:** the release path minus the Work Log (a due counted row has no live
    Today links), plus the stamp and the 🍂 entry.
  - **Priority steps:** the existing decay writer (priority field, rolled `scheduled`,
    Schedule Log `🎲` with the rung tail) composed with the 🍂 entry.
  - **Stay:** stamp, which clears `[keeps::]`, plus the `stayed in … after N keeps`
    entry.
  - **Drop:** the existing cancel and Cancel Log, plus the 🍂 `→ dropped` entry.
  - **Reword and Less often:** unchanged apart from the rung-qualified tails.
  - Record `decay:step` and `decay:stay` journal outcomes.
- **Batches.** Counted batches and Task Link sessions skip at-limit lane targets too.
- **Capability.** Bump `FRESHNESS_DECAY_CARD_VERSION` to 3, which also lights the
  ledger's lane marks.
- **Tests.** Cover each rung's `0 → 1 → 2 → card` sequence:
  - each step moves exactly one rung, and a P-level step ignores the roll streak;
  - Stay resets the count and allows two more keeps;
  - each answer is one transaction, and undo restores everything;
  - stale or duplicate markers write nothing;
  - Enter follows focus, Ctrl+Alt+F advances once, Esc never advances;
  - batches skip at-limit targets;
  - an unavailable step makes Stay the Enter row.
- **Docs.** Update bob-cli `docs/freshness.md` §2a (availability and the card) and §14
  (rollout), `docs/projects.md` ("Approved-decay decision planner"), and bob-plugins
  `docs/bob-navigation-hotkeys.md` and `docs/task-card.md`.

### live-e2e

This phase touches bob-plugins scripts and a bob-cli verification doc.

- **The e2e.** Following `scripts/obsidian-frees-marks-e2e*.cjs` (isolated scratch
  vault, `xvfb-run`, real Obsidian), add `scripts/obsidian-decay-ladder-e2e*.cjs`. With
  a config of 2 keeps on every rung, `enter: P1`, and `rolls: 2`, it drives Alt+F across
  simulated review days on one WIP task, one Next task, one Ready task, and one P4 task:
  - assert the exact task lines and Rotten Log entries after every keep, card answer
    (step, stay, drop), and Ctrl+Z;
  - assert that a counted batch skips at-limit targets;
  - capture the WIP, Ready, and P4 cards in light and dark themes for review.
- **Verification doc.** Record the run, its pass count, and the screenshots in a short
  bob-cli `docs/decay-ladder.md` live-verification page, matching `docs/frees-marks.md`.
  List any human or device checks that remain, such as the Mac smoke test.

### memory-config

This phase touches bob-cli memory and generated docs, plus chezmoi. Use
`/sase_memory_write` for every memory edit, and stay inside the accepted memory
decisions.

> [!decision] memory_task_decay_term **New strand `glossary:task-decay`.** Title: Task
> Decay. Aliases: decay, decay ladder, decay card, step down. It defines:
>
> - the ladder and its rungs, with WIP listed as the alias of In Progress;
> - keeps, allowances (the `freshness.decay.*_keeps` keys and the priority
>   `decay.rolls`);
> - the card answers: Enter steps, Alt+F stays and restarts the count, E rewords, D
>   drops;
> - that Enter never drops and P4 rolls again;
> - that nothing moves on a timer;
> - the Rotten Log entries that every keep, stay, step, and drop writes;
> - that Blocked is an overlay.
>
> It links `[[keep-streak]]`, `[[rotten-log]]`, `[[schedule-log]]`, and
> `[[decisions/task-decay-is-one-ladder]]`.

> [!decision] memory_decay_decision **New record `decisions:task-decay-is-one-ladder`.**
>
> - **Applies to:** bob-cli, bob-plugins, and the vault.
> - **Claim:** this plan's design.
> - **Why:** cites the journal evidence (88 Next keeps in 22 review days, zero Next
>   Rotten Log entries, WIP 14 against a cap of 8) and
>   `research:202611/task_decay_ladder/task_decay_ladder.md`.
> - **Rejected:** silent auto-demotion, lengthening intervals only, per-status lifetime
>   fields, an int-or-map `keeps`, folding rolls into `[keeps::]`, and a `held` flag.
> - **Cost:** card volume (about 1.7 a day at steady state), muscle memory (mitigated by
>   destination copy and undo), and three implementations kept in sync.
> - **Reopens when:** Stay dominates card answers for a month, or WIP grace proves too
>   short for agent epics.
>
> Mark each superseded-in-part record (`metadata.status: superseded-in-part`, append to
> `superseded_by`, and add one `[[decisions/task-decay-is-one-ladder]]` back-link
> sentence stating what was retired; never reword accepted bodies):
>
> - `rotten-keeps-use-priority-decay`: limit 3, Ready-only counting, Keep counting
>   again, and the P-level Not now reusing the roll recommendation.
> - `rotten-keeps-are-logged`: the Pending/Next exclusions and Pending summaries in the
>   Work Log.
> - `next-keeps-ask-an-optional-rotten-reason`: blank writes nothing, Next uncounted,
>   and early prompts (per `early_keeps`).
> - `task-lanes-are-sticky`: "only release lowers Next or Pending" and the rejection of
>   age-based Next decay. Consent-based ladder steps are not age-based.

> [!decision] memory_freshness_strands **Strand corrections.**
>
> - `keep-streak`: the current-rung streak; lanes count; any rung change clears it,
>   reconcile included; at the allowance the decay card asks.
> - `rotten-log`: the new grammar; WIP logs; blank answers always log 🤷; stays and
>   steps.
> - `task-freshness`: replace the decay sentence with a pointer to `[[task-decay]]`.
>   Freshness stamps alone never change a rung; only explicit card answers do.

Then:

- run `sase memory init`, `just docs-gen` (which regenerates
  `docs/reference/glossary.md` and `docs/develop/decisions.md`), and `just docs-check`;
- in chezmoi, replace the interim `keeps: 2` with the final per-rung block from
  [Configuration](#configuration), keeping `enter: P1` and `rolls: 2`;
- follow chezmoi's `AGENTS.md` (`chezmoi update -a --force` after its commit lands), and
  tell Bryan that athena and the Mac need the same update.

## Acceptance (epic)

1. The Ready card on Bryan's real config offers "Defer to P1" with no injected config.
   Bad configs show their specific message. A config change mid-card writes nothing.
2. Every rung follows `0 → 1 → 2 → card`, and Enter moves exactly one rung: WIP→Next,
   Next→Ready, Ready→P1, P1→P2, P2→P3, P3→P4, P4→P4. Only D drops.
3. Stay writes `stayed in <Rung> after N keeps`, clears `[keeps::]`, and allows two more
   keeps.
4. A blank WIP/Next/Ready keep writes a 🤷 entry with its rung tail. Freshness gestures
   never write or prompt for the Work Log.
5. The [counting exclusions](#counting-rules) hold. A reconcile lane raise clears the
   streak, so no false "Kept 2× in Next" card appears.
6. Each decision is one transaction with one undo. Stale input or duplicate markers
   write nothing.
7. Rust and both plugins agree on configs 0, 2, 999, `false`, the fallback, and invalid
   values, and on the Rotten Log vectors, legacy forms included.
8. The real-Obsidian e2e passes, and the card screenshots look right in both themes.
