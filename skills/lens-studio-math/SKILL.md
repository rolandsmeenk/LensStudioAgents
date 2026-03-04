---
name: lens-studio-math
description: Reference guide for Lens Studio's math types and utilities — vec2/vec3/vec4/quat/mat4, MathUtils (DegToRad, RadToDeg, clamp, remap, lerp, inverseLerp), and Lens Studio's right-handed coordinate system (vec3.forward()=(0,0,-1)). Covers construction, arithmetic (uniformScale, dot, cross, normalize, distance), quaternion creation from Euler angles (pitch/yaw/roll XYZ order), quat.lookAt with parallel-vector guard, quat.slerp, combining rotations with multiply, getWorldTransform/setWorldTransform, and practical recipes: billboard, frame-rate-independent smooth follow, angle between directions, world-to-screen UV, project-to-plane, color lerp via vec4, and parsing BLE sensor quaternion bytes. Use this skill for any 3D math, position/rotation/scale arithmetic, or coordinate-space conversion in a Lens Studio TypeScript script.
---

# Lens Studio Math — Reference Guide

Lens Studio scripts use custom math types rather than plain JavaScript numbers. This guide covers the most common types and operations.

---

## vec3 — 3D Vector

### Construction
```typescript
const a = new vec3(1, 0, 0)       // explicit
const b = vec3.zero()              // (0, 0, 0)
const c = vec3.one()               // (1, 1, 1)
const up = vec3.up()               // (0, 1, 0)
const forward = vec3.forward()     // (0, 0, -1) — Lens Studio uses right-hand Z-forward
const right = vec3.right()         // (1, 0, 0)
```

### Arithmetic
```typescript
const sum  = a.add(b)
const diff = a.sub(b)
const scaled = a.uniformScale(2.5)    // multiply all components by scalar
const divided = a.uniformScale(1/2.5) // divide
const negated = a.uniformScale(-1)

// Component-wise multiply
const product = new vec3(a.x * b.x, a.y * b.y, a.z * b.z)
```

### Geometry
```typescript
const len = a.length              // magnitude
const unit = a.normalize()        // unit vector (same direction, length 1)
const dot  = a.dot(b)             // scalar: angle relationship, projection
const cross = a.cross(b)          // perpendicular vector (right-hand rule)
const dist = a.distance(b)        // Euclidean distance

// Lerp (linear interpolation) — how Lens Studio does it:
import { mix } from 'SpectaclesInteractionKit.lspkg/Utils/animate'
const lerped = mix(a, b, t)       // t = 0 → a, t = 1 → b
// Or manually:
const lerped = a.add(b.sub(a).uniformScale(t))
```

---

## vec2 — 2D Vector

```typescript
const screen = new vec2(0.5, 0.5)   // centre of screen in normalised coords
const offset = screen.add(new vec2(0.1, 0))
const mag = screen.length
```

---

## vec4 — 4D Vector (colours, homogeneous coords)

```typescript
// Colour (RGBA, components in [0, 1])
const red   = new vec4(1, 0, 0, 1)
const clear = new vec4(0, 0, 0, 0)

// Apply to a material
material.mainPass.baseColor = new vec4(0.2, 0.8, 0.4, 1.0)
material.mainPass.opacity   = 0.5
```

---

## quat — Quaternion (rotation)

### Construction
```typescript
// From Euler angles (degrees → convert to radians first)
const DEG = Math.PI / 180
const rot = quat.fromEulerAngles(45 * DEG, 0, 0)  // pitch, yaw, roll

// Using MathUtils helper
const rot2 = quat.fromEulerAngles(
  30 * MathUtils.DegToRad,
  90 * MathUtils.DegToRad,
  0
)

// Identity (no rotation)
const identity = quat.quatIdentity()

// Look at a direction
const lookRot = quat.lookAt(forward, up)  // forward = where to point, up = world up

// From axis + angle
const axisRot = quat.angleAxis(45 * MathUtils.DegToRad, vec3.up())
```

### Combining rotations
```typescript
// Compose: apply rotA then rotB
const combined = rotB.multiply(rotA)

// Invert
const inv = rot.invert()

// Spherical interpolation (smooth rotation)
const slerpd = quat.slerp(startRot, endRot, t)  // t in [0, 1]
```

