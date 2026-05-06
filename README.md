# Marinara-RPG-RP-Mode-Extension — Roleplay Mode (v0.2.0)

A Marinara Engine client extension that overlays a custom RPG ruleset
on **Roleplay Mode** chats — without forking the engine. Sister project
to [Marinara-RPG-Extension](https://github.com/Kenhito/Marinara-RPG-Extension)
which targets Game Mode.

## Quick start (5 minutes)

The fastest path: grab the self-contained release at [`releases/v0.2.0/`](releases/v0.2.0/) and follow [`releases/v0.2.0/INSTALL-GUIDE.md`](releases/v0.2.0/INSTALL-GUIDE.md). It includes:

- **The framework JS** (`RPG-Extension-RP-Mode.js`) to paste into Marinara's Extensions panel
- **Two complete reference rulesets** (D&D 5e and Exalted 3e) with bundle + agents pre-built
- **Seven AI-feedable build documents** so you can have ChatGPT, Claude.ai, or Gemini author a ruleset for any other system (GURPS, Cyberpunk RED, Vampire, Mörk Borg — anything)
- **Step-by-step install + build guides** in plain language for non-technical users

If you just want to play D&D or Exalted in Roleplay Mode, three pastes and you're done.

If you want a system the framework doesn't ship, see [`releases/v0.2.0/BUILD-YOUR-OWN-RULESET.md`](releases/v0.2.0/BUILD-YOUR-OWN-RULESET.md) — three options for AI-assisted or manual authoring.

## What's new in v0.2.0

- **Typed damage** on health-style tracks (Bashing/Lethal/Aggravated for Storyteller systems, slashing/piercing/bludgeoning for D&D, etc.). Per-type colors, severity-stacking display, per-type "Take" + "heal worst" buttons. State-mutator routes `field="bashing" delta="+3"` automatically.
- **Sorcery / multi-turn casting category** in the spellbook. Spells filed under `"sorcery"` get auto-tagged in their lorebook entries; the state-mutator override walks declare → accumulate sorcerous motes → unleash with Willpower refund.
- **Agents decoupled from bundle.** New `tools/build-agents.mjs` produces `agents.json` separately. Users import via Marinara's Import Agents dialog. Prompt updates ship without forcing users to reinstall the ruleset.
- **CSP-safe formula evaluator.** The `{StatName}*N+M` parser uses recursive descent instead of `new Function`. Marinara pages with strict CSP now compute bar maxes correctly (Personal Motes on an Essence-7 Exalt now shows /31 instead of /10).
- **State-mutator field-name normalization** + max-clamp on numeric deltas + persisted dedupe across reloads.
- **Lorebook install path rewritten** to fix the silent-failure bulk endpoint.
- **Self-contained release folder** at `releases/v0.2.0/`.

See [`CHANGELOG.md`](CHANGELOG.md) for the full list.

## Why a separate project for Roleplay Mode?

Marinara Engine has four chat modes: `conversation`, `roleplay`,
`visual_novel`, and `game`. The two we care about for tabletop-style
RPG play are `game` (with hardcoded combat encounter modal, d20 skill
checks, reputation tags) and `roleplay` (without those engine-imposed
mechanics).

The Game-Mode extension works around the engine's hardcoded systems —
the 50-character reputation `action` cap, the d20-tag prompt format,
the encounter modal that always uses d20 + DC mechanics. The
Roleplay-Mode extension doesn't need any of those workarounds. RP mode
gives the overlay full control over dice math (via the in-extension
dice widget), unrestricted lorebook prose (no per-action character
caps), and a clean multi-agent surface for future state-tracker and
lore-lookup agents that don't fit the GM-mode single-narrator pattern.

## Capability matrix

| Feature                              | Game-Mode ext | RP-Mode ext (this) |
|--------------------------------------|:-------------:|:-------------------:|
| Custom dice math (any system)        | ✓ (in widget) | ✓ (in widget)       |
| Custom skill / attribute taxonomy    | ✓             | ✓                   |
| Tier-based skill proficiency         | ✓ (v0.4)      | ✓ (ported)          |
| Named skill specialties              | ✓ (v0.4)      | ✓ (ported)          |
| Equipment / inventory + bonuses      | ✓             | ✓                   |
| Lorebook auto-install (bundle)       | ✓             | ✓                   |
| Single-paste install via `bundle.json` | ✓           | ✓                   |
| Engine combat encounter modal        | hardcoded     | not present         |
| Engine d20-tag prompt format         | hardcoded     | not present         |
| Reputation `action` 50-char cap      | hardcoded     | not present         |
| Engine `world-state` cooperation     | n/a           | ✓ (default agent)   |
| Engine `prose-guardian` cooperation  | n/a           | ✓ (default agent)   |
| Engine `continuity` cooperation      | n/a           | ✓ (default agent)   |
| Multi-agent overlay (tracker, lore)  | hard          | ✓ (opt-in, off by default) |

## What ships in v0.0.1

- **Framework:** `extension/RPG-Extension-RP-Mode.js` (single-paste client
  extension; CSS embedded inline) plus `extension/RPG-Extension-RP-Mode.css`
  (companion stylesheet for direct edits).
- **Schema:** `schema/bundle.schema.json` — install bundle envelope
  with discriminator `"schema": "mrrp-bundle"`.
- **Tools:** `tools/build-bundle.mjs`, `tools/validate-bundle.mjs`,
  `tools/validate-ruleset.mjs`, `tools/embed-css.mjs`.
- **Reference rulesets:**
  - `rulesets/dnd5e/` — Dungeons & Dragons 5th Edition (SRD 5.1).
  - `rulesets/exalted3e/` — Exalted 3rd Edition.

### Optional sub-agents (off by default)

Each ruleset bundle installs five **optional** pre-generation sub-agents
alongside the main `gmAgent`. They install **disabled by default**;
flip individual ones on in **Marinara → Settings → Agents** to opt
into specific behaviors. Each sub-agent adds one model call per turn
while enabled, so enable only what your campaign needs.

| Sub-agent | What it does | Enable when… |
|---|---|---|
| `state-mutator` | Tells the narration model to emit hidden `[mrr-state: …]` tags when narrative changes the sheet (HP, conditions, inventory). Extension parses tags, applies the change, shows a toast. | You want narration to drive the sheet automatically. |
| `state-reminder` | Surfaces a short bulleted list of current PC state (HP, conditions, gear, resources) at the top of every turn. | Long sessions or complex sheets, the model is forgetting your stats. |
| `combat-adjudicator` | Wakes only in combat; restates initiative, action economy, attack/damage formulas, conditions in the active ruleset's terms. Outputs `"No combat active."` and stops in social/ambient scenes. | You run heavy-mechanics combat encounters. |
| `lore-query` | Wakes only when the latest user message is a rules question. Answers from the installed lorebook + system RAW. Outputs `"No rules query."` otherwise. | You frequently ask the model rules questions mid-RP. |
| `npc-bookkeeper` | Tracks active and recently-engaged NPC HP, conditions, tactical state, and telegraphed intentions across turns. Outputs `"No NPCs to track."` when no NPCs are in scene. | Combat/social scenes with multiple named NPCs you want continuity on. |

The same descriptions are also in each system's installed lorebook
(under "Optional Sub-Agents") so they surface in-engine when you ask
the chat about them.

To enable: open the gear icon in Marinara, go to **Settings → Agents**,
find the agent (named `MRRP: <System> — <Role>`), toggle on. The agent
persists across chats; toggle off any time.

---

Both rulesets ship with `bundle.json` (the install file) plus their
source files (`ruleset.json`, `gm-agent.md`, `lorebook.json`). Vibecoder
authors can skip the source files entirely and ship a `bundle.json`
alone; see `AUTHORING-PROMPT.md`.

## Install (target audience: testers, not vibecoders)

The install path is identical to the Game-Mode extension's path. Two
imports, end to end:

1. **Import the framework JS file.** Open Marinara at
   `http://localhost:7860`. Settings → Extensions → Add Extension →
   import `extension/RPG-Extension-RP-Mode.js` (Marinara's Extensions UI is
   file-import, not text-paste). CSS is embedded inline in the JS, so
   you only import the one file. Save and enable.

2. **Import a ruleset bundle.** A new "Ruleset" button appears in the
   chat header. Click it, then either import the `bundle.json` file
   directly (file-import button in the dialog) or paste its contents
   into the textarea. Click Install. Use either
   `rulesets/dnd5e/bundle.json` or `rulesets/exalted3e/bundle.json`.

The installer creates / updates one custom agent and one lorebook in
your Marinara database. Re-installing the same bundle is idempotent —
matched by `mrrpManaged: true` settings flag and lorebook tags.

### Cross-OS file paths

| Platform | Marinara data path |
|---|---|
| Linux   | `~/Marinara-Engine/packages/server/data/` |
| macOS   | `~/Library/Application Support/Marinara-Engine/data/` |
| Windows | `%APPDATA%\Marinara-Engine\data` |

The extension itself is platform-agnostic — these paths only matter if
you want to back up the database before testing.

## Coexistence with the Game-Mode extension

You can install BOTH overlays on the same Marinara instance. The
Roleplay-Mode extension uses the `mrrp-` namespace (localStorage keys,
agent type, lorebook tags, CSS classes, settings flags) where the
Game-Mode extension uses `mrr-`. The two installers won't see each
other's bundles, agents, or lorebook entries.

Use the Game-Mode extension in `chatMode: "game"` chats. Use the
Roleplay-Mode extension in `chatMode: "roleplay"` chats. The same
chat-AI-authored ruleset CAN target both, but the bundles and agent
prompts diverge enough that you'll likely author two variants per
system.

## Engine constraints — what this overlay still cannot do

- **It cannot change the engine's roleplay-mode default agents.**
  `world-state`, `prose-guardian`, `continuity`, and `expression`
  remain part of the default pipeline. The overlay agent runs alongside
  them as a `pre_generation` context-injection. Authors should write
  agent prompts that cooperate with — not replace — those defaults.
- **It cannot bind a custom dice widget into the engine's chat UI
  beyond what the floating widget provides.** The widget is positioned
  via CSS `position: fixed` and persists across chats; it is not
  embedded in the engine's message composer.
- **It cannot rewrite messages after the engine has streamed them.**
  All overlay influence happens via context injection before generation.

## Repo layout

```
extension/
  RPG-Extension-RP-Mode.js      # Single-paste client extension. CSS embedded.
  RPG-Extension-RP-Mode.css     # Source CSS — edit, then `npm run embed-css`.
schema/
  bundle.schema.json     # Install bundle envelope schema (discriminator: mrrp-bundle).
  ruleset.schema.json    # Ruleset-data schema (shared semantics with GM-mode ext).
tools/
  build-bundle.mjs       # ruleset.json + gm-agent.md + lorebook.json -> bundle.json
  validate-bundle.mjs    # Schema-validates bundle.json files.
  validate-ruleset.mjs   # Schema-validates ruleset.json files.
  build-character-card.mjs # V2 character card builder.
  embed-css.mjs          # Embed RPG-Extension-RP-Mode.css into RPG-Extension-RP-Mode.js.
rulesets/                # Reference systems — same four as the GM-mode sibling.
  dnd5e/                 # D&D 5e (SRD 5.1).
  exalted3e/             # Exalted 3rd Edition (paraphrased mechanics).
  fate-core/             # Fate Core (4dF + ladder).
  pathfinder2e/          # Pathfinder 2nd Edition (Remaster) — bundle-only.
agents/                  # Sub-agent prompt sources (same five as GM-mode).
  state-mutator.md       # Emits [mrrp-state: ...] tags from narration.
  state-reminder.md      # Surfaces current PC state each turn.
  combat-adjudicator.md  # Combat-only; restates initiative + math.
  lore-query.md          # Rules-question-only; cites lorebook + RAW.
  npc-bookkeeper.md      # Tracks NPC HP / conditions / intent across turns.
docs/                    # Authoring + install + engine-constraints reference.
  ADDING-RULESETS.md
  AUTHORING.md
  ENGINE-CONSTRAINTS.md  # RP-mode-specific (the GM-mode sibling has its own).
  INSTALL.md
  LOREBOOK-KEEPER-SCOPING.md
AGENTS.md                # AI-agent reference for THIS repo.
AUTHORING-PROMPT.md      # Paste-ready prompt for vibecoder authors.
CHANGELOG.md             # Keep-a-Changelog.
```

## Authoring your own ruleset

### The fast path — vibe-code with a chat AI

Open **`AUTHORING-PROMPT.md`**, copy it whole into a chat with a frontier model (Claude, GPT-5, Gemini Pro), and paste it as your system prompt. The prompt directs the AI to read this repo's authoritative source files, then produce a single `bundle.json` for your system. The AI knows what files it needs to consume:

| File | What the AI reads it for |
|---|---|
| `schema/bundle.schema.json` | The exact JSON shape required (discriminator `mrrp-bundle`, integer `position`, required fields). |
| `schema/ruleset.schema.json` | Attribute, skill, derived-stat, and resolution-mode constraints. |
| `docs/ADDING-RULESETS.md` | Full walkthrough including the RP-mode framing rules. |
| `docs/AUTHORING.md` | Bundle anatomy, eight-step authoring process, common pitfalls. |
| `docs/ENGINE-CONSTRAINTS.md` | What roleplay-mode bundles can and cannot change — different from the GM-mode sibling. |
| `agents/*.md` | The five optional sub-agent prompt sources. |
| One reference bundle (`rulesets/dnd5e/bundle.json` for d20, `exalted3e` for dice pool, `fate-core` for fate ladder, `pathfinder2e` for d20 with three-action economy) | Concrete example of every field populated correctly. |

Hard requirements your AI's output MUST hit:

- **RP-mode framing in `gmAgent.promptTemplate`.** Open with cooperative wording: "You provide rules guidance for ⟨system⟩ in Marinara Engine's roleplay mode, working alongside the engine's default world-state, prose-guardian, continuity, and expression agents." NOT "You are the GM." NOT "running inside Marinara Engine's Game Mode."
- **No `[d20: …]` engine-tag instructions** unless the system uses d20 mechanics — and even then, prefer the in-extension dice widget. Instruct the model to call out the check ("that's a Stealth check at DC 17"), then let the player resolve via the widget.
- **No 50-character reputation tag workaround paragraph.** That's a Game-Mode constraint; it does not apply here.
- **`gmAgent.promptTemplate` ≥ 50 characters** (schema minimum). Realistically aim for 800+ words covering: system identity, dice mechanic, cooperation framing, what to emit each turn.
- **Integer `position` 0 | 1 | 2** on every lorebook entry (not strings).
- **`schema: "mrrp-bundle"`** discriminator literal — NOT `"mrr-bundle"` (that's the GM-mode sibling).
- **Sub-agents install disabled by default.** If your bundle ships sub-agents in `additionalAgents[]` you genuinely want enabled on install, set `"enabled": true` on that item explicitly. Otherwise the user toggles them on per-agent in Settings → Agents.
- **Character cards (if shipped):** V2 spec (`spec: "chara_card_v2"`, `spec_version: "2.0"`).

Run `node tools/validate-bundle.mjs rulesets/your-system/bundle.json` after the AI hands back the result. The validator emits `path/expected/got/hint` records — paste any errors back to the AI for a corrected bundle.

### The developer path — assemble from sources

1. Copy the closest existing bundle directory (`rulesets/dnd5e/`, `exalted3e/`, `fate-core/`, or `pathfinder2e/`) to `rulesets/your-system/`.
2. Edit `ruleset.json` — your dice, attributes, skills, difficulty ladder, dice-tag format.
3. Run `node tools/validate-ruleset.mjs rulesets/your-system/ruleset.json` to confirm.
4. Edit `gm-agent.md` to teach the GM your mechanics and dice-tag format — apply the RP-mode framing rules above.
5. Build `lorebook.json` for your system's rules reference.
6. (Optional) Edit `agents/<role>.md` for sub-agents — see existing files in this repo's `agents/`.
7. Write `INSTALL.md` for end users.
8. Run `node tools/build-bundle.mjs rulesets/your-system/` to assemble the inputs into `bundle.json`.
9. Run `node tools/validate-bundle.mjs rulesets/your-system/bundle.json` to confirm shape.
10. Test in a real chat with a real model before declaring done.

### Resolution modes available today

`single-roll` (d20-style), `dice-pool` (Exalted, oWoD/nWoD), `d100-percentile` (BRP/CoC), `2d6-stat` (PbtA), `fate-ladder` (Fate Core/FAE). Adding a new mode is documented in `docs/ADDING-RULESETS.md`; PRs welcome.

## License

MIT. See `LICENSE`.

## Contributing

Issues and PRs welcome once the public repo lands. For now this is a
single-maintainer scaffold; please reach out before sending changes.
