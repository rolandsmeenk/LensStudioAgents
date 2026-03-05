---
name: lens-studio-2d-ui
description: Reference guide for 2D UI and screen-space interaction in Lens Studio — covering ScreenTransform anchors/offsets/size/pivot and coordinate conversions (localPointToScreenPoint, screenPointToLocalPoint, localPointToWorldPoint), ScreenImage (texture, stretch mode, color tint), Text component (content, font, alignment, color, size), ScreenRegionComponent for defining tap/touch areas, TouchComponent with TouchStartEvent/TouchMoveEvent/TouchEndEvent, tap input for phone lenses, LSTween UI animations (colorTo, moveTo on screen elements), multi-swatch color picker pattern, undo stack pattern, and common gotchas. Use this skill whenever a lens needs a 2D UI panel, tap interaction, on-screen buttons, text labels, color pickers, swipeable menus, or undo/redo — covering Drawing, Quiz, TappableQuestion, MemeSticker, MusicVideo, and HighScore examples.
---

# Lens Studio 2D UI — Reference Guide

Lens Studio's 2D UI layer uses **ScreenTransform** components to position elements in screen space. All screen-space elements are children of a **Full Frame Region** or **Safe Render Region** at the top of the scene hierarchy.

---

## ScreenTransform — Coordinate System

`ScreenTransform` uses a normalised coordinate system:
- **Origin `(0, 0)`** = centre of the parent region
- **`(1, 1)`** = top-right corner
- **`(-1, -1)`** = bottom-left corner

```typescript
const st = this.sceneObject.getComponent('Component.ScreenTransform')
```

### Anchors

Anchors pin an element to a region of its parent. Both `anchors.setMin` and `anchors.setMax` use a **[0, 1]** space (not the ScreenTransform [-1, 1] space):

```typescript
// Fill the entire parent
st.anchors.setMin(new vec2(0, 0))
st.anchors.setMax(new vec2(1, 1))

// Pin to the top-right corner only
st.anchors.setMin(new vec2(1, 1))
st.anchors.setMax(new vec2(1, 1))

// Top half of parent
st.anchors.setMin(new vec2(0, 0.5))
st.anchors.setMax(new vec2(1, 1))
```

### Offsets

Offsets add inset (in canvas units) from each anchor edge:

```typescript
st.offsets.setLeft(10)    // 10 units inset from left anchor
st.offsets.setRight(-10)  // 10 units inset from right anchor
st.offsets.setBottom(20)
st.offsets.setTop(-20)
```

### Position and size (point anchor)

When both anchors are the same point, the element is positioned by `position` and sized by `size`:

```typescript
st.anchors.setCenter(new vec2(0.5, 0.5))  // pin to parent centre
st.position = new vec2(0, 0.3)            // offset 30% up
st.size = new vec2(300, 60)               // 300×60 canvas units
```

---

## Coordinate Conversions

```typescript
// Screen-space (pixels) ← ScreenTransform local
const screenPx: vec2 = st.localPointToScreenPoint(new vec2(0, 0))

// ScreenTransform local ← screen-space pixels
const local: vec2 = st.screenPointToLocalPoint(touchPosScreenPx)

// World space (3D) ← ScreenTransform local
const worldPt: vec3 = st.localPointToWorldPoint(new vec2(0, 0))

// ScreenTransform local ← World space
const localPt: vec2 = st.worldPointToLocalPoint(hit.position)
```

---

## ScreenImage

`ScreenImage` renders a texture on a 2D quad in screen space.

```typescript
const screenImage = this.sceneObject.getComponent('Component.Image')

// Assign a texture
screenImage.mainPass.baseTex = myTexture

// Tint color (RGBA)
screenImage.mainPass.baseColor = new vec4(1, 0.5, 0, 1)

// Reset to white (no tint)
screenImage.mainPass.baseColor = new vec4(1, 1, 1, 1)

// Show/hide
screenImage.enabled = false
```

### Stretch modes (set in the Inspector)
| Mode | Behaviour |
|---|---|
| **Stretch** | Fills the ScreenTransform bounds, ignoring aspect ratio |
| **Fit** | Letterboxes to preserve aspect ratio |
| **Fill** | Crops to fill the bounds while preserving aspect ratio |
| **Pixel Perfect** | 1:1 pixel mapping (for UI icons) |

---

## Text Component

```typescript
const textComponent = this.sceneObject.getComponent('Component.Text')

// Set text content
textComponent.text = 'Score: ' + score

// Change font size
textComponent.size = 48

// Alignment
textComponent.horizontalAlignment = HorizontalAlignment.Center
textComponent.verticalAlignment   = VerticalAlignment.Center

// Color
textComponent.textFill.color = new vec4(1, 1, 1, 1)  // white

// Bold / italic via the text property (HTML-style tags)
textComponent.text = '<b>Bold</b> and <i>italic</i>'
```

### Setting text from untrusted sources (network data, user content)

Never assign network or user-provided strings directly — unclosed HTML tags crash the renderer. Strip tags first:

```typescript
// Strips HTML-style tags and caps length before displaying untrusted content
function safeSetText(component: Text, value: string, maxLength = 200): void {
  const stripped = (value ?? '')
    .slice(0, maxLength)        // cap length
    .replace(/<[^>]*>/g, '')   // strip all tag-like sequences
  component.text = stripped
}

// Usage — safe even if serverName contains malicious tags:
safeSetText(textComponent, serverName)
```

---

## Touch Input

### TapEvent (phone lenses, simple tap)
```typescript
const tapEvent = this.createEvent('TapEvent')
tapEvent.bind((eventData) => {
  const screenPos: vec2 = eventData.getPosition()  // screen pixels
  print('Tapped at: ' + screenPos.x + ', ' + screenPos.y)
  handleTap(screenPos)
})
```

