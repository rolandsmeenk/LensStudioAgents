---
name: spectacles-lens-essentials
description: Reference guide for foundational Lens Studio patterns on Spectacles — covering SIK components (PinchButton, DragInteractable, GrabInteractable, ScrollView), hand-tracking gestures, physics bodies/colliders/callbacks (including audio-on-collision), LSTween animation (position/scale/rotation/color tweens), prefab instantiation at runtime, materials (clone-before-modify), spatial anchors, on-device persistent storage (putString/getFloat), spatial images, and the Path Pioneer raycasting pattern. Use this skill for any Spectacles lens that needs interaction, motion, animation, physics, audio, or persistent local storage — including Essentials, Throw Lab, Spatial Persistence, Spatial Image Gallery, Path Pioneer, Public Speaker, Voice Playback, Material Library, and DJ Specs samples.
---

# Spectacles Lens Essentials — Reference Guide

A compact reference for the most commonly used systems when building Spectacles lenses in Lens Studio.

---

## Spectacles Interaction Kit (SIK)

SIK is Snap's prebuilt AR interaction library. It provides off-the-shelf hand tracking, pinch detection, UI controls, and interactable objects. Add it to a project via the Asset Library: search "Spectacles Interaction Kit".

### Key SIK Components

| Component | Purpose |
|---|---|
| `HandInputData` | Access hand pose, finger positions, pinch state per frame |
| `PinchButton` | Trigger an action on pinch; works with either hand |
| `DragInteractable` | Make any scene object draggable by hand |
| `GrabInteractable` | Grab and move objects with a fist gesture |
| `ScrollView` | Scrollable UI list driven by hand swipe |
| `ToggleButton` | On/off button, syncs visual state |

### Reading hand position in script
```typescript
import { HandInputData } from 'SpectaclesInteractionKit/Providers/HandInputData/HandInputData';

const handData = HandInputData.getInstance();

const updateEvent = this.createEvent('UpdateEvent');
updateEvent.bind(() => {
  const rightHand = handData.getDominantHand();
  if (rightHand.isPinching()) {
    const pinchPos = rightHand.getPinchPosition();
    print('Pinch at: ' + JSON.stringify(pinchPos));
  }
});
```

---

## Gesture Module

The Gesture Module is a lower-level API for custom gesture detection, separate from SIK.

```typescript
const gestureModule = require('LensStudio:GestureModule');

// Listen for a specific gesture
gestureModule.onGestureDetected.add((event) => {
  if (event.gestureType === GestureType.HandRaise) {
    print('Hand raised!');
  }
});
```

Common gesture types: `Pinch`, `Point`, `Ok`, `HandRaise`, `Palm`, `ThumbsUp`, `Victory`.

---

## Physics

Lens Studio uses a Bullet-based physics engine. Components: **Body**, **Collider**, and **Constraint**.

### Setting up a physics object
1. Add a **Physics Body** component (static, kinematic, or dynamic).
2. Add a **Collider** (Box, Sphere, Capsule, or Mesh).
3. Dynamic objects respond to gravity and forces automatically.

### Applying forces in script
```typescript
// Get the physics body component
const body = this.sceneObject.getComponent('Physics.BodyComponent');

// Apply an impulse at the object's center
body.applyImpulse(new vec3(0, 500, -200));

// Apply torque
body.applyTorqueImpulse(new vec3(0, 10, 0));

// Set velocity directly (useful for throwing)
body.velocity = velocity;
body.angularVelocity = angularVel;
```

### Throw mechanics (from Throw Lab)
Compute throw velocity from hand motion:
```typescript
// Sample hand position over N frames, compute delta / dt
const velocity = (currentPos.sub(prevPos)).uniformScale(1 / getDeltaTime());
body.velocity = velocity.uniformScale(throwStrength);
```

### Physics callbacks
```typescript
body.onCollisionEnter.add((collision) => {
  const other = collision.otherObject;
  print('Hit: ' + other.name);

  // Spawn particles at impact point
  if (collision.contacts.length > 0) {
    const point = collision.contacts[0].position;
    spawnParticles(point);
  }
});
```

---

## Raycasting

```typescript
const physics = require('LensStudio:Physics');

// Cast a ray from a world position along a direction
const hit = physics.raycast(origin, direction, /* maxDistance */ 100);
if (hit) {
  print('Hit: ' + hit.sceneObject.name + ' at ' + JSON.stringify(hit.position));
}

// Cast from the camera forward (gaze ray)
const camera = scene.findByName('Camera').getComponent('Camera');
const camTransform = camera.getSceneObject().getTransform();
const origin = camTransform.getWorldPosition();
const direction = camTransform.forward;
const hit = physics.raycast(origin, direction, 50);
```

---

## Audio

### Play audio
```typescript
const audioComponent = this.sceneObject.getComponent('Component.AudioComponent');
audioComponent.audioTrack = myAudioTrack;   // assign in inspector or via script
audioComponent.play(1);                       // play once (pass 0 for loop)
audioComponent.stop();
```

