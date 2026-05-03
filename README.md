# Marinara-RPG-RP-Mode-Extension

A Marinara Engine client extension that overlays a custom RPG ruleset
on **Roleplay Mode** chats — without forking the engine. Sister project
to [Marinara-RPG-Extension](https://github.com/Kenhito/Marinara-RPG-Extension)
which targets Game Mode.

> **Status:** v0.0.1 — initial scaffold. Not yet tested against a live
> Marinara install. Public release pending browser-test pass.

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
| Multi-agent overlay (tracker, lore)  | hard          | planned (v0.1+)     |

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
  bundle.schema.json     # Install bundle envelope schema.
  ruleset.schema.json    # Ruleset-data schema (shared semantics with GM ext).
tools/
  build-bundle.mjs       # ruleset.json + gm-agent.md + lorebook.json -> bundle.json
  validate-bundle.mjs    # Schema-validates bundle.json files.
  validate-ruleset.mjs   # Schema-validates ruleset.json files.
  embed-css.mjs          # Embed RPG-Extension-RP-Mode.css into RPG-Extension-RP-Mode.js.
rulesets/
  dnd5e/
  exalted3e/
docs/                    # (TBD)
AGENTS.md                # AI-agent reference for THIS repo.
AUTHORING-PROMPT.md      # Paste-ready prompt for vibecoder authors.
CHANGELOG.md             # Keep-a-Changelog.
```

## License

MIT. See `LICENSE`.

## Contributing

Issues and PRs welcome once the public repo lands. For now this is a
single-maintainer scaffold; please reach out before sending changes.
