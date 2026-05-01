# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

## [0.0.1] — 2026-05-01

Initial scaffold. Forked from `Marinara-RPG-Extension` v0.3.0 to target
Marinara Engine's **Roleplay Mode** (`chatMode: "roleplay"`) instead of
Game Mode (`chatMode: "game"`).

### Added
- Bundle envelope discriminator: schema string is now `mrrp-bundle`. The
  installer matches on this string and rejects bundles authored for any
  other Marinara overlay.
- `mrrp-` namespace across the framework — localStorage keys, agent
  type, agent settings flags, lorebook tags, CSS classes, embed style
  id, console debug surface, and bundle prompt prefix all now use
  `mrrp-` so this overlay can coexist on the same Marinara instance as
  the Game-Mode extension without collision.
- Two reference rulesets ported from the Game-Mode extension: D&D 5e
  (SRD 5.1) and Exalted 3rd Edition.

### Architectural changes from Game-Mode extension
- Targets Marinara's roleplay mode. Roleplay mode default agents are
  `world-state`, `prose-guardian`, `continuity`, `expression` — none of
  Game Mode's `game-master`, `combat`, `party-player`, or `quest`.
- No engine-imposed combat encounter modal, no d20-tag prompt format,
  no 50-character reputation `action` cap. The overlay owns all dice
  math and rules adjudication via the in-extension dice widget.
- Custom agents (`type: "mrrp-overlay-v1"`) dispatch in roleplay mode
  the same as in game mode — verified by reading
  `agent-pipeline.ts` / `agent-executor.ts` directly. Engine does not
  filter custom-typed agents by `chatMode`.

### Known limitations / next steps for v0.1
- Agent prompts in `rulesets/*/gm-agent.md` were ported with engine
  framing intact; they refer to "Game Mode" in places and include the
  reputation 50-char-cap workaround. These will be refreshed in a
  follow-up to drop game-mode-specific guidance and frame as roleplay
  cooperative-context-injection alongside the engine's default
  roleplay agents (world-state, prose-guardian, continuity).
- No browser-tested install yet against a running Marinara at
  `localhost:7860`. Public push deferred until that test passes.
- v0.0.1 ships a single overlay agent per ruleset. Multi-agent
  ambitions (state-tracker agent, lore-lookup agent) deferred to v0.1+.

[Unreleased]: https://github.com/Kenhito/Marinara-RPG-RP-Mode-Extension/compare/v0.0.1...HEAD
[0.0.1]: https://github.com/Kenhito/Marinara-RPG-RP-Mode-Extension/releases/tag/v0.0.1
