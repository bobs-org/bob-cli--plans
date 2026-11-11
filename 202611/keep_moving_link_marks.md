---
tier: epic
title:
  Keep-moving marks — show `⏭ N` after a Task Link whose close will keep its project
  moving
goal:
  "In today's ledger, a quiet `⏭ N` mark after a Task Link forecasts the `Keep ‹N›
  moving` successor picker that closing that task will really open: N is exactly the
  number of rows that group will offer, it appears only on the link whose close triggers
  the group, and it is silent whenever that cannot be proven. Keep-moving first stops
  skipping standalone top-level projects (only umbrella projects are skipped), so the
  screenshot's `job_email_recruiters` link shows `⏭ 4` and Ctrl+Enter then offers those
  same four tasks."
phases:
  - id: umbrella-rule
    title: Umbrella rule in the contract, memory, and Rust
    depends_on: []
    size: medium
    description:
      "umbrella-rule: in bob-cli, redefine leaf projects as every recognised project
      except umbrellas (root projects whose ^prj carries a Sub-projects line); update
      §12.15–§12.17, projects/pomodoro-homes/json-outputs docs, the accepted memory
      decisions, and Rust is_leaf_project plus its LP/KM tests."
  - id: umbrella-cycler
    title: Umbrella rule and `_generated/` fix in the Obsidian cycler
    depends_on:
      - umbrella-rule
    size: small
    description:
      "umbrella-cycler: in the bob-plugins task-status-cycler, apply the umbrella rule
      in successorLeafProject using the planner's held note text, stop generated tag
      pages from making parent stems ambiguous, update LP/KM tests and docs, build, and
      sync."
  - id: mark-contract
    title: Keep-moving mark display contract
    depends_on:
      - umbrella-rule
    size: small
    description:
      "mark-contract: in bob-cli docs, add docs/keep-moving-marks.md (the authoritative
      display contract with the KB/KT/KP vectors from this plan), register it,
      cross-link it from frees-marks and §12.15, and move the off-clock chip contract to
      side 4."
  - id: mark-core
    title: Pure keep-moving mark core and cross-plugin parity
    depends_on:
      - umbrella-cycler
      - mark-contract
    size: medium
    description:
      "mark-core: in bob-ledger-tools, add the pure, synchronous gates, leaf predicate,
      offer count, tooltip, DOM builder, and widget class, plus config readers, the
      vector tests, and an identity-set parity test against the cycler's real successor
      planner."
  - id: mark-surface
    title: Live Preview, Reading view, look, and liveness
    depends_on:
      - mark-core
    size: medium
    description:
      "mark-surface: in bob-ledger-tools, render the mark in Live Preview and Reading
      view, with the shared source-text cache, refresh triggers, baton-pass tick,
      toggle, CSS, off-clock chip at side 4, docs, a real-Obsidian e2e, build, and sync."
  - id: audit
    title: Real-vault audit against capture and the picker
    depends_on:
      - mark-surface
    size: small
    description:
      "audit: on a scratch vault only, compare every rendered mark with `bob capture
      --dry-run` keep_moving rows and with the Obsidian picker. Confirm the screenshot
      case shows `⏭ 4` and record the results in the contract's Live verification
      section."
decisions:
  memory_decision_record:
    ask: Add a decisions:keep-moving-skips-only-umbrellas record for the new leaf rule?
    memory:
      - decisions
    default: true
    requested:
      I agree with all of the requirements recommended in that research file (but keep
      in mind that you are responsible for planning and splitting up this work
      appropriately).
    answer: false
  memory_glossary_leaf:
    ask:
      Rewrite the Leaf Project glossary strand so a leaf is any project except an
      umbrella?
    memory:
      - glossary:leaf-project
    default: true
    corrects: are roots, never leaves.
    answer: true
proposed_by: bbugyi200.athena.1b8
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-11 07:57:56
status: wip
---

- **PROMPT:**
  [prompts/202611/keep_moving_link_marks.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/keep_moving_link_marks.md)

# Keep-moving marks: show `⏭ N` where closing a Task Link will really offer N

## Context

Bob already shows a quiet `🔑 N` after a ledger Task Link whose task is the last blocker
of N tasks (`docs/frees-marks.md`). Bryan wants a sibling mark that tells him, before he
presses Ctrl+Enter, how many tasks the successor offer will show from that task's
project. He wants it only on links into leaf projects.

His example is `~/tmp/screenshots/20261111_063929.png`, showing
`~/bob/2026/20261111.md`. There, `[[job_email_recruiters#^bryan-stephen-luzern-risk]]`
(under `() — WRITE`) should show `4`, because `job_email_recruiters.md` holds four other
Next tasks (`^ecogpt`, `^ray-lyles`, `^product-engineer-dan-dowd`,
`^senior-rust-java-engineer`) with no Task Link today. No other link on that screenshot
should get the mark.

Bryan asked us to review the research report
`research:202611/keep_moving_link_marks/keep_moving_link_marks.md` (read it with
`sase artifact read`). He agrees with **all** of its recommended requirements, and he
asked for a design that is intuitive, reliable, and beautiful. This plan adopts those
requirements, resolves the report's open questions the way it recommends, and fixes the
details it left loose (exact LP answers, post-close counting, an input-agreement guard,
the reuse of the existing source-text cache, and the true free fragment numbers).

### Facts verified for this plan

- **Today the example cannot work.** `job_email_recruiters.md` has `parent: "[[job]]"`,
  and `job.md` is an area. That makes it a top-level project, and today's leaf rule
  (`docs/task-dependencies.md` §12.15, §12.17 LP2) says a top-level project is a root
  and never a leaf. So closing the link offers **nothing**. Rust
  (`src/native/task_complete/keep_moving.rs` `is_leaf_project`, which checks
  `home.root_parent != home.key` from `HomeResolver::resolve_note`) and the cycler
  (`plugins/task-status-cycler/src/089c-successor-keep-moving.js`
  `successorLeafProject`) agree. A mark reading `4` there would promise a picker that
  never opens.
- **Vault census.** The only root projects with a `🧩 **Sub-projects:**` line are `bob`,
  `sase`, `sase_blog` (`root: true`), and `job_2026`. No root's line lists only `❌`
  children. Every strict leaf in the vault has 0 offerable tasks today.
- **Sub-projects marker parsing already exists.** In Rust,
  `src/native/projects/surface.rs` `project_surface` already reads R's first marker line
  through `subproject_displays_in_marker_line` (`SUBPROJECTS_MARKER_PREFIX` lives in
  `src/native/projects/mod.rs`). In the cycler, `089-successor-project.js` has
  `PROJECT_SUBPROJECTS_MARKER` and `projectMarkerHasOpenChild`.
- **The `_generated/` bug.** The cycler's `successorKeepMovingResolveParent` treats a
  parent stem as ambiguous when `vault.getMarkdownFiles()` has two files with that stem.
  `_generated/tag_pages/topic/sase.md` therefore makes every direct sub-project of
  `sase` a non-leaf. Rust's note index excludes `_generated`. The cycler's own
  `084-project-parent.js` already defines `PROJECT_PARENT_EXCLUDED_DIRECTORIES` and
  `projectParentPathIsIndexed`.
- **The planner the mark must match.** `planSuccessorKeepMoving` (089c) runs inside the
  successor planner in `085-successor-plan.js`, which dedups
  `hand_off > relation > project > project_finished > keep_moving` in
  `089b-successor-verdicts.js`. `leafProject` is injected as `(path) => ...`, and the
  planner already holds N's text (`projectNoteTextFor(noteTexts, path)`).
- **Ledger-tools precedents to copy.** 🔑 lives in
  `plugins/bob-ledger-tools/src/141b-frees-marks.js` (pure core) and
  `271b-plugin-frees-marks.js` (mixin). Its tests are
  `scripts/test-ledger-tools-frees-marks.cjs` and its e2e is
  `scripts/obsidian-frees-marks-e2e{,-lib}.cjs`. The widget sides are: 🔓 and the mail
  reply badge `side: 1`, 🔑 `side: 2`, and the off-clock chip `side: 3`
  (`271c-plugin-off-clock-link-chips.js`; Reading view in `271d-…-reading.js`). The free
  fragment names are `141d`/`141e` and `271g`/`271h`. Mixins are installed in
  `src/310-install-methods.js`.
- **A shared, async source-text cache already exists.** It lives in
  `279-plugin-task-theme-api.js`: `taskThemeSourceTextFor(path)` prefers open-editor
  text, `taskThemeRequestSourceText(path)` fills through `vault.cachedRead` and then
  fans out refreshes, and the cache is an LRU of 32. `271c` already replaces a cached
  entry on `metadataCache` `changed`. The mark reuses this cache and adds no new one.
- **Config.** The cycler reads `plan.keep_moving` and `plan.link_unblocked`
  (`087-successor-place.js` `loadKeepMoving` / `loadLinkUnblocked`). Both are
  stat-cached and default to `true`. Ledger-tools reads `plan:` in
  `320-plan-lane-and-config.js` `loadPlanCaps`. Bryan's config sets neither key, but it
  does configure the `relations.successor_labels` entry `starts after`.
- **Mask glyph precedent.** `styles.css` already ships the `--bob-off-clock-icon` mask.
  The vault's `task-statuses.css` sets
  `--task-status-next: var(--color-green, #3f8f5f)`.
- Bryan's vault task `bob.md ^leaf-task-link-icon` tracks this feature. Never edit the
  vault; that task is his to close.

## Shared design (binding for every phase)

### 1. Scope: keep-moving skips only umbrella projects

```text
ROOT(N)      ⇔  R2 root parent of N is N itself        (pomodoro-homes.md R2; Rust resolve_note)
UMBRELLA(N)  ⇔  ROOT(N)  ∧  N's open ^prj R has a direct-child `🧩 **Sub-projects:**`
                marker line (the one sync maintains) listing ≥ 1 wikilink child,
                in any state: open, ~~✅~~, or ~~❌~~
LEAF(N)      ⇔  N is a recognised project (§12.10: outside done/, non-terminal, exactly
                one open ^prj, no scan issue; never an area, daily note, or inbox file)
                ∧  ¬UMBRELLA(N)
```

- **What changes.** A standalone top-level project, a `root: true` project with no
  sub-projects, an orphan project, and a project whose parent cannot be resolved all
  become leaves. A root that has ever split out children stays an umbrella, even when
  every child is closed. That keeps `bob` quiet for good, and it preserves the original
  reason roots were excluded: "broad containers whose daily links routinely run out, so
  a prompt there would be noise". Middle projects with children (LP9) stay leaves, as
  today.
- **Source of truth.** Read the marker line from N's held text, never from child
  frontmatter. Both engines already hold N's text when they classify, so the check costs
  no I/O.
- **Cheap order.** If N has no marker line, N is a leaf without any parent walk. The R2
  walk runs only when the line exists, to tell a middle project (leaf) from a root
  umbrella.
- **`root: true`** no longer decides offer behaviour on its own. It is a root, so it is
  an umbrella only when it has the marker.
- **Generated and excluded directories never make a parent stem ambiguous.** This uses
  `PROJECT_PARENT_EXCLUDED_DIRECTORIES` in JS, matching Rust's index.
- **Name.** "Leaf Project" is redefined in place. Its only consumer is keep-moving. A
  leaf is now "a project that is not an umbrella", which also makes Bryan's own phrase
  ("split out job email tasks into a leaf project") true. Capture JSON stays additive,
  because more groups fire and no shape changes. No commit needs `!`.

### 2. Which links get the mark (gates, checked in this order; any failure ⇒ no mark)

```text
X  = the open #task a live ledger link resolves to (🔑's unique-target resolution,
     freesMarkLinkedTask; NO [id::] requirement)
N  = X's note     R = N's open ^prj     C = {X}
G0 toggle on; plan.link_unblocked ∧ plan.keep_moving; Tasks memo warm; N's text cached
G1 the link passes 🔑's link gates (docs/frees-marks.md "Which links get the mark"):
   today's canonical daily note; a dedicated Task Link row directly under an OPEN
   Pomodoro entry (running or `()` placeholder); not struck, embedded, prose, nested,
   fenced, Events, or under a closed entry
G2 LEAF(N) (§1)                                                              (K1, K2)
G3 X ≠ R                                                                     (K3)
G4 no live link anywhere in today's Pomodoros targets an OPEN task of N other than X;
   the ^prj R counts as N work; links to closed tasks and to N's sub-projects
   don't count. Live = §12's term: unstruck plain, embed `![[…]]`, or
   `#`-deferred Task Link sub-bullet under an open entry (embeds block G4
   even though they never render a mark)                                      (K4)
G5 R is not Blocked: not `[?]`, no own open prerequisite, no future project
   schedule, no #prj_is_blocked                                               (K5)
G6 X's own subtree in N holds no task embed (`![[…#^…]]`): embedded completion
   would widen C, so stay neutral
G7 agreement: N's cached text and the Tasks memo agree that X and every counted
   resident is open (matched by path and line, with block id when present; lines the
   memo does not index, such as #ref without #task, are skipped). Disagreement means
   "inputs are mid-update" ⇒ unknown ⇒ no mark until the next refresh
G8 OFFER ≥ 1 (an EMPTY(N) after the close is project_finished's, K6; a project
   group for N is impossible here because X has no recovered ^prj, K7)
```

The mark sits on **every** live dedicated link row to X (the same X linked twice shows
it twice; G4 treats X as one identity). Because of G4, the mark appears only on the
**last planned** link of its project. When a sibling closes, the survivor gains the mark
(the "baton pass").

### 3. The count (post-close simulation; parity is the backbone)

The mark computes what the cycler will compute _after_ a managed Done of X. It does not
use a pre-close approximation.

```text
RESIDENTS = open task lines parsed from N's text (#task lines plus #ref lines without
            #task, exactly as successorKeepMovingResidents does), excluding R and X;
            residents of N's sub-projects never count
OPEN_IDS  = ids of open tasks in the Tasks memo, minus X's [id::] (X is closing)
for t in RESIDENTS (status ∈ {'?', ' ', '*', '/'}; other marks ignored):
  blocked   ⇔ '?' ∨ #prj_is_blocked ∨ some dependsOn id ∈ OPEN_IDS
  scheduled ⇔ ¬blocked ∧ a single scheduled date after today
  hidden    ⇔ ¬blocked ∧ ¬scheduled ∧ whole-token #hide
  → FOLDED[blocked|scheduled|hidden] += 1, or else
  offered   ⇔ status ∈ {'/', '*', ' '} ∧ no live link today
              ∧ t is not a hand-off of X (t names X's id in [dependsOn::] and closing X
                frees it — 🔑 counts those)
              ∧ t is not a relation successor of X (an accepted LINKS entry on t
                resolving to X whose relation_label_key ∈ relations.successor_labels)
ROWS      = offered, ordered Pending → Next → Ready, then line in N
N_mark    = |ROWS|;  mark ⇔ G0–G8 ∧ N_mark ≥ 1
```

**Parity invariant.** On every fixture, the identity set of `ROWS` (path plus block id,
else path plus line) equals the selectable `keep_moving` rows of the cycler's full
successor planner (`085-successor-plan.js`, after verdict dedup) for a managed Done of X
on the same day text, note texts, and dependency index. The tooltip's fold line equals
that group's fold counts. A count of 0 holds exactly when the mark is absent. The cycler
is the reference: if a vector exposes a cycler disagreement with §12.15, fix the cycler
in the same phase and record it.

"No mark" is **not** a guarantee that no picker opens. For example, one `x` that closes
a Pomodoro holding two sibling links still opens a group. A shown mark is a promise.

### 4. Look (beautiful, quiet, 🔑's twin)

- **Host:**
  `span.bob-keep-moving-mark[role=img][aria-label][data-tooltip-position=top][data-count][data-block-id][data-project]`.
  It holds `span.bob-keep-moving-mark-glyph` (an empty, `aria-hidden` mask) followed by
  the count text. It has no `title` attribute and is listener-free, so it survives
  Reading-view re-renders. A DOM builder takes a fake `doc`.
- **Glyph:** a _filled_ skip-forward silhouette (triangle plus bar). It is the shape of
  the picker's `skip-forward` and the Mac's `forward.end.fill`, drawn as a CSS mask so
  it takes theme colour and never becomes a blue emoji tile. It is tinted with the
  **Next** status token, because picks land in Next. No new hex appears except the
  token's own fallback.
- **Count:** muted at rest, exactly like 🔑. The glyph carries the colour and the number
  stays quiet. There is no border and no filled capsule at rest. The count is exact,
  with no `9+` cap, and never `⏭ 0`.
- **CSS:** add this to `plugins/bob-ledger-tools/styles.css` right after the
  `.bob-frees-mark` block. Define the glyph variable where `--bob-off-clock-icon` is
  defined. Tune the path's optical centre in the e2e screenshot review if needed.

```css
/* beside --bob-off-clock-icon */
--bob-keep-moving-glyph: url("data:image/svg+xml,%3Csvg xmlns='http://www.w3.org/2000/svg' viewBox='0 0 16 16'%3E%3Cpath d='M3.5 2.75 11 8l-7.5 5.25z'/%3E%3Crect x='11.75' y='2.75' width='1.9' height='10.5' rx='.6'/%3E%3C/svg%3E");

.bob-keep-moving-mark {
  --bob-keep-moving-mark-ink: var(--task-status-next, var(--color-green, #3f8f5f));
  display: inline-flex;
  align-items: baseline;
  gap: 0.22em;
  margin-inline-start: 0.4em;
  padding: 0 0.42em;
  border-radius: 999px;
  font-family: var(--font-interface);
  font-size: 0.8em;
  font-variant-numeric: tabular-nums;
  line-height: 1;
  white-space: nowrap;
  vertical-align: baseline;
  color: var(--bob-keep-moving-mark-color, var(--text-muted));
  user-select: none;
  cursor: default;
  print-color-adjust: exact;
  transition:
    background-color 120ms ease,
    color 120ms ease;
}
.bob-keep-moving-mark-glyph {
  display: inline-block;
  inline-size: 0.95em;
  block-size: 0.95em;
  align-self: center;
  background-color: var(--bob-keep-moving-mark-ink);
  -webkit-mask: var(--bob-keep-moving-glyph) center / contain no-repeat;
  mask: var(--bob-keep-moving-glyph) center / contain no-repeat;
}
.bob-keep-moving-mark:hover {
  color: var(--text-normal);
  background-color: color-mix(
    in srgb,
    var(--bob-keep-moving-mark-ink) 12%,
    transparent
  );
}
.bob-keep-moving-mark.bob-keep-moving-mark-tick {
  animation: bob-keep-moving-mark-tick 180ms cubic-bezier(0.2, 0.9, 0.3, 1.25);
}
@keyframes bob-keep-moving-mark-tick {
  from {
    opacity: 0.3;
    scale: 0.6;
  }
  to {
    opacity: 1;
    scale: 1;
  }
}
@media (prefers-reduced-motion: reduce) {
  .bob-keep-moving-mark,
  .bob-keep-moving-mark.bob-keep-moving-mark-tick {
    animation: none;
    transition: none;
  }
}
@media (forced-colors: active) {
  .bob-keep-moving-mark {
    color: CanvasText;
  }
  .bob-keep-moving-mark-glyph {
    background-color: CanvasText;
  }
}
body:not(.bob-keep-moving-marks) .bob-keep-moving-mark {
  display: none;
}
```

A link with every mark reads, left to right, as past → dependents → project → clock.
That is the same order the picker lists its groups (hand-offs before keep-moving):

```text
	- ◐ bob_git#^wire-hooks 🔓 🔑 1 ⏭ 3 ⦸ off
```

### 5. Placement

- **Live Preview:** a zero-width `Decoration.widget` at the link token's end (after
  `]]`, before any trailing ` #`), with **`side: 3`**. The off-clock chip moves from
  `side: 3` to **`side: 4`**, a one-constant change. Order is decided by `side`, never
  by extension precedence. The widget covers no text, so it stays put while the caret
  reveals raw `[[…]]`.
- **Reading view:** insert the host right after the last of the link's trailing
  `span.bob-unblocked-glyph` / mail reply badge / `span.bob-frees-mark` siblings. With
  none of those, insert right after `a.internal-link`, or after its sole
  `del`/`s`/`strike` wrapper. Always insert **before** any `.bob-off-clock-link-chip`.
  The off-clock chip's own Reading insertion must also skip over a
  `.bob-keep-moving-mark` so it stays last.

### 6. Tooltip (`aria-label`, Obsidian tooltip; never contains `::`, `🔓`, or "unblocked")

```text
Closing this offers 4 tasks to keep job_email_recruiters moving
It's the last job_email_recruiters task planned today
• ‹title› · Next                         (up to 3 rows, ROWS order)
• ‹title› · Next
• ‹title› · Next
+1 more                                  (when N_mark > 3)
ready in job_email_recruiters · not dependents
not offered  2 blocked · 1 scheduled later · 1 hidden   (only non-zero parts; line omitted when all zero)
```

- Use the singular `offers 1 task` for one row. `‹N›` is N's note stem, as in §12.15.
- Titles are cleaned with `dependencyCleanTaskText`, then cut at 48 characters (47 plus
  `…`), following the §12.6 copy rule. Rows end with the lane name (`Pending`, `Next`,
  or `Ready`), never a raw status symbol.
- Any stray `::` becomes `:`.

### 7. Interaction, motion, toggle, liveness, performance

- **Plain click (mousedown):** place the caret at the widget position and focus the
  editor. It never navigates and never writes, matching ◐, 🔓, and 🔑.
- **Mod+click (primary button, Live Preview):** opens **N** in a new tab
  (`workspace.openLinkText(N's linktext, sourcePath, "tab")`). The rows ⏭ counts live
  in N, while 🔑's Mod+click opens X. The Reading-view host stays listener-free like
  🔑's.
- There is no tab stop and no live region.
- **Motion (baton pass):** each editor view keeps a bounded LRU (about 2,000 keys) of
  `X path\0blockId → last N_mark`, with "eligible but no mark" stored as 0. When a key
  already in the map changes to a new N ≥ 1, the host gets `bob-keep-moving-mark-tick`
  for about 400 ms. There is no tick on first paint, scroll, a viewport-only rebuild, or
  Reading view, and none under reduced motion. A count that drops to 0 simply goes away.
- **Toggle:** `toggle-keep-moving-marks` ("Toggle Task Link keep-moving marks"). It is a
  session toggle, default on, using `applyMarkGateClass("bob-keep-moving-marks", on)`
  plus `registerMarkGatePopoutSync`, with a Notice "Keep-moving marks on/off". v1 has no
  api namespace.
- **Liveness:** one coalesced `scheduleKeepMovingMarksRefresh()` (about 120 ms) runs on:
  - the `scheduleLiveWidgetRefresh()` fan-out (the Tasks `cache-update`);
  - `metadataCache` `changed` and `resolved` for today's daily note or any project note
    behind today's live links;
  - vault `rename` / `delete`;
  - a source-text cache fill (add it to the fan-out in `taskThemeRequestSourceText`);
  - the local-midnight rollover;
  - `layout-change`;
  - the toggle.

  Live Preview also rebuilds on `docChanged`, `viewportChanged`, the Live Preview ↔
  source switch, and a file switch, but never on selection alone. Out-of-order async
  results are discarded. Unknown and known-zero stay distinct internally and both render
  as nothing. A stale positive is never left visible after invalidation.

- **Performance:**
  - Parse the live-link set once from the **whole** day, because a link below the fold
    can fail G4.
  - Group by N, and evaluate G2–G8 and the count once per N per build.
  - Build widgets only inside visible ranges.
  - Never read from disk during render.
  - Request text only for project notes that pass the cheap frontmatter gates and sit
    behind an open X.
  - Raise the shared source-text LRU from 32 to 64 so marks, theme marks, home chips,
    and event pills don't thrash it.
  - A node perf test (200 links / 2,000 tasks, and 50 links / 3,000 tasks over 10
    project notes) keeps one build under about 25 ms.

### 8. Architecture: port the rule into ledger-tools, prove parity, don't call the cycler at render

This is the 🔑 precedent.

- The core is pure, synchronous, and never throws. Its I/O comes through injected
  accessors:
  - `noteTextFor(path)`;
  - `frontmatterFor(path)`;
  - link resolution;
  - the markdown file list (used for ambiguity);
  - config;
  - relation edges.

  Live Preview, Reading view, tests, and the parity harness all call the same functions.

- **Leaf predicate:** port the cycler's _fixed_ `successorLeafProject` (R2 frontmatter
  walk via `metadataCache`, with the `_generated/`-safe ambiguity rule), plus the
  umbrella marker check on N's text.
- **Residents:** parse them from N's text, so one snapshot decides recognition, the
  umbrella check, and residents. The Tasks memo supplies X's resolution, the vault-wide
  open-id set, and the G7 agreement check.
- **Live links (G4):** port the semantics of `parseSuccessorLedgerEntries` and
  `successorLinkMatchesIdentity`.
- **Config:** read `plan.keep_moving`, `plan.link_unblocked`, and
  `relations.successor_labels` stat-cached beside `loadPlanCaps`. Use the cycler's
  defaults (`true`), its invalid-value fallbacks, and its `relation_label_key`
  normalisation.
- **Relation successors:** read them through ledger-tools' own relation index. When
  successor labels are configured and the index is not ready, the state is unknown.
- **Rejected alternatives:**
  - A cycler preview API called at render. It adds new ledger-tools → cycler coupling
    and deploy ordering, and it needs a simulated close through the whole pipeline.
  - A Rust JSON count. That serves only the Mac and is deferred.

  If parity drift becomes a recurring cost, revisit a capability-detected cycler API.

### 9. Deferred (record in the contract with triggers)

- **Mac Capture idle agenda, Task Card, and nav pickers.** Trigger: Bryan asks. This
  needs a Rust/JSON count, per `decisions:mac-capture-is-a-thin-client`.
- **A dimmed "queued" variant on non-last sibling links.** It needs distinct copy and
  must not reuse the promise glyph. Trigger: Bryan asks.
- **A per-note `keep_moving: false` override.** Trigger: a non-umbrella project proves
  noisy.
- **Links in other daily notes, under closed entries, or in prose.**

## Conformance vectors

### LP — leaf vectors (both engines and the ledger port; replaces §12.17)

Each fixture states whether `^prj` carries a `🧩 **Sub-projects:**` marker line.

| #    | Note                                                                                                                               | Leaf?                                                           |
| ---- | ---------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------- |
| LP1  | `sase_goals.md` (project → `sase` project → `dev` area)                                                                            | leaf                                                            |
| LP2  | `sase.md` (project → `dev` area), marker present                                                                                   | no (umbrella)                                                   |
| LP3  | `sase_blog.md` (project, `root: true`, → `sase`), marker present                                                                   | no (umbrella)                                                   |
| LP4  | `sase_pre_blog.md` (project → `sase_blog`)                                                                                         | leaf                                                            |
| LP5  | `p.md` (project → sub-area `gtd_daily`), no marker                                                                                 | **leaf** (was no)                                               |
| LP6  | orphan project `o.md` (no `parent`), no marker                                                                                     | **leaf** (was no)                                               |
| LP7  | project whose `parent` is unresolved or ambiguous: no marker / marker present                                                      | **leaf** / no (umbrella; the walk stops, so it is its own root) |
| LP8  | `k.md` → untyped wrapper note → `sase`                                                                                             | leaf (wrappers are walked through)                              |
| LP9  | `m.md` (project → `sase`) with its own child project and marker                                                                    | leaf (not its own root)                                         |
| LP10 | task in `sase_goals.md` with `[root:: sase_blog]`                                                                                  | its note is still a leaf                                        |
| LP11 | area, daily note, inbox file, `done/` archive, terminal project                                                                    | never a leaf                                                    |
| LP12 | `parent` cycle `a ↔ b`, neither `root: true`                                                                                       | leaf (matches `resolve_note`)                                   |
| LP13 | `job_email_recruiters.md` (project → `job` area), no marker                                                                        | **leaf** (new; the screenshot)                                  |
| LP14 | `root: true` project, no marker                                                                                                    | **leaf** (new)                                                  |
| LP15 | root project whose marker lists only `~~[[x]]~~ ✅` / `~~[[y]]~~ ❌` children                                                      | no (umbrella; it split out children once)                       |
| LP16 | LP1 plus a file `_generated/tag_pages/topic/sase.md`                                                                               | leaf (excluded directories never make a stem ambiguous)         |
| LP17 | root project whose only `Sub-projects` text is not R's direct-child marker (prose elsewhere, a fenced block, another task's child) | leaf                                                            |

