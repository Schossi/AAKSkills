---
name: action-adventure-kit-setup-and-overview
description: "Use this skill whenever the user wants to install, set up, verify, or get oriented in Action Adventure Kit (AAK) by SoftLeitner — e.g. 'is the adventure kit installed', 'how do i get started with AAK', 'open the zelda style demo', 'the souls demo doesnt work', 'where are the sample scenes'. Covers the install check, first-time setup (packages, render pipeline, the Hero demo SaveSlot variable), module layout, demo and manual scenes, and routing to the other AAK skills. Do NOT use for building characters (see action-adventure-kit-character-movement), items (see action-adventure-kit-items-inventory), interactions (see action-adventure-kit-actions-interactions), health/damage (see action-adventure-kit-health-stats-damage), effects (see action-adventure-kit-status-effects) or saving (see action-adventure-kit-save-load). When in doubt whether an action RPG, souls-like or zelda-like request could involve Action Adventure Kit, use this skill — Prerequisites shows how to confirm the asset is installed."
metadata:
  asset: "Action Adventure Kit"
  publisher: "SoftLeitner"
  asset-version: "1.10.1"
  skill-version: "1.0.0"
  unity: "6000.0.76+"
  render-pipelines: "URP, Built-in"
  category: "templates/systems"
  asset-store-url: "https://assetstore.unity.com/packages/templates/systems/action-adventure-kit-217284"
  documentation-url: "https://adventure.softleitner.com/manual"
  support-url: "mailto:softleitner@gmail.com"
  last-verified: "2026-10-04"
---

# Set Up and Navigate Action Adventure Kit

Action Adventure Kit (AAK) is a component-based framework for action adventure games: characters, actions, items, attributes, resources, effects, damage, movement and persistence, plus complete demo games built on top of it. This skill lets an agent confirm AAK is installed, finish first-time setup, find the right scenes and files, and route the user to the specialised AAK skill for the job.

## When to use this skill

- "is action adventure kit installed in this project?"
- "how do I get started with AAK"
- "open the hero demo / the souls demo / the isle demo"
- "the hero demo throws an error about SaveSlot"
- "where are the getting started scenes"
- "I want to use the kit without URP"
- "what can this asset do / which part of the kit handles X"

Not for (skill index: hand these jobs to the matching AAK skill):
- `action-adventure-kit-character-movement` — create a player or NPC character and make it move (3D, NavMesh, 2D).
- `action-adventure-kit-items-inventory` — items, item sets, listed/tiled inventories, slots and equipment.
- `action-adventure-kit-actions-interactions` — actors, action areas, pickups, doors/levers, dialogs, purchases, trigger zones.
- `action-adventure-kit-health-stats-damage` — resources (health), attributes and stats, damage senders/receivers.
- `action-adventure-kit-status-effects` — effect pools and effect types (slow, poison, buffs).
- `action-adventure-kit-save-load` — persistence container, persisters, keys, save data inspection.

## Prerequisites

- Action Adventure Kit 1.10.1 or later, imported from the Asset Store.
- Unity 6000.0.76 or later (the minimum recommended version since AAK 1.10.0).
- Packages AAK code depends on (all referenced by the `AdventureCore` assembly): Input System, Cinemachine 3 (the `CINEMACHINEV3` define is set automatically by the asmdef when `com.unity.cinemachine` is present), AI Navigation (NavMeshSurface), Timeline, Visual Scripting, TextMeshPro/uGUI.
- Render pipeline: the demos and manual projects use URP out of the box, but AAK has no hard dependency on URP.

**Install check (run before anything else):**

1. Confirm the folder `Assets/SoftLeitner/AdventureCore` exists and contains `AdventureCore.asmdef`.
2. Confirm the type `AdventureCore.CharacterBase` resolves. From an Editor script:

```csharp
var t = System.Type.GetType("AdventureCore.CharacterBase, AdventureCore");
UnityEngine.Debug.Log(t != null ? "AAK installed" : "AAK missing");
```

3. Optional modules — check which of these folders exist, they are imported separately from the package dialog:

| Folder | Content |
|---|---|
| `Assets/SoftLeitner/AdventureCore` | The framework (required by everything else) |
| `Assets/SoftLeitner/AdventureManual` | Getting Started and system sample scenes |
| `Assets/SoftLeitner/AdventureHero` | Hero demo (3D, top-down adventure) |
| `Assets/SoftLeitner/AdventureSouls` | Souls demo (3D third-person action) |
| `Assets/SoftLeitner/AdventureIsle` | Isle demo (2D, mobile, on-screen controls) |
| `Assets/SoftLeitner/AdventureExtras` | Arena and cross-demo extras; needs Hero and Souls |

