---
tier: epic
title: Open Pomodoro names follow their tasks' types
goal: "Every live Task Link under one of today's open, typed Pomodoros has a task type
  that the entry's name contains, and the name holds no type that none of its links
  carry. Gestures keep this true at their shared write paths, the close settles each
  finished session's name, and `bob task reconcile` is the safety net. It repairs open
  names in both directions and never writes a `[type::]` field.

  "
decisions:
  reconcile_rollout:
    ask:
      Should reconcile apply name repairs when its phase lands, after a live-vault dry
      run?
    choices:
      apply:
        Repairs write as soon as the phase lands; a read-only live-vault dry run checks
        them first
      report_first:
        Ship report-only; a follow-up bead turns on writes after a day or two of reports
    default: apply
    why: Writer fixes land first, and every repair is reported and git-revertable
    answer: apply
  meet_links_grow_names:
    ask:
      Should a MEET Event Task link grow a work entry's name (BUILD to BUILD + MEET), as
      Enter landing does?
    choices:
      merge:
        Reconcile appends MEET like any type; the session counts toward MEET in Stats
      report: Reconcile leaves the name and reports the link; Enter landing is unchanged
    default: merge
    why:
      One add rule for landing and reconcile keeps every gesture's output a fixed point
    answer: merge
  memory_names_decision:
    ask:
      Add the decision record open-pomodoro-names-follow-task-links for names following
      task types?
    memory:
      - decisions
    default: true
    requested:
      I agree with all of the requirements recommended in that research file (but keep
      in mind that you are responsible for planning and splitting up this work
      appropriately).
    answer: true
  memory_glossary:
    ask:
      Update the Task Type and Merged Pomodoro glossary strands for name repair, close
      settle, and new entries?
    memory:
      - glossary:task-type
      - glossary:merged-pomodoro
    default: true
    corrects: Disagreement is shown as drift; it is not silently repaired.
    answer: true
phases:
  - id: plugin-writer-bugs
    title: Fix the Obsidian rename regression and the Task Card live-link lookup
    depends_on: []
    size: medium
    description:
      "plugin-writer-bugs: in bob-navigation-hotkeys, make Pomodoro rename (commit and
      preview) retype linked tasks again, and make the Task Card type, inbox route, and
      note-move live-link lookup search open entries only, by vault path plus block ID."
  - id: agree-rule
    title: Shared agreement rule, close settle, and Rust ledger applier
    depends_on: []
    size: medium
    description:
      "agree-rule: add the pure `agree_entry` and `settle_closed` planners, the MEET
      guard in `relocate`, new shared vector sections, and one Rust applier that agrees
      chosen entries in a day's text through a pluggable task-type lookup."
  - id: reconcile-pass
    title: Type-agreement pass in bob task reconcile
    depends_on:
      - agree-rule
      - plugin-writer-bugs
    size: medium
    description:
      "reconcile-pass: run the applier over today's open entries after blocked-link
      consumption; report renames and unrepaired cases in additive JSON and human
      output; narrow and test the `type_mismatch` lint; write the docs, decision record,
      and task-type glossary."
  - id: ledger-tools-mirror
    title: Ledger Tools JS mirror, api.themes additions, and inherited type marks
    depends_on:
      - agree-rule
    size: medium
    description:
      "ledger-tools-mirror: copy the vectors, mirror `agreeEntry`, `settleClosed`, and
      the relocation MEET guard in JS, expose them and an async `agreeLedgerEntries` on
      api.themes, and let the type mark include inherited types."
  - id: bob-writers
    title: Bob-side writers agree the entries they touch
    depends_on:
      - agree-rule
    size: medium
    description:
      "bob-writers: run the applier over every entry that a capture batch, `bob task
      cancel` or `complete`, Ensure Next, or a D8 MoveTo source changes, and match
      `find_live_link` on path plus block ID."
  - id: bob-close-and-naming
    title: Close settle and new typed entries in bob
    depends_on:
      - bob-writers
    size: medium
    description:
      "bob-close-and-naming: close names settle to the worked types and the continuation
      is agreement-stable; a capture `#NAME` / `=name` entry created by the same run
      retypes the tasks landing in it."
  - id: nav-writers
    title: Navigation hotkeys writers agree the entries they touch
    depends_on:
      - plugin-writer-bugs
      - ledger-tools-mirror
    size: medium
    description:
      "nav-writers: agree every open entry that a deferred link cleanup, Ctrl+Shift+M
      source, or Task Card MoveTo source touches, in the same write; Ctrl+Shift+M into a
      new named entry retypes."
  - id: prompt-and-cycler
    title: Block ID Prompt writers and the task-status-cycler close
    depends_on:
      - ledger-tools-mirror
    size: medium
    description:
      "prompt-and-cycler: block-id-prompt unlink, delete, and duplicate prune agree
      touched entries, and its new-entry picker retypes; the cycler close settles the
      closed name and keeps the continuation agreement-stable."
  - id: audit-and-records
    title: Live-vault audit, docs coherence, and the merged-Pomodoro glossary
    depends_on:
      - reconcile-pass
      - bob-close-and-naming
      - nav-writers
      - prompt-and-cycler
    size: small
    description:
      "audit-and-records: check the shared vectors are byte-identical, make the routing
      and merged-Pomodoro docs read as one contract, update the merged-Pomodoro
      glossary, and run a live-vault dry run, filing task beads for any writer that
      still leaks."
