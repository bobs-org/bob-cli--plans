---
tier: epic
title: Inline task mentions on 🪢 LINKED FROM
goal: "When one task's own text links another task (`[[note#^id]]`, embedded or struck
  too), the target's generated 🪢 LINKED FROM line gains an inert `[[S]] (💬 mentions)`
  entry that explains itself at a glance with a `💬 mentioned in` / `💬 embedded in`
  pill, a live quote, and a context card that jumps to the sentence. Nothing is written
  on the mentioning task except a silent `^id`, a dependency or relation on the pair
  wins, and Rust and both plugins agree on every shared vector.

  "
decisions:
  rollout:
    ask:
      Should the last phase switch mentions on by default once the vault-copy pilot
      passes?
    choices:
      enable: Mentions go live with the next release train; no config edit is needed
      dark:
        Everything ships off; Bryan opts in later with relations.linked_from_mentions
    default: enable
    why: The phase graph already deploys the plugins before the writer goes live.
    answer: enable
  memory_record:
    ask:
      Should the decisions web gain a record that task mentions are inert LINKED FROM
      entries?
    memory:
      - decisions:task-mentions-are-inert-linked-from-entries
    default: true
    requested:
      I agree with all of the requirements recommended in that research file (but keep
      in mind that you are responsible for planning and splitting up this work
      appropriately).
    answer: false
  memory_glossary:
    ask:
      Should the glossary add Task Mention and update Task Relation and Task Link to
      match?
    memory:
      - glossary:task-mention
      - glossary:task-relation
      - glossary:task-link
    default: true
    requested:
      I agree with all of the requirements recommended in that research file (but keep
      in mind that you are responsible for planning and splitting up this work
      appropriately).
    answer: true
phases:
  - id: contract
    title: Mention contract, shared vectors, decision record, and glossary
    depends_on: []
    size: medium
    description:
      "contract: write task-relations §19 plus the amended sections, the shared MN
      vector fixture, the config/JSON/guide docs, the decision record, and the glossary
      terms."
  - id: cli-core
    title: Rust mention scanner, reserved label, and writer guards
    depends_on:
      - contract
    size: medium
    description:
      "cli-core: add task_relations/mentions.rs driven by the shared vectors, reserve 💬
      labels in every parser and authored writer, and stop relations reverse from
      dropping mention entries."
  - id: cli-reconcile
    title: Reconcile projects mentions onto LINKED FROM
    depends_on:
      - cli-core
    size: medium
    description:
      "cli-reconcile: fold mention pairs into reconcile_relation_lines with precedence,
      minting, uncertainty, widened prefilters, the dark linked_from_mentions switch,
      and additive JSON and human output."
  - id: cli-capture
    title: Capture stages mention mirrors in its batch
    depends_on:
      - cli-reconcile
    size: medium
    description:
      "cli-capture: diff the mention pairs of every note a capture writes and stage the
      targets' LINKED FROM entries and source mints in the same all-or-nothing batch."
  - id: plugin-index
    title: Live mention index and relationIndex v2
    depends_on:
      - contract
    size: medium
    description:
      "plugin-index: teach bob-ledger-tools' live index the mention rules from the
      copied vectors, publish typed linkedFrom rows under relationIndex v2, and expose a
      pure scanMentions helper."
  - id: plugin-rows
    title: Mention rows, pills, quotes, and jump-to-mention
    depends_on:
      - plugin-index
    size: medium
    description:
      "plugin-rows: render mention entries in Live Preview and Reading view with
      grouping, pills, quote lines, no ×, mixed-pair markers, density limits, stale
      styling, and click-to-mention."
  - id: plugin-card
    title: Mention context card and embed-echo suppression
    depends_on:
      - plugin-rows
    size: medium
    description:
      "plugin-card: add the accessible why-is-this-here card on each mention pill and
      hide embedded-in entries inside their own host's transclusion."
  - id: plugin-nav
    title: Nav courtesy writes, stage rows, refusals, and counts
    depends_on:
      - plugin-index
    size: medium
    description:
      "plugin-nav: write mention mirrors when the cursor leaves an edited line, show
      mentions in the Linked-from stage and Task Card count, refuse removal and reserved
      labels, and downgrade unlinked pairs."
  - id: enable
    title: Vault-copy pilot and switch-on
    depends_on:
      - cli-reconcile
      - cli-capture
      - plugin-card
      - plugin-nav
    size: small
    description:
      "enable: pilot the full feature on a copy of the vault, then flip
      linked_from_mentions on per the rollout decision and record follow-up proposals."
proposed_by: bbugyi200.apollo.research.2v.linker.w0
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-10-29 17:20:19
status: wip
---

- **PROMPT:**
  [prompts/202610/inline_task_mentions.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202610/inline_task_mentions.md)

# Plan: Inline task mentions on 🪢 LINKED FROM

## Context

Bryan already writes task-to-task links by hand: "Merged into [[…]]", "OBSOLETE: See
[[…]]", embedded reading tasks under project tasks, and task titles that name other
tasks. None of them is visible from the task they point at. Obsidian's Backlinks pane
works per note and is flooded by Pomodoro ledger links, so nothing answers "which
**tasks** talk about this task?".

A five-way research swarm studied this and recommended a design. **Bryan accepted every
requirement it recommended (R1–R8).** Phase agents should read the report for rationale,
the vault survey, figures, and rejected alternatives:

```bash
sase artifact read file:explicit:63e0d1934d5f9088649a88f3 "Rationale for the inline task mentions epic"
```

(label
`research:202610/inline_task_mention_backlinks/inline_task_mention_backlinks.md`).

**This plan is the design of record.** Where it and the report differ, the plan wins. It
settles the report's open questions and makes these deliberate changes:

1. **Rollout without a human-timed wait.** The report ships plugins first, the CLI
   writer dark, then a later chezmoi flip. Here every phase builds behind
   `relations.linked_from_mentions` (default `false`), and the phase graph puts the
   plugin phases before the final `enable` phase. That phase pilots the feature on a
   vault copy and then flips the default, per the `rollout` decision. Only the Mac's
   15-minute cron reconciles the vault, so there is no mixed-version ping-pong to wait
   out.
2. **One JavaScript scanner.** bob-ledger-tools owns the JS mention scanner and exposes
   it as a pure `api.relationIndex.scanMentions` helper. Nav calls it instead of keeping
   its own copy, which would be a third implementation of the rules after Rust and
   ledger-tools.
