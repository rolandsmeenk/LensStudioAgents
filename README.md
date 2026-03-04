# Lens Studio Agents & Skills

This repository provides an **AGENTS.MD** and **Skills definitions** designed to be used with **Cursor** and/or **OpenAI Codex**, with a **primary focus on Lens Studio and Spectacles development**.

It is intended for developers building AR lenses and experiences for **Snap Spectacles**, leveraging Lens Studio's TypeScript scripting API, Spectacles Interaction Kit, and Snap's cloud and connectivity APIs. The goal is to offer a structured, AI-friendly knowledge base that accelerates development while staying aligned with Snap's platform conventions and best practices.

---

## Skills

All skill definitions live under `skills/`. These skills are available on [skill.sh](https://skill.sh). Each section below includes the install command for that skill.

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
World understanding and scoring — `WorldQueryModule` HitTestSession for real-world surface detection, `Physics.createGlobalProbe().rayCast` for scene-collider hits, aligning objects to surface normals, and the LeaderboardModule for in-lens global scoreboards.

```bash
npx skills add https://github.com/rolandsmeenk/LensStudioAgents --skill lens-studio-world-query
```

---

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