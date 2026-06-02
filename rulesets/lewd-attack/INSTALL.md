# Install — Lewd Attack ruleset

## Quick install (recommended)

**One file import + one bundle install.** Skip step 1 if the framework extension is already installed.

### 1. Install the framework extension (once per Marinara install)

In Marinara Engine: **Settings → Extensions → Add Extension** — Marinara's
Extensions screen accepts file uploads, not pasted text.

- **Import** the file `extension/RPG-Extension-RP-Mode.js` from the Marinara-RPG-RP-Mode-Extension repo. The CSS is embedded — there is no separate stylesheet to upload.
- Name: `Marinara-RPG-Extension`. Description: anything.
- Enable.

A **Ruleset** button appears in the chat header.

### 2. Install the Lewd Attack bundle

Click the **Ruleset** button. The dialog has three ways to load a bundle:

- **Option A — Choose file:** click **Choose file…** and pick `rulesets/lewd-attack/bundle.json` from disk. Click **Save and reload**.
- **Option B — Fetch URL:** point to wherever `bundle.json` is hosted.
- **Option C — Paste:** copy the contents of `bundle.json` into the textarea, click **Save and reload**.

The installer creates the lorebook ("MRRP: Lewd Attack Reference"), the custom GM agent, and activates the ruleset. The page reloads.

### 3. Import agents (optional but recommended)

- Marinara → **Settings → Agents → Import** — paste `rulesets/lewd-attack/agents.json` (if you've run `build-agents.mjs`) or manually create each agent from the `.md` files in `agents/`.
- Toggle on **state-mutator** and **state-reminder** for automatic sheet updates.
- Toggle on **combat-adjudicator** for structured combat resolution.
- Leave lore-query and npc-bookkeeper toggled on or off as preference.

## Sanity check

In a fresh Roleplay Mode chat:

1. Open the dice widget. It should render a d6 dice-pool form: Pool size, Difficulty.
2. Set Pool = 7 (e.g., Strength 3 + Agility 4 for a Swords attack), Difficulty = 2. Click **Roll d6s**.
3. You should see `[dice: 7d6 vs 5+ → N successes]` — counting 5s and 6s as successes.
4. Send to chat. The GM agent picks up the result and narrates accordingly.

## Character setup

1. In the floating sheet, set your 9 attributes (default 2, max 5 at creation).
2. Derived stats (HP, MP, Stamina, Sanity) auto-compute from attributes.
3. Set Lust to your starting 1d20 roll.
4. Use the Skills section to note which skills are Trained (at creation, you're Trained in class skills + 3 Sex Skills).

## Lust/Satisfaction workflow

During play, the narration model tracks Lust accumulation and Satisfaction during sex scenes. Key moments:
- **Daily Lust:** +1d4 per day (modified by traits like Lusty or Heat).
- **Lust triggers:** seeing enemies (armor coverage dependent), groping, fetish exposure.
- **Sex scenes:** Satisfaction accumulates from Sex Skill rolls. Compare to Lust threshold (10 default) for orgasm.
- **Mind Break:** Sanity hitting 0 during sex triggers roll on Mind Break table.

## Updating

Bundle update flow is the same as install — Choose file again or paste the new `bundle.json`. The installer detects the existing managed agent/lorebook and PATCHes rather than duplicating.

## Removing

Open the Ruleset dialog and click **Uninstall server data** to remove the lorebook and GM agent created by this install. Click **Clear** to wipe the local ruleset cache.

## Manual install (legacy / source-of-truth path)

1. **Settings → Extensions** — import `extension/RPG-Extension-RP-Mode.js`.
2. **Ruleset button** — Choose file / paste `rulesets/lewd-attack/ruleset.json`.
3. **Settings → Agents → Create Custom Agent** — copy the prompt from `rulesets/lewd-attack/gm-agent.md` (Phase: pre_generation, Result type: context_injection).
4. **Lorebooks → Import** — import `rulesets/lewd-attack/lorebook.json`.

The bundle path automates all steps from one file import.

## Engine compatibility notes

- **Reputation tags:** Marinara's `[reputation: npc="Name" action="..."]` tags cap `action` at 50 characters. Keep action descriptions short to avoid 400 errors.
- **Chat-ID sheet keying:** Marinara chat IDs rotate per session. Sheets in localStorage are keyed by chat ID; a fresh chat looks like a new character. Use the sheet's save/load buttons to export/import characters as JSON files.
- **Combat modal:** Marinara's combat-encounter modal is hardcoded d20-style. Lewd Attack uses narrative combat (chat-based) — ignore the modal and resolve through opposed dice pools in the chat widget.
- **`mrrp-` prefix:** This ruleset uses RP-mode tags. All state tags are `[mrrp-state: ...]`. Do not use `[mrr-state: ...]` (GM-mode prefix) — the parser will drop them.
