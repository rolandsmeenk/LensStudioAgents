# HitTestClassificationTS Reference

```typescript
/**
 * Specs Inc. 2026
 * Hit test classification example for surface type detection. Creates HitTestSession with classification
 * enabled, processes WorldQueryHitTestResult to identify surface types (Ground, None), integrates with
 * SIK InteractionManager for primary interactor tracking, and demonstrates real-time surface recognition
 * for context-aware AR experiences.
 */
import {
  InteractorTriggerType,
  InteractorInputType
} from "SpectaclesInteractionKit.lspkg/Core/Interactor/Interactor"
import {SIK} from "SpectaclesInteractionKit.lspkg/SIK"

const WorldQueryModule =
  require('LensStudio:WorldQueryModule') as WorldQueryModule;

@component
export class HitTestClassification extends BaseScriptComponent {
  private hitTestSession: HitTestSession;
  private primaryInteractor;

  onAwake() {
    this.hitTestSession = this.createHitTestSession();

    this.createEvent('UpdateEvent').bind(this.onUpdate);
  }

  createHitTestSession() {
    const options = HitTestSessionOptions.create();
    options.classification = true;

    const session = WorldQueryModule.createHitTestSessionWithOptions(options);
    return session;
  }

  onHitTestResult = (result: WorldQueryHitTestResult) => {
    if (result === null) {
      // Hit test failed
      return;
    }

    const hitPosition = result.position;
    const hitNormal = result.normal;
    const hitClassification = result.classification;

    switch (hitClassification) {
      case SurfaceClassification.Ground:
        print('Hit ground!');
        break;
      case SurfaceClassification.None:
        print('Hit unknown surface!');
        break;
    }
  };

  onUpdate = () => {
    this.primaryInteractor = SIK.InteractionManager.getTargetingInteractors().shift();
    if (this.primaryInteractor &&
        this.primaryInteractor.isActive() &&
        this.primaryInteractor.isTargeting()
    ) {
        const rayStartOffset = new vec3(this.primaryInteractor.startPoint.x, this.primaryInteractor.startPoint.y, this.primaryInteractor.startPoint.z + 30);
        const rayStart = rayStartOffset;
        const rayEnd = this.primaryInteractor.endPoint;

        this.hitTestSession.hitTest(rayStart, rayEnd, this.onHitTestResult);
    }
  };
}
```

