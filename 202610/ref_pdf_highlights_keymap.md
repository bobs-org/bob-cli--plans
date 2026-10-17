---
tier: tale
title: Cmd+Enter opens the selected reference task's PDF in Highlights
goal: In Obsidian on the Mac, Cmd+Enter on a reading task, or a Task Link or managed
  embed pointing at one, opens its library PDF in Highlights without writing the vault,
  while Cmd+Enter everywhere else keeps opening the link under the cursor in a new
  tab.
size: medium
decisions:
  non_mac:
    ask: Off macOS, should Cmd+Enter on a reading task open the PDF in the system
      default viewer?
    choices:
      notice: Show a "Highlights is macOS-only" notice; nothing opens
      default_app: Open the PDF in the system default viewer instead
    default: notice
    why: Highlights is the annotation source bob ref scan reads; other viewers never
      sync
    answer: notice
  ref_note_scope:
    ask: Should Cmd+Enter anywhere inside a managed reference note open that note's
      PDF?
    choices:
      task_lines: Only reading-task, managed-embed, v1-tracker, and Task Link lines
        open the PDF
      whole_note: Any ref-note line opens its PDF; Cmd+Enter on links there stops
        opening new tabs
    default: task_lines
    why: Matches the request and keeps built-in Cmd+Enter on links inside ref notes
    answer: task_lines
proposed_by: bbugyi200.athena.111
proposed_with: claude/opus
decided_by: auto
status: done
---

# Cmd+Enter opens the selected reference task's PDF in Highlights

## Goal

In Obsidian on the Mac, pressing `Cmd+Enter` while the cursor is on a reference reading
task (`#task #ref`) opens that reference's library PDF in the Highlights app. This also
works from a Task Link to that task (for example in today's Pomodoro) and from the
managed embed inside the ref note. Everywhere else, `Cmd+Enter` keeps doing what
Obsidian does today ("Open link under cursor in new tab"). The gesture never writes to
the vault. It does not change reading state, task lanes, freshness, or review state.
This matches the Bob Mac Capture Refs panel's "Return opens the original PDF in
Highlights and changes nothing in the vault".

The work lives in the **bob-plugins** linked repo (plugin `bob-navigation-hotkeys`). One
line also goes into the vault's `.obsidian/hotkeys.json`, and one sentence goes into
bob-cli's `docs/ref.md`. Open bob-plugins with the `/sase_repo` skill
(`sase repo open bob-plugins -r "<reason>"`) and use only the path it prints. Read that
repo's `AGENTS.md` first. Its rules are: edit `src/` fragments only, run
`npm run build`, never hand-edit `main.js`, keep each fragment at or under 1000 lines,
and run `bob plugins sync` after changes.

## Verified context (do not re-derive)

### Reading-task shapes and where the PDF lives

- **v2 reading task** (current; lives in an area/project/inbox note). Real example from
  `sase.md`:
  `- [/] #task #ref [[ref/chat/project_stats_cli_design|sase project stats: …]] [fresh:: 2026-10-17] [created:: 2026-10-16] ^ref-project-stats-cli-design`.
  The linked ref note `ref/chat/project_stats_cli_design.md` has the frontmatter
  `source_pdf: lib/chat/project_stats_cli_design.pdf`. That value is a plain,
  vault-relative path.
- **v1 tracker** (legacy; inside the ref note itself):
  `- [ ] #task #ref [[lib/chat/<stem>.pdf]] #hide ^ref`. The wikilink points straight at
  the PDF, and the note's own `source_pdf` names the same file.
- **Managed embed**: a v2 ref note's body has one line `![[<task-path>#^<id>]]` (no
  bullet) that transcludes its reading task.
- **Task Links**: a dedicated Task Link bullet (`- [[sase#^ref-…]]`, `- ![[…]]`, struck
  `~~[[…]]~~`, optional checkbox / 🍅 / trailing `#`) points at the reading task by
  block id.
- Library PDFs under `lib/` are git-tracked in the vault (not gitignored), so they exist
  on the Mac. `xlib/` intake is gitignored and is never a `source_pdf` target for
  scanned notes.
