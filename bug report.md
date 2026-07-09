Here is the complete, professional, and highly structured **`bug_report.md`** compiled to match the exact format specified in Section 10 of the Preliminary Round problem statement [1]. 

This report integrates your existing reports alongside the **new `Z` suffix timezone format fix** and the **cancellation concurrency lock fix**, presenting a total of 18 highly structured bugs.

---

# Bug Report: CoWork API — Preliminary Round

This document lists all bugs identified and fixed across the CoWork API repository [1]. Each entry specifies the exact file and lines, the logical root cause of the incorrect behavior, and the precise code modifications applied to resolve them [1].

---

### **Bug 1: Timezone Offset Conversion Loss**
* **File:** `app/timeutils.py`
* **Line Number:** 13
* **What the Bug Was & Why It Caused Incorrect Behavior:** 
  The function `parse_input_datetime` stripped the timezone information from incoming datetime strings using `.replace(tzinfo=None)` [1]. This dropped the UTC offset descriptor (e.g., `+06:00`) without adjusting the hour values mathematically, causing timestamps to be stored with a several-hour error in the database [1]. This violated **Business Rule 1** [1].
* **How It Was Fixed:** 
  Modified the helper to translate the datetime mathematically using `.astimezone(timezone.utc)` prior to removing the tzinfo wrapper:
  ```python
  if dt.tzinfo is not None:
      dt = dt.astimezone(timezone.utc).replace(tzinfo=None)
  ```

---

### **Bug 2: JWT Access Token Lifespan Multiplier**
* **File:** `app/auth.py`
* **Line Number:** 50
* **What the Bug Was & Why It Caused Incorrect Behavior:** 
  The token lifetime was defined as `ACCESS_TOKEN_EXPIRE_MINUTES * 60` minutes [1]. Since `ACCESS_TOKEN_EXPIRE_MINUTES` is defined as `15`, this calculated an expiration date 900 minutes (15 hours) in the future [1]. This violated **Business Rule 8** (access tokens must expire in exactly 900 seconds / 15 minutes) [1].
* **How It Was Fixed:** 
  Removed the multiplier to set the interval directly in minutes:
  ```python
  lifetime = timedelta(minutes=ACCESS_TOKEN_EXPIRE_MINUTES)
  ```

---

### **Bug 3: Token Revocation Verification Failure**
* **File:** `app/auth.py`
* **Line Number:** 97
* **What the Bug Was & Why It Caused Incorrect Behavior:** 
  Upon logout, the system blacklists the token's unique ID (`jti`) [1]. However, the token validator on line 97 checked the user identity subject (`sub`) against the blacklist set:
  ```python
  if payload.get("sub") in _revoked_tokens:
  ```
  Since the `sub` claim holds the user ID and does not match the token's `jti`, revoked tokens were never recognized as blacklisted, leaving logged-out access tokens active and usable [1].
* **How It Was Fixed:** 
  Updated the condition to validate the token's `"jti"` claim:
  ```python
  if payload.get("jti") in _revoked_tokens:
  ```

---

### **Bug 4: User Registration Ownership & Concurrency Clash**
* **File:** `app/routers/auth.py`
* **Line Numbers:** 23–60
* **What the Bug Was & Why It Caused Incorrect Behavior:** 
  If a duplicate username was submitted inside the same organization, the endpoint silently returned the existing user details with an HTTP `201 Created` status code [1]. This violated **Business Rule 15** [1]. Furthermore, under concurrent duplicate registration attempts, the database would throw an unhandled `IntegrityError` resulting in a crash and an HTTP `500 Internal Server Error` [1].
* **How It Was Fixed:** Raised a handled `409 USERNAME_TAKEN` error [1]. Additionally, wrapped organization and user creation in transaction-handling `try/except` blocks to perform rollbacks on concurrent collisions [1].

---

### **Bug 5: Single-Use Refresh Token Reuse Bypass**
* **File:** `app/routers/auth.py`
* **Line Numbers:** 72–75 and 80
* **What the Bug Was & Why It Caused Incorrect Behavior:** 
  The `/refresh` token rotation endpoint decoded incoming refresh tokens but did not check if the token's `jti` was already blacklisted [1]. Additionally, it did not blacklist the presented refresh token after verifying it [1]. This allowed refresh tokens to be reused indefinitely, violating **Business Rule 8** [1].
* **How It Was Fixed:** 
  Integrated verification against `_revoked_tokens` and called `revoke_access_token(data)` immediately upon successful rotation to enforce single-use execution:
  ```python
  from ..auth import _revoked_tokens, revoke_access_token
  if data.get("jti") in _revoked_tokens:
      raise AppError(401, "UNAUTHORIZED", "Token has been revoked")
  revoke_access_token(data)
  ```

---

