---
name: action-adventure-kit-character-movement
description: "Use this skill whenever the user wants to create, rig, or move a player character or NPC with Action Adventure Kit — e.g. 'make me a player i can walk around with wasd', 'set up a third person character', 'make an enemy that patrols between points', 'npc should walk on the navmesh', 'make a 2d top down or side scroller character', 'slow the player down in water', 'my character doesnt move'. Covers GenericCharacter, CharacterControllerMovement with Input System wiring, AnimatorProxy, NavMesh movements with patrol/approach/wander actions, 2D movements, and character instructions such as speed multipliers. Do NOT use for pickups, doors or dialogs (see action-adventure-kit-actions-interactions), health and damage (see action-adventure-kit-health-stats-damage), or saving position (see action-adventure-kit-save-load). When in doubt whether a character or movement request could involve Action Adventure Kit, use this skill — Prerequisites shows how to confirm the asset is installed."
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

# Create a Character with Action Adventure Kit

In AAK a character is a `CharacterBase` component (usually `GenericCharacter`) that ties together an actor ("what it does"), a movement ("where it is"), an inventory ("what it has") and resource/attribute/effect pools. This skill lets an agent build a controllable player, a NavMesh-driven NPC, or a 2D character, connect input and animation, and change movement through character instructions.

## When to use this skill

- "make a player character i can move with wasd"
- "set up the robot from the getting started guide as my player"
- "i need an enemy that patrols between these waypoints"
- "npc should walk to the player / wander around"
- "make a top down 2d character" / "make a platformer character"
- "slow the character down in mud"
- "character doesn't move when i press keys"

