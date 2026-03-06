---
name: lens-studio-marker-tracking
description: Learn how to track image markers and detach the tracked content into World Space in AR.
---
# Image Marker Tracking
Standard lens studio image tracking binds 3D content directly to a printed image marker, so when the marker is hidden, the object disappears. By combining the `MarkerTrackingComponent` with `DeviceTrackingMode.World`, you can execute the **"World-Locked Marker" pattern**. 

The World-Locked pattern detects the marker once to calculate its physical coordinate in the room, and then deliberately *unparents* your 3D content so it stays permanently pinned to that location in World-Space, regardless of whether the camera is still looking at the physical marker.

**Official docs:** [Spectacles Home](https://developers.snap.com/spectacles/home)

## Basic Marker Component Setup
First, make sure your Main Camera has a `DeviceTracking` component set to `World` mode. Then construct a standard `MarkerTrackingComponent`.

```typescript
// 1. Grab camera and assert tracking mode
const mainCamera = global.scene.getRootObject(0).getComponent("Component.Camera");
const deviceTracking = mainCamera.getSceneObject().getComponent("Component.DeviceTracking");

if (deviceTracking.getActualDeviceTrackingMode() === DeviceTrackingMode.World) {
    // World space is active! We can use world-locked markers.
}
```

## The World-Locked Pattern
This involves initially parenting your AR content to the marker track, listening for the `onMarkerFound` event, and then instantly swapping parents to root-level while using `setParentPreserveWorldTransform` to prevent jumping.

```typescript
@input 
markerTrackingComponent: MarkerTrackingComponent;

@input 
my3DPrefabParent: SceneObject;

onAwake() {
    this.createEvent("OnStartEvent").bind(() => {
        if (this.markerTrackingComponent) {
            this.markerTrackingComponent.onMarkerFound = () => {
                this.handleInitialDetection();
            };
        }
    });
}

handleInitialDetection() {
   // 1. Create a stable root parent at the scene origin
   const worldRoot = global.scene.createSceneObject("WorldRoot");
   
   // 2. Wait exactly half a second for the marker tracking to settle and avoid jittering
   const delay = this.createEvent("DelayedCallbackEvent");
   delay.bind(() => {
       // 3. Re-parent WITHOUT changing visual position
       this.my3DPrefabParent.setParentPreserveWorldTransform(worldRoot);
       
       // 4. Disable the tracker to save performance now that we are localized
       this.markerTrackingComponent.enabled = false;
   });
   delay.reset(0.5);
}
```

## Reference Example
The code logic below is extracted from the official `MarkerTrackingHelper.lspkg` package, demonstrating exactly how Spectacles AR handles hybrid Image/World persistence tracking logic out of the box.

See [the reference guide](references/ExtendedMarkerTracking.md) for details.