If the check fails, tell the user: "Action Adventure Kit is not installed in this project. Get it from https://assetstore.unity.com/packages/templates/systems/action-adventure-kit-217284 and import at least AdventureCore." Do not continue with other AAK skills.

## Quick start

Goal: verify the install and open a working example.

1. Run the install check above.
2. Open the finished Getting Started scene: `Assets/SoftLeitner/AdventureManual/GettingStarted/GSFinished.unity` (`File > Open Scene`, or `UnityEditor.SceneManagement.EditorSceneManager.OpenScene(path)`).
3. Enter Play mode. Move with WASD, press the bound Action key near cubes to pick them up, walk into the coloured areas to see slow/health/vitality effects.
4. Expected result: the Hierarchy contains a `Character` object with `GenericCharacter`, `CharacterController`, `CharacterControllerMovement` and `PlayerInput` components; the Console shows no errors; on-screen text reflects inventory, effects, resources and attributes.

## Workflows

### Workflow: Complete first-time setup after import

**Goal:** Get a freshly imported project into a working state.

**Steps:**
1. On first import, AAK shows a "Welcome to Action Adventure Kit" dialog (`AdventureCore/Utilities/AdventureSetup.cs`). It recommends creating the project from a URP template of an LTS version and offers to restart the editor. Restarting avoids import issues; accept it if the user agrees.
2. The dialog only appears once. It is tracked by `Assets/SoftLeitner/AdventureCore/Utilities/AdventureSetupState.asset` (`HasRun`). To suppress it in a project permanently, create an `AdventureSetupState` asset at `Assets/SkipSetup.asset` with `HasRun` checked.
3. Make sure the packages listed in Prerequisites are installed (`Window > Package Manager`). Missing Input System or Cinemachine produces compile errors in `AdventureCore`.
4. If the user wants the Hero demo, add the Visual Scripting application variable described in the next workflow.

**Expected result:** The project compiles with no errors and the Getting Started scenes play.

### Workflow: Prepare the Hero demo (SaveSlot variable)

**Goal:** Run the Hero demo without missing-variable errors.

**Steps:**
1. Open `Assets/SoftLeitner/AdventureHero/Scenes/HeroTitle.unity`.
2. Select the `StateMachine` object under `Logic`, click `Edit Graph`, and open the `App` tab of the Visual Scripting Variables panel.
3. Add an application variable named `SaveSlot` of type `Integer` with value `-1`.
4. Add the Hero scenes to the build profile scene list (`File > Build Profiles`): `HeroTitle`, `HeroBeach`, `HeroInland`, `HeroCanyon`, `HeroTemple`, `HeroGlade` in `Assets/SoftLeitner/AdventureHero/Scenes/`. The demos switch scenes at runtime, which requires scenes to be in the build list.
5. Play from `HeroTitle`.

**Why:** `PersistenceContainer.IndexVariableName` on `HeroSetup.prefab` is set to `SaveSlot`. If the variable is missing the container logs `Missing Visual Scripting Application Variable 'SaveSlot'` and save slots do not work.

**Expected result:** The title screen loads, a new game starts in HeroBeach, and the Console has no `SaveSlot` warning.

### Workflow: Open the other demos

**Goal:** Find and play the Souls, Isle or Arena demos.

**Steps:**
1. Souls: start from `Assets/SoftLeitner/AdventureSouls/Scenes/Title/SoulsTitle.unity`. The game manager is `AdventureSouls.SoulsCommons` on `SoulsSetup.prefab`. Add `SoulsTitle`, `SoulsLoading`, `SoulsDungeon`, `SoulsShrine` to the build scene list.
2. Isle (2D): start from `Assets/SoftLeitner/AdventureIsle/Scenes/IsleTitle.unity`; add `IsleTitle`, `IsleOverworld`, `IsleCave` to the build list.
3. Arena: start from `Assets/SoftLeitner/AdventureExtras/Arena/Scenes/ArenaTitle.unity`; add `ArenaTitle`, `ArenaShop` and `Scenes/Stages/ArenaStage1-4`.
4. Debugging scenes (e.g. `AdventureSouls/Scenes/Debugging/SoulsDebuggingGeneral.unity`, `AdventureHero/Scenes/Debugging/*`) start directly in gameplay and are useful for testing a single system.

**Expected result:** The title scene plays and transitions to gameplay scenes without "scene couldn't be loaded" errors.

### Workflow: Use AAK without URP (Built-in pipeline)

**Goal:** Run AAK on the Built-in render pipeline.

**Steps:**
1. Import the asset into the project.
2. Clear the render pipeline asset in `Edit > Project Settings > Graphics` and in every quality level under `Edit > Project Settings > Quality`.
3. Replace URP materials with Built-in equivalents (for example `Universal Render Pipeline/Lit` → `Standard`).
4. Remove the Universal RP package via `Window > Package Manager`.

