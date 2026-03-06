# ExtendedMarkerTracking Reference

```typescript
/**
 * Extended marker tracking controller with world tracking support. Manages marker tracking with
 * device tracking validation and proper camera hierarchy setup for AR experiences.
 */

@component
export class ExtendMarkerTrackingController extends BaseScriptComponent {
  
  @input markerTrackingComponent: MarkerTrackingComponent;
  @input parentObject: SceneObject;

  onAwake() {
    // 1. Retrieve the native Camera 
    const mainCamera: SceneObject = global.scene.getRootObject(0); 

    if (mainCamera) {
      const deviceTrackingComponent = mainCamera.getComponent("Component.DeviceTracking");
      
      // 2. Check if the device tracking mode is currently World space tracking
      if (
        deviceTrackingComponent &&
        deviceTrackingComponent.getActualDeviceTrackingMode() == DeviceTrackingMode.World
      ) {
        
        // 3. Initialize the Marker tracking callback 
        this.getSceneObject().setParent(mainCamera);
        this.createEvent("OnStartEvent").bind(() => {
          if (this.markerTrackingComponent.enabled) {
            this.markerTrackingComponent.onMarkerFound = () => {
              this.onMarkerFoundCallback();
            };
          }
        });
      }
    }
  }

  onMarkerFoundCallback() {
    print("Marker Found");

    // Create a new parent object outside of the Camera object
    const newParent = global.scene.createSceneObject("NewParent");
    
    // 4. Unparent the tracking structure to lock its world space transformation in the room
    const delayedEvent = this.createEvent("DelayedCallbackEvent");
    delayedEvent.bind(() => {
      // NOTE: Using setParentPreserveWorldTransform ensures the object doesn't fly away
      this.parentObject.setParentPreserveWorldTransform(newParent);
      this.markerTrackingComponent.enabled = false;
    });

    // 5. Fire logic 0.5s after discovery to avoid tracking jitters upon first sight
    delayedEvent.reset(0.5); 
  }
}
```
