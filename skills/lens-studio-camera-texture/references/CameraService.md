# CameraService Reference

```typescript
/**
 * Camera service for managing camera textures. Handles camera requests and texture
 * composition for both editor and device environments with different camera configurations.
 */
@component
export class CameraService extends BaseScriptComponent {
    @input editorCamera: Camera;
    @input screenCropTexture: Texture;
    
    @input
    private camModule: CameraModule;

    private isEditor = global.deviceInfoSystem.isEditor();

    onAwake() {
        this.createEvent("OnStartEvent").bind(this.start.bind(this));
    }

    start() {
        // Toggle the target CameraID based on if we are simulating on Desktop
        const camID = this.isEditor
            ? CameraModule.CameraId.Default_Color
            : CameraModule.CameraId.Right_Color;
            
        const camRequest = CameraModule.createCameraRequest();
        camRequest.cameraId = camID;

        // Fetch the active Texture
        const camTexture = this.camModule.requestCamera(camRequest);
        
        // Apply to a Crop Provider
        const camTexControl = camTexture.control as CameraTextureProvider;
        const cropTexControl = this.screenCropTexture.control as CropTextureProvider;
        cropTexControl.inputTexture = camTexture;
    }
}
```
