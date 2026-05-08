# Design: Uber / Lyft (Ride-Sharing)

## Problem Statement
Design a ride-sharing platform where riders request rides, nearby drivers receive the request, and the system matches them in real time.

---

## Step 1: Clarify Requirements

### Questions to Ask
- "Driver and rider matching only, or also payments, reviews?" → Focus on matching and real-time tracking
- "What's the acceptable matching latency?" → < 1 minute from request to driver accept
- "Surge pricing?" → Mention as extension
- "Scale?" → 1M rides/day, operating in 100 cities

### Functional Requirements
- Riders can request a ride (from current location to destination)
- System finds nearby available drivers
- Matched driver sees pickup location and navigates
- Real-time location tracking for both parties
- Ride status updates: matched, en route, arrived, in progress, completed

### Non-Functional Requirements
- Location updates must be processed in near real-time (< 5s staleness)
- Matching latency < 30 seconds
- High availability (99.99% — downtime = stranded users)
- Consistency: driver cannot be matched to two rides simultaneously

---

## Step 2: Capacity Estimation

```
Drivers on platform: 5 million
Active drivers at peak: 500,000
Rides/day: 10 million

Location updates:
  Active driver sends GPS every 5 seconds
  500,000 drivers × 12 updates/min × 60 min/hr ÷ 60 = 100,000 updates/sec at peak

Ride requests:
  10M/day / 86,400 ≈ 116 RPS (low; latency matters more than throughput)
```

---

## Step 3: High-Level Architecture

```
[Driver App]                          [Rider App]
     |                                      |
GPS update every 5s              Request ride (pickup, destination)
     |                                      |
[Location Service]              [Ride Service]
     |                                      |
[Location Store]              [Matching Service]
  (Redis Geo)                         |
     ↑                        find nearby drivers
     |                        (query Location Store)
     |                                |
     +────── matched driver ──────────+
     |
[Driver notified]            [Rider sees driver ETA]
     |                                |
     +────── WebSocket / Push ────────+

[Trip Service] ── stores trip state ──> [MySQL]
[Notification Service] ──> [APNs / FCM]
[Routing/ETA Service] ──> [Google Maps API / OSRM]
[Payment Service]
```

---

## Step 4: Location Tracking (Core Challenge)

### Driver Location Storage
```
Requirements:
- Update 500K drivers' locations every 5 seconds
- Query: "find all drivers within 5 km of (lat, lng)"
- Must support efficient geospatial queries

Solution: Redis Geospatial Commands
  GEOADD drivers:city_1 <lng> <lat> <driver_id>
  → Updates driver's position (replaces previous)

  GEORADIUS drivers:city_1 <rider_lng> <rider_lat> 5 km ASC COUNT 50
  → Returns up to 50 driver IDs within 5km, sorted by distance

Why Redis?
  - In-memory → sub-millisecond queries
  - Geospatial indexes built-in
  - 100K writes/sec is easily handled
```

### Location Update Flow
```
Driver app → POST /api/v1/driver/location
           { driver_id, lat, lng, heading, speed, available: true }

Location Service:
  1. GEOADD drivers:{city} lng lat driver_id  (Redis)
  2. Store in DB (async via Kafka) for trip history
  3. If driver is in active trip → push location to rider via WebSocket/SSE
```

### Sharding Location by City
```
drivers:city_london → Redis cluster shard 1
drivers:city_paris  → Redis cluster shard 2
drivers:city_nyc    → Redis cluster shard 3

Assignment: Based on city derived from GPS coordinates.
Works because rides are local (no inter-city matching needed).
```

---

## Step 5: Ride Matching Algorithm

```
Rider requests ride at (lat, lng):

1. Find candidates:
   GEORADIUS drivers:city_1 lat lng 3km ASC COUNT 20
   → Returns 20 nearest available driver IDs

2. Filter candidates:
   - Driver.available == true
   - Driver.vehicle_type matches requested type
   - Driver not already matched (check in-flight requests)

3. Score candidates:
   Score = distance (primary) + driver rating (secondary) + acceptance rate

4. Offer ride to best driver (timeout = 10 seconds)
   If driver rejects or no response → offer to next driver

5. On accept:
   - Set driver.available = false  (atomic in DB to prevent double-match)
   - Create trip record
   - Notify rider
```

### Preventing Double-Assignment
```
Use optimistic locking (DB CAS):

UPDATE drivers 
SET available = false, current_trip_id = :trip_id
WHERE driver_id = :driver_id AND available = true

If rows_affected == 0 → driver was already taken → try next candidate
```

---

## Step 6: Trip State Machine

```
REQUESTED → MATCHED → DRIVER_EN_ROUTE → ARRIVED → IN_PROGRESS → COMPLETED
                                                               ↘ CANCELLED
```

### Trip Table
```sql
CREATE TABLE trips (
  id              BIGINT PRIMARY KEY,
  rider_id        BIGINT NOT NULL,
  driver_id       BIGINT,
  status          ENUM('requested','matched','en_route','arrived','in_progress','completed','cancelled'),
  pickup_lat      DECIMAL(9,6),
  pickup_lng      DECIMAL(9,6),
  destination_lat DECIMAL(9,6),
  destination_lng DECIMAL(9,6),
  requested_at    DATETIME,
  matched_at      DATETIME,
  completed_at    DATETIME,
  fare_amount     DECIMAL(10,2),
  distance_km     DECIMAL(6,2)
);
-- Shard by rider_id
```

---

## Step 7: Real-time Location Sharing During Trip

```
During a trip, rider sees driver's location moving on map.

Options:
  A. WebSocket (bidirectional, low latency) — preferred
  B. SSE (server-to-client stream) — simpler
  C. Polling every 3s — wasteful but simple fallback

Flow:
  Driver app sends GPS every 3 seconds (faster during trip)
  → Location Service updates Redis + publishes to Kafka
  → Trip Service consumes Kafka event
  → Pushes driver location to rider via WebSocket connection
  → Rider map updates
```

---

## Step 8: ETA Calculation

```
Simple approach:
  ETA = Haversine distance / average speed
  (too inaccurate — doesn't account for traffic)

Better approach:
  Call routing service (Google Maps Directions API or OSRM)
  Input: driver location, pickup location
  Output: route + ETA with real-time traffic

Cache ETAs:
  Cache for same origin-destination pair for 30 seconds
  (Don't call Maps API for every 3-second location update)
```

---

## Step 9: API Design

```
POST /api/v1/rides/request
Body: { "pickup": {lat, lng}, "destination": {lat, lng}, "ride_type": "UberX" }
Response: { "ride_id": "123", "estimated_wait": "3 min", "fare_estimate": "$12" }

PUT  /api/v1/driver/location
Body: { "lat": 37.7749, "lng": -122.4194, "available": true }

GET  /api/v1/rides/{id}/status
Response: { "status": "en_route", "driver_location": {lat, lng}, "eta": "4 min" }

POST /api/v1/rides/{id}/cancel

WebSocket: ws://api.uber.com/rides/{id}/track
Events: { "type": "driver_location", "lat": ..., "lng": ... }
```

---

## Interview Tips

- Key insight: "Location storage in Redis Geospatial — 500K updates/sec, sub-millisecond radius queries."
- Matching: "Optimistic locking on driver.available to prevent double-assignment — atomic DB update."
- Scalability: "Shard by city — rides are local, so city-level partitioning avoids cross-shard queries."
- Tracking: "WebSocket for real-time driver location during trip; push is more efficient than poll."

## Common Follow-up Questions
- "How to handle surge pricing?" → Compute supply/demand ratio per geo cell; apply multiplier
- "How to handle driver going offline mid-trip?" → Last known location + re-routing; try to contact
- "How to scale to 10 cities?" → One Redis cluster per city, routed by geo lookup
- "How does ETA improve over time?" → ML model on historical trip data + real-time traffic