3. **The courtesy write triggers on leaving the edited line.** It reuses nav's existing
   hand-edit burst scheduler and diffs the mention pairs of the **whole note**, not one
   owner's subtree. That handles multi-task edits, `dd` of a whole task, and moves, and
   it writes only when the set of pairs changes.
4. **Excerpts are JavaScript-only.** Rust never stores or reports quotes, so the
   report's MN22 byte/UTF-16 offset agreement reduces to agreement on the occurrence
   set. JS excerpt expectations are written with `⟦…⟧` markers, never offsets.
5. **Dependencies suppress mentions only.** An authored relation on a dependency-joined
   pair keeps its LINKED FROM entry exactly as today (LR behaviour is unchanged).
6. **Simpler reserved-label rule.** The only label that may start with `💬` is exactly
   `💬 mentions`, and only on LINKED FROM.
7. **`bob relations reverse` must preserve mention entries.** It rebuilds affected
   LINKED FROM lines from the relation graph alone
   (`src/native/relation_labels/reverse.rs` around the "Desired from the final graph"
   block), which would silently drop mentions.

Open questions, settled: the glyph is `💬`; `^ref` reading tasks get `💬 embedded in`
rows like any task. Archived id-less sources are an accepted loss for now. "Make it a
relation…", source-side mention pills, and a previewed `done/` id backfill are
follow-ups, not part of this epic (see [Follow-ups](#follow-ups)).

Facts that keep this cheap (verified in code and in the report's vault survey):

- **Storage grammar does not change.** `<link> (💬 mentions)` already parses as a
  labelled LINKED FROM entry in today's Rust and JS. Every engine already guards LINKED
  FROM rows, so task-status-cycler and block-id-prompt need no change.
- The Rust label catalog counts only LINKS entries
  (`src/native/relation_labels/scan.rs`), so mentions cannot leak into suggestions.
- Day one is small: about 64 entries on 61 targets, 9 silent mints, and a maximum fan-in
  of 3.
- The mirror planners (`plan_linked_from_line`, `linked_from_desired_after_delta`, and
  their JS twins) work on `(link, label)` pairs and need no structural change.

## The design (every phase implements this)

### Vocabulary

- **Task mention**: an inert, derived back-reference. Task S's own text contains a block
  link to task T. It is stored on T's LINKED FROM line as `[[S]] (💬 mentions)`, which
  reads "S mentions this". A mention is **not a relation**. It has no label and no
  gestures, it is never a closing label, and it never enters the label catalog.
- **Occurrence**: one counted link inside S's text. A pair `(S, T)` has one or more
  occurrences.
- **Display kind** (plugins only): `embed` when every occurrence of the pair is an
  embed; otherwise `mention`.
- **Site** (plugins only): where an occurrence sits. One of `title`, `note` (a child
  bullet), `cancel_log`, `schedule_log`, `work_log`, or `rotten_log`.

### What counts as a mention (M1–M8)

A block link in S's text becomes an occurrence only if every rule holds. Rust and JS
both implement these rules and are pinned by the same vectors.

- **M1 Owner.** Walk up from the link's line to the nearest list-item ancestor that is a
  `#task` line under the vault's Tasks settings; the task's own line counts. A nested
  `#task` owns its own subtree. A link with no `#task` ancestor is not a mention. That
  excludes note prose, headings, and Pomodoro ledger rows, because Pomodoros are not
  `#task` lines.
- **M2 Edge lines.** A link on, or anywhere under, a Depends-On, BLOCKS, Sub-projects,
  LINKS, or LINKED FROM child is not a mention. This applies to every parse verdict
  except not-a-line, so malformed rows are excluded too. This rule also prevents
  feedback loops. Cancel, Schedule, Work, and Rotten Log headers and their entries
  **do** count.
- **M3 Spans.** Skip fenced code, inline code, `%%…%%` comments (inline and multi-line),
  inline Dataview field values (`[key:: …]` / `(key:: …)`), and blockquoted lines. Build
  on the shared `block_link_spans` scanner (`src/native/task_dependencies/mod.rs`) and
  its JS twin.
- **M4 Target.** The link must resolve, with the relation resolver (same note, basename,
  path, or the explicit `done/` form), to a `#task` line carrying that `^id`, unique in
  its note. Unresolved links, duplicate ids, non-task blocks (`^h-…`), heading links,
  note links, and self-links are ignored **silently**. There is no new warning kind.
- **M5 Forms.** Plain, aliased, embedded `![[…]]`, and struck `~~[[…]]~~` links all
  count. An occurrence is display `embed` when it is an embed, or when it is an
  exact-struck sole link (a bullet whose whole content is `~~[[…]]~~`; this is the
  cycler's retired embed). Otherwise it is display `mention`.
- **M6 Proxy rows.** A task whose description is exactly one embed is never a source.
  The description is what remains after the checkbox, tags, inline fields, and `^id`;
  for example `- [x] #task ![[n#^t]]`.
- **M7 Precedence.**
  - If S's Depends-On names T, or T's Depends-On names S, whatever the status, the pair
    gets **no mention entry**. T's Depends-On or BLOCKS line already names S.
  - If S's LINKS line has an accepted entry for T, the pair's entry is the **relation**
    entry with its label, as today.
  - Otherwise the pair gets the **mention** entry.
  - Dependencies never suppress relation entries.
- **M8 Scope.** Sources come from the same note set reconcile already scans for LINKS
  sources: active notes (writable, so they can be minted), the previous daily note
  (read-only), and every `done/**/*.md` (read-only). Targets must be writable for a
  stored entry, exactly as for relations.

### Projection

```text
pairs(T)  = { S : S's own text holds ≥ 1 occurrence of T under M1–M6, S ≠ T }
mirror(T) = sort_canonical( relation entries for T, as today
                          ∪ { ([[S]], "💬 mentions") : S ∈ pairs(T)
                                ∧ no dependency joins (S, T)
                                ∧ S has no relation entry for T } )
```

The sort order is the existing canonical BK21 order. Storage is never grouped; only the
chips group.

### Storage and the reserved label

```markdown
- [-] #task Add support for hidden, superseded memory strands! [cancelled:: 2026-08-30]
  ^old-supersede
  - OBSOLETE: See [[#^supersede]]!
- [x] #task Launch tale to make superseding better! [completion:: 2026-09-02] ^supersede
  - 🪢 **LINKED FROM:** [[#^old-supersede]] (💬 mentions)
```

- **Grammar unchanged.** A mention is `<link> (💬 mentions)` in the existing
  `<link>[' (' <label> ')']` mirror slot.
- **Reserved `💬`.** `💬` (U+1F4AC) as a label's first code point is reserved:
  - On LINKED FROM, exactly `💬 mentions` (after the usual collapse) marks a mention
    entry. Any other `💬…` label makes the line malformed.
  - On LINKS, any `💬…` label makes the line malformed. It is warned about and left
    untouched, per §3.6.
  - Every authored writer refuses a `💬…` label with `💬 is reserved for text mentions`.
    That covers capture `|` / `||`, the nav Links and Linked-from stage label fields,
    `bob relations rename` / `hide` / `unhide` / `reverse`, and
    `relations.cancel_labels` / `complete_labels` (an invalid entry is warned about and
    dropped).
- **Typed in memory.** Parsers return a typed kind (relation or mention). A mention's
  spelling never surfaces as a label: not in the catalog, suggestions, inverse phrases,
  rename spans, or the cancel/complete policy.
- **No stored excerpts.** Quotes are live only.

### Identity, minting, lifecycle, and uncertainty

- **Identity is (S, T)**, by vault path plus block id. Several occurrences collapse to
  one entry. A → B and B → A are independent.
- **Minting.** An active, writable, id-less source gets a silent
  `mint_block_id(description, used)` mint. It happens in the reconcile pass, capture
  batch, or nav courtesy write that first needs it, and only when the entry would
  actually be stored: T is writable, no dependency joins the pair, and there is no
  relation. The existing quiet-deferral rule (LR5) applies. Sources in `done/` or the
  previous daily note are never minted; if they have no id, they are not listed. Dry
  runs and rendering mint nothing. When either switch is off, nothing is minted for
  mentions.
- **Lifecycle.** Closing, cancelling, reopening, or archiving either end changes
  nothing. The cycler's embed ↔ strike retirement is the same pair, so zero bytes
  change. Deleting the link removes the entry on the next write path. Moves and renames
  rely on generic link rebasing.
- **Never prune on uncertainty.** A source is uncertain when any of these holds:
  - its note has a duplicate `^id` for S;
  - S's LINKS or Depends-On line is malformed;
  - T's Depends-On line is malformed (for that pair);
  - S's note has an unclosed code fence.

  Existing entries naming an uncertain source are kept byte-for-byte with their label,
  and the source is never newly added.

- **Inertness.** Every guarantee in §2 and LR18 carries over word for word. A mention
  never changes a status, lane, `[fresh::]`, `[keeps::]`, `[unblocked::]`, schedule,
  priority, `[id::]`, `[dependsOn::]`, the `]s` walk, the plan budget, successor offers,
  or any Task Link logic. The only edit on S is a minted `^id`.

### Kill switch and outputs

```yaml
relations:
  linked_from_lines: true # unchanged: false removes every LINKED FROM line
  linked_from_mentions: false # mentions; the enable phase decides the shipped default
```

- `false` removes only `(💬 mentions)` entries on writable tasks, deleting a line left
  empty. It mints nothing for mentions. Plugins show no mention chips, and nav and
  capture write none. The switch is lossless: flipping it back rebuilds byte-identical
  lines (MN13).
- An invalid value falls back to the default and warns, in the `linked_from_lines`
  wording pattern. Rust, nav, and ledger-tools read the key from
  `~/.config/bob/config.yml` (`BOB_CONFIG_FILE` / `XDG_CONFIG_HOME`).
- **`bob task reconcile` JSON** (additive; `schema_version` unchanged):
  - `linked_from_line_updates[]` rows gain `mentions_added` and `mentions_removed`, the
    mention subsets of `added` and `removed`.
  - `minted_relation_source_block_ids[]` rows gain `reason: "relation" | "mention"`. A
    source minted for both reports `relation`.
  - A new top-level `mention_edges` counts listed mention entries.
  - No new warning kinds. Human output shows mention counts in the existing `🪢` group,
    in its style, for example `+2 💬`.
- **Capture JSON** (additive; every `schema_version` stays 1):
  - Per item, mention mirrors appear in the existing `linked_from_line_updates[]`, with
    the same `mentions_added` / `mentions_removed` fields.
  - Mints appear in `minted_relation_block_ids[]` with `reason: "mention"`.
  - Skips appear in `mirror_skipped[]` with the existing reasons.
  - A draft with no task block links produces byte-identical output to today.

### Visual design

The row must answer three questions without a click: _what is deliberate_, _what was
said_, and _why this is here_. It must do that in Bob's existing chip language, with
**no new colour**: contract §10 says colour carries status, never direction. Text,
italics, and the `💬` glyph carry provenance.

Live Preview and Reading view, plugin present:

```text
☑ Launch tale to make superseding better!
  🪢 linked from  followed up by (○ Fix flaky capture retries  ↗ bob)
                  ┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄
                  💬 mentioned in (⊘ Add support for hidden, superseded mem…)
                  ▏ “OBSOLETE: See ⟦this task⟧!”
                  💬 embedded in  (☑ Read sase_sites_platform!  ↗ sase_sites)
```

- **Grouping.** Relations come first, then a hairline dotted divider
  (`1px dotted var(--background-modifier-border)`), then mentions. Each mention gets its
  own visual row, like labelled relation rows today. A row that holds only mentions has
  no divider.
- **Pill.** `💬 mentioned in` or `💬 embedded in`, in the existing muted small-caps
  `.bob-relation-pill`, plus `.is-mention` for italics (the convention for reported
  speech). The pill is focusable (`tabindex=0`, `aria-haspopup="dialog"`, aria label
  `Why is this here? ‹S› mentions this task in its text`).
- **Chip.** The unchanged `bob-dep-chip`: status mini-checkbox, about 40 characters of
  title, `↗ note` when cross-note, `#hide` muted, and closed tasks dimmed but never
  struck.
- **Quote line.** Shown only for display `mention` at a non-`title` site, when the
  occurrence's line still has a letter or digit after removing the list marker,
  checkbox, and every block-link token. It is one muted line with a 2 px left rule. The
  link token becomes its alias, or else **this task**, highlighted with
  `--text-highlight-bg`. Fit about 80 characters with a middle ellipsis that keeps the
  token visible. With several prose occurrences, quote the first in document order.
  Sole-link and title mentions get no quote, because the chip already says it all.
- **Click.** Clicking a mention chip opens S **at the first occurrence** (line and
  column), scrolled into view, with a brief highlight of about 1.2 s and no animation
  under reduced motion. Mod-click opens a new pane. Hovering keeps the existing Obsidian
  page preview of S.
- **Controls.** A mention chip has **no ×**. Removal gestures are refused with the
  notice below.
- **Mixed pair.** When S both relates to and mentions T, the relation chip keeps its
  label and ×, plus a small muted trailing `💬` titled `Also mentioned in its text`. The
  × removes only the relation; nav then downgrades the entry to a mention and shows the
  unlink notice.
- **Density.** The existing LC9 collapse (three or more closed → `✓×N`) applies per
  group. More than five open mentions show the first five plus a `+N more` toggle.
- **Stale.** A stored mention that is no longer live gets the existing dashed style,
  with the tooltip `No longer mentioned here · refreshes on the next reconcile`. A live
  mention missing from the text is appended in canonical order, as relations are today.
- **Embed echo.** When a LINKED FROM row renders inside a transclusion hosted in S's
  note, hide that row's `embedded in` entries whose source note is the host note. Hide
  the whole row if nothing else is left. Without this, an embedded sub-task would show
  "💬 embedded in ‹the task you are reading›" inside its own embed.
- **Without the plugin** the line still reads as English:
  `🪢 LINKED FROM: ^old-supersede (💬 mentions)`.

The context card opens on pill hover (after the usual hover delay), keyboard focus, or
click. It is non-modal, stays open while hovered (WCAG 1.4.13), and Escape dismisses it
and returns focus to the pill:

```text
┌────────────────────────────────────────────────────────────┐
│ 💬 Mentioned in another task's text · not a relation        │
│ ⊘ Add support for hidden, superseded memory strands!       │
│   sase_memory_webs · mentioned 2× in this task              │
│ ▏ “…OBSOLETE: See ⟦this task⟧! The new tale covers it…”     │
│   in a note under it                                        │
│ ▏ “Merged into ⟦this task⟧ after review”                    │
│   in its Cancel Log                                         │
│ ↵ Open at the mention                         Esc to close  │
└────────────────────────────────────────────────────────────┘
```

The card shows up to three occurrences (excerpt about ±60 characters, token highlighted,
location underneath), then `+N more`. Each listed occurrence is itself clickable and
opens S at that occurrence.

Task Card and Linked-from stage:

```text
Linked from…   3 · 1 relation · 2 💬

CURRENT
  followed up by   ○ Fix flaky capture retries                 bob
  💬 mentioned in  ⊘ Add support for hidden, superseded mem…   sase_memory_webs
                   “OBSOLETE: See ⟦this task⟧!”
```

**Copy strings** (exact; every phase uses these):

| Use                   | Text                                                                                                                                                     |
| --------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Stored label          | `💬 mentions`                                                                                                                                            |
| Pills                 | `💬 mentioned in` · `💬 embedded in`                                                                                                                     |
| Row tooltip           | `Tasks linking here · relations and 💬 text mentions · live`                                                                                             |
| Stale tooltip         | `No longer mentioned here · refreshes on the next reconcile`                                                                                             |
| Card header           | `💬 Mentioned in another task's text · not a relation`                                                                                                   |
| Card locations        | `in its title` · `in a note under it` · `in its Cancel Log` · `in its Schedule Log` · `in its Work Log` · `in its Rotten Log` · `embedded as a sub-task` |
| Card count            | `mentioned N× in this task` (only when N ≥ 2)                                                                                                            |
| Card action           | `↵ Open at the mention`                                                                                                                                  |
| Removal refusal       | `💬 This comes from a link in ‹S›'s text · edit it there (↵ opens it)`                                                                                   |
| Mixed marker title    | `Also mentioned in its text`                                                                                                                             |
| Unlink notice (mixed) | `Unlinked ‹S› · it still mentions this task in its text`                                                                                                 |
| Reserved label        | `💬 is reserved for text mentions`                                                                                                                       |
| Generated-line notice | `🪢 LINKED FROM is generated from other tasks' LINKS lines and text · change it with Ctrl+Shift+Alt+L`                                                   |
| Task Card count       | relations only: unchanged `N · ‹first name›`; mentions only: `M 💬 · ‹first name›`; both: `N · R relation(s) · M 💬`                                     |

### Update paths

| Path                                          | Trigger                                                               | Latency   | Writes                                                                                                                              |
| --------------------------------------------- | --------------------------------------------------------------------- | --------- | ----------------------------------------------------------------------------------------------------------------------------------- |
| ledger-tools live index                       | any edit, including unsaved editor text                               | instant   | none; chips update on an **existing** row                                                                                           |
| nav courtesy write                            | the cursor leaves an edited line and the note's mention pairs changed | seconds   | T's LINKED FROM (same note: one transaction with any S mint; cross-note: preimage-checked follow-up); silent on success and failure |
| capture                                       | a capture whose written notes change mention pairs                    | at commit | T's mirror and S's mint join the staged batch                                                                                       |
| `bob task reconcile` (Mac cron, every 15 min) | vim, agent, and sync edits; anything a courtesy write missed          | ≤ 15 min  | the authority: projects, mints, repairs, removes                                                                                    |

Reconcile stays the authority. Writes happen only when the _set_ of pairs changes, never
per keystroke.

## Phases

### Mention contract, shared vectors, decision record, and glossary

Repo: bob-cli (docs, test fixture, memory). This phase changes no Rust or JS code. Read
the `docs.md` reference memory before editing `docs/`.

1. **`docs/task-relations.md`.** Add **§19 Text mentions**: vocabulary, M1–M8, the
   projection, storage and the reserved label, identity, minting, lifecycle,
   uncertainty, inertness, the kill switch, the visual design, and the copy table, all
   from [The design](#the-design-every-phase-implements-this). Then amend:
   - **§1** (vocabulary: task mention), **§3.2** / **§3.6** (reserved `💬`), **§6**
     (projection), and **§7** (owners and steps now include mention pairs).
   - **§8** (new writers: the nav mention courtesy write and the capture mention mirror;
     the generated-line notice text) and **§9** (gestures refused on mention entries).
   - **§10** (chips: grouping, pill, quote, card, no ×, mixed marker, stale, density,
     and embed echo), **§11** (`relationIndex` v2), and **§12** (the switch and JSON).
   - **§13** (LR11 and inertness extend to mentions) and **§14**. Replace the "Adopting
     old idioms … not adopted" bullet with a pointer to §19: hand-written links in task
     text now surface as mentions, never as relations.
2. **MN vectors** in §19, as a table with these ids and outcomes (extend as needed;
   never renumber):

   | #    | Situation                                                                                                 | Outcome                                                                                                       |
   | ---- | --------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------- |
   | MN1  | S's title contains `[[#^t]]`; T is a `#task`                                                              | T gains `[[#^s]] (💬 mentions)`; S gains nothing except a possible `^id`                                      |
   | MN2  | S's child `Merged into [[n#^t]]`                                                                          | same, cross-note form; JS quote `Merged into ⟦this task⟧`                                                     |
   | MN3  | link under a nested `#task` beneath S                                                                     | the nested task is the source                                                                                 |
   | MN4  | link on an edge line of S, or on T's own LINKED FROM                                                      | no mention, no feedback loop                                                                                  |
   | MN5  | link in a Pomodoro child, heading, note prose, fenced or inline code, `%%…%%`, `[k:: …]`, or a blockquote | no mention                                                                                                    |
   | MN6  | target is `^h-…`, a heading, unresolved, a duplicate id, or S itself                                      | no mention, no warning                                                                                        |
   | MN7  | `- ![[n#^t]]` under S; T closes (cycler writes `~~[[n#^t]]~~`); T reopens                                 | one entry throughout, display `embed`; zero mirror bytes change                                               |
   | MN8  | S has `🪢 LINKS: duplicate of [[T]]` and prose `duplicate of [[T]]`                                       | one `(duplicate of)` entry; JS relation row marked also-mentioned                                             |
   | MN9  | S depends on T, or T depends on S, and S mentions T                                                       | no mention entry                                                                                              |
   | MN10 | S mentions T twice                                                                                        | one entry; JS count 2                                                                                         |
   | MN11 | source active and id-less                                                                                 | minted and listed (`reason: "mention"`)                                                                       |
   | MN12 | source in `done/` or the previous daily note, id-less                                                     | not listed; no write there                                                                                    |
   | MN13 | `linked_from_mentions: false`, then `true`                                                                | only mention entries removed, then rebuilt byte-identically                                                   |
   | MN14 | delete every LINKED FROM line, reconcile twice                                                            | identical bytes; the second run changes nothing                                                               |
   | MN15 | a `💬…` label through capture, a stage, rename/hide/reverse, or the close-label config                    | refused with the reserved-label message                                                                       |
   | MN16 | a pre-feature parser reads `(💬 mentions)`                                                                | accepted as a labelled entry                                                                                  |
   | MN17 | any mention add or remove                                                                                 | every status, lane, field, Pomodoro section, Depends-On, and BLOCKS line byte-identical except a minted `^id` |
   | MN18 | a mention typed, then the cursor leaves the line                                                          | T's row exists within seconds; the live chip showed before that if a row already existed                      |
   | MN19 | click, and stage remove, on a mention chip                                                                | opens S at the occurrence / refused with the notice                                                           |
   | MN20 | T embedded under S, viewed in S's note                                                                    | the embedded T shows no `embedded in` S echo                                                                  |
   | MN21 | proxy row `- [x] #task ![[n#^t]]`                                                                         | no mention                                                                                                    |
   | MN22 | Rust and JS on every fixture vector, with Unicode before the link                                         | identical occurrence sets                                                                                     |
   | MN23 | S has an uncertain condition (duplicate id, malformed LINKS/Depends-On, unclosed fence)                   | existing entries naming S kept; S never newly added                                                           |
   | MN24 | `bob relations reverse` touches T's mirror while S mentions T                                             | T's mention entry survives                                                                                    |
   | MN25 | aliased link `[[#^t\|the old plan]]` in a child bullet                                                    | one mention; JS quote shows `⟦the old plan⟧`                                                                  |

3. **Shared fixture** `tests/fixtures/task_mentions/vectors.json`: the single source of
   truth, following the `tests/fixtures/relation_labels/vectors.json` convention (§16
   "Shared vectors"). bob-plugins copies it byte-for-byte. Each vector carries:
   - its MN id and input notes (`{"path": "text"}`);
   - the expected **occurrence set** `[{source, target, count}]`, with `source` and
     `target` as `path#^id`, or `path:line` for id-less sources (both engines check
     this);
   - JS-only fields `display`, `site`, and `excerpt` with `⟦…⟧` markers;
   - optional Rust-only `mirror` (expected LINKED FROM lines per target) and `mints`.

   Cover at least MN1–MN12, MN21–MN23, and MN25 as fixture vectors. Give the fixture a
   `version` field.

4. **Other docs.**
   - `docs/task-status-hooks.md`: the reconcile JSON fields, and the widened read-only
     `done/` scan beside the existing exact-`done/`-target exception.
   - `docs/reference/configuration.md`: the `relations.linked_from_mentions` key,
     default `false`.
   - A new guide, `docs/guides/see-task-mentions.md` ("See which tasks mention a task"),
     next to `link-related-tasks.md`. It covers what counts, how to read the pill and
     quote, why there is no ×, how to turn a mention into a relation (add a LINKS
     entry), and the switch. Link it from the guides index and the site nav.
   - Run `just docs-gen` if needed, then `just docs-check`.
5. **Memory** (follow `/sase_memory_write`, then run `sase memory init`):

   > [!decision] memory_record Add the decision strand
   > `decisions:task-mentions-are-inert-linked-from-entries` ("Task Mentions Are Inert
   > LINKED FROM Entries"), in the existing record shape (Applies to / Claim / Why /
   > Cost / Reopens when / Evidence, with `metadata.status: accepted` and
   > `decided: 2026-10-29`). Claim: a block link to another `#task` in a task's own text
   > derives an inert `(💬 mentions)` entry on the target's LINKED FROM line; nothing is
   > written on the source but a silent `^id`; a dependency or relation on the pair
   > wins. Why: rejected alternatives are auto-writing LINKS on S, a separate MENTIONED
   > IN line, a new suffix grammar, render-only, title-only, and excluding struck links.
   > Cost: a reserved label, plus a reconcile pass over task text. Reopens when one
   > target collects more than 10 open mentions, log entries drive fan-in, unexpected
   > mints appear, or a usability check shows the merged row confuses Bryan. Evidence:
   > the research ref above, this plan, and §19.

   > [!decision] memory_glossary Change exactly three glossary strands and no others:
   >
   > - Add the strand `glossary:task-mention` ("Task Mention"; aliases `mention`,
   >   `text mention`). Define it in the glossary's voice, link `[[task-relation]]`,
   >   `[[task-link]]`, and the new decision record, and point at
   >   `docs/task-relations.md` §19.
   > - Update `glossary:task-relation`: the generated LINKED FROM line also lists
   >   `[[task-mention]]`s (`(💬 mentions)` entries), which are not relations. Keep
   >   every other sentence.
   > - Update `glossary:task-link`. Its last sentence currently says every LINKS /
   >   LINKED FROM entry is a task relation. Make it say that LINKS entries are task
   >   relations, and LINKED FROM entries are relations or task mentions; none is ever a
   >   Pomodoro Task Link.

**Done when** §19 and the amendments read as one contract, the fixture is valid JSON
covering the listed vectors, `just docs-check` passes, and `sase memory init` is clean.

### Rust mention scanner, reserved label, and writer guards

Repo: bob-cli.

- **Scanner.** Add `src/native/task_relations/mentions.rs`. Its pure entry point takes
  one note's lines and parsed tasks plus a target resolver and returns occurrences
  `{source task, target (path, id), line, embedded, struck, sole}` under M1–M6.
  - Reuse `block_link_spans`, `fenced_lines`, and the relation and dependency line
    recognisers (`is_managed_link_line`, the Sub-projects recogniser) for M2.
  - Add small span filters for `%%…%%`, inline fields, and blockquotes.
  - Find each task's subtree with the existing indentation helpers. Do not add a second
    tree walker if `structure.rs` or the dependency code already has one.
- **Fixture tests.** Read `tests/fixtures/task_mentions/vectors.json` and assert the
  occurrence set and counts for every vector (MN22 on the Rust side). Ignore JS-only
  fields.
- **Reserved label.**
  - Add `LabelError::Reserved` to `label.rs` (`as_str` → `reserved`), surfaced as
    `💬 is reserved for text mentions`.
  - `normalize_relation_label` refuses a `💬`-first label.
  - `parse_relation_label` / the LINKS parser mark `💬…` labels malformed. The LINKED
    FROM parser accepts exactly `💬 mentions` as a mention entry and marks any other
    `💬…` label malformed.
  - Give `RelationEntry` (or the LINKED FROM verdict) a typed kind, so callers never see
    the reserved spelling as a label.
  - Add LK/LW unit tests for each rule.
- **Writer guards.**
  - Capture `|` / `||` labels (capture-parse need/teaching plus capture-complete typed
    row `disabled_reason`) and `bob relations rename` / `hide` / `unhide` / `reverse`
    refuse with the reserved message.
  - `relations.cancel_labels` / `complete_labels` warn and drop a `💬…` entry
    (`src/native/config/relations.rs`).
  - Inverse phrases never apply to the reserved spelling.
- **Preserve mentions in every Rust LINKED FROM writer.**
  - `bob relations reverse` keeps every existing mention entry on an affected mirror,
    unless the reversal now gives that same source a relation entry for the target (the
    relation wins).
  - `bob relations rename` never matches mention entries.
  - Confirm the capture mirror's delta planner already preserves them.
  - Add MN24 and a rename test.

**Done when** `just check` passes, every fixture vector agrees, MN15 and MN24 are
covered, and no behaviour changes for vaults without mentions. Reconcile does not use
the scanner yet.

### Reconcile projects mentions onto LINKED FROM

Repo: bob-cli.

- **Config.** Add `linked_from_mentions: bool` (default `false`) to `RelationsConfig`,
  with the invalid-value warning and tests mirroring `linked_from_lines`. Add a
  `load_relations_linked_from_mentions_enabled()` beside the existing loader.
- **Prefilters.** In `task_status_hooks/relations.rs`, widen `collect_file_edges` and
  `load_relation_archives` from `contains("LINKS")` to also admit notes containing both
  `#task` and `#^` when mentions are enabled. Keep the explicit-`done/`-target loading.
- **Projection.** In `reconcile_relation_lines`, collect mention pairs per source with
  the cli-core scanner, then apply M7.
  - Dependency joins come from accepted Depends-On lines at both ends (active and
    archive), so status never matters.
  - Insert relation entries first, then mention entries with
    `(link, Some("💬 mentions"))` only for identities still absent.
  - Track uncertain mention sources (M-uncertainty rules) beside the existing
    `uncertain` set: existing entries naming them are kept with their stored label.
  - Mint only sources whose entry will be stored, under LR5.
  - Owners are every T with a non-empty mirror plus every writable task that already has
    a LINKED FROM child, as today.
- **Switch semantics.** With mentions off, `desired` drops only mention entries and
  mints nothing for mentions (MN13). `linked_from_lines: false` still removes
  everything.
- **Outputs.** Add the JSON fields from
  [Kill switch and outputs](#kill-switch-and-outputs) and human-output counts. Keep
  `--dry-run` parity (LR20).
- **Tests.** Cover MN1–MN14, MN17, MN21, and MN23 through the reconcile test harness,
  using the fixture's `mirror` / `mints` expectations where present. Include LR11 with
  mentions (MN14), "dependency suppresses mention but not relation", and "mention-only
  owner with mentions off loses only its mention entries".
- **Performance.** Time `bob task reconcile --dry-run -f json` on a copy of the vault
  before and after. Report both numbers in the phase notes. Optimise if the pass adds
  more than about 20% to the run.

**Done when** `just check` passes and, with a temp config enabling mentions, a dry run
over a vault copy reports plausible `mention_edges` (the report expected about 64) with
no non-LINKED-FROM, non-mint changes. Never run a live reconcile against `~/bob`.

### Capture stages mention mirrors in its batch

Repo: bob-cli.

- After capture plans every edit, and before commit, run the scanner on the pre- and
  post-capture snapshot of each note the capture writes. Diff the `(S, T)` pairs that
  have S in those notes.
- For each added pair that passes M7 (S's staged LINKS and Depends-On lines count, so a
  `~` or `&` in the same capture wins), stage T's LINKED FROM add through the existing
  `relations_mirror.rs` helpers (`stage_linked_from_add`, the delta planner) and mint S
  in the same batch if needed. For each removed pair, stage the removal unless S now has
  a relation entry.
- Respect both switches, archived and previous-daily targets, and malformed mirrors
  (`mirror_skipped`). A skip is never fatal; reconcile heals.
- Report through the JSON fields above. `--dry-run` previews the same rows and writes
  nothing.
- Bob Mac Capture needs no change: it is a thin client, and the output is additive.
- **Tests:**
  - a new task whose text holds `[[n#^t]]` (S minted, T's mirror staged);
  - a sub-bullet captured under an existing task;
  - `~` in the same capture (relation wins);
  - `&` (dependency, no mention);
  - an archived target (skipped);
  - mentions off (no-op);
  - a capture with no block links (byte-identical JSON);
  - rollback leaving every note untouched.

**Done when** `just check` passes and each case above is covered.

### Live mention index and relationIndex v2

Repo: bob-plugins (bob-ledger-tools). Read `AGENTS.md` there. Edit `src/` fragments, run
`npm run build`, keep each fragment ≤ 1000 lines, and finish with `npm test`,
`npm run docs:check`, and `bob plugins sync`.

- **Fixture.** Copy `tests/fixtures/task_mentions/vectors.json` byte-for-byte to
  `plugins/bob-ledger-tools/scripts/fixtures/task_mentions/vectors.json`, with a test
  that it parses and carries the expected `version`.
- **Scanner.** Extend `145-relation-index-model.js`, or add a sibling fragment, so
  `parseTaskRelationFile` also returns per-file mention occurrences.
  - Each occurrence is
    `{source, targetRef, line, ch, embedded, struck, sole, site, excerpt}`. `ch` is a
    UTF-16 column for the editor; `excerpt` is text with the token range.
  - Also return each task's accepted Depends-On targets and LINKS targets, for M7.
  - The LINKED FROM parser returns `kind: "mention"` for `(💬 mentions)`, and the LINKS
    parser rejects `💬…` labels.
- **Publish.** In `171-relation-index-lifecycle.js`, build typed reverse rows under the
  same generation as today. Resolve targets exactly as relations do, apply M7, and keep
  the active-editor overlay (instant updates).
- **API v2.** Bump `api.relationIndex.version` to 2.
  - `incoming()` and `outgoing()` keep returning **relations only**, so no v1 consumer
    can offer a removal it cannot perform.
  - New `linkedFrom(path, blockId)` returns typed rows:
    - `{kind: "relation", source, label, alsoMentioned, …}`;
    - `{kind: "mention", source, display: "mention"|"embed", count, occurrences: [{line, ch, site, excerpt}]}`
      (dependency-joined pairs omitted).
  - New pure `scanMentions(content, path)` returns that note's resolved `(S, T)` pairs
    with their occurrences, for nav's courtesy write.
  - `labelSuggestions` never includes the reserved spelling.
- **Switch.** Read `relations.linked_from_mentions` (default `false`, cached by stat key
  like nav's loader). When off, publish no mention rows and `scanMentions` returns `[]`.
- **Tests.** Fixture-driven occurrence, display, site, and excerpt checks (MN22 on the
  JS side); the v1/v2 API contract; overlay updates; and MN16 (a v1 consumer sees a
  mention entry as an ordinary labelled row).
- **Performance.** Re-run `scripts/measure-ledger-tools-relation-index.cjs` and record
  the cold-scan and warm-update numbers in the fragment comment, as the existing comment
  does. Keep the scan chunked.

**Done when** `npm test` passes, the API is documented in bob-plugins `docs/`, and the
plugins are synced.

### Mention rows, pills, quotes, and jump-to-mention

Repo: bob-plugins (bob-ledger-tools: `146-relation-chip-model.js`,
`153-relation-widget.js`, `278-plugin-relation-model.js`,
`282-plugin-relation-render.js`, `styles.css`). Same build, test, docs, and sync rules.

- **Model.** `relationChipModelForRow("from")` consumes `linkedFrom()` v2 rows when
  available, falling back to v1.
  - Stored `(💬 mentions)` entries map to mention chips: `kind`, `display`, `count`,
    `occurrences`, and `quote`.
  - The model returns relation chips and mention chips as two ordered groups in stored
    canonical order. Live chips missing from the text are appended per group.
- **Render** the [Visual design](#visual-design) in Live Preview and Reading view with
  identical DOM:
  - the divider and the pill (with `.is-mention`);
  - the quote line, with the middle-ellipsis fitter and the highlighted token;
  - no ×;
  - the mixed-pair trailing `💬`;
  - the per-group LC9 collapse and `+N more`;
  - the stale tooltip and the new row tooltip.

  Add CSS only with existing theme variables. Check light and dark themes.

- **Click.** Mention chips open S at the first occurrence's line and column with the
  brief highlight; Mod-click opens a new pane. Reuse `dependencyOpenTarget` and the
  existing open-at-line helpers wherever possible.
- **Tests** (extend `scripts/test-ledger-tools-relation-chips.cjs` or add a sibling):
  - grouping and ordering;
  - quote rules, including no quote for title and sole-link sites, alias tokens, and
    Unicode fitting;
  - no × on mentions;
  - the mixed marker;
  - stale styling;
  - density;
  - the click target line and column;
  - v1 fallback.

**Done when** `npm test` passes, the rows match the mock in both views (record a
fakenode DOM snapshot or an Obsidian screenshot in the phase notes), and the plugins are
synced.

### Mention context card and embed-echo suppression

Repo: bob-plugins (bob-ledger-tools). Same rules.

- **Card.** A non-modal popover anchored to the pill.
  - It opens on hover after the standard delay, on focus, or on click. It stays open
    while the pointer is over it, Escape closes it and returns focus to the pill, and it
    closes when the row re-renders.
  - Content and copy come from the [Visual design](#visual-design): header, S chip line
    with note and count, up to three occurrences with location labels from `site` /
    `display`, `+N more`, `↵ Open at the mention`, and `Esc to close`.
  - Enter or click on an occurrence opens it.
  - Use `role="dialog"` with `aria-label`; arrow keys move between occurrences.
  - Only one card is open at a time.
- **Embed echo (MN20).** In the Reading-view and embed render paths, detect when a
  LINKED FROM row renders inside a transclusion whose host note is S's note. Hide that
  row's `embedded in` entries from host-note sources, and hide the row when it is left
  empty.
  - First confirm in real Obsidian that list-item embeds render child rows. If they
    never do, record that and keep the rule as a guarded no-op.
- **Tests:**
  - card open and close by mouse, keyboard, and Escape;
  - focus return;
  - occurrence navigation;
  - copy strings;
  - embed-echo hide and keep cases.

**Done when** `npm test` passes, the card is keyboard- and touch-reachable, and the
plugins are synced.

### Nav courtesy writes, stage rows, refusals, and counts

Repo: bob-plugins (bob-navigation-hotkeys; verify-only in task-status-cycler and
block-id-prompt). Same rules.

- **Courtesy write (MN18).** Extend the existing hand-edit burst path
  (`600-plugin-dependency-mirror.js` → `594a-relation-hand-edit.js`). After the
  dependency and LINKS mirrors run, call `api.relationIndex.scanMentions` on the burst
  baseline and on the current note text, and diff the pair sets.
  - For each changed T, apply M7 (S's LINKS and Depends-On, T's Depends-On) and plan T's
    LINKED FROM with `linkedFromDesiredAfterDelta` / `planLinkedFromLine`, adding or
    removing `(S, 💬 mentions)`.
  - Mint S into the source transaction when the entry will be stored.
  - A same-note T joins one transaction annotated `RELATIONS_MACHINE_WRITE_ANNOTATION`.
    A cross-note T goes through `applyRelationFollowUps`.
  - Never show a notice for mention writes; reconcile heals failures.
  - Skip when the API is missing or older than v2, or when either switch is off. Read
    `relations.linked_from_mentions` with a loader beside
    `loadRelationsLinkedFromLinesEnabled`.
  - Undo behaves as the LINKS hand-edit mirror does today.
- **Unlink downgrade.** When `removeLink(S, T)`, chip ×, or the Links-to Ctrl+D removes
  a relation whose source still mentions T (per `scanMentions` on S's note), T's entry
  becomes `(💬 mentions)` instead of disappearing, and the notice is
  `Unlinked ‹S› · it still mentions this task in its text`.
- **Linked-from stage.**
  - CURRENT rows come from `linkedFrom()` v2: mention rows carry the pill, the excerpt
    subtitle, and the note.
  - ↵ opens the first occurrence. Toggling or removing a mention row is refused with the
    removal refusal notice.
  - Footer hints follow the selected row's kind.
  - Adding a relation for a source that already mentions T simply replaces the mention
    entry (identity delta).
- **Task Card.** The `Linked from…` count uses the copy table format
  (`312-task-card-relations.js`).
- **Reserved label.** Refuse `💬…` labels in every nav label field (Links and
  Linked-from stages, label management, rename) with the reserved message. Update the
  generated-line notice text.
- **Audit.** Every nav LINKED FROM writer (`594-plugin-relation-edit.js`,
  `595-relation-follow-ups.js`, project conversion, and any other caller of
  `planLinkedFromLine`) preserves mention entries it does not own.
- **Verify only.** task-status-cycler and block-id-prompt still treat a LINKED FROM row
  carrying `(💬 mentions)` as a managed relation row (Task Link, tree-close, and
  strike/restore guards). Add one regression test each.
- **Tests:**
  - courtesy write same-note and cross-note, mint, M7 suppression, removal, the switch
    off, and an old index API;
  - unlink downgrade;
  - stage rows and refusals;
  - the Task Card count variants;
  - the reserved label;
  - the writer audit.

**Done when** `npm test` passes, nav docs in bob-plugins describe the new rows and
refusals, and the plugins are synced.

### Vault-copy pilot and switch-on

Repo: bob-cli and bob-plugins.

1. **Pilot** on a fresh copy of `~/bob` in a temp directory, with a temp
   `BOB_CONFIG_FILE` that copies Bryan's config and sets
   `relations.linked_from_mentions: true`. Never write `~/bob`.
   - `bob task reconcile -b <copy> --dry-run -f json`: report `mention_edges`, targets,
     mints, and warnings against the report's survey (64 / 61 / 9), and explain any
     drift.
   - A live run on the copy, then `git -C <copy> diff --stat` plus a diff filter proving
     that only LINKED FROM lines and minted `^id`s changed (MN17).
   - MN14 on the copy: delete every LINKED FROM line, reconcile twice, and compare the
     bytes to the first run; the second run makes zero changes.
   - MN13 on the copy: flip the switch off and back on.
   - Run `npm run smoke:obsidian` in bob-plugins if it works on this host. Otherwise
     record that the visual check is Bryan's.
2. **Switch-on.**

   > [!decision] rollout = enable Flip the default to `true` in
   > `src/native/config/relations.rs` (and its tests), nav's and ledger-tools' loaders,
   > `docs/reference/configuration.md`, §12/§19, and the guide. Commit with a `feat:`
   > subject (not breaking). Run `just check`, `npm test`, and `bob plugins sync`.
   > Mentions go live when the next release train reaches the Mac.

   > [!decision] rollout = dark Leave every default `false`. Make sure the guide and §12
   > tell Bryan exactly how to opt in (`relations.linked_from_mentions: true` in his
   > chezmoi-managed config), and record the opt-in as the first follow-up.

3. **Follow-ups.** Record each item in [Follow-ups](#follow-ups) as a
   `PROPOSED FOLLOW-UP:` note on this phase's bead.

**Done when** the pilot numbers and diffs are in the phase notes and the rollout branch
is applied.

## Follow-ups

These are deliberately outside this epic:

- **"Make it a relation…"**: ⇧↵ on a mention row in the Linked-from stage opens S's
  Links stage prefilled with T. A label is prefilled only from non-closing labels, so
  "Duplicate of [[…]]" never prefills a configured cancel label.
- **Source-side mention pills**: render S's inline task link as a status pill.
- **Previewed one-shot `^id` backfill** for the roughly 20 id-less archived sources
  (mostly the 2026-10-19 "Merged into" cancels).
- **Usability check of the merged row**, per the report's reopen trigger. If Bryan can't
  tell why each entry exists, move mentions to a separate line. Typed kinds make that
  migration mechanical.

## Guardrails for every phase

- Releases ride the daily train. Never edit `Cargo.toml`'s version or `CHANGELOG.md`.
  Every change here is additive, so use `feat:` / `fix:` / `docs:` subjects without `!`.
- Never write `~/bob` directly. bob-plugins changes reach the vault only through
  `bob plugins sync`.
- Never prune on uncertainty, and never let a mention change anything but LINKED FROM
  bytes and a minted `^id`.
- If an implementation disagrees with a §19 vector, fix the doc and the fixture first,
  in the same change, and say so in the phase notes.
