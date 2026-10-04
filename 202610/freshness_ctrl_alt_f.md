---
tier: tale
title: Move freshness confirm-and-advance to Ctrl+Alt+F
goal:
  Ctrl+Alt+F confirms or completes the current Obsidian review task and advances through
  the existing review queue, while Alt+F keeps its in-place behavior and Alt+Shift+F is
  released.
size: small
proposed_by: bbugyi200.apollo.53
create_time: 2026-10-04 13:01:01
status: wip
---

# Move freshness confirm-and-advance to Ctrl+Alt+F

## Outcome and scope

Replace Alt+Shift+F with Ctrl+Alt+F for Bob's existing **Refresh task freshness and jump
to the next due task** command. On macOS this means the physical Control+Option+F keys;
use Obsidian's `Ctrl` modifier, not `Mod` or `Meta`. Alt+F continues to confirm in
place. The old chord must be released by the command registration and keyboard
interception paths so Bryan can use it for his other MacBook shortcut.

This is a tale of size small because the root cause and affected routes are known and
one coding agent can make the bounded remap, update its presentation, verify it, and
deploy it. No phase split is needed.

Work in bob-cli and its linked bob-plugins repository. Open bob-plugins with
`sase repo open bob-plugins -r "Implement the approved freshness shortcut remap"` and
read its AGENTS.md before editing. Use only the path that command returns. The linked
chezmoi source was inspected during planning: it contains neither this Obsidian command
mapping nor either F chord, so no dotfile change is currently indicated.

## Findings that guide implementation

- `plugins/bob-navigation-hotkeys/src/490-plugin-lifecycle.js` registers
  `refresh-task-freshness-and-advance` with `["Alt", "Shift"]` and routes it to
  `refreshTaskFreshness(editor, { advance: true })`. Keep that command ID, name, and
  behavior; replace its default modifiers with `["Ctrl", "Alt"]`.
- `src/480-review-jump-and-nav-api.js` in the same plugin defines
  `isReviewRefreshKeydown(event, wantShift)`. It currently rejects Control entirely and
  selects the advancing route by Shift.
- `src/540-plugin-decay-picker-and-cancel.js` registers capture listeners on both window
  and document for Vim normal mode; its handler currently uses `event.shiftKey` to
  choose `advance`. Changing only the registered default would leave Vim using the
  conflicting chord and reject the new chord.
- `src/060-decay-card-modal.js` currently treats any Alt+F variant as Keep. This can
  still consume Alt+Shift+F while the decay card is open and should use the supported
  refresh chord matcher during this remap.
- The PRE checklist hint embeds the old chord in navigation's
  `src/480-review-jump-and-nav-api.js` and ledger-tools'
  `plugins/bob-ledger-tools/src/120-freshness-footer.js`. The latter feeds the
  persistent footer through `reviewEntryView`; both the current API and navigation's
  fallback text must agree.
- Both plugins use ordered `src/fragments.json` source builds. Their committed `main.js`
  files are generated; edit source fragments and run `npm run build`. Each hand-edited
  fragment must remain at most 1000 lines.
- bob-cli's `bob plugins sync` deploys `manifest.json`, `main.js`, and `styles.css`. It
  does not manage `.obsidian/hotkeys.json`, so an existing user override can take
  precedence over a changed default.

## Implementation

1. **Remap every dispatch route in bob-plugins.** Change the registered advancing
   command's hotkey to Ctrl+Alt+F. Rename the matcher's boolean parameter to express
   advance intent and recognize exactly these physical chords: Alt+F with no
   Ctrl/Meta/Shift confirms in place; Ctrl+Alt+F with no Meta/Shift confirms and
   advances. Reject Alt+Shift+F, Ctrl+Alt+Shift+F, Meta variants, missing Alt, and
   unrelated keys. Preserve `code === "KeyF"` matching plus the existing `key` fallback
   so Option-modified characters on macOS still match the physical F key. Derive
   capture-handler `advance` from the advancing matcher rather than Shift. Preserve
   focused-editor and Vim-mode guards, held-key repeat rejection, WeakSet event
   deduplication, and numeric-prefix capture/reset. Make the decay card's refresh-key
   Keep handler use the same supported-chord predicate while retaining its opening-key
   and repeat protections.

2. **Update the shortcut's presentation.** Replace its old spelling in the PRE checklist
   notice and footer hint with `Ctrl+Alt+F done → next · ]s skip`, including the
   navigation fallback. Update relevant comments, test descriptions, bob-plugins README,
   and navigation's manifest description. Increment the patch versions of the changed
   navigation and ledger-tools plugins from their current versions (planning observed
   2.3.0 and 1.29.0). Regenerate both committed entrypoints using the repository's build
   command.

