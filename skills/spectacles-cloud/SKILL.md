---
name: spectacles-cloud
description: Reference guide for Snap Cloud (Supabase-powered backend) in Spectacles lenses — covering Postgres REST queries with the anon key, Row Level Security policies, Realtime WebSocket subscriptions with reconnect-on-sleep patterns, cloud storage uploads of base64 images captured by Spectacles, serverless Edge Functions, and companion web dashboard architecture. Use this skill whenever a lens needs persistent cloud data, needs to share data with a web app in real time, uploads captured images to a bucket, or calls a cloud function — covering Snap Cloud and World Kindness Day samples. Use spectacles-networking for plain REST calls to non-Snap backends, and spectacles-connected-lenses for in-session multiplayer state.
---

# Spectacles Cloud — Reference Guide

**Snap Cloud** is Snap's managed backend platform, powered by Supabase. It gives Spectacles lenses a full backend out of the box: relational database, real-time subscriptions, file storage, and serverless functions — all accessible from a lens via the standard Lens Studio `InternetModule` or the Fetch API.

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

Multiple users and a web companion app can all subscribe to the same database rows and see changes in real time.

---

## Setup

1. Go to [cloud.snap.com](https://cloud.snap.com) and create a project.
2. Note your **Project URL** and **anon/public API key**.
3. In your lens, store these as constants (or in a Config asset).
4. Enable **Internet Access** in Lens Studio: *Project Settings → Capabilities → Internet Access*.

---

## Database (Postgres via REST)

Snap Cloud exposes a standard Supabase PostgREST endpoint. All queries are plain HTTP:

### Read rows
```typescript
const PROJECT_URL = 'https://<project-ref>.supabase.co';
const API_KEY = 'your-anon-key';

async function getMessages(): Promise<any[]> {
  const r = await fetch(`${PROJECT_URL}/rest/v1/messages?order=created_at.desc&limit=20`, {
    headers: {
      'apikey': API_KEY,
      'Authorization': `Bearer ${API_KEY}`
    }
  });
  return r.json();
}
```

### Insert a row
```typescript
async function addMessage(content: string) {
  await fetch(`${PROJECT_URL}/rest/v1/messages`, {
    method: 'POST',
    headers: {
      'apikey': API_KEY,
      'Authorization': `Bearer ${API_KEY}`,
      'Content-Type': 'application/json',
      'Prefer': 'return=minimal'
    },
    body: JSON.stringify({ content, author: 'spectacles-user' })
  });
}
```

### Update / Delete
Use `PATCH` / `DELETE` with a query filter:
```typescript
// PATCH example – update a specific row
await fetch(`${PROJECT_URL}/rest/v1/messages?id=eq.${rowId}`, {
  method: 'PATCH',
  headers: { /* same headers */ },
  body: JSON.stringify({ content: 'updated text' })
});
```

---

## Realtime Subscriptions

Supabase Realtime uses WebSockets. Because Lens Studio's network stack supports WebSockets on Spectacles, you can subscribe to table changes directly.

Pattern (from the Snap Cloud and World Kindness Day samples):
1. Open a WebSocket to `wss://<project-ref>.supabase.co/realtime/v1/websocket?apikey=...`
2. Send a `phx_join` message for the channel you care about (e.g., `realtime:public:messages`).
3. Handle incoming `postgres_changes` events to update the scene.

```typescript
const ws = new WebSocket(
  `wss://${PROJECT_REF}.supabase.co/realtime/v1/websocket?apikey=${API_KEY}&vsn=1.0.0`
);

ws.onopen = () => {
  ws.send(JSON.stringify({
    topic: 'realtime:public:messages',
    event: 'phx_join',
    payload: {},
    ref: '1'
  }));
};

ws.onmessage = (event) => {
  const msg = JSON.parse(event.data);
  if (msg.event === 'postgres_changes') {
    const newRow = msg.payload.record;
    displayInScene(newRow);
  }
};
```

---

## Cloud Storage (Files / Images)

Upload and retrieve files (images, audio clips, 3D assets) stored in Supabase Storage buckets:

### Upload a file
```typescript
async function uploadImage(bucketName: string, path: string, base64Data: string) {
  const binary = Base64.decode(base64Data); // convert to ArrayBuffer
  await fetch(`${PROJECT_URL}/storage/v1/object/${bucketName}/${path}`, {
    method: 'POST',
    headers: {
      'apikey': API_KEY,
      'Authorization': `Bearer ${API_KEY}`,
      'Content-Type': 'image/jpeg'
    },
    body: binary
  });
}
```

### Get a public URL
```typescript
function getPublicUrl(bucketName: string, path: string): string {
  return `${PROJECT_URL}/storage/v1/object/public/${bucketName}/${path}`;
}
```

---

## Serverless Edge Functions

Edge Functions are Deno TypeScript functions deployed to Snap Cloud. Use them for:
- Server-side logic (e.g., picking a winner, sending push notifications)
- Aggregating data from multiple sources before returning to the lens
- Secrets management (keep API keys off the device)

Calling an edge function from a lens:
```typescript
async function callFunction(fnName: string, payload: object) {
  const r = await fetch(`${PROJECT_URL}/functions/v1/${fnName}`, {
    method: 'POST',
    headers: {
      'apikey': API_KEY,
      'Authorization': `Bearer ${API_KEY}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify(payload)
  });
  return r.json();
}
```

---

## Companion Web App Pattern

The **World Kindness Day** sample demonstrates a lens paired with a web app:
- Both talk to the same Supabase project via the same API key.
- The web app shows a live map of kind acts reported by Spectacles users.
- The lens submits new acts (with text + optional location) via a POST.
- The web app subscribes to Realtime and adds map pins without refreshing.

This pattern generalises to any lens + web dashboard pairing (leaderboards, moderation UIs, live analytics, etc.).

---

## Row-Level Security (RLS)

Supabase supports RLS policies to restrict who can read/write which rows. For a public-facing lens:
- Enable RLS on your tables.
- Write a policy like: `USING (true)` for public reads, `WITH CHECK (true)` for authenticated inserts.
- For user-specific data, pass a JWT from the lens (requires Snap's auth integration — see Snap Cloud docs).

Without RLS, any holder of the anon key can read and write all data — fine for prototypes, not for production.

---

## Common Gotchas

- **The anon key is not secret** — it's embedded in the lens. Use RLS policies to limit exposure.
- **WebSocket connections drop** when Spectacles goes to sleep — implement a reconnect loop with exponential back-off.
- **Large payloads** in Realtime subscriptions can slow the lens — only subscribe to columns you need, and filter server-side.
- **Supabase free tier** has connection and storage limits — upgrade the plan or add connection pooling for production.
- **CORS**: If building a companion web app, add your web app's domain to the Supabase CORS allow-list.
