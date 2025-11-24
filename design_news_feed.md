# System Design: Facebook News Feed

## 1. Requirements

### Functional Requirements
1.  **News Feed**: Users see a list of posts from friends and pages they follow.
2.  **Post**: Users can publish new posts (text, image, video).
3.  **Pagination**: Users can scroll infinitely to see more posts.
4.  **Real-time Updates (Optional)**: New posts appear automatically (or via "New Posts" button).

### Non-Functional Requirements
1.  **Latency**: Generating a feed must be extremely fast (under 200ms).
2.  **Availability**: High availability is critical. It's okay if a post takes a few seconds to appear (Eventual Consistency).
3.  **Scalability**: Must handle billions of users and posts.

## 2. Capacity Estimation

### Traffic & Storage
*   **DAU**: 1 Billion
*   **QPS (Read)**: Assume 10 reads per user/day -> $1B * 10 / 86400 \approx 115,000$ QPS.
*   **QPS (Write)**: Assume 1 post per user/day -> $1B / 86400 \approx 11,500$ QPS.
*   **Storage**: Posts are large (media), but metadata is small.

## 3. High-Level Architecture


## 4. Detailed Design

### A. Feed Publishing Models (Push vs Pull)

#### 1. Pull Model (Fan-out on Load)
*   **How**: When User A loads their feed, the system queries all people User A follows, fetches their recent posts, merges them, and sorts them by time.
*   **Pros**: Simple for write (just save post). Good for users with few friends.
*   **Cons**: **Slow read**. If User A follows 1000 people, we need 1000 DB queries (or complex joins) every time they load the page.

#### 2. Push Model (Fan-out on Write) - **Recommended for most users**
*   **How**: When User B posts, the system immediately finds all of User B's followers (say, User A and C) and pushes the `post_id` into User A's and User C's pre-computed feed cache.
*   **Pros**: **Instant read**. User A just reads their cache (O(1)).
*   **Cons**: **Slow write** for celebrities. If Justin Bieber posts (100M followers), we can't push to 100M caches instantly (The "Celebrity Problem").

#### 3. Hybrid Model (Smart Strategy)
*   **Normal Users**: Use **Push Model**.
*   **Celebrities**: Use **Pull Model**. When User A loads their feed, we fetch their "Push Cache" (normal friends) and merge it with "Pull" updates from celebrities they follow.

### B. Database Schema
*   **Users**: `user_id`, `name`, `profile_pic`.
*   **Posts**: `post_id`, `user_id`, `content`, `media_url`, `created_at`.
*   **Follows**: `follower_id`, `followee_id` (Best in a Graph DB or Cassandra).
*   **Feed Cache**: Key-Value store (Redis). Key: `user_id`, Value: `List<post_id>`.

## 5. API Design

1.  **Get News Feed**
    *   `GET /feed?cursor=xyz`
    *   Response: `{ "data": [ { "postId": "...", "content": "..." } ], "nextCursor": "abc" }`
2.  **Create Post**
    *   `POST /posts`
    *   Body: `{ "content": "Hello World", "mediaIds": [...] }`
