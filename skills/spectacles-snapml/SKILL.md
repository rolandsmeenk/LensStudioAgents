---
name: spectacles-snapml
description: Reference guide for on-device machine learning in Spectacles lenses using SnapML — covering MLComponent inspector configuration, supported model formats (TFLite/ONNX), input tensor shape setup, running inference per frame, parsing SSD/YOLO bounding box output tensors, INT8 quantization for performance, low-pass filter smoothing to reduce jitter, ObjectTracking3D for persistent tracking, and integrating detections with physics colliders. Use this skill whenever a lens needs to run a custom ML model on Spectacles without a cloud call, or when debugging input-shape mismatches — including SnapML Starter, SnapML Chess Hints, SnapML Pool. Use spectacles-ai instead if the model runs in the cloud via RSG.
---

# Spectacles SnapML — Reference Guide

**SnapML** lets you run custom machine learning models directly on the Spectacles hardware, with no cloud round-trip needed. Models run on the device's NPU or GPU, making inference fast enough for real-time AR (typically 10–30 fps depending on model size).

---

## Supported Model Formats

| Format | Notes |
|---|---|
| TensorFlow Lite (`.tflite`) | Primary format; recommended |
| ONNX (`.onnx`) | Supported via Lens Studio's ONNX importer |

Export your model as TFLite or ONNX and drag it into the Lens Studio Asset panel. Lens Studio will show it as an **ML Model** asset.

---

## Core API: `MLController`

The `MLController` component manages a model's lifecycle (load → input → run → output).

### Setup in Lens Studio
1. Add an **ML Component** to a Scene Object (Add Component → ML → ML Controller).
2. Assign your **ML Model** asset to the component.
3. Configure **inputs** (camera texture, custom texture, or float arrays) and **outputs** (bounding boxes, class scores, etc.) in the inspector.

### Running inference in script
```typescript
const mlComponent = this.sceneObject.getComponent('MLComponent');

const updateEvent = this.createEvent('UpdateEvent');
updateEvent.bind(() => {
  mlComponent.runImmediate(false);  // false = use camera texture input automatically
  // Outputs are now populated
  processOutputs();
});

function processOutputs() {
  // Access output tensors
  const outputData = mlComponent.getOutput('output_0').data as Float32Array;
  // Parse bounding boxes, class IDs, confidence scores...
}
```

---

## Object Detection Pattern

Object detection models output lists of bounding boxes + class IDs + confidence scores. The SnapML Starter sample demonstrates the canonical pattern:

### Parsing SSD-style output
```typescript
const NUM_DETECTIONS = 20;
const BOX_STRIDE = 4; // [ymin, xmin, ymax, xmax] per box

function parseDetections(rawBoxes: Float32Array, rawScores: Float32Array, threshold: number) {
  const detections: Detection[] = [];
  for (let i = 0; i < NUM_DETECTIONS; i++) {
    const score = rawScores[i];
    if (score < threshold) continue;
    const base = i * BOX_STRIDE;
    detections.push({
      ymin: rawBoxes[base],
      xmin: rawBoxes[base + 1],
      ymax: rawBoxes[base + 2],
      xmax: rawBoxes[base + 3],
      score,
      classId: i   // or from a separate classes tensor
    });
  }
  return detections;
}
```

### Projecting bounding boxes to screen space
Lens Studio gives you a camera with known FOV. Use the camera's unproject helper:
```typescript
// Convert normalised image coords [0,1] to world space
const camera = scene.findByName('Camera').getComponent('Camera');

function boxCenterToWorldPos(normX: number, normY: number, distance: number): vec3 {
  const screenPos = new vec2(normX * screen.getWidth(), normY * screen.getHeight());
  return camera.screenToWorld(screenPos, distance);
}
```

---

## Object Tracking

After detecting an object, track it across frames using the **AR Tracking** module, which uses the ML model's initial detection as a seed.

```typescript
const objectTracker = require('LensStudio:ObjectTracking3D');

// Start tracking once a detection is confirmed
const trackerSession = objectTracker.createSession({
  inputTexture: cameraTexture,
  boundingBox: initialDetectionBox,  // from ML output
});

trackerSession.onUpdate.add((trackedObject) => {
  // trackedObject.pose gives you the 3D transform to anchor AR content to
  myArObject.getTransform().setWorldTransform(trackedObject.pose);
});

trackerSession.start();
```

Stop tracking when the object leaves frame:
```typescript
trackerSession.onLost.add(() => {
  myArObject.enabled = false;
});
```

---

## Chess Hints Pattern (SnapML Chess Hints)

Combine object detection + networking + game logic:

1. **Detect** chess pieces on the board using a custom TFLite model.
2. **Classify** each piece (type + colour) from the detected bounding boxes.
3. **Map** piece positions to board coordinates (A1–H8 grid).
4. **Send** board state to a chess engine API via RSG or Fetch to get the best move.
5. **Overlay** the suggested move as an AR arrow in world space.
6. (Optional) **Share** the suggestion with another Spectacles user via Connected Lenses / Sync Kit.

---

## Pool Table Detection (SnapML Pool)

Complex multi-object detection where object count matters:
- A single model detects the pool table, 16 balls, and 6 pocket holes simultaneously.
- Use separate output head channels for table vs. balls vs. pockets.
- Compute suggested shot lines from detected ball/pocket positions using 2D geometry (lines, angle of incidence).
- Render shot-path lines as world-space LineRenderer meshes.

Detection stability tip: apply a low-pass filter to bounding box positions to reduce jitter:
```typescript
const SMOOTH = 0.3; // lerp factor
smoothedBox.x = lerp(smoothedBox.x, rawBox.x, SMOOTH);
smoothedBox.y = lerp(smoothedBox.y, rawBox.y, SMOOTH);
```

---

## Integrating with Physics

ML detections can drive physics interactions:
- Set a static physics body's position/size to match a detected bounding box.
- Other physics objects then collide with the "detected" real-world geometry.
- Useful for: detecting a real table and placing virtual objects that "rest" on it.

```typescript
const body = tableColliderObject.getComponent('Physics.BodyComponent');
const detectedBox = latestDetection.worldBoundingBox;
tableColliderObject.getTransform().setWorldPosition(detectedBox.center);
tableColliderObject.setScale(detectedBox.size);
```

---

## Performance Tips

| Tip | Reason |
|---|---|
| Run inference every 2–3 frames, not every frame | Most detection tasks don't need 60 Hz inference |
| Use quantised (INT8) models | Smaller, faster, comparable accuracy for most tasks |
| Resize input texture to match model input resolution | Avoid unnecessary upsampling inside the model |
| Use object tracking between inference frames | Tracker is cheaper than full detection |
| Profile in Lens Studio's Performance panel | Shows NPU vs. GPU time per frame |

---

## Common Gotchas

- **Input tensor shape must match exactly** — check the model's expected input shape (e.g., `[1, 320, 320, 3]`) and set the ML Component input resolution accordingly.
- **Output tensor interpretation varies by model architecture** (SSD, YOLO, EfficientDet, etc.) — read the model's original paper or training code to understand the output format.
- **On-device models cannot be updated without a lens update** — if you need dynamic model updates, use the RSG + cloud inference approach (`spectacles-ai` skill).
- **The SnapML AR Tracking component requires an initial detection** as a seed; it won't track without one.
- **Permissions**: Camera access must be enabled in Project Settings for ML to work on camera frames.
