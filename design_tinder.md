# System Design: Tinder (Dating App)

## 1. Requirements

### Functional Requirements
1.  **Profile**: Users can create a profile with photos and bio.
2.  **Recommendation**: Users see a stream of potential matches nearby.
3.  **Swipe**: Users can swipe Right (Like) or Left (Pass).
4.  **Match**: If both users swipe right, it's a match.

### Non-Functional Requirements
1.  **Low Latency**: Swiping must be instantaneous. Profile loading should be fast.
2.  **Consistency**: "Match" consistency is important (users shouldn't miss a match), but "Feed" consistency can be eventual.
3.  **Scalability**: Must handle millions of active users and billions of swipes per day.

## 2. Capacity Estimation

### Traffic & Storage
*   **DAU**: 50 Million
*   **Swipes per day**: $50M * 20 \text{ swipes} = 1 \text{ Billion swipes/day}$.
*   **Matches**: Assume 1% match rate -> 10 Million matches/day.
*   **Storage**: Images are the biggest consumer. Metadata is small.

## 3. High-Level Architecture
![Uploading Screenshot 2025-11-24 at 2.00.16 PM (2).png…]()


## 4. Detailed Design

### A. Geo-Spatial Indexing (Finding Nearby Users)
How to efficiently find "Active users within 10km"?

#### Option 1: SQL (`SELECT * FROM users WHERE lat BETWEEN ...`)
*   **Cons**: Too slow for millions of users.

#### Option 2: Geohash (Redis/PostGIS) - **Recommended**
*   **Concept**: Divide the world into a grid. Each grid cell has a unique string ID (Geohash). Nearby points have similar prefixes.
*   **Implementation**: Store `Geohash -> List<UserID>` in Redis.
*   **Query**: To find users near me, calculate my Geohash (e.g., `u4pru`) and query Redis for users in `u4pru` and its 8 neighbors.

### B. The "Match" Logic (Atomic Locking Strategy)
To ensure consistency and handle concurrent swipes (e.g., A and B swipe each other at the exact same time), we use a **Single Row per Pair** strategy with database locking.

1.  **Ordering**: Always sort user IDs so `user1_id < user2_id`. This ensures that whether A swipes B or B swipes A, we always access the same database row.
2.  **Schema**: The `Swipes` table has columns: `user1_id`, `user2_id`, `user1_likes_user2` (bool), `user2_likes_user1` (bool).
3.  **Flow**:
    *   User A swipes right on User B.
    *   Server determines `u1 = min(A, B)` and `u2 = max(A, B)`.
    *   **Transaction Start**:
        *   **Lock Row**: `SELECT * FROM Swipes WHERE user1_id = u1 AND user2_id = u2 FOR UPDATE`.
        *   **Update**: Set the appropriate column (e.g., if A is u1, set `user1_likes_user2 = TRUE`).
        *   **Check Match**: If *both* columns are now `TRUE`, it's a match!
    *   **Transaction Commit**.
4.  **Result**: If a match is found, create a record in `MatchDB` and notify users. This guarantees strict consistency and prevents race conditions.

### C. Database Schema
*   **Users**: `user_id`, `name`, `bio`, `location (lat, long)`, `last_active`.
*   **Swipes**: `user1_id`, `user2_id`, `user1 likes user2`, `user2 likes user1`.
*   **Matches**: `match_id`, `user1_id`, `user2_id`, `timestamp`.

## 5. API Design

1.  **Update Location** (Background)
    *   `POST /user/location` -> `{ "lat": 37.77, "long": -122.41 }`
2.  **Get Recommendations**
    *   `GET /recs?radius=10km`
3.  **Swipe**
    *   `POST /swipe`
    *   Body: `{ "targetUserId": "123", "action": "LIKE" }`
    *   Response: `{ "isMatch": true, "matchId": "xyz" }`
