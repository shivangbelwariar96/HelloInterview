# Table of Contents

- [gRPC vs REST Comparison](#grpc-vs-rest-comparison)
- [HTTP Streaming](#http-streaming)
- [System Design Cheatsheet](#system-design-cheatsheet)
- [Social Media Platform Design (Twitter/Facebook)](#social-media-platform-design-twitterfacebook)
- [NewsFeed System Design](#newsfeed-system-design)
- [Pastebin System Design](#pastebin-system-design)
- [Instagram System Design](#instagram-system-design)
- [Dropbox System Design](#dropbox-system-design)
- [YouTube/Netflix System Design](#youtubenetflix-system-design)
- [Yelp/Nearby Friends System Design](#yelpnearby-friends-system-design)
- [Rate Limiter System Design](#rate-limiter-system-design)
- [Log Collection and Analysis System Design](#log-collection-and-analysis-system-design)
- [Voting System Design](#voting-system-design)
- [Trending Topics System Design](#trending-topics-system-design)
- [Facebook Messenger System Design](#facebook-messenger-system-design)
- [Local Delivery (Gopuff) System Design](#local-delivery-gopuff-system-design)
- [Strava System Design](#strava-system-design)
- [Facebook Live Comments System Design](#facebook-live-comments-system-design)
- [Online Ticket Booking System Design](#online-ticket-booking-system-design)
- [E-Commerce Website (Amazon) System Design](#e-commerce-website-amazon-system-design)
- [Distributed Message Queue System Design](#distributed-message-queue-system-design)
- [Notification Service System Design](#notification-service-system-design)
- [Distributed Cache System Design](#distributed-cache-system-design)

https://github.com/shivangbelwariar96/HelloInterview/blob/main/Numbers-To-Remember.md


# gRPC vs REST Comparison

| Aspect | gRPC | REST |
|--------|------|------|
| **Protocol** | Uses HTTP/2, enabling multiplexed streams over a single TCP connection (multiple requests/responses simultaneously). This reduces latency. | Uses HTTP/1.1, where each request opens a new TCP connection or is pipelined; lacks multiplexing, resulting in higher overhead. |
| **Example** | A single gRPC connection can handle 100+ RPCs in parallel between microservices. | REST APIs often queue or open multiple connections under high load (e.g., RESTful weather API). |
| **Data Format** | Uses Protocol Buffers (protobuf) — compact, binary, schema-defined. Much faster serialization/deserialization. | Uses JSON or XML — human-readable but more verbose and slower to parse. |
| **Example** | `message User { string name = 1; int32 id = 2; }` in proto vs JSON: `{"name": "John", "id": 2}` — Protobuf is ~10x smaller and faster to transmit. | REST APIs like Twitter or GitHub serve JSON responses that are easily debugged or tested with tools like Postman or browser dev tools. |
| **Speed & Efficiency** | gRPC is significantly faster in both network latency and payload size. It's ideal for high-throughput internal communication. | REST is slower due to verbose payloads and lack of compression, but acceptable for most public-facing APIs. |
| **Benchmark Example** | In Netflix's case study, switching internal APIs from REST to gRPC improved payload efficiency by 60% and reduced response times by 30–50%. | A public REST API might deliver a 100KB JSON payload where gRPC would send a 10KB protobuf equivalent. |
| **Streaming** | gRPC supports client, server, and bidirectional streaming natively via HTTP/2 streams, making it ideal for chat, video, or data pipelines. | REST doesn't support native streaming. Workarounds like long polling, Server-Sent Events (SSE), or WebSockets are needed, each with limitations. |
| **Example** | Real-time fraud detection service streams transactions over gRPC for instant decisions. | REST API for stock prices may use polling or websockets for updates, which adds complexity. |
| **Contract** | Uses .proto files for strict typing and auto-generation of code. Ensures forward/backward compatibility. | REST typically lacks strict typing; OpenAPI/Swagger is used to document and sometimes validate endpoints, but not enforced across systems. |
| **Example** | protoc generates client/server code from .proto, guaranteeing consistency. | Swagger UI can document REST endpoints but does not enforce implementation correctness without external tooling. |
| **Tooling** | Protobuf compiler generates strongly typed client/server code in languages like Go, Java, Python, C++, etc. | REST client libraries like axios, fetch, requests are manually used to consume APIs. Tooling like Postman helps, but isn't tightly integrated. |
| **Example** | gRPC stub for UserService in Java or Python can be invoked directly as a method call, just like local objects. | REST: You construct URLs like GET /user/123, parse JSON responses, handle errors manually. |
| **Browser Support** | Browsers don't support gRPC natively due to HTTP/2 binary framing. Requires gRPC-Web or proxy layers like Envoy to convert HTTP/1 to HTTP/2. | Full native browser support — you can test REST APIs directly in browser tabs, use fetch() or tools like cURL without extra layers. |
| **Example** | A web dashboard that calls gRPC APIs must go through gRPC-Web + Envoy setup. | JavaScript `fetch('/api/user/123')` works out of the box. |
| **Interoperability** | Limited to systems that support gRPC and Protocol Buffers. Less friendly to 3rd-party devs without protobuf setup. | REST is universally supported — any device with HTTP stack can consume a REST API. |
| **Example** | IoT or legacy systems struggle with gRPC; REST can be consumed by Arduino or a cURL call on a router. | Stripe and Twilio provide REST APIs that any developer can use with zero setup. |
| **Use Cases** | Internal microservices, low-latency systems, streaming pipelines, mobile/backend comms. | Public APIs, third-party integrations, web services, or where simplicity and wide adoption matter. |
| **Example** | Google uses gRPC internally for most services; Kubernetes uses gRPC for kubelet API. | Facebook Graph API or OpenWeatherMap REST API used in frontend apps or integrations. |
| **Error Handling** | Standardized gRPC status codes like UNAVAILABLE, NOT_FOUND, INTERNAL. Automatically handled by gRPC clients. | REST uses HTTP status codes (e.g., 200, 404, 500), but payload structure is usually custom and inconsistent across APIs. |
| **Example** | A gRPC call might return Status.UNAVAILABLE and auto-retry via built-in client policy. | A REST call returning 500 might include a JSON error body you must parse manually: `{"error": "Database down"}`. |



# HTTP Streaming

An HTTP stream refers to a communication model where data is sent continuously over a single HTTP connection, rather than in discrete, complete responses. Instead of waiting for the entire response to be generated, the server pushes data incrementally to the client as it becomes available.

## Use Cases

HTTP streaming is useful when:
- You want to send real-time updates without re-establishing connections repeatedly.
- The response is large or long-running, and you want to start processing parts of it early.
- You're building systems like chat apps, video/audio streaming, server-sent events, or log tailing tools.

## How It Works
- A client makes an HTTP request.
- The server holds the connection open and keeps writing to the response body as data becomes available.
- The client reads chunks of the response without waiting for the entire message.

## Types of HTTP Streaming

| Type | Description |
|------|-------------|
| **Chunked Transfer Encoding (HTTP/1.1)** | Server sends response in chunks, without specifying total size upfront. |
| **Server-Sent Events (SSE)** | One-way server-to-client stream over HTTP. Ideal for notifications. |
| **HTTP/2 Streams** | Allows multiplexed, bi-directional streams on a single TCP connection. |
| **WebSockets (Not HTTP Streaming)** | Uses HTTP for handshake but switches to a full-duplex TCP stream. |


# System Design Cheatsheet



## 1. Bloom Filter  
**What it is:** A probabilistic data structure used to test whether an element is likely in a set.  
**What it does:** Answers "Is this element in the set?" with possible false positives but no false negatives.  
**How it works:** Uses a bit array of size \( m \) and \( k \) hash functions. To add an element, hash it with \( k \) functions, set corresponding bits to 1. To check membership, hash the element and verify if all bits are 1. False positives occur due to shared bits, but false negatives are impossible.  
**Pros:** Space-efficient, fast O(k) lookup and insertion, no false negatives.  
**Cons:** False positives possible (depends on \( m \), \( k \), and #elements), cannot remove elements (standard version), cannot retrieve elements.  
**Real-world examples:**  
- *Google Bigtable:* Reduces disk lookups for missing rows/columns.  
- *Apache Cassandra:* Checks SSTables for presence before disk access.  
- *CDNs (e.g., Akamai):* Caches URL checks to reduce redundant requests.  
- *Malware detection:* Google Safe Browsing uses it for URL blacklist checks.

---

## 2. Count-Min Sketch  
**What it is:** A probabilistic structure for estimating frequency of elements in a data stream.  
**What it does:** Tracks approximate counts with bounded error.  
**How it works:** Uses a 2D array \( d \times w \) and \( d \) hash functions. To increment count, hash with \( d \) functions, increment respective cells. To query, hash and take the min across \( d \) cells. Overestimates possible (due to collisions), underestimates not.  
**Pros:** Sublinear space, fast O(d) updates/queries, good for high-volume streams.  
**Cons:** Overestimation from collisions, error depends on \( w \), \( d \), cannot delete easily.  
**Real-world examples:**  
- *Network monitoring:* Cisco tracks flow sizes.  
- *Ad tech:* Google/Facebook estimate clicks/impressions.  
- *Streaming analytics:* Kafka/Spark for approximate frequency counts.  
- *DDoS detection:* Cloudflare tracks request frequency to detect anomalies.

---

## 3. HyperLogLog  
**What it is:** A probabilistic structure to estimate cardinality (distinct elements).  
**What it does:** Approximates number of unique items with high accuracy and low memory.  
**How it works:** Hash each item, count leading zeros in binary form, store max per register. Use harmonic mean of registers to estimate total cardinality.  
**Pros:** Extremely space-efficient (~12 KB for billions of items), constant memory, fast O(1) operations.  
**Cons:** Approximate (~2% error), no access to exact counts or elements, merging requires parameter alignment.  
**Real-world examples:**  
- *Redis:* Unique site visitor counts.  
- *Google Analytics:* Tracks unique users/events.  
- *Social media:* Twitter/Facebook count unique impressions.  
- *Databases:* PostgreSQL/BigQuery use it for `approx_distinct()` functionality.

---

## Summary Table

| Structure          | Purpose              | Mechanism                  | Pros                               | Cons                                | Real-world Uses                        |
|--------------------|----------------------|-----------------------------|------------------------------------|-------------------------------------|----------------------------------------|
| **Bloom Filter**   | Membership testing   | Bit array, multiple hashes  | Space-efficient, fast, no false negatives | False positives, no deletions        | Bigtable, Cassandra, CDNs, Safe Browsing |
| **Count-Min Sketch** | Frequency estimation | 2D array, min of hashed cells | Sublinear space, fast, stream-friendly | Overestimation, approximate           | Network monitoring, ad tech, DDoS detection |
| **HyperLogLog**    | Cardinality estimation | Hash-based leading zeros   | Minimal memory, fast, scalable      | Approximate, no exact counts         | Redis, Google Analytics, social media   |

These structures trade accuracy for efficiency, ideal for large-scale, resource-constrained systems where performance and scalability outweigh exactness.









## Read-Heavy Systems
These prioritize low-latency data retrieval, efficient caching, and scalability.

---
---

### 1. URL Shortener (Bitly)

**Key Generation:**  
Use **Base62** (a–z, A–Z, 0–9) to encode a global counter (e.g., counter `125` → Base62 = `cb`) for **uniqueness**, **ordering**, and **collision-free** generation.  
Alternative: truncated **MD5/SHA-256** of long URL (e.g., `hash("https://xyz.com") → abX1c9Z`) with **retry**, added **entropy** (timestamp, salt) on collision.  
_Custom aliases_ stored in a **separate namespace** with strict **uniqueness enforcement**.

**Collision Handling:**  
Use a **Bloom filter** to pre-check for collisions. On conflict: retry with new hash, add entropy, or fallback to counter.  
Custom aliases checked for uniqueness in alias index.

**DB Storage (Durable):**  
Use **DynamoDB**:  
`{ short_url (PK), long_url, created_at, expires_at, user_id, is_custom, metadata }`  
- TTL on `expires_at`  
- **GSI-1**: `user_id → short_url`  
- **GSI-2**: `is_custom → short_url`  
- Optional **LSI** on `created_at`

**Caching (In-Memory):**  
Use **Redis** for hot lookups.  
Cache schema:  
- `short:<key>` → long_url  
- `long:<hash(long_url)>` → short_key  
Use **TTL** or **LRU** eviction. Cache is **eventually consistent**, write-through on cache miss.

**Redirect Strategy:**  
Use **302 Temporary Redirect** (default) to allow updates.  
Use **301 Permanent Redirect** only for immutable, custom, or paid URLs.

**Sharding:**  
Shard by **short_url prefix** (first 1–2 Base62 chars).  
Avoid `user_id` sharding → leads to **hotspots**.  
In DynamoDB:  
- **PartitionKey** = prefix  
- **SortKey** = short_url

**Edge Cases:**  
- **Expired URLs**: TTL-based deletion  
- **Malicious URLs**: blocklist + async malware scanner (e.g., Safe Browsing)  
- **Custom Aliases**: separate storage/index  
- **Rate-Limiting**: token bucket per IP/user (in Redis)  
- **Validation**: URI parse + regex validation pre-insert

**Monitoring / Observability:**  
Track:  
- **QPS**, **latency**, **cache hit/miss**, **redirect errors**, **collision rate**  
Use dashboards, structured logs, metrics.  
Alert on spikes in latency, redirect failures, cache miss anomalies.

**Trade-offs:**  
- **Hashing**: *Stateless*, but collision-prone  
- **Counter**: *Deterministic*, globally unique, but requires distributed state  
- **SQL**: *Strong consistency*, limited scale  
- **NoSQL (DynamoDB)**: *Horizontal scale*, fast, TTL built-in  
- **Redis**: *Fast, volatile*, eventual consistency  
- **DynamoDB**: *Durable*, supports strong consistency  
- **Consistency vs Availability**:  
  Prioritize **availability** (fast redirects) →  
  - Redis: *eventually consistent*  
  - DynamoDB: *strong consistency* on writes

**Tables & Schemas:**  
- **DynamoDB Table**:  
  `short_url (PK), long_url, created_at, expires_at, user_id, is_custom, metadata`  
- **GSI-1**: `user_id → short_url`  
- **GSI-2**: `is_custom → short_url`  
- **Redis Keys**:  
  - `short:<key>` → long_url  
  - `long:<hash>` → short_key

**Functional Requirements:**  
- Shorten URL (auto + custom)  
- Redirect to original  
- TTL & expiry support  
- Alias support  
- Analytics/logs  
- Malicious URL protection

**Non-Functional Requirements:**  
- **Latency**: <50ms redirect, <100ms creation  
- **QPS**: ~10K  
- **Availability**: 99.99%  
- **Durability**: No loss  
- **Scalability**: Horizontal  
- **Abuse Protection**: rate limits, validation  
- **Cache Hit Ratio**: >90%

**Additional Optimizations:**  
- Use **CDN** for popular URLs  
- **Pre-warm Redis** on creation  
- Stream analytics to Kafka → ClickHouse  
- On cache miss: fetch from DB + refresh cache  
- Async analytics writes  
- Batch writes for efficiency
---
---



### 2. Image Hosting Service
**Improvements:**
- **Object Storage**: Use S3/GCS for durable, scalable storage. Store metadata (e.g., image_id, user_id, upload_time) in a relational DB (e.g., PostgreSQL) for querying.
- **CDN**: Use Cloudflare/Akamai to cache images globally, reducing latency and origin server load.
- **Deduplication**: Compute image hashes (e.g., perceptual hash) to identify duplicates. Store a single copy and reference it via metadata.
- **Resizing**: Pre-generate common sizes (thumbnail, medium, large) using a background job (Lambda + SQS) to reduce on-the-fly processing. Store variants in S3.
- **Edge Cases**: Handle large uploads (use multipart uploads), corrupted images (validate on upload), CDN cache invalidation (use versioning), bandwidth abuse (rate-limit downloads), and metadata inconsistencies (reconcile with S3).
- **Security**: Require signed URLs for private images and rate-limit uploads to prevent abuse.
- **Monitoring**: Track upload success rate, CDN hit ratio, storage costs, and image retrieval latency.
- **Trade-offs:**
  - **Pre-resize vs. On-the-fly**: Pre-resizing saves compute but increases storage; on-the-fly resizing is flexible but CPU-intensive. Choose pre-resizing for predictable latency and cost control.
  - **Perceptual vs. Exact Hash**: Perceptual hash detects near-duplicates (e.g., slightly edited images) but is slower; exact hash is faster but misses near-duplicates. Choose perceptual hash for better deduplication.
  - **S3 vs. Custom Storage**: S3 is managed but costly; custom storage is cheaper but complex. Use S3 for reliability and scalability.
- **Why Pre-resize + S3/CDN?**: Pre-resizing ensures fast delivery, S3 provides durability, and CDN minimizes latency for global users.
- **Scalability Estimates**: Handles ~5K image uploads/sec, ~10M daily active users, and ~1PB storage with S3 scaling and CDN caching.
- **Latency/Throughput Targets**: Target <100ms for image retrieval, <500ms for uploads, and ~5K requests/sec for downloads.

---
---

# Social Media Platform Design (Twitter/Facebook)

## Functional Requirements
- **Posts**: Support text, images, videos, and links.
- **Timelines**: Personalized feeds from followed users.
- **Relationships**: Directional follow and unfollow actions.
- **Interactions**: Like, comment, share, and react to posts.
- **Search**: Search by keywords, hashtags, and mentions.
- **Notifications**: Real-time alerts for interactions and mentions.

## Non-Functional Requirements
- **Scalability**: Handle 100M daily active users (DAU), 1B posts/day, 50K QPS for timeline retrieval.
- **Latency**: 
  - Timeline generation: <200ms
  - Post retrieval: <100ms
  - Relationship queries: <50ms
- **Availability**: 99.99% uptime with no single points of failure.
- **Consistency**:
  - Eventual consistency for posts and timelines.
  - Strong consistency for relationships and notifications.
## Improvements
- **Posts Storage**:
  - Use **Cassandra** with schema: `{post_id, user_id, content, timestamp, media_urls}`.
  - Time-based partitioning (e.g., by day/week) for efficient range queries.
  - Handles high write throughput (~12K posts/sec).

- **Timelines**:
  - **Hybrid Push-Pull Model**:
    - **Push**: Store posts in **Redis** lists (`timeline:user_id`) for active users.
    - **Pull**: Fetch on-demand for inactive users to reduce write load.
  - Precompute posts from influencers to mitigate fanout issues.
- **Relationships**:
  - Store in **Redis** as adjacency lists (`followers:user_id`, `following:user_id`) for fast access (<50ms).
  - Optionally use **Neo4j** for complex relationship queries (e.g., mutual follows).
- **Denormalization**:
  - Precompute timelines in Redis to avoid joins between user and post tables.
  - Update timelines incrementally when new posts are created.
- **Sharding**:
  - Shard posts by `post_id` using consistent hashing in Cassandra.
  - Shard timelines by `user_id` in Redis Cluster.
- **Caching**:
  - Cache hot posts and user profiles in Redis.
  - Use CDN for media content delivery.

- **Edge Cases**:
  - **Deleted Posts**: Use tombstones in Cassandra.
  - **Privacy Settings**: Filter content on read.
  - **Viral Posts**: Implement multi-level caching (Redis + Memcached).
  - **User Bans**: Maintain a banned list in Redis and check on read.
  - **Timeline Staleness**: Periodically refresh or pull for inactive users.

- **Monitoring**:
  - Track timeline latency, fanout throughput, graph query performance, post retrieval success rate.
  - Monitor Redis memory usage, Cassandra write throughput, and CDN hit rates.

## Trade-offs
- **Push vs. Pull for Timelines**:
  - **Push**: Low read latency but high write cost for users with many followers (influencers).
  - **Pull**: Write-efficient but slower for reads.
  - **Hybrid Model**: Balances both by pushing to active users and pulling for inactive ones.
- **Graph DB vs. Adjacency Lists**:
  - **Neo4j**: Flexible for complex queries but slower.
  - **Redis Adjacency Lists**: Faster for simple follow lookups.
  - **Choice**: Use Redis for low-latency timeline generation; Neo4j for advanced queries if needed.
- **Cassandra vs. DynamoDB**:
  - **Cassandra**: Better for high write throughput and open-source.
  - **DynamoDB**: Simpler to manage but more expensive.
  - **Choice**: Cassandra for write-heavy post storage.

## Why Hybrid + Cassandra?
- **Hybrid Model**: Optimizes for both influencers (push for low latency) and inactive users (pull to save writes).
- **Cassandra**: Scales horizontally for high write throughput, essential for handling 1B posts/day.
## Scalability Estimates
- Supports 100M DAU, 1B posts/day (~12K posts/sec), and 50K QPS for timeline retrieval.
- Achieved through Cassandra clusters for posts and Redis sharding for timelines.
## Latency/Throughput Targets
- **Timeline Generation**: <200ms (leveraging Redis precomputing).
- **Post Retrieval**: <100ms (via Cassandra indexing).
- **Relationship Queries**: <50ms (using Redis).
- **Post Ingestion**: ~10K posts/sec.

## Database Schemas
- **Cassandra Posts Table**:
  ```sql
  CREATE TABLE posts (
    post_id UUID,
    user_id UUID,
    content TEXT,
    timestamp TIMESTAMP,
    media_urls LIST<TEXT>,
    PRIMARY KEY (user_id, timestamp, post_id)
  ) WITH CLUSTERING ORDER BY (timestamp DESC);
  ```
  - Partitioned by `user_id`, clustered by `timestamp` for time-based queries.
  - **Consistency**: Tunable (e.g., quorum reads/writes) for high availability.
- **Redis Timelines**:
  - Stored as sorted sets: `timeline:user_id` with post_ids ordered by timestamp.
  - **Consistency**: Eventual.
- **Redis Relationships**:
  - Sets: `followers:user_id` and `following:user_id`.
  - **Consistency**: Strong via Redis transactions or Lua scripts.

## Cache Schemas
- **Post Cache**: `post:post_id` – serialized post object (eventual consistency).
- **User Cache**: `user:user_id` – serialized user profile (eventual consistency).
- **Hot Posts**: `hot_posts` – sorted set by engagement scores (eventual consistency).

## Consistency vs. Availability
- **Database (Cassandra)**:
  - Eventual consistency for posts and timelines (prioritizes availability).
  - Uses quorum reads/writes for balance.
- **Cache (Redis)**:
  - Eventual consistency for timelines and caches.
  - Strong consistency for relationships via atomic operations.
- **Tricks**:
  - Use Redis pipelining for batch writes.
  - Leverage Cassandra lightweight transactions for critical updates.

## Additional Considerations
- **Search**: Use **Elasticsearch** for full-text search (keywords, hashtags, mentions).
- **Notifications**: 
  - Queue with **Kafka**.
  - Deliver via WebSocket or APNs for real-time updates.
- **Media Storage**: Store in **S3**, serve via CDN.
- **Rate Limiting**: Implement caps on API requests and post creations.
- **Data Archiving**: Move old posts to **Glacier** for cost-effective storage.

---
---

# NewsFeed System Design

## Overview
This design outlines a scalable NewsFeed system capable of handling 50M daily active users (DAU), 500M posts per day, and 20K queries per second (QPS) for feed generation. It incorporates a hybrid push/pull model, efficient caching, and optimized ranking to meet latency and throughput targets.

## Functional Requirements
- **Feed Generation**: Generate personalized feeds based on posts from followed accounts.
- **Post Interactions**: Enable users to like, comment, and share posts.
- **Ranking**: Order posts by relevance using likes, recency, and user affinity.
- **Pagination**: Provide efficient scrolling with cursor-based pagination.
- **High-Follower Accounts**: Manage users with many followers (e.g., celebrities) without system overload.

## Non-Functional Requirements
- **Scalability**: Support 50M DAU, 500M posts/day, and 20K QPS.
- **Latency**:
  - Feed retrieval: <150ms
  - Ranking updates: <500ms
  - Feed generation: ~5K feeds/sec
- **Availability**: Ensure high availability with minimal downtime.
- **Consistency**: Use eventual consistency for feed content, addressing staleness.

## Design Improvements

### Push vs. Pull (Fanout-on-Write vs. Fanout-on-Read)
- **Fanout-on-Write**: For active users, push posts to Redis timelines immediately for fast retrieval.
- **Fanout-on-Read**: For inactive users, fetch posts on demand to reduce write load.
- **Hybrid Approach**: Combines both to balance latency and resource usage.
- **High-Follower Accounts**: 
  - Add a `precompute_flag` in the Follow table to skip precomputing feeds for accounts like Justin Bieber (90M+ followers).
  - In the async worker queue, ignore precompute requests for flagged accounts.
  - On read, merge partially precomputed feeds with recent posts from non-precomputed accounts.

### Caching
- **Redis**: Cache recent posts for quick access (e.g., `recent_posts:user_id` as a list).
- **Memcached**: Store full timelines to reduce database load (e.g., `timeline:user_id` as serialized data).
- **Multi-Level Caching**: Use both to handle hot content efficiently.

### Pagination
- Implement cursor-based pagination using a combination of `timestamp` and `post_id` for seamless scrolling.

### Ranking
- Use a weighted scoring algorithm based on:
  - Likes
  - Recency
  - User affinity
- Compute scores in a background job using Spark, then store in Redis (e.g., `ranked_feed:user_id` as a sorted set).

### Handling Edge Cases
- **Feed Staleness**: Periodically refresh feeds for inactive users.
- **Deleted Posts**: Remove from cache and exclude during feed generation.
- **Skewed User Activity**: Rate-limit influencers to prevent overload.
- **Privacy Filters**: Apply filters at read time based on user settings.
- **Cache Inconsistencies**: Revalidate cache with the database periodically.

### Monitoring
- Track key metrics:
  - Feed refresh latency
  - Ranking accuracy
  - Cache eviction rate
  - Feed generation success rate

## Trade-offs

### Fanout-on-Write vs. Fanout-on-Read
- **Fanout-on-Write**: 
  - Pros: Faster feed retrieval for active users.
  - Cons: Expensive for users with many followers.
- **Fanout-on-Read**: 
  - Pros: Write-efficient.
  - Cons: Slower reads.
- **Chosen Approach**: Hybrid model to optimize latency and cost.

### Real-time vs. Batch Ranking
- **Real-time Ranking**: 
  - Pros: Always fresh.
  - Cons: Computationally expensive.
- **Batch Ranking**: 
  - Pros: Efficient and scalable.
  - Cons: May be stale.
- **Chosen Approach**: Batch ranking with periodic updates for scalability.

### Redis vs. Memcached
- **Redis**: 
  - Pros: Supports complex data structures (e.g., sorted sets).
  - Cons: Heavier footprint.
- **Memcached**: 
  - Pros: Lightweight and simple.
  - Cons: Limited functionality.
- **Chosen Approach**: Use both for complementary caching needs.

## Why Hybrid + Redis?
- **Hybrid Model**: Minimizes write amplification for inactive users while ensuring low-latency reads for active users.
- **Redis**: Provides fast access to precomputed timelines and recent posts, critical for meeting the <150ms feed retrieval target.

## Scalability Estimates
- **Capacity**: Handles 50M DAU, 500M posts/day, and 20K QPS.
- **How**: Achieved through Redis and Memcached scaling, plus database partitioning.

## Latency and Throughput Targets
- **Feed Retrieval**: <150ms
- **Ranking Updates**: <500ms
- **Feed Generation**: ~5K feeds/sec

## Database Schemas
- **Posts Table** (e.g., Cassandra):
  - Schema: `{post_id, user_id, content, timestamp, media_urls}`
  - Partitioned by `user_id`, clustered by `timestamp DESC`.
- **Follow Table**:
  - Schema: `{follower_id, followee_id, precompute_flag}`
  - `precompute_flag` skips feed precomputation for high-follower accounts.
- **Feed Table** (precomputed feeds):
  - Schema: `{user_id, post_id, score, timestamp}`

## Cache Schemas
- **Redis**:
  - `recent_posts:user_id`: List of recent post IDs.
  - `ranked_feed:user_id`: Sorted set of post IDs by score.
- **Memcached**:
  - `timeline:user_id`: Serialized list of post IDs.

## Additional Considerations
- **Background Jobs**: Use Spark for batch ranking computation.
- **Async Worker Queue**: Efficiently manage feed precomputing, skipping high-follower accounts.
- **Rate Limiting**: Cap activity from high-traffic users to prevent abuse.

## Conclusion
This NewsFeed system uses a hybrid fanout model with Redis and Memcached caching to balance performance and scalability. It efficiently handles high-follower accounts by merging posts on read, computes rankings in batch for efficiency, and addresses edge cases with targeted strategies, meeting all specified targets.

---
---

# Pastebin System Design

## Functional Requirements
- **Paste Creation**: Users create text pastes (text, code, markdown) with optional expiration.
- **Paste Sharing**: Generate short URLs for sharing public/private pastes.
- **Paste Access**: Retrieve pastes via unique URLs; support private access with authentication.
- **Expiration**: Auto-delete pastes after a set time (e.g., 1 day, 1 week).
- **Syntax Highlighting**: Display code with language-specific formatting.

## Non-Functional Requirements
- **Scalability**: Handle ~1K paste creations/sec, ~10M daily active users, ~100K read QPS.
- **Latency**: <50ms for paste retrieval, <200ms for creation, ~2K pastes/sec ingestion.
- **Availability**: 99.99% uptime, no single points of failure.
- **Consistency**: Eventual for paste content, strong for metadata (e.g., paste_id, expiration).
- **Security**: Prevent malicious content, secure private pastes.

## Improvements
- **Text Storage**: Store metadata in **DynamoDB** {`paste_id`, `content`, `creation_time`, `expiration_time`, `user_id`, `is_private`}. Use **S3** for large pastes (>1MB) with S3 object key linked in DynamoDB.
- **Key Generation**: Generate short **base62** IDs (6-8 chars, ~62^6 URLs) via **ZooKeeper** distributed counter for uniqueness. Alternative: hash content with collision checks.
- **Caching**: Cache hot pastes in **Redis** (LRU eviction) and serve via **Cloudflare CDN** for global access.
- **Expiration**: Set **TTL** in Redis/DynamoDB for auto-deletion. Use S3 lifecycle policies for large pastes.
- **Access Control**: Public pastes accessible via URL; private pastes use **signed URLs** with short-lived tokens.
- **Edge Cases**: 
  - **Duplicate Pastes**: Hash-based deduplication using content SHA-256.
  - **Malicious Content**: Apply content filters (e.g., regex for XSS, malware signatures).
  - **High Read QPS**: CDN handles ~100K QPS; Redis for low-latency hot reads.
  - **Large Pastes**: Multipart uploads to S3 for reliability.
  - **Expired Paste Revival**: Archive to **Glacier** with user-triggered restore option.
- **Monitoring**: Track paste creation rate, cache hit ratio, expiration cleanup success, retrieval latency, CDN hit rate, DynamoDB read/write capacity.

## Trade-offs
- **Counter vs. Hashing**: 
  - **Counter**: Unique IDs, coordination overhead. 
  - **Hashing**: No coordination, collision risk. 
  - *Choice*: **ZooKeeper counter** for guaranteed uniqueness.
- **NoSQL vs. S3**: 
  - **DynamoDB**: Fast metadata access, costly for blobs. 
  - **S3**: Scalable for large data, slower metadata. 
  - *Choice*: DynamoDB for metadata, S3 for large content.
- **Redis vs. CDN**: 
  - **Redis**: Fast for hot data, memory-bound. 
  - **CDN**: Global scaling, static content only. 
  - *Choice*: Both for complementary performance.

## Why Counter + DynamoDB/S3?
- **Counter**: Ensures unique, short IDs without collision risks.
- **DynamoDB**: Low-latency metadata access for paste lookups.
- **S3**: Cost-effective, scalable storage for large pastes.

## Scalability Estimates
- **Creations**: ~1K pastes/sec via DynamoDB partitioning, ZooKeeper scaling.
- **Users**: ~10M DAU with Redis/CDN caching.
- **Reads**: ~100K QPS via CDN, Redis for hot pastes.

## Latency/Throughput Targets
- **Retrieval**: <50ms (Redis/CDN).
- **Creation**: <200ms (DynamoDB/ZooKeeper).
- **Ingestion**: ~2K pastes/sec.

## Database Schemas
- **DynamoDB Pastes Table**:
  ```sql
  Table: pastes
  Partition Key: paste_id (String)
  Attributes:
    content (String, small pastes only)
    s3_key (String, for large pastes)
    creation_time (Number, epoch ms)
    expiration_time (Number, epoch ms)
    user_id (String, optional)
    is_private (Boolean)
  ```
  - **Consistency**: Strong for metadata (paste_id, expiration_time) via consistent reads; eventual for content.
  - **Availability**: High with DynamoDB multi-AZ replication.

- **S3 Large Pastes**:
  - Bucket: `pastebin-content`, key: `paste_id/timestamp`.
  - Lifecycle policy: Delete after `expiration_time`.

## Cache Schemas
- **Redis**:
  - `paste:paste_id`: Serialized paste object (content, metadata).
  - **TTL**: Matches `expiration_time`.
  - **Consistency**: Eventual, sync with DynamoDB on miss.
- **CDN (Cloudflare)**:
  - Cache public pastes by URL (`/paste_id`).
  - **Consistency**: Eventual, purge on update/delete.
- **Tricks**: Redis pipelining for batch reads, CDN edge caching for static content.

## Consistency vs. Availability
- **DynamoDB**:
  - **Strong Consistency**: For paste_id, expiration_time to ensure accurate access/expiration.
  - **High Availability**: Multi-AZ, tunable consistency (quorum for writes).
- **Redis**:
  - **Eventual Consistency**: For cached pastes, refreshed on miss.
  - **High Availability**: Redis Cluster with replicas.
- **CDN**:
  - **Eventual Consistency**: Cache invalidation on paste updates/deletion.
  - **High Availability**: Global edge nodes.
- **Tricks**: Use DynamoDB conditional writes for metadata updates, Redis Lua scripts for atomic cache updates.

## Additional Considerations
- **Syntax Highlighting**: Use **Prism.js** or server-side rendering for code formatting.
- **Rate Limiting**: Cap paste creation/reads per user to prevent abuse.
- **Analytics**: Track paste views with **Kafka** and store in **Redshift** for insights.
- **Security**: Encrypt private pastes in S3 with KMS, validate input to block XSS/SQL injection.
- **Backup**: Periodic DynamoDB backups to S3, Glacier for long-term storage.

## Correctness Check
- **Original Design**: All components (DynamoDB, S3, ZooKeeper, Redis, CDN) align with scalability/latency goals. No incorrect details found.
- **Missing Details**: Added functional/non-functional requirements, detailed schemas, consistency/availability, and analytics/security for completeness.

## Summary
Pastebin uses **ZooKeeper** for unique base62 IDs, **DynamoDB** for metadata, **S3** for large pastes, **Redis/CDN** for caching, and **TTL** for expiration. It handles 10M DAU, 1K creations/sec, 100K read QPS with <50ms retrieval, <200ms creation, and robust edge case handling (duplicates, malicious content, large pastes). Security, analytics, and syntax highlighting ensure a FAANG-ready design.


---
---


# Instagram System Design

## Functional Requirements
- **Media Posting**: Upload images/videos with captions, tags, and locations.
- **Feed Generation**: Display chronological feeds of posts from followed users.
- **Follow System**: Enable follow/unfollow for directional relationships.
- **Interactions**: Support likes, comments, and shares.
- **Explore/Search**: Discover trending content, search by hashtags/users.
- **Stories**: Share ephemeral content (24-hour lifespan).

## Non-Functional Requirements
- **Scalability**: Handle ~100M daily active users (DAU), ~1B posts/day, ~50K QPS for feed retrieval.
- **Latency**: <200ms for feed generation, <100ms for media retrieval, ~10K posts/sec ingestion.
- **Availability**: 99.99% uptime, no single points of failure.
- **Consistency**: Eventual for feeds/posts, strong for relationships/interactions.
- **Security**: Secure private posts, prevent malicious uploads.

## Improvements
- **Media Storage**: Store images/videos in **S3** with metadata {`post_id`, `user_id`, `timestamp`, `caption`, `tags`, `location`} in **Cassandra** for high write throughput (~12K posts/sec). Use S3 lifecycle policies for archiving.
- **Feed Generation**: 
  - **Hybrid Push-Pull Model**: 
    - **Push**: Fanout posts to **Redis** timelines (`timeline:user_id`) for active users (<100K followers).
    - **Pull**: Fetch on-demand for inactive users or celebrities (>100K followers) to reduce write load.
  - **Celebrity Optimization**: Store celebrity posts in **Cassandra Posts DB** without fanout. Merge with precomputed feeds on read.
  - **Chronological Sorting**: Query **DynamoDB Follow Table** (partition: `followerId`, sort: `followedId`) for followed users, then **Cassandra Post Table** (partition: `userId`, sort: `createdAt+postId`) for recent posts. Combine, sort by timestamp, return with cursor/limit.
- **Caching**: 
  - **Redis**: Cache feeds (`timeline:user_id`, sorted set) and hot posts (`post:post_id`).
  - **Cloudflare CDN**: Cache media for global delivery.
  - **Smart Caching**: Aggressive caching for popular media, shorter TTLs for less-accessed content, pre-warm caches for trending items.
- **Sharding**: Shard posts by `post_id` (Cassandra) and timelines by `user_id` (Redis Cluster) using **consistent hashing**. Redis Cluster ensures durability across nodes.
- **Edge Cases**: 
  - **Deleted Posts**: Tombstone records in Cassandra.
  - **Privacy Settings**: Filter on read based on `is_private` flag.
  - **Viral Content**: Cache hot posts in Redis/Memcached.
  - **Media Corruption**: Validate uploads (format, size, checksum).
  - **Feed Inconsistencies**: Recompute timelines periodically or on cache miss.
- **Monitoring**: Track feed latency, media load time, cache hit ratio, post retrieval success rate, Redis memory usage, Cassandra write throughput, CDN hit rates.

## Trade-offs
- **Push vs. Pull**: 
  - **Push**: Fast feeds, high write cost for influencers. 
  - **Pull**: Write-efficient, slower reads. 
  - *Choice*: **Hybrid** for active users/celebrities.
- **Cassandra vs. DynamoDB**: 
  - **Cassandra**: High write throughput, complex setup. 
  - **DynamoDB**: Simpler, costlier. 
  - *Choice*: **Cassandra** for write-heavy posts.
- **S3 vs. Custom Storage**: 
  - **S3**: Managed, reliable, costly. 
  - **Custom**: Cheaper, complex. 
  - *Choice*: **S3** for reliability.

## Why Hybrid + Cassandra?
- **Hybrid**: Balances write amplification (celebrities) and read latency (active users).
- **Cassandra**: Scales horizontally for ~1B posts/day with low-latency writes.

## Scalability Estimates
- **Users**: ~100M DAU via Cassandra clusters, Redis sharding.
- **Posts**: ~1B/day (~12K posts/sec).
- **Reads**: ~50K QPS with CDN/Redis.

## Latency/Throughput Targets
- **Feed Generation**: <200ms (Redis precompute, Cassandra indexing).
- **Media Retrieval**: <100ms (CDN/S3).
- **Post Ingestion**: ~10K posts/sec.

## Database Schemas
- **Cassandra Posts Table**:
  ```sql
  CREATE TABLE posts (
    user_id UUID,
    post_id UUID,
    created_at TIMESTAMP,
    caption TEXT,
    tags LIST<TEXT>,
    location TEXT,
    s3_key TEXT,
    is_private BOOLEAN,
    PRIMARY KEY (user_id, created_at, post_id)
  ) WITH CLUSTERING ORDER BY (created_at DESC);
  ```
  - Partition: `user_id`, cluster: `created_at+post_id`.
  - **Consistency**: Eventual (quorum writes), high availability.
- **DynamoDB Follow Table**:
  ```sql
  Table: follows
  Partition Key: followerId (String)
  Sort Key: followedId (String)
  Attributes: created_at (Number)
  ```
  - **Consistency**: Strong for relationship updates, high availability.
- **Redis Timelines**:
  - `timeline:user_id`: Sorted set of `post_id` by timestamp.
  - **Consistency**: Eventual, refreshed on miss.

## Cache Schemas
- **Redis**:
  - `post:post_id`: Serialized post (metadata, s3_key).
  - `timeline:user_id`: Sorted set (post_ids, timestamps).
  - **Consistency**: Eventual, sync with Cassandra.
  - **TTL**: Short for cold content, long for hot.
- **CDN (Cloudflare)**:
  - Cache media by `s3_key`.
  - **Consistency**: Eventual, purge on update/delete.
- **Tricks**: Redis pipelining for batch writes, CDN pre-warming for trending media.

## Consistency vs. Availability
- **Cassandra**: 
  - **Eventual Consistency**: For posts (quorum reads/writes).
  - **High Availability**: Multi-region replication.
- **DynamoDB**: 
  - **Strong Consistency**: For follow relationships (conditional writes).
  - **High Availability**: Global tables.
- **Redis**: 
  - **Eventual Consistency**: For timelines, sync on miss.
  - **High Availability**: Redis Cluster with replicas.
- **CDN**: 
  - **Eventual Consistency**: Cache invalidation on updates.
  - **High Availability**: Global edge nodes.
- **Tricks**: Cassandra lightweight transactions for critical updates, Redis Lua scripts for atomic operations.

## Additional Considerations
- **Stories**: Store in **Redis** with 24-hour TTL, fanout to followers.
- **Search/Explore**: Use **Elasticsearch** for hashtags, users, locations.
- **Notifications**: **Kafka** queue, WebSocket/APNs for real-time delivery.
- **Rate Limiting**: Cap post uploads/interactions per user.
- **Security**: Encrypt private media in S3 with KMS, validate uploads for malware.
- **Analytics**: Log interactions in **Kafka**, aggregate in **Redshift**.

## Correctness Check
- **Original Design**: All components (S3, Cassandra, Redis, CDN, DynamoDB) align with scalability/latency goals. No incorrect details.
- **Missing Details**: Added stories, search, notifications, security, analytics, and detailed schemas for completeness.

## Summary
Instagram’s design uses a **hybrid push-pull model** with **Cassandra** for posts, **S3** for media, **Redis** for timelines, and **CDN** for delivery. It optimizes celebrities with pull-based feeds, ensures chronological sorting, and handles edge cases (privacy, viral content). Scales to 100M DAU, 1B posts/day, 50K QPS with <200ms feed latency, 99.99% uptime, and robust security/analytics.

---
---


# Dropbox System Design

## Functional Requirements
- **File Upload/Download**: Upload/download files of any type/size with progress tracking.
- **File Sharing**: Share files with specific users via secure links.
- **Versioning**: Maintain file version history with conflict resolution.
- **Syncing**: Real-time sync across devices.
- **Offline Access**: Cache files locally for offline use.
- **Deduplication**: Store identical files once to save space.

## Non-Functional Requirements
- **Scalability**: Handle ~10M daily active users (DAU), ~1PB storage, ~1K file uploads/sec.
- **Latency**: <500ms for file sync, <1s for uploads, ~1K files/sec processing.
- **Availability**: 99.99% uptime, no single points of failure.
- **Consistency**: Eventual for file content, strong for metadata (e.g., file_id, version).
- **Security**: Encrypt files in transit/rest, secure sharing with access control.

## Improvements
- **File Storage**: 
  - Store files in **S3** with metadata {`file_id`, `user_id`, `version`, `path`, `hash`, `status`} in **PostgreSQL** for relational queries.
  - Use **presigned URLs** for direct S3 uploads, bypassing backend. Client requests URL with metadata (`status: uploading`), uploads via PUT, backend updates to `uploaded` on S3 notification. URLs expire (e.g., 5min) for security.
- **Downloading**: 
  - Use **CloudFront CDN** to cache files globally, serving from nearest edge for low latency.
  - Generate **signed URLs** (5min expiry) for secure downloads, compatible with CDN.
- **Sharing Files**: 
  - **SharedFiles Table** in PostgreSQL links `user_id` to `file_id`. Query by `user_id` to list shared files.
  - Downside: Table scans slower than direct lookup but simpler than separate sharelist.
- **Versioning**: Store deltas in S3 for space efficiency; track versions in PostgreSQL (`version_history` table).
- **Syncing**: 
  - **WebSockets** for real-time sync notifications.
  - **Kafka** for file change events, fallback to polling for non-real-time clients.
- **Performance Optimization**: 
  - **CDN**: Low-latency downloads.
  - **Chunking**: Split files into 5-10MB chunks for parallel uploads, selective syncing.
  - **Compression**: Use **Gzip/Brotli** for text, skip for media (low compression gains). Compress before encryption.
- **Large Files**: 
  - Chunk into 5-10MB pieces, upload to S3 via **Multipart Upload API**.
  - Client-side **SHA-256** fingerprinting for file/chunk IDs.
  - Track progress in **FileMetadata Table** (`chunks` field), updated via S3 events.
  - Support resumable uploads by checking uploaded chunks.
- **Deduplication**: Compute **SHA-256** hashes, store identical files once, reference via metadata.
- **File Security**: 
  - **HTTPS** for transit encryption.
  - **S3 encryption** at rest with unique KMS keys.
  - Access control via **SharedFiles Table**, signed URLs for downloads.
- **Edge Cases**: 
  - **Concurrent Edits**: Resolve conflicts via versioning (e.g., create `file_v2`).
  - **Large Files**: Chunked uploads for reliability.
  - **Offline Access**: Local caching with sync on reconnect.
  - **File Corruption**: Validate uploads with checksums.
  - **Storage Quotas**: Enforce limits in PostgreSQL, reject uploads if exceeded.
- **Monitoring**: Track sync latency, storage usage, deduplication rate, upload success rate, CDN hit ratio, PostgreSQL query performance.

## Trade-offs
- **S3 vs. Custom Storage**: 
  - **S3**: Managed, reliable, costly. 
  - **Custom**: Cheaper, complex. 
  - *Choice*: **S3** for scalability.
- **PostgreSQL vs. NoSQL**: 
  - **PostgreSQL**: Strong for relational metadata, complex joins. 
  - **NoSQL**: Flexible, weak for joins. 
  - *Choice*: **PostgreSQL** for file relationships.
- **WebSockets vs. Polling**: 
  - **WebSockets**: Real-time, complex. 
  - **Polling**: Simpler, delayed. 
  - *Choice*: **WebSockets** for better UX.

## Why S3 + PostgreSQL?
- **S3**: Durable, scalable storage for files.
- **PostgreSQL**: Efficient relational queries for metadata, versioning, sharing.

## Scalability Estimates
- **Users**: ~10M DAU via S3 partitioning, PostgreSQL sharding.
- **Storage**: ~1PB with S3 scaling, deduplication.
- **Uploads**: ~1K files/sec with Multipart Upload, CDN.

## Latency/Throughput Targets
- **Sync**: <500ms (WebSockets/Kafka).
- **Uploads**: <1s (chunked, presigned URLs).
- **Processing**: ~1K files/sec.

## Database Schemas
- **PostgreSQL FileMetadata Table**:
  ```sql
  CREATE TABLE file_metadata (
    file_id UUID PRIMARY KEY,
    user_id UUID,
    path TEXT,
    version INT,
    hash TEXT,
    status ENUM('uploading', 'uploaded'),
    chunks JSONB,
    created_at TIMESTAMP,
    updated_at TIMESTAMP
  );
  ```
  - **Consistency**: Strong for metadata (file_id, status) via transactions.
  - **Availability**: High with multi-AZ replication.
- **PostgreSQL SharedFiles Table**:
  ```sql
  CREATE TABLE shared_files (
    user_id UUID,
    file_id UUID,
    shared_at TIMESTAMP,
    PRIMARY KEY (user_id, file_id)
  );
  ```
  - **Consistency**: Strong for access control.
- **PostgreSQL VersionHistory Table**:
  ```sql
  CREATE TABLE version_history (
    file_id UUID,
    version INT,
    s3_key TEXT,
    delta_key TEXT,
    created_at TIMESTAMP,
    PRIMARY KEY (file_id, version)
  );
  ```
  - **Consistency**: Eventual for deltas.

## Cache Schemas
- **Redis**:
  - `file:file_id`: Serialized metadata (path, version).
  - `sync:user_id`: Pending sync events (file_id, timestamp).
  - **Consistency**: Eventual, sync with PostgreSQL on miss.
  - **TTL**: Short for cold files, long for hot.
- **CloudFront CDN**:
  - Cache files by `s3_key`.
  - **Consistency**: Eventual, invalidate on updates.
- **Tricks**: Redis pipelining for batch sync, CDN pre-fetch for popular files.

## Consistency vs. Availability
- **PostgreSQL**: 
  - **Strong Consistency**: For metadata, sharing (ACID transactions).
  - **High Availability**: Multi-AZ, read replicas.
- **S3**: 
  - **Eventual Consistency**: For file content, strong for metadata reads.
  - **High Availability**: Multi-region.
- **Redis**: 
  - **Eventual Consistency**: For cached metadata, sync events.
  - **High Availability**: Redis Cluster.
- **CDN**: 
  - **Eventual Consistency**: Cache invalidation on file updates.
  - **High Availability**: Global edge nodes.
- **Tricks**: PostgreSQL advisory locks for concurrent edits, S3 event triggers for metadata updates.

## Additional Considerations
- **Search**: Use **Elasticsearch** for file name/path search.
- **Notifications**: **Kafka** for sync events, WebSockets for real-time updates.
- **Rate Limiting**: Cap uploads/downloads per user.
- **Security**: Encrypt files with **KMS**, validate uploads for malware.
- **Analytics**: Log file operations in **Kafka**, aggregate in **Redshift**.
- **Backup**: S3 versioning, PostgreSQL snapshots to **Glacier**.

## Correctness Check
- **Original Design**: All components (S3, PostgreSQL, WebSockets, Kafka, CDN) align with scalability/latency goals. No incorrect details.
- **Missing Details**: Added search, notifications, analytics, security, detailed schemas, offline access for completeness.

## Summary
Dropbox uses **S3** for files, **PostgreSQL** for metadata, **presigned URLs** for uploads, **CDN** for downloads, and **WebSockets/Kafka** for syncing. It supports large files (chunking), deduplication, versioning, and secure sharing. Scales to 10M DAU, 1PB storage, 1K uploads/sec with <500ms sync, <1s uploads, and robust edge case handling (conflicts, corruption, quotas).

---
---


# YouTube/Netflix System Design

## Functional Requirements
- **Video Upload**: Upload videos with metadata (title, description, tags).
- **Video Streaming**: Stream videos with adaptive bitrate (HLS/DASH).
- **Recommendations**: Personalized video suggestions based on user behavior.
- **Search**: Search videos by title, tags, or uploader.
- **Interactions**: Like, comment, share, and subscribe.
- **Playback Controls**: Pause, seek, resume, and adjust quality.

## Non-Functional Requirements
- **Scalability**: Handle ~1B daily active users (DAU), ~10PB storage, ~100K QPS for streaming.
- **Latency**: <200ms for video start, <500ms for recommendations, ~50K streams/sec.
- **Availability**: 99.99% uptime, no single points of failure.
- **Consistency**: Eventual for video metadata/recommendations, strong for user actions (likes, subscriptions).
- **Security**: Secure video access, prevent unauthorized uploads/downloads.

## Improvements
- **Video Storage**: 
  - Store videos in **S3** with metadata {`video_id`, `title`, `uploader_id`, `description`, `tags`, `upload_time`} in **DynamoDB** for simple key-value access.
  - Use **Cassandra** for metadata partitioning by `video_id` for efficient point lookups.
  - Enable **presigned URLs** for direct, multi-part S3 uploads, bypassing backend.
  - **Post-Processing**: Split videos into short segments (HLS/DASH), transcode into multiple formats, generate manifests, and store in S3.
- **Video Watching**: 
  - Fetch **VideoMetadata** from DynamoDB/Cassandra, containing manifest URL in S3.
  - Client downloads manifest, selects segments based on network conditions, dynamically switches formats for smooth playback.
- **CDN**: Use **Akamai/Cloudflare** to cache video segments globally, reducing S3 load and latency.
- **Bitrate Streaming**: 
  - **DAG Pipeline** (via **Temporal**): 
    - Upload full video to S3.
    - Split into segments using **ffmpeg**.
    - Transcode in parallel (multiple resolutions/formats).
    - Generate audio/transcripts, create manifest files.
    - Store segments/manifests in S3, pass URLs between workers.
    - Mark upload complete in metadata.
- **Resumable Uploads**: 
  - Chunk videos into 5-10MB pieces, use **SHA-256** fingerprinting.
  - Track status in **VideoMetadata** (`chunks` field), update via S3 event notifications.
  - Resume by skipping uploaded chunks, use **Multipart Upload API**.
- **Recommendations**: 
  - Compute via **Spark** (collaborative filtering), store in **Redis** (`recs:user_id`, sorted set).
  - Periodically refresh for freshness.
- **Sharding**: 
  - Shard video metadata by `video_id` (DynamoDB/Cassandra).
  - Shard user data (views, likes) by `user_id`.
- **Edge Cases**: 
  - **Buffering**: Pre-fetch chunks via CDN.
  - **Geo-Restrictions**: IP-based filtering at CDN.
  - **High Demand**: Scale CDN edge nodes.
  - **Video Corruption**: Validate uploads (checksums, format checks).
  - **Recommendation Staleness**: Refresh Redis cache periodically.
- **Monitoring**: Track streaming latency, recommendation click-through rate, CDN hit ratio, playback success rate, S3 read/write throughput, Spark job latency.

## Trade-offs
- **Pre-compute vs. Real-time Recs**: 
  - **Pre-compute**: Efficient, potentially stale. 
  - **Real-time**: Fresh, compute-heavy. 
  - *Choice*: **Pre-compute** with periodic updates.
- **DynamoDB vs. Cassandra**: 
  - **DynamoDB**: Simpler, costlier. 
  - **Cassandra**: Better write scaling, complex. 
  - *Choice*: **DynamoDB** for ease, Cassandra for heavy writes.
- **S3 vs. Custom Storage**: 
  - **S3**: Reliable, costly. 
  - **Custom**: Cheaper, complex. 
  - *Choice*: **S3** for durability.

## Why S3 + CDN?
- **S3**: Scalable, durable storage for videos/segments.
- **CDN**: Low-latency global delivery, reduces origin load.

## Scalability Estimates
- **Users**: ~1B DAU via S3 partitioning, DynamoDB/Cassandra scaling.
- **Storage**: ~10PB with S3.
- **Streaming**: ~100K QPS with CDN.

## Latency/Throughput Targets
- **Video Start**: <200ms (CDN, manifest fetch).
- **Recommendations**: <500ms (Redis).
- **Streams**: ~50K/sec.

## Database Schemas
- **DynamoDB VideoMetadata Table**:
  ```sql
  Table: videos
  Partition Key: video_id (String)
  Attributes:
    title (String)
    uploader_id (String)
    description (String)
    tags (List<String>)
    upload_time (Number)
    manifest_url (String)
    chunks (Map<String, String>)
  ```
  - **Consistency**: Eventual for metadata, strong for upload status.
  - **Availability**: Multi-region.
- **Cassandra Videos Table** (alternative):
  ```sql
  CREATE TABLE videos (
    video_id UUID,
    title TEXT,
    uploader_id UUID,
    description TEXT,
    tags LIST<TEXT>,
    upload_time TIMESTAMP,
    manifest_url TEXT,
    chunks MAP<TEXT, TEXT>,
    PRIMARY KEY (video_id)
  );
  ```
  - **Consistency**: Eventual, quorum writes.
- **DynamoDB UserActions Table**:
  ```sql
  Table: user_actions
  Partition Key: user_id (String)
  Sort Key: video_id (String)
  Attributes: action_type (String), timestamp (Number)
  ```
  - **Consistency**: Strong for likes/subscriptions.

## Cache Schemas
- **Redis**:
  - `video:video_id`: Serialized metadata (title, manifest_url).
  - `recs:user_id`: Sorted set of video_ids by score.
  - **Consistency**: Eventual, sync with DynamoDB/Cassandra.
  - **TTL**: Short for cold videos, long for trending.
- **CDN (Akamai/Cloudflare)**:
  - Cache segments by `manifest_url` or `segment_url`.
  - **Consistency**: Eventual, purge on updates.
- **Tricks**: Redis pipelining for batch recs, CDN pre-fetch for trending videos.

## Consistency vs. Availability
- **DynamoDB/Cassandra**: 
  - **Eventual Consistency**: For metadata, recommendations.
  - **High Availability**: Multi-region, quorum reads/writes.
- **S3**: 
  - **Eventual Consistency**: For segments, strong for metadata.
  - **High Availability**: Multi-region.
- **Redis**: 
  - **Eventual Consistency**: For cached metadata/recs.
  - **High Availability**: Redis Cluster.
- **CDN**: 
  - **Eventual Consistency**: Cache invalidation on updates.
  - **High Availability**: Global edge nodes.
- **Tricks**: DynamoDB conditional writes for status updates, Cassandra lightweight transactions for critical metadata.

## Additional Considerations
- **Search**: **Elasticsearch** for title/tags/uploader search.
- **Notifications**: **Kafka** for interaction events, WebSocket/APNs for real-time alerts.
- **Analytics**: Log views/interactions in **Kafka**, aggregate in **Redshift**.
- **Security**: Encrypt videos in S3 with **KMS**, validate uploads for malware.
- **Rate Limiting**: Cap uploads/views per user.
- **Subtitles**: Generate/store transcripts in S3, serve via manifest.

## Correctness Check
- **Original Design**: All components (S3, DynamoDB/Cassandra, CDN, Spark, Temporal) align with scalability/latency goals. No incorrect details.
- **Missing Details**: Added search, notifications, analytics, security, subtitles, detailed schemas for completeness.

## Summary
YouTube/Netflix uses **S3** for videos, **DynamoDB/Cassandra** for metadata, **CDN** for streaming, and **Redis** for recommendations. It supports resumable uploads, adaptive bitrate streaming (HLS/DASH), and personalized recs via **Spark**. Scales to 1B DAU, 10PB storage, 100K QPS with <200ms video start, <500ms recs, and robust edge case handling (buffering, geo-restrictions, corruption).


---
---


# Yelp/Nearby Friends System Design

## Functional Requirements
- **Geospatial Search**: Find businesses or friends by location (lat, lon) or predefined areas (city, neighborhood).
- **Text Search**: Search by business name, category, or keywords.
- **Reviews/Ratings**: Submit one review per user per business, compute average ratings.
- **Filtering**: Filter results by distance, category, rating, or privacy settings.
- **User Profiles**: Manage user preferences and location data.

## Non-Functional Requirements
- **Scalability**: Handle ~10M daily active users (DAU), ~100K QPS for searches, ~1M businesses.
- **Latency**: <100ms for geo-queries, <200ms for search results, ~5K searches/sec.
- **Availability**: 99.99% uptime, no single points of failure.
- **Consistency**: Eventual for search results/ratings, strong for review submissions.
- **Security**: Prevent location spoofing, ensure privacy controls.

## Write-Heavy Considerations
- **High Throughput**: Handle ~1 write/sec for reviews (low volume for 100M users).
- **Durability**: Ensure review/rating data persists reliably.
- **Ingestion Speed**: Fast writes for user reviews with minimal latency.

## Improvements
- **Geospatial Index**: Use **Redis Geo** for fast location queries {`business_id`, `lat`, `lon`}. Alternative: **PostgreSQL PostGIS** for complex geospatial operations.
- **Search**: Index business metadata (name, category, tags, location_id) in **Elasticsearch** for full-text and geo-based searches.
- **Caching**: Cache popular searches (`search:query:lat:lon`) and nearby results (`nearby:lat:lon:radius`) in **Redis** with TTL-based eviction (e.g., 1hr).
- **Ratings**: 
  - Store precomputed average in business record, updated incrementally: `(old_avg * old_count + new_rating) / (old_count + 1)`.
  - Avoid on-the-fly calculations for scalability.
- **One-Review-Per-Business**: Enforce via unique constraint on `user_id`, `business_id` in reviews table (PostgreSQL/DynamoDB).
- **Complex Search Queries**: 
  - Use **quadtree indexing** in Elasticsearch for geospatial searches.
  - Apply **Haversine formula** to filter by distance, then refine with name/category filters.
- **Predefined Location Names**: 
  - **Locations Table**: Map names (e.g., "San Francisco") to GeoJSON polygons, indexed for fast lookups.
  - Store precomputed `location_id` (e.g., `san_francisco`) in business records (Elasticsearch keyword field).
  - Query polygons, filter businesses by `location_id` to avoid per-request polygon checks.
- **Edge Cases**: 
  - **Location Spoofing**: Validate via IP geolocation.
  - **Stale Data**: Refresh caches/business data periodically.
  - **High QPS**: Use **load balancers** (e.g., AWS ALB).
  - **Privacy Settings**: Filter results on read based on user preferences.
  - **Inaccurate Coordinates**: Validate lat/lon inputs (range checks).
- **Monitoring**: Track search latency, geo-query accuracy, cache hit ratio, review write throughput, search success rate.

## Network Protocols & APIs
- **Protocols**: 
  - **HTTPS**: Secure API communication, encryption in transit.
  - **gRPC**: High-performance internal microservices (e.g., search, geo-queries).
  - **WebSocket**: Real-time location updates for friends (if applicable).
- **APIs**:
  - `POST /search`: Search businesses by query, lat/lon, filters (e.g., category, rating). Returns paginated results.
    ```json
    {
      "query": "coffee",
      "lat": 37.7749,
      "lon": -122.4194,
      "radius": 5000,
      "filters": {"category": "cafe", "min_rating": 4}
    }
    ```
  - `GET /nearby`: Fetch nearby businesses/friends by lat/lon, radius. Returns ranked list.
  - `POST /review`: Submit review/rating for a business, enforce one-per-user.
    ```json
    {
      "user_id": "uuid",
      "business_id": "uuid",
      "rating": 5,
      "comment": "Great service!"
    }
    ```

## Trade-offs
- **Redis Geo vs. PostGIS**: 
  - **Redis Geo**: Faster for simple queries, limited complexity. 
  - **PostGIS**: Handles complex geospatial ops, slower. 
  - *Choice*: **Redis Geo** for low-latency lookups.
- **Elasticsearch vs. DB**: 
  - **Elasticsearch**: Optimized for search, less durable. 
  - **DB**: Durable, slower for complex queries. 
  - *Choice*: **Elasticsearch** for fast searches.
- **Caching vs. No Caching**: 
  - **Caching**: Reduces latency, risks staleness. 
  - **No Caching**: Fresh, slower. 
  - *Choice*: **Caching** with TTL.

## Why Redis + Elasticsearch?
- **Redis**: Fast geo-queries and caching for low-latency responses.
- **Elasticsearch**: Efficient text/geo searches with quadtree indexing.

## Scalability Estimates
- **Users**: ~10M DAU via Redis clustering, Elasticsearch sharding.
- **Searches**: ~100K QPS with load balancing, caching.
- **Businesses**: ~1M with Elasticsearch indexing.

## Latency/Throughput Targets
- **Geo-Queries**: <100ms (Redis Geo).
- **Search Results**: <200ms (Elasticsearch).
- **Searches**: ~5K/sec.

## Database Schemas
- **PostgreSQL Businesses Table**:
  ```sql
  CREATE TABLE businesses (
    business_id UUID PRIMARY KEY,
    name TEXT,
    lat DOUBLE PRECISION,
    lon DOUBLE PRECISION,
    category TEXT,
    avg_rating FLOAT,
    review_count INT,
    location_id TEXT
  );
  ```
  - **Consistency**: Eventual for avg_rating, strong for metadata.
  - **Availability**: Multi-AZ replication.
- **PostgreSQL Reviews Table**:
  ```sql
  CREATE TABLE reviews (
    review_id UUID PRIMARY KEY,
    user_id UUID,
    business_id UUID,
    rating INT,
    comment TEXT,
    created_at TIMESTAMP,
    UNIQUE (user_id, business_id)
  );
  ```
  - **Consistency**: Strong for review writes (unique constraint).
- **PostgreSQL Locations Table**:
  ```sql
  CREATE TABLE locations (
    location_id TEXT PRIMARY KEY,
    name TEXT,
    polygon JSONB
  );
  ```
  - **Consistency**: Strong for lookups.

## Cache Schemas
- **Redis**:
  - `geo:businesses`: Geo set {`business_id`, `lat`, `lon`}.
  - `search:query:lat:lon`: Cached search results (JSON).
  - `nearby:lat:lon:radius`: Cached nearby results.
  - **Consistency**: Eventual, sync with DB on miss.
  - **TTL**: 1hr for searches, 10min for nearby.
- **Tricks**: Redis pipelining for batch geo-queries, LRU eviction for cache.

## Consistency vs. Availability
- **PostgreSQL**: 
  - **Strong Consistency**: For review writes, metadata (ACID).
  - **High Availability**: Multi-AZ, read replicas.
- **Elasticsearch**: 
  - **Eventual Consistency**: For search indices.
  - **High Availability**: Sharded clusters.
- **Redis**: 
  - **Eventual Consistency**: For cached results.
  - **High Availability**: Redis Cluster.
- **Tricks**: PostgreSQL advisory locks for review writes, Elasticsearch snapshot backups.

## Additional Considerations
- **Rate Limiting**: Cap search/review submissions per user.
- **Security**: Encrypt data in transit (HTTPS), validate IP for location.
- **Analytics**: Log searches/reviews in **Kafka**, aggregate in **Redshift**.
- **Notifications**: **Kafka** for review events, WebSocket for friend updates.
- **GeoJSON Data**: Import from public datasets (e.g., OpenStreetMap).

## Network Protocols & APIs (Write-Heavy Optimization)
- **Protocols**: 
  - **HTTPS**: Secure, durable API calls.
  - **gRPC**: Fast internal communication for review writes.
  - **WebSocket**: Real-time friend location updates.
- **APIs**: Optimized for low write latency, high durability.
  - `POST /review`: Asynchronous write with immediate acknowledgment, durable in PostgreSQL.
  - `GET /search`: Cache-first, fallback to Elasticsearch for freshness.

## Correctness Check
- **Original Design**: All components (Redis Geo, Elasticsearch, PostgreSQL) align with scalability/latency goals. No incorrect details.
- **Missing Details**: Added APIs, protocols, analytics, notifications, security, and detailed schemas for completeness.

## Summary
Yelp/Nearby Friends uses **Redis Geo** for fast location queries, **Elasticsearch** for text/geo searches, and **PostgreSQL** for durable reviews/ratings. It supports complex queries with **quadtree indexing**, predefined locations via **GeoJSON**, and enforces one-review-per-business. Scales to 10M DAU, 100K QPS with <100ms geo-queries, <200ms searches, and robust edge case handling (spoofing, privacy). APIs and protocols optimize for write-heavy review ingestion.

---
---

# Rate Limiter System Design

## Functional Requirements
- **Rate Limiting**: Enforce per-user or global API request limits (e.g., 100 requests/min).
- **Rule Management**: Allow service teams to create/update/delete rate-limiting rules.
- **Burst Handling**: Support temporary request bursts within defined limits.
- **Distributed Enforcement**: Apply consistent limits across multiple hosts.
- **Monitoring/Logging**: Track limit violations and usage for auditing.

## Non-Functional Requirements
- **Scalability**: Handle ~100K QPS for rate checks, ~1M users, ~10K limits/sec.
- **Latency**: <10ms for rate checks, ~50K checks/sec, <50ms for token refills.
- **Availability**: 99.99% uptime, no single points of failure.
- **Consistency**: Strong for token updates to prevent over-limiting.
- **Security**: Prevent abuse (e.g., DDoS), ensure accurate limits.

## Write-Heavy Considerations
- **High Throughput**: Handle frequent token updates for active users.
- **Durability**: Persist rule configurations reliably.
- **Ingestion Speed**: Fast token checks/updates with minimal latency.

## Improvements
- **Algorithm**: Use **token bucket** for burst flexibility, storing {`user_id`, `tokens`, `last_refill_timestamp`} in **Redis**. Refill tokens based on rate (e.g., 100/min).
- **TTL Logic**: Set **Redis TTL** to bucket refresh interval (e.g., 1min) to auto-expire stale counters.
- **Distributed**: 
  - Deploy **Redis Cluster** for scalability.
  - Use **Lua scripts** for atomic token updates, avoiding race conditions.
  - Each host runs a token bucket; requests for the same client may hit different hosts.
- **Negative Tokens**: Allow temporary negative token balances to handle distributed bursts, ensuring throttling compensates post-burst.
- **Inter-Host Communication**: 
  - Use **shared Redis cache** for token state synchronization across hosts.
  - Alternative: **Gossip protocol** for scalable updates, balancing reliability/speed.
  - Use **TCP** for reliable cache updates, **UDP** for low-latency gossip (if chosen).
- **Memory Management**: Remove inactive client buckets after timeout (e.g., 5min). Recreate buckets on new requests to optimize memory.
- **Rule Management**: 
  - Store rules {`rule_id`, `user_id`, `endpoint`, `limit`, `interval`} in **PostgreSQL**.
  - Provide **REST API** for rule CRUD operations, cached in Redis for fast access.
- **Edge Cases**: 
  - **Clock Skew**: Use monotonic clocks for consistent timestamps.
  - **Bursty Traffic**: Adjust bucket size for controlled bursts.
  - **DDoS Attacks**: Apply global rate limits, block abusive IPs.
  - **Token Exhaustion**: Queue requests or reject with retry headers.
  - **Misconfigured Limits**: Maintain audit logs for rule changes.
- **Monitoring**: Track rejection rate, token refill latency, Redis throughput, rate limit accuracy, rule update latency.

## Network Protocols & APIs
- **Protocols**: 
  - **HTTPS**: Secure API for rate checks and rule management.
  - **gRPC**: High-performance internal communication for token updates.
  - **TCP**: Reliable Redis synchronization.
- **APIs**:
  - `GET /rate/check`: Check if request allowed for user/endpoint.
    ```json
    {
      "user_id": "uuid",
      "endpoint": "/api/v1/data",
      "allowed": true,
      "remaining": 95,
      "reset_time": 1697056800
    }
    ```
  - `POST /rules`: Create/update rate limit rule.
    ```json
    {
      "user_id": "uuid",
      "endpoint": "/api/v1/data",
      "limit": 100,
      "interval": "60s"
    }
    ```

## Trade-offs
- **Token vs. Leaky Bucket**: 
  - **Token Bucket**: Allows bursts, API-friendly. 
  - **Leaky Bucket**: Strict rates, hardware-suited. 
  - *Choice*: **Token Bucket** for flexibility.
- **Redis vs. In-Memory**: 
  - **Redis**: Durable, distributed. 
  - **In-Memory**: Faster, volatile. 
  - *Choice*: **Redis** for reliability.
- **Global vs. Per-User Limits**: 
  - **Global**: Prevents abuse, coarse. 
  - **Per-User**: Precise, complex. 
  - *Choice*: Both for comprehensive control.

## Why Token Bucket + Redis?
- **Token Bucket**: Handles bursts, suitable for APIs.
- **Redis**: Scales distributed systems, ensures atomic updates.

## Scalability Estimates
- **Rate Checks**: ~100K QPS via Redis Cluster.
- **Users**: ~1M with sharded buckets.
- **Limits**: ~10K/sec rule enforcement.

## Latency/Throughput Targets
- **Rate Checks**: <10ms (Redis).
- **Checks**: ~50K/sec.
- **Token Refills**: <50ms.

## Database Schemas
- **PostgreSQL Rules Table**:
  ```sql
  CREATE TABLE rate_limit_rules (
    rule_id UUID PRIMARY KEY,
    user_id UUID,
    endpoint TEXT,
    limit INT,
    interval INT,
    created_at TIMESTAMP,
    updated_at TIMESTAMP
  );
  ```
  - **Consistency**: Strong for rule updates (ACID).
  - **Availability**: Multi-AZ replication.

## Cache Schemas
- **Redis**:
  - `bucket:user_id:endpoint`: Hash {`tokens`, `last_refill_timestamp`}.
  - `rules:endpoint`: Cached rule data (limit, interval).
  - **Consistency**: Strong via Lua scripts for token updates.
  - **TTL**: 1min for buckets, 1hr for rules.
- **Tricks**: Redis pipelining for batch checks, LRU for memory management.

## Consistency vs. Availability
- **PostgreSQL**: 
  - **Strong Consistency**: For rule management (transactions).
  - **High Availability**: Multi-AZ, read replicas.
- **Redis**: 
  - **Strong Consistency**: For token updates (Lua scripts).
  - **High Availability**: Redis Cluster with replicas.
- **Tricks**: Redis atomic operations for distributed buckets, PostgreSQL advisory locks for rule updates.

## Additional Considerations
- **Security**: Validate API keys, block abusive IPs via **WAF**.
- **Analytics**: Log rate limit events in **Kafka**, aggregate in **Redshift**.
- **Notifications**: Alert on frequent rejections via **SNS**.
- **Rule Auditing**: Store rule change logs in PostgreSQL.
- **Load Balancing**: Distribute requests via **AWS ALB**.

## Correctness Check
- **Original Design**: All components (token bucket, Redis, PostgreSQL) align with scalability/latency goals. No incorrect details.
- **Missing Details**: Added APIs, protocols, analytics, notifications, security, and detailed schemas for completeness.

## Summary
Rate Limiter uses **token bucket** with **Redis Cluster** for distributed, atomic rate checks, allowing bursts and negative tokens. Rules are managed in **PostgreSQL**, cached in Redis. Scales to 100K QPS, 1M users with <10ms checks, ~50K checks/sec. Handles edge cases (DDoS, skew) and uses **HTTPS/gRPC** for secure, fast communication. Comprehensive monitoring and analytics ensure FAANG-ready performance.


---
---


# Log Collection and Analysis System Design

## Functional Requirements
- **Log Ingestion**: Collect logs from diverse sources (apps, servers, services).
- **Log Processing**: Parse, enrich, and index logs for querying.
- **Real-time Querying**: Search logs by keywords, timestamps, or metadata.
- **Analytics**: Generate insights (e.g., error rates, usage trends) via batch processing.
- **Visualization**: Display logs and analytics in dashboards.
- **Alerting**: Notify on specific log patterns (e.g., errors).

## Non-Functional Requirements
- **Scalability**: Handle ~1M logs/sec, ~1PB storage, ~10K QPS for queries.
- **Latency**: <100ms for log ingestion, <500ms for queries, ~100K logs/sec processing.
- **Availability**: 99.99% uptime, no single points of failure.
- **Consistency**: Eventual for log indexing, strong for alerting rules.
- **Security**: Encrypt logs in transit/storage, control access.

## Write-Heavy Considerations
- **High Throughput**: Ingest ~1M logs/sec with minimal latency.
- **Durability**: Ensure no log loss during ingestion.
- **Ingestion Speed**: Fast, reliable write pipeline for logs.

## Improvements
- **Ingestion**: 
  - Use **Kafka** for high-throughput log ingestion, partitioned by `source` or `timestamp`.
  - Producers send logs to topics (e.g., `app_logs`, `server_logs`).
- **Processing**: 
  - **ELK Stack**: 
    - **Logstash**: Parse/enrich logs (e.g., extract fields, add timestamps).
    - **Elasticsearch**: Store/index logs for fast searches.
    - **Kibana**: Visualize logs via dashboards.
  - **Spark**: Batch analytics for trends (e.g., error spikes, usage patterns).
- **Buffering**: 
  - Buffer logs in **Kafka** to handle ingestion spikes.
  - Use **consumer groups** for parallel processing by Logstash workers.
- **Real-time Querying**: 
  - Index logs in **Elasticsearch** with time-based indices (e.g., `logs-YYYY-MM-DD`).
  - Optimize for keyword, range, and metadata searches.
- **Edge Cases**: 
  - **Log Spikes**: Scale Kafka partitions dynamically.
  - **Malformed Logs**: Validate (schema checks) before ingestion, divert invalid logs to dead-letter queue.
  - **Data Retention**: Set **TTL policies** in Elasticsearch (e.g., 30 days).
  - **Consumer Lag**: Scale Logstash/Spark workers, monitor lag.
  - **Log Duplication**: Use idempotency keys (e.g., `log_id`) in Kafka messages.
- **Monitoring**: Track ingestion latency, query response time, Kafka partition lag, log processing success rate, Elasticsearch indexing rate.

## Network Protocols & APIs
- **Protocols**: 
  - **HTTPS**: Secure log ingestion and query APIs.
  - **gRPC**: Fast internal communication for log processing.
  - **TCP**: Reliable Kafka producer/consumer connections.
- **APIs**:
  - `POST /logs`: Ingest logs with source, timestamp, and payload.
    ```json
    {
      "source": "app1",
      "timestamp": 1697056800000,
      "log_id": "uuid",
      "payload": {"level": "error", "message": "Failed to connect"}
    }
    ```
  - `GET /search`: Query logs by keyword, time range, or metadata.
    ```json
    {
      "query": "error",
      "start_time": 1697056800000,
      "end_time": 1697056860000,
      "filters": {"source": "app1"}
    }
    ```

## Trade-offs
- **Kafka vs. RabbitMQ**: 
  - **Kafka**: High-throughput, durable, partitioned. 
  - **RabbitMQ**: Simpler, less scalable. 
  - *Choice*: **Kafka** for log volume.
- **Elasticsearch vs. ClickHouse**: 
  - **Elasticsearch**: Flexible text search, general-purpose. 
  - **ClickHouse**: Faster analytics, less flexible. 
  - *Choice*: **Elasticsearch** for querying.
- **Real-time vs. Batch Processing**: 
  - **Real-time**: Fast, resource-heavy. 
  - **Batch**: Efficient, delayed. 
  - *Choice*: Both (Elasticsearch for real-time, Spark for batch).

## Why Kafka + ELK?
- **Kafka**: Reliable, scalable log ingestion with partitioning.
- **ELK**: Robust search, indexing, and visualization for logs.

## Scalability Estimates
- **Logs**: ~1M logs/sec via Kafka partitioning.
- **Storage**: ~1PB with Elasticsearch sharding, compression.
- **Queries**: ~10K QPS with Elasticsearch clusters.

## Latency/Throughput Targets
- **Ingestion**: <100ms (Kafka).
- **Queries**: <500ms (Elasticsearch).
- **Processing**: ~100K logs/sec.

## Database Schemas
- **Elasticsearch Logs Index**:
  ```json
  {
    "mappings": {
      "properties": {
        "log_id": {"type": "keyword"},
        "source": {"type": "keyword"},
        "timestamp": {"type": "date"},
        "level": {"type": "keyword"},
        "message": {"type": "text"},
        "metadata": {"type": "object"}
      }
    }
  }
  ```
  - **Consistency**: Eventual for indexing.
  - **Availability**: Sharded, replicated clusters.
- **Kafka Topics**:
  - `logs`: Partitioned by `source` or `timestamp`.
  - Message: `{log_id, source, timestamp, payload}`.
  - **Consistency**: Durable with replication.

## Cache Schemas
- **Redis**:
  - `query:query_hash`: Cached search results (JSON).
  - `metrics:source`: Aggregated metrics (e.g., error counts).
  - **Consistency**: Eventual, sync with Elasticsearch.
  - **TTL**: 10min for queries, 1hr for metrics.
- **Tricks**: Redis pipelining for batch queries, LRU eviction for cache.

## Consistency vs. Availability
- **Kafka**: 
  - **Strong Consistency**: For log durability (replication).
  - **High Availability**: Multi-broker clusters.
- **Elasticsearch**: 
  - **Eventual Consistency**: For log indexing.
  - **High Availability**: Sharded, replicated indices.
- **Redis**: 
  - **Eventual Consistency**: For cached queries/metrics.
  - **High Availability**: Redis Cluster.
- **Tricks**: Kafka idempotent producers, Elasticsearch snapshot backups.

## Additional Considerations
- **Security**: Encrypt logs in transit (HTTPS), at rest (Elasticsearch encryption).
- **Alerting**: Use **Kibana** for rule-based alerts, notify via **SNS**.
- **Rate Limiting**: Cap log ingestion per source to prevent abuse.
- **Analytics**: **Spark** for deep insights (e.g., anomaly detection).
- **Backup**: Archive logs to **S3/Glacier** for long-term storage.

## Correctness Check
- **Original Design**: All components (Kafka, ELK, Spark) align with scalability/latency goals. No incorrect details.
- **Missing Details**: Added APIs, protocols, alerting, security, analytics, and detailed schemas for completeness.

## Summary
Log Collection system uses **Kafka** for high-throughput ingestion, **ELK Stack** for parsing/storage/visualization, and **Spark** for analytics. It scales to 1M logs/sec, 1PB storage, 10K QPS with <100ms ingestion, <500ms queries. Handles edge cases (spikes, malformed logs) with buffering, validation, and TTLs. **HTTPS/gRPC** ensure secure, fast communication, making it FAANG-ready.

---
---


# Voting System Design

## Functional Requirements
- **Vote Submission**: Users submit votes for polls with one vote per user per poll.
- **Vote Counting**: Display real-time vote counts for each poll option.
- **Fraud Prevention**: Prevent duplicate votes and detect suspicious activity.
- **Vote Auditing**: Maintain auditable records of all votes.
- **Result Reporting**: Provide final tallies and vote history.

## Non-Functional Requirements
- **Scalability**: Handle ~10K votes/sec, ~1M voters, ~100K QPS for counts.
- **Latency**: <50ms for vote submission, <100ms for counts, ~5K votes/sec processing.
- **Availability**: 99.99% uptime, no single points of failure.
- **Consistency**: Strong for vote submissions to prevent duplicates, eventual for counts.
- **Security**: Prevent fraud, impersonation, and unauthorized access.

## Write-Heavy Considerations
- **High Throughput**: Process ~10K votes/sec during peak events.
- **Durability**: Ensure votes are persisted reliably for auditing.
- **Ingestion Speed**: Fast vote submission with immediate feedback.

## Improvements
- **Idempotency**: 
  - Use **vote_id** (`user_id + poll_id`) for uniqueness checks in **Redis** (`vote:user_id:poll_id`).
  - Store votes in **PostgreSQL** for durability.
- **Fraud Prevention**: 
  - **Rate-limiting**: Cap vote attempts per user/IP (Redis-based).
  - **CAPTCHA**: Trigger for suspicious patterns (e.g., rapid votes).
  - Log all votes in PostgreSQL for auditing.
- **Aggregation**: 
  - **Redis**: Real-time vote counts (`counter:poll_id:option_id`, increment).
  - **Spark**: Batch jobs for final tallies, reconciling inconsistencies.
- **Edge Cases**: 
  - **Vote Reversals**: Store vote history in PostgreSQL, allow updates with audit trail.
  - **Network Failures**: Implement retry logic with exponential backoff.
  - **Result Disputes**: Maintain detailed audit logs in PostgreSQL.
  - **Voter Impersonation**: Enforce **JWT-based auth** for vote submissions.
  - **High Vote Spikes**: Scale Redis Cluster, buffer votes in **Kafka** if needed.
- **Monitoring**: Track vote throughput, fraud detection rate, count accuracy, vote processing latency, Redis hit rate, PostgreSQL write latency.

## Network Protocols & APIs
- **Protocols**: 
  - **HTTPS**: Secure vote submission and count retrieval.
  - **gRPC**: Fast internal communication for vote processing.
  - **TCP**: Reliable Redis/PostgreSQL connections.
- **APIs**:
  - `POST /vote`: Submit vote for a poll option.
    ```json
    {
      "user_id": "uuid",
      "poll_id": "uuid",
      "option_id": "uuid",
      "timestamp": 1697056800000
    }
    ```
  - `GET /poll/:poll_id/counts`: Retrieve real-time vote counts.
    ```json
    {
      "poll_id": "uuid",
      "counts": {"option_id1": 500, "option_id2": 300}
    }
    ```

## Trade-offs
- **Real-time vs. Eventual Counts**: 
  - **Real-time**: User-friendly, risks inconsistency. 
  - **Eventual**: Consistent, delayed. 
  - *Choice*: **Real-time** with periodic Spark reconciliation.
- **Redis vs. DB**: 
  - **Redis**: Fast counters, volatile. 
  - **DB**: Durable, slower. 
  - *Choice*: **Redis** with PostgreSQL backup.
- **Rate-Limiting vs. CAPTCHA**: 
  - **Rate-Limiting**: Lightweight, coarse. 
  - **CAPTCHA**: Robust, intrusive. 
  - *Choice*: Both for layered security.

## Why Redis + PostgreSQL?
- **Redis**: Low-latency vote counting and idempotency checks.
- **PostgreSQL**: Durable storage for votes and audit trails.

## Scalability Estimates
- **Votes**: ~10K votes/sec via Redis Cluster, Kafka buffering.
- **Voters**: ~1M with sharded Redis/PostgreSQL.
- **Counts**: ~100K QPS for count queries.

## Latency/Throughput Targets
- **Vote Submission**: <50ms (Redis/PostgreSQL).
- **Counts**: <100ms (Redis).
- **Processing**: ~5K votes/sec.

## Database Schemas
- **PostgreSQL Votes Table**:
  ```sql
  CREATE TABLE votes (
    vote_id TEXT PRIMARY KEY, -- user_id + poll_id
    user_id UUID,
    poll_id UUID,
    option_id UUID,
    timestamp TIMESTAMP,
    UNIQUE (user_id, poll_id)
  );
  ```
  - **Consistency**: Strong for vote writes (unique constraint).
  - **Availability**: Multi-AZ replication.
- **PostgreSQL VoteHistory Table**:
  ```sql
  CREATE TABLE vote_history (
    history_id UUID PRIMARY KEY,
    vote_id TEXT,
    user_id UUID,
    poll_id UUID,
    option_id UUID,
    action TEXT, -- 'submit', 'reverse'
    timestamp TIMESTAMP
  );
  ```
  - **Consistency**: Strong for audit trails.

## Cache Schemas
- **Redis**:
  - `vote:user_id:poll_id`: Flag for idempotency (exists or not).
  - `counter:poll_id:option_id`: Vote count (integer).
  - **Consistency**: Strong for idempotency (Lua scripts), eventual for counts.
  - **TTL**: 24hr for vote flags, none for counters (reconciled by Spark).
- **Tricks**: Redis pipelining for batch vote checks, LRU for memory management.

## Consistency vs. Availability
- **PostgreSQL**: 
  - **Strong Consistency**: For vote submissions, history (ACID).
  - **High Availability**: Multi-AZ, read replicas.
- **Redis**: 
  - **Strong Consistency**: For idempotency checks (Lua scripts).
  - **Eventual Consistency**: For vote counts.
  - **High Availability**: Redis Cluster.
- **Tricks**: PostgreSQL advisory locks for vote writes, Redis atomic operations for counters.

## Additional Considerations
- **Security**: 
  - **JWT auth** for vote submissions.
  - Block suspicious IPs via **WAF**.
- **Analytics**: Log votes in **Kafka**, aggregate in **Redshift** for fraud analysis.
- **Notifications**: Alert on fraud detection via **SNS**.
- **Rate Limiting**: Use Redis-based token bucket for vote attempts.
- **Backup**: PostgreSQL snapshots to **S3/Glacier** for audit logs.

## Correctness Check
- **Original Design**: All components (Redis, PostgreSQL, Spark) align with scalability/latency goals. No incorrect details.
- **Missing Details**: Added APIs, protocols, analytics, notifications, security, and detailed schemas for completeness.

## Summary
Voting system uses **token bucket rate-limiting** with **Redis** for fast idempotency checks and real-time counts, backed by **PostgreSQL** for durable votes and audits. It scales to 10K votes/sec, 1M voters, 100K QPS with <50ms submissions, <100ms counts. Handles fraud (CAPTCHA, rate-limiting), reversals, and spikes with **Kafka** buffering and **Spark** reconciliation. **HTTPS/gRPC** ensure secure, fast communication, making it FAANG-ready.

---
---

# Trending Topics System Design

## Functional Requirements
- **Topic Counting**: Track mentions of topics (e.g., hashtags, keywords) across platforms.
- **Trending Rankings**: Rank topics by mentions and recency in a sliding window.
- **Query Trends**: Retrieve top trending topics with low latency.
- **Topic Normalization**: Handle variations (e.g., #AI vs. #ArtificialIntelligence).
- **Analytics**: Provide insights into topic trends over time.

## Non-Functional Requirements
- **Scalability**: Handle ~100K mentions/sec, ~1M topics, ~10K QPS for rankings.
- **Latency**: <50ms for mention updates, <200ms for rankings, ~10K trends/sec processing.
- **Availability**: 99.99% uptime, no single points of failure.
- **Consistency**: Eventual for counts/rankings, strong for configuration data.
- **Security**: Prevent manipulation of trends (e.g., bot-driven spikes).

## Write-Heavy Considerations
- **High Throughput**: Process ~100K mentions/sec during peak events.
- **Durability**: Persist counts for auditing and historical analysis.
- **Ingestion Speed**: Fast mention updates with minimal latency.

## Improvements
- **Counting**: 
  - Use **count-min sketch** in **Redis** for memory-efficient, approximate counting of topic mentions (`sketch:topic`).
  - Store exact counts for top topics in Redis for accuracy.
- **Sliding Window**: 
  - Implement 1-hour sliding window using **Redis sorted sets** (`mentions:topic`, score: timestamp).
  - Expire old mentions via Redis TTL or cleanup job.
- **Ranking**: 
  - Compute trending scores (mentions + recency with **exponential decay**) via **Spark** background job.
  - Cache rankings in Redis (`trends:window`, sorted set) for fast retrieval.
- **Edge Cases**: 
  - **Sudden Spikes**: Apply exponential decay to dampen bot-driven surges.
  - **Stale Trends**: Expire old data via TTL or periodic cleanup.
  - **Low-Memory**: Evict cold topics from Redis using LRU.
  - **Topic Ambiguity**: Normalize terms (e.g., lowercase, synonym mapping) before counting.
  - **Skewed Data**: Cap outlier mention counts to prevent manipulation.
- **Monitoring**: Track sketch accuracy, ranking latency, cache hit ratio, trend refresh rate, Redis memory usage, Spark job duration.

## Network Protocols & APIs
- **Protocols**: 
  - **HTTPS**: Secure API for mention ingestion and trend queries.
  - **gRPC**: High-performance internal communication for count updates.
  - **TCP**: Reliable Redis connections.
- **APIs**:
  - `POST /mentions`: Record topic mention.
    ```json
    {
      "topic": "#AI",
      "timestamp": 1697056800000,
      "source": "tweet"
    }
    ```
  - `GET /trends`: Retrieve top trending topics.
    ```json
    {
      "window": "1h",
      "limit": 10,
      "trends": [{"topic": "#AI", "score": 1500}, ...]
    }
    ```

## Trade-offs
- **Count-Min vs. Exact Counting**: 
  - **Count-Min**: Memory-efficient, approximate. 
  - **Exact**: Accurate, memory-heavy. 
  - *Choice*: **Count-Min** for scalability.
- **Sliding vs. Fixed Window**: 
  - **Sliding**: Precise, complex. 
  - **Fixed**: Simpler, less granular. 
  - *Choice*: **Sliding** for real-time trends.
- **Redis vs. In-Memory**: 
  - **Redis**: Durable, scalable. 
  - **In-Memory**: Faster, volatile. 
  - *Choice*: **Redis** for reliability.

## Why Count-Min + Redis?
- **Count-Min**: Scales for high-cardinality topics with low memory.
- **Redis**: Low-latency updates and ranking retrieval.

## Scalability Estimates
- **Mentions**: ~100K mentions/sec via Redis sharding.
- **Topics**: ~1M with count-min sketch compression.
- **Rankings**: ~10K QPS for trend queries.

## Latency/Throughput Targets
- **Mention Updates**: <50ms (Redis).
- **Rankings**: <200ms (Redis/Spark).
- **Processing**: ~10K trends/sec.

## Database Schemas
- **PostgreSQL Config Table** (for topic normalization rules):
  ```sql
  CREATE TABLE topic_config (
    config_id UUID PRIMARY KEY,
    topic TEXT,
    normalized_topic TEXT,
    created_at TIMESTAMP
  );
  ```
  - **Consistency**: Strong for configuration (ACID).
  - **Availability**: Multi-AZ replication.

## Cache Schemas
- **Redis**:
  - `sketch:topic`: Count-min sketch for mention counts.
  - `mentions:topic`: Sorted set {`mention_id`, `timestamp`}.
  - `trends:window`: Sorted set {`topic`, `score`}.
  - **Consistency**: Eventual for counts/rankings, sync with Spark.
  - **TTL**: 1hr for mentions, 10min for trends.
- **Tricks**: Redis pipelining for batch updates, LRU for cold topics.

## Consistency vs. Availability
- **PostgreSQL**: 
  - **Strong Consistency**: For topic normalization rules (transactions).
  - **High Availability**: Multi-AZ, read replicas.
- **Redis**: 
  - **Eventual Consistency**: For sketches, rankings.
  - **High Availability**: Redis Cluster with replicas.
- **Tricks**: Redis atomic operations for count updates, PostgreSQL advisory locks for config updates.

## Additional Considerations
- **Security**: 
  - Validate mentions via **WAF** to block bot spam.
  - Use **JWT auth** for API access.
- **Analytics**: Log mentions in **Kafka**, aggregate in **Redshift** for trend analysis.
- **Notifications**: Alert on anomalous spikes via **SNS**.
- **Rate Limiting**: Cap mention submissions per source (Redis token bucket).
- **Backup**: Archive historical trends to **S3/Glacier**.

## Correctness Check
- **Original Design**: All components (count-min sketch, Redis, Spark) align with scalability/latency goals. No incorrect details.
- **Missing Details**: Added APIs, protocols, analytics, notifications, security, and schemas for completeness.

## Summary
Trending Topics system uses **count-min sketch** in **Redis** for memory-efficient counting, **sliding window** for real-time tracking, and **Spark** for ranking. It scales to 100K mentions/sec, 1M topics, 10K QPS with <50ms updates, <200ms rankings. Handles spikes, ambiguity, and low-memory scenarios with **Kafka** for analytics and **HTTPS/gRPC** for secure, fast communication. FAANG-ready with robust monitoring and security.


---
---

# Facebook Messenger System Design

## Functional Requirements
- **Message Sending**: Send text, images, videos, and files in 1:1 or group chats.
- **Message Delivery**: Deliver messages in real-time to online users and store for offline users.
- **Notifications**: Push notifications for new messages.
- **Read Receipts**: Track message read status.
- **Chat Management**: Create/manage group chats, mute, or leave conversations.
- **Search**: Search message history by keywords or sender.

## Non-Functional Requirements
- **Scalability**: Handle ~100M daily active users (DAU), ~1B messages/day, ~50K QPS for delivery.
- **Latency**: <100ms for message delivery, <200ms for notifications, ~10K messages/sec ingestion.
- **Availability**: 99.99% uptime, no single points of failure.
- **Consistency**: Eventual for message delivery, strong for chat metadata (e.g., group membership).
- **Security**: Encrypt messages in transit/storage, ensure user authentication.

## Write-Heavy Considerations
- **High Throughput**: Ingest ~1B messages/day (~12K messages/sec peak).
- **Durability**: Persist messages reliably for history and offline delivery.
- **Ingestion Speed**: Fast message processing with minimal latency.

## Improvements
- **Message Ingestion**: 
  - Use **Kafka** for high-throughput queues, partitioned by `chat_id` for load balancing.
  - Producers send messages to topics (e.g., `messages:chat_id`).
- **Storage**: 
  - Store messages in **Cassandra** {`chat_id`, `message_id`, `content`, `timestamp`, `sender_id`, `is_read`} for scalable writes.
  - Partition by `chat_id`, cluster by `timestamp` for efficient retrieval.
- **Delivery**: 
  - **WebSockets** for real-time delivery to online users.
  - **APNs/GCM** for push notifications to offline users.
  - Fanout messages to group chat members via Kafka consumers.
- **Edge Cases**: 
  - **Offline Users**: Store messages in Cassandra, deliver on reconnect.
  - **Duplicates**: Use idempotency keys (`message_id`) in Kafka/Cassandra.
  - **Group Chats**: Fanout messages to all members, optimize for large groups with batch writes.
  - **Message Expiration**: Set **TTL policies** in Cassandra (e.g., 30 days for ephemeral messages).
  - **Delivery Failures**: Implement retry logic with exponential backoff, store failed deliveries in Kafka dead-letter queue.
- **Monitoring**: Track message delivery latency, notification success rate, Kafka queue backlog, Cassandra write throughput, WebSocket connection stability.

## Network Protocols & APIs
- **Protocols**: 
  - **HTTPS**: Secure API for message sending and chat management.
  - **WebSocket**: Real-time message delivery and read receipts.
  - **gRPC**: High-performance internal communication for message processing.
  - **TCP**: Reliable Kafka/Cassandra connections.
- **APIs**:
  - `POST /messages`: Send message to a chat.
    ```json
    {
      "chat_id": "uuid",
      "message_id": "uuid",
      "sender_id": "uuid",
      "content": "Hello!",
      "timestamp": 1697056800000
    }
    ```
  - `GET /chats/:chat_id/messages`: Retrieve message history.
    ```json
    {
      "chat_id": "uuid",
      "limit": 50,
      "cursor": "timestamp",
      "messages": [{"message_id": "uuid", "content": "Hello!", ...}]
    }
    ```

## Trade-offs
- **Kafka vs. RabbitMQ**: 
  - **Kafka**: Durable, scalable, high-throughput. 
  - **RabbitMQ**: Simpler, less robust. 
  - *Choice*: **Kafka** for message volume.
- **WebSockets vs. Polling**: 
  - **WebSockets**: Real-time, complex. 
  - **Polling**: Simpler, delayed. 
  - *Choice*: **WebSockets** for UX.
- **Cassandra vs. DynamoDB**: 
  - **Cassandra**: Cost-effective, scalable writes. 
  - **DynamoDB**: Simpler, costlier. 
  - *Choice*: **Cassandra** for scalability.

## Why Kafka + Cassandra?
- **Kafka**: Reliable, partitioned message ingestion.
- **Cassandra**: Scalable, high-throughput message storage.

## Scalability Estimates
- **Users**: ~100M DAU via Kafka partitioning, Cassandra sharding.
- **Messages**: ~1B/day (~12K messages/sec) with Kafka/Cassandra.
- **Delivery**: ~50K QPS for WebSocket/APNs.

## Latency/Throughput Targets
- **Message Delivery**: <100ms (WebSocket).
- **Notifications**: <200ms (APNs/GCM).
- **Ingestion**: ~10K messages/sec.

## Database Schemas
- **Cassandra Messages Table**:
  ```sql
  CREATE TABLE messages (
    chat_id UUID,
    message_id UUID,
    sender_id UUID,
    content TEXT,
    timestamp TIMESTAMP,
    is_read BOOLEAN,
    PRIMARY KEY (chat_id, timestamp, message_id)
  ) WITH CLUSTERING ORDER BY (timestamp DESC);
  ```
  - **Consistency**: Eventual for messages, quorum writes for durability.
  - **Availability**: Multi-region replication.
- **PostgreSQL Chats Table** (for metadata):
  ```sql
  CREATE TABLE chats (
    chat_id UUID PRIMARY KEY,
    type TEXT, -- '1:1', 'group'
    members JSONB,
    created_at TIMESTAMP
  );
  ```
  - **Consistency**: Strong for chat metadata (ACID).
  - **Availability**: Multi-AZ.

## Cache Schemas
- **Redis**:
  - `chat:chat_id:messages`: Recent messages (list, TTL: 1hr).
  - `user:user_id:online`: Online status (flag, TTL: 5min).
  - **Consistency**: Eventual, sync with Cassandra.
  - **TTL**: Short for transient data.
- **Tricks**: Redis pipelining for batch fanout, LRU for memory management.

## Consistency vs. Availability
- **Cassandra**: 
  - **Eventual Consistency**: For message storage (quorum reads/writes).
  - **High Availability**: Multi-region.
- **PostgreSQL**: 
  - **Strong Consistency**: For chat metadata (transactions).
  - **High Availability**: Multi-AZ, read replicas.
- **Redis**: 
  - **Eventual Consistency**: For cached messages, online status.
  - **High Availability**: Redis Cluster.
- **Tricks**: Cassandra lightweight transactions for critical updates, Redis Lua scripts for atomic fanout.

## Additional Considerations
- **Security**: 
  - Encrypt messages in transit (HTTPS/WebSocket) and at rest (**Cassandra encryption**).
  - Use **JWT auth** for API/WebSocket access.
- **Analytics**: Log message events in **Kafka**, aggregate in **Redshift** for usage patterns.
- **Notifications**: Optimize APNs/GCM with priority flags for urgent messages.
- **Rate Limiting**: Cap message sends per user (Redis token bucket).
- **Backup**: Archive old messages to **S3/Glacier**.

## Correctness Check
- **Original Design**: All components (Kafka, Cassandra, WebSockets, APNs/GCM) align with scalability/latency goals. No incorrect details.
- **Missing Details**: Added APIs, protocols, analytics, security, notifications, and detailed schemas for completeness.

## Summary
Facebook Messenger uses **Kafka** for message ingestion, **Cassandra** for storage, **WebSockets** for real-time delivery, and **APNs/GCM** for notifications. It scales to 100M DAU, 1B messages/day, 50K QPS with <100ms delivery, <200ms notifications. Handles offline users, duplicates, and group chats with robust edge case management. **HTTPS/gRPC/WebSocket** ensure secure, fast communication, making it FAANG-ready.

---
---

# Local Delivery (Gopuff) System Design

## Functional Requirements
- **Order Placement**: Customers place orders with items, delivery address, and payment.
- **Inventory Management**: Track real-time inventory across distribution centers (DCs).
- **Delivery Routing**: Optimize routes for drivers to meet 1-hour delivery.
- **Availability Lookup**: Check item availability considering traffic and geography.
- **Order Tracking**: Provide real-time order status and ETA.
- **Cancellation/Modification**: Allow order cancellations or changes pre-delivery.

## Non-Functional Requirements
- **Scalability**: Handle ~10K orders/sec, ~1M daily orders, ~100K QPS for inventory checks.
- **Latency**: <100ms for inventory checks, <500ms for routing, ~5K orders/sec processing.
- **Availability**: 99.99% uptime, no single points of failure.
- **Consistency**: Strong for order placement/inventory updates, eventual for ETAs.
- **Security**: Secure payment processing, protect user data.

## Write-Heavy Considerations
- **High Throughput**: Process ~10K orders/sec during peak demand.
- **Durability**: Persist orders and inventory updates reliably.
- **Ingestion Speed**: Fast order and inventory transactions with atomicity.

## Improvements
- **Order Ingestion**: 
  - Use **Kafka** for high-throughput order events, partitioned by `region` for load balancing.
  - Topics: `orders:region`, `inventory_updates:dc_id`.
- **Inventory**: 
  - Store real-time inventory in **Redis** (`inventory:dc_id:item_id`, quantity) for fast updates.
  - Persist in **PostgreSQL** for durability {`dc_id`, `item_id`, `quantity`, `last_updated`}.
- **Routing**: 
  - Use **OSRM** (Open Source Routing Machine) for traffic-aware route optimization.
  - Cache routes in **Redis** (`route:driver_id:order_id`, TTL: 10min) for reuse.
- **Traffic-Aware Availability Lookup**: 
  - **Nearby Service**: Syncs DC locations (`dc_id`, `lat`, `lon`) into memory periodically.
  - Filter DCs within 60-mile radius, query **Travel Time Service** for real-world drive times (<1hr).
  - Check inventory only for viable DCs, cache results in Redis (`availability:item_id:dc_id`).
- **Placing Orders (Strong Consistency)**: 
  - **Orders Service**: Uses **SERIALIZABLE PostgreSQL transactions** to check inventory, record order, and update stock atomically.
  - Colocate inventory/orders in PostgreSQL for strong consistency.
  - Fail transaction on out-of-stock items, return clear error.
- **Fast & Scalable Availability Lookup**: 
  - Handle ~20K availability queries/sec by caching in **Redis**.
  - Cache hit: Return result; miss: Query PostgreSQL, update cache.
- **Edge Cases**: 
  - **Out-of-Stock**: Real-time Redis/PostgreSQL checks during order placement.
  - **Delivery Delays**: Recompute routes via OSRM on traffic disruptions.
  - **Order Cancellations**: Rollback inventory in PostgreSQL transaction.
  - **Driver Unavailability**: Reassign orders via Kafka event to available drivers.
  - **Traffic Disruptions**: Update ETAs using Travel Time Service.
- **Monitoring**: Track order latency, inventory accuracy, delivery ETA accuracy, order success rate, Redis cache hit ratio, PostgreSQL transaction throughput.

## Network Protocols & APIs
- **Protocols**: 
  - **HTTPS**: Secure API for order placement and tracking.
  - **gRPC**: Fast internal communication for inventory/routing.
  - **WebSocket**: Real-time order status updates (ETA, delivery).
  - **TCP**: Reliable Kafka/PostgreSQL connections.
- **APIs**:
  - `POST /orders`: Place order with items and delivery details.
    ```json
    {
      "user_id": "uuid",
      "items": [{"item_id": "uuid", "quantity": 2}],
      "address": {"lat": 37.7749, "lon": -122.4194},
      "payment": {...}
    }
    ```
  - `GET /availability`: Check item availability by location.
    ```json
    {
      "item_id": "uuid",
      "lat": 37.7749,
      "lon": -122.4194,
      "dcs": [{"dc_id": "uuid", "available": true}]
    }
    ```
  - `GET /orders/:order_id/status`: Track order status and ETA.

## Trade-offs
- **Redis vs. DB for Inventory**: 
  - **Redis**: Fast, volatile. 
  - **DB**: Durable, slower. 
  - *Choice*: **Redis** with PostgreSQL backup.
- **Real-time vs. Batch Routing**: 
  - **Real-time**: Accurate, compute-heavy. 
  - **Batch**: Efficient, stale. 
  - *Choice*: **Real-time** for UX.
- **Kafka vs. RabbitMQ**: 
  - **Kafka**: Scalable, high-throughput. 
  - **RabbitMQ**: Simpler, less robust. 
  - *Choice*: **Kafka** for order volume.

## Why Kafka + Redis?
- **Kafka**: Reliable, partitioned order ingestion.
- **Redis**: Fast inventory checks and route caching.

## Scalability Estimates
- **Orders**: ~10K orders/sec via Kafka partitioning.
- **Daily Orders**: ~1M with PostgreSQL sharding.
- **Inventory Checks**: ~100K QPS with Redis caching.

## Latency/Throughput Targets
- **Inventory Checks**: <100ms (Redis).
- **Routing**: <500ms (OSRM).
- **Processing**: ~5K orders/sec.

## Database Schemas
- **PostgreSQL Inventory Table**:
  ```sql
  CREATE TABLE inventory (
    dc_id UUID,
    item_id UUID,
    quantity INT,
    last_updated TIMESTAMP,
    PRIMARY KEY (dc_id, item_id)
  );
  ```
  - **Consistency**: Strong for updates (SERIALIZABLE transactions).
  - **Availability**: Multi-AZ replication.
- **PostgreSQL Orders Table**:
  ```sql
  CREATE TABLE orders (
    order_id UUID PRIMARY KEY,
    user_id UUID,
    dc_id UUID,
    items JSONB,
    address JSONB,
    status TEXT,
    created_at TIMESTAMP
  );
  ```
  - **Consistency**: Strong for order placement.
- **PostgreSQL DCs Table**:
  ```sql
  CREATE TABLE dcs (
    dc_id UUID PRIMARY KEY,
    lat DOUBLE PRECISION,
    lon DOUBLE PRECISION,
    address TEXT
  );
  ```
  - **Consistency**: Strong for DC metadata.

## Cache Schemas
- **Redis**:
  - `inventory:dc_id:item_id`: Quantity (integer).
  - `route:driver_id:order_id`: Route data (JSON, TTL: 10min).
  - `availability:item_id:dc_id`: Availability status (boolean, TTL: 5min).
  - **Consistency**: Eventual, sync with PostgreSQL.
- **Tricks**: Redis pipelining for batch inventory checks, LRU for cache eviction.

## Consistency vs. Availability
- **PostgreSQL**: 
  - **Strong Consistency**: For orders/inventory (SERIALIZABLE transactions).
  - **High Availability**: Multi-AZ, read replicas.
- **Redis**: 
  - **Eventual Consistency**: For inventory/routes/availability.
  - **High Availability**: Redis Cluster.
- **Kafka**: 
  - **Strong Consistency**: For order event durability.
  - **High Availability**: Multi-broker clusters.
- **Tricks**: PostgreSQL advisory locks for inventory updates, Redis Lua scripts for atomic checks.

## Additional Considerations
- **Security**: 
  - Encrypt payment data with **PCI-compliant gateway**.
  - Use **JWT auth** for API access.
- **Analytics**: Log orders in **Kafka**, aggregate in **Redshift** for demand forecasting.
- **Notifications**: Real-time ETA updates via **WebSocket**, alerts via **SNS**.
- **Rate Limiting**: Cap order placements per user (Redis token bucket).
- **Backup**: PostgreSQL snapshots to **S3/Glacier**.

## Correctness Check
- **Original Design**: All components (Kafka, Redis, PostgreSQL, OSRM) align with scalability/latency goals. No incorrect details.
- **Missing Details**: Added APIs, protocols, analytics, notifications, security, and detailed schemas for completeness.

## Summary
Gopuff uses **Kafka** for order ingestion, **Redis** for inventory/routing, **PostgreSQL** for durable storage, and **OSRM** for traffic-aware routing. It ensures 1-hour delivery with **Travel Time Service** and scales to 10K orders/sec, 1M daily orders, 100K QPS with <100ms inventory checks, <500ms routing. Strong consistency for orders, robust edge case handling, and **HTTPS/gRPC/WebSocket** make it FAANG-ready.


---
---


# Strava System Design

## Functional Requirements
- **Activity Tracking**: Record GPS-based activities (runs, rides) with metrics (distance, pace, elevation).
- **Activity Upload**: Upload activities (bulk or real-time) from devices.
- **Real-time Sharing**: Allow friends to follow live activities.
- **Leaderboards**: Display real-time rankings by distance, region, or time range.
- **Analytics**: Provide stats (e.g., weekly miles, personal records).
- **Social Features**: Share activities, comment, and follow users.

## Non-Functional Requirements
- **Scalability**: Handle ~1M daily active users (DAU), ~10M activities/day, ~10K QPS for analytics.
- **Latency**: <200ms for activity ingestion, <500ms for analytics, ~5K activities/sec processing.
- **Availability**: 99.99% uptime, no single points of failure.
- **Consistency**: Eventual for activity data/leaderboards, strong for user metadata.
- **Security**: Protect GPS data, enforce privacy settings.

## Write-Heavy Considerations
- **High Throughput**: Ingest ~10M activities/day (~120 activities/sec peak).
- **Durability**: Persist activity data reliably for history and analytics.
- **Ingestion Speed**: Fast processing of GPS streams and bulk uploads.

## Improvements
- **Activity Ingestion**: 
  - Use **Kafka** for GPS/fitness data streams, partitioned by `user_id` for scalability.
  - Topics: `activities:user_id`, `live_updates:user_id`.
- **Storage**: 
  - Store activities in **Cassandra** {`activity_id`, `user_id`, `gps_points`, `timestamp`, `metrics`, `privacy`} for high write throughput.
  - Partition by `user_id`, cluster by `timestamp`.
- **Analytics**: 
  - Use **Spark** for batch analytics (leaderboards, stats).
  - Cache results in **Redis** (`stats:user_id`, `leaderboard:region`).
- **Offline Activity Tracking**: 
  - Record GPS data locally using device sensors, store in memory.
  - Persist to local storage (e.g., **Core Data/Room DB**) every ~10s to prevent loss.
  - Upload in bulk (optionally chunked) on completion or when online.
- **Real-time Activity Sharing**: 
  - Send location updates every 2-5s from athlete’s device to server.
  - Use **polling** (clients request updates every 2-5s with offset) instead of WebSockets for simplicity.
  - **Smart Buffering**: Delay updates by a few seconds for smooth movement, compensating for network latency.
- **Real-time Leaderboard**: 
  - Use **Redis Sorted Sets** (`leaderboard:run:USA`, member: `user_id`, score: distance).
  - Store activity timestamps in sorted sets, activity data in hashes.
  - Query with **ZRANGEBYSCORE** for time ranges, aggregate distances, and sort.
  - Cache results in Redis (`leaderboard:run:USA:1w`, TTL: 10min).
- **Edge Cases**: 
  - **GPS Errors**: Filter outliers (e.g., impossible speeds) during ingestion.
  - **Large Uploads**: Process chunked uploads via Kafka.
  - **Privacy Settings**: Filter activities on read based on `privacy` field.
  - **Activity Duplicates**: Use idempotency keys (`activity_id`) in Kafka/Cassandra.
  - **Incomplete Data**: Validate inputs (e.g., GPS format, metrics).
- **Monitoring**: Track ingestion latency, analytics accuracy, cache hit ratio, activity processing success rate, Redis memory usage, Spark job latency.

## Network Protocols & APIs
- **Protocols**: 
  - **HTTPS**: Secure API for activity uploads and analytics.
  - **gRPC**: Fast internal communication for data processing.
  - **TCP**: Reliable Kafka/Cassandra connections.
- **APIs**:
  - `POST /activities`: Upload activity (bulk or chunked).
    ```json
    {
      "user_id": "uuid",
      "activity_id": "uuid",
      "gps_points": [{"lat": 37.7749, "lon": -122.4194, "time": 1697056800}, ...],
      "metrics": {"distance": 10.5, "pace": 6.5},
      "privacy": "public"
    }
    ```
  - `GET /leaderboard/:type/:region/:time_range`: Retrieve leaderboard.
    ```json
    {
      "type": "run",
      "region": "USA",
      "time_range": "1w",
      "leaders": [{"user_id": "uuid", "distance": 50.2}, ...]
    }
    ```
  - `GET /live/:user_id`: Poll for real-time activity updates.

## Trade-offs
- **Cassandra vs. DynamoDB**: 
  - **Cassandra**: Cost-effective, scalable writes. 
  - **DynamoDB**: Simpler, costlier. 
  - *Choice*: **Cassandra** for scalability.
- **Real-time vs. Batch Analytics**: 
  - **Real-time**: Fresh, resource-heavy. 
  - **Batch**: Efficient, stale. 
  - *Choice*: **Batch** with periodic Redis updates.
- **Kafka vs. RabbitMQ**: 
  - **Kafka**: Durable, high-throughput. 
  - **RabbitMQ**: Simpler, less robust. 
  - *Choice*: **Kafka** for stream volume.

## Why Kafka + Cassandra?
- **Kafka**: Reliable, partitioned ingestion for GPS streams.
- **Cassandra**: Scalable storage for high-write activity data.

## Scalability Estimates
- **Users**: ~1M DAU via Kafka partitioning, Cassandra sharding.
- **Activities**: ~10M/day (~120 activities/sec) with Kafka/Cassandra.
- **Analytics**: ~10K QPS for leaderboard/stats queries.

## Latency/Throughput Targets
- **Ingestion**: <200ms (Kafka).
- **Analytics**: <500ms (Redis/Spark).
- **Processing**: ~5K activities/sec.

## Database Schemas
- **Cassandra Activities Table**:
  ```sql
  CREATE TABLE activities (
    user_id UUID,
    activity_id UUID,
    gps_points LIST<FROZEN<MAP<TEXT, DOUBLE>>>,
    timestamp TIMESTAMP,
    metrics MAP<TEXT, DOUBLE>,
    privacy TEXT,
    PRIMARY KEY (user_id, timestamp, activity_id)
  ) WITH CLUSTERING ORDER BY (timestamp DESC);
  ```
  - **Consistency**: Eventual for activity data, quorum writes.
  - **Availability**: Multi-region replication.
- **PostgreSQL Users Table** (for metadata):
  ```sql
  CREATE TABLE users (
    user_id UUID PRIMARY KEY,
    username TEXT,
    preferences JSONB,
    created_at TIMESTAMP
  );
  ```
  - **Consistency**: Strong for user metadata (ACID).
  - **Availability**: Multi-AZ.

## Cache Schemas
- **Redis**:
  - `stats:user_id`: Activity stats (JSON, TTL: 1hr).
  - `leaderboard:type:region`: Sorted set {`user_id`, `distance`}.
  - `live:user_id`: Recent location updates (list, TTL: 10s).
  - **Consistency**: Eventual, sync with Cassandra/Spark.
- **Tricks**: Redis pipelining for batch leaderboard queries, LRU for memory management.

## Consistency vs. Availability
- **Cassandra**: 
  - **Eventual Consistency**: For activity data (quorum reads/writes).
  - **High Availability**: Multi-region.
- **PostgreSQL**: 
  - **Strong Consistency**: For user metadata (transactions).
  - **High Availability**: Multi-AZ, read replicas.
- **Redis**: 
  - **Eventual Consistency**: For stats/leaderboards.
  - **High Availability**: Redis Cluster.
- **Tricks**: Cassandra lightweight transactions for critical updates, Redis Lua scripts for leaderboard updates.

## Additional Considerations
- **Security**: 
  - Encrypt GPS data in transit (HTTPS) and at rest (**Cassandra encryption**).
  - Use **JWT auth** for API access.
- **Analytics**: Log activities in **Kafka**, aggregate in **Redshift** for trends.
- **Notifications**: Push updates for comments/follows via **SNS**.
- **Rate Limiting**: Cap activity uploads per user (Redis token bucket).
- **Backup**: Archive activities to **S3/Glacier**.

## Correctness Check
- **Original Design**: All components (Kafka, Cassandra, Spark, Redis) align with scalability/latency goals. No incorrect details.
- **Missing Details**: Added APIs, protocols, analytics, notifications, security, and detailed schemas for completeness.

## Summary
Strava uses **Kafka** for activity ingestion, **Cassandra** for storage, **Redis** for real-time leaderboards, and **Spark** for analytics. It supports offline tracking, real-time sharing via **polling**, and scales to 1M DAU, 10M activities/day, 10K QPS with <200ms ingestion, <500ms analytics. Handles GPS errors, privacy, and duplicates with **HTTPS/gRPC** for secure, fast communication, ensuring FAANG-readiness.


---
---


# Facebook Live Comments System Design

## Functional Requirements
- **Comment Submission**: Users post comments on live videos via POST request.
- **Real-time Delivery**: Broadcast comments to all viewers in real-time.
- **Historical Comments**: Fetch comments made before joining via cursor-based pagination.
- **Comment Management**: Delete/moderate comments, handle spam.
- **Viewer Scaling**: Support millions of concurrent viewers per live video.
- **Analytics**: Track comment engagement for insights.

## Non-Functional Requirements
- **Scalability**: Handle ~100K comments/sec, ~1M viewers/video, ~50K QPS for delivery.
- **Latency**: <100ms for comment delivery, <200ms for ingestion, ~10K comments/sec processing.
- **Availability**: 99.99% uptime, no single points of failure.
- **Consistency**: Strong for comment storage to prevent duplicates, eventual for delivery.
- **Security**: Prevent spam, ensure authenticated access.

## Strong Consistency Considerations
- **Transactional Integrity**: Ensure comments are stored atomically to avoid duplicates or loss.
- **Failure Handling**: Handle server failures, network issues, or viewer disconnections gracefully.

## Improvements
- **Comment Ingestion**: 
  - Use **Kafka** for high-throughput comment streams, partitioned by `video_id` for load balancing.
  - Producers send comments to topics (e.g., `comments:video_id`).
- **Storage**: 
  - Store comments in **DynamoDB** {`video_id`, `comment_id`, `user_id`, `content`, `timestamp`, `is_deleted`} for scalability and speed.
  - Validate POST requests to `/comments/:liveVideoId` at the server, store in DynamoDB.
  - Alternative: **Cassandra** for cost-effective high-write throughput, partitioned by `video_id`.
- **Historical Comments**: 
  - Support **infinite scrolling** with **cursor-based pagination** via GET `/comments/:liveVideoId`.
  - Use DynamoDB’s **LastEvaluatedKey** to fetch N recent comments before a timestamp, ensuring stable, efficient scrolling.
- **Real-time Comment Broadcasting**: 
  - Use **Server-Sent Events (SSE)** over WebSockets for lightweight, one-way delivery.
  - SSE scales better for broadcasting comments to multiple clients, simpler to implement.
  - **Scaling SSE**: 
    - Horizontally scale servers to handle millions of viewers.
    - Use **Redis Pub/Sub** for low-latency comment broadcasting across servers.
    - Alternative: **Kafka** for guaranteed delivery but higher latency.
- **Viewer Allocation**: 
  - Route viewers of the same `video_id` to the same server using **consistent hashing** in a **Layer 7 load balancer** (e.g., NGINX/Envoy).
  - Alternative: **ZooKeeper** for dynamic `video_id`-to-server mappings, offering flexibility at higher complexity.
- **Delivery**: 
  - Use **WebSockets** as fallback for real-time delivery if SSE is insufficient.
  - Fanout comments via Redis Pub/Sub or Kafka to all viewer servers.
- **Edge Cases**: 
  - **Comment Spam**: Apply **rate-limiting** (Redis token bucket) and CAPTCHA for suspicious users.
  - **Deleted Comments**: Mark as tombstones in DynamoDB/Cassandra (`is_deleted: true`).
  - **High QPS**: Scale WebSocket/SSE servers and Redis Pub/Sub.
  - **Comment Ordering**: Use sequence numbers (`comment_id` with timestamp) for consistent display.
  - **Viewer Lag**: Buffer comments in Redis (`buffer:video_id`, TTL: 10s) to smooth delivery.
- **Monitoring**: Track comment delivery latency, notification success rate, Kafka queue backlog, DynamoDB write throughput, SSE connection stability, Redis Pub/Sub latency.

## Network Protocols & APIs
- **Protocols**: 
  - **HTTPS**: Secure API for comment submission and retrieval.
  - **SSE**: Lightweight, real-time comment broadcasting.
  - **WebSocket**: Fallback for real-time delivery.
  - **gRPC**: Fast internal communication for comment processing.
  - **TCP**: Reliable Kafka/DynamoDB connections.
- **APIs**:
  - `POST /comments/:liveVideoId`: Submit comment on live video.
    ```json
    {
      "user_id": "uuid",
      "comment_id": "uuid",
      "content": "Great stream!",
      "timestamp": 1697056800000
    }
    ```
  - `GET /comments/:liveVideoId`: Fetch comments with cursor pagination.
    ```json
    {
      "video_id": "uuid",
      "limit": 50,
      "cursor": "1697056800000",
      "comments": [{"comment_id": "uuid", "content": "Great stream!", ...}]
    }
    ```

## Trade-offs
- **Kafka vs. RabbitMQ**: 
  - **Kafka**: Scalable, durable, high-throughput. 
  - **RabbitMQ**: Simpler, less robust. 
  - *Choice*: **Kafka** for comment volume.
- **WebSockets vs. Polling**: 
  - **WebSockets**: Real-time, complex. 
  - **Polling**: Simpler, delayed. 
  - *Choice*: **SSE** for lightweight broadcasting, WebSockets as fallback.
- **Cassandra vs. DynamoDB**: 
  - **Cassandra**: Cost-effective, scalable writes. 
  - **DynamoDB**: Simpler, costlier. 
  - *Choice*: **DynamoDB** for simplicity, Cassandra for cost.

## Why Kafka + DynamoDB/SSE?
- **Kafka**: Reliable, partitioned comment ingestion.
- **DynamoDB**: Fast, scalable storage with strong consistency.
- **SSE**: Lightweight, efficient real-time broadcasting.

## Scalability Estimates
- **Comments**: ~100K comments/sec via Kafka partitioning.
- **Viewers**: ~1M per video with SSE/WebSocket scaling.
- **Delivery**: ~50K QPS for comment broadcasting.

## Latency/Throughput Targets
- **Comment Delivery**: <100ms (SSE/WebSocket).
- **Ingestion**: <200ms (Kafka/DynamoDB).
- **Processing**: ~10K comments/sec.

## Database Schemas
- **DynamoDB Comments Table**:
  ```sql
  Table: comments
  Partition Key: video_id (String)
  Sort Key: timestamp (Number)
  Attributes:
    comment_id (String)
    user_id (String)
    content (String)
    is_deleted (Boolean)
  ```
  - **Consistency**: Strong for writes (conditional updates).
  - **Availability**: Multi-region.
- **Cassandra Comments Table** (alternative):
  ```sql
  CREATE TABLE comments (
    video_id UUID,
    timestamp TIMESTAMP,
    comment_id UUID,
    user_id UUID,
    content TEXT,
    is_deleted BOOLEAN,
    PRIMARY KEY (video_id, timestamp, comment_id)
  ) WITH CLUSTERING ORDER BY (timestamp DESC);
  ```
  - **Consistency**: Eventual, quorum writes.
- **PostgreSQL Videos Table** (for metadata):
  ```sql
  CREATE TABLE videos (
    video_id UUID PRIMARY KEY,
    user_id UUID,
    title TEXT,
    created_at TIMESTAMP
  );
  ```
  - **Consistency**: Strong for video metadata (ACID).

## Cache Schemas
- **Redis**:
  - `comments:video_id`: Recent comments (list, TTL: 10min).
  - `buffer:video_id`: Buffered comments for lag (list, TTL: 10s).
  - **Consistency**: Eventual, sync with DynamoDB/Cassandra.
- **Tricks**: Redis pipelining for batch comment fetches, LRU for memory management.

## Consistency vs. Availability
- **DynamoDB/Cassandra**: 
  - **Strong Consistency**: For comment writes (conditional writes/quorum).
  - **High Availability**: Multi-region.
- **PostgreSQL**: 
  - **Strong Consistency**: For video metadata (transactions).
  - **High Availability**: Multi-AZ, read replicas.
- **Redis**: 
  - **Eventual Consistency**: For cached comments/buffers.
  - **High Availability**: Redis Cluster.
- **Tricks**: DynamoDB conditional writes for idempotency, Cassandra lightweight transactions for critical updates.

## Additional Considerations
- **Security**: 
  - Encrypt comments in transit (HTTPS/SSE) and at rest (**DynamoDB encryption**).
  - Use **JWT auth** for API/SSE access.
- **Analytics**: Log comments in **Kafka**, aggregate in **Redshift** for engagement insights.
- **Notifications**: Alert on spam detection via **SNS**.
- **Rate Limiting**: Cap comment submissions per user (Redis token bucket).
- **Backup**: Archive comments to **S3/Glacier**.

## Correctness Check
- **Original Design**: All components (Kafka, DynamoDB, SSE, WebSockets, Redis) align with scalability/latency goals. No incorrect details.
- **Missing Details**: Added APIs, protocols, analytics, notifications, security, alternative Cassandra storage, and detailed schemas for completeness.

## Summary
Facebook Live Comments uses **Kafka** for comment ingestion, **DynamoDB/Cassandra** for storage, **SSE** (with WebSocket fallback) for real-time broadcasting, and **Redis Pub/Sub** for viewer scaling. It supports 100K comments/sec, 1M viewers/video, 50K QPS with <100ms delivery, <200ms ingestion. Strong consistency for writes, cursor pagination for history, and robust edge case handling (spam, lag) with **HTTPS/gRPC/SSE** ensure FAANG-readiness.

---
---


# Online Ticket Booking System Design

## Functional Requirements
- **Ticket Reservation**: Reserve seats temporarily during booking.
- **Ticket Purchase**: Process payments and confirm bookings.
- **Event Browsing**: View event details, seat availability, and prices.
- **Virtual Waiting Queue**: Manage high-demand events with millions of concurrent users.
- **Order History**: Track user bookings and ticket status.
- **Cancellation/Refunding**: Handle cancellations and refund requests.

## Non-Functional Requirements
- **Scalability**: Handle ~10K bookings/sec, ~1M daily tickets, ~50K QPS for reservations.
- **Latency**: <200ms for seat reservation, <500ms for booking, ~5K bookings/sec processing.
- **Availability**: 99.99% uptime, no single points of failure.
- **Consistency**: Strong for booking and inventory to prevent overselling, eventual for event details.
- **Security**: Secure payment processing, prevent fraudulent bookings.

## Strong Consistency Considerations
- **Transactional Integrity**: Ensure atomic seat reservations and payment confirmations.
- **Failure Handling**: Manage payment failures, expired reservations, and double bookings.

## Improvements
- **Concurrency**: 
  - Implement **distributed locking** with **Redis** for seat reservations, using a 10-minute **TTL**.
  - On ticket selection, **Booking Service** acquires lock (`lock:ticket_id`), creates an "in-progress" booking in **PostgreSQL**, and redirects to payment with `bookingId`.
  - On payment completion, a webhook triggers a **PostgreSQL transaction** to update ticket status to "sold" and booking to "confirmed".
  - If TTL expires or user abandons, lock auto-releases, making ticket available.
- **High Concurrent View Requests**: 
  - Cache static event/venue data in **Redis/Memcached** using read-through strategy with TTL (e.g., 1hr).
  - Use **database triggers** to invalidate cache on updates.
  - Distribute traffic via **Layer 7 load balancer** (e.g., NGINX) across stateless API instances for horizontal scaling.
- **Virtual Waiting Queue**: 
  - For high-demand events, place users in a **queue** managed by **Redis** (`queue:event_id`, sorted set).
  - Establish **WebSocket** connection for real-time queue position updates and estimated wait times.
  - Dequeue users based on schedule, update PostgreSQL to grant access, and notify via WebSocket.
- **Reservation Flow**: 
  - Reserve seats in Redis (`reservation:event_id:ticket_id`, TTL: 10min).
  - Process payment via external processor.
  - Confirm booking in PostgreSQL with strong consistency.
- **Edge Cases**: 
  - **Payment Failures**: Rollback reservation in PostgreSQL, release Redis lock.
  - **Expired Reservations**: Auto-release seats via Redis TTL.
  - **Oversold Tickets**: Maintain audit logs in PostgreSQL for disputes.
  - **Double Bookings**: Use idempotency keys (`bookingId`) for payment requests.
  - **High Demand**: Queue requests in Redis/Kafka to prevent overload.
- **Monitoring**: Track booking success rate, lock contention, payment latency, seat reservation accuracy, queue wait times, Redis cache hit ratio.

## Network Protocols & APIs
- **Protocols**: 
  - **HTTPS**: Secure API for booking and event browsing.
  - **WebSocket**: Real-time queue updates and booking status.
  - **gRPC**: Fast internal communication for reservation/payment processing.
  - **TCP**: Reliable Redis/PostgreSQL connections.
- **APIs**:
  - `POST /reservations`: Reserve seats for an event.
    ```json
    {
      "user_id": "uuid",
      "event_id": "uuid",
      "ticket_ids": ["uuid1", "uuid2"],
      "bookingId": "uuid"
    }
    ```
  - `POST /bookings/:bookingId/confirm`: Confirm booking post-payment.
    ```json
    {
      "bookingId": "uuid",
      "payment_id": "uuid",
      "status": "confirmed"
    }
    ```
  - `GET /events/:event_id`: Fetch event details and seat availability.

## Trade-offs
- **Optimistic vs. Pessimistic Locking**: 
  - **Optimistic**: Scalable, risks conflicts. 
  - **Pessimistic**: Safe, slower. 
  - *Choice*: **Optimistic** for most cases, pessimistic (Redis locks) for high-demand events.
- **Redis vs. DB for Reservation**: 
  - **Redis**: Fast, volatile. 
  - **DB**: Durable, slower. 
  - *Choice*: **Redis** with PostgreSQL confirmation.
- **Queue vs. No Queue**: 
  - **Queuing**: Handles spikes, adds latency. 
  - **No Queuing**: Fast, risks failures. 
  - *Choice*: **Queuing** for high-contention events.

## Why Optimistic + Redis/DB?
- **Optimistic Locking**: Scales for typical loads, minimizes contention.
- **Redis/DB**: Redis for fast reservations, PostgreSQL for durability.

## Scalability Estimates
- **Bookings**: ~10K bookings/sec via Redis locking, Kafka buffering.
- **Daily Tickets**: ~1M with PostgreSQL sharding.
- **Reservations**: ~50K QPS with Redis caching.

## Latency/Throughput Targets
- **Seat Reservation**: <200ms (Redis).
- **Booking**: <500ms (PostgreSQL).
- **Processing**: ~5K bookings/sec.

## Database Schemas
- **PostgreSQL Tickets Table**:
  ```sql
  CREATE TABLE tickets (
    ticket_id UUID PRIMARY KEY,
    event_id UUID,
    seat_number TEXT,
    status TEXT, -- 'available', 'sold'
    booking_id UUID,
    updated_at TIMESTAMP
  );
  ```
  - **Consistency**: Strong for ticket updates (SERIALIZABLE transactions).
  - **Availability**: Multi-AZ replication.
- **PostgreSQL Bookings Table**:
  ```sql
  CREATE TABLE bookings (
    booking_id UUID PRIMARY KEY,
    user_id UUID,
    event_id UUID,
    ticket_ids JSONB,
    status TEXT, -- 'in-progress', 'confirmed', 'failed'
    created_at TIMESTAMP
  );
  ```
  - **Consistency**: Strong for booking status.
- **PostgreSQL Events Table**:
  ```sql
  CREATE TABLE events (
    event_id UUID PRIMARY KEY,
    name TEXT,
    venue_id UUID,
    date TIMESTAMP,
    details JSONB
  );
  ```
  - **Consistency**: Eventual for details, strong for inventory.

## Cache Schemas
- **Redis**:
  - `reservation:event_id:ticket_id`: Reservation status (flag, TTL: 10min).
  - `event:event_id`: Event details (JSON, TTL: 1hr).
  - `queue:event_id`: User queue positions (sorted set, TTL: event duration).
  - **Consistency**: Eventual, sync with PostgreSQL.
- **Tricks**: Redis pipelining for batch reservations, LRU for cache eviction.

## Consistency vs. Availability
- **PostgreSQL**: 
  - **Strong Consistency**: For bookings/tickets (SERIALIZABLE transactions).
  - **High Availability**: Multi-AZ, read replicas.
- **Redis**: 
  - **Eventual Consistency**: For reservations/event data.
  - **High Availability**: Redis Cluster.
- **Kafka**: 
  - **Strong Consistency**: For event durability.
  - **High Availability**: Multi-broker clusters.
- **Tricks**: PostgreSQL advisory locks for ticket updates, Redis Lua scripts for atomic reservations.

## Additional Considerations
- **Security**: 
  - Use **PCI-compliant payment gateway** for transactions.
  - Implement **JWT auth** for API/WebSocket access.
- **Analytics**: Log bookings in **Kafka**, aggregate in **Redshift** for demand analysis.
- **Notifications**: Real-time queue/booking updates via **WebSocket**, alerts via **SNS**.
- **Rate Limiting**: Cap booking attempts per user (Redis token bucket).
- **Backup**: PostgreSQL snapshots to **S3/Glacier**.

## Correctness Check
- **Original Design**: All components (Redis, PostgreSQL, Kafka, WebSocket) align with scalability/latency goals. No incorrect details.
- **Missing Details**: Added APIs, protocols, analytics, notifications, security, and detailed schemas for completeness.

## Summary
Online Ticket Booking uses **Redis** for fast reservations, **PostgreSQL** for durable bookings, **Kafka** for event ingestion, and **WebSocket** for queue updates. It scales to 10K bookings/sec, 1M daily tickets, 50K QPS with <200ms reservations, <500ms bookings. Strong consistency via SERIALIZABLE transactions, robust concurrency with distributed locks, and virtual queues ensure FAANG-readiness with **HTTPS/gRPC/WebSocket**.

---
---


# E-Commerce Website (Amazon) System Design

## Functional Requirements
- **Product Catalog**: Browse and search products by category, name, or attributes.
- **Shopping Cart**: Add/remove items, persist across sessions.
- **Checkout**: Process orders with payment and shipping details.
- **Order Management**: Track order status, handle cancellations/refunds.
- **Inventory Management**: Real-time stock updates across warehouses.
- **User Accounts**: Manage profiles, addresses, and payment methods.

## Non-Functional Requirements
- **Scalability**: Handle ~100M daily active users (DAU), ~1M orders/sec, ~100K QPS for searches.
- **Latency**: <100ms for searches, <500ms for checkout, ~10K orders/sec processing.
- **Availability**: 99.99% uptime, no single points of failure.
- **Consistency**: Strong for inventory/orders to prevent overselling, eventual for catalog searches.
- **Security**: Secure payment processing, prevent fraud, protect user data.

## Strong Consistency Considerations
- **Transactional Integrity**: Ensure atomic inventory updates and order confirmations.
- **Failure Handling**: Manage payment failures, partial orders, and cart conflicts.

## Improvements
- **Catalog**: 
  - Store products in **DynamoDB** {`product_id`, `name`, `category`, `price`, `attributes`} for flexible schemas.
  - Index in **Elasticsearch** for fast text and faceted search (e.g., by category, price range).
- **Cart**: 
  - Use **Redis** for in-memory carts (`cart:user_id`, hash of `item_id:quantity`, TTL: 7 days) to handle abandoned carts.
  - Persist critical cart updates to **DynamoDB** for durability.
- **Order Processing**: 
  - Implement **saga pattern** with **Kafka** for distributed transactions:
    - **Inventory Service**: Reserve stock.
    - **Payment Service**: Process payment.
    - **Shipping Service**: Generate shipping label.
  - Coordinate via Kafka topics (e.g., `order:reserve`, `order:pay`, `order:ship`).
- **Idempotency**: 
  - Ensure checkout idempotency using unique `order_id` in Kafka/DynamoDB.
  - Store `order_id` in Redis (`order:order_id`, flag) for quick duplicate checks.
- **Edge Cases**: 
  - **Out-of-Stock**: Real-time inventory checks in DynamoDB during checkout.
  - **Failed Payments**: Retry via Kafka events, rollback inventory if final failure.
  - **Partial Orders**: Compensate via saga rollback (release inventory, refund payment).
  - **Cart Conflicts**: Use **optimistic locking** in Redis (version field in `cart:user_id`).
  - **Order Fraud**: Apply anomaly detection (e.g., unusual purchase patterns) with **Spark** analytics.
- **Monitoring**: Track checkout success rate, search latency, inventory accuracy, order processing latency, Kafka queue lag, Elasticsearch query throughput.

## Network Protocols & APIs
- **Protocols**: 
  - **HTTPS**: Secure API for catalog, cart, and checkout.
  - **gRPC**: Fast internal communication for microservices (inventory, payment, shipping).
  - **TCP**: Reliable Kafka/DynamoDB connections.
- **APIs**:
  - `GET /products/search`: Search products by query or filters.
    ```json
    {
      "query": "laptop",
      "filters": {"category": "electronics", "max_price": 1000},
      "products": [{"product_id": "uuid", "name": "Laptop", ...}]
    }
    ```
  - `POST /cart`: Add item to cart.
    ```json
    {
      "user_id": "uuid",
      "item_id": "uuid",
      "quantity": 2
    }
    ```
  - `POST /orders`: Process checkout with payment.
    ```json
    {
      "user_id": "uuid",
      "order_id": "uuid",
      "items": [{"item_id": "uuid", "quantity": 2}],
      "payment": {...}
    }
    ```

## Trade-offs
- **Monolith vs. Microservices**: 
  - **Monolith**: Simpler, less scalable. 
  - **Microservices**: Flexible, complex. 
  - *Choice*: **Microservices** for scalability.
- **Eventual vs. Strong Consistency**: 
  - **Eventual**: Scalable, risks overselling. 
  - **Strong**: Safe, slower. 
  - *Choice*: **Strong** for inventory/orders.
- **DynamoDB vs. Cassandra**: 
  - **DynamoDB**: Simpler, costlier. 
  - **Cassandra**: Scalable, complex. 
  - *Choice*: **DynamoDB** for ease of use.

## Why Microservices + DynamoDB?
- **Microservices**: Independent scaling for catalog, cart, and order processing.
- **DynamoDB**: High read/write throughput, managed scaling.

## Scalability Estimates
- **Users**: ~100M DAU via microservices, DynamoDB partitioning.
- **Orders**: ~1M orders/sec with Kafka and DynamoDB.
- **Searches**: ~100K QPS with Elasticsearch sharding.

## Latency/Throughput Targets
- **Searches**: <100ms (Elasticsearch).
- **Checkout**: <500ms (DynamoDB/Kafka).
- **Processing**: ~10K orders/sec.

## Database Schemas
- **DynamoDB Products Table**:
  ```sql
  Table: products
  Partition Key: product_id (String)
  Attributes:
    name (String)
    category (String)
    price (Number)
    attributes (Map)
  ```
  - **Consistency**: Eventual for reads, strong for inventory updates.
  - **Availability**: Multi-region.
- **DynamoDB Orders Table**:
  ```sql
  Table: orders
  Partition Key: order_id (String)
  Attributes:
    user_id (String)
    items (List<Map>)
    status (String)
    timestamp (Number)
  ```
  - **Consistency**: Strong for writes (conditional updates).
- **DynamoDB Inventory Table**:
  ```sql
  Table: inventory
  Partition Key: item_id (String)
  Attributes:
    warehouse_id (String)
    quantity (Number)
    last_updated (Number)
  ```
  - **Consistency**: Strong for stock updates.

## Cache Schemas
- **Redis**:
  - `cart:user_id`: Cart items (hash, TTL: 7 days).
  - `order:order_id`: Idempotency flag (flag, TTL: 1hr).
  - `product:product_id`: Product details (JSON, TTL: 1hr).
  - **Consistency**: Eventual, sync with DynamoDB.
- **Tricks**: Redis pipelining for batch cart updates, LRU for cache eviction.

## Consistency vs. Availability
- **DynamoDB**: 
  - **Strong Consistency**: For inventory/orders (conditional writes).
  - **High Availability**: Multi-region.
- **Elasticsearch**: 
  - **Eventual Consistency**: For product search.
  - **High Availability**: Sharded clusters.
- **Redis**: 
  - **Eventual Consistency**: For carts/products.
  - **High Availability**: Redis Cluster.
- **Kafka**: 
  - **Strong Consistency**: For saga events.
  - **High Availability**: Multi-broker clusters.
- **Tricks**: DynamoDB conditional writes for idempotency, Kafka idempotent producers.

## Additional Considerations
- **Security**: 
  - Use **PCI-compliant payment gateway**.
  - Implement **JWT auth** for API access.
- **Analytics**: Log orders in **Kafka**, aggregate in **Redshift** for sales trends.
- **Notifications**: Order updates via **SNS**, real-time via **WebSocket**.
- **Rate Limiting**: Cap checkout attempts per user (Redis token bucket).
- **Backup**: Archive orders to **S3/Glacier**.

## Correctness Check
- **Original Design**: All components (DynamoDB, Elasticsearch, Redis, Kafka) align with scalability/latency goals. No incorrect details.
- **Missing Details**: Added APIs, protocols, analytics, notifications, security, and detailed schemas for completeness.

## Summary
Amazon E-Commerce uses **microservices** with **DynamoDB** for catalog/orders, **Elasticsearch** for search, **Redis** for carts, and **Kafka** for saga-based order processing. It scales to 100M DAU, 1M orders/sec, 100K QPS with <100ms searches, <500ms checkout. Strong consistency for inventory/orders, idempotent checkout, and robust edge case handling with **HTTPS/gRPC** ensure FAANG-readiness.

---
---

### 20. Online Messaging App (WhatsApp/Slack)
**Improvements:**
- **Message Queues**: Use Kafka for message ingestion and delivery. Partition by chat_id for scalability.
- **Delivery Receipts**: Store read/unread status in Redis and sync to DB (Cassandra) for durability.
- **Retries**: Implement exponential backoff for failed deliveries. Use dead-letter queues for persistent failures.
- **Offline Storage**: Store messages in Cassandra for offline users, with push notifications via APNs/GCM.
- To enable users to **send and receive media**, a separate, dedicated system for attachment management is the most efficient approach. When a user wants to send media, the Chat Server can provide a pre-signed URL, granting temporary permission to upload the file directly to blob storage. Once uploaded, the user sends the resulting URL as part of their message via the Chat Server. Recipients can then retrieve the media directly from the blob storage using another pre-signed URL for authorization. While this method requires managing the expiration of media after all recipients have downloaded it and ensuring proper encryption and security, it offloads the bandwidth and storage burden from the Chat Server and its database, leveraging purpose-built technology for media handling.
- To **handle billions of users**, horizontal scaling of Chat Servers is essential, coupled with a Pub/Sub system (like Redis) for efficient message routing across servers. Upon connection, each Chat Server subscribes to Pub/Sub topics corresponding to its connected users. When a message is sent, the originating server publishes it to the relevant recipient user topics. All Chat Servers subscribed to those topics receive the message and forward it to their connected users. While Pub/Sub offers "at most once" delivery, our existing Inbox feature ensures message persistence for offline users. This architecture introduces minor latency and necessitates all-to-all connections between Chat and Redis servers but provides the required scalability.
- To **handle multiple clients per user**, we need to track active client devices. We'll introduce a Clients table, linking users to their active clientIds. When retrieving chat participants, we'll fetch all clients for each user. Instead of subscribing to user IDs in Pub/Sub, Chat Servers will subscribe to individual clientId topics. When sending a message, it will be published to the Pub/Sub topic of each of the user's active clients. Consequently, our Inbox table will also become client-specific to ensure messages are stored and delivered to each device independently when it comes online. Implementing a limit on the number of clients per user (e.g., 3) will help manage storage and throughput.
- **Edge Cases**: Handle message ordering (use sequence numbers), duplicate messages (idempotency keys), group chats (fanout logic), message expiration (TTL policies), and delivery delays (retry logic).
- **Monitoring**: Track delivery latency, notification success rate, queue backlog, and message processing success rate.
- **Trade-offs:**
  - **Kafka vs. RabbitMQ**: Kafka is durable and scalable; RabbitMQ is simpler but less robust. Choose Kafka for high throughput.
  - **Push vs. Pull Notifications**: Push is real-time but complex; pull is simpler but delayed. Choose push for UX.
  - **Cassandra vs. DynamoDB**: Cassandra is cost-effective for writes; DynamoDB is simpler but costlier. Choose Cassandra for scalability.
- **Why Kafka + Cassandra?**: Kafka ensures reliable message delivery, and Cassandra scales for message storage.
- **Scalability Estimates**: Supports ~100M daily active users, ~1B messages/day, and ~50K QPS for delivery with Kafka and Cassandra scaling.
- **Latency/Throughput Targets**: Target <100ms for message delivery, <200ms for notifications, and ~10K messages/sec ingestion.

---
---

### 21. Task Management Tool
**Improvements:**
- **CRUD APIs**: Use REST APIs with a relational DB (PostgreSQL) for tasks {task_id, user_id, status, due_date}.
- **Auth**: Implement OAuth2 for user authentication and RBAC for task permissions.
- **Background Jobs**: Use Celery + RabbitMQ for task reminders and status updates.
- **Audit Trails**: Log all task changes in a separate table for accountability.
- **Edge Cases**: Handle concurrent task edits (optimistic locking), deleted users (soft deletes), overdue tasks (notifications), task duplication (idempotency keys), and permission errors (validate RBAC).
- **Monitoring**: Track API latency, job queue length, audit log size, and task update success rate.
- **Trade-offs:**
  - **SQL vs. NoSQL**: SQL is better for relational queries (e.g., task assignments); NoSQL is flexible but complex for joins. Choose SQL for simplicity.
  - **Celery vs. Custom Jobs**: Celery is robust but heavy; custom jobs are lightweight but error-prone. Choose Celery for reliability.
  - **REST vs. GraphQL**: REST is simpler but verbose; GraphQL is flexible but complex. Choose REST for ease of use.
- **Why PostgreSQL + Celery?**: PostgreSQL handles relational data well, and Celery ensures reliable background jobs.
- **Scalability Estimates**: Supports ~1M daily active users, ~10M tasks/day, and ~10K QPS for API calls with PostgreSQL and Celery scaling.
- **Latency/Throughput Targets**: Target <100ms for task CRUD, <500ms for reminders, and ~5K tasks/sec processing.

---
---

### 22. Tinder
**Improvements:**
- **Profile Storage**: Store user profiles in DynamoDB {user_id, preferences, location} for scalability.
- **Matchmaking**: Use Redis Geo for swipe matching based on location and preferences.
- **Transactions**: Use optimistic locking for swipe actions to prevent double-swipes.
- To handle the **high volume of swipe data** and the need for independent scaling, we'll use a dedicated Swipe Service and a separate Swipe Database (like Cassandra). When a user swipes on a profile, the client sends a request to the Swipe Service, which then records the swipe in the Swipe Database, partitioned by the swiping user's ID for efficient lookups. The Swipe Service then checks the database for a reciprocal swipe. If a match is found (both users swiped right on each other), the service notifies the user. This separation allows us to optimize the Swipe Service and Database for high write throughput and implement swipe-specific logic without impacting other parts of the system. Cassandra's write-optimized nature makes it suitable for the anticipated massive volume of daily swipes.
- When a mutual swipe occurs, the user who initiated the second swipe (Person B) is immediately **notified of the match** within the app. To notify the first swiper (Person A), who might have swiped much earlier, we'll leverage push notification services like APNS (for iOS) or FCM (for Android). When Person B's swipe creates a match, our Swipe Service will trigger a push notification to Person A's device, informing them of the new connection. This ensures both users are promptly aware of their mutual interest, regardless of when the initial swipe occurred.
- To ensure **consistent and low-latency swiping** and immediate match detection, we'll employ Redis for atomic operations alongside Cassandra for durable storage. When a user swipes, we'll use a Lua script in Redis to atomically record the swipe and check for a reciprocal "right" swipe. The Redis key will be a consistent hash of both user IDs, ensuring swipes between the same pair land on the same shard. If both users have swiped right, a match is immediately created. While this introduces the operational complexity of managing a Redis cluster, its atomic operations guarantee consistency for real-time matching. We can mitigate Redis memory usage by periodically flushing swipe data to Cassandra, treating Redis as a fast, consistent cache for recent swipes. This hybrid approach balances consistency and low latency with the scalability and durability of Cassandra.
- To ensure **low latency for feed/stack generation**, we'll combine pre-computation and an indexed database. We'll periodically pre-compute and cache initial feeds for active users based on their preferences and location, allowing for instant display when the app opens. As users swipe, if they exhaust the cache, we'll seamlessly fetch new profiles in real-time using an indexed database like Elasticsearch, which can efficiently handle the filtering queries. To **avoid stale profiles** due to changes in user preferences or location, we'll implement a short TTL for cached feeds and trigger background feed refreshes upon user-initiated actions like updating their filters or location. This hybrid approach provides immediate access to profiles while ensuring a continuous and relevant stream of potential matches with minimal delay. Tunable parameters like cache TTL and the number of pre-computed profiles offer flexibility in optimizing the system.
- To **avoid showing users profiles they've already swiped on**, we can combine caching with a Bloom filter for users with extensive swipe histories. For each user, we'll maintain a cache of recently swiped profile IDs for quick lookups. When generating a new feed, we'll first check this cache to exclude previously seen profiles. For users with a very large number of past swipes, where a simple cache might become inefficient, we'll also employ a Bloom filter. This probabilistic data structure can efficiently check if a profile has likely been swiped on before. While Bloom filters can have false positives (occasionally suggesting a profile has been seen when it hasn't), they guarantee no false negatives (never indicating a profile hasn't been seen when it has). This ensures we don't re-show swiped profiles, although it might slightly reduce the pool of available profiles due to the possibility of false positives. We'll need to manage the Bloom filter cache, considering updates and potential recovery, balancing accuracy (low false positive rate) with storage and performance.
- **Edge Cases**: Handle user bans (soft deletes), location changes (re-index), high swipe rates (rate-limiting), profile staleness (periodic refresh), and match conflicts (idempotency keys).
- **Monitoring**: Track match latency, swipe success rate, geo-query performance, and match accuracy.
- **Trade-offs:**
  - **Redis Geo vs. PostGIS**: Redis is faster for simple queries; PostGIS is robust for complex ops. Choose Redis for low-latency matching.
  - **Optimistic vs. Pessimistic Locking**: Optimistic scales better but risks conflicts; pessimistic is safe but slower. Choose optimistic for typical loads.
  - **DynamoDB vs. Cassandra**: DynamoDB is simpler but costlier; Cassandra is scalable but complex. Choose DynamoDB for ease of use.
- **Why Redis + DynamoDB?**: Redis ensures fast geospatial matching, and DynamoDB scales for profiles.
- **Scalability Estimates**: Supports ~10M daily active users, ~100M swipes/day, and ~50K QPS for matching with Redis and DynamoDB scaling.
- **Latency/Throughput Targets**: Target <50ms for swipe matching, <200ms for profile retrieval, and ~10K swipes/sec processing.

---
---

### 23. Online Auction
**Improvements:**
- **Bidding**: Use Redis for real-time bid counters and PostgreSQL for durable bid history {auction_id, user_id, bid_amount, timestamp}.
- **Concurrency**: Use pessimistic locking for high-contention auctions to prevent overbidding.
- **Notifications**: Push bid updates via WebSockets and store outbid notifications in Kafka.
- **Edge Cases**: Handle bid sniping (extend auction timer), failed bids (rollback), auction endings (auto-close), bidder fraud (rate-limiting), and network failures (retry logic).
- **Monitoring**: Track bid latency, lock contention, notification delivery, and bid success rate.
- **Trade-offs:**
  - **Pessimistic vs. Optimistic Locking**: Pessimistic is safe for auctions but slower; optimistic risks conflicts. Choose pessimistic for high-contention.
  - **Redis vs. DB for Bids**: Redis is fast but volatile; DB is durable but slower. Use Redis with DB backup.
  - **WebSockets vs. Polling**: WebSockets are real-time but complex; polling is simpler but delayed. Choose WebSockets for UX.
- **Why Pessimistic + Redis/DB?**: Pessimistic locking ensures bid integrity, and Redis/DB balances speed and durability.
- **Scalability Estimates**: Supports ~10K bids/sec, ~1M daily auctions, and ~50K QPS for bid updates with Redis and PostgreSQL scaling.
- **Latency/Throughput Targets**: Target <100ms for bid submission, <200ms for notifications, and ~5K bids/sec processing.

---
---

### 24. Robinhood
**Improvements:**
- **Trades**: Store trade orders in PostgreSQL {order_id, user_id, stock, amount, timestamp} with strong consistency.
- **Concurrency**: Use optimistic locking for order placement to handle race conditions.
- **Price Feeds**: Stream real-time stock prices via Kafka; cache in Redis for low-latency access.
- **Edge Cases**: Handle market volatility (circuit breakers), failed trades (rollback), partial fills (update order status), order duplicates (idempotency keys), and regulatory compliance (audit logs).
- **Monitoring**: Track trade success rate, price feed latency, lock contention, and order processing accuracy.
- **Trade-offs:**
  - **Optimistic vs. Pessimistic Locking**: Optimistic scales better but risks conflicts; pessimistic is slower but safe. Choose optimistic for typical loads.
  - **Kafka vs. WebSockets for Prices**: Kafka is durable for streams; WebSockets are simpler for clients. Choose Kafka for reliability.
  - **PostgreSQL vs. NoSQL**: PostgreSQL ensures strong consistency; NoSQL is flexible but risks inconsistencies. Choose PostgreSQL for trades.
- **Why Optimistic + PostgreSQL?**: Optimistic locking scales for trades, and PostgreSQL ensures transactional integrity.
- **Scalability Estimates**: Supports ~1M daily active users, ~10M trades/day, and ~50K QPS for price feeds with Kafka and PostgreSQL scaling.
- **Latency/Throughput Targets**: Target <100ms for trade submission, <50ms for price feeds, and ~5K trades/sec processing.

---
---

### 25. Google Docs
**Improvements:**
- **Document Storage**: Store docs in a NoSQL DB (Firestore) {doc_id, content, version, user_id} for flexible schemas.
- **Real-time Editing**: Use operational transformation (OT) or CRDTs for conflict-free edits; sync via WebSockets. To enable multiple users to concurrently edit the same document with consistency, we'll employ Operational Transformation (OT). Instead of sending entire document snapshots, clients will send individual edits (operations) to a central Document Service. This service will maintain a canonical version of the document and a history of operations. When an edit arrives, the Document Service will transform it based on the operations that have already been applied to ensure that the edit is applied in the correct context, regardless of the order in which concurrent edits arrive. These transformed operations are then persistently stored in an append-only Document Operations database (like Cassandra, partitioned by documentId and ordered by timestamp). After the edit is successfully stored, the Document Service acknowledges it to the originating client. This centralized OT approach ensures that all clients eventually converge to the same consistent document state, even with concurrent edits, while also being relatively memory-efficient.
- To enable **real-time viewing of concurrent edits**, when a user initially connects to a document, the server will push all previously applied operations to their client. This reconstructs the current state of the document. Subsequently, whenever any connected user makes an edit that is successfully processed and stored by the server, that same operation is then broadcast in real-time to all other clients currently viewing the same document. Upon receiving an operation from another user, each client applies the same Operational Transformation (OT) logic that the server uses. This ensures that even if local edits and remote edits occur and are processed in different orders, the OT on each client will correctly transform and apply the operations, guaranteeing that all users see a consistent, up-to-date version of the document in real-time.
- To enable users to **see each other's cursor positions and presence** in real-time, we'll handle this ephemeral data separately from the document content. When a user's cursor position changes, their client will send an update to the Document Service via the WebSocket connection. The Document Service will store this cursor position in its in-memory state, associated with the user's session. It will then broadcast this cursor update, along with a user identifier, to all other clients currently connected to the same document via their respective WebSocket connections. Similarly, when a new user connects, the Document Service will read the current cursor positions and presence of all other active users from its in-memory store and send this information to the newly connected client. To manage presence, the Document Service will monitor WebSocket connections; when a user disconnects, their cursor information will be removed from the in-memory store, and a "user left" notification will be broadcast to the remaining collaborators. This in-memory, WebSocket-driven approach ensures low-latency updates for real-time awareness without needing to persist this transient data in the document database.
- To achieve **WebSocket scalability for millions of concurrent document editors**, we'll horizontally scale the Document Service using a consistent hash ring, coordinated by ZooKeeper, to distribute document connections across multiple servers. When a client connects, an initial HTTP request is routed (possibly via a load balancer) to a Document Service instance, which then redirects the client to the specific server responsible for the requested document based on the hash ring. A direct WebSocket connection is established with this server. The server loads the document's operations and manages the WebSocket connections for that document. Real-time updates are broadcast directly to all connected clients on the same server. While consistent hashing minimizes redistribution during scaling events, it introduces complexities in managing ring transitions, potential temporary multi-server communication, and the critical need for robust monitoring, failure recovery, and client-side reconnection mechanisms. Careful capacity planning is also essential to ensure balanced load distribution across all servers.
- To **manage storage with billions of documents**, we'll implement periodic snapshotting and compaction of operations. Instead of an offline Compaction Service, we'll have the Document Service itself perform this task when a document becomes idle (i.e., when the last client disconnects). At this point, the Document Service already holds all the document's operations in memory. It can then offload the compaction process to a separate, low-priority process to avoid impacting the latency of active document operations. This process will consolidate the existing operations into a more efficient representation (potentially a single insert operation representing the current state). The resulting compacted operations are then written back to the Document Operations DB under a new documentVersionId. Finally, the Document Service updates the documentVersionId in the Document Metadata DB, ensuring that new clients loading the document will retrieve the compacted version. This online, service-driven approach simplifies coordination and leverages the Document Service's awareness of document activity for efficient storage management. We'll need to carefully manage the CPU resources allocated to the compaction process to prevent performance degradation for active users.
- **Versioning**: Store edit deltas in Firestore to support undo/redo and history.
- **Edge Cases**: Handle concurrent edits (merge conflicts), offline edits (local caching), large docs (chunked storage), edit storms (rate-limiting), and permission errors (validate access).
- **Monitoring**: Track sync latency, edit conflicts, storage usage, and edit success rate.
- **Trade-offs:**
  - **OT vs. CRDTs**: OT is simpler but requires server coordination; CRDTs are decentralized but complex. Choose OT for simplicity.
  - **Firestore vs. Cassandra**: Firestore is managed but costlier; Cassandra is scalable but complex. Choose Firestore for ease of use.
  - **WebSockets vs. Polling**: WebSockets are real-time but complex; polling is simpler but delayed. Choose WebSockets for UX.
- **Why OT + Firestore?**: OT ensures conflict-free edits, and Firestore scales for document storage.
- **Scalability Estimates**: Supports ~10M daily active users, ~1M docs/day, and ~50K QPS for edits with Firestore and WebSocket scaling.
- **Latency/Throughput Targets**: Target <100ms for edit sync, <500ms for doc retrieval, and ~10K edits/sec processing.

## Scheduler Services
These focus on timing, reliability, and eventual execution.

---
---

### 26. Web Crawler
**Improvements:**
- **Crawling Strategy**: Use BFS for broad coverage, with politeness rules (robots.txt, rate-limiting per domain).
- **Distributed Queues**: Use Kafka for URL queues, partitioned by domain to avoid overloading servers.
- **Duplicate Filter**: Use a Bloom filter in Redis to skip visited URLs.
- To achieve **fault tolerance** in our crawler, we'll employ a pipelined architecture with stages for URL fetching and content extraction, communicating via Kafka topics. Raw HTML and extracted text will be stored in blob storage, with metadata in a database. For handling fetch failures, the URL Fetcher consumer will implement an exponential backoff retry mechanism. If retries are exhausted, the failed URL will be published to a dead-letter topic in Kafka for later analysis. If a crawler instance fails, Kafka's offset tracking ensures that another instance will seamlessly resume processing from the last committed point, preventing data loss. Successful fetching and extraction will result in the consumer committing its offset in the Kafka topic. This staged approach, leveraging Kafka's durability and consumer management along with application-level retries, ensures that our crawling process is resilient to failures and maintains progress.
- Our crawler will respect website etiquette by adhering to **robots.txt** and implementing rate limiting. First, it fetches and stores robots.txt rules (disallowed paths, Crawl-delay) in our Metadata DB. Before crawling, it checks these rules, skipping disallowed URLs and respecting the Crawl-delay by checking the last crawl time. For rate limiting (1 req/sec/domain), we'll use a global Redis counter with a sliding window. Crawlers check this before requests. To avoid synchronized retries, we'll add random jitter to request delays. This dual approach ensures politeness by following website instructions and preventing server overload through controlled request frequency.
- To crawl 10B pages in under 5 days, we need **massive parallelism**. Estimating based on network bandwidth and practical utilization (around 7,500 pages/second per high-powered instance), we'd need roughly 4 such machines. The parser workers can be scaled dynamically using serverless technologies based on the processing queue length. DNS resolution is a crucial bottleneck; we can mitigate this with aggressive DNS caching and potentially using multiple DNS providers with round-robin.
**For efficiency**, we'll first check the Metadata DB to avoid re-crawling known URLs. To handle different URLs with the same content, we can hash the page content and store this hash in the Metadata DB with an index for fast lookups. Before crawling a new URL, we'll hash its content and check if the hash already exists, skipping duplicates. While a Bloom filter is another option for duplicate detection, offering probabilistic set membership testing, the indexed database approach is likely more practical due to its simplicity and the efficiency of modern database indexing. This combination of horizontal scaling, dynamic worker allocation, DNS optimization, and content-based duplicate detection will help us achieve the required scale and efficiency.
- To **avoid crawler traps**, we'll implement a maximum crawl depth. We'll add a depth field to our URL metadata, incrementing it with each followed link. If the depth exceeds a defined limit, we'll stop crawling that path, preventing infinite loops and ensuring we don't get stuck on artificially deep or self-linking structures.
- **Edge Cases**: Handle infinite loops (max depth), broken links (timeout), dynamic content (headless browser), server bans (respect robots.txt), and rate limits (backoff logic).
- **Monitoring**: Track crawl rate, queue backlog, duplicate hit rate, and crawl success rate.
- **Trade-offs:**
  - **BFS vs. DFS**: BFS ensures broad coverage; DFS is deeper but risks getting stuck. Choose BFS for general-purpose crawling.
  - **Bloom Filter vs. Hash Set**: Bloom filter is memory-efficient but has false positives; hash set is accurate but memory-heavy. Choose Bloom filter for scalability.
  - **Kafka vs. RabbitMQ**: Kafka is scalable for queues; RabbitMQ is simpler but less robust. Use Kafka for high throughput.
- **Why BFS + Kafka?**: BFS maximizes coverage, and Kafka scales for distributed crawling.
- **Scalability Estimates**: Supports ~1M URLs/hour, ~10K domains, and ~10K QPS for queue processing with Kafka and Redis scaling.
- **Latency/Throughput Targets**: Target <500ms for URL processing, <1s for page crawls, and ~5K URLs/sec processing.

---
---

### 27. Task Scheduler
**Improvements:**
- **Job Queues**: Use RabbitMQ for task distribution and Redis for scheduling (sorted sets with timestamps).
- **Retry Logic**: Implement exponential backoff for failed tasks. Store retries in a dead-letter queue.
- **Cron Triggers**: Parse cron expressions using a library (e.g., croniter) for flexible scheduling.
- **Edge Cases**: Handle missed schedules (backfill logic), duplicate tasks (idempotency keys), priority conflicts (priority queues), task timeouts (kill jobs), and resource exhaustion (limit concurrency).
- **Monitoring**: Track job execution time, retry rate, queue health, and scheduling accuracy.
- **Trade-offs:**
  - **RabbitMQ vs. Kafka**: RabbitMQ is simpler for task queues; Kafka is overkill for small-scale jobs. Choose RabbitMQ for ease of use.
  - **Redis vs. DB for Scheduling**: Redis is fast but volatile; DB is durable but slower. Use Redis with DB backup.
  - **Cron vs. Event-Based**: Cron is simple but rigid; event-based is flexible but complex. Choose cron for simplicity.
- **Why RabbitMQ + Redis?**: RabbitMQ ensures reliable task delivery, and Redis provides fast scheduling.
- **Scalability Estimates**: Supports ~10K tasks/sec, ~1M daily tasks, and ~10K QPS for scheduling with RabbitMQ and Redis scaling.
- **Latency/Throughput Targets**: Target <50ms for task scheduling, <500ms for execution, and ~5K tasks/sec processing.

---
---

### 28. Real-Time Notification System
**Improvements:**
- **Delivery**: Use WebSockets for real-time push and APNs/GCM for mobile notifications.
- **Webhooks**: Support webhooks for third-party integrations, with retry logic for failures.
- **Device Token Management**: Store tokens in a DB (DynamoDB) and invalidate stale tokens.
- **Edge Cases**: Handle offline users (store notifications), duplicate deliveries (idempotency), rate limits (queue throttling), token churn (periodic cleanup), and notification spam (filtering).
- **Monitoring**: Track delivery success rate, WebSocket latency, token churn, and notification processing success rate.
- **Trade-offs:**
  - **Push vs. Polling**: Push is real-time but complex; polling is simpler but delayed. Choose push for UX.
  - **WebSockets vs. Server-Sent Events**: WebSockets are bidirectional but heavy; SSE is lightweight but unidirectional. Choose WebSockets for interactivity.
  - **DynamoDB vs. Cassandra**: DynamoDB is simpler but costlier; Cassandra is scalable but complex. Choose DynamoDB for ease of use.
- **Why WebSockets + DynamoDB?**: WebSockets ensure real-time delivery, and DynamoDB scales for token storage.
- **Scalability Estimates**: Supports ~100M daily active users, ~1B notifications/day, and ~50K QPS for delivery with WebSocket and DynamoDB scaling.
- **Latency/Throughput Targets**: Target <100ms for notification delivery, <200ms for webhook retries, and ~10K notifications/sec processing.

---
---

### 29. LeetCode
**Improvements:**
- **Submission Queue**: Use RabbitMQ to queue code submissions for judging, partitioned by problem_id.
- **Judging**: Run submissions in isolated containers (Docker) with timeouts; store results in PostgreSQL.
- **Scheduling**: Use Celery for scheduling test case runs and result aggregation.
- To provide instant feedback, we'll run user-submitted code within **isolated Docker containers**. For each supported programming language, we'll have pre-configured containers with necessary dependencies and sandboxing. When a user submits code for a specific problem, the API Server will route it to an available container for that language. The container will execute the code against test cases in a secure, sandboxed environment. The execution result is then returned to the API Server, which persists it in the database and sends the feedback back to the user in real-time. This containerization approach offers a balance of isolation, resource efficiency, and fast execution compared to VMs or serverless functions (avoiding potential cold starts). We'll need to carefully configure container security and resource limits.
- To ensure **isolation and security**, our containerized code execution environment will implement several key restrictions. We'll mount the code directory as read-only and use temporary, short-lived directories for output. CPU and memory limits will be enforced to prevent resource exhaustion, and an explicit timeout will kill processes exceeding a defined duration. Network access will be disabled to prevent external calls. Finally, we'll leverage seccomp to restrict the system calls available to the container, minimizing the potential for host system compromise. These measures collectively create a robust sandbox for safely running untrusted user code.
- To **efficiently fetch the leaderboard**, we'll use a Redis sorted set. Upon each submission, we'll update both the main database and the Redis sorted set, using the competition ID as the key, the user's score as the score, and the user ID as the value. Clients will periodically poll the server (e.g., every 5 seconds) via an API endpoint that retrieves the top N users and their scores directly from the Redis sorted set using a ZREVRANGE command. This in-memory approach avoids expensive database queries for leaderboard reads, provides near real-time updates, and scales well. We can also implement client-side pagination to further optimize the initial load and subsequent fetches.
- To **handle 100,000 concurrent users** in competitions, we'll dynamically horizontally scale our language-specific code execution containers using auto-scaling groups (e.g., with ECS Auto Scaling). This allows us to automatically adjust the number of active containers based on CPU utilization and traffic demand. Alternatively, we could introduce an SQS queue between the API server and the containers to buffer submissions during peak loads, making the system asynchronous and requiring clients to poll for results. While potentially over-engineered for this scale, the queue offers the benefit of retries in case of container failures, adding robustness. The choice depends on prioritizing immediate feedback versus resilience and handling potential unexpected spikes.
- To **run test cases across different languages**, we'll use a standardized serialization format (e.g., JSON) for test case inputs and expected outputs. For each problem, we'll define a single set of these serialized test cases. For each supported language, we'll create a test harness that can deserialize the input into the language-specific data structures (e.g., a Python list into a TreeNode object), execute the user's code with this input, and then compare the resulting output with the deserialized expected output. This approach avoids maintaining language-specific test cases.
- **Edge Cases**: Handle malicious code (sandboxing), timeouts (kill containers), high submission rates (scale workers), test case failures (retry logic), and result disputes (audit logs).
- **Monitoring**: Track submission latency, judge accuracy, queue backlog, and submission success rate.
- **Trade-offs:**
  - **RabbitMQ vs. Kafka**: RabbitMQ is simpler for task queues; Kafka is overkill for submissions. Choose RabbitMQ for ease of use.
  - **Docker vs. VMs**: Docker is lightweight but less secure; VMs are secure but heavy. Choose Docker with sandboxing for speed.
  - **Celery vs. Custom Jobs**: Celery is robust but heavy; custom jobs are lightweight but error-prone. Choose Celery for reliability.
- **Why RabbitMQ + Docker?**: RabbitMQ ensures reliable queuing, and Docker provides scalable judging.
- **Scalability Estimates**: Supports ~10K submissions/sec, ~1M daily submissions, and ~10K QPS for judging with RabbitMQ and Docker scaling.
- **Latency/Throughput Targets**: Target <1s for submission judging, <500ms for queuing, and ~5K submissions/sec processing.

---
---

### 30. Ad Click Aggregator
**Improvements:**
- **Ingestion**: Use Kafka to collect click events, partitioned by ad_id. To enable advertisers to query ad click metrics at 1-minute intervals efficiently, we'll implement a real-time analytics pipeline using stream processing. When a click occurs, the click processing service will immediately write an event to a stream like Kafka or Kinesis. A stream processor such as Flink will consume these events and aggregate click metrics in memory in near real-time, potentially aggregating on minute boundaries but flushing results to an OLAP database every few seconds. Advertisers can then query this OLAP database to retrieve up-to-the-minute ad performance metrics with low latency. This approach avoids the scalability and latency issues of querying the transactional database directly and offers more flexibility than batch processing.
- To **scale to 10k clicks per second**, we'll horizontally scale the Click Processor Service behind a load balancer. The stream (Kafka/Kinesis) will be sharded by AdId to distribute load, requiring the stream processor (Flink) to process each shard in parallel. The OLAP database will also be horizontally scaled, potentially sharded by AdvertiserId to optimize advertiser-specific queries. To mitigate hot shards caused by popular ads, we can dynamically update the partition key for high-traffic AdIds by appending a random number, further distributing the load across more shards.
- To **ensure we don't lose click data**, we'll leverage the inherent fault tolerance of our stream (Kafka/Kinesis) by enabling persistent storage with a sufficient retention period (e.g., 7 days). This allows reprocessing of data if the stream processor fails. While stream processor checkpointing is an option, for our small aggregation windows, simply re-reading lost events from the persistent stream upon recovery is likely sufficient. To guarantee data accuracy, we'll implement periodic reconciliation. We'll dump raw click events to a data lake (like S3) and run hourly or daily batch jobs to re-aggregate the data, comparing the results with the stream processor's output to identify and correct any discrepancies in the OLAP database.
- To prevent **abuse from multiple clicks**, the Ad Placement Service will generate a unique, signed impression ID for each displayed ad. When a click occurs, the browser sends this ID. The Click Processor verifies the signature and then checks if the impression ID exists in a distributed cache (like Redis Cluster). If present, it's a duplicate and ignored. If not, the click is processed, added to the stream, and the impression ID is added to the cache. This method ensures idempotency by tracking unique ad views and preventing the counting of repeated clicks, even across aggregation windows.
- **Aggregation**: Process clicks in real-time with Flink; store aggregates in Redis for fast reads and PostgreSQL for durability.
- **Scheduling**: Use cron jobs to compute daily/weekly ad performance reports.
- **Edge Cases**: Handle click fraud (rate-limiting), delayed events (event-time processing), high throughput (scale Kafka), duplicate clicks (idempotency keys), and data skew (repartition Kafka).
- **Monitoring**: Track click latency, aggregation accuracy, Kafka lag, and click processing success rate.
- **Trade-offs:**
  - **Flink vs. Spark Streaming**: Flink is better for real-time; Spark is robust for batch. Choose Flink for low-latency aggregation.
  - **Redis vs. DB for Aggregates**: Redis is fast but volatile; DB is durable but slower. Use Redis with DB backup.
  - **Kafka vs. RabbitMQ**: Kafka is scalable for events; RabbitMQ is simpler but less robust. Use Kafka for high throughput.
- **Why Kafka + Flink?**: Kafka handles high-throughput clicks, and Flink ensures real-time aggregation.
- **Scalability Estimates**: Supports ~100K clicks/sec, ~1B daily clicks, and ~10K QPS for aggregates with Kafka and Flink scaling.
- **Latency/Throughput Targets**: Target <100ms for click ingestion, <500ms for aggregates, and ~50K clicks/sec processing.

## Trie / Proximity Systems
These focus on efficient data structures and low-latency retrieval.

---
---

### 31. Search Autocomplete System
**Improvements:**
- **Data Structure**: Use a trie backed by Redis for prefix lookups, with frequency scores for ranking.
- **Debouncing**: Implement client-side debouncing (300ms) to reduce server load.
- **Caching**: Cache popular queries in Redis (LRU eviction).
- **Typo Tolerance**: Use Levenshtein distance or n-grams for fuzzy matching.
- **Edge Cases**: Handle long queries (truncate trie depth), empty results (fallback to popular searches), high QPS (load balancing), query spam (rate-limiting), and stale suggestions (periodic refresh).
- **Monitoring**: Track query latency, cache hit ratio, suggestion accuracy, and query processing success rate.
- **Trade-offs:**
  - **Trie vs. Ternary Search Tree**: Trie is simple but memory-heavy; TST is compact but complex. Choose trie for simplicity and Redis integration.
  - **Exact vs. Fuzzy Matching**: Exact is fast but strict; fuzzy is flexible but slow. Use exact with fuzzy fallback for balance.
  - **Redis vs. In-Memory**: Redis is durable and scalable; in-memory is faster but volatile. Use Redis for reliability.
- **Why Trie + Redis?**: Trie ensures fast prefix lookups, and Redis scales for caching and ranking.
- **Scalability Estimates**: Supports ~100K QPS for queries, ~1M daily active users, and ~10M suggestions/day with Redis scaling.
- **Latency/Throughput Targets**: Target <50ms for suggestions, <100ms for caching, and ~50K queries/sec processing.

---
---

### 32. Ride-Sharing App (Uber/Lyft)
**Improvements:**
- **Matchmaking**: Use a geospatial index (Redis Geo or PostgreSQL PostGIS) to match riders with nearby drivers.
- **Location Tracking**: Stream driver locations via Kafka, processed by a real-time engine (Flink).
- **ETA Algorithms**: Use graph-based routing (OSRM) with traffic data for accurate ETAs.
- **Surge Pricing**: Compute dynamic pricing based on supply/demand ratios, cached in Redis.
- To handle frequent **driver location updates and efficient proximity searches**, we'll use Redis as a real-time in-memory geospatial data store. Drivers will send location updates directly to Redis, leveraging its GEOADD command for efficient storage using geohashing. For proximity searches, we'll use Redis's GEOSEARCH command to quickly find nearby drivers within a given radius. Redis's in-memory nature ensures low latency for both writes and reads, handling the high update frequency. To mitigate data loss, we'll enable Redis persistence (RDB/AOF) and use Redis Sentinel for high availability with automatic failover. The rapid recovery time of Redis minimizes the impact of potential failures.
- To **manage system overload from frequent driver location updates while ensuring accuracy**, we'll implement adaptive location update intervals. The driver's app will intelligently determine the update frequency based on factors like speed and movement patterns. When stationary or moving slowly, updates will be less frequent, conserving bandwidth and resources. Conversely, during rapid movement or directional changes, updates will become more frequent to maintain accuracy. This on-device logic reduces the overall number of pings to the server without significantly compromising the precision of location data for critical use cases like ride matching.
- To **prevent simultaneous ride requests to the same driver**, we'll use a distributed lock with a Time-To-Live (TTL) implemented in Redis. When a ride request is sent to a driver, the Ride Matching Service attempts to acquire a lock on the driver's ID in Redis with a 10-second TTL. If the lock is acquired, no other request can be sent to that driver. If the driver accepts, the lock is released. If the driver doesn't respond within the TTL, the lock expires, making the driver available for other requests. This ensures consistency by allowing only one active request per driver at any time.
- To **prevent dropped ride requests during peak demand**, we'll implement a queueing system with dynamic scaling. Incoming ride requests will be added to a distributed queue (like Kafka). The Ride Matching Service will consume requests from this queue. To handle increased load, the Ride Matching Service will automatically scale horizontally by adding more instances. Kafka's offset management ensures that requests are processed at least once, even if service instances fail. Partitioning queues by geographic region can further optimize processing. While a FIFO queue is the simplest approach, a priority queue could be considered to prioritize requests based on factors like driver proximity for better efficiency.
- To further scale the system for reduced latency and improved throughput, we'll implement **geo-sharding**. 1  We'll partition our data (services, queues, databases) based on geographic regions, ensuring that requests are typically routed to the closest shard, minimizing latency. 2  For read-heavy operations like fetching ride history, we'll utilize read replicas within each shard to increase read throughput. While proximity searches near geographic boundaries might require querying multiple shards (scatter-gather), the overall reduction in distance and the increased read capacity will significantly enhance performance and scalability. Consistent hashing will manage data distribution across shards, and replication will ensure fault tolerance.   
- **Edge Cases**: Handle driver cancellations (re-match logic), GPS errors (filter outliers), high demand (queue riders), surge abuse (cap multipliers), and network failures (retry logic).
- **Monitoring**: Track match latency, ETA accuracy, surge multiplier, and match success rate.
- **Trade-offs:**
  - **Redis Geo vs. PostGIS**: Redis is faster for simple queries; PostGIS is robust for complex geospatial operations. Choose Redis for low-latency matching.
  - **Real-time vs. Batch ETA**: Real-time is accurate but compute-heavy; batch is efficient but stale. Choose real-time for UX.
  - **Kafka vs. RabbitMQ**: Kafka is scalable for streams; RabbitMQ is simpler but less robust. Use Kafka for high throughput.
- **Why Redis + Kafka?**: Redis ensures fast geospatial queries, and Kafka handles real-time location streams.
- **Scalability Estimates**: Supports ~10M daily active users, ~1M rides/day, and ~50K QPS for matching with Redis and Kafka scaling.
- **Latency/Throughput Targets**: Target <100ms for driver matching, <500ms for ETA, and ~10K matches/sec processing.

---
---

### 33. Typeahead Suggestion
**Improvements:**
- **Data Structure**: Use a trie in Redis for prefix lookups, with frequency scores for ranking.
- **Caching**: Cache popular queries in Redis (LRU eviction).
- **Typo Tolerance**: Use n-grams or Levenshtein distance for fuzzy matching.
- **Edge Cases**: Handle long queries (limit trie depth), empty results (fallback to trending), high QPS (load balancing), query duplicates (idempotency), and stale suggestions (periodic refresh).
- **Monitoring**: Track suggestion latency, cache hit ratio, accuracy, and suggestion success rate.
- **Trade-offs:**
  - **Trie vs. Inverted Index**: Trie is fast for prefixes; inverted index is flexible for full-text. Choose trie for prefix focus.
  - **Exact vs. Fuzzy Matching**: Exact is fast but strict; fuzzy is flexible but slow. Use exact with fuzzy fallback.
  - **Redis vs. In-Memory**: Redis is durable and scalable; in-memory is faster but volatile. Use Redis for reliability.
- **Why Trie + Redis?**: Trie ensures fast prefix lookups, and Redis scales for caching.
- **Scalability Estimates**: Supports ~100K QPS for queries, ~1M daily active users, and ~10M suggestions/day with Redis scaling.
- **Latency/Throughput Targets**: Target <50ms for suggestions, <100ms for caching, and ~50K queries/sec processing.

---
---

### 34. Twitter Search
**Improvements:**
- **Search Index**: Use Elasticsearch to index tweets {tweet_id, content, user_id, timestamp} for full-text search.
- **Caching**: Cache recent/popular searches in Redis (TTL-based eviction).
- **Ranking**: Rank results by recency, relevance, and engagement using a weighted scoring algorithm.
- **Edge Cases**: Handle trending topics (cache hot queries), deleted tweets (tombstones), high QPS (scale Elasticsearch), search spam (rate-limiting), and stale results (periodic refresh).
- **Monitoring**: Track search latency, ranking accuracy, cache hit ratio, and search success rate.
- **Trade-offs:**
  - **Elasticsearch vs. Solr**: Elasticsearch is managed but costlier; Solr is open-source but complex. Choose Elasticsearch for ease of use.
  - **Real-time vs. Batch Indexing**: Real-time is fresh but heavy; batch is efficient but stale. Choose real-time for UX.
  - **Caching vs. No Caching**: Caching reduces latency but risks staleness; no caching is fresh but slower. Use caching with TTL for balance.
- **Why Elasticsearch + Redis?**: Elasticsearch handles complex searches, and Redis ensures fast caching.
- **Scalability Estimates**: Supports ~100M daily active users, ~1B tweets/day, and ~100K QPS for searches with Elasticsearch and Redis scaling.
- **Latency/Throughput Targets**: Target <100ms for search results, <200ms for caching, and ~50K searches/sec processing.

---
---

### 35. FB Post Search
**Improvements:**
- **Search Index**: Use Elasticsearch to index posts {post_id, content, user_id, timestamp} with privacy filters.
- **Caching**: Cache frequent searches in Redis (LRU eviction).
- **Ranking**: Rank by relevance, recency, and user affinity; compute scores via background jobs.
- **Edge Cases**: Handle private posts (filter on read), deleted posts (tombstones), high QPS (scale Elasticsearch), search spam (rate-limiting), and stale results (periodic refresh).
- **Monitoring**: Track search latency, privacy filter performance, cache hit ratio, and search success rate.
- **Trade-offs:**
  - **Elasticsearch vs. DB**: Elasticsearch is optimized for search; DB is durable but slower. Choose Elasticsearch for speed.
  - **Real-time vs. Batch Ranking**: Real-time is fresh but heavy; batch is efficient but stale. Choose batch with periodic updates.
  - **Caching vs. No Caching**: Caching reduces latency but risks staleness; no caching is fresh but slower. Use caching with TTL for balance.
- **Why Elasticsearch + Redis?**: Elasticsearch handles complex searches, and Redis ensures low-latency caching.
- **Scalability Estimates**: Supports ~100M daily active users, ~1B posts/day, and ~100K QPS for searches with Elasticsearch and Redis scaling.
- **Latency/Throughput Targets**: Target <100ms for search results, <200ms for caching, and ~50K searches/sec processing.

---
---

### 35. FB Post Search (we're not allowed to use a search engine like Elasticsearch or a pre-built full-text index (like Postgres Full-Text)):
- To enable users to **search posts** by keyword efficiently, we'll create an inverted index. This index will map keywords to the IDs of the posts that contain them. When a new post is ingested, the Ingestion Service will tokenize its content into individual keywords and then append the post's ID to the list associated with each of these keywords in our inverted index. For fast lookups, we'll store this inverted index in Redis, leveraging its in-memory capabilities. When a user searches for a keyword, we can directly retrieve the list of matching post IDs from Redis and return the corresponding posts. This approach provides significantly faster search compared to querying an unindexed database.
- To enable **sorting search results by recency or like count**, we'll maintain multiple indexes in Redis. For recency-based sorting, we'll use Redis lists. When a new post is created, its ID will be appended to the list associated with each of its keywords. For sorting by like count, we'll use Redis sorted sets. The score in the sorted set will represent the post's like count, and the value will be the post's ID. When a new post is created, its ID will be added to the sorted set for each of its keywords with an initial score of zero. When a like event occurs, we'll update the score of the corresponding post ID in the relevant sorted sets. This approach allows for efficient retrieval of results sorted by either criterion directly from the appropriate index.
- To **handle the large volume of search requests**, we'll implement aggressive caching at the edge using a Content Delivery Network (CDN) like Cloudflare or CloudFront. We'll configure our /search endpoint to include cache-control headers, instructing the CDN to cache search results. When a user performs a search, their request will first hit a geographically close CDN node. If the result is cached, it will be served with very low latency. If it's a cache miss, the CDN will forward the request to our API gateway and search service as usual. Combined with our in-memory Redis search cache, the CDN will significantly reduce the load on our origin servers and improve response times for the majority of search queries, especially for popular and repeated searches, leveraging our non-personalized search requirement.
- To **handle multi-keyword phrase** queries efficiently, we can implement bigrams (or shingles). During post ingestion, we'll create tokens for each consecutive pair of words (e.g., "Taylor Swift" becomes "Taylor Swift"). These bigram tokens will also be indexed in our "Likes" and "Creation" Redis indexes, alongside single-word keywords. When a user searches for a phrase like "Taylor Swift", we can directly look up the bigram "Taylor Swift" in our indexes and retrieve the relevant post IDs, rather than performing an intersection of the results for "Taylor" and "Swift". While this increases the index size, it significantly speeds up phrase queries, a common search pattern. We might consider probabilistic methods to index only frequent bigrams to mitigate the size increase.
- To address the **high volume of writes**, particularly for likes, we'll implement a two-stage architecture. For post creation, we'll use Kafka to buffer and distribute ingestion requests across multiple service instances, and we'll shard our Redis indexes by keyword to distribute the write load. For likes, instead of updating the index on every like, we'll only update the like count in our Redis "Likes" index when the count reaches specific milestones (e.g., powers of two). This significantly reduces write frequency, at the cost of the index being an approximation. To provide accurate, up-to-date like counts for sorting, when retrieving search results, we'll fetch a larger set of potential results from the approximate "Likes" index and then query a dedicated Like Service for the precise, current like counts for these posts before the final sorting and ranking. This two-stage approach balances write efficiency with read accuracy.
- To **optimize storage**, we can implement a tiered approach. First, we'll cap the size of our inverted indexes (e.g., to 1k-10k post IDs per keyword), significantly reducing storage for common terms. Second, we'll identify rarely searched keywords based on usage analytics and move their corresponding indexes from our in-memory Redis to cheaper, less frequently accessed blob storage like S3 or R2. When a search query comes in, we'll first check Redis. If the keyword's index isn't found, we'll retrieve it from blob storage, incurring a slight latency penalty for these less common searches. This strategy balances query performance for popular terms with cost-effective storage for the vast majority of less frequently accessed data.



## Aggregation Systems
These focus on processing and ranking high-cardinality data.

---
---

### 36. Top K Problems (YouTube Views, Searched Keywords, Spotify Songs)
**Improvements:**
- **Counting**: Use count-min sketch in Redis for approximate counting of views/searches/plays to save memory.
- **Ranking**: Compute top K items (e.g., views) via a background job (Spark) and cache results in Redis.
- **Sliding Window**: Track counts in a sliding window (e.g., 24 hours) using Redis sorted sets.
- **Edge Cases**: Handle sudden spikes (exponential decay in scoring), stale data (expire old counts), high cardinality (scale Redis), data skew (cap outliers), and duplicate counts (idempotency).
- **Monitoring**: Track sketch accuracy, ranking latency, cache hit ratio, and ranking success rate.
- **Trade-offs:**
  - **Count-Min vs. Exact Counting**: Count-min is memory-efficient but approximate; exact is accurate but heavy. Choose count-min for scalability.
  - **Sliding vs. Fixed Window**: Sliding is precise but complex; fixed is simpler but less granular. Choose sliding for real-time trends.
  - **Redis vs. In-Memory**: Redis is durable and scalable; in-memory is faster but volatile. Use Redis for reliability.
- **Why Count-Min + Redis?**: Count-min scales for high cardinality, and Redis ensures low-latency ranking.
- **Scalability Estimates**: Supports ~100K events/sec, ~1B daily events, and ~10K QPS for rankings with Redis and Spark scaling.
- **Latency/Throughput Targets**: Target <50ms for event updates, <500ms for rankings, and ~50K events/sec processing.

**System Architecture and Data Ingestion**
The architecture leverages a Lambda Architecture, combining a fast path for approximate, low-latency results and a slow path for accurate, batch-processed results.
Data ingestion begins with an API Gateway, which serves as the entry point for user requests (e.g., video views) and generates logs for each request. These logs capture events like video identifiers and are used to count frequencies.
To optimize resource usage, a background process on the API Gateway aggregates events into a small, in-memory hash table (frequency counts) for a short period (e.g., seconds) or until a size limit is reached, then flushes the data in a compact binary format to reduce network I/O. Alternatively, if API Gateway is resource-constrained, log parsing and aggregation can be offloaded to a dedicated log processing cluster.
Aggregated data is sent to a Distributed Messaging System like Apache Kafka, which partitions messages randomly across nodes to distribute load uniformly. Kafka's fault tolerance (via replication and retention, e.g., 7 days) ensures no data loss, enabling reprocessing if needed. This message bus also supports decoupling between producers (ingestion) and consumers (fast/slow processors), allowing horizontal scalability and backpressure handling.

**Fast Path (Approximate Results)**
The fast path prioritizes low-latency, approximate results for short intervals (e.g., 1-5 minutes).
A Fast Processor service consumes Kafka messages and uses a Count-Min Sketch data structure to aggregate frequencies in memory. Count-Min Sketch is a fixed-size, two-dimensional array (e.g., thousands of columns, 5 rows for hash functions) that approximates frequency counts with bounded memory, even for large datasets.
Each event (e.g., video view) is hashed by multiple functions, incrementing corresponding cells. Collisions may overestimate counts, but taking the minimum value across hash functions reduces errors. The Fast Processor maintains a min-heap to track the top k items, updating it as events are processed.
Every few seconds, the Count-Min Sketch and top k list are flushed to a Storage Service, which stores results in a database (e.g., for 1-minute intervals). Since Count-Min Sketch has a fixed size, data partitioning is unnecessary, simplifying the system and avoiding replication complexities.
However, approximate results tolerate some data loss (e.g., from host failures) as the slow path provides accurate results later. This path reduces request rates progressively: millions of events at the API Gateway are aggregated into fewer messages in Kafka, further reduced by Fast Processors, and stored as compact top k lists.
Count-Min Sketch merging is straightforward (element-wise sum), enabling distributed aggregation without coordination overhead.

**Slow Path (Accurate Results)**
The slow path ensures precise results for longer intervals (e.g., 1-hour or 1-day) using batch processing.
A Data Partitioner reads Kafka messages, parses them into individual events, and routes each event (e.g., video identifier) to a specific partition in another Kafka cluster, sharded by video ID. To handle hot partitions (e.g., popular videos), the Partitioner dynamically reassigns high-traffic IDs (e.g., by appending a random suffix).
Partition Processors read from each Kafka partition, aggregating data in memory for a longer period (e.g., 5 minutes), then batch it into files stored in a Distributed File System (e.g., HDFS or S3).
Two MapReduce jobs process these files:
1. The first computes frequency counts for each video
2. The second calculates the top k list
The Frequency Count job maps events to key-value pairs (video ID, count), shuffles and sorts by key, and reduces by summing counts. The Top K job maps frequency data into local top k lists per chunk, then reduces them into a final top k list. Results are stored in the Storage Service.
Kafka's replication ensures data durability, and MapReduce's fault tolerance (via task retries) guarantees accurate processing, though results are delayed (minutes to hours).
Optionally, Apache Spark can replace MapReduce if faster job completion is desired, especially for iterative workflows or near-real-time windows.

**Data Retrieval**
The API Gateway exposes a topK API to retrieve the top k list for a specified time interval. Requests are routed to the Storage Service, which queries its database.
- For short intervals (e.g., last 5 minutes), the system merges multiple 1-minute approximate lists from the fast path
- For exact 1-hour intervals, it retrieves precise lists from the slow path
- For custom intervals (e.g., 2 hours), merging hourly lists may yield approximate results, as precise computation requires the full dataset
Retrieval is optimized for low latency (tens of milliseconds) by pre-computing and storing top k lists, avoiding on-the-fly calculations. Large queries may result in approximations due to list merging limitations, and k should be bounded (e.g., a few thousand) to prevent performance degradation during merge operations.

**Scalability and Performance**
The system scales to handle millions of requests per second:
- The API Gateway cluster (thousands of hosts) distributes load via load balancing
- Kafka's partitioning and horizontal scaling manage high-throughput event streams
- Fast Processors scale by adding hosts, as Count-Min Sketch eliminates partitioning needs
- Partition Processors scale with Kafka shards, and MapReduce jobs leverage distributed computing
- The Storage Service uses a horizontally scaled database, potentially sharded by time interval or item ID
To handle data skew (e.g., popular videos), dynamic partitioning and load balancing mitigate hot spots.
Latency targets include:
- <100ms for event ingestion
- <1s for fast path results
- Minutes for slow path results
Throughput supports ~100K-1M events/sec, with k values up to several thousand (larger k may degrade performance due to merging costs). Sketch dimensions (width/height) can be tuned based on error tolerance and memory availability to balance performance and precision.

**Fault Tolerance and Accuracy**
Kafka's replication and retention ensure no data loss, allowing reprocessing if Fast or Partition Processors fail. The slow path's MapReduce jobs provide accurate results by processing the full dataset, reconciling any fast path inaccuracies.
For critical accuracy, periodic batch jobs can reprocess raw data from the Distributed File System to validate Storage Service data. The fast path sacrifices accuracy for speed, using Count-Min Sketch's probabilistic guarantees (tuned via width/height parameters for desired error bounds).
High availability is achieved via Kafka replication and database redundancy, though fast path data loss is tolerable due to slow path recovery. Recovery strategies include data replay from Kafka, and re-computation via scheduled MapReduce pipelines.

**Edge Cases and Optimizations**
**Hot Partitions:** Dynamic repartitioning in the Data Partitioner mitigates skew from popular items.
**Delayed Events:** Event-time processing in Partition Processors (using Kafka timestamps) handles out-of-order events.
**Duplicate Events:** Idempotency keys (e.g., unique request IDs) in the API Gateway prevent double-counting.
**Large k Values:** Limit k to thousands to avoid performance degradation in merging or storage.
**Resource Constraints:** If API Gateway hosts lack capacity for aggregation, offload log parsing to a separate cluster.
**Data Volume:** For massive datasets, increase Kafka partitions, Fast Processor hosts, or MapReduce nodes.
**Monitoring:** Track ingestion latency, Kafka lag, Fast Processor accuracy (via sampling), MapReduce job completion time, Storage Service query latency, and top k list consistency (via reconciliation jobs). Alert on high partition skew or database bottlenecks.

**Trade-offs**
**Count-Min Sketch vs. Hash Table:** Count-Min Sketch uses fixed memory but is approximate; hash tables are accurate but memory-intensive. Use Count-Min Sketch for fast path simplicity.
**Fast vs. Slow Path:** Fast path is simple and low-latency but approximate; slow path is accurate but complex and slow. Combine both for flexibility.
**Kafka vs. Kinesis:** Kafka is robust for high-throughput streams; Kinesis is managed but costlier. Use Kafka for control and scale.
**MapReduce vs. Spark:** MapReduce is reliable for batch; Spark is faster for iterative processing. Use MapReduce for simplicity, Spark for speed if needed.

**Why Lambda Architecture?**
The dual-path approach balances real-time (fast path) and accurate (slow path) requirements, leveraging Count-Min Sketch for simplicity and MapReduce for precision. Kafka ensures scalable ingestion, and the Storage Service enables fast retrieval.
While Lambda Architecture adds complexity, it offers flexibility and robustness, and is preferred in systems where both speed and correctness matter.

**Alternatives**
A single-path solution using Kafka and Apache Spark could simplify the architecture by partitioning data and aggregating in-memory top k lists, but it may struggle with massive datasets or long intervals without batch processing.
Count-Min Sketch alternatives (e.g., Lossy Counting, Space Saving) offer similar trade-offs but may require more tuning.

**Applications**
This design applies to trending services (e.g., Google Trends, X Trends), DDoS protection (identifying top IP addresses), or financial systems (most traded stocks), making it versatile for frequency-based analytics at scale.
The same architecture can extend to any high-frequency event stream where frequency over time matters, such as ad impressions, page hits, or telemetry.



# Distributed Message Queue System Design

## Overview and Requirements

A distributed message queue enables asynchronous communication between producers and consumers by decoupling message generation from consumption. Unlike synchronous communication, where a producer must wait for the consumer to process a request, a queue stores the message and allows the consumer to retrieve it later. 

This provides better resilience to consumer failures, backpressure, and varying consumption speeds. The queue is distributed in nature, meaning its messages and operations span multiple servers or data centers for scalability and availability. The queue guarantees that each message is consumed by exactly one consumer, which differentiates it from a publish-subscribe topic model where all subscribers receive the same message.

**Functional requirements** include:
- APIs to send and receive messages
- Optionally, APIs to create or delete queues and delete messages

**Non-functional requirements** include:
- Scalability to handle high throughput
- High availability across data centers
- Performance for low-latency operations
- Durability to persist messages safely
- Observability for both system operators and queue clients

## System Architecture and Frontend Layer

The system uses a layered architecture that includes:
- A virtual IP (VIP)
- Load balancer
- A stateless FrontEnd web service
- A Metadata service for queue configuration
- A Backend service that handles actual message persistence and delivery

DNS resolution maps the queue domain name to one of multiple VIPs, which load balance traffic to FrontEnd servers. To achieve high availability and scale, multiple load balancers are deployed with primary-secondary failover and DNS-based A record partitioning.

The FrontEnd service, spread across data centers, handles:
- SSL termination (often via a local proxy)
- Authentication and authorization
- Parameter validation
- Request deduplication (to support idempotency)
- Rate limiting (typically via leaky bucket)
- Request routing
- Caching
- Usage metrics
- Server-side encryption of message payloads

The goal is to keep this layer as stateless and lightweight as possible, pushing heavy responsibilities to specialized services behind it. When a message is sent, the FrontEnd consults the Metadata service to identify backend storage responsibilities, then forwards the encrypted message to a selected backend node or cluster.

## Metadata Layer and Queue Configuration

The Metadata service maintains configuration and metadata for each queue, such as creation time, owner, and access controls. It is backed by a persistent store and includes an in-memory caching layer for read efficiency.

There are three caching strategies:
1. Fully replicated cache (all nodes hold the same data)
2. Sharded cache with FrontEnd-aware routing (FrontEnd knows which shard to contact)
3. Sharded cache with indirection (FrontEnd queries any Metadata host, which then reroutes to the correct shard)

The latter two options rely on consistent hashing to distribute metadata and balance the load. This Metadata service is consulted for each message operation to ensure correct backend routing and enforcement of queue semantics.

## Backend Layer and Message Storage

The backend service is the heart of the message queue system. It handles message ingestion, replication, persistence, visibility timeout management, and cleanup.

Two key architectural models are considered:

### Model 1: Single Leader
Each queue has a single leader backend host responsible for both send and receive operations, and that leader handles replication to followers and message cleanup. This requires a cluster-level coordination service, called In-cluster Manager, to maintain mappings between queues, leaders, and followers. It also handles leader election, failure detection, and rebalancing when instances join or leave.

### Model 2: Leaderless
This model avoids explicit leaders by organizing backend instances into small clusters. Queues are mapped to entire clusters, and requests can go to any instance, which replicates messages internally. This requires an Out-cluster Manager to maintain mappings between queues and clusters, and to monitor cluster health, utilization, and partitioning for oversized queues.

Large queues may be sharded across leaders or clusters to spread the load. In either model, replication can be synchronous (wait for all copies before acknowledging producer) or asynchronous (acknowledge once written to one node), each with its trade-offs in latency and durability.

## Message Lifecycle and Visibility

Message deletion strategies vary based on use cases:
- One option delays deletion until a consumer explicitly confirms processing (e.g., via deleteMessage API)
- Another is based on visibility timeout, where consumed messages become temporarily invisible and reappear if not confirmed within the timeout (used in Amazon SQS)
- Alternatively, Apache Kafka retains all messages and relies on the consumer to track offsets

These differences impact delivery semantics:
- At-most-once (possible loss, never duplicated)
- At-least-once (no loss, but possible duplication)
- Exactly-once (no loss or duplication, but harder to guarantee in practice due to failures and retries at multiple stages)

Most distributed queues settle for at-least-once delivery as a practical trade-off. Ordering is another concern; strict FIFO across a distributed system is costly, so many implementations either avoid it or provide only partition-level ordering to balance throughput.

## Scalability and Durability Considerations

The system is built to scale horizontally across all layers. Load balancers, FrontEnd instances, Metadata shards, and backend clusters can each scale independently. Partitioning strategies and replication mechanisms ensure both high availability and throughput.

Backend hosts use memory for recent messages and local disk for durability. For long-term retention or compliance, data may be backed by persistent storage. Queue-to-cluster assignments can be rebalanced as usage patterns evolve. Hot queues (those with large message volume) may be dynamically partitioned to spread the load and avoid bottlenecks.

## Security and Observability

Communication between components and with clients is secured via SSL/TLS. Messages are encrypted at rest immediately upon ingestion at the FrontEnd layer. Proper access control ensures that only authorized producers and consumers can interact with specific queues.

Observability includes metric emission and structured logging at every layer. Monitoring systems ingest these logs and metrics to provide real-time health dashboards and alerts. In addition, queue clients should have visibility into the state of their queues (e.g., message count, delivery lag), enabling them to tune consumption rates or scale consumers.

Key metrics include:
- Message ingress rate
- Processing latency
- Replication lag
- Error counts

These are vital for both system operators and clients to ensure smooth operation.

## Advanced Features and Design Choices

The system supports both push and pull models for message delivery:
- In the pull model, consumers poll for messages periodically
- In the push model, backend services notify consumers when new messages arrive

While push is more efficient from the consumer's standpoint, pull is simpler to implement and provides better control in high-throughput environments.

Additionally, queue creation can be explicit (via an API call) or implicit (auto-created upon first use). Deletion is often restricted or available only via secure CLI interfaces to prevent accidental data loss.

Each component is designed for fault tolerance, with failover and heartbeat-based health checks to ensure that the system can sustain partial failures and continue processing messages.

## Evaluation

The resulting architecture meets the primary non-functional requirements:
- It is scalable (by design, via sharded and replicated components)
- Highly available (multi-data-center redundancy)
- Performant (through caching, asynchronous flows, and distributed routing)
- Durable (via configurable replication and disk persistence)

Each component has a clearly defined responsibility and is designed for modular growth. The system can support massive throughput and varied use cases, from real-time log ingestion to decoupling microservices in complex architectures.

## Applications

Distributed message queues are a critical component in many modern systems. They serve as:
- The glue between services in microservice architectures
- The foundation for log and event pipelines
- Building blocks for eventual consistency and retries

This design is suitable for both internal infrastructure (e.g., task scheduling, audit trails) and external APIs (e.g., cloud queues like AWS SQS, Google Pub/Sub). It forms the backbone of fault-tolerant, loosely coupled systems at scale.




# Notification Service System Design

## Overview

To build a notification service that delivers messages from publishers to subscribers in response to events, like credit card transaction alerts or API fault notifications, we'll design a system that's scalable, highly available, fast, and durable. 

The service will support three core APIs:
- Creating topics (named buckets for messages)
- Publishing messages to topics
- Subscribing to topics to receive messages

## System Architecture

### Frontend Layer

We'll begin with a load balancer to evenly distribute requests from publishers and subscribers to a FrontEnd service, which handles:
- Request validation
- Authentication
- Authorization
- SSL termination
- Caching
- Throttling
- Deduplication
- Usage data collection

To keep the FrontEnd lightweight and responsive, we'll offload log processing to separate agents that aggregate and transfer logs asynchronously for monitoring and auditing, ensuring the service remains robust under high load.

### Metadata Service

Metadata about topics and subscriptions will be stored in a Metadata service, a distributed cache that shields the database (e.g., PostgreSQL) and provides fast access to topic and subscriber details.

The Metadata service partitions data across hosts using a consistent hashing ring, with FrontEnd hosts computing an MD5 hash (based on topic name and owner ID) to locate the appropriate Metadata host.

To manage host discovery, we'll consider two options:
1. **Configuration service** - Tracks Metadata hosts via heartbeats and maps hash ranges
2. **Gossip protocol** - FrontEnd hosts dynamically discover Metadata hosts through peer-to-peer data sharing

The Gossip protocol avoids a coordinator's single point of failure, enhancing availability, while both approaches scale to support millions of topics, making this a great discussion point with the interviewer.

### Temporary Storage Service

Messages published to topics will be held in a Temporary Storage service, designed to store messages briefly when subscribers are available or for days if retries are needed due to subscriber unavailability.

To meet demands for speed, scalability, high availability, and persistence, we'll explore multiple storage options:

1. **NoSQL key-value or column store** like Apache Cassandra or Amazon DynamoDB is suitable, as we don't need:
   - ACID transactions
   - Complex queries
   - Document/graph capabilities
   - Messages are small (e.g., <1 MB)

2. **In-memory store** like Redis with persistence could provide low latency

3. **Distributed message queue** like Apache Kafka or Amazon Kinesis could leverage partitioning and replication for scalability and durability

Kafka, for instance, aligns well with our needs, and discussing these options—databases, in-memory stores, or message queues—shows depth and allows us to weigh trade-offs with the interviewer.

### Sender Service

The Sender service will retrieve messages from Temporary Storage and deliver them to subscribers, using a thread pool to read messages efficiently.

To avoid overwhelming Temporary Storage, we'll use semaphores to dynamically adjust the number of retrieval threads based on idle threads and desired read rates, helping Temporary Storage recover during performance issues.

After retrieving a message, the Sender queries the Metadata service for subscriber details (e.g., HTTP endpoints, email addresses) instead of storing them with the message, preventing Temporary Storage from bloating and avoiding the need for a document database.

#### Task Creator and Executor

For delivering messages to multiple subscribers in parallel, we'll use Task Creator and Executor components:
- **Task Creator** splits delivery into individual tasks per subscriber
- **Executor** manages these tasks with a thread pool (e.g., Java's ThreadPoolExecutor), using semaphores to control execution

If threads are unavailable, tasks are deferred, and the message is returned to Temporary Storage for another Sender host to process, ensuring resilience against slow hosts.

## Delivery Guarantees and Policies

To guarantee at-least-once delivery, the Sender will retry failed deliveries for hours or days until successful or a retry limit is reached, with subscribers able to define retry policies or redirect undelivered messages to a monitoring system for manual review.

To prevent spam, subscribers must confirm subscriptions via a confirmation message to their endpoint or email.

The FrontEnd service deduplicates publisher submissions, but subscribers must handle potential duplicates from retries caused by network or internal issues.

The system doesn't guarantee message order, as delivery tasks run independently, and retries or slow Sender hosts may disrupt sequence numbers or timestamps.

## Security and Monitoring

Security is prioritized with:
- Authentication for publishers
- Registration for subscribers
- SSL encryption for messages in transit and at rest

Monitoring will track:
- Service health (e.g., request latency, delivery failures)
- Customer metrics (e.g., pending messages)

This integrates with a monitoring system for comprehensive visibility.

## System Features

### Aggregation
- The FrontEnd aggregates logs for monitoring
- The Metadata service caches topic and subscriber data
- The Sender processes messages via parallel tasks

### Scheduling
- Retry failed deliveries based on subscriber-defined policies or retry limits

### Edge Cases
- Handle duplicates (FrontEnd deduplication, subscriber-side retry handling)
- Slow subscribers (task isolation)
- High throughput (scale Kafka and Sender)
- Unavailable subscribers (persistent storage with retries)
- Large subscriber lists (Metadata service caching)

### Monitoring
- Track request latency
- Delivery success rate
- Temporary Storage performance
- Metadata cache hit rate
- Customer metrics like messages awaiting delivery

## Trade-offs

- **Kafka vs. RabbitMQ**: Kafka scales better than RabbitMQ for high-throughput messaging but is more complex to manage
- **Cassandra/DynamoDB vs. Redis**: Cassandra or DynamoDB provides durable, scalable storage, while Redis offers low latency but requires persistence for durability
- **Gossip Protocol vs. Configuration Service**: The Gossip protocol for Metadata host discovery is decentralized but complex, whereas a Configuration service is simpler but introduces a coordinator
- **Task-based vs. Sequential Delivery**: Task-based delivery isolates slow subscribers but adds complexity compared to sequential delivery
- **Single Queue Architecture**: A single Kafka-based queue could simplify the architecture but may need tuning for massive subscriber lists

## Why Kafka + Task-Based Sender?

Kafka provides scalable, durable message storage and ingestion, while the task-based Sender ensures efficient, parallel delivery to large subscriber groups, meeting the notification service's requirements.

## Scalability Estimates

- Supports ~1M messages/sec
- ~1B daily messages
- ~10K QPS for topic/subscription operations with Kafka, Sender, and database scaling

## Latency/Throughput Targets

- <100ms for message ingestion
- <1s for delivery to available subscribers
- <50ms for metadata queries
- ~1M messages/sec processing

This design is adaptable to other fan-out data delivery systems, such as event-driven architectures or real-time alerting, making it a robust solution for system design interviews. All critical details from the document are covered, including the architecture, components, non-functional requirements (scalability, availability, performance, durability), and edge cases like duplicates, retries, and security.




# Distributed Cache System Design

**Improvements:** To design a distributed cache that accelerates data retrieval for web applications by storing frequently accessed data in memory, we’ll create a system that’s fast, scalable, and highly available, minimizing calls to a slower backing data store like a database or web service. The cache will support two core operations: put (store a key-value pair) and get (retrieve a value by key), with both keys and values as strings for simplicity. We’ll start with a load balancer to distribute client requests across a cache cluster, where each host stores a shard of the data, allowing us to handle large datasets that exceed a single machine’s memory. To select the appropriate cache shard, we’ll use consistent hashing, which maps keys to a logical circle and assigns them to hosts based on hash ranges, minimizing key rehashing when hosts are added or removed.

For the cache’s core implementation, we’ll use a least recently used (LRU) eviction policy, combining a hash table for O(1) key-value lookups and a doubly linked list to track usage order, evicting the least recently used item when the cache is full. Each cache host runs this LRU cache as a standalone process, accessible via TCP or UDP, and a cache client library integrated into service code handles shard selection and request routing. To ensure high performance, the LRU cache uses constant-time operations, and consistent hashing enables fast shard selection (O(log n) via binary search). To scale to high request volumes, we’ll add more cache hosts, each responsible for a subset of keys, and use memory-optimized hardware in public clouds for larger datasets. To address hot shards (shards receiving disproportionate requests), we’ll introduce master-slave replication, designating a master node per shard for put operations and read replicas for get operations, spreading load across nodes and data centers.

To enhance availability, replicas will reside in different data centers, ensuring data access during network partitions or data center failures. A Configuration service, like ZooKeeper or Redis Sentinel, will monitor cache hosts via heartbeats, manage leader election, and handle failover by promoting a replica to master if the master fails. The Configuration service also maintains a dynamic list of cache hosts, which cache clients poll to stay updated, avoiding manual file updates or static configurations that require redeployment. To prevent data loss, we’ll use asynchronous replication to prioritize performance, accepting rare data loss (treated as cache misses) since the backing data store remains the source of truth. For staleness, we’ll add a time-to-live (TTL) attribute to cache entries, with passive expiration (removing items when accessed and found expired) or active expiration (using a maintenance thread with probabilistic sampling to avoid scanning billions of items).

To simplify integration, the cache client can embed a local LRU cache (e.g., using Google Guava) to reduce distributed cache calls, hiding complexity from service teams. For security, we’ll restrict cache access to trusted clients via firewalls, avoiding direct internet exposure, and optionally encrypt data before storing (with performance trade-offs). Monitoring will track cache hits/misses, latency, faults, CPU/memory usage, and network I/O, with lightweight logging capturing request details (e.g., key, status code) for debugging and auditing.

**Aggregation:** The cache client aggregates local and distributed cache operations, while the Configuration service centralizes host discovery and monitoring.  
**Scheduling:** Active expiration threads run periodically to clean stale items, and the Configuration service processes heartbeats for host health checks.  
**Edge Cases:** Handle hot shards (add replicas), unavailable hosts (treat as cache misses), inconsistent client views (synchronize host lists via Configuration service), stale data (TTL with expiration), and high throughput (scale shards and replicas).  
**Monitoring:** Track cache hit/miss rates, request latency, shard load distribution, host health, and resource utilization.

**Trade-offs:** Consistent hashing minimizes key rehashing but risks uneven key distribution or domino effects (mitigated by adding virtual nodes or using Jump Hash). Asynchronous replication boosts performance but risks data loss, while synchronous replication ensures consistency but increases latency. A dedicated cache cluster isolates resources but raises costs, whereas co-located caches save hardware but compete for service resources. A Configuration service automates host discovery but adds complexity, unlike static files that are simpler but inflexible. Memcached is simple and fast but lacks replication, while Redis with Sentinel offers availability but is more complex.

**Why Consistent Hashing + LRU Cache?:** Consistent hashing ensures scalable, low-disruption sharding, and the LRU cache provides fast, memory-efficient data storage, meeting the performance and scalability needs of a distributed cache.  
**Scalability Estimates:** Supports ~1M requests/sec, ~1B daily operations, and ~10K QPS for cache operations with shard and replica scaling.  
**Latency/Throughput Targets:** Target <1ms for cache hits, <10ms for shard selection and network calls, and ~1M requests/sec processing.

This design is versatile, applicable to systems like distributed message queues, notification services, or rate limiters, making it a robust foundation for system design interviews. I’ve ensured all critical details from the document are included, covering the LRU algorithm, consistent hashing, replication, Configuration service, and trade-offs, with no key points missed. If you have specific concerns or need further clarification, please let me know!