3. **Update bob-cli's documented workflow.** Replace mentions of this gesture in
   `docs/freshness.md`, `docs/getting-started.md`, `docs/projects.md`, and the R12
   example in `docs/plan.md`; search for other current instructional references. Explain
   Control+Option+F once where it helps Mac users. Retain the established refresh
   behavior: ordinary tasks confirm and advance; PRE/POST checklist tasks complete
   through the cycler without a freshness stamp; Pending may ask for an optional Work
   Log; single due at-limit targets open the decision card; project/reference and
   counted/Task Link paths continue to use the existing writers and review queue. This
   remap requires no evaluator, CLI interface, or durable-memory changes.

4. **Verify the actual event routes and existing review behavior.** Extend existing
   tests rather than adding a separate harness. Cover the chord matrix in
   `scripts/test-navigation-freshness.cjs`, including a macOS-like event with
   `code: "KeyF"` and an Option-produced non-ASCII `key`, and the `key` fallback when
   `code` is unavailable. Extend its existing onload harness to check the default
   modifiers and call the registered command's editor callback with `advance: true`.
   Extend the real physical-key test in `scripts/test-navigation-keep-counting.cjs` to
   prove Ctrl+Alt+F is handled once across both capture listeners, dispatches advancing
   intent, preserves numeric-prefix semantics, and advances once after a successful
   operation. Prove Alt+F keeps its non-advancing route. Prove obsolete or
   extra-modifier chords remain unconsumed and neither refresh nor reset a pending
   count; repeated events and non-editor/modal targets remain ignored by the editor
   capture route. Update checklist and footer string expectations. In the existing
   decision-card tests, verify the supported refresh chords, held repeats, and rejection
   of the retired chord. Reuse existing coverage for Pending cancellation, stale-write
   refusal, checklist completion, decision-card outcomes, and walk anchoring rather than
   changing those mechanisms.

5. **Deploy the verified source and check the effective binding.** The bob-plugins
   AGENTS.md requires `bob plugins sync` after changes. Use the returned linked checkout
   explicitly as `--repo` and use `--no-pull` so deployment takes the exact verified
   source. Preview then deploy the two affected plugins separately with
   `--plugin bob-navigation-hotkeys` and `--plugin bob-ledger-tools`; inspect reports
   because skipped dirty files can coexist with a zero exit code. Respect existing
   dirty-file protection. Reload the plugins or Obsidian. In Settings → Hotkeys check
   this command for a saved override: if the old default was saved explicitly, reset
   this command to the new default or replace only its override with Ctrl+Alt+F. Check
   the effective Ctrl+Alt+F binding for collisions and report any unrelated user binding
   that needs a decision rather than removing it. Do not add a plugin startup migration
   that rewrites user hotkeys.

## Validation and acceptance

After the source edits, run `npm run build` in bob-plugins, then `npm run validate` and
`npm test`. Those scripts include generated-source checking, manifest and JavaScript
validation, and the existing review and Vim suites. Review the diff and run
`git diff --check` in both repositories. Search the edited plugins and current
documentation for `Alt+Shift+F` and `refresh-task-freshness-and-advance`; old-chord
mentions should remain only where explicitly describing the retired chord in regression
coverage or history. No new Rust behavior is introduced, so a full Rust test run is not
needed for the documentation-only bob-cli edits.

Use the two-plugin dry-run deployment reports to confirm the source repo and only the
intended plugin assets; after actual sync, use
`bob plugins list --repo <opened-bob-plugins-path> --no-pull --format json` to confirm
both plugins are synced. Resolve skips before claiming deployment succeeded.

On the MacBook, verify Control+Option+F on a due ordinary task in Vim normal mode and in
insert/non-Vim editing: it confirms and lands on the next due entry once. Verify Alt+F
confirms in place, Alt+Shift+F performs no Bob refresh/completion/advance, PRE confirms
by completion, and a Pending prompt advances only on acceptance (Escape writes and
advances nothing). Check the notice/footer advertise the new chord. Automated tests must
verify these dispatch and preservation guarantees; if a live Mac UI is unavailable,
report the remaining manual smoke check explicitly instead of claiming it ran. Because
custom plugins are excluded from vault Git sync, ensure the updated plugin source is
available to the MacBook's normal plugin deployment workflow and run `bob plugins sync`
there; vault note sync alone does not distribute these assets.
