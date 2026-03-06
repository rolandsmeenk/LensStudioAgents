# InstantiateOn3DGridsTS Reference

```typescript
/**
 * 3D grid instantiation utility that creates volumetric grid patterns in three-dimensional space.
 * Ideal for creating voxel-based structures, particle volumes, or 3D data visualizations.
 */
@component
export class InstantiateOn3DGridsTS extends BaseScriptComponent {
    @input prefab: ObjectPrefab;
    @input gridCenter: SceneObject;

    @input gridWidth: number = 5;
    @input gridHeight: number = 5;
    @input gridDepth: number = 5;

    @input spacingX: number = 1.5;
    @input spacingY: number = 1.5;
    @input spacingZ: number = 1.5;
    
    onAwake(): void {
        this.createEvent("OnStartEvent").bind(() => {
            this.instantiateGrid();
        });
    }
    
    instantiateGrid(): void {
        const totalWidth = (this.gridWidth - 1) * this.spacingX;
        const totalHeight = (this.gridHeight - 1) * this.spacingY;
        const totalDepth = (this.gridDepth - 1) * this.spacingZ;
        
        const centerPosition = this.gridCenter.getTransform().getWorldPosition();
        const startPosition = new vec3(
            centerPosition.x - totalWidth / 2,
            centerPosition.y - totalHeight / 2,
            centerPosition.z - totalDepth / 2
        );
        
        for (let x = 0; x < this.gridWidth; x++) {
            for (let y = 0; y < this.gridHeight; y++) {
                for (let z = 0; z < this.gridDepth; z++) {
                    const position = new vec3(
                        startPosition.x + x * this.spacingX,
                        startPosition.y + y * this.spacingY,
                        startPosition.z + z * this.spacingZ
                    );
                    
                    this.createPrefabInstance(position);
                }
            }
        }
    }
    
    private createPrefabInstance(position: vec3): void {
        if (this.prefab) {
            const instance = this.prefab.instantiate(this.sceneObject);
            instance.getTransform().setWorldPosition(position);
        }
    }
}
```
