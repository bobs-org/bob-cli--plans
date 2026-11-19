---
tier: tale
title: Finish property-card registry and failure cleanup
goal:
  The shared kit honors registry removal and mount failure cleanup, passes acceptance,
  and closes bob-cli-nh.6.1 and parent phase bob-cli-nh.6.
size: medium
proposed_by: bbugyi200.athena.bob-cli-nh.6.1.land
proposed_with: codex/gpt-6-astra
bead: bob-cli-nh.6.1
create_time: 2026-11-19 17:34:17
status: wip
---

- **PARENT:**
  [202611/property_card_kit.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/property_card_kit.md)
- **BEAD:**
  [bob-cli-nh.6.1](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-nh/bob-cli-nh.6.1.md)

# Finish property-card registry and host failure cleanup, then close the kit

Complete only the remaining work found while landing **bob-cli-nh.6.1**, then close that
epic and its directly parented phase **bob-cli-nh.6** in this coding turn. This is one
bounded implementation, not another epic. There is no separate land agent after this
tale. Do not wait for this work's future commit SHA, push, or CI result before closeout;
the host commits the final code after the turn. Leave the containing intentions epic
**bob-cli-nh** open for its waiting lander.

## Context already verified

The lander read every note on bob-cli-nh.6.1.1, .2, and .3, their source and commits,
both plans, and the artifact-backed acceptance summaries. The kit's commits in linked
bob-plugins are ccfc12b, 8d6fd78, 946ef56, and 2f1e016. The reviewed source is
bob-plugins 2f1e016 and bob-cli 992fd33d. The original baseline is bob-plugins 36d4801
with bob-cli 5b4c2636. No production edits were made by the lander; this tale fixes the
two demonstrated gaps below.

Read the durable context through audited commands:

```bash
sase bead read bob-cli-nh.6.1 -r "Need landing findings and all follow-up outcomes"
sase bead read bob-cli-nh.6 -r "Need parent phase scope and final DECISIONS"
sase artifact read plan:202611/property_card_kit.md "Need kit acceptance and preserved contracts"
sase artifact read plan:202611/period_intentions.md "Need original kit scope"
sase artifact read file:explicit:dec33b788961cadeb2b4261a "Need reproducible registry and host failure probes"
```

The probe archive includes a loader, two appended test snippets, and complete output. It
loads each existing test module with an extra test without modifying the repository.
From the opened bob-plugins checkout, use:

```bash
node <unpacked>/nh61-probe.cjs scripts/test-ledger-tools-schema-double-equals.cjs <unpacked>/nh61-registry-probe.txt
node <unpacked>/nh61-probe.cjs scripts/test-ledger-tools-property-card-host.cjs <unpacked>/nh61-host-probe.txt
```

The unmodified registry suite passes 25/25; its added unregister test fails with one
bank open instead of zero. The unmodified host suite passes 12/12; both added
failure-cleanup tests fail with ownership still true. These are remaining epic work, not
unrelated follow-ups.

Existing acceptance is preserved in these audited artifacts:

- file:explicit:5366ba3f1c91e381b416eb0b: ownership acceptance and baseline;
  file:explicit:d19bd1571d8869c3fbd0f94c: original reports/screenshots bundle.
- file:explicit:b9fee3d94935c22cdac23dbb: host acceptance;
  file:explicit:e0c4d18220635127d479ffc0: reports and host regression evidence.
- file:explicit:57b7adc6f806ce185b185519: final chrome acceptance;
  file:explicit:a50033287bd8564938d9063a: full reports, 24 exact pixel pairs,
  screenshots, source provenance, clean-base bank-setup repro, and deploy hashes.

The final prior CDP results were record 144/144, album 120/121, bank 75/78, and bank
setup 46/47 plus an arrangement-save timeout. All reported nonpassing cases match
audited clean-base failures. The chrome matrix is 24/24 exact. Lander independently
reran the copy-card model suite (59/59) and plugin validate (12/12, including bundle
freshness and docs inventory). There is no `symvision` recipe in the reviewed bob-cli
justfile; check availability again at closeout. Both nh.6.1 and nh.6 had no epic-symbol
entries.

## Fixed scope and integration

Open the implementation checkout with `/sase_repo`:

```bash
sase repo open bob-plugins -r "Finish property-card kit landing repairs"
```

Use only its returned path; read its AGENTS.md, architecture, validation, and deployment
instructions. Edit source fragments and rebuild generated main.js. Keep each fragment
within 1,000 lines. Read the docs reference memory before editing documentation. No new
CLI, card species, fields, palette names, command IDs, or memory edits. The intention
card remains bob-cli-nh.7 work. Parent DECISIONS remain day_summary_cap=sixty, the
specifically authorized glossary work in its own phase, and no declined decision-record
writes.