### **Bug 6: Bookings List Pagination, Limits, and Sorting**
* **File:** `app/routers/bookings.py`
* **Line Numbers:** 137–139
* **What the Bug Was & Why It Caused Incorrect Behavior:** 
  The paginated listing query was sorted in descending order (instead of ascending), offset by `page * limit` (which skipped the first page of results entirely), and used a hardcoded page size limit of `10` [1]. This violated **Business Rule 11** [1].
* **How It Was Fixed:** 
  Reconfigured the query chain with correct sorting, index-corrected offsets, and dynamic query limits:
  ```python
  items = (
      base.order_by(Booking.start_time.asc(), Booking.id.asc())
      .offset((page - 1) * limit)
      .limit(limit)
      .all()
  )
  ```

---

### **Bug 7: Multi-Tenancy Read Visibility Protection**
* **File:** `app/routers/bookings.py`
* **Line Numbers:** 164–166 and 169
* **What the Bug Was & Why It Caused Incorrect Behavior:** 
  The GET `/bookings/{booking_id}` endpoint Joined the Rooms table to verify organization ownership, but failed to assert user ownership for general members [1]. This allowed any member of an organization to read any other member's booking [1]. Additionally, the response payload on line 169 overwrote `"start_time"` with `booking.created_at`, returning incorrect timestamps [1].
* **How It Was Fixed:** 
  Enforced a member-level ownership assertion block and corrected the serialization override to map `start_time` [1]:
  ```python
  if user.role != "admin" and booking.user_id != user.id:
      raise AppError(404, "BOOKING_NOT_FOUND", "Booking not found")
  response["start_time"] = iso_utc(booking.start_time)
  ```

---

### **Bug 8: Export Multi-Tenancy Room Ownership Bypass**
* **File:** `app/routers/admin.py`
* **Line Numbers:** 72–76
* **What the Bug Was & Why It Caused Incorrect Behavior:** 
  The `/admin/export` endpoint accepted a query parameter `room_id` but did not verify if the requested room belonged to the administrator's organization, allowing administrators to export booking data belonging to other tenant organizations [1].
* **How It Was Fixed:** 
  Added a query to verify organization ownership of the requested room before executing the export:
  ```python
  if room_id is not None:
      room = db.query(Room).filter(Room.id == room_id, Room.org_id == admin.org_id).first()
      if room is None:
          raise AppError(404, "ROOM_NOT_FOUND", "Room not found")
  ```

---

### **Bug 9: Overlap logic and Double-Booking Boundaries**
* **File:** `app/routers/bookings.py`
* **Line Numbers:** 53–54
* **What the Bug Was & Why It Caused Incorrect Behavior:** 
  The room availability checker used `<=` and `>=` boundaries to detect conflicts. This incorrectly marked back-to-back bookings (e.g., one booking ending exactly when the next begins) as overlapping and blocked them, violating **Business Rule 3** [1].
* **How It Was Fixed:** 
  Updated the overlap comparison to use strict `<` inequalities to allow back-to-back reservations [1]:
  ```python
  if b.start_time < end and start < b.end_time:
      return True
  ```

---

### **Bug 10: Atomic Booking Creation Thread-Lock**
* **File:** `app/routers/bookings.py`
* **Line Numbers:** 18–20 and 79–132
* **What the Bug Was & Why It Caused Incorrect Behavior:** 
  The booking check-and-insert sequence was not synchronized. Under concurrent booking requests for the same room, multiple requests could simultaneously pass the `_has_conflict` verification check and persist overlapping bookings in the database, violating **Business Rule 3** [1].
* **How It Was Fixed:** 
  Thread-locked the critical section of the booking validation and database insertion flow using a global mutex:
  ```python
  import threading
  _booking_lock = threading.Lock()
  # Wrapped within create_booking:
  with _booking_lock:
      ...
  ```

---

### **Bug 11: Cancel Booking Refund notice levels & Cache Invalidation**
* **File:** `app/routers/bookings.py`
* **Line Numbers:** 213 and 231–232
* **What the Bug Was & Why It Caused Incorrect Behavior:** 
  Notice logic checks on line 213 checked if `notice_hours > 48` (assigning a 48-hour notice exactly 50% instead of 100%) and defaulted to 50% refund for notices under 24 hours (instead of 0%), violating **Business Rule 6** [1]. Additionally, cancelling did not invalidate the room's availability cache [1].
* **How It Was Fixed:** 
  Standardized notice checking to use exact timedelta parameters, and added an availability cache invalidation step upon cancellations:
  ```python
  if notice >= timedelta(hours=48):
      refund_percent = 100
  elif notice >= timedelta(hours=24):
      refund_percent = 50
  else:
      refund_percent = 0
  ...
  cache.invalidate_availability(booking.room_id, booking.start_time.date().isoformat())
  ```

