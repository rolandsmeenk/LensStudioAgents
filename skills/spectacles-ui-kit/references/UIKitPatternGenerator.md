# UIKitPatternGenerator Reference

```typescript
/**
 * Specs Inc. 2026
 * Runtime UI pattern generator that creates complete UIKit component hierarchies programmatically.
 * Generates common UI patterns like grids, menus, dialogs, and galleries without requiring prefabs.
 */
import { Frame } from "SpectaclesUIKit.lspkg/Scripts/Components/Frame/Frame";
import { GridLayout } from "SpectaclesUIKit.lspkg/Scripts/Components/GridLayout/GridLayout";
import { RectangleButton } from "SpectaclesUIKit.lspkg/Scripts/Components/Button/RectangleButton";
import { CapsuleButton } from "SpectaclesUIKit.lspkg/Scripts/Components/Button/CapsuleButton";
import { RoundButton } from "SpectaclesUIKit.lspkg/Scripts/Components/Button/RoundButton";

@component
export class UIKitPatternGenerator extends BaseScriptComponent {

    // ========================================
    // PATTERN SELECTION
    // ========================================

    @input
    @hint("Select the UI pattern to generate")
    @widget(new ComboBoxWidget([
        new ComboBoxItem("None", "none"),
        new ComboBoxItem("--- Custom UI Examples ---", "section1"),
        new ComboBoxItem("Simple Grid 2x2", "grid_2x2"),
        new ComboBoxItem("Simple Grid 3x3", "grid_3x3"),
        new ComboBoxItem("Simple Grid 4x4", "grid_4x4"),
        new ComboBoxItem("Card Grid with Icons", "card_grid"),
        new ComboBoxItem("Vertical Menu List", "menu_list"),
        new ComboBoxItem("--- Horizon OS Patterns ---", "section2"),
        new ComboBoxItem("Modal Dialog", "modal_dialog"),
        new ComboBoxItem("Content Section with Buttons", "content_section"),
        new ComboBoxItem("Image Content Section", "image_content_section"),
        new ComboBoxItem("Settings Menu", "settings_menu"),
        new ComboBoxItem("Action Sheet", "action_sheet"),
        new ComboBoxItem("Grid Gallery", "grid_gallery"),
    ]))
    patternType: string = "none";

    @input
    @hint("Regenerate pattern when changed at runtime")
    autoRegenerate: boolean = true;

    // ========================================
    // FRAME CONFIGURATION
    // ========================================

    @input
    @hint("Use Frame component as root container")
    useFrame: boolean = true;

    @input
    @hint("Frame appearance style")
    @widget(new ComboBoxWidget([
        new ComboBoxItem("Large (Far-field)", "Large"),
        new ComboBoxItem("Small (Near-field)", "Small"),
    ]))
    frameAppearance: string = "Large";

    @input
    @hint("Frame automatically shows when hovered and hides when not interacting")
    frameAutoShowHide: boolean = true;

    @input
    @hint("Frame follows user")
    frameFollowing: boolean = false;

    @input
    @hint("Frame size in cm (width, height, depth)")
    frameSize: vec3 = new vec3(24, 26, 1);

    @input
    @hint("Frame size offset for padding adjustment")
    frameSizeOffset: vec3 = new vec3(0, 0, 0);

    // ========================================
    // BUTTON CONFIGURATION
    // ========================================

    @input
    @hint("Default button type")
    @widget(new ComboBoxWidget([
        new ComboBoxItem("Rectangle Button", "RectangleButton"),
        new ComboBoxItem("Capsule Button", "CapsuleButton"),
        new ComboBoxItem("Round Button", "RoundButton"),
    ]))
    buttonType: string = "RectangleButton";

    @input
    @hint("Button style [NOTE: Cannot be changed at runtime, uses default PrimaryNeutral]")
    @widget(new ComboBoxWidget([
        new ComboBoxItem("Primary Neutral", "PrimaryNeutral"),
        new ComboBoxItem("Primary", "Primary"),
        new ComboBoxItem("Secondary", "Secondary"),
        new ComboBoxItem("Special", "Special"),
        new ComboBoxItem("Ghost", "Ghost"),
    ]))
    buttonStyle: string = "PrimaryNeutral";

    @input
    @hint("Button size (width, height, depth) in cm")
    buttonSize: vec3 = new vec3(4, 4, 1);

    @input
    @hint("Button corner radius [NOTE: Automatically calculated based on button size]")
    buttonCornerRadius: number = 0.5;

    @input
    @hint("Enable button shadows")
    buttonShadows: boolean = true;

    @input
    @hint("Play audio on button interactions [NOTE: Cannot be changed at runtime, uses default]")
    buttonAudio: boolean = true;

    // ========================================
    // LAYOUT CONFIGURATION
    // ========================================

    @input
    @hint("Grid cell size (width, height) in cm")
    gridCellSize: vec2 = new vec2(4, 4);

    @input
    @hint("Grid cell padding between cells")
    gridCellPadding: number = 0.5;

    @input
    @hint("Vertical layout spacing")
    verticalSpacing: number = 1.5;

    @input
    @hint("Horizontal layout spacing")
    horizontalSpacing: number = 1.5;

    @input
    @hint("Layout padding (left, top, right, bottom)")
    layoutPadding: vec4 = new vec4(0.5, 0.5, 0.5, 0.5);

    @input
    @hint("Horizontal alignment")
    @widget(new ComboBoxWidget([
        new ComboBoxItem("Left", "Left"),
        new ComboBoxItem("Center", "Center"),
        new ComboBoxItem("Right", "Right"),
    ]))
    horizontalAlignment: string = "Center";

    @input
    @hint("Vertical alignment")
    @widget(new ComboBoxWidget([
        new ComboBoxItem("Top", "Top"),
        new ComboBoxItem("Center", "Center"),
        new ComboBoxItem("Bottom", "Bottom"),
    ]))
    verticalAlignment: string = "Top";

    // ========================================
    // TEXT CONFIGURATION
    // ========================================

    @input
    @hint("Text size for button labels")
    textSize: number = 12;

    @input
    @hint("Text size for titles")
    titleTextSize: number = 16;

    @input
    @hint("Text size for sublabels")
    sublabelTextSize: number = 8;

    // ========================================
    // ADVANCED OPTIONS
    // ========================================

    @input
    @hint("Render order for all components")
    renderOrder: number = 0;

    @input
    @hint("Enable interaction on buttons")
    enableInteraction: boolean = true;

    @input
    @hint("Log button presses to console")
    logInteractions: boolean = true;

    // ========================================
    // INTERNAL STATE
    // ========================================

    private currentPattern: SceneObject = null;
    private lastPatternType: string = "none";
    private rootFrame: SceneObject = null;
    private delayedInitEvent: DelayedCallbackEvent = null;

    onAwake() {
        this.delayedInitEvent = this.createEvent("DelayedCallbackEvent");
        this.delayedInitEvent.bind(() => {
            this.generatePattern();
        });
        this.delayedInitEvent.reset(0.2);
    }

    onUpdate() {
        if (this.autoRegenerate && this.patternType !== this.lastPatternType) {
            this.lastPatternType = this.patternType;
            this.generatePattern();
        }
    }

    /**
     * Main pattern generation
     */
    generatePattern() {
        this.clearPattern();

        if (this.patternType.startsWith("section") || this.patternType === "none") {
            return;
        }

        print(`=== Generating Pattern: ${this.patternType} ===`);

        // Create root frame if enabled
        const container = this.useFrame ? this.createFrame() : this.sceneObject;

        // Generate pattern
        switch (this.patternType) {
            case "grid_2x2":
                this.currentPattern = this.createGridPattern(container, 2, 2);
                break;
            case "grid_3x3":
                this.currentPattern = this.createGridPattern(container, 3, 3);
                break;
            case "grid_4x4":
                this.currentPattern = this.createGridPattern(container, 4, 4);
                break;
            case "card_grid":
                this.currentPattern = this.createCardGrid(container);
                break;
            case "menu_list":
                this.currentPattern = this.createMenuList(container);
                break;
            case "modal_dialog":
                this.currentPattern = this.createModalDialog(container);
                break;
            case "content_section":
                this.currentPattern = this.createContentSection(container);
                break;
            case "image_content_section":
                this.currentPattern = this.createImageContentSection(container);
                break;
            case "settings_menu":
                this.currentPattern = this.createSettingsMenu(container);
                break;
            case "action_sheet":
                this.currentPattern = this.createActionSheet(container);
                break;
            case "grid_gallery":
                this.currentPattern = this.createGridGallery(container);
                break;
        }

        print(`✓ Pattern generated successfully`);
    }

    /**
     * Clear existing pattern
     */
    private clearPattern() {
        if (this.currentPattern) {
            this.currentPattern.destroy();
            this.currentPattern = null;
        }

        if (this.rootFrame && this.rootFrame !== this.sceneObject) {
            this.rootFrame.destroy();
            this.rootFrame = null;
        }

        // Clear children
        const children = this.sceneObject.getChildrenCount();
        for (let i = children - 1; i >= 0; i--) {
            this.sceneObject.getChild(i).destroy();
        }
    }

    /**
     * Create Frame component
     * Following UIKit docs pattern: create → initialize → set properties
     */
    private createFrame(): SceneObject {
        this.rootFrame = global.scene.createSceneObject("Frame");
        this.rootFrame.setParent(this.sceneObject);

        // Create Frame component using imported class
        const frameComp = this.rootFrame.createComponent(Frame.getTypeName()) as any;

        // IMPORTANT: Initialize Frame first (required by UIKit)
        frameComp.initialize();

        // Configure Frame with correct property names and methods
        const finalSize = new vec2(
            this.frameSize.x + this.frameSizeOffset.x,
            this.frameSize.y + this.frameSizeOffset.y
        );

        // Set properties AFTER initialization
        frameComp.innerSize = finalSize;
        frameComp.appearance = this.frameAppearance;
        frameComp.autoShowHide = this.frameAutoShowHide;

        // Use setFollowing method instead of property setter
        if (frameComp.setFollowing) {
            frameComp.setFollowing(this.frameFollowing);
        }

        print(`✓ Frame created: ${this.frameAppearance}, size: ${finalSize.x}×${finalSize.y}`);

        return this.rootFrame;
    }

    // ========================================
    // PATTERN GENERATORS
    // ========================================

    private createGridPattern(parent: SceneObject, rows: number, cols: number): SceneObject {
        const container = global.scene.createSceneObject(`Pattern_Grid_${rows}x${cols}`);
        container.setParent(parent);

        // Create GridLayout
        const gridLayout = this.createGridLayout(container, rows, cols);

        // Create buttons
        for (let i = 0; i < rows * cols; i++) {
            this.createButton(gridLayout, `${i + 1}`, "");
        }

        return container;
    }

    private createCardGrid(parent: SceneObject): SceneObject {
        const container = global.scene.createSceneObject("Pattern_CardGrid");
        container.setParent(parent);

        const gridLayout = this.createGridLayout(container, 3, 3);

        const labels = ["A", "B", "C", "D", "E", "F", "G", "H", "I"];
        for (let i = 0; i < 9; i++) {
            this.createButton(gridLayout, labels[i], `Label ${i + 1}`);
        }

        return container;
    }

    private createMenuList(parent: SceneObject): SceneObject {
        const container = global.scene.createSceneObject("Pattern_MenuList");
        container.setParent(parent);

        const layout = this.createVerticalLayout(container);

        const items = ["Option 1", "Option 2", "Option 3", "Option 4", "Option 5"];
        for (const item of items) {
            this.createButton(layout, item, "", "CapsuleButton", new vec3(14, 2.5, 1));
        }

        return container;
    }

    private createModalDialog(parent: SceneObject): SceneObject {
        const container = global.scene.createSceneObject("Pattern_ModalDialog");
        container.setParent(parent);

        const mainLayout = this.createVerticalLayout(container);

        // Title bar
        const titleBar = this.createHorizontalLayout(mainLayout);
        this.createText(titleBar, "Modal Title", this.titleTextSize);
        this.createButton(titleBar, "X", "", "RoundButton", new vec3(2, 2, 1));

        // Actions
        this.createButton(mainLayout, "Action 1", "", "CapsuleButton", new vec3(14, 2.5, 1));
        this.createButton(mainLayout, "Action 2", "", "CapsuleButton", new vec3(14, 2.5, 1));
        this.createButton(mainLayout, "Cancel", "", "CapsuleButton", new vec3(14, 2.5, 1));

        return container;
    }

    private createContentSection(parent: SceneObject): SceneObject {
        const container = global.scene.createSceneObject("Pattern_ContentSection");
        container.setParent(parent);

        const mainLayout = this.createVerticalLayout(container);

        this.createText(mainLayout, "Section Title", this.titleTextSize);

        const btnRow = this.createHorizontalLayout(mainLayout);
        this.createButton(btnRow, "Btn 1", "");
        this.createButton(btnRow, "Btn 2", "");
        this.createButton(btnRow, "Btn 3", "");

        return container;
    }

    private createImageContentSection(parent: SceneObject): SceneObject {
        const container = global.scene.createSceneObject("Pattern_ImageContent");
        container.setParent(parent);

        const mainLayout = this.createVerticalLayout(container);

        // Title bar
        const titleBar = this.createHorizontalLayout(mainLayout);
        this.createText(titleBar, "Image Content", this.titleTextSize);
        this.createButton(titleBar, "X", "", "RoundButton", new vec3(2, 2, 1));

        this.createText(mainLayout, "Section Title", this.textSize);

        // Grid
        const gridLayout = this.createGridLayout(mainLayout, 2, 3);
        for (let i = 0; i < 6; i++) {
            this.createButton(gridLayout, `${i + 1}`, "Label");
        }

        return container;
    }

    private createSettingsMenu(parent: SceneObject): SceneObject {
        const container = global.scene.createSceneObject("Pattern_Settings");
        container.setParent(parent);

        const layout = this.createVerticalLayout(container);

        const settings = ["Account", "Notifications", "Privacy", "Display", "Audio", "About"];
        for (const setting of settings) {
            this.createButton(layout, setting, ">", "CapsuleButton", new vec3(15, 2.5, 1));
        }

        return container;
    }

    private createActionSheet(parent: SceneObject): SceneObject {
        const container = global.scene.createSceneObject("Pattern_ActionSheet");
        container.setParent(parent);

        const layout = this.createVerticalLayout(container);

        this.createText(layout, "Choose an option", this.titleTextSize);
        this.createButton(layout, "Option 1", "", "CapsuleButton", new vec3(14, 2.5, 1));
        this.createButton(layout, "Option 2", "", "CapsuleButton", new vec3(14, 2.5, 1));
        this.createButton(layout, "Option 3", "", "CapsuleButton", new vec3(14, 2.5, 1));
        this.createButton(layout, "Cancel", "", "CapsuleButton", new vec3(14, 2.5, 1));

        return container;
    }

    private createGridGallery(parent: SceneObject): SceneObject {
        const container = global.scene.createSceneObject("Pattern_Gallery");
        container.setParent(parent);

        const gridLayout = this.createGridLayout(container, 4, 4);

        for (let i = 0; i < 16; i++) {
            this.createButton(gridLayout, `${i + 1}`, "");
        }

        return container;
    }

    // ========================================
    // COMPONENT CREATION
    // ========================================

    private createGridLayout(parent: SceneObject, rows: number, cols: number): SceneObject {
        const gridObj = global.scene.createSceneObject("GridLayout");
        gridObj.setParent(parent);

        // Create GridLayout component using imported class
        const grid = gridObj.createComponent(GridLayout.getTypeName());

        // Configure GridLayout with correct property names
        grid.rows = rows;
        grid.columns = cols;
        grid.cellSize = new vec2(this.gridCellSize.x, this.gridCellSize.y);  // vec2, not vec3
        grid.cellPadding = new vec4(this.gridCellPadding, this.gridCellPadding, this.gridCellPadding, this.gridCellPadding);  // vec4: L, T, R, B
        grid.layoutBy = 0;  // LayoutDirection.Row = 0

        // Initialize the grid layout
        grid.initialize();

        print(`  ✓ GridLayout: ${rows}×${cols}, cell size: ${this.gridCellSize.x}×${this.gridCellSize.y}`);

        return gridObj;
    }

    private createVerticalLayout(parent: SceneObject): SceneObject {
        // Manual vertical layout - no DirectionalLayout component
        const layoutObj = global.scene.createSceneObject("VerticalLayout");
        layoutObj.setParent(parent);

        // Store layout info on object for manual positioning
        (layoutObj as any)._isVerticalLayout = true;
        (layoutObj as any)._spacing = this.verticalSpacing;
        (layoutObj as any)._childCount = 0;

        print(`  ✓ VerticalLayout: spacing ${this.verticalSpacing} (manual positioning)`);

        return layoutObj;
    }

    private createHorizontalLayout(parent: SceneObject): SceneObject {
        // Manual horizontal layout - no DirectionalLayout component
        const layoutObj = global.scene.createSceneObject("HorizontalLayout");
        layoutObj.setParent(parent);

        // Store layout info on object for manual positioning
        (layoutObj as any)._isHorizontalLayout = true;
        (layoutObj as any)._spacing = this.horizontalSpacing;
        (layoutObj as any)._childCount = 0;

        print(`  ✓ HorizontalLayout: spacing ${this.horizontalSpacing} (manual positioning)`);

        return layoutObj;
    }

    /**
     * Create button component
     * Following UIKit docs pattern: create → set size → initialize → attach events
     */
    private createButton(parent: SceneObject, label: string, sublabel: string,
                        customType?: string, customSize?: vec3): SceneObject {

        const type = customType || this.buttonType;
        const size = customSize || this.buttonSize;

        const btnObj = global.scene.createSceneObject(`Btn_${label.replace(/[^a-zA-Z0-9]/g, "_")}`);
        btnObj.setParent(parent);

        // Create button component using imported class
        let button: any;
        if (type === "RectangleButton") {
            button = btnObj.createComponent(RectangleButton.getTypeName());
        } else if (type === "CapsuleButton") {
            button = btnObj.createComponent(CapsuleButton.getTypeName());
        } else if (type === "RoundButton") {
            button = btnObj.createComponent(RoundButton.getTypeName());
        }

        // Manual positioning if parent is a manual layout
        const parentAny = parent as any;
        if (parentAny._isVerticalLayout) {
            const yPos = -parentAny._childCount * (size.y + parentAny._spacing);
            btnObj.getTransform().setLocalPosition(new vec3(0, yPos, 0));
            parentAny._childCount++;
        } else if (parentAny._isHorizontalLayout) {
            const xPos = parentAny._childCount * (size.x + parentAny._spacing);
            btnObj.getTransform().setLocalPosition(new vec3(xPos, 0, 0));
            parentAny._childCount++;
        }

        // IMPORTANT: Set size BEFORE initialize (required by UIKit)
        if (type === "RoundButton") {
            button.width = size.x;
        } else {
            button.size = size;
        }

        // IMPORTANT: Initialize button after setting size
        button.initialize();

        // Configure button properties that have setters (after initialization)
        button.renderOrder = this.renderOrder;
        button.hasShadow = this.buttonShadows;

        // Note: style, playAudio, and cornerRadius cannot be set at runtime
        // - style: read-only getter, set during initialization via _style
        // - playAudio: private field without public setter
        // - cornerRadius: automatically set by button's visual based on size

        // Add interaction events (after initialization)
        if (this.enableInteraction && button.onTriggerUp) {
            button.onTriggerUp.add(() => {
                if (this.logInteractions) {
                    print(`    → Button pressed: ${label}`);
                }
            });
        }

        // Add text labels
        if (label) {
            this.createText(btnObj, label, this.textSize);
        }

        if (sublabel) {
            const sublabelObj = this.createText(btnObj, sublabel, this.sublabelTextSize);
            sublabelObj.getTransform().setLocalPosition(new vec3(0, -0.8, 0.1));
        }

        return btnObj;
    }

    private createText(parent: SceneObject, content: string, size: number): SceneObject {
        const textObj = global.scene.createSceneObject("Text");
        textObj.setParent(parent);

        const text = textObj.createComponent("Component.Text");
        text.text = content;
        text.size = size;
        text.horizontalAlignment = HorizontalAlignment.Center;
        text.verticalAlignment = VerticalAlignment.Center;

        return textObj;
    }
}

```