proposed_by: bbugyi200.athena.research.5q.linker.w0
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-05 10:26:32
status: wip
---

- **PROMPT:**
  [prompts/202611/pomodoro_names_follow_task_types.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/pomodoro_names_follow_task_types.md)

# Plan: Open Pomodoro names follow their tasks' types

## Why

Bryan wants every Obsidian task (typed explicitly or by inheritance) to have a type that
matches one of the types of the Pomodoro holding its Task Link. `bob task reconcile`
should be the fallback that adds or removes types on Pomodoro lines. The keymaps that
break the rule should be fixed so the fallback rarely fires. Custom Pomodoro names may
be an exception, but they must not drive the design or break.

This plan implements the recommended solution of the research report
`research:202611/pomodoro_names_follow_task_types/pomodoro_names_follow_task_types.md`
(read it with `sase artifact read` for evidence and line references). Bryan accepted all
of its requirement adjustments, A1 to A10, and its recommended answers to open questions
1 to 3 and 5. Question 4 is decision `meet_links_grow_names` below.

Two leaks are verified today:

- **I1, coverage:** a task's type is missing from its entry's name.
- **I2, support:** an entry's name keeps a type that none of its links carry.

Since merge-on-landing shipped (Nov 2), I2 dominates, because no link-removal path
except close, split, and the Ctrl+Shift+M bullet move ever removes a type.

The written policy says drift is "flagged and never silently repaired"
(`docs/task-types.md`, Routing section, Agreement bullet). This epic reverses that
policy for today's open entries and records the change as a decision.

## Shared design contract

All phases implement this one contract. Rust (`src/native/pomodoro_themes/`) is the
source of truth. The Ledger Tools JS mirror passes the same vectors
(`tests/fixtures/pomodoro_themes/vectors.json`, copied byte-for-byte into
`plugins/bob-ledger-tools/scripts/fixtures/pomodoro_themes/` in bob-plugins).

### Vocabulary

- **Entry**: a column-0 Pomodoro checkbox item under `## Pomodoros` in today's daily
  note. Its **name** is the ` — NAME` tail.
- **Type components** come from `routing::type_components` (unchanged). With a
  configured `types:` vocabulary, components in the vocabulary are types and the rest
  are labels. Without one, every component except inventory labels is a type.
- **Container**: an entry with no type components (unnamed `()`, `(unnamed)`, label-only
  names such as `LATER` or `SASE V18`). A container is never renamed, which covers
  custom names (A4). No new "custom name" class or flag.
- **MEET entry**: a name that normalizes to `meet` as a whole (`routing::is_meet_name`).
  It is never renamed by these rules (A5). `BUILD + MEET` is an ordinary merged entry.
- **Direct children** of an entry are its depth-1 child bullets. Each is classified as:
  - `Live { key, name }`: a dedicated Task Link bullet (the link plus its optional
    marker and nothing else, the definition `planDeferredPomodoroLinkCleanup` and the
    Rust ledger scanners already use) that is unstruck and not embedded, and points to
    an open task with an effective type.
  - `Worked { key, name }`: a dedicated Task Link bullet that is struck, `🍅`-marked, or
    an embedded completed link. It is typed by the task's effective type computed **as
    if the task were open** (`suggest_for_task` refuses closed tasks with `closed_task`,
    so add an as-if-open variant; the close already reads pre-close types for the same
    reason, see `capture_pomodoro_close/names.rs`).
  - `Unknown { reason }`: a dedicated Task Link bullet whose target is unresolved or
    ambiguous, whose task has no effective type, or whose type field needs repair.
  - `Prose`: anything else, including a bullet that mixes a link with text, mentions,
    and non-task embeds.

### The rule: `agree_entry(name, open, children, sources) → outcome`

The outcome carries the new name, `added`, `removed`, and `unrepaired` items.

1. **Skip.** A closed entry, a container, or a MEET entry keeps its name. Under a MEET
   entry, each non-MEET `Live` child is reported as unrepaired `meet_mismatch`.
