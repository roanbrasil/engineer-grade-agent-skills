---
name: websockets-realtime
description: Design and implement real-time communication — choose between WebSocket, SSE, and long polling; scale with Redis pub/sub; handle presence, auth, and horizontal scaling for chat, live feeds, collaborative editing, and streaming AI output.
---

# Real-Time Communication Patterns

## Choosing the Right Transport

```
  Is communication bidirectional?
       |
      YES --> WebSocket (or WebTransport for HTTP/3)
       |
      NO  --> Is it server → client only?
               |
              YES --> Is browser support / proxy compatibility important?
                       |
                      YES --> SSE (Server-Sent Events)
                       |
                      NO  --> WebSocket (simpler unified protocol)
               |
              NO  --> Is it client → server only?
                       YES --> Regular HTTP POST; no streaming needed
```

| Technique       | Direction          | Protocol  | Latency | Use Case                                      |
|-----------------|--------------------|-----------|---------|-----------------------------------------------|
| Long Polling    | server -> client   | HTTP      | High    | Legacy fallback; simple; no WS support        |
| SSE             | server -> client   | HTTP/1.1+ | Low     | Live feeds, notifications, streaming LLM text |
| WebSocket       | bidirectional      | WS/WSS    | Low     | Chat, collaborative editing, games, trading   |
| WebTransport    | bidirectional      | HTTP/3    | Lowest  | Low-latency gaming, real-time video, modern   |

---

## WebSocket Deep Dive

### Handshake and Lifecycle

```
  Client                              Server
    |                                   |
    |--- HTTP GET /ws ----------------> |
    |    Upgrade: websocket             |
    |    Connection: Upgrade            |
    |    Sec-WebSocket-Key: <base64>    |
    |    Sec-WebSocket-Version: 13      |
    |                                   |
    |<-- 101 Switching Protocols ------ |
    |    Upgrade: websocket             |
    |    Sec-WebSocket-Accept: <hash>   |
    |                                   |
    |<======= WS frames (both ways) ==> |
    |                                   |
    |--- Close frame (code 1000) -----> |
    |<-- Close frame ------------------- |
    |--- TCP FIN --------------------->  |
```

### Message Types and Framing

```
  WebSocket Frame:
  +-+-+-+-+-------+-+-------------+-------------------------------+
  |F|R|R|R| opcode|M| Payload len |    Extended payload length    |
  |I|S|S|S|  (4)  |A|    (7)      |          (16/64 bits)        |
  |N|V|V|V|       |S|             |                               |
  | |1|2|3|       |K|             |                               |
  +-+-+-+-+-------+-+-------------+-------------------------------+

  Opcodes:
    0x0  Continuation frame
    0x1  Text frame (UTF-8)
    0x2  Binary frame (ArrayBuffer)
    0x8  Close
    0x9  Ping
    0xA  Pong
```

- Text frames: JSON, plain text — human-readable, slightly larger
- Binary frames: MessagePack, Protobuf — compact, ideal for high-frequency data
- Ping/Pong: keepalive; server sends Ping, client replies Pong; detect dead connections

---

## WebSocket Server Patterns

### FastAPI (Python)