Since kit start, plugins gained copy chips (0b38f84, 76fa3f9, febfc6b, 0a4a91f, ccc0f72,
d6d2c33), successor-picker marks (e2cdc8d), and log recall (795fad3, 3968f0d). These are
not property cards. Log recall's lifecycle, mixin, exports, fragment and package
additions coexist with the kit and must survive. d6d2c33 fixed the old copy-card
row-order proposal. CLI changes since 5b4c2636 add intention readers/writers/link
repair, periods brief/problems, log recall/completion, and bank import
operands/deletion; no new property-card host or competing ownership path exists there.
Recheck drift since the reviewed heads and integrate actual conflicts without expanding
into those features.

## 1. Make the registry the sole dispatch authority

In `plugins/bob-ledger-tools/src/248m-plugin-schema-double-equals.js`:

- `schemaDoubleEqualsTargetKind` is still the original closed bank/album/record switch,
  despite the documented registry facade. Eliminate its separate predicate
  implementation. Preserve existing callers and built-in results through a facade over
  the shared matcher/entry implementation, and allow explicit instance registry context
  for extension resolution.
- `handleSchemaDoubleEqualsKeydown` currently handles a registry miss by calling that
  closed helper and `ensurePropertyCardBuiltins`. Thus unregistering bank succeeds until
  the next `==` on bank.md, which silently re-registers and opens it. Initialize
  built-ins at setup; a runtime miss must not mutate the registry or consume Vim input.
  Explicit new setup may restore built-ins as before.
- Remove the module-global `propertyCardExtraKinds` mutable authority. Registered
  extension kinds and their removal/disposal belong to the plugin instance; one instance
  must not affect another instance's gate. Keep the pure gate's legacy built-in inputs
  compatible, but supply the actual registry explicitly for extension tests/runtime. Do
  not retain a global list merely to preserve an implementation-specific test setup;
  preserve its semantic assertions.
- Keep bank → album → canonical record order, exact open arguments, all refusal
  reasons/order, capture-phase listeners, WeakSet single consumption, and Esc
  cancellation. Unsupported paths and throwing/no-match registries retain Vim.

Extend the existing test file with runtime unregister for a built-in, a fourth kind
resolved through the facade and dispatcher, two-instance isolation with the same
extension kind, unregister/teardown/re-setup, and once-only consumption. The archived
failing probe must pass. Do not weaken existing behavior assertions.

## 2. Enforce failure cleanup in the shared host

In `240c-plugin-property-card-host.js`, `refreshReading` catches a throwing
`paintReading` without invoking removal or release. In
`240d-plugin-property-card-host-editor.js`, constructor/update builds call
`buildDecorations` without failure cleanup; an update can throw while its old ownership
stays live. `commitMount` is currently used only by a unit test, not by production
adapter mounting. The documented host guarantee is therefore not enforced by the shared
lifecycle.

Put failure handling at the shared host boundary and connect the existing album, record,
and bank callbacks to it. The host must remove the failed Reading mount and release only
that mount's generation/path-bound ownership; failed editor construction/rebuild must
clear stale decorations and release the corresponding ownership without losing the
ability to recover on refresh. Null/failed mounts, thrown model/build/paint callbacks,
removal, and disposal must expose native Properties. A subsequent successful refresh
must recover. Use the existing adapter removal/cleanup hooks where they already own the
generation-bound handle; do not use a blanket scope release that revives the stale-owner
regression fixed by 8d6fd78. Clarify return/success contracts and either wire the shared
mount helper into production or remove a misleading unused helper. Document the actual
final contract concisely.

Keep species model/DOM/anchor/caches explicit. In particular preserve record embedded
display without hide ownership, frontmatter-only fallback, render reuse, walk rail,
subscription ownership, selection handling, and the workspace-event override that avoids
full refresh on file-open. Preserve bank Source-mode lint and album focus/expansion. Do
not normalize CSS or rewrite the already verified chrome.

Add failure-injection regressions at real host callbacks, including a successful mount
followed by a throwing/null repaint, failed editor construction/update, recovery,
teardown, and a stale callback versus a newer mount in the same scope. Use actual owned
native rows in adapter coverage, so assertions prove rows are untagged rather than just
counting callback invocations. Exercise all three adapters and two panes without
changing the current record/embed assertions.

## 3. Verify and deploy the repaired tree

Run focused registry, ownership, host, album, record/card/rail, and bank suites. Build
generated sources, run `npm test`, `npm run validate`, and the mounted
`BOB_CLI_DIR=<current checkout> npm run docs:check`. Use `just check` for bob-cli
file-change verification; never run `just check-full`. Preserve full logs and report
known unrelated failures honestly; only identical evidenced clean-base failures qualify
under the existing kit acceptance policy.

Rerun real album, full-scope record, and bank acceptance on athena using a
checkout-built Bob and provenance per validation.md. Use isolated scratch
vaults/profiles/ports and a short TMPDIR. Hold binary and fixtures fixed for
before/after comparisons against 2f1e016 when diagnosing any changed result. Run the
24-case actual-DOM chrome matrix against original 36d4801 and inspect the screenshots.
The bank setup baseline issues are already tracked; preserve their assertions and
recorded outcomes. Fresh regressions from these repairs must be fixed, not triaged away.
Use `/sase_monitor` before long builds/suites; wait for the staging command to exit,
then finalize so its successor can finish this same tale. Never end with an unmanaged
subprocess running.

