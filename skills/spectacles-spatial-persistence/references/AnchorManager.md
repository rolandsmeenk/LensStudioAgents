# AnchorManager Reference

```typescript
import {Anchor, State, UserAnchor} from "Spatial Anchors.lspkg/Anchor"
import {AnchorSession, AnchorSessionOptions} from "Spatial Anchors.lspkg/AnchorSession"
import Event, {PublicApi, unsubscribe} from "SpectaclesInteractionKit.lspkg/Utils/Event"
import {AnchorModule} from "Spatial Anchors.lspkg/AnchorModule"

/**
 * Singleton class that handles all invocations of the AnchorModule logic to retrieve spatially persistent anchors.
 */
export class AnchorManager {
  private anchorModule: AnchorModule
  private anchorSession?: AnchorSession
  
  private onAreaAnchorFoundEvent: Event<Anchor> = new Event<Anchor>()
  readonly onAreaAnchorFound: PublicApi<Anchor> = this.onAreaAnchorFoundEvent.publicApi()
  
  private currentAnchor: Anchor
  private anchorFoundUnsubscribes: unsubscribe[] = []

  private async createAnchorSession(areaId: string): Promise<AnchorSession> {
    const options = new AnchorSessionOptions()
    options.area = areaId
    options.scanForWorldAnchors = true
    try {
      const session = await this.anchorModule.openSession(options)
      return session
    } catch (error) {
      print("Error creating anchor session: " + error)
    }
  }

  private followSession(session: AnchorSession) {
    session.onAnchorNearby.add((anchor) => {
      this.anchorFoundUnsubscribes.push(
        anchor.onFound.add(() => {
          this.currentAnchor = anchor
          // We are ready to return the transform of the anchor
          this.onAreaAnchorFoundEvent.invoke(this.currentAnchor)
        })
      )
    })
  }

  public createAreaAnchor(anchorPosition: vec3, anchorRotation: quat): void {
    this.anchorSession
      .createWorldAnchor(mat4.compose(anchorPosition, anchorRotation, vec3.one()))
      .then(async (anchor): Promise<UserAnchor> => {
        if (this.anchorSession) {
          return await this.anchorSession.saveAnchor(anchor)
        }
      })
      .then(async (anchor) => {
        if (anchor.state === State.Found) {
          this.currentAnchor = anchor
          this.onAreaAnchorFoundEvent.invoke(this.currentAnchor)
        } 
      })
  }

  public updateAreaAnchor(anchorPosition: vec3, anchorRotation: quat): Promise<UserAnchor> {
    this.currentAnchor.toWorldFromAnchor = mat4.compose(anchorPosition, anchorRotation, vec3.one())
    return this.anchorSession.saveAnchor(this.currentAnchor)
  }
}
```
