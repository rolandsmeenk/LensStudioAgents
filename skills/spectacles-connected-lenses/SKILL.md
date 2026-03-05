---
name: spectacles-connected-lenses
description: Reference guide for real-time multiplayer AR on Spectacles using Connected Lenses and Spectacles Sync Kit — covering session creation/joining with joinOrCreateSession (including 'already-in-session' error handling), TransformSyncComponent for position/rotation replication, RealtimeStore for shared key-value state (max 512 bytes per key), NetworkEventSystem for one-shot broadcast events, EntityOwnership for physics authority, Lens Cloud for persistent cross-session data, and patterns for turn-based (Tic Tac Toe) and real-time physics (Air Hockey). Also covers late-joiner state sync, transform drift mitigation, and store size limits. Use this skill whenever multiple Spectacles users need to share AR objects or state — covering Tic Tac Toe, Air Hockey, Laser Pointer, High Five, Shared Sync Controls, Spectacles Sync Kit, and Think Out Loud samples.
---

# Spectacles Connected Lenses — Reference Guide

**Connected Lenses** let multiple Spectacles users share a real-time AR session. The primary framework is the **Spectacles Sync Kit**, built on top of Lens Studio's Lens Cloud networking layer.

**Official docs:** [Spectacles Home](https://developers.snap.com/spectacles/home) · [Connected Lenses](https://developers.snap.com/spectacles/about-spectacles-features/connected-lenses) · [Spectacles Sync Kit](https://developers.snap.com/spectacles/about-spectacles-frameworks/spectacles-sync-kit) (under Spectacles Frameworks)

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

Add via Asset Library (search **"Spectacles Sync Kit"**). Installs: `SyncKit` prefab, `RealtimeStore`, and helper components.

1. Drag the **SyncKit** prefab into your scene.
2. Set a unique **lens ID** on the SyncKit component.
3. Add **`SyncEntity`** + **`TransformSyncComponent`** to objects that need to replicate.

---

## Session Management

```typescript
const connectedLensModule = require('LensStudio:ConnectedLensModule')

// Join or create a session — handles the "already in session" case automatically
connectedLensModule.joinOrCreateSession({}, (session, error) => {
  if (error) {
    print('Session error: ' + error)
    // Common error codes:
    // 'already_in_session' — user is already in a session from another lens
    // 'session_not_found'  — tried to join a session that no longer exists
    return
  }
  print('Session joined. Users: ' + session.users.length)
})

// Listen for user join/leave after session is active
const session = connectedLensModule.getSession()
session.onUserJoined.add((user) => spawnUserAvatar(user))
session.onUserLeft.add((user)   => despawnUserAvatar(user.userId))

// Identify the local user
const myUserId = session.localUser.userId
```

---

## Transform Synchronisation

```typescript
import { TransformSyncComponent } from 'SpectaclesSyncKit/Components/TransformSyncComponent'

// Simply moving the object causes TransformSyncComponent to broadcast the change.
// Remote clients receive position/rotation/scale updates automatically.
this.sceneObject.getTransform().setWorldPosition(newPos)
```

Sync Kit interpolates on remote clients so motion appears smooth.

---

## RealtimeStore — Shared State

> **Size limit**: each key-value pair is capped at **512 bytes**. Store small values (indices, IDs, flags, short strings), not mesh data or large arrays.

```typescript
import { RealtimeStore } from 'SpectaclesSyncKit/Core/RealtimeStore'

// The store is provided by the SyncKit prefab — get it from the SyncEntity, not as a singleton
@input realtimeStore: RealtimeStore

// Write
this.realtimeStore.putString('gameState', 'playing')
this.realtimeStore.putFloat('player1Score', 3)
this.realtimeStore.putBool('isPlayer1Turn', true)

// Read
const state = this.realtimeStore.getString('gameState')

// React to remote changes — always validate incoming values
this.realtimeStore.onValueChanged.add((key: string, value: any) => {
  if (key === 'gameState') {
    // Validate before acting — any client can write to the store
    if (typeof value !== 'string') return
    updateGameUI(value as string)
  }
  if (key === 'player1Score') {
    const score = Number(value)
    if (!isFinite(score) || score < 0 || score > 9999) return  // reject invalid values
    updateScoreboard(score)
  }
})

// Restrict a key so only a specific user can write it
this.realtimeStore.setOwner('player1Score', myUserId)
```

---

## Custom Networked Events

One-shot events broadcast to all clients (good for collisions, high fives, scoring):

```typescript
import { NetworkEventSystem } from 'SpectaclesSyncKit/Core/NetworkEventSystem'

const net = NetworkEventSystem.getInstance()

net.send('SCORE', { userId: myUserId, points: 1 })

net.on('SCORE', (payload) => {
  updateScoreboard(payload.userId, payload.points)
})
```

---

## Turn-Based Pattern (Tic Tac Toe)

```typescript
const MY_ID = connectedLensModule.getSession().localUser.userId

function isMyTurn(): boolean {
  return this.realtimeStore.getString('currentPlayerId') === MY_ID
}

function onCellTapped(cellIndex: number): void {
  if (!isMyTurn()) return
  this.realtimeStore.putInt('cell_' + cellIndex, myPlayerIndex)
  this.realtimeStore.putString('currentPlayerId', getOtherPlayerId())
  checkWinCondition()
}
```

---

## Real-Time Physics Pattern (Air Hockey)

One client acts as physics authority; others receive positions:

```typescript
import { EntityOwnership } from 'SpectaclesSyncKit/Core/EntityOwnership'

const ownership = this.sceneObject.getComponent(EntityOwnership.getTypeName()) as EntityOwnership

if (ownership.isOwner()) {
  runPhysicsUpdate()           // simulate physics here
  // TransformSyncComponent broadcasts result
} else {
  physicsBody.bodyType = Physics.BodyType.Kinematic // just follow sync, don't simulate
}
```

---

## Persistent Shared Data (Across Sessions)

RealtimeStore is ephemeral — lost when the last user leaves. For persistence:

```typescript
const lensCloud = require('LensStudio:LensCloud')

lensCloud.put({ key: 'sharedNote', value: 'Hello AR!' }, 'public_all')

lensCloud.get('sharedNote', 'public_all', (result) => {
  if (result.success) displayNote(result.value)
})
```

> **⚠️ `public_all` is visible to every lens.** Any lens can read data stored at this access level. Use `private_user` for anything user-specific and `public_user` for data you intend to share only within your own lens.

Access levels: `private_user`, `public_user`, `public_all`.

---

## Common Gotchas

- **`joinOrCreateSession`** is the recommended entry point — it handles "already in session" gracefully.
- **`session.localUser`** is the correct property for the local user (not `activeUser`).
- **Last-write-wins** for store conflicts — use Snap Cloud edge functions (see `spectacles-cloud`) for authoritative server-side decisions.
- **Validate incoming RealtimeStore values** before acting on them — any connected client can write any value. Always check type and range.
- **RealtimeStore cap**: 512 bytes per key — store small values (indices, IDs, flags), not large payloads.
- **Design for latency** — events can arrive late or out of order; never assume instant receipt.
- **Sync Kit version** must match your Lens Studio version — re-import after upgrading.
