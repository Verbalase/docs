# Project Conventions

These conventions are specific to Space Station 14. They may talk about code or systems that aren't relevant to other projects, or those other projects may simply have a different opinion about code style.

## File Layout

1. Start with [using directives](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/keywords/using-directive) at the top of the file.

2. All classes should be explicitly namespaced. Use [file-scoped namespaces](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/proposals/csharp-10.0/file-scoped-namespaces), e.g. a single `namespace Content.Server.Atmos.EntitySystems;` before any class definitions instead of `namespace Content.Server.Atmos.EntitySystems { /* class here */ }`.

3. Always put all fields and auto-properties before any methods in a class definition.

## Methods

### Line breaks of parameter/argument lists

If you're defining a function and the parameter declarations are so long they don't fit on a single line, break them apart so you have **one parameter per line**. Some leeway is granted for closely tied parameter pairs like X/Y coordinates and pointer/length in C APIs.

Bad:

```csharp
public void CopyTo(ISerializationManager serializationManager, SortedDictionary<TKey, TValue> source, ref SortedDictionary<TKey, TValue> target,
    SerializationHookContext hookCtx, ISerializationContext? context = null)
```

Good:

```csharp
public void CopyTo(
    ISerializationManager serializationManager,
    SortedDictionary<TKey, TValue> source,
    ref SortedDictionary<TKey, TValue> target,
    SerializationHookContext hookCtx,
    ISerializationContext? context = null)
```

## Constants and CVars
If you have a specific value such as an integer you should generally make it either:
* a constant (const) if it's never meant to be changed
* a CVar if it's meant to be configured

This is so it is clear to others what it is. This is especially true if the same value is used in multiple places to make the code more maintainable.

### Public API Method Signature
All public Entity System API Methods that deal with entities and game logic should *always* follow a very specific structure.

All relevant `Entity<T?>` and `EntityUid` should come first.
The `T?` in `Entity<T?>` stands for the component type you need from the entity.
The question mark `?` must be present at the end to mark the component type as nullable.
Next, any arguments you want should come afterwards.

The first thing you should do in your method's body should then be calling `Resolve` for the entity UID and components.

<details>
  <summary>Example (click to expand)</summary>

```csharp
public void SetCount(Entity<StackComponent?> stack, int count)
{
    // This call below will set "Comp" to the correct instance if it's null.
    // If all components were resolved to an instance or were non-null, it returns true.
    if(!Resolve(stack, ref stack.Comp))
        return; // If the component wasn't found, this will log an error by default.

    // Logic here!
}
```

</details>

The `Resolve` helper performs a few useful checks for you. In `DEBUG`, it checks whether the component reference passed (if not null) is actually owned by the entity specified.

This helper will also log an error by default if the entity is missing any of the components that you attempted to resolve.
This error logging can be disabled by passing `false` to the helper's `logMissing` argument. You may want to disable the error logging for resolving optional components, `TryX` pattern methods, etc.

Please note that the `Resolve` helper also has overloads for resolving 2, 3 or even 4 components at once.
If you want to resolve components for multiple entities, or you want to resolve more than 4 components at once for a given entity, you'll need to perform multiple `Resolve` calls.

### Extension Methods

Extension methods (those with an explicit `this` for the first argument) should never be used on any classes directly related to simulation--that means `EntityUid`, components, or entity systems. Extension methods on `EntityUid` are used throughout the codebase, however this is bad practice and should be replaced with entity system public methods instead.

## UI

### XAML and C#-defined UIs
You should always use XAML over UIs defined entirely in C# code.
Extending existing C#-defined UIs is fine, but they should be converted eventually.

## Performance

### Iterator Methods vs returning collections
Always use [iterator methods](https://docs.microsoft.com/en-us/dotnet/csharp/iterators) over creating a new collection and returning it in your method.

Keep in mind, however, that iterator methods allocate a lot of memory.
If you need to reduce allocations as much as possible, use struct iterators.

### Sealed Classes
Your class must be marked as either `abstract`, `static`, `sealed` or `[Virtual]`. This is to avoid accidentally making classes inheritable when they shouldn't be and can improve performance slightly when accessing or invoking virtual members.

Use `sealed` if the class shouldn't be inherited, `[Virtual]` for the normal C# behavior (it mutes the compiler warning), `static` for classes that don't need to be instantiated, or `abstract` if it's meant for being inherited but not meant to be instantiated by itself.

### Events over updates
Where possible you should always have your system run code in response to an event rather than updating every tick. Your code may only take up 0.5% of CPU time but when 100 systems do this it's unnecessary.

### Variable capture
When using [lambdas](https://docs.microsoft.com/en-us/dotnet/csharp/language-reference/operators/lambda-expressions) or [local functions](https://docs.microsoft.com/en-us/dotnet/csharp/programming-guide/classes-and-structs/local-functions) be sure to **avoid variable captures**.

If you're adding a method that takes in a [Func delegate](https://docs.microsoft.com/en-us/dotnet/api/system.func-2), be sure to have an overload that **allows the caller to pass in custom data** to it.

<details>
  <summary>Example of what not to do (click to expand)</summary>

```csharp
void DoSomething(EntityUid otherEntity)
{
    // This is BAD. It will allocate on the heap a lot.
    var predicate = (EntityUid uid)
        => uid == otherEntity;

    // This method doesn't allow us to pass custom data,
    // so we're forced to do a costly variable capture.
    MethodWithPredicate(predicate);
}

void MethodWithPredicate(Func<EntityUid, bool> predicate)
{
    // We do something with the predicate here...
}
```

</details>

<details>
  <summary>Example of what to do (click to expand)</summary>

```csharp
void DoSomething(EntityUid otherEntity)
{
    // This is good and much more performant than the example before.
    var predicate = (EntityUid uid, EntityUid otherUid)
        => uid == otherUid;

    // Pass our custom data to this method.
    MethodWithPredicate<EntityUid>(predicate, otherEntity);
}

// This method allows you to pass custom data into the predicate.
void MethodWithPredicate<TState>(Func<EntityUid, TState, bool> predicate, TState state)
{
    // We do something with the predicate here, making sure to pass "state" to it...
}
```

</details>

## Naming

### Shared types
Shared types should only be prefixed with `Shared` if and only if there are server and/or client inherited types with the same name.

Example:
- If `FooComponent` only exists in shared, it doesn't need a prefix.
- If `BarComponent` exists in shared, server and client, the shared type should be prefixed with shared: `SharedBarComponent`.
