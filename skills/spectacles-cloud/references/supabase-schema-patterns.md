# Snap Cloud (Supabase) — Schema Patterns Reference

## Suggested table schema for common Spectacles use-cases

### Messages / annotations (World Kindness Day pattern)
```sql
create table public.acts (
  id          uuid primary key default gen_random_uuid(),
  content     text not null,
  author_id   text,
  latitude    float8,
  longitude   float8,
  image_path  text,             -- Storage bucket path
  created_at  timestamptz default now()
);
-- RLS: allow public read, allow any insert
alter table public.acts enable row level security;
create policy "public read"  on public.acts for select using (true);
create policy "public write" on public.acts for insert with check (true);
```

### Lens Cloud-style leaderboard (use instead of LeaderboardModule for more control)
```sql
create table public.scores (
  id          serial primary key,
  username    text not null,
  score       int  not null,
  lens_id     text not null,
  created_at  timestamptz default now()
);
create index on public.scores (lens_id, score desc);
```

## Realtime WebSocket reconnect on Spectacles sleep

```typescript
// Spectacles may suspend the WebSocket when the device sleeps/idles.
// Reconnect pattern from Snap Cloud sample:

const WS_URL = `wss://${PROJECT_REF}.supabase.co/realtime/v1/websocket?apikey=${API_KEY}&vsn=1.0.0`
let ws: WebSocket
let reconnectTimer: any = null

function connect(): void {
  ws = new WebSocket(WS_URL)

  ws.onopen = () => {
    print('Realtime connected')
    subscribeToTable()
    if (reconnectTimer) { reconnectTimer.enabled = false }
  }

  ws.onmessage = (ev) => {
    const msg = JSON.parse(ev.data)
    if (msg.event === 'postgres_changes') handleChange(msg.payload.record)
    // Supabase requires heartbeat reply:
    if (msg.event === 'heartbeat') {
      ws.send(JSON.stringify({ topic: 'phoenix', event: 'heartbeat', payload: {}, ref: '0' }))
    }
  }

  ws.onclose = (ev) => {
    print('Realtime closed: ' + ev.code + ' — retrying in 5s')
    scheduleReconnect()
  }

  ws.onerror = () => scheduleReconnect()
}

function scheduleReconnect(): void {
  if (reconnectTimer) return
  const ev = script.createEvent('DelayedCallbackEvent')
  reconnectTimer = ev
  ev.bind(() => { reconnectTimer = null; connect() })
  ev.reset(5)
}
```

## Image upload to Storage bucket

```typescript
async function uploadCameraFrame(
  bucketName: string,
  path: string,
  base64: string
): Promise<string | null> {
  const url = `${PROJECT_URL}/storage/v1/object/${bucketName}/${path}`
  const binary = Base64.decode(base64)

  const res = await fetch(url, {
    method: 'POST',
    headers: {
      'apikey': API_KEY,
      'Authorization': `Bearer ${API_KEY}`,
      'Content-Type': 'image/jpeg',
      'x-upsert': 'true'   // overwrite if same path exists
    },
    body: binary
  })

  if (!res.ok) {
    print('Upload failed: ' + res.status)
    return null
  }

  return `${PROJECT_URL}/storage/v1/object/public/${bucketName}/${path}`
}
```