```python
# main.py
from fastapi import FastAPI, WebSocket, WebSocketDisconnect
from typing import Dict, Set
import json

app = FastAPI()

class ConnectionManager:
    def __init__(self):
        # room_id -> set of WebSocket connections
        self.rooms: Dict[str, Set[WebSocket]] = {}

    async def connect(self, websocket: WebSocket, room_id: str):
        await websocket.accept()
        if room_id not in self.rooms:
            self.rooms[room_id] = set()
        self.rooms[room_id].add(websocket)

    def disconnect(self, websocket: WebSocket, room_id: str):
        self.rooms.get(room_id, set()).discard(websocket)
        if room_id in self.rooms and not self.rooms[room_id]:
            del self.rooms[room_id]

    async def broadcast(self, room_id: str, message: dict, exclude: WebSocket = None):
        dead = set()
        for ws in self.rooms.get(room_id, set()):
            if ws is exclude:
                continue
            try:
                await ws.send_json(message)
            except Exception:
                dead.add(ws)
        for ws in dead:
            self.rooms[room_id].discard(ws)

manager = ConnectionManager()

@app.websocket("/ws/{room_id}")
async def websocket_endpoint(websocket: WebSocket, room_id: str):
    # 1. Authenticate before accepting
    token = websocket.headers.get("Authorization") or \
            websocket.query_params.get("token")
    user = await verify_token(token)  # raise if invalid
    if not user:
        await websocket.close(code=4001, reason="Unauthorized")
        return

    await manager.connect(websocket, room_id)
    try:
        while True:
            data = await websocket.receive_json()
            # Validate message type
            if data.get("type") not in ("message", "typing", "ping"):
                continue
            await manager.broadcast(
                room_id,
                {"type": data["type"], "user": user.id, "payload": data.get("payload")},
                exclude=websocket
            )
    except WebSocketDisconnect:
        manager.disconnect(websocket, room_id)
        await manager.broadcast(room_id, {"type": "presence", "user": user.id, "status": "offline"})
```

### Spring Boot (Java)

```java
// WebSocketConfig.java
@Configuration
@EnableWebSocketMessageBroker
public class WebSocketConfig implements WebSocketMessageBrokerConfigurer {

    @Override
    public void configureMessageBroker(MessageBrokerRegistry registry) {
        // Enable in-memory broker for /topic and /queue destinations
        registry.enableSimpleBroker("/topic", "/queue");
        // Prefix for @MessageMapping methods
        registry.setApplicationDestinationPrefixes("/app");
    }

    @Override
    public void registerStompEndpoints(StompEndpointRegistry registry) {
        registry.addEndpoint("/ws")
                .setAllowedOriginPatterns("https://*.example.com")
                .withSockJS(); // fallback for older browsers
    }
}

// ChatController.java
@Controller
public class ChatController {

    private final SimpMessagingTemplate messagingTemplate;

    public ChatController(SimpMessagingTemplate messagingTemplate) {
        this.messagingTemplate = messagingTemplate;
    }

    @MessageMapping("/chat/{roomId}")
    public void handleMessage(@DestinationVariable String roomId,
                              @Payload ChatMessage message,
                              Principal principal) {
        message.setSender(principal.getName());
        message.setTimestamp(Instant.now());
        // Broadcast to all subscribers of this topic
        messagingTemplate.convertAndSend("/topic/room/" + roomId, message);
    }

    @MessageMapping("/chat/private")
    public void sendPrivate(@Payload PrivateMessage message, Principal principal) {
        // Send to specific user's queue
        messagingTemplate.convertAndSendToUser(
            message.getRecipient(), "/queue/messages", message
        );
    }
}
```

### TypeScript / Node.js (Socket.IO)

```typescript
// server.ts
import { Server, Socket } from 'socket.io';
import { createAdapter } from '@socket.io/redis-adapter';
import { createClient } from 'redis';

const io = new Server(httpServer, {
  cors: { origin: process.env.ALLOWED_ORIGINS?.split(',') },
  pingTimeout: 10_000,
  pingInterval: 30_000,
});

// Redis adapter for horizontal scaling
const pubClient = createClient({ url: process.env.REDIS_URL });
const subClient = pubClient.duplicate();
await Promise.all([pubClient.connect(), subClient.connect()]);
io.adapter(createAdapter(pubClient, subClient));

io.use(async (socket, next) => {
  // Auth middleware: token in handshake auth, not URL
  const token = socket.handshake.auth.token;
  try {
    socket.data.user = await verifyJwt(token);
    next();
  } catch {
    next(new Error('Unauthorized'));
  }
});

io.on('connection', (socket: Socket) => {
  const { user } = socket.data;

  socket.on('join-room', async (roomId: string) => {
    await socket.join(roomId);
    socket.to(roomId).emit('user-joined', { userId: user.id });
    // Track presence in Redis
    await redis.sadd(`room:${roomId}:members`, user.id);
  });

  socket.on('message', (data: { roomId: string; text: string }) => {
    // Rate limiting check here
    io.to(data.roomId).emit('message', {
      id: crypto.randomUUID(),
      userId: user.id,
      text: data.text,
      ts: Date.now(),
    });
  });

  socket.on('disconnect', async () => {
    for (const roomId of socket.rooms) {
      await redis.srem(`room:${roomId}:members`, user.id);
      socket.to(roomId).emit('user-left', { userId: user.id });
    }
  });
});
```

