---
name: action-adventure-kit-save-load
description: "Use this skill whenever the user wants to save or load game state with Action Adventure Kit — e.g. 'make my game save', 'remember player position and inventory', 'collected pickups should stay gone after reload', 'door should stay open after loading', 'add save slots', 'reset my save data', 'where is the save stored', 'duplicate key warning', 'progress isnt saved'. Covers PersistenceContainer, PlayerPrefSaver, PersistenceArea assets, ManualPersister and the components that take a persister, pickup persistence keys, Generate Missing Keys and Check Keys, save slots, persistence triggers, and the Window > ActionAdventure > SaveData(PlayerPref) editor. Do NOT use for setting up the systems being saved (see action-adventure-kit-items-inventory, action-adventure-kit-health-stats-damage, action-adventure-kit-character-movement). When in doubt whether a save, load or persistence request could involve Action Adventure Kit, use this skill — Prerequisites shows how to confirm the asset is installed."
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

# Save and Load Game State with Action Adventure Kit

AAK persistence is key/value based. A scene-level `PersistenceContainer` holds data per `PersistenceArea` (a ScriptableObject representing one save unit), loads it on `Awake` and saves changes automatically through a saver such as `PlayerPrefSaver`. Components that should be saved get a **persister** (`ManualPersister`) with a unique key; each component stores its data under that key plus its own sub-key (`MOV` movement, `INV` inventory, `RES` resources, `ATT` attributes, `EFF` effects). This skill lets an agent add saving to a scene, wire persisters, generate keys, use save slots, and inspect or reset saved data.

## When to use this skill

- "make the game save automatically"
- "player position, health and inventory should be restored after restarting"
- "pickups i already collected shouldn't respawn"
- "the gate should stay open once opened"
- "add 3 save slots"
- "how do i wipe my save while testing"
- "console says Missing Key / Duplicate Key"
- "nothing is saved"

Not for:
- Building the character, inventory, pools or actions themselves → the matching AAK skill (see `action-adventure-kit-setup-and-overview` for the index)
- Cloud saves or platform save APIs → write a custom `PersistenceSaverBase` (see Boundaries)

## Prerequisites

- Action Adventure Kit 1.10.1+ (`AdventureCore`), Unity 6000.0.76+. `PersistenceContainer` reads save slot indices from Visual Scripting application variables, so the Visual Scripting package must be present (it is an AAK dependency).
- The objects to save already exist (character, inventory, pools, actions).
- **Install check:** `System.Type.GetType("AdventureCore.PersistenceContainer, AdventureCore") != null` and the menu `Window > ActionAdventure > SaveData(PlayerPref)` exists. If not, tell the user to import Action Adventure Kit from https://assetstore.unity.com/packages/templates/systems/action-adventure-kit-217284 and stop.

## Quick start

Add saving to a scene with a character (Getting Started step 8). Run from an Editor script:

```csharp
using AdventureCore;
using UnityEditor;
using UnityEditor.SceneManagement;
using UnityEngine;

public static class AakAddPersistence
{
    public static void Run(GenericCharacter character, string areaPath = "Assets/Game/Persistence.asset")
    {
        var folder = System.IO.Path.GetDirectoryName(areaPath).Replace('\\', '/');
        if (!AssetDatabase.IsValidFolder(folder))
        {
            System.IO.Directory.CreateDirectory(folder);
            AssetDatabase.Refresh();
        }

        var area = ScriptableObject.CreateInstance<PersistenceArea>();
        area.Key = "A";                        // unique per area
        AssetDatabase.CreateAsset(area, areaPath);
        AssetDatabase.SaveAssets();

        // container + saver
        var go = new GameObject("Persistence");
        var container = go.AddComponent<PersistenceContainer>();
        container.Key = "GAME";                // prefix of every saved key
        container.Areas = new[] { area };
        container.Saver = go.AddComponent<PlayerPrefSaver>();
        // AutoLoad and AutoSave default to true

        // one persister for the character, shared by its components via sub-keys
        var persister = character.gameObject.AddComponent<ManualPersister>();
        persister.Area = area;
        persister.Key = GUID.Generate().ToString();
        character.Persister = persister;
        if (character.Movement is MovementBasePersisted m) m.Persister = persister;
        if (character.Inventory) character.Inventory.Persister = persister;
        if (character.ResourcePool) character.ResourcePool.Persister = persister;
        if (character.AttributePool) character.AttributePool.Persister = persister;
        if (character.EffectPool) character.EffectPool.Persister = persister;

        EditorUtility.SetDirty(character);
        EditorSceneManager.MarkSceneDirty(character.gameObject.scene);
    }
}
```

