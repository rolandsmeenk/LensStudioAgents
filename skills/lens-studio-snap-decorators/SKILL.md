---
name: lens-studio-snap-decorators
description: Use TypeScript decorators to bind lifecycle events and inject dependencies declaratively.
---
# Snap Decorators
The `SnapDecorators` package provides a set of TypeScript decorators that eliminate boilerplate code for event binding and component lookups. Instead of manually creating events in `onAwake`, you can use declarative annotations on your class methods and properties.

## Lifecycle Event Binding
Use decorators to bind methods directly to Lens Studio lifecycle events.

```typescript
import {
  bindStartEvent,
  bindUpdateEvent,
  bindDestroyEvent
} from "SnapDecorators.lspkg/decorators";

@component
export class MyAdvancedScript extends BaseScriptComponent {
  
  @bindStartEvent
  onStart() {
    print("Lens started!");
  }

  @bindUpdateEvent
  onUpdate() {
    // Runs every frame
  }

  @bindDestroyEvent
  onCleanup() {
    print("Component destroyed");
  }
}
```

## Dependency Injection
Avoid repeated `getComponent` calls by using `@depends` and `@provider`.

```typescript
import { depends, provider } from "SnapDecorators.lspkg/decorators";

@component
export class MyController extends BaseScriptComponent {
  
  // Injects a component found on the same SceneObject
  @depends("Component.RenderMeshVisual")
  mesh!: RenderMeshVisual;

  // Injects a component found on this object or any parent object
  @provider("Component.Camera")
  camera!: Camera;

  @bindStartEvent
  init() {
    print("Found camera: " + this.camera.getSceneObject().name);
  }
}
```

## Lazy Evaluation with @memo
The `@memo` decorator caches the result of a getter the first time it is accessed.

```typescript
import { memo } from "SnapDecorators.lspkg/decorators";

@component
export class DataProcessor extends BaseScriptComponent {
  
  @memo
  get expensiveCalculation() {
    print("Running heavy math...");
    return Math.sqrt(Math.random() * 1000);
  }
}
```

## Reference Example
The code logic below is extracted from the official `SnapDecorators.lspkg` package, providing the implementation of the decorators themselves.

See [the reference guide](references/decorators.md) for details.
