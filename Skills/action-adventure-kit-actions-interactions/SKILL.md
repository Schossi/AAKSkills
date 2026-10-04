---
name: action-adventure-kit-actions-interactions
description: "Use this skill whenever the user wants the player to interact with the world in Action Adventure Kit — e.g. 'pick up items with E', 'make a door that opens when i press a button', 'gate that costs a key', 'lever that triggers something', 'talk to an npc and show a dialog', 'add a shop to buy potions', 'do something when the player walks into this zone', 'collect coins by touching them', 'pickup gets stuck'. Covers actors, CharacterActionArea with input and prompt text, PickupAction, ObjectAction, ButtonAction, DialogAction, PurchaseAction, the START/ACT/END animation messages, GenericTriggerArea zones and ItemPickupArea. Do NOT use for movement or NPC patrols (see action-adventure-kit-character-movement), item definitions (see action-adventure-kit-items-inventory), or damage zones (see action-adventure-kit-health-stats-damage). When in doubt whether an interaction request could involve Action Adventure Kit, use this skill — Prerequisites shows how to confirm the asset is installed."
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

# Add Interactions and Trigger Areas with Action Adventure Kit

In AAK everything a character does — picking something up, opening a gate, talking, buying, attacking — is an **action** (`CharacterActionBase`) run by the character's **actor** (`CharacterActorBase`). A `CharacterActionArea` on the character finds nearby actions through trigger colliders and starts them on input. Separately, `GenericTriggerArea`/`GenericTriggerItem` pairs react to characters entering zones. This skill lets an agent wire the actor, create the common world interactions, and build trigger zones.

## When to use this skill

- "player should pick up the cubes when pressing the action button"
- "make a gate that opens with an animation and costs one key"
- "add a lever that opens a door"
- "npc says hello when i press E"
- "shop where i can buy a potion for 10 coins"
- "do something when the player enters this area"
- "coins should be collected just by touching them"
- "the 'Pick up' text never shows" / "my pickup action never ends"

Not for:
- Movement, NPC patrol/approach/wander → `action-adventure-kit-character-movement`
- Creating the items that are picked up or paid → `action-adventure-kit-items-inventory`
- Damage zones, traps, attacks dealing damage → `action-adventure-kit-health-stats-damage`
- Effects applied by zones → `action-adventure-kit-status-effects`
- Making pickups/gates remember their state → `action-adventure-kit-save-load`
- Timeline cutscenes (`TimelineAction`), Visual Scripting actions → see https://adventure.softleitner.com/manual/acting

## Prerequisites

- Action Adventure Kit 1.10.1+ (`AdventureCore`), Unity 6000.0.76+, Input System.
- A character built per `action-adventure-kit-character-movement` (with `PlayerInput` and `AnimatorProxy` on the model). Item-based interactions also need an inventory (`action-adventure-kit-items-inventory`).
- **Install check:** `System.Type.GetType("AdventureCore.CharacterActionArea, AdventureCore") != null`. If missing, tell the user to import Action Adventure Kit from https://assetstore.unity.com/packages/templates/systems/action-adventure-kit-217284 and stop.

## Quick start

Give an existing character an actor + action area, and add a pickup (Getting Started step 4). Run from an Editor script:

```csharp
using AdventureCore;
using UnityEditor;
using UnityEditor.Events;
using UnityEditor.SceneManagement;
using UnityEngine;
using UnityEngine.Events;
using UnityEngine.InputSystem;

public static class AakCreatePickup
{
    public static void Run(GenericCharacter character, ItemBase item)
    {
        // Actor + action area
        var actorGo = new GameObject("Actor");
        actorGo.transform.SetParent(character.transform, false);
        var actor = actorGo.AddComponent<MinimalCharacterActor>();
        actor.Character = character;
        character.Actor = actor;

        var area = actorGo.AddComponent<CharacterActionArea>();
        area.CharacterActor = actor;
        var areaCollider = actorGo.AddComponent<SphereCollider>();
        areaCollider.isTrigger = true;
        areaCollider.radius = 1f;
        areaCollider.center = new Vector3(0f, 1f, 0f);
        actorGo.AddComponent<Rigidbody>().isKinematic = true;   // static pickups have no rigidbody

        // bind the "Action" input to CharacterActionArea.OnStartAction
        var input = character.GetComponent<PlayerInput>();
        var events = new System.Collections.Generic.List<PlayerInput.ActionEvent>(input.actionEvents);
        var actionEvent = new PlayerInput.ActionEvent(input.actions.FindAction("Action"));
        UnityEventTools.AddPersistentListener(actionEvent,
            new UnityAction<InputAction.CallbackContext>(area.OnStartAction));
        events.Add(actionEvent);
        input.actionEvents = events.ToArray();

        // Pickup
        var pickup = GameObject.CreatePrimitive(PrimitiveType.Cube);
        pickup.name = "Pickup";
        pickup.transform.position = character.transform.position + new Vector3(2f, 0.5f, 0f);
        pickup.GetComponent<BoxCollider>().isTrigger = true;
        var action = pickup.AddComponent<PickupAction>();
        action.CharacterTriggerName = "Pickup";                  // animator trigger on the character
        action.Items = new ItemQuantity(item, 1);
        action.Instructions = new CharacterInstructionBase[] { new SuspendMovementInstruction() };

        EditorSceneManager.MarkSceneDirty(character.gameObject.scene);
    }
}
```

