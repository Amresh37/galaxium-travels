# Galaxium Travels — Code Quality Audit

**Date:** 2025  
**Scope:** Python backend, TypeScript frontend, Java hold service  
**Branch:** `bob-learning-path-branch`

---

## 1. Overview Table

| Component | Language | Files Analysed | Critical | High | Medium | Low |
|---|---|---|:---:|:---:|:---:|:---:|
| Python Backend | Python 3.x | `server.py`, `models.py`, `db.py`, `schemas.py`, `seed.py`, `services/booking.py`, `services/flight.py`, `services/user.py`, `tests/conftest.py`, `tests/test_rest.py`, `tests/test_services.py` | 1 | 3 | 5 | 4 |
| TypeScript Frontend | TypeScript / React 19 | `src/services/api.ts`, `src/hooks/useUser.tsx`, `src/types/index.ts`, `src/pages/Flights.tsx`, `src/pages/MyBookings.tsx`, `src/components/user/UserIdentification.tsx`, `src/components/bookings/BookingModal.tsx`, `src/components/bookings/BookingCard.tsx`, `src/utils/formatters.ts`, `src/main.tsx` | 0 | 2 | 4 | 3 |
| Java Hold Service | Java | *(directory absent from repository)* | — | — | — | 1 |
| **Totals** | | | **1** | **5** | **9** | **8** |

---

## 2. Per-Component Findings

### 2.1 Python Backend

---

#### [CRITICAL] Wildcard CORS origin grants unrestricted cross-origin access

**File:** [`server.py`](../../booking_system_backend/server.py:124)  
**Lines:** 124–130

```python
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_credentials=True,
    ...
)
```

**Description:** `allow_origins=["*"]` combined with `allow_credentials=True` is a prohibited configuration in browsers (the CORS specification forbids credentialed requests when the origin is a wildcard), yet the backend still serves this setting. Any origin can issue cross-site requests that carry cookies or authorization headers. Because authentication in this system is purely name/email based and already minimal, the actual browser-enforcement gap is partially moot — but the configuration models a widely-copied anti-pattern.

**Impact:** Any web page can make credentialed requests to the API on behalf of a visiting user. Depending on how authentication evolves, this could enable cross-site request forgery.

---

#### [HIGH] No authentication or authorisation on any REST endpoint

**File:** [`server.py`](../../booking_system_backend/server.py:139)  
**Lines:** 139–183

**Description:** All REST endpoints — including `POST /book`, `POST /cancel/{booking_id}`, `GET /bookings/{user_id}`, and `GET /user` — accept requests from any caller without requiring a token, session, or API key. The sole guard for booking is a name/user_id match inside the service layer, which is a business-rule check, not an authentication check. Similarly, any caller who knows (or guesses) a `user_id` can retrieve or cancel that user's bookings.

**Impact:** Full read/write access to all booking data without any credential. Any client can enumerate bookings, cancel bookings for arbitrary users, or register unlimited users.

---

#### [HIGH] User PII passed as plain query-string parameters

**File:** [`server.py`](../../booking_system_backend/server.py:175)  
**Lines:** 175–178

```python
@app.get("/user", ...)
def get_user_endpoint(name: str, email: str, db: Session = Depends(get_db)):
```

**Description:** `name` and `email` are accepted as GET query parameters. Query strings are recorded verbatim in server access logs, browser history, proxy logs, and HTTP `Referer` headers. Email addresses are personal data under most privacy frameworks.

**Impact:** User PII (name, email) is logged in plain text by any infrastructure component that records URL paths.

---

#### [HIGH] Seed data wipes and recreates the entire database on every server startup

**File:** [`seed.py`](../../booking_system_backend/seed.py:6)  
**Lines:** 6–59

```python
def seed():
    ...
    db.query(Booking).delete()
    db.query(User).delete()
    db.query(Flight).delete()
    db.commit()
```

**Description:** `seed()` is called unconditionally inside the FastAPI `lifespan` handler. Every cold start of the server destroys all production (or staging) data and replaces it with hardcoded demo records. There is no environment guard (`if os.getenv("ENV") == "development"`) and no idempotency check (e.g., skip if data already exists).

