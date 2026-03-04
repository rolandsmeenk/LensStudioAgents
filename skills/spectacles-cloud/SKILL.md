---
name: spectacles-cloud
description: Reference guide for Snap Cloud (Supabase-powered backend) in Spectacles lenses — covering Fetch API setup (requires Internet Access capability in Project Settings), Postgres REST queries with the anon key, Row Level Security policies, Realtime WebSocket subscriptions with correct postgres_changes event format and reconnect-on-sleep patterns, cloud storage uploads of base64 images captured by Spectacles, serverless Edge Functions, and companion web dashboard architecture. Use this skill whenever a lens needs persistent cloud data, needs to share data with a web app in real time, uploads captured images to a bucket, or calls a cloud function — covering Snap Cloud and World Kindness Day samples. Use spectacles-networking for plain REST calls to non-Snap backends, and spectacles-connected-lenses for in-session multiplayer state.
---

# Spectacles Cloud — Reference Guide

**Snap Cloud** is Snap's managed backend platform, powered by Supabase. It gives Spectacles lenses a full backend out of the box: relational database, real-time subscriptions, file storage, and serverless functions.

**Official docs:** [Spectacles Home](https://developers.snap.com/spectacles/home) · [Snap Cloud](https://developers.snap.com/spectacles/about-spectacles-features/snap-cloud) · [Internet Access](https://developers.snap.com/spectacles/about-spectacles-features/apis/internet-access) (required for fetch)

---

## Architecture Overview

```
Lens (Spectacles)
    │
    ├─── Snap Cloud REST API ──► Supabase Postgres DB
    │                         ├─ Realtime subscriptions
    │                         ├─ Storage buckets
    │                         └─ Edge Functions (serverless)
    │
Companion Web App ──────────► Same Supabase project
```

---

## Setup

1. Go to [cloud.snap.com](https://cloud.snap.com) and create a project.
2. Note your **Project URL** and **anon/public API key**.
3. Enable **Internet Access** in Lens Studio: *Project Settings → Capabilities → Internet Access*.  
   > Without this, all `fetch()` calls will silently fail on-device.
4. Store the URL and key as constants in your script.

---

## Database (Postgres via REST)

```typescript
const PROJECT_URL = 'https://<project-ref>.supabase.co'
const API_KEY = 'your-anon-key'

// Read rows
async function getMessages(): Promise<any[]> {
  const r = await fetch(`${PROJECT_URL}/rest/v1/messages?order=created_at.desc&limit=20`, {
    headers: {
      'apikey': API_KEY,
      'Authorization': `Bearer ${API_KEY}`
    }
  })
  if (!r.ok) { print('Read error: ' + r.status); return [] }
  return r.json()
}

// Insert a row
async function addMessage(content: string): Promise<void> {
  await fetch(`${PROJECT_URL}/rest/v1/messages`, {
    method: 'POST',
    headers: {
      'apikey': API_KEY,
      'Authorization': `Bearer ${API_KEY}`,
      'Content-Type': 'application/json',
      'Prefer': 'return=minimal'
    },
    body: JSON.stringify({ content, author: 'spectacles-user' })
  })
}

// Update a specific row
async function updateMessage(rowId: string, content: string): Promise<void> {
  await fetch(`${PROJECT_URL}/rest/v1/messages?id=eq.${rowId}`, {
    method: 'PATCH',
    headers: {
      'apikey': API_KEY,
      'Authorization': `Bearer ${API_KEY}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({ content })
  })
}
```

---

## Realtime Subscriptions

Supabase Realtime uses WebSockets. Lens Studio's network stack supports WebSockets on Spectacles.

```typescript
const ws = new WebSocket(
  `wss://${PROJECT_REF}.supabase.co/realtime/v1/websocket?apikey=${API_KEY}&vsn=1.0.0`
)

ws.onopen = () => {
  // Subscribe to a specific table using the correct postgres_changes format
  ws.send(JSON.stringify({
    topic: 'realtime:public:messages',
    event: 'phx_join',
    payload: {
      config: {
        postgres_changes: [
          { event: '*', schema: 'public', table: 'messages' }
        ]
      }
    },
    ref: '1'
  }))
}

ws.onmessage = (event) => {
  const msg = JSON.parse(event.data as string)
  if (msg.event === 'postgres_changes') {
    const newRow = msg.payload.data.record
    displayInScene(newRow)
  }
}

ws.onerror = (error) => print('WebSocket error: ' + JSON.stringify(error))
```

### Reconnect on Spectacles sleep

Spectacles can disconnect WebSockets when the device sleeps. Add a reconnect loop:

```typescript
let ws: WebSocket | null = null

function connect(): void {
  ws = new WebSocket(`wss://${PROJECT_REF}.supabase.co/realtime/v1/websocket?apikey=${API_KEY}&vsn=1.0.0`)
  ws.onopen = () => subscribeToTable(ws!)
  ws.onclose = () => scheduleReconnect()
  ws.onerror = () => scheduleReconnect()
}

function scheduleReconnect(): void {
  const retry = this.createEvent('DelayedCallbackEvent')
  retry.bind(() => connect())
  retry.reset(3) // retry after 3 seconds
}
```

---

## Cloud Storage (Files / Images)

```typescript
async function uploadImage(bucketName: string, path: string, base64Data: string): Promise<void> {
  const binary = Base64.decode(base64Data) // convert to ArrayBuffer
  await fetch(`${PROJECT_URL}/storage/v1/object/${bucketName}/${path}`, {
    method: 'POST',
    headers: {
      'apikey': API_KEY,
      'Authorization': `Bearer ${API_KEY}`,
      'Content-Type': 'image/jpeg'
    },
    body: binary
  })
}

function getPublicUrl(bucketName: string, path: string): string {
  return `${PROJECT_URL}/storage/v1/object/public/${bucketName}/${path}`
}
```

---

## Serverless Edge Functions

Edge Functions are Deno TypeScript functions deployed to Snap Cloud. Use them for server-side logic, aggregations, or keeping API keys off the device.

```typescript
async function callFunction(fnName: string, payload: object): Promise<any> {
  const r = await fetch(`${PROJECT_URL}/functions/v1/${fnName}`, {
    method: 'POST',
    headers: {
      'apikey': API_KEY,
      'Authorization': `Bearer ${API_KEY}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify(payload)
  })
  return r.json()
}
```

---

## Row-Level Security (RLS)

Supabase supports RLS policies to restrict who can read/write which rows:
- Enable RLS on your tables.
- Write a policy: `USING (true)` for public reads, `WITH CHECK (true)` for authenticated inserts.
- For user-specific data, pass a JWT from the lens (requires Snap's auth integration).

Without RLS, any holder of the anon key can read and write all data — fine for prototypes, not for production.

---

## Common Gotchas

- **Enable Internet Access capability** first — *Project Settings → Capabilities → Internet Access*. Without it, fetch silently fails on-device.
- **The anon key is not secret** — it's embedded in the lens. Use RLS policies to limit exposure.
- **WebSocket `postgres_changes` format**: the payload must include `{ config: { postgres_changes: [...] } }` in the `phx_join` message — a simpler topic-only join will not receive row change events.
- **WebSocket connections drop** when Spectacles goes to sleep — implement a reconnect loop with `DelayedCallbackEvent`.
- **Large payloads** in Realtime subscriptions can slow the lens — subscribe only to the columns you need.
- **Supabase free tier** has connection and storage limits — upgrade the plan or add connection pooling for production.
- **CORS**: If building a companion web app, add your web app's domain to the Supabase CORS allow-list.