After verification, `bob plugins sync` from the explicitly opened source as required by
that repo. Follow deployment conflict recovery without `-F` and verify deployed managed
hashes. Publish reports/provenance/screenshots through audited artifacts and cite them
in the epic verification note. Review any post-start drift before the final
verification/deploy. No closeout step depends on the future host commit, its SHA, push,
or CI.

## 4. Finish this landing in the coding turn

The lander triaged every child proposal and recorded the complete dispositions on
bob-cli-nh.6.1. Carry these into the close note; do not create duplicates or ask another
lander to finish:

- nh.6.1.1 notes 3/4 (real acceptance and deployment): declined as new tasks because its
  notes 5/10 and final chrome evidence complete both requirements.
- nh.6.1.1 note 6 (long TMPDIR): new small CI task **bob-cli-np**, related to closed
  web-clip-only fix 5b; that fixed adapter task is not reopened.
- nh.6.1.1 note 7 (album Ctrl+J): new large CI task **bob-cli-nq**; cause needs
  diagnosis, and the failure is identical on the original pre-kit baseline.
- nh.6.1.1 note 8: corroborated **bob-cli-m6** and **bob-cli-m7**, with phase evidence
  attribution; no duplicate bank test tasks.
- nh.6.1.1 note 9 and nh.6.1.2 notes 2/5: event-link timing corroborated **bob-cli-km**;
  dependency-stage timing corroborated **bob-cli-m1**. The broader ki similarity is not
  a reason to file another task for these nodes.
- nh.6.1.2 note 4 and nh.6.1.3 note 1 (copy-card row order): declined as already fixed
  by d6d2c33; current isolated model suite passes 59/59.
- nh.6.1.2 note 5 (record Markdown-neutrality race): DISCOVERED ISSUE on active
  **bob-cli-ms.8**, whose child .8.3.1.2 introduced that acceptance runner. No unrelated
  task; preserve the byte-neutrality assertion.
- nh.6.1.3 note 2: separate large CI tasks **bob-cli-nr** (bank Ctrl+K) and
  **bob-cli-ns** (one-Enter arrangement save); each relates to **bob-cli-mk** coverage
  work. They are not established to share a root cause. Both have clean-base 946ef56
  evidence and need diagnosis.
- nh.6.1.3 notes 2/3 (unchanged Mac reference word budgets): corroborated existing
  reopened **bob-cli-jz**, not this kit's remaining work.

All corroborations explicitly attribute the child evidence; the lander did not claim
fresh CDP or full-suite timing failures. If genuinely new unrelated work appears, use
`/sase_new_task` and record its outcome before closing.

Re-read all three child beads and notes, epic readiness, and the linked plan to confirm
these two gaps are resolved and no new work remains. Then:

1. Run `sase bead epic-symbols bob-cli-nh.6.1`. Resolve each entry by wiring,
   privatizing, a justified non-test pragma, or deletion; re-key only when a still-open
   later bead actually needs the exemption. Leave none keyed to a bead about to close.
2. Close normally with
   `sase bead close bob-cli-nh.6.1 --note "<source and commit audit, fixed registry/host gaps, integration review, test/visual/deploy evidence, and every follow-up disposition>"`.
   Never use force to make a successful nested landing advance. If symbols reject the
   close, clean them and retry. If named phases are incomplete, finish them; do not
   force them done.
3. Run `just symvision` if the recipe is available and confirm the whitelist is clean.
   If absent, record that fact rather than claiming a pass. Open the plans repo via
   `/sase_repo` and set `status: done` in the frontmatter of the PLAN path for
   `plan:202611/property_card_kit.md` shown by `sase bead read`.
4. Read `sase bead read bob-cli-nh.6.1 -r "Need the parent link"`. The parent is phase
   **bob-cli-nh.6**. Recheck its kit scope against both plans and final results; leave a
   concise API/evidence handoff for nh.7. Run `sase bead epic-symbols bob-cli-nh.6`,
   resolve/re-key any entries, and close only that phase normally with
   `sase bead close bob-cli-nh.6 --note "<verified shared ownership, registry, host/chrome, preserved behavior, acceptance/deployment, and nh.7 extension handoff>"`.
   Run available symvision again if needed after symbol changes. Leave **bob-cli-nh**
   and `plan:202611/period_intentions.md` open for their lander. If nh.6 is incomplete
   or ambiguous, record the concrete blocker on it and report it; never force a
   successful nested landing.
5. Mark this tale complete through its normal workflow, and finish with `/sase_final`,
   declaring every changed repository for host commit. Describe the real completion or
   blocker in the final response.
