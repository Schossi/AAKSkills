---
name: action-adventure-kit-status-effects
description: "Use this skill whenever the user wants status effects, buffs, or debuffs in Action Adventure Kit — e.g. 'slow the player in the swamp and show a slowed icon', 'poison that damages over time', 'speed boost for 10 seconds', 'strength buff potion', 'show active effects on screen', 'effect keeps stacking', 'effect never goes away'. Covers EffectType assets and effect prefabs (GenericEffect, TimeoutEffect, StatEffect, AttributeEffect, DamageEffect), EffectSet, EffectPool, adding and removing effects from zones, items, actions and code, external effects, stacking, build-up via EffectResourceValue, and effect UI. Do NOT use for instant damage or healing (see action-adventure-kit-health-stats-damage), item definitions (see action-adventure-kit-items-inventory), or saving effects (see action-adventure-kit-save-load). When in doubt whether a buff, debuff or status request could involve Action Adventure Kit, use this skill — Prerequisites shows how to confirm the asset is installed."
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

# Apply Status Effects with Action Adventure Kit

Effects are temporary behaviours on a character — slowed, poisoned, buffed. An `EffectType` asset identifies the effect and points to a prefab with an `EffectBase` component; adding the effect to a character's `EffectPool` instantiates that prefab under the pool, and removing it destroys the instance. Effects can apply character instructions, modify attributes or stats, deal damage over time, or time out on their own. This skill lets an agent create effect types, give characters an effect pool, and add/remove effects from zones, items, actions or code.

## When to use this skill

- "player should be slowed in the swamp and the ui should say Slowed"
- "poison that does 1 damage every second"
- "speed boost pickup that lasts 10 seconds"
- "strength potion: +5 strength for a minute"
- "show the list of active effects"
- "the effect stacks every time i enter the area"
- "effect is still there after leaving the zone"

Not for:
- One-off damage or healing, damage zones → `action-adventure-kit-health-stats-damage`
- Creating the potion/equipment that grants the effect → `action-adventure-kit-items-inventory`
- Trigger zone basics → `action-adventure-kit-actions-interactions`
- Saving active effects → `action-adventure-kit-save-load`

## Prerequisites

- Action Adventure Kit 1.10.1+ (`AdventureCore`), Unity 6000.0.76+.
- A character with `GenericCharacter`. Stat/attribute effects need an `AttributePool`; damage effects send damage directly to the character (no `TriggerDamageReceiver` needed), so a `ResourceDamage` effect needs a `ResourcePool` containing that resource (see the health skill).
- **Install check:** `System.Type.GetType("AdventureCore.EffectPool, AdventureCore") != null` and the menu `Assets > Create > Adventure > EffectType` exists. If not, tell the user to import Action Adventure Kit from https://assetstore.unity.com/packages/templates/systems/action-adventure-kit-217284 and stop.

## Quick start

Create a 5-second "Slow" effect (speed × 0.5), give the character an effect pool, and add the effect from code. Run from an Editor script:

```csharp
using AdventureCore;
using UnityEditor;
using UnityEditor.SceneManagement;
using UnityEngine;

public static class AakCreateSlowEffect
{
    public static EffectType Run(GenericCharacter character, string folder = "Assets/Game/Effects")
    {
        if (!AssetDatabase.IsValidFolder(folder))
        {
            System.IO.Directory.CreateDirectory(folder);
            AssetDatabase.Refresh();
        }

        // 1. effect type asset
        var type = ScriptableObject.CreateInstance<EffectType>();
        type.Key = "SLO"; type.Name = "Slow"; type.IsSingleton = true;
        AssetDatabase.CreateAsset(type, folder + "/Slow.asset");

        // 2. effect prefab pointing back to the type
        var temp = new GameObject("Slow");
        var effect = temp.AddComponent<TimeoutEffect>();
        effect.Type = type;
        effect.Duration = 5f;
        effect.Instructions = new CharacterInstructionBase[]
        {
            new MovementSpeedMultiplierInstruction { Value = 0.5f }
        };
        var prefab = PrefabUtility.SaveAsPrefabAsset(temp, folder + "/Slow.prefab");
        Object.DestroyImmediate(temp);

        type.Prefab = prefab.GetComponent<TimeoutEffect>();
        EditorUtility.SetDirty(type);

        // 3. effect set (needed for persistence / adding by key)
        var set = ScriptableObject.CreateInstance<EffectSet>();
        set.Effects = new[] { type };
        AssetDatabase.CreateAsset(set, folder + "/Effects.asset");
        AssetDatabase.SaveAssets();

        // 4. pool on the character
        var status = character.transform.Find("Status");
        if (!status)
        {
            status = new GameObject("Status").transform;
            status.SetParent(character.transform, false);
        }
        var pool = status.gameObject.AddComponent<EffectPool>();
        pool.Character = character;
        pool.Effects = set;
        character.EffectPool = pool;

        EditorUtility.SetDirty(character);
        EditorSceneManager.MarkSceneDirty(character.gameObject.scene);
        return type;
    }
}
```