2. **Add.** For each `Live` child, in child order, whose key equals no component key of
   the name (labels included, matching `routing::agrees`), append its type in catalog
   spelling, deduped by key.
   - A type is appended only if it would classify as a type component under the same
     sources. Otherwise it would become a label that can never be removed, so report it
     as `unconfigured_type` instead (A6).
   - `Worked` children never add. They only support (the close settles them).
   - **Decision `meet_links_grow_names`:**
     - `merge`: a MEET-typed `Live` child adds MEET like any other type.
     - `report`: a MEET-typed `Live` child under a non-MEET entry is never added. It is
       reported as `meet_link` and still counts as support if MEET is already present.
3. **Drop**, guarded (A2). Drop only when the entry has at least one child and every
   direct child is `Live` or `Worked`. Then remove each type component whose key no
   `Live` or `Worked` child carries. Labels always stay.
   - Any `Prose` or `Unknown` child protects every existing type. This is the escape
     hatch for a deliberately planned extra type: add a prose child.
   - Never drop the last type component. If the drop would leave no type component, keep
     the name unchanged. This happens only when the typed children are all `Worked`; the
     close settles that entry.
4. **Order.** When exactly one type is removed and exactly one is added, the added type
   takes the removed one's position (`— BOB + FIX` with a BUILD link becomes
   `— BOB + BUILD`, as `relocate`'s `replace_type_components` does). Otherwise drop
   first, then append the additions in child order.
5. **Bytes.** Only the name tail changes. The checkbox, timespan, `[t::]` / `[off::]`,
   and every child stay byte-identical. An entry that already agrees produces no
   rewrite. The rule is idempotent: `agree_entry(agree_entry(x)) == agree_entry(x)`.

| Before                        | Direct children                 | After                               | Why                                                                        |
| ----------------------------- | ------------------------------- | ----------------------------------- | -------------------------------------------------------------------------- |
| `— FIX`                       | BUILD live                      | `— BUILD`                           | add BUILD, drop FIX, substitute in place                                   |
| `— BUILD`                     | BUILD live, FIX live            | `— BUILD + FIX`                     | add (same result as an Enter merge)                                        |
| `— FIX + BUILD`               | FIX live                        | `— FIX`                             | the Nov 4 leftover                                                         |
| `— BUILD + FIX` (running)     | BUILD live, FIX struck          | unchanged                           | the worked link supports FIX                                               |
| `— BUILD + FIX`               | BUILD live, one unresolved link | unchanged                           | an Unknown child blocks removal                                            |
| `— BUILD + WRITE`             | BUILD live, prose "draft post"  | unchanged                           | prose protects the planned type                                            |
| `— BOB + FIX`                 | BUILD live                      | `— BOB + BUILD`                     | the label is kept                                                          |
| `— FIX`                       | BUILD worked only               | unchanged                           | never drop the last type; close settles                                    |
| `— LATER`, `— SASE V18`, `()` | anything                        | unchanged                           | container (A4)                                                             |
| `— MEET`                      | WRITE live                      | unchanged, `meet_mismatch` reported | A5                                                                         |
| `— FIX`                       | no children                     | unchanged                           | nothing to agree with (reconcile deletes childless planned entries anyway) |
| `[x] … — FIX`                 | BUILD worked                    | unchanged                           | closed history (A1)                                                        |

The existing helpers become special cases of this rule:

- the R2 Enter merge is the add step for one link;
- R7 `shed` is a drop with known before and after sets;
- relocation's rename branches are the substitution.

### Close settle (A7): `settle_closed(name, worked, sources)`

This replaces `merged::narrow_closed` (and JS `narrowClosed`) as the closed session's
name rule.

- A MEET entry, a container, or a close with no typed worked link keeps its name.
- Otherwise the closed name becomes the labels plus exactly the worked types:
  - type components whose key is worked stay, in the name's order and spelling;
  - worked types missing from the name are appended in link order, in catalog spelling,
    only if they classify as types;
  - when exactly one type is dropped and one is added, the added one substitutes in
    place.
- Single-type names may rename. A `— FIX` session that worked only a BUILD task closes
  as `— BUILD`.
- Prose children do not protect at close: the closed name records what was worked.
- "Worked" keeps R6's meaning: in progress, parked, or complete.

The **continuation** that a close creates (and the one successor linking reuses or
creates) must be a fixed point of `agree_entry` over its own children. Keep today's
`shed(name, before, carried)` for the no-carried-link case (it keeps the base). Then run
`agree_entry` over the continuation's actual children, so types that no carried link
supports are dropped as reconcile would drop them.

### New typed entries retype (A10)

When a gesture creates a **new** entry from a name the user typed or picked, the tasks
landing in it take the explicit retype path: `land(..., retype: true)`, the same D6 path
that renaming an entry uses. They do not merge into `NAME + T`.

- Under that path, a task whose type is already any component of the name stays
  consistent, so typing `BUILD + READ` keeps BUILD and READ tasks as they are.
- A container name types nothing.

Writers in scope:

