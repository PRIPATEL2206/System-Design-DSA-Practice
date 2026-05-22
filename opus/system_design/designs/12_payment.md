# Design: Payment System (Stripe/Razorpay)

**Frequency:** High | **Difficulty:** ⭐⭐⭐⭐ | **Companies:** Amazon, Google, Stripe, PayPal

---

## 1. Requirements

### Functional
- Process payments (credit card, UPI, wallet)
- Handle refunds
- Payment status tracking
- Webhook notifications to merchants
- Idempotent payment processing (no double-charge!)
- Multi-currency support
- Reconciliation

### Non-Functional
- **Scale:** 1M transactions/day
- **Consistency:** STRONG (cannot lose money or double-charge)
- **Availability:** 99.999% (payment must work)
- **Security:** PCI-DSS compliant, encrypted at rest and in transit
- **Latency:** < 3 seconds end-to-end

---

## 2. Architecture

```
┌──────────┐        ┌─────────────────────────────────────────────────────┐
│ Merchant │        │              Payment System                          │
│  Server  │        │                                                      │
└────┬─────┘        │  ┌───────────┐   ┌────────────┐   ┌─────────────┐  │
     │              │  │Payment API│──▶│  Payment   │──▶│  Payment    │  │
     │ POST /pay    │  │ Gateway   │   │  Service   │   │  Executor   │  │
     ├─────────────▶│  └───────────┘   └──────┬─────┘   └──────┬──────┘  │
     │              │                         │                 │         │
     │              │                    ┌────▼────┐      ┌─────▼──────┐  │
     │              │                    │  Ledger │      │ PSP Router │  │
     │              │                    │  (DB)   │      │(Visa/MC/UPI)│ │
     │              │                    └─────────┘      └─────┬──────┘  │
     │              │                                           │         │
     │              └───────────────────────────────────────────┼─────────┘
     │                                                          │
     │              ┌───────────────────────────────────────────▼─────────┐
     │              │       External Payment Processors                    │
     │              │  (Visa, Mastercard, UPI, PayPal, Banks)             │
     │              └─────────────────────────────────────────────────────┘
     │
     │ Webhook (payment_completed / failed)
     ◀──────────────────────────────────────
```

---

## 3. Payment Flow

```
Step-by-step:

1. Merchant calls: POST /v1/payments
   {
     amount: 9999,  // in smallest unit (paise/cents)
     currency: "INR",
     payment_method: "card",
     idempotency_key: "order_abc_123",  // CRITICAL
     metadata: { order_id: "abc" }
   }

2. Idempotency check:
   - Have we seen this idempotency_key before?
   - YES → return cached result (no re-processing!)
   - NO → proceed

3. Create payment record (status: CREATED)
   
4. Risk/Fraud check:
   - Velocity check (too many payments in short time?)
   - Amount threshold check
   - Geo anomaly detection
   - If suspicious → status: REQUIRES_ACTION (3D Secure)

5. Route to Payment Service Provider (PSP):
   - Card → Visa/Mastercard network
   - UPI → NPCI
   - Wallet → Provider API
   
6. PSP processes → returns success/failure

7. Update payment record:
   - Success → status: COMPLETED, credit merchant account
   - Failure → status: FAILED, return error to merchant

8. Send webhook to merchant: payment.completed / payment.failed

9. Reconciliation (end of day):
   - Compare our records with PSP settlement files
   - Flag discrepancies for manual review
```

---

## 4. Data Model (Double-Entry Ledger)

