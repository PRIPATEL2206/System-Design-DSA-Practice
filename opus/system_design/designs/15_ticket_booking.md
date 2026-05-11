# Design: Ticket Booking System (BookMyShow / IRCTC)

**Frequency:** High | **Difficulty:** ⭐⭐⭐ | **Companies:** Amazon, Flipkart, Google, Swiggy

---

## 1. Requirements

### Functional
- Browse events/movies with available shows
- View seat map with availability
- Select seats and book (with payment)
- Handle concurrent bookings (same seat)
- Booking confirmation & e-ticket
- Cancellation & refund

### Non-Functional
- **Scale:** 10M bookings/day, flash sales (100K concurrent for popular events)
- **Consistency:** STRONG (no double-booking of same seat!)
- **Latency:** < 2 seconds for booking
- **Availability:** 99.99%

---

## 2. The Core Challenge: No Double-Booking

```
Problem: 1000 users click "Book Seat A1" at the same time

WRONG approach:
  1. Check if seat available (SELECT)
  2. If yes, book it (INSERT)
  ⚠️ Race condition! 100 users pass step 1, all try step 2

CORRECT approaches:

Option A: Pessimistic Locking (SELECT FOR UPDATE)
  BEGIN;
  SELECT * FROM seats WHERE seat_id = 'A1' FOR UPDATE;  -- locks row
  -- only ONE transaction gets here, others WAIT
  UPDATE seats SET status = 'BOOKED', user_id = '123';
  COMMIT;
  ✅ Correct but can cause contention under high load

Option B: Optimistic Locking (version check)
  SELECT version FROM seats WHERE seat_id = 'A1';  -- version = 5
  UPDATE seats SET status = 'BOOKED', version = 6
    WHERE seat_id = 'A1' AND version = 5;
  -- If 0 rows affected → someone else booked it, retry or fail
  ✅ Better for high contention (no waiting, fast fail)

Option C: Distributed Lock (Redis)
  SETNX lock:seat:A1 "user_123" EX 300  -- 5 min lock
  If acquired → proceed with booking
  If not → "Seat being held by another user"
  ✅ Works across multiple app servers
```

---

## 3. Architecture

```
┌──────────┐        ┌──────────────┐        ┌─────────────────────┐
│  Client  │───────▶│  API Gateway │───────▶│  Booking Service    │
└──────────┘        └──────────────┘        └──────────┬──────────┘
                                                        │
                          ┌─────────────────────────────┼──────────────┐
                          │                             │              │
                     ┌────▼─────┐              ┌───────▼──────┐  ┌───▼────┐
                     │  Seat    │              │   Payment    │  │ Event  │
                     │  Service │              │   Service    │  │ Service│
                     │ (Redis   │              └──────────────┘  └────────┘
                     │  + DB)   │
                     └──────────┘
```

---

## 4. Booking Flow

```
┌─────────────────── Booking State Machine ────────────────────┐

  AVAILABLE → HELD (temporary, 10 min) → BOOKED (confirmed)
                    ↓ (timeout)             ↓ (cancel)
               AVAILABLE                 CANCELLED → AVAILABLE

1. User selects seat → HOLD seat (10 min timer)
   - Redis: SET seat:{event}:{seat} user_123 EX 600
   - If already held → "Seat unavailable, try another"

2. User proceeds to payment (within 10 min hold)
   - Payment Service charges card/UPI
   - If payment fails → release hold, seat back to AVAILABLE

3. Payment succeeds → CONFIRM booking
   - Write to DB: booking record + seat status = BOOKED
   - Release Redis hold (no longer needed, DB is source of truth)
   - Send confirmation email/SMS

4. Hold expires (user didn't pay in 10 min):
   - Redis key auto-expires
   - Seat becomes AVAILABLE again
   - No DB write needed (hold was only in Redis)

└──────────────────────────────────────────────────────────────┘
```

---

## 5. Data Model

```
events:
  id, name, venue_id, date, total_seats, available_seats

shows:
  id, event_id, start_time, end_time, pricing_tier

seats:
  id, show_id, row, number, category (VIP/regular),
  status (available/held/booked), price

bookings:
  id, user_id, show_id, seats[], total_amount,
  status (confirmed/cancelled), payment_id, created_at

Database: PostgreSQL (ACID for booking transactions)
Cache: Redis (seat availability, holds, flash sale queue)
```

---

## 6. Flash Sale (100K users, 1000 seats)

```
Problem: Aamir Khan movie release → 100K concurrent users for 1000 seats
  Regular DB will collapse under 100K simultaneous writes

Solution: Virtual Queue + Distributed Locking

┌────────────────────────── Flash Sale Architecture ──────────────────────┐

1. All users enter virtual queue (Redis sorted set by arrival time)
   ZADD queue:{show_id} <timestamp> <user_id>

2. Process in batches of 50:
   ZPOPMIN queue:{show_id} 50  → next 50 users get to select seats

3. Selected users get 3-minute window to pick seats and pay
   If they don't → timeout, next batch enters

4. Display queue position to waiting users:
   "You are #342 in queue. Estimated wait: 2 minutes"

Benefits:
  - DB never sees 100K simultaneous writes
  - Fair (first-come, first-served)
  - Controlled load (only 50 active sessions at a time)
  - Users know their position (good UX vs error pages)

└───────────────────────────────────────────────────────────────────────┘
```

---

## 7. Seat Map Optimization

```
Show seat availability in real-time:

Approach: Bitmap in Redis
  Key: seats:{show_id}
  Value: bitmap (1 bit per seat, 0=available, 1=taken)
  
  1000 seats = 125 bytes (tiny!)
  Check seat: GETBIT seats:show_123 42 → 0 (available)
  Book seat:  SETBIT seats:show_123 42 1 → mark as taken

  Send full bitmap to client → render seat map
  WebSocket: push updates when seats get booked (real-time map)

For display: Client renders bitmap as color-coded seat map
  Green (0) = available
  Red (1) = taken
  Yellow = held by current user
```

---

## 8. Scaling

```
Component           Scale Strategy
──────────────────────────────────────────────────────────
API Gateway         Auto-scale, rate limit per user
Booking Service     Stateless, horizontal (20+ instances for flash sale)
Seat Service        Shard by show_id (each show is independent)
Redis               Cluster, shard by show_id
PostgreSQL          Shard by venue/region, read replicas for browsing
Queue (flash sale)  One Redis key per show, distributed across cluster

Flash sale specific:
  - Pre-warm all caches 30 min before
  - Disable non-essential features (recommendations, analytics)
  - Dedicated server pool for flash sale events
  - DDoS protection at edge
```

---

## 9. Preventing Fraud

```
Scalper prevention:
  - CAPTCHA before joining queue
  - Max 6 tickets per user per event
  - Device fingerprinting (same device = same user)
  - Rate limit: max 1 booking attempt per 30 seconds
  - IP-based throttling

Bot detection:
  - Unusually fast seat selection (< 1 second)
  - No mouse movement / touch events
  - Known bot IPs/user agents
  - ML model on behavioral patterns
```
