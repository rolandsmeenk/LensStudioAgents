# SnapML — Model Format and Inference Reference

## Supported model formats

| Format | File ext | Notes |
|---|---|---|
| TensorFlow Lite | `.tflite` | Primary format. Use `tflite_convert` or TFJS export. |
| ONNX | `.onnx` | Import via Lens Studio → Asset panel drag-and-drop → ONNX Converter. |

> Only **inference** (forward pass) is supported — no on-device training.

## MLComponent inspector configuration

After dragging an `.tflite` / `.onnx` model into the Asset panel:

1. Add **ML Component** to a Scene Object (Add Component → ML → ML Controller).
2. Set **Model** field to your imported asset.
3. **Inputs** tab: set the source (camera texture, custom texture, or float array) and the expected **resolution** (must match model input exactly, e.g. `320×320`).
4. **Outputs** tab: name each output tensor (the names from the model's output layer).

## Checking input tensor shape

Use `MlComponentUtils` or check the MLComponent inspector for exact shape:

```typescript
// Log tensor shapes at startup for debugging
const comp = this.sceneObject.getComponent('MLComponent')
comp.getInputs().forEach((input, i) => {
  print(`Input[${i}]: ${input.name} shape=${JSON.stringify(input.shape)}`)
})
comp.getOutputs().forEach((output, i) => {
  print(`Output[${i}]: ${output.name} shape=${JSON.stringify(output.shape)}`)
})
```

Common error: `"input tensor shape mismatch"` — ensure the ML Component input resolution equals the model's expected `[H, W]`.

## INT8 quantization for performance

When exporting a model, prefer **INT8 post-training quantization** in TFLite Converter:

```python
# Export script (run offline, not in Lens Studio)
converter = tf.lite.TFLiteConverter.from_saved_model(saved_model_dir)
converter.optimizations = [tf.lite.Optimize.DEFAULT]       # enables INT8 quant
converter.representative_dataset = representative_data_gen  # calibration set
converter.target_spec.supported_ops = [tf.lite.OpsSet.TFLITE_BUILTINS_INT8]
tflite_model = converter.convert()
```

Benefits:
- 4× smaller model file
- 2-4× faster on Spectacles NPU
- Minimal accuracy loss for detection/classification tasks

## Inference throttling (every N frames)

```typescript
private frameCount = 0
private readonly INFERENCE_EVERY_N = 3   // run ML every 3rd frame

onUpdate(): void {
  if (++this.frameCount % this.INFERENCE_EVERY_N !== 0) return
  this.mlComponent.runImmediate(false)    // false = use pre-wired camera input
  this.processOutputs()
}
```

## Object detection output parsing (SSD style)

```typescript
// Standard SSD output layout: [num_boxes, 4] boxes + [num_boxes] scores + [num_boxes] classes
function parseSSD(mlComp: any, threshold = 0.5) {
  const boxes   = mlComp.getOutput('output_0').data as Float32Array   // [N×4] ymin/xmin/ymax/xmax
  const scores  = mlComp.getOutput('output_1').data as Float32Array   // [N]
  const classes = mlComp.getOutput('output_2').data as Float32Array   // [N]

  const detections = []
  for (let i = 0; i < scores.length; i++) {
    if (scores[i] < threshold) continue
    detections.push({
      ymin: boxes[i * 4],     xmin: boxes[i * 4 + 1],
      ymax: boxes[i * 4 + 2], xmax: boxes[i * 4 + 3],
      score: scores[i],
      classId: Math.round(classes[i])
    })
  }
  return detections
}
```

## Low-pass filter for stable bounding boxes

```typescript
const ALPHA = 0.3  // higher = more responsive, lower = smoother
let smoothX = 0, smoothY = 0, smoothW = 0, smoothH = 0

function smoothBox(raw: { x: number; y: number; w: number; h: number }) {
  smoothX = smoothX * (1 - ALPHA) + raw.x * ALPHA
  smoothY = smoothY * (1 - ALPHA) + raw.y * ALPHA
  smoothW = smoothW * (1 - ALPHA) + raw.w * ALPHA
  smoothH = smoothH * (1 - ALPHA) + raw.h * ALPHA
  return { x: smoothX, y: smoothY, w: smoothW, h: smoothH }
}
```
