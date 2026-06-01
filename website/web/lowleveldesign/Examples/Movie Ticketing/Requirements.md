---
sidebar_position: 1
---

# Movie Ticketing Requirements

## Functional Requirements

### Movie Discovery

- Users can search for movies by title.
- Users can browse movies playing at a given theater.

### Theater and Seat Layout

- Theaters have multiple screens.
- All screens share the same seat layout:
  - Rows: `A-Z`
  - Seats per row: `0-20`

### Showtime Seat Selection

- Users can view available seats for a showtime.
- Users can select specific seats for a showtime.

### Reservation Management

- Users can book multiple seats in a single reservation.
- A successful booking returns a confirmation ID.
- Users can cancel a reservation by confirmation ID, releasing the reserved seats.

## Concurrency Requirements

- If multiple users try to book the same seat concurrently, exactly one booking succeeds.

## Out of Scope

- Payment processing (assume payment always succeeds)
- Variable seat layouts or seat types (all seats identical)
- Rescheduling (cancel and rebook instead)
- UI / rendering