1. Make sure the character has an inventory and its animator has a `Pickup` trigger whose clip sends `START` and `END` animation events (the Getting Started `GSRobo` controller does; see the animation-message workflow otherwise).
2. Run `AakCreatePickup.Run(character, item)`.
3. Enter Play mode, walk to the cube and press the Action key.

Expected result: the Hierarchy has `Character/Actor` with `MinimalCharacterActor`, `CharacterActionArea`, a trigger `SphereCollider` and a kinematic `Rigidbody`, plus a `Pickup` cube. In Play mode, pressing Action near the cube plays the pickup animation, adds 1 item to the inventory, and destroys the cube when the animation sends `END`.

## Workflows

### Workflow: Understand the START / ACT / END message protocol

**Goal:** Make animation-driven actions start, act and finish correctly.

**Steps:**
1. Actions that play a character animation (`PickupAction`, `ObjectAction`, `ButtonAction`, `PurchaseAction`) set an animator trigger (`CharacterTriggerName`/`TriggerName`) and then wait for messages from the animation:
   - `START` — the animation has begun (default `StartMessage`).
   - `ACT` — the moment of impact, e.g. the hand reaches the handle; `ObjectAction` triggers the object animator here (default `ActMessage`).
   - `END` — the action is finished and the actor is freed (default `EndMessage`).
2. Send them with animation events on the clip, function `OnAnimation` (or `OnMessage`), string parameter `START`/`ACT`/`END`. The events reach the action through `AnimatorProxy` → character → current action. The Getting Started clips `GSPickup.anim` (START at 0, END at 1) and `GSOpen.anim` (START, ACT at 0.67, END) are working references.
3. No suitable animator state? Fill the action's `CharacterAnimation` (`PlayableAnimationParameters`: `Clip`, `Speed`, `LeadIn`/`LeadOut`, `Messages`, `Instructions`). The clip is played directly on the character's animator and the action ends when it finishes, so no animator trigger is needed.

**Expected result:** The action's `IsHappening` turns true on start and false after `END`; the actor can start the next action.

### Workflow: Gate or door with an item cost (ObjectAction)

**Goal:** A gate that opens with synchronized character and gate animations and consumes a key (Getting Started step 5).

**Steps:**
1. Place the gate model with an `Animator` that has an `Open` trigger and an `Opened` state (reference: `Assets/SoftLeitner/AdventureManual/GettingStarted/Prefabs/GSGate.prefab`).
2. Add a child `Action` placed where the character should stand. Add `AdventureCore.ObjectAction` and a trigger `BoxCollider` (size 2).
3. Configure: `Name` = `Open` (prompt text), `Cost` = `{ Item: key item, Quantity: 1 }`, `CharacterTriggerName` = `Open`, `ObjectCharacterTarget` = the `Action` transform (character is moved there), `ObjectTriggerName` = `Open`, `ObjectStateName` = `Opened` (state jumped to when loading an already opened gate), `ObjectAnimator` = the gate's animator.
4. Optional: `ObjectAnimatorEnd` + `ObjectTriggerNameEnd`/`ObjectStateNameEnd` for a second object triggered on `END`; `Acting` UnityEvent fires on `ACT`; `Persister` (a `ManualPersister`) remembers that it was used.

**Expected result:** Near the gate the prompt reads `Open`; with a key in the inventory, pressing Action moves the character to the spot, plays both animations, removes the key and leaves the gate open. Without a key `CanStart` is false (the `CharacterActionAreaText` shows it in `ColorDisabled`).

