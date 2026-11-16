# Seat Classes Plan: Economy, Business, and Galaxium

## Overview

Galaxium Travels already has a **nearly complete** seat class implementation across its full stack.
The three classes — Economy, Business, and Galaxium — are modeled in the database, exposed via the
REST API and MCP tools, and displayed in the frontend with per-class pricing, seat availability, and
booking flow.

**Confirmed multipliers (user decision):** Economy 1×, Business 2×, Galaxium 4×.

**Two gaps exist:**
1. The Java `PricingService.java` uses wrong class names and static prices disconnected from the
   Python backend — causing silent price discrepancies between the quote and the final booking.
2. The `FlightCard` and `BookingModal` show availability counts and prices, but the per-class
   **feature/benefit descriptions** are only visible inside the booking modal. They should be more
   prominent in the flight listing card.

---

## Business Logic Answers

### 1. Does each seat class have its own availability counter per flight?

**Yes.** Each `Flight` row in the database has three independent counters:
- `economy_seats_available` — 60% of total seats (e.g. 6 seats)
- `business_seats_available` — 30% of total seats (e.g. 3 seats)
- `galaxium_seats_available` — 10% of total seats (e.g. 1 seat)

When a booking is made, only the counter for the selected class is decremented. When cancelled,
only that class's counter is restored. The classes do **not** share a common pool — booking
Economy has zero effect on the Business or Galaxium seat counts.

Source: `booking_system_backend/services/booking.py` lines 36-76,
        `booking_system_backend/models.py` lines 28-30.

### 2. Is the booked seat class visible in the user's booking history?

**Yes, fully.** The `BookingCard` component already:
- Shows the class name ("Economy", "Business", "Galaxium Class")
- Renders a matching icon (Plane / Crown / Rocket) in the class's accent colour
  (blue / purple / alien-green)
- Displays the `price_paid` field beneath the class label

This is visible on the **MyBookings** page (`/my-bookings`) for both active and past bookings.

Source: `booking_system_frontend/src/components/bookings/BookingCard.tsx` lines 15-46, 126-141.

### 3. Does FlightCard show seat availability per class?

**Yes, already.** Each of the three seat-class rows in `FlightCard` shows:
- The class name and icon
- The price for that class (`economy_price`, `business_price`, `galaxium_price`)
- The number of seats remaining (`X left`), or "Sold Out" if zero
- A warning colour (solar-orange) when only 1–2 seats remain

What is **not** shown in `FlightCard` today are the feature/benefit bullets for each class. Those
currently only appear inside the `BookingModal` after the user clicks "Select Seat Class".
Sub-Task 4 addresses this.

Source: `booking_system_frontend/src/components/flights/FlightCard.tsx` lines 30-155.

---

## Sub-Task 1 — Fix Java PricingService: add `galaxium`, align multipliers with Python backend

**Status:** `[ ] pending`

### Intent
`PricingService.java` currently maps only `economy`, `business`, and `first` to static hardcoded
prices, with a spurious per-flight noise factor (`flightId % 3`). When a user requests a Galaxium
quote, `"galaxium"` falls through to the economy default price. The quoted price has no relationship
to the actual `base_price` of the flight visible in the flight listing, causing a silent discrepancy
between what the user sees at selection time and what they are charged at booking.

The fix fetches the flight's `base_price` from the Python backend and applies the agreed multipliers
so that quoted prices match the booking prices exactly.

### Expected Outcomes
- `"galaxium"` quote price = `base_price × 4`
- `"business"` quote price = `base_price × 2`
- `"economy"` quote price = `base_price × 1`
- The noise factor (`flightId % 3`) is removed
- `"first"` is removed as a valid class; an unknown class returns a clear error
- Quoted price in the modal matches the price charged at booking

### Todo List
1. In `PricingService.java`, replace `BASE_PRICES` with a `MULTIPLIERS` map:
   `{"economy": 1.0, "business": 2.0, "galaxium": 4.0}`.
2. Add a method to `PythonBackendClient.java` (or reuse an existing one) to call the Python
   `GET /api/flights` endpoint, filter by `flight_id`, and return the `base_price` integer.
3. Update `calculatePrice(flightId, seatClass)` to: fetch `base_price` from Python, look up the
   multiplier, return `(long)(base_price * multiplier)`.
4. If `seatClass` is not in the multiplier map, throw `IllegalArgumentException` with a clear message.
5. Update any `@Value` / `application.properties` entries if the Python backend URL is not already
   wired through `PythonBackendClient`.