1. Run `AakAddPersistence.Run(character)`.
2. Enter Play mode, move the character / pick something up, exit Play mode, enter Play mode again.

Expected result: the scene has a `Persistence` object with `PersistenceContainer` (Key `GAME`, one area) and `PlayerPrefSaver`; the character has a `ManualPersister` with a GUID key that is referenced by the character, movement, inventory and pools. On the second Play the character starts where it was and the inventory/pool values are restored. The data is stored in PlayerPrefs under `GAME_0_A`.

## Workflows

### Workflow: Set up the container, saver and areas

**Goal:** Configure where and how data is saved.

**Steps:**
1. Create areas with `Assets > Create > Adventure > PersistenceArea`: unique `Key`; `IsGlobal` for data shared across save slots (settings); leave it off for per-slot game data. Typical split: one area per level/scene plus one global area.
2. Add one `PersistenceContainer` per scene (it sets the static `PersistenceContainer.Instance` and runs early: script execution order -500). Fields:
   - `Key` — prefix for saved keys (game identifier).
   - `Index` — save slot; `IndexVariableName` reads it from a Visual Scripting application variable (the Hero demo uses `SaveSlot`).
   - `Saver` — `PlayerPrefSaver` (built in). Leave empty to disable loading/saving, for example in debug scenes.
   - `Areas` — every area used in this scene. **Only listed areas are loaded and saved.**
   - `AutoLoad` (load in Awake, default on), `AutoSave` (save once at end of frame after changes, default on), `Buffer` (carry data between scenes in memory).
3. `PlayerPrefSaver.Debug` stores data in memory only, so every Play session starts fresh; `DebugData` supplies predefined data in that mode.
4. Saved keys are `{Container.Key}_{Index}_{Area.Key}` for local areas and `{Container.Key}_{Area.Key}` for global areas.

**Expected result:** After a change in Play mode, `PlayerPrefs.HasKey("GAME_0_A")` is true.

### Workflow: Persist characters and their systems

**Goal:** Save position, inventory, health, attributes and effects.

**Steps:**
1. Add one `ManualPersister` per character (`Area`, unique `Key`, optional `CustomContainer` if a scene has several containers).
2. Assign it to every `Persister` field on that character: `CharacterBase.Persister` (also identifies the character), the movement (`MovementBasePersisted.Persister`, saved every `Interval` = 1 s; `CameraPivot` also saves look direction), `InventoryBase.Persister` (saved on change, or manually with `PersistManual`), `ResourcePool.Persister` (`PersistenceInterval` 1 s), `AttributePool.Persister` (on change), `EffectPool.Persister` (`PersistenceInterval` 1 s). They don't collide because each adds its own sub-key.
3. Inventories also need their `ItemSet` and effect pools their `EffectSet`, so saved keys can be turned back into assets.

**Expected result:** After reload the character's position, items, resource values, attributes and active effects match the saved state.

### Workflow: Persist world objects (pickups, gates, crates)

**Goal:** World changes survive reloads.

