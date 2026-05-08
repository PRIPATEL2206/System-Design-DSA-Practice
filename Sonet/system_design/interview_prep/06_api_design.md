# API Design & Communication

## API Paradigms Comparison

| Feature | REST | GraphQL | gRPC |
|---------|------|---------|------|
| Protocol | HTTP/1.1 | HTTP/1.1 | HTTP/2 |
| Data format | JSON | JSON | Protobuf (binary) |
| Typing | No schema | Schema (SDL) | Strongly typed (.proto) |
| Over-fetching | Yes | No | No |
| Learning curve | Low | Medium | High |
| Caching | Easy (HTTP cache) | Hard | Hard |
| Best for | Public APIs, web | Complex queries, mobile | Internal microservices |

---

## 1. REST (Representational State Transfer)

### Core Principles
```
1. Stateless     — Server stores no client state between requests
2. Uniform interface — Resources accessed via standard HTTP methods
3. Client-Server — Separation of concerns
4. Cacheable     — Responses can be cached
```

### HTTP Methods
| Method | Action | Idempotent | Body |
|--------|--------|-----------|------|
| GET | Read resource | Yes | No |
| POST | Create resource | No | Yes |
| PUT | Replace entire resource | Yes | Yes |
| PATCH | Partial update | No (usually) | Yes |
| DELETE | Delete resource | Yes | No |

### RESTful URL Design
```
Good:
  GET    /users/{id}           — Get user
  GET    /users/{id}/posts     — Get user's posts
  POST   /users                — Create user
  PUT    /users/{id}           — Update user
  DELETE /users/{id}           — Delete user
  GET    /posts?page=2&limit=20 — Paginate posts

Bad:
  GET  /getUser?id=123         — Verb in URL
  POST /deleteUser             — Wrong method for delete
  POST /users/create           — Redundant "create"
```

### Pagination
```
1. Offset-based:  GET /posts?offset=20&limit=10
   Pro: Random access, simple
   Con: Inconsistent if items inserted during pagination (items skipped/duplicated)

2. Cursor-based:  GET /posts?cursor=abc123&limit=10
   Pro: Consistent even with new inserts; better for infinite scroll
   Con: Can't jump to arbitrary page
   
3. Page-based:    GET /posts?page=3&limit=10
   Pro: Simple for users
   Con: Same issues as offset
```

**Interview tip:** Use cursor-based pagination for feeds (Twitter, Instagram). Offset for admin dashboards.

### API Versioning
```
URL versioning:    /api/v1/users  (most common, easy to route)
Header versioning: Accept: application/vnd.myapp.v1+json
Query param:       /users?api_version=1
```

---

## 2. GraphQL

**What it solves:** Client over-fetching and under-fetching. Client specifies exactly what it needs.

```graphql
# REST requires 3 calls: /users/1, /users/1/posts, /users/1/friends
# GraphQL does it in one:

query {
  user(id: 1) {
    name
    email
    posts(last: 5) {
      title
    }
    friends(first: 10) {
      name
    }
  }
}
```

**Best for:** Mobile apps (bandwidth-sensitive), complex data graphs, when frontend teams are changing requirements rapidly.

**Watch out for:** N+1 query problem (use DataLoader), complex caching, large queries can overload server.

---

## 3. gRPC

**What it is:** Google's high-performance RPC framework using Protocol Buffers (binary serialisation) over HTTP/2.

```protobuf
// Define the service in .proto file
service UserService {
  rpc GetUser (UserRequest) returns (UserResponse);
  rpc StreamUsers (Empty) returns (stream UserResponse);
}

message UserRequest { int64 user_id = 1; }
message UserResponse { int64 id = 1; string name = 2; string email = 3; }
```

### gRPC vs REST
| Aspect | REST | gRPC |
|--------|------|------|
| Speed | Slower (JSON parsing) | 5–10x faster (binary, HTTP/2) |
| Browser support | Native | Needs proxy (grpc-web) |
| Code generation | Manual | Auto-generated from .proto |
| Streaming | Hard (SSE, WebSocket) | Native bidirectional streaming |

