---
name: financial-systems-patterns
description: Expert financial systems design patterns — invoke when building payment processing, ledgers, accounting, money arithmetic, reconciliation, fraud detection, or any system where correctness of monetary values is critical.
---

# Financial Systems Design Patterns

## Double-Entry Bookkeeping

Every financial transaction touches at least two accounts: one account is debited and another is credited. The invariant is strict — debits must always equal credits. This is the foundation of all modern accounting systems and the reason financial systems can detect corruption or bugs: if debits != credits, something is wrong.

### Chart of Accounts

```
ASSETS       (debit increases, credit decreases)
LIABILITIES  (credit increases, debit decreases)
EQUITY       (credit increases, debit decreases)
REVENUE      (credit increases, debit decreases)
EXPENSES     (debit increases, credit decreases)
```

The accounting equation holds at all times: `Assets = Liabilities + Equity`

### Ledger Entry Schema

```sql
CREATE TABLE ledger_entries (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    transaction_id  UUID NOT NULL,          -- groups related entries
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    account_id      UUID NOT NULL REFERENCES accounts(id),
    entry_type      VARCHAR(6) NOT NULL CHECK (entry_type IN ('DEBIT', 'CREDIT')),
    amount_minor    BIGINT NOT NULL CHECK (amount_minor > 0),  -- always positive
    currency        CHAR(3) NOT NULL,        -- ISO 4217 code (USD, EUR, GBP)
    description     TEXT NOT NULL,
    reference       VARCHAR(255),            -- external reference (payment ID, invoice #)
    created_by      UUID NOT NULL            -- actor (user, service, job)
);

-- Every transaction must balance
CREATE TABLE transactions (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    description TEXT NOT NULL,
    metadata    JSONB
);
```

### Why Minor Units (Cents, Pence)

Never store money as floating-point. `0.1 + 0.2 = 0.30000000000000004` in IEEE 754. Rounding errors compound over millions of transactions.

```
$12.50  -> stored as  1250 (cents)
£99.99  -> stored as  9999 (pence)
JPY 500 -> stored as   500 (yen have no minor units; exponent = 0)
BHD 1.5 -> stored as  1500 (Bahraini Dinar has 3 decimal places)
```

ISO 4217 defines the exponent (number of decimal places) per currency. Always look it up — do not assume 2.

### Immutability: Never Update or Delete

Ledger entries are immutable. If you made an error, create a reversing entry:

```sql
-- Original (wrong)
INSERT INTO ledger_entries (transaction_id, account_id, entry_type, amount_minor, currency, description)
VALUES ('txn-001', 'acc-revenue', 'CREDIT', 10000, 'USD', 'Invoice #42 payment');

-- Reversal
INSERT INTO ledger_entries (transaction_id, account_id, entry_type, amount_minor, currency, description)
VALUES ('txn-002', 'acc-revenue', 'DEBIT', 10000, 'USD', 'REVERSAL: Invoice #42 payment (txn-001)');

-- Correction
INSERT INTO ledger_entries (transaction_id, account_id, entry_type, amount_minor, currency, description)
VALUES ('txn-002', 'acc-revenue', 'CREDIT', 9500, 'USD', 'Invoice #42 payment (corrected)');
```

### Balance Calculation

```sql
-- Current balance for an account
SELECT
    SUM(CASE WHEN entry_type = 'CREDIT' THEN amount_minor ELSE 0 END) -
    SUM(CASE WHEN entry_type = 'DEBIT'  THEN amount_minor ELSE 0 END) AS balance_minor
FROM ledger_entries
WHERE account_id = :account_id
  AND currency = :currency;

-- Validate transaction balance (debits = credits)
SELECT transaction_id,
       SUM(CASE WHEN entry_type = 'DEBIT'  THEN amount_minor ELSE 0 END) AS total_debits,
       SUM(CASE WHEN entry_type = 'CREDIT' THEN amount_minor ELSE 0 END) AS total_credits
FROM ledger_entries
WHERE transaction_id = :transaction_id
GROUP BY transaction_id
HAVING SUM(CASE WHEN entry_type = 'DEBIT'  THEN amount_minor ELSE 0 END) !=
       SUM(CASE WHEN entry_type = 'CREDIT' THEN amount_minor ELSE 0 END);
-- Returns rows only if unbalanced — should be empty
```

---

## Money as Value Object

