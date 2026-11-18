# Part 2B -- Architecture Review and Target-State Extension

## 1. Purpose and review scope

This document reviews the inherited Seller Return & Refund architecture
and proposes a target-state extension for a reliable, maintainable
Magento 2 implementation.

The review focuses on separation of responsibilities, refund financial
integrity, concurrent submissions, ERP integration reliability,
lifecycle management, worklist and settlement performance, consistent
financial presentation, security and operational readiness.

**Scope note:** This is an architectural review of the supplied design
and its described implementation characteristics. Recommendations are
target-state proposals, not proof that every proposed component already
exists in the repository or has been implemented and load-tested.

## 2. Executive assessment

The existing Magento modular-monolith approach is a reasonable
foundation for this business capability. The design identifies
refund-specific persistence, ERP integration, refund events and a
defined business lifecycle.

The main concern is not the choice of Magento or the modular monolith.
It is the need to strengthen the boundaries and guarantees around
financial operations.

The highest-risk areas are:

1.  Preventing concurrent or duplicate refunds against the same seller
    order.
2.  Avoiding loss or duplication when ERP requests fail or time out.
3.  Ensuring that the Admin UI reports persisted server state rather
    than optimistic client-side success.
4.  Enforcing valid and auditable lifecycle transitions.
5.  Keeping financial figures consistent across PDF, order details,
    history, email and downstream exports.

I recommend retaining the modular monolith and improving
application-service boundaries, database transaction strategy and
asynchronous integration reliability. Introducing microservices or a
message broker is not justified solely by the stated assignment
requirements and volumes.

## 3. Current architecture assessment

The supplied architecture overview describes several areas that require
attention:

-   The Admin controller performs multiple responsibilities, including
    validation, calculation, persistence and ERP interaction.
-   ERP communication is performed synchronously as part of the operator
    workflow.
-   Retry and error-handling behaviour needs stronger classification and
    recovery semantics.
-   The UI uses frequent polling to observe refund status.
-   Worklist retrieval can perform per-row reads and does not adequately
    constrain the result set through pagination.
-   Settlement scans need a query and indexing strategy aligned with
    their filter and ordering patterns.
-   Different presentation surfaces can calculate or assemble financial
    figures independently.

These are architectural risks to validate against the actual
implementation and runtime behaviour. The recommendations below address
the risks without requiring a wholesale rewrite.

## 4. Risk register

Priority reflects potential business impact and blast radius, not
implementation complexity.

  -----------------------------------------------------------------------------
  Priority          Risk              Potential impact  Recommended response
  ----------------- ----------------- ----------------- -----------------------
  P0                Refund            Concurrent        Use an order-level
                    concurrency is    operators may     database lock,
                    not sufficiently  exceed refundable revalidate quantities
                    enforced          quantities or     inside the transaction
                                      create            and protect lifecycle
                                      conflicting       updates with version
                                      refunds           checks

  P0                ERP operations    ERP downtime or   Persist an outbox event
                    are coupled to    latency can block atomically and process
                    the Admin request operators and     ERP calls
                                      create ambiguous  asynchronously
                                      outcomes          

  P0                Client-side       Operators may act Return and display
                    optimistic        on a refund that  persisted server-side
                    success is not    was not durably   state; distinguish
                    authoritative     saved or accepted saved, pending and
                                                        completed

  P1                Controller owns   Difficult         Move business workflows
                    too much business testing, reuse    into application
                    orchestration     and maintenance   services

  P1                Lifecycle         Stale or invalid  Introduce a
                    transitions are   updates can       state-transition
                    not centrally     overwrite newer   component and
                    enforced          state             conditional/versioned
                                                        updates

  P1                ERP failures are  Incorrect         Classify definitive
                    not consistently  retries,          business errors,
                    classified        duplicate         transient failures and
                                      requests or       ambiguous outcomes
                                      refunds stuck in  separately
                                      intermediate      
                                      states            

  P1                Financial         PDF, email, order Persist a canonical
                    presentation      details and       financial snapshot and
                    logic is          exports can       share it across
                    duplicated        disagree          presentation surfaces

  P1                Event and ERP     Sensitive         Redact secrets and
                    payload retention information may   personal data, restrict
                    needs explicit    be retained or    access and define
                    controls          exposed           retention
                                      unnecessarily     

  P1                Settlement        Large scans may   Inspect query plans,
                    processing lacks  become slow or    add justified indexes
                    a proven query    delay             and process bounded
                    strategy          reconciliation    batches

  P2                Worklist          Admin response    Use server-side
                    retrieval risks   time and database pagination, filtering
                    N+1 queries and   load can degrade  and set-based queries
                    unbounded results                   

  P2                Frequent browser  Increased         Refresh on user actions
                    polling creates   requests and      and use a configurable,
                    avoidable load    database pressure less frequent refresh
                                                        where needed
  -----------------------------------------------------------------------------

