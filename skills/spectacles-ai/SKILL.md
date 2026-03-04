---
name: spectacles-ai
description: Reference guide for AI-powered AR lenses using the Remote Service Gateway (RSG), LLM calls (OpenAI, Gemini, Claude, Lyria), ASR/TTS, camera-frame capture and base64 encoding for vision models, streaming token-by-token responses, agentic tool-call loops, and depth caching. Use this skill for any Spectacles lens that calls an LLM, streams AI tokens to a text component, sends camera frames to a cloud vision model, implements voice-in/voice-out (ASR + TTS with correct session.stop() cleanup), or builds a multi-step agentic loop — covering AI Playground, Agentic Playground, AI Music Gen, Crop, and Depth Cache samples. Use spectacles-networking instead for plain REST/Fetch calls that are not AI-related.
---

# Spectacles AI — Reference Guide

Use this guide when building lenses that connect to AI/ML cloud services. The primary bridge between a lens and external AI models on Spectacles is the **Remote Service Gateway (RSG)**. This is different from the Fetch API (raw HTTP) — the RSG is Snap's managed, authenticated proxy layer that handles auth, rate limiting, and response streaming.

---

## Core Concepts

### Remote Service Gateway (RSG)
- Available via `global.remoteServiceModule` (in TypeScript: `import ... from 'LensStudio:RemoteServiceModule'`)
- Makes authenticated calls to registered cloud endpoints defined in your Lens Studio project's **Remote Services** panel.
- Returns results asynchronously; use callbacks or `async/await` with `promisify`.

### Key APIs on Spectacles

| Capability | API / Module | Notes |
|---|---|---|
| LLM inference | RSG → your backend or Snap RSG built-in endpoints | Stream tokens with `onPartialResponse` |
| Speech-to-Text | `global.asrModule` (ASR Module) | Continuous or one-shot transcription |
| Text-to-Speech | `global.ttsModule` | Synthesises audio from a string |
| Camera frame access | `CameraModule` | Grab frames for vision model input |
| AI Vision / object detection | RSG + depth texture / camera frame | Encode frame → send to cloud vision API |
| Depth texture | `DepthTextureProvider` | Access per-pixel depth for 3D projection |

---

## Remote Service Gateway — Patterns

### Declaring a remote service
In Lens Studio, open **Remote Services** (Project panel → Remote Services) and add an endpoint. Each service has an ID string you reference in script.

### Making a basic call
```typescript
// TypeScript (Lens Studio scripting)
const remoteServiceModule = require('LensStudio:RemoteServiceModule');

function callLLM(prompt: string, onResult: (text: string) => void) {
  const request = RemoteServiceHttpRequest.create();
  request.endpoint = 'my-llm-service';   // matches service ID in project settings
  request.method = RemoteServiceHttpRequest.HttpRequestMethod.Post;
  request.body = JSON.stringify({ prompt });

  remoteServiceModule.performHttpRequest(request, (response) => {
    if (response.statusCode === 200) {
      onResult(JSON.parse(response.body).text);
    }
  });
}
```

### Streaming responses (token-by-token)
The RSG supports streaming. Register a `onPartialResponse` callback before sending:
```typescript
request.onPartialResponse = (partial: string) => {
  // Update UI in real-time as tokens arrive
  textComponent.text += partial;
};
```

---

## Speech-to-Text (ASR)

```typescript
const asrModule = require('LensStudio:AsrModule');

const session = asrModule.startSession();
session.onTranscriptUpdate.add((event) => {
  if (event.isFinal) {
    print('User said: ' + event.transcript);
    session.stop();
  }
});
```
- `isFinal` distinguishes partial (streaming) from committed transcripts.
- Always call `session.stop()` when done; leaving sessions open drains battery.

---

## Text-to-Speech (TTS)

```typescript
const ttsModule = require('LensStudio:TtsModule');

function speak(text: string) {
  const options = TtsTextToSpeechOptions.create();
  options.text = text;
  ttsModule.speak(options, (audioComponent) => {
    audioComponent.play(1);   // play once
  });
}
```
- The callback receives a ready-to-play `AudioComponent`.
- TTS audio competes with other audio sources — set appropriate `AudioMixerChannel`.

---

## Camera Access for Vision

```typescript
// Grab the current camera frame as a texture
const cameraModule = require('LensStudio:CameraModule');

const request = CameraModule.createCameraRequest();
request.cameraId = CameraModule.CameraId.Default_Color;
const cameraTexture = cameraModule.requestCamera(request);

// Encode for transmission
const encodeOptions = EncodeTextureOptions.create();
Base64.encodeTextureAsync(cameraTexture, encodeOptions, (base64) => {
  // Send base64 string to vision API via RSG
  callVisionAPI(base64);
});
```

### Depth Cache pattern (from Depth Cache sample)
Cache multiple depth frames for richer 3D analysis:
- Capture frames on a timer using `ScriptComponent.bind('UpdateEvent', ...)`
- Store frames in a ring buffer
- On user interaction, select the most relevant cached frame and send to a cloud vision model that can project pixels into 3D using the depth data

---

## Agentic Loop Pattern

For autonomous multi-step AI interactions (as in the Agentic Playground sample):

1. User speaks → ASR transcription
2. Transcription sent to LLM via RSG
3. LLM returns an action or a tool call (e.g., `{ "action": "show_object", "params": {...} }`)
4. Lens executes the action in the AR scene
5. Lens sends a tool result back to the LLM for the next turn
6. Loop continues until LLM returns a final response

Key design notes:
- Store conversation history in a JS array and include it in each request body.
- Use a `thinking` or `loading` UI state while waiting for LLM responses.
- Implement a hard iteration cap (e.g., max 10 tool calls) to avoid infinite loops.

---

## AI Music Generation (Lyria / Gemini)

The **AI Music Gen** sample uses Google's Lyria model via RSG:
- User describes genre/vibe via STT
- Description send to Gemini for tag extraction
- Tags forwarded to Lyria endpoint to generate an audio clip
- Clip played back through the `AudioComponent`
- 3D visualiser driven by `AudioSpectrum` data (FFT output)

Pattern:
```
STT → Gemini text → Lyria audio → AudioComponent → 3D mesh driven by FFT
```

---

## Gesture + Vision: Crop Pattern

The **Crop** sample uses a pinch gesture to define a region, captures that region's camera pixels, sends them to a vision model, and overlays the model's output back in world space:

1. Track hand pinch position with SIK `HandInputData`
2. Convert screen position to UV coordinates
3. Crop the camera texture at those UVs (shader or JS crop)
4. Encode cropped region and send to RSG vision endpoint
5. Display result in a world-space panel near the gesture origin

---

## Common Gotchas

- **RSG is not available in the Lens Studio simulator** — test AI features on-device.
- **Large base64 payloads** can hit RSG body-size limits; resize or downsample images before encoding.
- **ASR leaves the microphone open** until you call `session.stop()` — important for privacy and battery.
- **TTS and game audio** share a mixer — prioritise with `AudioMixerChannel.priority`.
- Use **`async/await` with a promisify wrapper** to avoid callback pyramids in complex agentic loops.
- Always handle `response.statusCode !== 200` cases — network errors are common on Spectacles (the device moves around).
