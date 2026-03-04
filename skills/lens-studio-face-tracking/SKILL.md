---
name: lens-studio-face-tracking
description: Reference guide for face and body AR tracking in Lens Studio — covering FaceTrackingComponent setup (multi-face, faceIndex), FaceInset and FaceMask effects, 2D and 3D Face Attachments (hat/mouth/left_eye/right_eye anchors), Face Mesh UV texturing, Face Landmarks (68 keypoints), Face Expression weights for mouth-open/eye-blink detection, Eye Tracking component (left/right eye direction), Upper Body Tracking 3D asset (hips/spine/shoulder attachment points), Upper Body Mesh for seamless selfie occlusion, and Face Retouch/Eye Color/Face Liquify/Face Stretch effects. Use this skill for any phone or front-camera lens involving faces, selfie effects, makeup, face masks, 3D head ornaments, body tracking, or expression-driven animations — covering the vast majority of Snapchat lens content.
---

# Lens Studio Face Tracking — Reference Guide

Face tracking is the foundation of most Snapchat phone lenses. Lens Studio tracks faces in real time using the front camera, providing mesh, landmarks, attachment points, and expression weights.

---

## FaceTrackingComponent

### Setting up face tracking

Add a **Head** asset from the Add menu (Scene Hierarchy panel → + → Face → Head). This creates a face tracking hierarchy automatically.

To access the face tracking component in script:

```typescript
// Get the FaceTracking component from the head object
const faceTracking = headSceneObject.getComponent('FaceTracking')
```

### Multi-face tracking

Lens Studio can track up to 3 faces simultaneously. Use `faceIndex` to distinguish them:

```typescript
@input faceIndex: number = 0  // 0 = first face, 1 = second, 2 = third

const faceTracking = headSceneObject.getComponent('FaceTracking')
faceTracking.faceIndex = this.faceIndex
```

Each tracked face has its own head scene object. Duplicate the head hierarchy for each additional face.

---

## 2D & 3D Face Attachments

Objects parented to specific named landmarks follow the face in 3D.

### Common attachment anchor names

| Anchor | Position |
|---|---|
| `hat` | Top of head |
| `forehead` | Centre forehead |
| `left_eye` | Left eye centre |
| `right_eye` | Right eye centre |
| `nose_tip` | Tip of nose |
| `mouth` | Centre of mouth |
| `chin` | Bottom of chin |
| `left_ear` | Left ear |
| `right_ear` | Right ear |

### Adding a 3D attachment

1. Add your 3D object to the scene.
2. Parent it to the corresponding anchor object in the face tracking hierarchy (e.g., drag it under `hat` in the Scene Hierarchy panel).
3. Use **Adjust** mode on the face mesh to position the object correctly.

### Scripting 2D attachment position
```typescript
// Get the screen-space position of a face landmark
const faceTracking = headObject.getComponent('FaceTracking')

const updateEvent = this.createEvent('UpdateEvent')
updateEvent.bind(() => {
  // Screen-space position of the nose tip (normalised UV)
  const nosePos: vec2 = faceTracking.getLandmark(FaceTracking.LandmarkIndex.NoseTip)
  print(`Nose at: ${nosePos.x}, ${nosePos.y}`)
})
```

---

## Face Mesh

The Face Mesh conforms a textured mesh to the detected face geometry.

### Applying a texture to the face mesh

1. Add a **Face Mesh** to the scene (+ → Face → Face Mesh).
2. Create or import a texture that maps to the UV layout of the face mesh.
3. Assign the texture to the Face Mesh's material.

### Scripting face mesh properties
```typescript
const meshVisual = faceMeshObject.getComponent('Component.BaseMeshVisual')
const mat = meshVisual.material.clone()
meshVisual.material = mat

// Tint the face mask
mat.mainPass.baseColor = new vec4(0.2, 0.8, 0.4, 0.6)  // RGBA
mat.mainPass.blendMode = BlendMode.Normal
```

---

## Face Landmarks (68 Keypoints)

Lens Studio provides 68 facial landmark points that track with the face.

```typescript
const faceTracking = headObject.getComponent('FaceTracking')

// Get all 68 landmark positions (in local face space)
const landmarks: vec3[] = faceTracking.getFacePositions()

// Common landmark indices (0-indexed)
// 0–16: Jaw line
// 17–21: Left eyebrow
// 22–26: Right eyebrow
// 27–35: Nose bridge and base
// 36–41: Left eye
// 42–47: Right eye
// 48–67: Mouth

const leftEyeCenter: vec3 = landmarks[37]   // mid-left-eye
const rightEyeCenter: vec3 = landmarks[43]  // mid-right-eye
const mouthCenter: vec3 = landmarks[51]     // upper lip centre
```

---

## Face Expressions (Expression Weights)