Not for:
- Picking things up, doors, levers, talking to NPCs → `action-adventure-kit-actions-interactions`
- Health, damage, stats → `action-adventure-kit-health-stats-damage`
- Timed buffs/debuffs (status effects) → `action-adventure-kit-status-effects`
- Saving character position → `action-adventure-kit-save-load`
- Lock-on cameras, ragdolls, Timeline-driven cutscene movement → see the manual (https://adventure.softleitner.com/manual/movement, /character)

## Prerequisites

- Action Adventure Kit 1.10.1+ with `AdventureCore`. The Quick start also uses assets from `AdventureManual` (GSRobo prefab, GSInput actions).
- Unity 6000.0.76+, Input System package (the Player Settings "Active Input Handling" must include the new Input System).
- NavMesh workflows need the AI Navigation package (`Unity.AI.Navigation.NavMeshSurface`).
- **Install check:** `System.Type.GetType("AdventureCore.GenericCharacter, AdventureCore") != null`, and for the Quick start the asset `Assets/SoftLeitner/AdventureManual/GettingStarted/Prefabs/GSRobo.prefab` exists. If the type is missing, tell the user to import Action Adventure Kit from https://assetstore.unity.com/packages/templates/systems/action-adventure-kit-217284 and stop. If only the GS assets are missing, ask the user to import `AdventureManual` or use their own model and input actions.

## Quick start

Create a WASD-controlled player in the open scene, mirroring Step 1 of the official Getting Started guide. Run this from an Editor script (for example a static method in an `Editor` folder or your C# execution tool):

```csharp
using AdventureCore;
using UnityEditor;
using UnityEditor.Events;
using UnityEditor.SceneManagement;
using UnityEngine;
using UnityEngine.Events;
using UnityEngine.InputSystem;

public static class AakCreatePlayer
{
    public static void Run()
    {
        const string gs = "Assets/SoftLeitner/AdventureManual/GettingStarted/";

        var root = new GameObject("Character");
        Undo.RegisterCreatedObjectUndo(root, "Create AAK Character");

        // CharacterController first: CharacterControllerMovement requires it
        var controller = root.AddComponent<CharacterController>();
        controller.center = new Vector3(0f, 1f, 0f);
        controller.height = 2f;
        controller.radius = 0.5f;
        controller.skinWidth = 0.001f;

        var character = root.AddComponent<GenericCharacter>();
        var movement = root.AddComponent<CharacterControllerMovement>();

        // visual model with Animator + AnimatorProxy
        var roboPrefab = AssetDatabase.LoadAssetAtPath<GameObject>(gs + "Prefabs/GSRobo.prefab");
        var robo = (GameObject)PrefabUtility.InstantiatePrefab(roboPrefab, root.transform);
        character.Animator = robo.GetComponent<Animator>();
        robo.GetComponent<AnimatorProxy>().Character = character;

        character.Movement = movement;
        movement.Pivot = robo.transform;                       // rotated toward the move direction (required)
        movement.Camera = Camera.main ? Camera.main.transform : null; // input becomes camera-relative

        // input: Invoke Unity Events, Move -> CharacterControllerMovement.OnMove
        var input = root.AddComponent<PlayerInput>();
        input.actions = AssetDatabase.LoadAssetAtPath<InputActionAsset>(gs + "Other/GSInput.inputactions");
        input.notificationBehavior = PlayerNotifications.InvokeUnityEvents;
        var moveEvent = new PlayerInput.ActionEvent(input.actions.FindAction("Move"));
        UnityEventTools.AddPersistentListener(moveEvent,
            new UnityAction<InputAction.CallbackContext>(movement.OnMove));
        input.actionEvents = new[] { moveEvent };

        EditorSceneManager.MarkSceneDirty(root.scene);
        Selection.activeGameObject = root;
    }
}
```

Steps:
1. Make sure the scene has a floor with a collider and a `MainCamera`-tagged camera looking at the origin.
2. Run `AakCreatePlayer.Run()`.
3. Enter Play mode and press WASD.

Expected result: the Hierarchy contains `Character` with `CharacterController`, `GenericCharacter`, `CharacterControllerMovement`, `PlayerInput` and a child `GSRobo`. In Play mode the character moves relative to the camera and turns to face the move direction. Console is clean.

## Workflows

### Workflow: Rig a player character on your own model

**Goal:** Turn any humanoid or prop model into an AAK player.

**Steps:**
1. Create an empty root GameObject (reset transform). Add `CharacterController`, then `AdventureCore.GenericCharacter`, then `AdventureCore.CharacterControllerMovement`.
2. Put the visual model as a child. On the model's GameObject (the one with the `Animator`) add `AdventureCore.AnimatorProxy` and set `Character` to the root's `GenericCharacter`. The proxy forwards animation events (`OnAnimation`, `OnMessage`, `OnMessages`) and root motion (`OnAnimatorMove`) to the character.
3. On `GenericCharacter` set `Animator` (the model's animator) and `Movement` (the `CharacterControllerMovement`). `Actor` and `Inventory` are optional and set up by the interactions and items skills.
4. On `CharacterControllerMovement` set:
   - `Pivot` — the model transform; it is rotated toward the move direction. Required.
   - `Camera` — the camera transform for camera-relative input; leave empty for world-relative input.
   - `SpeedRunning` (default 5), `SpeedSprinting` (default 10), `JumpHeight` (1.2), `Gravity` (-15).
   - `DisableRootMotion` — check it if the animator's clips contain root motion you do not want applied.
   - `GroundingSphere` (+ `GroundingSphereRadius` 0.25, `GroundingSphereLayers`) if `CharacterController` grounding is jittery.
5. Add `PlayerInput` with an Input Actions asset, `Behavior = Invoke Unity Events`, and bind events to the movement's public callbacks: `OnMove` (Vector2), `OnSprint` (button), `OnJump` (button), `OnRotate` (Vector2, for aim/look).
6. To drive animator parameters from movement state, add `AdventureCore.AnimatorMovementSetter`, assign `Animator` and `Movement`, and fill only the parameter names your controller has (`SpeedFactorForwardParameter`, `IsGroundedParameter`, `IsMovingParameter`, `IsSprintingParameter`, etc.).

**Expected result:** The character moves, jumps (if bound) and turns; animator parameters update if `AnimatorMovementSetter` is used.

### Workflow: Create a NavMesh NPC that patrols

**Goal:** An NPC that walks between waypoints on a NavMesh.

**Steps:**
1. Bake a NavMesh: add `Unity.AI.Navigation.NavMeshSurface` to the level root and call `BuildNavMesh()` (or click Bake). AAK 1.10 uses NavMeshSurface for its demos.
2. Create the NPC root: add `UnityEngine.AI.NavMeshAgent`, `AdventureCore.GenericCharacter`, and one movement:
   - `NavMeshAgentMovement` — plain agent steering.
   - `NavMeshSteerMovement` — rotates toward the target first (`RotationSpeed` 10, `RotationTolerance` 5).
   - `NavMeshTankMovement` — rotates in place before moving (`RotationSpeed` 2, `RotationTolerance` 1).
   Set `GenericCharacter.Movement` to it. Add the model + `AnimatorProxy` as for the player.
3. Add an actor: a child GameObject `Actor` with `AdventureCore.MinimalCharacterActor`, `Character` = the NPC's character; set `GenericCharacter.Actor` to it.
4. Add a child of `Actor` named `Patrol` with `AdventureCore.NavPatrolAction`: `Positions` = waypoint transforms, `Speed` (for example 2), `Owner` = the actor. `Movement` may be left empty (it falls back to the character's movement).
5. Start the action when the scene starts. Either from code: `npc.GetComponentInChildren<AdventureCore.CharacterActorBase>().StartAction("Patrol");` (actors find child actions by GameObject name), or without code: add `AdventureCore.BehaviourEventTrigger` and bind its `Started` event to `NavPatrolAction.StartAction()`.

Alternatives: `NavApproachAction` (`Target`, `Distance`, `Speed`, `Align`) walks to a target; `WanderAction` (`Timeout` 3s) walks in random directions. A `NavMeshAgentMovement` can also be driven from code with `StartApproach(destination, finishedCallback, distance, speed)` / `StopApproach()`.

**Expected result:** In Play mode the NPC walks to `Positions[0]`, then cycles through all waypoints.

### Workflow: Create a 2D character

**Goal:** A top-down or side-scrolling 2D character.

**Steps:**
1. Root GameObject with `Rigidbody2D`, a 2D collider, `GenericCharacter`, and either `AdventureCore.TopDown2DMovement` or `AdventureCore.SideScroller2DMovement` (both require `Rigidbody2D`). Set `GenericCharacter.Movement`.
2. Set `Pivot` to the sprite/visual transform and `RotationMode` (`Mirror` flips it; top-down also offers `Rotate`). Set `Environment` to the layers that count as ground/walls (default layer mask 1).
3. Speeds: `SpeedRunning` 5, `SpeedSprinting` 10; side scroller adds `SpeedAir` 5, `SpeedClimbing` 3, `JumpSpeed` 15, `Gravity` -50.
4. Bind `PlayerInput` events to `OnMove`, `OnSprint`, and for the side scroller `OnJump`.
5. For a complete working example, open `Assets/SoftLeitner/AdventureIsle/Scenes/IsleOverworld.unity` (top-down) and inspect its player.

**Expected result:** The character moves in 2D and collides with the `Environment` layers.

### Workflow: Change movement with character instructions

**Goal:** Temporarily slow, stop, or otherwise modify a character (e.g. while standing in mud).

**Steps:**
1. Instructions are serializable classes deriving from `AdventureCore.CharacterInstructionBase` (stored with `[SerializeReference]`). They are added to a character and removed again, and the character recomputes its state. Common ones: `MovementSpeedMultiplierInstruction` (`Value`), `MovementSpeedMaximumInstruction`, `MovementSpeedMinimumInstruction`, `SuspendMovementInstruction`, `SuspendMovementControlInstruction`, `SuspendControlInstruction`, `AnimatorBoolInstruction`.
2. Zone-based (Getting Started step 2): add `AdventureCore.GenericTriggerItem` and a trigger `CapsuleCollider` to the character root (set its `Character` field). Create the zone object with a trigger collider and `AdventureCore.GenericTriggerArea`, then add a `MovementSpeedMultiplierInstruction` with `Value = 0.5` to its `Instructions` list in the Inspector. From code:

```csharp
area.Instructions = new AdventureCore.CharacterInstructionBase[]
{
    new AdventureCore.MovementSpeedMultiplierInstruction { Value = 0.5f }
};
UnityEditor.EditorUtility.SetDirty(area);
```

3. Permanent: add instructions to `CharacterBase.Instructions` (applied on Start).
4. Runtime: `character.AddInstruction(instruction)` / `character.RemoveInstruction(instruction)` using the same instance for both calls.

**Expected result:** While the character is inside the area, `character.MovementBase.SpeedMultiplier` is `0.5`; it returns to `1` on exit.

## Verification

- The character root has a `CharacterBase` subclass and `character.MovementBase != null` (i.e. `Movement` assigned).
- `CharacterControllerMovement.Pivot != null`.
- If a model is used: `character.Animator != null` and the model has `AnimatorProxy` with `Character == character`.
- `PlayerInput.notificationBehavior == PlayerNotifications.InvokeUnityEvents` and the `Move` action event has a persistent listener targeting the movement (`moveEvent.GetPersistentMethodName(0) == "OnMove"`).
- In Play mode after simulated input (`movement.OnMove(Vector2.up)` for a few frames), `character.transform.position` changes and `movement.IsMoving` is true.
- NPC: `NavPatrolAction.IsHappening == true` in Play mode and the NPC's position changes over time.
- Console contains no `NullReferenceException` from `CharacterControllerMovement` or `MovementBase`.

## API quick reference

| Entry point | Type | What it does |
|---|---|---|
| `AdventureCore.GenericCharacter` | MonoBehaviour | Standard character; fields `Actor`, `Movement`, `Inventory`, `Animator`, pools |
| `AdventureCore.CharacterBase.AddInstruction(CharacterInstructionBase)` / `RemoveInstruction` | Method | Apply/remove a temporary modification |
| `AdventureCore.CharacterBase.GetCharacter(string id)` | Static method | Find a character by `Id` (or persister key) |
| `AdventureCore.CharacterControllerMovement` | MonoBehaviour | 3D player movement on a `CharacterController`; `OnMove`, `OnSprint`, `OnJump`, `OnRotate` |
| `AdventureCore.NavMeshAgentMovement` (+ `NavMeshSteerMovement`, `NavMeshTankMovement`) | MonoBehaviour | NavMesh movement; `StartApproach`, `StopApproach` |
| `AdventureCore.TopDown2DMovement`, `SideScroller2DMovement` | MonoBehaviour | 2D movements on `Rigidbody2D` |
| `AdventureCore.MovementBase.TeleportCharacter(Vector3, Quaternion)` | Method | Instantly move a character |
| `AdventureCore.AnimatorProxy` | MonoBehaviour | Forwards animator events and root motion to the character |
| `AdventureCore.AnimatorMovementSetter` | MonoBehaviour | Writes movement state into animator parameters |
| `AdventureCore.NavPatrolAction`, `NavApproachAction`, `WanderAction` | CharacterActionBase | NPC movement behaviours |
| `AdventureCore.MinimalCharacterActor` | CharacterActorBase | Simplest actor; starts an action only if none is running or the current one can end |

## Common issues

- **Symptom:** `NullReferenceException` in `CharacterControllerMovement` on Play. **Cause:** `Pivot` not assigned. **Fix:** Set `Pivot` to the model transform (or the root if there is no model).
- **Symptom:** Character doesn't move; no errors. **Cause:** `PlayerInput` not set to `Invoke Unity Events`, or the `Move` event has no listener, or `GenericCharacter.Movement` is empty. **Fix:** Check the three, as in Verification.
- **Symptom:** Character moves in world directions instead of relative to the camera. **Cause:** `CharacterControllerMovement.Camera` empty. **Fix:** Assign the main camera's transform.
- **Symptom:** Animator events cause "has no receiver" errors or actions never finish. **Cause:** Missing `AnimatorProxy` on the GameObject that has the `Animator`, or its `Character` field is empty. **Fix:** Add/assign it.
- **Symptom:** Character slides or jitters strangely with animations. **Cause:** Root motion from clips combined with code movement. **Fix:** Check `DisableRootMotion`, or use `MovementMotionMultiplierInstruction` to scale root motion.
- **Symptom:** Grounding flickers on slopes/steps. **Cause:** `CharacterController.isGrounded` jitter. **Fix:** Enable `GroundingSphere` and set `GroundingSphereLayers`.
- **Symptom:** NPC doesn't move. **Cause:** No baked NavMesh, the agent is off the mesh, or the patrol action was never started. **Fix:** Bake the `NavMeshSurface`, place the NPC on it, start the action by name from its actor.
- **Symptom:** Slow zone has no effect. **Cause:** Character lacks a `GenericTriggerItem` on a GameObject with a collider (or on its Rigidbody), or `Key` values on area and item don't match. **Fix:** Add the trigger item next to a collider; leave `Key` empty on both or make them match.

## Boundaries

- AAK ships no general-purpose character models or animator controllers for your game; use your own, or adapt the demo characters (GSRobo, Hero, Souls).
- Movement components don't do pathfinding beyond Unity's NavMesh. For flying or swimming you need a custom `MovementBase` subclass.
- Climbing exists only in `SideScroller2DMovement` (climb areas); 3D climbing/vaulting is not built in.
- Camera control (lock-on, free look) is handled by separate components (`LockableCameraFreeLook`, `LockOnManager`) built on Cinemachine; see https://adventure.softleitner.com/manual/souls-player.
- Behaviour trees are not included; NPC logic in the demos uses actions plus Visual Scripting state machines.
