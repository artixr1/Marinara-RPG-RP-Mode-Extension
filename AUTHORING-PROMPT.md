# AUTHORING-PROMPT.md — Vibecoder One-Paste Template

> **CURRENT (Phase 5 / 6 / 7):** for the live schema — Resources cluster, sections.order / sections.hidden, autocalc derived stats with valueFormula / tooltipFormula / formulaShort, commitmentModel + commitmentPool, state-banner resource type, the **nine resolution modes** (single-roll, dice-pool, d100-percentile, 2d6-stat, fate-ladder, roll-under, stance-modal-pool, dice-pool-sum with Wild Die, narrative-handled), the item.hardness / item.moteCommitment + item.motePool auto-inheritance, **pipGranularity** (OpenD6 sub-die precision), **effects.onSpend** (Character-Points style spend-driven bonuses), **roundCounters[]** (per-round combat penalties), **penaltyKind** flat/dice on track cells, **abilities.groups[]** (V20 Talents/Skills/Knowledges trinity), and the **meritsFlaws** section type — see **[`docs/AUTHORING-PHASE-6.md`](docs/AUTHORING-PHASE-6.md)**. Paste that file into your chat AI's context alongside the prompt block below when authoring a new bundle.
>
> **v0.2 reference:** the AI-authoring documentation at [`releases/v0.2.0/docs-for-ai/`](releases/v0.2.0/docs-for-ai/) covers the pre-Phase-5 surface (system-agnostic agent architecture, typed damage, sorcery/multi-turn casting, the build pipeline). Read it AFTER `docs/AUTHORING-PHASE-6.md` if you want the deeper agent and pipeline context.
>
> This file remains as a pre-v0.2 single-paste template. For a current-schema-aware bundle, paste the prompt block below PLUS `docs/AUTHORING-PHASE-6.md` into your chat AI; both fit comfortably in a single context window.

> ## ⚠️ Before authoring: does the schema support your system's dice mechanic?
>
> The schema supports **nine resolution modes** — single-roll, dice-pool, d100-percentile, 2d6-stat, fate-ladder, roll-under, stance-modal-pool, dice-pool-sum (with Wild Die), and narrative-handled. They cover the vast majority of shipping tabletop systems.
>
> **If your system's dice mechanic isn't one of the nine, STOP.** Don't try to encode it under the closest mode (that produces a sheet/widget that lies). Ask Kenhito in the **Marinara Extension community thread** (linked from the project README) or open an issue at the project's GitHub repo. Describe: system name + page reference for the resolution rule, shape of one full roll, any special mechanics (exploding dice, opposed rolls, multi-stat pulls, stance toggles), and what specifically breaks if you try to fit it under an existing mode. Schema additions are Kenhito's job, not yours.

---

Copy everything below the `--- BEGIN PROMPT ---` line into a chat-AI
window (Claude, ChatGPT, Gemini, etc.). Replace `<RPG SYSTEM NAME>`
with the system you want a bundle for. The AI will produce a single
`bundle.json` file ready to install via the extension's Ruleset
dialog.

You do not need Node, Git, or JavaScript to use this path. Just paste,
copy the JSON the AI produces, and paste it into Marinara.

--- BEGIN PROMPT ---

You are authoring a single `bundle.json` install file for the
**Marinara-RPG-RP-Mode-Extension**, a client extension that overlays a
custom RPG ruleset on Marinara Engine's Roleplay Mode. Your output will
be pasted directly into the extension's installer dialog without
review or post-processing, so it must be valid JSON the first time.

The system you are authoring for is: **<RPG SYSTEM NAME>**

## What you are producing

A single JSON file — a "bundle envelope" — that contains three
components glued together:

1. A **ruleset** — attributes, skills, derived stats, dice math.
2. An **agent prompt** — system prose that injects rules guidance into
   each turn's pre-generation context.
3. A **lorebook** — keyword-triggered reference entries that the
   engine pulls in when the AI references certain terms.

## Required envelope structure

```json
{
  "schema": "mrrp-bundle",
  "version": 1,
  "minExtensionVersion": "0.0.1",
  "authorId": "<your-author-handle, lowercase, no spaces>",
  "generator": { "name": "vibecoder-prompt", "version": "1.0.0" },
  "ruleset":  { /* see ruleset rules below */ },
  "gmAgent":  { /* see agent rules below */ },
  "lorebook": { /* see lorebook rules below */ }
}
```

## Ruleset content rules

`ruleset` MUST contain:

- `id` — kebab-case system identifier, e.g. `"dnd5e"`, `"exalted3e"`,
  `"blades-in-the-dark"`. Stable; used in idempotency keys.
