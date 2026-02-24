# Randomness & Determinism in Prediction

## The Problem
Using `RobustRandom` in shared code causes mispredicts because server and client roll different results. Client also generates different results each prediction tick.

**Example symptoms:** Random spawning creates jumping/teleporting effects during prediction, randomized colors flicker, locations change unpredictably.

## Current Workaround (Until RandomPredicted Merges)

### Wrong Way - Causes Mispredicts
```csharp
// This will mispredict - different results on client/server
bool result = _random.Prob(0.5f);
var randomItem = _random.Pick(items);
```

### Correct Way - Predicted Randomness
```csharp
// EntitySystem dependencies:
// [Dependency] private readonly IGameTiming _timing = default!;
// [Dependency] private readonly IRobustRandom _random = default!;

// Create deterministic seed both client and server can calculate
var seed = SharedRandomExtensions.HashCodeCombine(
    (int)_timing.CurTick.Value, 
    GetNetEntity(uid).Id
);
var rand = new System.Random(seed);

// Now use System.Random methods (same interface as IRobustRandom)
bool resultPredicted = rand.Prob(0.5f);
var randomItemPredicted = rand.Pick(items);
float randomFloatPredicted = rand.NextFloat();
```

### Seed Components
- **Current Game Tick**: Ensures different results across time
- **NetEntity ID**: Ensures different results per entity
- **Both needed**: Using only game tick makes all randomness in same tick identical

## Security Considerations

### ⚠️ Warning: Potential Cheating Vector
Cheaters who know `NetEntity` ID could theoretically influence results by timing inputs to specific game ticks.

### Keep Large Advantages Unpredicted
**Keep unpredicted for high-value randomness:**
- Random item spawning with valuable items
- Telecrystal discounts in shops
- Antagonist selection
- Objective assignment
- Any major advantage-giving RNG

### Safe to Predict
- Visual effects randomness
- Audio variation
- Minor gameplay elements
- Cosmetic randomization

## Future Improvements

**Coming soon:** [RandomPredicted PR](https://github.com/space-wizards/RobustToolbox/pull/5849) will provide built-in predicted randomness methods, eliminating need for manual seed calculation.

## Common Randomness Issues

### Issue: Random Values Cause Mispredicts
**Symptom:** Entities jump around, colors flicker, random spawning behaves erratically
**Cause:** Using `_random` instead of deterministic `System.Random` with shared seed
**Solution:** Create `System.Random` with `HashCodeCombine(CurTick, NetEntityId)` seed

### Issue: All Random Values Same Within Tick
**Symptom:** Multiple random operations in same tick return identical results
**Cause:** Using only `CurTick` as seed, not including entity ID
**Solution:** Always combine both tick and entity ID in seed

### Issue: Predictable Random Patterns
**Symptom:** Cheaters can predict random outcomes
**Cause:** Using predicted randomness for high-value game mechanics
**Solution:** Keep valuable randomness server-only and unpredicted

## Best Practices
- Use deterministic `System.Random` with shared seeds for predicted RNG
- Always combine game tick and entity ID for unique seeds
- Keep high-value randomness unpredicted for security
- Test randomness behavior during prediction thoroughly
- Document which random elements are predicted vs server-authoritative