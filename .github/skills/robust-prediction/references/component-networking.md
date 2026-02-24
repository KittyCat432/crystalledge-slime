# Component Networking Setup

## Basic Component Networking Attributes

```csharp
[NetworkedComponent]
[AutoGenerateComponentState]
public sealed partial class YourComponent : Component
{
    [AutoNetworkedField]
    [DataField("fieldName")]
    public string SomeField = string.Empty;
}
```

**Key Points:**
- `NetworkedComponent`: Enables networking for component
- `AutoGenerateComponentState`: Auto-generates networking code
- `AutoNetworkedField`: Networks specific fields that change after spawning
- Don't network prototype values that never change

## Field Delta Optimization

**When to use field deltas:**
- Large components with many networked fields
- Fields changed at different rates
- Reduces network load

```csharp
// Dirty entire component (sends all networked fields)
Dirty(uid, component);

// Dirty specific field only (more efficient)
DirtyField(uid, component, nameof(YourComponent.SpecificField));
```

## NetSync Control

**Purpose:** Disable component networking when needed.

```csharp
[NetworkedComponent]
public sealed partial class YourComponent : Component
{
    // Inherited from base Component class
    // Setting to false disables all networking for this component
    public bool NetSync = true; // default
}

// In system code:
component.NetSync = false;
Dirty(uid, component); // Does nothing now - component won't be networked
```

**Use cases:**
- Allow clients to modify component datafields without server overwriting
- Keep specific components unpredicted on purpose
- Client-only temporary state that shouldn't sync to server
- Performance optimization for purely local data

## Component State Management

### Dirty Components When Changed
```csharp
// Always dirty after changing datafields
component.SomeField = newValue;
Dirty(uid, component);

// Or use field deltas for better performance
component.SpecificField = newValue;
DirtyField(uid, component, nameof(YourComponent.SpecificField));
```

### Manual Component States (Advanced)
If you need more control than `AutoGenerateComponentState`:

```csharp
// Override GetComponentState and HandleComponentState methods
public override ComponentState GetComponentState()
{
    // Custom state generation logic
}

public override void HandleComponentState(ComponentState? curState, ComponentState? nextState)
{
    // Custom state handling logic
}
```

## Common Networking Issues

### Issue: NetworkedComponent Misuse
**Critical warning:** Only use `[NetworkedComponent]` on shared components!

```csharp
// ❌ WRONG - Server-only component with NetworkedComponent
// In Content.Server/...
[NetworkedComponent] // This breaks other components!
public sealed class ServerOnlyComponent : Component { }

// ✅ CORRECT - Only shared components
// In Content.Shared/...
[NetworkedComponent]
public sealed class SharedComponent : Component { }
```

**Symptoms when misused:**
- Random other components stop networking  
- Widespread mispredicts across unrelated systems
- UIs not populating with data
- Silent failure (no errors or warnings)

## Best Practices
- Only use `[NetworkedComponent]` on components in `Content.Shared`
- Use `AutoGenerateComponentState` for simple cases
- Network only fields that change after spawning
- Use field deltas (`DirtyField`) for large components
- Always call `Dirty()` or `DirtyField()` after changing networked fields
- Use `NetSync = false` for client-only or unpredicted data
- Add guard statements to prevent unnecessary networking