| Writer                                                                                                              | Phase                  |
| ------------------------------------------------------------------------------------------------------------------- | ---------------------- |
| Capture `#NAME` / `=name` when the destination entry is created by the same capture run                             | `bob-close-and-naming` |
| Block ID Prompt new-entry picker (`078-pomodoro-theme-write.js`, which merges today)                                | `prompt-and-cycler`    |
| Ctrl+Shift+M into a new named entry, or a bullet split into a new entry (`651-pomodoro-bullet-type-move-commit.js`) | `nav-writers`          |

Enter into an **existing** entry still merges, and Alt+Enter still retypes. A10 is
required by A8: the merged result `FIX + BUILD` holding only a BUILD link is not a fixed
point, and reconcile would strip FIX.

### Fixed points (A8)

Every gesture's ledger output must already satisfy the rule:
`agree_entry(output) == output` for each entry the gesture touched. Writers enforce this
at shared choke points (the Rust applier and the JS `agreeLedgerEntries`), not gesture
by gesture. When type inputs are unavailable or incomplete, a writer skips agreement and
never blocks the gesture; reconcile catches up.

### What stays the same

- Reconcile never writes `[type::]` fields and never moves links (A3).
- Closed entries are history (A1). Only the close itself names a closed session.
- A hand-typed entry name is not authoritative (A9). Reconcile reverts a hand-typed
  `— BUILD` over `— FIX` when the links disagree. The supported rename is the
  Ctrl+Shift+M rename keymap, which retypes the tasks once `plugin-writer-bugs` lands.
- Bob Mac Capture stays a thin client. It needs no change, because capture-side
  agreement happens inside bob.
- No new CLI flags or subcommands. The JSON additions are additive, and `schema_version`
  stays 1.
- bob-cli release rules apply: never edit the `Cargo.toml` version or `CHANGELOG.md`.
  Mark a commit `!` only where a phase below says so.

## Phase `plugin-writer-bugs`

Repo: bob-plugins (open it with `sase repo open bob-plugins`), plugin
`bob-navigation-hotkeys`. Edit `src/` fragments, run `npm run build`, never hand-edit
`main.js`, and keep each fragment at or below 1000 lines.

1. **Rename regression.** Commit `71ae1cb` (Nov 2) changed the meaning of the `{}`
   options, so the rename path now asks `planLanding` to merge. The task writes then
   drop the merge result, and the toast says "keeps X".
   - In the rename branch of `650-plugin-move-commit.js` (the
     `pomodoroBulletMoveTypePlanRows(renameTypeApi, freshPreview, plan.name, {})` call)
     and in the rename preview in `200a-pomodoro-bullet-type-move.js` (the
     `pomodoroBulletMoveTypePlanRows(preview.api, preview, ..., {})` call), pass
     `{ typeMode: "retype" }`.
   - Delete `gestureRetypeRenamePlanForTask` if it is still unused, or route through it.
   - Add a regression test that loads the production Ledger Tools `planLanding` /
     routing helpers instead of the stub the existing test uses. Cover a typed name, a
     Recent row, an inherited-type task (Clear), and an explicit-field task (Set).
2. **Task Card live-link lookup.** Fix `taskCardTypeFindLiveLink` in
   `316-task-card-type-relocation.js`.
   - Search only open entries; skip closed and cancelled ones.
   - Match on (resolved vault path, block ID), never on the block ID alone. A same-ID
     link into another note never matches.
   - The carried-task case must relocate: a `🍅` link under a closed entry plus a live
     link under the open continuation.
   - Two or more open matches return an `ambiguous` result. The preview says so instead
     of "no live link · ledger unchanged". The type write may proceed without
     relocation, and reconcile repairs the name.
   - Update every caller: `352-picker-task-card-theme-stage.js`,
     `655-plugin-inbox-route.js`, and the `700-exports.js` export.
3. Tests: run `npm run build`, `npm test`, and `npm run docs:check` when docs change.
   Update the plugin docs if they describe the rename or Task Card consequence copy. Run
   `bob plugins sync`.

Acceptance:

- Renaming an entry from `— FIX` to `— BUILD` in the plugin harness retypes its FIX
  tasks. The preview shows the retype count.
- A carried task's Task Card type edit relocates its live link.

## Phase `agree-rule`

Repo: bob-cli. This phase adds the pure planners and the shared applier. It changes no
user-visible behavior except the `relocate` MEET guard.

1. **Pure planners.** Add `src/native/pomodoro_themes/agree.rs`, beside `routing.rs` and
   `merged.rs`, with `agree_entry` and `settle_closed` exactly as in the shared
   contract.
   - The child classes are plain data (`AgreeChild::{Live, Worked, Unknown, Prose}`).
   - The outcome carries `name`, `added`, `removed`, and `unrepaired` (codes
     `meet_mismatch`, `unconfigured_type`, `unknown_link`, and `meet_link` under the
     `report` branch of `meet_links_grow_names`).
   - Keep `merged::narrow_closed` until phase `bob-close-and-naming` switches its last
     caller.
