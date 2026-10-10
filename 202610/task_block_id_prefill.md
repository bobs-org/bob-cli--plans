---
tier: tale
title: Prefill Obsidian task block-ID prompts with useful names
goal: Every task block-ID assignment prompt offers a meaningful, editable, collision-aware
  default without writing before confirmation.
size: medium
proposed_by: bbugyi200.apollo.67
status: done
---

# Prefill Obsidian task block-ID prompts with useful names

## Outcome and scope

Every Bob-owned Obsidian prompt that assigns a task a missing block ID opens ready to
accept a meaningful, valid, available suggestion. Enter accepts it; typing replaces the
selected suggestion. Merely opening or cancelling a prompt never assigns an ID.
Explicitly prefilled existing-ID rename prompts retain their current ID; previously
empty marker-triggered rename prompts for task blocks can suggest a name from the target
task as well.

Implement this as one medium tale: two cooperating prompt implementations, one small
deterministic naming contract, focused runtime tests, documentation, and plugin
deployment. No phase graph or independent implementation agents are needed.

Implementation is primarily in the linked `bob-plugins` repository. Open it with
`sase repo open bob-plugins -r "Implement approved task block-ID prefill plan"` and use
only the returned checkout. Read its AGENTS.md. Edit source fragments, keep each below
1000 lines, register new fragments in `src/fragments.json`, and generate `main.js` with
`npm run build`. Update the relevant documentation in bob-cli.

This is prompt default behavior, not automatic assignment or an ID migration. Keep
task/link/dependency writes, lane changes, review-walk settlement, and undo contracts
intact. No new CLI interface, service, model call, settings screen, or Mac Capture
change is required. Preserve the pinned Rust/cycler successor minting contract and
project conversion's automatic IDs.

## Findings and implementation map

Paths in this table are relative to the linked bob-plugins checkout.

| Surface                                                                                                  | Current implementation                                                                                                       | Required result                                                                                   |
| -------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------- |
| Ctrl+6 / selected-block command on an ID-less task                                                       | `block-id-prompt/src/050-source-discovery-and-edits.js` emits `direct-add`; `090-prompt-modals.js` starts blank              | Seed from that task's headline and its note                                                       |
| `^^` task picker, including same-note and cross-note wiki-link variants                                  | `100-plugin-lifecycle.js::selectTaskLinkTask` emits `link-task-complete` with `task.rawLine` and `prefillId: false`          | Seed from the selected target, not the source link or its alias                                   |
| Ctrl+Shift+Enter on an ID-less task                                                                      | `120-plugin-pomodoro-links.js::openPomodoroTaskLink` chooses a Pomodoro, then opens the blank `link-task-pomodoro` modal     | Seed after the Pomodoro choice; preserve the later inbox-route gate                               |
| Task Card Depends on, direct dependency entry, Task Link entry, single/count/batch, same-note/cross-note | `bob-navigation-hotkeys/src/400-picker-local-task.js::showBlockIdStage` uses a simple slug or any existing `[id::]` verbatim | Use the same naming contract and only a valid, available legacy ID as a preferred value           |
| Rename with an explicit current-ID prefill                                                               | `direct-rename` and `link-block` sources set `prefillId: true`                                                               | Preserve the selected current ID and the existing rename/backlink writer                          |
| Marker-triggered task rename (`[[note#^old]]@`, `[[note#^^old]]`)                                        | `020-markers-and-references.js` supplies `oldId` but no prefill flag                                                         | Seed a readable suggestion from the uniquely resolved target task; retain normal rename semantics |

The general modal currently checks only `source.prefillId && source.oldId` before
setting its input. Its preview load is asynchronous. All its entry points converge on
`100-plugin-lifecycle.js::openBlockIdPrompt`; use that convergence instead of
independent naming logic per gesture.