§12.16 changes: **KM6** becomes "N is an umbrella (LP2, LP3, LP15) → nothing". Add
**KM33**: "N is a standalone top-level project, a `root: true` project without
sub-projects, or an orphan (LP5, LP6, LP13, LP14) → a `keep_moving` group exactly as in
KM1".

### KB — which links and the count (mark)

Unless a row says otherwise, the fixture is:

- today's daily note, with `## Pomodoros` holding `- [ ] () — WRITE` /
  `\t- [[job_email_recruiters#^luzern]]`;
- N = `job_email_recruiters.md` (project → `job` area, no marker), with an open,
  unblocked `^prj` R (which may carry `#hide`);
- X = `- [*] #task Reply to Luzern ^luzern`.

| #    | Situation                                                                                                                                                               | Mark                                                                                                           |
| ---- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------- |
| KB1  | residents Pending P, Next B, Ready A; X is N's only live link                                                                                                           | `⏭ 3`; tooltip rows P, B, A                                                                                   |
| KB2  | another open resident U also has a live link (plain, `#`-deferred, or `![[…]]` embed; any open entry)                                                                   | none on X; none on U                                                                                           |
| KB3  | KB2, then U's link is struck or U closes                                                                                                                                | X gains `⏭ N` with one tick (Live Preview)                                                                    |
| KB4  | U is linked only struck, under a closed entry, in another day's note, or in prose                                                                                       | X marked; U counted when offerable                                                                             |
| KB5  | R itself is live-linked                                                                                                                                                 | none                                                                                                           |
| KB6  | N is an umbrella (LP2, LP3, LP15)                                                                                                                                       | none                                                                                                           |
| KB7  | N has 4 other Next residents and nothing else open (the screenshot)                                                                                                     | `⏭ 4`                                                                                                         |
| KB8  | N is a sub-project (LP1)                                                                                                                                                | marked as KB1                                                                                                  |
| KB9  | every other resident is Blocked, scheduled after today, or `#hide`                                                                                                      | none                                                                                                           |
| KB10 | X is N's last open task                                                                                                                                                 | none (`project_finished` owns that close)                                                                      |
| KB11 | resident D depends only on X                                                                                                                                            | D not counted (🔑 counts it); folds follow the parity invariant                                                |
| KB12 | resident S has `starts after [[X]]` in LINKS, and `starts after` is a configured successor label                                                                        | S not counted                                                                                                  |
| KB13 | R is `[?]`, has an open prerequisite, has a future project schedule, or carries `#prj_is_blocked`                                                                       | none                                                                                                           |
| KB14 | `plan.keep_moving: false`; `plan.link_unblocked: false`; toggle off; cold Tasks memo; N's text not yet cached; successor labels configured but relation index not ready | none (each)                                                                                                    |
| KB15 | X has no `[id::]`                                                                                                                                                       | marked                                                                                                         |
| KB16 | X is linked under two open entries                                                                                                                                      | both rows show the same `⏭ N`                                                                                 |
| KB17 | N's parent stem collides with `_generated/tag_pages/topic/<stem>.md`                                                                                                    | same answer as without the tag page                                                                            |
| KB18 | a resident scheduled today; a `#ref` line without `#task`; a nested subtask; a recurring task due today                                                                 | each counted                                                                                                   |
| KB19 | a resident's `[dependsOn::]` names an id that resolves to nothing                                                                                                       | counted (unresolved never blocks)                                                                              |
| KB20 | X's subtree embeds a task (`![[…#^…]]`)                                                                                                                                 | none (neutral)                                                                                                 |
| KB21 | X Done or Cancelled; or the link is struck, embedded, prose, nested, under a closed entry, in Events, fenced, or in a daily note other than today's                     | none                                                                                                           |
| KB22 | X lives in a sub-project K of N                                                                                                                                         | evaluated for K only; N's residents never count                                                                |
| KB23 | N's cached text and the memo disagree on whether X or a counted resident is open                                                                                        | none until they agree                                                                                          |
| KB24 | every row above                                                                                                                                                         | ROWS identity set == the cycler planner's `keep_moving` rows for a managed Done of X; folds equal; 0 ⇔ no mark |

