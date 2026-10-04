---
name: action-adventure-kit-health-stats-damage
description: "Use this skill whenever the user wants health, stamina, attributes, stats, or damage in Action Adventure Kit — e.g. 'give my player 100 health', 'add a health bar', 'lava that hurts the player', 'healing zone', 'stamina that regenerates', 'vitality should raise max hp', 'strength increases damage', 'player takes half fire damage', 'enemy doesnt take damage', 'what happens at zero health'. Covers resources (ResourceType, ResourceValue, ResourcePool), attributes and stats with ResourceMaximum links, damage kinds, TriggerDamageSender/Receiver, SphereDamageSender, damage modifiers and messages, and debug UI. Do NOT use for buffs, debuffs or poison over time (see action-adventure-kit-status-effects), potions (see action-adventure-kit-items-inventory), or saving values (see action-adventure-kit-save-load). When in doubt whether a health, stat or damage request could involve Action Adventure Kit, use this skill — Prerequisites shows how to confirm the asset is installed."
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

# Add Health, Stats and Damage with Action Adventure Kit

AAK separates three kinds of numbers: **resources** (float values that change frame by frame, like health or stamina), **attributes** (whole numbers that change rarely, like vitality or strength) and **stats** (values computed from attributes, like max health = vitality × 10). **Damage** is sent by senders to receivers as `DamageEvent`s whose `DamageKind` decides the effect — `ResourceDamage` changes a resource. This skill lets an agent give characters health and stats, link stats to resource maximums, build damaging and healing zones or attacks, and react to damage.

## When to use this skill

- "give the player 100 hp and show it on screen"
- "lava should hurt the player every tick, a fountain heals"
- "add stamina that refills by itself"
- "vitality attribute raises max health"
- "strength should scale weapon damage"
- "enemy takes half damage from arrows"
- "play a hit reaction when damaged" / "do something when health reaches 0"
- "my trap doesn't do any damage"

Not for:
- Poison, burning, buffs that last a while → `action-adventure-kit-status-effects`
- Potions and equipment that change these values → `action-adventure-kit-items-inventory`
- Saving health/attributes → `action-adventure-kit-save-load`
- Weapon/attack animations and combos → demo specific, see https://adventure.softleitner.com/manual/souls-player and /damage

## Prerequisites

- Action Adventure Kit 1.10.1+ (`AdventureCore`), Unity 6000.0.76+.
- A character with `GenericCharacter` (see `action-adventure-kit-character-movement`).
- **Install check:** `System.Type.GetType("AdventureCore.ResourcePool, AdventureCore") != null` and the menu `Assets > Create > Adventure > ResourceType` exists. If not, tell the user to import Action Adventure Kit from https://assetstore.unity.com/packages/templates/systems/action-adventure-kit-217284 and stop.

## Quick start

Give a character 100 health and add a damage zone (Getting Started step 6). Run from an Editor script:

```csharp
using AdventureCore;
using UnityEditor;
using UnityEditor.SceneManagement;
using UnityEngine;

public static class AakCreateHealth
{
    public static void Run(GenericCharacter character, string folder = "Assets/Game/Stats")
    {
        if (!AssetDatabase.IsValidFolder(folder))
        {
            System.IO.Directory.CreateDirectory(folder);
            AssetDatabase.Refresh();
        }

        var healthType = ScriptableObject.CreateInstance<ResourceType>();
        healthType.Key = "HP"; healthType.Name = "Health";
        AssetDatabase.CreateAsset(healthType, folder + "/Health.asset");

        var damage = ScriptableObject.CreateInstance<ResourceDamage>();
        damage.ResourceType = healthType;           // Add = false -> removes health
        AssetDatabase.CreateAsset(damage, folder + "/HealthDamage.asset");
        AssetDatabase.SaveAssets();

        // Status object holding the pools
        var status = new GameObject("Status");
        status.transform.SetParent(character.transform, false);
        var healthGo = new GameObject("Health");
        healthGo.transform.SetParent(status.transform, false);
        var health = healthGo.AddComponent<ResourceValue>();
        health.Type = healthType; health.Maximum = 100f; health.Value = 100f;

        var pool = status.AddComponent<ResourcePool>();
        pool.Values = new[] { health };
        character.ResourcePool = pool;

        // the character receives damage through a TriggerDamageReceiver next to its collider
        var receiver = character.gameObject.AddComponent<TriggerDamageReceiver>();
        receiver.Character = character;

        // damage zone
        var zone = GameObject.CreatePrimitive(PrimitiveType.Cube);
        zone.name = "DamageZone";
        zone.transform.position = character.transform.position + new Vector3(-3f, 0.5f, 0f);
        zone.GetComponent<BoxCollider>().isTrigger = true;
        var sender = zone.AddComponent<TriggerDamageSender>();
        sender.Damages = new[] { new DamageParameter { Kind = damage, Value = 1f } };
        sender.SendTick = true;
        sender.TickRate = 0.1f;

        EditorUtility.SetDirty(character);
        EditorSceneManager.MarkSceneDirty(character.gameObject.scene);
    }
}
```

