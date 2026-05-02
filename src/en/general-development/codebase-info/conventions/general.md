# General Programming Conventions

These conventions are not really specific to Space Station 14, and you should be following these no matter what project you are working on. Any experienced programmer should know these by heart.

## Don't copy paste code

If you're every looking at another piece of code and think "I want to do the same thing as this": **DO NOT** copy paste it. Make a new function or some other kind of abstraction that allows you to re-use as much code as possible.

Copy-pasting code is a gigantic maintenance hazard as, in the future, if somebody needs to update the code you copied, they now have to do it in *two* places (and be aware that those *two places* even exist).

Of course, there are places where you may think you're "copy pasting" unavoidable code. For example, the basic structure for making an `EntitySystem` that does a thing always has a class definition, some dependencies, an `override void Initialize()`, and so on. This kind of "boilerplate" is fine to copy as there's really no way to avoid it.

## Don't use magic strings/numbers

These are kind of a subset of "don't copy paste code". A "magic" value is any case in which you have a value in your code, say a string or a number, and it needs to be *that* specific value because it has to be the same as some other value somewhere else.

The name of the game here is "make sure that if two values have to match, it's practically impossible for them not to be." Be that via compiler error, unit test failure, a guaranteed crash on startup, whatever.

In the simplest case, such magic values should simply be stored in a `const` or `static readonly` that gets referenced from multiple locations, so the C# compiler enforces they will always be the same. If you need to reference a prototype ID from C#, you should define the prototype ID in a `static readonly ProtoId<T>`, since our validation tooling ensures the IDs in those fields are always valid.

## Comments

- Comment code at a high level to explain *what* the code is doing, and more importantly, *why* code is doing what it is doing.

- When documenting classes, structs, methods, properties/fields, and class members, use [XML docs](https://docs.microsoft.com/en-us/dotnet/csharp/language-reference/xmldoc/). DataFields and Public methods should always be documented.
    - Example:
  ```csharp
      /// <summary>
      /// Resets the InteractCounter on the <see cref="FooComponent"/>.
      /// </summary>
      /// <remarks>
      /// This is a public method other systems can call to interact with FooComponent!
      /// Remember that public methods should always use docstring.
      /// </remarks>
      [PublicAPI]
      public void ResetInteractCounter(Entity<FooComponent?> ent)
  ```

### Why Not What

Some folks blindly adhere to "comment the why, not the what" and think that "code should be self-documenting and comments a last resort". Below we present a few examples that we hope will change your mind.

#### Example 1

```csharp
var fractionalPressureChange = Atmospherics.R * (outlet.Air.Temperature / outlet.Air.Volume + inlet.Air.Temperature / inlet.Air.Volume);
```

All of the variables are named in a self-documenting way (*R* gets a pass because that is the ideal gas constant, and physics conventions existed long before computers, so this is following convention). Obviously, the comment should *not* be:

```csharp
// Take R and multiply it by the ratio of outlet temperature divided by outlet air volume and add it to ...
var fractionalPressureChange = Atmospherics.R * (outlet.Air.Temperature / outlet.Air.Volume + inlet.Air.Temperature / inlet.Air.Volume);
```

Because this only explains what the code is literally doing, which you could have gathered from any cursory reading of the code. **However, you still have absolutely no idea what this code is doing and why**, even though the code is self-documenting.

You don't know where this magic formula came from, what it's trying to accomplish, or even if the formula is correct. Therefore, this needs to be documented:


```csharp
// We want moles transferred to be proportional to the pressure difference, i.e.
// dn/dt = G*P

// To solve this we need to write dn in terms of P. Since PV=nRT, dP/dn=RT/V.
// This assumes that the temperature change from transferring dn moles is negligible.
// Since we have P=Pi-Po, then dP/dn = dPi/dn-dPo/dn = R(Ti/Vi - To/Vo):
var dPdn = Atmospherics.R * (outlet.Air.Temperature / outlet.Air.Volume + inlet.Air.Temperature / inlet.Air.Volume);
```

#### Example 2

```csharp
if (HasComp<MindContainerComponent>(uid))
    return;

// more stuff
```

Obviously, this code skips "more stuff" if the entity represented by *uid* already has a MindContainerComponent. This code is as self-documenting as it gets, it literally just returns early if there is a MindContainer. What needs to be documented is *why* this code needs to skip *uid*s that already have a MindContainerComponent:


```csharp
// Don't let players who drink cognizine be eligible for a ghost takeover
if (HasComp<MindContainerComponent>(uid))
    return;
```

## Strings and Identifiers

Human-readable text should never be used as an identifier or vice versa. In one direction, that means no putting human-readable text (result of localization functions) in a dictionary key, comparing with `==`, etc... In the other direction, that means things like "never show `Enum.ToString()` to a user directly."

This avoids spaghetti when these inevitably have to be decoupled for various reasons, and avoids inefficiency and bugs from comparing human-readable strings.

Example:

```csharp
private void UpdateDisplay(Gender gender)
{
    // This can't be localized! And the capitalization is kinda weird!
    // Don't do this!
    GenderLabel.Text = gender.ToString();

    // This is good!
    GenderLabel.Text = Loc.GetString($"gender-{gender}");
}
```

### Invariant comparisons on human-readable strings

If you're doing something like a filter/search dialog, use `CurrentCulture` comparisons over human-readable strings. Do not use invariant cultures.

## Properties

In a property setter, the value of the property should always literally become the `value` given. None of this:

```csharp
public string Name
{
    get => _name;
    private set => _name = Loc.GetString(value);
}
```

## Properly order members in a type

When laying out the contents of a type, you should **always** put fields above all other instance members. When reading a piece of code, the best way to get familiar with it is to look at the data it operates on. If fields and other members are mixed randomly, it can be much harder to understand the code.

For this rule, auto-properties (e.g. `string FooBar { get; set; }`) are considered the same as fields, since they have an internal field. Non-auto properties (e.g. `string FooBar => _field.Trim();`) do not, so should not be mixed.

Bad:

```csharp
class FooBar
{
    private int _field;

    public void Update() {
        _field *= 2;
        Counter += 1;
    }

    public int Counter { get; set; }
}
```

Good:

```csharp
class FooBar
{
    private int _field;
    public int Counter { get; set; }

    public void Update() {
        _field *= 2;
        Counter += 1;
    }

}
```
