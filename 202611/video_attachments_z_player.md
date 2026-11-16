---
tier: epic
title: "Video attachments: %c embeds clips and Z plays them"
goal: "With a video path on the clipboard, `%c` embeds the clip under the new task as
  `![[vid/NAME.mp4|400]]`; in Obsidian the clip sits in the same 400 px column as
  screenshots, and `Z` opens a calm, keyboard-first player that shares the image
  viewer's chrome, origin line, and `y`/`r` gestures. Every attachment bob copies is one
  vault sync can carry, and Bob Mac Capture previews a video as a video.

  "
phases:
  - id: cli-video
    title: "bob-cli: the video attachment kind"
    depends_on: []
    size: medium
    description:
      "cli-video: add AttachmentKind::Video (mp4 mov webm mkv) copied to vid/ and
      embedded at 400 px on every %c route and bob attachments import; add size_bytes to
      attachment JSON and sizes to human output; fix the would-save padding; write the
      shared media catalog in docs/attachments.md; mark the commit with !."
  - id: plugin-video-viewer
    title: "bob-plugins: the VideoZoomModal player"
    depends_on: []
    size: medium
    description:
      "plugin-video-viewer: add behavior-preserving media hooks to ImageZoomModal and
      build VideoZoomModal on them (start policy, Space/click play, arrow seeks,
      readout, zoom/pan, o/y/r, in-place failure card, full decoder teardown) with pure
      helpers and Node-harness tests; not yet wired to Z."
  - id: cli-guards
    title: "bob-cli: streamed copies and sync guards for every attachment"
    depends_on:
      - cli-video
    size: medium
    description:
      "cli-guards: stream attachment hashing and copying (no whole-file buffers, abort
      when the source changes after planning), refuse attachments of 95 MiB or more and
      destinations the vault .gitignore ignores, warn at 50 MiB, document the new codes,
      and record the decision strand; mark the commit with !."
  - id: plugin-z-dispatch
    title: "bob-plugins: Z dispatches videos, imports video paths, sizes inline clips"
    depends_on:
      - cli-video
      - plugin-video-viewer
    size: medium
    description:
      "plugin-z-dispatch: teach the Z resolver and path-bullet import about the video
      catalog, open VideoZoomModal for videos, degrade against an old bob, ship the
      inline 400 px embed CSS, update the image-zoom docs, add the real-Obsidian video
      e2e harness, and deploy with bob plugins sync."
  - id: mac-video-card
    title: "bob-mac-capture: videos in the clip card"
    depends_on:
      - cli-video
    size: medium
    description:
      "mac-video-card: count videos in the clip header, show size_bytes, and render a
      cancellable, cached AVFoundation poster with a play glyph and duration badge for
      video rows, staying a thin client of bob's kind; green exact-SHA macOS CI."
  - id: plugin-video-transport
    title: "bob-plugins: scrubber, frame step, speed, mute, fullscreen"
    depends_on:
      - plugin-z-dispatch
    size: medium
    description:
      "plugin-video-transport: add the hairline scrubber with hover time bubble and drag
      seek, frame stepping, playback speed, g/G, mute, fullscreen, and the play/pause
      flash to VideoZoomModal; extend the real-Obsidian e2e and docs; deploy."
decisions:
  memory_video_decision:
    ask: Record the accepted design as decisions:clip-videos-embed-from-vid?
    memory:
      - decisions:clip-videos-embed-from-vid
    default: true
    requested:
      I agree with all of the requirements recommended in that research file (but keep
      in mind that you are responsible for planning and splitting up this work
      appropriately).
    answer: false
proposed_by: bbugyi200.athena.research.63.linker.w0
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-16 17:31:28
status: wip
---

- **PROMPT:**
  [prompts/202611/video_attachments_z_player.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/video_attachments_z_player.md)

# Plan: Video attachments — `%c` embeds clips, `Z` plays them

## Context

Bryan's `macrec` recorder writes small, fast-start H.264 MP4s (2560×1548, 30 fps, no
audio by default, 250–650 KiB for 6–12 s clips, capped at 10 minutes). It mirrors them
to every host and puts the `~/…` path on the clipboard. Today `%c` already copies such a
clip into the vault, but it lands in `file/` as a plain `[[file/x.mp4]]` link that does
not play. `Z` then refuses it with `Not an image`.

This epic follows the consolidated research report
`research:202611/task_video_capture_z_viewer/task_video_capture_z_viewer.md`. Bryan
accepted every requirement in it. Read it with
`sase artifact read research:202611/task_video_capture_z_viewer/task_video_capture_z_viewer.md "<why>"`
before starting a phase. If the research sidecar is not cloned in your workspace, read
it through its explicit artifact ref instead: `file:explicit:8c41a4511100743145bfa1bd`.
This plan is the authority wherever the two differ. The differences are deliberate and
come from the runtime spike below.