The navigation helper `020-config-load-and-tasks.js::suggestBlockIdFromTask` also serves
project conversion in `120-project-notes.js`. Do not silently change those automatic IDs
by replacing this helper globally. Introduce a prompt-specific helper and migrate the
prompt call site. Navigation's existing raw task snapshots, target-note snapshots,
`validateBlockIdCandidate`, and batch reservations are the integration points.

Existing inspiration is bob-cli's `src/native/capture_block_ids.rs` and the pure JS port
in `task-status-cycler/src/086-successor-ids.js`: phrase-aware readable names, stopword
removal, and deterministic fallback. Their exact successor behavior is pinned by
`docs/task-dependencies.md` section 11.7 and must not change. The prompt policy below
deliberately handles readable Markdown and unlimited collision suffixes without changing
those automatic writers.

## Naming contract

Implement a pure function over task text, the target note's occupied IDs, and pending
reserved IDs. It must not read files, mutate a task, or inspect other plugins at
runtime. Follow the repository's copy-small-helpers convention: small equivalent
fragments in the two prompt-owning plugins, exercised by one shared fixture table. Avoid
a new bundler or a runtime dependency on task-status-cycler.

1. **Use the actual task headline.** Start with `task.rawLine`, or the verified first
   task line of the selected block. Strip the checkbox/list/blockquote prefix, trailing
   block ID, Dataview inline fields, recognized Tasks emoji date/recurrence/priority
   metadata, and standalone tags outside code/link labels. Do not use UI prefixes such
   as reference-task decorations, `(untitled task)`, a truncated display label,
   descendants, Work Logs, Schedule Logs, or Depends-On children. Recognizers must
   preserve semantic numbers such as issue `#123` and technical text inside code spans.
   Only the suggestion is cleaned; source text is never rewritten by this step.
2. **Prefer an explicitly named subject.** Scan the cleaned headline in text order for
   inline code, straight/curly double-quoted text, parenthetical text, wikilinks, and
   Markdown links. Use the first such phrase with meaningful words, taking up to four
   non-stopwords. Wikilinks use the visible alias, otherwise the target basename without
   its heading/block suffix; Markdown links use the visible label, never the URL. Treat
   each link as one construct rather than mistaking its URL parentheses for a named
   phrase. Skip empty/stopword-only spans. Bare URLs and syntax tokens do not become
   names.
3. **Otherwise keep the task's action.** Use the first three meaningful prose words,
   dropping the existing capture STOPWORDS set. Retain action verbs, since
   `fix-apollo-machine` and `review-apollo-machine` convey different tasks. Use visible
   link text in this prose projection too. No inference from note names, sections,
   dates, priority, or task status is needed for a note-scoped ID.
4. **Produce a compact slug.** Normalize Latin accents with Unicode NFKD and remove
   combining marks; tokenize ASCII letters/digits, lowercase, and join with hyphens.
   Punctuation and underscores separate words, never glue them. Generate only
   `[a-z0-9]+(?:-[a-z0-9]+)*`, at most 32 characters including a collision suffix. Cut
   at word boundaries where possible; if an individual token alone exceeds the limit,
   truncate that token. If no usable words remain (including an all-non-Latin or
   punctuation-only headline), use `task`.
5. **Choose the first free name.** Try the semantic stem, then `stem-2`, `stem-3`, etc.,
   without a `-9` cutoff. Reserve suffix room within the limit, trimming the base
   without a trailing hyphen. Keep the meaningful stem on collision; do not degrade to
   `task` merely because nine similar tasks already exist. Count IDs on all blocks,
   including closed/hidden tasks and ordinary paragraphs, using the same recognition and
   exact case-sensitive comparison as the relevant submit validator. Include pending
   batch reservations. IDs used in other notes do not make an ID unavailable, except any
   pre-existing conservative batch reservation policy is retained rather than refactored
   in this task.
