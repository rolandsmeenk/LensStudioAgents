---
name: spectacles-networking
description: Reference guide for the Lens Studio Fetch API and WebView component in Spectacles lenses — covering GET/POST requests, JSON and binary response handling, custom request headers, Bearer auth, polling with DelayedCallbackEvent, timeout patterns with Promise.race, CORS/HTTPS requirements, and bidirectional WebView messaging. Use this skill for any lens that calls a REST API, polls a JSON endpoint, loads remote images, embeds a webpage, or talks to a custom backend — including the Fetch sample. Use spectacles-ai for LLM/RSG calls, or spectacles-cloud for Supabase/Snap Cloud integration.
---

# Spectacles Networking — Reference Guide

Two main mechanisms for network access in Spectacles lenses:

1. **Fetch API** — raw HTTP requests (like the browser `fetch`), for calling any URL.
2. **Web View** — embed a web page in the AR scene, with bidirectional JS messaging.

The **Remote Service Gateway (RSG)** is a third option but is specific to AI/cloud-model calls — see the `spectacles-ai` skill for that.

---

## Fetch API

Lens Studio's Fetch API mirrors the browser `fetch` interface. It is an **experimental API** on Spectacles (check the compatibility list for your OS version).

### Basic GET
```typescript
fetch('https://api.example.com/data')
  .then(response => response.json())
  .then(data => {
    print(JSON.stringify(data));
  })
  .catch(err => {
    print('Fetch error: ' + err);
  });
```

### POST with JSON body
```typescript
fetch('https://api.example.com/submit', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({ key: 'value' })
})
  .then(r => r.json())
  .then(data => { /* handle response */ });
```

### With authentication headers
```typescript
fetch('https://api.example.com/secure', {
  headers: {
    'Authorization': 'Bearer ' + myToken,
    'Content-Type': 'application/json'
  }
});
```

---

## Async / Await Pattern

Lens Studio's scripting engine supports `async/await`. Use it to flatten callback chains:

```typescript
@component
export class FetchExample extends BaseScriptComponent {
  async onAwake() {
    try {
      const response = await fetch('https://api.example.com/items');
      const items: any[] = await response.json();
      this.displayItems(items);
    } catch (e) {
      print('Network error: ' + e);
    }
  }

  displayItems(items: any[]) {
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

Always check `response.ok` before parsing:
```typescript
if (!response.ok) {
  print('HTTP error: ' + response.status);
  return;
}
```

---

## Web View

Web View renders a full web page into a texture that can be applied to any mesh in the scene. Good for: dashboards, maps, rich content that's hard to build natively in Lens Studio.

### Setup in Lens Studio
1. Add a **Web View** component to a Scene Object.
2. Set the URL in the component inspector, or set it at runtime via script.
3. Apply the Web View texture to a **Screen Image** or a **Mesh Visual** material.

### Scripting the Web View
```typescript
// Set URL at runtime
webViewComponent.setUrl('https://myapp.example.com');

// Inject JavaScript into the page
webViewComponent.evaluateJavaScript("document.getElementById('status').textContent = 'AR connected'");

// Receive messages from the page (the page calls window.lensStudioMessage(data))
webViewComponent.onMessageReceived.add((message: string) => {
  const data = JSON.parse(message);
  // react to data from the web page
});
```

### Sending messages from web page to lens
In the web page's JavaScript:
```javascript
// This calls back into the lens
window.lensStudioMessage(JSON.stringify({ action: 'buttonClicked', id: 42 }));
```

### Layout tips
- Web View renders at a fixed resolution you specify on the component; scale the mesh to match the aspect ratio.
- Touch/pointer events from the user's hand pinch are forwarded to the Web View automatically via SIK integration.

---

## Common Patterns

### Poll for updates on a timer
```typescript
const updateEvent = this.createEvent('UpdateEvent');
let elapsed = 0;
const POLL_INTERVAL = 5; // seconds

updateEvent.bind(() => {
  elapsed += getDeltaTime();
  if (elapsed >= POLL_INTERVAL) {
    elapsed = 0;
    this.pollRemoteData();
  }
});
```

### Cache responses to avoid thrashing the network
```typescript
let cachedData: any = null;
let cacheTime = 0;
const CACHE_TTL = 30; // seconds

async function getWithCache(url: string) {
  const now = getTime();
  if (cachedData && (now - cacheTime) < CACHE_TTL) return cachedData;
  const r = await fetch(url);
  cachedData = await r.json();
  cacheTime = now;
  return cachedData;
}
```

---

## Experimental API Considerations

The Fetch API is marked **Experimental** on Spectacles. Key implications:
- It may not be available on older OS versions — check the Spectacles compatibility list.
- To enable: in Lens Studio, go to **Project Settings → Experimental APIs** and enable Fetch.
- Do not ship production lenses with unguarded Fetch calls without testing the target OS version.
- The Remote Service Gateway is the *stable, production* alternative for backend calls that can be proxied through Snap's infrastructure.

---

## Common Gotchas

- **CORS**: Web APIs must allow Spectacles' origin. If you control the backend, add `Access-Control-Allow-Origin: *` or the appropriate origin.
- **HTTPS only**: Snap does not allow plain HTTP endpoints on Spectacles.
- **No cookies or localStorage** in the Fetch context (unlike a browser).
- **Timeouts**: Fetch does not automatically time out — wrap calls with `Promise.race` and a timeout promise if your backend might be slow.
- **Web View performance**: Large or animation-heavy web pages can hurt framerate. Prefer static or lightly interactive pages.
