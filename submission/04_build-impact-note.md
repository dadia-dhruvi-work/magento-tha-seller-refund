# 04 - Build Impact Note

## Defect Addressed

The PR-02 changes introduced a cumulative refund quantity validation defect in `RefundValidator`.

The validator compared the newly requested refund quantity against the original ordered quantity instead of the remaining refundable quantity. This allowed a refund request to exceed the quantity remaining after previous refunds.

For example:
- Original ordered quantity: 10
- Previously refunded quantity: 6
- Newly requested quantity: 5
- Remaining refundable quantity: 4

The request should be rejected, but the PR-02 implementation could accept it.

## Root Cause and Fix

Restored cumulative quantity validation in:

`app/code/Acme/SellerRefund/Model/RefundValidator.php`

The validator now calculates:

- Previously refunded quantity from the supplied order-item aggregation.
- Remaining refundable quantity using `bcsub()`.
- Requested quantity against the remaining quantity using `bccomp()`.

BCMath is used to preserve the existing four-decimal quantity precision.

## Regression Test

Added `testCumulativeRefundExceedingOrderedQuantityIsRejected()` to:

`app/code/Acme/SellerRefund/Test/Unit/Model/RefundValidatorTest.php`

The test verifies that an order item with quantity 10, of which 6 has already been refunded, rejects a new refund request for quantity 5.

The regression test failed against the PR-02 implementation and passed after the correction.

## Verification Results

- Targeted validator tests: 8 passed.
- Complete unit suite: 30 tests, 85 assertions passed.
- Integration suite: 7 tests, 37 assertions passed.
- `git diff --check`: passed.

The integration test output included a `/usr/sbin/sendmail: not found` environment warning. It did not cause test failures.

## Concurrency and Transaction Safety

The existing refund submission flow was inspected.

`RefundProcessor::submit()` starts a database transaction, acquires an order-level lock through `OrderLock`, calculates previously refunded quantities, validates the submission, and persists the refund and outbox entry before committing.

`OrderLock` uses `SELECT ... FOR UPDATE` on the sales order row. This serializes refund submissions for the same order when they follow this processing path.

This was verified by code inspection. A dedicated concurrent-submission test was not executed.

## Scope and Impact

The change prevents cumulative over-refunding when previous refund quantities are already recorded. It restores the intended validation behaviour without changing the refund lifecycle, ERP processing, or outbox workflow.

No changes were made to the supplied review branches or to the refund processing architecture as part of this fix.
