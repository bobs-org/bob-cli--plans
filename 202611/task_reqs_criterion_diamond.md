---
tier: epic
title: Task requirements wear the criterion diamond
goal: "Every task requirement is drawn with one quiet, constant-width diamond in its
  owner's ink, at every depth, in Obsidian, Bob Mac Capture, the successor cards, and
  CLI text. The mark's fill shows the requirement's state, indentation alone shows its
  depth, and the Mac draft shows honest bytes. Nothing about how requirements behave,
  are stored, or are typed changes.

  "
decisions:
  memory_glossary_req:
    ask:
      Rewrite the Task Requirement glossary strand's rendering and teaching lines for
      the diamond?
    memory:
      - glossary:task-requirement
    corrects:
      Obsidian and Bob Mac Capture draw a tightly kerned hash run in the owner's ink
      over
    default: true
    answer: true
  memory_req_mark_record:
    ask:
      Add a "Requirements Wear The Criterion Diamond" decision record to the decisions
      web?
    memory:
      - decisions:requirements-wear-the-criterion-diamond
    default: false
    answer: false
phases:
  - id: glyph-contract
    title: The requirement mark becomes a token in bob-cli and the plugin twins
    depends_on: []
    size: medium
    description:
      "glyph-contract: req_run becomes a per-state text glyph, the CLI strings and RV
      vectors carry the diamond, the close-card glyph becomes the `requirement` token,
      the contract page gets the full rendering spec, and the plugin twins and vector
      copies change in lockstep so every gate stays green."
  - id: line-annotations
    title: Bob annotates requirement lines in previews and the idle agenda
    depends_on:
      - glyph-contract
    size: medium
    description:
      "line-annotations: an additive `req: {depth, state}` object on capture preview
      block lines and idle-agenda lines, computed by the recognizer, so the Mac can draw
      the diamond without inferring anything from `[ ]`."
  - id: obsidian-marks
    title: Obsidian draws the criterion diamond in notes
    depends_on: []
    size: medium
    description:
      "obsidian-marks: bob-ledger-tools replaces the hash-run mask with four diamond
      masks in a constant 1em slot, state by fill, owner ink, the verified display
      fixes, real tooltips, a click guard that teaches instead of ticking, the shared
      card mark, and a light/dark taste pass."
  - id: obsidian-cards
    title: Obsidian successor cards wear the diamond
    depends_on:
      - glyph-contract
      - obsidian-marks
    size: small
    description:
      "obsidian-cards: Navigation Hotkeys draws the shared mark on requirement rows and
      the group heading, flips it with the staged verdict, and uses a diamond header
      icon on requirements-only cards."
  - id: mac-editor
    title: The Mac draft shows requirement bytes as a teal chip
    depends_on: []
    size: medium
    description:
      "mac-editor: Bob Mac Capture deletes the hash overlay, the byte concealment, and
      its caret-reveal plumbing, and paints the real `[ ]` bytes as a teal chip like the
      existing route and theme washes."
  - id: mac-diamonds
    title: Mac previews, agenda, and Pull-In card draw the diamond
    depends_on:
      - glyph-contract
      - line-annotations
      - mac-editor
    size: medium
    description:
      "mac-diamonds: one SwiftUI requirement mark drawn from bob's state in the live
      preview, the idle agenda, and the Pull-In card rows, header, and filter, with
      staged-verdict flips, regenerated fixtures, and updated Mac docs."
  - id: memory
    title: Memory records for the criterion diamond
    depends_on:
      - obsidian-cards
      - mac-diamonds
    size: small
    description:
      "memory: apply only the accepted memory decisions with /sase_memory_write, then
      regenerate memory and the glossary page."
proposed_by: bbugyi200.athena.1f8
proposed_with: claude/opus
decided_by: auto
create_time: 2026-11-23 03:54:17
status: wip
---

- **PROMPT:**
  [prompts/202611/task_reqs_criterion_diamond.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/task_reqs_criterion_diamond.md)

# Plan: Task requirements wear the criterion diamond

## Context

Bryan likes task requirements (epic `bob-cli-on`, now closed) but not how they look. A
requirement is drawn as a depth-counted hash run (`⌗⌗`, `⌗⌗⌗`, `⌗⌗⌗⌗`). In Obsidian it
reads as Markdown `##` / `###` headings. In the Mac capture panel it is grey monospace
text that looks like broken, typed Markdown. He asked to change how requirements are
displayed without changing how they behave, once `bob-cli-on` was complete, and asked
the planner to lead the design: intuitive, reliable, and beautiful.

Before starting any phase, read
`research:202611/task_reqs_display_redesign/task_reqs_display_redesign.md` with
`sase artifact read`. For geometry and CSS detail, also read its `__cld.md` (the
criterion diamond spec) and `__gem.md` (defect audit, CSS architecture, Mac chip)
companions in the same directory. Bryan endorsed every requirement the report
recommends. Memory edits are the exception: they happen only in the `memory` phase,
under the accepted memory decisions. Where this plan is more specific than the research,
this plan wins.

The research's "decide now" timing correction (ADJ-1) is moot: `bob-cli-on` closed
before this plan. Its card phases drew the run through `taskReqRun`, `req_run`, and a
Swift `depthGlyph`, so the swap is still small. The ride-along defect "Mac
caret-adjacency reveal" was fixed by the epic's landing tale (bob-mac-capture
`0b458fb1`); this plan deletes that mechanism along with the concealment it served.

