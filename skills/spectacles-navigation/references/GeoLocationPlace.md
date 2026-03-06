# GeoLocationPlace Reference

```typescript
/**
 * Specs Inc. 2026
 * Geographic place implementation for navigation based on latitude/longitude coordinates. Maps GeoPositions
 * to world space relative to UserPosition, calculates physical distance using Haversine formula, computes
 * bearing for directional arrows, detects arrival within configurable visit radius, and tracks visit state
 * for progress monitoring in location-based experiences.
 */
import NativeLogger from "SpectaclesInteractionKit.lspkg/Utils/NativeLogger"
import {calculateBearing, getPhysicalDistanceBetweenLocations, normalizeAngle} from "./NavigationUtils"
import {Place} from "./Place"
import {UserPosition, UserPositionStatus} from "./UserPosition"

const TAG = "[GeoLocationPlace]"
const log = new NativeLogger(TAG)

/**
 * Represents a {@link Place} in geo space.
 * Provides functionality for mapping {@link GeoPosition}s into world space based of the {@link UserPosition}.
 *
 * @version 1.0.0
 */
export class GeoLocationPlace extends Place {
  protected readonly userPosition: UserPosition
  private readonly distanceToVisit: number
  protected geoPosition: GeoPosition

  constructor(
    geoPosition: GeoPosition,
    distanceToVisit: number,
    name: string,
    icon: Texture,
    description: string,
    userPosition: UserPosition,
  ) {
    super(name, icon, description)
    this.userPosition = userPosition
    this.geoPosition = geoPosition
    this.distanceToVisit = distanceToVisit
  }

  public getGeoPosition(): GeoPosition | null {
    return this.geoPosition
  }

  public requestNewGeoPosition(geoPosition: GeoPosition): boolean {
    this.geoPosition = geoPosition
    return true
  }

  public getRelativePosition(): vec3 | null {
    return this.calculateRelativePosition(0)
  }

  public getOrientation(): quat {
    return quat.quatIdentity()
  }

  private getPhysicalDistance(userPosition: UserPosition): number | null {
    const userGeoPosition = userPosition.getGeoPosition()
    if (isNull(userGeoPosition)) {
      return null
    }
    return getPhysicalDistanceBetweenLocations(userGeoPosition, this.getGeoPosition())
  }

  protected calculateRelativePosition(yOffset: number): vec3 | null {
    const userTransform = this.userPosition.getRelativeTransform()
    const cameraForward = userTransform.back
    const userForward = cameraForward.projectOnPlane(vec3.up()).normalize()
    const distance = this.getPhysicalDistance(this.userPosition)
    const bearing = this.getBearing(this.userPosition) - this.userPosition.getBearing()
    const pinAltitude = this.getGeoPosition().altitude
    const userAltitude = this.userPosition.getGeoPosition()?.altitude ?? 0

    const projectedPosition: vec3 = this.userPosition
      .getRelativeTransform()
      .getWorldPosition()
      .add(
        quat
          .fromEulerAngles(0, -bearing, 0)
          .multiplyVec3(userForward)
          .uniformScale(distance * 100),
      )
      .add(new vec3(0, yOffset + (pinAltitude - userAltitude) * 100, 0))

    projectedPosition.y = userTransform.getWorldPosition().y
    return projectedPosition
  }

  public getBearing(userPosition: UserPosition): number | null {
    if (userPosition.status !== UserPositionStatus.GeoLocalizationAvailable) {
      log.i("Bearing requested, but user position is " + userPosition.status)
      return null
    }
    const userGeoPosition = userPosition.getGeoPosition()
    const locationBearing = calculateBearing(userGeoPosition, this.getGeoPosition())
    return normalizeAngle(locationBearing)
  }

  protected checkVisited(): boolean {
    const distance = this.getPhysicalDistance(this.userPosition)
    return !isNull(distance) && distance < this.distanceToVisit
  }
}

```