6. **Respect explicit existing identity.** An explicitly prefilled rename ID wins
   unchanged. For dependency assignment, reuse a normalized `[id::]` only when it
   satisfies the block-ID grammar and is free under the same validator and reservation
   set; otherwise generate from task text. Do not sanitize/decode a path-qualified value
   such as `Tasks__target` into a fake block ID. Existing dependency-field
   preservation/canonicalization remains the writer's decision. For a marker-triggered
   rename suggestion, exclude only the selected old-ID occurrence from occupancy; never
   globally remove duplicate occurrences.

Pin at least these exact examples (no collisions unless indicated):

| Headline / context                                               | Suggested value                    |
| ---------------------------------------------------------------- | ---------------------------------- |
| `- [ ] #task Fix flaky gkeep test [priority:: high]`             | `fix-flaky-gkeep`                  |
| ``- [?] #task Add support for new `%hold` directive``            | `hold`                             |
| `- [ ] #task Review [[sase#^fix-apollo\|apollo fix]] notes`      | `apollo-fix`                       |
| `- [ ] #task Read [Response API guide](https://example.test/v1)` | `response-api-guide`               |
| `- [ ] #task Renew the library books`                            | `renew-library-books`              |
| `- [ ] #task Book flights [scheduled:: 2026-10-13] #travel`      | `book-flights`                     |
| Same task, with `book-flights` through `book-flights-9` occupied | `book-flights-10`                  |
| ``- [ ] #task Fix `capture_task_id` ``                           | `capture-task-id`                  |
| `- [ ] #task Réparer café`                                       | `reparer-cafe`                     |
| `- [ ] #task 🚀`, with `task` and `task-2` occupied              | `task-3`                           |
| `Book flights [id:: Tasks__target]` without a trailing block ID  | `book-flights`                     |
| Same headline with `[id:: flights]`, free in the note and batch  | `flights` in the dependency prompt |

The backslash before a pipe in the table is Markdown table escaping, not input.

## Integration and lifecycle

1. Add the pure naming helpers and the shared fixture suite. Expose them through the
   existing test helper exports. Test both copies against the same vectors. Do not
   export a new versioned plugin API for this internal behavior.
2. Add a source-to-suggestion resolver to block-id-prompt. Resolve task identity,
   headline, and occupancy from one coherent target snapshot. Direct-add may be invoked
   on non-task Markdown blocks too: preserve their current behavior. A block containing
   several tasks is not a license to guess which child to name; use only a verified task
   at the selected block's root. Task recognition here must include closed tasks
   supported by the direct-block command, not just the open `#task` picker filter.
3. Seed the modal's **value**, not its placeholder. Initialize the input and its
   listeners before asynchronous resolution. Local task prompts can seed immediately;
   cross-note prompts may display a brief loading state, with Save disabled until the
   target snapshot is resolved. Track user edits (including clearing the field) and
   modal lifetime: a late result must never overwrite typing, select text after the user
   has moved focus, or touch a closed/replaced modal. Focus and select an untouched
   generated value once. Enter and Save continue through the existing `submitBlockId`
   path and double-submit guard. If the target cannot be read or uniquely resolved,
   report the existing style of notice/error and do not present a supposedly valid ready
   suggestion.
4. Use authoritative target content, not the source note or metadata-cache block list.
   Same-note reads use the editor. Cross-note reads prefer an open target editor when
   present, otherwise the vault. Keep suggestion, validation, and write preconditions
   coherent: if adding this open-target support to block-id-prompt, make its
   task-assignment path write through that editor and retain its preimage guards rather
   than combining a buffer suggestion with a stale disk overwrite. If multiple live
   target buffers disagree, refuse the operation instead of guessing. Keep this change
   scoped to task assignment; unrelated rename/backlink rewrites need no refactor.
5. Integrate the prompt-specific generator into navigation's shared block-ID stage for
   every mode (`single`, `batch`, `counted-source`, `vault-single`, and
   `vault-counted`). Use the raw target task and `getBlockIdStageContent` plus
   `getBlockIdReservedIds`. Ensure a cross-note snapshot is available rather than
   silently falling back to the dependent note. Use `validateBlockIdCandidate` before
   preferring a legacy `[id::]`. Keep the existing selected-input and preview behavior,
   batch cancellation, and writer invariants.
