# Design: Payment System (Stripe / PayPal)

## Problem Statement
Design a payment processing system that handles money movement between buyers and sellers reliably, with zero tolerance for financial errors.

---

## Step 1: Clarify Requirements

### Questions to Ask
- "Payment types: card payments, bank transfers, or both?" → Card payments (Visa/MC) + wallets
- "Support for recurring payments (subscriptions)?" → Yes
- "Multi-currency?" → Yes
- "Refunds and disputes?" → Yes
- "Fraud detection?" → Mention as component
- "Scale?" → 1 million transactions/day

### Functional Requirements
- Accept payment: charge a card / debit account
- Record payment ledger (double-entry bookkeeping)
- Refunds
- Recurring billing
- Transaction history per user/merchant
- Webhook notifications to merchants on payment events

### Non-Functional Requirements
- ACID transactions: money must never be created or destroyed (exactly-once)
- Idempotency: same payment request processed exactly once
- High availability (99.999% — payment downtime = direct revenue loss)
- Audit trail: every action logged permanently
- Regulatory compliance: PCI DSS (card data), AML, KYC

---

## Step 2: Capacity Estimation

```
1 million transactions/day
  = 11.6 TPS average
  = 100 TPS peak (holidays, flash sales)

(Much lower QPS than social apps — correctness > throughput)

Storage:
  Transaction record = 1 KB
  1M/day × 1 KB = 1 GB/day
  5-year = 1.8 TB (manageable in one PostgreSQL cluster)
  
Ledger entries: 2 entries per transaction (double-entry)
  2M/day, ~3.6 TB in 5 years
```

---

## Step 3: High-Level Architecture

```
[Merchant / Mobile App]
        |
        | POST /charges
        ↓
[API Gateway]
  (rate limiting, TLS, auth)
        |
[Payment Service]
  - Idempotency check
  - Validate request
  - Create payment intent
        |
        +──────────────────────────────────────────+
        |                                          |
[Risk Engine]                          [Card Processor]
(Fraud score < threshold)              (Visa / Mastercard)
        |                                          |
        | approve / decline                        | auth code
        ↓                                          ↓
[Ledger Service]                       [Settlement Service]
(double-entry bookkeeping)             (batch T+1 settlement)
        |
[Notification Service]
(webhooks to merchant, email to user)
```

---

## Step 4: Idempotency — The Most Critical Feature

```
Problem:
  Merchant charges user $100.
  Network times out.
  Merchant retries.
  User charged $200.

Solution: Idempotency Keys

Client sends unique key with every request:
  POST /charges
  Idempotency-Key: "order_123_attempt_1"
  Body: { "amount": 10000, "currency": "USD", "card": "..." }

Server logic:
  1. Check if idempotency_key was seen before
  2. If yes → return original response (don't charge again)
  3. If no → process charge → store (key → response) in DB
  4. Return response

Idempotency key storage:
  Table: idempotency_keys (key, response, created_at, expires_at)
  Keep for 24–72 hours.
  
Atomicity:
  INSERT INTO idempotency_keys (key, ...) ON CONFLICT DO NOTHING
  → If two concurrent requests with same key arrive: only one inserts.
  → Other gets existing record, returns same response.
```

---

## Step 5: Double-Entry Bookkeeping

Every financial transaction must follow double-entry accounting:
**Every debit has a matching credit. Total must always balance.**

```
Payment of $100 from User A to Merchant B:

Debit  | User A Receivable    | $100
Credit | Merchant B Payable   | $100

This records the obligation. On settlement:
Debit  | Merchant B Payable   | $100
Credit | Bank Settlement      | $100

Balance check: SUM(all entries) = 0 (always)
Any imbalance → bug → alert immediately
```

### Ledger Table
```sql
CREATE TABLE ledger_entries (
  id              BIGINT PRIMARY KEY,
  transaction_id  VARCHAR(50) NOT NULL,
  account_id      BIGINT NOT NULL,
  amount          BIGINT NOT NULL,     -- in cents, NEVER float (floating point errors!)
  currency        CHAR(3) NOT NULL,
  entry_type      ENUM('debit','credit'),
  created_at      DATETIME NOT NULL,
  description     VARCHAR(255)
);
-- Append-only (never update/delete ledger entries)
-- Index: (account_id, created_at) for balance calculation
```

**Never use FLOAT for money.** Always use integers (cents) or DECIMAL(19,4).

