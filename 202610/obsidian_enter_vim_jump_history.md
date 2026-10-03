---
tier: tale
title: Make Obsidian Enter link jumps work with Vim Ctrl+O and Ctrl+I
goal: Restore the originating note and cursor with Ctrl+O after an Enter link jump,
  support Ctrl+I forward traversal, and preserve ordinary Vim jumps.
size: medium
proposed_by: bbugyi200.apollo.4k
status: done
---

# Make Obsidian Enter link jumps work with Vim Ctrl+O and Ctrl+I

## Objective and scope

After normal-mode `<Enter>` opens a Markdown link, `<C-o>` must return to the
originating note and exact cursor position; `<C-i>` must return forward. Support
same-note headings/blocks and existing destination tabs as well as notes opened in the
current tab. Preserve ordinary Vim jump-history behavior, including counts and
search/`gg`/`G` jumps interleaved with link jumps.

This is one **medium tale**: the implementation, integration, regression tests, and
deployment form one bounded change in `bob-plugins`, suitable for one coding agent. No
CLI change, new plugin dependency, memory change, or separate epic is needed. Only this
plan is authored during planning; implementation and deployment follow plan approval.

## Repository access and evidence

Use `/sase_repo` to open `bob-plugins` with an implementation-specific reason, then read
its `AGENTS.md`. Use the returned checkout, never deployed plugin files under the vault.
`plugins/*/main.js` are source files, not generated bundles.

The investigation inspected `bob-plugins` at `da7cce4`. Relevant code:

- `plugins/task-status-cycler/main.js`: `registerVimMappings()` owns normal-mode `<CR>`
  and `<BS>`; `handleVimEnterLinkOrFallthrough()` delegates to navigation, retaining the
  line-motion fallback when no link is handled. Its registration already handles delayed
  availability of `window.CodeMirrorAdapter.Vim`.
- `plugins/bob-navigation-hotkeys/main.js`: `handleVimLineLinkAction()` resolves links,
  opens one candidate immediately, or constructs `LinkCandidatePickerModal`.
  `handleVimEnterLinkAction()` and `handleVimBackspaceLinkAction()` share this path.
  Bare Enter examines the current line; explicit `N<Enter>` examines the line N lines
  below. Preserve `repeatIsExplicit` semantics and cursor origin.
- `openOrCreateLinkCandidate()` currently only calls `captureActiveFilePosition()`
  before `openResolvedLink()` or `createNoteFromLinkCandidate()`. `filePositions` is a
  latest-position-per-file cache for other navigation, not chronological history.
- `openResolvedLink()` supports `#heading` and `#^block` links and deliberately
  activates an existing destination leaf. It may return without `openLinkText()` for a
  plain link to an already-open note. `openMarkdownFileWithLeafReuse()`,
  `findMarkdownLeafByPath()`, `activateWorkspaceLeaf()`, and the cursor restoration
  helpers provide reusable pieces.
- The multiple-link picker calls `openOrCreateLinkCandidate()` only when an item is
  chosen. Note creation is asynchronous and may fail or be unavailable.
- Navigation's `registerVimMappingsWhenReady()`/`registerVimMappings()` currently owns
  the `\s` action. Use that lifecycle for the added integration; retain the existing
  ownership of Enter in task-status-cycler.

Upstream was inspected through `/sase_repo`, not by downloading repository files:

- `gh:replit/codemirror-vim`, commit `8640966b6977f84d2197e6adfd521fb737184587`,
  `packages/codemirror-vim-core/vim.js`: native `<C-o>`/`<C-i>` dispatch `jumpListWalk`.
  That action calls `jumpList.move(cm, signedCount)` and then
  `cm.setCursor(mark.find())` in the current editor. `createCircularJumpList()` stores
  100 bookmarks and compares only coordinates. Native motions with `toJumplist` call
  `jumpList.add(cm, oldPosition, newPosition)`. `getVimGlobalState_()` exposes the list
  but is explicitly a testing hook.
- A read-only Node probe instantiated upstream `initVim`, verified an empty list has no
  back target, then walked an A-editor bookmark from a B editor. It returned A's
  coordinates without any note-switch operation. This confirms why merely pushing a
  bookmark cannot solve cross-note navigation. It does **not** establish which
  CodeMirror revision the user's running Obsidian bundles.
- `gh:esm7/obsidian-vimrc-support`, commit `1ff5d97afcf3161f202aa460697233d550f6aa3f`,
  documents optional mappings to `app:go-back`/`app:go-forward`. Those are workspace
  history, not a shared Vim position history. The actual running vault Vimrc was not
  inspected. Chezmoi's opened source did not contain the Obsidian keymap configuration.

## Behavior contract