## 5. Proposed target-state architecture

The target design retains Magento as a modular monolith and separates
responsibilities into clear layers.

``` mermaid
flowchart TB
    subgraph UI["1. Presentation Layer"]
        ADMIN["Magento Admin UI"]
        CUSTOMER["Customer-facing UI"]
        PDF["Refund PDF / Email"]
    end

    subgraph APP["2. Application Layer"]
        CTRL["Admin Controllers<br/>ACL and Request Validation"]
        SUBMIT["Refund Submission Service"]
        CALC["Refund Calculation Service"]
        CASH["Cash Registration Service"]
        COMPLETE["Refund Completion Service"]
    end

    subgraph DOMAIN["3. Domain Layer"]
        VALIDATOR["Refund Validator"]
        STATE["Refund State Machine"]
        SNAPSHOT["Financial Snapshot Provider"]
        REPO["Refund Repository"]
        EVENT["Refund Event Recorder"]
    end

    subgraph DB["4. Persistence Layer - MariaDB"]
        REFUND[("Refund Header")]
        ITEMS[("Refund Items")]
        EVENTS[("Refund Events / Audit")]
        OUTBOX[("Transactional Outbox")]
        SETTLEMENT[("Settlement Records")]
    end

    subgraph ASYNC["5. Asynchronous Processing Layer"]
        CRON["Magento Cron / CLI Scheduler"]
        DISPATCH["Outbox Dispatcher"]
        RETRY["Retry and Reconciliation Handler"]
    end

    subgraph INTEGRATION["6. External Integration Layer"]
        CLIENT["ERP Refund API Client"]
        ERP["ERP System<br/>Create / Read Status / Confirm"]
        EXPORT["Settlement Export Adapter"]
        CORE["Core System / Seller Ledger"]
    end

    ADMIN --> CTRL
    CUSTOMER --> SNAPSHOT
    PDF --> SNAPSHOT

    CTRL --> SUBMIT
    CTRL --> CALC
    CTRL --> CASH
    CTRL --> COMPLETE

    SUBMIT --> VALIDATOR
    SUBMIT --> STATE
    SUBMIT --> REPO
    CALC --> VALIDATOR
    CASH --> STATE
    COMPLETE --> STATE

    REPO --> REFUND
    REPO --> ITEMS
    EVENT --> EVENTS

    SUBMIT --> OUTBOX
    CASH --> OUTBOX
    COMPLETE --> OUTBOX
    EVENT --> OUTBOX

    REFUND --- ITEMS
    REFUND --- EVENTS
    OUTBOX --- EVENTS

    CRON --> DISPATCH
    DISPATCH --> OUTBOX
    DISPATCH --> CLIENT
    CLIENT <--> ERP

    DISPATCH --> RETRY
    RETRY --> OUTBOX

    DISPATCH --> EXPORT
    EXPORT --> SETTLEMENT
    EXPORT --> CORE

    SNAPSHOT --> REFUND
    SNAPSHOT --> ITEMS
```

### 5.1 Architecture components and responsibilities

#### Presentation layer

-   Provides Magento Admin interfaces for refund submission, worklist
    management and cash registration.
-   Customer-facing components display persisted refund information.
-   Controllers perform authentication/authorization checks, ACL
    validation, request validation and response generation.
-   Controllers should not own financial calculations, persistence
    orchestration or external ERP workflows.

#### Application layer

Dedicated application services coordinate use cases:

  -----------------------------------------------------------------------
  Component                           Responsibility
  ----------------------------------- -----------------------------------
  `RefundSubmissionService`           Coordinates validation, persistence
                                      and creation of the integration
                                      event

  `RefundCalculationService`          Calculates refundable subtotal,
                                      shipping, tax and grand total

  `CashRegistrationService`           Validates and records offline
                                      bank-transfer details

  `RefundCompletionService`           Coordinates status verification,
                                      confirmation and completion
                                      workflow
  -----------------------------------------------------------------------

These services should be invoked through Magento's dependency injection
configuration and should depend on interfaces where doing so improves
testability and replaceability.

