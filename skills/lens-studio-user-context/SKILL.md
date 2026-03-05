---
name: lens-studio-user-context
description: Reference guide for Snapchat user data and social features in Lens Studio — covering UserContextSystem (display name, Bitmoji, profile picture), Bitmoji 2D (requestBitmoji2DResource + RemoteMediaModule fetch pattern), Bitmoji 3D (requestBitmoji3DResource, AnimationMixer for playback), Bitmoji Head with live facial animation, Friends API (FriendsComponent, FriendInfo, listing friends and their Bitmojis), Dynamic Response Poster/Responder mechanic (tappable areas, reading Poster data in the Responder flow with DynamicResponseComponent), and LeaderboardModule (Leaderboard Custom Component, Score Widget, submit/retrieve with OrderingType and UsersType). Use this skill whenever a lens needs the current user's name or avatar, accesses friends' Bitmoji or data, implements a send-and-respond mechanic, or adds a global score leaderboard.
---

# Lens Studio User Context — Reference Guide

This guide covers Snapchat social APIs available in Lens Studio: user identity, Bitmoji avatars, friends, social sharing (Dynamic Response), and leaderboards.

---

## UserContextSystem

`UserContextSystem` provides information about the current Snapchat user.

```typescript
const userContextSystem = global.userContextSystem

// Get the current user's SnapchatUser object
const currentUser: SnapchatUser = userContextSystem.getCurrentUser()
print('Display name: ' + currentUser.displayName)

// Check if the user has a Bitmoji
if (currentUser.hasBitmoji()) {
  loadBitmoji2D(currentUser)
}
```

---

## Bitmoji 2D (Sticker)

Load a user's Bitmoji as a flat 2D texture:

```typescript
const bitmojiModule = require('LensStudio:BitmojiModule')
const remoteMediaModule = require('LensStudio:RemoteMediaModule')

function loadBitmoji2D(user: SnapchatUser): void {
  // Step 1: Request the Bitmoji 2D resource URL
  const resource = bitmojiModule.requestBitmoji2DResource(user)

  // Step 2: Fetch the resource and apply it as a texture
  remoteMediaModule.loadResourceAsImageTexture(
    resource,
    (texture: Texture) => {
      // Apply the texture to a screen image or material
      screenImage.mainPass.baseTex = texture
      print('Bitmoji 2D loaded for: ' + user.displayName)
    },
    (error: string) => {
      print('Failed to load Bitmoji 2D: ' + error)
    }
  )
}
```

---

## Bitmoji 3D

Load a user's full 3D Bitmoji avatar into the scene:

```typescript
function loadBitmoji3D(user: SnapchatUser, parent: SceneObject): void {
  // Step 1: Request the 3D resource
  const resource = bitmojiModule.requestBitmoji3DResource(user)

  // Step 2: Fetch and instantiate as a scene object
  remoteMediaModule.loadResourceAsSceneObject(
    resource,
    (bitmojiObject: SceneObject) => {
      bitmojiObject.setParent(parent)
      bitmojiObject.getTransform().setLocalPosition(vec3.zero())
      print('Bitmoji 3D loaded for: ' + user.displayName)

      // Animate: find the AnimationMixer on the loaded Bitmoji
      const animator = bitmojiObject.getComponent('Component.AnimationMixer')
      if (animator) {
        animator.startAllAnimations()
      }
    },
    (error: string) => {
      print('Failed to load Bitmoji 3D: ' + error)
    }
  )
}
```

### Playing a specific animation on a Bitmoji 3D
```typescript
const animator = bitmojiObject.getComponent('Component.AnimationMixer')

// Play a named animation clip
animator.setClipWeight('wave', 1.0)
animator.setClipEnabled('wave', true)

// Stop
animator.setClipEnabled('wave', false)
```

---

## Bitmoji Head (Live Face Animation)

The Bitmoji Head reacts to the user's face in real time using face tracking.

1. Add a **Bitmoji Head** to the scene (+ → Bitmoji → Bitmoji Head).
2. Connect it to the **FaceTracking** component on your Head object.
3. The Bitmoji head will mirror the user's expressions (mouth open, eyebrows, etc.) automatically.

To access the Bitmoji head in script:
```typescript
const bitmojiHead = bitmojiHeadObject.getComponent('Component.BitmojiHeadComponent')
bitmojiHead.faceTracker = faceTrackingComponent
```

---

## Friends API

### Listing friends with FriendsComponent

```typescript
// Add a FriendsComponent to your scene object in the inspector,
// then access it from script:
const friendsComponent = this.sceneObject.getComponent('Component.FriendsComponent')

// Get the list of friends (up to the platform limit)
friendsComponent.getFriends((friends: SnapchatUser[]) => {
  friends.forEach(friend => {
    print('Friend: ' + friend.displayName)
    loadBitmoji2D(friend)   // load their Bitmoji (reuse function from above)
  })
})
```