**Impact:** Any container restart, deployment, or crash-loop in a non-dev environment will permanently destroy all real user and booking records.

---

#### [MEDIUM] No input validation on name or string fields in service layer

**Files:** [`services/user.py`](../../booking_system_backend/services/user.py:6), [`services/booking.py`](../../booking_system_backend/services/booking.py:7)  
**Lines:** `user.py:6`, `booking.py:7`

**Description:** The service functions accept `name: str` and `email: str` but impose no length, format, or content constraints beyond what Pydantic's `EmailStr` provides for email. An empty string, a string of 10,000 characters, or a string containing SQL metacharacters (benign with SQLAlchemy's ORM, but present in `ErrorResponse.details` strings built with f-strings) would all be accepted. The `BookingRequest` schema has no minimum-length validators for `name`.

**Impact:** Overly long strings can inflate database storage and degrade query performance. The absence of name normalisation means `"Alice"` and `"  alice  "` are treated as different users.

---

#### [MEDIUM] Booking status stored as unconstrained string column

**File:** [`models.py`](../../booking_system_backend/models.py:27)  
**Line:** 27

```python
status = Column(String, nullable=False)
```

**Description:** The `Booking.status` column is a plain `String` with no `CheckConstraint` or SQLAlchemy `Enum` type. Only the service layer enforces the three valid values (`"booked"`, `"cancelled"`, `"completed"`) through literal assignment, but nothing prevents a direct database write or a future code path from storing an arbitrary string.

**Impact:** Database-level invariant not enforced; data integrity depends entirely on application code. A seed or migration script error, or a direct DB edit, could introduce unrecognised statuses that break UI rendering and service logic.

---

#### [MEDIUM] No structured logging — only a bare `print()` in seed and no logging in services

**Files:** [`seed.py`](../../booking_system_backend/seed.py:59), all files under `services/`  
**Lines:** `seed.py:59`

```python
print("Database seeded with elaborate demo data!")
```

**Description:** The backend uses no logging framework (`logging`, `structlog`, etc.). Application events, errors, and operational messages are either absent or emitted via `print()`. Service functions return `ErrorResponse` objects but never emit any log entry when a business-rule failure occurs (e.g., `FLIGHT_NOT_FOUND`, `NAME_MISMATCH`). MCP tool wrappers also have no logging.

**Impact:** Operational visibility is zero. Failed booking attempts, name-mismatch errors, and DB exceptions leave no server-side trace, making incident investigation and anomaly detection impossible.

---

#### [MEDIUM] `datetime.utcnow()` is deprecated in Python 3.12+

**File:** [`services/booking.py`](../../booking_system_backend/services/booking.py:49)  
**Line:** 49

```python
booking_time=datetime.utcnow().isoformat()
```

**Description:** `datetime.utcnow()` produces a naive datetime (no timezone info) and has been deprecated since Python 3.12. The string stored in SQLite lacks a timezone suffix (no `Z` or `+00:00`), whereas seed data appends `Z` manually. This inconsistency means booking times stored via the service and times stored via seed cannot be compared or sorted reliably without extra normalisation.

**Impact:** Inconsistent datetime formatting across the dataset. When Python 3.12 deprecation becomes a removal in a future version, this will raise a `DeprecationWarning` and eventually break.

---

#### [MEDIUM] Race condition: seat decrement is not atomic

**File:** [`services/booking.py`](../../booking_system_backend/services/booking.py:19)  
**Lines:** 19, 44

```python
if flight.seats_available < 1:
    return ErrorResponse(...)
...
flight.seats_available -= 1
db.commit()
```

**Description:** The check-then-decrement sequence is not wrapped in a database-level lock (`SELECT ... FOR UPDATE`) or a conditional update (`UPDATE flights SET seats_available = seats_available - 1 WHERE seats_available > 0`). Under concurrent requests for the last available seat, two threads can both pass the `< 1` guard and both commit, driving `seats_available` to `-1`.

**Impact:** Double-booking is possible under concurrent load. `seats_available` can become negative.

