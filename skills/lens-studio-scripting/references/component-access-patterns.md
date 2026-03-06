# Lens Studio TypeScript — Component Cross-Access Patterns

Sourced from `Essentials/Assets/AccessComponents/`.

## TS-to-TS (same project)

```typescript
// ComponentA.ts
@component
export class ComponentA extends BaseScriptComponent {
  getValue(): number { return 42 }
}

// ComponentB.ts
import { ComponentA } from './ComponentA'

@component
export class ComponentB extends BaseScriptComponent {
  @input otherObject: SceneObject

  onAwake() {
    this.createEvent('OnStartEvent').bind(() => {
      const a = this.otherObject.getComponent(
        ComponentA.getTypeName()
      ) as unknown as ComponentA
      print('Got value: ' + a.getValue())
    })
  }
}
```

## TS-to-JS (accessing a JS component from TS)

### Option A — Declaration file
Create `MyJSComponent.d.ts`:
```typescript
declare class MyJSComponent {
  myMethod(): void
  myValue: number
}
```
Then in TS:
```typescript
const jsComp = this.sceneObject.getComponent('ScriptComponent') as unknown as MyJSComponent
jsComp.myMethod()
```

### Option B — No declaration (unsafe cast)
```typescript
const jsComp: any = childObject.getComponent('ScriptComponent')
jsComp.myMethod()
```

## Accessing a component on a child SceneObject

```typescript
// Essentials/Assets/AccessComponents/AccessComponentOnChildSceneObject
@component
export class AccessChildComponent extends BaseScriptComponent {
  @input targetChild: SceneObject

  onAwake() {
    this.createEvent('OnStartEvent').bind(() => {
      // By component type string
      const audio = this.targetChild.getComponent('Component.AudioComponent')

      // Or iterate children to find by name
      const count = this.sceneObject.getChildrenCount()
      for (let i = 0; i < count; i++) {
        const child = this.sceneObject.getChild(i)
        if (child.name === 'AudioChild') {
          const a = child.getComponent('Component.AudioComponent')
          a?.play(1)
        }
      }
    })
  }
}
```

## `require` pattern for built-in Lens Studio modules

```typescript
// Always use require for LensStudio: prefixed modules
const WorldQueryModule = require('LensStudio:WorldQueryModule')
const asrModule = require('LensStudio:AsrModule')
const ttsModule = require('LensStudio:TtsModule')
const bleModule = require('LensStudio:BleModule')
const locationModule = require('LensStudio:LocationModule')
```

## Generic component with type parameters (TS feature)

From `AccessTSfromTS_TSComponentA.ts`:
```typescript
// Generic method — TS supports it even in Lens Studio
processData<T>(data: T): { processed: T; timestamp: number } {
  return { processed: data, timestamp: Date.now() }
}

// Caller
const result = componentA.processData<string>('hello')
print(result.processed) // 'hello'
```

