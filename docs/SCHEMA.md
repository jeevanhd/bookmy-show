## PostgreSQL Schema

```sql
CREATE EXTENSION IF NOT EXISTS "pgcrypto";

CREATE TABLE venues (
	id BIGSERIAL PRIMARY KEY,
	name TEXT NOT NULL,
	city TEXT NOT NULL,
	capacity INTEGER NOT NULL CHECK (capacity > 0),
	created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_venues_city ON venues (city);

CREATE TABLE events (
	id BIGSERIAL PRIMARY KEY,
	venue_id BIGINT NOT NULL REFERENCES venues (id) ON DELETE RESTRICT,
	name TEXT NOT NULL,
	start_time TIMESTAMPTZ NOT NULL,
	status TEXT NOT NULL CHECK (status IN ('upcoming', 'on_sale', 'sold_out', 'cancelled')),
	total_seat_count INTEGER NOT NULL CHECK (total_seat_count > 0),
	created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_events_venue ON events (venue_id);

CREATE TABLE users (
	id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
	email TEXT NOT NULL UNIQUE,
	phone TEXT NOT NULL UNIQUE,
	name TEXT NOT NULL,
	created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_users_created_at ON users (created_at DESC);

CREATE TABLE seats (
	id BIGSERIAL PRIMARY KEY,
	event_id BIGINT NOT NULL REFERENCES events (id) ON DELETE CASCADE,
	section TEXT NOT NULL,
	row_label TEXT NOT NULL,
	seat_number INTEGER NOT NULL CHECK (seat_number > 0),
	price NUMERIC(10, 2) NOT NULL CHECK (price > 0),
	category TEXT NOT NULL CHECK (category IN ('VIP', 'General', 'Premium')),
	status TEXT NOT NULL CHECK (status IN ('available', 'held', 'booked')),
	held_until TIMESTAMPTZ,
	held_by UUID REFERENCES users (id) ON DELETE SET NULL,
	version INTEGER NOT NULL DEFAULT 0 CHECK (version >= 0),
	created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
	UNIQUE (event_id, section, row_label, seat_number)
);

CREATE INDEX idx_seats_event_status ON seats (event_id, status);
CREATE INDEX idx_seats_event_category ON seats (event_id, category);

CREATE TABLE bookings (
	id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
	user_id UUID NOT NULL REFERENCES users (id) ON DELETE RESTRICT,
	event_id BIGINT NOT NULL REFERENCES events (id) ON DELETE RESTRICT,
	status TEXT NOT NULL CHECK (status IN ('pending', 'confirmed', 'failed', 'refunded')),
	total_amount NUMERIC(10, 2) NOT NULL CHECK (total_amount > 0),
	payment_reference TEXT,
	created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
	updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_bookings_user ON bookings (user_id, created_at DESC);
CREATE INDEX idx_bookings_event ON bookings (event_id);
CREATE INDEX idx_bookings_unresolved ON bookings (created_at DESC)
	WHERE status IN ('pending', 'failed');

CREATE TABLE booking_seats (
	booking_id UUID NOT NULL REFERENCES bookings (id) ON DELETE CASCADE,
	seat_id BIGINT NOT NULL REFERENCES seats (id) ON DELETE RESTRICT,
	price_paid NUMERIC(10, 2) NOT NULL CHECK (price_paid > 0),
	created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
	PRIMARY KEY (booking_id, seat_id),
	UNIQUE (seat_id)
);

CREATE INDEX idx_booking_seats_booking_id ON booking_seats (booking_id);
```

### Commentary

UUID for booking.id avoids exposing sequential order volume, supports idempotency keys across services, and prevents hot spots on insert under heavy concurrency compared to SERIAL.

The seats.version column enables optimistic locking for high-throughput updates, allowing the application to detect and reject stale writes without holding long locks.

held_until ensures seat holds expire even if an application node crashes, providing a deterministic release mechanism that does not rely on in-memory timers.

The partial index on bookings.status keeps hot queries on unresolved orders fast while avoiding index bloat from long-term confirmed and refunded records.
