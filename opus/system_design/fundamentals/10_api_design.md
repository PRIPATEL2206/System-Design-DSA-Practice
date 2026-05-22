# Chapter 10: API Design

## REST API Design

### URL Structure
```
GET    /users              → List all users
GET    /users/123          → Get user 123
POST   /users              → Create a user
PUT    /users/123          → Replace user 123
PATCH  /users/123          → Partially update user 123
DELETE /users/123          → Delete user 123

Nested resources:
GET    /users/123/orders          → List orders for user 123
GET    /users/123/orders/456      → Get specific order
POST   /users/123/orders          → Create order for user 123
```

### Best Practices
```
1. Use nouns, not verbs:    ✅ /users   ❌ /getUsers
2. Use plural:              ✅ /users   ❌ /user
3. Use kebab-case:          ✅ /user-profiles  ❌ /userProfiles
4. Version your API:        /api/v1/users
5. Filter with query params: /users?status=active&role=admin
6. Pagination:              /users?page=2&limit=20
7. Sorting:                 /users?sort=-created_at (- for desc)
```

### Pagination Strategies
```
Offset-based (simple):
  GET /posts?page=3&limit=20
  ⚠️ Problem: Slow for large offsets (DB skips N rows)

Cursor-based (scalable):
  GET /posts?cursor=eyJpZCI6MTIzfQ&limit=20
  ✅ Consistent with inserts/deletes
  ✅ O(1) regardless of page number
  Used by: Twitter, Facebook, Slack APIs

Keyset (seek):
  GET /posts?after_id=123&limit=20
  WHERE id > 123 ORDER BY id LIMIT 20
  ✅ Fast (uses index), no skipping
```

### Response Format
```json
{
  "data": {
    "id": "123",
    "name": "Prince",
    "email": "prince@example.com"
  },
  "meta": {
    "request_id": "req_abc123",
    "timestamp": "2024-01-15T10:30:00Z"
  }
}

// List response with pagination
{
  "data": [...],
  "pagination": {
    "total": 150,
    "page": 2,
    "limit": 20,
    "next_cursor": "eyJpZCI6MTQwfQ"
  }
}

// Error response
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Email is required",
    "details": [
      {"field": "email", "message": "must not be blank"}
    ]
  }
}
```

---

## Rate Limiting

### Algorithms
```
1. Token Bucket
   ─────────────
   Bucket holds N tokens. Each request takes 1 token.
   Tokens refill at rate R per second.
   Empty bucket → 429 Too Many Requests.
   ✅ Allows bursts (bucket can be full)

2. Leaky Bucket
   ─────────────
   Requests queue up. Process at fixed rate.
   Queue full → reject.
   ✅ Smooth output rate (no bursts)

3. Fixed Window
   ─────────────
   Count requests per time window (e.g., 100/minute).
   Reset counter at window boundary.
   ⚠️ Burst at window edges (200 requests in 2 seconds)

4. Sliding Window Log
   ──────────────────
   Keep timestamp of each request. Count in last N seconds.
   ✅ Accurate. ⚠️ Memory-heavy.

5. Sliding Window Counter
   ──────────────────────
   Weighted: current_window × weight + previous_window × (1-weight)
   ✅ Low memory, good accuracy.
```

### Rate Limit Headers
```
X-RateLimit-Limit: 100        (max requests per window)
X-RateLimit-Remaining: 45     (remaining in current window)
X-RateLimit-Reset: 1640000000 (when window resets, Unix timestamp)
Retry-After: 30               (seconds to wait, on 429)
```

---

## Authentication & Authorization

```
Authentication (AuthN): WHO are you?
Authorization (AuthZ):  WHAT can you do?

┌─────────────────────────────────────────────────────────────────┐
│ Method          │ Best For           │ Notes                    │
├─────────────────┼────────────────────┼──────────────────────────┤
│ API Key         │ Server-to-server   │ Simple, no user context  │
│ JWT Token       │ Stateless auth     │ Self-contained, expires  │
│ OAuth 2.0       │ Third-party access │ "Login with Google"      │
│ Session Cookie  │ Web apps           │ Server-side state        │
└─────────────────┴────────────────────┴──────────────────────────┘
```

### JWT Flow
```
1. Client → POST /login {email, password}
2. Server validates → returns JWT
3. Client sends JWT in header: Authorization: Bearer <token>
4. Server validates JWT signature (no DB lookup needed!)
5. JWT expires → client refreshes with refresh token

JWT Structure: header.payload.signature (base64 encoded)
```

### OAuth 2.0 Flow (Authorization Code)
```
1. User clicks "Login with Google"
2. Redirect to Google: /authorize?client_id=X&redirect_uri=Y&scope=email
3. User consents on Google
4. Google redirects back with authorization code
5. Your server exchanges code for access token (server-to-server)
6. Use access token to get user info from Google
```

---

## API Gateway Pattern

```
┌─────────┐         ┌─────────────────┐
│ Mobile  │────────▶│                 │──▶ User Service
│ Client  │         │                 │──▶ Order Service
└─────────┘         │   API Gateway   │──▶ Payment Service
┌─────────┐         │                 │──▶ Notification Service
│ Web     │────────▶│  (single entry) │
│ Client  │         └─────────────────┘
└─────────┘

Gateway handles:
- Routing (URL → service mapping)
- Authentication/Authorization
- Rate limiting
- Request/response transformation
- Caching
- Monitoring/Logging
- Circuit breaking
```

---

## GraphQL vs REST

```
┌──────────────────┬─────────────────────┬─────────────────────┐
│                  │ REST                │ GraphQL             │
├──────────────────┼─────────────────────┼─────────────────────┤
│ Data fetching    │ Multiple endpoints  │ Single endpoint     │
│ Over-fetching    │ Common              │ Client picks fields │
│ Under-fetching   │ Multiple round trips│ Nested queries      │
│ Versioning       │ /v1/ /v2/          │ Schema evolution    │
│ Caching          │ HTTP caching easy   │ More complex        │
│ Learning curve   │ Low                 │ Medium              │
│ Best for         │ CRUD, public APIs   │ Complex UIs, mobile │
└──────────────────┴─────────────────────┴─────────────────────┘
```

---

## Idempotency

An operation is **idempotent** if calling it multiple times produces the same result.

```
Idempotent:     GET, PUT, DELETE
Not idempotent: POST

Why it matters: Network failures → client retries → duplicate requests!

Solution: Idempotency key
1. Client generates unique key (UUID) per logical request
2. Sends: X-Idempotency-Key: <uuid>
3. Server checks: "Have I processed this key before?"
   - Yes → return cached result (don't re-execute)
   - No → process, store result keyed by idempotency key

Critical for: Payment APIs, order creation, any non-idempotent mutation
```
