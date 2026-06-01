---
sidebar_position: 2
---

# Movie Ticketing Entities

## Core Entities

### BookingSystem

- Acts as the orchestrator for the movie ticketing system.
- Owns the list of theaters.
- Provides the main entry point for search, booking, and cancellation operations.

### Theater

- Represents one cinema location.
- Stores metadata such as:
  - `theaterId`
  - `name`
  - `location`
- Contains multiple screens / halls.
- Contains the showtimes available at that theater.

### Screen

- Represents one movie hall inside a theater.
- Stores metadata such as:
  - `screenId`
  - `seatLayout`
- Owns the physical seat layout, not the per-showtime booking state.

### Showtime

- Represents one specific movie showing at a specific theater, screen, and time.
- Stores:
  - `showtimeId`
  - `theaterId`
  - `screen`
  - `movie`
  - `startTime`
  - `reservations`
  - `seatAvailability` / `reservedSeats`
- Owns seat booking state because the same screen can be reused across different showtimes.

### Movie

- Stores movie metadata used for searching and display.
- Example fields:
  - `movieId`
  - `title`

### Reservation

- Represents a successful booking for one showtime.
- Stores:
  - `confirmationId`
  - `showtime`
  - `seatIds`

## Key Relationships

```text
BookingSystem -> List<Theater>

Theater -> List<Screen>
Theater -> List<Showtime>

Screen -> screenId
Screen -> SeatLayout

Showtime -> Movie
Showtime -> Theater
Showtime -> Screen
Showtime -> startTime
Showtime -> SeatAvailability / ReservedSeats
Showtime -> List<Reservation>

Reservation -> confirmationId
Reservation -> Showtime
Reservation -> List<SeatId>
```

## Ownership Rule

- `Screen` owns the physical seat layout.
- `Showtime` owns the availability and reservation state for those seats.