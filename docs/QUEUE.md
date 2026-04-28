## Async Order Flow (SQS)

### Section 1: Why Async?

Synchronous payment calls collapse the database pool during a burst.

Assume peak RPS = 25,000. Payment calls are 20% at 0.8s, regular calls 80% at 0.02s.

Connections held = (0.8 _ RPS _ 0.02) + (0.2 _ RPS _ 0.8) = 0.176 \* RPS.

At 25,000 RPS, connections held = 4,400, far above a 500-connection pool. The pool exhausts around 2,841 RPS, so keeping payment in the request path is not viable. Async processing keeps API calls short and releases connections quickly.

### Section 2: The Queue Message Format

```json
{
  "bookingId": "...",
  "userId": "...",
  "totalAmount": 0.0,
  "paymentToken": "...",
  "seatIds": ["..."],
  "eventId": 0,
  "idempotencyKey": "..."
}
```

- bookingId: primary identifier for the order and payment status updates
- userId: ownership and audit trail
- totalAmount: prevents recalculation drift at processing time
- paymentToken: gateway token or payment method reference
- seatIds: the exact seats to confirm, no additional lookup required
- eventId: used for seat status updates and validation
- idempotencyKey: prevents double-charging on retries

### Section 3: The Worker Logic - Success and Failure Paths

1. Receive message and validate idempotencyKey.
2. Fetch booking; if already confirmed or refunded, ack and stop.
3. Call payment provider with idempotencyKey.
4. On success:
   - Begin DB transaction
   - Update booking.status = confirmed
   - Update seats.status = booked, clear held_until and held_by
   - Insert booking_seats rows (or verify they already exist)
   - Commit and ack message
5. On failure:
   - Begin DB transaction
   - Update booking.status = failed
   - Release seats to available, clear hold fields
   - Commit
   - Allow retry up to max receive count
6. If retries exhausted, send to DLQ and alert operations.

### Section 4: Edge Cases

**Server crashes after SQS publish but before API responds**

The booking exists in pending state. The client may retry; idempotency returns the existing booking. The worker processes the queued message and finalizes status.

**Payment gateway timeout (unknown outcome)**

The worker retries using the same idempotencyKey. If the gateway eventually succeeds, subsequent retries return success without double-charge. If the gateway never confirms, the booking is marked failed after max retries.

### Section 5: SQS Configuration

- Visibility timeout: 120 seconds. It must exceed the payment call p95 plus DB commit time.
- Max receive count: 5 before DLQ. This balances transient gateway errors with user wait time.
