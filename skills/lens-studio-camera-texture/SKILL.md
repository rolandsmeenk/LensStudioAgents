---
name: lens-studio-camera-texture
description: Learn how to programmatically request and manage raw camera textures and crops.
---
# Camera Textures
Lens Studio allows developers to intercept the background camera feed for customized rendering loops or image processing. On mobile phone lenses, this is straightforward, but on **Spectacles**, you must specifically address the correct camera IDs (e.g. `Right_Color`) since the device utilizes multi-camera input for rendering and tracking.

**Official docs:** [Spectacles Home](https://developers.snap.com/spectacles/home)

## Requesting the Background Feed
The `CameraModule` handles raw camera requests. 

> [!IMPORTANT]
> Because the Lens Studio Editor cannot simulate dual-camera passthrough, you should use `Default_Color` when running in the simulation, and `Right_Color` when compiling for the actual Spectacles device.

```typescript
@component
export class CameraFeedIntercept extends BaseScriptComponent {
    @input
    camModule: CameraModule;
    
    start() {
        const isEditor = global.deviceInfoSystem.isEditor();
        const camID = isEditor ? CameraModule.CameraId.Default_Color : CameraModule.CameraId.Right_Color;
        
        const camRequest = CameraModule.createCameraRequest();
        camRequest.cameraId = camID;
        // Optionally downsample the resolution to save performance metrics
        // camRequest.imageSmallerDimension = isEditor ? 352 : 756;
        
        // This Texture can now be applied to any material or UI element
        const rawCameraTexture = this.camModule.requestCamera(camRequest);
    }
}
```

## Applying to a Material
Once you have retrieved the `CameraTexture` from the `CameraModule`, you can inject it directly into a standard Lens Studio Material. Just set the `mainPass` parameter where a sampler2D is expected (typically `baseColor`).

```typescript
@input
targetMaterial: Material;

//...
this.targetMaterial.mainPass.baseColor = rawCameraTexture;
```

## Cropping and Manipulating
If you are passing camera visual data into a SnapML model or a UI sub-view, you frequently need to crop it. You can supply the camera feed to a `CropTextureProvider`.

```typescript
@input 
screenCropTexture: Texture;

//...
const camTexControl = rawCameraTexture.control as CameraTextureProvider;
const cropTexControl = this.screenCropTexture.control as CropTextureProvider;

// Inject the live camera into the Crop generator
cropTexControl.inputTexture = rawCameraTexture;

// The screenCropTexture is now a modified subsection of the camera feed!
```

## Reference Example
The script below is extracted from the official `CompositeCameraTexture` package to demonstrate a robust approach for toggling camera request properties based on the environment (Editor vs Device) for optimal development.

See [the reference guide](references/CameraService.md) for details.