- `name` — human display name, e.g. `"Dungeons & Dragons 5th Edition"`.
- `version` — semver string, e.g. `"1.0.0"`.
- `dice` — object describing the system's dice mechanic. Examples:
  `{"mode": "single-roll", "die": 20}`,
  `{"mode": "dice-pool", "die": 10, "successOn": 7, "doubleOn": 10}`,
  `{"mode": "fate-ladder", "die": "fudge", "count": 4}`,
  `{"mode": "dice-pool-sum", "die": 6}` (OpenD6 / WEG Star Wars / Mini Six).
  Choose ONE of: `single-roll | dice-pool | d100-percentile | 2d6-stat | fate-ladder | roll-under | stance-modal-pool | dice-pool-sum | narrative-handled`.
  See `docs/AUTHORING-PHASE-6.md` section 1 for the shape of each mode's full `resolution` block (especially `roll-under`, `stance-modal-pool`, `dice-pool-sum` with Wild Die, and `narrative-handled` which were added after Phase 5).
- `resolution` — short human-readable description of how rolls work in
  this system.
- `attributes` — array of `{name, abbreviation, min, max, default,
  description}` objects. Typically 4–8 attributes per system.
- `skills` — array of `{name, attribute, description}` objects.
  `attribute` references one of the attribute names.

`ruleset` MAY contain:
- `derivedStats` — array of `{name, formula, description, kind?}`
  with `formula` referencing other stats by name in `{Name}` placeholders.
- `conditions` — array of `{name, description}` for trackable status
  effects.
- `skillProficiency` — `{tiers: [{code, label, rollBonusFormula}], default}`
  for tier-based proficiency systems (PF2e U/T/E/M/L, D&D U/T/E,
  Exalted U/C/F).
- `skillSpecialties` — `{enabled: true, valueLabel, valueKind: "dice"|"value"|"successes", defaultValue}`
  for systems with named per-skill specialties (Exalted, oWoD).

## Agent prompt rules

`gmAgent` MUST contain:

- `name` — short display name, e.g. `"D&D 5e Rules Helper"`. Avoid the
  word `"Override"` (Game-Mode-extension framing) — this overlay
  cooperates with Roleplay Mode's default agents (`world-state`,
  `prose-guardian`, `continuity`, `expression`), it does not override
  them.
- `description` — one-line summary.
- `phase` — `"pre_generation"`. (Other phases are valid in the engine
  but not what we want here — pre-generation injects rules guidance
  before the model composes the turn.)
- `promptTemplate` — multi-paragraph prose handed to the LLM as system
  context before each turn. Minimum 50 characters.

The `promptTemplate` should:

- Identify the system being adjudicated.
- Describe the dice mechanic and difficulty ladder.
- Enumerate skills, attributes, and derived stats so the AI knows what
  to reference when narrating.