### KT — tooltip

| #   | Case                                            | aria-label                                                                                                                                                                                                                          |
| --- | ----------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| KT1 | KB1 with P `Ship it`, B `Wire it`, A `Write it` | `Closing this offers 3 tasks to keep job_email_recruiters moving\nIt's the last job_email_recruiters task planned today\n• Ship it · Pending\n• Wire it · Next\n• Write it · Ready\nready in job_email_recruiters · not dependents` |
| KT2 | one row                                         | first line `Closing this offers 1 task to keep job_email_recruiters moving`                                                                                                                                                         |
| KT3 | five rows                                       | three rows, then `+2 more`                                                                                                                                                                                                          |
| KT4 | folds: 2 blocked, 0 scheduled, 1 hidden         | last line `not offered  2 blocked · 1 hidden`                                                                                                                                                                                       |
| KT5 | a 60-character title                            | first 47 characters plus `…`                                                                                                                                                                                                        |
| KT6 | a title with links, tags, fields, and `^id`     | cleaned by `dependencyCleanTaskText`                                                                                                                                                                                                |
| KT7 | a stray `::`                                    | becomes `:`                                                                                                                                                                                                                         |
| KT8 | any model                                       | contains neither `🔓` nor `unblocked`                                                                                                                                                                                               |

### KP — placement and interaction

