---
name: spectacles-connected-lenses
description: Reference guide for real-time multiplayer AR on Spectacles using Connected Lenses and Spectacles Sync Kit — covering session creation/joining (including 'already-in-session' error handling), TransformSyncComponent for position/rotation replication, RealtimeStore for shared key-value state, NetworkEventSystem for one-shot broadcast events, EntityOwnership for physics authority, Lens Cloud for persistent cross-session data, and patterns for turn-based (Tic Tac Toe) and real-time physics (Air Hockey). Also covers late-joiner state sync, transform drift mitigation, and store size limits. Use this skill whenever multiple Spectacles users need to share AR objects or state — covering Tic Tac Toe, Air Hockey, Laser Pointer, High Five, Shared Sync Controls, Spectacles Sync Kit, and Think Out Loud samples.
---

# Spectacles Connected Lenses — Reference Guide

**Connected Lenses** let multiple Spectacles users share a real-time AR session. The primary framework is the **Spectacles Sync Kit**, built on top of Lens Studio's Lens Cloud networking layer.

---

## Architecture

```
User A (Spectacles) ──┐
                       ├─── Lens Cloud (Snap servers) ─── Shared session state
User B (Spectacles) ──┘                                   (transforms, store, events)
```

The Sync Kit handles session creation/joining, object ownership, delta sync, and conflict resolution.

---

## Sync Kit Setup

Add via Asset Library (search "Spectacles Sync Kit"). Installs: `SyncKit` prefab, `RealtimeStore`, and helper components.

1. Drag the **SyncKit** prefab into your scene.
2. Set a unique **lens ID** on the SyncKit component.
3. Add **`SyncEntity`** + **`TransformSyncComponent`** to objects that need to replicate.

---

## Transform Synchronisation

```typescript
import { TransformSyncComponent } from 'SpectaclesSyncKit/Components/TransformSyncComponent';

// Simply moving the object causes TransformSyncComponent to broadcast the change.
// Remote clients receive position/rotation/scale updates automatically.
this.sceneObject.getTransform().setWorldPosition(newPos);
```

Sync Kit interpolates on remote clients so motion appears smooth.

---

## RealtimeStore — Shared State

```typescript
import { RealtimeStore } from 'SpectaclesSyncKit/Core/RealtimeStore';

const store = RealtimeStore.getInstance();

// Write
store.putString('gameState', 'playing');
store.putFloat('player1Score', 3);
store.putBool('isPlayer1Turn', true);

// Read
const state = store.getString('gameState');

// React to remote changes
store.onValueChanged.add((key, value) => {
  if (key === 'gameState') updateGameUI(value as string);
});

// Restrict a key to one user (optional)
store.setOwner('player1Score', myUserId);
```

---

## Custom Networked Events

One-shot events broadcast to all clients (good for collisions, high fives, scoring):

```typescript
import { NetworkEventSystem } from 'SpectaclesSyncKit/Core/NetworkEventSystem';

const net = NetworkEventSystem.getInstance();

net.send('SCORE', { userId: myUserId, points: 1 });

net.on('SCORE', (payload) => {
  updateScoreboard(payload.userId, payload.points);
});
```

---

## Session Management

```typescript
const connectedLensModule = require('LensStudio:ConnectedLensModule');
const session = connectedLensModule.getSession();

print('Users: ' + session.users.length);

session.onUserJoined.add((user) => spawnUserAvatar(user));
session.onUserLeft.add((user) => despawnUserAvatar(user.userId));
```

---

## Turn-Based Pattern (Tic Tac Toe)

```typescript
const MY_ID = connectedLensModule.getSession().localUser.userId;

function isMyTurn(): boolean {
  return store.getString('currentPlayerId') === MY_ID;
}

function onCellTapped(cellIndex: number) {
  if (!isMyTurn()) return;
  store.putInt('cell_' + cellIndex, myPlayerIndex);
  store.putString('currentPlayerId', getOtherPlayerId());
  checkWinCondition();
}
```

---

## Real-Time Physics Pattern (Air Hockey)

One client acts as physics authority; others receive positions:

```typescript
import { EntityOwnership } from 'SpectaclesSyncKit/Core/EntityOwnership';

const ownership = this.sceneObject.getComponent(EntityOwnership.getTypeName());

if (ownership.isOwner()) {
  runPhysicsUpdate();           // simulate physics here
  // TransformSyncComponent broadcasts result
} else {
  physicsBody.bodyType = Physics.BodyType.Kinematic; // just follow sync, don't simulate
}
```

---

## Persistent Shared Data (Across Sessions)

RealtimeStore is ephemeral — lost when the last user leaves. For persistence:

```typescript
const lensCloud = require('LensStudio:LensCloud');

lensCloud.put({ key: 'sharedNote', value: 'Hello AR!' }, 'public_all');

lensCloud.get('sharedNote', 'public_all', (result) => {
  if (result.success) displayNote(result.value);
});
```

Access levels: `private_user`, `public_user`, `public_all`.

---

## Common Gotchas

- **One session at a time** — handle the case where the user is already in a session.
- **Last-write-wins** for store conflicts — use server-side logic (Snap Cloud edge functions) for authoritative decisions.
- **Design for latency** — events can arrive late or out of order; never assume instant receipt.
- **Sync Kit version** must match your Lens Studio version — re-import after upgrading.
- **Store size** — store small values (indices, IDs, flags), not large payloads or mesh data.