Governing memory: `decisions:mac-capture-is-a-thin-client` (bob classifies and the app
renders) and `decisions:capture-sigils-say-where-what-how` (`%c` already says "clip", so
there is no `%v`). Also `decisions:bob-cli-releases-ride-a-daily-train`: never edit the
Cargo.toml version or CHANGELOG.md, and mark contract breaks with `!`.

### Runtime spike (Phase 0, done during planning)

The planner ran the research's Phase-0 questions against real Obsidian on athena. The
setup was Obsidian 1.15.1, Electron 43.8.9, and Chrome 150, in a scratch vault under
`xvfb-run`, driven over CDP in the same way as the bob-plugins `obsidian-*-e2e.cjs`
harnesses. The fixtures were the real screencast plus ffmpeg remuxes and encodes.

| Question                                                                   | Observed                                                                                                                                                                                                                                                |
| -------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| DOM of `![[vid/a.mp4\|400]]`                                               | `div.internal-embed.media-embed.video-embed[width="400"]` wraps `<video controls preload="metadata">`. The `width` attribute is present, but the video renders at the full pane width (591 px). So `\|400` is inert natively, and CSS can make it real. |
| `![[vid/a.mov]]` (no width)                                                | Same wrapper with an `alt` attribute and no `width`.                                                                                                                                                                                                    |
| `.m4v`                                                                     | `metadataCache` resolves it, but Obsidian renders a generic `file-embed mod-generic` with no player. **m4v stays a file.**                                                                                                                              |
| mp4 / mov / mkv (H.264), webm (VP9)                                        | All play in both Live Preview and Reading view, and all seek correctly over `app://`.                                                                                                                                                                   |
| `vid/UP.MP4`, `vid/My clip (1).mp4`                                        | Both render players, so bob keeps the source's extension case and sanitized name.                                                                                                                                                                       |
| HEVC                                                                       | `canPlayType('video/mp4; codecs="hvc1…"')` is empty on Linux. macOS is unverified (VideoToolbox may decode it). The failure card covers the gap.                                                                                                        |
| Detecting audio before play                                                | `audioTracks` is unavailable. After `loadeddata` with `preload="auto"`, `webkitAudioDecodedByteCount` reads 2334 for an AAC clip and 0 for the silent mp4 and webm. **Use this signal.**                                                                |
| Corrupt file                                                               | `error` fires with code 4 (`MEDIA_ERR_SRC_NOT_SUPPORTED`) and an empty message; `play()` rejects with `NotSupportedError`.                                                                                                                              |
| `requestVideoFrameCallback`, `requestFullscreen`, `app.openWithDefaultApp` | All three are present (each is a function).                                                                                                                                                                                                             |
| Typed CSS `attr()`                                                         | `CSS.supports("width", "attr(width px)")` is `true`.                                                                                                                                                                                                    |

### Shared media catalog (the cross-repo contract)

`cli-video` writes this table into bob-cli `docs/attachments.md`, and every other repo
cites it. Extensions are matched case-insensitively.

| Kind    | Extensions                                           | Directory | Written as           | `Z`                                                        |
| ------- | ---------------------------------------------------- | --------- | -------------------- | ---------------------------------------------------------- |
| `image` | avif bmp gif heic ico jpeg jpg png svg tif tiff webp | `img/`    | `![[img/NAME\|400]]` | Image viewer (decodable types: all except heic, tif, tiff) |
| `video` | mp4 mov webm mkv                                     | `vid/`    | `![[vid/NAME\|400]]` | Video player                                               |
| `file`  | everything else (including m4v, avi, ogv)            | `file/`   | `[[file/NAME]]`      | —                                                          |

Why the video set is exactly this: it is the intersection of the formats Obsidian
renders as a player and the vault's git allowlist. `m4v` renders as a generic file
(verified), `avi` does not play in Chromium, and the live vault `.gitignore` ignores
`ogv`. Any `.gitignore` re-include is not needed: the vault allowlist works by extension
(`!*.mp4 !*.mov !*.webm !*.mkv`), so `vid/` files sync as they are.

### What the user sees when done

With `~/tmp/screencasts/20261116_155943.mp4` on the clipboard,
`Fix the walk footer flicker %c` writes:

```markdown
- [ ] #task Fix the walk footer flicker [created::2026-11-16]
  - ![[vid/20261116_155943.mp4|400]]
```

