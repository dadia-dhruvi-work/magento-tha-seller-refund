# 06 – Questions and Next Steps

## 1. Open Questions

The following points require further clarification and discussion:

1. **Refund Lifecycle:** Can an order have multiple partial refunds, and what defines a completed refund?
2. **Concurrency:** How should simultaneous refund requests for the same seller order be handled?
3. **ERP Integration:** How should the system handle ERP timeouts, failed requests and duplicate submissions?
4. **Retry Mechanism:** Should failed ERP operations be retried automatically or require manual intervention?
5. **Financial Calculations:** Which source should be considered authoritative for refund amounts, taxes and shipping calculations?
6. **Settlement:** What should happen when ERP confirmation succeeds but settlement export fails?
7. **Performance:** What are the expected performance and scalability requirements for refund processing and settlement operations?

These questions will help validate business assumptions and finalise implementation decisions.

## 2. Next Steps

1. Resolve the development environment limitation and get Magento running.
2. Complete the remaining code investigation.
3. Identify and implement the required code improvement with automated tests.
4. Complete the build impact documentation.
5. Validate all deliverables and prepare the final repository submission.
6. Prepare for the technical discussion, focusing on architecture decisions, code review findings and implementation trade-offs.

## 3. Current Blocker

The local development environment is currently blocked due to hardware limitations (4 GB RAM). An alternative development environment is required to complete implementation and runtime testing.