### Anti-Patterns

```java
// WRONG: float
float price = 9.99f;
float tax = price * 0.1f;  // 0.99900001 -- not 0.999

// WRONG: double
double total = 0.1 + 0.2;  // 0.30000000000000004

// WRONG: storing as decimal string without controlled parsing
String amount = "12.50";  // what's the currency? what are the rounding rules?
```

### Java/Kotlin Money Value Object

```kotlin
import java.math.BigDecimal
import java.math.RoundingMode
import java.util.Currency

data class Money(
    val amountMinor: Long,       // amount in minor units (cents, pence, etc.)
    val currency: Currency
) {
    companion object {
        fun of(amount: BigDecimal, currency: Currency): Money {
            val scale = currency.defaultFractionDigits
            val scaled = amount.setScale(scale, RoundingMode.HALF_EVEN)
            val minor = scaled.movePointRight(scale).toLong()
            return Money(minor, currency)
        }

        fun ofMinor(amountMinor: Long, currencyCode: String): Money =
            Money(amountMinor, Currency.getInstance(currencyCode))
    }

    fun toDecimal(): BigDecimal {
        val scale = currency.defaultFractionDigits
        return BigDecimal(amountMinor).movePointLeft(scale)
    }

    operator fun plus(other: Money): Money {
        require(currency == other.currency) {
            "Cannot add ${currency.currencyCode} and ${other.currency.currencyCode}"
        }
        return Money(amountMinor + other.amountMinor, currency)
    }

    operator fun minus(other: Money): Money {
        require(currency == other.currency) {
            "Cannot subtract ${other.currency.currencyCode} from ${currency.currencyCode}"
        }
        return Money(amountMinor - other.amountMinor, currency)
    }

    operator fun times(factor: BigDecimal): Money {
        val scale = currency.defaultFractionDigits
        val result = toDecimal().multiply(factor).setScale(scale, RoundingMode.HALF_EVEN)
        return of(result, currency)
    }

    fun isPositive() = amountMinor > 0
    fun isNegative() = amountMinor < 0
    fun isZero()     = amountMinor == 0L

    override fun toString() = "${currency.currencyCode} ${toDecimal()}"
}

// Usage
val price  = Money.ofMinor(999, "USD")    // $9.99
val tax    = price * BigDecimal("0.10")   // $1.00 (HALF_EVEN rounding)
val total  = price + tax                  // $10.99
```

### Rounding Modes

