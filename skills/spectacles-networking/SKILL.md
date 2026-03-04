---
name: spectacles-networking
description: Reference guide for the Lens Studio Fetch API and WebView component in Spectacles lenses — covering Fetch API setup (requires enabling Experimental APIs in Project Settings), GET/POST requests, JSON and binary response handling, custom request headers, Bearer auth, polling with DelayedCallbackEvent, timeout patterns with Promise.race, CORS/HTTPS requirements, multipart form data uploads, and bidirectional WebView messaging. Use this skill for any lens that calls a REST API, polls a JSON endpoint, loads remote images, embeds a webpage, or talks to a custom backend — including the Fetch sample. Use spectacles-ai for LLM/RSG calls, or spectacles-cloud for Supabase/Snap Cloud integration.
---

# Spectacles Networking — Reference Guide

Two main mechanisms for network access in Spectacles lenses:

1. **Fetch API** — raw HTTP requests (like the browser `fetch`), for calling any URL.
2. **Web View** — embed a web page in the AR scene, with bidirectional JS messaging.

The **Remote Service Gateway (RSG)** is a third option but is specific to AI/cloud-model calls — see the `spectacles-ai` skill for that.

---

## Setup

Before using the Fetch API, enable it in Lens Studio:

> *Project Settings → Spectacles → Experimental APIs → ✅ Fetch API*

Also enable **Internet Access**:

> *Project Settings → Capabilities → ✅ Internet Access*

Without these, all `fetch()` calls silently fail on-device.

---

## Fetch API

### Basic GET
```typescript
fetch('https://api.example.com/data')
  .then(response => response.json())
  .then(data => {
    print(JSON.stringify(data))
  })
  .catch(err => {
    print('Fetch error: ' + err)
  })
```

### POST with JSON body
```typescript
fetch('https://api.example.com/submit', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({ key: 'value' })
})
  .then(r => r.json())
  .then(data => { /* handle response */ })
```

### With authentication headers
```typescript
fetch('https://api.example.com/secure', {
  headers: {
    'Authorization': 'Bearer ' + myToken,
    'Content-Type': 'application/json'
  }
})
```

---

## Async / Await Pattern

```typescript
@component
export class FetchExample extends BaseScriptComponent {
  async onAwake() {
    try {
      const response = await fetch('https://api.example.com/items')
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

| Method | Returns | Use for |
|---|---|---|
| `response.json()` | `Promise<any>` | JSON APIs |
| `response.text()` | `Promise<string>` | Plain text or HTML |
| `response.arrayBuffer()` | `Promise<ArrayBuffer>` | Binary data (images, audio) |
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

- **Enable Fetch API** in *Project Settings → Spectacles → Experimental APIs* before testing on-device.
- **Enable Internet Access** in *Project Settings → Capabilities* or network calls silently fail.
- **CORS**: Web APIs must allow Spectacles' origin. If you control the backend, add `Access-Control-Allow-Origin: *`.
- **HTTPS only**: Snap does not allow plain HTTP endpoints on Spectacles.
- **No cookies or localStorage** in the Fetch context (unlike a browser).
- **Fetch does not time out automatically** — wrap calls with `Promise.race` if your backend might be slow.
- **Web View performance**: Large or animation-heavy web pages can hurt framerate. Prefer static or lightly interactive pages.
- **Multipart uploads**: do not set `Content-Type` manually when using `FormData` — the browser-like fetch API sets it automatically with the correct `boundary`.
