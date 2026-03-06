# Math Recipes — Runtime Gizmos and Geometry Reference

Sourced from `Essentials/Assets/RuntimeGizmos/` and `Essentials/Assets/MiniDemos/`.

## DotProduct demo — angle between gaze and object

```typescript
// Essentials/Assets/MiniDemos/DotProduct/TS/DotProductDemoTS.ts
// Shows how dot product gives the cosine of angle between two directions

function isLookingAt(camera: SceneObject, target: SceneObject, threshold: number = 0.9): boolean {
  const camForward = camera.getTransform().forward
  const toTarget = target.getTransform().getWorldPosition()
    .sub(camera.getTransform().getWorldPosition())
    .normalize()
  const dot = camForward.dot(toTarget)
  return dot > threshold  // 0.9 ≈ within ~26° cone
}
```

## ScaleBasedOnDistance — scale to maintain apparent size

```typescript
// Essentials/Assets/MiniDemos/DirectionShadows/ScaleBasedOnDistance.ts
@component
export class ScaleBasedOnDistance extends BaseScriptComponent {
  @input reference: SceneObject   // the camera or anchor point
  @input baseScale: vec3 = vec3.one()
  @input scaleFactor: number = 1.0

  onAwake() {
    this.createEvent('UpdateEvent').bind(() => {
      const dist = this.sceneObject.getTransform().getWorldPosition()
        .distance(this.reference.getTransform().getWorldPosition())
      const s = this.baseScale.uniformScale(dist * this.scaleFactor)
      this.sceneObject.getTransform().setLocalScale(s)
    })
  }
}
```

## Runtime line / polyline drawing

```typescript
// Essentials/Assets/RuntimeGizmos/Line.ts pattern
// Draw a world-space line segment each frame using a thin box mesh:

function drawLine(start: vec3, end: vec3, lineObj: SceneObject, thickness: number = 0.002): void {
  const mid = start.add(end).uniformScale(0.5)
  const len = start.distance(end)
  const dir = end.sub(start).normalize()

  lineObj.getTransform().setWorldPosition(mid)
  lineObj.getTransform().setWorldRotation(quat.lookAt(dir, vec3.up()))
  lineObj.getTransform().setLocalScale(new vec3(thickness, thickness, len))
}
```

## Drag and Drop (DragAndDrop.ts)

```typescript
// Essentials/Assets/MiniDemos/DragAndDrop/DragAndDrop.ts
// Key pattern: track whether pinch went down ON the object

@component
export class DragAndDrop extends BaseScriptComponent {
  private isDragging = false
  private dragOffset: vec3 = vec3.zero()

  // Hook up to SIK interactable's onDragStart / onDragUpdate callbacks:
  onDragStart(handPos: vec3): void {
    this.isDragging = true
    this.dragOffset = this.sceneObject.getTransform().getWorldPosition().sub(handPos)
  }

  onDragUpdate(handPos: vec3): void {
    if (!this.isDragging) return
    this.sceneObject.getTransform().setWorldPosition(handPos.add(this.dragOffset))
  }

  onDragEnd(): void {
    this.isDragging = false
  }
}
```