---

### **Bug 12: Cancel Booking Rounding**
* **File:** `app/routers/bookings.py`
* **Line Number:** 218
* **What the Bug Was & Why It Caused Incorrect Behavior:** 
  Calculations on cancellation refund amounts utilized Python's default `round()` function [1]. Python uses Banker's rounding (rounding to the nearest even number), which rounds `.5` values down in certain cases (e.g. 500.5 to 500), violating **Business Rule 6** (half-cents must round up) [1].
* **How It Was Fixed:** 
  Replaced Banker's rounding with standard mathematical rounding half-cents up:
  ```python
  refund_amount_cents = int(booking.price_cents * (refund_percent / 100.0) + 0.5)
  ```

---

### **Bug 13: Refund Ledger Log Rounding**
* **File:** `app/services/refunds.py`
* **Line Number:** 15
* **What the Bug Was & Why It Caused Incorrect Behavior:** 
  The ledger logging logic truncated the refund decimals via `int()` [1]. This created mismatching refund amounts between the cancellation endpoint's return value and the actual database ledger log record, violating **Business Rule 6** [1].
* **How It Was Fixed:** 
  Matched the rounding strategy to use mathematical rounding half-cents up:
  ```python
  amount_cents = int(booking.price_cents * (percent / 100.0) + 0.5)
  ```

---

### **Bug 14: Reference Code Race Condition**
* **File:** `app/services/reference.py`
* **Line Numbers:** 20–24
* **What the Bug Was & Why It Caused Incorrect Behavior:** 
  Reference code generation used an unprotected 0.12-second sleep delay between reading and incrementing the monotonic counter [1]. This caused parallel booking requests to retrieve identical sequence counts and assign duplicate reference codes [1].
* **How It Was Fixed:** 
  Thread-locked the counter increments to serialize code generation:
  ```python
  import threading
  _counter_lock = threading.Lock()
  # Inside next_reference_code():
  with _counter_lock:
      ...
  ```

---

### **Bug 15: Room Statistics Thread Safety**
* **File:** `app/services/stats.py`
* **Line Numbers:** 17–33
* **What the Bug Was & Why It Caused Incorrect Behavior:** 
  Room stats read-and-write modifications contained thread sleep delays [1]. Under concurrent booking creation or cancellation events, statistics updates would overwrite each other, causing revenue and booking count discrepancies [1].
* **How It Was Fixed:** 
  Thread-locked stats tracking operations to guarantee consistency across concurrent operations:
  ```python
  import threading
  _stats_lock = threading.Lock()
  # Inside record_create() & record_cancel():
  with _stats_lock:
      ...
  ```

---

### **Bug 16: Rate Limit Concurrency Bypass**
* **File:** `app/services/ratelimit.py`
* **Line Numbers:** 21–30
* **What the Bug Was & Why It Caused Incorrect Behavior:** 
  The rate-limiting bucket evaluations featured a thread sleep without synchronization [1]. Concurrent requests would read identical bucket values and bypass rate limit checks [1].
* **How It Was Fixed:** 
  Protected rate-limiting bucket checks using a global `_rate_limit_lock`:
  ```python
  import threading
  _rate_limit_lock = threading.Lock()
  # Inside record_and_check():
  with _rate_limit_lock:
      ...
  ```

---

### **Bug 17: Deadlock in Out-of-Band Notifications**
* **File:** `app/services/notifications.py`
* **Line Numbers:** 31–34
* **What the Bug Was & Why It Caused Incorrect Behavior:** 
  `notify_created` acquired locks in the order `_email_lock` -> `_audit_lock`, while `notify_cancelled` acquired locks in the order `_audit_lock` -> `_email_lock` [1]. This inconsistent locking sequence caused permanent thread deadlocks, violating the application's liveness constraints [1].
* **How It Was Fixed:** 
  Standardized lock acquisition to always request `_email_lock` prior to `_audit_lock` in both workflows:
  ```python
  def notify_cancelled(booking) -> None:
      with _email_lock:
          with _audit_lock:
              _write_audit("cancelled", booking)
          _send_email("cancelled", booking)
  ```

---

### **Bug 18: Datetime Output Suffix Non-Compliance**
* **File:** `app/timeutils.py`
* **Line Number:** 20
* **What the Bug Was & Why It Caused Incorrect Behavior:** 
  The response helper `iso_utc()` generated strings with a `+00:00` suffix [1]. However, strict contract tests and clients expected the standard **`Z`** UTC designator, causing integration assertions to fail [1].
* **How It Was Fixed:** 
  Modified the formatting output to replace `+00:00` with the `Z` suffix cleanly:
  ```python
  def iso_utc(dt: datetime) -> str:
      return dt.replace(tzinfo=timezone.utc).isoformat().replace("+00:00", "Z")
  ```