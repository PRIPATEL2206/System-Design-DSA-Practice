# Computer Networks — Complete Deep Dive

## Why Networks Matter
Every system design question involves networking. Understanding how data travels from client to server (and back) is essential for debugging, performance, and architecture.

---

## 1. OSI Model & TCP/IP Model

```
OSI (7 Layers)          TCP/IP (4 Layers)       What Happens Here
──────────────          ─────────────────        ─────────────────
7. Application    ←──→  Application              HTTP, DNS, FTP, SMTP
6. Presentation                                  Encryption, Compression
5. Session                                       Session management
4. Transport      ←──→  Transport                TCP, UDP (port-to-port)
3. Network        ←──→  Internet                 IP, Routing (host-to-host)
2. Data Link      ←──→  Network Access           Ethernet, WiFi, MAC
1. Physical                                      Cables, Radio waves

Data encapsulation (sending):
Application: [DATA]
Transport:   [TCP Header | DATA]              → Segment
Network:     [IP Header | TCP Header | DATA]  → Packet
Data Link:   [ETH | IP | TCP | DATA | CRC]   → Frame
Physical:    101010110010...                   → Bits
```

---

## 2. TCP (Transmission Control Protocol)

### TCP 3-Way Handshake
```
Client                          Server
  │                                │
  │──── SYN (seq=x) ────────────▶│   "I want to connect"
  │                                │
  │◀─── SYN-ACK (seq=y, ack=x+1)──│   "OK, I'm ready too"
  │                                │
  │──── ACK (ack=y+1) ───────────▶│   "Great, let's talk"
  │                                │
  │═══════ CONNECTION ESTABLISHED ═══│

Why 3 steps? To synchronize sequence numbers both ways.
```

### TCP 4-Way Termination
```
Client                          Server
  │──── FIN ─────────────────────▶│   "I'm done sending"
  │◀─── ACK ─────────────────────│   "Got it"
  │                                │   (server may still send data)
  │◀─── FIN ─────────────────────│   "I'm done too"
  │──── ACK ─────────────────────▶│   "Goodbye"
  │                                │
  │─── TIME_WAIT (2×MSL) ────────│   (wait for lost packets)
```

### TCP Features
```
Reliable delivery:     Sequence numbers + ACKs + retransmission
Ordered:              Sequence numbers ensure correct order
Flow control:         Receiver advertises window size (don't overwhelm me)
Congestion control:   Slow start → Congestion avoidance → Fast recovery

Congestion Window (cwnd):
  Slow Start: cwnd doubles every RTT (exponential growth)
  Threshold: Switch to linear growth (additive increase)
  Packet loss: Cut cwnd in half (multiplicative decrease)
  
  ┌───────────────────────────────────┐
  │        Congestion Window          │
  │  cwnd                             │
  │   ▲        /\                     │
  │   │      /    \    /\             │
  │   │    /        \/    \           │
  │   │  /                  \         │
  │   │/                     \...     │
  │   └──────────────────────────▶ t  │
  │   slow    threshold   loss        │
  │   start                           │
  └───────────────────────────────────┘
```

### TCP vs UDP
```
┌──────────────────┬───────────────────┬───────────────────┐
│ Feature          │ TCP               │ UDP               │
├──────────────────┼───────────────────┼───────────────────┤
│ Connection       │ Connection-oriented│ Connectionless    │
│ Reliability      │ Guaranteed        │ Best-effort       │
│ Ordering         │ Yes               │ No                │
│ Header size      │ 20-60 bytes       │ 8 bytes           │
│ Speed            │ Slower            │ Faster            │
│ Flow control     │ Yes               │ No                │
│ Use cases        │ HTTP, Email, File │ DNS, Video, Gaming│
│                  │ transfer, SSH     │ VoIP, IoT         │
└──────────────────┴───────────────────┴───────────────────┘
```

---

## 3. HTTP (HyperText Transfer Protocol)

### HTTP/1.1 vs HTTP/2 vs HTTP/3
```
HTTP/1.1:
  - One request per connection (or keep-alive for sequential)
  - Head-of-line blocking (one slow response blocks everything)
  - Text-based headers (large, uncompressed)

HTTP/2:
  - Multiplexing: Multiple streams on ONE connection
  - Header compression (HPACK)
  - Server push (proactive resource sending)
  - Binary framing (more efficient than text)
  - Still TCP underneath → TCP head-of-line blocking

HTTP/3:
  - Based on QUIC (UDP underneath)
  - No TCP head-of-line blocking
  - Built-in encryption (TLS 1.3 in the protocol)
  - 0-RTT connection resumption (instant reconnect)
  - Better for mobile (handles network switching)
```

### HTTPS & TLS
```
TLS Handshake (simplified):
1. Client Hello: "I support these cipher suites"
2. Server Hello: "Let's use this cipher, here's my certificate"
3. Client verifies certificate (chain of trust → CA)
4. Key exchange: Agree on shared secret (Diffie-Hellman)
5. Both derive encryption keys from shared secret
6. Encrypted communication begins

What certificate proves:
  - Server is who it claims to be (authentication)
  - Data can't be read by middleman (encryption)
  - Data can't be tampered with (integrity)
```

