---
name: lens-studio-interactive-solvers
description: Implement proximity-based triggers and smooth tethering behaviors for interactive objects.
---
# Interactive Solvers
Solvers provide reusable mathematical logic for common AR interaction patterns. The two primary solvers available for Spectacles are **Distance Triggers** (for proximity events) and **Tethering** (for objects that follow the user or other targets).

## Proximity Triggers
Using `DistanceEventsTS`, you can define specific distance thresholds (in meters) that trigger callbacks when a user or object crosses them.

```typescript
// Define multiple thresholds
const myProximityTrigger = this.sceneObject.getComponent("Component.DistanceEventsTS");
myProximityTrigger.target = mainCamera;
myProximityTrigger.distances = [1.0, 3.0, 5.0];

// Handle entering a range
myProximityTrigger.onEnterRange = (distance) => {
    print(`User entered the ${distance}m zone!`);
};

// Handle exit
myProximityTrigger.onExitRange = (distance) => {
    print(`User left the ${distance}m zone.`);
};
```

## Tethering (Lazy Follow)
The `TetherTS` script implements a "constrained follow" behavior, where an object only repositions itself after the target has moved beyond a certain horizontal or vertical limit. This prevents objects from jittering or feeling too tightly bound to the user's head.

```typescript
const tether = this.sceneObject.getComponent("Component.TetherTS");
tether.target = mainCamera;
tether.offset = new vec3(0, -10, 50); // Relative position
tether.verticalDistanceFromTarget = 15.0; // Don't move if user only tilts head slightly
tether.horizontalDistanceFromTarget = 20.0; // Don't move if user only shifts slightly
tether.lerpSpeed = 5.0; // Smooth movement speed
```

## Smooth Movement Calculation
Interactive solvers typically use the following linear interpolation (lerp) pattern for smooth updates:

```typescript
const newPos = vec3.lerp(
    currentPos,
    targetGoalPos,
    lerpSpeed * getDeltaTime()
);
myTransform.setWorldPosition(newPos);
```

## Reference Example
The code logic below is extracted from the official `Solvers.lspkg` package, detailing the distance threshold and tethering math.

See the reference guides:
- [DistanceEventsTS.md](references/DistanceEventsTS.md)
- [TetherTS.md](references/TetherTS.md)