`bob capture` prints `✓ saved      vid/20261116_155943.mp4 · 242.8 KiB`, with the size
dimmed. In Obsidian the clip plays inline at 400 px with native controls. `Z` on that
line opens the black-stage player: a silent clip autoplays in a loop, `Space` pauses,
`←`/`→` seek, and `1` shows a paused frame at 1:1 so small text is readable. `y`/`r`
copy or restore the original path, and `o` hands the clip to the default app. A clip
Obsidian can't decode shows a calm failure card instead of vanishing. Bob Mac Capture's
preview reads "1 video" and shows a poster frame with ▶ and `0:06`.

### Phase graph

```
cli-video ──┬─► cli-guards
            ├─► mac-video-card
            └─► plugin-z-dispatch ─► plugin-video-transport
plugin-video-viewer ─┘
```

`cli-video` and `plugin-video-viewer` start in parallel. `cli-guards`,
`plugin-z-dispatch`, and `mac-video-card` then run in parallel; nothing in the plugin or
the Mac app depends on the guards, because both render bob's error and warning messages
generically.

### Rules every phase follows

- Work in the repository the phase names. Open linked repos with `/sase_repo`
  (`sase repo open bob-plugins …`, `sase repo open bob-mac-capture …`), read their
  `AGENTS.md`, and use only the path the command prints.
- bob-plugins: edit `src/` fragments only and never hand-edit generated `main.js`. Keep
  each hand-edited fragment at or below 1000 lines. Register new fragments in
  `src/fragments.json`, export new pure helpers through `700-exports.js` for tests, add
  new test scripts to `package.json` `test:unit`, then run `npm run build`, `npm test`,
  and `npm run docs:check`. Run `bob plugins sync` after every change, and follow
  AGENTS.md if sync refuses an overwrite. Keep the published URL `image-zoom.md` (rename
  only its title).
- bob-cli: read `sase memory read cli_rules.md docs.md` first. Run the full test and
  lint gate plus `just docs-gen` and `just docs-check`.
- Never write test fixtures into the live vault `~/bob`. Use scratch vaults under
  `/tmp`, and synthesize media with ffmpeg at test time; commit no binary media.

## Phase `cli-video` — bob-cli: the video attachment kind

Files: `src/native/capture_clip/{model,files,tests}.rs`,
`src/native/attachments/{cli,tests}.rs`, and `src/native/capture/output.rs`
(confirmation lines). Also a shared byte formatter, the docs, and the generated CLI
reference.

1. **Kind and catalog.** Add `AttachmentKind::Video`, serialized as `"video"`. Add
   `AttachmentKind::as_str()`, and make `attachments/cli.rs` use it instead of its own
   `match`. Add `VIDEO_EXTENSIONS = ["mp4", "mov", "webm", "mkv"]` next to
   `IMAGE_EXTENSIONS`. `attachment_kind` checks images first, then videos, and
   everything else is `File`. The directory per kind is `img`, `vid`, or `file`.
2. **Embed.** Rename `IMAGE_EMBED_WIDTH` to `EMBED_WIDTH` (400). `attachment_reference`:
   Video → `![[vid/NAME|400]]`. Reuse, hash suffixes, sanitizing, the 10-path cap, the
   origin journal, and rollback are already kind-agnostic; prove that with tests and do
   not change them.
3. **Size in the contract.** Add `size_bytes: u64` to `AttachmentOutput` and serialize
   it. It is additive in `bob capture -f json` `clip.attachments[]` (aggregated history
   entries included) and in `bob attachments import -f json`. Take it from the source
   metadata when planning.
4. **Human output.**
   - Fix `print_clip_file_confirmation`'s padding: `{verb:<10}` glues `would save` to
     the path. Pad to the longest verb plus one space so every row aligns.
   - Append a dimmed ` · 242.8 KiB` to every attachment confirmation (not snippets).
   - `bob attachments import` human rows append the same dimmed size.
   - Move `vault_sync.rs`'s private `human_bytes` into a shared `pub(crate)` helper and
     reuse it (one decimal; B/KiB/MiB).
5. **Help.** `bob attachments import`: about "Copy image, video, and file paths into the
   vault". In the long help, images land in `img/` and videos in `vid/` (both embedded
   at 400px), and other files land in `file/`. `PATH` help reads "Absolute image, video,
   or file path…". Add a `.mp4` example. Regenerate with `just docs-gen`.
6. **Docs (contract first).**
   - `docs/attachments.md`: open with the path → embed → path loop for images _and_
     videos, and add the shared media catalog table above as the single catalog both
     repos cite.
   - `docs/capture.md`: in "Clipboard shapes and limits", videos are copied to `vid/`
     and embedded at 400px.
   - `docs/reference/json-outputs.md`: `kind` is `"image" | "video" | "file"`; add
     `size_bytes` to capture clip attachments and import attachments.
   - `docs/vault-git-sync.md`: one line noting that `vid/` clips sync through the
     extension allowlist.
