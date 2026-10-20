---
tier: epic
title: Z zooms the image on the cursor line in a read-only viewer
goal: 'In Obsidian''s Vim normal mode, pressing Z on a line holding an image embed,
  a plain image wikilink, or a Markdown image opens that image fitted to the window
  in a small bob-owned viewer with keyboard and mouse zoom and pan. Esc, q, Ctrl+[
  or Z return to the same note, cursor, scroll, and normal mode, and no part of the
  gesture ever writes the vault. Obsidian''s native click and Zoom-button lightbox
  keep working unchanged.

  '
decisions:
  image_zoom_record:
    ask: Add a decisions record that Z opens a read-only owned image viewer and native
      mouse paths stay native?
    memory:
    - decisions
    default: false
    answer: false
phases:
- id: target
  title: Line-scoped image target resolver
  depends_on: []
  size: medium
  description: 'target: add the bob-navigation-hotkeys image-zoom mixin fragment that
    finds embed, wikilink, and Markdown image references on the live cursor line,
    skips inline and fenced code, picks one by the containing/next/previous rule,
    and resolves it against the source note to a resource URL or an exact notice,
    with fixture tests.'
- id: viewer
  title: Owned ImageZoomModal viewer
  depends_on: []
  size: medium
  description: 'viewer: add a public-API Modal fragment plus CSS that opens an image
    fitted to the window with bounded zoom, device-pixel 1:1, keyboard/drag/wheel
    pan and zoom, backdrop and key close, a guarded opening keystroke, and full listener
    cleanup, with geometry, key-map, and lifecycle tests.'
- id: wire
  title: Zoom command, Z binding, docs, and deploy
  depends_on:
  - target
  - viewer
  size: medium
  description: 'wire: register "Zoom image on cursor line", open the viewer from the
    resolver, restore the source editor on close, bump the manifest, document the
    gesture, map nmap Z in the vault vimrc, deploy with bob plugins sync, and hand
    Bryan the Mac smoke checklist.'
proposed_by: bbugyi200.apollo.research.26.linker.w0
proposed_with: claude/opus
decided_by: auto
create_time: 2026-10-20 09:07:39
status: done
bead_id: bob-cli-a1
---

- **PROMPT:** [prompts/202610/z_image_zoom.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202610/z_image_zoom.md)
- **BEAD:** [bob-cli-a1](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-a1/README.md)

# Plan: `Z` zooms the image on the cursor line

## Background

Bryan accepted every requirement adjustment (A1–A8) in
`research:202610/obsidian_z_image_inspection/obsidian_z_image_inspection.md`. That
report is the design source. Read its "Recommended design" and "Viewer controls"
sections before starting any phase. The facts below were re-verified while planning.

- **Native lightbox gaps.** Obsidian 1.13+ ships a fullscreen lightbox, but it is
  mouse-only. No command opens it, and Vim normal mode consumes its `Space` key
  (`<Space>` maps to `l`). Plain image links such as `[[img/x.png]]` (239 in 117 notes,
  against 16 embeds) never reach it.
- **`Z` is free.** `Z`, `ZZ` and `ZQ` are unbound in CodeMirror Vim, in Obsidian's Vim
  additions, in `obsidian_vimrc.md`, and in bob-plugins. A `Z` mapping fires
  immediately, with no wait for a second key.
- **Repository facts (bob-plugins @ `ff36987`, `plugins/bob-navigation-hotkeys/`):**
  - Fragments are listed in `src/fragments.json`. `npm run build` generates `main.js`,
    which is never hand-edited. Each hand-edited fragment stays at or under 1000 lines.
    `npm test` begins with `build:check`, and every test file is registered in the
    single-line `test` script of the root `package.json`.
  - Mixins are installed in `src/690-install-plugin-mixins.js`, which rejects duplicate
    method names. Free functions and classes that tests need are exported through
    `module.exports.helpers` in `src/700-exports.js`.
  - Link helpers (prototype methods in `src/680-plugin-link-parse.js`):
    `parseWikiLinkAt`, `extractMarkdownDestination`, `normalizeLinkTarget` (unquotes,
    unwraps `<…>`, applies `safeDecodeUri`), `stripLinkSubpath`, and
    `resolveLinkTargetFile(target, sourcePath)`. The last one strips `.md` and the
    subpath, then calls `metadataCache.getFirstLinkpathDest(lookup, sourcePath)`.
    **`parseMarkdownLinkAt` rejects `!`-prefixed links, so it cannot parse
    `![alt](dest)`.**
  - Span helpers (free functions in `src/180-editor-transactions.js`):
    `parseTransclusionToggleTargetAt(line, index)` recognizes `![[`, `[[`, `![` and `[`
    forms and returns `markerIndex`, `startIndex` and `endIndex`. Also there:
    `findNonWikiBracketGroupEnd`, `findClosingMarkdownLabelBracket`,
    `findClosingMarkdownDestinationParen` (handles `<…>` and balanced or escaped
    parentheses) and `hasMarkdownDestination`. `findTransclusionToggleTargets` also
    refuses Depends-On and BLOCKS lines; do not call it.
  - Other helpers:
    - `getMarkdownLineContexts(content)` (`210-…`) returns `{inFrontmatter, inFence}`
      for each line.
    - `isExternalLinkTarget` and `normalizeVaultRelativePath` live in `090-…`.
    - `isCtrlLeftBracketKeydown` and `isTaskCardCloseKeydown` (Esc, bare `q`/`Q`,
      `Ctrl+[`) live in `470-…`.
    - `contentElCleanup` lives in `190-…`.
  - The closest sibling is `src/677-plugin-ref-pdf-open.js` (Cmd+Enter → PDF),
    registered in `src/490-plugin-lifecycle.js` as `open-reference-pdf-in-highlights`.
    Its tests in `scripts/test-navigation-hotkeys-ref-pdf.cjs` build
    `Object.create(NavigationHotkeysPlugin.prototype)` with a fake `app` whose vault
    write methods throw. That is the pattern to copy.
  - Test harness: `scripts/navigation-hotkeys-harness.cjs` stubs `obsidian`, providing
    only `MarkdownView`, `Modal` (= `ModalStub`), `Notice` (which pushes to `notices`),
    `Plugin` and `parseYaml`. It sets `global.window.setTimeout` to run callbacks
    immediately.
    - **`ModalStub`** (`scripts/modal-harness.cjs`) has `modalEl` and `contentEl`, but
      no `scope`, `containerEl` or `titleEl`.
    - **`ElementStub`** supports `createDiv`, `createEl`, `addEventListener`,
      `setAttribute` and `classList`, and accepts arbitrary properties.
- **Public Obsidian API** (`obsidian-api` @ `9abd960`):
  - `Modal.open()` "show[s] the modal on the active window", so pop-out windows work.
  - `Modal.scope` is "the keymap scope that is active while the modal is open".
  - `Scope.register(null, null, fn)` captures every key, and a handler that returns
    `false` prevents the default.
  - `Vault.getResourcePath(file: TFile)` returns a browser-loadable URI.

  The viewer needs no undocumented internals.

## Behavior contract (every phase builds toward this)

**Safety (S).**

- **S1. Never writes.** The command and viewer never write the vault or the editor. No
  `|width` edits, no link→embed conversion, no cursor move, no jump-list entry, no Vim
  mode change, and no reads of other notes.
- **S2. Native paths untouched.** Native click, double-click and Zoom-button behavior,
  and Obsidian's own `g+`/`g-`/`:image` verbs, stay exactly as they are. Nothing
  intercepts clicks on rendered images.
- **S3. No new files.** The only vault file touched is `obsidian_vimrc.md`, in the
  `wire` phase. There are no bob-cli or bob-mac-capture changes, because this is not
  part of the capture contract.

**Targeting (T).** These rules are pure and use the live buffer, not `metadataCache`
links.

- **T1. Recognized forms.** Image references are recognized in these forms:
  - `![[t]]` and `![[t|400]]`
  - `[[t]]` and `[[t|alias]]`
  - `![alt](dest)`, `![alt|400](dest)` and `[text](dest)`

  Destinations may use `<…>`, carry a `"title"`, use balanced or escaped parentheses, or
  contain `%20`-style escapes. For vault targets, strip `|alias`/`|size` and `#subpath`.

- **T2. Image test.** A reference is an _image candidate_ when:
  - **Vault target:** its path, with any `?query` removed, ends with `png`, `jpg`,
    `jpeg`, `gif`, `webp`, `svg`, `bmp` or `avif` (case-insensitive).
  - **`http(s)://` target:** it uses embed syntax `![…](url)`, or its URL pathname ends
    with one of those extensions.

  Other URI schemes (`file:`, `app:`, `data:`, `obsidian:`, …) are never candidates.
  Nothing is fetched to find out whether a link is an image.

- **T3. Code is skipped.** References inside an inline code span are ignored. Code spans
  follow CommonMark backtick runs: a run of N backticks closes only on a run of exactly
  N, and an unmatched run is literal. A line inside a fenced code block has no
  candidates. Frontmatter lines are scanned like any other line.
- **T4. Pick order.** Take the candidate whose `[start, end)` contains the cursor `ch`,
  else the first candidate starting at or after `ch`, else the last one ending at or
  before `ch`. Here `start` is the `!` for embeds and the `[` otherwise. Never look
  beyond the current line.
- **T5. Resolution.**
  - **Vault target:** `resolveLinkTargetFile(target, sourceNotePath)`. Use the source
    note's path, never search by basename, and never build `file://` URLs. A resolved
    file whose extension is not in the T2 set is "not an image". Otherwise
    `src = app.vault.getResourcePath(file)` and `title = file.name`.
  - **URL target:** `src` is the URL exactly as written. `title` is the decoded last
    pathname segment, else the hostname.
- **T6. Notices** (exact text):
  - **No candidates:** `No image on this line`, including fenced lines and empty lines.
  - **Cursor inside a non-image reference:** when the line has no candidates and the
    cursor sits inside a recognized non-image reference, such as a note or `.pdf`,
    `Not an image: <target>`.
  - **Unresolved target:** `Image not found: <target>`.

  Notices are the only side effect of a failure.

**Viewer (V).**

- **V1. Opening state.** The viewer opens fitted (`contain`) to the window minus a
  padding of 32 CSS px on each side. It never upscales automatically past natural size
  (`fit = min(1, boxW/natW, boxH/natH)`). It always loads the original file, so a `|400`
  hint is ignored.
- **V2. Zoom bounds.** Zoom steps are ×1.2, matching native, within these bounds:
  - `min = min(fit, oneToOne)`
  - `max = 8 × max(fit, oneToOne, 1)`
  - `oneToOne = 1 / devicePixelRatio`, the device-pixel 1:1 scale at which a Retina
    screenshot looks as it did when captured

  The readout shows `round(scale × devicePixelRatio × 100)%`, so `1` reads `100%`.

- **V3. Viewer controls.** Every key is captured on the modal scope and none reaches the
  note or Obsidian's global hotkeys.
  - **Close:** `Esc`, `q`/`Q`, `Ctrl+[`, `Z`. Close keys ignore `event.repeat`.
  - **Zoom in / out:** `+` and `=` zoom in, `-` zooms out. Both act on the viewer only
    and never resize the embed.
  - **Fit / 1:1:** `0` fits to the window, `1` shows device-pixel 1:1.
  - **Pan:** `h`/`j`/`k`/`l` and the arrow keys pan by 15% of the viewport. With `Shift`
    the step is ×3.
  - **Wheel and pinch** (`wheel` with `ctrlKey`): zoom toward the pointer.
  - **Drag:** pans, but only when the image overflows the viewport.
  - **Double-click:** toggles fit ↔ 1:1 at the pointer.
  - **Click outside the image:** closes, unless the click ended a drag.
  - **Any other key:** swallowed, with no effect.
- **V4. Closing returns to the editor.** It returns to the same note, cursor, scroll,
  and Vim normal mode. If that editor is no longer the active view, focus is not stolen.
  `ZZ` is a harmless open-then-close.

## Shared conventions (all phases)

- **Where to work.**
  - Open bob-plugins with `/sase_repo` (`sase repo open bob-plugins`) and work only in
    the path it prints.
  - Follow its `AGENTS.md`: edit `src/` fragments, run `npm run build`, and never
    hand-edit `main.js`.
  - New fragments must be added to `src/fragments.json`, and every new
    `scripts/test-*.cjs` file must be appended to the root `package.json` `test` script.
- **Comments and naming.** Match the surrounding code: short `//` comments explaining
  _why_, frozen result objects, `bob-image-zoom-*` CSS classes, and guards around
  optional Obsidian members so the Node harness runs.
- **Parallel landing.** `target` and `viewer` may run at the same time. They share three
  things:
  - `src/fragments.json`, at distant lines, so it merges cleanly
  - the one-line `package.json` `test` script
  - generated `main.js`

  When a rebase conflicts on these:
  - keep both test-file entries
  - regenerate `main.js` with `npm run build`, never by hand-merging
  - rerun `npm test`

- **Version and description.** Only `wire` bumps the manifest version and appends to its
  description. `target` and `viewer` add dormant code with no command, and leave the
  manifest unchanged.
- **Validation (every phase).** `npm test`, `npm run validate`, and `git diff --check`
  must all pass.
- **Deploy (every phase).** Bob-plugins requires `bob plugins sync` after any change.
  Run it scoped: `--repo <bob-plugins path> --no-pull --plugin bob-navigation-hotkeys`.
  Run `--dry-run` first, then the same command without it. Never add `--force` to get
  past an unexpected dirty-destination refusal. Report that refusal instead.

## Phase `target`: line-scoped image target resolver

Add `src/679-plugin-image-zoom.js`, listed right after `678-plugin-templater-trigger.js`
in `fragments.json`. It defines `BobNavigationHotkeysImageZoomMixin` and an
`IMAGE_ZOOM_EXTENSIONS` frozen set. Install the mixin in `690-install-plugin-mixins.js`
next to `BobNavigationHotkeysRefPdfOpenMixin`. Pure parts are mixin methods that never
touch `this.app`, so they can reuse the `680` prototype helpers. Tests call them through
the prototype, so `700-exports.js` needs no change in this phase.

1. **`findInlineCodeSpans(line)`** returns frozen `[start, end)` ranges per T3.
2. **`findImageZoomRefs(line)`** walks the line the way `findTransclusionToggleTargets`
   does, using `parseTransclusionToggleTargetAt`, then `findNonWikiBracketGroupEnd`,
   else `+1`. It does this _without_ that function's Depends-On/BLOCKS refusals, and it
   drops spans whose `markerIndex` falls inside an inline code span.
   - **Output.** It returns frozen refs:
     `{ form: "wiki" | "markdown", embed, start, end, rawTarget, target, external }`.
   - **Wiki targets:** the text between `[[` and `]]`, up to the first `|`, passed
     through `normalizeLinkTarget`. Then `stripLinkSubpath`.
   - **Markdown targets:** find the destination with `findClosingMarkdownLabelBracket`
     and `findClosingMarkdownDestinationParen`, then decode it with
     `extractMarkdownDestination`.
   - **External targets** (`isExternalLinkTarget`) keep the whole URL, including any
     `#`. Vault targets also drop the subpath.
3. **`isImageZoomCandidate(ref)`** applies T2. For URLs, parse with `new URL()` inside
   `try` and test its `pathname`. Only `http:` and `https:` qualify.
4. **`pickImageZoomRef(candidates, ch)`** applies T4 and returns a ref or `null`.
5. **`resolveImageZoomTargetOnLine({ lineText, ch, sourcePath, inCodeFence })`** is the
   only method that uses `this.app`. It applies T5 and T6 and returns one of:
   - `{ kind: "image", source: "vault", path, src, title }`
   - `{ kind: "image", source: "url", src, title }`
   - `{ kind: "notice", message }`

   It never reads or writes note content.

**Tests:** `scripts/test-navigation-image-zoom-target.cjs`.

- **Parser fixtures from real vault lines:**
  - `- ![[img/20260901_085041.png|400]]`
  - `See [[img/vim_veteran_redir.png]]`
  - `> ![[ref/a.assets/b.png|549]]`
  - `> [!quote] Image ![[img/x.png]]`, where the callout bracket must not confuse the
    scan
  - an image link inside single and double backticks, which is ignored
  - `[[note]]` and `[[doc.pdf]]`, which are not candidates
  - two image refs on one line
  - a line with no refs
  - `![alt](<img/Screen Shot.png>)`
  - `![](img/a%20b.png "t")`
  - `[x](img/p(1).png)`
- **Picker:** cursor before, inside (at `start`, inside, at `end - 1`), between and
  after refs, on a line with one ref and with two.
- **Resolution** against a fake `metadataCache.getFirstLinkpathDest` that records its
  `sourcePath`:
  - duplicate basenames resolved per source note
  - a relative `../img/x.png`
  - Unicode and space names
  - a missing file → `Image not found: …`
  - cursor on a `[[doc.pdf]]` with no candidates → `Not an image: doc.pdf`
  - a fenced line → `No image on this line`
  - `getResourcePath` called with the resolved file
  - `![](https://e.com/render?id=1)` passes through as-is
  - `[x](https://e.com/page)` is not a candidate
  - `[x](https://e.com/a.PNG?v=2)` is a candidate
  - every vault read and write stub throws, and none of them is called

## Phase `viewer`: owned `ImageZoomModal` viewer

Add `src/207-image-zoom-modal.js`, listed right after `206-successor-picker-modal.js`.
Append a commented `/* Image zoom viewer */` block to
`plugins/bob-navigation-hotkeys/styles.css`. Export `ImageZoomModal` and the pure
helpers through `700-exports.js` `helpers`. The fragment holds free functions plus the
class, roughly 250–350 lines. Keep it standalone so it could later be extracted.

**Pure helpers.** These are deterministic, with no DOM. Use the constants
`IMAGE_ZOOM_STEP = 1.2`, `IMAGE_ZOOM_PADDING = 32`, `IMAGE_ZOOM_MAX_FACTOR = 8` and
`IMAGE_ZOOM_PAN_FRACTION = 0.15`.

- **`computeImageZoomFitScale(natural, viewport, padding)`** and
  **`computeImageZoomBounds(fit, oneToOne)`** implement V1 and V2.
- **`zoomImageZoomState(state, factor, point, ctx)`** sets the new scale, clamped to the
  bounds. It keeps the image point under `point` fixed:
  `offset' = p − (p − offset) × (s'/s)`. Offsets are the image centre relative to the
  viewport centre.
- **`clampImageZoomPan(state, natural, viewport)`** centres an axis that fits and clamps
  an overflowing axis to `±(scaled − viewport)/2`.
- **`imageZoomKeyAction(event)`** maps an event to an action:
  - **Actions:** `close`, `zoomIn`, `zoomOut`, `fit`, `actual`,
    `pan:{left,right,up,down}` with a `large` flag for `Shift`, or `ignore`.
  - **Close keys** are matched with `isTaskCardCloseKeydown`, plus `Z`
    (`event.key === "Z"`, with no Ctrl, Meta or Alt).
  - **Composition:** `isComposing` and keyCode 229 → `ignore`.
- **`imageZoomWheelFactor(event)`** returns `exp(−deltaY × k)`. Here
  `k = event.ctrlKey ? 0.01 : 0.0015`, and `deltaY` is multiplied by 16 when
  `deltaMode === 1`.

**`ImageZoomModal extends Modal`.** The constructor is
`constructor(app, { src, title, onClosed, defer })`. `defer` defaults to
`(cb) => window.setTimeout(cb, 0)` and is injectable so tests can hold the arming tick.

1. **DOM (`onOpen`).**
   - Add `bob-image-zoom-container` to `containerEl` (guarded) and
     `bob-image-zoom-modal` to `modalEl`.
   - Build a header (title, plus the zoom readout `bob-image-zoom-readout`), a full-size
     viewport (`bob-image-zoom-viewport`, `tabindex="-1"`), an
     `<img class="bob-image-zoom-img" draggable="false" alt=title>`, and a subdued
     footer hint: `Esc · q · Z close   + / − zoom   0 fit   1 actual   hjkl pan`.
   - While loading, show `Loading…`.
2. **Keys.**
   - If `this.scope` exists, register a single catch-all
     `this.scope.register(null, null, (evt) => { this.handleImageZoomKey(evt); return false; })`.
     Keep the handle so it can be unregistered.
   - **Arming guard.** Every key is ignored until the `defer` callback fires, so the
     opening `Z` keystroke cannot close the viewer. That callback also focuses the
     viewport element, so stray keys never land in the editor. The deferred focus wins
     even if Vim's ex-command completion refocuses the editor first.
   - Do not also add a modal `keydown` listener, which would double-handle keys.
   - **Idempotent close.** Obsidian's `Modal` binds `Escape` in its own scope too, so
     closing must be idempotent. Every handler returns early once the modal is closed,
     and `onClosed` still fires once.
3. **Load.**
   - Attach `load`/`error` listeners with one `AbortController` `signal` that every
     listener in the modal shares, then set `src`.
   - **On load,** read `naturalWidth`/`naturalHeight`. When they are 0 (an SVG without
     intrinsic size), treat the padded fit box as the natural size.
   - Read the viewport size from the viewport element's `clientWidth`/`clientHeight`,
     falling back to the window's `innerWidth`/`innerHeight`.
   - Start in "fit mode" and render.
   - **On error,** show `new Notice("Could not load image: <title>")` and close.
   - Every async callback returns early once the modal is closed.
4. **Render.**
   - Set the `<img>` width and height to `natural × scale` CSS px rather than using a
     scale transform, which keeps it crisp.
   - Position it with `left/top: 50%` plus
     `translate(-50%, -50%) translate(x px, y px)`.
   - Update the readout, and toggle `is-pannable` on the viewport when either axis
     overflows.
   - Read `devicePixelRatio` from the modal document's `defaultView`, falling back to 1.
5. **Mouse and resize.**
   - **Wheel:** a non-passive listener that calls `preventDefault`.
   - **Drag:** pointer events with guarded `setPointerCapture`. Past a 4 px threshold,
     suppress the click that follows.
   - **Double-click** and **backdrop click** behave per V3. Backdrop click means the
     pointer is outside the rendered image rect.
   - **Window `resize`** on the modal's own window recomputes fit. In fit mode it
     refits. Otherwise it keeps the scale, re-clamped to the new bounds, and re-clamps
     the pan. Fit mode lasts until a zoom, pan, or `1`, and `0` restores it.
6. **`onClose`.**
   - Mark the modal closed, abort the controller, and unregister the scope handler.
   - Remove the `<img>` `src` attribute so a pending load is cancelled.
   - Call `contentElCleanup(contentEl)`.
   - Call `onClosed` exactly once, inside `try`.

**CSS.**

- `.modal-container.bob-image-zoom-container .modal-bg` gets an opaque black at 0.9
  alpha, matching the native lightbox.
- `.modal.bob-image-zoom-modal` fills 100vw × 100vh: no max size, padding, border,
  radius, shadow or background.
- Hide its `.modal-close-button`.
- The viewport is
  `position: absolute; inset: 0; overflow: hidden; user-select: none; touch-action: none`.
- The `<img>` uses `max-width: none`, with `cursor: grab`/`grabbing` when pannable.
- The header and footer use light text on a translucent dark background,
  `--font-ui-small` and `pointer-events: none`, so clicks fall through to the viewport.
- Never add or target `.lightbox`, because native CSS and themes style it.

**Tests:** `scripts/test-navigation-image-zoom-viewer.cjs`.

- **Geometry:**
  - fit never exceeds 1, and it respects padding for landscape, portrait and tiny images
  - the min and max bounds
  - stepping clamps at both bounds
  - zoom-at-point keeps the point fixed
  - clamp centres small axes and limits large ones
- **Key table:**
  - every V3 key, including `Q`, `Shift+=`/`+`, `HJKL` and the arrows
  - `Ctrl+q` and `Meta+z` → `ignore`
  - composing → `ignore`
- **Lifecycle,** on `ModalStub`: give the modal a fake `scope` that records
  `register`/`unregister`, and a manual `defer` queue.
  - keys before arming are ignored
  - `Z` closes after arming, and a repeated `Z` does not
  - load sets fit state and dimensions
  - a load error → notice, closed, `onClosed` called once
  - each close key → `onClosed` once, scope handler unregistered, signal aborted
  - a stray `x` key changes nothing
  - the app's vault stubs throw and are never called

## Phase `wire`: command, `Z` binding, docs, and deploy

1. **`zoomImageOnLine(editor, view)`** in `679-plugin-image-zoom.js` runs these steps:
   1. **Guard.** Return quietly when the view, file or editor is missing.
   2. **Reading view.** When `view.getMode && view.getMode() !== "source"`, show
      `Zoom image needs the editor; in Reading view, click the image` and stop.
   3. **One viewer at a time.** Return when `this.activeImageZoomModal` is already open.
   4. **Resolve.** Read `head = editor.getCursor("head")` and
      `lineText = editor.getLine(head.line)`. Read the fence state from
      `getMarkdownLineContexts(editor.getValue())[head.line]`. Call
      `resolveImageZoomTargetOnLine`. A notice result shows `new Notice(message)` and
      stops.
   5. **Snapshot.** Capture `{ view, filePath, editor, anchor, head, scroll }`, where
      `scroll` is `editor.getScrollInfo()` when it exists.
   6. **Open.** Open `new ImageZoomModal(this.app, { src, title, onClosed })`. Store it
      in `this.activeImageZoomModal`. `onClosed` clears that field and calls
      `restoreImageZoomSourceEditor(snapshot)`.

   **`restoreImageZoomSourceEditor(snapshot)`** runs one tick later, and only when all
   three hold: the plugin is not unloading, the active `MarkdownView` is still
   `snapshot.view`, and its file path is unchanged. Then it:
   - calls `setSelection(anchor, head)` only if the selection differs
   - calls `scrollTo(left, top)` only if the scroll differs
   - calls `editor.focus()`

   Otherwise it does nothing. It never touches Vim state or jump history.

2. **Command registration.** In `src/490-plugin-lifecycle.js`, directly after
   `open-reference-pdf-in-highlights`, add this, with no `hotkeys`:

   ```js
   // The vault vimrc maps normal-mode Z to this command; it has no default hotkey.
   this.addCommand({
     id: "zoom-image-on-line",
     name: "Zoom image on cursor line",
     editorCallback: (editor, view) => this.zoomImageOnLine(editor, view),
   });
   ```

   Initialize `this.activeImageZoomModal = null` in `onload`. In `onunload`, close it
   (guarded) and clear it.

3. **Tests:** `scripts/test-navigation-image-zoom-command.cjs`.
   - **Registration.** Follow the `onload` stub pattern from
     `test-navigation-open-notes-uri.cjs`, but have `addCommand` record each spec.
     Assert the id, the name, `editorCallback` and the absence of `hotkeys`.
   - **Opening.** An embed line and a plain-link line each open one modal with the
     resolved `src`.
   - **Notices.** No image, missing image and Reading view each show their notice and
     open nothing. A second invocation while open opens nothing.
   - **Close and restore.** Closing restores focus when the active view is unchanged,
     without calling `setSelection` or `scrollTo` when nothing moved. Closing does not
     call `focus()` when another view is active. Unload closes an open viewer.
   - **No writes.** The editor's `replaceRange`/`setValue` and the vault write stubs
     throw, and none of them is called.
4. **Manifest.** Bump `plugins/bob-navigation-hotkeys/manifest.json` one minor version
   from the current value (2.46.1 was inspected, so 2.47.0). Append one sentence to its
   description:

   > `Z` (mapped in the vault vimrc) or the "Zoom image on cursor line" command opens
   > the image on the cursor line — an embed, image link, or Markdown image — in a
   > read-only full-window viewer with keyboard and mouse zoom and pan; it never writes
   > the vault, and native image clicks are unchanged.

5. **Docs.** The plugin reference is the documentation route for a new hotkey.
   - Add `docs/image-zoom.md`, modeled on `docs/highlights.md`, with the title "Zoom an
     image on the cursor line" and a one-line description. It contains:
     - the targeted forms and the pick rule
     - the viewer key and mouse table
     - "never writes the vault"
     - native click, double-click and the Zoom button still own embeds and Reading view
     - `g+`/`g-`/`:image grow` resize embeds and write `|width`
     - `Z` comes from the vault `obsidian_vimrc.md`, not a plugin default
     - out of scope: galleries, counts, PDFs (Cmd+Enter), transcluded notes, mobile
     - a Related section
   - Add it to `docs/nav.yml` after "Open reference PDF in Highlights".
   - In `docs/bob-navigation-hotkeys.md`'s "Notes, tabs, and keyboard layers":
     - add one sentence about `Z`
     - add a clause to the custom-binding bullet: the vault vimrc maps `Z` to
       `zoom-image-on-line`
     - add a Related link
   - Run `npm run docs:gen` to refresh the generated facts and commands tables. Then run
     `npm run docs:check` and `npm test`. The bob-plugins README is a short front door
     and does not change.
6. **Deploy, then bind.** Deploy first, so the command exists before the key that calls
   it.
   - Run the scoped `bob plugins sync` (dry-run, then real) from Shared conventions.
   - In the Bob vault's `obsidian_vimrc.md`, append this block after the review
     mappings. Use the existing `obcommand` bridge: do not enable vimrc JavaScript or
     add a plugin `mapCommand` for `Z`.

     ```vim
     " Z opens the image on the cursor line in a read-only viewer (never resizes).
     " g+ / g- / :image grow resize a selected embed and write |width into the note.
     exmap bob_zoom_image obcommand bob-navigation-hotkeys:zoom-image-on-line
     nmap Z :bob_zoom_image<CR>
     ```

   - Deliver the vimrc change through the git vault sync workflow
     (`docs/vault-git-sync.md` in bob-cli), as the `[S`/`]S` mapping change did. Include
     the vault and plugin changes in SASE finalization, and preserve unrelated vault
     changes. Do not re-enable Obsidian Sync.

7. **Hand off the smoke checklist.** Agent hosts cannot drive the Mac UI. Say so in the
   final report, and do not claim a UI pass. Give Bryan this checklist:
   1. Reload Bob Navigation Hotkeys and the vimrc.
   2. Optionally, the 10-second native check from research §Risks step 1.
   3. `Z` on a `- ![[img/…|400]]` line and on a `See [[img/….png]]` line opens the
      viewer fitted to the window. `1` is crisp, `+`/`-`/`0`, `hjkl`, drag and wheel or
      pinch work, and double-click toggles.
   4. `Esc`, `q`, `Ctrl+[` and `Z` each return to the same cursor, scroll and normal
      mode. `ZZ` opens then closes.
   5. `x` and `dd` pressed inside the viewer change nothing.
   6. A line with no image shows a notice only.
   7. In Insert and Visual mode, `Z` behaves as before.
   8. The viewer works in a pop-out window, in Source mode, and from the palette with
      Vim off.
   9. `git -C ~/bob diff` is empty apart from the vimrc commit.

> [!decision] image_zoom_record Use your `/sase_memory_write` skill. In the bob-cli
> repo, add one strand to the `decisions` web, slug `image-zoom-is-a-read-only-viewer`,
> following the existing records' shape: Applies to, Claim, Why with the rejected
> alternatives, Cost, and Reopens when.
>
> - **Applies to:** bob-plugins `bob-navigation-hotkeys` and the Bob vault vimrc.
> - **Claim:** `Z` opens a read-only, plugin-owned viewer for the image on the cursor
>   line, and native click and Zoom-button paths stay native.
> - **Rejected alternatives:**
>   - click interception or a community lightbox plugin, which would fight native clicks
>   - native-lightbox delegation through the undocumented `livePreview.showImage`, which
>     fails silently, splits the key vocabulary, and covers only 16 embeds
>   - resize verbs, which write `|width`
> - **Reopens when:** Bryan prefers the native feel for embeds after using v1.
> - **Evidence:** cite the research ref above and the implementing commits.
>
> Then run `sase memory init`.

## Epic acceptance

- **Checks pass.** `npm test`, `npm run validate`, `npm run docs:check` and
  `git diff --check` pass in bob-plugins. `bob plugins sync` reports
  bob-navigation-hotkeys in sync.
- **Command registered.** The generated commands table lists `Zoom image on cursor line`
  / `zoom-image-on-line` / `—`.
- **Vimrc mapped.** The vault vimrc has exactly one `Z` mapping, pointing at that id.
- **No writes.** No test observes an editor or vault write on any path.

## Out of scope (research follow-ons; file only if missed after use)

- Native-lightbox delegation for embeds.
- A note-wide gallery (`]`/`[`).
- Counted `NZ`.
- `Enter`/`[[`/`]]` treating image links as targets.
- `gx` to open in Preview.app.
- A Reading-view keyboard trigger.
- PDFs, Mermaid, Canvas, mobile, absolute filesystem paths, and images inside
  transcluded notes.
