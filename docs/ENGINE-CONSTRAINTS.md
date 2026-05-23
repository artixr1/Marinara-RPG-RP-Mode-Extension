# Engine constraints — what this overlay can and cannot do

This page is the honest-tradeoff document for the **roleplay-mode** extension. The Game-Mode sibling repo has its own ENGINE-CONSTRAINTS doc enumerating Game-Mode-specific issues (combat-modal d20-shape, 50-character reputation action cap, encounter route hardcoding). This document covers what's true for **roleplay-mode** chats specifically — a different constraint surface.

Verified against Marinara Engine v1.5.6 (April 2026). If a future Marinara release changes any of these, this doc is wrong; please open a PR.

## What the overlay can change

### Cooperative GM-side rules guidance

Your custom `gmAgent` is a `pre_generation` `context_injection` agent that runs **alongside** Marinara's default roleplay-mode agents (`world-state`, `prose-guardian`, `continuity`, `expression`). It contributes rules brief — "this action calls for a Stealth check at DC 17, the player's modifier is +6, the in-extension dice widget will resolve" — without trying to be the GM. The default agents continue to handle world bookkeeping, prose quality, narrative continuity, and emotional register.

### Optional sub-agents (`additionalAgents[]`)

Each bundle can ship two mutually-exclusive sub-agent paths in `additionalAgents[]`, plus optional per-system parallel-phase overlays:

- **Canonical (recommended):** `combat-overseer`, `context-fuser`, `state-mutator`. Two pre-gen AI calls per turn + one post-proc.
- **Legacy (v0.4.x compatibility):** `combat-adjudicator`, `npc-bookkeeper`, `lore-query`, `state-reminder`, `state-mutator`. Four pre-gen AI calls per turn + one post-proc.
- **Per-system parallel overlays:** e.g. `anima-banner-monitor` + `charm-cooldown-tracker` for Exalted, `blood-pool-tracker` for VTM. Run alongside the narrator without blocking it.

Per the post-2026-05-04 install behavior, sub-agents install **disabled by default** in RP mode — users opt into a path via Settings → Agents. Bundle authors can mark a specific agent as `"enabled": true` on the additionalAgents item to ship it enabled (use sparingly). Running both pre-gen paths simultaneously causes double-coverage; pick one.

### Lorebook content + keyword triggers

Standard Marinara lorebook with keyword-based scan injection. Use `position: 0|1|2` (integer), tune `tokenBudget`, leave `recursiveScanning: false` unless you've measured a real benefit. The bundle's lorebook installs idempotently via the `mrrp-managed` + `mrrp:<id>` tag pair.

### Character sheet UI replacement

The extension replaces the engine's built-in attribute panel with a ruleset-aware sheet that respects your `ruleset.json`'s attribute count, groupings, abbreviations, and min/max bounds. Skills, derived stats, conditions, and sheet sections all render from the ruleset spec. This works identically to the GM-mode sibling extension.

### Floating dice widget

The widget supports the same nine resolution modes as the GM-mode sibling: `single-roll` (d20), `dice-pool` (Xd10 with doubles), `d100-percentile`, `2d6-stat with bands`, `fate-ladder` (4dF with style margin), `roll-under` (3d6 / 1d100 ≤ target), `stance-modal-pool` (L&F / Stewpot / Trophy Dark), `dice-pool-sum` (OpenD6 / WEG Star Wars / Mini Six with optional Wild Die explode), `narrative-handled` (prose-resolved / GM-overrules). Resolution happens client-side; the formatted result can be sent into the chat input.

### Per-character persistence + cross-chat save/load

Same as GM-mode: characters persist to localStorage keyed to chat ID; the sheet header has save and load buttons that download/upload all characters in the active chat.

## What the overlay does NOT do (and cannot, without forking)

### Cannot replace Marinara's default roleplay agents

The default roleplay-mode agent set (`world-state`, `prose-guardian`, `continuity`, `expression`) is server-managed and is the right thing to leave in place for a roleplay-mode chat. Your `gmAgent` and any optional sub-agents COOPERATE with these — they don't override them. If you find yourself wanting to disable a default agent, that's a sign you should be running the GM-mode extension instead.

### Cannot intercept the streaming generation pipeline

