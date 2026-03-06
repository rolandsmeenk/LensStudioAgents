---
name: spectacles-mocopi-integration
description: Connect Sony Mocopi motion capture hardware to Spectacles using WebSockets for real-time skeletal tracking.
---
# Mocopi Integration
Sony **mocopi** is a mobile motion capture system that can stream skeletal data over a local network. Using the `MocopiReceiver` package, you can drive AR avatars and interactive characters in Spectacles lenses using real-time full-body motion capture.

**Official site:** [Sony mocopi](https://www.sony.net/Products/mocopi/)

## Connection Workflow
1. **Host Server:** Run the mocopi desktop/mobile app and enable "Send" mode (UDP/WebSocket).
2. **WebSocket Client:** Use `MocopiWebSocketClient` in Lens Studio to connect to the IP/Port of the host.
3. **Skeleton Mapping:** Listen for the `SkeletonDefinition`, which contains the hierarchical bone structure.
4. **Frame Processing:** Update your 3D avatar bones every time a `FrameData` packet arrives.

## Basic Implementation
The `MocopiMainController` manages the communication between the network client and the avatar.

```typescript
import { MocopiWebSocketClient } from "./MocopiWebSocketClient";
import { MocopiAvatarController } from "./MocopiAvatarController";

@component
export class MyMocopiBridge extends BaseScriptComponent {
  @input webSocketClient: MocopiWebSocketClient;
  @input avatarController: MocopiAvatarController;

  onAwake() {
    this.webSocketClient.onSkeletonReceived = (skeleton) => {
      // Initialize the avatar mesh with the correct bone count and hierarchy
      this.avatarController.initializeAvatar(skeleton);
    };

    this.webSocketClient.onFrameReceived = (frame) => {
      // Apply real-time rotations/positions to the avatar bones
      this.avatarController.updateFrame(frame);
    };
  }

  public connect() {
    this.webSocketClient.connect();
  }
}
```

## Avatar Configuration
Your 3D model must be rigged with a hierarchy that matches the mocopi bone structure (hips, spine, neck, head, and limbs).

> [!TIP]
> Use the `ResetButton` pattern in your lens to allow users to recalibrate the avatar's world position if it drifts from the tracking center.

## Reference Example
The logic below is extracted from the official `MocopiReceiver.lspkg` package, demonstrating the main orchestration logic for real-time motion capture.

See [the reference guide](references/MocopiMainController.md) for details.
