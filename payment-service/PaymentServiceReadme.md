Absolutely. Here’s a clean **interview-ready system design document** for the Payment Service we discussed. I’ve also corrected a couple of things from our discussion where the design can be made more production-grade.

# Payment Service — System Design

## 1. Problem Statement

Design a **highly available payment service** for an e-commerce platform such as Amazon/Flipkart.

The service should support:

* Credit/debit card and UPI payments
* Creating a payment
* Tracking payment status
* Preventing duplicate payments/charges
* Handling synchronous and asynchronous responses from payment gateways
* Handling duplicate webhook events
* Persisting payment records reliably
* Allowing clients to query payment status
* Supporting high availability and scalability

### Out of scope

* Product/catalog management
* Inventory management
* Order fulfillment
* Refunds
* Delivery tracking

---

# 2. Functional Requirements

### 1. Create payment

Client should be able to initiate a payment:

```http
POST /payments
```

Example:

```json
{
  "orderId": "ORD123",
  "userId": "USER456",
  "paymentMethod": "UPI",
  "amount": 4999
}
```

The service creates a payment record and sends the payment request to an external payment gateway.

---

### 2. Track payment status

Possible states:

```text
CREATED
    ↓
PROCESSING
    ↓
SUCCESS
    ↓
FAILED
```

We can also have:

```text
PROCESSING → PENDING
```

when the gateway hasn't given us a final response yet.

A better production design is to explicitly define this as a **payment state machine**, rather than allowing arbitrary status transitions.

---

### 3. Prevent duplicate payments

The same payment request may arrive multiple times because of:

* User double-clicking Pay
* Client retry
* Network timeout
* Load balancer retry
* Mobile app retry

Therefore, the API should support an **idempotency key**.

Example:

```http
POST /payments
Idempotency-Key: abc-123
```

The same idempotency key should result in the same payment operation rather than creating another payment.

> This is preferable to relying only on `userId + productId`.

---

### 4. Receive asynchronous gateway updates

The external gateway may process the payment asynchronously.

It calls:

```http
POST /payments/webhook
```

Example:

```json
{
  "merchantPaymentId": "MP12345",
  "gatewayTransactionId": "GW98765",
  "status": "SUCCESS"
}
```

We use `merchantPaymentId` to identify our payment.

---

### 5. Query payment status

Client can query:

```http
GET /payments/{paymentId}
```

Example response:

```json
{
  "paymentId": "PAY123",
  "status": "PROCESSING"
}
```

---

# 3. High-Level Architecture

```text
                 ┌─────────────────┐
                 │   Mobile/Web    │
                 │      Client     │
                 └────────┬────────┘
                          │
                          ▼
                 ┌─────────────────┐
                 │   API Gateway   │
                 └────────┬────────┘
                          │
                          ▼
                 ┌─────────────────┐
                 │  Load Balancer  │
                 └────────┬────────┘
                          │
             ┌────────────┼────────────┐
             ▼            ▼            ▼
       ┌──────────┐ ┌──────────┐ ┌──────────┐
       │ Payment  │ │ Payment  │ │ Payment  │
       │ Service  │ │ Service  │ │ Service  │
       │    #1    │ │    #2    │ │    #3    │
       └────┬─────┘ └────┬─────┘ └────┬─────┘
            │             │            │
            └─────────────┼────────────┘
                          │
                ┌─────────┴─────────┐
                │                   │
                ▼                   ▼
          ┌───────────┐       ┌───────────┐
          │   Redis   │       │ PostgreSQL │
          │           │       │            │
          └───────────┘       └───────────┘
                │
                │
                ▼
       ┌────────────────────┐
       │ External Payment   │
       │ Gateway / PSP      │
       └─────────┬──────────┘
                 │
                 │ Webhook
                 ▼
       ┌────────────────────┐
       │ Payment Service    │
       │ /webhook endpoint  │
       └────────────────────┘
```

---

# 4. Payment Flow

Let's walk through the most important flow.

### Step 1 — User clicks Pay

Client sends:

```http
POST /payments
```

with:

```text
orderId
userId
amount
paymentMethod
idempotencyKey
```

---

### Step 2 — Payment service checks idempotency

Payment service checks whether this idempotency key has already been processed.

If yes:

```text
Return existing payment
```

If no:

```text
Continue
```

This protects against duplicate client requests.

---

### Step 3 — Create payment record

Create:

```text
paymentId = PAY123
merchantPaymentId = MP456
status = PROCESSING
```

in PostgreSQL.

---

### Step 4 — Call payment gateway

Payment service sends:

```text
merchantPaymentId
amount
payment method
callback/webhook URL
```

to the external gateway.

The important part is that **we generate the merchant/payment reference on our side**.

---

