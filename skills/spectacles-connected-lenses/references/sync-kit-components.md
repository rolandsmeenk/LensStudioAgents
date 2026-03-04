# Spectacles Sync Kit — Component Reference

## SyncEntity + TransformSyncComponent (scene hierarchy)

```
SceneObject "Puck" (Air Hockey)
├── SyncEntity         ← manages ownership and network identity
├── TransformSyncComponent  ← auto-syncs position/rotation/scale
└── Physics.BodyComponent  ← only simulate on owner
```

## RealtimeStore key-value types

| Method | Type |
|---|---|
| `store.putString(key, val)` | string |
| `store.getString(key)` | string |
| `store.putInt(key, val)` | integer |
| `store.getInt(key)` | integer |
| `store.putFloat(key, val)` | float |
| `store.getFloat(key)` | float |
| `store.putBool(key, val)` | boolean |
| `store.getBool(key)` | boolean |
| `store.onValueChanged.add((key, val) => {})` | subscribe to changes |
| `store.setOwner(key, userId)` | lock key to one user |

> **Size limit**: store only small values (IDs, indices, flags). Avoid storing mesh data or long strings.

## NetworkEventSystem — broadcast events

```typescript
import { NetworkEventSystem } from 'SpectaclesSyncKit/Core/NetworkEventSystem'

const net = NetworkEventSystem.getInstance()

// Send (all clients, including sender, receive it)
net.send('HI_FIVE', { fromUser: myUserId })

// Receive
net.on('HI_FIVE', (payload: { fromUser: string }) => {
  playHighFiveAnimation(payload.fromUser)
})
```

Good for: one-shot notifications (collisions, high fives, score events, gestures)  
Not for: continuous position data (use TransformSyncComponent for that)

## Session management + late-joiner pattern

```typescript
const session = connectedLensModule.getSession()

// On any user joining (including yourself), init their avatar
session.onUserJoined.add((user) => {
  spawnAvatar(user.userId)
  // Late-joiner state sync: send current game state to the new user
  if (session.localUser.isSessionCreator) {
    net.send('STATE_SYNC', getCurrentGameState())
  }
})

session.onUserLeft.add((user) => despawnAvatar(user.userId))
```

## EntityOwnership — physics authority

```typescript
import { EntityOwnership } from 'SpectaclesSyncKit/Core/EntityOwnership'

// Only the owner simulates physics
if (ownership.isOwner()) {
  physicsBody.applyImpulse(force)
  // TransformSyncComponent broadcasts the result to others
} else {
  physicsBody.bodyType = Physics.BodyType.Kinematic  // jus t follow sync
}

// Transfer ownership on grab
ownership.requestOwnership()
```

## Lens Cloud — persistent data (survives session end)

```typescript
const lensCloud = require('LensStudio:LensCloud')

// Access levels
type AccessLevel = 'private_user' | 'public_user' | 'public_all'

lensCloud.put({ key: 'highScore', value: JSON.stringify(42) }, 'public_all')

lensCloud.get('highScore', 'public_all', (result) => {
  if (result.success) print('Persisted score: ' + JSON.parse(result.value))
})
```
