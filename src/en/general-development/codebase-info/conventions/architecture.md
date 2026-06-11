# Code Architecture Conventions

## Composition > Singletons

- **Composition** is when a single component holds a single function.
- **Singleton** is when a single component holds multiple functions.

In general, the main coding principle you have to follow is to **prefer composition to singletons**.

The issue with "singleton" design is the lack of modularity. When all functionality is tied to a single component, it becomes harder to tweak the structure further.

**Always split the functionality between multiple components and systems**. The simplest way to imagine how it should work is to think about how to implement your mechanic without ever actually saying its name in C#.

<details>
    <summary>Bingle code design example</summary>

Let's take Bingles as an example for this architecture strategy.

On the base level, Bingle mechanics are simple:
- A pit spawns from a gamerule;
- This pit works just like a chasm, gets points when items or people fall in, and spawns Bingles when enough points are reached;
- Tiles spread randomly around the pit when some amount of points is reached, and living Bingles get upgraded when they drag enough items into the pit.

### The intuitive way

The most straightforward way to add Bingles is to assign a single component to the Bingle pit and a single component to Bingles, then code all necessary mechanics in a single system. That will be the fastest and easiest path, and everything will work just fine for a bit of time.

But that strategy makes it really hard to implement a new feature for bingles because of how hardcoded they are to their own structure!

### How to implement Bingles properly

Pit should use the same code as chasms, but instead of deleting the entities it should store them and add points to some generic internal counter. That requires rewriting `ChasmSystem` to be more generic, since it's hardcoded to deleting fallen entities, and adding components like `ChasmContainerComponent` that force the chasm to store the entity and `ChasmChargeComponent` that adds a charge to a chasm entity.

Then a new system that spawns entities when a certain amount of charges on an entity is reached is required, so the pit can spawn Bingles when enough items are dragged in.

A similar system for tile spreading is needed in order to randomly spawn bingle floors around the pit.

As the result, instead of just copy-pasting or hardcoding, we made a lot of generic components that can be later used for any other purposes, not just Bingles!

</details>

## In-simulation or out-of-simulation

```admonish warning
This convention is *very* poorly enforced by our current codebase. Keep that in mind if you see something that seemingly violates it.
```

Broadly, all code in the game should be separated based on whether it is *inside* the "simulation" or *outside* it. The "simulation" is a encompassing term that basically means "the contents of the actual game".

For example, the following things are "inside" the simulation:
- Basically everything concerning entities: interactions, physics, atmos, etc.
- IC chat
- Round state (lobby, in-game, post-game)

The following examples are "outside" the simulation:
- OOC chat
- Adminhelp
- Admin votes
- Basically anything talking to an external service, such as the database or a Discord webhook

We always need locations in the code where these two sides of the codebase exchange data. (For example, a player connecting is initially handled out of simulation, but the simulation needs to be notified of new players to spawn them in somehow.) Exactly how this should be done depends on a case-by-case basis, and it can take effort to do properly, but it is vitally important for code architecture.

A thought experiment to think about this is "should this logic stop working if the game were to be paused by an admin." If such a pause button were to exist, we would like to completely stop the game logic (no time would progress, nobody could move, etc), but we'd still like people to be able to connect to the server, talk in OOC chat, ask an admin *why* the game is still paused, and so on.

```admonish info
The game server currently already automatically pauses like this when no players are online, to save resources. This isn't purely theoretical! But perhaps hard to observe at the moment.
```

Time in the simulation may accelerate or slow down relative to "real time", depending on server settings or performance issues. On the client, the simulation is constantly committing time travel as part of network prediction. The simulation doesn't actually *exist* on the client until connected to a server!

Here are some of the differences between how in-simulation and out-of-simulation code should be written:

| Thing you want to do                | in-simulation           | out-of-simulation                                                                                |
|-------------------------------------|-------------------------|--------------------------------------------------------------------------------------------------|
| "Default place" for singleton code. | Make an `EntitySystem`  | Use a manager: make a new class, register it with IoC, and call it from `EntryPoint` or similar. |
| Check elapsed time                  | `IGameTiming.CurTime`   | `IGameTiming.RealTime`, `(R)Stopwatch`, `DateTime`, etc.                                         |
| Send custom network messages        | Networked entity events | Custom `NetMessage`                                                                              |

## Dependencies
Inside an entity system, prefer a system dependency instead of resolving the system using the IoCManager.

Bad:
```csharp
var random = IoCManager.Resolve<IRobustRandom>();
random.Prob(0.1f);
```

Good:
```csharp
[Dependency] private readonly IRobustRandom _random = default!;
_random.Prob(0.1f);
```

## Code Design Choices

### C# Events vs EventBus Events
The EventBus should generally be used over C# events where possible. C# events can leak, especially when used with components which can be created or removed at any time.

C# events should be used for out-of-simulation events, such as UI events.
Remember to *always* unsubscribe from them, however!

### Async vs Events
For things such as DoAfter, always use events instead of async.

Async for any game simulation code should be **avoided at all costs**, as it's generally virulent, cannot be serialized (in the case of DoAfter, for example), and usually causes icky code.
Events, on the other hand, tie in nicely with the rest of the game's architecture, and although they aren't as convenient to code, they are definitely way more lightweight.

### Enums vs Prototypes
The usage of enums for in-game types is *heavily discouraged*.
You should always use prototypes over enums.
Example: In-game tool "kinds" or "types" should use prototypes instead of enums.