7. **Tests** (in `capture_clip/tests.rs` and `attachments/tests.rs`; set HOME through
   the existing env guards):
   - The exact example path renders `\t- ![[vid/20261116_155943.mp4|400]]`.
   - Uppercase `.MP4`, quoted, `<…>`, `file://` with percent escapes, and Unicode names.
   - Each of mov, webm, and mkv goes to `vid/`; m4v, avi, and ogv go to `file/`.
   - A mixed png + mp4 + pdf clipboard gives three kinds in order.
   - `%c3` history with a video entry.
   - Reuse of an identical `vid/` file and an 8-hex suffix on a content clash.
   - Import JSON has `kind: "video"`, `embed`, and `size_bytes`.
   - The dry-run padding line and the size suffix.
8. **Commit** with `!`, for example `feat(capture)!: embed clipboard videos from vid/`.
   The documented written output changes from `[[file/x.mp4]]` to `![[vid/x.mp4|400]]`
   even though the JSON change is additive.

## Phase `plugin-video-viewer` — bob-plugins: the `VideoZoomModal` player

Plugin: `plugins/bob-navigation-hotkeys`. This phase builds the player and its tests;
`Z` does not reach it until `plugin-z-dispatch`. Image behavior must not change, and the
existing `test-navigation-image-zoom-*.cjs` suites are the regression net.

1. **Hooks in `ImageZoomModal`** (`207-image-zoom-modal.js`,
   `207a-image-zoom-origin.js`):
   - Extract the media-specific steps into overridable methods whose defaults keep
     today's exact behavior:
     - creating the media element (`<img class="bob-image-zoom-img">`);
     - binding its load and error events;
     - starting the load (setting `src`);
     - the readout text;
     - the primary footer chip list, plus a trailing-chip hook after `y`/`r`;
     - the click-on-media action (none);
     - releasing media on close (removing `src`).
   - The transformed element stays `this.imgEl`, so zoom, fit, 1:1, pan, drag, wheel,
     and double-click are reused unchanged. You may add a `mediaEl` alias.
2. **Pure helpers** in a new `207b-video-zoom-model.js`, exported for tests:
   - `VIDEO_ZOOM_EXTENSIONS`, matching the catalog.
   - `videoZoomKeyAction(event)`, for the keys in the table below.
   - `formatVideoZoomTime(seconds, duration)`: `m:ss.d` under an hour and `h:mm:ss` from
     an hour up.
   - `clampVideoZoomSeek(current, delta, duration)`.
   - `videoZoomStartPolicy({ audioBytes, reducedMotion })` returns
     `{ autoplay, loop, hint }`.
   - `videoZoomReadout(state)`.
   - `videoZoomFailureCopy(code, title)`.
3. **`VideoZoomModal extends ImageZoomModal`** in a new `207c-video-zoom-modal.js`:
   - **Element and classes.** The modal carries both class sets
     (`bob-image-zoom … bob-video-zoom`) so the stage, header, origin line, chips, and
     toast are shared. The element is
     `<video class="bob-image-zoom-img bob-video-zoom-video" preload="auto" playsinline disablepictureinpicture disableremoteplayback>`,
     with no native `controls`.
   - **Natural size.** Take it from `videoWidth`/`videoHeight` at `loadedmetadata`. For
     a 0×0 (audio-only) stream, show a centered `♪ Audio only` placeholder, disable
     zoom, and keep the transport working.
   - **Start policy at `loadeddata`.** `hasAudio` is true when
     `webkitAudioDecodedByteCount > 0`, and also when that value is not a number.
     - Silent clips autoplay with `loop = true`.
     - Audible clips, and every clip under `prefers-reduced-motion: reduce`, open
       paused, with the readout hint `· Space to play` until the first play.
     - Always handle the `play()` promise. A rejection shows the paused state, or the
       failure card for `NotSupportedError`. No late async result may act after close.
   - **Readout** (tabular numbers):
     - `▶︎ 0:03.2 / 0:06.2` while playing and `⏸︎ 0:03.2 / 0:06.2` while paused. Use
       U+FE0E so macOS renders text glyphs, not emoji.
     - Append ` · 150%` only when zoom is off fit, and ` · muted` only when the clip has
       audio and is muted.
     - Drive it with `requestAnimationFrame` while playing so tenths tick smoothly, and
       cancel the loop on pause and close.
   - **Clicks.** A click on the video toggles play and pause. Drags that pan suppress
     that click, a click on the backdrop closes the modal (image parity), and a
     double-click toggles fit and 1:1.
   - **`o`.** Pause, then call `this.app.openWithDefaultApp(vaultPath)` (verified
     present) and toast `✓ Opened in default app`. If the call is missing or throws,
     toast `! Can't open outside Obsidian here`.
   - **Failure card.** Do not close on `error`. Show a centered card in the toast
     styling (rgba(0,0,0,.72), 1 px rgba(255,255,255,.2) border, 12 px radius, max-width
     420 px):
     - Title: `Can't play NAME here`.
     - One muted reason line, chosen by error code:
       - 4: "Obsidian's player can't decode this file — the codec may be unsupported, or
         the file may be damaged."
       - 3: "Decoding stopped partway — the file may be damaged."
       - 2: "Obsidian couldn't read the file."
       - otherwise: "Playback failed."
     - A chip row: `o open in default app · y copy path · Esc close`.
     - In this state the footer drops the transport and zoom chips.
   - **Teardown** (close and plugin unload, which already closes
     `activeImageZoomModal`):
     - pause, remove `src`, and call `load()` to release the decoder;
     - cancel the rAF and frame callbacks and abort every listener;
     - restore the editor exactly as images do;
     - leave no sound and no listeners behind.