### Common HTTP Headers
```
Request:
  Host: example.com                    (which server)
  Authorization: Bearer <token>         (authentication)
  Content-Type: application/json        (body format)
  Accept: application/json              (expected response format)
  Cache-Control: no-cache               (caching directive)
  Cookie: session_id=abc123             (session tracking)

Response:
  Status: 200 OK / 404 Not Found / 500 Error
  Content-Type: application/json
  Set-Cookie: session_id=abc123; HttpOnly; Secure
  Cache-Control: max-age=3600           (cache for 1 hour)
  ETag: "abc123"                        (content version)
  Access-Control-Allow-Origin: *        (CORS)
```

---

## 4. DNS (Domain Name System)

```
Query: "What is the IP of www.google.com?"

Resolution path:
┌──────────┐     ┌──────────────┐     ┌──────────┐     ┌─────────────┐
│ Browser  │────▶│ Local DNS    │────▶│ Root DNS │────▶│.com TLD DNS │
│  Cache   │     │  Resolver    │     │(13 global)│    │             │
└──────────┘     └──────────────┘     └──────────┘     └──────┬──────┘
                        │                                       │
                        │◀──── "Ask ns1.google.com" ────────────┘
                        │
                        │     ┌──────────────────┐
                        └────▶│ google.com       │
                              │ Authoritative DNS│
                              │ "IP: 142.250.x.x"│
                              └──────────────────┘

Record Types:
  A      → Domain → IPv4 address
  AAAA   → Domain → IPv6 address
  CNAME  → Alias → Another domain name
  MX     → Domain → Mail server
  NS     → Domain → Nameserver
  TXT    → Domain → Text (SPF, DKIM for email verification)
  SOA    → Zone authority information

TTL (Time To Live): How long to cache the response
  High TTL (hours/days): Fewer lookups, slow propagation of changes
  Low TTL (seconds): Fast updates, more DNS traffic
```

---

## 5. Sockets & Connection Handling

```
Socket: Endpoint for communication = IP + Port

TCP Socket Flow:
Server:                          Client:
socket()                         socket()
bind(IP, port)
listen(backlog)
accept() ← blocks               connect(server_ip, port)
recv/send                        send/recv
close()                          close()

Port ranges:
  0-1023:     Well-known (HTTP=80, HTTPS=443, SSH=22, DNS=53)
  1024-49151: Registered (MySQL=3306, PostgreSQL=5432, Redis=6379)
  49152-65535: Dynamic/ephemeral (client-side)

Multiplexing (handle many connections):
  select():     Monitor FDs, O(n) per call, limit 1024 FDs
  poll():       Like select but no FD limit
  epoll():      O(1) per event, scales to millions (Linux)
  kqueue():     Like epoll (BSD/macOS)
  
  Reactor pattern: Event loop + epoll (Node.js, Nginx, Redis)
```

---

## 6. Network Security

```
Common Attacks:
──────────────
DDoS:           Overwhelm with traffic (SYN flood, amplification)
Man-in-Middle:  Intercept communication (prevented by TLS)
DNS Spoofing:   Return fake IP for domain
ARP Spoofing:   Redirect traffic on LAN
SQL Injection:  Malicious input in queries
XSS:            Inject scripts into web pages
CSRF:           Trick user's browser into making requests

Defenses:
─────────
TLS/HTTPS:      Encrypts all traffic
CORS:           Restrict cross-origin requests
CSP:            Restrict what scripts can run
Rate Limiting:  Prevent brute force / DDoS
WAF:            Web Application Firewall (filter malicious requests)
VPN:            Encrypted tunnel for all traffic
```

---

## 7. Key Protocols Quick Reference

```
Protocol  Port   Transport   Purpose
────────────────────────────────────────────────
HTTP      80     TCP         Web browsing
HTTPS     443    TCP         Secure web
SSH       22     TCP         Secure shell
FTP       21     TCP         File transfer
SMTP      25     TCP         Send email
DNS       53     UDP/TCP     Name resolution
DHCP      67/68  UDP         Auto IP assignment
WebSocket 80/443 TCP         Bidirectional real-time
gRPC      443    TCP(HTTP/2) High-perf RPC
MQTT      1883   TCP         IoT messaging
```

---

## 8. Interview Questions (Top 20)

1. What happens when you type "google.com" in browser? (Full journey!)
2. Explain TCP 3-way handshake. Why not 2-way?
3. TCP vs UDP — when to use which?
4. How does HTTPS work? Explain TLS handshake.
5. What is DNS? Walk through a DNS query.
6. Explain HTTP/2 multiplexing.
7. What is a CDN? How does it work?
8. Explain REST vs gRPC.
9. What is a WebSocket? When to use it vs HTTP?
10. How does a load balancer work? L4 vs L7?
11. What is NAT? Why does it exist?
12. Explain TCP congestion control (slow start, AIMD).
13. What is the difference between HTTP 1.1, 2, and 3?
14. How do cookies and sessions work?
15. What is CORS and why does it exist?
16. Explain the difference between forward proxy and reverse proxy.
17. What is TCP head-of-line blocking? How does HTTP/3 solve it?
18. How does traceroute work?
19. What is a subnet mask? How does subnetting work?
20. Explain how a VPN works.
