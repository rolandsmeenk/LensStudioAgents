---
name: spectacles-networking
description: Reference guide for the Lens Studio Fetch API and WebView component in Spectacles lenses — covering InternetModule (Lens Studio 5.9+), Fetch API via internetModule.fetch(Request) with bytes/text/json response handling, performHttpRequest, Internet Access capability, GET/POST requests, custom headers, Bearer auth, polling, timeouts, CORS/HTTPS, WebSocket and RemoteMediaModule for media from URLs, and bidirectional WebView messaging. Use this skill for any lens that calls a REST API, polls a JSON endpoint, loads remote images, embeds a webpage, or talks to a custom backend — including the Fetch sample. Use spectacles-ai for LLM/RSG calls, or spectacles-cloud for Supabase/Snap Cloud integration.
---

# Spectacles Networking — Reference Guide

Two main mechanisms for network access in Spectacles lenses:

1. **InternetModule** — HTTP/HTTPS via the **Fetch API** (recommended) or **performHttpRequest**. From Lens Studio 5.9, these APIs live in `InternetModule`; prior to 5.9 they were in `RemoteServiceModule`.
2. **Web View** — embed a web page in the AR scene, with bidirectional JS messaging.

The **Remote Service Gateway (RSG)** is a third option but is specific to AI/cloud-model calls — see the `spectacles-ai` skill for that.

