---
name: spectacles-snapml
description: Reference guide for on-device machine learning in Spectacles lenses using SnapML — covering MLComponent inspector configuration, supported model formats (TFLite/ONNX), input tensor shape setup, synchronous (runImmediate) and asynchronous (runScheduled) inference patterns, parsing SSD/YOLO bounding box output tensors, INT8 quantization for NPU performance (avoid FP32 layers), low-pass filter smoothing to reduce jitter, ObjectTracking3D for persistent tracking, and integrating detections with physics colliders. Use this skill whenever a lens needs to run a custom ML model on Spectacles without a cloud call, or when debugging input-shape mismatches — including SnapML Starter, SnapML Chess Hints, SnapML Pool. Use spectacles-ai instead if the model runs in the cloud via RSG.
---

# Spectacles SnapML — Reference Guide

**SnapML** lets you run custom machine learning models directly on the Spectacles hardware, with no cloud round-trip needed. Models run on the device's NPU (preferred) or GPU, making inference fast enough for real-time AR (typically 10–30 fps depending on model size).

> **Simulator note**: On the Lens Studio desktop simulator, MLComponent falls back to CPU. Always profile on-device for accurate performance numbers.

---

## Supported Model Formats

| Format | Notes |
|---|---|
| TensorFlow Lite (`.tflite`) | Primary format; recommended for NPU |
| ONNX (`.onnx`) | Supported via Lens Studio's ONNX importer |

Export your model as TFLite or ONNX and drag it into the Lens Studio Asset panel. Lens Studio will show it as an **ML Model** asset.

---

## Core API: `MLComponent`

The `MLComponent` manages a model's lifecycle (load → input → run → output).

### Setup in Lens Studio
1. Add an **ML Component** to a Scene Object (Add Component → ML → ML Controller).
2. Assign your **ML Model** asset to the component.
3. Configure **inputs** (camera texture, custom texture, or float arrays) and **outputs** in the inspector.

### Synchronous inference (per frame, blocking)
```typescript
const mlComponent = this.sceneObject.getComponent('MLComponent')

const updateEvent = this.createEvent('UpdateEvent')
updateEvent.bind(() => {
  mlComponent.runImmediate(false)  // false = use camera texture input automatically
  processOutputs()
})

function processOutputs(): void {
  const outputData = mlComponent.getOutput('output_0').data as Float32Array
  // Parse bounding boxes, class IDs, confidence scores...
}
```

### Asynchronous inference (non-blocking, better framerate)

Running ML synchronously every frame can drop framerate. Use `runScheduled` to run inference async:

```typescript
onAwake(): void {
  const mlComponent = this.sceneObject.getComponent('MLComponent')

  // Enable scheduled (async) mode
  mlComponent.runScheduled(true)

  // Outputs are updated automatically when inference completes
  mlComponent.onRunningFinished.add(() => {
    processOutputs(mlComponent)
  })
}
```

Running inference every 2–3 frames also helps when you don't need 60 Hz detection:
```typescript
let frameCounter = 0

updateEvent.bind(() => {
  frameCounter++
  if (frameCounter % 3 === 0) {
    mlComponent.runImmediate(false)
    processOutputs()
  }
})
```

---

## Object Detection Pattern

Object detection models output lists of bounding boxes + class IDs + confidence scores.

### Parsing SSD-style output
```typescript
const NUM_DETECTIONS = 20
const BOX_STRIDE = 4 // [ymin, xmin, ymax, xmax] per box

interface Detection {
  ymin: number; xmin: number; ymax: number; xmax: number
  score: number; classId: number
}

function parseDetections(
  rawBoxes: Float32Array,
  rawScores: Float32Array,
  threshold: number
): Detection[] {
  const detections: Detection[] = []
  for (let i = 0; i < NUM_DETECTIONS; i++) {
    const score = rawScores[i]
    if (score < threshold) continue
    const base = i * BOX_STRIDE
    detections.push({
      ymin: rawBoxes[base],
      xmin: rawBoxes[base + 1],
      ymax: rawBoxes[base + 2],
      xmax: rawBoxes[base + 3],
      score,
      classId: i
    })
  }
  return detections
}
```

### Projecting bounding boxes to screen space
```typescript
const camera = scene.findByName('Camera').getComponent('Camera') as Camera

function boxCenterToWorldPos(normX: number, normY: number, distance: number): vec3 {
  const screenPos = new vec2(normX * screen.getWidth(), normY * screen.getHeight())
  return camera.screenToWorld(screenPos, distance)
}
```

---

## Smoothing Detection Jitter (Low-pass Filter)

```typescript
const SMOOTH = 0.3 // lerp factor — lower = smoother, higher = more responsive

const smoothedBox = { x: 0, y: 0, w: 0, h: 0 }

function smoothDetection(rawBox: {x: number, y: number, w: number, h: number}): void {
  smoothedBox.x = smoothedBox.x + (rawBox.x - smoothedBox.x) * SMOOTH
  smoothedBox.y = smoothedBox.y + (rawBox.y - smoothedBox.y) * SMOOTH
  smoothedBox.w = smoothedBox.w + (rawBox.w - smoothedBox.w) * SMOOTH
  smoothedBox.h = smoothedBox.h + (rawBox.h - smoothedBox.h) * SMOOTH
}
```

---

## Object Tracking (between inference frames)

After detecting an object, track it across frames using ObjectTracking3D:

```typescript
const objectTracker = require('LensStudio:ObjectTracking3D')

const trackerSession = objectTracker.createSession({
  inputTexture: cameraTexture,
  boundingBox: initialDetectionBox,  // from ML output
})

trackerSession.onUpdate.add((trackedObject) => {
  myArObject.getTransform().setWorldTransform(trackedObject.pose)
})

trackerSession.onLost.add(() => {
  myArObject.enabled = false
})

trackerSession.start()
```

The tracker is cheaper than full detection — run ML every N frames and fill gaps with tracking.

---

## Integrating with Physics

ML detections can drive physics interactions:

```typescript
const body = tableColliderObject.getComponent('Physics.BodyComponent')
const detectedBox = latestDetection.worldBoundingBox
tableColliderObject.getTransform().setWorldPosition(detectedBox.center)
tableColliderObject.getTransform().setWorldScale(detectedBox.size)
```

---

## NPU Performance Tips

| Tip | Reason |
|---|---|
| Use INT8-quantised models | Smaller, faster; NPU is optimised for INT8 |
| Avoid FP32 layers in the model | FP32 ops may fall back from NPU to GPU |
| Match input texture resolution exactly | Avoid upsampling inside the model |
| Use `runScheduled(true)` for async inference | Keeps the AR framerate smooth |
| Run inference every 2–3 frames | Most detection tasks don't need 60 Hz |
| Profile in Lens Studio's Performance panel | Shows NPU vs. GPU time per frame |

---

## Common Gotchas

- **Input tensor shape must match exactly** — check the model's expected input shape (e.g., `[1, 320, 320, 3]`) and set the ML Component input resolution accordingly.
- **Output tensor interpretation varies** by architecture (SSD, YOLO, EfficientDet) — read the model paper or training code.
- **On-device models cannot be updated without a lens update** — use RSG + cloud inference (`spectacles-ai`) if you need dynamic model updates.
- **Desktop simulator uses CPU** — always test performance on-device (Spectacles) for realistic NPU numbers.
- **Camera access permission** must be enabled in Project Settings for ML to work on camera frames.
- **ObjectTracking3D requires an initial detection** as a seed — it won't track without one.
