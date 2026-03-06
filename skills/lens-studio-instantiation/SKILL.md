---
name: lens-studio-instantiation
description: Learn how to dynamically instantiate prefabs algorithmically in grid or spherical patterns.
---
# Programmatic Instantiation
In Lens Studio, standard objects are placed visually in the Hierarchy. However, for dynamic systems like particle generators, voxels, UI grids, or data visualization, you will need to rapidly spawn AR elements programmatically into specific volumetric geometric patterns at runtime using `instantiate()`.

**Official docs:** [Spectacles Home](https://developers.snap.com/spectacles/home)

## Generating Prefabs
To spawn a new object, you apply `.instantiate()` against a valid `ObjectPrefab` asset field.

```typescript
@input
targetPrefab: ObjectPrefab;

// Spawn attached to the current script root
const newObject = this.targetPrefab.instantiate(this.sceneObject);

// Re-position via the Transform
newObject.getTransform().setWorldPosition(new vec3(10, 0, 0));
```

## Pattern: Spawning in a 3D Volumetric Grid
If you want to spawn `X * Y * Z` number of objects centering radially outwards in a voxel cube...

```typescript
@input myPrefab: ObjectPrefab;

const gridWidth = 5;
const gridHeight = 5;
const gridDepth = 5;
const spacing = 1.0; 

function buildGrid(centerPos: vec3) {
    // 1. Calculate boundaries
    const totalW = (gridWidth - 1) * spacing;
    const totalH = (gridHeight - 1) * spacing;
    const totalD = (gridDepth - 1) * spacing;
    
    // 2. Compute Offset
    const startPos = new vec3(
        centerPos.x - (totalW / 2),
        centerPos.y - (totalH / 2),
        centerPos.z - (totalD / 2)
    );
    
    // 3. Loop along axes
    for (let x = 0; x < gridWidth; x++) {
       for (let y = 0; y < gridHeight; y++) {
          for (let z = 0; z < gridDepth; z++) {
              
              const spawnTarget = new vec3(
                  startPos.x + (x * spacing),
                  startPos.y + (y * spacing),
                  startPos.z + (z * spacing)
              );
              
              // 4. Fire spawn
              const instance = myPrefab.instantiate(global.scene.getRootObject(0));
              instance.getTransform().setWorldPosition(spawnTarget);
          }
       }
    }
}
```

## Other Mathematical Placements
The same math logic can be configured to execute other common shapes using trigonometric algorithms instead of linear `for` loops. Look for existing TS implementation scripts inside `Instantiation.lspkg` in the official samples:

1. **InstantiateOn2DGrids** - Wall displays and UI.
2. **CirclePerimeterInstantiator** - Spawns objects radially using `Math.sin(angle)` and `Math.cos(angle)`.
3. **RandomPointsInsideSphere** - Calculating a random radius with `Math.pow(Math.random(), 1/3) * maxRadius`.

## Reference Example
The code logic below is extracted from the official `Instantiation.lspkg` package, demonstrating a production-ready volumetric 3D grid layout behaviour.

See [the reference guide](references/InstantiateOn3DGridsTS.md) for details.
