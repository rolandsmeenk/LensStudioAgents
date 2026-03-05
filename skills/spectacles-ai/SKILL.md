---
name: spectacles-ai
description: Reference guide for AI-powered AR lenses using the Remote Service Gateway (RSG) — covering LLM calls (OpenAI, Gemini, Claude), AI music generation (Lyria), ASR speech-to-text with language/accuracy options, TTS text-to-speech, camera frame capture and base64 encoding for vision models, streaming token-by-token responses via onPartialResponse, multi-turn conversation history, and agentic tool-call loops. Use this skill whenever a lens needs to call an LLM or AI model via RSG, stream AI tokens to a UI text element, transcribe voice (ASR) or synthesise speech (TTS), send a camera image to a cloud vision API, or run a multi-step autonomous loop. Do NOT use this for plain REST calls to non-AI backends — use spectacles-networking for those.
---

# Spectacles AI — Reference Guide

The primary bridge between a lens and external AI models on Spectacles is the **Remote Service Gateway (RSG)**. RSG is Snap's managed, authenticated proxy layer that handles auth, rate limiting, and response streaming for any registered endpoint — not just AI endpoints.

**Official docs:** [Spectacles Home](https://developers.snap.com/spectacles/home) · [Features Overview](https://developers.snap.com/spectacles/about-spectacles-features/overview) (Camera Module for vision input)

---

## Core Concepts

### Remote Service Gateway (RSG)
- Available via `require('LensStudio:RemoteServiceModule')`
- Makes authenticated calls to registered cloud endpoints defined in your Lens Studio project's **Remote Services** panel.
- Returns results asynchronously; use callbacks or `async/await`.

### Key APIs on Spectacles

| Capability | API / Module | Notes |
|---|---|---|
| LLM inference | RSG → your backend or Snap RSG built-in endpoints | Stream tokens with `onPartialResponse` |
| Speech-to-Text | `AsrModule` | 40+ languages, accuracy modes, mixed-language |
| Text-to-Speech | `TtsModule` | Synthesises audio from a string |
| Camera frame access | `CameraModule` | Grab frames for vision model input |
| AI Vision / object detection | RSG + depth texture / camera frame | Encode frame → send to cloud vision API |
| Depth texture | `DepthTextureProvider` | Access per-pixel depth for 3D projection |

---

## Remote Service Gateway — Patterns

### Declaring a remote service
In Lens Studio, open **Remote Services** (Project panel → Remote Services) and add an endpoint. Each service has an ID string you reference in script.

### Making a basic call
```typescript
const remoteServiceModule = require('LensStudio:RemoteServiceModule')

function callLLM(prompt: string, onResult: (text: string) => void): void {
  const request = RemoteServiceHttpRequest.create()
  request.endpoint = 'my-llm-service'   // matches service ID in Remote Services panel
  request.method = RemoteServiceHttpRequest.HttpRequestMethod.Post
  request.body = JSON.stringify({ prompt })

  remoteServiceModule.performHttpRequest(request, (response) => {
    if (response.statusCode === 200) {
      onResult(JSON.parse(response.body).text)
    } else {
      print('RSG error: ' + response.statusCode)
    }
  })
}
```

### Streaming responses (token-by-token)
```typescript
request.onPartialResponse = (partial: string) => {
  // Update UI in real-time as tokens arrive
  textComponent.text += partial
}
```

---

## Speech-to-Text (ASR)

### Full example with language and accuracy options
```typescript
const asrModule = require('LensStudio:AsrModule')

@component
export class VoiceInput extends BaseScriptComponent {
  onAwake(): void {
    const options = AsrModule.Options.create()

    // Language: BCP-47 tag, e.g. 'en-US', 'fr-FR', 'ja-JP'
    // 40+ languages supported. Leave unset for auto-detect.
    options.locale = 'en-US'

    // Accuracy: Balanced (faster, lower power) or High (more accurate, slower)
    options.accuracy = AsrModule.Accuracy.Balanced   // or AsrModule.Accuracy.High

    // Mixed language: allow transcription to switch languages mid-sentence
    options.mixedLanguages = false

    const session = asrModule.startSession(options)

    session.onTranscriptUpdate.add((event: AsrModule.TranscriptUpdateEvent) => {
      print('Transcript: ' + event.transcript)

      if (event.isFinal) {
        // Committed utterance — safe to send to LLM
        print('Final transcript: ' + event.transcript)
        print('Detected language: ' + event.language)
        session.stop()   // IMPORTANT: always stop to free the microphone
      }
    })

    session.onError.add((error) => {
      print('ASR error: ' + error)
      session.stop()  // also stop on error to free the microphone
    })
  }
}
```

- `isFinal = false` means a streaming partial result — the word list may still change.
- `isFinal = true` means the utterance is committed and ready for processing.
- Always call `session.stop()` when done — leaving ASR open drains battery and blocks the microphone.

---

## Text-to-Speech (TTS)

```typescript
const ttsModule = require('LensStudio:TtsModule')

function speak(text: string): void {
  const options = TtsTextToSpeechOptions.create()
  options.text = text

  ttsModule.speak(options, (audioComponent) => {
    audioComponent.play(1)   // play once
  })
}
```

### Cleanup on playback finish
```typescript
ttsModule.speak(options, (audioComponent) => {
  audioComponent.onPlaybackFinished.add(() => {
    print('TTS finished')
  })
  audioComponent.play(1)
})
```

- TTS audio competes with other audio sources — set appropriate `AudioMixerChannel`.
- Calling `speak()` again before the previous audio finishes will queue the new utterance.

---

## Camera Access for Vision

```typescript
const cameraModule = require('LensStudio:CameraModule')

const request = CameraModule.createCameraRequest()
request.cameraId = CameraModule.CameraId.Default_Color
const cameraTexture = cameraModule.requestCamera(request)

// Encode for transmission
const encodeOptions = EncodeTextureOptions.create()
Base64.encodeTextureAsync(cameraTexture, encodeOptions, (base64: string) => {
  callVisionAPI(base64)
})
```

### Depth Cache pattern
Cache multiple depth frames for richer 3D analysis:
- Capture frames on a timer using `DelayedCallbackEvent`
- Store frames in a ring buffer (`const buffer: string[] = []`)
- On user interaction, select the most relevant cached frame and send to a cloud vision model

---

## Agentic Loop Pattern

For autonomous multi-step AI interactions (as in the Agentic Playground sample):

1. User speaks → ASR transcription
2. Transcription sent to LLM via RSG
3. LLM returns a tool call (e.g., `{ "action": "show_object", "params": {} }`)
4. Lens executes the action in the AR scene
5. Lens sends a tool result back to the LLM for the next turn
6. Loop continues until LLM returns a final response

Key design notes:
- Store conversation history in a typed array and include it in each request body.
- Use a `thinking` or `loading` UI state while waiting for LLM responses.
- Implement a hard iteration cap (e.g., max 10 tool calls) to avoid infinite loops.

```typescript
interface Message { role: 'user' | 'assistant'; content: string }

const history: Message[] = []
const MAX_HISTORY = 20   // trim to avoid leaking earlier turns and hitting size limits
const MAX_TOOL_CALLS = 10 // hard cap to prevent runaway loops
let toolCallCount = 0

async function chat(userText: string): Promise<void> {
  if (toolCallCount++ >= MAX_TOOL_CALLS) {
    displayText('Reached iteration limit')
    return
  }

  history.push({ role: 'user', content: userText })
  // Keep only the last MAX_HISTORY messages
  if (history.length > MAX_HISTORY) history.splice(0, history.length - MAX_HISTORY)

  const request = RemoteServiceHttpRequest.create()
  request.endpoint = 'my-llm'
  request.method = RemoteServiceHttpRequest.HttpRequestMethod.Post
  request.body = JSON.stringify({ messages: history })

  remoteServiceModule.performHttpRequest(request, (response) => {
    const reply = JSON.parse(response.body).content as string
    history.push({ role: 'assistant', content: reply })
    displayText(reply)
  })
}
```

---

## AI Music Generation (Lyria / Gemini)

Pattern from the **AI Music Gen** sample:

```
ASR → Gemini text → Lyria audio → AudioComponent → 3D mesh driven by FFT
```

1. User describes genre/vibe via ASR
2. Description sent to Gemini via RSG for tag extraction
3. Tags forwarded to Lyria endpoint to generate an audio clip
4. Clip played through `AudioComponent`
5. 3D visualiser driven by `AudioSpectrum` FFT data

---

## Gesture + Vision: Crop Pattern

From the **Crop** sample — pinch to define a region, send to vision model:

1. Track hand pinch position with SIK `HandInputData`
2. Convert screen position to UV coordinates
3. Crop the camera texture at those UVs
4. Encode cropped region and send to RSG vision endpoint
5. Display result in a world-space panel near the gesture origin

---

## Permissions & Privacy

Combining camera, microphone, or location with **internet connectivity** triggers Snap's **Transparent Permission** system: the OS shows a consent dialog on launch and the device LED blinks during capture.

**Important exception:** Calls via the **Remote Service Gateway (RSG)** do not count as external connectivity. You can freely combine camera or microphone with RSG (LLMs, ASR, TTS, vision APIs) in a published lens without triggering the Transparent Permission prompt. This is the recommended pattern for AI-powered lenses.

---

## Common Gotchas

- **RSG is not available in the Lens Studio simulator** — test AI features on-device.
- **Large base64 payloads** can hit RSG body-size limits; resize or downsample images before encoding.
- **ASR leaves the microphone open** until you call `session.stop()` — always stop on both `isFinal` and `onError` to protect privacy and battery.
- **ASR accuracy modes**: `Balanced` is faster; `High` gives better results for commands with technical vocabulary.
- **TTS and game audio** share a mixer — prioritise with `AudioMixerChannel`.
- Use **`async/await`** to avoid callback pyramids in complex agentic loops.
- Always handle `response.statusCode !== 200` cases — network errors are common on Spectacles (the device moves around).
- **Agentic loops**: always enforce a hard iteration cap in code (not just in comments) and trim conversation history to avoid leaking earlier sensitive user input.