---

## SSE — Server-Sent Events

### Protocol Format

```
  HTTP/1.1 200 OK
  Content-Type: text/event-stream
  Cache-Control: no-cache
  Connection: keep-alive
  X-Accel-Buffering: no        <-- disable nginx buffering

  data: {"type":"update","value":42}\n\n

  id: 1234\n
  event: price-update\n
  data: {"symbol":"AAPL","price":182.50}\n\n

  : keep-alive comment\n\n      <-- prevents proxy timeout

  retry: 5000\n\n               <-- tell client to reconnect after 5s
```

Key rules:
- Each field on its own line; event ends with blank line (`\n\n`)
- `id:` field: browser sends as `Last-Event-ID` header on reconnect
- `event:` field: named event type (default: `message`)
- `: comment` lines for keepalive pings (ignored by browser)

### FastAPI SSE

```python
# sse_endpoint.py
from fastapi import FastAPI, Request
from fastapi.responses import StreamingResponse
import asyncio
import json

app = FastAPI()

async def event_generator(request: Request, channel: str):
    """Yields SSE-formatted strings."""
    event_id = 0
    try:
        async with redis_pubsub(channel) as pubsub:
            async for message in pubsub.listen():
                if await request.is_disconnected():
                    break
                if message["type"] == "message":
                    event_id += 1
                    data = json.dumps(json.loads(message["data"]))
                    yield f"id: {event_id}\ndata: {data}\n\n"
                else:
                    # Keepalive comment every 15s
                    yield ": keepalive\n\n"
    except asyncio.CancelledError:
        pass  # client disconnected

@app.get("/events/{channel}")
async def stream_events(request: Request, channel: str):
    return StreamingResponse(
        event_generator(request, channel),
        media_type="text/event-stream",
        headers={
            "Cache-Control": "no-cache",
            "X-Accel-Buffering": "no",    # nginx
            "X-Content-Type-Options": "nosniff",
        }
    )

# Streaming LLM output (ChatGPT-style)
@app.post("/chat/stream")
async def stream_llm(request: Request, body: ChatRequest):
    async def generate():
        async for chunk in llm_client.stream(body.prompt):
            yield f"data: {json.dumps({'text': chunk.delta})}\n\n"
        yield "data: [DONE]\n\n"

    return StreamingResponse(generate(), media_type="text/event-stream")
```

### Spring Boot SSE

```java
// SseController.java
@RestController
public class SseController {

    private final Map<String, SseEmitter> emitters = new ConcurrentHashMap<>();

    @GetMapping(value = "/events/{userId}", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
    public SseEmitter subscribe(@PathVariable String userId) {
        SseEmitter emitter = new SseEmitter(Long.MAX_VALUE);

        emitter.onCompletion(() -> emitters.remove(userId));
        emitter.onTimeout(() -> emitters.remove(userId));
        emitter.onError(e -> emitters.remove(userId));

        emitters.put(userId, emitter);

        // Send initial event
        try {
            emitter.send(SseEmitter.event()
                .name("connected")
                .data(Map.of("userId", userId)));
        } catch (IOException e) {
            emitters.remove(userId);
        }

        return emitter;
    }

    // Called from your business logic when an event occurs
    public void sendToUser(String userId, String eventType, Object data) {
        SseEmitter emitter = emitters.get(userId);
        if (emitter == null) return;
        try {
            emitter.send(SseEmitter.event()
                .id(String.valueOf(System.currentTimeMillis()))
                .name(eventType)
                .data(data));
        } catch (IOException e) {
            emitters.remove(userId);
            emitter.completeWithError(e);
        }
    }
}
```