| #   | Case                                              | Result                                                                |
| --- | ------------------------------------------------- | --------------------------------------------------------------------- |
| KP1 | Live Preview                                      | widget at the link end with `side: 3`; no line-height change          |
| KP2 | link with 🔓, 🔑, ⏭, and the off chip            | order: link, 🔓, 🔑, ⏭, off chip (`side: 4`)                         |
| KP3 | Reading view, no other marks                      | host directly after `a.internal-link` (or its sole `del`/`s` wrapper) |
| KP4 | Reading view with 🔓 and/or 🔑, plus the off chip | host after the last of 🔓/🔑; off chip still last                     |
| KP5 | caret on the row (raw `[[…]]` revealed)           | mark stays at the link end                                            |
| KP6 | toggle off, then on                               | marks vanish and return; row height unchanged                         |
| KP7 | plain click / Mod+click                           | caret placed, nothing written / N opens in a new tab                  |

## Phase `umbrella-rule` — umbrella rule in the contract, memory, and Rust (bob-cli)

Make the §1 rule the contract and implement it in Rust.

1. **Docs** (read `sase memory read docs.md` first):
   - `docs/task-dependencies.md` §12.15 Vocabulary. Rewrite the Leaf project bullet to
     the §1 definitions (ROOT, UMBRELLA, LEAF) and the "What changes" and "Source of
     truth" notes. Update the K2 line to `LEAF(N): N is not an umbrella (§1)`, and the
     §12.15 Gates list G4 to say "umbrella check on N's held text; the R2 walk only when
     the marker exists". Keep the anchor `#1215-keep-moving-offers-leaf-projects`
     unchanged.
   - Apply the §12.16 KM6 rewrite and add KM33. Replace §12.17 with the LP table above,
     including the LP16 note that Rust's index already excludes `_generated`.
   - Rewrite `docs/projects.md` "Leaf projects" (lines ~79–93). Update
     `docs/pomodoro-homes.md` lines ~80–86 (a `root: true` project is a leaf unless it
     has sub-projects). Update `docs/reference/json-outputs.md` ~204 ("A leaf project is
     a subproject with a different root parent" becomes the umbrella wording).
   - Run `just docs-gen` (it regenerates `docs/develop/decisions.md` and
     `docs/reference/glossary.md` from memory), then `just docs-check` until it passes.
2. **Memory.** Use `/sase_memory_write`, and do only what the accepted decisions allow:
   - `memory_decision_record`: create
     `sase/memory/decisions/keep-moving-skips-only-umbrellas.md`. Copy the shape of an
     existing record (frontmatter `title`, `aliases`, `origin: requested`,
     `scope: project`, `summary`, then **Applies to / Claim / Why / Cost / Reopens
     when**).
     - **Claim:** the §1 rule.
     - **Why:** the example and census; the original "broad containers" reason, kept by
       umbrellas; struck children count.
     - **Rejected alternatives:** strict leaf (marks nowhere), reparenting (a stopgap),
       every project (reopens noise for `bob` and `sase`), pure tree leaf (removes
       working behaviour from LP9 middles).
     - **Evidence:** the research artifact
       `research:202611/keep_moving_link_marks/keep_moving_link_marks.md` and this plan.
     - **Cost:** `root: true` alone no longer silences a project.
     - **Reopens when:** a non-umbrella project proves noisy, or the marker stops being
       sync-maintained.
   - `memory_glossary_leaf`: rewrite the `glossary:leaf-project` strand (body and
     `summary`) to "a recognised project note that is not an umbrella". Define an
     umbrella as a root project ([[root-parent]] of itself) whose `^prj` lists
     sub-projects. Keep the keep-moving sentence, "derived at gesture time, never
     stored", and the `[root::]` sentence. Drop the stale "names the reporting role"
     framing.
   - Then run `sase memory init` and re-run `just docs-gen`.
3. **Rust.**
   - Add `subprojects_listed: usize` to `ProjectSurface` in
     `src/native/projects/surface.rs`. It counts every display on R's first marker line,
     whatever its state, computed beside `open_subprojects`.
   - Change `is_leaf_project` in `src/native/task_complete/keep_moving.rs`. Keep the
     archive, inbox, and `project_surface` gates. If `subprojects_listed == 0`, return
     `true` without resolving. Otherwise resolve and return `false` only when
     `home.kind == Project`, `home.key == path`, and `home.root_parent == home.key`.
     Update its doc comment.
   - Update `src/native/task_complete/tests/keep_moving_tests.rs`: the LP asserts
     (LP2/LP3/LP7/LP15 with markers; LP5/LP6/LP7/LP13/LP14 leaf; LP17), KM6 (umbrella →
     nothing), and a new KM33 test that dry-runs a standalone top-level project like the
     screenshot and gets a `keep_moving` group with its four rows. Keep the
     `CountingHomeSource` read-count expectations honest (no marker ⇒ no ancestor
     reads).
   - Run the repo's standard Rust checks (fmt, clippy, tests per the justfile).
   - Do not touch `Cargo.toml`'s version or `CHANGELOG.md`
     (`decisions:bob-cli-releases-ride-a-daily-train`). The change is additive, so the
     commit gets no `!`, e.g.
     `feat(keep-moving): skip only umbrella projects, not every root`.

## Phase `umbrella-cycler` — the same rule in task-status-cycler (bob-plugins)

Open bob-plugins with `/sase_repo` and read its `AGENTS.md`. The bob-cli contract
(§12.15–§12.17 as rewritten by `umbrella-rule`) is authoritative.

- In `plugins/task-status-cycler/src/089c-successor-keep-moving.js`,
  `successorLeafProject(app, path, noteText)`:
  - Require project type, as today.
  - If `noteText` has no `🧩 **Sub-projects:**` marker line directly under its open
    `^prj` listing ≥ 1 wikilink child (any state), return `true` with no walk. Reuse the
    line-scan rules of `projectMarkerHasOpenChild` in `089-successor-project.js`,
    factored into a shared "listed children" helper.
  - Otherwise, the old walk decides: root (`root: true`, area parent, no parent,
    unresolved or ambiguous parent) ⇒ `false`; a project parent or a non-self cycle ⇒
    `true`.
  - A missing `noteText` ⇒ `false` (neutral).
  - Pass `text` from both `leafProject` call sites in `085-successor-plan.js` (and from
    `planSuccessorKeepMoving`'s own call, which already holds `text`). Keep
    `options.leafProject` injectable as `(path, text) => bool`.
- `successorKeepMovingResolveParent`: when counting stem matches, ignore files for which
  `projectParentPathIsIndexed` is false (`084-project-parent.js`). This is LP16.
- Update `scripts/test-task-status-cycler-keep-moving.cjs`:
  - LP1–LP17 (markers in fixture text);
  - KM6 → umbrella;
  - a KM33 end-to-end case using a standalone top-level project with 4 Next residents,
    where Ctrl+Enter on the live link yields a 4-row `Keep ‹N› moving` group;
  - an LP16 case with a `_generated/tag_pages/topic/sase.md` file in the fake vault.
- Update the keep-moving paragraph of `docs/task-status-cycler.md` (~line 161) and the
  description row to say "any project except an umbrella". Run `npm run docs:check`.
- Bump the plugin version per repo convention. Run `npm run build` and `npm test` (at
  least `npm run build:check` and the cycler keep-moving and successor test files), then
  `bob plugins sync`, following AGENTS.md's refusal procedure if sync refuses.

## Phase `mark-contract` — display contract (bob-cli docs)

Write `docs/keep-moving-marks.md`, the authoritative display contract, in the style of
`docs/frees-marks.md`. It needs these sections:

- **What you see:** the screenshot ledger with `⏭ 4`, plus the all-marks row.
- **Principles:**
  - display-only;
  - truthful or neutral (a shown mark is a promise; no mark is not a guarantee);
  - one rule shared with the picker (the parity invariant);
  - what is deliberately not counted, and why: hand-offs (🔑), relation successors,
    project-finished pull-ins, and residents of sub-projects.
- **Which links:** §2.
- **The count rule:** §3.
- **Anatomy and look:** §4, with the full CSS.
- **Placement, Tooltip, Interaction, Motion, Liveness, Toggle, Performance:** §5–§7.
- **Architecture note:** §8.
- **Deferred:** §9.
- **Conformance vectors:** KB, KT, KP copied exactly, with LP referenced in §12.17.
- **Live verification:** a placeholder section that the `audit` phase fills.

Name the JS mirror files: core `src/141d-keep-moving-marks-model.js` and
`src/141e-keep-moving-marks-render.js`; mixins `src/271g-plugin-keep-moving-marks.js`
and `src/271h-plugin-keep-moving-marks-reading.js`; tests
`scripts/test-ledger-tools-keep-moving-marks.cjs` and
`scripts/test-keep-moving-marks-parity.cjs`. State that there is no api namespace.

Then:

- Register the page next to `frees-marks` in `docs/_meta/contracts.yml` (summary roughly
  "Keep-moving marks: a quiet `⏭ N` after the last planned Task Link of a project,
  forecasting the Keep ‹N› moving picker"), in both `mkdocs.yml` spots, and in
  `docs/_meta/url_contract.txt` if the gate requires it.
- In `docs/frees-marks.md`, point the "keep-moving offers (⏭, §12.15)" not-counted
  bullet at the new page. Update FP2's order note: ⏭ follows 🔑.
- In `docs/off-clock-link-chips.md`, move the chip to `side: 4` in Placement and OP1.
  Its order note now reads 🔓 1, 🔑 2, ⏭ 3, off 4, and its Reading insertion goes after
  any ⏭ host.
- Add one bullet at the end of §12.15: "Ledger forecast: the
  [Keep-moving marks](keep-moving-marks.md) `⏭ N` after a Task Link equals the rows
  this group will offer."
- Run `just docs-gen` if registries changed, then `just docs-check` until it passes.
- Commit as `docs(keep-moving-marks): add ledger keep-moving mark display contract`. Do
  not touch `Cargo.toml` or `CHANGELOG.md`.

## Phase `mark-core` — pure core and parity (bob-plugins bob-ledger-tools)

Open bob-plugins with `/sase_repo` and read its `AGENTS.md`. Implement
`docs/keep-moving-marks.md` exactly. The contract wins: if it is wrong, fix it in
bob-cli in this phase.

- **`src/141d-keep-moving-marks-model.js`** (pure, never throws, ≤ 1000 lines):
  - `keepMovingMarkLeafProject(path, text, ctx)`: §1 plus LP1–LP17. Port the
    `successorLeafProject` fixed in `umbrella-cycler`, with `_generated/`-safe
    ambiguity. `ctx` injects `frontmatterFor`, `resolveLinkpath`, and `markdownPaths`.
  - The live-link parse for G4. Port `parseSuccessorLedgerEntries` and
    `successorLinkMatchesIdentity` semantics, covering plain, embed, and `#`-deferred
    links under open entries, `()` placeholders, and same-note `[[#^id]]`.
  - The resident parse from N's text: `#task` plus `#ref`-without-`#task`, with status,
    block id, `[id::]`, `[dependsOn::]`, scheduled, `#hide`, and `#prj_is_blocked`.
    Recognition and R's G5 checks mirror `projectSurfaceState`, `prjInheritsBlock`, and
    `projectScheduledIsFuture`.
  - `keepMovingMarkCount(X, project, ctx)` implementing §3. It returns
    `{state: "unknown"|"zero"|"positive", count, rows, folded, projectPath, projectStem}`.
  - `keepMovingMarkCandidateLinks(content, memo, today, options)`. It mirrors
    `freesMarkCandidateLinks`: one resolution and one count per X per build, eligible
    zero entries kept for the LRU, and the `isVisible`, `onResolve`, and `onCount`
    hooks. The public `keepMovingMarkLinks` filters to `count ≥ 1`.
- **`src/141e-keep-moving-marks-render.js`:** `keepMovingMarkTooltip(model)` (§6),
  `buildKeepMovingMarkElement(doc, model)` (§4 host, listener-free), and
  `KeepMovingMarkWidget`.
  - The widget's `eq` compares block id, count, tooltip, project, and source path.
  - Its `toDOM` adds only the caret-place mousedown and the Mod+click that opens N, plus
    the tick class removal (about 400 ms), copied from `FreesMarkWidget`.
- **Config:** add a stat-cached reader beside `loadPlanCaps` in
  `320-plan-lane-and-config.js` for `plan.keep_moving`, `plan.link_unblocked`, and
  `relations.successor_labels`. Missing or invalid values read `true` (labels read
  empty), matching the cycler, with injectable `fsModule`/`parseYaml` for tests.
- Add both fragments to `src/fragments.json` right after `141c-off-clock-link-chips.js`.
- **Tests:**
  - `scripts/test-ledger-tools-keep-moving-marks.cjs`: KB1–KB23 (as pure model vectors),
    KT1–KT8, LP1–LP17, element and widget building, and the perf budget (§7). Cite each
    vector id in a comment.
  - `scripts/test-keep-moving-marks-parity.cjs`: load both harnesses
    (`ledger-tools-harness.cjs` and `task-status-cycler-harness.cjs`; see
    `scripts/test-ledger-tools-off-clock-link-chips.cjs` for a cross-plugin precedent).
    For every KB fixture where the mark is computed, build the cycler's inputs for a
    Ctrl+Enter Done of X:
    - C = {X};
    - N's staged text with X `[x]`;
    - today's text as baseline, and as the current day with X's link struck;
    - the same dependency index, config, and relation labels.

    Run the full successor planner (`085-successor-plan.js`, through verdict dedup).
    Assert identity-set equality of rows, equal folds, and "0 ⇔ no mark" (KB24). Include
    LP parity: `keepMovingMarkLeafProject` equals the cycler's `successorLeafProject` on
    every LP fixture.

  - Add both test files to `package.json`'s `test` script.

- Bump the plugin version per repo convention. Run `npm run build`, `npm test`, and
  `npm run docs:check`, then `bob plugins sync`. The fragments are inert until
  `mark-surface` wires them; that is fine.

## Phase `mark-surface` — the mark on screen (bob-plugins bob-ledger-tools)

Implement the runtime half of `docs/keep-moving-marks.md`. Use `271b`/`271c`/`271d` as
the method-for-method template.

- **`src/271g-plugin-keep-moving-marks.js`** (`BobLedgerToolsKeepMovingMarksMixin`):
  - `setupKeepMovingMarks`;
  - `keepMovingMarksAvailable`;
  - `createKeepMovingMarksExtension` (`Prec.highest` ViewPlugin);
  - `keepMovingMarkShouldRebuild` (a `StateEffect` refresh, doc, viewport, Live Preview,
    and file switches);
  - `buildKeepMovingMarkDecorations` (`side: 3`, visible ranges only, LRU tick,
    `suppressMotion` on viewport-only rebuilds);
  - `toggleKeepMovingMarks`;
  - the refresh, schedule, rollover, changed-file, and teardown methods.

  Accessors wire into the core:
  - `noteTextFor`: `taskThemeSourceTextFor`; on a miss, call
    `taskThemeRequestSourceText` and treat the result as unknown;
  - `frontmatterFor`: `metadataCache`;
  - `resolveLinkpath`: `getFirstLinkpathDest`;
  - `markdownPaths`: `vault.getMarkdownFiles`;
  - config;
  - the relation index.

- **`src/271h-plugin-keep-moving-marks-reading.js`:** a Reading sweep that adds,
  updates, or removes `.bob-keep-moving-mark` hosts in place in today's note in every
  window, with KP3/KP4 insertion. Register both mixins in `310-install-methods.js` and
  `src/fragments.json` (after `271f`).
- **Shared changes:**
  - Add `scheduleKeepMovingMarksRefresh` to the fill fan-out in
    `taskThemeRequestSourceText` (`279-plugin-task-theme-api.js`), and raise that LRU
    cap to 64.
  - In `271c`, move the off-clock chip to `side: 4`. In `271d`, teach the off-clock
    Reading insertion to step past a `.bob-keep-moving-mark`, with a test.
  - Hook the §7 liveness triggers, including the `scheduleLiveWidgetRefresh()` fan-out.
- **CSS:** the §4 block in `styles.css`.
- **Docs (bob-plugins):** in `docs/bob-ledger-tools.md`, update the description row, the
  commands table (`Toggle Task Link keep-moving marks` | `toggle-keep-moving-marks`),
  and add a short section beside the frees-mark one linking
  `https://bobs-org.github.io/keep-moving-marks/`. Then `npm run docs:inventory` /
  `npm run docs:check`.
- **Tests:**
  - `scripts/test-ledger-tools-keep-moving-marks-surface.cjs`: KP1–KP7, the toggle, the
    rollover, cache-miss ⇒ request plus no mark, then fill ⇒ refresh ⇒ mark, the
    baton-pass tick only on a changed key, no tick on viewport-only rebuilds, and the
    Reading sweep removing stale hosts.
  - `scripts/obsidian-keep-moving-marks-e2e{,-lib}.cjs` plus a
    `scripts/test-obsidian-keep-moving-marks-e2e.cjs` `--self-test`, copied from the
    frees e2e. Run it in an isolated scratch vault (never the live vault; `xvfb-run -a`,
    Tasks `taskFormat: dataview`), in light and dark × Live Preview and Reading. Cover:
    - the KB7 `⏭ 4` row;
    - KB2 → KB3 baton pass, with one tick burst;
    - the KP2 all-marks order row;
    - a CDP hover whose visible `.tooltip` equals the aria-label;
    - a plain click that writes nothing;
    - a Mod+click that opens N;
    - the toggle, with row height unchanged;
    - glyph baseline alignment and colour against the Next token, from reviewed
      screenshots.

    Record measured metrics in the contract's Live verification section, as frees did.

- Add the new tests to `package.json`. Bump the version. Run `npm run build` and
  `npm test`, then `bob plugins sync`.

## Phase `audit` — real-vault audit (scratch only)

Never touch `~/bob` or its git state.

- **Build the scratch vault.** Make a scratch clone of the vault repo
  (`git clone --no-checkout ~/bob <tmp>`), check out the research snapshot `d5d358c4`,
  and copy `2026/20261111.md` to the audit day's daily-note path. Do the same for a
  second scratch copy of the current `HEAD`.
- **Build bob.** Build `bob` in release mode from bob-cli `master` (which includes
  `umbrella-rule`). Point it at the scratch vault through its vault-path override.
- **Collect and compare.** Use the e2e harness (with the synced plugins) to collect
  every rendered `.bob-keep-moving-mark` (link, `data-count`, `data-project`, and the
  tooltip rows). For each mark:
  - run `bob capture --dry-run -f json '!<note stem>:<block id>'` on the same scratch
    vault, and compare the identity set of its `keep_moving` rows with the mark's rows;
  - in the scratch Obsidian, run Ctrl+Enter on the link, read the picker's
    `Keep ‹N› moving` rows, compare, and Esc (which writes nothing).
- **Check unmarked links.** For every unmarked live single link into a leaf project,
  assert that the dry run's `keep_moving` is empty, except for documented neutral cases
  (KB14, KB20, KB23).
- **The screenshot snapshot must show exactly one ⏭:** `⏭ 4` on
  `job_email_recruiters#^bryan-stephen-luzern-risk`, and nothing on `bob`, `sase`,
  `sase_blog`, `cash`, or `body` links. The picker must open as
  `Keep job_email_recruiters moving` with `^ecogpt`, `^ray-lyles`,
  `^product-engineer-dan-dowd`, and `^senior-rust-java-engineer`, nothing preselected.
- **Record the results.** Fill the Live verification section of
  `docs/keep-moving-marks.md` in bob-cli with counts, mismatches (expected: zero),
  environment, and screenshots reviewed, then run `just docs-check`. Fix any mismatch at
  its source: the core or contract in its repo, or the cycler/Rust if they disagree with
  §12.15. If that grows beyond a small fix, record a `PROPOSED FOLLOW-UP:` on this
  phase's bead.

## Verification (epic level)

- `cargo` checks pass in bob-cli. `npm run build:check`, `npm test`, and
  `npm run docs:check` pass in bob-plugins. `just docs-check` passes in bob-cli.
- LP1–LP17 agree across Rust, the cycler, and the ledger port. KB24 parity holds on
  every fixture.
- In real Obsidian, the screenshot ledger shows exactly `⏭ 4` on the recruiter link.
  Ctrl+Enter opens `Keep job_email_recruiters moving` with the same four rows, and Esc
  writes nothing.
- The mark is quiet:
  - muted count, Next-tinted glyph, and 🔑's metrics;
  - 🔓 🔑 ⏭ off order;
  - no line-height change;
  - a single baton-pass tick;
  - silence on doubt.