1. A successful Enter jump records its actual source `{path, line, ch}` and settled
   destination. Snapshot the original cursor before a count selects another line and
   before a picker takes focus. A chosen picker item produces one jump;
   opening/filtering/cancelling a picker produces none.
2. Back/forward traverses a single ordered history of supported Vim jumps and Enter link
   jumps. Each location includes file identity, so equal coordinates in different files
   remain distinct. `A -> B -> C`, back twice, forward twice must round-trip. Same-note
   links and multiple locations in the same note work too.
3. Honor positive counts for both directions and stop at the oldest/newest valid entry.
   Traversal never records a new jump. A successful new jump after going back discards
   the forward branch; a failed/cancelled action does not.
4. Preserve the actual position being left when beginning back traversal, so forward can
   restore it even after ordinary cursor movement at the destination. Do not make the
   first Ctrl+O merely record the current position or require an extra keypress before
   returning to the link's origin.
5. Return to the original leaf when it still displays the recorded file. Otherwise reuse
   an existing leaf for that file, then open it using existing navigation policy.
   Restore cursor and editor focus after the correct document is ready, clamping
   positions after edits. Preserve tab pinning and avoid duplicate tabs.
6. Existing Enter grammar, aliases, embedded wiki links, internal Markdown links,
   heading/block resolution, counts, no-link line motion, and note creation remain
   intact. Apply the same recording to Backspace's shared link-opening path, but not to
   its no-link line motion. Unrelated navigation commands need not start recording link
   jumps as part of this change.
7. Record only completed navigation. External/non-Markdown links, failed opens, failed
   creation, and same-file/same-position no-ops add nothing. Keep existing failure
   notices and contain asynchronous errors.
8. Intercept history keys only in Vim normal mode. Preserve insert-mode Ctrl+O's
   one-normal-command behavior, visual mode, non-Vim editing, and plain Tab. Test the
   actual Ctrl+I event representation without globally remapping Tab.

## Implementation

### 1. Add a bounded, file-aware history model in navigation

Keep this history distinct from `filePositions`, alternate-file navigation, and Obsidian
workspace history. Implement small testable helpers in navigation's existing
source/helper-export pattern. Store an ordered list, traversal index, and at most 100
locations. Store copied coordinates and paths, with optional leaf/editor identity for
correct restoration. Deduplicate adjacent identical locations by path **and** position;
truncate forward entries only on successful recording, and release retained
bookmarks/references on eviction and unload.

Use live bookmarks for native jumps while their editor still represents the same file,
falling back to saved/clamped coordinates after a document or leaf is gone. Do not trust
a reused editor's old bookmark for a different file. Refresh saved positions while their
original editor is available. Update paths on vault rename; skip deleted/unresolvable
entries without creating files. A recoverable open error must not corrupt the traversal
index or discard forward history.

### 2. Integrate ordinary Vim jumps through a narrow compatibility bridge

Install once when Vim becomes available. Feature-detect
`Vim.getVimGlobalState_().jumpList` and its `add`/`move` functions rather than assuming
an upstream package version is the bundled Obsidian version. Wrap the existing list's
`add` to mirror native jump endpoints into the file-aware history, while calling the
original method with its original receiver and arguments. Match the supplied editor to
its Markdown view/file; do not assign an unrelated active file to a background editor.
Native search/`gg`/`G` jumps and explicit link jumps must enter the same timeline, not
competing stacks with ambiguous precedence.

Register named normal-mode back/forward actions in navigation's mapping lifecycle,
passing Vim's count through. Before any file-aware entries exist, delegate to the
original native walk behavior; after history exists, its boundaries are no-ops, not
fallbacks that replay stale native entries. Do not recursively feed `<C-o>` or `<C-i>`
back through their new mapping. Keep the native list and other global Vim state intact;
never reset registers, marks, search state, or the whole global list.

Isolate the private API dependency and test its absence. If the expected bridge cannot
be installed, preserve the original keys and ordinary link opening, emit one actionable
compatibility diagnostic, and report that limitation. Do not ship a silent link-only
replacement that disables normal Vim jumps. Verify the bridge against the available
Obsidian runtime during smoke testing where possible.

Uninstall owned hooks/mappings on plugin disposal, restore the native `add` only if it
is still this plugin's wrapper, cancel deferred navigation, and release the history.
Repeated registration/reload must not stack wrappers or duplicate jumps. Preserve
pre-existing mappings on unload and avoid removing a later mapping owned by another
plugin. Account for Vimrc reload/load ordering in the runtime check; do not introduce
repeated blanket remapping of the user's keymap.

### 3. Record the Enter transaction at the correct boundary

Capture an immutable origin in `handleVimLineLinkAction()` before opening the single
candidate or picker. Pass an optional navigation context through
`LinkCandidatePickerModal` and `openOrCreateLinkCandidate()` so unrelated callers keep
their current behavior. Consume that context only when navigation succeeds.