### Extracting info
```typescript
// Get the forward direction of a rotation
const fwd = rot.multiplyVec3(vec3.forward())

// Get Euler angles back
const euler = rot.toEulerAngles()  // returns vec3 in radians
const yawDeg = euler.y * (180 / Math.PI)
```

---

## MathUtils

```typescript
// Constants
MathUtils.DegToRad   // π / 180
MathUtils.RadToDeg   // 180 / π
MathUtils.PI         // Math.PI
MathUtils.TwoPI      // 2 * Math.PI
MathUtils.HalfPI     // Math.PI / 2

// Clamp a value to a range
const clamped = MathUtils.clamp(value, 0, 1)

// Remap a value from one range to another
const remapped = MathUtils.remap(value, inMin, inMax, outMin, outMax)

// Linear interpolation (scalars)
const t = MathUtils.lerp(a, b, alpha)

// Inverse lerp (find t given a value)
const alpha = MathUtils.inverseLerp(a, b, value)
```

---

## Transform: Position, Rotation, Scale

```typescript
const t = sceneObject.getTransform()

// World space
const worldPos = t.getWorldPosition()           // vec3
const worldRot = t.getWorldRotation()           // quat
const worldScale = t.getWorldScale()            // vec3
t.setWorldPosition(new vec3(0, 1, -2))
t.setWorldRotation(quat.quatIdentity())
t.setWorldScale(vec3.one())

// Local space (relative to parent)
const localPos = t.getLocalPosition()
t.setLocalPosition(new vec3(0, 0.5, 0))
t.setLocalRotation(quat.fromEulerAngles(0, 45 * MathUtils.DegToRad, 0))
t.setLocalScale(new vec3(2, 2, 2))

// Apply a full transform matrix
t.setWorldTransform(someMatrix)
const matrix = t.getWorldTransform()           // mat4
```

---

## Practical Recipes

### Billboard (always face camera)
```typescript
// In UpdateEvent:
const camPos = mainCamera.getSceneObject().getTransform().getWorldPosition()
const objPos = this.sceneObject.getTransform().getWorldPosition()
const dir = camPos.sub(objPos).normalize()
const rot = quat.lookAt(dir, vec3.up())
this.sceneObject.getTransform().setWorldRotation(rot)
```

### Smooth follow (lerp toward target)
```typescript
const SPEED = 5 // units / second
// In UpdateEvent:
const current = this.sceneObject.getTransform().getWorldPosition()
const target  = this.targetObject.getTransform().getWorldPosition()
const alpha = MathUtils.clamp(getDeltaTime() * SPEED, 0, 1)
this.sceneObject.getTransform().setWorldPosition(mix(current, target, alpha))
```

### Angle between two directions
```typescript
function angleBetween(a: vec3, b: vec3): number {
  const dot = a.normalize().dot(b.normalize())
  return Math.acos(MathUtils.clamp(dot, -1, 1)) * MathUtils.RadToDeg
}
```

### Project world position to screen UV
```typescript
// Convert a world point to a normalised screen position [0,1]
function worldToScreenUV(cam: Camera, worldPos: vec3): vec2 {
  const screenPos = cam.worldSpaceToScreenSpace(worldPos)
  return new vec2(
    screenPos.x / screen.getWidth(),
    screenPos.y / screen.getHeight()
  )
}
```

### Flatten a vector to the XZ plane (ignore Y)
```typescript
function flattenXZ(v: vec3): vec3 {
  return new vec3(v.x, 0, v.z).normalize()
}
```

---

## Common Gotchas

- **`vec3.forward()` is `(0, 0, -1)`** in Lens Studio (right-handed, Z pointing toward viewer). Don't assume `(0, 0, 1)`.
- **Never modify vec3/quat in-place** — most ops return new instances. `a.add(b)` returns a new vec3; `a` is unchanged.
- **Euler angle order** in `quat.fromEulerAngles(x, y, z)` is pitch, yaw, roll (XYZ). Mixing up the order causes unexpected rotations.
- **`quat.lookAt(forward, up)` is frame-dependent** — if `forward` is nearly parallel to `up`, the rotation becomes undefined. Guard with a dot product check.
- **Gimbal lock** doesn't affect quaternions, but converting to/from Euler angles can still produce surprising results for rotations near ±90° on X.
- **`getDeltaTime()`** returns seconds as a float — multiply by it for frame-rate-independent speeds.