| Mode       | Use Case                               | Example (2.5 -> int) |
|------------|----------------------------------------|----------------------|
| HALF_UP    | Retail prices, consumer-facing amounts | 3                    |
| HALF_EVEN  | Banking (Banker's Rounding)            | 2 (rounds to even)   |
| FLOOR      | Interest calculations favoring bank    | 2                    |
| CEILING    | Interest calculations favoring customer| 3                    |

Define the rounding mode per operation in your domain spec. Do not leave it to language defaults.

### Currency Conversion

```kotlin
data class FxConversion(
    val fromAmount: Money,
    val toAmount: Money,
    val rate: BigDecimal,       // exact rate used
    val rateSource: String,     // "ECB", "Fixer.io", "internal"
    val rateTimestamp: Instant, // when the rate was fetched
    val appliedAt: Instant      // when the conversion happened
)

// NEVER do implicit conversion
// ALWAYS record the rate and timestamp for audit
fun convert(amount: Money, toCurrency: Currency, rate: BigDecimal): FxConversion {
    val converted = amount * rate
    // adjust to target currency minor units
    return FxConversion(
        fromAmount = amount,
        toAmount = Money.of(converted.toDecimal(), toCurrency),
        rate = rate,
        rateSource = "ECB",
        rateTimestamp = Instant.now(),
        appliedAt = Instant.now()
    )
}
```

---

## Idempotency in Payments

### The Problem

Networks fail. Clients retry. Without idempotency:
```
Client sends: charge $100
Network fails after server processes but before response
Client retries: charge $100 again
Result: customer charged $200
```

### Idempotency Key Pattern

```
Client                          Server
  |                               |
  |-- POST /payments              |
  |   Idempotency-Key: uuid-123  |
  |   body: {amount: 1000}       |
  |                               |-- Store key + process payment
  |<-- 200 {payment_id: p-abc}   |-- Store response
  |                               |
  |-- POST /payments (retry)      |
  |   Idempotency-Key: uuid-123  |
  |                               |-- Key exists! Return stored response
  |<-- 200 {payment_id: p-abc}   |  (payment NOT processed again)
```

### Database Implementation

```sql
CREATE TABLE idempotency_keys (
    key         VARCHAR(255) PRIMARY KEY,
    response    JSONB NOT NULL,
    status_code INT NOT NULL,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    expires_at  TIMESTAMPTZ NOT NULL DEFAULT NOW() + INTERVAL '48 hours'
);

CREATE INDEX idx_idempotency_keys_expires_at ON idempotency_keys(expires_at);
```

```kotlin
@Transactional
fun processPayment(request: PaymentRequest, idempotencyKey: String): PaymentResponse {
    // Check for existing result first
    val existing = idempotencyRepo.findByKey(idempotencyKey)
    if (existing != null) {
        return existing.response  // Return cached result — do NOT process again
    }

    // Process payment
    val response = paymentGateway.charge(request)

    // Store result (INSERT ... ON CONFLICT DO NOTHING is race-safe)
    idempotencyRepo.storeResult(
        key = idempotencyKey,
        response = response,
        statusCode = 200
    )

    return response
}
```

```sql
-- Race-safe insertion (concurrent retries won't double-insert)
INSERT INTO idempotency_keys (key, response, status_code)
VALUES (:key, :response, :status_code)
ON CONFLICT (key) DO NOTHING;

-- If DO NOTHING fired, fetch the winner's stored response
SELECT response, status_code FROM idempotency_keys WHERE key = :key;
```

### Key Requirements for Callers

- Client generates a UUID per payment attempt (not per retry of same attempt)
- Same idempotency key = same request; different payment = different key
- Key must be sent in header: `Idempotency-Key: <uuid>`
- Keys expire after 24-48 hours; documented in API spec
- After expiry, the same key can be reused (risk: caller must manage key lifecycle)

---

## Payment State Machine

```
                    INITIATED
                        |
                        v
                     PENDING
                        |
              +---------+---------+
              |                   |
              v                   v
         PROCESSING           CANCELLED
              |
     +--------+--------+
     |                 |
     v                 v
 COMPLETED           FAILED
     |
     v
 REFUND_REQUESTED
     |
     v
  REFUNDED
```

### State Transition Table

| From State        | To State          | Trigger                          | Guard                         |
|-------------------|-------------------|----------------------------------|-------------------------------|
| INITIATED         | PENDING           | Payment submitted                | Amount > 0, valid account     |
| PENDING           | PROCESSING        | Gateway accepted                 | Gateway response OK           |
| PENDING           | CANCELLED         | User/system cancels              | Before PROCESSING             |
| PROCESSING        | COMPLETED         | Gateway confirms success         | Gateway confirms              |
| PROCESSING        | FAILED            | Gateway decline / timeout        | Gateway declines              |
| COMPLETED         | REFUND_REQUESTED  | Refund requested                 | Within refund window          |
| REFUND_REQUESTED  | REFUNDED          | Refund processed by gateway      | Gateway confirms refund       |

### Schema

```sql
CREATE TABLE payments (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    status          VARCHAR(20) NOT NULL DEFAULT 'INITIATED',
    amount_minor    BIGINT NOT NULL,
    currency        CHAR(3) NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE payment_status_history (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    payment_id      UUID NOT NULL REFERENCES payments(id),
    from_status     VARCHAR(20),            -- NULL for initial state
    to_status       VARCHAR(20) NOT NULL,
    actor           VARCHAR(255) NOT NULL,  -- user ID, service name, or 'system'
    reason          TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

### Kotlin State Machine Implementation

```kotlin
enum class PaymentStatus {
    INITIATED, PENDING, PROCESSING, COMPLETED, FAILED, CANCELLED,
    REFUND_REQUESTED, REFUNDED
}

val ALLOWED_TRANSITIONS = mapOf(
    PaymentStatus.INITIATED        to setOf(PaymentStatus.PENDING),
    PaymentStatus.PENDING          to setOf(PaymentStatus.PROCESSING, PaymentStatus.CANCELLED),
    PaymentStatus.PROCESSING       to setOf(PaymentStatus.COMPLETED, PaymentStatus.FAILED),
    PaymentStatus.COMPLETED        to setOf(PaymentStatus.REFUND_REQUESTED),
    PaymentStatus.REFUND_REQUESTED to setOf(PaymentStatus.REFUNDED),
    PaymentStatus.FAILED           to emptySet(),
    PaymentStatus.CANCELLED        to emptySet(),
    PaymentStatus.REFUNDED         to emptySet()
)

fun transition(payment: Payment, toStatus: PaymentStatus, actor: String, reason: String? = null) {
    val allowed = ALLOWED_TRANSITIONS[payment.status] ?: emptySet()
    check(toStatus in allowed) {
        "Illegal transition: ${payment.status} -> $toStatus for payment ${payment.id}"
    }
    val fromStatus = payment.status
    payment.status = toStatus
    payment.updatedAt = Instant.now()
    paymentRepo.save(payment)
    historyRepo.save(PaymentStatusHistory(
        paymentId = payment.id,
        fromStatus = fromStatus,
        toStatus = toStatus,
        actor = actor,
        reason = reason
    ))
}
```

---

## Reconciliation

### Architecture

```
Internal System          Payment Processor (Stripe / Adyen)
     |                            |
     |  [webhooks - real-time] ---|
     |                            |
     |  [settlement report - T+1]-|
     |                            |
Reconciliation Job
     |
     +-- Load internal payments for period
     +-- Load processor report for period
     +-- Match by processor reference ID
     +-- Flag discrepancies:
         - Missing in internal (ghost payment)
         - Missing in processor (orphan record)
         - Amount mismatch
         - Status mismatch
     +-- Generate reconciliation report
     +-- Alert on-call if discrepancy count exceeds threshold
```

### Idempotent Webhook Handling

```kotlin
@Transactional
fun handleWebhook(event: WebhookEvent) {
    // Deduplicate by event ID (processor provides this)
    if (webhookEventRepo.existsByExternalId(event.id)) {
        log.info("Duplicate webhook event ${event.id} — skipping")
        return  // 200 OK response; do not reprocess
    }

    // Process the event
    processEvent(event)

    // Mark as processed
    webhookEventRepo.save(WebhookEvent(
        externalId = event.id,
        type = event.type,
        processedAt = Instant.now()
    ))
}
```

### Polling Fallback

```
Webhook arrives      -> Process immediately
                     -> Also schedule verification check in 5 min

Verification check   -> Re-query processor for payment status
                     -> If status diverges from internal state, reconcile
                     -> This catches missed webhooks

Nightly batch        -> Full reconciliation against settlement file
                     -> Mandatory regardless of webhook reliability
```

---

## Audit Trail

```sql
CREATE TABLE audit_log (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    entity_type VARCHAR(50) NOT NULL,   -- 'payment', 'account', 'user'
    entity_id   UUID NOT NULL,
    action      VARCHAR(50) NOT NULL,   -- 'STATUS_CHANGE', 'AMOUNT_ADJUSTED'
    actor_type  VARCHAR(20) NOT NULL,   -- 'user', 'service', 'scheduled_job'
    actor_id    VARCHAR(255) NOT NULL,
    from_value  JSONB,                  -- previous state (null for creates)
    to_value    JSONB NOT NULL,         -- new state
    ip_address  INET,
    user_agent  TEXT,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Append-only: no UPDATE, no DELETE
-- Separate from operational tables: own schema or database
-- Indexed for lookup by entity
CREATE INDEX idx_audit_log_entity ON audit_log(entity_type, entity_id, created_at);
```

Regulatory requirements:
- PCI-DSS: retain audit logs for 12 months (3 months immediately available)
- SOC 2: changes to financial data must be logged with actor and timestamp
- GDPR: audit logs may contain personal data — factor into retention and deletion policies

---

## Currency Handling

### Multi-Currency Storage

```sql
-- Store raw amounts with their currency; never implicitly convert
CREATE TABLE account_balances (
    account_id      UUID NOT NULL,
    currency        CHAR(3) NOT NULL,
    balance_minor   BIGINT NOT NULL DEFAULT 0,
    PRIMARY KEY (account_id, currency)
);

-- FX rate log for audit
CREATE TABLE fx_rates (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    from_ccy    CHAR(3) NOT NULL,
    to_ccy      CHAR(3) NOT NULL,
    rate        NUMERIC(20, 10) NOT NULL,
    source      VARCHAR(50) NOT NULL,  -- 'ECB', 'Fixer.io'
    fetched_at  TIMESTAMPTZ NOT NULL,
    valid_until TIMESTAMPTZ
);
```

### Settlement Flow

```
Customer pays EUR 100
    |
    v
Stored as EUR 10000 (minor units) in internal ledger
    |
    v
Merchant settlement (weekly)
    |
    v
Convert EUR -> USD at rate from fx_rates table
    |
    +-- Record FX conversion with rate_id foreign key
    +-- Debit EUR account, credit USD account
    |
    v
Initiate USD bank transfer to merchant
```

---

## Fraud Detection Patterns

### Velocity Checks

```kotlin
fun checkVelocity(userId: String, amount: Money): RiskSignal {
    val oneHourAgo = Instant.now().minusSeconds(3600)
    val recentCount = txnRepo.countByUserSince(userId, oneHourAgo)
    val recentTotal = txnRepo.sumByUserSince(userId, oneHourAgo)

    return when {
        recentCount > 10        -> RiskSignal.HIGH("velocity: $recentCount txns/hour")
        recentTotal > Money.ofMinor(500_00, "USD") -> RiskSignal.MEDIUM("velocity: high volume")
        else                    -> RiskSignal.LOW
    }
}
```

### Risk Scoring Pipeline

```
Transaction arrives
        |
        v
+-------+--------+
| Velocity check |  -> HIGH risk score if N txns/hour exceeded
+-------+--------+
        |
        v
+-------+--------+
| Amount anomaly |  -> score if amount > 3 std dev from user avg
+-------+--------+
        |
        v
+-------+--------+
| Geo anomaly    |  -> score if country differs from last 30 days
+-------+--------+
        |
        v
+-------+--------+
| Device check   |  -> score if new device + amount > threshold
+-------+--------+
        |
        v
   Aggregate score
        |
   +----+----+
   |         |
 LOW       HIGH
   |         |
 Allow     Block / 3DS challenge / manual review
```

### Risk Score Table

```sql
CREATE TABLE transaction_risk_assessments (
    transaction_id      UUID PRIMARY KEY,
    velocity_score      INT NOT NULL DEFAULT 0,   -- 0-100
    amount_score        INT NOT NULL DEFAULT 0,
    geo_score           INT NOT NULL DEFAULT 0,
    device_score        INT NOT NULL DEFAULT 0,
    aggregate_score     INT NOT NULL DEFAULT 0,
    decision            VARCHAR(20) NOT NULL,      -- ALLOW, BLOCK, CHALLENGE
    assessed_at         TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

---

## Anti-Patterns Checklist

| Anti-Pattern                                    | Correct Approach                              |
|-------------------------------------------------|-----------------------------------------------|
| `double amount = 9.99`                          | `Long amountMinor = 999` (cents)              |
| Updating ledger entries to fix mistakes         | Reversing entries + correcting entries        |
| Currency conversion without storing the rate    | Store rate, source, timestamp with conversion |
| Payment API without idempotency key             | Require `Idempotency-Key` header              |
| Skipping state transitions (PENDING -> COMPLETED)| Enforce state machine; reject illegal jumps  |
| Relying solely on webhooks for status           | Webhook + polling + nightly reconciliation    |
| Audit log in the same table as operational data | Separate append-only audit log table          |
| Implicit currency in money fields               | Always store ISO 4217 currency code alongside |
| Processing the same webhook event twice         | Deduplicate by external event ID              |
| Assuming all currencies have 2 decimal places   | Look up ISO 4217 exponent per currency        |

## Production Checklist

- [ ] All money values stored as integer minor units (BIGINT)
- [ ] Currency stored as ISO 4217 code alongside every monetary field
- [ ] Ledger entries are immutable; errors corrected with reversals
- [ ] Every transaction validated: debits == credits
- [ ] Payment API requires idempotency key on all mutating operations
- [ ] Idempotency keys stored with TTL (24-48h)
- [ ] Payment state machine enforced in code; illegal transitions rejected
- [ ] Every state transition logged to payment_status_history
- [ ] Webhook handler deduplicates by external event ID
- [ ] Nightly reconciliation against processor settlement file
- [ ] Audit log is append-only and separate from operational tables
- [ ] FX rates stored with source, timestamp, and rate_id on conversion records
- [ ] Rounding mode documented and consistent per operation type
- [ ] Fraud risk scoring applied before processing high-value transactions
