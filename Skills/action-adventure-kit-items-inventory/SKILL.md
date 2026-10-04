---
name: action-adventure-kit-items-inventory
description: "Use this skill whenever the user wants items, an inventory, or equipment with Action Adventure Kit — e.g. 'add an inventory to my player', 'create a coin item that stacks', 'make a health potion i can drink', 'add a sword the player can equip', 'grid inventory like resident evil', 'limit how many arrows i can carry', 'show the inventory on screen', 'items disappear after loading'. Covers item assets (GenericItem, UsableItem, EquipmentItem, PrefabItem, PlaceholderItem, CapacityRaisingItem), ItemSet, ListedInventory and TiledInventory, slots and equipping, capacities, and inventory UI text. Do NOT use for the pickup interaction (see action-adventure-kit-actions-interactions), attributes and stats (see action-adventure-kit-health-stats-damage), or persistence setup (see action-adventure-kit-save-load). When in doubt whether an item, loot or inventory request could involve Action Adventure Kit, use this skill — Prerequisites shows how to confirm the asset is installed."
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

# Build Items and Inventories with Action Adventure Kit

AAK items are ScriptableObject assets deriving from `AdventureCore.ItemBase`; inventories are components deriving from `AdventureCore.InventoryBase` that hold `InventoryItem` entries (item + quantity + optional values) and manage equipment slots. This skill lets an agent create item assets, give a character an inventory, equip items to slots, limit capacities, and show the inventory in UI.

## When to use this skill

- "give my player an inventory"
- "create a stackable coin item and an item set"
- "make a potion that heals 20 hp when used"
- "add a sword the character can equip and that appears in its hand"
- "i want a tetris style grid inventory"
- "player can carry max 30 arrows, quiver upgrade raises it to 60"
- "a money bag should give 10 coins"
- "show what's in the inventory as text"

Not for:
- Picking items up from the world, buying them, item costs on doors → `action-adventure-kit-actions-interactions`
- Defining attributes/stats/resources that equipment or potions modify → `action-adventure-kit-health-stats-damage`
- Effects added by equipment/potions → `action-adventure-kit-status-effects`
- Making the inventory save → `action-adventure-kit-save-load`

## Prerequisites

- Action Adventure Kit 1.10.1+ (`AdventureCore`), Unity 6000.0.76+.
- A character set up with `GenericCharacter` (see `action-adventure-kit-character-movement`).
- **Install check:** `System.Type.GetType("AdventureCore.ListedInventory, AdventureCore") != null` and the menu `Assets > Create > Adventure > Items > GenericItem` exists. If not, tell the user to import Action Adventure Kit from https://assetstore.unity.com/packages/templates/systems/action-adventure-kit-217284 and stop.

## Quick start

Create a stackable item, an item set, and a listed inventory on an existing character (Getting Started step 4). Run from an Editor script:

```csharp
using AdventureCore;
using UnityEditor;
using UnityEditor.SceneManagement;
using UnityEngine;

public static class AakCreateInventory
{
    public static void Run(GenericCharacter character, string folder = "Assets/Game/Items")
    {
        if (!AssetDatabase.IsValidFolder(folder))
        {
            System.IO.Directory.CreateDirectory(folder);
            AssetDatabase.Refresh();
        }

        var coin = ScriptableObject.CreateInstance<GenericItem>();
        coin.Key = "COIN";          // unique, used by persistence
        coin.Name = "Coin";
        coin.Stackable = true;
        AssetDatabase.CreateAsset(coin, folder + "/Coin.asset");

        var set = ScriptableObject.CreateInstance<ItemSet>();
        set.Items = new ItemBase[] { coin };
        AssetDatabase.CreateAsset(set, folder + "/Items.asset");
        AssetDatabase.SaveAssets();

        var go = new GameObject("Inventory");
        go.transform.SetParent(character.transform, false);
        var inventory = go.AddComponent<ListedInventory>();
        inventory.ItemSet = set;
        inventory.Items.Add(new InventoryItem(coin, 5));   // starting items

        character.Inventory = inventory;
        EditorUtility.SetDirty(character);
        EditorSceneManager.MarkSceneDirty(character.gameObject.scene);
    }
}
```

1. Select or find the character (`Object.FindFirstObjectByType<AdventureCore.GenericCharacter>()`).
2. Run `AakCreateInventory.Run(character)`.
3. Expected result: `Assets/Game/Items/Coin.asset` and `Items.asset` exist; the character has a child `Inventory` with `ListedInventory` containing `Coin x5`; `GenericCharacter.Inventory` references it. In Play mode `character.InventoryBase.GetQuantity(coin) == 5`.

## Workflows

### Workflow: Create item assets

**Goal:** Define the items of the game.

**Steps:**
1. Use `Assets > Create > Adventure > Items > <Type>`, or `ScriptableObject.CreateInstance<T>()` + `AssetDatabase.CreateAsset` as in the Quick start.
2. Choose the type:

| Type | Use for | Key fields |
|---|---|---|
| `GenericItem` | Keys, materials, quest items, currency | `Stackable` |
| `UsableItem` | Potions, food, consumables | `Attributes`, `Resources` (ResourceQuantity[]), `Effects`, `InstantUse`; stackable, usable, equippable |
| `EquipmentItem` | Weapons, armour, rings | `Prefab`, `AttributeRequirements`, `StatRequirements`, `AttributeValues`, `StatValues`, `Effects`, `Instructions` |
| `PrefabItem` | Equippable item that just spawns a prefab | `Prefab` |
| `PlaceholderItem` | Money bag (adds 10 coins), heart pieces (4 → 1 container) | `Content` (ItemQuantity), `PrerequisiteCount` |
| `CapacityRaisingItem` | Bigger quiver/wallet | `Item`, `Capacity` |

3. Fill the common `ItemBase` fields: `Key` (unique, **required for persistence**), `Name`, `Description`, `Image`, `Visual`, `Category` (an `ItemCategory` for UI grouping), `Slot` (an `ItemSlotCategory` deciding which slots it fits).
4. Create categories with `Assets > Create > Adventure > ItemCategory` and `Assets > Create > Adventure > ItemSlotCategory`.

**Expected result:** Item assets exist with unique, non-empty `Key`s.

### Workflow: Register items in an ItemSet

**Goal:** Let inventories resolve items by key (needed whenever inventories are saved/loaded).