**Official docs:** [Internet Access](https://developers.snap.com/spectacles/about-spectacles-features/apis/internet-access) | [Spectacles Home](https://developers.snap.com/spectacles/home)

---

## Setup

- Enable **Internet Access**: *Project Settings → Capabilities → ✅ Internet Access*. Without this, all requests fail on-device.
- **Fetch API**: Spectacles OS v5.58.6621+ and Lens Studio v5.3+. Use **InternetModule** (Lens Studio 5.9+): add the module to your project and call `internetModule.fetch(request)`.
- **HTTPS** is required for publishable lenses; **HTTP** requires [Experimental APIs](https://developers.snap.com/spectacles/permission-privacy/experimental-apis) and cannot be published.
- Fetch only works in Preview when Device Type Override is set to Spectacles.

---

## Fetch API (InternetModule)

Use `InternetModule` and construct a `Request`, then call `internetModule.fetch(request)`. Response bodies use `response.text()`, `response.json()`, or `response.bytes()` (not `body`, `blob`, or `arrayBuffer`).

### Basic GET
```typescript
const internetModule = require('LensStudio:InternetModule')

// In a component: get controller via internetModule.getModule() or add InternetModule as @input
this.createEvent('OnStartEvent').bind(async () => {
  const request = new Request('https://api.example.com/data', { method: 'GET' })
  const response = await internetModule.fetch(request)
  if (response.status === 200) {
    const data = await response.json()
    print(JSON.stringify(data))
  }
})
```

### POST with JSON body
```typescript
const request = new Request('https://api.example.com/submit', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({ key: 'value' })
})
const response = await this.internetModule.fetch(request)
const data = await response.json()
```

### With authentication headers
```typescript
const request = new Request('https://api.example.com/secure', {
  headers: {
    'Authorization': 'Bearer ' + myToken,
    'Content-Type': 'application/json'
  }
})
const response = await this.internetModule.fetch(request)
```

---

## Async / Await Pattern

```typescript
@component
export class FetchExample extends BaseScriptComponent {
  private internetModule = require('LensStudio:InternetModule')

  async onAwake() {
    try {
      const request = new Request('https://api.example.com/items')
      const response = await this.internetModule.fetch(request)
      if (!response.ok) {
        print('HTTP error: ' + response.status)
        return
      }
      const items: any[] = await response.json()
      this.displayItems(items)
    } catch (e) {
      print('Network error: ' + e)
    }
  }

  displayItems(items: any[]): void {
    // update scene objects
  }
}
```

---

## Handling Responses

On Spectacles, use `bytes()`, `text()`, and `json()` for response bodies (not `body`, `blob`, or `arrayBuffer`).

| Method | Returns | Use for |
|---|---|---|
| `response.json()` | `Promise<any>` | JSON APIs |
| `response.text()` | `Promise<string>` | Plain text or HTML |
| `response.bytes()` | Supported | Binary data (images, audio) |
| `response.status` | `number` | HTTP status code |
| `response.ok` | `boolean` | `true` if status 200–299 |
| `response.headers.get('Content-Type')` | `string \| null` | Read a response header |

Always check `response.ok` before parsing:
```typescript
if (!response.ok) {
  print('HTTP error: ' + response.status)
  return
}
```

---

## performHttpRequest and Remote Media

For a simpler callback-based API, use `internetModule.performHttpRequest(RemoteServiceHttpRequest.create(), callback)`. For downloading images, video, glTF, or audio from URLs, use `internetModule.makeResourceFromUrl(url)` and then `RemoteMediaModule.loadResourceAsImageTexture()` (or `loadResourceAsVideoTexture`, `loadResourceAsGltfAsset`, `loadResourceAsAudioTrackAsset`). See [Internet Access](https://developers.snap.com/spectacles/about-spectacles-features/apis/internet-access) for full examples. Use `global.deviceInfoSystem.isInternetAvailable()` and `onInternetStatusChanged` to react to connectivity changes.

---

## Multipart Form Data (File Upload)

```typescript
async function uploadFile(url: string, filename: string, blob: Blob): Promise<void> {
  const formData = new FormData()
  formData.append('file', blob, filename)

  const response = await fetch(url, {
    method: 'POST',
    headers: { 'Authorization': 'Bearer ' + myToken },
    body: formData
    // Do NOT set Content-Type manually — fetch sets it with the correct boundary
  })
  if (!response.ok) print('Upload failed: ' + response.status)
}
```

---

## Web View

Web View renders a full web page into a texture that can be applied to any mesh in the scene.

### Setup in Lens Studio
1. Add a **Web View** component to a Scene Object.
2. Set the URL in the component inspector, or set it at runtime via script.
3. Apply the Web View texture to a **Screen Image** or a **Mesh Visual** material.

### Scripting the Web View
```typescript
// Set URL at runtime
webViewComponent.setUrl('https://myapp.example.com')

// Inject JavaScript into the page
webViewComponent.evaluateJavaScript("document.getElementById('status').textContent = 'AR connected'")

// Receive messages from the page (the page calls window.lensStudioMessage(data))
webViewComponent.onMessageReceived.add((message: string) => {
  const data = JSON.parse(message)
  // react to data from the web page
})
```

### Sending messages from web page to lens
In the web page's JavaScript:
```javascript
// This calls back into the lens
window.lensStudioMessage(JSON.stringify({ action: 'buttonClicked', id: 42 }))
```

---

## Common Patterns

### Poll for updates on a timer
```typescript
let elapsed: number = 0
const POLL_INTERVAL = 5 // seconds

const updateEvent = this.createEvent('UpdateEvent')
updateEvent.bind(() => {
  elapsed += getDeltaTime()
  if (elapsed >= POLL_INTERVAL) {
    elapsed = 0
    this.pollRemoteData()
  }
})
```

### Cache responses to avoid thrashing the network
```typescript
let cachedData: any = null
let cacheTime: number = 0
const CACHE_TTL = 30 // seconds

async function getWithCache(url: string): Promise<any> {
  const now = getTime()
  if (cachedData && (now - cacheTime) < CACHE_TTL) return cachedData
  const r = await fetch(url)
  cachedData = await r.json()
  cacheTime = now
  return cachedData
}
```

### Request timeout with Promise.race
```typescript
function fetchWithTimeout(url: string, timeoutMs: number): Promise<Response> {
  const timeoutPromise = new Promise<never>((_, reject) =>
    setTimeout(() => reject(new Error('Request timed out')), timeoutMs)
  )
  return Promise.race([fetch(url), timeoutPromise])
}
```

---

## Common Gotchas

- **InternetModule (Lens Studio 5.9+)**: Use `require('LensStudio:InternetModule')` and `internetModule.fetch(request)`; global `fetch` may not be available. Enable **Internet Access** in *Project Settings → Capabilities* or network calls fail.
- **Request constructor**: Does not support taking another `Request` as input; use a URL string.
- **Response bodies**: Use `response.bytes()`, `response.text()`, `response.json()` — `body`, `blob`, `arrayBuffer` are not supported.
- **CORS**: Web APIs must allow Spectacles' origin. **HTTPS** required for publishable lenses; HTTP requires Experimental APIs.
- **Fetch does not time out automatically** — wrap with `Promise.race` if your backend might be slow.
- **Web View performance**: Large or animation-heavy web pages can hurt framerate. Prefer static or lightly interactive pages.
- **Multipart uploads**: do not set `Content-Type` manually when using `FormData` — fetch sets it with the correct `boundary`.
