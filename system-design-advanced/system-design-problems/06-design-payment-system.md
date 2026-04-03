---
slug: design-payment-system
title: Design a Payment System
readTime: 35 min
orderIndex: 6
premium: false
---

# Design a Payment System

Design a payment processing platform like Stripe or PayPal — capable of processing transactions reliably, safely, and at scale. Payments are the highest-stakes area of system design: money cannot be lost, created, or counted twice.

---

## Step 1: Requirements

### Functional Requirements
- Accept payments from customers via credit/debit card
- Process refunds
- Support multiple currencies
- Webhooks to notify merchants of payment status
- Payment history and reporting

### Non-Functional Requirements
- Correctness above all else: money must never be lost or double-charged
- Idempotency: retrying a payment request never creates a duplicate charge
- Exactly-once execution: a payment either succeeds once or fails — never succeeds twice
- High availability: 99.99%
- Auditability: every money movement must be traceable
- PCI DSS compliance (card data handling)

---

## Step 2: Back-of-Envelope Estimation

```
Stripe processes ~250M transactions/day
= 250,000,000 / 86,400 ≈ 2,900 TPS average
Peak: ~5,000 TPS

Each transaction record: ~1 KB
Storage: 2,900 × 1 KB = 2.9 MB/sec = 250 GB/day
10 years: ~900 TB of transaction history

Latency target: < 2 seconds for payment completion (user-facing)
```

---

> **Pause and think:** A customer clicks "Pay" and their network times out. They click "Pay" again. Without special handling, what happens? How do you prevent charging them twice?

## Step 3: Key Concept — Idempotency