4. **Footer chips** (two lines at most at 1280 px): `Esc close`, `Space play`,
   `← → seek`, `+ − zoom`, `0 fit`, `1 1:1`, then a divider, then `y …`, `r …`, and
   `o open`. Drop the `hjkl pan` chip for videos; the docs list it.
5. **CSS** (`styles.css`, next to the image-zoom block): add `.bob-video-zoom-*` rules
   for the video element, placeholder, and failure card. Respect
   `prefers-reduced-motion`.
6. **Tests:**
   - Add `scripts/test-navigation-video-zoom-viewer.cjs`. Use a fake `<video>` with
     controllable `readyState`, `duration`, `currentTime`, `paused`,
     `webkitAudioDecodedByteCount`, events, and a `play()` promise.
   - Cover the key map, time format, seek clamps, and start-policy matrix (silent,
     audible, unknown, reduced motion).
   - Cover the `play()` rejection paths, the failure card and its copy, the `o` toast
     paths, and inherited `y`/`r`.
   - Cover close during load, during playback, and during the origin lookup: no `play()`
     after close and `src` is cleared.
   - Assert that the image suites pass unchanged.

| Input                                                 | Action                                                             |
| ----------------------------------------------------- | ------------------------------------------------------------------ |
| `Esc`, `q`/`Q`, `Ctrl+[`, `Z`                         | Pause and close; restore cursor, scroll, and Vim mode              |
| `Space`, click on video                               | Play / pause                                                       |
| `←` / `→`                                             | Seek −/+ 1 s; with `Shift`, −/+ 10 s; clamped to the clip          |
| `+`/`=`, `-`, `0`, `1`                                | Zoom in / out, fit, device-pixel 1:1 (image parity)                |
| `h` `j` `k` `l` (+`Shift`), wheel, drag, double-click | Pan and zoom exactly as images do (arrows belong to the transport) |
| `y` / `r`                                             | Copy the original path / restore the link to it (inherited)        |
| `o`                                                   | Pause and open in the default app                                  |

## Phase `cli-guards` — bob-cli: streamed copies and sync guards for every attachment

These guards close holes that already exist (research §"Pre-existing bugs"): one file of
95 MiB or more wedges every vault-sync cycle on its host, gitignored types such as
`.heic` embed on one host only, and dry-run buffers whole files twice.

1. **Streaming.**
   - `PlannedFile` stops holding `Vec<u8>` for attachments. It holds a source descriptor
     (`Path(PathBuf)` for attachments, `Inline(Vec<u8>)` for snippets) plus the planned
     `size` and `sha256`.
   - Hash through a bounded buffer (64 KiB). Reservation matches compare the stored
     hashes.
   - `file_matches` short-circuits on a size mismatch, then compares streamed hashes.
   - `ClipPlan::save` streams source → temp file while re-hashing. On a size or hash
     mismatch it removes the temp file and fails with
     `clipboard attachment changed after planning: <source>`.
   - The `create_new` fallback in `install_new_file` copies from the temp file instead
     of from memory.
   - `attachments/origin.rs` streams its vault-file and source hashes.
   - Dry-run stays side-effect-free and no longer allocates file-sized buffers.
2. **Size guard** (every kind, before any hashing):
   - Read `fs::metadata(source).len()` first.
   - At or above `vault_sync::LARGE_FILE_LIMIT_BYTES` (95 MiB), refuse the whole capture
     or import. Never silently downgrade to a link.
     - Message:
       `~/tmp/screencasts/x.mp4 is 120.4 MiB; vault sync refuses files of 95 MiB or more, so bob won't copy it — trim or compress it first`.
     - Code: `attachment_too_large` for capture (`CaptureError::io_code`), and
       `too_large` for import.
   - At or above `LARGE_FILE_WARNING_BYTES` (50 MiB; make it `pub(crate)`), warn.
     - `ClipPlan` carries warnings outside `ClipOutput`.
     - Capture emits
       `CaptureWarning::Structured { code: "attachment_large", message: "vid/x.mp4 is 62.3 MiB; vault sync warns at 50 MiB and refuses 95 MiB" }`.
     - Import JSON gains an additive `warnings: [string]`, and import human output
       prints the warnings to stderr in yellow.
   - Give the planner an injectable limits seam so tests need no 50 MiB hashes. Use
     sparse `set_len` files for the refusal path, as `highlights_ref/intake.rs` already
     does.
