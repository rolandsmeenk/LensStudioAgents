# MocopiMainController Reference

```typescript
/**
 * Main orchestrator for mocopi motion capture integration.
 * Coordinates WebSocket client and avatar controller components.
 */

import { SkeletonDefinition, FrameData } from "./MocopiDataTypes"

@component
export class MocopiMainController extends BaseScriptComponent {
  @input webSocketClient: MocopiWebSocketClient
  @input avatarController: MocopiAvatarController
  @input showAvatar: boolean = true

  onAwake() {
    // Setup event handlers
    this.webSocketClient.onSkeletonReceived = (skeleton: SkeletonDefinition) => {
      this.handleSkeletonReceived(skeleton)
    }

    this.webSocketClient.onFrameReceived = (frame: FrameData) => {
      this.handleFrameReceived(frame)
    }
  }

  private handleSkeletonReceived(skeleton: SkeletonDefinition) {
    if (this.avatarController && this.showAvatar) {
      this.avatarController.initializeAvatar(skeleton)
    }
  }

  private handleFrameReceived(frame: FrameData) {
    if (this.avatarController && this.showAvatar) {
      this.avatarController.updateFrame(frame)
    }
  }

  public connect() {
    if (this.webSocketClient) {
      this.webSocketClient.connect()
    }
  }
}
```