### Browser SSE Client

```typescript
// client.ts
const es = new EventSource('/events/channel-id', { withCredentials: true });

es.addEventListener('price-update', (e: MessageEvent) => {
  const { symbol, price } = JSON.parse(e.data);
  updateChart(symbol, price);
});

es.addEventListener('error', () => {
  // Browser reconnects automatically — do not manually reconnect here
  // Browser sends Last-Event-ID header so server can resume from last event
  console.warn('SSE connection lost, browser will retry...');
});

// Cleanup
// es.close();
```

### SSE vs WebSocket

| Concern              | SSE                              | WebSocket                        |
|----------------------|----------------------------------|----------------------------------|
| Direction            | Server -> Client only            | Bidirectional                    |
| Protocol             | HTTP (works through all proxies) | Upgrade (some proxies block)     |
| Auto-reconnect       | Built-in (browser handles it)    | Must implement manually          |
| Binary support       | No (text only)                   | Yes (binary frames)              |
| HTTP/2 multiplexing  | Yes (many streams, 1 connection) | No (1 connection per stream)     |
| Load balancer compat | Stateless; any server can serve  | Sticky sessions required         |

---

## Presence and State Sync

### Presence with Redis

```python
# presence.py
import asyncio
from redis.asyncio import Redis

PRESENCE_TTL = 35  # seconds; heartbeat every 30s + 5s buffer

class PresenceManager:
    def __init__(self, redis: Redis):
        self.redis = redis

    async def user_online(self, user_id: str, room_id: str):
        pipe = self.redis.pipeline()
        pipe.sadd(f"room:{room_id}:online", user_id)
        pipe.setex(f"presence:{user_id}", PRESENCE_TTL, "1")
        await pipe.execute()
        await self.publish_presence(room_id, user_id, "online")

    async def user_offline(self, user_id: str, room_id: str):
        pipe = self.redis.pipeline()
        pipe.srem(f"room:{room_id}:online", user_id)
        pipe.delete(f"presence:{user_id}")
        await pipe.execute()
        await self.publish_presence(room_id, user_id, "offline")

    async def heartbeat(self, user_id: str):
        """Call every 30s to keep presence alive."""
        await self.redis.setex(f"presence:{user_id}", PRESENCE_TTL, "1")

    async def get_online_users(self, room_id: str) -> set[str]:
        return await self.redis.smembers(f"room:{room_id}:online")

    async def publish_presence(self, room_id: str, user_id: str, status: str):
        await self.redis.publish(
            f"presence:{room_id}",
            json.dumps({"user": user_id, "status": status})
        )
```

### Collaborative Editing — CRDTs with Yjs

```typescript
// server.ts (y-websocket provider)
import { WebSocketServer } from 'ws';
import { setupWSConnection } from 'y-websocket/bin/utils';

const wss = new WebSocketServer({ port: 1234 });

wss.on('connection', (ws, req) => {
  // y-websocket handles Yjs CRDT sync automatically
  // No conflict resolution logic needed — CRDTs merge automatically
  setupWSConnection(ws, req, { docName: getDocName(req) });
});

// client.ts
import * as Y from 'yjs';
import { WebsocketProvider } from 'y-websocket';
import { QuillBinding } from 'y-quill';

const doc = new Y.Doc();
const provider = new WebsocketProvider('wss://collab.example.com', 'doc-id', doc);
const text = doc.getText('content');

// Bind to Quill editor — conflicts resolved automatically by Yjs
const binding = new QuillBinding(text, quillEditor, provider.awareness);

// Awareness: cursor positions, user colors (ephemeral state)
provider.awareness.setLocalState({
  user: { name: 'Alice', color: '#ff6b6b' },
  cursor: { anchor: 0, head: 10 }
});
```

### Optimistic UI

