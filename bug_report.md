# CoWork Booking API — Bug Report

**CoWork: Multi-Tenant Coworking Space Booking API — Preliminary Round**

24 planted bugs found and fixed.
(paths, status codes, error codes, JSON field names, JWT claims).

Line numbers refer to the **original, unfixed** codebase.

---

## Auth & tokens

### 1. Access tokens lasted 15 hours instead of 15 minutes
**File:** `app/auth.py`, line 50  

The lifetime was computed as `timedelta(minutes=ACCESS_TOKEN_EXPIRE_MINUTES * 60)`. The config
value is already in minutes (15), so multiplying by 60 while still passing it as `minutes=` gives
900 **minutes**. The spec wants access tokens to expire in exactly 900 **seconds**.

**Fix:** `timedelta(minutes=ACCESS_TOKEN_EXPIRE_MINUTES)` — now `exp - iat == 900`.

---

### 2. Logout didn't actually log you out
**File:** `app/auth.py`, line 86 (store) vs line 97 (check)  

`revoke_access_token` stored the token's `jti` in the revoked set, but `get_token_payload`
compared `payload.get("sub")` — the user id — against that set. A user id is never equal to a
uuid hex, so no token was ever considered revoked.

**Fix:** check `is_revoked(payload.get("jti"))`, matching what logout stores via `revoke_jti`.

---

### 3. Refresh tokens could be replayed forever
**Files:** `app/routers/auth.py`, lines 81–93; `app/auth.py`  

The refresh endpoint handed out new tokens but never remembered that the old one was spent, so the
same refresh token could be reused indefinitely. Concurrent refreshes with the same token could
also both succeed without synchronization.

**Fix:** shared `revoke_jti` / `is_revoked`; `consume_refresh_token()` guarded by `_refresh_lock`.
First use records the `jti`, any later use gets `401`.

---

### 4. Registering a taken username returned the existing user
**File:** `app/routers/auth.py`, lines 37–43  

Instead of the required `409 USERNAME_TAKEN`, registering a duplicate username in an org quietly
returned `201` with the existing user's id and role.

**Fix:** raise `AppError(409, "USERNAME_TAKEN", ...)`.

---

### 5. Concurrent registrations could crash with a 500
**File:** `app/routers/auth.py`, lines 26–30 and 45–53  

Org creation and user creation were plain check-then-insert. If two requests raced to create the
same new org (or the same username), the loser hit an unhandled `IntegrityError` → HTTP 500.

**Fix:** both commits catch `IntegrityError`. Losing the org race means the org now exists, so
the caller joins as member; losing the username race returns `409 USERNAME_TAKEN`.

---

## Datetimes

### 6. Timezone offsets were thrown away instead of converted
**File:** `app/timeutils.py`, lines 12–13  

`parse_input_datetime` did `dt.replace(tzinfo=None)` on offset-carrying inputs, keeping the
wall-clock time and deleting the offset. `2026-07-10T10:00:00+06:00` was stored as `10:00`
instead of `04:00 UTC`, corrupting conflicts, quota windows, refund notice, and availability.

**Fix:** `dt.astimezone(timezone.utc).replace(tzinfo=None)` — convert first, then strip.

---

### 7. Malformed datetime crashed booking creation
**File:** `app/routers/bookings.py` `create_booking`  

An invalid or empty `start_time` / `end_time` made `datetime.fromisoformat` raise `ValueError`,
which was unhandled → HTTP 500. Availability and usage-report already mapped bad dates to
`400 INVALID_BOOKING_WINDOW`; the booking path did not.

**Fix:** wrap both `parse_input_datetime` calls in `try/except ValueError` →
`AppError(400, "INVALID_BOOKING_WINDOW", ...)`.

---

## Creating bookings

### 8. A 5-minute grace window for past start times
**File:** `app/routers/bookings.py`, line 86  

The check was `if start <= now - timedelta(seconds=300)`, allowing bookings up to 5 minutes in
the past. The spec is explicit: strictly in the future, no grace window.

**Fix:** `if start <= now`.

---

