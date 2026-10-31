---
tier: tale
title: Label Pomodoro home chips by root parent
goal:
  Pomodoro line chips name only root parents, so a session linking bob and its
  sub-project bob_command_mode_improvements reads `bob`.
size: medium
decisions:
  scope:
    ask:
      Should bob's own homes label and chip also switch, so `bob plan` and the Mac
      Capture agenda match Obsidian?
    choices:
      everywhere:
        Rust contract, vectors, and the Obsidian mirror all switch; the Mac agenda
        follows unchanged
      obsidian:
        Only Obsidian chips switch; bob JSON, `bob plan`, and the Mac agenda keep
        nearest-home labels
    default: everywhere
    why:
      Rust owns the R7 contract Obsidian mirrors; one rule keeps Obsidian and the Mac
      agenda in sync
    answer: everywhere
  memory_glossary_root_parent:
    ask:
      Should the Root Parent glossary note say that Pomodoro home chips are labelled by
      root parent?
    memory:
      - glossary:root-parent
    default: false
    answer: false
proposed_by: bbugyi200.apollo.bo
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-10-31 10:32:50
status: wip
---

# Label Pomodoro home chips by root parent

## Goal

The quiet chip at the end of each Pomodoro line in Obsidian names the **nearest home**
(first area/project on the note chain) of every linked task. When a session links
`bob#^…` tasks and also `bob_command_mode_improvements#^prj` (a project whose `parent`
is `bob`, with no `root: true`), the chip reads `○ bob · bob_command_mode_improvements`.
Bryan wants the chip to name only **root parents** (docs/pomodoro-homes.md R2), so that
row reads `○ bob`.

Root parents are already computed in Rust (`Home.root_parent` in
`src/native/pomodoro_homes/resolve.rs`) and drive Stats and the Day strip, but the
chip's `label`/`chip` (R7) still group by nearest home. The Obsidian chip is a
JavaScript mirror (bob-plugins `bob-ledger-tools`) of the Rust contract, and both run
the same `vectors.json`. The mirror has no general root walk yet.

> [!decision] scope = everywhere Change the shared contract in bob-cli first (Rust,
> vectors, docs), then mirror it in bob-plugins. `bob plan`, capture outputs,
> `capture-pomodoros --tasks` (the Mac idle agenda), the unnamed-entry chip in
> `src/native/pomodoro.rs`, ref Today output, and Stats session labels all show root
> parents. Bob Mac Capture needs no change because it renders bob's `label`/`chip`.
> Follow every section below.

> [!decision] scope = obsidian Skip **Part A**'s Rust changes and keep bob-cli's
> `vectors.json` byte for byte. In bob-cli, add only a short paragraph to R7 of
> docs/pomodoro-homes.md: the Obsidian ledger chip, its tooltip, and the Today-mark
> session label group the same parts by root parent (R2) using the rules below, while
> bob's JSON `label`/`chip` keep nearest homes. In bob-plugins, do **Part B**, but put
> the changed label, chip, and hue expectations and the new root cases in a JS-only
> fixture (`plugins/bob-ledger-tools/scripts/fixtures/pomodoro_homes/root_chips.json`,
> keyed by case name and entry index). The shared-vector test still checks every shared
> field and uses that fixture to override `label`, `chip`, and `hue`.

## The new rule (R7, root-parent labels)

Apply these rules in Rust and JS alike:

1. **Parts stay per nearest home.** `homes.parts`, their shares and minutes,
   `lead_home`, and every Stats dimension are unchanged, because Stats needs the project
   granularity. No JSON field is added, removed, or retyped. The schema does not change,
   so this is a behavior change, not a `!` break.
2. **Group parts by root parent.** A part's root is its home's `root_parent` (R2).
   Bucket parts are their own root (`@unfiled`, `@unlinked`). The root label is the
   lowercase stem of the root path, or `unfiled` / `unlinked` for a bucket, using the
   same stem rule as home labels.
3. **Root share** is the sum of its parts' whole-percent shares. **Root tasks** is the
   sum of its parts' task counts. **Root order** is the ledger order of its first part.