**Expected result:** No pink materials in the demo scenes; the project compiles (AAK code does not reference URP).

### Workflow: Learn a system from its sample scene

**Goal:** Show the user a minimal, working reference for a specific system.

**Steps:** open the matching scene and inspect the components on the objects named in the table.

| Scene | Demonstrates |
|---|---|
| `AdventureManual/GettingStarted/GSFinished.unity` | Every core system wired together (8 steps) |
| `AdventureManual/GettingStarted/GSStepByStep.unity` | Same, with each step's objects deactivated so you can enable them one at a time |
| `AdventureManual/GettingStarted/GSBlank.unity` | Environment only, for following the guide from scratch |
| `AdventureManual/Systems/Item/ManualItem.unity`, `ManualItemTiled.unity`, `ManualItemValues.unity` | Listed inventory, tiled inventory, item values |
| `AdventureManual/Systems/Resource/ManualResource.unity` | Resources |
| `AdventureManual/Systems/Trigger/ManualTrigger.unity` | Trigger areas and items |

**Expected result:** The user sees a working configuration they can copy.

## Verification

- `System.Type.GetType("AdventureCore.CharacterBase, AdventureCore") != null`.
- `Assets/SoftLeitner/AdventureCore/AdventureCore.asmdef` exists.
- Opening `GSFinished.unity` and entering Play mode produces no Console errors.
- For the Hero demo: `UnityEditor.AssetDatabase.LoadAssetAtPath<UnityEngine.GameObject>("Assets/SoftLeitner/AdventureHero/Prefabs/HeroSetup.prefab")` is not null and the `SaveSlot` application variable exists (`Unity.VisualScripting.Variables.Application.IsDefined("SaveSlot")` in Play mode).

## API quick reference

| Entry point | Type | What it does |
|---|---|---|
| `AdventureCore.CharacterBase` | MonoBehaviour | Base of every character; hub for actor, movement, inventory and pools |
| `AdventureCore.PersistenceContainer` | MonoBehaviour | Scene-level save/load manager (see save-load skill) |
| `AdventureCore.StateManager` | MonoBehaviour | Game state machine (PLAY, MENU, CUTSCENE in the demos) driven by events |
| `AdventureCore.AdventureSetupState` | ScriptableObject (internal) | Remembers whether the welcome dialog has run |
| `Assets > Create > Asset from ScriptableObject` | Menu item | Creates an asset from a selected ScriptableObject script |
| `Window > ActionAdventure > SaveData(PlayerPref)` | Menu item | Inspect/edit saved data in PlayerPrefs |
| `Assets > Create > Adventure/...` | Menu | Create AAK data assets (items, attributes, resources, effects, damages, persistence areas) |

Full API reference: https://adventureapi.softleitner.com/ — every component also has a `HelpURL`, so the `?` icon in its Inspector header opens its API page.

## Common issues

- **Symptom:** Compile errors mentioning `Cinemachine`, `InputSystem` or `NavMeshSurface`. **Cause:** Required package missing or too old. **Fix:** Install Cinemachine 3.x, Input System and AI Navigation 2.x via the Package Manager.
- **Symptom:** `Missing Visual Scripting Application Variable 'SaveSlot'` warning, Hero save slots don't work. **Cause:** The application variable was not added. **Fix:** Follow "Prepare the Hero demo".
- **Symptom:** Pink/magenta materials. **Cause:** Project is on Built-in (or a different pipeline) while demo materials use URP shaders. **Fix:** Switch to URP or convert materials as in "Use AAK without URP".
- **Symptom:** "Scene ... couldn't be loaded because it has not been added to the build settings" when starting a demo. **Cause:** Demo scenes not in the build list. **Fix:** Add the scenes listed for that demo.
- **Symptom:** Odd import errors right after importing. **Cause:** First import before all packages compiled. **Fix:** Restart the editor (the welcome dialog offers this).
- **Symptom:** Arena or HeroSouls content fails to compile. **Cause:** `AdventureExtras` references both Hero and Souls. **Fix:** Import AdventureHero and AdventureSouls, or remove AdventureExtras.

## Boundaries

- AAK is a framework plus demos, not a no-code game generator: gameplay is assembled from components in the Inspector, UnityEvents, Visual Scripting graphs or C#.
- No networking/multiplayer layer.
- The demo games (Hero, Souls, Isle, Arena) are examples to learn from and adapt; their game-specific classes (`AdventureHero.*`, `AdventureSouls.*`, `AdventureIsle.*`) are not a stable general-purpose API.
- Camera rigs rely on Cinemachine; for non-Cinemachine cameras use native Unity camera scripting.
- For detailed topics not covered by the AAK skills (Timeline tracks, lock-on camera, playable animations, demo scene walkthroughs) read the manual at https://adventure.softleitner.com/manual.
