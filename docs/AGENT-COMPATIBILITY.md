# Agent compatibility — Marinara built-ins alongside the RP-mode sub-agents

This file is the in-repo reference for which of Marinara Engine's built-in
agents are safe to enable in a chat that already runs this extension's five
`mrrp-` sub-agents (`state-mutator`, `state-reminder`, `combat-adjudicator`,
`lore-query`, `npc-bookkeeper`). The goal is to avoid two failure modes:

1. **Double-tap.** Two agents inject contradictory or duplicate context every
   turn. Wastes tokens, confuses the narration model.
2. **Silent overwrite.** Built-in editor or tracker agents mutate output that
   our extension parses (`[mrrp-state: ...]` tags) or owns
   (`mrrp-managed`-tagged lorebook entries) without warning.

The matrix below was derived directly from engine source — see [Sources](#sources)
at the bottom. Regenerate when Marinara ships new built-ins.

## Our 5 sub-agents (recap)

All five are `pre_generation` `context_injection` agents. All install
**disabled by default** (post-2026-05-04 audit). All in the `mrrp-` namespace.

| Sub-agent | What it does |
|---|---|
| `state-mutator` | Tells the narration model to emit `[mrrp-state: ...]` tags when narrative establishes a durable sheet change (HP, conditions, inventory). The extension's chat-message observer parses the tags from the rendered message and applies them to the active character's `mrrp-sheet-{chatId}-{characterId}` localStorage. |
| `state-reminder` | Surfaces a short bulleted list of current PC mechanical state (HP, resources, conditions, equipped gear) at the top of every turn so narration stays consistent with the sheet. |
| `combat-adjudicator` | Wakes only in combat. Restates initiative, action economy, attack/damage formulas in the **active ruleset's** terms. Outputs `"No combat active."` and stops in ambient/social scenes. |
| `lore-query` | Wakes only when the latest user message is a rules question. Cites the installed lorebook + system RAW. Outputs `"No rules query."` otherwise. |
| `npc-bookkeeper` | Tracks active and recently-engaged NPC HP, conditions, tactical state, and intent across turns. Outputs `"No NPCs to track."` when no NPCs are in scene. |

## Engine roleplay defaults — already on, leave them on

Marinara auto-installs four agents into every new `chatMode: "roleplay"`
chat (source: `~/Marinara-Engine/packages/shared/src/constants/chat-modes.ts:23-28`):

| Agent | Verdict | Reason |
|---|---|---|
| `world-state` | **KEEP ON** | Tracks scene chrome — date, time, weather, location, present characters. Different content from `state-reminder` (PC mechanical state). They coexist cleanly. |
| `prose-guardian` | **KEEP ON** | Writing-variety enforcement. Orthogonal to every mrrp- agent. |
| `continuity` | **KEEP ON** | Contradiction detection against established lore. Orthogonal. |
| `expression` | **KEEP ON** | VN sprite picker. Visual concern, no overlap. |

## Verdict legend

- **SAFE** — orthogonal function. Enable freely.
- **CAUTION** — no logical conflict, but real risk of clobbering or polluting
  something this extension owns. Test in a throwaway chat first.
- **DO NOT ENABLE** — direct conflict with a mrrp- sub-agent. Pick one,
  never both.
- **N/A** — Conversation-mode-only agent; doesn't apply in roleplay.

## Writer category

| Agent | Verdict | Notes |
|---|---|---|
| `director` | **SAFE** | Narrative beat injector. Cooperates with mrrp- agents — they handle mechanics, it handles plot motion. Enable when scenes feel static. |
| `prompt-reviewer` | **SAFE** | One-shot preset analyzer. Doesn't run per-turn. Run once when authoring a new ruleset prompt, then disable. |
| `secret-plot-driver` | **SAFE** | Hidden long-term arc memory. Operates on a private channel that doesn't see `[mrrp-state:]` tags. |
| `editor` | **CAUTION** | Post-process **rewrites the assistant response.** Reads all agent data (trackers, prose, continuity) and edits to fix "factual errors, outfit/stat contradictions, repetition." It receives the full reply *including* our `[mrrp-state:]` tags — it could view them as artifacts and strip or paraphrase them, breaking sheet updates. Test in a throwaway chat first; if your tags survive a few turns of editing, fine. Source: `agent-prompts.ts:500`. |
| `knowledge-retrieval` | **DO NOT ENABLE** | Conflicts with `lore-query`. Both retrieve and inject lorebook content. `knowledge-retrieval` runs every turn (per-entry summarization); `lore-query` runs only on rules questions. Running both = double lorebook injection and contradictory framing. |
| `knowledge-router` | **DO NOT ENABLE** | Same conflict as `knowledge-retrieval`, lower-cost catalog-pick variant. Still injects every turn. Pick `lore-query` (sharper, ruleset-RAW-aware) or one of these (broader, story-context-aware), not both. |

## Tracker category

| Agent | Verdict | Notes |
|---|---|---|
| `quest` | **SAFE** | Quest objective tracker. The mrrp- sheet doesn't track quests — orthogonal. Enable for long campaigns. |
| `background` | **SAFE** | Selects scene background image. Visual-only. |
| `card-evolution-auditor` | **SAFE** | Proposes Marinara character-card edits when narrative drift is detected. Operates on engine character cards, not on `mrrp-sheet-*` localStorage — different storage targets. |
| `character-tracker` | **DO NOT ENABLE** | Conflicts with `npc-bookkeeper` (NPCs) and partially with `state-reminder` / `state-mutator` (PC HP/mood). Tracks "every character — present, mood, actions, appearance, outfit, thoughts, per-character stats (HP, etc.)." Two trackers writing competing snapshots into context every turn. If you want the broader "every character" view it offers, disable `npc-bookkeeper` first. |
| `persona-stats` | **DO NOT ENABLE** | Conflicts with `state-mutator` and `state-reminder`. Tracks "Satiety, Energy, Hygiene, and other custom stats" for the player persona — same conceptual surface as our PC sheet but a different storage target (engine widget vs. `mrrp-sheet-{chatId}-{characterId}` localStorage). Two systems updating PC stats from the same narrative produces drift. |
| `custom-tracker` | **CAUTION** | Tracks user-defined fields. Whether it conflicts depends entirely on what fields you point it at. Tracking "favorite color" or "audience reactions" → no conflict. Tracking "HP" or "spell slots" → duplicates `state-mutator` and the two will diverge. Only point `custom-tracker` at fields the mrrp- sheet does NOT cover. |

## Misc category

| Agent | Verdict | Notes |
|---|---|---|
| `illustrator` | **SAFE** | Image-prompt generator at key moments. Visual-only. |
| `html` | **SAFE** | Injects a directive encouraging inline HTML/CSS for in-world UIs (signs, posters, terminals). Orthogonal. |
| `chat-summary` | **SAFE** | Rolling summary every X messages. Cooperative — long-RP memory aid. |
| `spotify` | **SAFE** | Music control. Orthogonal. |
| `cyoa` | **SAFE** | CYOA choices generator (Roleplay-mode only per spec). Orthogonal. |
| `echo-chamber` | **SAFE** | Simulated live-stream chat reactions. Orthogonal. |
| `haptic` | **SAFE** | Haptic toy control via Buttplug.io. Orthogonal. |
| `combat` | **DO NOT ENABLE** | Conflicts with `combat-adjudicator`. Engine `combat` runs in `parallel` phase tracking generic initiative/HP/turns. Ours runs `pre_generation` enforcing the **active ruleset's** dice math (D&D d20+mod, Exalted dice pool, Fate ladder, PbtA 2d6+stat). Two combat agents = two contradictory injections every combat turn. Pick ours when running an mrrp- ruleset. |
| `lorebook-keeper` | **CAUTION** | Auto-creates lorebook entries from story events. **Risk: it may write to a lorebook our installer owns** (anything tagged `mrrp-managed` or `mrrp:<rulesetId>`). On re-install, our installer does delete-then-add per entry, so any auto-keeper additions to the ruleset lorebook get blown away. Mitigation: create a separate, untagged lorebook ("Campaign Lore" or similar) and scope `lorebook-keeper` writes to that one. Don't let it touch the ruleset lorebook. |

## Conversation-only — N/A in roleplay

| Agent | Verdict | Notes |
|---|---|---|
| `schedule-planner` | **N/A** | Weekly character schedules, Conversation mode. |
| `response-orchestrator` | **N/A** | Group-Conversation message routing. |
| `autonomous-messenger` | **N/A** | Unprompted DMs, Conversation mode. |

## Direct-conflict summary

If you remember nothing else:

- **Don't enable `combat`** while running mrrp- `combat-adjudicator`. Two combat brains.
- **Don't enable `knowledge-retrieval` or `knowledge-router`** while running mrrp- `lore-query`. Two lorebook injectors.
- **Don't enable `character-tracker`** while running mrrp- `npc-bookkeeper`. Two NPC trackers.
- **Don't enable `persona-stats`** while running mrrp- `state-mutator`. Two PC-stat brains writing to different stores.

## Enable-with-care landmines

- `editor` can rewrite the model's reply *including* our `[mrrp-state:]` tags.
  Test before relying on it.
- `lorebook-keeper` can pollute or get clobbered by our lorebook installer.
  Scope it to a separate untagged lorebook.
- `custom-tracker` is fine for fields the mrrp- sheet doesn't cover. Don't
  point it at HP, resources, conditions, or inventory.

## How to enable a Marinara built-in

Marinara stores agents per-chat. To turn one on:

1. Open the chat's gear / settings drawer.
2. Go to **Settings → Agents → Writer Agents** (for writer-category) or the
   matching tracker / misc panel.
3. Find the agent (named per the matrix above).
4. Toggle on. Persists for that chat.

To turn off, toggle off. To remove from the chat entirely, use the engine's
agent management UI; the built-in itself stays installed at the engine level.

## Sources

This matrix is derived from these engine files. When Marinara ships new
built-ins, regenerate from these:

| What | Path |
|---|---|
| Built-in agent registry (28 agents) | `~/Marinara-Engine/packages/shared/src/types/agent.ts:195` (`BUILT_IN_AGENTS`) |
| Built-in agent ID enum | `~/Marinara-Engine/packages/shared/src/types/agent.ts:151` (`BUILT_IN_AGENT_IDS`) |
| Agent prompt templates | `~/Marinara-Engine/packages/shared/src/constants/agent-prompts.ts` |
| Roleplay mode default agents | `~/Marinara-Engine/packages/shared/src/constants/chat-modes.ts:23-28` |
| Pipeline / no `chatMode` filtering | `~/Marinara-Engine/packages/server/src/services/agents/agent-pipeline.ts` |
| Our 5 sub-agent sources | `~/projects/Marinara-RPG-RP-Mode-Extension/agents/*.md` |

The wiki mirror (with `[[wikilinks]]` for Obsidian navigation) is at
`~/cc-wiki/How-To/Marinara-RP-Agent-Compatibility.md`.
