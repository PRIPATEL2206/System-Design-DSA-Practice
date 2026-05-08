# Building Blocks of System Design

## The Standard Request Flow

```
User/Client
     |
    [DNS]               ← Resolves domain to IP
     |
    [CDN]               ← Serves cached static assets nearby
     |
[Load Balancer]         ← Distributes traffic across servers
     |
[Reverse Proxy]         ← Optional: SSL termination, auth, routing
     |
[API Gateway]           ← Rate limiting, auth, request routing
     |
[App Servers]           ← Business logic (stateless, scalable)
   / | \
  /  |  \
[Cache] [DB] [Message Queue]
```

---

## 1. DNS (Domain Name System)

**What it does:** Converts `www.example.com` → `93.184.216.34`

```
Browser → Recursive Resolver → Root NS → TLD NS (.com) → Authoritative NS → IP
```

### Types of DNS Records
| Record | Purpose |
|--------|---------|
| A | Domain → IPv4 address |
| AAAA | Domain → IPv6 address |
| CNAME | Alias → another domain |
| MX | Mail exchange server |
| TXT | Verification, SPF, DKIM |

### DNS for Load Balancing
- **Round-robin DNS:** Return multiple IPs, rotate requests
- **Geo DNS:** Return nearest server IP based on user location
- **TTL control:** Lower TTL = faster failover, more DNS queries

**Interview tip:** DNS is the first layer of load balancing/routing. Mention geo-routing for global systems.

---

## 2. CDN (Content Delivery Network)

**What it does:** Caches static content (images, JS, CSS, videos) at edge servers close to users, reducing latency and origin server load.

```
User in India ──> CDN Edge (Mumbai) ──hit──> serve cached asset
                                    ──miss──> Origin Server → cache → serve
User in US    ──> CDN Edge (Virginia) (separate cache)
```

### Push vs Pull CDN
| Type | How it works | Use when |
|------|-------------|----------|
| **Pull** | CDN fetches from origin on first miss, caches it | Most common. Easy setup. Content updated often. |
| **Push** | You upload files to CDN proactively | Large files, rarely updated (videos, binaries) |

### What to Cache on CDN
- Static assets: images, CSS, JS, fonts
- Video/audio files
- API responses that are public and infrequently changing

### What NOT to Cache on CDN
- Personalised content (user-specific data)
- Real-time data (prices, stock tickers)
- Write requests

**Interview tip:** Mention CDN for any system with global users or heavy media content (YouTube, Instagram, etc.).

---

## 3. Load Balancer

**What it does:** Distributes incoming traffic across multiple servers so no single server is overwhelmed.

```
         [Load Balancer]
        /       |       \
[Server 1] [Server 2] [Server 3]
```

### Load Balancing Algorithms

| Algorithm | How it works | When to use |
|-----------|-------------|-------------|
| **Round Robin** | Requests go 1→2→3→1→2→3 | Servers are equal capacity |
| **Weighted Round Robin** | Server 1 gets 2x traffic of Server 2 | Servers have different capacity |
| **Least Connections** | Route to server with fewest active connections | Long-lived connections (WebSocket) |
| **IP Hash** | Hash client IP → always same server | Stateful apps that need sticky sessions |
| **Consistent Hashing** | Hash request → ring → nearest server | Distributed caches, session affinity |
| **Random** | Pick a random server | Simple, works well with many servers |

### L4 vs L7 Load Balancer
| Type | Operates at | What it sees | Speed |
|------|------------|-------------|-------|
| **L4** (Transport) | TCP/UDP | IP + Port only | Faster, less CPU |
| **L7** (Application) | HTTP | Headers, URL, body | Slower, but smarter routing |

**L7 use cases:** Route `/api/` to API servers, `/static/` to CDN, `/stream/` to video servers.

### Health Checks
```
Load Balancer pings /health on each server every N seconds.
If X consecutive failures → mark server as DOWN → stop sending traffic.
When server recovers → mark UP → resume traffic.
```

---

## 4. Reverse Proxy vs Forward Proxy

| Type | Who configures it | Hides |
|------|------------------|-------|
| **Forward Proxy** | Client | Client's IP from server |
| **Reverse Proxy** | Server | Server's IP from client |

### Reverse Proxy Benefits
- SSL termination (decrypt HTTPS, forward plain HTTP internally)
- Compression (gzip responses)
- Caching (basic response caching)
- Security (hide backend, block malicious requests)
- Single entry point

**Examples:** Nginx, HAProxy, AWS ALB

---

## 5. API Gateway

**What it does:** A managed entry point for all API calls. Sits in front of your microservices.

```
[Client]
   |
[API Gateway]
   |── /users/*   ──> [User Service]
   |── /orders/*  ──> [Order Service]
   |── /payment/* ──> [Payment Service]
```

### API Gateway Responsibilities
```
1. Authentication & Authorization (verify JWT token)
2. Rate Limiting (100 requests/min per API key)
3. Request Routing (route to correct microservice)
4. Protocol Translation (REST to gRPC)
5. Request/Response transformation
6. Logging & Monitoring
7. SSL Termination
```

**Examples:** AWS API Gateway, Kong, Nginx, Apigee

---

## 6. Service Discovery

In microservices, services need to find each other. Two approaches:

### Client-Side Discovery
```
[Service A] ──asks──> [Service Registry (Consul/Eureka)] ──returns IP──> 
[Service A] ──calls──> [Service B at returned IP]
```

### Server-Side Discovery (more common)
```
[Service A] ──calls──> [Load Balancer] ──consults Registry──> routes to [Service B]
```

**Examples:** AWS ECS service discovery, Kubernetes DNS, Consul, Eureka

---

## Quick Reference

| Component | Solves |
|-----------|--------|
| DNS | Name resolution, geo-routing |
| CDN | Latency for static content, global users |
| Load Balancer | Single-server bottleneck, fault tolerance |
| Reverse Proxy | Security, SSL termination, compression |
| API Gateway | Auth, rate limiting, routing for microservices |
| Service Discovery | Microservices finding each other dynamically |
