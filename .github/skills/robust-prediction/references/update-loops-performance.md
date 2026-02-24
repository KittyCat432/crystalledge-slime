# Update Loops & Performance Optimization

## The Performance Problem
Dirtying and networking is expensive. Avoid dirtying entities every single tick from update loops.

## Smart Networking Patterns

### Timestamp + Rate Pattern (Recommended)
Instead of networking current values repeatedly, send timestamp + rate of change:

```csharp
// ❌ BAD - Dirty every tick (expensive networking)
public override void Update(float frameTime)
{
    component.SomeValue += frameTime;
    Dirty(uid, component); // Networks every frame!
}

// ✅ GOOD - Smart networking with rate of change
public override void Update(float frameTime)
{
    // Send timestamp + rate instead of current value repeatedly
    // Client can calculate current value: lastValue + (currentTime - timestamp) * rate  
    if (shouldUpdateRate)
    {
        component.LastUpdateTime = _timing.CurTime;
        component.ChangeRate = newRate;
        DirtyField(uid, component, nameof(YourComponent.ChangeRate));
    }
}
```

**Examples:** [HungerSystem](https://github.com/space-wizards/space-station-14/blob/master/Content.Shared/Nutrition/EntitySystems/HungerSystem.cs), [BatterySystem](https://github.com/space-wizards/space-station-14/blob/master/Content.Shared/Power/EntitySystems/SharedBatterySystem.API.cs)

### Predicted Update Loops
Old code often accumulates frametime, which is bad for prediction due to expensive network updates.

**❌ Bad - Frametime Accumulator:**
```csharp
[AutoNetworkedField]
public float Accumulator = 0f;

public override void Update(float frameTime)
{
    Accumulator += frameTime;
    if (Accumulator >= UpdateInterval)
    {
        Accumulator -= UpdateInterval;
        // Do update logic
        Dirty(uid, component); // Networks entire component every frame!
    }
}
```

**✅ Good - Timestamp Comparison:**
```csharp
[AutoNetworkedField]
public TimeSpan NextUpdateTime = TimeSpan.Zero;

public override void Update(float frameTime)
{
    var currentTime = _timing.CurTime;
    if (currentTime >= NextUpdateTime)
    {
        NextUpdateTime = currentTime + TimeSpan.FromSeconds(UpdateInterval);
        // Do update logic
        DirtyField(uid, component, nameof(YourComponent.NextUpdateTime)); // Only networks timestamp
    }
}
```

## Optimization Techniques

### Guard Statements
Only dirty when values actually change:

```csharp
public void SetExampleDataField(Entity<ExampleComponent?> ent, int newValue)
{
    if (!Resolve(ent, ref ent.Comp))
        return; // Component doesn't exist
    
    if (ent.Comp.ExampleDataField == newValue)
        return; // No change, skip networking
        
    ent.Comp.ExampleDataField = newValue;
    Dirty(ent); // Only network when necessary
}
```

### Field Deltas
For components with multiple networked fields or fields changed at different rates:

```csharp
// Use DirtyField for specific field updates
DirtyField(uid, component, nameof(YourComponent.SpecificField));

// Instead of Dirty() which sends all networked fields
Dirty(uid, component); // Sends everything - less efficient
```

### Component Access Restrictions
Prevent misuse and forgotten Dirty() calls:

```csharp
[Access(typeof(YourSystem))] // Only YourSystem can modify this component
public sealed partial class YourComponent : Component
{
    // Prevents other systems from forgetting to call Dirty()
    // Forces use of proper setter API methods
}
```

## Common Performance Issues

### Issue: Excessive Network Traffic from Update Loops
**Symptom:** High bandwidth usage, poor performance during gameplay
**Cause:** Dirtying components every frame instead of using smart networking
**Solution:** Use timestamp + rate pattern, guard statements, and `DirtyField` for specific fields

### Issue: Large Components Network Too Much Data
**Symptom:** Network spikes when components update
**Cause:** Using `Dirty()` instead of `DirtyField()` for selective updates
**Solution:** Use field deltas to only network changed fields

## Best Practices
- Use timestamp comparison instead of frametime accumulators
- Add guard statements to prevent unnecessary `Dirty()` calls  
- Use `DirtyField()` for large components with multiple networked fields
- Consider network bandwidth impact when adding networked fields
- Use rate-of-change patterns for continuously changing values
- Implement component access restrictions with `[Access(typeof(System))]`