4. **Label:** sort roots by share, descending, with ties kept in root order. Show at
   most two root labels joined by `·`, then add ` +N` for the rest. An unlinked session
   keeps an empty label.
5. **Chip:** same as the label, except it is suppressed for an unlinked session or when
   the session name's case-insensitive `+` component set equals the set of **root**
   labels. A `— SASE` session that links only `sase_blog_notes` tasks now has no chip.
6. **Hue:** the dot takes the **lead root's** area hue. The lead root has the most
   distinct tasks, with ties going to the first task in ledger order (the same rule
   Stats count mode already uses for roots). A root's hue equals each of its parts' hue,
   because the root lies on the same chain below the same top-level area. Take it from
   the root's first part. `lead_home` itself is unchanged.
7. **Tooltip (Obsidian only):** one line per root, in label order: `<root> · N task(s)`
   plus ` · Mm` when attributed.

## Part A — bob-cli (Rust contract)

1. `src/native/pomodoro_homes/label.rs` and `session.rs`: build root groups in
   `attribute_session` from `grouped` and the computed shares, using
   `HomeGroup.home.root_parent`. Rework `label()` and `chip()` to take the root groups
   (for example a small `LabelPart { label, share }` slice, or the groups themselves)
   instead of `&[HomePart]`. Set `homes.hue` from the lead root (rule 6). Add a helper
   next to `note_stem` in `resolve.rs` for the root label
   (`note_stem(Path::new(&root_parent))` for real roots, bucket labels otherwise), and
   reuse Stats' existing root-label helper if one already exists.
2. `tests/fixtures/pomodoro_homes/vectors.json`. Hand-verify each change against R2 and
   do not regenerate blindly. Expected changes to existing cases:
   - `subproject-archive-homes`: label/chip `sase · bob` (sase 80, bob 20), hue `blue`.
   - `named-sessions-use-task-homes` entry 4 (`sase_blog`): label/chip `sase`.
   - `resolution-and-lints`: entry 1 (`done/task_done` → hack) becomes `sase`. Entry 4
     (`cycle_a`) becomes `cycle_b`, because the outermost project reached stands in as
     root. Confirm that Rust's `root_parent` agrees.
   - `chips, forecast shares, empty forecasts, and session identity`: entries 1 and 6
     (`— SASE` → sase_blog_notes) get label `sase` and chip `null`. Entry 2
     (`— SASE + BOB`) gets label `sase · bob` and chip `null`. Entry 7 (`— TIE`) gets
     label/chip `sase · bob`.
   - `depth-and-parent-shapes` entry 1: label/chip `sase`.
   - `root-fields` entry 4 (`inbox-goals`, `[root:: sase_goals]`): label/chip `sase`.
   - All other entries keep their values, and `expected_lead_homes` does not change.
   - Update the case table in docs/pomodoro-homes.md if any case's purpose changes.
3. Add notes to the mini-vault: `rooted_blog.md` (`type: [[project]]`,
   `parent: [[sase]]`, `root: true`), `rooted_blog_notes.md` (project, parent
   `rooted_blog`), `quoted_root.md` (project, parent `sase`, `root: "TRUE"`), and
   `quoted_root_child.md` (project, parent `quoted_root`). Add a case
   `root-parent-chips` that pins:
   - A named forecast linking `sase#^a` and `sase_blog_notes#^b`, which gives label/chip
     `sase`.
   - A closed 10m session linking `rooted_blog_notes#^c` and `rooted_blog#^d`, which
     gives label `rooted_blog`.
   - A forecast linking `sase_goals#^e`, `sase_blog#^f`, `bob#^g`, and `bob#^h`, which
     gives label `sase · bob`, hue `blue` (lead root sase wins the 2–2 tie by ledger
     order), and lead home `bob.md`.
   - A closed session linking `quoted_root_child#^i`, which gives label `quoted_root`.

   Hand-calculate shares, minutes, and health, and check that the existing
   `ambiguous-basename` and other cases still hold.

4. Fix the Rust tests whose expectations named subprojects, if any:
   `capture_pomodoros.rs`, `capture_complete/tests`, `stats/homes.rs`, plan, pomodoro,
   ref Today, and capture-start chip tests. Run `just check` and `just lint` and have
   both pass.
