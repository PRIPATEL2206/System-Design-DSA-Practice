# Computer Networking — Essentials for Tech Interviews

## The OSI/TCP-IP Model (Simplified)

```
Application Layer  │ HTTP, HTTPS, WebSocket, DNS, SMTP
Transport Layer    │ TCP, UDP
Network Layer      │ IP, ICMP, Routing
Link/Physical      │ Ethernet, WiFi, ARP
```

---

## HTTP & HTTPS

### HTTP Methods
| Method | Purpose | Idempotent | Safe |
|--------|---------|-----------|------|
| GET | Read data | Yes | Yes |
| POST | Create resource | No | No |
| PUT | Replace resource | Yes | No |
| PATCH | Partial update | No | No |
| DELETE | Remove resource | Yes | No |

### HTTP Status Codes (Know These)
| Range | Category | Common Codes |
|-------|----------|-------------|
| 2xx | Success | 200 OK, 201 Created, 204 No Content |
| 3xx | Redirect | 301 Permanent, 302 Temporary, 304 Not Modified |
| 4xx | Client Error | 400 Bad Request, 401 Unauthorized, 403 Forbidden, 404 Not Found, 429 Too Many Requests |
| 5xx | Server Error | 500 Internal Error, 502 Bad Gateway, 503 Service Unavailable |

### HTTP/1.1 vs HTTP/2 vs HTTP/3
| Feature | HTTP/1.1 | HTTP/2 | HTTP/3 |
|---------|---------|--------|--------|
| Connections | 1 request per connection (or keep-alive) | Multiplexed streams over 1 connection | Multiplexed over QUIC (UDP) |
| Head-of-line blocking | Yes (per connection) | TCP level only | None (UDP-based) |
| Header compression | None | HPACK | QPACK |
| Server push | No | Yes | Yes |

### HTTPS
- HTTP + TLS encryption
- TLS handshake: Client Hello → Server Hello + Certificate → Key Exchange → Encrypted communication
- Prevents: Eavesdropping, tampering, impersonation

---

## TCP vs UDP

| Feature | TCP | UDP |
|---------|-----|-----|
| Connection | Connection-oriented (3-way handshake) | Connectionless |
| Reliability | Guaranteed delivery, ordering | Best effort, no guarantees |
| Flow control | Yes (sliding window) | No |
| Congestion control | Yes (slow start, AIMD) | No |
| Overhead | Higher (headers, ACKs) | Lower |
| Use cases | Web, email, file transfer | Video streaming, gaming, DNS, VoIP |

### TCP 3-Way Handshake
```
Client → SYN → Server
Client ← SYN-ACK ← Server
Client → ACK → Server
Connection established!
```

### TCP 4-Way Termination
```
Client → FIN → Server
Client ← ACK ← Server
Client ← FIN ← Server
Client → ACK → Server
Connection closed (after TIME_WAIT)
```

---

## DNS (Domain Name System)

### How DNS Resolution Works
```
1. Browser cache → OS cache → Router cache
2. If miss: Recursive resolver (ISP's DNS)
3. Root nameserver → "Ask .com TLD server"
4. TLD server → "Ask google.com authoritative server"
5. Authoritative server → "IP is 142.250.x.x"
6. Cache at each level (TTL-based)
```

### DNS Record Types
| Type | Purpose | Example |
|------|---------|---------|
| A | Name → IPv4 | google.com → 142.250.190.14 |
| AAAA | Name → IPv6 | google.com → 2607:f8b0::... |
| CNAME | Alias to another name | www.google.com → google.com |
| MX | Mail server | google.com → smtp.google.com |
| NS | Nameserver delegation | google.com → ns1.google.com |

### DNS in System Design
- **DNS load balancing:** Return different IPs (round-robin)
- **GeoDNS:** Return IP closest to user's location
- **TTL trade-off:** Low TTL = fast failover but more DNS queries. High TTL = cached but slow to update.

---

## WebSockets

### When to Use
- Real-time bidirectional communication (chat, gaming, live updates)
- Server needs to push data without client polling

### How It Works
```
1. Client sends HTTP upgrade request
   GET /chat HTTP/1.1
   Upgrade: websocket
   Connection: Upgrade

2. Server responds 101 Switching Protocols

3. Full-duplex communication over persistent TCP connection
```

### WebSocket vs Alternatives
| Method | Direction | Latency | Use Case |
|--------|-----------|---------|----------|
| HTTP Polling | Client → Server (repeated) | High (interval delay) | Simple, low-frequency updates |
| Long Polling | Client waits for server response | Medium | Notifications (old approach) |
| SSE (Server-Sent Events) | Server → Client (one-way) | Low | Live feeds, stock prices |
| WebSocket | Bidirectional | Lowest | Chat, gaming, collaboration |

---

## REST vs gRPC vs GraphQL

| Feature | REST | gRPC | GraphQL |
|---------|------|------|---------|
| Protocol | HTTP/1.1+ JSON | HTTP/2 + Protobuf | HTTP + JSON |
| Performance | Good | Excellent (binary, streaming) | Good |
| Flexibility | Fixed endpoints | Fixed contracts | Client-defined queries |
| Use case | Public APIs | Internal microservices | Complex, nested data |
| Learning curve | Low | Medium | Medium |
| Streaming | No (polling/SSE) | Bidirectional streaming | Subscriptions |

---

## Load Balancing Algorithms

| Algorithm | How It Works | Best For |
|-----------|-------------|----------|
| Round Robin | Rotate through servers | Equal-capacity servers |
| Weighted Round Robin | Proportional to capacity | Different-sized servers |
| Least Connections | Send to least busy | Long-lived connections |
| IP Hash | Same client → same server | Session affinity needed |
| Random | Random selection | Simple, stateless |

### L4 vs L7 Load Balancer
- **L4 (Transport):** Routes based on IP + port. Fast, simple. (AWS NLB)
- **L7 (Application):** Routes based on URL, headers, cookies. Smart routing. (AWS ALB, Nginx)

---

## Network Security Basics

### Common Attacks
| Attack | Description | Mitigation |
|--------|-------------|-----------|
| DDoS | Overwhelm with traffic | Rate limiting, CDN, WAF |
| Man-in-the-Middle | Intercept communication | HTTPS/TLS |
| DNS Spoofing | Fake DNS response | DNSSEC |
| SQL Injection | Malicious SQL in input | Parameterized queries |
| XSS | Inject client-side scripts | Input sanitization, CSP |

### Firewalls & Proxies
- **Firewall:** Filter traffic by rules (allow/deny by IP, port, protocol)
- **Forward Proxy:** Client → Proxy → Server (hides client identity)
- **Reverse Proxy:** Client → Proxy → Server (hides server, adds caching/SSL/LB). Examples: Nginx, Cloudflare

---

## Interview Questions

**Q: What happens when you type google.com in browser?**
DNS → TCP connect → TLS handshake → HTTP GET / → Server response → Browser render

**Q: How does HTTPS prevent eavesdropping?**
TLS uses asymmetric crypto for key exchange, then symmetric crypto for data. Even if you intercept the encrypted data, you can't decrypt without the session key.

**Q: TCP vs UDP for video streaming?**
UDP — dropped frames are better than delayed frames. TCP's retransmission adds latency that ruins real-time playback.

**Q: How would you design for low latency?**
- CDN for static content (reduce distance)
- Connection pooling (avoid handshake overhead)
- HTTP/2 multiplexing (reduce connections)
- Regional deployments (geo-routing)
- Caching at every layer

**Q: What's a CDN and why use it?**
Network of edge servers worldwide that cache static content. User gets served from nearest edge → lower latency, less load on origin.
