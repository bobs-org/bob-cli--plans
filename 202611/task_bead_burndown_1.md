---
tier: epic
title: "Task-bead burn-down 1: the gates that keep failing for every agent"
goal: "The five open task beads that most often turn agents' check runs red for reasons
  unrelated to their work are fixed and closed. The lander triages what is left and
  proposes the next five-bead epic under the same rules, and the chain continues until
  no open task beads remain.

  "
decisions:
  docs_headroom:
    ask:
      Trim the two bob-plugins reference pages that sit within 10 words of the cap
      before the next edit trips it?
    default: true
    why:
      Agents hit these two pages over and over (18 and 12 +1s); each failure blocks an
      unrelated landing
    answer: true
phases:
  - id: wispr-kicker
    title: Whitespace-insensitive WISPr dossier kicker assertion
    depends_on: []
    size: small
    description:
      "wispr-kicker: make the wispr_summary lib test check the letter-spaced MEETING
      NOTES kicker without depending on pdftotext's spacing, then close bob-cli-q9."
  - id: docs-mount-index
    title: Stale ignored mount index is not an authored landing page
    depends_on:
      - wispr-kicker
    size: small
    description:
      "docs-mount-index: stop bob-docs calling an untracked, gitignored mount index.md a
      committed landing page, add a tools/docs test, then close bob-cli-it."
  - id: frees-marks-budget
    title: Stable frees-marks 25 ms CPU budget measurement
    depends_on:
      - docs-mount-index
    size: medium
    description:
      "frees-marks-budget: replace the single cold CPU sample in the bob-plugins
      frees-marks 200-link/2000-task gate with the warm best-of-N pattern that fixed
      sibling gates, keep the 25 ms target, then close bob-cli-io."
  - id: plugin-docs-headroom
    title: Editing headroom on the near-cap bob-plugins reference pages
    depends_on:
      - frees-marks-budget
    size: medium
    description:
      "plugin-docs-headroom: per docs_headroom, condense the bob-navigation-hotkeys and
      bob-ledger-tools reference pages to at least 300 words under the cap, or cancel
      both beads, then close bob-cli-ob and bob-cli-il."
proposed_by: bbugyi200.athena.1h4
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-27 10:16:11
status: wip
---

- **PROMPT:**
  [prompts/202611/task_bead_burndown_1.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/task_bead_burndown_1.md)

# Plan: Task-bead burn-down 1 — the gates that keep failing for every agent

## Context

