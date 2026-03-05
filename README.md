# Lens Studio Agents & Skills

This repository provides an **AGENTS.MD** and **Skills definitions** designed to be used with **Cursor** and/or **OpenAI Codex**, with a **primary focus on Lens Studio and Spectacles development**.

It is intended for developers building AR lenses and experiences for **Snap Spectacles**, leveraging Lens Studio's TypeScript scripting API, Spectacles Interaction Kit, and Snap's cloud and connectivity APIs. The goal is to offer a structured, AI-friendly knowledge base that accelerates development while staying aligned with Snap's platform conventions and best practices.

---

## Skills

All skill definitions live under `skills/`. Each section below includes the install command for that skill.

### Lens Studio Core

#### lens-studio-scripting
TypeScript component system — `@component`/`@input` decorators, lifecycle (onAwake, OnStartEvent, UpdateEvent, timers, onDestroy), component access patterns, prefab instantiation, logging, and null-reference fixes.

```bash
npx skills add https://github.com/rolandsmeenk/LensStudioAgents --skill lens-studio-scripting
```

#### lens-studio-math
3D math types and utilities — `vec2`, `vec3`, `vec4`, `quat`, `mat4`, `MathUtils` (lerp, remap, clamp), quaternion construction, `quat.lookAt`, `slerp`, world/local transforms, and practical recipes (billboard, smooth follow, world-to-screen).

```bash
npx skills add https://github.com/rolandsmeenk/LensStudioAgents --skill lens-studio-math
```

#### lens-studio-world-query
World understanding and scoring — `WorldQueryModule` HitTestSession for real-world surface detection, semantic surface classification (floor/wall/ceiling/table, Spectacles only), `Physics.createGlobalProbe().rayCast` for scene-collider hits, SIK targeting interactor ray pattern, aligning objects to surface normals, and the LeaderboardModule for in-lens global scoreboards.

```bash
npx skills add https://github.com/rolandsmeenk/LensStudioAgents --skill lens-studio-world-query
```

#### lens-studio-face-tracking
Face and body AR tracking — `FaceTrackingComponent` (multi-face, `faceIndex`), 68 face landmarks, Face Mesh UV texturing, expression weights (mouth open, eye blink, smile, brow raise), 2D/3D face attachments (hat/eye/mouth anchors), Eye Tracking component, Face Retouch/Eye Color/Liquify/Stretch effects, Upper Body Tracking 3D, and Upper Body Mesh for seamless selfie occlusion.

```bash
npx skills add https://github.com/rolandsmeenk/LensStudioAgents --skill lens-studio-face-tracking
```

#### lens-studio-user-context
Snapchat user data and social features — `UserContextSystem` (display name, Bitmoji, profile picture), Bitmoji 2D/3D loading via `BitmojiModule` + `RemoteMediaModule`, Bitmoji Head with live facial animation, Friends API (`FriendsComponent`, `FriendInfo`, Bitmoji Selfies & Stickers), Dynamic Response Poster/Responder mechanic, and Leaderboard.

```bash
npx skills add https://github.com/rolandsmeenk/LensStudioAgents --skill lens-studio-user-context
```

#### lens-studio-vfx
VFX Graph particle system — `VFXComponent`, `.asset.properties` for passing data into a VFX graph at runtime (position, color, float, direction), particle emitter settings (rate, lifetime, shape, blend mode, world vs local space), spawning particles from script, sending face/body tracking data and screen-space positions to VFX, and the SpawnOnTap pattern.

```bash
npx skills add https://github.com/rolandsmeenk/LensStudioAgents --skill lens-studio-vfx
```

#### lens-studio-materials-shaders
Materials and shaders — clone-before-modify rule, `mainPass.baseColor/opacity/baseTex`, blend modes (Normal/Add/Screen/Multiply), depth and cull settings, render order, `RenderTarget` setup for post-processing pipelines, material variants, graph-based Material Editor node system, exposing graph parameters to script, and colour-animated materials.

```bash
npx skills add https://github.com/rolandsmeenk/LensStudioAgents --skill lens-studio-materials-shaders
```

