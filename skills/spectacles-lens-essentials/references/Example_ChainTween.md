# Example_ChainTween Reference

```typescript
/**
 * Specs Inc. 2026
 * Example demonstrating chained tween animations. Shows how to sequence multiple tweens
 * where one animation starts automatically after the previous one completes.
 */
import { LSTween } from "./LSTween";
import { RotationInterpolationType } from "./RotationInterpolationType";

@component
export class Example_ChainTween extends BaseScriptComponent {
  onAwake() {
    const transform = this.getTransform();
    const initScale = transform.getLocalScale();

    const rotateTween = LSTween.rotateOffset(
      transform,
      quat.angleAxis(MathUtils.DegToRad * 45, vec3.up()),
      1000,
      RotationInterpolationType.LERP
    ).onEveryStart((o) => {
      transform.setLocalScale(initScale);
    });

    const scaleTween = LSTween.scaleOffset(
      transform,
      vec3.one().uniformScale(2),
      1000
    );

    rotateTween.chain(scaleTween);
    scaleTween.chain(rotateTween);

    rotateTween.start();
  }
}

```

