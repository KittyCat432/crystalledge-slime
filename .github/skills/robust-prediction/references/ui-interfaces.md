# UI & User Interfaces in Prediction

## Predicted Popups

### The Problem
Using regular popup methods in predicted code causes popups to show multiple times during prediction.

### Popup APIs

**❌ Wrong - shows multiple times:**
```csharp
_popup.PopupEntity("Message", uid);
```

**✅ Correct - predicted popup:**
```csharp
_popup.PopupPredicted("Message", uid, user); // user = local player entity

// Other predicted variants
_popup.PopupClient("Message", uid, user); // Only for specified user  
_popup.PopupCursorPredicted("Message", coordinates, user);
_popup.PopupCoordinatesPredicted("Message", coordinates, user);
```

## Predicted Audio

### Audio APIs

**❌ Wrong - plays multiple times:**
```csharp
_audio.PlayEntity("/Audio/sound.ogg", uid);
```

**✅ Correct - predicted audio:**
```csharp
_audio.PlayPredicted("/Audio/sound.ogg", uid, user);
```

## Predicted Bound User Interfaces (BUIs)

Traditional BUIs send `BoundUserInterfaceState` for client updates. With prediction, this duplicates already-networked component data.

### Traditional BUI (Inefficient)
```csharp
// Server sends both component state AND BUI state (duplicate networking)
public sealed class TraditionalBuiState : BoundUserInterfaceState
{
    public string Data; // Same data already in networked component!
}
```

### Predicted BUI Pattern (Efficient)

**Remove BUI state entirely** - use component data directly:

```csharp
// 1. Component with AutoGenerateComponentState
[NetworkedComponent]
[AutoGenerateComponentState(true)] // raiseAfterAutoHandleState = true!
public sealed partial class YourComponent : Component
{
    [AutoNetworkedField]
    public string Data = string.Empty;
}

// 2. System updates UI when component state changes
public override void Initialize()
{
    SubscribeLocalEvent<YourComponent, AfterAutoHandleStateEvent>(OnAfterHandleState);
    SubscribeLocalEvent<EntInsertedIntoContainerMessage>(OnContainerChanged); // If needed
}

private void OnAfterHandleState(EntityUid uid, YourComponent comp, ref AfterAutoHandleStateEvent args)
{
    UpdateUi(uid, comp); // Update UI whenever component data changes
}

// 3. Shared virtual method for UI updates
public virtual void UpdateUi(EntityUid uid, YourComponent comp)
{
    // Base implementation - override in client system
}

// 4. Client system override
public override void UpdateUi(EntityUid uid, YourComponent comp)
{
    if (!TryComp<UserInterfaceComponent>(uid, out var uiComp))
        return;
        
    var bui = uiComp.GetBoundUserInterface(YourUiKey.Key);
    bui?.Update(); // Update UI with current component data
}

// 5. Use SendPredictedMessage for user input
public void OnButtonPressed()
{
    SendPredictedMessage(new YourBuiMessage()); // Not SendMessage!
}
```

### Key Differences from Traditional BUI
1. **No BUI State**: Remove `BoundUserInterfaceState` class entirely
2. **Component Data**: Read UI data directly from networked component using `TryComp`
3. **AfterAutoHandleState**: Update UI when component state changes (requires `raiseAfterAutoHandleState = true`)
4. **Virtual UpdateUi**: Shared method for UI updates, overridden in client
5. **SendPredictedMessage**: Use for user input instead of `SendMessage`
6. **Container Events**: Subscribe to insert/remove events if UI depends on container contents

### Example Implementation
See [Space Station 14 PR #33835](https://github.com/space-wizards/space-station-14/pull/33835) for complete predicted BUI example.

## Predicted API Constraints
- **Must know user entity** (local player) for popups/audio
- **Cannot predict in update loops** (no single user available)
- **Cannot predict in container events** (no user context)
- **Server assumes client predicted** - don't mix predicted/non-predicted calls

## Common UI Issues

### Issue: Popups Show Multiple Times  
**Symptom:** Same popup appears repeatedly
**Cause:** Using `PopupEntity` instead of predicted variant
**Solution:** Replace with `PopupPredicted(message, entity, user)`

### Issue: Audio Plays Repeatedly
**Symptom:** Sound effects play multiple times simultaneously
**Cause:** Using `PlayEntity` instead of predicted variant  
**Solution:** Replace with `PlayPredicted(sound, entity, user)`

### Issue: BUI Shows Stale Data During Prediction
**Symptom:** User interface doesn't update immediately with predicted changes
**Cause:** BUI using `BoundUserInterfaceState` instead of component data
**Solution:** Convert to predicted BUI using `AfterAutoHandleState` and `SendPredictedMessage`

## Best Practices
- Always use predicted popup/audio APIs in shared code
- Convert BUIs to use component data instead of duplicate BUI states
- Ensure UI updates during prediction with `AfterAutoHandleState`
- Use `SendPredictedMessage` for user input from predicted UIs
- Test UI responsiveness during various network conditions