```typescript
// Optimistic update pattern
async function sendMessage(roomId: string, text: string) {
  const tempId = `temp-${Date.now()}`;

  // 1. Apply locally immediately (optimistic)
  addMessage({ id: tempId, text, status: 'sending' });

  try {
    // 2. Confirm with server
    const confirmed = await api.postMessage(roomId, text);
    // 3. Replace temp message with confirmed
    replaceMessage(tempId, { ...confirmed, status: 'sent' });
  } catch (err) {
    // 4. Rollback on failure
    removeMessage(tempId);
    showError('Message failed to send');
  }
}
```

---

## Scaling WebSockets Horizontally

### The Sticky Session Problem

```
  WITHOUT Redis (broken):

  Client A ---> Server 1  (connection lives here)
  Client B ---> Server 2  (connection lives here)
  Server 1 broadcasts "hello" --> only Client A sees it

  WITH Redis pub/sub (correct):

  Client A ---> Server 1 ---subscribe---> Redis channel "room:123"
  Client B ---> Server 2 ---subscribe---> Redis channel "room:123"

  Any server publishes to "room:123" in Redis
       |
       +--> Server 1 receives --> sends to Client A
       +--> Server 2 receives --> sends to Client B
```

### Redis Pub/Sub Pattern (Python)

```python
# redis_pubsub.py
import asyncio
from redis.asyncio import Redis

redis = Redis.from_url(os.environ["REDIS_URL"])

# Publisher (any server, any request handler)
async def publish_to_room(room_id: str, event: dict):
    await redis.publish(f"room:{room_id}", json.dumps(event))

# Subscriber (one per server process, shared across all WS connections)
class RoomBroadcaster:
    def __init__(self, connection_manager: ConnectionManager):
        self.cm = connection_manager
        self.pubsub = redis.pubsub()

    async def subscribe_room(self, room_id: str):
        await self.pubsub.subscribe(f"room:{room_id}")

    async def listen(self):
        async for message in self.pubsub.listen():
            if message["type"] != "message":
                continue
            channel = message["channel"].decode()
            room_id = channel.split(":")[-1]
            data = json.loads(message["data"])
            # Broadcast to all local WS connections for this room
            await self.cm.broadcast(room_id, data)
```

### Nginx Configuration for WebSocket

```nginx
upstream websocket_backend {
    ip_hash;  # sticky sessions: same client IP -> same server
    server ws1.internal:8000;
    server ws2.internal:8000;
    server ws3.internal:8000;
    keepalive 64;
}

server {
    listen 443 ssl http2;
    server_name ws.example.com;

    location /ws/ {
        proxy_pass http://websocket_backend;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;

        # Don't timeout idle WebSocket connections
        proxy_read_timeout 3600s;
        proxy_send_timeout 3600s;

        # Disable buffering for SSE endpoints
        # proxy_buffering off;
    }
}
```

### Capacity Planning

```
Linux defaults:
  /proc/sys/fs/file-max          = 1,048,576 (system-wide)
  ulimit -n                      = 1,024 (per process — TOO LOW)

Raise per-process limit:
  # /etc/security/limits.conf
  *  soft  nofile  100000
  *  hard  nofile  100000

  # or systemd service
  [Service]
  LimitNOFILE=100000

Each WebSocket connection uses:
  ~1 file descriptor
  ~4-8 KB kernel memory (TCP buffers)
  ~your app's per-connection data

1 server with 16 GB RAM can handle ~100K-500K idle connections
(depends on message rate and application logic)
```

### Heartbeat Implementation

```python
# FastAPI heartbeat
@app.websocket("/ws/{client_id}")
async def websocket_endpoint(websocket: WebSocket, client_id: str):
    await websocket.accept()

    async def heartbeat():
        while True:
            await asyncio.sleep(30)
            try:
                await websocket.send_json({"type": "ping"})
            except Exception:
                return  # connection dead; outer loop will catch

    heartbeat_task = asyncio.create_task(heartbeat())
    try:
        while True:
            try:
                data = await asyncio.wait_for(
                    websocket.receive_json(),
                    timeout=40  # 30s ping + 10s grace
                )
            except asyncio.TimeoutError:
                break  # no pong received; close connection
            if data.get("type") == "pong":
                continue
            await handle_message(websocket, data)
    except WebSocketDisconnect:
        pass
    finally:
        heartbeat_task.cancel()
        cleanup(client_id)
```