#### Domain layer

  -----------------------------------------------------------------------
  Component                           Responsibility
  ----------------------------------- -----------------------------------
  `RefundValidator`                   Enforces eligibility, refundable
                                      quantity and business rules

  `RefundStateMachine`                Defines and validates permitted
                                      lifecycle transitions

  `FinancialSnapshotProvider`         Exposes consistent persisted
                                      financial figures

  `RefundRepository`                  Encapsulates refund persistence and
                                      retrieval

  `RefundEventRecorder`               Records auditable lifecycle events
  -----------------------------------------------------------------------

The domain layer should not depend on Admin controllers or browser
behaviour.

#### Persistence layer

MariaDB stores:

-   Refund header and lifecycle state.
-   Refund item and financial snapshots.
-   Refund events and audit history.
-   Outbox messages.
-   Settlement records and acknowledgements.

Refund data and the corresponding outbox event must be committed in the
same database transaction. This prevents a successfully committed refund
from being left without a durable record of the required downstream
work.

#### Asynchronous processing layer

Magento cron or CLI processes pending outbox messages.

The dispatcher should:

-   Claim eligible events safely so multiple workers do not process the
    same event concurrently.
-   Commit the claim before performing the external HTTP request.
-   Avoid holding database transactions open during network calls.
-   Record attempts and the outcome of each operation.
-   Apply bounded retries and backoff for transient failures.
-   Reconcile ambiguous outcomes before replaying an operation.
-   Support authorised manual replay with an audit record.

The design should provide at-least-once delivery with idempotent
processing. It should not claim exactly-once HTTP delivery.

#### External integration layer

-   `ErpRefundClient` encapsulates ERP Create Refund, Read Refund Status
    and Confirm Refund calls.
-   `SettlementExportAdapter` encapsulates communication with the core
    system or seller ledger.
-   External request/response handling, timeouts, error mapping and
    correlation identifiers belong in the integration layer.

## 6. Key architecture recommendations

### 6.1 Refund calculation and concurrency

Refundable quantity should be derived from persisted order and refund
data:

`Remaining quantity = Ordered quantity - Previously refunded quantity`

The requested quantity must not exceed the remaining quantity.

Recommended submission sequence:

1.  Begin a database transaction.
2.  Acquire an order-level lock using the appropriate order/seller-order
    row and `SELECT ... FOR UPDATE`.
3.  Re-read prior refunds and calculate current remaining quantities.
4.  Validate eligibility and requested quantities.
5.  Check for an existing active refund and a duplicate submission
    identity.
6.  Persist the refund header, item snapshots, audit event and outbox
    event.
7.  Commit the transaction.

The lock protects the financial quantity calculation from concurrent
submissions. A version check or conditional update should additionally
protect lifecycle transitions from stale writes.

The exact lock target and isolation behaviour must be verified against
the repository's schema and database engine.

### 6.2 Transactional outbox and ERP reliability

A refund must not be lost because an external ERP call fails after the
local refund has been committed.

Recommended behaviour:

-   Save the refund and outbox event atomically.
-   Dispatch the event asynchronously.
-   Use stable business idempotency keys for retries.
-   Record each attempt, response classification and correlation
    identifier.
-   Use bounded exponential backoff with jitter where appropriate.
-   Respect `Retry-After` for HTTP 429 responses.
-   Reconcile timeouts and other ambiguous outcomes before creating a
    new logical operation.
-   Route exhausted or definitive failures to an operator-visible
    failure state.

Do not hold a database transaction open while waiting for ERP.

### 6.3 Idempotency

Use stable business identifiers:

-   Create Refund: `refund_no`.
-   Confirm Refund: `erp_refund_id`, where this is the ERP operation's
    agreed idempotency key.

A network attempt may have a separate correlation ID for observability,
but retries of the same logical operation must retain the same business
idempotency identity.

Magento should also protect the local submission endpoint against
duplicate browser submissions and client retries.

The ERP contract must confirm that the proposed identifiers are actually
enforced as idempotency keys by ERP.

### 6.4 ERP error classification

  -----------------------------------------------------------------------
  Outcome                             Recommended handling
  ----------------------------------- -----------------------------------
  Successful 2xx                      Persist the returned result and
                                      operation state

  Definitive business validation      Record a terminal business failure;
  rejection                           do not retry unchanged input

  404 immediately after creation      Apply only a bounded retry if ERP
                                      documents eventual consistency

  409 conflict                        Reconcile the existing operation
                                      and compare its business identity

  429 rate limit                      Back off and respect `Retry-After`

  5xx                                 Bounded retry with backoff

  Timeout / connection reset          Treat as ambiguous; query or
                                      reconcile before replay
  -----------------------------------------------------------------------

Error handling must follow the actual ERP API contract. HTTP status
alone may not be enough to determine whether a financial operation was
committed.

### 6.5 Lifecycle and state management