### 9. Zero-hour and negative-duration bookings were accepted
**File:** `app/routers/bookings.py`, lines 89–94  

Only `duration > 8h` was rejected. There was no minimum check and no end-after-start check, so
`end == start` sailed through, and an `end` before `start` by a whole number of hours also passed
and created a booking with a negative price.

**Fix:** reject `end <= start` and `duration_hours < MIN_DURATION_HOURS (1)`.

---

### 10. Back-to-back bookings were treated as conflicts
**File:** `app/routers/bookings.py`, line 50 (`_has_conflict`)  

The overlap test used `<=` on both sides. The spec says overlap iff
`existing.start < new.end AND new.start < existing.end`, and back-to-back is allowed. With the
inclusive version, a booking ending at 10:00 blocked another starting at 10:00.

**Fix:** strict `<` on both comparisons.

---

### 11. Two people could book the same slot at the same time
**File:** `app/routers/bookings.py`, lines 100–118  

The conflict check and insert were separate steps with a planted `_pricing_warmup` sleep between
them. Two concurrent requests for the same slot both saw "free" and both committed.

**Fix:** module-level `_booking_lock` makes check-conflict → check-quota → insert → commit
atomic. Sleep removed.

---

### 12. The quota check had the same race
**File:** `app/routers/bookings.py`, lines 55–71  

Same check-then-act pattern with a planted `_quota_audit` sleep: parallel requests all counted
the same snapshot and all inserted past the 3-bookings-per-24h limit.

**Fix:** quota check runs inside the same `_booking_lock`.

---

### 13. Caches went stale in both directions
**File:** `app/routers/bookings.py`, lines 120–122 (create) and 215–218 (cancel)  

Creating a booking invalidated the availability cache but not the usage-report cache, so
`/admin/usage-report` kept serving pre-booking numbers. Cancelling did the mirror image: it
invalidated the report but not availability, so cancelled bookings stayed "busy" in
`/rooms/{id}/availability`.

**Fix:** create also calls `cache.invalidate_report(org_id)`; cancel also calls
`cache.invalidate_availability(room_id, date)`.

---

## Listing & reading bookings

### 14. Pagination was wrong three different ways
**File:** `app/routers/bookings.py`, lines 136–140  

In one query: descending order instead of ascending-by-`start_time`;
`offset(page * limit)` instead of `(page - 1) * limit`, which skips the entire first page; and a
hardcoded `limit(10)` that ignored the caller's `limit` parameter.

**Fix:** `order_by(start_time.asc(), id.asc()).offset((page - 1) * limit).limit(limit)`.

---

### 15. Any member could read any booking in their org
**File:** `app/routers/bookings.py`, lines 156–163  

`GET /bookings/{id}` only checked that the booking's room belonged to the caller's org. Members
may read only their own bookings; another member's id should behave as `404 BOOKING_NOT_FOUND`.
The cancel endpoint below it already had the ownership check — it was missing from the read path.

**Fix:** non-admins who don't own the booking get `404 BOOKING_NOT_FOUND`.

---

### 16. Booking detail returned the wrong start_time
**File:** `app/routers/bookings.py`, line 166  

After serializing correctly, the handler did
`response["start_time"] = iso_utc(booking.created_at)` — overwriting the real start time with the
creation timestamp.

**Fix:** removed the line; `serialize_booking` already emits the correct `start_time`.

---

## Cancelling & refunds

### 17. Refund tiers were wrong at both ends
**File:** `app/routers/bookings.py`, lines 198–206  

Two problems in one if-chain. Notice was floored to whole hours and compared with `> 48`, so
cancelling with exactly 48 hours of notice landed in the 50% bucket instead of 100%. The
`else` branch — under 24 hours — returned 50 instead of 0.

**Fix:** compare the `timedelta` directly: `>= 48h → 100`, `>= 24h → 50`, else `0`.

---

### 18. The refund amount in the response could differ from the ledger
**Files:** `app/services/refunds.py`, lines 14–17; `app/routers/bookings.py`, line 208  