1. Run `AakCreateSlowEffect.Run(character)`.
2. In Play mode call `character.EffectPool.Add(slowType)` (or `slowType.Add(character)`).

Expected result: `Slow.asset`, `Slow.prefab` and `Effects.asset` exist; the character has `Status` with an `EffectPool`. In Play mode a `Slow(Clone)` child appears under `Status`, `character.MovementBase.SpeedMultiplier` is `0.5`, and after 5 seconds the instance is destroyed and the multiplier is back to `1`.

## Workflows

### Workflow: Create an effect type

**Goal:** Define a new effect.

**Steps:**
1. `Assets > Create > Adventure > EffectType`: set a unique `Key`, `Name`, `Image` (UI icon), `IsSingleton` (default true: adding again reuses the existing instance instead of creating a second one).
2. Create a prefab with one effect component and set its `Type` back to the asset:

| Component | Behaviour | Key fields |
|---|---|---|
| `GenericEffect` | Applies instructions and messages while active; lasts until removed | `Instructions`, `StartMessage`, `EndMessage` |
| `TimeoutEffect` | GenericEffect that removes itself after `Duration` seconds; re-adding a singleton resets the timer | `Duration` |
| `StatEffect` | Adds `Value` to a stat while active | `Stat`, `Value`, `Priority` |
| `AttributeEffect` | Adds `Value` to an attribute while active | `Attribute`, `Value`, `Priority` |
| `DamageEffect` | Sends `Damages` to the character every `Interval` seconds; lasts until removed | `Interval`, `Damages` |

3. Assign the prefab's component to `EffectType.Prefab`. The prefab can also contain visuals (particles), since it's instantiated under the character's pool.
4. Register the type in an `EffectSet` (`Assets > Create > Adventure > EffectSet`; **Find in Folder** / **Find in Resources** buttons fill it) and assign the set to every `EffectPool.Effects`.

**Expected result:** `EffectType.Prefab != null`, `EffectType.Prefab.Type == type`, and the type is in the set.

### Workflow: Give a character an effect pool

**Goal:** Let a character receive effects.

**Steps:**
1. Add `AdventureCore.EffectPool` (conventionally on a `Status` child). Set `Character` (required: effects read the character through the pool), `Effects` (the `EffectSet`), optional `Persister`.
2. Assign it to `CharacterBase.EffectPool`.
3. Events: `Added`/`Removed` (`UnityEvent<EffectBase>`), `Changed`.

**Expected result:** `character.EffectPool == pool` and `pool.Character == character`.

### Workflow: Apply effects from zones, items and actions

**Goal:** Trigger effects from gameplay (Getting Started step 3).

**Steps:**
1. Zone: on a `GenericTriggerArea` (see the interactions skill), add persistent listeners: `CharacterAdded` → `EffectType.AddExternal(CharacterBase)` and `CharacterRemoved` → `EffectType.Remove(CharacterBase)`, with the effect asset as the target. Use `GenericEffect` (no timeout) so the effect lasts exactly as long as the character is inside. From code:

```csharp
// the entering/leaving character is forwarded as the argument
UnityEditor.Events.UnityEventTools.AddPersistentListener(area.CharacterAdded,
    new UnityEngine.Events.UnityAction<AdventureCore.CharacterBase>(slowType.AddExternal));
UnityEditor.Events.UnityEventTools.AddPersistentListener(area.CharacterRemoved,
    new UnityEngine.Events.UnityAction<AdventureCore.CharacterBase>(slowType.Remove));
UnityEditor.EditorUtility.SetDirty(area);
```
2. Items: `UsableItem.Effects` (added once per use) and `EquipmentItem.Effects` (active while equipped).
3. Actions: bind an action's `Starting`/`Ending` events to `EffectType.Add(CharacterActionBase)` / `Remove(CharacterActionBase)`.
4. Build-up (e.g. poison meter): use an `EffectResourceValue` in the character's resource pool with `EffectPool` + `EffectType`; the effect is added when the value reaches its maximum and removed when it drains back to 0.
5. Code: `pool.Add(type, isExternal)`, `pool.Remove(type)`, `pool.Add("SLO")` (by key via the set), `pool.HasEffect(type)`, `pool.GetEffect(type)`, `pool.ClearEffects()`.

