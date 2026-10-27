---
tier: epic
title: 'Backlog batch B2: macOS completion flake, Mermaid class, vault-format docs,
  and assets split'
goal: 'Resolve the four most important remaining bob-cli beads after batch A1: re-check
  the macOS completion flake against the new deadline pin, restore the web-clip Mermaid
  classification, correct the vault-format walk order, and decide the assets-crate
  split before the package breaches its cap. The lander then starts the next backlog
  epic, so the chain continues until no open task beads remain.'
decisions:
  assets_split:
    ask: Split bob-obsidian assets into a separate crate now?
    choices:
      split_now: Split now before the next asset addition breaches the cap
      defer: Defer the split and keep monitoring package size
    default: split_now
    why: the crate measures 7.84 MiB against an 8 MiB warning and 9 MiB failure
    answer: split_now
  b9_close_gate:
    ask: What macOS evidence closes bob-cli-b9?
    choices:
      ci_green: Close only after green macOS CI runs of the exact test
      evidence_note: Leave open with a coordination note if macOS proof is unavailable
    default: ci_green
    why: a macOS-only flake cannot be proven fixed from Linux runs alone
    answer: ci_green
phases:
- id: macos-completion-flake
  title: Re-check the macOS aliases completion flake after the deadline pin
  depends_on: []
  size: large
  description: 'macos-completion-flake: read bob-cli-b9 with audited reads, re-check
    it after f00854be (complete() now uses bob_completion_command with the 10000 ms
    test deadline), investigate complete_values empty-list path in tests/cli/aliases.rs,
    apply the same pin only if it shares the deadline path, prove with loaded Linux
    runs, obtain macOS CI evidence via gh, and close only per b9_close_gate = ci_green;
    otherwise leave a coordination note. Do not assume a Linux pass proves macOS.'
- id: mermaid-class
  title: Keep language-mermaid through web-clip extraction
  depends_on: []
  size: medium
  description: 'mermaid-class: read bob-cli-d5, fix scripts/web_clip/web_clip_adapter.py
    so Defuddle extraction preserves language-mermaid (tag/restore via data attribute
    or post-process), extend --self-test to cover the real capture path, verify source.md/code.json
    record kind mermaid, and close bob-cli-d5 with evidence.'
- id: vault-format-docs
  title: Correct the vault-format walk order and check the page
  depends_on: []
  size: small
  description: 'vault-format-docs: read bob-cli-d8, fix docs/reference/vault-format.md
    lines 171-172 to PRE → NEW → PROJECTS → PENDING → NEXT → RECURRING → TICKLER →
    ROTTEN → POST per src/native/freshness/state.rs Tier and docs/freshness.md, check
    nearby explanations, run just docs-check, and close bob-cli-d8. Docs only, no
    behavior or memory changes.'
- id: assets-split
  title: Split embedded assets before the package breaches its cap
  depends_on: []
  size: large
  description: 'assets-split: read bob-cli-ai, if assets_split = split_now move web-clip
    fonts, mermaid/defuddle vendor bundles, and large assets payloads into a bob-obsidian-assets
    crate keeping include_bytes access, keep just package-check green with headroom,
    and close bob-cli-ai; if defer, record size evidence and leave the bead open with
    a monitor note. Do not breach the 9 MiB failure cap.'
proposed_by: bbugyi200.athena.bob-cli-d7.land
proposed_with: muse/muse-spark-1.3-contributor
decided_by: auto
create_time: 2026-10-27 16:30:17
status: wip
bead_id: bob-cli-da
---

- **PROMPT:** [prompts/202610/backlog_batch_b2.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202610/backlog_batch_b2.md)
- **PARENT:** [202610/backlog_batch_a1.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/backlog_batch_a1.md)
- **BEAD:** [bob-cli-da](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-da/README.md)

# Backlog batch B2: macOS completion flake, Mermaid class, vault-format docs, and assets split

## Context and triage record

Batch A1 (bob-cli-d7) closed d6, 8q, 58, 5q, and 6c, repaired the manual-number
predicate, and marked `202610/backlog_batch_a1.md` done. On 2026-10-27 the coder listed
all 106 open task beads, skipped the 4 snoozed beads until they wake (bob-cli-6q,
bob-cli-aa, bob-cli-ah, bob-cli-cp), and read candidates with audited `sase bead read`:
b9, d4, d5, d8, d9, 5m, 5n, cv, ai, 8n, 30, and sase-1oh. No verified obsolete or
duplicate tasks were closed: d5, d8, ai, and b9 are all live and distinct. The sase-repo
hardening beads (5m, 5n, cv) stay out of this bob-cli epic; cv's second fix (preserve
the full dirty tree before relaunch reset) remains unconfirmed on sase master after
closed sase-1oh, and belongs in the sase project, not here. d4 needs the bob-mac-capture
repo and d9's runner file is not in this checkout, so both defer to later batches with
their repos opened first.

Why these four beads next, ranked per the chain rule (red gates and flakes first, then
correctness, then misleading docs, then features):

- **bob-cli-b9** (flake, large): macOS CI
  `aliases::completion_offers_group_members_status_flags_and_alias_options` returns `[]`
  under full-suite load; f00854be pins the same deadline path on athena but macOS proof
  is still missing.
- **bob-cli-d5** (bug, medium): web-clip Defuddle drops `language-mermaid`, so saved web
  sources catalog diagrams as plain code.
- **bob-cli-d8** (bug, small): `docs/reference/vault-format.md:171-172` teaches the
  retired REFERENCES tier order.
- **bob-cli-ai** (feature, large): `bob-obsidian` measures 7.84 MiB compressed against
  an 8 MiB warning and 9 MiB failure; about 0.2 MiB of headroom remains.

