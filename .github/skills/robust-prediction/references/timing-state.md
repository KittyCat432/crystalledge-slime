# Timing & State Management

## IGameTiming.IsFirstTimePredicted

Returns `true` the first time code runs, `false` in subsequent prediction ticks while waiting for server.

**Server behavior:** Always returns `true` (server doesn't predict)

### Usage Pattern
```csharp
if (!_timing.IsFirstTimePredicted)
    return; // Skip on prediction resimulation

// Audiovisual code here (UI updates, etc.)
```

### ⚠️ WARNING - Common Misuse
- This is NOT a magic bullet for mispredicts!
- Only for audiovisual information, not game logic
- Most APIs already include this guard internally
- [Currently misused across codebase](https://github.com/space-wizards/space-station-14/issues/41116) - don't copy existing bad patterns
- Using this incorrectly DISABLES prediction instead of fixing the underlying issue

### When to Use IsFirstTimePredicted
✅ **Appropriate uses:**
- UI updates that should only happen once
- Client-side visual effects
- Console logging/debugging
- One-time client notifications

❌ **Inappropriate uses:**
- Hiding actual mispredicts
- Game logic that should be deterministic
- Working around networking issues
- Preventing proper prediction

## IGameTiming.ApplyingState

Returns `true` while client rewinds to received server state from previous game tick.

### Usage Pattern
```csharp
if (_timing.ApplyingState)
    return; // Don't run during state application

// Normal game logic here
```

### Why ApplyingState is Needed
Some events are networked with component states and raised on both client/server even when not predicted. During state application, these events fire again, potentially causing duplicate effects.

### Container Event Scenarios

**A) Predicted insertion:** 
1. Event raised locally on both client/server 
2. Server networks state 
3. Client finds no differences 
4. Event fires repeatedly during resimulation

**B) Non-predicted insertion:** 
1. Event raised server-side only 
2. Server sends state 
3. Client applies state and raises event once

### Common Events Requiring ApplyingState Guards
- `EntInsertedIntoContainerMessage` / `EntGotInsertedIntoContainerMessage`
- `EntRemovedFromContainerMessage` / `EntGotRemovedFromContainerMessage`  
- `ContainerIsInsertingAttemptEvent` / `ContainerGettingInsertedAttemptEvent`
- `ContainerIsRemovingAttemptEvent` / `ContainerGettingRemovedAttemptEvent`
- `DamageChangedEvent`
- `HandCountChangedEvent`
- `GotEquippedEvent` / `GotEquippedHandEvent`
- `GotUnequippedEvent` / `GotUnequippedHandEvent`
- `DroppedEvent`

## Event Handling Best Practices

### Guard Pattern for Container Events
```csharp
private void OnContainerInserted(EntityUid uid, SomeComponent comp, EntInsertedIntoContainerMessage args)
{
    if (_timing.ApplyingState)
        return; // Don't run during state reconciliation
        
    // Normal event handling logic
    DoSomethingWithInsertion(uid, args.Entity);
}
```

### Debugging Prediction Issues
```csharp
private void OnSomeEvent(EntityUid uid, SomeComponent comp, SomeEvent args)
{
    if (_timing.IsFirstTimePredicted)
    {
        // Debug logging - only on first prediction
        Log.Debug($"Event fired for {uid}");
    }
    
    if (_timing.ApplyingState)
        return; // Skip during reconciliation
        
    // Actual game logic
}
```

## Common Timing Issues

### Issue: Code Runs Multiple Times
**Symptom:** Breakpoints hit 10+ times, console spam
**Cause:** Normal prediction behavior - client resimulates frequently
**Solution:** Use predicted APIs for UI/audio, understand this is expected

### Issue: Container Events Fire Multiple Times
**Symptom:** Container insertion/removal logic executes repeatedly
**Cause:** Events networked with component states, no `ApplyingState` guard
**Solution:** Add `if (_timing.ApplyingState) return;` guard to event handlers

### Issue: UI Flickers During Prediction
**Symptom:** Interface elements appear/disappear rapidly
**Cause:** UI updates running on every prediction tick
**Solution:** Use `IsFirstTimePredicted` guard for UI-only code

## Best Practices
- Use `ApplyingState` guards for events that fire during state reconciliation
- Only use `IsFirstTimePredicted` for audiovisual effects, not game logic
- Understand that code running multiple times during prediction is normal
- Test event handlers carefully for proper prediction behavior
- Don't use timing guards to hide actual mispredicts - fix the root cause