---
tier: tale
title: Omit the active group from the GTD review footer summary
goal:
  Show the current review group once in Obsidian's footer while preserving other group
  counts and review behavior.
size: small
proposed_by: bbugyi200.apollo.5g
status: done
---

- **AGENTS:**
  - [bbugyi200.apollo.5g](https://github.com/bobs-org/bob-cli--agents/blob/main/sessions/bbugyi200.apollo.5g.md)
- **COMMITS:**
  - [69006fd](https://github.com/bobs-org/bob-plugins/commit/69006fd07d3dfb82bda8108d84a860de71519692)
    — feat(ledger-tools): omit active group from review footer summary

# Omit the active group from the GTD review footer summary

## Outcome

When Obsidian's desktop morning-review footer shows a current review group, omit that
group's redundant count from the trailing group summary. Apply this to every group,
using the existing cursor-based definition of the current row.

The user's screenshot shows:

```text
⟳ Review 1/94 · REFERENCES 1/1 · Reference · REFERENCES 1 · ROTTEN 92 · POST 1 · ✓ 4/20 today
```

The intended result is:

```text
⟳ Review 1/94 · REFERENCES 1/1 · Reference · ROTTEN 92 · POST 1 · ✓ 4/20 today
```

This is one small tale: one coding agent can complete the focused presentation change,
update the existing expectation and documentation, build, validate, and deploy the
affected plugin.

## Findings and constraints

Implementation lives in the linked `bob-plugins` repository. Open it using the
`/sase_repo` skill with
`sase repo open bob-plugins -r "Implement active review group omission in the footer"`,
and use the returned checkout. Read its `AGENTS.md`. All plugin paths below are relative
to that repository; the one `docs/freshness.md` path explicitly belongs to bob-cli.

- `plugins/bob-ledger-tools/src/120-freshness-footer.js` contains `freshnessFooterView`.
  It currently constructs `groups` and `groupsText` from all nonempty tiers before
  resolving `current`, then renders both `current.label + tierRank/tierTotal` and the
  unchanged group list. This is the cause of the duplication.
- `freshnessFooterGroups(counts)` supplies the full ordered histogram. Keep that
  helper's contract. Apply omission in the view projection only.
- `plugins/bob-ledger-tools/src/240-plugin-freshness-status.js` resolves a verified
  cursor entry, supplies it to the view, and includes `groupsText` in its paint key.
  Cursor-only updates reuse the warm memo.
- `freshnessFooterPaint` and `freshnessFooterFit` already omit an empty groups span; the
  stylesheet excludes omitted spans from separator generation.
- `groupsText` also feeds the tooltip and accessible button label. Those surfaces can
  share the filtered summary because the current group remains represented by the
  context text.
- The audited `decisions:review-walk-is-tiered` record governs queue semantics. Preserve
  its nine-tier order, all queue counts and ranks, commitment and upkeep calculations,
  visibility rules, and navigation behavior. No evaluator, public API version,
  vault-note, configuration, or memory change is needed.
- Baseline: `node --test scripts/test-ledger-tools-freshness-footer.cjs` passes all 16
  tests. Its PRE/POST case currently expects `PRE 1 · POST 1` while PRE is active; that
  expectation must change.

## Implementation

1. In `freshnessFooterView`, resolve `current` through the existing validation and
   normalization first. Then obtain the ordered groups and filter out the group whose
   machine `key` equals `current.tier`, if `current` exists. Build `groupsText` from
   that filtered array and return the same array as `groups`. Compare machine keys,
   never display labels, task states, or text prefixes. If current resolution fails or
   the cursor is outside the queue, retain all nonempty groups. Derive fresh arrays
   without mutating counts or queue rows.
2. Let the existing rendering, tooltip, accessible label, and width-fitting code consume
   this view. An empty filtered summary hides only its span; a nonempty queue still
   displays the current position, group, and meter. Keep `freshnessFooterGroups` and the
   shared entry presentation unchanged.
3. Update the existing PRE/POST test's summary expectation to `POST 1`. Use the existing
   suites and focused read-only probes for verification; this display change does not
   need a new test harness or framework.
4. Update the footer paragraph in `bob-plugins/README.md` and the desktop footer
   paragraph in bob-cli's `docs/freshness.md` to describe omission of the current group
   and the full summary when no review row is current. Bump the ledger-tools manifest
   patch version (currently `1.29.2`) and update its README version row consistently.
   Run `npm run build` to regenerate `plugins/bob-ledger-tools/main.js`; never hand-edit
   that bundle. Keep edited source fragments within the repository's 1000-line limit.

## Verification and acceptance

Use the existing footer harness or temporary read-only probes against its exported
helpers to check these behaviors:

| Situation                                                                            | Required result                                                                                                          |
| ------------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------ |
| Screenshot counts, REFERENCES active                                                 | Context is `REFERENCES 1/1`; summary is exactly `ROTTEN 92 · POST 1`; total remains 94.                                  |
| Each of PRE, NEW, PROJECTS, PENDING, NEXT, RETURNED, REFERENCES, ROTTEN, POST active | Only the matching summary group is absent; other nonempty groups retain their order and counts.                          |
| A group's task state differs from its walk tier, or its label is customized          | Omission follows the resolved machine tier.                                                                              |
| No current row or current input rejected by existing validation                      | All nonempty groups appear.                                                                                              |
| Only the active group remains                                                        | Empty `groups`/`groupsText`; groups span is omitted without a separator; footer remains visible.                         |
| Cursor moves between groups and then off a review row                                | Newly active group disappears and the previous group returns; the full summary returns off-row, using the existing memo. |

Also inspect tooltip and aria text: the current-group context remains present and the
redundant standalone group count is absent. Confirm an empty queue still hides the
entire footer and narrow-width fitting still works.

After building, run `npm test` and `npm run validate` in bob-plugins. These include
generated-bundle checks and the existing footer, queue, cursor, navigation, and
lifecycle coverage. Inspect the final diff for only the intended source, regenerated
bundle, existing expectation, version, and docs changes. No Rust behavior or build
change is involved.

## Deployment

After validation, satisfy bob-plugins' required vault sync using the exact opened
checkout as `<opened-bob-plugins-path>`:

```sh
bob plugins sync --no-pull --repo <opened-bob-plugins-path> --plugin bob-ledger-tools --dry-run
bob plugins sync --no-pull --repo <opened-bob-plugins-path> --plugin bob-ledger-tools
```

Inspect sync output for copied or unchanged files and any skipped files; exit zero alone
does not prove deployment. Report the actual sync result. If a live Obsidian session is
available, reload the plugin and check an active review row, a group transition, and an
off-row cursor. Otherwise report that live visual verification remains unperformed and
that Obsidian must load the updated plugin to display the change. Do not claim a live UI
check from the automated suite alone.