2. **Relocate MEET guard.** `routing::relocate` never renames or grows a MEET entry. A
   running MEET source is `Unchanged`. A planned MEET source may still `MoveTo` or
   `CreateBelow`, but never `RenamePlaceholder`. The `type_mismatch` lint covers what is
   left.
3. **Shared vectors.** In `tests/fixtures/pomodoro_themes/vectors.json`, add an
   `agree_entry` section, a `settle` section, and MEET cases in `relocate`.
   - `agree_entry` needs every row of the contract table plus: substitution, append
     order, label preservation, the worked-only guard, empty children, the unconfigured
     type, a MEET-typed live link under `— BUILD` (expected output follows the accepted
     `meet_links_grow_names` answer only), and an idempotence assertion that every
     expected output maps to itself.
   - `settle` needs: single-type rename, missing worked type appended, labels kept, MEET
     unchanged, and no worked links.
   - The Rust tests in `pomodoro_themes/tests.rs` and `routing.rs` consume the new
     sections. Do not mark them `rust_only`: the JS mirror runs them.
4. **Ledger applier.** Add `src/native/pomodoro_themes/agree_apply.rs`, one reusable
   applier for reconcile and every bob writer:
   - Input: a day's text, an entry selector (all open entries, or a set of entry lines),
     and a `TaskTypeLookup` trait. The trait classifies one link target (route or path
     plus block ID, with struck, embedded, and marker facts) into `Live` / `Worked` /
     `Unknown`. Callers implement it over their own snapshots: reconcile's staged
     outputs, or the capture planner's staged files.
   - It scans entries with the existing ledger scanner (`capture_pomodoros::scan` plus
     the span helpers `relocation_apply` uses), classifies direct children, runs
     `agree_entry`, and rewrites name tails through `relocation_apply::rename_entry` (or
     a shared extraction of it).
   - Output: the new text plus per-entry renames (`line`, `before`, `after`, `added`,
     `removed`) and unrepaired items.
   - It preserves CRLF line endings and a missing final newline.
   - Provide a default lookup over a `HomeResolver` (one per run, as
     `plan_budget::with_type_mismatch_lints` does) with the as-if-open typing for closed
     tasks.
   - Test the applier on in-memory fixtures, including idempotence and byte
     preservation.
5. **Docs** (read `sase memory read docs.md` first).
   - In `docs/task-types.md` "Routing: agreement, landing, placement, relocation",
     remove the stale sentences ("Named capture `#NAME` / `=name` forms still call
     `land` with `retype: true` until the capture merge phase" and "Task Card
     integration remains later work").
   - Add an "Agreement rule" subsection describing `agree_entry` and the settle rule.
   - Add the new sections to "Shared vectors". Update the sentence claiming the routing
     vectors are reserved for a later JS mirror if it is still stale.
   - Leave the Agreement bullet's repair policy to `reconcile-pass`.

Commit as `feat(task-types): ...`, without `!`.

## Phase `reconcile-pass`

Repo: bob-cli, `src/native/task_status_hooks/`. Read the `docs.md` reference memory
before editing docs.

1. **Slot.** In `sync.rs`, run the applier after `consume_blocked_links_after_compose`
   and before `apply_guarded_outputs`.
   - Blocked-link consumption runs after composition and deletes the entries it empties,
     so an earlier pass would grow names for links that are about to vanish.
   - Operate on today's daily output in `compose.outputs` (fall back to
     `normalized_daily` when the daily note has no output).
   - Only today's daily note, and only open entries.
   - Renaming never empties an entry, so no later cleanup is needed.
2. **Lookup.** Implement `TaskTypeLookup` over this run's resolver and staged
   post-images, so a task whose line this run rewrote is typed from the new line.
   - Inspect each entry's own children. Never use the deduplicated `today_tasks` list,
     which keeps only the first occurrence.
   - Reuse one `HomeResolver` and cache per run.
3. **Writes.** Name-tail edits are structural: they obey the quiet period and the
   stale-input guard like the rest of the daily rewrite.
   - Extend the guarded read-set to the type configuration (the config holding `types:`
     and `plan.default_type` / `plan.reading_type`) and to the area and project notes
     that the inheritance walk read.
   - A run changing any of these between planning and writing must defer, not write
     stale names.
4. **Output.** All additions are additive; `schema_version` stays 1.
   - `type_agreement`: `[{line, before, after, added[], removed[], reason}]`, where
     `reason` is a short human phrase such as `BUILD has no link`.
   - `type_agreement_unrepaired`: `[{line, entry, link?, code, message}]` for
     `meet_mismatch`, `unknown_link`, `unconfigured_type`, and `meet_link`.
   - Human output reads like `≈ renamed — FIX + BUILD → — FIX (BUILD has no link)`, with
     "would rename" under `--dry-run`. List the unrepaired items briefly.
   - Count renames in `change_count` in `output.rs`. Dry run and apply must plan
     identical edits.