#### lens-studio-2d-ui
2D UI and screen-space interaction — `ScreenTransform` anchors/offsets/size/pivot and coordinate conversions, `ScreenImage` texture and tint, `Text` component, `TouchComponent` (TouchStartEvent/MoveEvent/EndEvent), `TapEvent` for phone lenses, `ScreenRegionComponent` for touch blocking, LSTween UI animations, multi-swatch color picker pattern, and undo stack pattern.

```bash
npx skills add https://github.com/rolandsmeenk/LensStudioAgents --skill lens-studio-2d-ui
```

### Spectacles Platform

#### spectacles-lens-essentials
Foundational Spectacles patterns — Spectacles Interaction Kit (SIK) components (PinchButton, DragInteractable, GrabInteractable, ScrollView), hand-tracking gestures, physics bodies/colliders/callbacks, LSTween animation, materials, spatial anchors, persistent on-device storage, and spatial images.

```bash
npx skills add https://github.com/rolandsmeenk/LensStudioAgents --skill spectacles-lens-essentials
```

#### spectacles-navigation
Location-based AR and navigation — Custom Locations scanning and localisation events, Mapbox Map Component, Snap Places API (POI search, routing), outdoor turn-by-turn navigation with a compass-bearing AR arrow, indoor waypoint navigation with Dijkstra, live GPS, and GPS↔world-space coordinate conversions.

```bash
npx skills add https://github.com/rolandsmeenk/LensStudioAgents --skill spectacles-navigation
```

#### spectacles-connected-lenses
Real-time multiplayer AR — Connected Lenses and Spectacles Sync Kit for session creation/joining, TransformSyncComponent, RealtimeStore shared state, NetworkEventSystem broadcast events, EntityOwnership for physics authority, turn-based (Tic Tac Toe) and real-time physics (Air Hockey) patterns, and Lens Cloud for persistent cross-session data.

```bash
npx skills add https://github.com/rolandsmeenk/LensStudioAgents --skill spectacles-connected-lenses
```

#### spectacles-networking
Fetch API and WebView — GET/POST requests, JSON and binary response handling, custom request headers, Bearer auth, polling with timers, timeout patterns, CORS/HTTPS requirements, and bidirectional WebView messaging.

```bash
npx skills add https://github.com/rolandsmeenk/LensStudioAgents --skill spectacles-networking
```

#### spectacles-cloud
Snap Cloud (Supabase-powered backend) — Postgres REST queries, Row Level Security, Realtime WebSocket subscriptions with reconnect-on-sleep patterns, cloud storage uploads of base64 images, serverless Edge Functions, and companion web dashboard architecture.

```bash
npx skills add https://github.com/rolandsmeenk/LensStudioAgents --skill spectacles-cloud
```

#### spectacles-ai
AI-powered AR lenses — Remote Service Gateway (RSG) for LLM calls (OpenAI, Gemini, Claude, Lyria), ASR/TTS, camera-frame capture and base64 encoding for vision models, streaming token-by-token responses, agentic tool-call loops, AI Music Gen, and depth caching.

```bash
npx skills add https://github.com/rolandsmeenk/LensStudioAgents --skill spectacles-ai
```

#### spectacles-ble
Bluetooth Low Energy — BleModule API (experimental), scanning for peripherals by service UUID, connecting, discoverServices/discoverCharacteristics, read-once and notify-stream patterns, DataView byte parsing, writing command bytes and floats, the disconnection/reconnect loop, and the Spectacles Mobile Kit for phone↔lens JSON messaging.

```bash
npx skills add https://github.com/rolandsmeenk/LensStudioAgents --skill spectacles-ble
```

#### spectacles-snapml
On-device machine learning — MLComponent configuration, supported model formats (TFLite/ONNX), input tensor setup, running inference per frame, parsing SSD/YOLO bounding box output tensors, INT8 quantisation for performance, low-pass filter smoothing, ObjectTracking3D, and integrating detections with physics colliders.

```bash
npx skills add https://github.com/rolandsmeenk/LensStudioAgents --skill spectacles-snapml
```

---

## Documentation

If you want to know more about AGENTS & SKILLS, look at the official documentation.

### AGENTS.MD

[https://agents.md/](https://agents.md/)

### SKILLS.MD

[https://agentskills.io/](https://agentskills.io/)