The extension watches finished chat messages (debounced ~1.5s after the last DOM mutation) and parses tags like `[mrrp-state: …]` AFTER the message stabilizes. It cannot:

- Modify mid-stream tokens.
- Replace the streaming agent's output before it appears.
- Hook into Marinara's prompt-assembly pipeline.

If a sub-agent emits a malformed `[mrrp-state: …]` tag, you see it in the rendered message and the extension's parser silently skips invalid tags — but the model has already generated the prose around it.

### Cannot add custom tools to the model

Marinara has its own custom-tool subsystem (`webhook` / `script` / `static` execution types). The extension does NOT register Marinara custom tools. Dice resolution happens client-side via the floating widget; sheet mutation happens client-side via the state-mutator tag protocol. If you need the model to **invoke** a tool (e.g., "model decides to call `lookup_lore`"), you'd add that directly via Marinara's Settings → Custom Tools UI — outside the bundle's scope.

### Cannot persist custom state outside Marinara's primitives

Character sheets persist via localStorage keyed to chat ID. There is no overlay-managed cross-chat database. If you need cross-campaign global state (a single character that follows you across chats, a shared NPC pool, a campaign log), you'd build that yourself behind a Marinara webhook tool — see the GM-mode sibling's `docs/AUTHORING.md` for the pattern.

### Cannot guarantee the `state-mutator` tag protocol works on every model

The state-mutator agent instructs the narration model to emit `[mrrp-state: …]` tags inline. Frontier models (Claude 4, GPT-5, Gemini Pro) follow these instructions reliably; smaller / local models may emit malformed tags or omit them entirely. If you're running a 7B local model, the state-mutator probably isn't earning its token cost — keep it disabled and mutate the sheet manually.

### Cannot validate ruleset prompts at install time

`tools/validate-bundle.mjs` checks the JSON schema (required fields, integer position, string lengths, regex patterns). It does NOT evaluate whether your `gmAgent.promptTemplate` actually produces useful rules guidance, whether the `[mrrp-state: …]` instructions in your state-mutator will be followed by your model, or whether your lorebook entries trigger correctly on real chat content. Those are runtime concerns — test in a real chat, with your actual model, before declaring a bundle done.

## Authoring implications

When authoring a new ruleset bundle for this overlay:

- **Frame the gmAgent cooperatively.** "You provide rules guidance for ⟨system⟩ in Marinara Engine's roleplay mode, working alongside the engine's default world-state, prose-guardian, continuity, and expression agents." NOT "You are the GM." NOT "running inside Marinara Engine's Game Mode."

- **Don't include the 50-character reputation tag workaround.** That's a Game-Mode-specific engine constraint; it doesn't apply here.

- **Prefer the in-extension dice widget over engine-format dice tags.** Even for d20-based systems (D&D 5e, Pathfinder 2e), instruct the narration model to call out the check ("that calls for an Athletics check at DC 15") and let the player resolve via the widget. Only emit `[d20: …]` engine tags when the player has explicitly asked for automatic resolution.

- **Test with the default agents enabled.** The whole point of roleplay-mode is to cooperate with `world-state`, `prose-guardian`, etc. If your bundle behaves differently with those agents disabled vs enabled, the bundle's framing is wrong.

- **Keep sub-agent prompts terse and gated.** Each enabled sub-agent fires every turn. Use explicit early-out gates ("If no combat is active, output exactly: 'No combat active.' and stop.") to keep cost bounded.

## What's the practical effect of these constraints?

A roleplay-mode bundle is a *style sheet* for narration mechanics — it tells the cooperating agent stack "in this campaign, here's how rules are framed and how rolls are formatted." It does not seize control of the narrative pipeline; it leans on the engine's defaults and adds focused mechanical guidance. Bundles that fight that posture (ones that try to be the sole GM, ones that demand the model emit engine tags for every check, ones that want streaming interception) belong in the GM-mode sibling repo.

## Cross-references

- `docs/ADDING-RULESETS.md` — full authoring walkthrough.
- `docs/AUTHORING.md` — bundle anatomy and the eight-step authoring process.
- `AUTHORING-PROMPT.md` — single-paste prompt for vibe-coding a new bundle with a chat AI.
- The Game-Mode sibling repo's `docs/ENGINE-CONSTRAINTS.md` — Game-Mode-specific constraints (combat modal, encounter routes, reputation cap).