6. Revalidate against fresh content when accepting. A suggestion is not a reservation
   and typing does not reserve an ID. If a task changes or another block takes the
   suggested ID while the modal is open, reject the stale write with the normal
   feedback; keep the typed value editable. Do not silently replace a user's chosen ID
   on Save. Continue the existing inbox route flow with the accepted ID in its
   reserved-ID checks: the destination is chosen later, so do not pre-choose it or
   bypass its collision checks.

## Verification and completion

Add behavior tests, not just slug snapshots:

- Both helper copies pass the shared vectors, including mixed markup, empty and
  stopword-only text, accents/non-Latin text, meaningful digits, long tokens,
  word-boundary shortening, suffix length, collisions beyond nine, ordinary
  blocks/closed tasks, and pending reservations.
- Exercise the real modal initialization and value/selection, Enter/Save, custom text,
  cancellation, and failed submission. Cover local direct-add, all existing `^^` marker
  spellings via their common path, remote target selection, Pomodoro linking, and the
  existing inbox route continuation. Preserve direct-add task placement and
  source-marker cleanup semantics; opening/cancelling writes no ID, backlink, lane, or
  review stamp.
- Cover delayed reads, typing/clearing before resolution, closing/reopening before
  resolution, unreadable/ambiguous targets, stale task text, and an ID occupied after
  opening. An open unsaved target's ID must win over stale disk/cache data.
- Navigation tests cover every stage mode, valid/free versus invalid/taken legacy
  `[id::]`, two same-text tasks in one batch, correct target-note occupancy, and
  cancellation after one or more staged choices. Keep existing dependency writers'
  legacy-field, canonical-ID, cross-note preparation, and atomicity tests passing.
- Regression tests pin existing-ID rename defaults and backlink behavior, non-task
  prompts, project conversion's existing `suggestBlockIdFromTask`, and the unchanged
  successor SB vectors. For marker-triggered task rename, prove the suggested name
  ignores only the selected old token and cannot hide ambiguity.

Use/extend `scripts/block-id-prompt-harness.cjs` and the existing picker view/runtime
test patterns; export the modal for testing if needed. Register new test files in the
explicit `package.json` test list. Run focused new/affected tests during work, then
`npm run build`, `npm test`, and `npm run validate` in bob-plugins. Inspect
`git diff --check` and ensure generated bundles are current. No Rust tests are needed
for documentation-only bob-cli edits; run the applicable checks if actual Rust code
changes become necessary.

Document the user-visible defaults and examples in bob-plugins README and update bob-cli
`docs/task-dependencies.md` section 6.5 to name the new prompt policy and legacy-ID
handling. Keep capture/successor sections accurate about their existing algorithm. Bump
the two affected plugin manifest versions and their README table entries consistently
with the repository's release convention. Do not edit SASE memory as part of this plan.

After checks pass, fulfill the linked repository's mandatory deployment step: from the
opened checkout, run
`bob plugins sync --no-pull --repo "$PWD" --plugin block-id-prompt --dry-run --format json`,
then the corresponding real sync; repeat for `bob-navigation-hotkeys`. Inspect results
for skipped files as well as the exit code. Do not use `--force` to clobber unrelated
vault edits. Report what was actually deployed and whether an Obsidian reload or manual
verification remains. Where an interactive Obsidian session is available, verify a
direct-add, cross-note picker, Pomodoro link, and dependency batch with Enter, manual
replacement, and Escape. If no UI session is available, report that limitation alongside
runtime-harness coverage rather than claiming an interactive smoke test.

Acceptance: every supported new-task-ID prompt reaches its ready state with a selected
nonempty, valid, collision-aware value; same inputs have the same default in both prompt
implementations; user edits and cancellation remain authoritative; existing task/link
contracts pass their regression tests; and both changed plugins are built and deployment
results are reported.