---

#### [LOW] `declarative_base()` import uses deprecated path

**File:** [`models.py`](../../booking_system_backend/models.py:1)  
**Line:** 1

```python
from sqlalchemy.ext.declarative import declarative_base
```

**Description:** `sqlalchemy.ext.declarative.declarative_base` was soft-deprecated in SQLAlchemy 1.4 and will emit a deprecation warning in 2.x. The canonical import is `from sqlalchemy.orm import declarative_base`.

**Impact:** Produces deprecation warnings on SQLAlchemy 2.x; will eventually fail on a future major release.

---

#### [LOW] Test `db_session` fixture does not roll back transactions between tests

**File:** [`tests/conftest.py`](../../booking_system_backend/tests/conftest.py:26)  
**Lines:** 26–35

```python
@pytest.fixture(scope="function")
def db_session():
    Base.metadata.create_all(bind=test_engine)
    session = TestingSessionLocal()
    try:
        yield session
    finally:
        session.close()
        Base.metadata.drop_all(bind=test_engine)
```

**Description:** The fixture tears down by dropping all tables, which is correct but expensive. However, it does not use the common pattern of wrapping each test in a savepoint / nested transaction and rolling back, meaning test isolation is coarser and any test that commits without the fixture completing can affect a subsequent test within the same `scope="function"` run if exception handling diverges.

**Impact:** Test isolation is adequate for the current suite but brittle; a future test that does not commit cleanly could leave data visible to another test in edge cases.

---

#### [LOW] No test for the `GET /bookings/{user_id}` path with a non-integer `user_id`

**Files:** [`tests/test_rest.py`](../../booking_system_backend/tests/test_rest.py:139)  
**Lines:** 139–178

**Description:** The REST tests for `/bookings/{user_id}` cover empty and populated states but do not exercise invalid inputs such as a non-integer path segment (e.g., `/bookings/abc`), which FastAPI handles by returning a `422 Unprocessable Entity`. There are no tests for the `422` response shape on any endpoint.

**Impact:** No test coverage for FastAPI's built-in validation error path; any change to error response format would go undetected.

---

#### [LOW] MCP tool wrappers silently swallow the DB session if an unhandled exception occurs before `finally`

**File:** [`server.py`](../../booking_system_backend/server.py:19)  
**Lines:** 19–97 (all MCP tools)

**Description:** All MCP tool functions open `db = SessionLocal()` and close it in a `finally` block. However, if an unexpected exception is raised before `db.close()` is entered (e.g., in the `if isinstance` check on a non-ErrorResponse, non-BookingOut object), the session is still closed correctly by `finally` — but no rollback is issued before the close. The SQLAlchemy session will roll back implicitly on close for incomplete transactions, but this is implicit behaviour, not explicit.

**Impact:** Implicit rollback reliance; if connection pooling behaviour changes, uncommitted writes could persist in edge cases. This is a minor style and reliability concern.

---

### 2.2 TypeScript Frontend

---

#### [HIGH] User identity data stored in `localStorage` without any integrity check

**File:** [`src/hooks/useUser.tsx`](../../booking_system_frontend/src/hooks/useUser.tsx:12)  
**Lines:** 12–20, 26–28

```ts
const stored = localStorage.getItem(USER_STORAGE_KEY);
if (stored) {
  try {
    return JSON.parse(stored);
  } catch {
    return null;
  }
}
```

**Description:** The `UserProvider` reads the user object from `localStorage` on initialisation and uses it directly as the application's identity context without any schema validation. The parsed object is trusted as a valid `User` (with `user_id`, `name`, `email`) even if it has been tampered with — e.g., manually altered in DevTools to a different `user_id`. All subsequent API calls (booking, cancellation, viewing bookings) use this unverified `user_id`.

**Impact:** A user can locally elevate their `user_id` to impersonate another user, book flights on their behalf, or view their booking history. This is a client-side trust issue compounded by the lack of server-side authentication.

---

#### [HIGH] `isErrorResponse` parameter typed as `any`, bypassing type safety

