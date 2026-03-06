# RSG Configuration Reference

## Remote Service Gateway — project setup

In Lens Studio, open the **Project Settings → Remote Services** panel (or the **Remote Services** asset in the Asset panel). For each AI endpoint, add an entry:

| Field | Value |
|---|---|
| **Service ID** | A short string you use in script (e.g. `my-llm-service`) |
| **Base URL** | The upstream endpoint URL (your backend or Snap's managed gateway) |
| **Auth method** | Snap-managed auth (no key in script) or custom header |

> **Note:** The RSG is unavailable in the Lens Studio desktop simulator. Test AI features on a physical Spectacles device.

## Requesting streaming responses

```typescript
const remoteServiceModule = require('LensStudio:RemoteServiceModule')

function streamLLMResponse(prompt: string, onToken: (t: string) => void, onDone: () => void) {
  const req = RemoteServiceHttpRequest.create()
  req.endpoint = 'my-llm-service'
  req.method   = RemoteServiceHttpRequest.HttpRequestMethod.Post
  req.body     = JSON.stringify({ prompt, stream: true })

  // Streaming tokens
  req.onPartialResponse = (partial: string) => {
    onToken(partial)
  }

  remoteServiceModule.performHttpRequest(req, (response) => {
    if (response.statusCode === 200) {
      onDone()
    } else {
      print('RSG error ' + response.statusCode + ': ' + response.body)
    }
  })
}
```

## ASR session lifecycle (correct cleanup)

```typescript
const asrModule = require('LensStudio:AsrModule')
let asrSession: any = null

function startListening(): void {
  if (asrSession) { asrSession.stop(); asrSession = null }   // stop previous!
  asrSession = asrModule.startSession()
  asrSession.onTranscriptUpdate.add((event) => {
    if (event.isFinal) {
      const text = event.transcript
      asrSession.stop()                  // ← ALWAYS stop to save battery / mic
      asrSession = null
      handleTranscript(text)
    }
  })
}
```

## Base64 camera frame for vision API

```typescript
const cameraModule = require('LensStudio:CameraModule')

const req2 = CameraModule.createCameraRequest()
req2.cameraId = CameraModule.CameraId.Default_Color
const camTexture = cameraModule.requestCamera(req2)

// Encode current frame
const encOpts = EncodeTextureOptions.create()
encOpts.jpegQuality = 0.7          // reduce payload size

Base64.encodeTextureAsync(camTexture, encOpts, (base64String: string) => {
  sendToVisionAPI(base64String)
})
```

## Agentic loop — iteration cap

```typescript
let turnCount = 0
const MAX_TURNS = 10

async function agentStep(messages: any[]): Promise<void> {
  if (turnCount++ >= MAX_TURNS) {
    displayMessage('Max iterations reached.')
    return
  }
  const response = await callLLM(messages)
  if (response.tool_call) {
    const result = await executeTool(response.tool_call)
    messages.push({ role: 'tool', content: result })
    await agentStep(messages)           // recurse
  } else {
    displayMessage(response.content)    // terminal response
  }
}
```