On 2026-11-27 the project had 327 open task beads. All of them went through a read-only
triage. 25 were closed with concrete evidence: 24 were already fixed or moot (each close
note cites its fix commit, check command, or the human reviewer's decline), and one,
bob-cli-ca, was a duplicate of bob-cli-bo. That left 302 open task beads. Every bead the
triage did not close was judged still valid, or impossible to disprove cheaply.

This epic takes the five beads that cost the most agent time right now. Each one either
fails on every run or keeps breaking a gate that every landing must pass:

| Bead       | Type    | +1s         | Why it is first                                                                                                |
| ---------- | ------- | ----------- | -------------------------------------------------------------------------------------------------------------- |
| bob-cli-q9 | ci      | 8 in 2 days | Fails deterministically in `cargo test --lib`, so it turns `just check` red on athena for every bob-cli change |
| bob-cli-it | bug     | 11          | False `just docs-check` failure in reused workspaces, often misdiagnosed as a red master                       |
| bob-cli-io | flake   | 59          | Fails bob-plugins `npm test` more often than any other node                                                    |
| bob-cli-ob | feature | 18          | bob-navigation-hotkeys reference page is at 5997/6000 words and has repeatedly failed docs-check               |
| bob-cli-il | feature | 12          | bob-ledger-tools reference page is at 5994/6000 words and has repeatedly failed docs-check                     |

The phases run strictly one after another.

Rules every phase follows:

- Never edit the `Cargo.toml` version or `CHANGELOG.md`, and mark any CLI or JSON
  contract break with `!` (decisions:bob-cli-releases-ride-a-daily-train). None of these
  phases should change a contract.
- Open bob-plugins only through the `/sase_repo` skill, and work only at the path it
  prints. Follow bob-plugins `AGENTS.md`. These phases change only tests and docs, so
  run `bob plugins sync` only if that file requires it for such changes.
- Phase workers never create beads. Record discovered work as `PROPOSED FOLLOW-UP:`
  notes on your own phase bead.
- When your phase's verification passes, close each task bead it owns with
  `sase bead close <id> --note "<commit SHA(s) and the exact verification commands and results>"`.
  If verification fails, leave the bead open and say why in a note.
- Read `docs.md` reference memory before touching `docs/` or `tools/docs`.

## Phase wispr-kicker — bob-cli-q9

`native::meetings::profile::tests::wispr_summary_renders_one_or_two_pages_with_stable_words`
fails at `src/native/meetings/profile.rs:489` on athena (pdftotext 25.03.0). The cause
is `assert!(text.contains("MEETING NOTES"), "kicker: {text}")`. The dossier print
profile letter-spaces the kicker, so pdftotext extracts it as `M E E T I N G N OT E S`.
The rendered PDF is correct. Only the assertion depends on how the extractor spaces
glyphs.

- Compare the kicker without whitespace: strip all whitespace from both the extracted
  text and the expected `MEETINGNOTES`, then check containment. Keep the failure message
  printing the raw text.
- Keep every other assertion as it is: no TOC, TL;DR, the Owner/Action columns, the page
  count, and the stable words. Do not change the print profile's letter-spacing.
- If other assertions in the same module check letter-spaced text through `pdf_text`,
  give them the same treatment, and name them in the commit.
- Verify: `cargo test --lib -- wispr_summary` passes on athena, then `just check`
  passes, apart from any already-tracked flake named in your note.

## Phase docs-mount-index — bob-cli-it

`tools/docs/bob_docs/mounts.py::validate_mount_targets` (the `plan.source_hub` branch
around lines 807-813) raises "collides with the committed landing page … never silently
replace an authored landing page" whenever the target `index.md` exists and is not in
the mount manifest. It never checks whether that file is committed. In long-lived
workspaces the file is old mount output: untracked, and gitignored by `.gitignore` lines
4 and 7.

- Classify the existing index before raising. `tracked = _tracked_files(root)` is
  already computed, and the module's `path_ignored(root, index_rel)` reports gitignored
  paths.
  - Tracked, or untracked but not ignored: this is an authored page. Keep the current
    refusal.
  - Untracked and gitignored: this is stale mount output. Treat it like the existing
    stale-output cleanup treats leftover copies, and let the mount replace it. The
    cleanup's rule of never deleting non-ignored files stays as it is.
- Add a `tools/docs/tests` case for each class: an ignored, untracked, unmanifested
  `index.md` under a source-hub mount target is replaced and the check passes; a tracked
  `index.md` still refuses. Model the fixtures on the existing mount tests.
- Verify: run the tools/docs pytest suite the way the `justfile` does. Reproduce the bug
  by planting a stale ignored `docs/reference/plugins/index.md` that is not in
  `mount-manifest.json`; `just docs-check` must pass with it in place, and it must still
  refuse for a tracked landing page. Then run a clean `just docs-check`.

## Phase frees-marks-budget — bob-cli-io

bob-plugins `scripts/test-ledger-tools-frees-marks.cjs` (around line 481) has the node
"200-link / 2000-task build stays within the 25 ms budget". bob-plugins 84ba9d41
switched it from wall clock to a single cold `process.cpuUsage` sample, but under a full
`npm test` it still measures 26.7–36 ms of CPU, while isolated runs measure 16–18 ms.
The extra time is cold-start and contention noise, not algorithm cost.

- Use the measurement pattern from bob-plugins abbac5ac, which fixed the sibling
  event-link-times and keep-moving-marks gates (closed bob-cli-km and bob-cli-nw): a few
  warmup builds, then the best (minimum) CPU time over several samples. If the repo has
  a shared helper, use it; if abbac5ac inlined the pattern, factor a small shared helper
  only when that stays local to the test scripts.
- Keep the 25 ms target and the deterministic assertions (200 rows, one resolution, one
  count). Do not raise the target. The gate must still catch a real regression, so say
  in the test comment why the minimum of N samples measures algorithm cost.
- Out of scope: bob-cli-f8 (task-wikilink cold-chunk node) and bob-cli-m1 / bob-cli-pg
  (other plugin perf gates) stay open. If the new helper would obviously fix one of
  them, record that as a `PROPOSED FOLLOW-UP:`.
- Verify: `node --test scripts/test-ledger-tools-frees-marks.cjs` passes, and then at
  least three consecutive full `npm test` runs at normal parallelism pass this node.
  Record the measured best-of-N values in the close note.

## Phase plugin-docs-headroom — bob-cli-ob, bob-cli-il

> [!decision] docs_headroom

Two mounted bob-plugins reference pages sit right at the 6000-word reference cap
(`tools/docs/bob_docs/lint.py:55`): `docs/bob-navigation-hotkeys.md` at 5997 and
`docs/bob-ledger-tools.md` at 5994. Any edit that documents a nav-hotkeys or
ledger-tools change fails `just docs-check`, and that has happened repeatedly. The
plugin-apis half of bob-cli-ob was already fixed by bob-plugins 79d71bfd (7510/8000 on
the advisory develop cap); confirm it is still under.

- Bring each page to at least 300 words under the cap. Do not raise the cap.
- Condense duplicated or restated prose, or move one coherent section to the page the
  docs routing rules say owns it. Keep all behavior, defaults, keybindings, and contract
  links. Preserve inbound anchors, or update every caller if a section moves (search
  both bob-plugins and bob-cli docs).
- Verify: bob-plugins `npm run docs:check` with explicit local mounts, then bob-cli
  `just docs-check` against the updated linked bob-plugins. Report the new word counts
  in the close notes.

> [!decision] docs_headroom = no Make no doc edits. Close bob-cli-ob and bob-cli-il as
> canceled
> (`sase bead close <id> -R canceled -r "Reviewer opted out of docs headroom work"`),
> citing the reviewer's docs_headroom = no answer.

## Landing and the next epic

The land agent lands the phases as usual. It then keeps the burn-down going:

1. **Confirm this epic's beads.** bob-cli-q9, bob-cli-it, bob-cli-io, bob-cli-ob, and
   bob-cli-il are each closed with a verification note, or the land note explains why
   one stays open.
2. **Triage what is left.** List the open task beads (`sase bead list -t task -n 0`).
   Close duplicates and obsolete beads only on concrete, checkable evidence: a fix
   commit with file:line, a passing check command, an existing memory strand that
   already says what the bead asks, a decision that makes it moot, or a human reviewer
   who declined that memory record (close as canceled). A bead's age, a low +1 count, or
   doubts about whether anyone wants it are not grounds to close. Keep by default. Every
   bead open at the end of the 2026-11-27 triage was already checked, so focus on beads
   created or updated since then, and on beads whose fix may have landed meanwhile. For
   large batches, fan out read-only helper agents and do every close yourself.
3. **Plan the next epic.** If any open task bead is neither snoozed nor owned by an
   active epic, use `/sase_plan` to write the next epic and `sase plan propose` it,
   under the same rules:
   - At most 5 phases and at most 5 task beads (the 5 most important). One phase may
     work several beads.
   - Phases run strictly in sequence: each phase depends on the one before it.
   - Keep decisions to a minimum, but use them to confirm any work that may not be
     wanted. Every feature bead, every memory bead, and every bead the triage marked as
     needing Bryan's opt-in gets a decision. Its "no" answer closes the bead as
     canceled. Author memory decisions with `/sase_memory_write`; a decision record that
     Bryan declined at an earlier plan review is closed, not re-proposed.
   - Read `roster:decisions` and cite each governing decision as `decisions:<keyword>`.
   - Priority order: deterministic gate failures, then correctness or data-loss bugs,
     then frequent flakes (by +1 count), then docs-gate breakers, then other bugs, then
     memory fixes for content that is now false, then features and new memory records
     (opt-in).
   - Copy this "Landing and the next epic" section into the new plan, updated, so that
     epic's lander continues the chain until no open task beads remain.
4. **Stop the chain** when only snoozed beads remain, or only beads owned by an active
   epic. List those beads with their wake times or owning epics in the land note.

### Next in line (2026-11-27 triage — re-verify before planning)

Highest-impact KEEP beads not in this epic, in rough priority order:

1. bob-cli-or with bob-cli-pb — two arms of `evaluate_guard` in
   `src/native/plugins/deploy.rs` skip the inclusion proof and falsely refuse plugin
   deploys; refusals have stalled epics.
2. bob-cli-qt — real-Obsidian e2e runners leak scratch directories into athena's /tmp
   (`obsidian-home-cards-e2e.cjs:1062`, hardcoded `/tmp/bm-e2e`).
3. bob-cli-id — chezmoi `poseidon_chezmoi_isolation_test` fails 2 of 4 cases on every
   push.
4. bob-cli-6g — an ordinary-task `@route` capture ignores `project_name_aliases` and
   creates a stray note (reproduces with `bob capture -d`).
5. Flakes by +1 count: bob-cli-f8 (22), bob-cli-m1 (13), bob-cli-5l (12), bob-cli-ej
   (11; retitle to the task-link-marks-live viewport node), bob-cli-bo (6; now also
   covers the lib nodes from duplicate bob-cli-ca), bob-cli-gd (4), bob-cli-pg (3).
6. Correctness bugs: bob-cli-oh and bob-cli-qa (debug-build panics, both reproduced),
   bob-cli-cd (bob-emoji plugin files tracked in vault git), bob-cli-qx, bob-cli-qz,
   bob-cli-pe, bob-cli-pu, bob-cli-qs, bob-cli-ox, bob-cli-oy, bob-cli-k4, bob-cli-hh.
7. Actionable now: bob-cli-g3 (its retire-deprecated-JSON-keys trigger has passed;
   breaking, so mark it `!`), bob-cli-ny (one-release intentions alias, now due),
   bob-cli-nc (a one-line daily-chore switch to `bob tick -a`), bob-cli-ma (Mac
   task-pickers page at 5978/6000).
8. sase-side beads (bob-cli-l1, bob-cli-jl, bob-cli-jn, bob-cli-jm, bob-cli-ei,
   bob-cli-qy, bob-cli-gy). The fixes live in the sase repo; plan them only through
   `/sase_repo`, or leave a note asking Bryan to move them.
