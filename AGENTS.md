# AGENTS.md — AI-Agent Reference for Marinara-RPG-RP-Mode-Extension

> **For AUTHORING A NEW RULESET (e.g., GURPS, Cyberpunk, Mörk Borg, etc.):** the canonical, system-agnostic, self-contained build documentation is at [`releases/v0.2.0/docs-for-ai/`](releases/v0.2.0/docs-for-ai/) — seven numbered Markdown files designed to be fed to any chat AI. Use those for new-ruleset authoring.
>
> This file (AGENTS.md) is for the deeper architectural concerns of working ON the framework itself.

## v0.2.1 changes — important context

Major changes since v0.2.0 (sheet ergonomics, item lifecycle, multi-system damage). All twenty-two rounds are itemized in `CHANGELOG.md` under `[v0.2.1]`; this is the cliff-notes view for anyone porting a new ruleset.

1. **State-mutator inventory tags accept the full dialog field set.** `[mrrp-state: action="add" field="inventory" add="X" ...]` now reads optional attrs `slot`, `damage`, `attack_attr`, `attack_proficient` (bool), `use_effect`, `consumable` (bool), `notes`, and `category` (`"equipment"` or `"item"`). Snake-case keys are LLM-friendly inside a tag. Empty strings are treated as "leave alone" so repeated adds with the same name bump quantity and enrich blank fields without clobbering populated ones. Document this in your per-ruleset state-mutator override under `# Inventory schema (full field list — extension-confirmed)`.
2. **Inventory items are normalized at every load.** `mergeSheet` runs every loaded item through `normalizeInventoryItem(it, idx)` which assigns a deterministic `item-heal-{nameSlug}-{idx}` id and fills missing dialog fields. Pre-fix items (no id, no category) self-heal in place — no migration script needed. Equipment-slot references survive across reloads because the synthetic id is name+index-keyed (stable).
3. **Damage parser is multi-system.** `parseDamageExpression(s)` returns one of three kinds: `dnd` (NdM[+K] [type], sums dice), `exalted` (`12L`/`12B`/`12A`/`12dL`/`12 Lethal`, rolls Nd10 and counts 7+ as successes per Exalted 3e damage convention — 10s do NOT double on damage), `flat` (`5 fire`, posts the value as-is). All three call sites — `rollWeaponDamage`, `useItem`, `castSpell` (`ability.damageDice`) — route through it. Add new ruleset notations by extending the parser; the call sites stay unchanged.
4. **Bars always show editable max alongside current.** `renderBar` no longer gates the max input behind `!hasExplicitMax`. For ruleset-capped bars (Exalted Motes via `maxFormula`, anything with literal `max`) the input default-displays the formula/literal value; typing persists to `state.sheet.derivedMax[name]` and overrides. `computeMax()` checks `derivedMax` FIRST so manual edits stick. `refresh()` re-syncs both inputs to current state after every state-mutator delta — fixes the "stale UI confuses the GM agent on next refresh" bug.
5. **Per-character bar caps via removal of literal `max`.** The exalted3e Willpower derived-stat dropped its `"max": 10` so each character's WP cap is user-set (canonically 5–10 for Solars). When porting a ruleset, only declare a literal `max` if the cap is universal across all characters; otherwise omit it and let `derivedMax` carry per-character values.
6. **Renderer for `renderAs: "value"` shows "/N" cap inline** when the derived stat has `max` or `maxFormula` declared. Essence reads "5 / 10" instead of bare "5". CSS class: `.mrrp-row__cap`.
7. **Header labels are per-ruleset** via `header.raceLabel` / `header.classLabel`. Exalted ships `"Type"` / `"Caste/Aspect"` (slash works for both Caste-using Exalted types and Aspect-using Dragon-Bloods). D&D ships `"Race"` / `"Class"`. Defaults to "Race"/"Class" when unset.

When in doubt about state-mutator inventory schema or damage notation, check `CHANGELOG.md [v0.2.1]` Round 18 / Round 19 / Round 22 entries — they document the wire format with worked examples.

---

## v0.2 changes — important context

Major architectural changes since v0.0.1:

