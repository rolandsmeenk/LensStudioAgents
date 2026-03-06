# DistanceEventsTS Reference

```typescript
/**
 * Distance-based event system for triggering callbacks at configurable distance thresholds.
 * Monitors distance to target, supports multiple threshold ranges with enter/exit/percent events,
 * configurable greater-than or less-than triggering.
 */

@component
export class DistanceEventsTS extends BaseScriptComponent {
    @input target!: SceneObject;
    @input distances: number[] = [];
    @input triggerOnGreaterThan: boolean = true;

    private _ranges: DistanceRange[] = [];
    private _distanceToTarget: number = 0;
    
    onUpdate(): void {
        if (!this.target) return;
        
        const myPosition = this.sceneObject.getTransform().getWorldPosition();
        const targetPosition = this.target.getTransform().getWorldPosition();
        
        this._distanceToTarget = targetPosition.distance(myPosition);
        
        // Update all registers
        for (const range of this._ranges) {
            range.update(this._distanceToTarget);
        }
    }
}

export class DistanceRange {
    private _insideRange: boolean = false;
    private _wasInsideRange: boolean = false;
    
    update(distance: number): void {
        this._insideRange = (distance >= this.minDistance && distance <= this.maxDistance);
        
        if (this._insideRange && !this._wasInsideRange) {
            this.triggerOnEnterRange();
        }
        
        if (!this._insideRange && this._wasInsideRange) {
            this.triggerOnExitRange();
        }
        
        this._wasInsideRange = this._insideRange;
    }
}
```
