# Networking Conventions

## Field Deltas

Field deltas allow you to send only specific fields of a component over the network instead of the entire state. This is done by adding `fieldDeltas: true` to your `AutoGenerateComponentState` attribute:

```csharp
[RegisterComponent, NetworkedComponent, AutoGenerateComponentState(fieldDeltas: true)]
public sealed partial class MyComponent : Component
{
    [DataField, AutoNetworkedField]
    public bool IsActive;

    [DataField, AutoNetworkedField]
    public int Value;
}
```

### When to use field deltas

Field deltas are great when:
- Your component has fields that change at different rates
- Only a subset of fields typically changes at once
- You have a bunch of networked fields and don't want to send all of them every time

A good rule of thumb: If you have 3+ fields and they often change independently, consider field deltas. For components with just 1-2 fields, it's usually simpler to skip them.

### Marking fields as dirty

When you change a field and want to network just that field, use `DirtyField` instead of `Dirty`:

```csharp
// Instead of this:
comp.IsActive = true;
Dirty(uid, comp);  // Would send ALL networked fields

// Do this:
comp.IsActive = true;
DirtyField(uid, comp, nameof(MyComponent.IsActive));  // Only sends IsActive
```

For a component with many fields where usually only one or two change at a time, field deltas can reduce network traffic by 80-90%. The more fields you have, the more you'll benefit from field deltas.

Even for components with just 3-4 fields, if they change independently (e.g., one field updates frequently, others rarely), field deltas can still be worth it.

Field deltas add a little overhead for tracking field changes, but this is usually outweighed by the bandwidth savings. The generator automatically handles most of the implementation complexity.

## TimeSpans

### Using TimeSpans

You should always use `TimeSpan` over `float` for defining static periods of time, such as intervals. Update loops should compare against `CurTime` instead of accumulating `frametime`.

### Handling paused entities

When working with `TimeSpan` fields that are modified during runtime (like timers or countdowns), you need to handle entity pausing properly. SS14 provides two important mechanisms for this.

### AutoGenerateComponentPause and AutoPausedField

The `[AutoGenerateComponentPause]` and `[AutoPausedField]` attributes work together to automatically adjust `TimeSpan` fields when an entity is unpaused:

- `[AutoGenerateComponentPause]` is applied to a component class and automatically generates code to handle unpausing.
- `[AutoPausedField]` is applied to individual `TimeSpan` fields within that component that should be adjusted when the entity is unpaused.

These attributes should **always** be used for `DataField` `TimeSpan` properties that are modified by other systems during runtime, such as timers or cooldowns.

<details>
  <summary>Example usage (click to expand)</summary>

```csharp
[RegisterComponent, AutoGenerateComponentPause]
public sealed partial class CooldownComponent : Component
{
    [DataField, AutoPausedField]
    public TimeSpan CooldownEnd;

    [DataField, AutoPausedField]
    public TimeSpan? OptionalTimer;
}
```
</details>

### TimeOffsetSerializer

The `TimeOffsetSerializer` is used for serializing `TimeSpan` values that are offset by the current game time.

- It automatically offsets a `TimeSpan` by the game's current time during serialization/deserialization
- If the entity is paused, it uses the time at which the entity was paused as the reference point
- It prevents unintentional saving of time offsets to maps during mapping (prototypes always serialize as zero)

Similar to `AutoPausedField`, the `TimeOffsetSerializer` should always be used for runtime-modified `TimeSpan` fields that represent absolute times rather than durations.

<details>
  <summary>Example usage (click to expand)</summary>

```csharp
[DataField(customTypeSerializer: typeof(TimeOffsetSerializer))]
public TimeSpan NextActivationTime;
```
</details>
