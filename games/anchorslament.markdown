---
layout: article
title: Anchor's Lament
mode: immersive
header:
    theme: dark
article_header:
    type: overlay
    theme: dark
    background_color: '#79a'
    background_image:
        gradient: 'linear-gradient(135deg, #a974, #110a)'
        src: /assets/images/anchorslament/preview_banner.gif
show_title: true
permalink: /anchorslament/
---
<link rel="stylesheet" href="/assets/css/index.css">

<p class="gameinfo">
<b>Company:</b> Imperial Playgrounds
<br>
<b>Status:</b> In development (unreleased)
<br>
<b>Engine:</b> Unity
<br>
<b>Team size:</b> 1 programmer (me), 1 designer, various artists
<br>
<b>My participation:</b> 2 months, from project start (June - August 2025)
<br>
<b>My role:</b> Programming (gameplay, system structure, tooling, database & network)
</p>

Anchor's Lament is an unreleased Mobile/PC game by Imperial Playgrounds where I worked as the **main programmer** for 2 months during the project start in summer 2025. My job was to **get as much of the game framework up and running as possible** before I had to get back to Yrgo in August, at which point new programmers would be assigned to continue where I left off.

The team had multiple artists and a game designer/director that also implemented content on the side using my frameworks as they were created.

# My work

### Overview
- Implementation of **asynchronous combat** against other players
  - This included working with database and network code (specifically using Supabase), auth/account creation, requesting and uploading data, basic security rules and more.
- Full, modular **gameplay loop and combat system**
- Implementation of content and features in discussion with the game designer
- Editor tooling for future developers

I was responsible for the entire process from empty Unity template to working game, with intent on the development being eventually continued by other programmers after I've left.

My job did not include UX design or UI polish.

## Modular fish action system
The possibly most important system I've created for this game is the **fish action system**.

Combat in this game is executed between **two grids of fishes**. Outside of combat, players buy fishes that they can slot into their 4x2 grid - most take up one slot, but large ones can take up multiple slots. Once the player has a grid of fishes (with equipment and/or upgrades), they enter combat, and another player's team is loaded from the database.

Inside combat, an anchor rhythmically pulses on each column of the grid in order, and all fishes that occupy that column are triggered, and their abilities are executed.

All the steps that lead up to this are also interesting, but I want to focus a for bit on **how the fishes' abilities are created and executed**.

### FishActions
*So, a bit of background.*

Each fish in your grid is essentially a pure C# class containing its equipment and upgrades, plus a reference to a **FishType asset**, which is a ScriptableObject that contains the definition of the fish.

Apart from specifying the name, portrait, sell value and tags of fishes, the FishType asset also contains a list of *FishActions*, which define the fish's behavior in combat.

A *FishAction* is also a type of scriptableobject, but instead of being a data container, it **contains the execution logic for a concrete "ability"** that can be assigned to a fish. These can be set to execute at a specific moment (such as at the beginning of combat), but it's usually when the fish is activated by the anchor.

When a fish is activated by the anchor, for example, the fish takes all its FishActions that should execute on the anchor beat, and calls `Execute()` on them. **Creating classes that derive from FishAction** allows them to override the abstract `Execute()` method with its own logic to create new types of action. These subclasses can then be created as assets (since they are ScriptableObjects) and their values can be edited to create variants of that action that can then be assigned to various FishTypes.

Examples of fish actions are "dealing damage", "poisoning a random enemy fish", "activating the fish in the slot below in the grid", or more complex ones like "healing the team when an adjacent fish becomes poisoned", or "executing another list of actions if the fish is activated when the team has low health".
{:.info}

**However, this simple approach had a limitation** - if two fishes needed to deal different amounts of damage, then you would have to create two separate instances of the Damage fishaction scriptableobject, specify different values for each of them, and then assign the correct one to the correct fish. This would quickly become an unmaintainable mess for any serious amount of fishes, and I knew this system could be improved.

**So I had an idea.**

### Instantiating FishActions

What if each FishType contained not just a list of references to FishAction assets, but also **specific overrides for the values of each FishAction**, similar to how prefab overrides work?

So I replaced the list of FishActions in each FishType with a list of a new custom serializable class, *ActionInstance*, that contained a reference to a FishAction asset, and **a list of override values to apply to that action's fields**. This new class is serialized inside the FishType itself, so the value overrides are directly associated with that specific FishType.

Then, when the fish enters combat, it tells each of its ActionInstances to *instantiate* its FishAction, **apply the field overrides using reflection**, and return the new instantiated FishAction.

