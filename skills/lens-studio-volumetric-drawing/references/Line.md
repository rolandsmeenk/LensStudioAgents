# Line Reference

```typescript
/**
 * Runtime line gizmo for visualizing connections between objects.
 * Renders a customizable line with gradient colors and styles.
 */

@component
export class Line extends BaseScriptComponent {
  @input public startPointObject!: SceneObject
  @input public endPointObject!: SceneObject
  @input private lineMaterial!: Material
  @input public _beginColor: vec3 = new vec3(1, 1, 0)
  @input public _endColor: vec3 = new vec3(1, 1, 0)
  @input private lineWidth: number = 0.5

  private line!: InteractorLineRenderer

  onAwake() {
    this.line = new InteractorLineRenderer({
      material: this.lineMaterial,
      points: [
        this.startPointObject.getTransform().getLocalPosition(),
        this.endPointObject.getTransform().getLocalPosition()
      ],
      startColor: withAlpha(this._beginColor, 1),
      endColor: withAlpha(this._endColor, 1),
      startWidth: this.lineWidth,
      endWidth: this.lineWidth,
    })
    this.line.getSceneObject().setParent(this.sceneObject)
  }
  
  onUpdate() {
    if (!this.startPointObject || !this.endPointObject || !this.line) return
    
    this.line.points = [
      this.startPointObject.getTransform().getLocalPosition(),
      this.endPointObject.getTransform().getLocalPosition()
    ]
  }
}
```
