# Navigation — GPS Utilities and Coordinate Conversions Reference

## Bearing (compass heading) between two GPS points

```typescript
interface LatLng { latitude: number; longitude: number }

function computeBearing(from: LatLng, to: LatLng): number {
  const φ1  = from.latitude  * Math.PI / 180
  const φ2  = to.latitude    * Math.PI / 180
  const dλ  = (to.longitude - from.longitude) * Math.PI / 180

  const y = Math.sin(dλ) * Math.cos(φ2)
  const x = Math.cos(φ1) * Math.sin(φ2) - Math.sin(φ1) * Math.cos(φ2) * Math.cos(dλ)
  return (Math.atan2(y, x) * 180 / Math.PI + 360) % 360   // [0, 360)
}
```

## Haversine distance (metres between two GPS points)

```typescript
function haversineDistance(from: LatLng, to: LatLng): number {
  const R  = 6371000   // Earth radius in metres
  const φ1 = from.latitude  * Math.PI / 180
  const φ2 = to.latitude    * Math.PI / 180
  const dφ = (to.latitude  - from.latitude)  * Math.PI / 180
  const dλ = (to.longitude - from.longitude) * Math.PI / 180

  const a = Math.sin(dφ/2) ** 2 + Math.cos(φ1) * Math.cos(φ2) * Math.sin(dλ/2) ** 2
  return R * 2 * Math.atan2(Math.sqrt(a), Math.sqrt(1 - a))
}
```

## AR directional arrow aligned to compass

```typescript
// In UpdateEvent, after reading GeoLocation and computing bearing
function updateArrow(arrowObj: SceneObject, targetLatLng: LatLng, currentLatLng: LatLng, deviceHeading: number): void {
  const bearingDeg = computeBearing(currentLatLng, targetLatLng)
  const relYawDeg  = bearingDeg - deviceHeading        // relative to device facing

  arrowObj.getTransform().setLocalRotation(
    quat.fromEulerAngles(0, relYawDeg * MathUtils.DegToRad, 0)
  )
}
```

## Metres to degree conversions

| Axis | Formula |
|---|---|
| Latitude | 1 degree ≈ 111,320 metres |
| Longitude | 1 degree ≈ 111,320 × cos(lat°) metres |

```typescript
function metresToDegreesLat(metres: number): number {
  return metres / 111320
}
function metresToDegreesLng(metres: number, latDeg: number): number {
  return metres / (111320 * Math.cos(latDeg * Math.PI / 180))
}
```

## GeoLocation event subscription

```typescript
const geo = require('LensStudio:GeoLocation')

let lastLocation: LatLng | null = null
let deviceHeading = 0

geo.onLocationUpdate.add((loc) => {
  lastLocation = { latitude: loc.latitude, longitude: loc.longitude }
})

geo.onHeadingUpdate.add((heading) => {
  // heading.trueHeading — degrees clockwise from true north
  deviceHeading = heading.trueHeading
})
```

## Custom Locations — localisation state machine

```typescript
const locationModule = require('LensStudio:LocationModule')

enum LocalState { Scanning, Localised, Lost }
let state: LocalState = LocalState.Scanning

locationModule.onLocalisationUpdate.add((status) => {
  if (status === LocationModule.LocalisationStatus.Localised) {
    state = LocalState.Localised
    arContentRoot.enabled = true
    showUI('Localised!')
  } else if (status === LocationModule.LocalisationStatus.Failed) {
    state = LocalState.Lost
    arContentRoot.enabled = false
    showUI('Could not localise. Walk around the scanned area.')
  }
})
```

> **Scanning coverage**: the location must be scanned from multiple directions and lighting conditions. A scan covering only one entrance angle will fail for users approaching from other directions.