**Steps:**
1. `PickupAction` has its own fields: set `PersistenceArea`; leave `PersistenceKey` empty and generate it (next workflow). Collected pickups are destroyed on load.
2. `ObjectAction` (gates, chests): add a `ManualPersister` and assign it to `ObjectAction.Persister`; set `ObjectStateName` so the object jumps to its used state on load.
3. Destructibles: `DestructionPersister` saves that its GameObject was destroyed and destroys it again on load.
4. Custom flags: `ManualPersister.SetBool/SetInt/SetFloat` from UnityEvents, or `PersistenceContainer.Instance.Set(area, key, value)` / `Get<T>(area, key, default)` from code.
5. React to saved flags: `BoolPersistenceTrigger`/`IntPersistenceTrigger`/`FloatPersistenceTrigger` (`Area`, `Key`, `Observe`, `Changed`, `ChangedInverted`, optional `Compare` + `ComparisonValue` → `ComparisonChanged`), e.g. to open a door when a lever flag is set.
6. Extras: `PlaytimePersister` (play time, for save-slot screens) and `RandomStatePersister` (keeps random loot consistent between sessions).

**Expected result:** Collected pickups stay gone, opened gates stay open, destroyed crates stay destroyed.

### Workflow: Generate and check keys

**Goal:** Every persister has a unique key.

**Steps:**
1. Select the `PersistenceContainer` in the open scene. Under `----- DEBUG -----` in its Inspector click **Generate Missing Keys**: every component implementing `IPersister` with an empty key gets a new GUID (logged as `Generated Keys for:` + names).
2. Click **Check Keys** to log `Missing Key <name>` and `Duplicate Key:<key>` problems.
3. From code, the equivalent of Generate Missing Keys is:

```csharp
foreach (var b in Object.FindObjectsByType<MonoBehaviour>(FindObjectsSortMode.None))
    if (b is AdventureCore.IPersister p && string.IsNullOrWhiteSpace(p.PersistenceKey))
    {
        p.PersistenceKey = UnityEditor.GUID.Generate().ToString();
        UnityEditor.EditorUtility.SetDirty(b);
    }
```

4. Duplicated objects (Ctrl+D) copy keys. After duplicating persisted objects, clear the copies' keys and generate again.

**Expected result:** Check Keys logs nothing; the scene is saved with the new keys.

### Workflow: Inspect, edit and reset saves

**Goal:** Debug persistence.