### Workflow: Lever / button with cooldown (ButtonAction)

**Goal:** A reusable switch.

**Steps:** add `AdventureCore.ButtonAction` + trigger collider to the lever. Set `Name`, optional `Cost`, `Cooldown` (seconds before it can be used again), `CharacterTriggerName`, `ObjectAnimator` + `ObjectTriggerName`. Hook what the lever does to its `Acting` event (fires on `ACT`) or the base `Starting`/`Ending` events.

**Expected result:** The lever can be used repeatedly, at most once per `Cooldown`, and its events fire each time.

### Workflow: NPC dialog (DialogAction)

**Goal:** Show text when the player interacts with an NPC.

**Steps:**
1. Make sure the scene contains a dialog component deriving from `AdventureCore.DialogBase`: `DialogTMP` / `DialogUI` (uGUI) or `DialogElement` (UI Toolkit). The easiest route is to copy a dialog from a demo UI. A dialog with an empty `Key` is the main dialog.
2. On the NPC add a trigger collider and `AdventureCore.DialogAction`: `Title`, `Texts` (one page per entry), `Image`, `Buttons` (default `Ok`), `Selection`, `Key` (empty = main dialog).
3. React to the answer with `Closed` (`UnityEvent<DialogResult>`).

**Expected result:** Pressing Action near the NPC shows the dialog; closing it fires `Closed` and ends the action.

### Workflow: Shop purchase (PurchaseAction)

**Goal:** Buy items for currency.

**Steps:** add `AdventureCore.PurchaseAction` + trigger collider to the shop item. `Items` = items received, `ItemCosts` = items paid (e.g. 10 coins), optionally resource or attribute costs. `OnlyGrounded` (default true), `NameFormat` (default `Buy {0}({1})`). Leave `ActMessage` empty to exchange items at the start, or set it to exchange on that animation message.

**Expected result:** The prompt shows e.g. `Buy Potion(Coin x 10)`; using it removes the coins and adds the potion; without enough coins `CanStart` is false.

### Workflow: Trigger zones (GenericTriggerArea)

**Goal:** Run logic when characters enter/leave an area.

**Steps:**
1. The character needs an `AdventureCore.GenericTriggerItem` with `Character` set, on a GameObject with a collider or on the Rigidbody that owns the collider (the root with `CharacterController` works). Getting Started adds a trigger `CapsuleCollider` next to it.
2. The zone: collider with `isTrigger` + `AdventureCore.GenericTriggerArea`.
3. Use any combination of:
   - `Instructions` — added while inside (e.g. `MovementSpeedMultiplierInstruction`).
   - `CharacterAdded` / `CharacterRemoved` (`UnityEvent<CharacterBase>`) — bind to methods taking a character, e.g. `EffectType.AddExternal` / `EffectType.Remove`, `ItemBase.Add`, `ResourceType.Fill`, `AttributeType.Add`.
   - `EnterMessage` / `ExitMessage` — character messages (space separated).
   - `Key` — must match the item's `Key` (space-separated lists allowed) to filter which items count; leave both empty to match everything.
4. Auto-collect items on touch: use `AdventureCore.ItemPickupArea` (a `GenericTriggerArea`) with `PickupItems`, `PickupMode` (`Always`/`Partial`/`Fully` depending on inventory space) and optional `PickedPrefab`.

**Expected result:** Entering the zone fires `CharacterAdded` once and applies instructions; leaving reverses them.

### Workflow: Start an action from code

**Goal:** Trigger actions without the action area (AI, scripts, UI buttons).

**Steps:**
- `actor.StartAction(action)` or `action.StartAction(actor)`; `force: true` interrupts the current action, `jumpStart: true` skips to the end state.
- Actions that are children of the actor can be started by GameObject name: `actor.StartAction("Patrol")`.
- If `CharacterActionBase.Owner` is set, the parameterless `action.StartAction()` works (usable from UnityEvents).
- `GenericAction` (resource `Costs`) and `TimedAction` (`Duration`) are building blocks that don't need animations.

**Expected result:** `action.IsHappening` becomes true and `actor.ActionStarted` fires.

## Verification

