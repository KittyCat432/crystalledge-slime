# Troubleshooting & Common Issues

## Quick Diagnostic Checklist

When prediction isn't working:

1. **System location**: Is it in `Content.Shared`?
2. **Component attributes**: `[NetworkedComponent]`, `[AutoGenerateComponentState]`?
3. **Field networking**: `[AutoNetworkedField]` on changing data?
4. **Predicted APIs**: Using `PopupPredicted`/`PlayPredicted`?
5. **Dependencies**: Are all called systems predicted/shared?
6. **Component dirtying**: Calling `Dirty()` after changes?
7. **Client system**: Does empty client system exist if needed?
8. **Entity operations**: Using `PredictedSpawn*`/`PredictedDeleteEntity`?
9. **Container events**: Added `ApplyingState` guards?
10. **Timing guards**: Using `IsFirstTimePredicted` only for audiovisual code?
11. **Event subscriptions**: Check if events need `ApplyingState` protection
12. **Update loops**: Using timestamp comparison instead of accumulator?
13. **Field deltas**: Using `DirtyField` for specific field updates?
14. **PVS range**: Is issue due to entity being outside 25x25 range?
15. **Security**: Are sensitive components properly restricted?
16. **Randomness**: Using deterministic `System.Random` with shared seed?
17. **Entity references**: Do components reference entities that might get deleted?
18. **BUI prediction**: Using predicted BUI patterns?
19. **NetSync**: Is component networking properly enabled?
20. **Component scope**: Is `[NetworkedComponent]` only on shared components?

## Major Issue Categories

### Prediction Setup Issues

**System not predicting at all:**
- Move system to `Content.Shared`
- Add empty client system if using abstract pattern
- Verify all dependencies are shared
- Check component has `[NetworkedComponent]`

**Components not networking:**
- Add `[AutoGenerateComponentState]` to component
- Mark changing fields with `[AutoNetworkedField]`
- Call `Dirty()` after modifying fields
- Verify not using `[NetworkedComponent]` on server-only components

### Visual/Audio Problems

**Popups/audio repeat multiple times:**
- Replace `PopupEntity` with `PopupPredicted`
- Replace `PlayEntity` with `PlayPredicted`
- Ensure user parameter is provided

**UI doesn't update during prediction:**
- Convert to predicted BUI pattern
- Use `AfterAutoHandleState` for UI updates
- Replace `SendMessage` with `SendPredictedMessage`

### Performance Issues

**High network traffic:**
- Use timestamp + rate patterns instead of constant dirtying
- Add guard statements to prevent unnecessary `Dirty()` calls
- Use `DirtyField()` instead of `Dirty()` for large components

**Poor update loop performance:**
- Replace frametime accumulators with timestamp comparison
- Avoid dirtying every tick
- Use field deltas for selective updates

### Entity Management Errors

**"Predicting deletion of networked entity":**
- Replace `DeleteEntity` with `PredictedDeleteEntity`
- Use `PredictedQueueDeleteEntity` for queued deletion

**Duplicate entities after spawning:**
- Replace `Spawn`/`SpawnAtPosition` with `PredictedSpawnAtPosition`
- Use `PredictedSpawnAttachedTo` instead of `SpawnAttachedTo`

**"Can't resolve MetaDataComponent" errors:**
- Null out entity references when entities are deleted
- Wait for future `WeakEntityReference` implementation

### Randomness Problems

**Random values cause prediction jumps:**
- Create `System.Random` with deterministic seed
- Use `HashCodeCombine(CurTick, NetEntityId)` for seed
- Don't use `_random` directly in shared code

### Container Event Issues

**Events fire multiple times:**
- Add `if (_timing.ApplyingState) return;` guards
- Common with container insert/remove events
- Check if event is networked with component states

### Security & Session Issues

**Sensitive information leaked to clients:**
- Use `SendOnlyToOwner` for player-specific data
- Use `SessionSpecific` with custom visibility logic
- Restrict component visibility appropriately

### PVS-Related Issues

**Prediction doesn't work at distance:**
- Remember 25x25 PVS range limitation
- Use PVS overrides sparingly if needed
- Consider if system should be predicted at all

**UdderSystem debug assert:**
- Clear solution references on container removal
- See specific workaround in PVS system guide

## Advanced Troubleshooting

### IRobustCloneable Issues

**Reference type corruption during prediction:**
- Implement `IRobustCloneable` for custom reference types
- Provide proper deep copy in `Clone()` method
- Test object state persistence during prediction

### Architecture Problems

**Shared code can't access server systems:**
- Add virtual methods in shared system
- Override in server implementation
- Convert dependencies to shared if possible

### Event Prediction Problems

**Shared subscriptions not predicting:**
- Ensure events are raised in predicted contexts
- Check if events are server-only
- Consider if prediction is appropriate for the system

## Debugging Techniques

### Logging Prediction Behavior
```csharp
if (_timing.IsFirstTimePredicted)
    Log.Debug("First prediction run");

if (_timing.ApplyingState)
    Log.Debug("Applying server state");
```

### Component State Debugging
```csharp
// Log when component is dirtied
Dirty(uid, component);
Log.Debug($"Dirtied component on {uid}");
```

### Network Traffic Analysis
- Monitor component state frequency
- Check for unnecessary `Dirty()` calls
- Measure bandwidth impact of changes

## When Prediction May Not Be Appropriate

- Systems requiring global knowledge (atmos, power)
- High-security randomness (antag selection)
- Complex multi-entity interactions
- Systems with heavy PVS limitations
- Performance-critical server-only calculations