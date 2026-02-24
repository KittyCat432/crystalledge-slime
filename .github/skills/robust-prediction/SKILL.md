---
name: robust-prediction
description: 'RobustToolbox prediction system guide: converting server-only code to predicted shared systems. Use when implementing prediction for EntitySystems, fixing prediction issues, setting up NetworkedComponent attributes, or working with PopupPredicted and PlayPredicted APIs in RobustToolbox engine.'
---

# RobustToolbox Prediction System

A comprehensive guide for implementing and troubleshooting client-side prediction in RobustToolbox-based games. This skill provides quick navigation to specialized guides covering different aspects of the prediction system.

## When to Use This Skill

- Converting existing server-only EntitySystems to predicted systems 
- Implementing client-side prediction for game mechanics
- Fixing prediction-related issues (multiple popups, audio playing repeatedly)
- Setting up proper component networking with NetworkedComponent attributes
- Working with predicted APIs like PopupPredicted and PlayPredicted
- Troubleshooting prediction time travel and resimulation issues
- Understanding shared vs server code architecture in RobustToolbox

## How Prediction Works

Without prediction, client inputs are sent to server → server simulates → results sent back to client with noticeable latency delay.

With prediction, each client runs its own simulation for immediate local feedback while server maintains authoritative state. When client receives server state (from the past due to latency), client rewinds to that point, applies server corrections, and resimulates forward while reapplying inputs.

### Key Concepts
- **Time Travel**: Client constantly rewinds and resimulates game state
- **Authoritative Server**: Server state is always "truth" 
- **Local Prediction**: Client predicts own inputs only, others' actions still networked
- **Reconciliation**: Client corrects disagreements with server state

## Quick Start Checklist

1. **Move Code to Shared**: Components and systems to `Content.Shared`
2. **Setup Networking**: Add `[NetworkedComponent]` and `[AutoGenerateComponentState]`
3. **Configure Fields**: Mark changing fields with `[AutoNetworkedField]`
4. **System Architecture**: Create shared systems or abstract base with client/server implementations
5. **Use Predicted APIs**: Replace popup/audio methods with predicted variants
6. **Handle Dependencies**: Convert all called systems to shared/predicted
7. **Dirty Components**: Call `Dirty()` after changing networked fields

For detailed implementation steps, see the specialized guides below.

## Specialized Guides

### Core Implementation
- **[Component Networking](references/component-networking.md)** - Setup NetworkedComponent, field deltas, NetSync control
- **[Entity Management](references/entity-management.md)** - Predicted spawning/deletion, WeakEntityReference issues
- **[Timing & State Management](references/timing-state.md)** - IsFirstTimePredicted, ApplyingState, event handling

### Performance & Optimization  
- **[Update Loops & Performance](references/update-loops-performance.md)** - Efficient networking patterns, timestamp-based loops
- **[PVS System](references/pvs-system.md)** - Player Visibility System, nullspace, PVS overrides and limitations

### User Experience
- **[UI & Interfaces](references/ui-interfaces.md)** - Predicted popups/audio, BUI conversion patterns
- **[Randomness & Determinism](references/randomness-determinism.md)** - Predicted RNG, security considerations

### Security & Advanced Topics  
- **[Security & Sessions](references/security-sessions.md)** - SendOnlyToOwner, SessionSpecific, cheating prevention

### Problem Solving
- **[Troubleshooting Guide](references/troubleshooting.md)** - Comprehensive diagnostic checklist, common issues and solutions

## Architecture Patterns

### System Organization
```csharp
// Option 1: Single shared system (preferred for simple cases)
public sealed class YourSystem : EntitySystem { }

// Option 2: Abstract base with implementations  
public abstract class SharedYourSystem : EntitySystem { }
public sealed class YourSystem : SharedYourSystem { } // Server & Client
```

### Component Setup
```csharp
[NetworkedComponent]
[AutoGenerateComponentState]
public sealed partial class YourComponent : Component
{
    [AutoNetworkedField]
    public string NetworkedField = string.Empty;
}
```

### Dependencies Rule
- **Shared code** can ONLY call other shared code
- **Server/Client code** can call shared + their own code  
- Convert all dependencies to shared before predicting a system

## Common Pitfalls to Avoid

- Using `PopupEntity` instead of `PopupPredicted` → multiple popups
- Using `DeleteEntity` instead of `PredictedDeleteEntity` → networking errors  
- Forgetting to call `Dirty()` after field changes → stale client data
- Using `[NetworkedComponent]` on server-only components → silent failures
- Missing client system implementation → prediction disabled
- Using `_random` directly in shared code → mispredicts

## Best Practices Summary

- **Start simple** with basic systems before complex prediction
- **Test thoroughly** across different network conditions  
- **Use field deltas** (`DirtyField`) for large components
- **Guard statements** to prevent unnecessary networking
- **Security first** - restrict sensitive data with `SendOnlyToOwner`
- **PVS awareness** - remember 25x25 range limitations
- **Document** which systems are predicted for team reference

## References

Each specialized guide contains detailed code examples, troubleshooting steps, and best practices for its specific domain. Start with the troubleshooting guide if you're experiencing issues, or component networking guide if you're beginning implementation.

For database-related prediction work, also reference the `robust-db` skill for migration and model management.