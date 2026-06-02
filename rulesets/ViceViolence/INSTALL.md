# Vice & Violence — Marinara-RPG-RP-Mode-Extension Installation

A ruleset bundle for the lewd, slapstick, sword & sorcery sandbox TTRPG **Vice & Violence**, running on Marinara Engine's Roleplay Mode.

## Prerequisites

- Marinara Engine running at `http://localhost:7860` (or wherever your instance lives)
- The Marinara-RPG-RP-Mode-Extension framework JS installed (one-time, system-independent)
- The `bundle.json` and `agents.json` files from this folder

## Step 1 — Install the client extension (one-time)

1. Go to Marinara → **Settings** → **Extensions** → **Add Extension**
2. Paste the framework JS (`RPG-Extension-RP-Mode.js`) into the JS field. Leave CSS empty.
3. Refresh Marinara. You should see a floating character sheet and a "Ruleset" button in the chat header.

## Step 2 — Activate the ruleset (paste bundle)

1. Click the **gear icon** on the floating sheet → **Ruleset**
2. Either use the **file import** button to select `bundle.json`, or paste the full contents of `bundle.json` into the textarea
3. Click **Install**. The dialog will confirm: ruleset + lorebook provisioned.

## Step 3 — Import the agents

1. Go to Marinara → **Settings** → **Agents** (the RP mode tab)
2. Click **Import** → paste the full contents of `agents.json`
3. Confirm. The six agents install as `MRRP: Vice & Violence — <Role>`.

## Step 4 — Enable the agents you want

All agents except the main narrator install **disabled by default**. Every enabled sub-agent costs one AI model call per turn. Recommended setup:

- **MRRP: Vice & Violence — Main** (always enabled, comes pre-enabled) — the actual narrator
- **MRRP: Vice & Violence — State Mutator** (toggle ON) — auto-updates your sheet when narration changes stats
- **MRRP: Vice & Violence — State Reminder** (toggle ON) — surfaces Health, Motes, Exertion, and status effects each turn
- **MRRP: Vice & Violence — Combat Adjudicator** (toggle ON) — wakes during combat to remind of auto-hit attacks, Armour Saves, and action economy

To toggle: Marinara → Settings → Agents → click the toggle next to each agent.

## Step 5 — Build a character

V&V characters start at **Level Zero** with no class. In the sheet:

1. Set your **Smarts, Brawn, Guts, Charm** ability scores (typically between -2 and +3 for a new character)
2. Set your **Health** (typically 8-12)
3. Set your **Motes** (0 if you don't start with spells)
4. Set your **Exertion** to 3
5. Use the **Inventory** section to track your weapon, armour, potions, rations, water, and light sources
6. Use the **Vices** section (Backgrounds) to track any Vices you accumulate

## Step 6 — Play

1. Start a new chat
2. The GM agent will narrate in the V&V tone (slapstick, comedic, lewd)
3. When you need to roll a Dice Challenge (DC12 Smarts to cast Fireball), type your intended action and the narrator will prompt the roll
4. Combat attacks auto-hit — just roll damage (weapon + material + Brawn)
5. The State Mutator will emit hidden `[mrrp-state: ...]` tags that update your sheet automatically

## Troubleshooting

**"My Health didn't update after a fight":** The State Mutator must be enabled (Settings → Agents → toggle ON). Also check that the narrator actually described damage — if armour deflected the blow, no Health delta is emitted.

**"The narrator keeps asking for attack rolls":** V&V attacks auto-hit. If the narrator says "roll to hit," remind it in-character or with an OOC note that V&V only rolls damage. The combat adjudicator agent helps prevent this.

**"Reputation tags are failing":** Marinara's `[reputation: npc="Name" action="..."]` tags have a 50-character limit on `action`. Keep action descriptions short (e.g., "saved from bandits" not "heroically rescued from a pack of vicious bandits").

**"My sheet is empty in a new chat":** Marinara's chat IDs rotate per session. Sheets are keyed to chat ID in localStorage. Use the sheet's **Save/Load** buttons to export/import your character as a JSON file between sessions.

**"The narrator uses D&D terms (AC, proficiency, spell slots)":** Remind the narrator that V&V uses Auto-Hit Attacks, Armour Saves, Motes, and Ability Checks. The combat adjudicator agent reinforces V&V vocabulary.

## Reinstalling

You can reinstall the bundle and agents as many times as you want — it's idempotent. Updates replace existing entries. If you reinstall the bundle, your enabled agent toggles are preserved (the installer only sets `enabled` on CREATE, not on PATCH).