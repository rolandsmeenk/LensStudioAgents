---
name: lens-studio-volumetric-drawing
description: Render smooth 3D tube geometry and splines dynamically for visual effects and drawing apps.
---
# Volumetric Drawing
Lens Studio's `VolumetricLine` package allows you to create high-quality 3D tubes by extruding circular cross-sections along a path. Unlike simple 2D lines, these have actual volumetric geometry that responds to lighting, can be capped, and uses spline interpolation for perfect smoothness.

## Creating a 3D Line
To define a path, you provide an array of `SceneObject` points.

```typescript
const line = this.sceneObject.getComponent("Component.VolumetricLine");

// Define the control points in the room
line.pathPoints = [pointStart, pointMid, pointEnd];

// Configure quality
line.radius = 2.0; // Thickness in cm
line.circleSegments = 16; // Hoop detail
line.interpolationSteps = 10; // Spline smoothness
line.smoothness = 0.5; // Tension

// Color and Material
line.material = myNeonMaterial;
line._color = new vec3(0, 1, 1); // Cyan
```

## Spline Interpolation (Catmull-Rom)
The package utilizes Catmull-Rom splines to ensure that the line passes smoothly through every control point without sharp corners.

```typescript
// Internal math pattern for spline calculation
function catmullRom(p0: vec3, p1: vec3, p2: vec3, p3: vec3, t: number) {
    // Spline tension math...
    return resultPos;
}
```

## Debug Gizmos
For visualizing spatial relationships without committing to full geometry, you can use the `Line` gizmo, which provides a performant 2D line that renders in 3D space with gradient support.

```typescript
const gizmo = this.sceneObject.getComponent("Component.Line");
gizmo.startPointObject = userHand;
gizmo.endPointObject = targetObject;
gizmo.beginColor = new vec3(1, 0, 0); // Red
gizmo.endColor = new vec3(0, 0, 1); // Blue
```

## Performance Tips
- **Interpolation Steps:** High values (>20) significantly increase vertex count. Stay low unless the path is very long.
- **Continuous Update:** If the points aren't moving, disable `continuousUpdate` to save CPU and GPU cycles.
- **Mesh Topology:** Default is set to Triangles. Use `RenderMeshVisual` to tint and shade.

## Reference Examples
The code logic below is extracted from the official `VolumetricLine.lspkg` and `RuntimeGizmos.lspkg` packages.

See the reference guides:
- [VolumetricLine.md](references/VolumetricLine.md)
- [Line.md](references/Line.md)