1. Run `AakCreateHealth.Run(character)`.
2. Enter Play mode and walk into `DamageZone`.

Expected result: assets `Health.asset` and `HealthDamage.asset` exist; the character has `Status` (with `ResourcePool`) and `Status/Health` (`ResourceValue`, 100/100) plus a `TriggerDamageReceiver`. In Play mode, standing in the zone lowers `pool.GetValue(healthType)` by 1 every 0.1 s.

## Workflows

### Workflow: Resources (health, stamina, mana)

**Goal:** Give a character float resources.

**Steps:**
1. Create a `ResourceType` per resource (`Assets > Create > Adventure > ResourceType`) with a unique `Key` and a `Name`.
2. On the character, add one GameObject per resource under a `Status` object with a value component:
   - `ResourceValue` — `Type`, `Minimum`, `Maximum`, `Value` (starting value).
   - `ChangingResourceValue` — adds `Rate` (change per second, e.g. stamina regeneration) and `Lag` (delay after an external change before regeneration resumes).
   - `EffectResourceValue` — a changing value that adds an `EffectType` when full (build-up like poison); see the effects skill.
3. Add `ResourcePool` (on `Status`), list every value in `Values`, and assign it to `CharacterBase.ResourcePool`.
4. Change at runtime: `pool.AddResource(type, amount, source)`, `RemoveResource`, `FillResource(type)`, `ClearResource(type)`, `ResetResources()`; query with `GetValue`, `GetMaximum`, `GetRatio`, `HasResourceQuantity`. From UnityEvents you can use `ResourceType.Fill(character)` / `Clear(character)`.

**Expected result:** `pool.GetValue(type)` returns the starting value; changes are clamped to `Minimum..Maximum`.

### Workflow: Attributes and stats (vitality → max health)

**Goal:** Drive resource maximums and other numbers from attributes (Getting Started step 7).

**Steps:**
1. Create an `AttributeType` (`Assets > Create > Adventure > AttributeType`, unique `Key`, e.g. `VIT`).
2. Create a stat (`Assets > Create > Adventure > Stats > ...`):
   - `MultipliedStat` — `AttributeType` × `Multiplier` (Getting Started: `GSHealthMaximum` = Vitality × 10).
   - `TieredStat` — `AttributeType` with `Tiers` (`Threshold`, `Improvement`) for diminishing returns.
   - `ConstantStat` — just the base value set in `AttributePool.BaseStats`.
3. Add `AttributePool` to `Status`, assign it to `CharacterBase.AttributePool`, and fill:
   - `Attributes` — every attribute the character has, with starting values (e.g. Vitality 10). **Attributes not listed are ignored by `Add`/`Remove`.**
   - `BaseStats` — base values for stats.
   - `ResourceMaximums` — `{ ResourceValue: Health, AttributeStat: HealthMaximum }`; whenever the stat changes, the resource maximum is set to it.
4. Change attributes: `attributePool.Add(type, n)`, `Remove`, `Set`; read `GetValue(attributeType)` or `GetValue(stat)`. From UnityEvents: `AttributeType.Add(character)` / `Remove(character)` (±1).

**Expected result:** With Vitality 10 and Multiplier 10, health `Maximum` becomes 100; adding 1 Vitality raises it to 110.

### Workflow: Damage zones, traps and healing

**Goal:** Areas or objects that hurt or heal characters.