### Relevant Context
- File to edit: `booking_system_inventory_hold_service/src/main/java/com/galaxium/holdservice/service/PricingService.java`
- HTTP client: `booking_system_inventory_hold_service/src/main/java/com/galaxium/holdservice/client/PythonBackendClient.java`
- Python source of truth: `booking_system_backend/services/booking.py` lines 8-13
- Called from: `booking_system_inventory_hold_service/src/main/java/com/galaxium/holdservice/service/QuoteService.java` line 36

---

## Sub-Task 2 — Update Python multipliers to 1×/2×/4×

**Status:** `[ ] pending`

### Intent
The Python backend currently uses multipliers of 1.0×, 2.5×, and 5.0×. The user has confirmed the
correct values should be 1×, 2×, and 4×. These must be updated so that the booking price (Python)
and the quote price (Java, after Sub-Task 1) use identical multipliers.

The `FlightOut` schema computes `economy_price`, `business_price`, and `galaxium_price` from
`base_price` using these multipliers, so the flight listing will also reflect the new prices
automatically once updated.

### Expected Outcomes
- `economy_price = base_price × 1` (unchanged)
- `business_price = base_price × 2` (was 2.5×)
- `galaxium_price = base_price × 4` (was 5.0×)
- `price_paid` on new bookings reflects the new multipliers
- Seed data prices in existing tests are updated if they assert specific `price_paid` values

### Todo List
1. In `booking_system_backend/services/booking.py`, change `SEAT_CLASS_MULTIPLIERS` to:
   `{'economy': 1.0, 'business': 2.0, 'galaxium': 4.0}`.
2. Check `booking_system_backend/services/flight.py` — the `_flight_to_out()` function computes
   `economy_price`, `business_price`, `galaxium_price`; update those multipliers to match.
3. Run `pytest` and fix any test that asserts a specific `price_paid` or computed price value that
   assumed the old multipliers.

### Relevant Context
- Multipliers: `booking_system_backend/services/booking.py` lines 8-13
- Computed prices for FlightOut: `booking_system_backend/services/flight.py` (look for `_flight_to_out`)
- Tests: `booking_system_backend/tests/test_services.py`, `booking_system_backend/tests/test_rest.py`

---

## Sub-Task 3 — Add backend tests for seat class pricing and per-class booking

**Status:** `[ ] pending`

### Intent
The existing test suite has no test that verifies the calculated `price_paid` when booking in
`business` or `galaxium` class, nor a REST-level test for `seat_class` passed through the API.
Adding these tests makes the pricing logic (after the multiplier update in Sub-Task 2)
regression-proof.

### Expected Outcomes
- Test: booking in `business` class → `price_paid == base_price * 2`
- Test: booking in `galaxium` class → `price_paid == base_price * 4`
- Test: cancelling a `business` booking → `business_seats_available` restored (not economy)
- REST test: `POST /book` with `seat_class: "galaxium"` → response has `seat_class == "galaxium"`
- All tests pass with `pytest`

### Todo List
1. In `test_services.py` under `TestBookingService`, add `test_book_flight_business_pricing`:
   create a flight with `base_price=1000`, book business, assert `price_paid == 2000` and
   `business_seats_available` decremented.
2. Add `test_book_flight_galaxium_pricing`: same pattern, assert `price_paid == 4000`.
3. Add `test_cancel_booking_business_restores_business_seat`: create a business booking manually,
   cancel it, verify `business_seats_available` incremented (not economy).
4. In `test_rest.py`, add `test_book_flight_galaxium_seat_class`: call `POST /book` with
   `seat_class: "galaxium"`, assert response `seat_class == "galaxium"` and `price_paid == base_price * 4`.
5. Run `pytest` to confirm all pass.

### Relevant Context
- Existing tests: `booking_system_backend/tests/test_services.py` class `TestBookingService` (line 267)
- Existing REST tests: `booking_system_backend/tests/test_rest.py`
- Conftest: `booking_system_backend/tests/conftest.py` — in-memory SQLite, `dependency_overrides`
- Depends on Sub-Task 2 (multipliers must be updated first so test assertions are correct)

---

## Sub-Task 4 — Add Java unit tests for the fixed PricingService

**Status:** `[ ] pending`

### Intent
After Sub-Task 1 fixes `PricingService.java`, a unit test makes the new logic regression-proof and
documents the expected multiplier for each class.

### Expected Outcomes
- Test: `economy` → `base_price × 1`
- Test: `business` → `base_price × 2`
- Test: `galaxium` → `base_price × 4`
- Test: unknown class → `IllegalArgumentException` (or appropriate error response)
- `mvn test-compile` and `mvn test` pass

### Todo List
1. Create `PricingServiceTest.java` under
   `booking_system_inventory_hold_service/src/test/java/com/galaxium/holdservice/service/`.