3. **Sync-ability guard.**
   - When the vault has a `.git` entry, pipe every planned destination (vault-relative,
     reused ones included) through one `git -C <vault> check-ignore --stdin` call per
     `plan_attachments` call.
   - Refuse each ignored destination. Code `attachment_not_synced` for capture and
     `not_synced` for import. Message:
     `img/IMG_0042.heic would never leave this host: the vault .gitignore ignores it (\`\*\`
     at .gitignore:3) — convert it to an allowed type first`.
   - If you use `-v` to name the rule, skip lines whose pattern starts with `!`. They
     are re-includes, not ignores: in the live vault `vid/test.mp4` prints
     `.gitignore:58:!*.mp4`.
   - Run the guard in dry-run too, so Mac Capture's preview shows the refusal before
     submit.
   - Fail open (skip the guard) when the vault is not a work tree, when `git` is
     missing, or when it exits 128. Never block a capture on a git hiccup.
4. **Docs.**
   - `docs/attachments.md`: add the limits and both refusals, plus the import codes
     `too_large`/`not_synced` and the `warnings` field.
   - `docs/capture.md`: add the limits under "Clipboard shapes and limits".
   - `docs/reference/json-outputs.md`: add the new codes and `warnings`.
   - `docs/vault-git-sync.md`: cross-link the shared 95/50 MiB policy.
5. **Tests:**
   - Planned files hold no attachment bytes, and dry-run creates nothing.
   - A source mutated between plan and save aborts and leaves no files.
   - Fallback install, reuse, and suffix behavior is unchanged.
   - At the limit the capture is refused, with no note write; at the warning size it
     warns and still writes.
   - Gitignore guard: in a temp git vault with `*`, `!*.png`, and `!*/`, `.heic` is
     refused and `.png` passes. Re-include lines are not treated as ignores. A non-git
     vault and a missing `git` both skip the guard. Isolate `HOME` and
     `GIT_CONFIG_NOSYSTEM` so global excludes can't leak in.
   - Origin streaming returns the same states as before.
6. **Memory decision record.**
   > [!decision] memory_video_decision
   - Write the strand through the edit-and-republish flow in `/sase_memory_write`. File:
     `sase/memory/decisions/clip-videos-embed-from-vid.md`. Model its frontmatter on the
     other strands: keyword, aliases, summary, and `metadata.status: accepted` with the
     `decided` date.
   - Applies to bob-cli, bob-plugins, and Bob Mac Capture.
   - Claim:
     - `%c` and `bob attachments import` classify mp4, mov, webm, and mkv as `video`,
       copy them to `vid/`, and write `![[vid/NAME|400]]`.
     - Every attachment copy refuses files of 95 MiB or more and destinations the vault
       `.gitignore` ignores (warning at 50 MiB), so every embed syncs.
     - `Z` plays videos in the shared viewer.
   - Why: list the rejected alternatives from the research table (`IMAGE_EXTENSIONS`
     reuse, keeping `file/`, `%v`, GIF, transcoding, LFS, native controls) and the spike
     facts. Cite the research ref, this plan, and the two bob-cli commits.
   - Cost: two refusals that did not exist before, and the m4v/avi exclusions.
   - Reopens when: Obsidian renders m4v as a player, the vault allowlist changes, `vid/`
     passes about 1 GiB (consider LFS), or HEVC sources become common (consider
     `import --transcode`).
   - Run `sase memory init` afterwards.
7. **Commit** with `!`, for example
   `feat(attachments)!: refuse attachments vault sync can't carry`. Inputs that copied
   yesterday are refused today.

## Phase `plugin-z-dispatch` — bob-plugins: `Z` dispatches videos, imports video paths, sizes inline clips

1. **Resolver** (`679-plugin-image-zoom.js`):
   - A candidate is any vault reference whose extension is in `IMAGE_ZOOM_EXTENSIONS` or
     `VIDEO_ZOOM_EXTENSIONS`. Keep the containing → next → previous pick rule.
   - The resolved target's `kind` is `"image"` or `"video"`, and `"notice"` is
     unchanged.
   - External http(s) URLs stay image-only; web video is out of scope.
   - Legacy `[[file/old.mp4]]` links in all four forms (wiki or Markdown, embed or
     plain) open as videos with no note migration.
   - Add `ico` to `IMAGE_ZOOM_EXTENSIONS`, so the plugin set is bob's image set minus
     heic, tif, and tiff, matching the catalog.