**Steps:**
1. Create damage kinds (`Assets > Create > Adventure > Damages > ...`):
   - `ResourceDamage` — `ResourceType` to change, `Add` (check for healing), `ShowBar` (pops a bar via `ResourceBarManager`).
   - `GenericDamage` — carries a value/color only; react to it via messages, modifiers or events.
2. Receiver: add `TriggerDamageReceiver` to the character on a GameObject with a collider (the root with `CharacterController` works) and set `Character`. `ResourceDamage` changes the receiver character's `ResourcePool`, so `Character` must be set. Optional: `Filter` (accepted kinds only), `Modifiers`, `Messages`.
3. Sender options:
   - `TriggerDamageSender` on a trigger collider: `Damages` (`DamageParameter { Kind, Value }`), `SendStart` (on enter, default true), `SendTick` + `TickRate` (periodic), `SendEnd` (on exit), `SendOnce` (once per activation, for weapons), `Direction`/`Magnitude` (push vector). `Activate()`/`Deactivate()` toggle the GameObject, e.g. from animation events for attack hitboxes.
   - `SphereDamageSender` — `Physics.OverlapSphere` with `Radius`/`LayerMask`, `SendStart`/`SendUpdate`; good for explosions.
   - `ProjectileDamageSender`, `MovingDamageSender` — for projectiles and moving hazards.
4. Healing zone: same as damage, with a `ResourceDamage` whose `Add` is checked (Getting Started `GSHealDamage`).
5. Test from code without a sender: `damageKind.SendTen(character)` (also `SendHalf`, `SendOne`, `SendTwo`, `SendFive`).

**Expected result:** Receivers inside the sender lose (or gain) the resource as configured; `TriggerDamageReceiver.Damaged` and `CharacterBase.Damaged` fire.

### Workflow: Modify and react to damage

**Goal:** Resistances, damage scaling, hit reactions, death.

**Steps:**
1. Modifiers (`DamageModifier { DamageKind, Value, Stat, Method }`; empty `DamageKind` = all kinds; `Stat` reads the value from a character stat; `Method` from `DamageModifierMethod`: `Add`, `Subtract`, `Multiply`, `Divide`, `Max`, `Min`; e.g. `Multiply` 0.5 for half damage). Put them in `CharacterBase.DamageReceiverModifiers` (defence), `DamageSenderModifiers` (attack power), or on individual senders/receivers.
2. Messages: `DamageMessage { Kind, Tag, MinValue (0), MaxValue (999), Message, Triggered }` in `CharacterBase.DamageReceivedMessages` sends a character message (e.g. a `Hit` animator reaction via `MessageEvents`) or fires `Triggered`.
3. Events: `CharacterBase.Damaging` (per event, still modifiable), `Damaged` (after all events), `DamagingOther`/`DamagedOther` for sent damage.
4. Death / zero health: AAK has no built-in death state. Listen to the resource's `ResourceValue.Changed` (`UnityEvent<float>`) and handle `value <= 0` (the demos play a death action, respawn or reload):

```csharp
health.Changed.AddListener(v => { if (v <= 0f) Debug.Log($"{character.name} died"); });
```

