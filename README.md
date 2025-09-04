# Cinema Booking System — Design (Mermaid)

> **Candidate:** _Esaú Jaret Gamboa Torres_  
> **Date:** _2025-09-03_  
> **Scope:** Conceptual and logical design only. No code implementation.

---

## 0)Non‑Functional Notes
- The cinema chain has multiple **rooms**, each with fixed **seats**.  
- A **screening** is a scheduled movie show in a room at a start time and (known) end time.  
- Users can **search** by movie title and date, **view availability**, **book specific seats**, and **cancel** bookings.  
- No payment processing is considering for this test; cancellation policy is “free before start time”.  
- Concurrency matters: Friday peaks may approach **4,500 online attendees** (example). We design for:
  - **Seat holds** (short‑lived locks) to avoid double booking.
  - **Idempotent** booking requests and safe retries.
  - Fast lookups via **room/seat map caching** and **screening availability** projections.

---

## 1) Entity‑Relationship Diagram

```mermaid
erDiagram
	user {
		uuid id PK ""  
		string email  ""  
		string password_hash  ""  
		string first_name  ""  
		string last_name  ""  
		datetime created_at  ""  
	}

	booking {
		uuid id PK ""  
		uuid user_id FK ""  
		uuid screening_id FK ""  
		string status  ""  
		string reference_code  ""  
		datetime created_at  ""  
		datetime confirmed_at  ""  
		datetime cancelled_at  ""  
		string cancellation_reason  ""  
		string idempotency_key  ""  
	}

	movie {
		uuid id PK ""  
		string title  ""  
		string synopsis  ""  
		int duration_minutes  ""  
		string rating  ""  
		date release_date  ""  
		datetime created_at  ""  
		datetime updated_at  ""  
	}

	screening {
		uuid id PK ""  
		uuid movie_id FK ""  
		uuid room_id FK ""  
		datetime starts_at  ""  
		datetime ends_at  ""  
		string status  ""  
		int base_price_cents  ""  
	}

	room {
		uuid id PK ""  
		string name  ""  
		int seat_rows  ""  
		int seat_cols  ""  
		int capacity  ""  
	}

	seat {
		uuid id PK ""  
		uuid room_id FK ""  
		string label  ""  
		int row_index  ""  
		int col_index  ""  
	}

	booking_seat {
		uuid id PK ""  
		uuid booking_id FK ""  
		uuid seat_id FK ""  
	}

	seat_hold {
		uuid id PK ""  
		uuid user_id FK ""  
		uuid screening_id FK ""  
		uuid seat_id FK ""  
		datetime expires_at  ""  
		string hold_token  ""  
	}

	user||--o{booking:"places"
	movie||--o{screening:"scheduled"
	room||--o{screening:"hosts"
	room||--o{seat:"contains"
	screening||--o{booking:"has"
	booking||--o{booking_seat:"includes"
	seat||--o{booking_seat:"assigned"
	screening||--o{seat_hold:"temp_lock"
	user||--o{seat_hold:"creates"


```

**Notes**
- `SEAT_HOLD` prevents race conditions by locking a seat for a short TTL (e.g., 5 minutes). Expired holds are cleaned by a job.
- `BOOKING.status` transitions ensure clean cancellation/expiry.

---

## 2) Class Design (Key Services & Entities)

```mermaid
classDiagram
  direction LR

  class User {
    +uuid id
    +string email
    +string passwordHash
    +string firstName
    +string lastName
  }

  class Movie {
    +uuid id
    +string title
    +int durationMinutes
    +string rating
    +create()
    +update()
    +delete()
  }

  class Room {
    +uuid id
    +string name
    +int capacity
    +getSeatMap(): Seat[]
  }

  class Seat {
    +uuid id
    +string label
    +int rowIndex
    +int colIndex
  }

  class Screening {
    +uuid id
    +uuid movieId
    +uuid roomId
    +Date startsAt
    +Date endsAt
    +string status
  }

  class Booking {
    +uuid id
    +uuid userId
    +uuid screeningId
    +string status
    +confirm()
    +cancel(reason)
  }

  class BookingSeat {
    +uuid id
    +uuid bookingId
    +uuid seatId
  }

  class SeatHold {
    +uuid id
    +uuid userId
    +uuid screeningId
    +uuid seatId
    +Date expiresAt
  }

  class AuthService {
    +register(email, pwd): User
    +login(email, pwd): Token
    +validateToken(token): User
  }

  class MovieService {
    +addMovie(dto): Movie
    +updateMovie(id, dto): Movie
    +deleteMovie(id)
    +listMovies(filter): Movie[]
  }

  class ScreeningService {
    +schedule(dto): Screening
    +reschedule(id, dto): Screening
    +cancel(id)
    +findByTitleAndDate(title, date): Screening[]
    +getAvailability(screeningId): Availability
  }

  class SeatLockManager {
    +holdSeats(userId, screeningId, seatIds): HoldToken
    +renewHold(holdToken): void
    +releaseHold(holdToken): void
    +expireHolds(): void
  }

  class BookingService {
    +createBooking(userId, screeningId, seatIds, idempotencyKey): Booking
    +confirmBooking(bookingId): Booking
    +cancelBooking(bookingId, reason): Booking
    +getUserBookings(userId): Booking[]
  }

  class UserService {
    +getProfile(userId): User
    +updateProfile(userId, dto): User
  }

  class Availability {
    +Seat[] available
    +Seat[] held
    +Seat[] booked
  }

  AuthService ..> User : returns
  MovieService ..> Movie
  ScreeningService ..> Screening
  ScreeningService ..> Availability
  SeatLockManager ..> SeatHold
  BookingService ..> Booking
  BookingService ..> BookingSeat
  Booking --* BookingSeat
  Screening --* Booking
  Screening --* SeatHold
  Room --* Seat
```

