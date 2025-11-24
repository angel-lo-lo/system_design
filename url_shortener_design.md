# System Design: URL Shortening Service (e.g., TinyURL, bit.ly)

## 1. Requirements

### Functional Requirements
1.  **Shorten URL**: Given a long URL, return a unique short URL.
2.  **Redirect**: Given a short URL, redirect the user to the original long URL.
3.  **Custom Alias (Optional)**: Users should be able to pick a custom alias for their URL.
4.  **Expiry (Optional)**: Users can define an expiration time for the URL.

### Non-Functional Requirements
1.  **High Availability**: The system should be up and running at all times.
2.  **Low Latency**: URL redirection should happen in milliseconds.
3.  **Scalability**: The system should handle high read/write throughput.
4.  **Durability**: Shortened links should not be lost.

## 2. Capacity Estimation & Constraints

### Traffic Estimates
*   **Write Ratio**: 1
*   **Read Ratio**: 100 (Read-heavy system)
*   **New URLs per month**: 100 Million
*   **Requests per second (Write)**: $100M / (30 * 24 * 3600) \approx 40$ URLs/sec
*   **Requests per second (Read)**: $40 * 100 = 4,000$ requests/sec

### Storage Estimates
*   Assume average URL length: 100 bytes.
*   Storage per year: $100M * 100 \text{ bytes} * 12 \text{ months} \approx 120 \text{ GB}$
*   Over 5 years: $120 \text{ GB} * 5 = 600 \text{ GB}$ (Manageable with a single DB instance, but replication needed for availability).

## 3. API Design

### REST Endpoints

1.  **Create Short URL**
    *   `POST /api/v1/urls`
    *   **Request Body**:
        ```json
        {
          "longUrl": "https://www.example.com/very/long/url",
          "customAlias": "optional-alias",
          "expireDate": "2025-12-31T00:00:00Z"
        }
        ```
    *   **Response**: `201 Created`
        ```json
        {
          "shortUrl": "https://tiny.url/xyz123"
        }
        ```

2.  **Redirect**
    *   `GET /{shortUrl}`
    *   **Response**: `301 Permanent Redirect` (or `302 Found` if analytics are needed).

## 4. Database Design

Since we need to store billions of records and the relationships are simple (no complex joins), a **NoSQL store (Key-Value)** like DynamoDB, Cassandra, or Riak is suitable. However, a Relational DB (MySQL/PostgreSQL) is also fine given the storage size (600GB is not huge).

### Schema (Relational Style)
| Column | Type | Description |
| :--- | :--- | :--- |
| `id` | BIGINT | Primary Key, Auto Increment |
| `short_key` | VARCHAR(7) | The unique short code (Indexed) |
| `long_url` | VARCHAR(2048) | The original URL |
| `created_at` | DATETIME | Creation timestamp |
| `expires_at` | DATETIME | Expiration timestamp |

## 5. High-Level Architecture

```mermaid
graph TD
    User[User/Client]
    LB[Load Balancer]
    WebServer[Web Servers]
    Cache[Distributed Cache (Redis)]
    DB[(Database)]
    KGS[Key Generation Service]

    User -->|1. Shorten/Redirect| LB
    LB --> WebServer
    WebServer -->|2. Check Cache| Cache
    Cache -- Hit --> WebServer
    Cache -- Miss --> WebServer
    WebServer -->|3. Read/Write| DB
    WebServer -->|4. Get Key| KGS
    KGS -->|5. Assign Token| WebServer
```

### Components
1.  **Load Balancer**: Distributes incoming traffic across web servers.
2.  **Web Servers**: Stateless services that handle API requests.
3.  **Key Generation Service (KGS)**: Pre-generates unique keys to ensure no collisions and faster writes.
4.  **Database**: Stores the mapping of Short Key <-> Long URL.
5.  **Cache**: Stores frequently accessed URLs (e.g., top 20% of URLs generate 80% of traffic) to reduce DB load.

## 6. Detailed Design & Tradeoffs

### A. Key Generation Strategies

#### Option 1: Hash Function (MD5/SHA256)
*   **Method**: Hash the Long URL. Take the first 7 characters.
*   **Pros**: Simple.
*   **Cons**: Collisions possible. Different users shortening the same URL get the same key (might be okay, but prevents per-user analytics if not handled).

#### Option 2: Auto-Increment ID + Base62 Encoding
*   **Method**: Use DB auto-increment ID (1, 2, 3...) and convert to Base62 (`[a-z, A-Z, 0-9]`).
*   **Example**: ID `12345` -> Base62 `dnh`.
*   **Pros**: Guaranteed uniqueness. Short keys.
*   **Cons**: Vulnerable to enumeration (competitors can guess volume). Requires distributed ID generator (Snowflake) in distributed systems.

#### Option 3: Key Generation Service (KGS) - **Recommended**
*   **Method**: A standalone service generates random 6-7 character strings and stores them in a "Unused Key DB". Web servers fetch a batch of keys.
*   **Pros**: No collisions (checked at generation time). Fast (keys are ready). No coordination needed between web servers.
*   **Cons**: KGS is a single point of failure (needs replication).

### B. Redirect Status: 301 vs 302

*   **301 (Permanent Redirect)**: Browser caches the redirection.
    *   **Pro**: Reduces server load.
    *   **Con**: Cannot track analytics (click counts) effectively because requests don't hit the server after the first time.
*   **302 (Found / Temporary Redirect)**: Browser does not cache the redirect (or caches for a short time).
    *   **Pro**: Accurate analytics.
    *   **Con**: Higher server load.
    *   **Decision**: Use **302** if analytics are a business requirement. Use **301** for pure performance.