2. **Dispatch.**
   - `zoomImageOnLine` opens `VideoZoomModal` for videos, with the same vault context
     and origin lookup. Never block on the origin promise.
   - Notices:
     - `No image or video on this line`
     - `Not an image or video: X`
     - `Video not found: X` (images keep `Image not found`)
     - Reading view: `Viewing media needs the editor; in Reading view, use the embed`
   - Command id stays `zoom-image-on-line`, because the vault vimrc maps `Z` to it. The
     display name becomes `View image or video on cursor line`.
3. **Path-only bullets** (`679a-image-zoom-path.js`):
   - `findImagePathBullet` accepts video extensions; rename it `findMediaPathBullet` and
     update its tests. A non-media extension throws `Not an image or video: X`.
   - A `kind: "video"` import result opens `VideoZoomModal` with the pill
     `Imported to vid/x.mp4 · r to undo` (or `Linked existing …`).
   - Old bob (`kind: "file"` for a video extension): show the Notice
     `Update bob to import videos · saved file/x.mp4; note unchanged`. Never claim a
     video bob did not classify.
   - Change "Importing images needs…" to "Importing media needs…".
4. **Inline embed CSS** (verified DOM: wrapper `div.internal-embed.video-embed[width]`
   around `<video>`):
   ```css
   .internal-embed.video-embed > video {
     max-width: 100%;
     height: auto;
     background: #000;
     border-radius: var(--radius-s);
     border: 1px solid var(--background-modifier-border);
   }
   .internal-embed.video-embed[width="400"] > video {
     width: 400px;
   }
   @supports (width: attr(width px)) {
     .internal-embed.video-embed[width] > video {
       width: attr(width px);
     }
   }
   ```
   With this CSS, author-edited widths work on the verified Chromium, and 400 px works
   everywhere.
5. **Docs** (`docs/image-zoom.md`, keeping the URL):
   - Retitle the page "View an image or video on the cursor line".
   - Add a Videos section: recognized forms, the start policy, the key table, the
     failure card, `o`, and the inline 400 px column.
   - Generalize "Import an image path" to media paths.
   - Update the command name in `docs/bob-navigation-hotkeys.md` and its generated
     inventory.
6. **Tests and e2e.**
   - Update the `image-zoom-target`, `command`, and `path` suites for video candidates,
     notices, legacy `file/` videos, and old-bob degradation.
   - Add the real-Obsidian harness `scripts/obsidian-video-zoom-e2e.cjs` (plus its
     `-lib.cjs`). Model it on `obsidian-redo-history-e2e.cjs`: an isolated vault and
     profile, an owned process, and CDP keys. Add the offline wrapper
     `scripts/test-obsidian-video-zoom-e2e.cjs --self-test` to `npm test`.
   - Synthesize fixtures with ffmpeg at runtime: a silent H.264 mp4, an H.264+AAC mp4, a
     VP9 webm, and corrupt bytes. Without ffmpeg or a display, report `unrun` with
     exit 2.
   - Scenarios:
     - the inline `video.clientWidth === 400`;
     - `Z` on `![[vid/silent.mp4|400]]` opens `.bob-video-zoom` and `currentTime`
       advances;
     - the audible clip opens paused;
     - `Space` toggles, `→` seeks +1 s, and `Z` on a legacy `[[file/x.mp4]]` link plays;
     - the corrupt clip shows the failure card and the modal stays open;
     - `Esc` closes, the `video` `src` is cleared, and the cursor is restored;
     - with `BOB_BIN` pointing at a bob built from bob-cli master (as the album e2e
       does), `Z` on a `~/…/clip.mp4` path bullet imports it to `vid/` and `r`
       round-trips.
   - Run it with `xvfb-run -a`.
7. Run `npm run build`, `npm test`, `npm run docs:check`, the real e2e, and
   `bob plugins sync`.

## Phase `mac-video-card` — bob-mac-capture: videos in the clip card

Stay inside `decisions:mac-capture-is-a-thin-client`: use bob's `kind`, never the file
extension, and never read the pasteboard.

1. **`CaptureCore/CaptureClipPresentation.swift`:**
   - The header counts videos: `pluralCount(videos, unit: "video")`. Keep the existing
     files → images order and append videos ("1 image · 1 video").
   - `AttachmentRow` gains `isVideo` and the decoded size. Decode `size_bytes` with
     `decodeIfPresent`, so an older bob still decodes.
   - The summary reads `vid/x.mp4 · video · 242.8 KiB`, using bob's one-decimal binary
     units.
   - The accessibility label reads "Video, 6 seconds, vid/x.mp4" once the duration is
     known.