**File:** [`src/services/api.ts`](../../booking_system_frontend/src/services/api.ts:110)  
**Lines:** 110–113

```ts
export const isErrorResponse = (
  response: any
): response is ErrorResponse => {
  return response && response.success === false;
};
```

**Description:** The `response` parameter is typed as `any`, which defeats the purpose of the type guard. TypeScript will not warn if `isErrorResponse` is called with a completely unrelated type. The guard also checks only `response.success === false`, meaning any object with `success: false` would satisfy it — including partially shaped error payloads missing `error` or `error_code`.

**Impact:** Silently accepts malformed error payloads. Type narrowing after `isErrorResponse` trusts the full `ErrorResponse` shape even if the server returned a partial or different object structure.

---

#### [MEDIUM] `catch (error: any)` used pervasively — typed error handling absent throughout

**Files:** [`src/pages/Flights.tsx`](../../booking_system_frontend/src/pages/Flights.tsx:50), [`src/pages/MyBookings.tsx`](../../booking_system_frontend/src/pages/MyBookings.tsx:41), [`src/components/bookings/BookingModal.tsx`](../../booking_system_frontend/src/components/bookings/BookingModal.tsx:46), [`src/components/user/UserIdentification.tsx`](../../booking_system_frontend/src/components/user/UserIdentification.tsx:60)  
**Lines:** Multiple

```ts
} catch (error: any) {
  toast.error(error.details || error.error || 'Failed to load bookings');
  console.error(error);
}
```

**Description:** Every `catch` block uses `error: any` and speculatively accesses `.details` and `.error` properties. `error` in a catch clause is `unknown` by TypeScript's strict rules, and the use of `: any` suppresses that protection. If the interceptor in `api.ts` throws something other than an `ErrorResponse`-shaped object (e.g., a native `Error` from a timeout), the property accesses silently return `undefined` and the `toast.error` call receives `undefined`.

**Impact:** Silent fallback to `undefined` in `toast.error` can display a blank notification to the user. The pattern also makes it harder to add structured error telemetry later.

---

#### [MEDIUM] No client-side input length or format validation before API calls

**File:** [`src/components/user/UserIdentification.tsx`](../../booking_system_frontend/src/components/user/UserIdentification.tsx:22)  
**Lines:** 22–26

```ts
if (!name.trim() || !email.trim()) {
  toast.error('Please fill in all fields');
  return;
}
```

**Description:** The only client-side validation is a blank-field check. There is no minimum/maximum length enforcement on `name`, no email format regex or `EmailStr`-equivalent check on the client before the request is sent, and no restriction on special characters. The `<Input type="email">` HTML attribute provides browser-level validation but is bypassable programmatically.

**Impact:** Unnecessary API round-trips for obviously invalid inputs; no user feedback for malformed emails beyond what the browser natively provides. Backend returns a generic error rather than a specific client-visible field validation message.

---

#### [MEDIUM] `useEffect` in `UserProvider` duplicates the `useState` initialiser's `localStorage` write

**File:** [`src/hooks/useUser.tsx`](../../booking_system_frontend/src/hooks/useUser.tsx:36)  
**Lines:** 36–41

```ts
useEffect(() => {
  if (user) {
    localStorage.setItem(USER_STORAGE_KEY, JSON.stringify(user));
  }
}, [user]);
```

**Description:** `setUser` already writes to `localStorage` on every call (lines 25–28). The `useEffect` at lines 36–41 duplicates this write on every render cycle where `user` is truthy — including the initial render when the user was loaded from `localStorage`. This means every component mount with a logged-in user writes to `localStorage` redundantly.

**Impact:** Redundant synchronous `localStorage.setItem` calls on every render with a user. Minor performance issue; also obscures the intended source of truth for the `localStorage` write.

---

#### [MEDIUM] Redirect target for unauthenticated booking page is `/flights`, not a login prompt

**File:** [`src/pages/MyBookings.tsx`](../../booking_system_frontend/src/pages/MyBookings.tsx:23)  
**Lines:** 23–26

```ts
if (!user) {
  navigate('/flights');
  return;
}
```

