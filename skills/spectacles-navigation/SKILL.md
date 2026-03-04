---
name: spectacles-navigation
description: Reference guide for location-based AR and navigation on Snap Spectacles — covering Custom Locations scanning and localisation events (including failure/rescan patterns), the Mapbox Map Component (zoom, tile loading, mapStyle), the Snap Places API (POI search, routing), outdoor turn-by-turn navigation with a compass-bearing AR arrow, indoor waypoint navigation with Dijkstra, the GeoLocation API for live GPS, and coordinate conversions between GPS and world space. Use this skill for any lens that anchors AR to a real-world room, building, or outdoor place, shows a map, guides the user somewhere, or queries nearby businesses — covering Custom Locations, Outdoor Navigation, and Navigation Kit samples.
---

# Spectacles Navigation — Reference Guide

Spectacles supports three levels of location-awareness, from precise indoor anchoring to city-scale outdoor navigation.

| Level | API / Feature | Precision |
|---|---|---|
| Indoor / room-scale | Custom Locations | cm-level |
| Outdoor turn-by-turn | Map Component + Outdoor Navigation module | metre-level |
| City-scale lookup | Snap Places API | Address / POI-level |

---

## Custom Locations

Custom Locations let you scan a real-world area (a room, a building entrance, an outdoor plaza) and anchor AR content to it persistently. Users who enter the same scanned area see the content placed exactly where you put it.

### Workflow
1. **Scan** the location using the Spectacles app (Settings → Custom Locations → Scan).
2. **Download** the scan mesh and anchor as a Lens Studio asset.
3. **Place** AR content relative to the anchor in Lens Studio.
4. **Publish** the lens — users within the scanned area will localise automatically.

### Scripting localisation events
```typescript
const locationModule = require('LensStudio:LocationModule');

// Subscribe to localisation state changes
locationModule.onLocalisationUpdate.add((status) => {
  if (status === LocationModule.LocalisationStatus.Localised) {
    print('Localised! Showing AR content.');
    arContentRoot.enabled = true;
  } else {
    arContentRoot.enabled = false;
  }
});
```

### Multiple anchors in one location
You can have several anchors (e.g., one per room). Activate scene objects based on which anchor is active:
```typescript
locationModule.onAnchorUpdate.add((anchors) => {
  anchors.forEach(anchor => {
    const obj = anchorObjectMap[anchor.id];
    if (obj) {
      obj.enabled = anchor.isActive;
      obj.getTransform().setWorldTransform(anchor.transform);
    }
  });
});
```

---

## Map Component

The Map Component renders a Mapbox-powered map tile as a texture or a 3D extruded map. It is used as the base for both the Outdoor Navigation and Navigation Kit samples.

### Setup
1. In Lens Studio, add a **Map Component** to a Scene Object.
2. Configure tile style, zoom level, and centre coordinates in the inspector.
3. Apply the map texture to a plane mesh (2D flat map) or use the extruded 3D mode.

### Controlling the map at runtime
```typescript
const mapComponent = this.sceneObject.getComponent('MapComponent');

// Centre the map on a GPS coordinate
mapComponent.location = { latitude: 48.8566, longitude: 2.3522 }; // Paris

// Change zoom
mapComponent.zoom = 15; // 1 (world) to 20 (building level)

// Switch style
mapComponent.mapStyle = MapComponent.MapStyle.Dark;
```

### Coordinate helpers
Lens Studio's `GeoLocation` API gives you the user's current GPS coordinates:
```typescript
const geo = require('LensStudio:GeoLocation');

geo.onLocationUpdate.add((location) => {
  const { latitude, longitude, altitude, accuracy } = location;
  mapComponent.location = { latitude, longitude };
});
```

---

## Outdoor Navigation

The **Outdoor Navigation** sample integrates the Map Component with the Snap Places API to provide turn-by-turn walking directions.

### Places API — search nearby POIs
```typescript
const placesModule = require('LensStudio:SnapPlacesModule');

placesModule.searchNearby({
  category: 'restaurant',
  radius: 500,       // metres
  maxResults: 10
}, (results) => {
  results.forEach(place => {
    print(place.name + ' at ' + place.address);
    addPinToMap(place.location.latitude, place.location.longitude);
  });
});
```

### Routing
Compute a walking route between two GPS coordinates:
```typescript
placesModule.getDirections({
  origin: userLocation,
  destination: { latitude: destLat, longitude: destLng },
  mode: 'walking'
}, (route) => {
  // route.steps: array of { instruction, startLocation, endLocation, distance, duration }
  route.steps.forEach(step => drawStepOnMap(step));
  showNextInstruction(route.steps[0]);
});
```

### AR directional arrow
Display a 3D arrow that points toward the next waypoint, updating as the user moves:
```typescript
const updateEvent = this.createEvent('UpdateEvent');
updateEvent.bind(() => {
  const userPos = getCurrentGpsPosition();
  const waypointPos = currentWaypoint.location;

  // Compute bearing (compass degrees from user to waypoint)
  const bearing = computeBearing(userPos, waypointPos);

  // Convert bearing to a world-space yaw rotation
  const arrowYaw = bearing - geo.heading; // geo.heading = device compass
  arrowObject.getTransform().setLocalRotation(
    quat.fromEulerAngles(0, arrowYaw * (Math.PI / 180), 0)
  );
});

function computeBearing(from: LatLng, to: LatLng): number {
  const dLng = (to.longitude - from.longitude) * Math.PI / 180;
  const lat1 = from.latitude * Math.PI / 180;
  const lat2 = to.latitude * Math.PI / 180;
  const y = Math.sin(dLng) * Math.cos(lat2);
  const x = Math.cos(lat1) * Math.sin(lat2) - Math.sin(lat1) * Math.cos(lat2) * Math.cos(dLng);
  return (Math.atan2(y, x) * 180 / Math.PI + 360) % 360;
}
```

---

## Indoor Navigation (Navigation Kit)

The **Navigation Kit** sample demonstrates waypoint-based indoor navigation using Custom Locations:

1. **Scan** the building and place `NavigationWaypoint` scene objects in Lens Studio at key corridor turns.
2. **Connect** waypoints with a graph (adjacency list).
3. Run **Dijkstra's or A*** at runtime to find the shortest path from current location to destination.
4. Render path segments using a **LineRenderer** or mesh stamps.
5. Update the active waypoint as the user physically moves closer to each one.

```typescript
// Check proximity to advance to next waypoint
const ADVANCE_THRESHOLD = 1.5; // metres
const userPos = getCurrentWorldPos(); // from scene camera
const nextWaypoint = path[currentWaypointIndex];
const dist = nextWaypoint.getTransform().getWorldPosition().distance(userPos);
if (dist < ADVANCE_THRESHOLD) {
  currentWaypointIndex++;
  displayNavigationStep(path[currentWaypointIndex]);
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

- **Custom Locations require a scan** — you cannot create an anchor programmatically. The scan must be done manually with the Spectacles app before publishing.
- **GPS accuracy on Spectacles** is typically 3–10 metres; don't rely on it for sub-metre accuracy.
- **Compass heading drifts** — apply a complementary filter combining compass + IMU data for smoother arrow rotation.
- **Map tiles require internet** — ensure the lens has internet access enabled, and show a loading state while tiles download.
- **Outdoor Navigation requires OS support** — check the Spectacles compatibility list for your target OS version.
- **Scan coverage matters** — a poorly scanned Custom Location will fail to localise; the scan mesh should cover the full area visible from all entrances.