The four phases share no files and no ordering, so they run in parallel. Each phase
reads its own bead first with `sase bead read <id> -r "<why>"` and closes it with
`sase bead close <id> --note "<evidence>"`, never with `--force`.

## Phase macos-completion-flake (bob-cli-b9)

Read bob-cli-b9 and its coordination note from batch A1 (f00854be pins `complete_values`
through `bob_completion_command`). Investigate how
`complete_values(&["bob", "task", ""])` invokes the completion engine in
`tests/cli/aliases.rs` and whether a timeout or empty-stdout path can silently return
`[]`. Apply the deadline pin only if it shares the completion-suite path; do not change
production deadlines. Prove with repeated loaded Linux runs of the aliases module, then
obtain macOS CI evidence with `gh` (the exact test node on macos-latest, full suite).
Close bob-cli-b9 only when `b9_close_gate = ci_green` is satisfied; if macOS proof is
unavailable, leave the bead open with a coordination note recording the Linux evidence
and what macOS run is needed.

## Phase mermaid-class (bob-cli-d5)

Read bob-cli-d5. In `scripts/web_clip/web_clip_adapter.py`, the Defuddle 0.19.4
extraction in `run_capture` drops the `language-mermaid` class before `sanitize_html`
runs; the sanitizer allowlist widening alone cannot restore it. Tag each `pre`/`code`
language class in a `data-` attribute before Defuddle runs and restore it after
extraction (or post-process Defuddle's output the same way). Extend `--self-test` so the
real capture path (not just `sanitize_html`) keeps `language-mermaid`. Verify with a
fixture capture (`-H` with `BOB_WEB_CLIP_KEEP_WORKDIR=1`): `render/source.html` keeps
the class, the saved `source.md` uses a ```mermaid fence, and `code.json`records`kind:
"mermaid"`. Close bob-cli-d5 with the fixture and self-test evidence.

## Phase vault-format-docs (bob-cli-d8)

Read bob-cli-d8. Correct `docs/reference/vault-format.md:171-172` to the current
contract in `src/native/freshness/state.rs` Tier and `docs/freshness.md`: PRE → NEW →
PROJECTS → PENDING → NEXT → RECURRING → TICKLER → ROTTEN → POST. Check the surrounding
explanations on that page against the same contract and fix only what contradicts it.
Run `just docs-check`. No behavior changes, no memory changes. Close bob-cli-d8 with the
docs-check evidence.

## Phase assets-split (bob-cli-ai)

Read bob-cli-ai. If `assets_split = defer`, record the current
`tools/release/package_check.py` measurement on the bead and leave it open with a
monitor note; change nothing. If `split_now`, move the large embedded third-party assets
(`scripts/web_clip/fonts/`, `scripts/web_clip/vendor/mermaid.min.js`,
`scripts/web_clip/vendor/defuddle.full.js`, and the large `assets/` payloads) into a
separate `bob-obsidian-assets` crate that `bob-obsidian` depends on, preserving
`include_bytes!` access patterns and versioning per the repo's release conventions. Keep
`just package-check` green with comfortable headroom under the 8 MiB warning (hard
failure above 9 MiB). Close bob-cli-ai with the measurement before and after.

## Landing

1. Confirm that the beads this epic resolves are closed with evidence (or left open with
   the reviewer's declined decision or missing gate noted). Triage every phase's
   `PROPOSED FOLLOW-UP:` notes through `/sase_new_task`.
2. **Successor epic (required).** As the lander's final action, author and propose the
   next backlog epic with `/sase_plan`, following the same rules:
   - List every open task bead (`sase bead list -t task -n 0`). Read candidates with
     audited `sase bead read`. Close only verified duplicates or obsolete beads, with
     evidence notes: `-R superseded` for duplicates or replaced work, `done` for work
     already fixed. Re-check this plan's triage record rather than trusting it.
   - Choose the at most five most important remaining beads, in at most five phases,
     with explicit `depends_on` and unique slugs. One phase may work several beads. Rank
     red gates and flakes, then correctness and data-safety bugs, then memory that
     misleads agents, then Bryan-facing features.
   - Add plan decisions that let Bryan opt out of feature work or memory changes that
     may not be wanted. Feature beads get a yes/no toggle. Memory changes get
     `/sase_memory_write` memory decisions. Keep other decisions to a minimum.
   - Skip snoozed beads until they wake: bob-cli-ah 2026-11-04, bob-cli-aa 2026-11-06,
     bob-cli-cp 2026-11-16, and bob-cli-6q, whose four weeks of journal data arrive
     around 2026-11-09. Beads that need Bryan's hands get a gate, question, or decision
     in the plan, never a guess. Those are Mac acceptance runs (bob-cli-ay, bob-cli-cx),
     the Mac crontab (bob-cli-30), sudo installs (bob-cli-ax), approved write passes
     (bob-cli-cm, bob-cli-co), and the running-app UX review (bob-cli-20).
   - Copy this successor rule into the new plan's own Landing section, so each lander
     keeps the chain going until no open task beads remain.
3. Suggested starting order for the next batch, from this triage (re-verify first):
   - bob-cli-d4 (Mac CI flake, needs the bob-mac-capture repo opened first);
   - bob-cli-d9 (command-mode FIFO flake; locate its runner repo first);
   - the sase-runner beads bob-cli-5m, bob-cli-5n, bob-cli-cv (consider asking Bryan to
     move these to the sase project; check sase master before cv);
   - the small docs correction bob-cli-8n (vault AGENTS.md engine sentence);
   - the small memory corrections: bob-cli-ar, bob-cli-4o, bob-cli-7a, bob-cli-bu, and
     bob-cli-1x;
   - the decision-record memory backlog, then features.