5. **Lint.** `bob plan`'s `type_mismatch` (`plan_budget::with_type_mismatch_lints`) now
   reports only what reconcile refuses: MEET mismatches, unconfigured types, and unknown
   links.
   - Update its doc comment ("flagged and never silently repaired" becomes false).
   - Add its first tests: a repaired-shape case yields no lint, and a MEET case yields
     one.
6. **Decision `reconcile_rollout`.**

   > [!decision] reconcile_rollout = apply The pass writes. Before committing, build the
   > workspace binary and run `bob task reconcile --dry-run -f json` against Bryan's
   > live vault (read-only). Check that every `type_agreement` item is correct under the
   > rule. Record the count and any surprises in the phase bead notes.

   > [!decision] reconcile_rollout = report_first Plan and report the renames, but leave
   > name tails unwritten even when not under `--dry-run`. Human output says "would
   > rename". Use `/sase_new_task` to file a `small` feature task bead: "turn on
   > reconcile type-agreement writes after reviewing reports".

7. **Docs.**
   - In `docs/task-status-hooks.md`, add a "Pomodoro type agreement" section near "Empty
     Pomodoros". Cover the slot, the rule summary, guards, writes, output fields, and
     the costs: hand-typed renames revert, running sessions can be renamed while you
     watch, and reconcile makes unattended writes behind the quiet period.
   - Update the `type_mismatch` row in `docs/plan.md`.
   - In `docs/task-types.md`, rewrite the Agreement bullet: writers keep agreement, and
     reconcile repairs today's open names; no task field is ever written.
   - Update the gesture table there.
   - Run `just docs-check`.
8. **Memory.** Use `/sase_memory_write` and honor this epic's memory decisions.
   - `memory_names_decision`: add
     `sase/memory/decisions/open-pomodoro-names-follow-task-links.md`.
     - **Applies to:** bob-cli, bob-plugins, and the vault.
     - **Claim:** the shared contract's rule and enforcement layers (writers, close
       settle, reconcile).
     - **Why:** the task is the deliberate, durable fact; names change at today's daily
       only; Enter already merges.
     - **Rejected alternatives:** warning-only; reconcile retypes tasks; reconcile moves
       links; add-only; removal only with same-run proof; rebuilding names with no
       guards; provenance or pinned metadata; read-time derived names; an in-editor
       normalizer.
     - **Cost:** the four costs listed in the docs step.
     - **Reopens when:** deliberately planned types get dropped more than rarely.
     - **Evidence:** the research ref and this plan.
     - No existing record is superseded: D5 lived in `docs/`, not in a record.
   - `memory_glossary` (task-type strand): in the `task-type` strand, replace
     "Disagreement is shown as drift; it is not silently repaired." with one sentence
     saying that writers keep open names in agreement and `bob task reconcile` repairs
     today's open names, never task fields. Link
     `[[decisions:open-pomodoro-names-follow-task-links]]`.
   - Run `sase memory init`.

Tests (integration, using production routing helpers, never stubs):

- running twice changes nothing the second time;
- dry run equals apply;
- closed entries stay byte-identical;
- CRLF and a missing final newline are preserved;
- same-run Blocked consumption followed by agreement;
- running and planned entries;
- containers, labels, MEET, prose, Unknown, and worked support;
- a target note rewritten by this run is typed from its post-image.

Commit as `feat(task-reconcile): ...`, without `!` (additive JSON, same command).

## Phase `ledger-tools-mirror`

Repo: bob-plugins, plugin `bob-ledger-tools`.

1. Copy `tests/fixtures/pomodoro_themes/vectors.json` from bob-cli byte-for-byte (from
   the commit that `agree-rule` landed). Update the fixture `README.md` with the commit
   hash and the section list. The JS test runs every section not marked `rust_only`,
   including `agree_entry` and `settle`.
2. In `src/279-plugin-task-type-routing.js`, mirror `agreeEntry` and `settleClosed`, and
   add the MEET guard to `planRelocation`.
3. Add to the api.themes surface in `src/279-plugin-task-theme-api.js` (additive;
   `version` stays 1):
   - `agreeEntry(input)` and `settleClosed(name, worked)`.
   - Async `agreeLedgerEntries({ path, content, entryLines })`, the JS counterpart of
     the Rust applier. It classifies each listed open entry's direct children, resolves
     each link to a task line (metadata cache plus `cachedRead` or the open editor's
     text), and types it with `taskTypeEffectiveTypeFor`. Add an as-if-open path for
     completed tasks if the suggester refuses them.
   - It applies `agreeEntry` and rewrites only name tails. It returns
     `{ content, changed, entries: [{ line, before, after, added, removed }], unrepaired }`
     and never writes a file; callers fold the content into their own guarded write.
   - When `entryLines` is omitted, it agrees every open entry.
   - When resolution is incomplete for an entry, the affected children are `Unknown`,
     which blocks only drops.
