# Player Visibility System (PVS)

## Overview
Server only networks entities within **25x25 square range** around client's attached entity to reduce network load.

## PVS Behavior
- **In range**: Entity runs normally, receives updates
- **Out of range**: Entity paused, moved to nullspace on client until re-entering PVS
- **Server**: Always has full knowledge of all entities (authoritative)
- **Client**: Limited to PVS range information

## PVS Implications for Prediction
- Client cannot predict anything outside PVS range
- Some systems cannot be predicted (atmos, power) due to PVS limitations
- Entities outside PVS won't have update loops running on client

## Nullspace
Empty default map where entities spawn without specific location.

**Common nullspace entities:**
- Antagonist objectives (hidden from other players)
- Mind and mind role entities (antag status)
- Abstract data entities (no physical location)

**Security benefit:** Entities on different maps aren't networked to players, preventing cheaters from reading sensitive data.

## PVS Overrides
**Use sparingly** - adds networking overhead!

```csharp
// Make entity visible to specific player
AddSessionOverride(entityUid, playerSession);
// Example: Wizard recall ability for far away marked items

// Make entity visible to all players
AddGlobalOverride(entityUid);
// Example: Singularity (large distortion overlay effect)

// Allow player to see entities around target location
AddViewSubscriber(playerSession, targetEntityUid);
// Example: Security cameras for remote viewing
```

## Common PVS Issues

### Issue: Prediction Doesn't Work Outside View Range
**Symptom:** Entities far from player don't predict properly
**Cause:** Player Visibility System (PVS) limits client knowledge to 25x25 range
**Solution:** This is intended behavior. Use PVS overrides sparingly if truly needed

### Issue: UdderSystem Debug Assert (Solution PVS Issue)
**Problem:** Debug assert when predicted update loop calls `SharedSolutionContainerSystem.ResolveSolution` and entity leaves PVS range.

**Workaround:**
```csharp
private void OnEntRemoved(Entity<UdderComponent> entity, ref EntRemovedFromContainerMessage args)
{
    // Make sure the removed entity was our contained solution
    if (entity.Comp.Solution == null || args.Entity != entity.Comp.Solution.Value.Owner)
        return;

    // Clear cached reference as entity leaves PVS range
    entity.Comp.Solution = null;
}
```

**Status:** Temporary workaround until proper fix. See [SS14 issue #42218](https://github.com/space-wizards/space-station-14/issues/42218).

## Best Practices
- Remember client knowledge limited to 25x25 range around player
- Use PVS overrides sparingly and only when absolutely necessary
- Consider PVS implications when designing prediction systems
- Test prediction behavior at PVS boundaries