1. **Agents decoupled from bundle.** `tools/build-bundle.mjs` produces `bundle.json` (ruleset + lorebook only). `tools/build-agents.mjs` produces a separate `agents.json`. Users install agents through Marinara's Import Agents dialog.
2. **System-agnostic agent baselines + per-system overrides.** Five role agents have shared baselines at `agents/<role>.md`. Per-system overrides at `rulesets/<system>/agents/<role>.md` win when present.
3. **Typed damage** on tracks via `damageTypes: [...]` on `renderAs: "track"` derived stats.
4. **Sorcery / multi-turn casting** ability category and state-mutator workflow.
5. **CSP-safe formula evaluator** using recursive-descent instead of `new Function`.
6. **State-mutator resolver normalization, max-clamp, persisted dedupe.**
7. **Lorebook install rewritten** — per-entry POST with delete-then-add.

When in doubt, `releases/v0.2.0/docs-for-ai/` is the current truth for ruleset authoring.

---

This file is the dense reference for AI agents (Claude, Codex, ChatGPT)
that need to make targeted changes to this repository. It pairs with
the (more user-facing) `README.md` and the (vibecoder-facing)
`AUTHORING-PROMPT.md`.

If you are an AI helping a human edit this repo, **read this file end
to end before proposing changes.** It is small enough that the cost is
trivial relative to the cost of guessing wrong about engine contracts
or the bundle envelope.

---

## What this repo is