2. **`BobMacCapture/ClipCardView.swift`:**
   - For video rows, generate the poster with `AVAssetImageGenerator`: frame at about
     0.1 s, `appliesPreferredTrackTransform`, `maximumSize` twice the thumbnail point
     size, and generous time tolerances.
   - Read the duration from `AVURLAsset.load(.duration)` and draw it as a small `0:06`
     badge, with a translucent ▶ glyph over the poster.
   - Load inside `.task(id: row.source)` so it is cancellable, and cache the results in
     memory, keyed by source path and modification date.
   - A missing or undecodable source keeps the plain summary row, as images do.
3. Confirm the app renders unknown structured capture warnings (for example
   `{"code":"attachment_large","message":"…"}`) by their bob-authored `message`. If it
   does not, render them that way. Add a decode fixture test.
4. **Tests:**
   - Presentation tests: counts for video, image + video, and file + video; size text
     present and absent; old-bob JSON without `size_bytes`.
   - A poster-loader test behind a small protocol, if the target allows it.
5. **Docs and verification.**
   - Update the clip-card section of the app's `docs/`, then run `just docs-check`.
   - Follow AGENTS.md: `just portable-check` on athena, a scratch typecheck on the Mac,
     and `Scripts/macos-ci-check` before landing.
   - After pushing, watch the exact-SHA macOS run to green with `/sase_monitor`. If the
     Mac is offline, say that the macOS targets are unverified.

## Phase `plugin-video-transport` — bob-plugins: scrubber, frame step, speed, mute, fullscreen

Add these to `VideoZoomModal` in a new fragment (for example
`207d-video-zoom-transport.js`), keeping every fragment at or below 1000 lines.

1. **Scrubber.**
   - A 3 px hairline sits just above the footer with 16 px side insets. Played is white
     at 0.9, buffered 0.35, and the track 0.15.
   - Hover grows it to 6 px and shows a time bubble.
   - Click seeks. Dragging pauses and resumes afterward if the clip was playing.
   - `pointer-events` is enabled only on the scrubber, and keyboard focus stays on the
     viewport.
   - The failure card hides it.
2. **Keys.**

   | Input     | Action                                                                                      |
   | --------- | ------------------------------------------------------------------------------------------- |
   | `,` / `.` | Step one frame back / forward (pauses)                                                      |
   | `[` / `]` | Speed down / up through 0.25, 0.5, 0.75, 1, 1.25, 1.5, 2 (readout ` · 0.5×` when not 1)     |
   | `g` / `G` | Jump to the first / last frame (pauses on `G`)                                              |
   | `m`       | Mute / unmute; the chip shows only when the clip has audio                                  |
   | `f`       | Toggle fullscreen on the modal element; `Esc` leaves fullscreen first and never also closes |

   For frame stepping, learn the frame duration from `requestVideoFrameCallback`
   `mediaTime` deltas (fallback 1/30 s). Land mid-frame to avoid boundary rounding.

3. **Play/pause flash.** A centered 56 px glyph fades over about 350 ms on every toggle.
   A paused start shows it resting at low opacity until the first play. Suppress both
   under reduced motion.
4. **Footer.**
   - Add `, . frame` and `[ ] speed`, and `m mute` only when the clip has audio.
   - Keep the footer at two lines at most at 1280 px. Drop the zoom chips before the
     transport chips; the docs list everything.
5. **Docs.** Complete the key table in `docs/image-zoom.md`.
6. **Tests.**
   - Unit-test the key map, speed stepping, frame math, fullscreen Esc ordering, and
     drag-seek state.
   - Extend the real-Obsidian e2e: scrubber drag seeks; `.` advances exactly one frame;
     `]` gives 1.25×; `f` then `Esc` leaves fullscreen with the modal still open; `m`
     toggles mute on the audible clip; no flash under emulated `prefers-reduced-motion`.
   - Run `npm run build`, `npm test`, `npm run docs:check`, the real e2e, and
     `bob plugins sync`.

## Out of scope

These are the research's A9 deferrals:

- a `%v` directive, transcoding, Git LFS, and bob-generated posters;
- audio files, galleries, and YouTube/URL video;
- timestamp links, mobile, and Finder `Cmd+C` file-URL clipboards;
- frame grab (`s` saving a PNG sibling).

Each has a reopen trigger in the research and in the decision record.

## Done means

- The request's exact example works end to end on athena: `%c` gives an inline 400 px
  clip, and `Z` plays it.
- Every phase's gates are green.
- The final report names what only Bryan can confirm on the Mac: HEVC `.mov` playback,
  macOS fullscreen, and real off-host sync of a captured `vid/` file.