Expression weights give you normalised [0,1] values for various facial actions.

```typescript
const faceTracking = headObject.getComponent('FaceTracking')

const updateEvent = this.createEvent('UpdateEvent')
updateEvent.bind(() => {
  const expressions = faceTracking.getFaceExpressionWeights()

  // Common expression keys:
  const mouthOpen: number   = expressions[FaceTracking.Expressions.MouthOpen]
  const leftBlink: number   = expressions[FaceTracking.Expressions.EyeBlinkLeft]
  const rightBlink: number  = expressions[FaceTracking.Expressions.EyeBlinkRight]
  const smileLeft: number   = expressions[FaceTracking.Expressions.SmileLeft]
  const smileRight: number  = expressions[FaceTracking.Expressions.SmileRight]
  const browRaiseLeft: number = expressions[FaceTracking.Expressions.BrowRaiseLeft]

  if (mouthOpen > 0.5) {
    print('Mouth open!')
    triggerEffect()
  }

  if (leftBlink > 0.8 && rightBlink < 0.2) {
    print('Left wink detected')
  }
})
```

---

## Eye Tracking

```typescript
// Add an Eye Tracking component to your scene object
const eyeTracking = this.sceneObject.getComponent('Component.EyeTrackingComponent')

const updateEvent = this.createEvent('UpdateEvent')
updateEvent.bind(() => {
  // Eye gaze direction in world space
  const leftGazeDir: vec3  = eyeTracking.leftEye.gaze
  const rightGazeDir: vec3 = eyeTracking.rightEye.gaze

  // Eye openness (0 = closed, 1 = fully open)
  const leftOpenness: number  = eyeTracking.leftEye.openness
  const rightOpenness: number = eyeTracking.rightEye.openness
})
```

---

## Face Effects

These are inspector-based effects added via the Add Component menu. Most require no scripting.

| Effect | What it does | Scripting access |
|---|---|---|
| **Face Retouch** | Skin smoothing, teeth whitening | Inspector sliders |
| **Eye Color** | Recolours the iris | `eyeColorComponent.color` |
| **Face Liquify** | Warp face geometry (e.g., bigger eyes, slimmer face) | Inspector sliders |
| **Face Stretch** | Distort facial proportions for comedy effects | Inspector sliders |
| **Face Inset** | Embeds a live camera view in the face area | Position/scale in inspector |
| **Face Mask** | Overlays a 2D texture mapped to face UV | Material on Face Mesh |
| **Face Texture** | Replaces the face with a static or animated image | Texture asset |

### Changing eye color at runtime
```typescript
const eyeColorComponent = this.sceneObject.getComponent('Component.EyeColorVisual')
eyeColorComponent.color = new vec4(0.0, 0.4, 1.0, 1.0)  // blue eyes
```

---

## Upper Body Tracking (Spectacles + Front Camera)

### Upper Body Tracking 3D

Provides a subset of humanoid attachment points: hips, spine bones, neck, head, shoulders, and upper arms.

1. Add **Object Tracking → Upper Body Tracking** from the Scene Hierarchy + menu.
2. The component tracks the skeleton and exposes attachment points via the `ObjectTracking3D` component.

```typescript
const bodyTracking = upperBodyObject.getComponent('Component.ObjectTracking3D')

bodyTracking.onObjectFound.add(() => print('Upper body detected'))
bodyTracking.onObjectLost.add(() => print('Upper body lost'))
```

### Upper Body Mesh

Creates a full, seamless 3D mesh of the upper body — ideal for selfie lenses. Can be used as:
- An **occluder** for body accessories (virtual clothes, jewellery)
- A **physics collider** surface
- A **texture target** for seamless face-to-neck blending

Add via Scene Hierarchy → + → Upper Body Mesh. Automatically includes:
- `Head Component` (with Face Mesh and skull)
- `Object Tracking 3D` (configured for Upper Body)
- `Upper Body Mesh` (the body surface mesh)

> **Note:** Upper Body Mesh does not support external custom meshes — use the built-in mesh and apply custom materials.

---

## Common Gotchas

- **`getFaceExpressionWeights()` returns a Dictionary** — check available keys using `Object.keys()` during development to verify available expression names on your target OS.
- **Multi-face**: each faceIndex needs its own separate Head scene object hierarchy — one FaceTracking component per face.
- **Eye tracking** is a separate component from face tracking — add `EyeTrackingComponent` in addition to FaceTracking.
- **Upper Body Mesh** is not available on all OS versions — check the Spectacles compatibility list.
- **Face Retouch** and morphing effects run on a GPU pass — avoid enabling more effects than needed for production performance.
- **Face Attach objects**: always test object placement by recording with a real face; the Lens Studio simulator's face is symmetrical and doesn't reflect real-world drift.