**Method Highlights**
- `SeatLockManager.holdSeats` enforces atomic holds; fails if any seat is unavailable.  
- `BookingService.createBooking` validates an active hold and writes `BOOKING` + `BOOKING_SEAT` atomically; idempotency by `idempotencyKey`.

---

## 3)Sequence Diagram- Reserve Seats

```mermaid
sequenceDiagram
  autonumber
  actor U as User
  participant FE as Frontend (Web/Mobile)
  participant AS as AuthService
  participant SS as ScreeningService
  participant LM as SeatLockManager
  participant BS as BookingService

  U->>FE: Search "Inception" on 2025-09-05
  FE->>SS: findByTitleAndDate(title, date)
  SS-->>FE: [Screening...]
  U->>FE: Select screening #S1
  FE->>SS: getAvailability(S1)
  SS-->>FE: SeatMap [available/held/booked]
  U->>FE: Pick seats [A10, A11]
  FE->>LM: holdSeats(userId, S1, [A10, A11])
  LM-->>FE: holdToken (expiresAt)
  note over FE: Show countdown (e.g., 5 minutes)
  U->>FE: Confirm reservation
  FE->>BS: createBooking(userId, S1, [A10, A11], idemKey)
  BS-->>FE: Booking(id, status=HELD)
  FE->>BS: confirmBooking(bookingId)
  BS-->>FE: Booking(status=CONFIRMED, reference_code)
  FE-->>U: Show confirmation & QR/code
```

---

## 4) Sequence Diagram — Cancel Reservation

```mermaid
sequenceDiagram
  autonumber
  actor U as User
  participant FE as Frontend
  participant BS as BookingService
  participant LM as SeatLockManager

  U->>FE: Open My Bookings
  FE->>BS: getUserBookings(userId)
  BS-->>FE: [Booking...]
  U->>FE: Cancel booking #B123
  FE->>BS: cancelBooking(B123, reason="User request")
  BS-->>FE: Booking(status=CANCELLED)
  note over BS,LM: Seats become available (release any residual holds)
  FE-->>U: Cancellation confirmed
```

---

## 5) State Diagrams

### 5.1 Booking Lifecycle
```mermaid
stateDiagram-v2
  [*] --> HELD
  HELD --> CONFIRMED: confirm()
  HELD --> EXPIRED: hold TTL elapsed
  CONFIRMED --> CANCELLED: cancel(reason)
  EXPIRED --> [*]
  CANCELLED --> [*]
```

### 5.2 Seat Availability (per Screening)
```mermaid
stateDiagram-v2
  [*] --> AVAILABLE
  AVAILABLE --> HELD: holdSeats()
  HELD --> AVAILABLE: releaseHold() / TTL expired
  HELD --> BOOKED: booking confirmed
  BOOKED --> AVAILABLE: booking cancelled before start
  BOOKED --> [*]: after screening
```

### 5.3 Screening Status
```mermaid
stateDiagram-v2
  [*] --> SCHEDULED
  SCHEDULED --> OPEN: sales window start
  OPEN --> CLOSED: sales window end / starts_at reached
  SCHEDULED --> CANCELLED: operational cancel
  CLOSED --> [*]
  CANCELLED --> [*]
```

---

## 6) APIs

- **Auth**: `POST /auth/register`, `POST /auth/login`, `GET /profile`  
- **Movies**: `POST /movies`, `PUT /movies/{id}`, `DELETE /movies/{id}`, `GET /movies?title=`  
- **Screenings**: `POST /screenings`, `PUT /screenings/{id}`, `DELETE /screenings/{id}`, `GET /screenings?title=&date=`  
- **Availability**: `GET /screenings/{id}/availability`  
- **Holds**: `POST /screenings/{id}/holds` (seatIds), `DELETE /holds/{token}`  
- **Bookings**: `POST /bookings` (idempotency-key), `POST /bookings/{id}/confirm`, `POST /bookings/{id}/cancel`, `GET /my/bookings`

---feature/esau-gamboa-movie-app
