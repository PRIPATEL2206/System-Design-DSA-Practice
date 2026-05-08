# Design: Stock Exchange / Trading Platform

## Problem Statement
Design a stock exchange platform where traders can place buy/sell orders, and the system matches them fairly, at ultra-low latency.

---

## Step 1: Clarify Requirements

### Questions to Ask
- "Full exchange (order matching) or just a brokerage UI?" → Full exchange with matching engine
- "Order types: market, limit, stop-loss?" → Market and limit orders
- "Latency target?" → Sub-millisecond matching (this is the crux)
- "Scale?" → 1 million orders/second (peak, like NYSE)
- "Real-time price feed?" → Yes (market data streaming)

### Functional Requirements
- Place buy/sell orders (market + limit)
- Order Book: maintain buy and sell orders per stock
- Matching Engine: match buy and sell orders (price-time priority)
- Real-time trade execution and confirmation
- Market data feed (live price, order book depth)
- Order cancellation

### Non-Functional Requirements
- Ultra-low latency: < 1ms order-to-match at exchange (not web tier)
- Strict ordering (price-time priority — no jumping the queue)
- Exactly-once execution (no double fills)
- High availability with immediate failover
- Audit trail of every order and trade

---

## Step 2: Capacity Estimation

```
NYSE peak: ~2 billion trades/day, 50,000 messages/sec peak
Nasdaq: similar

Let's target: 1 million orders/sec at peak

Market data:
  Every trade triggers a price update broadcast
  1M trades/sec × 100 subscribers = 100 million market data updates/sec
  → Requires multicast / pub-sub at massive scale
  
Storage:
  Order = 200 bytes, 1M/sec = 200 MB/sec = 17 TB/day (hot storage)
  Cold archive: tape/glacier (decades of regulatory requirement)
```

---

## Step 3: High-Level Architecture

```
[Trader (FIX Protocol / REST)]
        |
[Order Gateway]
  - FIX protocol parsing
  - Authentication
  - Rate limiting
  - Pre-trade risk checks
        |
[Order Queue (Kafka — guaranteed ordering per symbol)]
        |
[Matching Engine]
  (single-threaded per symbol, in-memory order book)
        |
        +────────────────────────────+
        |                           |
[Trade Execution DB]       [Market Data Publisher]
(immutable trade records)  (price feed to subscribers)
        |                           |
[Order State DB]           [WebSocket / Multicast]
(current order status)     (streaming to traders/displays)
```

---

## Step 4: The Matching Engine (Core)

The matching engine is the heart of the exchange. It must be:
- **Single-threaded per symbol** (to guarantee ordering, no locking needed)
- **In-memory** (no DB calls in hot path — must be nanosecond fast)

### Order Book Structure
```
Symbol: AAPL

BUY side (Bids):         SELL side (Asks):
Price    Qty   OrderIDs  Price    Qty   OrderIDs
$149.95  500   [A, B]    $150.00  200   [C]
$149.90  300   [D]       $150.05  400   [E, F]
$149.85  200   [G]       $150.10  100   [G]

Best Bid = $149.95  (highest buy price)
Best Ask = $150.00  (lowest sell price)
Spread = $0.05

Order book is a sorted map (price → queue of orders at that price).
  Buy side: sorted DESCENDING (highest first)
  Sell side: sorted ASCENDING (lowest first)
  
Implementation: Java TreeMap or C++ std::map (O(log n) insert/delete)
Or: Price-level array for known range (O(1) lookup) — used in production HFT
```

### Price-Time Priority Matching
```
Rules:
  1. Price priority: better price (higher buy, lower sell) matched first
  2. Time priority: at same price, earlier order matched first (FIFO)

Matching a MARKET BUY order for 300 shares:
  Take from best ask: C has 200 shares at $150.00 → fill 200 shares
  Take from next ask: E has 400 shares at $150.05 → fill remaining 100 shares
  Total: 300 shares filled at avg price ~$150.01

Matching a LIMIT BUY at $150.00 for 300 shares:
  Check if any ask price ≤ $150.00
  C has 200 @ $150.00 → match 200 shares
  E has 400 @ $150.05 → price > limit → STOP
  100 shares unmatched → add to buy side of order book at $150.00
```

---

## Step 5: Order Processing Pipeline

