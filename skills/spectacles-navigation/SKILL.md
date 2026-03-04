---
name: spectacles-navigation
description: Reference guide for location-based AR and navigation on Snap Spectacles — covering Custom Locations scanning and localisation events (including full LocalisationStatus enum: Localised, NotLocalised, LocalisationLost, Unavailable, plus failure/rescan patterns), the Mapbox Map Component (zoom, tile loading, mapStyle), the Snap Places API (POI search, routing), outdoor turn-by-turn navigation with a compass-bearing AR arrow, indoor waypoint navigation with Dijkstra, the GeoLocation API for live GPS (requires Location capability), and coordinate conversions between GPS and world space. Use this skill for any lens that anchors AR to a real-world room, building, or outdoor place, shows a map, guides the user somewhere, or queries nearby businesses — covering Custom Locations, Outdoor Navigation, and Navigation Kit samples.
---

# Spectacles Navigation — Reference Guide

Spectacles supports three levels of location-awareness, from precise indoor anchoring to city-scale outdoor navigation.

**Official docs:** [Spectacles Home](https://developers.snap.com/spectacles/home) · [Spectacles Navigation Kit](https://developers.snap.com/spectacles/about-spectacles-frameworks/spectacles-navigation-kit) · [Start Building](https://developers.snap.com/spectacles/get-started/start-building/build-your-first-spectacles-lens-tutorial)

| Level | API / Feature | Precision |
|---|---|---|
| Indoor / room-scale | Custom Locations | cm-level |
| Outdoor turn-by-turn | Map Component + Outdoor Navigation module | metre-level |
| City-scale lookup | Snap Places API | Address / POI-level |

---

## Custom Locations

Custom Locations let you scan a real-world area (a room, a building entrance, an outdoor plaza) and anchor AR content to it persistently. Users who enter the same scanned area see content placed exactly where you put it.

### Workflow
1. **Scan** the location using the **Spectacles app** (Settings → Custom Locations → Scan) — scanning must be done on-device, not in the Lens Studio desktop simulator.
2. **Download** the scan mesh and anchor as a Lens Studio asset.
3. **Place** AR content relative to the anchor in Lens Studio.
4. **Publish** the lens — users within the scanned area will localise automatically.

### Scripting localisation events
```typescript
const locationModule = require('LensStudio:LocationModule')

locationModule.onLocalisationUpdate.add((status: LocationModule.LocalisationStatus) => {
  switch (status) {
    case LocationModule.LocalisationStatus.Localised:
      print('Localised! Showing AR content.')
      arContentRoot.enabled = true
      break

    case LocationModule.LocalisationStatus.NotLocalised:
      print('Not yet localised — scanning...')
      arContentRoot.enabled = false
      break

    case LocationModule.LocalisationStatus.LocalisationLost:
      print('Localisation lost — user may have moved too far.')
      arContentRoot.enabled = false
      // Prompt user to look around the scanned area again
      break

    case LocationModule.LocalisationStatus.Unavailable:
      print('Custom Locations unavailable on this device/OS version.')
      break
  }
})
```

### Multiple anchors in one location
```typescript
locationModule.onAnchorUpdate.add((anchors: LocationModule.Anchor[]) => {
  anchors.forEach(anchor => {
    const obj = anchorObjectMap[anchor.id]
    if (obj) {
      obj.enabled = anchor.isActive
      obj.getTransform().setWorldTransform(anchor.transform)
    }
  })
})
```

---

## Map Component

The Map Component renders a Mapbox-powered map tile as a texture or a 3D extruded map.

### Setup
1. In Lens Studio, add a **Map Component** to a Scene Object.
2. Configure tile style, zoom level, and centre coordinates in the inspector.
3. Apply the map texture to a plane mesh (2D flat map) or use the extruded 3D mode.

### Controlling the map at runtime
```typescript
const mapComponent = this.sceneObject.getComponent('MapComponent')

mapComponent.location = { latitude: 48.8566, longitude: 2.3522 } // Paris

mapComponent.zoom = 15 // 1 (world) to 20 (building level)

mapComponent.mapStyle = MapComponent.MapStyle.Dark
```

### Coordinate helpers
```typescript
// Convert GPS coords to world-space vec3
const worldPos: vec3 = mapComponent.latLngToWorldPos(latitude, longitude)

// Convert world-space pos back to GPS
const gpsCoord = mapComponent.worldPosToLatLng(worldPos)
```

### Live GPS (GeoLocation API)

> **Requires** enabling the **Location** capability: *Project Settings → Capabilities → Location*

```typescript
const geo = require('LensStudio:GeoLocation')

geo.onLocationUpdate.add((location) => {
  const { latitude, longitude, altitude, accuracy } = location
  mapComponent.location = { latitude, longitude }
  print(`GPS accuracy: ±${accuracy}m`)
})
```

GPS accuracy on Spectacles is typically 3–10 metres; don't rely on it for sub-metre precision.

---

## Outdoor Navigation

The **Outdoor Navigation** sample integrates the Map Component with the Snap Places API to provide turn-by-turn walking directions.

### Places API — search nearby POIs
```typescript
const placesModule = require('LensStudio:SnapPlacesModule')

placesModule.searchNearby({
  category: 'restaurant',
  radius: 500,       // metres
  maxResults: 10
}, (results) => {
  results.forEach(place => {
    print(place.name + ' at ' + place.address)
    addPinToMap(place.location.latitude, place.location.longitude)
  })
})
```

### Routing
```typescript
placesModule.getDirections({
  origin: userLocation,
  destination: { latitude: destLat, longitude: destLng },
  mode: 'walking'
}, (route) => {
  route.steps.forEach(step => drawStepOnMap(step))
  showNextInstruction(route.steps[0])
})
```

### AR directional arrow
```typescript
const updateEvent = this.createEvent('UpdateEvent')
updateEvent.bind(() => {
  const userPos = getCurrentGpsPosition()
  const waypointPos = currentWaypoint.location

  const bearing = computeBearing(userPos, waypointPos)
  const arrowYaw = bearing - geo.heading // geo.heading = device compass
  arrowObject.getTransform().setLocalRotation(
    quat.fromEulerAngles(0, arrowYaw * (Math.PI / 180), 0)
  )
})

function computeBearing(from: {latitude: number, longitude: number},
                         to:   {latitude: number, longitude: number}): number {
  const dLng = (to.longitude - from.longitude) * Math.PI / 180
  const lat1 = from.latitude * Math.PI / 180
  const lat2 = to.latitude  * Math.PI / 180
  const y = Math.sin(dLng) * Math.cos(lat2)
  const x = Math.cos(lat1) * Math.sin(lat2) - Math.sin(lat1) * Math.cos(lat2) * Math.cos(dLng)
  return (Math.atan2(y, x) * 180 / Math.PI + 360) % 360
}
```

---

## Indoor Navigation (Navigation Kit)

1. **Scan** the building and place `NavigationWaypoint` scene objects in Lens Studio.
2. **Connect** waypoints with an adjacency list.
3. Run **Dijkstra's or A\*** at runtime to find the shortest path.
4. Render path segments using a **LineRenderer** or mesh stamps.
5. Update the active waypoint as the user physically moves closer.

```typescript
const ADVANCE_THRESHOLD = 1.5 // metres
const userPos = getCurrentWorldPos()
const nextWaypoint = path[currentWaypointIndex]
const dist = nextWaypoint.getTransform().getWorldPosition().distance(userPos)
if (dist < ADVANCE_THRESHOLD) {
  currentWaypointIndex++
  displayNavigationStep(path[currentWaypointIndex])
}
```

---

## Coordinate Conversions

| Conversion | Utility |
|---|---|
| GPS → world space | `mapComponent.latLngToWorldPos(lat, lng)` |
| World space → GPS | `mapComponent.worldPosToLatLng(worldPos)` |
| Metres per degree (latitude) | ≈ 111,320 m per degree |
| Metres per degree (longitude) | ≈ 111,320 × cos(lat) m per degree |

---

## Common Gotchas

- **Custom Locations require a scan on-device** — you cannot create an anchor programmatically or in the desktop simulator.
- **GPS accuracy on Spectacles** is typically 3–10 metres; don't rely on it for sub-metre accuracy.
- **`LocalisationStatus.LocalisationLost`** is distinct from `NotLocalised` — it means the user was localised but the tracking was disrupted. Prompt them to look around the scanned area.
- **`LocalisationStatus.Unavailable`** means the OS or device doesn't support Custom Locations — handle gracefully.
- **Location capability**: always enable *Project Settings → Capabilities → Location* before using the GeoLocation API, or the module will be undefined.
- **Compass heading drifts** — apply a complementary filter combining compass + IMU data for smoother arrow rotation.
- **Map tiles require internet** — ensure the lens has internet access enabled, and show a loading state while tiles download.
- **Scan coverage matters** — a poorly scanned Custom Location will fail to localise; the scan mesh should cover the full area visible from all entrances.
