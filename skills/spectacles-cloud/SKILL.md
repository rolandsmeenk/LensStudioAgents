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

> **⚠️ Security — anon key is not secret:** The anon key is compiled into the lens and can be extracted by anyone who decompiles it. Anyone who obtains the key can read and write data while RLS is disabled. Always enable RLS (see below). For production lenses, proxy all database access through a Snap Cloud **Edge Function** so the key never leaves the server.

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
  if (!retryCount) retryCount = 0
  if (retryCount++ >= 5) { print('[Cloud] WebSocket gave up after 5 retries'); return }
  const retry = this.createEvent('DelayedCallbackEvent')
  retry.bind(() => connect())
  retry.reset(3) // retry after 3 seconds
}
let retryCount = 0
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

### Recommended production pattern: Edge Function as auth proxy

For any lens you share publicly, route database access through an Edge Function instead of calling the REST API directly from the lens:

```
Lens  →  Edge Function (holds service-role key)  →  Supabase DB
```

The Edge Function can validate inputs, enforce business rules, and return only the data the lens needs — the raw DB key never leaves the server. The lens only needs the anon key to call the Edge Function, and with RLS that key has no direct table access.

### Lens-side call (TypeScript)

```typescript
// The lens calls the Edge Function — it never touches the DB directly
async function addMessageSecure(content: string): Promise<void> {
  const r = await fetch(`${PROJECT_URL}/functions/v1/add-message`, {
    method: 'POST',
    headers: {
      'apikey': API_KEY,         // anon key only — Edge Function validates and writes
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({ content })
  })
  if (!r.ok) print('Error: ' + r.status)
}
```

### Edge Function (Deno TypeScript, runs on Snap Cloud)

```typescript
// supabase/functions/add-message/index.ts
import { createClient } from 'https://esm.sh/@supabase/supabase-js'

Deno.serve(async (req) => {
  const { content } = await req.json()

  // Validate inputs server-side before writing
  if (!content || typeof content !== 'string' || content.length > 500) {
    return new Response('Invalid input', { status: 400 })
  }

  // Use the service-role key here — it never leaves the server
  const supabase = createClient(
    Deno.env.get('SUPABASE_URL')!,
    Deno.env.get('SUPABASE_SERVICE_ROLE_KEY')!  // NOT the anon key
  )

  const { error } = await supabase
    .from('messages')
    .insert({ content, author: 'spectacles-user' })

  if (error) return new Response(error.message, { status: 500 })
  return new Response('OK', { status: 200 })
})
```

---

## Row-Level Security (RLS)

Supabase supports RLS policies to restrict who can read/write which rows. **Always enable RLS on every table before sharing a lens.**

```sql
-- Minimal public read (ok for leaderboards / shared content)
CREATE POLICY "public_read" ON messages
  FOR SELECT USING (true);

-- User-scoped write (requires Snap auth JWT)
CREATE POLICY "owner_insert" ON messages
  FOR INSERT WITH CHECK (auth.uid() = user_id);

-- Avoid USING (true) on INSERT/UPDATE/DELETE — this lets any key holder modify all data
```

> **⚠️ `USING (true)` on writes is not protection.** It means any holder of the anon key can mutate every row. Use it only for genuinely public read-only data, and never on write operations in a shared lens.

For user-specific data, pass a JWT from the lens (requires Snap's auth integration), or use the Edge Function proxy pattern.

---

## Permissions & Privacy

Combining internet access with camera, microphone, or location triggers Snap's **Transparent Permission** system: when the lens launches, the OS shows the user a dialog listing which sensitive data the lens accesses, and the device LED blinks while capture is active. Plan your UX around this prompt — it will appear before the lens starts.

---

## Common Gotchas

- **Enable Internet Access capability** first — *Project Settings → Capabilities → Internet Access*. Without it, fetch silently fails on-device.
- **The anon key is embedded in the lens.** Anyone can extract it from a published lens. Always enable RLS and consider the Edge Function proxy pattern for production.
- **The Supabase Realtime WebSocket URL requires the anon key** (`?apikey=...`) — this is a Supabase protocol requirement, not something you can avoid. Mitigate exposure with RLS: even if someone extracts the key, RLS limits what they can read or write.
- **WebSocket `postgres_changes` format**: the payload must include `{ config: { postgres_changes: [...] } }` in the `phx_join` message — a simpler topic-only join will not receive row change events.
- **WebSocket connections drop** when Spectacles goes to sleep — implement a reconnect loop with `DelayedCallbackEvent`.
- **Large payloads** in Realtime subscriptions can slow the lens — subscribe only to the columns you need.
- **Supabase free tier** has connection and storage limits — upgrade the plan or add connection pooling for production.
- **CORS**: If building a companion web app, add your web app's domain to the Supabase CORS allow-list.