4. **Inherited type marks.** The ledger type mark (`268-plugin-ledger-type-marks.js`,
   which skips any task whose type is inherited) now marks every live link whose
   effective type (field or inherited) disagrees with its entry, using the same
   classification.
5. Tests for `agreeLedgerEntries` use real routing helpers. Then run `npm run build`,
   `npm test`, `npm run docs:check` (update the api.themes reference docs), and
   `bob plugins sync`.

## Phase `bob-writers`

Repo: bob-cli.

1. **Capture batch.** Add a final agreement stage to `CaptureBatchPlanner`
   (`src/native/capture/batch.rs` / `commit.rs`), run before commit on today's staged
   daily note.
   - The touched entries are the open entries whose direct child lines differ between
     the batch's original daily snapshot and its staged daily. Map entries by identity
     across the batch's edits, not by raw line number.
   - Run the applier with a lookup over the planner's staged files.
   - This covers capture unlink and drop, Ensure Next, link into running, dependency
     drops, consumed links, and every future capture writer.
   - Report the renames in capture JSON as an additive `type_agreement` array and in
     human output, and plan them identically under `--dry-run`.
2. **Other bob writers.** Route the ledger edits of `bob task cancel`,
   `bob task complete`, and the D8 `MoveTo` source in `relocation_apply.rs` through the
   applier over the entries they touched. `MoveTo` leaves the source with an extra type
   today.
3. **`find_live_link` identity.** In `relocation_apply.rs`, match on the resolved (path,
   block ID), not the block ID alone. A same-ID link to another note never matches.
   Several open matches are ambiguous and leave the ledger unchanged.
4. **Tests.** For each writer above, assert that the output is a fixed point:
   `agree_entry` over each touched entry changes nothing. Include a capture unlink that
   removes the last FIX link from `— BUILD + FIX`, and a cancel that does the same.
5. **Docs.** Update `docs/capture.md` (the output contract) and `docs/task-types.md`
   (gesture table, R7 wording) for agreement on touched entries.

Commit as `feat(capture): ...`, without `!` (the JSON is additive).

## Phase `bob-close-and-naming`

Repo: bob-cli.

1. **Close settle.** In `src/native/capture_pomodoro_close/names.rs`:
   - Compute the closed name with `settle_closed` from the worked links' pre-close
     types.
   - Compute the continuation name with `shed` followed by `agree_entry` over the
     continuation's carried children, so it is a fixed point.
   - Keep `CloseNames::shed` accurate.
   - Check `routing.rs`'s other `narrow_closed` / `shed` callers (around the close
     vectors). Delete `merged::narrow_closed` once nothing calls it, but keep its vector
     section if JS still runs it.
   - Single-type closes may now rename, and MEET and `x0` closes still never rename.
   - Update the close tests and the R6 text in `docs/task-types.md`.
2. **New typed entries (A10).** When capture `#NAME` / `=name` creates the destination
   entry in the same capture run, land with `retype: true` in `capture/task_theme.rs`
   (today it always uses `retype: false`).
   - An entry created earlier in the same batch counts as new.
   - The task field is set or cleared through the existing Set/Clear path, and
     `task_theme` JSON reports it.
   - Existing destinations still merge.
   - Update "Named captures merge types" in `docs/task-types.md` and the matching
     `docs/capture.md` text.
   - Mark the commit `feat(capture)!` with a `BREAKING CHANGE:` footer, because the
     documented `#NAME` behavior for new entries changes.
3. **Tests.**
   - A close of `— FIX` that worked only BUILD closes as `— BUILD`.
   - The continuation is agreement-stable.
   - `x0` and MEET closes are unchanged.
   - Capture `Fix it @route:id#FIX` with a BUILD-inherited task and no FIX entry creates
     `— FIX` and writes `[type:: FIX]`.
   - The same capture into an existing `— FIX` still merges.

## Phase `nav-writers`

Repo: bob-plugins, plugin `bob-navigation-hotkeys`.

1. **Deferred link cleanup.** `planDeferredPomodoroLinkCleanup`
   (`160-pomodoro-reorder.js`) additionally returns `touchedEntryLines`: the
   post-cleanup lines of open entries that lost a child but survived.
2. **Shared commit core.** `commitLinkPickerNoteWrites`
   (`510-plugin-link-commit-and-lane.js`) runs `api.themes.agreeLedgerEntries` on the
   daily cleanup content for those entries before writing, inside the same preimage,
   write, and rollback transaction. This covers Alt+N, Ctrl+Shift+Enter unlink, Alt+P
   and Ctrl+Enter roll, future schedule, decay Drop, cancel, and lane links.
