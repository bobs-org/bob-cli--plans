---
tier: tale
title: Add the Mac Menu Bar Pomodoro Indicator (mac pom) glossary term
goal: bob-cli's glossary web defines the Hammerspoon menu-bar Pomodoro indicator under
  the keyword "Mac Menu Bar Pomodoro Indicator" with alias "mac pom", the roster advertises
  it, and reading it links only the Pomodoro term family.
size: small
proposed_by: bbugyi200.athena.0wk.f0
status: done
---

# Plan: Add the "Mac Menu Bar Pomodoro Indicator" (mac pom) glossary term

## Objective

Bryan's macOS menu bar has a Hammerspoon status item that shows the current Pomodoro. It
just gained a green `NO POMODORO` reminder flash (chezmoi commit `69e95220`,
`feat(hammerspoon): flash NO POMODORO green one minute in ten`). There is no SASE
vocabulary for it yet. As a result, prompts call it "the pomodoro indicator in the menu
bar on my macbook", and agents have to rediscover what it is, where it lives, and that
it depends on `bob pomodoro` output.

Add one term to bob-cli's `glossary` memory web:

- **Keyword:** `Mac Menu Bar Pomodoro Indicator`
- **Alias:** `mac pom`

Once added, the always-loaded roster advertises the term, prompts that say "mac pom" or
the full name get highlighted, and `sase memory read glossary:<term>` gives an agent the
indicator's identity, states, data contract, and source location in one read.

The user asked for this memory change in their prompt, so it is authorized. Follow the
`/sase_memory_write` skill's "Edit And Republish" path.

## Design

### Name and alias

- The keyword is the user's full name for the thing, in the glossary's Title Case style.
  It renders on the roster as `Mac Menu Bar Pomodoro Indicator (mac pom)`, sorted
  between Keep Streak and Pomodoro.
- `mac pom` is the only alias, exactly as the user gave it. Every alias is inlined into
  every agent's instructions and broadens prompt highlighting, so more synonyms would
  cost tokens every turn. Broader ones ("pomodoro indicator", "menu bar pomodoro") would
  also collide with the `bob pomodoro tmux` status line, which is a different indicator.
- A planning-time dry run of SASE's real glossary matcher, with this draft added to the
  current strands, confirmed:
  - "the mac pom flashes" → this term (via `mac pom`).
  - "two mac poms" → this term (derived plural).
  - "fix the mac menu bar pomodoro indicator" → this term (longest match wins over
    Pomodoro).
  - "mac pomodoro thing" → Pomodoro only, as intended.
- The slug and file name follow the existing kebab-case-of-keyword convention:
  `sase/memory/glossary/mac-menu-bar-pomodoro-indicator.md`.

### What the definition covers

The definition follows the shape of the best existing strands (Task Dependency Link,
Project Task). It says what the item is and where it appears, what it reads, how each
state looks, what it is not, and where its source and spec live:

1. **Identity:** the Hammerspoon status item in the macOS menu bar on Bryan's MacBook
   that shows the current Pomodoro.
2. **Data contract:**
   - It is a read-only consumer of `bob pomodoro --show-stale`. It polls every 15 s, on
     wake and unlock, from its Refresh menu item, and once when the countdown crosses
     zero.
   - It ticks locally between polls and never writes the vault.
   - It names `--show-stale` because without that flag a long-overdue session would look
     idle. This is the fact a bob-cli agent changing `bob pomodoro` needs most.
3. **States, using recognizable title shapes:**
   - Running shows `THEME (50m) · 🍅 12:34`. THEME is the Pomodoro's ` — NAME`, or
     `UNTITLED` when unnamed.
   - Overdue adds the `→ HH:MM` stop time and a red `+MM:SS` count.
   - From ten minutes overdue it shows a flashing `OVERDUE` badge.
   - An idle green `NO POMODORO` flashes as a green pill for its first minute on screen,
     then one minute in every ten. The cycle restarts when the label reappears or the
     Mac wakes or unlocks.
   - Failures hide the item.
4. **Disambiguation:** it is not the `bob pomodoro tmux` status line and not Bob Mac
   Capture's menu-bar item. Both are real, nearby things an agent could confuse it with.
5. **Where it lives:**
   - Source is the linked `chezmoi` repo under `home/dot_hammerspoon/`: `init.lua` is
     the runtime and `pomodoro_countdown.lua` is the pure presentation policy.
   - That repo's README "Pomodoro menu bar" section is the spec.
   - Hence the instruction to coordinate `bob pomodoro` output changes with it.

It deliberately leaves out hex colors, the no-break-space padding, font fallbacks, theme
truncation, tooltip and dropdown contents, and test details. Those are volatile spec
details already owned by the chezmoi README. Copying them into memory would only create
drift.

### Mention hygiene (implicit links)

The glossary descriptor sets `closure: mentions` (that is, `link_reference: implicit`).
Any case-insensitive prose mention of another term's keyword, alias, or derived plural
becomes a link and is pulled into `sase memory read` closures. Code spans are skipped.
So the body:

- **Mentions Pomodoro in prose on purpose.** Reading this term then also prints the
  Pomodoro definition, which defines "current Pomodoro" and the ` — NAME` suffix.
  Through Pomodoro's own existing mentions, the closure also brings in Task Link and
  Task Dependency Link. That is pre-existing behavior and acceptable.
- **Avoids accidental mentions.** The planning dry run caught an earlier draft whose
  phrase "`--show-stale` keeps it there" linked Keep Streak through its `keeps` alias.
  The final text says "holds it there" instead. Do not reintroduce "keep(s)",
  "freshness", "task link", "work log", "schedule log", "ref note", "prj", or "dep link"
  when editing the wording.
- **Uses no explicit `[[...]]` links.** The only glossary term it names (Pomodoro) is
  already linked by mention. "Bob Mac Capture" names a product, not a memory note, so it
  gets no link.

### Not changed

- **The Pomodoro strand gets no back-reference.** With implicit mentions, adding "mac
  pom" there would drag this term into every Pomodoro read, which is noise for the many
  agents that only need ledger grammar. The reverse link is already counted on the
  `sase memory web show glossary` index.
- **No chezmoi, bob-cli source, or docs changes.**

## Implementation

1. Invoke the `/sase_memory_write` skill. The edit is authorized by the user's prompt
   and this approved plan.

2. Create `sase/memory/glossary/mac-menu-bar-pomodoro-indicator.md` with exactly this
   content. Line-wrap prose at 88 columns like the other wrapped strands, and keep the
   code spans intact.

   ```markdown
   ---
   keyword: Mac Menu Bar Pomodoro Indicator
   aliases:
     - "mac pom"
   ---

   The Hammerspoon status item in the macOS menu bar on Bryan's MacBook that shows the
   current Pomodoro at a glance. It is a read-only consumer of
   `bob pomodoro --show-stale`: it polls every 15 seconds, on wake and unlock, from its
   Refresh menu item, and once when the countdown crosses zero; it ticks the countdown
   locally in between and never writes the vault. A running session reads
   `THEME (50m) · 🍅 12:34`, where THEME is the Pomodoro's ` — NAME` (`UNTITLED` when
   unnamed). Once overdue, a `→ HH:MM` stop time joins the title and the `+MM:SS` count
   turns red; from ten minutes overdue the count becomes an `OVERDUE` badge flashing red
   at 1 Hz, and `--show-stale` holds it there rather than letting it read as idle. Empty
   output shows a green `NO POMODORO` with no tomato, which flashes as a green pill for
   its first minute on screen and then for one minute in every ten, restarting that
   cycle whenever the label reappears or the Mac wakes or unlocks. A failed or
   unparsable run hides the item. It is neither the `bob pomodoro tmux` status line nor
   Bob Mac Capture's menu-bar item. It lives in the linked `chezmoi` repo under
   `home/dot_hammerspoon/` (`init.lua` runtime, `pomodoro_countdown.lua` presentation
   policy); that repo's README "Pomodoro menu bar" section is its spec, so coordinate
   `bob pomodoro` output changes with it.
   ```

3. Run `sase memory init` as the skill instructs. Never hand-edit generated files.
   Expected regenerated project files:
   - The roster block in `sase/memory/glossary.md`.
   - The Glossary Terms roster in `AGENTS.md`, `CLAUDE.md`, `GEMINI.md`, `OPENCODE.md`,
     and `QWEN.md`.
   - `sase/memory/README.md`.

   At planning time, `sase memory init --check` already reported one unrelated drift: a
   one-token count rounding change (+2/−2) in the **home** memory README under the
   chezmoi source directory. If init also rewrites that file, it is expected
   tool-generated noise and not part of this change. Do not hand-edit it.

## Verification

Run from the bob-cli checkout:

```bash
sase memory init --check            # no remaining project memory drift
sase memory web show glossary       # new row: keyword, slug, alias "mac pom"
sase memory show "glossary:mac pom" # alias resolves
sase memory show glossary:mac-menu-bar-pomodoro-indicator
git diff --check
```

Confirm all of the following:

- The roster line in the regenerated `CLAUDE.md`/`AGENTS.md` reads
  `Mac Menu Bar Pomodoro Indicator (mac pom)` and sits between `Keep Streak (keeps)` and
  `Pomodoro`.
- `sase memory show "glossary:mac pom"` prints the new definition followed by Pomodoro.
  Task Link and Task Dependency Link may follow through Pomodoro's own mentions. It must
  **not** pull in Keep Streak, Task Freshness, Work Log, Schedule Log, or the
  project/reference terms. If it does, find the accidental mention in the body and
  reword it.
- The rendered body keeps every code span intact, including `` ` — NAME` `` with its
  leading space and em dash, and `🍅`.
- The diff touches only the new strand plus the generated files listed above, plus the
  known home-README noise if init writes it.

## Out of scope

- A back-reference from the Pomodoro strand, or more aliases.
- Documenting the menu-bar consumer in bob-cli's README "Pomodoro" section.
- Any change to the indicator itself or to `bob pomodoro` output.