- Nav already treats `ref/**.md` with a non-empty `source_pdf` as a _managed reference
  note_. See `isManagedRefNote(path, frontmatter)` in
  `plugins/bob-navigation-hotkeys/src/210-rename-and-line-context.js`.

### Highlights launch convention (from bob-mac-capture)

`Sources/BobMacCapture/Refs/RefsPanelModel.swift` opens the library original (never a
copy) with Highlights' bundle id **`net.highlightsapp.universal`**. It activates the app
and never builds a shell command. The plugin should do the same thing with
`execFile("/usr/bin/open", ["-b", "net.highlightsapp.universal", <absolute pdf>])`. If
that fails, retry once with `["-a", "Highlights", <absolute pdf>]`, which covers
non-App-Store builds. Pass arguments as an argv array, never through a shell.

### Obsidian hotkey precedence (checked in the Obsidian bundle installed on athena)

- Obsidian's built-in `editor:open-link-in-new-leaf` ("Open link under cursor in new
  tab") has the default hotkey **`Mod+Enter`**, which is `Cmd+Enter` on macOS. Its
  `checkCallback(false)` opens the clickable token under the cursor in a new tab. With
  no token, it dispatches an `open-link` CustomEvent on `activeDocument.activeElement`,
  which is how panes like the file explorer handle `Cmd+Enter`.
- `hotkeyManager.bake()` orders **custom keys (`hotkeys.json`) first, then defaults**,
  and defaults stay in registration order (built-ins first). `onTrigger` runs the
  **first** matching command and stops. `executeCommand` just runs the command's
  `checkCallback(false)` / `callback()` and returns true. No "first command whose check
  passes" fallthrough exists.
- Obsidian listens for `keydown` on `window` in the capture phase, so its hotkeys run
  before any CodeMirror keymap or plugin DOM listener.
- **Consequence:** a plugin _default_ hotkey of `Cmd+Enter` would never fire, because
  the built-in default always wins. The binding must be a **custom entry in the vault's
  `.obsidian/hotkeys.json`**. Since that entry outranks the built-in everywhere, the
  command must **delegate to `editor:open-link-in-new-leaf`** whenever it is not on a
  reading-task target. This keeps today's `Cmd+Enter` behavior byte-for-byte.
- Use the modifier **`"Meta"`**, not `"Mod"`. `compileModifiers` maps `Mod` to `Ctrl`
  off macOS, and Ctrl+Enter is the task-status-cycler close gesture on athena's Linux
  Obsidian. `Meta` is Cmd on macOS and Super elsewhere.
- Precedent: commands bound only through `hotkeys.json` declare **no** code default (for
  example `jump-to-next-open-task` in `490-plugin-lifecycle.js`; vault commit `60a9d654`
  added its `hotkeys.json` entry).
- The vault `hotkeys.json` currently binds nothing to `Enter`.

### Reusable nav helpers (shared fragment scope)

- `REF_AFTER_TASK_RE` (010): the `#task #ref` pair.
- `isObsidianTaskAtLine(content, line, contexts?, lines?)`.
- `getMarkdownLineContexts(text)`: `inFence`.
- `isManagedRefNote`.
- `normalizeVaultRelativePath`.
- `joinFilesystemPath` (090).
- `this.getVaultBasePath()` (640).
- `this.parseWikiLinkAt(line, i, { allowTransclusion })`.
- `this.resolveLinkTargetFile(target, sourcePath)`.
- `this.stripLinkSubpath`.
- `this.isMarkdownFile` (680).
- `parseLinkPickerTaskLink(lineText)` (220): the dedicated Task Link grammar.
- `findUniqueLinkPickerTargetLine(content, blockId, { allowClosed: true })` (220).
- `this.readLinkPickerNoteContent(path, file)` (500): live editor buffer first.
- `isEditorEventTarget`-style focus checks (610).
- The lazy `require("child_process")` + injectable `execFile` pattern in
  `plugins/bob-project-tasks/main.js` (`snapshotSpawnBob`).

## Behavior specification