Instantiating a ScriptableObject at runtime creates a temporary instance that can be accessed like any other. They should, however, be Destroy():ed once no longer needed (in this case when combat is over).
{:.warning}

This way, in combat, each fish's FishActions are unique instances that **belong to that fish only**. All values on that FishAction can be edited freely without affecting any other fish, allowing them to store data between activations, *per-fish*, which is incredibly useful for more complicated actions.

### Editing FishAction settings in the inspector
The remaining step was to actually **allow designers to edit these overrides in the editor**.

All fields in a FishAction should obviously not be exposed to designers - instead, I wanted the creator of the FishAction to specify which fields should be exposed to designers. I achieved this by creating a custom C# attribute, `[ActionSetting]` - putting this attribute on a field, public or not, will expose it to designers in the editor.

I then created a custom PropertyDrawer for the ActionInstance class that renders a slot where you can assign a FishAction, and if it has a FishAction assigned, it takes that action, iterates through its fields using reflection, and generates editable areas for all fields marked with `[ActionSetting]`.

<div class="center">
<img src="/assets/images/anchorslament/actionsettingattribute.png" title="An example of using the ActionSetting attribute. The field in the inspector is generated based on the data type of the original field. The value can also be changed even if it's marked as readonly, as the value can be set using reflection regardless.">
<p class="imagedesc"><i>Fields marked with </i>[ActionSetting(name, description)]<i> appear as editable in the inspector.<br>If a description is specified, it appears when you hover over the field.</i></p>
</div>



Here's an example of implementing a FishAction in practice:

```cs
[CreateAssetMenu(menuName = "Anchor's Lament/Fish Actions/Basic Damage", order = 100)]
// FishAction itself inherits from ScriptableObject
public class FA_Damage : FishAction
{
    // Makes these variables editable on each per-fish instance of the action.
    [ActionSetting("Damage to deal")] TieredInt DamageValues;
    [ActionSetting("Another setting")] readonly string AnotherSetting;
    
    // Serialized fields are editable on the scriptableobject definitions but not per-fish
    [SerializeField] float ExampleFloat;

    protected override void Execute(ActionParameters action)
    {
        // ActionParameters contains data about the context of the action, such as involved
        // fishes, and the level (1-4) to execute the action at (usually being the fish's level).

        // TieredInt (from TieredValue<T>) is a class with separate values for each level of the action (1-4).
        float damage = DamageValues[action.TierInt];

        // Any script can attach custom event data to our ActionParameters.

        if (action.HasEventData(out PendingAttack attack))
            attack.DamageToDeal += damage;
        else
            MyFish.PerformAttackWithPopup((int)damage); // MyFish refers to the fish that this action instance belongs to
           
        // Notifies listeners that the fish attacked, so other fishes can react if they want to.
        action.CallbackFishDidThing(EFishActionCallback.ATTACK, damage);
    }

    public override string GetDescription() => $"Deals damage to the enemy team.";
}
```

And this is how it turns out:

<div class="left">
<img src="/assets/images/anchorslament/damageactionasset.png" title="Note how the serializable ExampleFloat field appears in the inspector. This is handy, as multiple assets can be created from the same FishAction class, with varying settings and varying names, for when it's convenient to share the same code for multiple behaviors.">
<p class="imagedesc center"><i>The scriptableobject asset created from FA_Damage.<br>This is then assigned to fishes.</i></p>
</div>
<div class="right">
<img src="/assets/images/anchorslament/damageaction.png" title="Note how the ExampleFloat is not visible here as it is not marked with [ActionSetting]. The input area for each field is generated based on the data type of the field. These can be made custom for certain data types if necessary too - just like the TieredInt and other TieredValue<T>s are. The reason the script uses TieredInt and not TieredValue<int> is that unity can only serialize inflated generics. TieredInt is simply an empty class inheriting from TieredValue<int>.">
<p class="imagedesc center"><i>How the action appears when editing it on a fish.<br>Note how the TieredInt field is split into 4 boxes for each action level.</i></p>
</div>

<br><br><br><br><br><br><br><br><br><br><br>
### Extras

There's more features I've baked into this system.
- FishActions can bind methods to events to **receive callbacks**, allowing them to be reactive
- FishActions can contain **nested FishActions** that they can execute at will
  - This is done using the SubAction class, which is essentially a container for a list of FishActions.
  - These can of course be assigned in the inspector, creating a nested menu of ActionInstances
  - This allows the creation of really interesting conditional FishActions, pretty much **allowing you to program using prebuilt FishAction blocks**. An example would be an action that runs one set of actions if the team's health is below 50%, and another set if above.
