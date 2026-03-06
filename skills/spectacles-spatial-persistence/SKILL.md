---
name: spectacles-spatial-persistence
description: Learn how to persist and restore AR content across sessions using World Anchors.
---
# Spatial Persistence
Spectacles support persisting digital content (anchors) across sessions using the **Spatial Anchors API**. This enables users to place an object in their physical room, close the lens or reboot the device, and have the object reappear in the exact same physical location when they return.

**Official docs:** [Spectacles Home](https://developers.snap.com/spectacles/home) · [Spatial Anchors](https://developers.snap.com/spectacles/about-spectacles-features/apis/spatial-anchors)

> [!WARNING]
> The DeviceTracking component's mode **MUST** be set to `World` for Spatial Anchors to function.

## Creating and Persisting a World Anchor
Persisting content is a two-step process: first you create a local World Anchor using `AnchorSession`, then you explicitly save it to the device using `AnchorSession.saveAnchor()`.

```typescript
import { AnchorSession, AnchorSessionOptions } from "Spatial Anchors.lspkg/AnchorSession";
import { AnchorModule } from "Spatial Anchors.lspkg/AnchorModule";
import { State } from "Spatial Anchors.lspkg/Anchor";

// 1. Get the AnchorModule
const anchorModule = global.scene.getRootObject(0).getComponent(AnchorModule.getTypeName());

// 2. Configure Options
const options = new AnchorSessionOptions();
options.area = "MyRoomConfig"; // Group anchors by an area ID
options.scanForWorldAnchors = true;

// 3. Open Session
const session = await anchorModule.openSession(options);

// 4. Create the World Anchor in front of the camera
const spawnTransform = mat4.compose(myWorldPos, myWorldRot, vec3.one());
session.createWorldAnchor(spawnTransform).then(async (anchor) => {
    // 5. Save the anchor persistently 
    const savedAnchor = await session.saveAnchor(anchor);
    
    // Check if the physical anchor was immediately recognized by tracking
    if (savedAnchor.state === State.Found) {
       print(`Anchor saved and found! ID: ${savedAnchor.id}`);
    }
});
```

## Restoring Persisted Anchors on App Start
When users reload the Lens, you need to use the same `area` string to scan for previously saved anchors.

```typescript
// After opening the session with scanForWorldAnchors = true
session.onAnchorNearby.add((anchor) => {
    print(`Found an anchor matching ID: ${anchor.id}`);
    
    // Wait until the tracking system locks onto the physical location
    anchor.onFound.add(() => {
        print(`Anchor is physically localized at: ${anchor.toWorldFromAnchor.column3}`);
        
        // Spawn/move your preserved 3D content to this matrix transform
        myPrefab.getTransform().setWorldTransform(anchor.toWorldFromAnchor);
    });
});
```

## Deleting Anchors
To clear persistence state, you can reset the entire space:
```typescript
await session.reset();
```

## Reference Example
The logic snippet below is extracted directly from the official Spatial Persistence sample, demonstrating an `AnchorManager` singleton that handles session generation and tracking events cleanly.

See [the reference guide](references/AnchorManager.md) for details.
