## Concurrency Strategy

This system must prevent double-booking under a sudden 5 lakh user surge while staying within a $2,000/month infrastructure budget.

### Option A: PostgreSQL SELECT FOR UPDATE

**How it prevents double-booking**

Use a single transaction to lock seat rows and transition them from available to held or booked:

```sql
BEGIN;

SELECT id
FROM seats
WHERE event_id = $1
	AND id = ANY($2)
	AND status = 'available'
ORDER BY id
FOR UPDATE;

UPDATE seats
SET status = 'held',
		held_until = NOW() + INTERVAL '2 minutes',
		held_by = $3,
		version = version + 1
WHERE id = ANY($2)
	AND status = 'available';

INSERT INTO bookings (id, user_id, event_id, status, total_amount)
VALUES ($4, $3, $1, 'pending', $5);

COMMIT;
```

**Hard limit: connection pool math**

Connections held = (% non-payment RPS _ avg_query_time_s) + (% payment RPS _ payment_hold_time_s)

With max_connections = 500, 20% payment calls at 0.8s, and 80% regular calls at 0.02s:

Connections held = (0.8 _ RPS _ 0.02) + (0.2 _ RPS _ 0.8) = 0.176 \* RPS

Pool exhaustion when 0.176 \* RPS = 500 -> RPS ~= 2,841.

**Deadlock risk with multi-seat bookings**

If two transactions lock seats in different orders, deadlocks occur. Mitigation: always order by seat_id in the lock query and reject any seat that is not immediately available (or use NOWAIT with retries).

### Option B: Redis SETNX Distributed Lock

**How it prevents double-booking**

Lock each seat in Redis before writing to the database.

- Lock key: seat_lock:{event_id}:{seat_id}
- Command: SET key value NX PX <ttl_ms>
- Release: Lua script that deletes only if the token matches

```lua
if redis.call("GET", KEYS[1]) == ARGV[1] then
	return redis.call("DEL", KEYS[1])
end
return 0
```

**Redis failure behavior**

If Redis is down, seat holds are rejected (fail closed) rather than allowing a blind DB update. If a lock is lost due to Redis restart, the database still enforces uniqueness and version checks, so double-booking is still prevented but user experience degrades.

**TTL value and tradeoff**

A 120s TTL balances payment and user think time. Too short risks lock expiry before payment confirms, too long increases availability loss when users abandon.

### Our Choice: Hybrid (Redis lock + DB enforcement)

We choose a hybrid strategy because the expected peak RPS (25k+) exceeds the 2.8k RPS pool limit of pure SELECT FOR UPDATE, and the $2,000/month budget does not allow scaling the database pool to tens of thousands of concurrent locks. Redis absorbs the burst, while the database remains the source of truth with a unique constraint on booking_seats.seat_id and optimistic locking on seats.

**Known limits**

- Redis cluster failure reduces availability (holds fail closed).
- TTL tuning is sensitive to payment latency and user behavior.

**When to switch**

If peak RPS stays below ~2k and we can keep payment off the main request path, we could simplify to pure SELECT FOR UPDATE. If Redis becomes the dominant failure mode, we would fall back to DB-only locking and stricter admission control.