The command id is `open-reference-pdf-in-highlights` and its name is
`Open reference PDF in Highlights (or link in new tab)`. It is a plain `callback`
command, **not** `editorCallback`. An editor-only command would swallow `Cmd+Enter` in
the file explorer and other panes, because Obsidian runs the first match even when it
does nothing. It has **no** code default hotkey; `hotkeys.json` owns the binding.

On trigger:

1. **Editor context.** Use the active `MarkdownView` and its `editor` only when the view
   is in source or live-preview mode and `activeDocument.activeElement` is inside that
   view's `.cm-editor` (or is `body`, which covers palette invocation). Otherwise
   **delegate** (step 6).
2. **Reading-task line.** Let `L` be the cursor line. If `L` is a `#task` line
   (`isObsidianTaskAtLine`, not in a fence or frontmatter) that matches
   `REF_AFTER_TASK_RE`, it is a reading task. Any checkbox counts: Ready, Next, In
   Progress, Blocked `[?]`, and closed `[x]`/`[-]` all open. Resolve its PDF:
   - Walk the wikilinks on `L` in order (allow `!` transclusions). Resolve each target
     with its subpath stripped (`resolveLinkTargetFile(target, notePath)`). If the
     target is a `.pdf` file, use its path (v1). If it is a Markdown file that is a
     managed ref note, use its `source_pdf` (v2). The first hit wins.
   - If no link hits and the note containing `L` is itself a managed ref note, use that
     note's `source_pdf` (a v1 tracker without a link).
   - If nothing hits, show the notice `Reading task links no reference PDF`, or
     `<ref note path> has no source_pdf (legacy reference)` when the link resolved to a
     ref note without one. Do **not** delegate.
3. **Block-link line.** Otherwise, if `L` is a dedicated Task Link
   (`parseLinkPickerTaskLink`) or a bare block-link/embed line
   (`^\s*!?\[\[<target>#\^<id>(\|alias)?\]\]\s*$`, which covers the managed embed),
   resolve the target note and its unique block line with `allowClosed: true`. Read
   through `readLinkPickerNoteContent`. If that line is a `#task #ref` task, resolve it
   as in step 2, with the target note as the source path and the "own note" fallback. If
   the target is missing, duplicated, not a task, or not a reading task, **delegate**:
   it is an ordinary link, and the built-in opens it in a new tab as before.
4. **Whole ref note (decision).**

   > [!decision] ref_note_scope = whole_note If steps 2–3 found nothing and the active
   > note is a managed ref note, use its own `source_pdf`. Under the default
   > `task_lines`, skip this step.

5. **Open.**
   - Normalize the PDF path. It must be vault-relative, end in `.pdf`
     (case-insensitive), contain no `..` segment, and not be absolute. Otherwise show
     `Unsafe source_pdf: <value>`.
   - Confirm `await app.vault.adapter.exists(path)`. Otherwise show
     `Reference PDF not found: <path>`, for example when the PDF has not synced to this
     machine yet.
   - Build the absolute path with `joinFilesystemPath(getVaultBasePath(), path)`. If
     there is no base path, show `Opening PDFs needs the desktop app`.
   - Platform:

     > [!decision] non_mac = notice Off macOS (`Platform.isMacOS !== true`), show
     > `Highlights is macOS-only: <path>` and stop.

     > [!decision] non_mac = default_app Off macOS, call `app.openWithDefaultApp(path)`
     > when it is a function and show `Opened <basename> in the default PDF viewer`.
     > Otherwise use the `notice` text.

   - On macOS, run `execFile("/usr/bin/open", ["-b", HIGHLIGHTS_BUNDLE_ID, abs])` with a
     short timeout (for example 15 s). If it errors, retry once with
     `["-a", "Highlights", abs]`. On success, show `Opened <basename> in Highlights`. If
     both fail, show
     `Could not open <basename> in Highlights: <first non-empty stderr line or error message>`.
   - Load `child_process` lazily inside the launch method in a `try`. The plugin is
     `isDesktopOnly: false` and must still load on mobile.