**Description:** When an unauthenticated user navigates directly to `/bookings`, they are silently redirected to `/flights` with no explanation. The destination page shows flight cards with no indication that the user was redirected due to an auth requirement.

**Impact:** Poor user experience; the user receives no feedback about why they were redirected. Additionally, the redirect is not guarded against a race condition with the `loadData` async call (line 27) which checks `if (!user)` a second time, meaning two separate code paths handle the same guard condition.

---

#### [LOW] `console.error` used for production error logging in page components

**Files:** [`src/pages/Flights.tsx`](../../booking_system_frontend/src/pages/Flights.tsx:52), [`src/pages/MyBookings.tsx`](../../booking_system_frontend/src/pages/MyBookings.tsx:43)

```ts
console.error(error);
```

**Description:** Caught errors are logged directly to `console.error`. There is no error reporting service integration (e.g., Sentry, Datadog) and no structured log format. In a production build, these messages are visible only in browser DevTools and leave no server-side trace.

**Impact:** Zero observability into client-side errors in production. Errors that affect users cannot be detected, triaged, or tracked without a user report.

---

#### [LOW] No 404 handling for the wildcard route — silently falls back to `<Home />`

**File:** [`src/App.tsx`](../../booking_system_frontend/src/App.tsx:17)  
**Line:** 17

```tsx
<Route path="*" element={<Home />} />
```

**Description:** Any unrecognised URL renders the Home page with no indication to the user (or to monitoring) that they have hit an invalid route. There is no dedicated 404 component and no HTTP status signal (though React SPA routing is client-side, this affects crawlers and link previews).

**Impact:** Users who follow a broken or mistyped link see the Home page without explanation. Contributes to a confusing navigation experience.

---

#### [LOW] Flight data is not re-validated on the client after booking — stale seat count displayed

**File:** [`src/pages/Flights.tsx`](../../booking_system_frontend/src/pages/Flights.tsx:75)  
**Lines:** 75–78

```ts
const handleBookingSuccess = () => {
  loadFlights();
};
```

**Description:** On booking success, the entire flight list is re-fetched. This is a full-page reload of all flight data to update a single `seats_available` counter. While functionally correct, there is no optimistic UI update and no error handling if `loadFlights` fails after a successful booking. A failure of the reload leaves the UI showing the pre-booking seat count indefinitely.

**Impact:** If `loadFlights()` throws after a successful booking, the flight card shows a stale (pre-booking) seat count. No user-visible error is surfaced for this secondary failure.

---

### 2.3 Java Hold Service

---

#### [LOW] Service directory absent from repository

**Path:** `booking_system_inventory_hold_service/` *(does not exist)*

**Description:** The Java inventory hold service referenced in the task scope and presumably in the high-level architecture diagram (`high-level-architecture.png`) is not present anywhere in the repository. No `pom.xml`, `build.gradle`, or `.java` source files were found. It is unclear whether the service was never implemented, was removed, or lives in a separate repository.

**Impact:** Any architecture-level dependencies on the hold service (e.g., seat reservation before payment, inventory locking to prevent double-booking at the application level) are unimplemented. The gap between the documented architecture and the actual codebase is not tracked in the repository.

---

## 3. Summary

| Severity | Count | Areas |
|---|:---:|---|
| Critical | 1 | CORS wildcard + credentials |
| High | 5 | No auth/authz on API, PII in query strings, destructive seed on startup, unvalidated localStorage identity, `any`-typed error guard |
| Medium | 9 | No input length validation (backend + frontend), unconstrained status column, no structured logging, deprecated `utcnow()`, non-atomic seat decrement, `catch(any)` everywhere, duplicate localStorage write, unauthenticated redirect UX |
| Low | 8 | Deprecated SQLAlchemy import, test fixture rollback pattern, missing 422 tests, implicit MCP session rollback, `console.error` in production, wildcard 404 route, stale seat count after booking, absent Java service |

The most impactful cluster is the combination of **no server-side authentication** (High), **client-side identity stored without integrity checks** (High), and **wildcard CORS with credentials** (Critical) — together these mean the entire booking system can be manipulated by any actor who can reach the server.
