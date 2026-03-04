# SIK Gestures + Buttons Reference

Sourced from `Essentials/Assets/HookUpGesturesEvents/HookUpGesturesEventsTS.ts`
and `Essentials/Assets/HookUpButtonsEvents/HookUpButtonsEventsTS.ts`.

## Hand pinch gesture (SIK HandInputData)

```typescript
import { SIK } from 'SpectaclesInteractionKit.lspkg/SIK'
import { mix } from 'SpectaclesInteractionKit.lspkg/Utils/animate'
import NativeLogger from 'SpectaclesInteractionKit.lspkg/Utils/NativeLogger'

const log = new NativeLogger('GestureExample')

@component
export class GestureExample extends BaseScriptComponent {
  @input destinationRef: SceneObject
  @input lerpSpeed: number = 0.1

  private leftHand = SIK.HandInputData.getHand('left')   // or 'right'
  private trackedObj: SceneObject = null
  private isMoving = false

  onAwake(): void {
    this.createEvent('OnStartEvent').bind(() => this.onStart())
    this.createEvent('UpdateEvent').bind(() => this.onUpdate())
  }

  private onStart(): void {
    // Subscribe to hand pinch events
    this.leftHand.onPinchDown.add(() => {
      this.isMoving = !this.isMoving
      log.d('Pinch: isMoving = ' + this.isMoving)
    })

    // Other available gesture events:
    // this.leftHand.onPinchUp.add(...)
    // this.leftHand.onIndexDown.add(...)
    // this.leftHand.onIndexUp.add(...)
  }

  private onUpdate(): void {
    if (!this.isMoving || !this.trackedObj) return
    const cur = this.trackedObj.getTransform().getWorldPosition()
    const dst = this.destinationRef.getTransform().getWorldPosition()
    const pos = mix(cur, dst, this.lerpSpeed)
    this.trackedObj.getTransform().setWorldPosition(pos)
  }
}
```

## Interactable button (SIK Interactable)

```typescript
import { Interactable } from 'SpectaclesInteractionKit.lspkg/Components/Interaction/Interactable/Interactable'
import { InteractorEvent } from 'SpectaclesInteractionKit.lspkg/Core/Interactor/InteractorEvent'

@component
export class ButtonExample extends BaseScriptComponent {
  @input @allowUndefined createButton: Interactable
  @input @allowUndefined destroyButton: Interactable
  @input prefabToInstantiate: ObjectPrefab

  private activeInstance: SceneObject = null

  onAwake(): void {
    this.createEvent('OnStartEvent').bind(() => {
      this.createButton?.onInteractorTriggerStart((e: InteractorEvent) => {
        if (!this.activeInstance) this.spawnPrefab()
      })
      this.destroyButton?.onInteractorTriggerStart((e: InteractorEvent) => {
        if (this.activeInstance) {
          this.activeInstance.destroy()
          this.activeInstance = null
        }
      })
    })
  }

  private spawnPrefab(): void {
    this.activeInstance = this.prefabToInstantiate.instantiate(null)
    this.activeInstance.name = 'SpawnedObject'
    // schedule self-destruct
    const ev = this.createEvent('DelayedCallbackEvent')
    ev.bind(() => {
      this.activeInstance?.destroy()
      this.activeInstance = null
    })
    ev.reset(5)
  }
}
```

## Available hand events (PinchDetector / GestureProvider)

```typescript
leftHand.onPinchDown           // finger pinch started
leftHand.onPinchUp             // finger pinch released
leftHand.onIndexDown           // index tap started
leftHand.onIndexUp             // index tap released
leftHand.onPalmTap             // palm face-up tap
```
