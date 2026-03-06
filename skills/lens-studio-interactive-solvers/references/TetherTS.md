# TetherTS Reference

```typescript
/**
 * Basic tethering system with distance-based repositioning.
 * Follows target with automatic repositioning when vertical or horizontal distance thresholds are exceeded.
 */

@component
export class TetherTS extends BaseScriptComponent {
    @input target!: SceneObject;
    @input offset: vec3 = new vec3(0, 0, 0);
    @input lerpSpeed: number = 5.0;

    @input verticalDistanceFromTarget: number = 0.1;
    @input horizontalDistanceFromTarget: number = 0.1;

    private _targetPosition: vec3 = new vec3(0, 0, 0);
    
    onUpdate(): void {
        if (!this.target) return;
        
        if (this.shouldReposition()) {
            this._targetPosition = this.calculateNewTargetPosition();
        }
        
        this.updateContentPosition();
    }
    
    private shouldReposition(): boolean {
        const myPos = this.sceneObject.getTransform().getWorldPosition();
        const targetPos = this.target.getTransform().getWorldPosition();
        
        const toTarget = myPos.sub(targetPos);
        
        const verticalDistance = Math.abs(toTarget.y);
        const horizontalDistance = Math.sqrt(toTarget.x * toTarget.x + toTarget.z * toTarget.z);
        
        return verticalDistance > this.verticalDistanceFromTarget || 
               horizontalDistance > this.horizontalDistanceFromTarget;
    }
    
    private updateContentPosition(): void {
        const myTransform = this.sceneObject.getTransform();
        const currentPos = myTransform.getWorldPosition();
        
        const newPos = vec3.lerp(
            currentPos,
            this._targetPosition,
            this.lerpSpeed * getDeltaTime()
        );
        
        myTransform.setWorldPosition(newPos);
    }
}
```