### Step 5 — Gateway processes payment

There are two possibilities.

### Synchronous

Gateway immediately responds:

```text
SUCCESS
```

or

```text
FAILED
```

Payment service updates PostgreSQL.

### Asynchronous

Gateway initially says:

```text
PROCESSING
```

and later sends:

```text
POST /payments/webhook
```

---

# 5. Webhook Flow

Suppose gateway sends:

```json
{
  "merchantPaymentId": "MP456",
  "gatewayTransactionId": "GW789",
  "status": "SUCCESS"
}
```

Payment service:

1. Validates webhook authenticity
2. Finds payment using `merchantPaymentId`
3. Checks current payment state
4. Updates payment state
5. Returns success to gateway

Example:

```text
PROCESSING → SUCCESS
```

---

# 6. Duplicate Webhooks

This is an important interview topic.

Payment gateways can retry webhooks.

For example:

```text
Webhook #1 → SUCCESS
Webhook #2 → SUCCESS
Webhook #3 → SUCCESS
```

We must not process these as three separate payments.

We can make webhook processing **idempotent**.

For example, maintain:

```text
gatewayTransactionId
```

with a unique constraint.

Or maintain:

```text
webhookEventId
```

with a unique constraint.

Then:

```text
if event already processed:
      return 200
else:
      process event
```

Also, our payment state machine should prevent invalid transitions such as:

```text
SUCCESS → SUCCESS
SUCCESS → FAILED
```

unless explicitly allowed by the business rules.

---

# 7. Database Design

A simplified `payments` table:

| Column                 | Purpose                    |
| ---------------------- | -------------------------- |
| payment_id             | Our internal payment ID    |
| order_id               | Associated order           |
| user_id                | Customer                   |
| amount                 | Payment amount             |
| currency               | Currency                   |
| payment_method         | CARD / UPI                 |
| status                 | Current payment state      |
| idempotency_key        | Prevent duplicate requests |
| merchant_payment_id    | Reference sent to gateway  |
| gateway_transaction_id | Gateway's transaction ID   |
| created_at             | Creation time              |
| updated_at             | Last update                |

Important indexes/constraints:

```text
PRIMARY KEY(payment_id)

UNIQUE(idempotency_key)

UNIQUE(merchant_payment_id)

UNIQUE(gateway_transaction_id)
```

Depending on the business semantics, `idempotency_key` may be scoped by user/client rather than globally unique.

---

# 8. Why PostgreSQL?

Your choice of PostgreSQL was reasonable.

We want:

* Strong consistency
* ACID transactions
* Durable payment records
* Constraints
* Unique indexes
* Reliable updates
* Auditability

For payment systems, correctness is more important than simply throwing a NoSQL database at the problem.

---

# 9. Where Redis Fits

Redis can be useful for:

* Fast idempotency checks
* Short-lived locks
* Rate limiting
* Caching payment status
* Reducing database load

But there's an important correction to our earlier discussion:

**Redis should not be the ultimate source of truth for whether a payment happened.**

PostgreSQL/payment ledger should remain authoritative.

For example:

```text
Client
   ↓
Payment Service
   ↓
Redis → fast idempotency check
   ↓
PostgreSQL → source of truth
```

And Redis expiration should **not** mean:

> "After 10 minutes the user can safely make another payment."

Payment lifecycle can take longer than the cache TTL.

---

# 10. Availability

To make Payment Service highly available:

```text
                 Load Balancer
                /      |      \
               /       |       \
          Payment   Payment   Payment
          Service   Service   Service
             #1        #2        #3
```

Payment service instances should be **stateless**.

Therefore any instance can process any request.

We can horizontally scale based on:

* Requests/sec
* CPU
* Memory
* Latency
* Queue depth, if queues are introduced

---

# 11. Database High Availability

PostgreSQL itself becomes a potential bottleneck/SPOF.

Production architecture could use:

```text
             Payment Services
                    │
                    ▼
             DB Connection Pool
                    │
             ┌──────┴──────┐
             │             │
          Primary       Replica
             │
          Writes       Reads
```

With managed PostgreSQL, we can have:

* Multi-AZ deployment
* Automatic failover
* Read replicas
* Automated backups
* Point-in-time recovery

For payment writes, we need the primary/authoritative writer.

---

# 12. Gateway Failure

This is one of the most important parts.

Suppose:

```text
Payment Service
       │
       ▼
Payment Gateway
       │
       X timeout
```

We **cannot immediately assume the payment failed**.

Why?

Because the gateway might have processed the payment but our request timed out.

For example:

```text
Payment Service → Gateway
                  ↓
              Payment SUCCESS
                  ↓
              Response lost
                  X
Payment Service → TIMEOUT
```

If we retry blindly, we might charge the customer twice.

