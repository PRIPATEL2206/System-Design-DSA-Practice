# Chapter 3: Networking & Protocols

## The Network Stack (Simplified)

```
┌─────────────────────────────────┐
│  Application Layer (HTTP, WS)   │  ← You design here
├─────────────────────────────────┤
│  Transport Layer (TCP, UDP)     │  ← Reliable vs fast
├─────────────────────────────────┤
│  Network Layer (IP)             │  ← Routing
├─────────────────────────────────┤
│  Link Layer (Ethernet, WiFi)    │  ← Physical
└─────────────────────────────────┘
```

---

## HTTP/HTTPS

The foundation of web communication.

### HTTP Methods
```
GET     → Read resource (idempotent, cacheable)
POST    → Create resource (not idempotent)
PUT     → Replace resource (idempotent)
PATCH   → Partial update
DELETE  → Remove resource (idempotent)
```

### HTTP Status Codes (Know These!)
```
2xx Success    200 OK, 201 Created, 204 No Content
3xx Redirect   301 Moved Permanently, 304 Not Modified
4xx Client Err 400 Bad Request, 401 Unauthorized, 403 Forbidden, 404 Not Found, 429 Too Many Requests
5xx Server Err 500 Internal Error, 502 Bad Gateway, 503 Service Unavailable
```

### HTTP/2 vs HTTP/1.1
```
HTTP/1.1: One request per connection (head-of-line blocking)
HTTP/2:   Multiplexed streams, header compression, server push
HTTP/3:   Based on QUIC (UDP), even faster handshake
```

---

## TCP vs UDP

```
                    TCP                     UDP
                    ─────────               ─────────
Connection         Connection-oriented     Connectionless
Reliability        Guaranteed delivery     Best effort
Ordering           In-order               No ordering
Speed              Slower (handshake)      Faster
Use cases          HTTP, file transfer     Video streaming, gaming, DNS
```

**When to use what in system design:**
- Chat messages → TCP (reliable delivery matters)
- Live video → UDP (speed > reliability)
- VoIP → UDP (latency-sensitive)
- File upload → TCP (can't lose data)

---

## WebSockets

Persistent, bidirectional communication over a single TCP connection.

```
Traditional HTTP:
Client ──request──▶ Server
Client ◀──response── Server
(connection closed)

WebSocket:
Client ══════════════ Server
       ←─ messages ─→
(connection stays open)
```

**Use WebSockets when:**
- Real-time updates (chat, notifications, live feeds)
- Bidirectional communication needed
- Server needs to push data to client

**Scaling challenges:**
- Each connection uses server memory (~10KB)
- 1M connections ≈ 10GB RAM just for connections
- Need sticky sessions or connection state management

---

## Long Polling vs WebSocket vs SSE

```
┌──────────────────┬────────────────┬──────────────────┬───────────────┐
│ Feature          │ Long Polling   │ WebSocket        │ SSE           │
├──────────────────┼────────────────┼──────────────────┼───────────────┤
│ Direction        │ Client→Server  │ Bidirectional    │ Server→Client │
│ Connection       │ Repeated HTTP  │ Persistent TCP   │ Persistent HTTP│
│ Overhead         │ High (headers) │ Low              │ Low           │
│ Use case         │ Fallback       │ Chat, gaming     │ Notifications │
│ Browser support  │ Universal      │ Modern browsers  │ Modern (no IE)│
└──────────────────┴────────────────┴──────────────────┴───────────────┘
```

---

## DNS (Domain Name System)

```
User types "google.com"
    │
    ▼
[Browser Cache] → found? → use it
    │ not found
    ▼
[OS Cache / hosts file] → found? → use it
    │ not found
    ▼
[Recursive Resolver (ISP)] 
    │
    ▼ asks
[Root DNS] → "Ask .com TLD server"
    │
    ▼ asks
[.com TLD] → "Ask Google's authoritative server"
    │
    ▼ asks
[Authoritative DNS] → "142.250.80.46"
    │
    ▼
Response cached at each level (TTL)
```

**DNS in System Design:**
- DNS-based load balancing (Round Robin, Geo-routing)
- Low TTL for quick failover
- DNS is often the first bottleneck users hit

---

## CDN (Content Delivery Network)

```
Without CDN:
User (India) ────── 200ms ──────▶ Origin (US)

With CDN:
User (India) ── 20ms ──▶ CDN Edge (Mumbai) ── cache miss? ──▶ Origin (US)
```

### CDN Strategies
```
Pull CDN:  First request goes to origin, CDN caches response (lazy)
Push CDN:  You upload content to CDN proactively (eager)

Pull: Good for dynamic/frequently changing content
Push: Good for large static files (videos, images)
```

### When to use CDN:
- Static assets (images, CSS, JS, videos)
- Global user base (reduce latency)
- High read-to-write ratio content
- Reduce load on origin servers

---

## gRPC

Google's high-performance RPC framework using Protocol Buffers.

```
REST API:              gRPC:
─────────              ─────
JSON (text)            Protobuf (binary) → smaller payload
HTTP/1.1              HTTP/2 → multiplexing
Request/Response       Streaming (unary, server, client, bidirectional)
```

**Use gRPC for:**
- Microservice-to-microservice communication (internal)
- When latency/bandwidth matters
- When you need streaming
- When you have strict schema requirements

**Use REST for:**
- Public-facing APIs
- Browser clients (limited gRPC support)
- Simplicity and debugging ease

---

## Key Concepts for Interviews

### Connection Pooling
Reuse existing connections instead of creating new ones for every request. Critical for database connections (expensive to create).

### Keep-Alive
HTTP connections stay open for multiple requests (default in HTTP/1.1).

### Rate Limiting at Network Level
- Token bucket / leaky bucket algorithms
- Applied at load balancer or API gateway
- Return 429 Too Many Requests

### Circuit Breaker Pattern
```
CLOSED → (failures exceed threshold) → OPEN
OPEN → (timeout expires) → HALF-OPEN
HALF-OPEN → (success) → CLOSED
HALF-OPEN → (failure) → OPEN
```
Prevents cascading failures in microservices.