**Steps:**
1. Container Inspector buttons: **Save** and **Reload Scene** (Play mode only), **Delete Data** (all areas of the current index), **Delete Local Data**, **Delete Global Data**, **Edit Save Data** (pick an area, opens the PlayerPref editor), **Export** (writes one JSON file per area).
2. `Window > ActionAdventure > SaveData(PlayerPref)` opens the PlayerPref save editor: enter container key, index and area, then **Get**/**Set**/**Delete**; add/remove entries with **+**/**-**.
3. In Play mode, a single selected `ManualPersister` shows a **Clear** button that removes its own key.
4. From code: `PersistenceContainer.Instance.DeleteData()` / `DeleteData(index, global)`, `Save()`, `Load()`, `Clear(area)`.

**Expected result:** After Delete Data, the next Play starts from the scene's authored state.

### Workflow: Save slots

**Goal:** Multiple independent saves.

**Steps:**
1. Before the gameplay scene loads, choose the slot: set `PersistenceContainer.Index`, or set the Visual Scripting application variable named in `IndexVariableName` (e.g. `Unity.VisualScripting.Variables.Application.Set("SaveSlot", 2)`).
2. Local areas are suffixed with the index; global areas are shared.
3. Delete one slot with `container.DeleteData(index)`.

**Expected result:** Slots 0, 1, 2 store separate `GAME_<index>_<area>` keys.

## Verification

- Exactly one `PersistenceContainer` in the scene (or persisters use `CustomContainer`), with `Saver` set and `Areas` containing every area referenced by persisters in the scene.
- Every `IPersister` in the scene has a non-empty, unique `PersistenceKey` (run the loop above in check mode, or Check Keys logs nothing).
- Character components reference the persister: `character.Persister`, `((MovementBasePersisted)character.MovementBase).Persister`, `character.InventoryBase.Persister`, pools' `Persister`.
- Play-mode round trip: change state → `PersistenceContainer.Instance.Save()` → `PlayerPrefs.HasKey(container.GetKey(area))` is true → reload the scene → state restored.
- Console has no `Missing Visual Scripting Application Variable` warning when `IndexVariableName` is used.

## API quick reference

| Entry point | Type | What it does |
|---|---|---|
| `AdventureCore.PersistenceContainer` | MonoBehaviour | `Instance`, `Get<T>`, `Set<T>`, `Save`, `Load`, `Clear`, `DeleteData`, `GetKey(area)` |
| `AdventureCore.PlayerPrefSaver` | PersistenceSaverBase | Saves JSON to PlayerPrefs; `Debug` for in-memory |
| `AdventureCore.PersistenceSaverBase` | Abstract MonoBehaviour | Base for custom savers (`Save`, `Load`, `Delete`, `Export`, `Import`) |
| `AdventureCore.PersistenceArea` | ScriptableObject | Save unit (`Key`, `IsGlobal`) |
| `AdventureCore.ManualPersister` | PersisterBase | Key/area for a component; `SetBool/SetInt/SetFloat`, `Get<T>`, `Clear` |
| `AdventureCore.DestructionPersister`, `PlaytimePersister`, `RandomStatePersister` | ManualPersister | Specialised persisters |
| `AdventureCore.BoolPersistenceTrigger` (+ Int, Float) | MonoBehaviour | UnityEvents from saved values |
| `AdventureCore.IPersister` | Interface | Anything with `PersistenceKey`/`PersistenceArea` (used by Generate Missing Keys) |
| `Window > ActionAdventure > SaveData(PlayerPref)` | Editor window | View/edit saved PlayerPrefs data |

## Common issues

- **Symptom:** Nothing is saved. **Cause:** `Saver` empty, area not listed in `Areas`, `AutoSave` off, or persister fields not assigned. **Fix:** Check the container and component `Persister` fields.
- **Symptom:** State from a previous test keeps coming back. **Cause:** Old save in PlayerPrefs. **Fix:** Delete Data on the container, or enable `PlayerPrefSaver.Debug` while iterating.
- **Symptom:** Two objects share state (e.g. collecting one pickup removes another). **Cause:** Duplicate keys from duplicated GameObjects. **Fix:** Clear the copies' keys, Generate Missing Keys, confirm with Check Keys.
- **Symptom:** Pickups never persist. **Cause:** `PickupAction.PersistenceArea` empty (disables persistence for that pickup) or empty `PersistenceKey`. **Fix:** Set the area and generate keys.
- **Symptom:** Items missing after load. **Cause:** Inventory `ItemSet` missing the item, or empty item `Key`. **Fix:** See the items skill.
- **Symptom:** `Missing Visual Scripting Application Variable 'SaveSlot'`. **Cause:** `IndexVariableName` set but the variable isn't defined. **Fix:** Add the application variable, or clear `IndexVariableName`.
- **Symptom:** Character position isn't restored. **Cause:** The movement's `Persister` not assigned, or a `SuspendMovementPersistenceInstruction` is active. **Fix:** Assign the persister; check active instructions.

## Boundaries

- Only `PlayerPrefSaver` is built in. File, cloud or console storage requires a custom `PersistenceSaverBase` subclass implementing `Save`, `Load`, `Delete` (and optionally `Export`/`Import`).
- No save-data encryption, versioning or migration; data is JSON key/value pairs.
- Only objects with persisters are saved; arbitrary scene state (transforms of unrelated objects, physics) isn't captured automatically.
- Save-slot selection UI is demo specific (Hero/Souls title scenes); build your own with `Index` and `DeleteData(index)`.
