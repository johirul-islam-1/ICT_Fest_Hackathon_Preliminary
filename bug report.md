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

if payload.get("jti") in _revoked_tokens: