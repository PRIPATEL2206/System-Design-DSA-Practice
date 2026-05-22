# Design: Hotel Booking System (Airbnb / Booking.com)

## Problem Statement
Design a hotel/property booking system where users can search for available rooms, make reservations, and receive confirmations.

---

## Step 1: Clarify Requirements

### Questions to Ask
- "Hotel chains only or individual properties (Airbnb-style)?" → Both
- "Real-time availability, or is slight delay OK?" → Real-time (no overbooking allowed)
- "Concurrent booking of same room?" → Must prevent — strong consistency for booking
- "Search radius and filters?" → City/date search with price/amenity filters
- "Payments?" → Mention as component; not core focus
- "Scale?" → 1M hotels, 10M searches/day, 1M bookings/day

### Functional Requirements
- Search hotels by location, dates, guest count
- View hotel/room details, availability, price
- Book a room (with payment)
- View/cancel bookings
- Hotel management: add/update rooms, set prices

### Non-Functional Requirements
- No double booking: two users cannot book the same room for the same dates
- Search: eventual consistency OK (slightly stale results fine)
- Booking: strong consistency required (correctness > speed)
- High availability (99.99%)
- Read-heavy: search >> booking (100:1 ratio)

---

## Step 2: Capacity Estimation

```
Hotels: 1 million properties, 10 rooms each avg = 10 million rooms
Searches/day: 10 million (100 RPS avg, 500 RPS peak)
Bookings/day: 1 million (11 RPS avg)

Search is read-heavy (100:1 vs writes).
Booking is write-critical (must be correct).
```

---

## Step 3: High-Level Architecture

```
[User]
  |
  |── Search ──────────────────────────────────────────────────────────────────────┐
  |                                                                                 ↓
  |                                                                         [Search Service]
  |                                                                         [Elasticsearch]
  |
  |── View Room Details + Availability ──────────────────────────────────────────┐
  |                                                                               ↓
  |                                                                     [Availability Service]
  |                                                                     [Redis + MySQL]
  |
  |── Book Room ──────────────────────────────────────────────────────────────────┐
                                                                                   ↓
                                                                         [Booking Service]
                                                                         [MySQL (ACID)]
                                                                                   |
                                                                         [Payment Service]
                                                                                   |
                                                                         [Notification Service]
```

---

## Step 4: Search Service

### What Users Search For
```
Search query:
  City: "New York"
  Check-in: 2024-12-20
  Check-out: 2024-12-23
  Guests: 2
  Filters: price < $200/night, WiFi, pool
  
Returns: List of available hotels matching criteria, sorted by relevance/price
```

### Search Architecture (Elasticsearch)
```
Elasticsearch index for hotels:
{
  "hotel_id": 123,
  "name": "Grand Hyatt New York",
  "location": { "lat": 40.7549, "lon": -73.9742 },
  "city": "New York",
  "rating": 4.5,
  "amenities": ["wifi", "pool", "gym", "parking"],
  "price_min": 150,
  "min_room_size": 1
}

Search query:
  geo_distance: 10km from NYC center
  + filter: has_availability(check-in, check-out, guests)
  + filter: price_min < 200
  + filter: amenities includes "wifi"
  + sort: relevance × rating × price

Availability data not stored in Elasticsearch (too dynamic).
Elasticsearch returns candidate hotels → check availability in DB.
```

### Search + Availability Separation
```
Two-phase search:
1. Elasticsearch: "Which hotels match my filters in NYC?" → hotel IDs
2. Availability Service: "Which of these hotel IDs have rooms for Dec 20-23?" → filter list
3. Return intersected results with prices

Why not put availability in Elasticsearch?
  Availability changes every booking (1M/day).
  Elasticsearch is not optimised for frequent updates to millions of documents.
  Availability in MySQL (relational, ACID) — correct source of truth.
  Cache availability in Redis (for fast reads).
```

---

## Step 5: Availability Service

### How to Check Availability
```
room_inventory table:
  room_id | date       | total_rooms | reserved_rooms
  101     | 2024-12-20 | 5           | 3              (2 available)
  101     | 2024-12-21 | 5           | 5              (0 available — full!)
  101     | 2024-12-22 | 5           | 4              (1 available)

Query "is room 101 available Dec 20-22?":
  SELECT MIN(total_rooms - reserved_rooms) AS min_avail
  FROM room_inventory
  WHERE room_id = 101
    AND date BETWEEN '2024-12-20' AND '2024-12-22'
  
  min_avail = 0 → NOT available (Dec 21 is full)

Redis Cache:
  Key: "avail:{room_id}:{date}"
  Value: rooms_available (integer)
  
  Fast check before DB query.
  Invalidate on booking confirmation.
```

---

## Step 6: Booking — Preventing Double Booking (Core Challenge)

**This is the most important part to get right.**

### Concurrency Problem
```
Two users both see "1 room available" on Dec 20.
Both click "Book" simultaneously.
Both get confirmed → overbooking!
```

### Solution: Optimistic Locking + DB Transaction
```sql
BEGIN TRANSACTION;

-- Step 1: Check availability (with FOR UPDATE lock)
SELECT total_rooms - reserved_rooms AS available
FROM room_inventory
WHERE room_id = :room_id 
  AND date BETWEEN :check_in AND :check_out
  AND (total_rooms - reserved_rooms) > 0
FOR UPDATE;  -- ← acquires row-level lock

-- If any date returns 0 available → ROLLBACK, return "No rooms available"

-- Step 2: Reserve the room
UPDATE room_inventory
SET reserved_rooms = reserved_rooms + 1
WHERE room_id = :room_id
  AND date BETWEEN :check_in AND :check_out;

-- Step 3: Create booking record
INSERT INTO bookings (id, user_id, room_id, check_in, check_out, total_price, status)
VALUES (:booking_id, :user_id, :room_id, :check_in, :check_out, :price, 'PENDING');

COMMIT;
```

```
FOR UPDATE ensures exclusive row lock.
Second concurrent request blocks until first commits.
If first succeeds → second checks availability again → may find 0 → returns error.
No overbooking possible.
```

### Two-Phase Booking Flow
```
Phase 1: RESERVATION (hold room for 10 minutes)
  - Check availability + lock
  - Create PENDING booking
  - Redirect user to payment page
  - Set 10-minute expiry timer

Phase 2: CONFIRMATION (after payment)
  - Process payment
  - If success → update booking status to CONFIRMED
  - If fail → release reservation (decrement reserved_rooms)

Why two phases?
  Payment can take 10-30 seconds.
  Don't want to hold DB lock during payment.
  PENDING state reserves the room; timer releases it if payment fails.
```

---

## Step 7: Data Model

### hotels table
```sql
CREATE TABLE hotels (
  id          BIGINT PRIMARY KEY,
  name        VARCHAR(200),
  description TEXT,
  city        VARCHAR(100),
  address     TEXT,
  latitude    DECIMAL(9,6),
  longitude   DECIMAL(9,6),
  rating      DECIMAL(3,2),
  amenities   JSON
);
```

### rooms table
```sql
CREATE TABLE rooms (
  id          BIGINT PRIMARY KEY,
  hotel_id    BIGINT NOT NULL,
  room_type   VARCHAR(50),    -- "Standard", "Deluxe", "Suite"
  capacity    INT,
  price_night DECIMAL(10,2),
  amenities   JSON
);
```

### room_inventory table
```sql
CREATE TABLE room_inventory (
  room_id         BIGINT NOT NULL,
  date            DATE NOT NULL,
  total_rooms     INT NOT NULL,
  reserved_rooms  INT NOT NULL DEFAULT 0,
  PRIMARY KEY (room_id, date)
);
-- Pre-populated for next 2 years
-- Index: (room_id, date)
```

### bookings table
```sql
CREATE TABLE bookings (
  id              VARCHAR(36) PRIMARY KEY,  -- UUID
  user_id         BIGINT NOT NULL,
  room_id         BIGINT NOT NULL,
  check_in        DATE NOT NULL,
  check_out       DATE NOT NULL,
  total_price     DECIMAL(10,2),
  status          ENUM('pending','confirmed','cancelled','completed'),
  created_at      DATETIME,
  expires_at      DATETIME,                  -- for pending bookings
  payment_id      VARCHAR(50)
);
```

---

## Step 8: API Design

```
GET  /api/v1/hotels/search?city=NYC&check_in=2024-12-20&check_out=2024-12-23&guests=2
Response: { "hotels": [ { "id", "name", "rating", "min_price", "available_rooms" } ] }

GET  /api/v1/hotels/{id}
Response: { hotel details + room types + availability calendar }

POST /api/v1/bookings/reserve
Body: { "room_id": 101, "check_in": "2024-12-20", "check_out": "2024-12-23", "guests": 2 }
Response: { "booking_id": "uuid", "status": "pending", "expires_at": "...", "price": 450 }

POST /api/v1/bookings/{id}/confirm
Body: { "payment_token": "tok_abc" }
Response: { "status": "confirmed", "confirmation_number": "BK123456" }

DELETE /api/v1/bookings/{id}  (cancel)
GET    /api/v1/users/{id}/bookings
```

---

## Step 9: Scaling Challenges

### High Search Load
```
10M searches/day vs 1M bookings.
Search (Elasticsearch) scaled independently from Booking (MySQL).
Add Elasticsearch nodes for read scaling.
Cache popular search results (NYC Dec 20-23) for 5 minutes in Redis.
```

### Booking DB Scaling
```
MySQL primary handles all booking writes.
Read replicas serve availability checks.
Shard room_inventory by hotel_id for very large deployments.
CQRS: separate write model (MySQL) from read model (Redis cache).
```

---

## Interview Tips

- Key insight: "Search uses Elasticsearch for geo + filter queries; booking uses MySQL with FOR UPDATE locking — they have different consistency requirements."
- Two-phase booking: "Reserve (holds room, starts payment timer), then Confirm (after payment). Prevents holding DB locks during payment."
- Never say "just use a DB" for search: "City + date + availability is a multi-dimensional query — Elasticsearch with geo search is the right tool."

## Common Follow-up Questions
- "How to handle cancellations with partial refunds?" → Policy table per hotel; calculate refund based on days until check-in
- "How to show a hotel calendar?" → Redis cache per room per month; update on every booking
- "How to handle hotel overbooking (airlines also do this)?" → Track overbook ratio; notify affected users; compensate
- "How to implement dynamic pricing?" → Price rules engine; ML model for demand forecasting; update prices hourly
