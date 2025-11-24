# System Design: Ticket Booking System e.g. ticketmaster

## 1. Requirements

### Functional Requirements
1.  **View Events**: Users can search and view event details.
2.  **Book Tickets**: Users can select and book seats for an event.
3.  **Hold Seats**: Selected seats should be held for a specific time (e.g., 5 mins) while the user pays.
4.  **Waitlist (Optional)**: If sold out, users can join a waitlist.

### Non-Functional Requirements
1.  **High Concurrency**: The system must handle massive spikes in traffic (e.g., Taylor Swift concert ticket sales).
2.  **Consistency**: **Strict Consistency** is required. Double booking is unacceptable.
3.  **Availability**: The system should be highly available for searching/viewing, but booking consistency takes precedence over availability (CAP Theorem: CP over AP for booking).
4.  **Fairness**: First come, first served.

## 2. Capacity Estimation

### Traffic & Storage
*   **DAU**: 10 Million
*   **Peak Traffic**: During popular sales, traffic can spike 100x.
*   **QPS**:
    *   Normal: 1,000 QPS
    *   Peak: 100,000+ QPS
*   **Storage**: Events and venue data are small. The main volume is bookings and active sessions.

## 3. High-Level Architecture


## 4. Detailed Design

### A. Handling Concurrency (The "Double Booking" Problem)
This is the hardest part. Thousands of users try to click the same seat at the exact same millisecond.

#### Strategy 1: Optimistic Locking (Version Control)
*   **How**: Read the seat row. When updating, check if the version number matches.
*   `UPDATE seats SET status='HELD', user_id=123, version=v2 WHERE id=1 AND version=v1`
*   **Pros**: Good for low contention.
*   **Cons**: High failure rate under high load (many users get "Sorry, someone took it").

#### Strategy 2: Redis Distributed Lock (Recommended)
*   **How**: Use Redis `SET NX` (Set if Not Exists) to lock a specific seat ID with a TTL (Time To Live) of 5 minutes.
*   **Flow**:
    1.  User clicks "Book Seat A1".
    2.  Server tries `SET seat_A1_lock "user_123" NX EX 300`.
    3.  If return is `OK`: Seat is held. Proceed to DB transaction.
    4.  If return is `NULL`: Seat is already taken. Show error.
*   **Pros**: Extremely fast, handles high concurrency well.

### B. Database Schema
*   **Events**: `event_id`, `name`, `venue_id`, `start_time`.
*   **Venues**: `venue_id`, `name`, `seat_map`.
*   **Seats**: `seat_id`, `event_id`, `section`, `row`, `number`, `status` (AVAILABLE, HELD, BOOKED).
*   **Bookings**: `booking_id`, `user_id`, `event_id`, `seat_ids`, `status` (PENDING, CONFIRMED, CANCELLED), `created_at`.

### C. Search
*   Use **Elasticsearch** for full-text search (e.g., "Taylor Swift in New York").
*   Sync data from DB to Elasticsearch asynchronously.

## 5. API Design

1.  **Search Events**
    *   `GET /events?q=concert&city=NY`
2.  **Get Seat Map**
    *   `GET /events/{id}/seats` -> Returns status of all seats (cached).
3.  **Reserve Seats**
    *   `POST /bookings/reserve`
    *   Body: `{ "eventId": 123, "seatIds": ["A1", "A2"] }`
    *   Response: `{ "bookingId": "xyz", "expiresAt": "..." }`
4.  **Confirm Booking (Payment)**
    *   `POST /bookings/{id}/confirm`

## 6. Tradeoffs

### Active Polling vs WebSockets for Seat Status
*   **Scenario**: User is looking at the seat map. Another user buys a seat. The map should update.
*   **Decision**: **WebSockets**. Pushing updates to the client is more efficient than thousands of clients polling the server every second.
