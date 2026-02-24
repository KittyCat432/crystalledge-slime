# Entity Management & Prediction

## Predicted Entity Spawning

### The Problem with Regular Spawning in Shared Code
If you call regular spawn methods in shared code, both client and server spawn entities separately, resulting in duplicates.

### Spawning Entities

**❌ Wrong in shared code - creates duplicates:**
```csharp
Spawn(prototype, coordinates);           // Both client and server spawn separately
SpawnAtPosition(prototype, coordinates);  // Results in duplicate entities
SpawnAttachedTo(prototype, parent);
```

**✅ Correct - predicted spawning:**
```csharp
PredictedSpawnAtPosition(prototype, coordinates);   // Client predicts, server authoritative
PredictedSpawnAttachedTo(prototype, parent);       // Client entity replaced by server entity

// Note: Spawn() has no predicted equivalent - use SpawnAtPosition alternatives
```

### Current Spawning Limitations
- Predicted entities are NOT reconciled with server state
- Client-side predicted entities cannot be interacted with
- Animated sprites reset when server entity replaces predicted one (visual glitches)
- See [Entity spawn prediction v2](https://github.com/space-wizards/RobustToolbox/issues/5845) for future improvements

## Predicted Entity Deletion

### Deletion Methods

**❌ Wrong - causes errors:**
```csharp
DeleteEntity(uid);        // Error: "Predicting the deletion of a networked entity"
QueueDeleteEntity(uid);
```

**✅ Correct - predicted deletion:**
```csharp
PredictedDeleteEntity(uid);       // Moves entity to nullspace first, then server deletes
PredictedQueueDeleteEntity(uid);  // Queued version of predicted deletion
```

### How Predicted Deletion Works
1. Client moves entity to nullspace (appears to disappear)
2. Server deletes entity normally
3. Server networks deletion to client
4. Client confirms deletion

## WeakEntityReference Problem

### The Issue
`EntityUid` fields in components cause networking errors when referenced entities get deleted.

**Current Issue:** ECS convention states `EntityUid` should always reference valid entities. When autonetworked:
1. Server converts `EntityUid` → `NetEntity` (reads `MetaDataComponent`)
2. Sends to client
3. Client converts `NetEntity` → `EntityUid`
4. **If entity deleted** → `MetaDataComponent` deleted → **ERROR**

### Typical Error Message
```
Can't resolve "Robust.Shared.GameObjects.MetaDataComponent" on entity 957132/n0D!
```

**WizDen servers:** 20,000+ of these errors daily!

### Current Workaround (Complex)
**Manual cleanup required:**
- Set datafield to `null` when entity deleted
- Track with marker components  
- Lots of boilerplate code
- Easy to forget and introduce bugs

### Future Solution
**Engine improvements needed:**
- `WeakEntityReference` type for entities that may be deleted
- Automatic relations system to null out references
- See [RobustToolbox issue #6152](https://github.com/space-wizards/RobustToolbox/issues/6152)

## Common Entity Management Issues

### Issue: Duplicate Entities After Spawning
**Symptom:** Two identical entities exist, one doesn't respond to interactions
**Cause:** Using `Spawn`/`SpawnAtPosition` in shared code instead of predicted variants
**Solution:** Replace with `PredictedSpawnAtPosition`/`PredictedSpawnAttachedTo`

### Issue: "Predicting the deletion of a networked entity" Error
**Symptom:** Error when trying to delete entities in predicted code
**Cause:** Using `DeleteEntity`/`QueueDeleteEntity` instead of predicted variants
**Solution:** Replace with `PredictedDeleteEntity`/`PredictedQueueDeleteEntity`

### Issue: Animation Resets When Entity Spawns
**Symptom:** Animated sprites restart when predicted entity replaced by server entity
**Cause:** Current limitation of predicted spawning system
**Solution:** Known limitation, tracked in [Entity spawn prediction v2](https://github.com/space-wizards/RobustToolbox/issues/5845)

### Issue: "Can't resolve MetaDataComponent" Errors
**Symptom:** Networking errors when entities reference deleted entities
**Cause:** `EntityUid` datafields pointing to deleted entities
**Solution:** Manually null out references when entities deleted (or wait for `WeakEntityReference`)

## Best Practices
- Use predicted spawning/deletion methods in shared code
- Properly handle entity deletion in references (null out or use future `WeakEntityReference`)
- Test entity lifecycle carefully in prediction scenarios
- Be aware of current limitations with predicted entity spawning
- Consider using client-side entities for UI previews and temporary effects