```
Every money movement has TWO entries (debit + credit):

Payment completed ($100 from buyer to merchant):
┌─────────────────────────────────────────────────────────────┐
│ Entry 1: DEBIT  buyer_account   -$100  (money leaves)      │
│ Entry 2: CREDIT merchant_account +$97  (money arrives)     │
│ Entry 3: CREDIT platform_fee     +$3   (our commission)    │
│                                                              │
│ SUM of all entries = 0 (ALWAYS! This is the invariant)     │
└─────────────────────────────────────────────────────────────┘

Tables:
─────────
payments:
  id, idempotency_key, amount, currency, status,
  merchant_id, payment_method, psp_reference,
  created_at, updated_at

ledger_entries:
  id, payment_id, account_id, 
  debit_amount, credit_amount,
  currency, created_at

accounts:
  id, owner_type (merchant/buyer/platform),
  balance, currency

Database: PostgreSQL with SERIALIZABLE isolation
  (Money MUST be consistent — no race conditions)
```

---

## 5. Idempotency (The #1 Priority)

```
Problem: Network timeout → client retries → payment processed TWICE!

Solution: Idempotency key per payment attempt

┌──────────────────────────────────────────────────────┐
│  Request 1: {idempotency_key: "abc", amount: 100}   │
│  → Process payment → Store result keyed by "abc"     │
│  → Return: {id: "pay_1", status: "completed"}       │
│                                                       │
│  Request 2: {idempotency_key: "abc", amount: 100}   │
│  → Lookup "abc" → Found! Return cached result       │
│  → Return: {id: "pay_1", status: "completed"}       │
│  → NO second charge!                                 │
└──────────────────────────────────────────────────────┘

Implementation:
  Table: idempotency_keys
    key VARCHAR UNIQUE, response JSONB, created_at, expires_at
  
  Check: INSERT ... ON CONFLICT (key) DO NOTHING
  If insert succeeded → process (first time)
  If conflict → return stored response (duplicate)
```

---

## 6. Handling Failures

```
Exactly-once semantics for money:

1. PSP call timeouts:
   - Do NOT retry blindly (might double-charge)
   - Query PSP: "what happened to payment X?"
   - If PSP says processed → mark as completed
   - If PSP says unknown → mark as unknown, alert ops

2. Our system crashes mid-processing:
   - Payment record in DB with status PROCESSING
   - Recovery worker scans for stuck PROCESSING payments
   - Reconciles with PSP → updates final status

3. State machine ensures valid transitions ONLY:
   CREATED → PROCESSING → COMPLETED
                        → FAILED
   COMPLETED → REFUND_INITIATED → REFUNDED
   
   Cannot go: COMPLETED → COMPLETED (double-process)
   Cannot go: FAILED → COMPLETED (without new attempt)
```

---

## 7. Refunds

```
POST /v1/payments/{payment_id}/refund
{ amount: 5000 }  // partial refund allowed

Flow:
1. Validate: payment is COMPLETED, refund amount ≤ original
2. Create refund record (status: INITIATED)
3. Call PSP refund API
4. On success: Create ledger entries (reverse the original)
5. Update balances (merchant debited, buyer credited)
6. Webhook: refund.completed

Timing: Refund may take 5-10 business days (bank processing)
```

---

## 8. Scaling & Reliability

```
Component          Strategy
────────────────────────────────────────────────────
Payment API        Stateless, horizontal scaling
Database           PostgreSQL with synchronous replication
                   (cannot lose payment data!)
PSP Integration    Circuit breaker + fallback PSP
                   (if Visa fails → try alternate processor)
Webhooks           Async queue (Kafka) + retry with exponential backoff
Reconciliation     Daily batch job comparing our DB vs PSP files
Monitoring         Alert if: success rate < 99%, latency > 3s,
                   reconciliation mismatch > 0.01%
```

---

## 9. Security

```
PCI-DSS Compliance:
- Never store raw card numbers (use PSP tokenization)
- Encrypt all data at rest (AES-256)
- TLS 1.3 for all communication
- Audit logs for every action
- Network segmentation (payment systems isolated)
- Regular penetration testing

Token Vault:
  Raw card: 4242424242424242
  Token: tok_abc123 (meaningless if stolen)
  Only PSP can de-tokenize
```
