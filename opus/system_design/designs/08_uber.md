# Design: Uber / Ride-Sharing Service

**Frequency:** Very High | **Difficulty:** ⭐⭐⭐⭐ | **Companies:** Uber, Google, Amazon, Flipkart

---

## 1. Requirements

### Functional
- Rider requests a ride (pickup → destination)
- Match rider with nearest available driver
- Real-time location tracking (driver moving)
- ETA calculation
- Fare estimation & payment
- Trip history

### Non-Functional
- **Scale:** 20M trips/day, 5M concurrent drivers
- **Latency:** Match within 10 seconds, location update < 1s
- **Availability:** 99.99% (can't leave riders stranded)
- **Consistency:** Trip state must be strongly consistent

### Estimation
```
Trips/sec: 20M / 86400 ≈ 230 trips/sec
Location updates: 5M drivers × 1 update/4 sec = 1.25M updates/sec
Storage (trips): 20M × 2KB = 40 GB/day
Location data: 1.25M × 50 bytes = 62.5 MB/sec (5.4 TB/day if stored)
```

---

## 2. High-Level Design

```
┌──────────┐                                      ┌──────────┐
│  Rider   │                                      │  Driver  │
│   App    │                                      │   App    │
└────┬─────┘                                      └────┬─────┘
     │                                                  │
     │  HTTP/WebSocket                    WebSocket (location)
     │                                                  │
┌────▼───────────────────────────────────────────────────▼────┐
│                      API Gateway                            │
└────┬──────────┬──────────────┬──────────────┬───────────────┘
     │          │              │              │
┌────▼────┐ ┌──▼───────┐ ┌───▼──────┐ ┌────▼──────┐
│  Trip   │ │ Matching │ │ Location │ │  Pricing  │
│ Service │ │ Service  │ │ Service  │ │  Service  │
└────┬────┘ └──┬───────┘ └───┬──────┘ └───────────┘
     │         │             │
┌────▼────┐ ┌──▼───────┐ ┌───▼──────┐
│ Trip DB │ │Supply/   │ │ Location │
│(Postgres)││Demand DB │ │  Store   │
└─────────┘ └──────────┘ │  (Redis) │
                          └──────────┘
```

---

## 3. Location Service (The Heart)

### The Challenge
- 5M drivers sending GPS every 4 seconds
- Need to query: "Find drivers within 3 km of (lat, lng)"
- Must be FAST (sub-100ms)

### Solution: Geospatial Indexing

```
Option 1: Geohash + Redis (Chosen)
────────────────────────────────────
Geohash: Convert (lat, lng) → string prefix
  - "tdr1w" (5 chars) ≈ 5km × 5km grid cell
  - "tdr1wm" (6 chars) ≈ 1.2km × 0.6km grid cell
  - Nearby locations share prefix!

Redis GEO commands:
  GEOADD drivers <lng> <lat> "driver_123"
  GEORADIUS drivers <lng> <lat> 3 km COUNT 20

Why Redis GEO?
  - O(log N + M) radius search (N=total, M=results)
  - 1.25M writes/sec → sharded Redis handles this
  - In-memory = sub-millisecond queries

Option 2: QuadTree (used by some systems)
──────────────────────────────────────────
Divide map into quadrants recursively
Each leaf contains ≤ N drivers
Query: traverse tree to find relevant cells
⚠️ Harder to update (drivers move constantly)

Option 3: S2 Geometry (Google's approach)
──────────────────────────────────────────
Maps sphere to cells of various sizes
Cells have unique IDs (64-bit)
Better for global coverage than geohash
```

### Location Update Flow
```
Driver GPS update (every 4 seconds):
1. Driver app → WebSocket → Location Service
2. Update Redis GEO: GEOADD drivers <lng> <lat> "driver_id"
3. Update driver's current location in memory
4. If driver on active trip → push location to rider (WebSocket)
5. Async: Write to location history (Kafka → Cassandra) for analytics
```

---

## 4. Matching Algorithm

```
Rider requests ride:
1. Get rider's location
2. Query Location Service: "Available drivers within 3km"
3. Filter: vehicle type, rating, direction of travel
4. Rank by: distance + ETA (not just straight-line distance)
5. Send request to top driver
6. Driver has 15 seconds to accept
7. Timeout → try next driver
8. Match confirmed → create trip

Optimization: Supply-Demand matching
─────────────────────────────────────
Instead of greedy nearest-driver:
- Batch requests every few seconds
- Solve assignment problem (minimize total wait time)
- Consider: driver heading toward rider (less detour)
- Surge areas: prioritize high-demand zones

     Rider A ----→ Driver 1 (nearest but heading away)
     Rider A ----→ Driver 2 (slightly farther but heading toward) ← BETTER
```

---

## 5. Trip Lifecycle

```
States:
  REQUESTED → MATCHED → DRIVER_ENROUTE → ARRIVED → IN_PROGRESS → COMPLETED

┌──────────┐    match    ┌──────────┐   arrive   ┌──────────┐
│REQUESTED │───────────▶│  MATCHED │───────────▶│ ARRIVED  │
└──────────┘            └──────────┘            └──────────┘
                                                      │
                              ┌────────────────────────┘
                              │ start ride
                              ▼
                        ┌───────────┐   end ride  ┌───────────┐
                        │IN_PROGRESS│────────────▶│ COMPLETED │
                        └───────────┘             └───────────┘

Trip stored in PostgreSQL (ACID for state transitions):
- trip_id, rider_id, driver_id
- status, pickup_location, dropoff_location
- start_time, end_time
- fare, payment_status
- route (polyline of GPS points)
```

---

## 6. ETA & Routing

```
ETA Calculation:
1. Graph of road network (nodes = intersections, edges = roads)
2. Edge weights = travel time (based on speed limit + current traffic)
3. Dijkstra/A* for shortest path
4. Real-time traffic: Update edge weights from driver GPS data

Architecture:
┌─────────────────────────────────────────────────────────┐
│                    Routing Service                       │
│                                                          │
│  Road Graph (pre-loaded in memory, ~10GB for a city)    │
│  + Real-time traffic overlay (from Kafka stream)        │
│  + ML model for prediction (time of day, events)        │
│                                                          │
│  Input: (start_lat, start_lng, end_lat, end_lng)       │
│  Output: route polyline + ETA                           │
└─────────────────────────────────────────────────────────┘

Optimization: Pre-compute ETA between major zones
  - Divide city into ~1000 zones
  - Cache zone-to-zone ETA matrix (updated every 5 min)
  - Fine-grained routing only for final/first mile
```

---

## 7. Pricing / Surge

```
Dynamic Pricing:
  base_fare + (per_km × distance) + (per_min × time) × surge_multiplier

Surge Calculation:
  surge = demand / supply (per geographic zone)
  
  Zone "Downtown" at 6pm:
    Requests: 500/min (demand)
    Available drivers: 50 (supply)
    Ratio: 10 → surge_multiplier = 2.5x

  Recalculated every 2-5 minutes per zone
  Smoothed to avoid dramatic jumps

Implementation:
  - Demand: Count ride requests per zone per minute
  - Supply: Count available drivers per zone
  - ML model predicts optimal multiplier
  - Display to user BEFORE they confirm (transparency)
```

---

## 8. Scaling

```
Component          Strategy
───────────────────────────────────────────────────────────
Location Service   Shard Redis by geohash region (city-level)
Trip Service       Shard PostgreSQL by city or trip_id
Matching           Per-city instance (each city independent)
Routing            Per-city (each city has its own road graph)
WebSocket          Horizontal scale + session affinity
Payment            Separate service, strong consistency

Multi-city architecture:
┌──────────────────────────────────────────────────────┐
│              Global Services                          │
│  (User accounts, Payment, Support)                   │
└────────────────────────┬─────────────────────────────┘
                         │
         ┌───────────────┼───────────────┐
         │               │               │
    ┌────▼────┐    ┌────▼────┐    ┌────▼────┐
    │Bangalore│    │ Mumbai  │    │  Delhi  │
    │ Region  │    │ Region  │    │ Region  │
    │(all svcs)│   │(all svcs)│   │(all svcs)│
    └─────────┘    └─────────┘    └─────────┘
    
Each city/region is essentially independent for real-time operations
```

---

## 9. Failure Handling

```
Driver app crashes mid-trip:
  - Trip stays IN_PROGRESS
  - System waits for reconnect (2 min)
  - If no reconnect → alert rider, reassign or end trip

Location service outage:
  - Matching degrades (use last known positions)
  - Active trips continue (riders have driver's last location)
  - Circuit breaker prevents cascade

Payment failure:
  - Complete trip first (don't leave rider stranded)
  - Retry payment async
  - If persistent failure → bill later / credit system
```