### Sequencing with the true-case drawers epic

The true-case drawers epic `bob-cli-ph` started when `bob-cli-on` closed. Its rendering
phases edit the same code this epic edits:

- the plugin requirement twins and fixtures;
- `bob-ledger-tools` `styles.css`, beside the requirement section;
- the capture preview block lines (an additive `drawer` annotation);
- the Mac `paintHighlighting` and `BlockDiffCard`.

Its design also reserves the diamond for requirements ("square for a task, ring for a
Task Link, diamond for a requirement, dot for a note, the glyph for a drawer") and
matches the drawer tray's stroke weight to the requirement glyph. The two looks must be
tuned side by side, so this epic runs after the drawer rendering lands:

- **Approve with a wait.** Approve with `--wait 'bead=bob-cli-ph.7,bead=bob-cli-ph.8'`
  (or the same wait in the approval gate). Those are `obsidian-drawers` and
  `mac-drawers`. Through their dependencies they cover every drawer phase that touches a
  requirement surface: contract, plugin readers, reader rollout, writers, and capture
  drafts. The drawer migration, cutover, rename, and memory phases may still be running.
  They touch recognizers and memory, not rendering, so rebase normally.
- **Check again in every root phase.** Step 0 of `glyph-contract`, `obsidian-marks`, and
  `mac-editor`: `sase bead read bob-cli-ph.7` and `bob-cli-ph.8`. If either is still
  open, stop without changes and say so.
- **Build on what landed.**
  - Keep every drawer word (`NOTES`, `**Notes**`, or whatever `bob-cli-ph` left) exactly
    as it is in any string you touch. Change only the requirement glyph.
  - Reuse the drawer epic's patterns:
    - the owner-ink attribute pipeline;
    - attribute-only rendering;
    - Mac delimiter dimming (never hiding);
    - the additive preview annotation.

### The design in one paragraph

A requirement wears one small diamond where its checkbox was, in its owner's ink, the
same size at every depth. The diamond's fill says where the requirement stands:

- hollow `◇` while pending;
- filled `◆` once met with its task;
- faded when moot or excluded;
- slashed when dropped;
- an arrow `→` when carried into its own task.

Indentation, which already shows nesting, is the only depth cue. The tooltip, the
accessible label, and card JSON still name the exact level. The Mac draft keeps the real
`[ ]` bytes visible as a teal chip, like its route and theme chips. The live preview,
the idle agenda, and both successor cards draw the same diamond from state bob reports.
Plain text surfaces print `◇`. The grammar of marks becomes:

- a square for a task;
- a ring for a Task Link;
- a diamond for a requirement;
- a tray for a drawer;
- a dot for a note.

### Design calls made here beyond the research

1. **Wait for the drawer rendering phases** (above).
2. **The Mac draft never hides bytes.** The research recommends the native chip. This
   plan also deletes everything that only existed to conceal bytes: the overlay, the
   concealment branch, the caret-adjacency reveal, its repaint fingerprint, and the
   req-only selection plumbing. That plumbing is "the one lockstep exception" to the
   rule that selection callbacks never write the draft, so deleting it restores the
   invariant. Fewer moving parts makes the panel more reliable.
3. **The idle agenda joins.** The research missed one capture-panel surface. The empty
   panel's agenda shows task blocks whose requirement rows arrive as `kind: bullet` with
   text `[ ] …`. `line-annotations` annotates those lines, and `mac-diamonds` draws the
   diamond there too.
4. **Cards preview the staged outcome.** On both successor cards, a row's mark is `◇` by
   default and turns slashed when Drop is staged, or `→` when Own task or Pull in is
   staged. What you stage is what lands. A staged Met stays `◇`: the requirement is
   still pending until the close lands, and a card full of filled diamonds would be
   heavier without being clearer.
5. **Clicking a diamond teaches; it never ticks.** In Obsidian the mark now receives
   pointer events, so it can carry a real tooltip and hover halo. A capture-phase guard
   swallows the click and shows refusal copy that depends on the state (§5). It never
   writes.
6. **The diamond's ink is exactly its owner's ⌗ ink.** The shipped requirement ink
   ignores the `inherited` and `explicit` type states that the `#task` mark handles. The
   diamond reuses the owner's mark ink rules, so the two never disagree.
7. **Words.**
   - The group scope chip says `requirements`, not `reqs`.
   - The teaching line is rewritten for one requirement at a time.
   - The human met line uses `✓`, so it equals the met receipt. `◆` would collide with
     the CLI's default-Pomodoro marker.
   - The card footer summary `reqs ✓n ⊘n →n` keeps its compact form.
8. **One text glyph per state, never per depth.** `req_glyph(state)` (Rust) and
   `taskReqGlyph(state)` (JS) replace the run helpers. The RV vectors keep their
   historical `run` key, now `"◇"` on every row, so the fixture schema doesn't churn.
   Its README says what `run` now means.
9. **Mac ink is the identity teal** on every Mac surface. No new hue field is added
   (§8).

## Ground rules for every phase

- **Rust is the authority.** Copy and vectors live in bob-cli first. Plugins copy
  bob-cli fixtures byte for byte. The Mac stays a thin client
  ([[decisions:mac-capture-is-a-thin-client]]): it draws requirement marks only where
  bob says a line is a requirement, never by sniffing `[ ]`.
- **Display only.** No vault bytes, recognizer rules, states, gestures, keys, capture
  grammar (no `GRAMMAR_REVISION` bump), or JSON keys change. The only JSON changes are
  display values (`copy.glyph`, `copy.scope`, `copy.teaching`, `copy.human_met`, row
  `run`) and one additive optional object (`req`). None is a contract break, so no `!`
  appears in any commit subject. Each commit message names the new strings.
- **Other repositories.** Open `bob-plugins` and `bob-mac-capture` through `/sase_repo`
  and work only at the path it prints. Read each repo's `AGENTS.md` first.
- **bob-plugins.**
  - Edit `src/` fragments only, each at 1000 lines or fewer. Add a fragment rather than
    grow a near-cap one; for example, `264-plugin-task-tag-marks.js` is already at 924.
  - Run `npm run build` and `npm test`. Add any new test file to the `test:unit` list.
  - Deploy only the plugins you changed, with `bob plugins sync -p <id>`. Never pass
    `-F` unprompted.
- **Bob Mac Capture.**
  - Compile every Swift change through the scratch Mac copy and `just typecheck-tests`,
    as its `AGENTS.md` describes. Then run `just portable-check` and
    `Scripts/macos-ci-check`.
  - After every push, watch the exact-SHA macOS run to green with `/sase_monitor`.
  - If the Mac is unreachable, report the macOS targets as unverified.
- **Docs and gates.**
  - Read the `docs.md` reference memory with `/sase_memory_read` before editing docs.
  - In bob-cli, run `just docs-gen`, `just docs-check`, and `just check`.
- **Releases.** Never edit the Cargo.toml version or CHANGELOG.md.
- **Contract page.** `docs/task-requirements.md` §4 Rendering is the contract.
  `glyph-contract` writes it in full from the Design of record below. A later phase
  edits §4 only when its implementation had to deviate, and then makes §4 match what
  shipped.
- **Discovered work.** Phase workers record `PROPOSED FOLLOW-UP:` notes on their own
  phase bead and never create beads. Phases outside `memory` never edit memory.

## Design of record

### 1. Shape grammar

| Line        | Mark                                        | State shown by   |
| ----------- | ------------------------------------------- | ---------------- |
| Task        | checkbox square, plus the `⌗` identity mark | the checkbox     |
| Task Link   | ring                                        | ring fill        |
| Requirement | **diamond**                                 | **diamond fill** |
| Drawer      | tray (from `bob-cli-ph`)                    | —                |
| Note        | dot bullet                                  | —                |

"Identity by shape, state by fill" (`docs/task-link-marks.md`) now covers requirements
too. A diamond is the established milestone and criterion mark, and it is a square
turned 45°: related to the checkbox it replaces, but plainly not tickable. `◇` has no
other use in the three repos. `◆` appears only as the CLI default-Pomodoro marker, so
CLI text never prints `◆` for a requirement.

### 2. Geometry

All masks use a 16×16 viewBox, `fill="none"` unless stated, `stroke="black"`,
`stroke-width="1.75"`, and round caps and joins. That is the stroke family of the `⌗`,
the book, and the rings.

| Mark    | Path                                          | Notes                                                                                                  |
| ------- | --------------------------------------------- | ------------------------------------------------------------------------------------------------------ |
| Hollow  | `M8 1.7 14.3 8 8 14.3 1.7 8Z`                 | The half-diagonal, 6.3, sits just above the ring's 6.05 radius, so it reads as a quiet peer of a ring. |
| Filled  | same path, `fill="black"` and the same stroke | Its outline matches the hollow diamond exactly.                                                        |
| Slashed | the hollow path plus `M4.4 11.6 11.6 4.4`     | The same `/` as the cancelled ring, lengthened to cross both edges.                                    |
| Arrow   | `M2.4 8h10.4M9.2 4.3 12.9 8 9.2 11.7`         | Carried: the requirement moved out.                                                                    |

The Mac draws the same normalized points in a SwiftUI `Shape`. Text surfaces use these
fallbacks:

- `◇` (U+25C7) for pending, moot, excluded, and flagged;
- `◆` (U+25C6) for met;
- `⊘` for dropped;
- `→` for carried.

### 3. Slot, size, and ink

- **Slot.** The mark is a constant **1em** square with the existing `0.24em` trailing
  margin. Depth-1 text does not move, because today's depth-1 run is also 1em plus
  0.24em, and every requirement in the vault is at depth 1. Deeper requirements come
  back into line. Requirement text at any depth starts at the same offset from its own
  bullet.
- **Size.** The mask is drawn at **0.92em**, centered in the slot. If the taste pass
  shows it too quiet, the single tuning knob is 0.92em → 1em, never a louder shape.
- **Ink.**
  - Obsidian uses the owner's `#task`-mark ink: its type hue through the same per-hue
    and per-state rules (including `inherited`, `explicit`, `muted`, `unconfigured`, and
    `repair`), or the identity teal.
  - The Mac uses the identity teal everywhere.
  - Ink never encodes state.
- **Weight.** About a 1.4–1.6px line at body size: below the checkbox, a peer of the
  ring, and well above a note bullet.

### 4. States

| State    | When                                                    | Mark                  | Mark opacity            | Text                                                      |
| -------- | ------------------------------------------------------- | --------------------- | ----------------------- | --------------------------------------------------------- |
| Pending  | `[ ]` under an open owner                               | hollow, owner ink     | 0.9 at rest, 1 on hover | normal                                                    |
| Met      | owner done, or a hand-ticked `[x]` / `[X]`              | filled                | 0.62                    | muted, **never struck**                                   |
| Moot     | owner cancelled                                         | hollow                | 0.42                    | muted                                                     |
| Dropped  | `[-]`                                                   | slashed               | 0.55                    | muted and struck; Cancel Log beneath                      |
| Carried  | `[>]` stub                                              | arrow, in the slot    | 0.9                     | normal; the stub's wikilink keeps its live Task Link ring |
| Excluded | an ancestor is dropped or carried                       | hollow                | 0.42                    | muted, not struck                                         |
| Flagged  | pending, with any other hand mark (`[?]`, `[*]`, `[/]`) | hollow, repair orange | 0.9                     | normal                                                    |

The derived-met and hand-ticked-met cases look identical. No requirement row ever wears
a lane tint, rail, or gradient from the vault's `task-statuses.css`: a requirement has
no lane. There is no progress fraction (`2/4`) and no "Requirements" container: under
derived met, a pending ratio cannot move while the owner is open.

### 5. Interaction, labels, and copy

**Tooltip and accessible label** (identical strings, never containing `::`):

```text
Requirement of “{Owner}” · {State}[ · level {d} when d ≥ 2][ · goes with “{Ancestor}” when Excluded]
Met when this task closes. Drop it or make it a task if the scope changes.
[Alt+[ drop · Ctrl+Shift+T own task]   ← pending only
[Alt+] restore]                         ← dropped only
```

`{State}` is one of `Pending`, `Met`, `Moot`, `Dropped`, `Carried`, `Excluded`, or
`Flagged [{c}]`.

**Teaching line.** "Met when this task closes. Drop it or make it a task if the scope
changes." The same bytes appear in the Obsidian tooltip, RV `copy.teaching`, the Rust
`copy::TEACHING`, and the cycler's `REQ_CLOSE_COPY.TEACHING`.

**Click guard (Obsidian).** A click, tap, or mousedown on a requirement mark never
toggles it and never writes. It shows one Notice:

| State            | Notice                                                                                             |
| ---------------- | -------------------------------------------------------------------------------------------------- |
| Pending, Flagged | `Requirements are met when “{Owner}” closes — Ctrl+Enter on the task` (today's Ctrl+Enter refusal) |
| Met              | `Met with “{Owner}”`                                                                               |
| Moot             | `Moot — “{Owner}” was cancelled`                                                                   |
| Dropped          | `Dropped — Alt+] restores it`                                                                      |
| Carried          | `Carried — open the linked task` (today's copy)                                                    |
| Excluded         | `Goes with “{Ancestor}”`                                                                           |

**Unchanged interaction:**

- the Live Preview cursor reveal of the raw `- [ ] `;
- the tag-marks session toggle that restores native checkboxes;
- every gesture, key, and refusal in `docs/task-requirements.md` §5–§7.

### 6. Obsidian mechanics (bob-ledger-tools)

- **Masks.** Define four mask variables once on `body`: `--bob-req-glyph-pending`,
  `-met`, `-dropped`, and `-carried`. Delete the 32×16 run mask and the sixteen
  depth→count rules.
- **Host and glyph.**
  - The checkbox `input` is the 1em slot and the hover-halo host:
    - `appearance: none`, with no native border, background, or check;
    - `border-radius: 999px`, `position: relative`;
    - `pointer-events: auto`, `cursor: default`.
  - The glyph lives in `input::before`:
    - `background-color: var(--bob-req-ink)`;
    - the state's mask, centered at 0.92em;
    - `print-color-adjust: exact`.
  - `input::after` (including `:checked::after`) is suppressed on requirement rows, so
    theme or snippet check, dash, and arrow glyphs never bleed through.
- **States.** `[data-bob-req-state]` picks the mask and opacity from §4. Flagged is
  `[data-bob-req-state="pending"]:not([data-task=" "])`, which reads Obsidian's own
  `data-task`.
  - Delete the carried line/`li` `::after` arrow. In reading view it rendered below the
    whole nested subtree.
  - Excluded now fades its mark.
- **Neutralize.** Reset `task-statuses.css` lane backgrounds, gradients, and rails on
  every `[data-bob-req="true"][data-task]` line and `li`, not only `[-]`. Set
  `--checklist-done-decoration: none` and no line-through on `[x]` / `[X]` requirement
  rows.
- **Hover and motion.**
  - On hover, opacity goes to 1 with a 14% ink halo on the input, over 120ms.
  - `prefers-reduced-motion` removes the transition.
- **Forced colors.** Under `@media (forced-colors: active)`, the glyph is drawn in
  `CanvasText` (`GrayText` for faded states) with `forced-color-adjust: none`. Without
  this, a mask drawn with `background-color` vanishes in Windows High Contrast.
- **Labels.**
  - Put `aria-label`, `data-tooltip-position="top"`, and `aria-readonly="true"` on the
    input. Obsidian draws `aria-label` tooltips.
  - Remove `title` from the input, the Live Preview line, and the reading-view `li`, so
    no slow OS tooltip leaks over the prose.
- **Toggle off.** When tag marks are turned off, reading view clears rows it already
  rendered: their `data-bob-req*` attributes, labels, and saved originals. It no longer
  leaves stale attributes behind.
- **Shared card mark.** Add `span.bob-req-mark[data-state]`, a 1em inline-flex box whose
  text is the Unicode fallback. When ledger-tools styles load, the text becomes
  transparent and `::before` draws the mask for `data-state` in `--bob-req-ink` (default
  teal). With ledger-tools off, the Unicode fallback shows.
- **Gating.** Everything stays gated by `body.bob-task-tag-marks`, as today.

### 7. Successor cards

- **Obsidian (Navigation Hotkeys).**
  - Each requirement row starts with `span.bob-req-mark`, using the staged outcome as
    `data-state`:
    - `pending` for default or Met;
    - `dropped` for Drop;
    - `carried` for Own task or Pull in.
  - The 42px monospace run slot goes away. Rows indent by depth as today, which is now
    the only depth cue. The group heading starts with a pending mark.
  - A requirements-only card's modal icon becomes Lucide `diamond`, not `list-check`.
  - Staged color cues (red drop, green carry) and the receipt card are unchanged.
- **Mac Pull-In card.**
  - The same staged-outcome mark, in a fixed slot of about 1em, replaces the computed
    `⌗` run. The header and the filter capsule draw a pending mark.
  - The default verdict capsule reads **Met with task** instead of `Met`.
  - Indentation and the meta line (`level d`) keep the depth.
- **CLI human card.** Each row prints `◇` before its card number and text, indented by
  depth.
- **Footer summary.** `reqs ✓{met} ⊘{dropped} →{carried}`, unchanged.

### 8. Bob Mac Capture

- **Draft editor: a teal chip on honest bytes.**
  - For each depth-bearing `req_marker` span, paint its first three bytes `[ ]` with a
    teal foreground (a new `.requirementMarker` palette category). Add
    `requirementWash(increaseContrast:)` at the record-route wash levels (0.16, or 0.28
    with Increase Contrast).
  - The trailing space and the `- ` stay unstyled.
  - Nothing is ever hidden, and caret moves never repaint.
  - Expansion, overtype absorption, and atomic Backspace are unchanged.
- **Preview, agenda, Pull-In: the diamond.**
  - One `RequirementMark` view (a `Shape` per §2 variant, teal, opacity per §4) is drawn
    wherever bob annotates a line with `req`:
    - `BlockDiffCard` rows, replacing the `.checkbox` token's drawing;
    - idle-agenda child rows, dropping the leading `[c] ` from the shown text;
    - Pull-In rows, the header, and the filter capsule (§7).
  - The mark's size is about 0.8× the line height. Its accessibility label is
    `requirement, {state}`.
  - Older `bob` without `req` keeps today's look; nothing fails to decode.
- **No owner hue on the Mac.** Teal is the honest default the plan always allowed. An
  owner hue would need `capture-parse` to resolve owner types for new and existing
  tasks, which is new behavior. If wanted later, record it as a follow-up.

### 9. bob-cli strings and wire values

| Where                                                        | Today                                                       | New                                                                       |
| ------------------------------------------------------------ | ----------------------------------------------------------- | ------------------------------------------------------------------------- |
| `task_reqs::req_run(depth)`                                  | `⌗⌗` / `⌗⌗⌗` / `⌗⌗⌗⌗`                                       | `req_glyph(state)` from `EffectiveReqState`, per §2 fallbacks             |
| `bob capture` requirement summary                            | `⌗⌗ adds 4 requirements to “S” · files …`                   | `◇ adds 4 requirements to “S” · files …`                                  |
| Human met line (`copy::human_met_line`, RV `copy.human_met`) | `⌗⌗ {n} requirements met with “{S}”`                        | `✓ {n} requirements met with “{S}”` (equals `met_receipt`)                |
| `bob task split` candidate list                              | `styler.cyan(req_run(depth))`                               | `styler.cyan(◇)`                                                          |
| Human Pull-In card row (`req_choice.rs`)                     | `{indent}⌗⌗ {n} {text}`                                     | `{indent}◇ {n} {text}`                                                    |
| `pull_in.groups[].copy.glyph` (requirements)                 | `"⌗⌗"`                                                      | `"requirement"`, a semantic token like `relation` and `note_pull`         |
| `copy.scope`                                                 | `{n} reqs` / `{n} reqs in {b} branches`                     | `1 requirement` / `{n} requirements` / `{n} requirements in {b} branches` |
| `copy.teaching`                                              | `Part of this task's scope. …`                              | the §5 teaching line                                                      |
| RV row `run` (internal model and fixtures, not on the wire)  | depth-counted run                                           | `"◇"` on every row                                                        |
| Obsidian Ctrl+] notice                                       | `⌗⌗ First requirement[ · filed {n} note(s) under {drawer}]` | `◇ First requirement[ · …]`                                               |
| Preview `lines[]` and agenda lines                           | —                                                           | additive `req: {depth, state}` on requirement lines only                  |

### 10. What does not change

- Requirement bytes, the recognizer, depth limits, states, and the derived-met rule.
- Drop, restore, carry, toggle, typing, and every refusal and gesture.
- Capture grammar, the `req_marker` span, and `GRAMMAR_REVISION`.
- The RQ, RL, RW, and SP vectors. The RV vectors change only in the display strings in
  §9.
- The vault: no migration, and `bob task tidy` stays quiet.
- Raw surfaces (source mode, GitHub, mobile, plugins off) still show honest `- [ ]`.

## Phases

### glyph-contract — the mark becomes a token (bob-cli, then the plugin twins in lockstep)

0. Run the sequencing check (Context). Read the research report.
1. **Helper.** In `src/native/task_reqs.rs`, replace `req_run(depth)` with
   `req_glyph(state)` over `EffectiveReqState`, returning the §2 text fallback. Its doc
   comment says depth never changes the glyph.
2. **Strings.** Make every §9 change in bob-cli:
   - `capture/output.rs` (the hard-coded `⌗⌗` summary);
   - `task_complete/req_close.rs`: `human_met_line` delegates to `met_receipt`, the row
     `run`, the group `glyph` token, `scope`, and `TEACHING`;
   - `capture/req_choice.rs` (the human card row);
   - `task_split/command.rs`.

   Remove `req_run`.

3. **RV vectors.** In `tests/fixtures/req_close/vectors.json`, change only:
   - `copy.human_met` and `copy.teaching`;
   - every requirements `copy.glyph` and `copy.scope`;
   - every row `run`.

   Make sure the diff touches nothing else. The README documents the `requirement` token
   and `run` as the text-fallback mark (historical name).

4. **Tests.** Update the bob-cli tests that pin these strings:
   - `tests/cli/capture/reqs_existing.rs` and `reqs_new.rs`;
   - `src/native/task_complete/tests/req_close_tests.rs`;
   - any `reqs` scope assertions.

   Add a unit test that `req_glyph` is the same at depths 1–16 for each state.

5. **Docs.**
   - `docs/task-requirements.md`:
     - rewrite the intro line, §4 (in full, from §1–§8 of this Design of record), the §5
       Ctrl+] notice row, and the §7 card line;
     - remove the "(Shipped in …)" phase notes about the old run.
   - `docs/task-dependencies.md`: the human met line (near line 1496), the §12.5 glyph
     list, §12.13, and the §12.25 depth-run, `copy.glyph`, scope, receipt, and
     JSON-example lines.
   - `docs/capture.md`: the requirement summary line.
   - `docs/task-split.md`: the run sentence and examples.

   Leave `docs/task-tag-marks.md` to `obsidian-marks`. Then run `just docs-gen` and
   `just docs-check`.

6. **Plugin twins** (bob-plugins, in the same phase so the twin-copy tests, which read
   bob-cli's canonical fixtures, never go red):
   - `taskReqRun` → `taskReqGlyph(state)` in the cycler owner twin
     `195-task-reqs-twin.js`, byte-copied to nav `695` and Block ID Prompt `166`. Update
     the three export fragments.
   - The req-close twins `198a` → nav `699`: human met line, glyph token, scope, row
     `run`, and `REQ_CLOSE_COPY.TEACHING`.
   - Re-copy RV `README.md` and `vectors.json` into the cycler and nav fixtures.
   - Change the Ctrl+] notice in `199-req-toggle.js` to `◇ First requirement…`, keeping
     its drawer wording.
   - In nav `206g`, `successorRequirementGlyph` stops counting hashes and returns
     `taskReqGlyph(row state)`. Its slot styling is `obsidian-cards`' job.
   - Update every test and e2e assertion on `⌗`. Known sites: the cycler req-toggle and
     req-close-vector tests, the successor picker and verdict tests, and
     `obsidian-task-reqs-e2e.cjs`.
   - Update bob-plugins `docs/task-status-cycler.md` (the notice).
   - Run `npm run build` and `npm test`, then sync `task-status-cycler`,
     `bob-navigation-hotkeys`, and `block-id-prompt`.
7. **Gate.** Run `just check` in bob-cli. A grep for `⌗⌗` in both repos' sources,
   fixtures, and docs finds only historical or migration prose. The single `⌗` task mark
   is untouched.

### line-annotations — bob says which lines are requirements (bob-cli)

1. Add one serialized type, `req: {depth, state}`, with `state` taken from
   `EffectiveReqState::as_str()`. Put it as an optional field on `BlockLineJson`
   (`src/native/capture/block_diff.rs`), omitted on non-requirement lines, beside the
   drawer annotation that `bob-cli-ph` added. Follow its naming and doc pattern.
2. Fill it for every preview producer whose lines can be task children: task blocks,
   project-note section blocks, and note-file lines. Run the recognizer on the snapshot
   each line comes from:
   - after the write for `unchanged`, `added`, and `changed` lines;
   - before the write for `removed` lines.

   Then owner-done → met and owner-cancelled → moot follow automatically.

3. Add the same optional `req` to idle-agenda `AgendaLine`
   (`src/native/capture_pomodoros_agenda.rs`) for requirement rows. Leave `kind` and
   `text` unchanged for older clients. This changes no line the preview shows; the
   NOTES-scoop preview gap stays with `bob-cli-po`.
4. Tests:
   - dry-run JSON for a new task with requirements at depths 1–3;
   - a close that meets them;
   - dropped, carried, and excluded lines;
   - a removed requirement line;
   - a note bullet and a drawer entry, which carry no `req`;
   - an agenda block with requirements.
5. Docs:
   - `docs/reference/json-outputs.md` (block lines and agenda lines);
   - the preview section of `docs/capture.md`;
   - the idle-agenda contract page that documents `AgendaLine`.

   Then run `just docs-gen`, `just docs-check`, and `just check`.

### obsidian-marks — the diamond in notes (bob-plugins bob-ledger-tools)

0. Run the sequencing check. Read `docs/task-tag-marks.md` and the `bob-cli-ph`
   drawer-glyph CSS that landed beside the requirement section.
1. **CSS.** Rewrite the requirement section of `plugins/bob-ledger-tools/styles.css` to
   §2–§4 and §6: the four masks, the 1em slot, `::before` glyph, `::after` suppression,
   states, flagged, owner-ink parity, neutralized tints, `[x]` normalization, hover,
   reduced motion, forced colors, print, and the shared `.bob-req-mark`.
2. **JS.** In `src/264-plugin-task-reqs.js` and `src/139-task-reqs.js`, or a new
   fragment if a cap would bind:
   - the §5 tooltip and label strings;
   - labels moved onto the input, with every `title` removed;
   - the capture-phase click guard (one listener registered through the plugin's DOM
     event registry, for pointer, mouse, and touch down and click), with the §5 notices;
   - the reading-view toggle-off cleanup;
   - `TASK_REQ_TEACHING_COPY` set to the §5 teaching line.
3. **Unit tests** (`test-ledger-tools-task-tag-marks.cjs` and any new file):
   - CSS assertions: four masks, no depth map, `::before` mask, `::after` suppressed,
     widened tint reset, forced colors;
   - attributes and labels on the input, with no `title` anywhere;
   - flagged rows;
   - a guarded click that writes nothing and shows the state's notice;
   - toggle-off cleanup.
4. **Real Obsidian.**
   - Extend `scripts/obsidian-task-reqs-e2e.cjs` with a look group that probes computed
     styles:
     - depth-1 text x-offset equals the pre-change baseline;
     - requirement text at depths 1–4 sits at one offset from its own bullet;
     - a `[?]` row has a transparent background;
     - hand `[x]` and derived met share decoration and color;
     - clicking a diamond leaves the file bytes unchanged and shows the notice;
     - cursor reveal and the toggle still work.
   - Following `obsidian-copy-chips-look-e2e.cjs`, take light and dark screenshots of a
     state-sheet note. It should show every state under open, done, and cancelled
     owners, a nested branch with notes, the drawer that holds them, Task Links, and a
     carried stub. Store them with `sase artifact create`.
   - Do a taste pass beside the drawer tray and task-link rings. If the only adjustment
     needed is the 0.92em → 1em knob, make it and record why.
5. **Docs.** Update bob-plugins `docs/bob-ledger-tools.md`, and bob-cli
   `docs/task-tag-marks.md` (the requirement example line and "Requirement marks"
   section). Then run `npm run build` and `npm test`, and sync `bob-ledger-tools`.

### obsidian-cards — the diamond on successor cards (bob-plugins Navigation Hotkeys)

1. In `206g-successor-picker-requirements.js`:
   - rows render `span.bob-req-mark` with the staged-outcome `data-state` (§7) and text
     `taskReqGlyph(state)`;
   - the group heading gets a pending mark;
   - the owner ink, when present, sets `--bob-req-ink`.

   In `206-successor-picker-modal.js`, requirements-only cards use the Lucide `diamond`
   icon.

2. Clean up `plugins/bob-navigation-hotkeys/styles.css`: remove the monospace run-slot
   rules. Keep depth indentation, staged color cues, and conflict rings.
3. Tests:
   - `test-navigation-successor-picker.cjs` and `-verdicts.cjs`: mark state flips with
     each staged verdict, the heading mark, and the icon;
   - with ledger-tools absent, the Unicode fallback stays visible.

   In `obsidian-req-close-card-e2e.cjs`, assert the `data-state` flips. Store light and
   dark screenshots of a card with Met, Drop, and Own task rows as artifacts.

4. Update the bob-plugins docs for the successor card. Then run `npm run build` and
   `npm test`, and sync `bob-navigation-hotkeys`.

### mac-editor — the teal chip (bob-mac-capture)

0. Run the sequencing check. Read the `bob-cli-ph` Mac drawer painting that landed in
   `paintHighlighting`.
1. **Delete:**
   - `RequirementMarkerGlyphOverlay` and its use in `AutosizingCaptureEditor`;
   - `requirementMarkerSpans()` and `concealedRequirementMarkerSpans`;
   - the concealment branch in `paintHighlighting`;
   - `lastConcealedRequirementMarkerStarts` and
     `refreshRequirementMarkerConcealmentIfNeeded`;
   - the req-only `selectionUTF8` plumbing, restoring "selection callbacks never write
     `attributedDraft`";
   - any helper left unused.

   Keep `CaptureMonospaceGlyphOrigin.wrappingOrigin`, which the route-emoji overlay
   uses.

2. **Paint** the chip per §8. Add `req_marker` to `captureSemanticCategory`. Make sure
   the picker-marker highlight clear and restore paths cannot strip the requirement
   wash; if they can, restore it the way they restore the project-note wash.
3. **Tests.** In `CaptureEditorHighlightTests`, replace the four concealment tests with:
   - the wash covers exactly `[ ]` (3 bytes) with a teal foreground;
   - the Increase Contrast level;
   - no clear-foreground runs;
   - identical paint with the caret on and off the span;
   - Backspace still removes the whole marker (new: no test exists);
   - `- [` + space still expands.
4. **Docs.** Rewrite the requirement-marker paragraph of `docs/capture-panel.md` for the
   chip.
5. **Gates and follow-up.** Run the Mac gates, push, and watch exact-SHA CI to green.
   Record a `PROPOSED FOLLOW-UP:` that `bob-cli-pp`'s requirement half is gone with the
   overlay, while the route-emoji half remains.

### mac-diamonds — the diamond in previews, agenda, and Pull-In (bob-mac-capture)

1. Add `RequirementMark(state:)`, the §2 shapes, teal, with §4 opacities. Decode the
   state from bob's strings; unknown strings become pending. It is the app's first
   custom `Shape`, and it lives beside the existing marks.
2. Decode the additive `req` on `CapturePomodoroBlockLine` (`CaptureBlockLine`) and
   `CaptureAgendaLine` with `decodeIfPresent`, lossily, like `reason` and `origin`.
3. **Preview.** In `BlockDiffCard`, rows whose line has `req` draw the mark in place of
   the `.checkbox` token. Keep the `tokens.joined() == content` invariant, and keep
   existing change coloring.
4. **Agenda.** Child rows with `req` draw the mark and drop the leading `[c] ` from the
   shown text. Swift only does this when bob said the line is a requirement.
5. **Pull-In card.**
   - Rows draw the staged-outcome mark in a fixed ~1em slot. Delete
     `CapturePullInChoiceSection.depthGlyph`.
   - The section header and filter capsule draw a pending mark for requirement groups:
     `group == "requirements"`, or `copy.glyph == "requirement"`.
   - The default verdict capsule says **Met with task**.
6. **Fixtures.** Regenerate the req-close fixtures with
   `Scripts/generate-req-close-fixtures.sh`, using a `bob` built from bob-cli master
   (with the glyph token, the scope wording, and `req` annotations). Byte-copy the RV
   fixture into `Tests/Fixtures/req_close/`, and update the provenance notes.
7. **Tests.**
   - Replace `testDepthGlyphsFollowMarkerRule` with mark-by-state and staged-flip tests.
   - Decoding: `req` present, absent, and malformed.
   - `BlockDiffRow` and agenda row presentation.
8. **Docs.** Update `docs/capture-previews.md` (preview and Pull-In rows),
   `docs/protocol.md` (the `req` annotation), and the agenda section of
   `docs/capture-panel.md`. Then run the gates and watch exact-SHA CI to green.

### memory — records (bob-cli memory)

Use `/sase_memory_write`, and apply only the accepted decisions below. Then run
`sase memory init`, `just docs-gen` (which regenerates `docs/reference/glossary.md`),
and `just docs-check`. If every memory decision was declined, close the phase without
changes.

> [!decision] memory_glossary_req In `glossary:task-requirement`, replace the rendering
> paragraph and the teaching copy. New text: Obsidian and Bob Mac Capture draw one
> constant requirement mark in the owner's ink (teal on the Mac). It is a hollow `◇`
> while pending, a filled `◆` when met, faded when moot or excluded, slashed when
> dropped, and `→` when carried. Indentation alone shows depth. The Mac draft keeps the
> real `[ ]` bytes as a teal chip. The mark never ticks. Teaching copy: _Met when this
> task closes. Drop it or make it a task if the scope changes._ Keep the strand's other
> paragraphs unchanged.

> [!decision] memory_req_mark_record Add
> `decisions:requirements-wear-the-criterion-diamond`, in the web's standard record
> shape:
>
> - **Applies to:** bob-cli, bob-plugins, and Bob Mac Capture.
> - **Claim:** one constant mark; state by fill; indentation is the only depth cue; the
>   Mac draft never hides bytes; marks come from bob state, never from `[ ]` bytes.
> - **Why:** the Context and Design calls of this plan.
> - **Rejected:** depth-counted runs, a constant `⌗⌗`, a `req` text label, rings, ghost
>   checkboxes, `↳`, pills, rails, and progress fractions.
> - **Cost:** a second look change days after the first; one more shape to learn; CLI
>   met lines use `✓`, not `◆`.
> - **Reopens when:** Bryan finds the diamond too quiet after the 1em knob, or wants
>   per-requirement ticks (a model change, not a style change).
> - **Evidence:** the research ref, this plan, and the phase commits.
>
> Link it to [[decisions:task-reqs-are-met-with-their-task]]. That record does not
> describe rendering, so it is not marked superseded.

## Acceptance evidence

The land agent verifies and cites each item:

- **No vault changes.** No vault line changes, and a live `bob task tidy` dry run
  reports nothing to tidy. The RQ, RL, RW, and SP vector files are byte-identical to the
  base. The RV diff touches only §9 strings.
- **Obsidian (screenshots and e2e).**
  - Depth-1 text does not move.
  - Depths 1–4 align to their own bullets.
  - Every state is identifiable in a grayscale copy of the screenshots.
  - Hand `[x]` and derived met look the same.
  - `[?]` carries no lane tint and wears the repair orange.
  - The cursor reveal and the tag-marks toggle work.
  - A click on a diamond writes nothing and teaches.
  - Cards render readably with bob-ledger-tools disabled.
- **Mac.**
  - The draft never hides a byte, and the caret is always visible.
  - The diamond shows in the preview, agenda, and Pull-In card from bob's `req` and
    state, and a pre-annotation `bob` still decodes.
  - Exact-SHA macOS CI is green, or the Mac targets are reported unverified.
- **CLI.** `bob capture`, receipts, `bob task split`, and the human card print `◇` / `✓`
  in a plain terminal. A repo-wide `⌗⌗` grep finds only migration or history prose.
- **Accessibility.** VoiceOver and Obsidian's labels say "requirement" with its state,
  and nothing invites a tick.