- ActionSettings can be marked as *advanced*, hiding them unless you enable a toggle, which can remove clutter for less-commonly-used settings.

Below is an example using all of these:

```cs
public class FA_WhenAdjacentFishPerformsAction : FishAction
{
    [ActionSetting(desc: "Which slots count as adjacent?", isAdvanced: true)] EAdjacentMode adjacentMode;
    [ActionSetting] SubAction subAction;
    [ActionSetting] EFishActionCallback actionTypeToListenTo;

    // Called when the FishAction is instantiated and added to a fish
    public override void Initialize(TeamInstance team, FishInstance fish)
    {
        base.Initialize(team, fish);
        foreach (AdjacentPos adjacentPos in fish.GetAdjacentSlots(adjacentMode))
        {
            // Binds a function to when something happens in an adjacent slot
            team.BindSlotEvent(adjacentPos, (type, value) => OnAdjacentSlotAction(type, value, adjacentPos.Source));
        }
    }

    public void OnAdjacentSlotAction(EFishActionCallback actionType, float value, Vector2Int closestFishPos)
    {
        // Creates the parameters to send to the subactions.
        // ActionExecutionMoment is set to TRIGGER so it knows it has been executed from a custom callback.
        ActionParameters subActionParameters = new(ActionExecutionMoment.TRIGGER, MyFish, MyFish.Tier, closestFishPos);

        // If this callback matches any of the ones we want to listen to. NONE will listen to all.
        if ((actionTypeToListenTo | actionType) != EFishActionCallback.NONE || actionTypeToListenTo == EFishActionCallback.NONE)
        {
            // Executes the nested action(s) (if any)
            subAction.Execute(subActionParameters);
        }
    }

    protected override void Execute(ActionParameters action) { }
    public override string GetDescription() => "Run a subaction when an adjacent fish performs a specific action";
}
```
<br>

# Draggable UI
Since this game is designed with mobile in mind, a lot of the UI was going to be qutie drag and drop-centric. There was going to be item slots, sell zones, organizing items in menus and what not. Knowing this, I wanted to prepare for the inevitable mess of having tons of different draggable classes by creating an unified system for it in advance.

My plan was to utilize a generic base class `DraggableItem<T>` for draggable items, and while it is dragged, it will raycast below it to check for UI that implements `IDragDropListener<T>` of the same type `T`.

This is how I set up the classes:

```cs
public abstract class DraggableItem : MonoBehaviour, IDragHandler, IBeginDragHandler, IEndDragHandler
{
    // Code for picking up, following cursor, etc.
}

public abstract class DraggableItem<T> : DraggableItem
{
    [SerializeField] protected T MyData;

    public override void OnDrag(PointerEventData eventData)
    {
        // Raycast for UI elements of type IDragDropTarget<T> and notify them about events
    }
}

public interface IDragDropTarget { }

public interface IDragDropTarget<in T> : IDragDropTarget
{
    public void OnDraggableDrop(T data, DraggableItem source);
    public void OnDraggableHoverStart(T data, DraggableItem source);
    public void OnDraggableHover(T data, DraggableItem source) {}
    public void OnDraggableHoverEnd(T data, DraggableItem source);
}
```

Now, this was already incredibly convenient as **any UI element** (or GameObject too if it has a collider and the camera is given a PhysicsRaycaster component) below a dragged item will be notified when a hover starts, is ongoing, ends, or when it is dropped into.

`DraggableItem` works great for draggable UI with no data associated with it, and it ONLY contains the dragging code itself (no notifying listeners here), making this class easily extendable for other uses.
<br>
`DraggableItem<T>` works great for draggable UI with data associated with it, for when listeners need to know *what's* being dragged above/onto them.
{:.success}

I also needed to make item slots that can have items **both inserted into them AND taken out of them**. It should also support swapping the contents of two slots by dragging one above another while both are full.

To solve this, I made a class that derives from `DraggableItem<T>` and ALSO implements the `IDragDropTarget<T>` of the same type `T`.

```cs
public abstract class ItemSlot<T> : DraggableItem<T>, IDragDropTarget<T>, IItemSlot<T>
{
    // Implements the interfaces, mostly
}

public interface IItemSlot
{
    public bool IsEmpty();
    public void ClearValue();
    public bool TryInsert(dynamic insertion);
    public Vector2 GetFramePos();
}
public interface IItemSlot<out T> : IItemSlot
{
    public T GetValue(out bool success);
    public T TrySwapWith(dynamic insertion, out bool success);
}
```