---

## Step 6: Payment State Machine

```
CREATED → PROCESSING → AUTHORIZED → CAPTURED → SETTLED
                     ↘ DECLINED
                               ↘ FAILED
                                         ↓
                              REFUND_PENDING → REFUNDED

State transitions are atomic DB updates.
Invalid transitions rejected (e.g., can't CAPTURE a DECLINED payment).

Implementation:
  UPDATE payments 
  SET status = 'AUTHORIZED', authorized_at = NOW()
  WHERE id = :payment_id AND status = 'PROCESSING'
  
  IF rows_affected == 0 → state transition rejected (concurrent update)
```

---

## Step 7: Card Processing Integration

```
Stripe-like flow:
1. Client creates Payment Intent (server-side)
2. Client-side SDK collects card details
3. Card data tokenised client-side (never hits your server) → PCI DSS compliance
4. Token sent to your server → forward to payment processor (Visa/MC)
5. Authorization response received
6. Capture (charge) the authorized amount

Two-step auth + capture:
  Auth: "Reserve $100 on card" (no money moved yet)
  Capture: "Actually charge the $100" (money moved)
  
  Useful for: 
    - Hotels (reserve on checkin, charge on checkout)
    - Marketplaces (auth on order, capture on shipment)
```

---

## Step 8: Reliability Patterns

### Retry with Exactly-Once Semantics
```
Payment calls to card processor may time out.
Retry policy:
  1. Generate unique idempotency key for each charge attempt
  2. Pass key to card processor
  3. If timeout → retry with SAME key
  4. Processor deduplicates using key → no double charge
```

### Reconciliation
```
Background job runs every hour:
  Compare your DB with card processor statement.
  Find discrepancies: payments in processor but not in your DB, or vice versa.
  Alert on discrepancies > $1.
  Automatic fix for known patterns; manual review for unknowns.
```

### Circuit Breaker for Payment Processor
```
If payment processor returns errors:
  > 5% error rate in 1 minute → open circuit breaker
  → Reject new payments immediately (fail fast, don't queue)
  → Alert on-call engineer
  → After 5 minutes → half-open → test with 1% of traffic
```

---

## Step 9: Database Choice

```
PostgreSQL (NOT NoSQL) for core payment data:
  - ACID transactions required (money cannot be partially recorded)
  - Complex queries (reconciliation, balance calculation)
  - Foreign key constraints
  - Row-level locking for concurrent updates

Sharding strategy:
  Shard by merchant_id (all transactions from one merchant on same shard)
  → Merchant-level queries don't cross shards
  → User transactions may span shards (acceptable for payment history)

Read replicas for:
  - Reporting / analytics queries
  - Dashboard reads
  → Frees primary for write-critical operations
```

---

## Step 10: API Design

```
POST /api/v1/payment_intents
Idempotency-Key: unique-client-key
Body: { "amount": 1000, "currency": "usd", "customer_id": "cus_123" }
Response: { "id": "pi_123", "status": "requires_payment_method", "client_secret": "..." }

POST /api/v1/payment_intents/{id}/capture
Response: { "status": "succeeded", "amount_captured": 1000 }

POST /api/v1/refunds
Body: { "payment_intent_id": "pi_123", "amount": 500 }
Response: { "id": "re_123", "status": "pending" }

GET /api/v1/payments/{id}
GET /api/v1/customers/{id}/payments?page=1
```

---

## Interview Tips

- Open with: "Payments are the highest-stakes system — correctness over performance. ACID, idempotency, and audit trail are non-negotiable."
- Idempotency: "Every payment API must accept an idempotency key — this is how Stripe prevents double charges."
- Money as integers: "Never use float for currency. Always use integer cents or DECIMAL(19,4)."
- Double-entry ledger: "This is how every financial system keeps its books balanced — debit always matches credit."

## Common Follow-up Questions
- "How to handle multi-currency?" → Store amount + currency. Convert using exchange rate at transaction time. Never convert in ledger.
- "How to handle fraud?" → Risk score from ML model at payment intake. Block high-risk. Step-up auth (3DS) for medium risk.
- "What is PCI DSS compliance?" → Standard for handling card data. Key rule: never store raw card numbers on your servers. Use tokenisation.
- "How to do batch payouts to sellers?" → Aggregate daily, batch transfer via ACH/wire, mark settled in ledger