3. **Other cleanup writers.** Audit every other `planDeferredPomodoroLinkCleanup` caller
   that writes its own daily plan, and add the same step: `250-task-move.js`,
   `216-project-dependency-stage.js`, `218` / `219` project subtree, `551` post-cancel,
   `594` / `596` relation edit and cancel, and `665` project conversion. Prefer one
   small shared helper over per-site code.
4. **Ctrl+Shift+M** (`651-pomodoro-bullet-type-move-commit.js`).
   - The source entry agrees through `agreeEntry` / `agreeLedgerEntries` instead of
     `shed`. Keep the preserve-source behavior when the source keeps a base.
   - A move into a new named entry, or a split into a new entry, uses
     `typeMode: "retype"` (A10).
   - Moves into existing entries keep Enter merge and Alt+Enter retype.
   - Update the preview copy.
5. **Task Card relocation.** A `MoveTo` (`316-task-card-type-relocation.js`) agrees the
   source entry in the same write.
6. **Degrade.** When api.themes is missing or types are incomplete, skip agreement and
   still perform the gesture.
7. **Tests.** Each gesture's daily output is a fixed point under `agreeEntry`, using the
   production Ledger Tools helpers where the harness allows. Then run `npm run build`,
   `npm test`, `npm run docs:check`, and `bob plugins sync`.

## Phase `prompt-and-cycler`

Repo: bob-plugins.

1. **block-id-prompt.**
   - Its link removal paths (unlink, delete, and duplicate prune in
     `060-link-cleanup-and-deletion.js` and `075-pomodoro-link-targets.js`) agree the
     touched open entries through `api.themes.agreeLedgerEntries` in the same editor
     transaction.
   - The new-entry picker write in `078-pomodoro-theme-write.js` (which merges today)
     retypes the linked task when the entry is new (A10).
2. **task-status-cycler close** (`170-plugin-pomodoro.js`, the close-names block that
   calls `themes.narrowClosed` and `themes.shed`).
   - The closed name uses `themes.settleClosed(name, worked)`.
   - The continuation runs `shed`, then agrees over its carried children.
   - Keep the MEET skip.
   - The shared `settle` vectors keep it in parity with the Rust close.
3. **Tests, build, docs, sync** as above.

## Phase `audit-and-records`

Repo: bob-cli, plus a read of bob-plugins.

1. Confirm that bob-plugins' `pomodoro_themes/vectors.json` is byte-identical to
   bob-cli's. Re-copy it if a later phase changed the vectors.
2. **Docs coherence.** Read the "Routing" and "Merged Pomodoros" sections of
   `docs/task-types.md` end to end.
   - R6 now describes the settle. R7 describes agreement-based shedding.
   - The gesture table lists removal paths and new typed entries.
   - No sentence still says drift is never repaired.
   - Fix the phrasing so the sections read as one contract, then run `just docs-check`.
3. **Memory** (`memory_glossary` (merged-pomodoro strand), via `/sase_memory_write`). In
   the `merged-pomodoro` strand:
   - replace the close-naming sentences, including "Single-type entries never rename",
     with the settle and the agreement-stable continuation;
   - add that a new entry created from a typed name retypes the tasks landing in it;
   - keep the MEET exemption;
   - link `[[decisions:open-pomodoro-names-follow-task-links]]`;
   - run `sase memory init`.
4. **Live-vault audit.** Build the binary and run `bob task reconcile --dry-run -f json`
   against the live vault (read-only).
   - For each `type_agreement` item, find the writer through the vault's git history
     (the sync commit just before the drift names the gesture).
   - File each confirmed leak with `/sase_new_task` as a `bug` task bead with evidence.
     This includes any of the research report's unverified writers (whole-entry move,
     Ctrl+X absorbing a container, capture "link into running", `stage_merge_rename`,
     subtask, `parent:`, and project moves, the successor check, and `bob types rename`
     not rewriting `plan.default_type`).
   - Do not fix them in this phase.

## Non-goals and follow-ups

- No "custom name" class, flag, or syntax. Containers already cover custom names.
- No provenance or pinned-component metadata on names. Revisit only if the prose guard
  proves insufficient.
- No in-editor normalizer and no read-time derived names.
- Reconcile never touches previous daily notes or closed entries.
- The research report's unverified row-8 writers are fixed only when the live audit or
  later reconcile reports point at them.

## Verification

- `just test` (or the repo's equivalent `cargo` test targets), `cargo clippy`, and
  `just docs-check` in bob-cli.
- `npm test`, `npm run build:check`, and `npm run docs:check` in bob-plugins, followed
  by `bob plugins sync`.
- Both suites run the shared `agree_entry`, `settle`, and `relocate` vectors from
  byte-identical files.
- Reconcile run twice changes nothing the second time, and dry run equals apply.
- After the writer phases, a corrected gesture followed by reconcile yields zero
  `type_agreement` items.