**Steps:**
1. `Assets > Create > Adventure > ItemSet`.
2. Fill `Items`, or click **Find in Folder** in its Inspector (collects every `ItemBase` asset in the set's folder and subfolders) or **Find in Resources**.
3. Assign the set to `InventoryBase.ItemSet` on every inventory.

**Expected result:** `ItemSet.Items` contains every item; at runtime `set.GetItem("COIN")` returns the asset. Duplicate keys throw `Item Set ... contains multiple objects with key:...` when the set loads.

### Workflow: Add a listed or tiled inventory

**Goal:** Give a character an inventory of the right kind.

**Steps:**
1. Create a child GameObject (e.g. `Inventory`) under the character and add:
   - `AdventureCore.ListedInventory` — a simple list; stackable items share an entry.
   - `AdventureCore.TiledInventory` — a grid; `InventorySize` (default 16×8) and `ItemSizes` (per-item `Size` for items larger than 1×1). Reference scene: `Assets/SoftLeitner/AdventureManual/Systems/Item/ManualItemTiled.unity`.
2. Set `ItemSet`, optional starting `Items`, `EquipOnAdded` (auto-equip into an empty matching slot), `UnequipOnEmpty`, and `Capacities` (List of `ItemQuantity`: max quantity per item; unlisted items are unlimited).
3. Assign the inventory to the character's `Inventory` field. The character initializes it in `Awake`.
4. Change it at runtime through the API, not by editing `Items` (the tooltip warns against manual changes during play):

```csharp
var inv = character.InventoryBase;
int notAdded = inv.AddItems(new AdventureCore.ItemQuantity(coin, 3));   // returns remainder
bool has = inv.HasItemQuantity(coin, 2);
inv.RemoveItems(new AdventureCore.ItemQuantity(coin, 2));
```

**Expected result:** `inv.GetQuantity(coin)` reflects additions/removals; `Changed` fires after each change.

### Workflow: Equip items into slots

**Goal:** Equip a sword so it appears on the character.

**Steps:**
1. Create an `ItemSlotCategory` (e.g. `Weapon`) and set it as the sword item's `Slot`. Make the sword an `EquipmentItem` or `PrefabItem` with `Prefab` set.
2. On the character add a slot component (usually on the hand bone or a child object):
   - `InstantiatingItemSlot` — instantiates the item's `Prefab` under `Parent` while equipped.
   - `ActionItemSlot` — same, plus starts an action found in the instantiated prefab (attacks); input via `Input(CallbackContext)` or `StartAction()`.
   - `UsableItemSlot` — instantiates the prefab when used (consumables); `ActMessage` default `ACT`.
   - `GenericItemSlot` — no visuals, for any `ItemBase`.
3. Set the slot's `Category` to the same `ItemSlotCategory`, `Index` (distinguishes e.g. left/right hand), and a unique `Key` (persistence).
4. Add the slot to `InventoryBase.Slots`.
5. Equip: `inv.EquipItems(inv.GetItem(sword))` (optional slot index), or pre-set `InventoryItem.EquippedSlot` on a starting item, or enable `EquipOnAdded`. Unequip with `inv.UnequipItems(inventoryItem)`.

**Expected result:** The slot's `EquippedInventoryItem` is the sword's entry; an instance of the sword prefab exists under the slot's `Parent`; `EquipmentItem` attribute/stat values and effects are applied while equipped.

### Workflow: Usable items (potions)

**Goal:** A potion that restores health.

**Steps:**
1. Create a `UsableItem` with `Resources` = `[{ Type: <health ResourceType>, Quantity: 20 }]` (resource types: see the health skill).
2. Use it from code: `inv.UseItems(inv.GetItem(potion))` (checks `CanUseItems`), or equip it to a `UsableItemSlot` and call the slot's `Use()`.
3. Set `InstantUse` for items that should be used immediately when they arrive in the inventory instead of being stored.

**Expected result:** Quantity decreases by 1 and the character's health increases by 20 (capped at its maximum).

### Workflow: Show the inventory in UI

**Goal:** Display inventory contents.

**Steps:**
1. Quick debug text: add `AdventureCore.InventoryText` to a uGUI `Text` or TextMeshPro object, set `Inventory` and `Text` or `TMPText`. It lists `Name x Quantity`.
2. Full UI: the `UI/Item` folder of AdventureCore contains panels (`ListedInventoryPanel`, `TiledInventoryPanel`, `ItemSlotPanel`, `ItemNotifications`) and UI Toolkit elements (`ListedInventoryElement`, `InventoryItemElement`, `ItemSlotElement`). Copy a configured one from the demo UI prefabs rather than building from scratch.

**Expected result:** The text shows the current items and updates when the inventory changes.

## Verification

- Every item asset has a non-empty, unique `Key`: `AssetDatabase.FindAssets("t:ItemBase")` → load each → no empty or duplicate `Key`.
- Each inventory's `ItemSet` is assigned and contains all items the inventory can hold.
- `character.InventoryBase == inventory` (i.e. `GenericCharacter.Inventory` assigned).
- Play mode: `character.InventoryBase.GetQuantity(item)` matches the expected count after adding/removing.
- Equipment: `slot.EquippedInventoryItem?.Item == sword` and the slot's `Parent` has a child instance.
- Console has no `Item Set ... contains multiple objects with key` exceptions.

## API quick reference

| Entry point | Type | What it does |
|---|---|---|
| `AdventureCore.ItemBase` (+ `GenericItem`, `UsableItem`, `EquipmentItem`, `PrefabItem`, `PlaceholderItem`, `CapacityRaisingItem`) | ScriptableObject | Item definitions |
| `AdventureCore.ItemSet` | ScriptableObject | All items, resolved by `Key` (`GetItem`, `TryGetItem`) |
| `AdventureCore.ItemQuantity(ItemBase, int)` | Class | Item + amount used by most APIs |
| `AdventureCore.ListedInventory` / `TiledInventory` | MonoBehaviour | Inventories |
| `InventoryBase.AddItems` / `RemoveItems` / `CanAddItems` / `CanRemoveItems` | Methods | Change contents (return remainder / bool) |
| `InventoryBase.GetQuantity(ItemBase)` / `HasItemQuantity` / `GetItem` | Methods | Query contents |
| `InventoryBase.EquipItems(InventoryItem, int?)` / `UnequipItems` / `UseItems` | Methods | Equipment and consumption |
| `AdventureCore.InstantiatingItemSlot`, `ActionItemSlot`, `UsableItemSlot`, `GenericItemSlot` | MonoBehaviour | Equipment slots |
| `InventoryBase.Changed`, `EquipmentChanged` | UnityEvent | React to changes |
| `AdventureCore.InventoryText` | MonoBehaviour | Debug text display |

Item values (durability, ammo, upgrades stored per entry): `InventoryBase.SetItemIntValue/GetItemIntValue` etc., see `Assets/SoftLeitner/AdventureManual/Systems/Item/ManualItemValues.unity` and https://adventure.softleitner.com/manual/item.

## Common issues

- **Symptom:** Items vanish or come back wrong after loading a save. **Cause:** Empty/duplicate item `Key`, or the item is missing from the inventory's `ItemSet`. **Fix:** Give each item a unique key and add it to the set (Find in Folder).
- **Symptom:** `NullReferenceException` when adding items via a pickup or `ItemBase.Add(character)`. **Cause:** `GenericCharacter.Inventory` not assigned. **Fix:** Assign the inventory.
- **Symptom:** Equip does nothing. **Cause:** No slot whose `Category` equals the item's `Slot` and whose item type fits (`InstantiatingItemSlot` needs a `PrefabItem`/`EquipmentItem`), or the slot isn't listed in `InventoryBase.Slots`. **Fix:** Match categories and register the slot.
- **Symptom:** Can't add more of an item. **Cause:** `Capacities` limit reached (`GetRemainingCapacity`). **Fix:** Raise the capacity or use a `CapacityRaisingItem`.
- **Symptom:** Equipment can't be equipped. **Cause:** `AttributeRequirements`/`StatRequirements` not met. **Fix:** Lower requirements or raise the character's attributes.
- **Symptom:** Two entries instead of one stacked entry. **Cause:** `GenericItem.Stackable` is off. **Fix:** Enable it.

## Boundaries

- No built-in loot tables beyond `ItemChance` helpers; random drops in the demos are scripted per game.
- No crafting system out of the box; build it on top of `CanRemoveItems`/`RemoveItems`/`AddItems`.
- No trading UI beyond `PurchaseAction` (see the interactions skill).
- Inventory UI must be styled per game; AAK provides functional panels/elements, not a final look.
- Item ScriptableObjects are shared definitions; per-instance data belongs in item values on the `InventoryItem`, never on the asset.