5. Temporarily invulnerable: add `SuspendReceiveDamageInstruction` to the character (e.g. on a dodge action's `Instructions`).

**Expected result:** Modified damage values appear in `DamageEvent.Value`; messages/events fire for matching damage; the death handler runs when health reaches 0.

### Workflow: Show values in UI

**Goal:** Debug text or bars.

**Steps:**
1. Text: `ResourcePoolText` (`ResourcePool`, `Text`/`TMPText`) and `AttributePoolText` (`AttributePool`, `Text`/`TMPText`) on UI text objects.
2. Bars: `ResourceBarPanel` (uGUI), `ResourceBarElement` (UI Toolkit) or `ResourceBarTMP`, with `ResourceType` and `ResourcePool`. For pop-up bars over characters add a `ResourceBarManager` (singleton) with `Prefabs` per `ResourceType`, `Parent` and `TemporaryDuration`; `ResourceDamage.ShowBar` uses it. `ResourcePool.BarTarget` is the transform the bar follows.

**Expected result:** UI reflects current values and updates on change.

## Verification

- `character.ResourcePool != null` and `pool.Values` contains a `ResourceValue` per expected `ResourceType`; `pool.ContainsResource(type)` is true in Play mode.
- Every `ResourceType`/`AttributeType` asset has a unique, non-empty `Key`.
- `TriggerDamageReceiver.Character` is set and the receiver is on a GameObject with a collider (or on the collider's Rigidbody).
- Damage zone: in Play mode with the character inside, `pool.GetValue(type)` decreases over time (tick) or once (start).
- Attributes: `attributePool.GetValue(stat)` equals the expected formula; `health.Maximum` equals the linked stat.
- Console has no `NullReferenceException` from `ResourcePool.Initialize` (`Values` null) or `ResourceDamage.OnDamage`.

## API quick reference

| Entry point | Type | What it does |
|---|---|---|
| `AdventureCore.ResourceType` / `AttributeType` | ScriptableObject | Identify resources/attributes (`Key`, `Name`, `Image`) |
| `AdventureCore.ResourceValue` / `ChangingResourceValue` | MonoBehaviour | One resource; `Changed`, `MaximumChanged`, `Set`, `Change` |
| `AdventureCore.ResourcePool` | MonoBehaviour | `AddResource`, `RemoveResource`, `FillResource`, `GetValue`, `GetRatio`, `ResetResources` |
| `AdventureCore.AttributePool` | MonoBehaviour | `Add`, `Remove`, `Set`, `GetValue(AttributeType/AttributeStat)`, `ResourceMaximums` |
| `AdventureCore.MultipliedStat`, `TieredStat`, `ConstantStat` | ScriptableObject | Stat formulas |
| `AdventureCore.ResourceDamage`, `GenericDamage` | DamageKind | What damage does; `SendOne/SendTen(character)` for tests |
| `AdventureCore.TriggerDamageSender` / `TriggerDamageReceiver` | MonoBehaviour | Trigger-based damage |
| `AdventureCore.SphereDamageSender` | MonoBehaviour | Overlap-sphere damage |
| `AdventureCore.DamageModifier`, `DamageMessage` | Serializable | Scale damage / react to damage |
| `AdventureCore.ResourcePoolText`, `AttributePoolText`, `ResourceBarManager` | MonoBehaviour | UI |

## Common issues

- **Symptom:** Damage zone does nothing. **Cause:** Receiver missing, receiver not on a collider/rigidbody GameObject, `TriggerDamageReceiver.Character` empty, or `TriggerDamageSender.Damages` empty. **Fix:** Check each, per Verification.
- **Symptom:** Damage only happens once while standing in lava. **Cause:** `SendTick` off (only `SendStart`). **Fix:** Enable `SendTick` and set `TickRate`.
- **Symptom:** Weapon hits the same enemy several times per swing. **Cause:** Collider phases in and out. **Fix:** Enable `SendOnce` and activate/deactivate the sender per attack.
- **Symptom:** `AttributePool.Add` has no effect. **Cause:** The attribute isn't listed in `AttributePool.Attributes`. **Fix:** Add it with a starting value.
- **Symptom:** Max health doesn't follow vitality. **Cause:** No `ResourceMaximums` entry, or the stat's `AttributeType` differs from the changed attribute. **Fix:** Link the value and the correct stat.
- **Symptom:** `NullReferenceException` in `ResourcePool.Initialize` when creating pools from code. **Cause:** `Values` array not assigned. **Fix:** Always assign `Values` (and `Attributes`, `BaseStats`, `ResourceMaximums` on `AttributePool`) explicitly.
- **Symptom:** Character can't be damaged at all. **Cause:** `IsReceiveDamageSuspended` via a `SuspendReceiveDamageInstruction` (e.g. from an active action or effect), or a receiver `Filter` excluding the kind. **Fix:** Remove the instruction or adjust the filter.

## Boundaries

- No built-in death/respawn flow, experience/levelling formulas or combat AI; build them on the events above (the demos show complete examples).
- Resources are floats and attributes are integers by design; use resources for anything continuous.
- Damage-over-time is an effect (`DamageEffect`), not a sender setting other than ticking zones; see the effects skill.
- Hitbox shapes are Unity colliders; AAK does not do per-bone hit detection beyond placing senders/receivers on bones yourself.