- Character: `GenericCharacter.Actor != null`; actor object has `CharacterActionArea` with `CharacterActor` set, a trigger collider, and a Rigidbody (kinematic) or the interactables have one.
- `PlayerInput` has an action event whose persistent method is `OnStartAction` targeting the area.
- Each interactable has a trigger collider on **the same GameObject** as its `CharacterActionBase` (the area uses `other.GetComponent<CharacterActionBase>()`).
- Play mode: near the interactable, `area.HasAction == true` and `area.Text` equals the expected prompt (`Pick up <ItemName>`, the `Name` of an `ObjectAction`, etc.).
- After interacting: inventory quantity changed as expected; pickup GameObject destroyed; gate animator in `Opened` state (`animator.GetCurrentAnimatorStateInfo(0).IsName("Opened")`).
- No "AnimationEvent has no receiver" warnings; actor not stuck (`actor.IsActing == false` after completion).

## API quick reference

| Entry point | Type | What it does |
|---|---|---|
| `AdventureCore.MinimalCharacterActor` | CharacterActorBase | Runs one action at a time; discards requests while busy |
| `AdventureCore.BufferedCharacterActor` | CharacterActorBase | Buffers a request for `Timeout` (1 s) until the current action can end |
| `AdventureCore.CharacterActionArea` | MonoBehaviour | Finds nearby actions; `OnStartAction(CallbackContext)`, `HasAction`, `Text`, `Changed` |
| `AdventureCore.CharacterActionAreaText` | MonoBehaviour | Shows the area's current action name (`Text`/`TMPText`) |
| `AdventureCore.PickupAction` | Action | Adds `Items`/`ItemsArray`, destroys itself on `END` |
| `AdventureCore.ObjectAction` | Action | Character + object animation, optional item `Cost`, persistable |
| `AdventureCore.ButtonAction` | Action | Reusable with `Cooldown` |
| `AdventureCore.DialogAction` | Action | Shows a `DialogBase` dialog |
| `AdventureCore.PurchaseAction` | Action | Exchanges costs for items |
| `CharacterActorBase.StartAction(name/action)`, `CharacterActionBase.StartAction()` | Methods | Start from code |
| `AdventureCore.GenericTriggerArea` / `GenericTriggerItem` | MonoBehaviour | Zones with instructions, events and messages |
| `AdventureCore.ItemPickupArea` | GenericTriggerArea | Adds items on enter |

## Common issues

- **Symptom:** Prompt never appears near an interactable. **Cause:** No Rigidbody on either side (two static triggers don't generate events), the action's collider is not a trigger, or the action component is on a different GameObject than its collider. **Fix:** Add a kinematic Rigidbody to the actor's area object; keep collider and action together.
- **Symptom:** Pressing the key does nothing. **Cause:** `Action` input not bound to `CharacterActionArea.OnStartAction`, or `GenericCharacter.Actor` empty. **Fix:** Bind the event; assign the actor.
- **Symptom:** Item added but the pickup object stays and no further action can start. **Cause:** The character animation never sends `END` (missing animator trigger/state or animation events), so the action never finishes. **Fix:** Add `START`/`END` events (`OnAnimation`) to the clip, or use `CharacterAnimation` with a clip instead of `CharacterTriggerName`.
- **Symptom:** Gate opens on the character but the gate itself doesn't animate. **Cause:** No `ACT` event in the character clip, or `ObjectTriggerName` doesn't match the gate animator trigger. **Fix:** Add `ACT` and match the names.
- **Symptom:** Prompt is grey and the action can't start. **Cause:** `CanStart` false: missing `Cost`/`ItemCosts`, inventory full for a pickup, `ButtonAction` cooling down, `PurchaseAction.OnlyGrounded` while airborne. **Fix:** Satisfy the condition.
- **Symptom:** Trigger zone ignores the player. **Cause:** No `GenericTriggerItem` on the character collider/rigidbody, or `Key` values differ. **Fix:** Add the item; clear or match `Key`s.
- **Symptom:** `DialogAction` throws or shows nothing. **Cause:** No `DialogBase` in the scene (or none with the given `Key`). **Fix:** Add a dialog UI from a demo.

## Boundaries

- AAK does not provide branching dialogue trees; `DialogAction` shows pages with buttons and reports a `DialogResult`. Use Visual Scripting or a dialogue asset for complex conversations.
- No quest system; quests in the demos are persistence flags plus Visual Scripting.
- Animations for interactions are not generated; supply clips with `START`/`ACT`/`END` events or use the GS/demo clips.
- Combat attacks are also actions, but weapon/attack setups are demo-specific (Souls/Hero); see https://adventure.softleitner.com/manual/souls-player.