### Record and play back voice (from Voice Playback sample)
```typescript
const voiceML = require('LensStudio:VoiceML');

let recordedBuffer: AudioBuffer | null = null;

// Start recording
voiceML.startRecording((buffer) => {
  recordedBuffer = buffer;
});

// Stop and play back
voiceML.stopRecording();
if (recordedBuffer) {
  audioComponent.playAudioBuffer(recordedBuffer);
}
```

### Audio mixer channels
Use different channels to control relative volume:
```typescript
audioComponent.mixerChannel = 'Music';   // or 'SFX', 'Voice'
```

---

## Animation with LSTween

LSTween (bundled in SIK) is a Lens Studio tween library for smooth property animation.

```typescript
import { LSTween } from 'SpectaclesInteractionKit/Utils/LSTween/LSTween';

// Move an object to a target position over 0.5 seconds
LSTween.moveToWorld(sceneObject, targetPosition, 0.5)
  .easing(TWEEN.Easing.Quadratic.Out)
  .start();

// Scale up
LSTween.scaleTo(sceneObject, new vec3(1, 1, 1), 0.3).start();

// Fade a screen image
LSTween.colorTo(screenImage, new vec4(1, 1, 1, 0), 0.4).start(); // fade out
```

Chain tweens with `.onComplete`:
```typescript
LSTween.moveTo(obj, posA, 0.5)
  .onComplete(() => LSTween.moveTo(obj, posB, 0.5).start())
  .start();
```

---

## Materials & Shaders

### Modifying material properties at runtime
```typescript
const meshVisual = this.sceneObject.getComponent('Component.RenderMeshVisual');
const mat = meshVisual.material.clone(); // clone so you don't affect other objects using same material
meshVisual.material = mat;

// Set a float or colour pass property
mat.mainPass.baseColor = new vec4(1, 0, 0, 1); // red
mat.mainPass.opacity = 0.5;
```

### Graph Materials in Lens Studio
- Use the **Graph Material Editor** (Material Editor → New → Graph Material) for visual shader authoring.
- Use **Code Material Editor** (GLSL) for low-level custom shaders.
- The **Material Library** sample contains ready-to-use materials: Fresnel, holographic, wireframe, iridescent, etc. Import them from that project as starting points.

### Post Effects
Add post-processing via **Post Effect** assets (e.g., bloom, chromatic aberration, colour grade). Apply them to the Camera's **Post Effects** list in the inspector.

---

## Spatial Images (2D → 3D)

The **Spatial Image** API converts a flat image into a depth-aware 3D representation.

```typescript
const spatialImageModule = require('LensStudio:SpatialImageModule');

spatialImageModule.createSpatialImageFromTexture(myTexture, (spatialImage) => {
  // spatialImage is a scene object with a rendered 3D mesh
  spatialImage.setParent(scene.getRootObject());
  spatialImage.getTransform().setWorldPosition(targetPosition);
});
```

Tip: let the user browse their Snapchat memories via the Snap ML `UserImageProvider` and convert selected images to spatial images for an AR gallery (as in the Spatial Image Gallery sample).

---

## Spatial Anchors & Persistent Storage

Spatial anchors attach content to a fixed real-world location that persists across sessions.

```typescript
const spatialAnchorModule = require('LensStudio:SpatialAnchorModule');

// Create an anchor at a world position
spatialAnchorModule.createAnchor(worldPosition, (anchor) => {
  // anchor.id is a string you store in persistent storage for retrieval later
  saveToStorage('my_anchor', anchor.id);
});

// Later: restore the anchor
const anchorId = loadFromStorage('my_anchor');
spatialAnchorModule.getAnchor(anchorId, (anchor) => {
  sceneObject.getTransform().setWorldPosition(anchor.worldPosition);
});
```

### Persistent Storage (on-device)
```typescript
const storage = global.persistentStorageSystem;

// Store a value (string, number, bool)
storage.store.putString('username', 'Roland');
storage.store.putFloat('highScore', 42.5);

// Retrieve
const name = storage.store.getString('username');
const score = storage.store.getFloat('highScore');
```

---

## Path Creation & Raycasting (Path Pioneer pattern)

The **Path Pioneer** sample shows drawing a walkable path in the real world:
1. Raycast from the hand pinch point onto the World Mesh.
2. Store hit positions as waypoints.
3. Use a line renderer (or instantiated mesh stamps) to draw the path.
4. Animate a character or particle along the stored waypoints.

---

## Common Gotchas

- **SIK components expect a specific scene hierarchy** — read the SIK setup guide in its README before restructuring the scene.
- **Physics and the World Mesh**: enable the World Mesh Collider in *Project Settings → World Understanding* so physics objects land on real surfaces.
- **Cloning materials**: always call `material.clone()` before modifying properties at runtime, otherwise all objects sharing that material change together.
- **`getDeltaTime()`** is your friend for frame-rate-independent motion; don't hard-code per-frame deltas.
- **Spatial anchors** require the user to rescan the area if they move far away; give the user a visual "anchor not found" state.
- **Audio latency**: use pre-loaded `AudioTrack` assets rather than loading from URL for low-latency sound effects.
