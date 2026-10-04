# AAK Skills

Agent skills for [Action Adventure Kit](https://assetstore.unity.com/packages/slug/217284) (AAK), a component-based action adventure framework for Unity by [SoftLeitner](https://softleitner.com).

Each skill is a `SKILL.md` file describing one part of AAK - what it's for, how to use it, and common pitfalls - so an AI coding assistant can answer questions and make changes in an AAK project without guessing at the API.

## Install

Clone or download this repo, then copy whichever skill folders you want into wherever your tool of choice looks for skills:

git clone https://github.com/Schossi/AAKSkills.git

Each folder (`action-adventure-kit-*/SKILL.md`) is self-contained and can be copied on its own - check your tool's docs for where it expects skill files to live.

## Skills

| Skill | Covers |
|---|---|
| `action-adventure-kit-setup-and-overview` | Confirming AAK is installed, first-run setup, module layout, demo and manual scenes, using AAK without URP |
| `action-adventure-kit-character-movement` | `GenericCharacter`, `CharacterControllerMovement` with Input System wiring, NavMesh NPCs (patrol/approach/wander), 2D movement, character instructions |
| `action-adventure-kit-items-inventory` | Item assets, `ItemSet`, listed and tiled inventories, slots and equipment, capacities |
| `action-adventure-kit-actions-interactions` | Actors and action areas, pickups, gates/doors, buttons, dialogs, purchases, trigger zones |
| `action-adventure-kit-health-stats-damage` | Resources (health, stamina), attributes and stats, damage senders/receivers, modifiers |
| `action-adventure-kit-status-effects` | `EffectType`/effect prefabs, `EffectPool`, timed and zone-based buffs and debuffs |
| `action-adventure-kit-save-load` | `PersistenceContainer`, persisters and keys, save slots, inspecting and resetting save data |

Start with `action-adventure-kit-setup-and-overview` - it confirms AAK is present and points to the right sibling skill for the task at hand.

Each skill includes prerequisites, a quick start, detailed workflows, verification steps, and common issues, and is pinned to a specific AAK version via its frontmatter (`asset-version`, `unity`, `last-verified`).

## Links

- [Unity Asset Store](https://assetstore.unity.com/packages/slug/217284)
- [Manual](https://adventure.softleitner.com/manual)
- [API Reference](https://adventureapi.softleitner.com/)

## License

MIT - see [LICENSE](LICENSE).
