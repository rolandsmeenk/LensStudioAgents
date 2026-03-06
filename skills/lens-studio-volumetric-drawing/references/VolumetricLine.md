# VolumetricLine Reference

```typescript
/**
 * Enhanced 3D volumetric line component with Catmull-Rom spline interpolation.
 * Creates smooth tube geometry by extruding circular cross-sections along a path.
 */

@component
export class VolumetricLine extends BaseScriptComponent {
  @input public pathPoints: SceneObject[];
  @input private _radius: number = 10.0;
  @input private _circleSegments: number = 16;
  @input private _interpolationSteps: number = 10;
  @input private _smoothness: number = 0.5;
  @input public material: Material;
  @input public _color: vec3 = new vec3(1, 1, 0);

  onStart(): void {
    if (!this.pathPoints || this.pathPoints.length < 2) return;
    this.setupMeshVisual();
    this.generateMesh();
  }

  private generateMesh(): void {
    const pathPositions = this.getPathPositions();
    this.generateTubeGeometry(pathPositions);
    // ... apply to RenderMeshVisual
  }

  private catmullRomSpline(p0: vec3, p1: vec3, p2: vec3, p3: vec3, t: number, tension: number): vec3 {
    const t2 = t * t;
    const t3 = t2 * t;
    const tensionFactor = tension * 0.5;

    const v0 = p2.sub(p0).uniformScale(tensionFactor);
    const v1 = p3.sub(p1).uniformScale(tensionFactor);

    const a = p1.uniformScale(2).sub(p2.uniformScale(2)).add(v0).add(v1);
    const b = p2.uniformScale(3).sub(p1.uniformScale(3)).sub(v0.uniformScale(2)).sub(v1);
    const c = v0;
    const d = p1;

    return a.uniformScale(t3).add(b.uniformScale(t2)).add(c.uniformScale(t)).add(d);
  }
}
```
