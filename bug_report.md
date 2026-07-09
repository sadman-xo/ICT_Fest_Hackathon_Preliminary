# Bug Report

## 1. Conflicting Booking Detection Bug
**File/Line:** `app/routers/bookings.py:54`
**What the bug was:** Booking conflict detection used `<=` comparison for time boundaries, allowing bookings to be created that touched adjacent bookings (e.g., booking A ends at 10:00, booking B could start at 10:00, causing overlap).
**How it was fixed:** Changed `b.start_time <= end and start <= b.end_time` to `b.start_time < end and start < b.end_time` to ensure bookings don't touch.

## 2. Race Condition in Reference Code Generation
**File/Line:** `app/services/reference.py:22`
**What the bug was:** Multiple threads could read the same counter value before incrementing, causing duplicate reference codes and potential IntegrityError.
**How it was fixed:** Added `counter_lock = Lock()` and wrapped the counter read/increment in `with counter_lock:` block.

## 3. Race Condition in Rate Limiting
**File/Line:** `app/services/ratelimit.py:19`
**What the bug was:** The `_buckets` dictionary was accessed without thread synchronization, causing race conditions where rate limits could be bypassed or incorrectly enforced.
**How it was fixed:** Added `_buckets_lock = Lock()` and wrapped bucket operations with `with _buckets_lock:`.

## 4. Race Condition in Stats Tracking
**File/Line:** `app/services/stats.py:4,17`
**What the bug was:** The `_stats` dictionary was modified without thread locking, causing data races when multiple threads update stats concurrently.
**How it was fixed:** Added `import threading` and `_lock = threading.Lock()`, wrapped all stats modifications with `with _lock:`.

## 5. Deadlock in Notification System
**File/Line:** `app/services/notifications.py:31-35`
**What the bug was:** Lock ordering was inconsistent between notification functions. `notify_created` acquired `_email_lock` then `_audit_lock`, while `notify_cancelled` acquired `_audit_lock` then `_email_lock`, creating potential for deadlock.
**How it was fixed:** Changed `notify_cancelled` to acquire locks in same order (`_email_lock` first, then `_audit_lock`).

## 6. Incorrect Booking Window Validation
**File/Line:** `app/routers/bookings.py:93-96`
**What the bug was:** Original code allowed `start_time <= now - timedelta(seconds=300)` (5-minute grace period), which incorrectly permitted bookings starting in the past.
**How it was fixed:** Changed to strict `if start <= now:` validation.

## 7. Missing End Time Validation & Duration Constraints
**File/Line:** `app/routers/bookings.py:98-111`
**What the bug was:** No validation that `end_time` must be after `start_time`, and duration validation only checked maximum, not minimum, allowing invalid bookings with zero or negative duration.
**How it was fixed:** Added `if end <= start:` validation, changed duration check to `duration_hours > MAX_DURATION_HOURS or duration_hours < MIN_DURATION_HOURS`.

## 8. Incorrect Pagination Offset
**File/Line:** `app/routers/bookings.py:184`
**What the bug was:** Pagination used `.offset(page * limit)` with hardcoded `.limit(10)`, causing wrong items to be returned for pages > 1 and ignoring the `limit` parameter.
**How it was fixed:** Changed to `.offset((page-1) * limit).limit(limit)`.

## 9. Wrong Sorting Order for Bookings
**File/Line:** `app/routers/bookings.py:183`
**What the bug was:** Bookings were sorted by `start_time.desc()` showing future bookings first, which is counterintuitive for listing.
**How it was fixed:** Changed to `start_time.asc()` to show chronological order.

## 10. Wrong Field in Booking Response
**File/Line:** `app/routers/bookings.py:215`
**What the bug was:** `get_booking` endpoint returned `booking.created_at` in `start_time` field instead of actual `booking.start_time`.
**How it was fixed:** Changed to `iso_utc(booking.start_time)`.

## 11. Race Condition in Booking Creation
**File/Line:** `app/routers/bookings.py:122-126`
**What the bug was:** No locking around room and user during booking creation, allowing concurrent bookings to create conflicts or exceed quota.
**How it was fixed:** Added `room_locks` and `user_locks` using `defaultdict(Lock)`, wrapped critical section with both locks.

## 12. Race Condition in Booking Cancellation
**File/Line:** `app/routers/bookings.py:244-246`
**What the bug was:** No locking during cancellation, allowing double-cancellation or refund issues.
**How it was fixed:** Added `booking_locks` dict and wrapped cancellation logic with `booking_lock`.

## 13. Incorrect Refund Logic
**File/Line:** `app/routers/bookings.py:252-257`
**What the bug was:** Original refund logic had wrong thresholds (`notice_hours > 48` gave 100%, `notice >= 24h` gave 50%, else 50%) - everyone got 50% refund. Missing 0% case for <24h notice.
**How it was fixed:** Changed to proper tier logic: `>= 48h` → 100%, `>= 24h` → 50%, else → 0%.

## 14. Incorrect Refund Amount Rounding
**File/Line:** `app/services/refunds.py:14-22`
**What the bug was:** Using `int(refund_dollars * 100)` for rounding caused banker's rounding (round half to even), not standard rounding.
**How it was fixed:** Used `Decimal` with `ROUND_HALF_UP` for proper currency rounding.

## 15. Incorrect Token Revocation Check
**File/Line:** `app/auth.py:116`
**What the bug was:** Checking `payload.get("sub") in _revoked_tokens` instead of `payload.get("jti")`, checking wrong field for token revocation.
**How it was fixed:** Changed to check `payload.get("jti") in _revoked_tokens`.

## 16. Missing Refresh Token Revocation
**File/Line:** `app/routers/auth.py:82-87`
**What the bug was:** Refresh tokens were not being revoked after use, allowing reuse of refresh tokens (security vulnerability).
**How it was fixed:** Added `is_refresh_token_revoked()` check and `revoke_refresh_token(data)` call in refresh endpoint.

## 17. Duplicate Registration Accepted
**File/Line:** `app/routers/auth.py:44-45`
**What the bug was:** Registration endpoint returned existing user instead of error when username already exists, violating business rules.
**How it was fixed:** Changed to raise `AppError(409, "USERNAME_TAKEN", ...)` for duplicate username.

## 18. Wrong Time Parsing for Timezone-Aware Datetimes
**File/Line:** `app/timeutils.py:13-14`
**What the bug was:** `parse_input_datetime` only stripped timezone info without converting to UTC, causing incorrect time interpretation for non-UTC timezones.
**How it was fixed:** Changed to `dt.astimezone(timezone.utc).replace(tzinfo=None)` to properly convert to UTC before stripping.

## 19. Missing Unique Constraint on Reference Code
**File/Line:** `app/models.py:55`
**What the bug was:** `reference_code` column was missing `unique=True`, allowing duplicate reference codes in database violating business requirements.
**How it was fixed:** Added `unique=True` to the column definition.