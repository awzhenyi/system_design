---
sidebar_position: 3
---

BookingService
- theatres: `List<Theatre>`
- searchMovies(title: String) -> List<Showtime>
- getShowtimes(theatre: Theatre) -> List<Showtime>
- bookSeats(showtimeId: String, seatIds: `List<String>`) -> Reservation
- cancelReservation(seatIds: `List<String>`)

```java

public class BookingSystem {
    private List<Theatre> theatres;

    public BookingSystem(List<Theatre> theatres) {
        this.theatres = theatres;
    }

    public List<Showtime> searchMovies(String title) {
        return theatres.stream()
            .flatMap(theatre -> theatre.getShowtimes().stream())
            .filter(showtime -> showtime.getMovie().getName().equalsIgnoreCase(title))
            .toList();
    }

    public List<Showtime> getShowtimes(Theatre theatre) {
        if (theatre == null) {
            return Collections.emptyList();
        }

        return theatres.stream()
            .filter(t -> t.equals(theatre))
            .findFirst()
            .map(Theatre::getShowtimes)
            .orElse(Collections.emptyList());
    }

    public synchronized Reservation bookSeats(String showtimeId, List<String> seatIds) {
        if (showtimeId == null) || CollectionUtils.isEmpty(seatIds) {
            throw new IllegalArgumentException("Invalid booking request");
        }

        Showtime showtime = showtimesById.get(showtimeId);
        if (showtime == null) {
            throw new NoSuchElementException(String.format("showtimeId not found: %s", showtimeId));
        }

        Reservation reservation = new Reservation(UUID.randomUUID().toString(), showtime, seatIds);
        showtime.book(reservation);
        
        reservationById.put(reservation.getConfirmationId(), reservation);
        return reservation;

    }

}
```

Showtime
- id: String
- theatre: Theatre
- datetime: DateTime
- screen: Screen
- movie: Movie
- reservations: `List<Reservation>`

```java
public class Showtime {

    public void book(Reservation reservation) {

        synchronized (this) {
            List<String> seatIds = reservation.getSeatIds();
            if (seatIds == null) || seatIds.isEmpty()) {
                throw new IllegalArgumentException("Must select at least one seat");
            }
            
            for (var seatId : seatIds) {
                if (!isValid(seatId)) {
                    throw new IllegalArgumentException("Invalid seat: " + seatId);
                }

                if (!isAvailable(seatId)) {
                    throw new IllegalStateException("Seat" + seatId + " is not available");

                }

                reservations.add(reservation);
            }

        }
    }


}
```

Screen
- layout: ScreenLayout
- 