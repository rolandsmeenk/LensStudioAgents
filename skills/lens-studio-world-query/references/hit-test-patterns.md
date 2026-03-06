# World Query — HitTest Patterns Reference

Sourced from `Essentials/Assets/MiniDemos/DirectionShadows/DirectionalWorldQuery.ts`.

## Full hit test session setup + per-frame update

```typescript
const WorldQueryModule = require('LensStudio:WorldQueryModule')

@component
export class SurfacePlacer extends BaseScriptComponent {
  @input raySource: SceneObject         // from here
  @input direction: SceneObject         // pointing at this
  @input indicator: SceneObject         // place here on hit
  @input rayLength: number = 100
  @input filterEnabled: boolean = true  // smooths jitter

  private hitTestSession: HitTestSession

  onAwake(): void {
    const options = HitTestSessionOptions.create()
    options.filter = this.filterEnabled
    this.hitTestSession = WorldQueryModule.createHitTestSessionWithOptions(options)

    this.createEvent('UpdateEvent').bind(this.onUpdate.bind(this))
  }

  private onUpdate(): void {
    const from = this.raySource.getTransform().getWorldPosition()
    const to   = this.direction.getTransform().getWorldPosition()
    const dir  = to.sub(from).normalize()
    const end  = from.add(dir.uniformScale(this.rayLength))

    this.hitTestSession.hitTest(from, end, (result) => {
      if (result === null) {
        this.indicator.enabled = false
        return
      }
      this.placeOnSurface(result.position, result.normal)
    })
  }

  private placeOnSurface(pos: vec3, normal: vec3): void {
    this.indicator.getTransform().setWorldPosition(pos)

    const EPSILON = 0.01
    const isHorizontal = 1 - Math.abs(normal.normalize().dot(vec3.up())) < EPSILON
    const lookDir = isHorizontal ? vec3.forward() : normal.cross(vec3.up())
    this.indicator.getTransform().setWorldRotation(quat.lookAt(lookDir, normal))
    this.indicator.enabled = true
  }
}
```

## Retry with multiple ray lengths (if surfaces are far away)

```typescript
function tryHitWithMultipleLengths(
  session: HitTestSession,
  from: vec3,
  dir: vec3,
  lengths: number[],
  onHit: (result: HitTestResult) => void
): void {
  for (const length of lengths) {
    const end = from.add(dir.uniformScale(length))
    session.hitTest(from, end, (result) => {
      if (result !== null) onHit(result)
    })
  }
}

// Usage
tryHitWithMultipleLengths(session, origin, forward, [1, 2, 5, 10], placeObject)
```

## Physics probe — scene collider raycast (not real world)

```typescript
// From Essentials/Assets/Raycast/SimpleRaycastTS.ts
@component
export class SimpleRaycastTS extends BaseScriptComponent {
  @input rayStart: SceneObject
  @input rayEnd: SceneObject
  @input endPointAttachment: SceneObject

  onAwake() {
    this.createEvent('UpdateEvent').bind(() => this.doRaycast())
  }

  private doRaycast(): void {
    const probe = Physics.createGlobalProbe()
    const self = this

    probe.rayCast(
      this.rayStart.getTransform().getWorldPosition(),
      this.rayEnd.getTransform().getWorldPosition(),
      function(hit) {
        if (hit && self.endPointAttachment) {
          self.endPointAttachment.getTransform().setWorldPosition(hit.position)
          print('Hit: ' + hit.collider.getSceneObject().name)
        }
      }
    )
  }
}
```

