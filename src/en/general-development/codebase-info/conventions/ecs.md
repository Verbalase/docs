# Entity-Component-System Conventions

## Prototypes

### Prototype data-fields
Don't cache prototypes, use prototypeManager to index them when they are needed. You can store them by their ID. When using data-fields that involve prototype ID strings, use ProtoId<T>. For example, a data-field for a list of prototype IDs should use something like:
```csharp
[DataField]
public List<ProtoId<ExamplePrototype>> ExampleTypes = new();
```

### EntityUid in Logs
When using `EntityUid` in admin logs, use the `IEntityManager.ToPrettyString(EntityUid)` method.

<details>
  <summary>Admin log with entities example (click to expand)</summary>

```csharp
// If you're in an entity system...
_adminLogs.Add(LogType.MyLog, LogImpact.Medium, $"{ToPrettyString(uid)} did something!");

// If you're not in an entity system...
_adminLogs.Add(LogType.MyLog, LogImpact.Medium, $"{entityManager.ToPrettyString(uid)} did something!");
```

</details>

### Optional Entities
If you need to pass "optional" entities around, you should use a nullable `EntityUid` for this.
Never use `EntityUid.Invalid` to denote the absence of `EntityUid`, always use `null` and nullability so we have compile-time checks.
e.g. `EntityUid? uid`

## Components

### Component data access modifiers
All data in components should be public.

### Component property setters
You may not have setters with any logic whatsoever in properties. Instead, you should create a setter method in your entity system, and apply the `[Friend(...)]` attribute to the component so only that system can modify it.
Your component may use properties with setter logic for *ViewVariables integration* (until we have a better system for that).

### Component access restrictions
The `[Access(...)]` attribute allows you to specify which types can read or modify data in your class, while prohibiting every other type from modifying it.

Components should specify their access restrictions whenever possible, usually only allowing the entity systems that wrap them to modify their data.

### Shared Component inheritance
If a shared component is inherited by server and client-side counterparts, it should be marked as *abstract*.

## Entity Systems

### Game logic
Game logic should *always* go in entity systems, not components.
Components should *only* hold data.

### Proxy Methods
When possible, try using the `EntitySystem` [proxy methods](https://github.com/space-wizards/RobustToolbox/blob/master/Robust.Shared/GameObjects/EntitySystem.Proxy.cs) instead of using the `EntityManager` property.

<details>
  <summary>Examples (click to expand)</summary>

```csharp
// Without proxy methods - bad
EntityManager.GetComponent<MetaDataComponent>(uid).EntityName;

// With proxy methods - good
Name(uid);

// Without proxy methods - bad
EntityManager.GetComponent<TransformComponent>(uid).Coordinates;

// With proxy methods - good
Transform(uid).Coordinates;
```

</details>

### Update loops
A lot of old code is accumulating frametime inside update loops to decide when to next run it.

Accumulator example (bad):
```csharp
  public override void Update(float frameTime)
  {
    var query = EntityQueryEnumerator<UpdateLoopExampleComponent>();
    while (query.MoveNext(out var uid, out var comp))
    {
      comp.Accumulator += frameTime;

      if (comp.Accumulator < UpdateInterval) 
          continue;

      comp.Accumulator -= UpdateInterval;
      
      // Code here
    }
```

This is bad because of those reasons:
1. This makes the update loop impossible to synchronize between server and client, causing prediction and networking issues.
2. This approach uses the `float` type, and it is not precise enough in case of update loops, so it may cause rounding issues when the game is launched for a long time.
3. This constantly does the addition operation, which isn't bad on its own, but when there are hundreds of systems doing that the overhead can beocme noticeable.

All of the above problems can be fixed by using `TimeSpan` type and `IGameTiming`.

TimeSpan example (good):
```csharp
    [Dependency] private readonly IGameTiming _timing = default!;

    public override void Initialize()
    {
        SubscribeLocalEvent<UpdateLoopExampleComponent, MapInitEvent>(OnMapInit)
    }
    
    private void OnMapInit(Entity<UpdateLoopExampleComponent> ent, ref MapInitEvent args)
    {
        // Set the first update time after the entity is spawned.
        // Without this it would update every single tick until NextUpdate catches up with the server time.
        ent.Comp.NextUpdate = _timing.CurTime + ent.Comp.UpdateInterval;
        Dirty(ent);
    }

    public override void Update(float frameTime)
    {
        // CurTime is calculated so we do it only once outside the update loop instead of for every sigle entity.
        var curTime = _timing.Curtime;
        // Loop over all components, ignoring paused entities.
        var query = EntityQueryEnumerator<UpdateLoopExampleComponent>();
        while (query.MoveNext(out var uid, out var comp))
        {
            if (comp.NextUpdate < curTime)
                continue; // Not enough time has passed since the last update.
        
            // Set the time for the next update.
            // Don't use
            // comp.NextUpdate = curTime + UpdateInterval;
            // because that eats the remainder with every update, causing the update loop to run slightly less often
            // than given by UpdateInterval, which will be imprecise and can cause problems over large time durations.
            comp.NextUpdate += UpdateInterval;
        
            // Dirty the component so that the client can reroll the NextUpdate datafield during predcition.
            // Without this you will get mispredicts.
            Dirty(uid, comp);
        
            // Do stuff here.
        }
    }
```


## Events

### Method Events vs Entity System Methods
Method Events are events that you raise when you want to perform a certain action. Example:
```csharp
// This would change the damage on the entity by 10.
RaiseLocalEvent(uid, new ChangeDamageEvent(10));
```
On the other hand, Entity System Methods are methods you call on systems to perform an action.
```csharp
// This would change the damage on the entity by 10.
EntitySystem.Get<DamageableSystem>().ChangeDamage(uid, 10);
```

Method Events are *prohibited*, always use Entity System Methods instead.
There's an exception to this, however.

You may use Method Events as long as they're wrapped by an Entity System Method.
In the example above, this would mean that `DamageableSystem.ChangeDamage()` would internally raise the `ChangeDamageEvent`, which would then by handled by any subscriptors...

```admonish info
Ensure events are unsubscribed from when systems are shutdown. Proxy methods like `Subs.CVar()`or `SubscribeLocalEvent` already take care of it, note that you do not need to unsubscribed inside managers, as their lifetime ensures that when they shutdown, the rest of the client / server is also shutting down, making unsubscribing not necessary.
```

### Event naming
- Always suffix your events with `Event`.
  Example: `DamagedEvent`, `AnchorAttemptEvent`...

- Always name your event handler like this: `OnXEvent`
  Example: `OnDamagedEvent`, `OnAnchorAttemptEvent`...

### Struct by-ref events
Events should always be structs, not classes, and should always be raised by ref. If possible it should also be readonly if applicable.
They should also have the [ByRefEvent] attribute.

In practice this will look like the following:
```cs
var ev = new MyEvent();
RaiseLocalEvent(ref ev);
```