Therefore:

> **Timeout ≠ payment failure.**

This is a critical payment-system principle.

We need:

* Idempotency at gateway
* Unique merchant payment ID
* Retry policy
* Reconciliation
* Webhooks
* Payment status inquiry API

---

# 13. Reconciliation

A production payment system should have a reconciliation process.

For example:

```text
PROCESSING payments
        ↓
Reconciliation Job
        ↓
Query gateway
        ↓
Compare states
        ↓
Correct our database
```

Suppose our DB says:

```text
PAY123 → PROCESSING
```

but gateway says:

```text
PAY123 → SUCCESS
```

Reconciliation updates:

```text
PROCESSING → SUCCESS
```

This protects us from lost webhooks and network failures.

---

# 14. Client Notification

For the UI, we discussed three options.

### Option 1 — Polling

```text
GET /payments/PAY123
```

every few seconds.

Simple and reliable.

### Option 2 — SSE

Client opens:

```text
GET /payments/PAY123/events
```

Server sends an event when status changes.

Good for one-way server → client updates.

### Option 3 — WebSocket

Useful when we need bidirectional real-time communication.

For this use case, **polling or SSE is generally sufficient**.

---

# 15. Important Non-Functional Requirements

I'd summarize them in an interview like this:

### 1. Correctness / Consistency

Most important.

We must avoid:

```text
Double charge
Incorrect payment state
Lost payment
```

---

### 2. Availability

Payment initiation should remain available despite individual service-instance failures.

---

### 3. Durability

Once a payment record is committed, it shouldn't disappear because of a server/database failure.

---

### 4. Reliability

The system should correctly handle:

* Gateway timeout
* Network failure
* Duplicate request
* Duplicate webhook
* Lost webhook
* Service restart

---

### 5. Scalability

Payment services should scale horizontally.

```text
100 TPS
   ↓
500 TPS
   ↓
5000 TPS
```

We should be able to add instances rather than redesigning the entire system.

---

### 6. Security

Payment systems require strong security:

* TLS
* Authentication/authorization
* Webhook signature verification
* Encryption at rest
* Encryption in transit
* Don't store raw card numbers/CVV
* PCI-DSS considerations
* Audit logs

---

### 7. Observability

Track:

```text
payment_created
payment_processing
payment_success
payment_failed
gateway_timeout
webhook_received
webhook_duplicate
```

And metrics such as:

```text
Payment success rate
Gateway latency
Payment failure rate
Webhook failure rate
Duplicate request rate
Pending payments
```

Use distributed tracing with a correlation/payment ID.

---

# 16. Rough Scale Estimate

Our interview assumption:

```text
10M DAU
10% purchase
= 1M payment attempts/day
```

Average:

```text
1,000,000 / 86,400
≈ 11.6 TPS
```

If we design for a 10× peak:

```text
≈ 116 TPS
```

We can round that to:

```text
~120 TPS peak
```

For a massive sale, we could provision considerably higher capacity.

### Important interview lesson

Don't say:

> "One EC2 instance can handle 1 million TPS."

Capacity depends heavily on:

* Application
* CPU
* memory
* database
* network
* request complexity
* downstream dependencies

Instead say:

> "I'll initially assume ~100–200 TPS per application instance and validate that assumption through load testing."

That's much stronger in an interview.

---

# 17. Most Important Interview Concepts From This Problem

If you want to revisit this problem later, these are the **key concepts I'd want you to remember**:

```text
Payment Service
      │
      ├── Idempotency
      │
      ├── Payment State Machine
      │
      ├── PostgreSQL
      │
      ├── Redis
      │
      ├── Gateway Integration
      │
      ├── Webhooks
      │
      ├── Duplicate Webhook Handling
      │
      ├── Gateway Timeout Handling
      │
      ├── Reconciliation
      │
      ├── HA / Horizontal Scaling
      │
      ├── Observability
      │
      └── Security
```

And the **single most important principle**:

> **Never equate a gateway timeout with payment failure.**

The gateway may have charged the customer even though your service never received the response. That's why **idempotency + unique payment reference + webhook + reconciliation** are so important.

---

## How I'd expect you to present this in a future interview

You don't need to dump all of this immediately.

A good senior-level progression is:

**Requirements → APIs → High-level architecture → Data model → Payment flow → Idempotency → Gateway failures → Webhooks → Scalability/HA → Reliability/reconciliation → Security/observability → Capacity estimation.**

And honestly, **this is the area I'd like you to practice more**: your architectural thinking during today's mock was good, but you sometimes reached the right solution by describing it conversationally rather than naming the underlying design pattern. Getting comfortable with terms like **idempotency, state machine, reconciliation, retry semantics, and eventual consistency** will make your answers much sharper.