Use a central state-transition policy rather than allowing controllers
or UI code to set arbitrary statuses.

The main refund lifecycle should be distinguished from the status of
individual integration operations.

For example:

-   Main refund lifecycle: `created`, `refund_pending`, `cash_refunded`,
    `erp_confirmed` (final business-state names must match the FRD).
-   Create operation: `pending`, `succeeded`, `failed`.
-   Status-check operation: `pending`, `succeeded`, `failed`.
-   Confirm operation: `pending`, `succeeded`, `failed`.
-   Settlement export: `pending`, `acknowledged`, `failed`.

The final settlement state should only be reached when all required
business acknowledgements have been recorded.

A timeout must not be interpreted automatically as a definitive
rejection.

### 6.6 Centralised financial snapshot

Persist a canonical refund financial snapshot containing, as applicable:

-   Original order and invoice figures.
-   Refunded item quantities.
-   Product subtotal.
-   Shipping refund.
-   Tax-exclusive amount.
-   Consumption tax and tax-rate groups.
-   Refund grand total.
-   Currency and refund date.
-   Per-item tax and price details.

The PDF, order details, order history, email and downstream export
should consume the same persisted values.

Original order and invoice figures must not be overwritten by refund
presentation logic. The snapshot should be immutable after the refund is
committed, except through an explicitly authorised and audited
correction process if the business permits corrections.

### 6.7 Worklist performance

Recommended changes:

-   Implement server-side pagination.
-   Apply filters and sorting at database level.
-   Fetch required seller, SKU and order summary information using
    set-based queries.
-   Avoid loading an order model separately for every refund row.
-   Add indexes only after inspecting the actual filter, join and sort
    patterns.
-   Measure query count and response time against representative data.

### 6.8 Settlement processing

Recommended changes:

-   Inspect the settlement query and its execution plan.
-   Add composite indexes aligned to actual status/date predicates and
    ordering.
-   Process records in bounded batches.
-   Avoid per-refund queries for the latest event; use an appropriate
    join, grouped query or persisted processing marker.
-   Make each exported adjustment idempotent.
-   Persist the export acknowledgement and correlation details.
-   Provide a reconciliation path for records with missing or ambiguous
    acknowledgements.

### 6.9 Admin UI refresh behaviour

The browser must not decide that a refund succeeded based on an
optimistic local update.

Recommended behaviour:

-   Return the persisted state after each server action.
-   Refresh the worklist after operator actions.
-   Use a configurable, less frequent refresh interval only where
    needed.
-   Show distinct states such as saved, ERP pending, failed and
    completed.
-   Consider event-driven updates only if future requirements justify
    the infrastructure and operational cost.

### 6.10 Security and auditability

-   Enforce ACL permissions on every privileged server-side action.
-   Record actor, timestamp, action, previous state and new state for
    lifecycle transitions.
-   Redact credentials, tokens and unnecessary personal or financial
    information from stored payloads and logs.
-   Restrict access to raw ERP request/response data.
-   Store ERP credentials using secure Magento configuration mechanisms.
-   Protect state-changing requests against CSRF and replay.
-   Define retention and access policies for audit records and
    integration payloads.
-   Ensure manual retry/replay is permission-controlled and audited.

## 7. Performance and non-functional validation

The assignment requirements provide the following operating assumptions
and performance expectations:

  Metric                        Target / expected volume
  --------------------------- --------------------------
  Monthly orders                                 250,000
  Monthly refunds                                  5,000
  Campaign peak submissions                25 per minute
  Concurrent operators                                40
  Daily settlement scan                   20,000 records
  Worklist first page                  Under 1.5 seconds

These values should be used as validation inputs, not treated as already
demonstrated performance.

  -----------------------------------------------------------------------
  Area                                Validation approach
  ----------------------------------- -----------------------------------
  Refund submission                   Measure response time and database
                                      transaction duration under
                                      representative load

  Concurrency                         Run simultaneous submissions
                                      against the same seller order and
                                      verify that refundable quantity is
                                      never exceeded

  Idempotency                         Repeat the same submission and ERP
                                      operation after simulated timeouts;
                                      verify no duplicate financial
                                      action

  Outbox                              Simulate worker termination, ERP
                                      downtime and recovery; verify
                                      durable processing and bounded
                                      retries

  Settlement                          Benchmark the 20,000-record scan
                                      and inspect query plans

  Worklist                            Measure first-page response time,
                                      query count and pagination
                                      behaviour

  Presentation                        Compare all customer-facing
                                      financial values against the
                                      persisted refund snapshot

  Security                            Test role permissions, CSRF
                                      protection, payload redaction and
                                      audit access

  Recovery                            Verify that stuck and ambiguous
                                      operations can be reconciled and
                                      safely replayed
  -----------------------------------------------------------------------

