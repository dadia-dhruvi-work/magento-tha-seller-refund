# Part 2A – Seller Return & Refund Design

## 1. Architecture approach

I propose a database-backed transactional outbox architecture for the Seller Return & Refund subsystem.

Magento remains responsible for refund validation, financial calculation, local persistence and lifecycle management. ERP remains the system of record for the seller credit note.

The Admin controller should remain a thin presentation boundary. Business orchestration belongs in application services, while ERP communication is handled asynchronously by an outbox consumer.

The proposed architecture uses existing Magento Open Source capabilities, MariaDB and cron/CLI consumers. No AMQP broker is required.

## 2. End-to-end refund flow

1. CS operator opens an eligible delivered seller order.
2. Magento loads seller-order lines and previously refunded quantities.
3. Operator selects products and refund quantities.
4. Magento calculates refundable subtotal, shipping, tax and grand total server-side.
5. Operator reviews and explicitly confirms the calculated amount.
6. Refund application service starts a database transaction and acquires the required order-level lock.
7. Remaining refundable quantities are recalculated and validated against current data.
8. Magento persists the refund header, line snapshots, audit event and ERP outbox event atomically.
9. The database transaction commits, and the Admin receives a saved/pending response.
10. An outbox consumer submits Create Refund to ERP using the stable `refund_no` idempotency key.
11. ERP returns `erp_refund_id` and the pending credit-note status.
12. Finance performs the actual bank transfer offline.
13. CS operator registers the transaction number and transaction date.
14. Magento persists cash-refund completion and creates the next durable integration event.
15. ERP status is read and verified as `refund-pending`.
16. Magento submits Confirm Refund using `erp_refund_id` as the idempotency key.
17. ERP confirmation and settlement adjustment export acknowledgements are recorded independently.
18. The refund is marked fully settled once the required acknowledgements are received.
19. Customer-facing surfaces and downstream systems use the persisted refund snapshot.

## 3. Sequence diagram

```mermaid
sequenceDiagram
    autonumber
    actor CS as CS Operator
    participant UI as Magento Admin
    participant S as Refund Service
    participant DB as MariaDB
    participant O as Outbox Consumer
    participant ERP as ERP API
    actor F as Finance
    participant X as Settlement Export
    participant C as Customer Channels

    CS->>UI: Open seller order
    UI->>S: Load refundable quantities
    S->>DB: Read order and refund history
    DB-->>S: Current quantities
    S-->>UI: Refundable lines and amounts

    CS->>UI: Select quantities
    UI->>S: Calculate refund
    S-->>UI: Server-calculated total
    CS->>UI: Confirm submission

    UI->>S: Submit refund
    S->>DB: Begin transaction and lock order
    S->>DB: Revalidate quantities and calculate
    S->>DB: Save refund, items, event and outbox
    S->>DB: Commit
    S-->>UI: Saved; ERP notification pending

    O->>DB: Claim outbox event
    O->>ERP: Create Refund (refund_no)

    alt ERP accepts
        ERP-->>O: erp_refund_id and pending
        O->>DB: Record ERP ID and success
    else Transient failure or timeout
        ERP-->>O: Error or ambiguous result
        O->>DB: Record attempt and schedule retry
    else Business rejection
        ERP-->>O: Definitive rejection
        O->>DB: Record terminal failure
    end

    F->>F: Perform offline bank transfer
    CS->>UI: Register transaction number and date
    UI->>S: Register cash refund
    S->>DB: Persist cash details and outbox event

    O->>ERP: Read Refund Status
    alt ERP is refund-pending
        ERP-->>O: Pending
        O->>ERP: Confirm Refund (erp_refund_id)
        ERP-->>O: Confirmed
        O->>DB: Record confirmation
    else ERP unavailable or unexpected status
        ERP-->>O: Error or unexpected state
        O->>DB: Record and reconcile
    end

    O->>X: Export settlement adjustment
    X-->>O: Acknowledgement
    O->>DB: Record export acknowledgement
    Note over DB: Complete only after required acknowledgements

    O->>C: Trigger customer updates
```

## 4. Key technical decisions

### 4.1 Refund calculation and concurrency

Refundable quantity:

`Remaining quantity = Ordered quantity - Previously refunded quantity`

The requested quantity must never exceed the remaining quantity.

Use an order-level database lock with `SELECT ... FOR UPDATE` to serialize financial quantity calculations. Re-read the current refund history after acquiring the lock.

Use the existing `version` column for optimistic concurrency on refund lifecycle updates.

### 4.2 Transactional outbox

Refund persistence and outbox-event creation must happen in the same database transaction.

The consumer should:
- Claim eligible events safely.
- Use bounded retries and exponential backoff.
- Record every attempt.
- Handle stale or ambiguous outcomes through reconciliation.
- Support authorised manual replay.
- Avoid holding database transactions open during HTTP calls.

### 4.3 Idempotency

- Create Refund: stable `refund_no`.
- Confirm Refund: stable `erp_refund_id`.
- Individual HTTP attempts: separate correlation identifiers.

Retries must not create new logical refund identities.

### 4.4 ERP error handling

| Result | Handling |
|---|---|
| 2xx | Record success |
| Definitive business rejection | Terminal failure |
| 404 after creation | Bounded eventual-consistency retry |
| 409 | Reconcile existing operation |
| 429 | Backoff and retry |
| 5xx | Bounded retry |
| Timeout | Treat as ambiguous and reconcile |

### 4.5 Tax and shipping

Tax is calculated in Magento using the original order tax rate.

The business tax codes are:
- `999`: Tax exempt
- `010`: 10%
- `008`: 8% reduced rate

The ERP payload uses `TAX_EXCLUDED`, with tax-exclusive amounts and calculated tax amounts.

Full refunds must include only the remaining refundable shipping allocated to seller items.

### 4.6 Financial snapshot

The persisted refund snapshot is the canonical source for:
- Reissued receipt PDF.
- Customer order details.
- Order history.
- Refund confirmation email.
- Downstream core-system export.

Original order and invoice totals must remain unchanged.

## 5. Phased delivery plan

| Epic | Scope | Dependencies | Acceptance gate |
|---|---|---|---|
| E1 – Domain and data foundation | Refund entities, schema, lifecycle, audit | None | Schema and lifecycle validated |
| E2 – Refund calculation and integrity | Eligibility, tax, shipping, quantity and concurrency | E1 | Financial and concurrency tests pass |
| E3 – Admin experience | Refund UI, worklist, confirmation, ACL | E1, E2 | Admin workflow and permission tests pass |
| E4 – ERP integration | Outbox, Create, Read, Confirm, retry and reconciliation | E1, E2 | Failure and idempotency tests pass |
| E5 – Finance and settlement | Cash registration and settlement adjustment export | E4 | Settlement reconciliation passes |
| E6 – Customer presentation | PDF, order details, history and email | E1, E5 | Consistent financial figures across channels |
| E7 – Production readiness | Performance, security, monitoring and recovery | E1–E6 | Load, security and recovery gates pass |

Epics may run in parallel where their dependencies and interface contracts are agreed.

## 6. Important assumptions and clarification points

1. FRD §4 describes Read Refund Status as operator-triggered, whereas §9.2 describes a scheduled batch. Confirm the intended trigger. Proposed approach: durable asynchronous processing initiated by the operator action, with scheduled recovery.
2. FRD §2 describes `refund_no` as store-allocated, while §9.5 describes ERP allocation. Proposed approach: allocate it in Magento before Create Refund to provide a stable idempotency key.
3. FRD §9.3 and §9.4 describe separate ERP-confirmation and settlement-export completion points. Track both independently and define the terminal business state explicitly.
4. Confirm the exact definition of an eligible seller order and the treatment of mixed seller/first-party orders.

## 7. Acceptance criteria

- Refund quantities cannot exceed remaining refundable quantities.
- Concurrent submissions cannot create excess refunds.
- Repeated submissions cannot create duplicate financial actions.
- Refund persistence and outbox events are atomic.
- External failures do not lose committed refunds.
- Retries and replay are authorised, bounded and auditable.
- ERP and settlement acknowledgements remain traceable.
- Customer-facing surfaces use consistent financial snapshots.
- Worklist and settlement operations meet the stated performance requirements.