6. **Delegate.** Call `app.commands.executeCommandById("editor:open-link-in-new-leaf")`
   once, inside a `try`. If that command is missing, do nothing. Never call it after a
   reading-task target was identified, even if opening failed.

Guarantees:

- No vault, editor, or frontmatter writes on any path.
- No jump-history entry.
- No vim count consumed.
- Notices only.

## Implementation steps

1. **New fragment** `plugins/bob-navigation-hotkeys/src/677-plugin-ref-pdf-open.js`. Add
   it to `src/fragments.json` immediately after `"675-plugin-goto-task-uri.js"`. It
   contains:
   - constants `HIGHLIGHTS_BUNDLE_ID = "net.highlightsapp.universal"`,
     `HIGHLIGHTS_APP_NAME = "Highlights"`, `HIGHLIGHTS_OPEN_TIMEOUT_MS`, and
     `OPEN_LINK_IN_NEW_TAB_COMMAND_ID = "editor:open-link-in-new-leaf"`;
   - pure helpers `parseRefPdfBlockLinkLine(lineText)` (Task Link via
     `parseLinkPickerTaskLink`, else the bare embed/link regex → `{ target, blockId }`
     or `null`), `normalizeRefSourcePdfPath(value)` (→ normalized path or `null`), and
     `buildHighlightsOpenAttempts(absPath)` (→ the two argv arrays);
   - `class BobNavigationHotkeysRefPdfOpenMixin` with:
     - `openReferencePdfInHighlightsOrLink()`, the command body;
     - `getRefPdfEditorContext()`;
     - `async resolveReferencePdfAtCursor(editor, file)` →
       `{ kind: "none" } | { kind: "error", message } | { kind: "pdf", path }`;
     - `resolveReferencePdfFromTaskLine(lineText, sourcePath)`;
     - `async openReferencePdfPath(path, options = {})`. Its options are test seams:
       `execFile`, `isMacOS`, `basePath`, `exists`;
     - `delegateOpenLinkInNewTab()`.

   Read `obsidian.Platform` defensively, because the test harness's obsidian stub has no
   `Platform`. Match nearby code: two-space indent, short comments that explain _why_,
   frozen result objects.

2. **Install and export.** Add the mixin to the `installBobNavigationHotkeysMixins` list
   in `690-install-plugin-mixins.js`. Add the pure helpers and constants to
   `module.exports.helpers` in `700-exports.js`.
3. **Register the command** in `490-plugin-lifecycle.js`, near the other `open-*`
   commands, with `callback: () => void this.openReferencePdfInHighlightsOrLink()` and
   no `hotkeys`. Add a one-line comment saying that the vault `hotkeys.json` binds
   `Meta+Enter`, because a default would lose to Obsidian's built-in `Mod+Enter`.
4. **Tests.** Add a new file, `scripts/test-navigation-hotkeys-ref-pdf.cjs`, using
   `navigation-hotkeys-harness.cjs` (`Object.create(NavigationHotkeysPlugin.prototype)`
   with a stubbed `app`, as `test-navigation-hotkeys-rename.cjs` does). Add it to the
   `test` script list in `package.json`. Cover:
   - A v2 line opens via ref-note frontmatter. Assert `execFile` gets `/usr/bin/open`
     and `["-b","net.highlightsapp.universal","<base>/lib/chat/x.pdf"]`, the success
     notice appears, and nothing writes: stub `vault.modify`, `vault.process`, and
     `editor.replaceRange` to throw.
   - Ready, Next, Blocked `[?]`, `[x]`, and `[-]` reading tasks all open. `#REF` casing
     works.
   - A v1 tracker `[[lib/chat/x.pdf]] #hide ^ref` opens through the PDF link. A v1
     tracker without a link falls back to the note's own `source_pdf`.
   - A Task Link bullet in a daily note opens in its plain, embedded, struck, and 🍅
     forms. So does the bare managed embed line `![[sase#^ref-x]]` in a ref note.
   - These **delegate exactly once** without calling `execFile`: a non-ref `#task` line,
     prose, a heading, a Task Link to a non-ref task, a Task Link to a missing block, a
     `#task #ref` line inside a code fence, and focus outside the editor.
   - These show notices **without** delegating: a reading task whose ref note lacks
     `source_pdf`, an unresolvable ref link, a missing PDF on disk, and unsafe
     `source_pdf` values (`../x.pdf`, `/abs/x.pdf`, `ref/x.md`).
   - A `-b` failure retries with `-a Highlights`. When both fail, the notice carries the
     first stderr line.
   - The off-macOS behavior matches the chosen `non_mac` branch.
   - The `ref_note_scope` behavior matches the chosen branch on a non-task line of a
     managed ref note.
   - Command registration: id `open-reference-pdf-in-highlights`, no `hotkeys` property.