A Marinara Engine **client extension** plus per-ruleset **install
bundles** that overlay custom RPG rules on Marinara's `chatMode:
"roleplay"` chats. Sister project to `Marinara-RPG-Extension`, which
targets `chatMode: "game"`.

The extension consists of:

1. A single-paste **client extension** — `extension/RPG-Extension-RP-Mode.js`,
   loaded into Marinara via Settings → Extensions. CSS is embedded
   inline into the JS string.
2. Per-ruleset **install bundles** — `rulesets/<system>/bundle.json` —
   pasted into the in-extension Ruleset dialog. The installer
   provisions a custom Marinara agent and a lorebook from each bundle.

---

## Engine contracts (verified by reading Marinara source)

These contracts came from reading the engine source at `~/Marinara-Engine/`
directly. They are stable as of Marinara `0.x` series (the version this
repo targets).

### Chat mode

```ts
type ChatMode = "conversation" | "roleplay" | "visual_novel" | "game";
```
(`packages/shared/src/types/chat.ts:6`.)

This repo targets `"roleplay"`. The overlay framework does not check
`chatMode` itself — it works in any mode where an extension can render
DOM and an agent can be installed. We document `roleplay` as the
intended target so the agent prompts and lorebook content are
authored with the right framing.

### Roleplay-mode default agents

```ts
roleplay: {
  defaultAgents: ["world-state", "prose-guardian", "continuity", "expression"],
}
```
(`packages/shared/src/constants/chat-modes.ts:23-28`.)

These are the engine's built-in agents that fire by default in a new
roleplay chat. Custom agents installed via this extension's bundle
run alongside them, not in place of them.

### Custom agent dispatch

The engine does **not** filter custom-typed agents by `chatMode`.
There are zero `chatMode` references in
`packages/server/src/services/agents/agent-pipeline.ts` or
`agent-executor.ts`. Dispatch is by `enabled` flag and `phase`
(`pre_generation` | `parallel` | `post_processing`).

Bundles in this repo install one agent with `type: "mrrp-overlay-v1"`
and `phase: "pre_generation"`. The engine's
`AGENT_RESULT_TYPE_MAP[type] ?? "context_injection"` fallback
(`agent-executor.ts:961`) means our type-string falls through cleanly
to `context_injection`, which is what we want.

### Engine API surface (from extension code)

The extension is invoked as `new Function("marinara", source)(marinara)`.
The `marinara` object exposes (`packages/client/src/components/CustomThemeInjector.tsx:72-146`):

- `extensionId`, `extensionName`
- `addStyle(css)` — append a `<style>` to head
- `addElement(parent, tag, attrs)` — DOM helper with auto-cleanup
- `apiFetch(path, opts)` — fetches `/api/<path>`. **Drop the `/api`
  prefix in calls** — `apiFetch("/agents")` not `apiFetch("/api/agents")`.
- `on(target, event, handler)` — event listener with auto-cleanup
- `setInterval(fn, ms)` / `setTimeout(fn, ms)` — auto-cleanup variants
- `observe(target, cb, opts)` — MutationObserver helper
- `onCleanup(fn)` — runs when extension is disabled

**Critical invariant:** the entire `RPG-Extension-RP-Mode.js` file runs as a
Function body. **No `import`, `export`, or top-level `await`.** All
top-level statements run on extension load.

### Lorebook routes used by the installer

- `POST /api/lorebooks` — create
- `PATCH /api/lorebooks/:id` — update
- `DELETE /api/lorebooks/:id` — remove
- `POST /api/lorebooks/:id/entries/bulk` — bulk add entries

Lorebook entry `position` is an **integer 0|1|2** (NOT a string).
Position semantics: `0` = before character defs (system context);
`1` = after; `2` = depth-injected (uses `entry.depth`).

### Agent routes used by the installer

- `POST /api/agents` — create
- `PATCH /api/agents/:id` — update
- `DELETE /api/agents/:id` — remove

Agent config schema requires `type: string`, `name: string(1-200)`,
`phase: pre_generation|parallel|post_processing`. Optional:
`description`, `enabled`, `connectionId`, `promptTemplate`,
`settings: object`. **No `resultType` field on the agent config** —
the engine derives it from `type`.

---

## Namespace (`mrrp-`)

Everything this overlay reserves on the user's Marinara instance is
prefixed `mrrp-` (or `MRRP_` / `mrrp` for camelCase / constants), so it
can coexist with the Game-Mode sister extension's `mrr-` namespace.
**Do not introduce any `mrr-` identifier into this repo** — that would
collide with the Game-Mode extension and break user installs that have
both.

| Namespace prefix | Used for |
|---|---|
| `mrrp-` | localStorage keys, CSS classes, lorebook tags, prompt-prefix `[mrrp-v1:...]`, `EMBED_STYLE_ID` |
| `MRRP_` | JS constants (`MRRP_AGENT_TYPE`, `MRRP_TAG_MANAGED`, `MRRP_TAG_RS_PFX`, `MRRP_PROMPT_PFX`) |
| `mrrp` (camelCase) | Agent settings flags (`mrrpManaged`, `mrrpRulesetId`, `mrrpAuthorId`, `mrrpBundleSchema`) |
| `mrrp:` | Lorebook ruleset-id tag prefix |

The bundle envelope discriminator is `"schema": "mrrp-bundle"`. The
installer routes on this string and rejects anything else, including
GM-mode extension bundles (which use `"schema": "mrr-bundle"`).

### localStorage keys

- `mrrp-active-ruleset` — active ruleset JSON blob
- `mrrp-active-ruleset-url` — last fetch URL
- `mrrp-ruleset-library` — `{[id]: {name, version, ruleset}}` map
- `mrrp-sheet-{chatId}-{characterId}` — per-character sheet state
- `mrrp-chars-{chatId}` — character list for that chat
- `mrrp-active-char-{chatId}` — selected character for that chat
- `mrrp-sheet-pos`, `mrrp-dice-pos` — drag positions
- `mrrp-sheet-size` — user-resized sheet dimensions
- `mrrp-sheet-collapsed-{chatId}` — collapsed/expanded sheet preference

### Console-callable diagnostics

- `window.mrrpDebug.dump()` — dump all extension state
- `mrrpDebug.state()` — current in-memory state object
- `mrrpDebug.read("KEY")` — read a localStorage key
- `mrrpDebug.forceSave()` — force-persist the active sheet

---

## Bundle envelope

```json
{
  "schema": "mrrp-bundle",
  "version": 1,
  "minExtensionVersion": "0.0.1",
  "authorId": "kenhito",
  "ruleset":  { /* full ruleset.json content */ },
  "gmAgent":  { "name": "...", "phase": "pre_generation", "promptTemplate": "..." },
  "lorebook": { "name": "...", "entries": [ /* ... */ ] }
}
```

The `gmAgent` field name is preserved from the Game-Mode extension for
schema continuity (the build tool, validator, and installer all
reference it). The displayed agent name is up to the bundle author.
For RP-mode overlays, prefer something like `"<System> Ruleset Helper"`
or `"<System> Rules Reference"` rather than the GM-mode-style
`"Override"` framing.

A bundle MAY also ship `additionalAgents[]` — see the next section.

---

## Sub-agent install behavior (`additionalAgents[]`, post-2026-05-04)

The installer treats the bundle's main `gmAgent` and its optional `additionalAgents[]` differently:

**Main `gmAgent`:** installs `enabled: true`. On re-install (PATCH), continues to set `enabled: true`. Load-bearing — the bundle doesn't work without it.

**Sub-agents in `additionalAgents[]`:** install **`enabled: false` by default.** Bundle authors can opt a specific sub-agent into enabled-on-install by setting `"enabled": true` on the additionalAgents item. The installer reads `ag.enabled === true` from the bundle data; absent or any non-true value → installer creates the agent disabled. Rationale: every enabled sub-agent fires every turn and costs a model call; users should explicitly enable only the ones they want via Marinara → Settings → Agents.

**Toggle preservation on re-install (PATCH):** the install code carries `enabled` only in the CREATE (POST) body, never in the UPDATE (PATCH) body. So if a user enables a sub-agent in Settings → Agents and later re-installs the bundle, their enabled choice survives:

```js
// extension/RPG-Extension-RP-Mode.js (around line 460-490)
var subBody = { /* full body including enabled: ag.enabled === true */ };
var subBodyForUpdate = Object.assign({}, subBody);
delete subBodyForUpdate.enabled;       // preserve user toggle on re-install
return existingSub
  ? apiFetch("/agents/" + existingSub.id, { method: "PATCH", body: JSON.stringify(subBodyForUpdate) })
  : apiFetch("/agents", { method: "POST", body: JSON.stringify(subBody) });
