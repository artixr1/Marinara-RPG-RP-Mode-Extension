# AGENTS.md — AI-Agent Reference for Marinara-RPG-RP-Mode-Extension

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

1. A single-paste **client extension** — `extension/ruleset-loader.js`,
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

**Critical invariant:** the entire `ruleset-loader.js` file runs as a
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

---

## Build / validation tools

```bash
# Validate every ruleset.json under rulesets/
node tools/validate-ruleset.mjs --all

# Build bundle.json from source files for every ruleset/<dir>/
node tools/build-bundle.mjs --all

# Validate every bundle.json under rulesets/
node tools/validate-bundle.mjs --all

# Re-embed CSS into ruleset-loader.js after editing ruleset-loader.css
node tools/embed-css.mjs
```

The single-source-of-truth for the embedded CSS is
`extension/ruleset-loader.css`. After editing the CSS, run
`embed-css.mjs` to update the `EMBEDDED_CSS` string between the
`/* EMBEDDED_CSS_BEGIN */` and `/* EMBEDDED_CSS_END */` markers in
`ruleset-loader.js`.

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
- **Run `node --check extension/ruleset-loader.js` after any JS edit.**
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
