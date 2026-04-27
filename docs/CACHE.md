## Cache Design (Redis)

### Required Cache Entries

**Event details**

- Key: event:{event_id}
- TTL: 300 seconds (event metadata changes rarely, but status can change near on-sale time)
- Invalidation: event update or status change -> delete key
- Expiry mode: event-driven invalidation with TTL as safety

**Seat availability count per event per category**

- Key: availability:{event_id}:{category}
- TTL: 30 seconds (fast-moving data, short freshness window)
- Invalidation: any seat status change for that event+category -> delete key
- Expiry mode: event-driven invalidation with TTL as safety

**Static seat map layout for an event**

- Key: seatmap:{event_id}
- TTL: 86,400 seconds (24 hours; layout is stable once published)
- Invalidation: seat map edit or layout migration -> delete key
- Expiry mode: event-driven invalidation with TTL as safety

**Explicitly NOT cached**

- Individual seat status. It is highly volatile and staleness risks double-booking. Only counts are cached; per-seat truth is always read from the database or protected by locks.

### Invalidation Strategy

Use cache-aside with delete-on-write. The database is the source of truth; cache is a fast read-through layer.

```text
When seat status changes:
	1) Update DB (source of truth)
	2) Delete Redis key: availability:{event_id}:{category}

Next read:
	1) Cache miss -> query DB
	2) Populate cache with TTL
```