Await target opening/activation and the correct Markdown editor's arrival before
capturing the destination; a resolved open promise alone may precede heading/block
cursor placement. Use bounded, cancellable readiness checks tied to the expected
file/leaf and an operation token. Never capture an unrelated active note or run a late
cursor restoration into a newer navigation. If the user changes context while a picker
is open, reject a stale origin rather than recording a false jump.

Guard record/replay operations so programmatic cursor restoration and incidental native
callbacks cannot record duplicates. Serialize history traversal requests, including
counts and rapid repeated keys, and prevent overlapping link-open transactions from
racing the history index. Keep existing synchronous handled/ fallthrough returns at the
task-status-cycler boundary while containing the async work in navigation. Successful
creation participates once its editor is ready; Templater failure creates no history
entry. Do not alter note templates/content.

### 4. Add regression coverage and document the key behavior

Add `scripts/test-navigation-jump-history.cjs` and register it in `package.json`'s test
command, using the repository's Node tests and Obsidian stubs. Export only the helpers
needed for meaningful state and integration tests. Exercise the real Enter delegation,
candidate-opening/picker callbacks, native-list bridge, and registered history actions,
rather than only asserting mapping names.

Update the existing mapping assertion in `scripts/test-navigation-hotkeys.cjs` that
currently expects only `\s`. Add focused task-status-cycler regression coverage if its
delegation boundary changes. Update `README.md` with the normal- mode round-trip,
counts, history lifetime, native-jump compatibility, and existing Enter/Backspace
behavior. Bump the navigation manifest version and its README row; bump another plugin
only if its shipped source changes.

## Validation and acceptance

Automated cases must include:

- Enter A -> B, Ctrl+O -> original A position, Ctrl+I -> B; same coordinates in
  different files; A -> B -> C; counted traversal and both boundaries.
- Heading and block links within A and across notes, including an already-open B leaf
  and delayed destination selection. Returning to a closed source leaf must reopen/reuse
  A without creating duplicates or changing pin state.
- Bare versus counted Enter, including return to the original cursor rather than the
  counted link line; multiple-link choice and cancellation; Backspace's shared link
  path; no-link fallback; alias/transclusion/Markdown link preservation.
- Ordinary cursor motion before going back, native `gg`/search jumps before and after a
  link jump, forward-branch replacement, and a failed jump while on an old branch.
  Verify native callbacks are mirrored once and replay is not recorded.
- Failed link open, creation failure/success, stale picker, rapid back/forward, delayed
  old callbacks, deleted/renamed files, clamping after edits, 100-location retention,
  unavailable Vim API, delayed registration, unload, and reload.
- Normal-mode-only dispatch, unchanged insert/visual/non-Vim behavior, and Ctrl+I versus
  Tab. Native fallback with no new history must not recurse.

Run focused tests first, then `npm test` and `npm run validate` in `bob-plugins`. Use
`/sase_monitor` if a command will outlast the provider turn. No bob-cli Rust checks are
necessary unless implementation unexpectedly changes bob-cli source.

After tests, deploy the changed plugin(s), as required by `bob-plugins/AGENTS.md`:

```bash
bob plugins sync --repo "$PLUGIN_REPO" --no-pull --plugin bob-navigation-hotkeys --dry-run
bob plugins sync --repo "$PLUGIN_REPO" --no-pull --plugin bob-navigation-hotkeys
```

`PLUGIN_REPO` means the checkout path returned by this implementing agent's
`sase repo open`, never the default canonical checkout. Sync task-status-cycler
similarly only if changed. Do not add `--force` to override dirty deployed files. Reload
the changed plugin(s) in Obsidian for interactive testing when available.

Smoke-test A containing a note link, heading/block links, and a multiple-link line:
verify exact cursor round trips with B closed and already open, counted history,
interleaved `gg`/search, cancelling the picker, insert mode, and Tab. Verify which
handler actually receives Ctrl+O/Ctrl+I after Vimrc loading. If Obsidian is not
available, report these runtime checks as pending rather than claiming they passed. The
automated integration tests and successful deployment remain required.

## Risks and chosen tradeoffs

Merely adding native bookmarks cannot switch files; mapping to workspace back/ forward
misses same-note positions and leaf reuse; a second link-only stack loses chronology
with normal Vim jumps. The chosen file-aware timeline with a narrow native-add bridge
addresses all three while keeping Enter's current navigation policy. Its primary risk is
Obsidian's private Vim adapter surface, managed through feature detection, lifecycle
isolation, compatibility tests, and runtime checks.

History is session-local and bounded. Numeric fallback positions are clamped after
closed-note edits; this is not durable semantic tracking of arbitrary edited text. No
change to task lanes, dependencies, review state, or vault note content belongs in this
work.
