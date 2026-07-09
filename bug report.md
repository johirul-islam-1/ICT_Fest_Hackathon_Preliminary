# Bug Report

1. Location

    File: app/timeutils.py

    Line Number: 13

The Bug and Why It Caused Incorrect Behavior

In app/timeutils.py, the parse_input_datetime function is meant to convert any incoming ISO 8601 datetime string into a naive UTC datetime object for storage. 
The replace(tzinfo=None) method only strips the timezone metadata from the datetime object without altering the hour, minute, or second values. Consequently, if a client sent a datetime with an offset (e.g., "2026-07-10T13:00:00+06:00"), the code simply deleted the +06:00 offset, storing it in the database as "13:00:00" instead of correctly converting it to UTC ("07:00:00").

This violated Business Rule 1, which mandates that input datetimes carrying a UTC offset must be converted to UTC before storage or comparison.

The bug was fixed by calling astimezone(timezone.utc) to perform the mathematical timezone translation to UTC before discarding the timezone info via .replace(tzinfo=None).

2. Location

    File: app/auth.py

    Line Number: 50 (or 51, depending on exact imports)

The Bug and Why It Caused Incorrect Behavior

In app/auth.py, the create_access_token function sets the expiration time (exp) of the JWT:
code Python

lifetime = timedelta(minutes=ACCESS_TOKEN_EXPIRE_MINUTES * 60)

Since ACCESS_TOKEN_EXPIRE_MINUTES is defined as 15 in config.py, multiplying it by 60 passed 900 to the minutes parameter of timedelta [1]. This created a token that expired in 900 minutes (15 hours) instead of 900 seconds (15 minutes), violating Business Rule 8 [1].
How It Was Fixed

The unnecessary multiplication by 60 was removed from the minutes parameter:
code Python

lifetime = timedelta(minutes=ACCESS_TOKEN_EXPIRE_MINUTES)

3. Part 2: Minimal Bug Report
Location

    File: app/auth.py

    Line Number: 97

The Bug and Why It Caused Incorrect Behavior

On logout, the system invalidates the presented access token by recording its unique identifier jti (JWT ID) into the _revoked_tokens blacklist [1].

However, on line 97, the token verification helper checks the token's subject/user ID (sub) against the revoked set:
code Python

if payload.get("sub") in _revoked_tokens:

Because user IDs (e.g. "1") do not match token UUIDs (e.g. "f31a78b5..."), the check always evaluated to False. This bypassed token revocation completely, leaving logged-out access tokens fully active and usable for their entire remaining lifespan [1].
How It Was Fixed

The check was updated to validate the token's "jti" claim against the _revoked_tokens set instead of "sub" [1]:
code Python

if payload.get("jti") in _revoked_tokens


4. 
Location

  - File: app/routers/auth.py
  - Line Number: 23-60 (within the /register endpoint)

The Bug and Why It Caused Incorrect Behavior

In app/routers/auth.py, the registration endpoint /register checks if a username
is already taken inside the specified organization [1]. However, if a duplicate
username is found, the endpoint silently returns the existing user's data with
an HTTP 201 Created status code [1]:

if existing is not None:
    return {
        "user_id": existing.id,
        "org_id": org.id,
        "username": existing.username,
        "role": existing.role,
    }

This violates Business Rule 15, which specifies that a duplicate username within
the organization must raise an HTTP 409 USERNAME_TAKEN error [1]. Additionally,
returning another user's details represents a critical security and data leakage
vulnerability.

Furthermore, under concurrent requests, if two users try to register the same
username simultaneously, the database unique constraint will trigger a raw
IntegrityError [1]. Because this database exception is unhandled, it crashes the
server with an unhandled HTTP 500 Internal Server Error [1].

How It Was Fixed

The code was modified to raise a handled 409 USERNAME_TAKEN exception via the
AppError exception class [1]. Additionally, all database commit() operations
were wrapped in try/except blocks to safely catch and handle any concurrent
database-level IntegrityError violations:

from sqlalchemy.exc import IntegrityError # Ensure import

@router.post("/register", status_code=201)
def register(payload: RegisterRequest, db: Session = Depends(get_db)):
    try:
        org = db.query(Organization).filter(Organization.name == payload.org_name).first()
        if org is None:
            org = Organization(name=payload.org_name)
            db.add(org)
            db.commit()
            db.refresh(org)
    except IntegrityError:
        db.rollback()
        org = db.query(Organization).filter(Organization.name == payload.org_name).first()

    existing = (
        db.query(User)
        .filter(User.org_id == org.id, User.username == payload.username)
        .first()
    )
    if existing is not None:
        raise AppError(409, "USERNAME_TAKEN", "Username already taken")

    user = User(
        org_id=org.id,
        username=payload.username,
        hashed_password=hash_password(payload.password),
        role="admin" if db.query(User).filter(User.org_id == org.id).count() == 0 else "member",
    )
    db.add(user)
    try:
        db.commit()
    except IntegrityError:
        db.rollback()
        raise AppError(409, "USERNAME_TAKEN", "Username already taken")
    db.refresh(user)
    return {
        "user_id": user.id,
        "org_id": org.id,
        "username": user.username,
        "role": user.role,
    }

5.
Location

  - File: app/routers/auth.py
  - Line Numbers: 72–75 (revocation check) and 80 (revocation call)

The Bug and Why It Caused Incorrect Behavior

In app/routers/auth.py, the token rotation endpoint /refresh decoded the refresh
token but failed to check if its unique ID (jti) had already been blacklisted or
revoked . Additionally, once verified, the endpoint did not revoke the
presented refresh token . This allowed any valid refresh token to be reused
indefinitely, violating Business Rule 8 (refresh tokens must be single-use only)
.

How It Was Fixed

Lines 72 to 75 were added to import the blacklist and verify if the token has
been revoked, and line 80 was added to revoke the token immediately after
verification to prevent reuse :

@router.post("/refresh")
def refresh(payload: RefreshRequest, db: Session = Depends(get_db)):
    data = decode_token(payload.refresh_token)
    if data.get("type") != "refresh":
        raise AppError(401, "UNAUTHORIZED", "Wrong token type")
    
    # Lines 72-75 added:
    from ..auth import _revoked_tokens, revoke_access_token
    if data.get("jti") in _revoked_tokens:
        raise AppError(401, "UNAUTHORIZED", "Token has been revoked")

    user = db.query(User).filter(User.id == int(data["sub"])).first()
    if user is None:
        raise AppError(401, "UNAUTHORIZED", "Unknown user")

    # Line 80 added:
    revoke_access_token(data)

    return {
        "access_token": create_access_token(user),
        "refresh_token": create_refresh_token(user),
        "token_type": "bearer",
    }

6. 
Location

  - File: app/routers/bookings.py
  - Line Numbers: 137–139

The Bug and Why It Caused Incorrect Behavior

In app/routers/bookings.py, the list_bookings query chain on lines 137–139
contained three database-level bugs [1]:

items = (
    base.order_by(Booking.start_time.desc(), Booking.id.asc())  # Line 137: Descending instead of ascending
    .offset(page * limit)                                      # Line 138: Page offset calculation is off by 1 page
    .limit(10)                                                 # Line 139: Hardcoded limit of 10 instead of using requested parameter
    .all()
)

This violated Business Rule 11 [1]:

1.  Sorting must be ascending by start_time (ties ascending by id).
2.  Page N with limit L must offset by (N - 1) * L (offsetting by page * limit
    completely skipped the first page of results).
3.  The page size was locked to 10 instead of respecting the dynamic limit query
    parameter.

How It Was Fixed

Lines 137–139 were updated with the corrected sorting direction, mathematically
correct offset formula, and dynamic query limits [1]:

items = (
    base.order_by(Booking.start_time.asc(), Booking.id.asc())
    .offset((page - 1) * limit)
    .limit(limit)
    .all()
)