2. Use Mockito to mock the `PythonBackendClient` (or HTTP call) so the test returns a fixed
   `base_price` (e.g. 1 000 000).
3. Assert `calculatePrice(flightId, "economy") == 1_000_000`.
4. Assert `calculatePrice(flightId, "business") == 2_000_000`.
5. Assert `calculatePrice(flightId, "galaxium") == 4_000_000`.
6. Assert that an unknown class throws `IllegalArgumentException`.
7. Run `mvn test`.

### Relevant Context
- Depends on Sub-Task 1 (needs the final signature of the fixed `PricingService.calculatePrice`)
- Existing Java tests: `booking_system_inventory_hold_service/src/test/java/com/galaxium/holdservice/`
- Build file: `booking_system_inventory_hold_service/pom.xml`
- After Sub-Task 1 completes, record the final method signature here before starting Sub-Task 4.

---

## Sub-Task 5 — Make seat class features more prominent in FlightCard and BookingCard

**Status:** `[ ] pending`

### Intent
The per-class feature/benefit bullets (e.g. "Luxury pods", "Personal concierge", "Zero-G
experience") currently only appear inside the `BookingModal` after the user has already clicked
"Select Seat Class". A user browsing the flight listing cannot compare class benefits without
opening the modal. Making the highlights visible on the card helps users make an informed choice
earlier.

The goal is **not** to replicate the full bullet list from `BookingModal` onto `FlightCard` —
that would make each card too tall. Instead, add one short tagline per class beneath the price/seats
row in `FlightCard`, and add a small "perks badge" strip to `BookingCard` so the booked class
benefits are summarised in the history view.

### Expected Outcomes
- `FlightCard`: each seat-class row shows a one-line tagline (e.g. "Standard seating · Snacks",
  "Priority boarding · Gourmet meals", "Luxury pod · Zero-G experience").
- `BookingCard`: the seat class section shows 1–2 short perk highlights beneath the class name.
- The taglines/perks are defined as data constants (not duplicated in both components).
- No layout regressions on existing card dimensions; taglines collapse gracefully on small screens.
- `tsc -b` and ESLint pass with no new errors.

### Todo List
1. Create `booking_system_frontend/src/data/seatClasses.ts` — export a `SEAT_CLASS_CONFIG` map
   keyed by `SeatClass` with: `name`, `tagline`, `perks: string[]`, `icon`, `color`, `bgColor`,
   `borderColor`.
2. Update `FlightCard.tsx`: replace the inline `seatClasses` array with values from
   `SEAT_CLASS_CONFIG`; add a `<p>` tagline line beneath the icon+name row in each seat class tile.
3. Update `BookingModal.tsx`: replace its inline `seatClasses` array with `SEAT_CLASS_CONFIG` so
   the full `features` list in the select step still comes from the same source.
4. Update `BookingCard.tsx`: beneath the `getSeatClassName()` label, render a 1–2 item perk
   summary from `SEAT_CLASS_CONFIG[booking.seat_class].perks`.
5. Run `tsc -b` and ESLint; fix any type errors.

### Relevant Context
- `FlightCard.tsx`: `booking_system_frontend/src/components/flights/FlightCard.tsx` lines 30-61
  (inline `seatClasses` array to be replaced)
- `BookingModal.tsx`: `booking_system_frontend/src/components/bookings/BookingModal.tsx` lines 66-100
  (parallel inline array to consolidate)
- `BookingCard.tsx`: `booking_system_frontend/src/components/bookings/BookingCard.tsx` lines 15-46
  (class name/icon helpers — add perks display here)
- `SeatClass` type: `booking_system_frontend/src/types/index.ts`

---

## Implementation Order

Sub-Tasks 2 and 5 are independent of each other and of Sub-Task 1; they can be done in parallel.
Sub-Task 3 must run after Sub-Task 2 (multipliers must be final before writing price assertions).
Sub-Task 4 must run after Sub-Task 1 (needs the final `PricingService` signature).

```
Sub-Task 1  ──────────────────────────────────► Sub-Task 4
Sub-Task 2  ──────────────────────────────────► Sub-Task 3
Sub-Task 5  (independent, frontend only)
```

---

## Notes

- **No schema migrations needed**: ORM models already have all three seat class columns.
- **No new API endpoints needed**: Python already exposes everything the Java service needs.
- **Seed data**: `seed.py` already distributes bookings across all three classes; no changes needed
  unless the new multipliers cause test assertions in `conftest.py` to break.
- The price inconsistency between Quote (Java) and Booking (Python) is **silent** — the user sees
  one price in the hold step and a different one in their booking history. Sub-Tasks 1 and 2
  together close this gap completely.