### TouchComponent (multi-touch, drag, phone lenses)
```typescript
const touchComponent = this.sceneObject.getComponent('Component.TouchComponent')

// Touch start
touchComponent.addMTouchStartCallback((eventData) => {
  const pos = eventData.getPosition()
  print('Touch start at: ' + pos.x + ', ' + pos.y)
})

// Touch move (drag)
touchComponent.addMTouchMoveCallback((eventData) => {
  const pos = eventData.getPosition()
  drawAtPosition(pos)
})

// Touch end
touchComponent.addMTouchEndCallback((eventData) => {
  print('Touch ended')
  finalizeStroke()
})
```

### ScreenRegionComponent (define touch-active area)

`ScreenRegionComponent` on a scene object marks it as a specific region type. Use it to prevent touches from passing through your UI:

```typescript
// In Inspector: add ScreenRegionComponent, set region type to TouchBlocking
// This stops the world from receiving taps that hit the UI panel
```

---

## UI Buttons for Phone Lenses

For phone lenses (not Spectacles), use tap regions rather than SIK PinchButton:

```typescript
// Pattern: invisible ScreenImage + TouchComponent as a button

@component
export class TapButton extends BaseScriptComponent {
  @input label: string = 'Button'
  onTapped: (() => void) | null = null

  onAwake(): void {
    const touch = this.sceneObject.getComponent('Component.TouchComponent')
    touch.addMTouchStartCallback(() => {
      if (this.onTapped) this.onTapped()
    })
  }
}
```

---

## LSTween for UI Animations

Use `LSTween` (part of the Spectacles Interaction Kit) for smooth UI transitions:

```typescript
import { LSTween } from 'SpectaclesInteractionKit.lspkg/Utils/LSTween/LSTween'

// Fade in a screen image
LSTween.colorTo(screenImage, new vec4(1, 1, 1, 1), 0.3).start()  // fade to opaque

// Fade out
LSTween.colorTo(screenImage, new vec4(1, 1, 1, 0), 0.3).start()  // fade to transparent

// Slide in from the right
const startPos = new vec2(1.5, 0)    // off screen right
const endPos   = new vec2(0, 0)      // on screen centre
screenTransform.position = startPos
LSTween.moveToScreen(screenTransform.sceneObject, endPos, 0.4)
  .easing(TWEEN.Easing.Quadratic.Out)
  .start()

// Scale pop (bounce-in a button)
LSTween.scaleTo(this.sceneObject, new vec3(1, 1, 1), 0.2)
  .easing(TWEEN.Easing.Back.Out)
  .start()
```

Chain animations with `.onComplete`:
```typescript
LSTween.colorTo(panel, new vec4(1, 1, 1, 0), 0.3)  // fade out
  .onComplete(() => panel.enabled = false)           // then hide
  .start()
```

---

## Color Picker Pattern

From the **Drawing** example — multiple color swatches, with a visual selection indicator:

```typescript
@component
export class ColorPicker extends BaseScriptComponent {
  @input swatches: SceneObject[]   // one per color
  @input colors: vec4[] = []       // matching color values
  @input selectionRing: SceneObject

  private selectedIndex: number = 0

  onAwake(): void {
    this.swatches.forEach((swatch, i) => {
      const touch = swatch.getComponent('Component.TouchComponent')
      touch.addMTouchStartCallback(() => this.selectColor(i))
    })
  }

  selectColor(index: number): void {
    this.selectedIndex = index
    // Move the selection ring to the chosen swatch
    const swatchST = this.swatches[index].getComponent('Component.ScreenTransform')
    const selST    = this.selectionRing.getComponent('Component.ScreenTransform')
    selST.position = swatchST.position    // match position

    // Apply color to the drawing material
    drawingMaterial.mainPass.penColor = this.colors[index]
  }
}
```

---

## Undo Stack Pattern

From the **Drawing** example:

```typescript
class UndoStack {
  private readonly maxSize = 20
  private stack: (() => void)[] = []  // each item is a "undo function"

  push(undoFn: () => void): void {
    if (this.stack.length >= this.maxSize) {
      this.stack.shift()  // drop oldest
    }
    this.stack.push(undoFn)
  }

  undo(): void {
    const fn = this.stack.pop()
    if (fn) fn()
  }

  get canUndo(): boolean {
    return this.stack.length > 0
  }
}
```

---

## Common Gotchas

- **Anchors use [0, 1] space**, not the [-1, 1] ScreenTransform local space — mixing them up is the most common ScreenTransform bug.
- **`position` only works when both anchor points are the same** — if `anchors.min ≠ anchors.max` (stretch mode), position is ignored; use offsets instead.
- **Touch events don't automatically block** — a touch on a UI panel passes through to the world unless a `ScreenRegionComponent` with `TouchBlocking` is present.
- **`ScreenRegionComponent` region types**: `TouchBlocking` is the most common for UI; `SafeRender` and `FullFrame` define the playfield boundaries.
- **`touchComponent.addMTouchStartCallback` vs `TapEvent`**: `TapEvent` fires only on completed, non-moved taps; `TouchComponent` callbacks fire immediately on contact — use `TouchComponent` for drawing and drag interactions.
- **Pivot point** affects how a ScreenTransform rotates and scales — a pivot of `(0, 0)` rotates around the centre, `(-1, -1)` around the bottom-left corner.
- **Text HTML tags**: Lens Studio supports `<b>`, `<i>`, `<color=#rrggbbaa>`, `<size=N>` tags in `textComponent.text`. Non-closing tags crash the text renderer.
- **Never set `textComponent.text` to untrusted string content directly** (e.g., data from the network or from Dynamic Response). Untrusted strings containing unclosed HTML tags will crash the renderer; strip or escape them first.