**External vs internal:** `AddExternal` marks the effect as owned by an outside source (a zone, equipment). `ClearEffects()` removes only internal effects, so a respawn/cleanse does not strip effects that the zone or equipment will remove itself.

**Expected result:** Entering the zone adds one instance; leaving removes it; `ClearEffects()` leaves external effects in place.

### Workflow: Show active effects

**Goal:** Display effects in UI.

**Steps:**
1. Debug text: `AdventureCore.EffectPoolText` on a uGUI `Text` or TextMeshPro object; set `EffectPool` and `Text`/`TMPText`.
2. Icons/bars: `EffectBarPanel` (uGUI) or `EffectBarElement` (UI Toolkit) show the effect `Image`s; copy a configured one from a demo UI.

**Expected result:** The text lists the names of active effects and updates on add/remove.

## Verification

- `character.EffectPool != null`, `pool.Character == character`, `pool.Effects` contains every effect type used.
- For each `EffectType`: non-empty unique `Key`, `Prefab != null`, `Prefab.Type == type`.
- Play mode after adding: `pool.HasEffect(type) == true` and an instance exists as a child of the pool's GameObject; after removal/timeout `HasEffect` is false and the child is gone.
- Instruction effects: the instruction's effect is observable (e.g. `MovementBase.SpeedMultiplier == 0.5f` while slowed).
- Console has no `NullReferenceException` from `GenericEffect.Start` (pool `Character` missing) or `EffectPool.Add` (null `Prefab`).

## API quick reference

| Entry point | Type | What it does |
|---|---|---|
| `AdventureCore.EffectType` | ScriptableObject | Effect identity; `Add(character)`, `AddExternal(character)`, `Remove(character)` |
| `AdventureCore.EffectSet` | ScriptableObject | All effect types; `GetEffect(key)` |
| `AdventureCore.EffectPool` | MonoBehaviour | `Add(type, isExternal)`, `Add(key)`, `Remove(type)`, `HasEffect`, `GetEffect`, `ClearEffects`, `CurrentEffects` |
| `AdventureCore.GenericEffect` / `TimeoutEffect` | EffectBase | Instruction/message effects, optionally timed |
| `AdventureCore.StatEffect` / `AttributeEffect` | EffectBase | Modify a stat/attribute while active |
| `AdventureCore.DamageEffect` | EffectBase | Damage over time |
| `AdventureCore.EffectBase.Terminate()` | Method | Remove this effect from its pool |
| `AdventureCore.EffectResourceValue` | ResourceValue | Adds an effect when the value fills up |
| `AdventureCore.EffectPoolText`, `EffectBarPanel`, `EffectBarElement` | MonoBehaviour | UI |

## Common issues

- **Symptom:** `NullReferenceException` when an effect starts. **Cause:** `EffectPool.Character` not set. **Fix:** Assign the character on the pool.
- **Symptom:** `EffectPool.Add` throws. **Cause:** `EffectType.Prefab` empty, or the prefab's `Type` doesn't point to the type. **Fix:** Wire both directions.
- **Symptom:** Effect stacks (multiple instances). **Cause:** `IsSingleton` unchecked. **Fix:** Check it (re-adding then refreshes a `TimeoutEffect`'s timer).
- **Symptom:** Poison never ends. **Cause:** `DamageEffect` has no duration. **Fix:** Remove it explicitly (zone exit, cure item, timer in your code), or derive a custom effect combining interval damage and a timeout.
- **Symptom:** Zone effect remains after leaving. **Cause:** Only `CharacterAdded` was wired, or a `TimeoutEffect` was used with an add-only setup. **Fix:** Wire `CharacterRemoved` → `EffectType.Remove`.
- **Symptom:** Effects missing after loading, or "add by key" returns nothing. **Cause:** Type not in the pool's `EffectSet`, or duplicate/empty keys. **Fix:** Add it to the set with a unique key.
- **Symptom:** Stat/attribute effect has no visible result. **Cause:** Character has no `AttributePool`, or the stat/attribute isn't part of it. **Fix:** See the health skill.

## Boundaries

- No built-in effect immunity/resistance tables; implement them with damage modifiers, `CanStart` checks or a custom `EffectPool` subclass.
- Effects are per-character; area-wide weather or global modifiers need your own systems.
- Custom effect logic requires a C# subclass of `EffectBase` (override `Add`, `GetValue`/`SetValue` for persistence). This includes timed stat/attribute buffs: `StatEffect` and `AttributeEffect` have no duration, so subclass one and add a timer that calls `Terminate()` and resets in `Add`, as `TimeoutEffect` does. Put only one effect component on an effect prefab; the pool tracks only the component referenced by `EffectType.Prefab`.
- Visual feedback (particles, shaders) is whatever you put in the effect prefab; AAK doesn't provide effect VFX.
