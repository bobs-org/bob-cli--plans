---
tier: tale
title: Finish the Mac Capture documentation landing
goal:
  Repair the remaining documentation integration defects, verify the finished work, and
  close bob-cli-a0.11.1 and parent phase bob-cli-a0.11.
size: medium
proposed_by: bbugyi200.apollo.bob-cli-a0.11.1.land
proposed_with: codex/gpt-6-astra
bead: bob-cli-a0.11.1
status: done
---

- **PARENT:**
  [202610/mac_capture_reference.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/mac_capture_reference.md)
- **BEAD:**
  [bob-cli-a0.11.1](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-a0/bob-cli-a0.11.1.md)
- **AGENTS:**
  - [bbugyi200.apollo.bob-cli-a0.11.1.land](https://github.com/bobs-org/bob-cli--agents/blob/main/sessions/bbugyi200.apollo.bob-cli-a0.11.1.land.md)
- **COMMITS:**
  - [b9859d9](https://github.com/bobs-org/bob-cli--plans/commit/b9859d96d0109d66e13def962c1df57bff396781)
    — chore(plans): mark mac_capture_reference done

# Finish the Mac Capture documentation landing

Complete the remaining documentation repairs and land epic `bob-cli-a0.11.1`, then close
its parent phase `bob-cli-a0.11`. One agent can make these bounded changes and finish
the landing in the same turn. Do not create another epic or leave a separate landing
step for a future agent.

## Context and completed review

Use `/sase_repo` to open `bob-mac-capture`, read its `AGENTS.md`, and use only the
printed checkout path. Read these through audited commands:

- `sase bead read bob-cli-a0.11.1 -r "Need landing audit and follow-up dispositions"`
- Each of `bob-cli-a0.11.1.1` through `bob-cli-a0.11.1.4`, including every note.
- `sase bead read bob-cli-a0.11 -r "Need the parent phase scope and decisions"`
- `sase artifact read plan:202610/mac_capture_reference.md "Need original epic acceptance criteria"`
- `sase artifact read plan:202610/bob_public_docs_site.md "Need mac-ref requirements and selected decisions"`

Use `/sase_memory_read` for `docs.md` and `sase_beads.md`; consult the tailnet memory
before remote verification. Read any additional domain memory required by the repository
instructions. There are no memory changes in this tale.

The land audit is recorded on `bob-cli-a0.11.1`. All four original phases are closed,
and their section mappings and subsequent verification notes have been reviewed. The
epic implemented reference and developer mounts, 20 Mac Markdown pages, six synthetic
panel images with provenance, the media importer and docs wrapper, five recovery guides,
and the README front door. All 13 fenced examples from Mac README baseline `065908f`
remain in the current documentation. Of 125 long baseline paragraphs, 100 remain
verbatim; the others were redistributed or rewritten into the documented destinations.
Preserve this coverage.

Reviewed epic commits:

- bob-cli: `1f3b90e5`, `8ba29959`, `d4b1ab31`, `8ced873d`.
- bob-mac-capture: `1fff6aa`, `3b944d8`, `bd9b1e2`, `48c871a`.

Reviewed concurrent changes:

- bob-cli `110624b8`: reusable publication runner declarations and GitHub JSON color
  handling. No conflicting mount change. Live read-only run/artifact queries through the
  Mac importer parsed successfully with ordinary settings and with `FORCE_COLOR=1` and
  `NO_COLOR` removed. Do not add speculative importer changes.
- Mac `b0282a8`: Refs row identity repair on cross-section reranking; compatible with
  the documented ranking contract.
- Mac `ddea5df`: adds a Docs block that now duplicates the epic's README block. This
  needs integration below.

Fresh `just docs-test` passed all eight script tests during the audit. The previous
phase notes record full docs gates, both strict builders, and the media phase's 32
browser image cases. Final checks after this tale's changes are still required. No
`--epic-symbol` entries were present at audit time; neither repository currently exposes
a `symvision` recipe. Recheck at closeout.

The parent decisions remain MIT, search indexing enabled, and published design
decisions. Keep the existing source ownership: bob-cli owns its two Mac hub landings;
Mac owns the mounted reference/developer content. Preserve published paths, the literal
README `## Signing`, synthetic media, thin-client behavior, and all permission and
destructive-action caveats.

## 1. Integrate the README and repair reference pointers

In bob-mac-capture:

1. Consolidate the two `## Docs →` blocks and repeated site URL in `README.md` into one
   useful front-door section. Preserve Get started, Reference, Develop, Guides,
   major-topic links, install instructions, MIT information, and the exact `## Signing`
   heading used by `BundleSigningInspector`.
2. Resolve phase `bob-cli-a0.11.1.4` note #2:
   - `Sources/RefsCore/RefsRanking.swift` has two comments pointing to Sorting in the
     removed README Bob Refs section. Point them to `docs/refs.md#sorting` or its
     canonical published URL.
   - `Sources/BobMacCapture/Refs/RefsKeyRouter.swift` describes itself as the README
     keyboard table. Point it to the new `docs/keyboard.md` reference.
   - Leave `BundleSigningInspector`'s README Signing reference intact.
   - These comment-only repairs are the narrow integration exception to the original
     plan's prohibition on Swift file edits, required by the landing instruction to
     resolve issues caused by this migration. Change no executable Swift tokens and do
     not reformat surrounding files. Follow the Mac compile requirements even for these
     source-file changes.

In bob-cli, replace the final sentence of `docs/develop/mac-capture/index.md` that
describes what may change "in this epic" with durable contributor guidance about source
ownership and `just docs-check`. Keep agent workflow and temporary landing restrictions
out of public pages.

Follow-up triage is complete: `/sase_new_task` searches across task types and statuses,
the one-week task sweep, and active-epic inspection found no duplicate for the stale
comments. Their cause is this epic's README split, so no standalone task was created.
The proposal is accepted here, not postponed or declined. Copy that disposition into the
eventual close note.

## 2. Correct the migrated recovery and privacy claims

Make concise, source-verified documentation changes without changing application
behavior:

1. In bob-cli `docs/guides/fix-mac-capture-bob-resolution.md`, correct the **If it goes
   wrong** claim that an unsigned `bob` causes `Not resolved`.
   `Sources/CaptureCore/BobExecutableResolver.swift` checks that an override is absolute
   and executable, or finds an executable in its four default paths; it does not inspect
   code signatures. Explain those actual failure causes. Keep platform signing
   requirements distinct from resolver diagnosis.
2. In bob-cli `docs/guides/recover-mac-capture.md`, distinguish retained closing from
   Control-C cancellation. The latter calls `stashDraftAndClose`, which clears the draft
   even when stashing is disabled; `CanceledDraftStash.push` returns nil at capacity
   zero. Task-Go-To cancellation also deliberately discards without stashing. Explain
   when a draft can be recovered, link to Settings/Canceled drafts, and avoid promising
   recovery for those exceptions. Align the corresponding Submit/Stash summary in Mac
   `docs/capture-panel.md` and any directly contradictory keyboard summary. Preserve
   existing permanent Discard, Delete All, and capacity/overflow rules.
3. Reconcile Mac `docs/permissions.md` with `docs/settings.md` and
   `Sources/BobMacCapture/NotificationService.swift`. The privacy paragraph says
   captured text is never included in notification bodies, but `singleCaptureBody` uses
   `semanticText` from the capture response and the settings page already documents that
   behavior. Describe raw draft/argument redaction versus semantic task/note text in
   success notifications accurately. Preserve the guarantees for logs, signposts,
   Diagnostics, UserDefaults, failure messages, stash files, and scan notifications
   containing reference titles. Link to the existing notification-copy reference instead
   of copying its detailed catalog.

These repairs complete the accuracy and consistency of the extracted pages; they do not
authorize runtime, protocol, installer, permission, or vault edits. No separate
unrelated follow-up was identified in this audit.

## 3. Verify the finished change and recheck drift

- Review the final diff for comment-only Swift changes, preserved Signing, exactly one
  README Docs block, correct relative/absolute links, and coherent retention and privacy
  statements. Do not add tests that merely snapshot prose.
- Re-read any commits that landed after the reviewed heads above in both repositories,
  including fetched base-branch changes. Integrate anything that affects these repairs
  or the Mac mounts. Do not overwrite unrelated work.
- Use `/sase_repo` to open `bob-plugins` if needed by the combined docs assembly; read
  its instructions and do not edit it. Supply explicit source paths to authorized
  checkouts when running the docs gates.
- Run bob-cli `just check`, never `just check-full`. The latter is not authorized. Run
  `just docs-check` with the Mac source override, the Mac
  `BOB_CLI_DIR=<authorized bob-cli checkout> just docs-check` wrapper, and bob-cli
  `just docs-build-material` for the strict fallback. Inspect the touched rendered pages
  and README links. The media files are unchanged; repeat image checks only if new
  evidence or markup changes justify them.
- Follow Mac `AGENTS.md`: on a host without Xcode, run `just portable-check`; if Swift
  is absent, use the documented Swift-capable tailnet host, athena. Because a comment in
  `Sources/BobMacCapture` changes, make the prescribed best-effort scratch-copy compile
  on the Mac with `Scripts/bundle.sh` and `just typecheck-tests`, then delete that
  scratch copy. Do not use Bryan's checkout or install into his running app. If the Mac
  is offline, report the macOS-target verification limitation explicitly; do not claim
  it passed. Never skip the required compilation or plan to wait for this tale's own CI.
- Use `/sase_monitor` for long verification commands with a continuation that completes
  the closeout below. Distinguish unrelated baseline failures from regressions,
  recording evidence and following `/sase_new_task` when an actual unrelated issue needs
  triage. Fix regressions before closing.

## 4. Finish this epic and its parent phase in the same landing

This is the tale's final step and belongs to its coder. Do not stop after the edits or
wait for this work's commit SHA, push, or CI: the host creates the commit after the
turn. Closeout occurs against the verified working tree.

1. Re-read `bob-cli-a0.11.1` and its descendants. Confirm all original phases remain
   complete and any new linked work is complete. If this tale has its own open
   descendant bead, close that completed bead normally first so it cannot block the
   epic. Never use force to advance a successful landing.
2. Run `sase bead epic-symbols bob-cli-a0.11.1`. For every returned Justfile exemption,
   resolve the symbol by wiring it, privatizing it, adding a valid non-test pragma, or
   deleting it according to the Symvision policy; re-key only if an actually open later
   bead still needs that exemption. None may remain keyed to this epic or a closing
   phase.
3. Close normally:

   ```bash
   sase bead close bob-cli-a0.11.1 --note "<specific source/commit and migration verification, concurrent-change integration, final test results, comment-proposal disposition, and any verification limitation>"
   ```

   Include the prior audit plus the actual results from this tale. If rejected for
   symbols, finish cleanup and retry. If rejected for unfinished phases, finish or
   reopen the work. Never force merely to make close succeed. Force with an explicit
   reason and canceled/superseded resolution is only for a deliberate abandonment, never
   this successful nested landing.

4. After the epic closes, run `just symvision` if available. If no recipe exists, record
   that explicitly; do not add one or claim a pass.
5. Open the plans sidecar with `/sase_repo` before modifying it. The epic's audited plan
   is `plan:202610/mac_capture_reference.md`; its workspace-relative PLAN path is
   `sase/repos/plans/202610/mac_capture_reference.md`. In the authorized plans checkout,
   set `status: done` in `202610/mac_capture_reference.md` frontmatter, preserving the
   other fields.
6. Run `sase bead read bob-cli-a0.11.1 -r "Need the parent link"`. Its expected parent
   is the **phase** `bob-cli-a0.11`. Re-read that phase, confirm its child epic
   fulfilled the mac-ref requirements and all its descendants are closed, run
   `sase bead epic-symbols bob-cli-a0.11`, and resolve/re-key any leftovers by the same
   rules. Then close only that parent phase normally:

   ```bash
   sase bead close bob-cli-a0.11 --note "<verified README coverage, reference/developer mounts, media provenance, recovery guides, integration repairs, and final checks>"
   ```

   Leave containing epic `bob-cli-a0` and its plan status to its waiting land agent. If
   the parent phase is incomplete or its link differs unexpectedly, record a precise
   blocker on that parent and report it instead of forcing.

7. Use `/sase_final` as the final action before a normal final response. Include every
   modified repository, including the plan sidecar, in the declaration. Report which
   beads closed and the real verification results.
