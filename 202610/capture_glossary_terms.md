---
tier: tale
title: Define the Capture and Bulk Capture glossary terms
goal:
  Agents share one precise meaning of "capture" and "bulk capture" across bob capture
  and Bob Mac Capture.
size: small
decisions:
  memory_capture_term:
    ask: Add the glossary strand glossary/capture.md defining Capture?
    memory:
      - glossary:capture
    requested:
      When you are done thinking, add excellent (but concise) new glossary terms for
      "capture" and "bulk capture".
    default: true
    answer: true
  memory_bulk_capture_term:
    ask:
      Add the glossary strand glossary/bulk-capture.md defining Bulk Capture (aliases
      multi-item, batch capture)?
    memory:
      - glossary:bulk-capture
    requested:
      When you are done thinking, add excellent (but concise) new glossary terms for
      "capture" and "bulk capture".
    default: true
    answer: true
proposed_by: bbugyi200.apollo.7v
decided_by: auto
create_time: 2026-10-15 12:48:12
status: wip
---

# Define the Capture and Bulk Capture glossary terms

## Outcome and scope

Add two new terms to bob-cli's `glossary` memory web:

- **Capture**: one `bob capture` run over one draft.
- **Bulk Capture** (aliases `multi-item capture`, `batch capture`): a capture whose
  draft holds two or more capture items.

Then regenerate the published memory indexes and check how each new term resolves and
what its closure pulls in. Bryan asked for both terms in his prompt. No code, docs,
existing strands, decision records, or linked repos change. No bead is created.

## What the terms mean (planning analysis)

Planning read `docs/capture.md`, `src/native/capture/` and
`src/native/capture_language/draft.rs`, the bob-mac-capture README, the
`mac-capture-is-a-thin-client` decision, the history
(`a8c9ad8 feat(capture)!: add atomic batch capture support`), and beads `bob-cli-27`,
`bob-cli-27.1`, and `bob-cli-4i`, which say "bulk". It also ran read-only
`bob capture --dry-run` and `capture-parse` checks. Four things the strands must get
right:

1. **A capture is one run of a whole command language.** "Capture" sounds like GTD inbox
   capture, but `bob capture` also creates bullets, sub-bullets, and project notes. It
   queues reference jobs, links, starts, completes, and relates existing tasks, and
   starts, adjusts, shifts, and closes Pomodoros. A capture is the unit of submission
   and atomicity. Bob Mac Capture is a thin client: it sends a draft as exactly one
   `bob capture` call and never splits it into several mutating subprocesses. bob plans
   against staged snapshots and checks each note's preimage before writing, so a run
   writes all or nothing. `--dry-run` is the same planner without the commit, and it is
   how the Mac app builds its live preview.
2. **The grammar works on capture items.** It does not work on lines or on the whole
   draft. An item is a parent line plus optional authored child bullets, so a multi-line
   draft can still be a single item. Unfinished picker queries (`:dee`, `/fix`, `!fix`,
   `&`, a prose-terminal `+`) are refused, never captured as prose. Planning confirmed
   this with `--dry-run`.
3. **Bulk capture is not a separate mode or command.** It follows from the item
   separators: blank or whitespace-only lines, a blank-line-free block of two or more
   bare-URL lines (one item per URL, from `is_url_list_line` in `draft.rs`), and a line
   that chains session operators (`+2 =x`, one item per token). Its semantics are
   sequential composition under one staged planner (`CaptureBatchPlanner`) with
   all-or-nothing rollback. Each item is judged alone, which is why bulk linking and
   bulk `!` completion work. Only `@@route` spans items. The visible differences are a
   `captures` array in JSON, `items` in `capture-parse`, and `1/N` human numbering. The
   Mac app shows a stacked preview and a count-titled notification, keeps the
   **Capture** footer verb instead of a single-item verb, and Shift-Return in the `!`
   picker appends a blank line plus `!`.
4. **"batch" is ambiguous.** Docs and code use "batch" for every capture's staged write
   set, single items included (batch-level `pomodoro_blocks`/`task_blocks`,
   `plan.strict` refusals). "Bulk" and "multi-item" always mean two or more items. The
   Bulk Capture strand states this.

## Implementation

### 1. Add the two strands

Run `/sase_memory_write` and record that you used it before editing memory. Read
existing terms only through `/sase_memory_read`. Follow the existing strand convention
(compare `keep-streak.md`): frontmatter has `keyword` and optional `aliases`, and no
flat-note `type:` key. Rewrapping lines is fine. Keep the wording, inline code, and
links.

> [!decision] memory_capture_term

Create `sase/memory/glossary/capture.md`:

```markdown
---
keyword: Capture
---

One `bob capture` run over one draft: its arguments, piped stdin, or Bob Mac Capture's
panel, which submits the whole draft as exactly one `bob capture` call and hides only
once bob reports success (see [[decisions:mac-capture-is-a-thin-client]]). bob splits
the draft into capture items, each a parent line plus optional authored child bullets,
and each item's markers decide what it does. That is far more than GTD inbox capture: an
item may create a task (default `mac_inbox.md`, an [[inbox-file]]), bullet, sub-bullet,
or project note; queue a bare public URL as a background reference job, never touching
the network; link, start, complete, or relate existing tasks (`@route:id`, `!note:id`,
`&`, `*`); or start, adjust, shift, or close today's Pomodoro (`=`, `+N`, `++N`, `=x`).
bob plans every edit against in-memory snapshots of the target notes and today's daily
note, then writes all or nothing: a note changed on disk since planning, or any failed
item, aborts the run and rolls back anything already written. `--dry-run` runs the same
planner without writing; the app's live preview is `bob capture --dry-run --no-clip`. An
unfinished picker query (`:dee`, `/fix`, `!fix`, `&`, `Call the bank +`) is refused,
never captured as prose. The editor-facing `capture-*` companion commands are not
captures, nor are `bob gkeep pull`, `bob ref create`, or Obsidian keymaps. A draft with
two or more items is a [[bulk-capture]]; the grammar lives in `docs/capture.md`.
```

> [!decision] memory_bulk_capture_term

Create `sase/memory/glossary/bulk-capture.md`:

```markdown
---
keyword: Bulk Capture
aliases:
  - "multi-item capture"
  - "batch capture"
---

A capture whose draft holds two or more capture items. Blank or whitespace-only lines
separate items; a blank-line-free block of two or more bare-URL lines splits into one
item per URL, and a line chaining session operators (`+2 =x`, `=x =`) into one item per
operator. Authored child bullets never start an item. Each item is still judged alone,
with its own markers and whole-item forms, so bulk linking (several `@route:id` items)
and bulk completion (one `!note:id` per item) work. Only a `@@route` declaration spans
items: it routes every item that lacks its own route or mode marker. One staged planner
(`CaptureBatchPlanner`) runs the items in order, so each item sees the edits planned
before it (insertion points, block-ID checks, Pomodoro lineup numbers) exactly as
sequential captures would, and any item's failure rolls the whole draft back. JSON
output puts the first result in the legacy top-level fields plus an ordered `captures`
array (`capture-parse` adds `items`), and human output numbers items `1/N`. Bob Mac
Capture still submits one `bob capture` call: the preview stacks items under a count,
the footer stays **Capture** instead of a single-item verb such as Link, Start, Close,
Complete, or Queue, the notification is titled with the count, and Shift-Return in the
`!` picker appends a blank line plus `!` for the next completion. Docs and code also say
"batch" for any capture's staged writes, even a single item's.
```

Why the bodies are shaped this way:

- **Mention economy.** The glossary web uses `closure: mentions`. Any other term's
  keyword, alias, or derived plural in prose pulls in that term's whole body,
  transitively. Inline code is skipped, and `-`/`_` count as word characters. Both
  bodies were written to avoid prose triggers: "keeps" (Keep Streak alias, hence "puts
  the first result"), "freshness", "work log", "schedule log", "task link", "dep link",
  and "inbox file" as a phrase. Capture reaches Bulk Capture and Inbox File only through
  `[[...]]` links, so readers of Capture are not charged for either body. Bulk Capture
  mentions "capture" in prose on purpose, because its definition depends on Capture.
- **The Mac app is part of each definition, not a separate term.** Bryan asked about
  both `bob capture` and bob-mac-capture. Each strand says what the app does with that
  concept, and the thin-client decision carries the rationale.

### 2. Regenerate the published memory indexes

Run `sase memory init`. Expect these eight generated or roster files to change, the same
set as earlier glossary additions (`16aba77`, `ea92b38`): `AGENTS.md`, `CLAUDE.md`,
`GEMINI.md`, `OPENCODE.md`, `QWEN.md`, `sase/memory/README.md`, and
`sase/memory/glossary.md`, plus the two new strands. Never hand-edit generated output.
The only semantic change should be `Bulk Capture (multi-item capture, batch capture)`
and `Capture` added to the `GLOSSARY TERMS` roster.

### 3. Verify

- `sase memory init --check` reports no drift.
- `sase memory web show glossary` lists both strands with slugs `capture` and
  `bulk-capture`, and Bulk Capture shows both aliases.
- `sase memory read glossary:capture -r "<why>"` prints the Capture body once, with no
  unresolved-link warnings. Its related terms should be only Pomodoro, Task Link (via
  Pomodoro), and Project Note. The linked references should be
  `decisions:mac-capture-is-a-thin-client`, `glossary:inbox-file`, and
  `glossary:bulk-capture`. It must not print the Bulk Capture, Inbox File, Keep Streak,
  Task Freshness, Work Log, or Schedule Log bodies.
- `sase memory read "glossary:bulk capture" -r "<why>"` prints Bulk Capture with Capture
  as a depth-1 related term, plus Capture's own closure. It must not print Keep Streak,
  Task Freshness, Work Log, or Schedule Log.
- `sase memory read "glossary:multi-item capture" -r "<why>"` and
  `sase memory read "glossary:batch capture" -r "<why>"` both resolve to Bulk Capture.
- If a check pulls an unexpected term, reword only the phrase that triggers it, keep its
  meaning, and re-run the check.
- Expected and accepted: Area Note ("capture targets"), Inbox File ("Bob Mac Capture",
  "capture route"), and Mac Menu Bar Pomodoro Indicator ("Bob Mac Capture's menu-bar
  item") already mention "capture" in prose, so reading them now also pulls in Capture.
  No existing strand mentions "bulk capture" or either alias. Do not reword existing
  strands. That would be an unrequested memory edit.
- `git status` shows only the two new strands and the generated files from step 2.