**Best for:** Internal microservice communication where performance matters.

---

## 4. Real-time Communication

### WebSocket
```
Client ──── upgrade to WebSocket ────> Server
       <───────── bidirectional ──────>

Use: Chat, live feeds, multiplayer games, collaborative editing
Pro: Full duplex, low overhead after handshake
Con: Stateful (harder to load balance), no HTTP caching
```

### Server-Sent Events (SSE)
```
Client ──── GET /events ────> Server
       <─── stream of events ──

Use: Notifications, live scores, stock prices (one-way from server)
Pro: Works over HTTP, easy to implement, auto-reconnect
Con: One-directional (server to client only)
```

### Long Polling
```
Client sends request → server holds it open until new data arrives → respond
Client immediately sends another request (loop)

Use: Fallback when WebSocket/SSE not available
Con: Higher overhead, many open connections
```

### Which to Use?
| Scenario | Best Option |
|----------|------------|
| Chat app (bidirectional) | WebSocket |
| Live notification feed | SSE |
| Real-time dashboard | SSE |
| Multiplayer game | WebSocket |
| Legacy support | Long Polling fallback |

---

## 5. Rate Limiting

**Why:** Protect services from abuse, ensure fair usage, prevent DDoS.

### Algorithms

#### Token Bucket (most common)
```
- Bucket holds up to N tokens
- Tokens added at rate R per second
- Each request consumes 1 token
- If no tokens → reject request

Pro: Allows burst traffic (use up accumulated tokens)
Use: API rate limiting (N requests per minute, but allow short bursts)
```

#### Fixed Window Counter
```
- Count requests in current 1-minute window
- If count > limit → reject
- Reset counter at start of each window

Con: "Edge" problem — 100 req at 11:59 + 100 req at 12:00 = 200 req in 2 seconds
```

#### Sliding Window Log
```
- Store timestamp of each request
- Count requests in last 60 seconds
- More accurate, but more memory (store all timestamps)
```

#### Sliding Window Counter
```
- Blend of fixed window + rolling window (approximation)
- Memory efficient, accurate enough for most cases
```

### Implementation with Redis
```
Rate limit 100 req/min per user:

1. INCR user:123:ratelimit
2. If result == 1, SET EXPIRE to 60 seconds
3. If result > 100, reject with 429 Too Many Requests
4. Else allow
```

### Where to Rate Limit
- API Gateway (most common)
- Load Balancer
- Application layer (Redis counter)

---

## API Design Best Practices

```
Security:
- Always use HTTPS
- Authenticate every request (JWT, OAuth, API key)
- Validate input (prevent injection)
- Use CORS properly

Error Handling:
- Return meaningful HTTP status codes
  200 OK, 201 Created, 204 No Content
  400 Bad Request, 401 Unauthorized, 403 Forbidden
  404 Not Found, 409 Conflict, 422 Unprocessable Entity
  429 Too Many Requests, 500 Internal Server Error
- Return error body: { "error": "message", "code": "ERR_001" }

Performance:
- Support pagination (never return unbounded lists)
- Use compression (gzip)
- Cache GET responses (Cache-Control headers)
- Implement ETag for conditional requests
```

---

## Quick Reference

| Concept | One-line |
|---------|---------|
| REST | HTTP methods on resource URLs; stateless; easy to cache |
| GraphQL | Client specifies exactly what it needs; no over-fetching |
| gRPC | Binary protocol; fast; best for internal microservices |
| WebSocket | Full-duplex; use for chat, games, live updates |
| SSE | Server-to-client stream; use for notifications, feeds |
| Token bucket | Rate limiting; allows bursts; most common algorithm |
| Idempotency | Same request → same result; critical for retries |
| Cursor pagination | Consistent results even with concurrent inserts |
