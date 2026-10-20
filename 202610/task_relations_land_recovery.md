---
tier: tale
size: medium
title:
  "Land task relations: restore the lost Mac relation card, fix relation docs, close
  bob-cli-a2"
goal:
  Bob Mac Capture on origin/master has the bob-cli-a2.14 relation card, preview rows,
  and capability warning; the relation guide and capture-parse help match real behavior;
  and epic bob-cli-a2 is closed with its plan marked done.
proposed_by: bbugyi200.athena.bob-cli-a2.land
proposed_with: claude/opus
bead: bob-cli-a2
status: done
---

- **PARENT:**
  [202610/task_relations.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/task_relations.md)
- **BEAD:**
  [bob-cli-a2](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-a2/README.md)
- **AGENTS:**
  - [bbugyi200.athena.bob-cli-a2.land](https://github.com/bobs-org/bob-cli--agents/blob/main/sessions/bbugyi200.athena.bob-cli-a2.land.md)
- **COMMITS:**
  - [4907f00](https://github.com/bobs-org/bob-cli/commit/4907f00ddd02635fe4f6698efcae21eb17b8554a)
    — docs(capture): quote multi-word relation labels and fix capture-parse help

# Land task relations: restore the lost Mac relation card, fix relation docs, close bob-cli-a2

You are finishing the landing of epic `bob-cli-a2` (task relations). The land agent has
already verified every phase, triaged every proposed follow-up, and recorded the
outcomes on the epic (`sase bead read bob-cli-a2 -r "..."`; see the `LAND TRIAGE` note).
Only the work below remains, and you also close the epic. Nothing resumes the landing
after you, so finish it in this turn.

## Context

### The lost a2.14 commit

Phase `bob-cli-a2.14` (mac-preview) reported "Landed commit e128340 in bob-mac-capture",
but it made that commit with a raw `git commit` inside a workspace's linked
`bob-mac-capture` checkout. Because the tree was clean afterward, its finalizer had
nothing to commit or push. The linked checkout was later re-cloned, so e128340 no longer
exists anywhere, and `origin/master` (now `7a41234`) has no relation card,
`relation_updates` preview rows, or `relation_grammar` capability warning.

The land agent rebuilt e128340 from a2.14's recorded muse session: 60 per-edit patches
plus its recorded `bob capture --dry-run` fixture commands. Two artifacts hold it:

- `file:explicit:decdc28e609c23a140df59b5` is the complete recovered diff
  (`git diff --binary`) against `bob-mac-capture` `7a41234`. It covers 19 files with
  +1640/-21, which is exactly e128340's recorded stat. `git apply --check` passes on
  `7a41234`. In a scratch clone, `just portable-check` passed 1472 tests, the count
  a2.14 reported. Its six fixtures were regenerated with the current `bob` through
  a2.14's own commands; they embed temp vault paths, as existing fixtures do.
- `file:explicit:ec5a23c2ae683a0b47c1c3dc` is a provenance tarball with `replay.py`,
  `gen-fixtures.sh`, and the 60 raw muse patch JSONs. You need it only if the diff no
  longer applies. In that case, replay onto a fresh tree with
  `python3 replay.py <unpacked-dir> <bob-mac-capture-root>`. The script splits each
  recorded path at `/sase/repos/linked/bob-mac-capture/`, so point it at an unpacked
  directory laid out as `tool-outputs/<uuid>/call_*-tool_patch.json`. Then run
  `gen-fixtures.sh "$(command -v bob)" <bob-mac-capture-root>`.

Get a file path for an artifact with `sase artifact path <ref>`.

a2.14's intended commit message, which you should reuse as the commit subject and body:

```text
feat(capture): add ~ relation card, preview rows, and capability warning

Add the Shift-Return relation-label card (field, suggestion chips with
counts from the relation_label probe, live source-label-target sentence,
typed-row replacement insert, Tab cycling, Escape to picker,
disabled_reason blocks submit), tolerant relation_updates /
linked_from_line_updates / minted_relation_block_ids / mirror_skipped
decoding with 🪢 preview rows, archived live-only suffix, managed
LINKS/LINKED FROM suppression, separate notification counts, minted ID
captions, and the old-Bob relation_grammar capability warning with
submit-twice gate. Add LX1/LX2/LX3-relabeled/LX10-batch/mixed fixtures,
decoding and presentation tests, protocol and preview docs.
```

The macOS-only targets (`BobMacCapture`, `BobMacCaptureTests`, including the new
`CaptureRelationPanelTests`) have never been compiled. a2.14 ran while the Mac was
offline. The Mac (`ssh mac`, user `bbugyi`) answered SSH during landing, but treat it as
best-effort.

### Two doc defects caused by the epic (epic note #1)

- `docs/guides/link-related-tasks.md` shows unquoted multi-word labels that do not parse
  as relations. Line 24,
  `'Investigate retry handling ~service:retry-loop|follow-up of'`, is captured as
  literal task text. Line 37,
  `'@bob+retry-investigation ~service:retry-loop|related to'`, parses as mode
  `sub_bullet` with body `to` and label `related`. Quoted labels (`|"follow-up of"`,
  `|"related to"`) parse correctly. Verify with
  `bob capture-parse -f json -- '<draft>'`. A sweep of bob-cli, bob-plugins, and
  bob-mac-capture docs found no other unquoted multi-word relation labels.
- `bob capture-parse --help` (`src/native/capture_parse.rs`, the `~` paragraph near
  line 366) still ends with "and `bob capture` currently refuses any item with relations
  with a temporary unsupported error." That has been false since `8755ffc8` shipped
  execution. `docs/reference/cli/capture-parse.md` is generated from this help, and no
  test pins the stale sentence.

## Steps

### 1. Restore the a2.14 work in bob-mac-capture

1. Open the linked repo with your `/sase_repo` skill:
   `sase repo open bob-mac-capture -r "Restore lost bob-cli-a2.14 relation card commit"`.
   Use only the printed path, and read its `AGENTS.md`.
2. Confirm that `git log origin/master` still lacks the relation card. For example,
   `grep -rn "CaptureRelationPanelTests\|relationGrammar" Sources Tests --include=*.swift`
   should find nothing in `Sources/`. If someone else already restored it, skip to
   step 2.
3. Apply the recovered diff:
   `git apply "$(sase artifact path file:explicit:decdc28e609c23a140df59b5)"`. If master
   has moved and the apply fails, try `git apply -3`. If that also fails, use the
   provenance tarball replay described above. Then resolve any genuine conflicts by
   hand, keeping both sides' intent.
4. **Do NOT run `git commit` (or `git stash` or `git reset`) in this repo.** Leave the
   changes uncommitted so this turn's `/sase_final` declaration commits and pushes them.
   A raw commit is exactly how a2.14's work was lost. Commit through the finalizer using
   the message above, keeping the `SASE_BEAD=[bob-cli-a2.14]` provenance if the
   finalizer accepts a bead reference.
5. Run `just portable-check` in the linked repo. Expect it to pass, with at least 1472
   tests if master has not moved.
6. macOS typecheck, best-effort over SSH. Copy the working tree to a scratch directory
   on the Mac and typecheck it there. Never pull into or modify the Mac's own checkout:

   ```bash
   mac_scratch=$(ssh -o ConnectTimeout=20 -o BatchMode=yes mac 'mktemp -d /tmp/bob-mac-capture-a2land.XXXXXX')
   rsync -a --delete --exclude='.git' --exclude='.build' <linked-path>/ "mac:$mac_scratch/"
   ssh -o ConnectTimeout=20 -o BatchMode=yes mac "cd '$mac_scratch' && ./Scripts/bundle.sh --identity - && just typecheck-tests"
   ssh -o ConnectTimeout=20 -o BatchMode=yes mac "cd '$mac_scratch' && swift test --filter 'CaptureRelationPanelTests|CaptureDependencyPanelTests'"
   ssh -o ConnectTimeout=20 -o BatchMode=yes mac "rm -rf '$mac_scratch'"
   ```

   This is the same rsync and `bundle.sh` / `typecheck-tests` flow a2.15 used. Fix any
   compile errors or test failures in the restored Swift sources, rerun
   `just portable-check`, and repeat the Mac check. A failure in a test file the
   recovered diff does not touch, and that also fails on a clean `origin/master` copy,
   is pre-existing: compare against the open macOS-CI tasks (`bob-cli-7o`, `bob-cli-8e`,
   `bob-cli-8j`, `bob-cli-9s`, `bob-cli-7h`, `bob-cli-7p`) and do not fix it here. If
   the Mac is unreachable (SSH timeout), record that in the close note. The macOS CI run
   that follows the finalizer's push will cover it, so do not wait for it.

7. Run `just docs-test` in the linked repo, since the diff edits `docs/protocol.md` and
   `docs/capture-previews.md`.
8. Do not install or update the Bob Mac Capture app on the Mac. Signing needs Bryan's
   unlocked GUI keychain, and that update is already recorded as a Bryan follow-up
   (`bob-cli-a2.15` note #2).

### 2. Fix the relation docs and capture-parse help in bob-cli

1. In `docs/guides/link-related-tasks.md`, quote the two multi-word labels: line 24
   becomes `...~service:retry-loop|"follow-up of"'` and line 37 becomes
   `...~service:retry-loop|"related to"'`. Check both with
   `bob capture-parse -f json --`. Each should report one `relations[]` entry with the
   full label, and the line 37 draft should parse as mode `task_dependency`.
2. In `src/native/capture_parse.rs`, replace the stale clause "and `bob capture`
   currently refuses any item with relations with a temporary unsupported error." with
   an accurate one that mirrors the neighbouring `&` / `*` sentences. For example:
   "`bob capture` executes recognized `~` / `~~` modifiers through the staged batch
   writer: it appends or relabels the source task's managed 🪢 LINKS line, mints any
   block IDs the edge needs, and stages the target's generated 🪢 LINKED FROM mirror;
   relations never block, consume, promote, or schedule anything." Keep the string's
   existing `\` line-join style. Check the claims against `docs/capture.md` § "Linking
   tasks with '~' and '~~'" and `docs/task-relations.md`.
3. Run `just docs-gen` to regenerate `docs/reference/cli/capture-parse.md`. Confirm that
   `rg "temporary unsupported" docs src` finds nothing.
4. Run `just check`. Then run `just docs-check`. Its two Material template tests
   (`test_templates_in_material`, `test_component_chips_in_material`) fail on a clean
   tree because of a known environment issue that is recorded on `bob-cli-a0`. Ignore
   those two, but nothing else may fail. Do not run `just check-full`.

### 3. Close out epic bob-cli-a2 (final step)

1. Run `sase bead epic-symbols bob-cli-a2`. At landing it printed "No --epic-symbol
   entries". If entries appear, resolve each one by wiring, privatizing, adding a
   non-test pragma, or deleting it. Re-key one only when a still-open later bead needs
   the exemption.
2. Close the epic. Never use `--force`.

   ```bash
   sase bead close bob-cli-a2 --note "<verification>"
   ```

   The note must say what you verified, in substance: all 15 phases were re-verified
   against their commits (bob-cli ae9629ec, 6d1f0a9c, b88a70e6, 1e1d9826, a02f9b33,
   fa9669dd, 8755ffc8, eefdeb47; bob-plugins 524bf72, 1497854, 8d33457, 03bb94b, 3ba8bf9
   with a2.11's stage folded in; the relation-index OOM fix b61f214; bob-mac-capture
   bb7127a). Decisions were honored: builtin inverse phrases, a LINKED FROM ×, glossary
   memory edits only, and no decisions strand. Post-epic drift needed no integration
   changes: `@@@` template start 0e9f915a and Mac templates ad85b85 expand to plain
   capture text upstream of relation parsing, and image zoom is a separate keymap.
   Report the a2.14 restore and its checks (portable-check count, macOS typecheck result
   or Mac-unreachable), the doc and help fixes, `just check` and `just docs-check`
   results, and the triage outcomes from the epic's `LAND TRIAGE` note. Those outcomes
   are: Material docs-test issue recorded on `bob-cli-a0`; +1 on `bob-cli-5l`, `6v`,
   `6w`, `7i`, and `3w`; the glossary proposal declined as already done; the six
   deliberate non-goals declined with reasons; and `bob-cli-a5` / `a6` kept for the
   relation-index OOM follow-up.

3. Run `just symvision` and confirm the whitelist is clean.
4. Set `status: done` in the frontmatter of the epic's plan file
   `plan:202610/task_relations.md`. Get its path from
   `sase artifact path plan:202610/task_relations.md` or from the PLAN line of
   `sase bead read bob-cli-a2`. The field currently reads `status: wip`.
5. The epic has no `parent_bead`, so there is no parent to close.
6. End with `/sase_final`. Declare commits for bob-cli (docs and help fix) and for the
   opened bob-mac-capture repo (the restored a2.14 work), plus any plans or beads repo
   changes it lists.

## Out of scope

- The Material/github-callouts docs-test fix, which belongs to `bob-cli-a0`.
- The known bob-plugins npm failures (`bob-cli-6v`, `6w`, `7i`, `3w`) and the ETXTBSY
  flake (`bob-cli-5l`).
- Relation-index publish coalescing (`bob-cli-a5`) and the Obsidian startup smoke check
  (`bob-cli-a6`).
- The six deliberate non-goals listed in the epic plan's "Deliberately not doing"
  section.
- Installing the Mac app.
