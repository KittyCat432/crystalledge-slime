# Security & Session Management

## Session-Specific Networking

By default, all clients receive full component information within PVS range. Cheaters may exploit this, so restrict sensitive information appropriately.

## SendOnlyToOwner

Component only networked to player attached to the entity.

```csharp
[NetworkedComponent]
public sealed partial class PacifiedComponent : Component
{
    // This component's SendOnlyToOwner = true in constructor or via attribute
    // Only the pacified player knows about their pacification status
    // Other players don't need this for prediction (can't predict others' inputs)
}
```

**Use cases:**
- Traitor abilities only user needs to know
- Personal traits/effects (pacified, diseases, etc.)
- Private player status information
- Character-specific abilities or limitations

**Implementation:**
```csharp
public override void Initialize()
{
    base.Initialize();
    SendOnlyToOwner = true; // Set in component constructor
}
```

## SessionSpecific

Raises `ComponentGetStateAttemptEvent` for each player, allows custom networking logic with fine-grained control.

```csharp
[NetworkedComponent] 
public sealed partial class RevolutionaryComponent : Component
{
    // SessionSpecific = true enables per-player networking decisions
}

// In system:
public override void Initialize()
{
    SubscribeLocalEvent<RevolutionaryComponent, ComponentGetStateAttemptEvent>(OnGetStateAttempt);
}

private void OnGetStateAttempt(EntityUid uid, RevolutionaryComponent comp, ref ComponentGetStateAttemptEvent args)
{
    // Only network to revolutionaries and admins
    if (!IsRevolutionary(args.Player) && !IsAdmin(args.Player))
        args.Cancelled = true;
}
```

**Performance warning:** Raises many events (one per player per component), use sparingly!

**Future improvement needed:** Current API requires lots of boilerplate. Could be simplified to component whitelists.

### Example: Conditional Visibility
```csharp
private void OnGetStateAttempt(EntityUid uid, SomeSecretComponent comp, ref ComponentGetStateAttemptEvent args)
{
    var player = args.Player;
    
    // Multiple conditions for visibility
    if (HasComp<AdminComponent>(player.AttachedEntity) ||  // Admins can always see
        HasComp<SameTeamComponent>(player.AttachedEntity) || // Team members
        uid == player.AttachedEntity) // Owner themselves
    {
        return; // Allow networking
    }
    
    args.Cancelled = true; // Hide from others
}
```

## Security Best Practices

### Information Classification
**Classify data by security level:**

1. **Public** - Everyone should know (health, position, most gameplay data)
2. **Owner-only** - Only the player needs to know (personal abilities, status effects)  
3. **Team/Role-specific** - Only certain players need to know (revolutionary list, team objectives)
4. **Admin-only** - Only administrators should see (debug info, sensitive data)

### Cheating Prevention Strategies

#### Don't Network What You Don't Need
```csharp
// ❌ BAD - Networks sensitive data unnecessarily
[AutoNetworkedField]
public bool IsTraitor = false; // Everyone can see this!

// ✅ GOOD - Keep server-side only if clients don't need it
// Don't network at all if prediction doesn't require it
public bool IsTraitor = false; // Not networked
```

#### Use Prediction-Safe Alternatives
```csharp
// Instead of networking sensitive flags, derive them from safe data
public bool CanUseTool => HasComp<ToolUserComponent>(uid); // Safe to network
// Server-side: Add/remove ToolUserComponent based on traitor status
```

#### Separate Client and Server Data
```csharp
[NetworkedComponent]
public sealed partial class WeaponComponent : Component
{
    [AutoNetworkedField]
    public float Damage = 10f; // Safe to network
    
    // DON'T network sensitive modifiers - keep server-side
    public float TraitorDamageMultiplier = 1.5f; // Server calculates final damage
}
```

### Common Security Issues

#### Issue: Sensitive Information Leaked to Clients
**Symptom:** Cheaters can see antagonist status or hidden information
**Cause:** All components networked to all clients in PVS by default
**Solution:** Use `SendOnlyToOwner` or `SessionSpecific` to restrict component visibility

**Example Fix:**
```csharp
// Before: Everyone knows who's a traitor
[NetworkedComponent] // ❌ Networks to everyone
public sealed partial class TraitorComponent : Component { }

// After: Only the traitor knows
[NetworkedComponent]
public sealed partial class TraitorComponent : Component
{
    public TraitorComponent()
    {
        SendOnlyToOwner = true; // ✅ Only traitor sees this
    }
}
```

#### Issue: Team Information Exposure
**Symptom:** Players can determine team composition through client inspection
**Cause:** Team-specific components visible to all players
**Solution:** Use `SessionSpecific` with team-based visibility logic

#### Issue: Prediction Bypasses Security
**Symptom:** Cheaters can predict restricted actions by modifying client
**Cause:** Security checks only on server, not in shared prediction code
**Solution:** Implement security checks in shared code that are prediction-safe

```csharp
// Shared security check - runs on both client and server
public bool CanPerformAction(EntityUid user)
{
    // Use only information that's safe to network
    return HasComp<AuthorizedComponent>(user) && 
           !HasComp<RestrictedComponent>(user);
}
```

## Security Guidelines for Prediction

### Safe Information to Network
- Player position and movement
- Public health status
- Inventory contents (usually)
- Visual appearance data
- Public interaction capabilities

### Dangerous Information to Restrict
- Antagonist status and team membership
- Hidden abilities or knowledge
- Admin flags and permissions  
- Security clearance levels
- Private objectives and goals
- Cheat detection data

### Prediction Security Patterns

#### Server Authority Pattern
```csharp
// Client predicts optimistically, server validates and corrects
public void TryPerformAction(EntityUid user)
{
    // Client side: Use only publicly visible data for prediction
    if (!CanPerformBasicAction(user))
        return;
        
    // Server side: Additional secret checks
    if (_net.IsServer && !HasSecretPermission(user))
    {
        // Reject and sync client
        return;
    }
    
    // Perform action
}
```

#### Safe Capability Broadcasting
```csharp
// Instead of networking "IsTraitor", network capabilities
[AutoNetworkedField]
public bool CanUseSyndieTools = false; // Safe capability flag

// Server sets this based on traitor status, clients only see capability
private void UpdateCapabilities(EntityUid uid)
{
    var comp = Comp<CapabilitiesComponent>(uid);
    comp.CanUseSyndieTools = HasComp<TraitorComponent>(uid);
    Dirty(uid, comp);
}
```

## Best Practices
- Classify all networked data by security sensitivity
- Use `SendOnlyToOwner` for personal information
- Use `SessionSpecific` for team/role-based visibility
- Don't network data that clients don't need for prediction
- Implement server authority for security-critical actions
- Test security measures against client modification attempts
- Document which components have restricted visibility
- Review networking decisions with security implications in mind