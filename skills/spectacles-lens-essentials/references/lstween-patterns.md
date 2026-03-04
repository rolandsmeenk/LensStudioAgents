# LSTween — Animation Patterns Reference

Sourced from `Essentials/Assets/Animations/Example_PrefabInstantiateTween.ts`.

## Scale + Rotation tween on a spawned prefab

```typescript
import { LSTween } from 'LSTween.lspkg/LSTween'
import { RotationInterpolationType } from 'LSTween.lspkg/RotationInterpolationType'
import Easing from 'LSTween.lspkg/TweenJS/Easing'

@component
export class SpawnWithTween extends BaseScriptComponent {
  @input prefab: ObjectPrefab
  @input minScale: vec3 = new vec3(0.5, 0.5, 0.5)
  @input maxScale: vec3 = new vec3(2, 2, 2)
  @input minRotDeg: vec3 = new vec3(-45, -45, -45)
  @input maxRotDeg: vec3 = new vec3(45, 45, 45)
  @input durationMs: number = 1500

  onAwake(): void {
    if (!this.prefab) return
    const inst = this.prefab.instantiate(null)
    const t = inst.getTransform()
    t.setLocalScale(this.minScale)
    t.setLocalPosition(vec3.zero())
    this.animateScale(t)
    this.animateRotation(t)
  }

  private animateScale(t: Transform): void {
    LSTween.scaleFromToLocal(t, this.minScale, this.maxScale, this.durationMs)
      .easing(Easing.Circular.InOut)
      .onStart(() => print('Scale start'))
      .onComplete(() => print('Scale done'))
      .start()
  }

  private animateRotation(t: Transform): void {
    const DEG = MathUtils.DegToRad
    const startRot = quat.fromEulerAngles(
      this.minRotDeg.x * DEG, this.minRotDeg.y * DEG, this.minRotDeg.z * DEG)
    const endRot = quat.fromEulerAngles(
      this.maxRotDeg.x * DEG, this.maxRotDeg.y * DEG, this.maxRotDeg.z * DEG)

    t.setLocalRotation(startRot)
    LSTween.rotateFromToLocal(t, startRot, endRot, this.durationMs, RotationInterpolationType.SLERP)
      .easing(Easing.Cubic.In)
      .onComplete(() => print('Rotation done'))
      .start()
  }
}
```

## Common LSTween API surface

```typescript
// Position (world)
LSTween.moveToWorld(sceneObj, targetPos, durationMs).easing(Easing.Quadratic.Out).start()

// Position (local)
LSTween.moveToLocal(sceneObj, localPos, durationMs).start()

// Scale
LSTween.scaleTo(sceneObj, new vec3(1,1,1), durationMs).start()
LSTween.scaleFromToLocal(transform, from, to, durationMs).start()

// Rotation
LSTween.rotateFromToLocal(transform, startQuat, endQuat, durationMs, RotationInterpolationType.SLERP).start()

// Colour / opacity (screen image or material)
LSTween.colorTo(screenImage, new vec4(1,1,1,0), durationMs).start()  // fade out

// Chain tweens
LSTween.moveTo(obj, posA, 500)
  .onComplete(() => LSTween.moveTo(obj, posB, 500).start())
  .start()
```

## Available easing functions

```
Easing.Linear.None
Easing.Quadratic.{In, Out, InOut}
Easing.Cubic.{In, Out, InOut}
Easing.Circular.{In, Out, InOut}
Easing.Elastic.{In, Out, InOut}
Easing.Back.{In, Out, InOut}
Easing.Bounce.{In, Out, InOut}
```
