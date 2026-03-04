---
name: spectacles-lens-essentials
description: Reference guide for foundational Lens Studio patterns on Spectacles — covering the GestureModule (pinch down/up/strength, targeting, grab, phone-in-hand with correct TypeScript API), SIK components (PinchButton, DragInteractable, GrabInteractable, ScrollView), hand-tracking gestures, physics bodies/colliders/callbacks (including audio-on-collision), LSTween animation (position/scale/rotation/color tweens), prefab instantiation at runtime, materials (clone-before-modify), spatial anchors, on-device persistent storage (putString/getFloat), spatial images, and the Path Pioneer raycasting pattern. Use this skill for any Spectacles lens that needs interaction, motion, animation, physics, audio, or persistent local storage — including Essentials, Throw Lab, Spatial Persistence, Spatial Image Gallery, Path Pioneer, Public Speaker, Voice Playback, Material Library, and DJ Specs samples.
---

# Spectacles Lens Essentials — Reference Guide

A compact reference for the most commonly used systems when building Spectacles lenses in Lens Studio.

**Official docs:** [Spectacles Home](https://developers.snap.com/spectacles/home) · [Features Overview](https://developers.snap.com/spectacles/about-spectacles-features/overview) · [Spatial Design](https://developers.snap.com/spectacles/best-practices/design-for-spectacles/introduction-to-spatial-design)

---

## GestureModule (Spectacles Gesture API)

The docs describe the **Gesture Module** as an ML-based API for reliable gesture detection (pinch, targeting, grab). Use it for raw events when you need more control than SIK components.

The `GestureModule` is the Spectacles-native API for reliable ML-based gesture detection. Use it for raw pinch, targeting, and grab events when you need more control than SIK's higher-level components offer.

```typescript
@component
export class GestureExample extends BaseScriptComponent {
  private gestureModule: GestureModule = require('LensStudio:GestureModule')

  onAwake(): void {
    // --- Pinch ---
    this.gestureModule
      .getPinchDownEvent(GestureModule.HandType.Right)
      .add((args: PinchDownArgs) => {
        // args.confidence: 0–1, how confident the model is
        // args.palmOrientation: vec3, palm facing direction
        print('Right pinch down, confidence: ' + args.confidence)
      })

    this.gestureModule
      .getPinchStrengthEvent(GestureModule.HandType.Right)
      .add((args: PinchStrengthArgs) => {
        // args.strength: 0 = no pinch, 1 = full pinch
        print('Pinch strength: ' + args.strength)
      })

    this.gestureModule
      .getPinchUpEvent(GestureModule.HandType.Right)
      .add((args: PinchUpArgs) => {
        // args.palmOrientation: vec3
        print('Right pinch up')
      })

    // Use GestureModule.HandType.Left, .Right, or .Both
  }
}
```

### Targeting Gesture (index finger pointing)

```typescript
this.gestureModule
  .getTargetingStartEvent(GestureModule.HandType.Right)
  .add(() => print('Started pointing'))

this.gestureModule
  .getTargetingEndEvent(GestureModule.HandType.Right)
  .add(() => print('Stopped pointing'))
```

### Grab Gesture (fist)

```typescript
this.gestureModule
  .getGrabStartEvent(GestureModule.HandType.Both)
  .add(() => print('Grab started (either hand)'))

this.gestureModule
  .getGrabEndEvent(GestureModule.HandType.Both)
  .add(() => print('Grab released'))
```

### Phone-in-Hand Detection

```typescript
this.gestureModule
  .getPhoneInHandEvent(GestureModule.HandType.Right)
  .add(() => print('User is holding a phone in their right hand'))
```

> **GestureModule vs SIK**: Use `GestureModule` when you need raw events and confidence values. Use SIK's `PinchButton`, `DragInteractable`, etc. when you want high-level UI components with built-in visual feedback.

---

## Spectacles Interaction Kit (SIK)

SIK is Snap's prebuilt AR interaction library. Add it to a project via the Asset Library: search **"Spectacles Interaction Kit"**. All SIK imports use the `SpectaclesInteractionKit.lspkg` package path.

### Key SIK Components

| Component | Purpose |
|---|---|
| `HandInputData` | Access hand pose, finger positions, pinch state per frame |
| `PinchButton` | Trigger an action on pinch; works with either hand |
| `DragInteractable` | Make any scene object draggable by hand |
| `GrabInteractable` | Grab and move objects with a fist gesture |
| `ScrollView` | Scrollable UI list driven by hand swipe |
| `ToggleButton` | On/off button, syncs visual state |

### ToggleButton
```typescript
import { ToggleButton } from 'SpectaclesInteractionKit.lspkg/Components/UI/ToggleButton/ToggleButton'

const toggleButton = this.sceneObject.getComponent(ToggleButton.getTypeName()) as ToggleButton

// React to toggle events
toggleButton.onStateChanged.add((isOn: boolean) => {
  print('Toggle is now: ' + (isOn ? 'ON' : 'OFF'))
  lampObject.enabled = isOn
})

// Read current state
if (toggleButton.isToggledOn) {
  print('Button is currently ON')
}

// Force a state programmatically
toggleButton.toggle()
```

### ScrollView
```typescript
import { ScrollView } from 'SpectaclesInteractionKit.lspkg/Components/UI/ScrollView/ScrollView'

const scrollView = this.sceneObject.getComponent(ScrollView.getTypeName()) as ScrollView

// Listen for scroll position changes
scrollView.onScrollPositionChanged.add((normalizedPos: number) => {
  // normalizedPos: 0 = top, 1 = bottom
  print('Scroll position: ' + normalizedPos)
  updateVisibleItems(normalizedPos)
})
```

### Reading hand position in script
```typescript
import { HandInputData } from 'SpectaclesInteractionKit.lspkg/Providers/HandInputData/HandInputData'

const handData = HandInputData.getInstance()

const updateEvent = this.createEvent('UpdateEvent')
updateEvent.bind(() => {
  const rightHand = handData.getDominantHand()
  if (rightHand.isPinching()) {
    const pinchPos = rightHand.getPinchPosition()
    print('Pinch at: ' + JSON.stringify(pinchPos))
  }
})
```

---

## Physics

Lens Studio uses a Bullet-based physics engine. Components: **Body**, **Collider**, and **Constraint**.

### Setting up a physics object
1. Add a **Physics Body** component (static, kinematic, or dynamic).
2. Add a **Collider** (Box, Sphere, Capsule, or Mesh).
3. Dynamic objects respond to gravity and forces automatically.

### Applying forces in script
```typescript
const body = this.sceneObject.getComponent('Physics.BodyComponent')

// Apply an impulse at the object's center
body.applyImpulse(new vec3(0, 500, -200))

// Apply torque
body.applyTorqueImpulse(new vec3(0, 10, 0))

// Set velocity directly (useful for throwing)
body.velocity = velocity
body.angularVelocity = angularVel
```

### Throw mechanics (from Throw Lab)
```typescript
// Sample hand position over N frames, compute delta / dt
const velocity = (currentPos.sub(prevPos)).uniformScale(1 / getDeltaTime())
body.velocity = velocity.uniformScale(throwStrength)
```

### Physics callbacks
```typescript
body.onCollisionEnter.add((collision) => {
  const other = collision.otherObject
  print('Hit: ' + other.name)

  if (collision.contacts.length > 0) {
    const point = collision.contacts[0].position
    spawnParticles(point)
  }
})
```

---

## Audio

### Play audio
```typescript
const audioComponent = this.sceneObject.getComponent('Component.AudioComponent')
audioComponent.audioTrack = myAudioTrack   // assign in inspector or via script
audioComponent.play(1)                       // play once (pass 0 for loop)
audioComponent.stop()
```

### Record and play back voice (from Voice Playback sample)
```typescript
const voiceML = require('LensStudio:VoiceML')

let recordedBuffer: AudioBuffer | null = null

voiceML.startRecording((buffer: AudioBuffer) => {
  recordedBuffer = buffer
})

voiceML.stopRecording()
if (recordedBuffer) {
  audioComponent.playAudioBuffer(recordedBuffer)
}
```

### Audio mixer channels
```typescript
audioComponent.mixerChannel = 'Music'   // or 'SFX', 'Voice'
```

### Audio-reactive visuals with AudioSpectrum

`AudioSpectrum` gives you per-frame frequency band data from any `AudioComponent` — useful for visualisers, beat-reactive effects, or driving shader parameters.

```typescript
const audioSpectrum = this.sceneObject.getComponent('Component.AudioSpectrumComponent')

const updateEvent = this.createEvent('UpdateEvent')
updateEvent.bind(() => {
  // bands: Float32Array of frequency magnitudes (length depends on band count setting)
  const bands = audioSpectrum.getBands()
  const bass  = bands[0]   // low frequency (kick drum, bass)
  const mid   = bands[Math.floor(bands.length / 2)] // midrange
  const high  = bands[bands.length - 1] // high frequency (hi-hat, sibilance)

  // Drive a VFX property or shader uniform:
  vfxComponent.asset.properties['intensity'] = bass
  mat.mainPass.baseColor = new vec4(mid, 0.2, high, 1.0)

  // Drive an object's scale
  const s = 1.0 + bass * 2.0
  this.sceneObject.getTransform().setLocalScale(new vec3(s, s, s))
})
```

> Set up `AudioSpectrumComponent` in the Inspector: assign the `AudioComponent` source, set band count (32 or 64 are common), and choose linear or logarithmic scale.

---

## Animation with LSTween

LSTween (bundled in SIK) is a Lens Studio tween library for smooth property animation.

```typescript
import { LSTween } from 'SpectaclesInteractionKit.lspkg/Utils/LSTween/LSTween'

// Move an object to a target position over 0.5 seconds
LSTween.moveToWorld(sceneObject, targetPosition, 0.5)
  .easing(TWEEN.Easing.Quadratic.Out)
  .start()

// Scale up
LSTween.scaleTo(sceneObject, new vec3(1, 1, 1), 0.3).start()

// Fade a screen image
LSTween.colorTo(screenImage, new vec4(1, 1, 1, 0), 0.4).start() // fade out
```

Chain tweens with `.onComplete`:
```typescript
LSTween.moveTo(obj, posA, 0.5)
  .onComplete(() => LSTween.moveTo(obj, posB, 0.5).start())
  .start()
```

---

## Materials & Shaders

### Modifying material properties at runtime
```typescript
const meshVisual = this.sceneObject.getComponent('Component.RenderMeshVisual')
const mat = meshVisual.material.clone() // clone so you don't affect other objects using same material
meshVisual.material = mat

mat.mainPass.baseColor = new vec4(1, 0, 0, 1) // red
mat.mainPass.opacity = 0.5
```

---

## Spatial Images (2D → 3D)

```typescript
const spatialImageModule = require('LensStudio:SpatialImageModule')

spatialImageModule.createSpatialImageFromTexture(myTexture, (spatialImage) => {
  spatialImage.setParent(scene.getRootObject(0))
  spatialImage.getTransform().setWorldPosition(targetPosition)
})
```

---

## Spatial Anchors & Persistent Storage

```typescript
const spatialAnchorModule = require('LensStudio:SpatialAnchorModule')

// Create an anchor at a world position
spatialAnchorModule.createAnchor(worldPosition, (anchor) => {
  saveToStorage('my_anchor', anchor.id)
})

// Later: restore the anchor
const anchorId = loadFromStorage('my_anchor')
spatialAnchorModule.getAnchor(anchorId, (anchor) => {
  sceneObject.getTransform().setWorldPosition(anchor.worldPosition)
})
```

### Persistent Storage (on-device)
```typescript
const storage = global.persistentStorageSystem

storage.store.putString('username', 'Roland')
storage.store.putFloat('highScore', 42.5)

const name = storage.store.getString('username')
const score = storage.store.getFloat('highScore')
```

---

## Display & sizing (Spectacles)

From the [Spatial Design](https://developers.snap.com/spectacles/best-practices/design-for-spectacles/introduction-to-spatial-design) docs: the displays achieve full overlap at **1.1 m** from the user; at that distance the visible content area is ~**1000×1397 px** (~53×77 cm). The **focus plane is at 1 m** — place highly detailed content near this distance. The display is **portrait ~3:4**. Design for hands-free and natural interactions; the OS reserves space on the hand for a system button; the rest is available for your lens.

---

## Common Gotchas

- **GestureModule requires Spectacles** — it is not available for phone lenses or in the desktop simulator.
- **SIK components expect a specific scene hierarchy** — read the SIK setup guide in its README before restructuring the scene.
- **Physics and the World Mesh**: enable the World Mesh Collider in *Project Settings → World Understanding* so physics objects land on real surfaces.
- **Cloning materials**: always call `material.clone()` before modifying properties at runtime, otherwise all objects sharing that material change together.
- **`getDeltaTime()`** is your friend for frame-rate-independent motion.
- **Spatial anchors** require the user to rescan the area if they move far away; give the user a visual "anchor not found" state.
- **Audio latency**: use pre-loaded `AudioTrack` assets rather than loading from URL for low-latency sound effects.