### FriendInfo Component

Shows a single friend's avatar and display name in a UI panel:

1. Add a **FriendInfo** component to a scene object.
2. Set the `friend` input to a `SnapchatUser` from the Friends API.

```typescript
const friendInfoComponent = this.sceneObject.getComponent('Component.FriendInfoComponent')
friendInfoComponent.friend = selectedFriend  // set to a SnapchatUser from getFriends()
```

### Bitmoji Selfies & Stickers Component

Displays a side-by-side or combined Bitmoji image for the user and a friend:

```typescript
const selfieStickerComponent = this.sceneObject.getComponent('Component.BitmojiSelfiesStickerComponent')

// Set user and friend for a combined selfie Bitmoji sticker
selfieStickerComponent.primaryUser = currentUser
selfieStickerComponent.secondaryUser = selectedFriend
```

---

## Dynamic Response (Poster / Responder mechanic)

Dynamic Response lets a lens share data and Snap media between a **Poster** (the person who sends a Snap) and **Responders** (friends who receive it and tap to open).

### Flow
1. Poster opens the lens, customises it, and sends/posts a Snap.
2. Responder taps the Snap; the lens opens in Responder mode with data from the Poster.

### Setup
1. Download the **Dynamic Response** component from the Asset Library.
2. Add the `DynamicResponseComponent` to a scene object.
3. Define **tappable areas** in the inspector (regions the Responder can tap on the received Snap).

### Reading Poster data in Responder mode
```typescript
const dynamicResponse = this.sceneObject.getComponent('Component.DynamicResponseComponent')

dynamicResponse.onResponderActivated.add(() => {
  // We are in Responder mode — read data the Poster embedded
  // Always sanitise: Poster data is a plain string with no schema enforcement
  const raw: string = dynamicResponse.getPosterData('myKey') ?? ''
  const posterData = raw.slice(0, 256)  // cap length; validate further if driving logic
  print('Poster sent: ' + posterData)

  // Show the Responder-specific UI
  responderUI.enabled = true
  posterUI.enabled    = false
})
```

### Writing data as the Poster
```typescript
// Called when the Poster is about to send the Snap
dynamicResponse.setPosterData('myKey', 'Hello Responder!')
dynamicResponse.setPosterData('score', '42')
```

### Checking which mode we're in
```typescript
if (dynamicResponse.isPoster()) {
  print('We are the Poster — show customisation UI')
} else if (dynamicResponse.isResponder()) {
  print('We are the Responder — read Poster data')
}
```

---

## Leaderboard Module

For the full API reference with code examples, see the `lens-studio-world-query` skill. Summary:

```typescript
const leaderboardModule = require('LensStudio:LeaderboardModule')

const options = Leaderboard.CreateOptions.create()
options.name = 'MY_LEADERBOARD'
options.ttlSeconds = 0          // 0 = permanent
options.orderingType = Leaderboard.OrderingType.Descending

leaderboardModule.getLeaderboard(options,
  (lb) => {
    // Submit
    lb.submitScore(100, (info) => print('Submitted for ' + info.snapchatUser.displayName), print)

    // Retrieve top 10
    const r = Leaderboard.RetrievalOptions.create()
    r.usersLimit = 10
    r.usersType = Leaderboard.UsersType.Global
    lb.getLeaderboardInfo(r,
      (records, myRecord) => {
        records.forEach((rec, i) => print(`#${i+1}: ${rec.snapchatUser?.displayName} — ${rec.score}`))
      },
      print
    )
  },
  print
)
```

---

## Common Gotchas

- **`UserContextSystem` requires user consent** — if the user hasn't granted the Bitmoji permission, `hasBitmoji()` returns `false`. Always check before requesting.
- **Bitmoji loading is async** — always update the UI in the `remoteMediaModule` callback, not immediately after calling `requestBitmoji3DResource`.
- **`getFriends()` list size** is platform-limited — don't assume you can get all friends; design for partial lists.
- **Dynamic Response Poster data is an unvalidated string.** Always sanitise it (cap length, check format) before using it to drive UI or game logic — a crafted Snap could inject arbitrary content.
- **Dynamic Response tappable areas**: if you add tappable areas in the inspector, the platform's Call-to-Action (CTA) button is replaced by the tappable shimmer on the received Snap.
- **Dynamic Response is not available in the Lens Studio simulator** — test Poster/Responder flow via Snapchat on two devices.
- **Leaderboard scores are submitted from the client.** There is no server-side score validation — for competitive lenses, use a Snap Cloud Edge Function to verify scores before writing to the leaderboard.
- **Leaderboard names are global** to the lens — two lenses with the same name string share the same leaderboard. Use unique names.
