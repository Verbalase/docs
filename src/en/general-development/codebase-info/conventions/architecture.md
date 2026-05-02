# Architecture Conventions

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

### Dependencies On Other Systems
Inside an entity system, prefer a system dependency instead of resolving the system using the IoCManager. For example, instead of:

```csharp
var random = IoCManager.Resolve<IRobustRandom>();
random.Prob(0.1f);
```

Add an entity system dependency:

```csharp
[Dependency] private readonly IRobustRandom _random = default!;
_random.Prob(0.1f);
```

### C\# Events vs EventBus Events
The EventBus should generally be used over C# events where possible. C# events can leak, especially when used with components which can be created or removed at any time.

C# events should be used for out-of-simulation events, such as UI events.
Remember to *always* unsubscribe from them, however!

### Async vs Events
For things such as DoAfter, always use events instead of async.

Async for any game simulation code should be avoided at all costs, as it's generally virulent, cannot be serialized (in the case of DoAfter, for example), and usually causes icky code.
Events, on the other hand, tie in nicely with the rest of the game's architecture, and although they aren't as convenient to code, they are definitely way more lightweight.

### Enums vs Prototypes
The usage of enums for in-game types is *heavily discouraged*.
You should always use prototypes over enums.
Example: In-game tool "kinds" or "types" should use prototypes instead of enums.