---

## Security Hardening

### Authentication: Token in First Message (Not URL)

```typescript
// WRONG: token in URL is logged by every proxy/CDN/server
const ws = new WebSocket('wss://api.example.com/ws?token=SECRET');

// CORRECT option 1: token in first message after connect
const ws = new WebSocket('wss://api.example.com/ws');
ws.onopen = () => {
  ws.send(JSON.stringify({ type: 'auth', token: getAuthToken() }));
};

// CORRECT option 2: token in Sec-WebSocket-Protocol header (hack but works)
const ws = new WebSocket('wss://api.example.com/ws', ['v1', `token.${btoa(token)}`]);

// Socket.IO: use handshake auth
const socket = io('wss://api.example.com', {
  auth: { token: getAuthToken() }
});
```

### Server-Side Auth Enforcement

```python
@app.websocket("/ws")
async def websocket_endpoint(websocket: WebSocket):
    # 1. Check Origin header before accepting
    origin = websocket.headers.get("origin", "")
    allowed = {"https://app.example.com", "https://admin.example.com"}
    if origin not in allowed:
        await websocket.close(code=4003, reason="Forbidden origin")
        return

    await websocket.accept()

    # 2. Require auth message within 5 seconds
    try:
        auth_msg = await asyncio.wait_for(websocket.receive_json(), timeout=5.0)
    except asyncio.TimeoutError:
        await websocket.close(code=4008, reason="Auth timeout")
        return

    if auth_msg.get("type") != "auth":
        await websocket.close(code=4001, reason="Expected auth message")
        return

    user = await verify_token(auth_msg.get("token", ""))
    if not user:
        await websocket.close(code=4001, reason="Invalid token")
        return

    # 3. Now handle messages with rate limiting
    rate_limiter = RateLimiter(max_calls=10, period=1.0)  # 10 msg/s
    async for message in websocket.iter_json():
        if not await rate_limiter.check(user.id):
            await websocket.send_json({"error": "rate_limited"})
            continue
        await process_message(websocket, user, message)
```

### Rate Limiting

```python
# Token bucket rate limiter using Redis
class RateLimiter:
    def __init__(self, redis: Redis, max_calls: int, period: float):
        self.redis = redis
        self.max_calls = max_calls
        self.period = period

    async def check(self, key: str) -> bool:
        pipe = self.redis.pipeline()
        now = time.time()
        window_key = f"ratelimit:{key}:{int(now / self.period)}"
        pipe.incr(window_key)
        pipe.expire(window_key, int(self.period) + 1)
        count, _ = await pipe.execute()
        return count <= self.max_calls
```

---

## Long Polling (Legacy Fallback)

```python
# FastAPI long poll
import asyncio

pending: Dict[str, asyncio.Event] = {}
messages: Dict[str, list] = {}

@app.get("/poll/{client_id}")
async def long_poll(client_id: str, last_id: int = 0):
    event = pending.setdefault(client_id, asyncio.Event())

    try:
        # Wait up to 30s for a message
        await asyncio.wait_for(event.wait(), timeout=30.0)
        event.clear()
        new_messages = [m for m in messages.get(client_id, []) if m["id"] > last_id]
        return {"messages": new_messages}
    except asyncio.TimeoutError:
        return {"messages": []}  # client re-polls immediately

def push_to_client(client_id: str, message: dict):
    messages.setdefault(client_id, []).append(message)
    if client_id in pending:
        pending[client_id].set()
```

---

## Anti-Patterns

### 1. Storing Active Connections in a Database

```python
# WRONG: DB is slow; connections are ephemeral; crashes leave stale rows
async def connect(websocket, user_id):
    await db.execute("INSERT INTO connections (user_id, server) VALUES (?, ?)", ...)

# CORRECT: in-memory (single server) or Redis (multi-server)
connections: Dict[str, WebSocket] = {}

async def connect(websocket, user_id):
    connections[user_id] = websocket
    await redis.setex(f"conn:{user_id}:server", 60, SERVER_ID)
```

### 2. Broadcasting to All Connections (Not Rooms)

```python
# WRONG: O(n) broadcast to every connected client
async def broadcast(message):
    for ws in all_connections.values():
        await ws.send_json(message)

# CORRECT: broadcast only to room members
async def broadcast_to_room(room_id: str, message: dict):
    for ws in rooms[room_id]:
        await ws.send_json(message)
```

### 3. No Reconnection Logic on Client

```typescript
// WRONG: dead after first disconnect
const ws = new WebSocket('wss://api.example.com/ws');
ws.onclose = () => console.log('disconnected'); // lost forever

// CORRECT: exponential backoff reconnection
function connect() {
  let retryDelay = 1000;
  let ws: WebSocket;

  function tryConnect() {
    ws = new WebSocket('wss://api.example.com/ws');
    ws.onopen = () => { retryDelay = 1000; }; // reset on success
    ws.onclose = (e) => {
      if (e.code !== 1000) { // not a normal close
        setTimeout(tryConnect, Math.min(retryDelay, 30_000));
        retryDelay *= 2;
      }
    };
  }

  tryConnect();
  return () => ws?.close(1000); // cleanup function
}
```

### 4. Blocking Nginx Buffering for SSE

```nginx
# WRONG: nginx buffers SSE events; client sees them in batches
location /events/ {
    proxy_pass http://backend;
}

# CORRECT: disable buffering for SSE
location /events/ {
    proxy_pass http://backend;
    proxy_buffering off;
    proxy_cache off;
    proxy_set_header X-Accel-Buffering no;
}
```

### 5. Not Validating WebSocket Message Content

```python
# WRONG: trusting message structure
async def handle_message(websocket, data):
    await db.insert(data["table"], data["record"])  # SQL injection / unauthorized table

# CORRECT: validate every message
from pydantic import BaseModel, validator

class ChatMessage(BaseModel):
    type: Literal["message", "typing"]
    room_id: str
    text: str

    @validator("text")
    def text_length(cls, v):
        if len(v) > 2000:
            raise ValueError("Message too long")
        return v

async def handle_message(websocket, raw_data):
    try:
        msg = ChatMessage.model_validate(raw_data)
    except ValidationError:
        await websocket.send_json({"error": "invalid_message"})
        return
    await process_chat_message(websocket, msg)
```

---

## Production Checklist

**WebSocket**
- [ ] WSS only — never unencrypted `ws://` in production
- [ ] Authenticate before accepting OR within 5s of connect (never trust based on URL token)
- [ ] Validate `Origin` header on upgrade; reject unknown origins
- [ ] Validate every incoming message with schema (Pydantic, Zod, Bean Validation)
- [ ] Rate limit messages per connection (token bucket in Redis)
- [ ] Heartbeat: server pings every 30s; close if no pong within 10s
- [ ] Graceful close: send Close frame with code 1000; wait for Close reply
- [ ] Reconnection with exponential backoff on client
- [ ] Redis adapter or pub/sub for multi-server deployments
- [ ] Sticky sessions configured in load balancer (`ip_hash` in nginx)
- [ ] `ulimit -n` raised to ≥ 100,000 on WebSocket servers
- [ ] Separate WebSocket servers from HTTP API servers (different scaling profile)

**SSE**
- [ ] `Content-Type: text/event-stream` with `Cache-Control: no-cache`
- [ ] `X-Accel-Buffering: no` header to prevent nginx/CDN buffering
- [ ] Keepalive comment (`: ping\n\n`) every 15s to prevent proxy timeouts
- [ ] `id:` field on every event so browser can resume after reconnect
- [ ] `Last-Event-ID` handled on server to replay missed events
- [ ] Test through all proxies in your stack (nginx, CDN, API gateway)

**General**
- [ ] Monitor open connection count (alert if growing unboundedly — leak indicator)
- [ ] Instrument message latency (p50/p95/p99)
- [ ] Connection errors logged with close codes for debugging
- [ ] Load test at 2x expected peak connections before launch