```

**Schema:** `bundle.schema.json` `additionalAgents` items declare `enabled` as an optional boolean (default false). Additive — existing bundles in production stay valid.

**Sub-agent prompt content** (`additionalAgents[].promptTemplate`): each sub-agent must follow RP-mode framing rules. No "Game Mode" / "GM model" / "you are the GM" language. The cooperating-with-default-agents posture extends to every agent the bundle ships. The reference prompts at `agents/<role>.md` in this repo are the source-of-truth — copy from there into the bundle.

**Lorebook documentation convention:** every bundle that ships sub-agents should also include one lorebook entry titled "Optional Sub-Agents — what they do and how to enable" so the user can ask the chat about them in-engine. The dnd5e/exalted3e reference bundles show the canonical content.

**Chat-message observer hardening (post-2026-05-04):** the chat-message `MutationObserver` no longer watches `characterData` mutations. The debounce uses `marinara.setTimeout` plus a monotonic-token cancel pattern instead of raw `setTimeout`/`clearTimeout`, so pending parses cannot fire after the extension is disabled. See `function watchChatMessages()` around line 3065 of `extension/RPG-Extension-RP-Mode.js`.

---

## Build / validation tools

```bash
# Validate every ruleset.json under rulesets/
node tools/validate-ruleset.mjs --all

# Build bundle.json from source files for every ruleset/<dir>/
node tools/build-bundle.mjs --all

# Validate every bundle.json under rulesets/
node tools/validate-bundle.mjs --all

# Re-embed CSS into RPG-Extension-RP-Mode.js after editing RPG-Extension-RP-Mode.css
node tools/embed-css.mjs
```

The single-source-of-truth for the embedded CSS is
`extension/RPG-Extension-RP-Mode.css`. After editing the CSS, run
`embed-css.mjs` to update the `EMBEDDED_CSS` string between the
`/* EMBEDDED_CSS_BEGIN */` and `/* EMBEDDED_CSS_END */` markers in
`RPG-Extension-RP-Mode.js`.

---

## Authoring a new ruleset

Two paths:

1. **Source-file workflow** (this repo's path) — author
   `ruleset.json`, `gm-agent.md`, and `lorebook.json` in
   `rulesets/<system>/`, then run `npm run build-bundles`.
2. **Vibecoder workflow** — paste `AUTHORING-PROMPT.md` into a chat-AI
   window with the system you want and ship the resulting `bundle.json`
   directly. No Node, no Git, no JS skills required.

Both paths produce identical `bundle.json` artifacts.

---

## Things to remember when editing this repo

- **Never introduce `mrr-` identifiers.** Always `mrrp-`.
- **Run `node --check extension/RPG-Extension-RP-Mode.js` after any JS edit.**
  The file is loaded into a `new Function` and a syntax error breaks
  the whole extension silently.
- **Run `node tools/embed-css.mjs` after editing the CSS.** The
  `EMBEDDED_CSS` string in the JS will otherwise drift from the source
  CSS.
- **Run `npm run validate-bundles` after editing any `bundle.json` or
  the schema.** All four bundles must validate before commit.
- **Cross-OS path examples in any new doc** — Linux + macOS + Windows.
  Don't ship a doc that only works on one platform.
- **No `git push` between releases unless Kenhito explicitly authorizes
  it.** Local commits accumulate; releases batch them. Force-push is
  always re-authorized per push.
- **The `gm-agent.md` filename is preserved from GM extension for
  build-tool continuity** even though this is the RP-mode repo. The
  prompt content inside should be RP-mode framed.