Now, you may notice the interfaces look a bit weird. `dynamic insertion`? `<out T>` and `<in T>`? What's going on?

Let me explain.

### Covariance and contravariance

*Covariance and contravariance in generics* was a concept that I stumbled upon while trying to solve a very particular problem with this system. For those who don't know what covariance/contravariance is or what it even means, let me explain it to you by walking you through the same process I went through:

After creating the `ItemSlot<T>` class, I realized some slots may need to contain multiple types of data, and not be restricted to a single data type. Inventory slots, for example, should be able to contain data of types `Fish` and `Consumable` and `Equipment` and probably even more in the future.

So how do I solve this? I can't exactly add multiple ItemSlot<T> scripts to the same object - then the slot would be able to contain two types of items at the same time.

I then had a flash of inspiration - wouldn't simple polymorphism solve this easily?

So I went ahead and created a new interface, `IInventoryItem`, with one function for getting the icon of the item. I then implemented this interface in the `Fish`, `Consumable` and `Equipment` classes, and created a new class `InventorySlot : ItemSlot<IInventoryItem>`.

Since those 3 classes now all implement `IInventoryItem`, a slot that contains `IInventoryItem`s should be able to contain all 3 data types now, no?

The answer is, yes. But also no.

#### The problem
I noticed that the new slot no longer sent nor received any callbacks on hover/drop. This was... unexpected, and after lots of research and wrapping my head around why, I discovered the reason:

I expected that polymorphism would apply inside generics; that `IDragDropTarget<Fish>` could be converted into `IDragDropTarget<IInventoryItem>`, since `Fish` could be converted into `IInventoryItem`. But this is not the case.

By default, generic type parameters do not exhibit any polymorphism. But they can be made to, using the `in` or `out` keywords in their definition, like I did with `IDragDropTarget<in T>`. `in` means that derived classes of `T` can be *inserted* into the functions defined in the interface, but then neither `T` or derived classes can be *returned* from them. This is *contravariance*, and this rule allows `IInterface<BaseClass>` to be casted to `IInterface<DerivedClass>`. `out` means the opposite; `T` and derivated classes can be *returned*, but never *inserted*, and this is called *covariance*, instead allowing `IInterface<DerivedClass>` to be converted to `IInterface<BaseClass>`.

Here's an example.

```cs
// Covariance (T is OUTputted)
IEnumerable<string> strings = new List<string>();  
IEnumerable<object> objects = strings; // IEnumerable<string> is casted to IEnumerable<object>

// Contravariance (T is INserted)
Action<object> objectAction;
Action<string> stringAction = objectAction; // Action<object> is casted to Action<string>
```
#### The solution
With this knowledge, I realized that `IDragDropTarget<T>` only contains functions that *receive* data of type `T`, and `IItemSlot<T>` only really needs to *output* data of type `T` - IF the functions that are used to set/swap the data of the slot have a `dynamic` input type, which is completely fine as it can be type checked easily and fail if an incompatible item is inserted into the slot.

#### The result
Now, with the interfaces being variant, UI elements with `IDragDropTarget<IInventoryItem>` receive events even when a `DraggableItem<Fish>` is dropped onto them, and `DraggableItem<Fish>` can, in turn, insert a `Fish` into an `IItemSlot<IInventoryItem>` just fine.

This system has been the backbone of almost all UI in the game, and has been a powerful time saver for all drag-and-drop functionality, such as sell slots, equipping items onto fishes, adjusting your loadout, dragging rewards of various types into your inventory, and so on.

<br>

# Other work
I've done so much more on this project than this, but much of it isn't really worth discussing in any detail. Honorable mentions, though, are the system for editing the loadout grid, which intelligently merges slots when you insert a fish larger than 1x1, the database integration (albeit quite run-of-the-mill), all the developer cheats (which allow you to, for example, select assets in the asset list and add them to your inventory with a hotkey), the convenient integration of equipment abilities for fishes (which lets you assign FishActions to an equipment piece, and these are then added to the fish's actions in combat), and more.

Overall, this project has been a ton of fun and I've learned an incredible amount during it.

Since the game is unreleased and still under development, I sadly cannot post any link so you can check it out for yourself. But maybe in the future.

#### Legal
I have received full written permission by Imperial Playgrounds to describe my contributions to this project within this portfolio.

<br>

<br>

[Back to the main page](/){:.button.button--outline-success.button--rounded.button--xl}
{:.center}

<br>