```
Trader → [Order Gateway]
  1. Parse order (FIX protocol or REST/JSON)
  2. Auth check (is this trader allowed?)
  3. Pre-trade risk check:
     - Sufficient balance/margin?
     - Daily limit not exceeded?
     - Circuit breaker: price movement > 10% → reject
  4. Assign sequence number (monotonically increasing, per symbol)
  5. Publish to Kafka with key = symbol_id
     (same symbol → same partition → strictly ordered)

Kafka → [Matching Engine]
  Single consumer per symbol reads from Kafka partition.
  Processes in strict sequence number order.
  Updates in-memory order book.
  Emits trade events back to Kafka (executed trades).

Matching Engine → [Trade Publisher]
  Publish trades to:
    - Trader confirmation queue (your order filled)
    - Market data feed (price update broadcast)
    - Audit log (immutable record)
```

---

## Step 6: Market Data Feed

```
Every trade changes the last traded price and order book depth.
This must be broadcast to all subscribers in real-time.

Options:
  1. WebSocket (web clients): each subscriber has WebSocket connection
     Push updates on every trade
     
  2. UDP Multicast (professional trading systems):
     Single packet → received by all subscribers simultaneously
     Much more efficient than N TCP connections
     Used by NYSE, Nasdaq, CME

  3. Fan-out queue (SQS/Kafka):
     Trade event → topic → N consumer groups each push to their subscribers

Market data types:
  Level 1: Last price, bid/ask (for retail)
  Level 2: Full order book depth (for professionals, costs more)

Throttling:
  Some securities trade 1000x/sec.
  Aggregate updates over 100ms windows → 10 updates/sec per symbol.
  Reduces data volume without losing meaningful information.
```

---

## Step 7: Data Model

### orders table (append-only)
```sql
CREATE TABLE orders (
  id              BIGINT PRIMARY KEY,
  symbol          VARCHAR(10) NOT NULL,
  trader_id       BIGINT NOT NULL,
  side            ENUM('buy','sell'),
  order_type      ENUM('market','limit','stop'),
  quantity        INT NOT NULL,
  limit_price     DECIMAL(12,4),              -- NULL for market orders
  status          ENUM('pending','partial','filled','cancelled','rejected'),
  filled_qty      INT DEFAULT 0,
  avg_fill_price  DECIMAL(12,4),
  sequence_num    BIGINT,                     -- exchange-assigned, monotonic
  created_at      DATETIME(6)                 -- microsecond precision
);
```

### trades table (immutable execution records)
```sql
CREATE TABLE trades (
  id              BIGINT PRIMARY KEY,
  symbol          VARCHAR(10) NOT NULL,
  buy_order_id    BIGINT NOT NULL,
  sell_order_id   BIGINT NOT NULL,
  price           DECIMAL(12,4) NOT NULL,
  quantity        INT NOT NULL,
  executed_at     DATETIME(6)
);
-- Never deleted; regulatory requirement to keep forever
```

---

## Step 8: High Availability for Matching Engine

```
Matching engine is stateful (in-memory order book).
How to handle failure?

Primary-Backup with Synchronous Replication:
  Primary processes orders, backup receives every event.
  On primary failure → backup has complete state.
  Failover in < 10ms (pre-warmed backup).

State reconstruction:
  On cold start, replay events from Kafka (event sourcing).
  Order book rebuilt by replaying all orders from beginning of day.
  Checkpoint (snapshot) taken at market open.

Geographic redundancy:
  Two datacenters in same metro area (< 1ms between them).
  Active-passive: secondary datacenter is hot standby.
  In disaster: fail over to secondary (< 5 min).
```

---

## Step 9: Circuit Breakers & Safeguards

```
Exchange-level circuit breakers:
  If S&P 500 drops 7% → 15-minute trading halt (SEC rule)
  If individual stock moves ±10% in 5 minutes → pause that symbol

Pre-trade risk:
  Per-trader position limits
  Automated order cancel on disconnect (FIX: Cancel-on-Disconnect)
  
Market manipulation detection:
  Pattern matching on order flow
  Flag spoofing (large order placed + cancelled immediately)
  Report to compliance system
```

---

## Interview Tips

- Key insight: "Matching engine must be single-threaded per symbol, in-memory. No DB in the hot path."
- Ordering guarantee: "Kafka partition per symbol ensures strict ordering — no out-of-order matching."
- This is NOT a web CRUD app: "Forget HTTP latency. The matching engine uses memory-mapped structures for nanosecond performance."
- Reliability: "Event sourcing — every order is stored before processing. On failure, replay events to rebuild state."

## Common Follow-up Questions
- "How to prevent front-running?" → Sequence numbers assigned at gateway; FIFO within same price level
- "How to handle after-hours trading?" → Same system with different hours configuration; reduced liquidity
- "How to support options/futures?" → Different matching rules (different option contracts); separate matching engine instances
- "How to do the settlement?" → T+2 settlement (trades settle 2 days later); DTCC handles clearing in US