- Tell the AI how to format dice calls in narration (e.g. "When a
  skill check is needed, write `[<system>: skill="X" target="Y"]` and
  let the player roll.").
- Frame the agent as **cooperative** with the engine's default agents
  — say "you provide rules guidance alongside the world-state and
  continuity agents", not "you are the GM".
- NOT include the 50-character reputation cap workaround (that's a
  Game-Mode constraint and does not apply here).
- NOT include d20-tag format instructions unless the system actually
  uses d20 mechanics — and even then, prefer the in-extension dice
  widget over engine-format tags.

`gmAgent.settings` is optional. The installer will set
`mrrpManaged: true`, `mrrpBundleSchema: "mrrp-bundle"`,
`mrrpRulesetId: <ruleset.id>`, `mrrpAuthorId: <authorId>` automatically
on install.

## Optional sub-agents (`additionalAgents[]`)

A bundle MAY include focused sub-agents in `additionalAgents[]`. Two paths are available; pick one (do NOT enable both simultaneously or you get double-coverage):

**Recommended canonical path (post-2026-05-22 default):** the merged pool — `combat-overseer` (pre_generation, combat math + NPC roster in one prompt), `context-fuser` (pre_generation, rules-query answers + player-state reminder in one prompt), and `state-mutator` (post_processing). Reference prompts live at `agents/combat-overseer.md`, `agents/context-fuser.md`, `agents/state-mutator.md`. Two pre-gen AI calls per turn + one post-proc.

**Legacy path (v0.4.x compatibility):** the original five — `combat-adjudicator`, `npc-bookkeeper`, `lore-query`, `state-reminder`, `state-mutator`. Each has its own narrow responsibility. Reference prompts live at `agents/<role>.md`. Four pre-gen AI calls per turn + one post-proc — roughly 40% more cost than the canonical path. Kept available for users who prefer per-responsibility focus.

**Per-system parallel-phase overlays:** systems whose mechanics include resources no other RPG has (Exalted's anima banner, Charm cooldowns; VTM's blood pool) ship additional `parallel`-phase agents at `rulesets/<id>/agents/<role>.md`. They run alongside the main narrator without blocking it — zero added per-turn latency. Examples: `anima-banner-monitor`, `charm-cooldown-tracker` (Exalted); `blood-pool-tracker` (VTM).

**Install posture:** sub-agents install **disabled by default**. The user opts in per-agent in Marinara → Settings → Agents — RP mode has the per-agent toggle UI, so the bundle ships the menu and the user picks the path. If a specific sub-agent is so essential to your bundle that it should fire on first install, set `"enabled": true` on that item explicitly — but use sparingly; every enabled sub-agent costs one model call per turn (parallel overlays excepted).

**On re-install (PATCH), the user's enabled-toggle is preserved.** The installer carries `enabled` only on the initial CREATE. So a user who toggled a sub-agent on after install still has it on after re-install of the bundle.

**GM-mode alignment break (2026-05-22):** the sibling [GM-mode extension](https://github.com/Kenhito/Marinara-RPG-Extension) ships ONLY the canonical pool (no legacy four, no menu, all `enabled:true`) because GM-mode has no per-agent toggle UI in Marinara Settings. RP-mode preserves the two-path model because RP users CAN toggle.

**Document each sub-agent in the lorebook.** Conventionally, ship one lorebook entry titled "Optional Sub-Agents — what they do and how to enable" that lists each agent's purpose + the Settings → Agents flow. The dnd5e and exalted3e reference bundles in this repo show the canonical content.

**RP-mode framing for the sub-agent prompts.** Each sub-agent should follow the same cooperative framing rules as the main `gmAgent`: no "Game Mode" / "GM model" / "you are the GM" language. The cooperating-with-default-agents posture extends to every agent the bundle ships.

## Lorebook content rules

`lorebook` MUST contain:

- `name` — display name.
- `entries` — array of entry objects.

Each entry MUST contain:

- `name` — display name, e.g. `"Action Economy"`, `"Critical Success"`.
- `content` — multi-paragraph reference text that gets injected when
  any of the entry's keys appears in recent chat context.
- `keys` — array of trigger strings (case-insensitive substring match).

Each entry MAY contain:

- `position` — integer 0|1|2 (default 0). 0 = before character defs;
  1 = after; 2 = depth-injected.
- `depth` — integer (default 4). Used when `position === 2`.
- `selective` — boolean (default false).
- `constant` — boolean (default false). Always-on; ignores `keys`.
- `secondaryKeys` — array of secondary trigger strings.

A good lorebook for a TTRPG ruleset has ~15-25 entries covering: core
mechanics, conditions / status effects, common spells or abilities,
classes / archetypes, ancestries / origins, healing / recovery rules,
and any system-specific concepts that an AI without system knowledge
would otherwise hallucinate.

## Output format

Respond with the JSON file ONLY. No markdown fences, no commentary, no
preamble. The file must:

- Be valid JSON (no trailing commas, no comments, no JS-style backticks).
- Start with `{` and end with `}`.
- Use 2-space indentation.
- Have the `schema` field set EXACTLY to `"mrrp-bundle"` (lowercase).
- Have the `version` field set EXACTLY to `1`.
- Be self-contained — no external references, all content inline.

If the system requires research (rare classes, niche conditions),
prefer **fewer correct entries** over many entries with hallucinated
content. The lorebook is read by the AI during play; incorrect entries
will silently corrupt narration.

--- END PROMPT ---

## After the AI produces the bundle

1. Save the JSON output as `my-bundle.json` somewhere local (a text
   editor's save dialog, a gist, anywhere — just make sure it's a
   `.json` file on disk).
2. Open Marinara at `http://localhost:7860` (or wherever your install
   lives).
3. Install the framework if you haven't yet — Settings → Extensions →
   Add Extension → import `extension/RPG-Extension-RP-Mode.js` from this
   repo's release zip. (Marinara's Extensions UI is file-import, not
   text-paste.)
4. Click the Ruleset button that appears in the chat header.
5. Either import your `my-bundle.json` file (file-import button in the
   dialog) or paste its contents into the textarea.
6. Click Install. The dialog will report success and provision the
   custom agent + lorebook automatically.

You can re-install the same bundle as many times as you want — it's
idempotent. Updates replace existing entries; tags and idempotency
flags prevent duplicate agents/lorebooks.
