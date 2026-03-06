---
name: spectacles-ui-kit
description: Reference guide for Spectacles UI Kit (FastUI/UIKit) — covering programmatic UI generation without prefabs. Includes Frame configuration (Large/Small appearance, autoShowHide, setFollowing), GridLayout (rows, columns, cellSize vec2, cellPadding vec4), and programmatic Button creation (RectangleButton, CapsuleButton, RoundButton). Shows the correct creation lifecycle (createComponent → set size → initialize() → set properties → bind events). Use this skill when building complex 2D spatial layouts, grids, menus, dialogs, or galleries entirely in TypeScript instead of assembling them manually in the Lens Studio inspector.
---

# Spectacles UI Kit — Programmatic UI Guide

While Lens Studio provides UI prefabs, you can construct complex interfaces entirely in TypeScript using the **Spectacles UI Kit**. This approach (sometimes called FastUI) is ideal for dynamic grids, scalable menus, and data-driven galleries.

> **Import Note:** Components belong to `SpectaclesUIKit.lspkg`.

---

## Component Lifecycle (CRITICAL)

When creating UI Kit components via script, you **must** follow this exact lifecycle sequence or the layout will fail to render:

1. `global.scene.createSceneObject()`
2. `createComponent('TypeName')`
3. **Set physical size/dimensions**
4. **Call `.initialize()`**
5. **Set visual properties (appearance, margins)**
6. **Bind interaction events**

---

## 1. Creating a Frame (The Root Container)

The `Frame` component holds panels, adds a background plate, and handles focus behaviors.

```typescript
import { Frame } from "SpectaclesUIKit.lspkg/Scripts/Components/Frame/Frame"

const frameObj = global.scene.createSceneObject("MainFrame")
const frameComp = frameObj.createComponent(Frame.getTypeName()) as any

// 1. Initialize Frame first (required by UIKit)
frameComp.initialize()

// 2. Set properties AFTER initialization
frameComp.innerSize = new vec2(24, 26) // Width, Height (cm)

// Appearance: "Large" (Far-field) or "Small" (Near-field)
frameComp.appearance = "Large"

// Auto-show/hide based on user gaze interaction
frameComp.autoShowHide = true

// Use the setFollowing method (do not set a property directly)
if (frameComp.setFollowing) {
  frameComp.setFollowing(false)
}
```

---

## 2. Creating a Grid Layout

`GridLayout` automatically arranges child objects in rows and columns.

```typescript
import { GridLayout } from "SpectaclesUIKit.lspkg/Scripts/Components/GridLayout/GridLayout"

const gridObj = global.scene.createSceneObject("GridLayout")
gridObj.setParent(frameObj)

const grid = gridObj.createComponent(GridLayout.getTypeName())

// Configure Grid geometry
grid.rows = 3
grid.columns = 3

// cellSize is a vec2 (width, height)
grid.cellSize = new vec2(4, 4)

// cellPadding is a vec4 (Left, Top, Right, Bottom)
grid.cellPadding = new vec4(0.5, 0.5, 0.5, 0.5)

// layoutBy: 0 = Row-first, 1 = Column-first
grid.layoutBy = 0 

// Initialize (locks in the geometry)
grid.initialize()
```

---

## 3. Creating Buttons

You can create `RectangleButton`, `CapsuleButton`, or `RoundButton`.

```typescript
import { RectangleButton } from "SpectaclesUIKit.lspkg/Scripts/Components/Button/RectangleButton"

const btnObj = global.scene.createSceneObject("Button_1")
btnObj.setParent(gridObj) // Add to grid

const button = btnObj.createComponent(RectangleButton.getTypeName()) as any

// 1. Set size BEFORE initialize!
// Note: For RoundButton use button.width = 4;
button.size = new vec3(4, 4, 1)

// 2. Initialize
button.initialize()

// 3. Configure styling post-initialization
button.renderOrder = 0 // matches parent frame
button.hasShadow = true

// Note: Button style (Primary, Secondary, Ghost) is handled in
// constructor/inspector. Changing it at runtime is unsupported.

// 4. Bind interactions
button.onTriggerUp.add(() => {
  print("Button clicked!")
})

// Optional: Add a text label manually
const textObj = global.scene.createSceneObject("Label")
textObj.setParent(btnObj)
const textComp = textObj.createComponent("Component.Text")
textComp.text = "Click Me"
textComp.size = 12
textComp.horizontalAlignment = HorizontalAlignment.Center
textComp.verticalAlignment = VerticalAlignment.Center
```

---

## 4. Manual Layouts (Optional)

If `GridLayout` is too rigid, you can calculate positions manually for simple vertical or horizontal stacks:

```typescript
// Example: Vertical Layout loop
const spacing = 1.5
const btnHeight = 2.5

for (let i = 0; i < 5; i++) {
  const btnObj = createMyButton() // generic wrapper returning your button 
  const yPos = -i * (btnHeight + spacing)
  btnObj.getTransform().setLocalPosition(new vec3(0, yPos, 0))
  btnObj.setParent(menuContainer)
}
```

---

## Common Gotchas

- **Initialization Order:** Passing properties to a UI element *before* `.initialize()` rarely works. Set base dimensions usually before, then init, then appearance.
- **RoundButton Size:** Uses `.width`, not `.size`.
- **Corner Radius:** Automatically calculated by the component based on size constraints. Don't overwrite it directly.
- **`LayoutDirection` enum:** If you lack the TS import for `LayoutDirection.Row`, just pass the integer `0`.
- **Text Sizing:** Text components map `size` arbitrarily based on their transform hierarchy. 12-16 is a good starting point for button labels in a Large frame.

---

## Reference Examples
*   [UIKitPatternGenerator.ts](references/UIKitPatternGenerator.md) - An official comprehensive programmatic UI builder demonstrating all component types and lifecycle requirements.