5. Docs: in docs/pomodoro-homes.md, rewrite R7 using the rules above, note in R2 and R9
   that `label`/`chip`/`hue` name root parents while `parts` stay per home, and update
   the intro and examples where needed. Grep docs/ for chip examples that show a
   sub-project label (`capture.md`, `plan.md`, `reference/cli/capture-pomodoros.md`) and
   fix only examples that are now wrong. Run `just docs-gen` if any generated reference
   changes, then run `just docs-check`.

## Part B — bob-plugins mirror (Obsidian)

Open the linked repo with `/sase_repo` (`bob-plugins`), read its `AGENTS.md`, edit only
`src/` fragments, and never edit `main.js` by hand.

1. `plugins/bob-ledger-tools/src/142-pomodoro-home-chips.js`:
   - Add `pomodoroHomeRootFlag(frontmatter)`: true for boolean `true`, or for a string
     that trims to `true` case-insensitively, matching Rust's
     `trim_yaml_scalar(..).eq_ignore_ascii_case("true")`.
   - In `pomodoroHomeResolve`, compute `root` and `root_label` exactly as Rust's
     `resolve_note` does. If the home is an area, root is the home. If it is a project,
     let `a` be the first area index above the home, or the chain end. Root is the first
     project with the root flag in `chain[h..a]`, else the last project there. Bucket
     parts carry `@unfiled` / `@unlinked`.
   - Carry the root through `pomodoroHomeShareParts` and `pomodoroHomeSessionModel`.
     Build root groups, then compute `label`, `chip`, `hue`, and `tooltip` by rules 2–7.
     Keep `parts` and `lead_home` as they are. `pomodoroHomeDisplayLabel` can take root
     groups.
   - Keep the fragment at 1000 lines or fewer. If it would go over, move the root-group
     helpers into a new fragment and list it in `fragments.json`.
2. `279-plugin-root-marks-model.js`: make `rootMarkRootPath` return the new resolver's
   `root` for the target, keeping its name and export, so root marks and chips cannot
   disagree. `scripts/test-ledger-tools-root-marks.cjs` must still pass. If one of its
   cases depends on the old walk past an area, follow R2 and adjust that test.
3. Copy bob-cli's updated `vectors.json` byte for byte into
   `plugins/bob-ledger-tools/scripts/fixtures/pomodoro_homes/`. Update that README so it
   cites the bob-cli change by its commit subject, because the SHA is not known until
   commit. In `scripts/test-ledger-tools-pomodoro-home-chips.cjs`, make
   `parseFrontmatter` turn a bare `true` / `false` into booleans (Obsidian's shape)
   while quoted values stay strings. Update the Live Preview, Reading view, and DOM
   tests that expected `sase_blog`, which now render `sase`. Check the today-marks
   tests, because `todayHomeLabelsForCache` reads `session.label`.
4. Update the plugin `manifest.json` description and the hand-written chip prose in
   `docs/bob-ledger-tools.md` so they say the chips name each linked task's root parent.
   Run `npm run docs:gen`.
5. Run `npm run build`, `npm test`, and `npm run docs:check`, and have all three pass.
   Then run `bob plugins sync`.

## Memory

> [!decision] memory_glossary_root_parent In `sase/memory/glossary/root-parent.md`,
> extend the sentence about the Day strip and Stats dash so it also says Pomodoro home
> chips (Obsidian, `bob plan`, and the capture agenda) are labelled by root parent. Use
> `/sase_memory_write`, run `sase memory init`, then `just docs-gen` so
> docs/reference/glossary.md matches, then `just docs-check`.

## Verification

- `just check`, `just lint`, and `just docs-check` pass in bob-cli.
- `npm test` and `npm run docs:check` pass in bob-plugins, and `bob plugins sync`
  succeeded.
- Spot check: `bob capture-pomodoros --all --tasks --format json` on today's daily note
  (read only). The 1010-1035 BUILD entry, which links `bob#^…` tasks and
  `bob_command_mode_improvements#^prj`, reports `homes.label` `bob`, and its `parts`
  still list both homes.
- Out of scope: Stats dimensions, `lead_home`, the Day strip, capture routing, and Bob
  Mac Capture code.