The response computed the amount with Python's `round()` (banker's rounding) while the RefundLog
computed it separately with floats and `int()` (truncation). Neither matched half-up rounding, and
the spec requires the response amount to equal the stored amount. Example: 3333 cents at 50% is
1666.5 — the correct answer is 1667; both code paths said 1666.

**Fix:** one computation, integer-only, in `log_refund`:
`(price_cents * percent + 50) // 100`. The cancel endpoint returns `refund_entry.amount_cents`.

---

### 19. Cancelling twice in parallel produced two refunds
**File:** `app/routers/bookings.py`, lines 195–214  

The flow was: check status → planted `_settlement_pause` sleep → flip status → commit. Two
concurrent cancels both read `confirmed`, both logged a refund. The refund was also committed
before the status flip, so a crash in between could refund a booking that stayed confirmed.

**Fix:** cancel runs under `_booking_lock`; `db.expire(booking)` re-reads fresh status; status
flip and RefundLog written in a single commit.

---

## The services

### 20. A textbook deadlock in notifications
**File:** `app/services/notifications.py`, lines 24–35  

`notify_created` took the email lock, then the audit lock. `notify_cancelled` took them in the
opposite order. One create and one cancel landing together could deadlock and wedge the whole
service. The sleeps inside the critical sections made the bad interleaving easy to hit.

**Fix:** both functions acquire email → audit. Sleeps kept to simulate I/O.

---

### 21. Duplicate reference codes under load
**File:** `app/services/reference.py`, lines 17–21  

`next_reference_code` read the counter, slept 120 ms (`_format_pause`), then wrote back
`current + 1`. Concurrent creates read the same counter value and got the same code.

**Fix:** read-and-increment atomic under a `threading.Lock`; sleep removed.

---

### 22. The rate limiter forgot requests under load
**File:** `app/services/ratelimit.py`, lines 18–26  

Each request copied the user's bucket, slept 100 ms (`_settle_pause`), appended its timestamp,
and wrote the copy back. Concurrent requests clobbered each other's appends, letting bursts sail
past the 20-per-60-seconds limit.

**Fix:** trim + append + count happen atomically under a `threading.Lock`; sleep removed.

---

### 23. Room stats drifted away from reality
**File:** `app/routers/rooms.py` `room_stats` (backed by `app/services/stats.py`)  

Stats were served from in-memory counters with a racy read-modify-write pattern and a planted
`_aggregate_pause` sleep. Under concurrent bookings the increments overwrote each other and
`/rooms/{id}/stats` stopped matching the actual booking table.

**Fix:** derive live from the database (`COUNT` + `COALESCE(SUM(price_cents), 0)` over
confirmed bookings). Removed `stats.record_create` / `record_cancel` from the booking paths.

---

### 24. The CSV export leaked other orgs' data
**File:** `app/services/export.py`, lines 48–52  


With `include_all=true` and a `room_id`, the export used `fetch_bookings_raw(room_id)` with no
org filter. An admin of org A could pass a room id belonging to org B and download org B's
booking history.

**Fix:** all paths go through org-scoped `_fetch_scoped`; when `room_id` is set, validate the
room belongs to the caller's org (else `404 ROOM_NOT_FOUND`).


## Summary

| # | Area |
|---|------|
| 1 | Access token expiry |
| 2 | Logout revocation claim |
| 3 | Refresh token reuse + race |
| 4 | Duplicate username |
| 5 | Registration IntegrityError |
| 6 | UTC datetime parsing |
| 7 | Malformed datetime → 500 |
| 8 | Past start grace window |
| 9 | Missing duration validation |
| 10 | Back-to-back conflict |
| 11 | Double-booking race |
| 12 | Quota race |
| 13 | Stale cache invalidation |
| 14 | Pagination / ordering |
| 15 | Member booking visibility |
| 16 | Wrong start_time in detail |
| 17 | Refund tier logic |
| 18 | Refund rounding / mismatch |
| 19 | Cancel race |
| 20 | Notification deadlock |
| 21 | Reference code race |
| 22 | Rate limit race |
| 23 | Room stats drift |
| 24 | Export multi-tenancy |

**Total: 24 bugs**