This is the most critical concept in payment systems. A payment request may be retried due to:
- Network timeouts (client doesn't know if payment succeeded)
- Client crashes after sending the request
- Retry logic in the SDK

Without idempotency: retry = double charge. Catastrophic.

```
Without idempotency:
  Client sends: POST /charges { amount: $100 }
  Network times out → client doesn't know if charge happened
  Client retries: POST /charges { amount: $100 }
  → Two charges of $100 created!

With idempotency key:
  Client sends: POST /charges
    Idempotency-Key: client-uuid-abc123
    { amount: $100 }

  Server: check if "client-uuid-abc123" already processed
    No → process charge, store result keyed by idempotency key
    Return result

  Client retries (same idempotency key):
  Server: check if "client-uuid-abc123" already processed
    Yes → return stored result without processing again
    Return same result
```

**Idempotency key storage:**
```sql
CREATE TABLE idempotency_keys (
  key          VARCHAR(255) PRIMARY KEY,
  request_hash VARCHAR(64),   -- hash of request body to detect changed requests
  response     JSONB,         -- stored response
  created_at   TIMESTAMP DEFAULT NOW(),
  expires_at   TIMESTAMP      -- purge after 24 hours
);
```

---

## Step 4: Data Model — Double-Entry Bookkeeping

Every payment is recorded as two accounting entries. This is how banks have worked for 500 years.

```
Principle: Every debit has a corresponding credit.
Total sum of all entries must always equal zero.

Payment of $100 from Alice to Merchant:

Account               | Debit    | Credit
───────────────────────┼──────────┼────────
Alice's wallet         | $100.00  |
Merchant's wallet      |          | $100.00

Every dollar debited from one account must be credited to another.

Refund of $100 to Alice:
Account               | Debit    | Credit
───────────────────────┼──────────┼────────
Merchant's wallet      | $100.00  |
Alice's wallet         |          | $100.00
```

```sql
-- Ledger entries (immutable — never updated or deleted)
CREATE TABLE ledger_entries (
  id             UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  transaction_id UUID NOT NULL,
  account_id     UUID NOT NULL,
  amount         DECIMAL(20, 8) NOT NULL,  -- positive for credit, negative for debit
  currency       VARCHAR(3) NOT NULL,      -- ISO 4217: USD, EUR, GBP
  entry_type     ENUM('DEBIT', 'CREDIT') NOT NULL,
  created_at     TIMESTAMP DEFAULT NOW()
);

-- Account balances (denormalized for fast reads)
CREATE TABLE accounts (
  id             UUID PRIMARY KEY,
  user_id        UUID NOT NULL,
  currency       VARCHAR(3) NOT NULL,
  balance        DECIMAL(20, 8) NOT NULL DEFAULT 0,
  updated_at     TIMESTAMP DEFAULT NOW()
);

-- Payment records
CREATE TABLE payments (
  id               UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  idempotency_key  VARCHAR(255) UNIQUE,
  amount           DECIMAL(20, 8) NOT NULL,
  currency         VARCHAR(3) NOT NULL,
  status           ENUM('pending', 'processing', 'succeeded', 'failed', 'refunded'),
  payer_id         UUID,
  payee_id         UUID,
  payment_method_id UUID,
  created_at       TIMESTAMP DEFAULT NOW(),
  updated_at       TIMESTAMP DEFAULT NOW()
);
```

---

## Step 5: Architecture

```
Customer                    Merchant
    │                           │
    │  Payment form              │
    ▼                           ▼
┌──────────────────────────────────────────────────────┐
│                   API Gateway                        │
│          Auth, Rate Limiting, TLS Termination        │
└──────────────────────┬───────────────────────────────┘
                       │
            ┌──────────▼──────────┐
            │   Payment Service   │
            │                     │
            │ 1. Validate request │
            │ 2. Check idempotency│
            │ 3. Create payment   │
            │ 4. Write ledger     │
            └──────────┬──────────┘
                       │
          ┌────────────┼────────────┐
          ▼            ▼            ▼
  ┌──────────┐  ┌──────────┐  ┌──────────┐
  │  PSP     │  │  Fraud   │  │  Ledger  │
  │ Service  │  │ Detection│  │  Service │
  │(Stripe,  │  │          │  │          │
  │ Braintree│  └──────────┘  └──────────┘
  └──────────┘
          │
          ▼
  ┌──────────────────┐
  │  Webhook Service │  → notify merchants of payment events
  └──────────────────┘
```

---

## Step 6: Payment Flow

```
Step 1: Customer submits payment form
  Browser → POST /v1/payment_intents
    { amount: 1000, currency: "usd", payment_method: "pm_abc" }
    Idempotency-Key: "client-uuid-xyz"

Step 2: Payment Service
  a. Check idempotency key — not seen before → proceed
  b. Create payment record with status="pending"
  c. Run fraud checks (async, < 50ms heuristic check)
  d. Call PSP (Stripe/Braintree) to authorize card:
     POST stripe.com/v1/charges { amount, currency, source }

Step 3: PSP processes
  PSP → Card Network (Visa/Mastercard) → Issuing Bank
  Issuing Bank approves/declines → response to PSP → response to us
  (This takes 0.5–2 seconds)

Step 4: Handle PSP response
  Success:
    a. Update payment status → "succeeded"
    b. Write ledger entries (debit payer, credit payee) in DB transaction
    c. Store idempotency response
    d. Publish event: "payment.succeeded" → Kafka
    e. Return 200 with payment result

  Failure:
    a. Update payment status → "failed"
    b. Return error with decline reason

Step 5: Async side effects (from Kafka events)
  → Email confirmation to customer
  → Webhook to merchant
  → Analytics event
  → Balance update triggers
```

---

> **Pause and think:** Your server calls Stripe, the card is charged successfully, but your server crashes before saving the result to your database. The customer retries. How do you avoid charging them again?

## Step 7: Handling Failures

### Failure Between PSP Success and DB Write

```
Worst case scenario:
  1. We call PSP → card charged successfully
  2. PSP returns 200 OK
  3. Our server crashes before writing to DB
  4. Customer sees error, retries with same idempotency key
  5. We call PSP again → card charged again!

Solution: Reconciliation
  PSP provides a unique charge ID for every transaction
  We store the PSP charge ID immediately before charging
  On retry: check "do we already have a PSP charge for this idempotency key?"
    Yes → return that result (don't charge again)
    No  → charge (first attempt)
```

### Idempotent Retry Logic

```
Payment attempt:
  1. Write to payments table: status=PENDING, idempotency_key=X, psp_charge_id=NULL
  2. Call PSP
  3. PSP succeeds → write psp_charge_id, update status=SUCCEEDED

On retry with same idempotency key:
  1. Check: idempotency_key=X already in DB with status=SUCCEEDED
  2. Return existing result (never call PSP again)
```

### Compensating Transactions

When part of a multi-step payment fails, undo completed steps:

```
Process order:
  Step 1: Reserve inventory     ✅
  Step 2: Charge payment        ✅
  Step 3: Create shipping label ❌ (API down)

Compensate:
  Undo Step 2: Issue refund via PSP
  Undo Step 1: Release inventory reservation

Write compensation events to ledger → full audit trail
```

---

## Step 8: Reconciliation

At end of day, verify our records match PSP records:

```
Our DB:  100 charges totaling $50,000
PSP DB:  101 charges totaling $50,100

Discrepancy:
  → Find the charge in PSP not in our DB
  → Was it a double-charge from a crash? → Refund via PSP
  → Was it a valid charge we missed? → Add to our DB

Reconciliation process:
  1. Download PSP transaction report (CSV/API)
  2. Compare with internal ledger
  3. Flag discrepancies for manual review
  4. Automated: reconcile within tolerance; alert beyond it
```

---

## Step 9: Webhooks for Merchants

Merchants need asynchronous notification of payment events:

```
1. Payment succeeds
2. Publish to Kafka: "payment.succeeded" { payment_id, amount, merchant_id }
3. Webhook Service consumes event:
   a. Look up merchant's webhook URL
   b. POST to webhook URL with HMAC signature:
      POST https://merchant.com/webhooks
      X-Algoroq-Signature: sha256=abc...
      { "event": "payment.succeeded", "payment_id": "...", "amount": 100 }

4. If merchant webhook fails (non-200 response):
   → Retry with exponential backoff: 1min, 5min, 30min, 2h, 6h, 24h
   → After 72 hours of failure: mark as permanently failed, alert merchant

5. Merchant webhook is idempotent (MUST handle duplicate delivery)
```

---

## Step 10: Scaling

### Database
- Use PostgreSQL with strong ACID guarantees
- Shard by merchant_id — each shard handles all payments for a set of merchants
- Read replicas for reporting queries

### PSP Integration
- Retry with exponential backoff on PSP network failures
- Circuit breaker: stop calling PSP if error rate is too high
- Fallback PSP: if primary PSP is down, route to secondary

---

## Key Takeaways

1. **Idempotency is the most important concept** — use idempotency keys on every write operation
2. **Double-entry bookkeeping ensures money is never created or destroyed** — every debit has a credit
3. **Store PSP charge ID before calling PSP** — enables reconciliation after crashes
4. **Reconciliation catches discrepancies** — your records vs PSP records, run daily
5. **Webhooks must be retried with exponential backoff** — merchants have downtime too
6. **Compensating transactions undo failed multi-step flows** — the Saga pattern in practice
7. **Never store raw card numbers** — PCI DSS requires tokenization; let the PSP handle card data

---