Operational metrics should include outbox backlog and age, retry counts,
ERP latency and error rates, refunds stuck in intermediate states,
settlement acknowledgement delays and worklist response times.

## 8. Deployment and operational considerations

1.  Review declarative schema changes and the corresponding schema
    whitelist.
2.  Ensure existing refund records remain compatible with new state and
    event handling.
3.  Deploy application changes and schema updates through the
    established Magento deployment process.
4.  Configure and verify the required cron/CLI processing.
5.  Validate ERP credentials, connectivity, timeouts and API contract
    configuration.
6.  Run refund submission, ERP failure/recovery, cash registration and
    settlement smoke tests.
7.  Monitor outbox backlog, failed operations and intermediate refund
    states after deployment.
8.  Document rollback and recovery procedures, especially where database
    state and external ERP state may differ.

Database changes should be backward-compatible with existing records
wherever possible. Deployment must not create a window in which
committed refunds can no longer be processed.

## 9. Sign-off questions and assumptions

The following questions should be resolved before production sign-off.

### Business workflow

1.  What precisely defines an active refund for the
    one-active-refund-per-order rule?
2.  Can an order have multiple completed partial refunds?
3.  Can a refund be cancelled after ERP Create Refund succeeds?
4.  What is the correction process if Finance registers an incorrect
    bank transaction reference?

### ERP integration

5.  Does ERP enforce idempotency for Create Refund using the
    store-allocated `refund_no`?
6.  How can Magento determine whether ERP processed a request when the
    HTTP response timed out?
7.  What is the expected consistency delay for Read Refund Status?
8.  What is the documented behaviour for a 409 conflict?
9.  Can ERP retrieve an existing credit note using the store-generated
    refund identifier?

### Data and concurrency

10. Is the refund `version` field updated atomically for every lifecycle
    transition?
11. Which database row should be locked to protect simultaneous refunds
    on the same seller order?
12. What are the retention and access policies for refund events and ERP
    payloads?

### Performance and operations

13. What are the actual SQL execution plans for worklist and settlement
    queries?
14. How much outbox backlog must the system tolerate during ERP
    downtime?
15. Who monitors failed or stuck refunds, and what is the escalation
    process?
16. What are the required recovery-time and recovery-point objectives?

### Financial controls

17. Which original invoice and tax records are authoritative for
    partial-refund calculations?
18. What is the retention period for qualified tax invoices and audit
    events?
19. What exact combination of ERP confirmation and settlement-export
    acknowledgement defines business completion?

## 10. Prioritised implementation plan

  -----------------------------------------------------------------------
  Priority                Workstream              Key outcome
  ----------------------- ----------------------- -----------------------
  1                       Refund integrity        Concurrency protection,
                                                  quantity revalidation
                                                  and local idempotency

  2                       ERP reliability         Transactional outbox,
                                                  stable idempotency,
                                                  retry and
                                                  reconciliation

  3                       Lifecycle control       Central transition
                                                  policy, version checks
                                                  and auditable state
                                                  changes

  4                       Financial consistency   Canonical persisted
                                                  snapshot shared by all
                                                  presentation surfaces

  5                       Operational performance Paginated worklist,
                                                  optimised settlement
                                                  queries and measured
                                                  indexes

  6                       Security and operations ACL, payload redaction,
                                                  monitoring and recovery
                                                  procedures
  -----------------------------------------------------------------------

Some workstreams can proceed in parallel once their interfaces and data
contracts are agreed.

## 11. Final recommendation

**Recommendation: Conditional architecture approval.**

The Magento modular-monolith approach is suitable for the stated
business capability and expected assignment scale. A wholesale rewrite
or introduction of microservices is not justified by the requirements
currently available.

Before production approval, the following must be demonstrated:

1.  Concurrent submissions cannot exceed refundable quantities or create
    conflicting refunds.
2.  Committed refunds cannot be lost when ERP is unavailable.
3.  ERP retries and reconciliation cannot create duplicate logical
    financial operations.
4.  Lifecycle transitions are valid, atomic where required and
    auditable.
5.  Settlement completion is based on the required acknowledgements.
6.  Customer-facing surfaces use consistent persisted financial values.
7.  Worklist and settlement performance meet the stated acceptance
    targets.
8.  Security, monitoring and recovery procedures are tested.

This approach improves financial integrity and operational reliability
while preserving the existing Magento architecture and avoiding
unnecessary infrastructure complexity.
