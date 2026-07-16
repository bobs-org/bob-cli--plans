---
plan: 202607/hide_deterministic_task_fields.md
---
 Can you help me hide these ugly Obsidian `dependsOn` and `id` dataview properties somehow (using icons/emojis maybe)? The values for these properties are always determined deterministicly from the transcluded block links or block ID, respectively. See #sshot for what these look like now. I want you to lead the design on this one. Make sure you design this feature so it is intuitive, reliable, and (last but not least) beautiful! Think this through thoroughly and create a plan using your `/sase_plan` skill. Choose and author the appropriate
tier, validate and revalidate until it passes, then submit it with `sase plan propose` (as the skill instructs)
before making any file changes.

%xprompts_enabled:false
### Questions and Answers

#### Q1: id field

> How should [id:: ...] render? Its value is just <file>__<blockid>, and the ^blockid caret is already visible at the end of the line in Live Preview. Your vault already hides the [h:: <sha256>] field completely with the exact same CSS trick, so there is a proven precedent for erasing machine-only fields.

- [x] **Hide it completely (recommended)** — display:none, same as the existing [h::] rule. The ^caret IS the icon - an extra glyph would just be new noise. Cursor on the line still shows raw text for editing.
- [ ] **Collapse to a muted glyph** — Show a small quiet marker so you can tell at a glance the task is a dependency target, even in Reading view where Obsidian hides carets.
- [ ] **Leave id alone, only fix dependsOn** — Narrow the change to just the dependsOn pill.

#### Q2: dependsOn field

> How should [dependsOn:: ...] render? Note: an emoji like the ⛔ that Obsidian Tasks itself uses cannot be recoloured and ignores your light/dark theme - it would be a saturated foreign object in a stylesheet where literally every colour is a color-mix(var(--...)). A monochrome glyph inherits --text-muted and recedes.

- [x] **Muted glyph + hover reveals value (recommended)** — Quiet theme-aware marker (e.g. a link/chain glyph) instead of the pill. Hovering pops the value in an absolutely-positioned tooltip, so no text reflow. Keeps a signal that a dependency exists.
- [ ] **Muted glyph, no hover reveal** — Simplest. To see the value, put the cursor on the line (Live Preview shows raw text natively).
- [ ] **⛔ emoji** — Matches the signifier Obsidian Tasks renders inside transclusions, so source and embed agree - at the cost of a loud, non-themeable colour.
- [ ] **Hide it completely** — Maximum quiet. The transcluded bullets below the task already show the dependencies.

#### Q3: CSS location

> Where should this CSS live? The pills are styled by ~/bob/.obsidian/snippets/dataview-properties.css, which is tracked in the VAULT's git repo - not in bob-plugins and not in chezmoi. The [h::] hide precedent sits in that same file. bob-plugins CSS is by convention modal-only and has never touched the editor surface, and `bob plugins sync` only deploys manifest.json/main.js/styles.css - there is no snippet sync path.

- [x] **Vault snippet, extend dataview-properties.css (recommended)** — Sits 5 lines from the [h::] precedent, keeps all inline-field presentation in one file, no plugin coupling, toggleable in Appearance. Requires the implementing agent to edit + commit in the ~/bob vault repo.
- [ ] **bob-plugins plugin styles.css** — Version-controlled and reviewed in bob-plugins, deployed via `bob plugins sync`. But it splits the pill rules from the hide rules across two repos and couples presentation to a plugin being enabled.
- [ ] **Something else / let me explain** — e.g. a new snippet file, or a new snippet sync path in bob-cli.

#### Q4: Drift safety

> Scope question on reliability. You said these values are always deterministic - that is true on the happy path, but I found it is NOT guaranteed: dependsOn edits are additive/subtractive so legacy/manual entries survive; the migration script needs a 5-tier resolution chain (including a positional guess) precisely because dependsOn <-> transclusion is not a bijection; and task-status-cycler's parser is all-or-nothing, so ONE malformed id silently discards every dependency on the line and the task quietly looks unblocked. Hiding the pill hides that drift.

- [ ] **Hide now, add a drift check later (recommended)** — Keep this plan a focused presentation change. I will note the drift gap as explicit follow-up work rather than silently accepting it.
- [ ] **Include a drift check in this plan** — Bigger scope: something must compare dependsOn against the transclusions/canonical form and flag mismatches - a plugin, or a `bob` check wired into nightly. Makes the plan an epic.
- [x] **Not worried, just hide them** — Drift is rare enough and `!` reconciles it; do not spend effort here.

#### Q5: Deploy vault

> How should commit 540479ba reach the live ~/bob vault?

- [x] **Apply it now** — Fast-forward the live vault while preserving its unrelated modified files.
- [ ] **I will sync it** — Leave the commit pushed and let me use my own sync routine.

%xprompts_enabled:true