5. **Version and docs** in bob-plugins:
   - Bump `plugins/bob-navigation-hotkeys/manifest.json` from `2.41.0` to `2.42.0`, and
     add one sentence to its `description`.
   - Update the README Plugins table row: bump the version and append one concise
     sentence.
   - Add a short README section, `## Open reference PDF in Highlights`, after the Go to
     task URI section. It covers the trigger lines, resolution order, notices, the
     never-writes guarantee, and why the binding lives in `hotkeys.json` as `Meta+Enter`
     with delegation. Warn that re-recording it in Obsidian Settings on the Mac may save
     `Mod`, which would claim Ctrl+Enter on Linux.
6. **Build and verify** in bob-plugins: `npm run build`, then `npm test` (this includes
   `build:check`), then `npm run validate`. Fix everything until green. Confirm every
   hand-edited fragment is at or under 1000 lines.
7. **Deploy to athena's vault.** Run
   `bob plugins sync -r <bob-plugins path printed by sase repo open> -n -p bob-navigation-hotkeys -d`,
   then run it again without `-d`.
8. **Bind the key in the vault.**
   - Run `bob vault-sync` first so you edit the latest `~/bob/.obsidian/hotkeys.json`.
   - Add exactly one entry and leave every existing entry unchanged:
     `"bob-navigation-hotkeys:open-reference-pdf-in-highlights": [{ "modifiers": ["Meta"], "key": "Enter" }]`.
     Keep Obsidian's `JSON.stringify(…, null, 2)` formatting and the trailing-newline
     state. `git -C ~/bob diff .obsidian/hotkeys.json` must show only the added lines.
   - Validate with `python3 -I -m json.tool`.
   - Run `bob vault-sync` again, then confirm `bob vault-sync status --json` reports
     success. Do not hand-commit in the vault; vault-sync owns its commits.
9. **bob-cli docs.** In `docs/ref.md`, next to the existing Refs-panel sentence, add one
   sentence. It says that in Obsidian, `Cmd+Enter` on a reading task (or a Task Link or
   managed embed pointing at one) opens its `source_pdf` in Highlights through
   bob-navigation-hotkeys, without writing the vault.

## Verification

- Automated: the step 6 commands pass, and the new test file runs inside `npm test`.
- athena (Linux Obsidian, if running): after a plugin reload, `Cmd`/`Super+Enter` on a
  reading task shows the `non_mac` branch's result. `Ctrl+Enter` still closes tasks.
  `Mod+Enter` (Ctrl+Enter) behavior is unchanged.
- The Mac can't be checked from athena; report this to Bryan as not done. After the
  bob-plugins commit lands, Bryan runs `ssh mac 'bob plugins sync'` (or
  `just install-all` on the Mac) and lets vault-sync deliver `hotkeys.json`. Then he
  checks:
  - `Cmd+Enter` on a v2 reading task, a Pomodoro Task Link to it, and the ref note's
    managed embed opens the PDF in Highlights;
  - `Cmd+Enter` on an ordinary wikilink still opens it in a new tab;
  - `Cmd+Enter` in the file explorer still opens the focused file in a new tab;
  - the Task Card / decay card `Cmd+Enter` alias still applies the recommendation;
  - `Ctrl+Enter` still closes tasks.

## Out of scope

- Opening at a specific page or annotation.
- Bob Mac Capture changes or a Highlights-app override setting.
- Any `bob` CLI command.
- Changing how reading tasks or ref notes are written.
