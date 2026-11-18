# 06 – Questions and Next Steps

## 1. Open Questions

The following points require further clarification and discussion before production sign-off:

1. **Refund Lifecycle:** Can an order have multiple partial refunds, and what defines a completed refund from the business perspective?
2. **Concurrency:** Do all refund creation paths use the existing order-level lock, and are the relevant database tables using InnoDB? Should a dedicated concurrent-submission test be added?
3. **ERP Integration:** How should the system handle ERP timeouts, failed requests and duplicate submissions, particularly when ERP processing succeeds but Magento does not receive the response?
4. **Retry Mechanism:** Which ERP failures should be retried automatically, and which require manual intervention?
5. **Financial Calculations:** Which source should be authoritative for refund amounts, taxes, discounts and shipping calculations?
6. **Settlement:** What should happen when ERP confirmation succeeds but settlement export fails? How should reconciliation be performed?
7. **Performance:** What are the expected performance and scalability requirements for refund processing and settlement operations?

These questions should be resolved with the business, Finance and integration stakeholders before finalising production-level design decisions.

## 2. Completed Work

1. Reviewed both supplied PR branches against the main branch.
2. Documented code review findings, severity and recommendations.
3. Prepared the end-to-end refund design and phased delivery plan.
4. Reviewed the inherited architecture and documented sign-off risks.
5. Implemented the cumulative refund quantity validation fix.
6. Added a regression test demonstrating the over-refund defect.
7. Executed the targeted validator tests, complete unit suite and integration suite.
8. Documented the implementation impact and verification results.

## 3. Recommended Next Steps

1. Validate the remaining ERP idempotency, retry and timeout scenarios.
2. Add dedicated concurrency tests for simultaneous refund submissions.
3. Verify financial calculations against representative business scenarios, including discounts, shipping and tax rounding.
4. Validate the complete refund lifecycle with Finance and ERP stakeholders.
5. Perform a clean-clone verification of the documented setup and test commands.
6. Complete repository access and submission checks.
7. Prepare for the technical discussion, focusing on code review findings, architecture decisions, implementation trade-offs and areas requiring further validation.

## 4. Environment and Verification Notes

The initial local development environment was constrained by laptop hardware limitations (4 GB RAM). Development and runtime testing were completed using GitHub Codespaces instead.

The targeted validator tests, complete unit suite and integration suite passed in that environment. The integration test output included a missing sendmail executable warning, which did not cause test failures.

A dedicated concurrent-submission test and clean-clone verification remain outstanding.
