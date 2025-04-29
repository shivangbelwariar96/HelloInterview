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



# System Design Cheatsheet

This cheatsheet covers key system design concepts, data structures, and real-world applications, focusing on scalability, latency, and trade-offs. It includes probabilistic data structures, read-heavy systems, write-heavy systems, strong consistency systems, trie/proximity systems, aggregation systems, and distributed systems. Each section provides improvements, examples, and metrics, with added CAP theorem context, failure modes, cost considerations, and security best practices. A glossary and new sections (Distributed File System, Distributed Lock Manager, Event Sourcing System, API Gateway) enhance completeness.

## Glossary
- **Consistent Hashing**: Distributes data across nodes to minimize rehashing during scaling.
- **Fanout**: Distributing data (e.g., posts) to multiple destinations (e.g., followers).
- **Sharding**: Partitioning data across multiple servers for scalability.
- **CAP Theorem**: A system can prioritize only two of Consistency, Availability, and Partition Tolerance.
- **Eventual Consistency**: Non-critical updates may propagate with delay.
- **Strong Consistency**: All reads reflect the latest write.
- **Geohashing**: Encoding locations into strings for efficient geospatial queries.
- **Idempotency**: Ensuring repeated operations produce the same result.

## Probabilistic Data Structures

### 1. Bloom Filter
**What**: Probabilistic structure testing element presence in set.

**Does**: Checks "Is element in set?" with false positives, no false negatives.

**How**: Bit array (size \( m \)), \( k \) hash functions. Add: hash element, set bits to 1. Check: hash, verify all bits 1. False positives from shared bits; no false negatives. False positive probability: \( (1 - e^{-kn/m})^k \).

**Pros**: Space-efficient, fast O(k) lookup/insert, no false negatives.

**Cons**: False positives (depends on \( m \), \( k \), #elements), no deletions (standard), no element retrieval.

**Variants**: Counting Bloom Filters use counters to support deletions.

**Examples**:
- Google Bigtable: Cuts disk lookups for missing rows/columns.
- Apache Cassandra: Checks SSTables pre-disk access.
- CDNs (Akamai): Caches URL checks, reduces redundant requests.
- Malware detection: Google Safe Browsing for URL blacklist.
- Bitcoin SPV nodes: Filters transactions relevant to wallets.

**CAP Context**: AP (prioritizes availability, eventual consistency for updates).

**Failure Modes**: Hash collisions increase false positives; mitigated by tuning \( m \) and \( k \).

**Cost Considerations**: Low memory footprint; scales well but requires tuning to balance false positives and storage.

**Security**: Ensure hash functions are collision-resistant (e.g., MurmurHash).

### 2. Count-Min Sketch
**What**: Probabilistic frequency estimator for data streams.

**Does**: Tracks approximate counts, bounded error.

**How**: 2D array \( d \times w \), \( d \) hash functions. Increment: hash, up cells. Query: hash, min across \( d \) cells. Overestimates from collisions, no underestimates. Error bound: \( \epsilon = e/w \), probability \( 1 - \delta \), where \( \delta = 1/e^d \).

**Pros**: Sublinear space, fast O(d) updates/queries, suits high-volume streams.

**Cons**: Overestimation from collisions, error tied to \( w \), \( d \), no easy deletions.

**Variants**: Count-Min-Delete tracks negative counts for deletions.

**Examples**:
- Network monitoring: Cisco tracks flow sizes.
- Ad tech: Google/Facebook estimate clicks/impressions.
- Streaming analytics: Kafka/Spark for approx counts.
- DDoS detection: Cloudflare tracks request frequency.
- Apache Flink: Real-time frequency counting in event streams.

**CAP Context**: AP (availability for writes, eventual consistency for counts).

**Failure Modes**: Collisions overestimate counts; mitigated by increasing \( w \) or \( d \).

**Cost Considerations**: Minimal memory; cost scales with array size and stream volume.

**Security**: Protect against adversarial inputs that exploit hash collisions.

### 3. HyperLogLog
**What**: Probabilistic cardinality estimator.

**Does**: Approximates unique items, high accuracy, low memory.

**How**: Hash items, count leading zeros, store max per register. Harmonic mean for total estimate. Standard error: \( 1.04 / \sqrt{m} \), where \( m \) is number of registers.

**Pros**: Space-efficient (~12 KB for billions), constant memory, fast O(1) ops.

**Cons**: Approx (~2% error), no exact counts/elements, merging needs parameter alignment.

**Variants**: HyperLogLog++ improves accuracy for small cardinalities.

**Examples**:
- Redis: Unique site visitors.
- Google Analytics: Tracks unique users/events.
- Social media: Twitter/Facebook count unique impressions.
- Databases: PostgreSQL/BigQuery for approx_distinct().
- Snowflake: Approximate distinct counts in analytics.

**CAP Context**: AP (availability for updates, eventual consistency for counts).

**Failure Modes**: Register saturation for very high cardinalities; mitigated by increasing \( m \).

**Cost Considerations**: Extremely low memory; cost-effective for large-scale cardinality estimation.

**Security**: Use robust hash functions to prevent manipulation.

### Summary Table
| Structure | Purpose | Mechanism | Pros | Cons | Uses | Error Rate |
|-----------|---------|-----------|------|------|------|------------|
| Bloom Filter | Membership testing | Bit array, multi-hashes | Space-efficient, fast, no false negatives | False positives, no deletions | Bigtable, Cassandra, CDNs, Safe Browsing, Bitcoin SPV | \( (1 - e^{-kn/m})^k \) |
| Count-Min Sketch | Frequency estimation | 2D array, min hashed cells | Sublinear space, fast, stream-friendly | Overestimation, approx | Network, ad tech, DDoS detection, Flink | \( \epsilon = e/w \) |
| HyperLogLog | Cardinality estimation | Hash-based leading zeros | Minimal memory, fast, scalable | Approx, no exact counts | Redis, Analytics, social media, Snowflake | \( 1.04 / \sqrt{m} \) |

**Trade-offs**: These structures trade accuracy for efficiency, ideal for large-scale systems prioritizing performance over exactness.

## Read-Heavy Systems
Focus: low-latency retrieval, caching, scalability.

### 1. URL Shortener (Bitly)
**Improvements**:
- **Key Gen**: Base62 encoding (7 chars ~\( 62^7 \approx 3.5T \) URLs). Redis counter or SHA-256 hash (truncated) for uniqueness.
- **Collisions**: Bloom Filter checks existing shorts. Append suffix or retry on clash.
- **DB**: DynamoDB for high throughput: {short_url, long_url, creation_time, expiration_time, user_id}.
- **Redirect**: 302 (temporary).
- **Caching**: Redis for hot URLs (LRU). Cache short-to-long, long-to-short mappings.
- **Sharding**: By short_url prefix, not user_id (avoids hotspots).
- **Rate Limiting**: Token bucket in Redis to prevent creation abuse.
- **Analytics**: Track clicks via Kafka, aggregate with Spark.
- **Edge Cases**: Expired URLs (TTL), malicious URLs (blocklist), custom aliases, rate-limited requests, invalid URLs, URL validation for malformed/phishing links.
- **Monitoring**: Redirect latency, cache hit ratio, collision rate, RPS, click analytics.

**Architecture Diagram**: Client -> CDN -> Load Balancer -> API (Redis cache, DynamoDB) -> Kafka (analytics).

**Trade-offs**:
- Hashing vs Counter: Hashing risks collisions; counter ensures uniqueness.
- SQL vs NoSQL: NoSQL scales better for writes.
- In-Memory vs Persistent: Redis fast, DynamoDB durable.

**Why Counter + NoSQL?** Unique keys, horizontal scaling.

**Scalability**: ~10K QPS redirects, ~1B URLs, ~100M DAU.

**Latency**: <50ms redirect, <100ms creation.

**CAP Context**: AP (availability for redirects, eventual consistency for analytics).

**Failure Modes**: Cache misses increase latency; mitigated by pre-warming Redis. DynamoDB outages; mitigated by multi-region replication.

**Cost Considerations**: DynamoDB costly for high writes; optimize with reserved capacity. Redis cost scales with memory.

**Security**: HTTPS, URL validation, rate limiting, blocklist for malicious URLs.

### 2. Image Hosting Service
**Improvements**:
- **Storage**: S3/GCS for durability. Metadata (image_id, user_id, upload_time) in PostgreSQL.
- **CDN**: Cloudflare/Akamai for global caching.
- **Deduplication**: Perceptual hash for near-duplicates.
- **Resizing**: Pre-generate sizes (Lambda + SQS) for fast delivery.
- **Compression**: Use WebP/JPEG XL to reduce storage/bandwidth.
- **Content Moderation**: AWS Rekognition for inappropriate images.
- **Edge Cases**: Large uploads (multipart), corrupted images, CDN invalidation, bandwidth abuse, metadata inconsistencies, partial uploads, image format validation.
- **Security**: Signed URLs for private images, rate-limit uploads.
- **Monitoring**: Upload success, CDN hit ratio, storage costs, retrieval latency.

**Architecture Diagram**: Client -> CDN -> API (PostgreSQL metadata, S3 storage) -> Lambda (resizing) -> Rekognition (moderation).

**Trade-offs**:
- Pre-resize vs On-the-fly: Pre-resize saves compute, ups storage.
- Perceptual vs Exact Hash: Perceptual catches near-duplicates, slower.
- S3 vs Custom: S3 managed, costly; custom cheaper, complex.

**Why Pre-resize + S3/CDN?** Fast delivery, durability, low latency.

**Scalability**: ~5K uploads/sec, ~10M DAU, ~1PB storage.

**Latency**: <100ms retrieval, <500ms uploads, ~5K req/sec downloads.

**CAP Context**: CP (consistency for metadata, availability via CDN).

**Failure Modes**: S3 outages; mitigated by multi-region buckets. Metadata inconsistencies; mitigated by transactional writes.

**Cost Considerations**: S3 storage costs high; optimize with lifecycle policies. CDN reduces egress costs.

**Security**: Signed URLs, encryption at rest, content moderation.

### 3. Social Media (Twitter/Facebook)
**Improvements**:
- **Posts**: Cassandra: {post_id, user_id, content, timestamp}. Time-based partitioning.
- **Timelines**: Hybrid push-pull: push to active users (Redis), pull for inactive. Pull-only for high-follower users (celebrities).
- **Relationships**: Graph DB (Neo4j) or Redis adjacency lists.
- **Denormalization**: Precompute timelines, incremental updates.
- **Sharding**: Posts by post_id, timelines by user_id. Consistent hashing.
- **Content Ranking**: ML models for engagement scoring.
- **Ingestion**: Kafka for high write throughput.
- **Edge Cases**: Deleted posts (tombstones), privacy (filter on read), viral posts (cache), user bans, timeline staleness, spam detection, content moderation.
- **Monitoring**: Timeline latency, fanout throughput, graph query perf, post retrieval success.

**Architecture Diagram**: Client -> CDN -> API (Cassandra posts, Redis timelines, Neo4j relationships) -> Kafka (ingestion) -> ML (ranking).

**Trade-offs**:
- Push vs Pull: Push fast, costly for influencers; hybrid balances.
- Graph DB vs Lists: Lists faster for simple lookups.
- Cassandra vs DynamoDB: Cassandra suits high writes.

**Why Hybrid + Cassandra?** Optimizes active/inactive users, high write throughput.

**Scalability**: ~100M DAU, ~1B posts/day, ~50K QPS timelines.

**Latency**: <200ms timeline, <100ms post retrieval, ~10K posts/sec.

**CAP Context**: AP (availability for reads, eventual consistency for likes/comments).

**Failure Modes**: Cassandra outages; mitigated by replication. Timeline staleness; mitigated by periodic pulls.

**Cost Considerations**: Cassandra cost-effective for writes; Redis memory costs scale with timelines.

**Security**: HTTPS, content moderation, spam filtering, privacy controls.

### 4. NewsFeed System
**Improvements**:
- **Push vs Pull**: Fanout-on-write for active, fanout-on-read for inactive.
- **Caching**: Redis for recent posts, Memcached for timelines.
- **Pagination**: Cursor-based (timestamp + post_id, tie-break with post_id).
- **Ranking**: Weighted scoring (likes, recency) via Spark, cached in Redis.
- **Large Followers**: Flag high-follow users, skip precompute, merge on read.
- **Personalization**: ML models for user-specific ranking.
- **Ingestion**: Kafka for posts.
- **Search**: Elasticsearch for post search.
- **Edge Cases**: Stale feeds, deleted posts, skewed activity, privacy filters, cache inconsistencies, feed deduplication, user segmentation.
- **Monitoring**: Feed refresh latency, ranking accuracy, cache eviction, feed success.

**Architecture Diagram**: Client -> API (Redis/Memcached cache, Cassandra posts) -> Kafka (ingestion) -> Spark (ranking) -> Elasticsearch (search).

**Trade-offs**:
- Fanout-on-Write vs Read: Write fast for active, read for inactive.
- Real-time vs Batch Ranking: Batch efficient, periodic updates.
- Redis vs Memcached: Both for complementary caching.

**Why Hybrid + Redis?** Cuts write load, low-latency feeds.

**Scalability**: ~50M DAU, ~500M posts/day, ~20K QPS feeds.

**Latency**: <150ms feed retrieval, <500ms ranking, ~5K feeds/sec.

**CAP Context**: AP (availability for feeds, eventual consistency for rankings).

**Failure Modes**: Cache evictions cause latency spikes; mitigated by pre-warming. Kafka lag; mitigated by consumer scaling.

**Cost Considerations**: Redis/Memcached memory costs; optimize with eviction policies. Spark for ranking adds compute cost.

**Security**: Privacy filters, HTTPS, rate limiting.

### 5. Pastebin
**Improvements**:
- **Storage**: DynamoDB: {paste_id, content, creation_time, expiration_time, user_id}. S3 for large pastes.
- **Key Gen**: Base62 (6-8 chars) via ZooKeeper counter or content hash.
- **Caching**: Redis for hot pastes (LRU).
- **Expiration**: TTL in Redis/DB. DynamoDB Streams for cleanup.
- **Access Control**: Signed URLs for private pastes.
- **Syntax Highlighting**: Server-side rendering with Pygments.
- **Versioning**: Support editable pastes with version history.
- **Content Moderation**: Filter malicious code.
- **Edge Cases**: Duplicates (perceptual hash dedup), malicious content, high QPS (CDN), large pastes (multipart), expired revival, paste size limits (10MB), abuse prevention.
- **Monitoring**: Creation rate, cache hit, expiration cleanup, retrieval latency.

**Architecture Diagram**: Client -> CDN -> API (DynamoDB metadata, S3 content, Redis cache) -> Pygments (highlighting).

**Trade-offs**:
- Counter vs Hashing: Counter ensures uniqueness, hashing risks collisions.
- NoSQL vs S3: DynamoDB for metadata, S3 for content.
- Redis vs CDN: Both for optimal perf.

**Why Counter + DynamoDB/S3?** Unique IDs, fast metadata, scalable content.

**Scalability**: ~1K creations/sec, ~10M DAU, ~100K QPS reads.

**Latency**: <50ms retrieval, <200ms creation, ~2K pastes/sec.

**CAP Context**: CP (consistency for metadata, availability via S3).

**Failure Modes**: S3 latency for large pastes; mitigated by CDN. DynamoDB throttling; mitigated by reserved capacity.

**Cost Considerations**: S3 storage costs; use lifecycle policies. DynamoDB write costs; optimize with batching.

**Security**: Signed URLs, content moderation, rate limiting.

### 6. Instagram
**Improvements**:
- **Media Storage**: S3 for media, Cassandra for metadata: {post_id, user_id, timestamp}.
- **Feed Gen**: Hybrid push-pull: push to active followers (Redis), pull for inactive. Pull-only for celebrities (>100K followers).
- **Caching**: Redis + CDN for feeds/media. Smart caching for hot content.
- **Sharding**: Posts by post_id, timelines by user_id, consistent hashing.
- **Low Latency**: Hybrid fanout: write for <100K followers, read for celebs. Redis Cluster for durability.
- **Chronological Sort**: Query follows (DynamoDB: followerId, followedId), posts (userId, createdAt+postId).
- **Stories**: Ephemeral posts with Redis TTL.
- **Recommendations**: ML-based explore feeds.
- **Ingestion**: Kafka for posts.
- **Search**: Elasticsearch for hashtags.
- **Edge Cases**: Deleted posts, privacy, viral content, media corruption, feed inconsistencies, content filtering, rate limiting.
- **Monitoring**: Feed latency, media load time, cache hit, post retrieval success.

**Architecture Diagram**: Client -> CDN -> API (Cassandra metadata, S3 media, Redis timelines) -> Kafka (ingestion) -> ML (recommendations) -> Elasticsearch (search).

**Trade-offs**:
- Push vs Pull: Hybrid balances fast reads, scalable writes.
- Cassandra vs DynamoDB: Cassandra for high writes.
- S3 vs Custom: S3 for reliability.

**Why Hybrid + Cassandra?** Optimizes active/inactive users, high write throughput.

**Scalability**: ~100M DAU, ~1B posts/day, ~50K QPS feeds.

**Latency**: <200ms feed, <100ms media, ~10K posts/sec.

**CAP Context**: AP (availability for feeds, eventual consistency for likes).

**Failure Modes**: Cassandra outages; mitigated by replication. CDN failures; mitigated by fallback to S3.

**Cost Considerations**: S3 storage costs; optimize with compression. Cassandra cost-effective for writes.

**Security**: HTTPS, content filtering, rate limiting, privacy controls.

### 7. Dropbox
**Improvements**:
- **Storage**: S3 for files, PostgreSQL for metadata: {file_id, user_id, version, path}. Presigned URLs for uploads.
- **Downloading**: CDN for global caching, signed URLs for secure downloads.
- **Sharing**: SharedFiles table links userId to fileId.
- **Versioning**: Store deltas in S3, version history in DB. Delta encoding for bandwidth efficiency.
- **Syncing**: WebSockets for real-time, Kafka for events, or polling.
- **Fast Ops**: CDN for downloads, chunking for uploads, compression (Gzip, Brotli).
- **Large Files**: Chunk into 5-10MB, track in FileMetadata, resumable uploads via S3 multipart.
- **Security**: HTTPS, S3 encryption, signed URLs, access via shareList, client-side encryption.
- **Deduplication**: Content hashing to save storage.
- **Sync Processing**: Kafka Streams for event processing.
- **Edge Cases**: Concurrent edits, large files, offline access, corruption, quotas, file conflicts, bandwidth throttling.
- **Monitoring**: Sync latency, storage usage, dedup rate, upload success.

**Architecture Diagram**: Client -> CDN -> API (PostgreSQL metadata, S3 files) -> WebSockets/Kafka (sync) -> Kafka Streams (processing).

**Trade-offs**:
- S3 vs Custom: S3 managed, costly; custom cheaper, complex.
- PostgreSQL vs NoSQL: PostgreSQL for relations.
- WebSockets vs Polling: WebSockets for UX.

**Why S3 + PostgreSQL?** Durable storage, relational metadata.

**Scalability**: ~10M DAU, ~1PB storage, ~1K uploads/sec.

**Latency**: <500ms sync, <1s uploads, ~1K files/sec.

**CAP Context**: CP (consistency for metadata, availability via S3).

**Failure Modes**: S3 outages; mitigated by multi-region buckets. Sync conflicts; mitigated by conflict resolution.

**Cost Considerations**: S3 storage costs; optimize with lifecycle policies. PostgreSQL costs; use read replicas for scale.

**Security**: Client-side encryption, signed URLs, access controls.

### 8. YouTube/Netflix
**Improvements**:
- **Storage**: S3 for videos, DynamoDB for metadata: {video_id, title, uploader_id}. HLS/DASH for streaming.
- **Watching**: Adaptive bitrate: fetch manifest, select segments by network. CDNs cache manifests and segments.
- **CDN**: Akamai/Cloudflare for global caching.
- **Processing**: DAG pipeline (Temporal): split, transcode, generate manifests.
- **Resumable Uploads**: Chunk videos, track in VideoMetadata, resume via S3 events.
- **Recommendation**: Spark for collab filtering, cache in Redis.
- **Encoding**: H.264, AV1 for compression.
- **DRM**: Digital Rights Management for piracy prevention.
- **Ingestion**: Kafka for view events.
- **Analytics**: Flink for real-time view trends.
- **Sharding**: Metadata by video_id, users by user_id.
- **Edge Cases**: Buffering, geo-restrictions, high demand, corruption, stale recs, playback failures, geo-fencing.
- **Monitoring**: Streaming latency, rec CTR, CDN hit, playback success.

**Architecture Diagram**: Client -> CDN -> API (DynamoDB metadata, S3 videos) -> Kafka (events) -> Spark/Flink (recs/analytics).

**Trade-offs**:
- Pre-compute vs Real-time Recs: Pre-compute efficient, periodic updates.
- DynamoDB vs Cassandra: DynamoDB simpler, costlier.
- S3 vs Custom: S3 reliable, costly.

**Why S3 + CDN?** Durable storage, low-latency delivery.

**Scalability**: ~1B DAU, ~10PB storage, ~100K QPS streaming.

**Latency**: <200ms start, <500ms recs, ~50K streams/sec.

**CAP Context**: AP (availability for streaming, eventual consistency for recs).

**Failure Modes**: CDN outages; mitigated by fallback to origin. Buffering; mitigated by adaptive bitrate.

**Cost Considerations**: S3 storage and CDN egress costs; optimize with compression. DynamoDB read/write costs; use caching.

**Security**: DRM, HTTPS, geo-fencing, rate limiting.

### 9. Yelp/Nearby Friends
**Improvements**:
- **Geospatial Index**: Redis Geo or PostGIS for {business_id, lat, lon}. Geohashing for query optimization.
- **Search**: Elasticsearch for text/geo searches.
- **Caching**: Redis for popular searches.
- **Reviews**: PostgreSQL for reviews, precompute averages.
- **Recommendations**: ML-based collaborative filtering.
- **Ingestion**: Kafka for reviews.
- **Edge Cases**: Spoofing, stale data, high QPS, privacy, inaccurate coords, location spoofing detection, business verification.
- **Ratings**: Precompute averages incrementally.
- **One-Review**: Unique constraint on userId, businessId.
- **Complex Queries**: Quadtree indexing, Haversine for distance.
- **Location Names**: Precompute identifiers, inverted index.
- **Monitoring**: Search latency, geo-query accuracy, cache hit, search success.

**Architecture Diagram**: Client -> API (Redis Geo, Elasticsearch search, PostgreSQL reviews) -> Kafka (ingestion) -> ML (recommendations).

**Trade-offs**:
- Redis Geo vs PostGIS: Redis faster for simple queries.
- Elasticsearch vs DB: Elasticsearch optimized for search.
- Caching vs No Caching: Caching with TTL for balance.

**Why Redis + Elasticsearch?** Fast geo-queries, complex searches.

**Scalability**: ~10M DAU, ~100K QPS searches, ~1M businesses.

**Latency**: <100ms geo-queries, <200ms search, ~5K searches/sec.

**CAP Context**: AP (availability for searches, eventual consistency for reviews).

**Failure Modes**: Elasticsearch latency; mitigated by caching. Location spoofing; mitigated by IP validation.

**Cost Considerations**: Elasticsearch compute costs; optimize with reserved instances. Redis memory costs; use TTLs.

**Security**: HTTPS, location spoofing detection, business verification.

## Write-Heavy Systems
Focus: high throughput, durability, ingestion speed.

### 10. Rate Limiter
**Improvements**:
- **Algorithm**: Token bucket for bursts over leaky bucket. Redis: {user_id, tokens, last_refill}.
- **TTL**: Redis TTL matches bucket refresh.
- **Distributed**: Redis cluster, Lua scripts for atomicity.
- **Negative Tokens**: Allow temp negatives for bursts.
- **Inter-Host Comm**: Gossip or shared cache for token sync.
- **Memory**: Drop inactive buckets post-timeout.
- **Rules**: Tool for teams to set limits.
- **Sliding Window**: Alternative using Redis sorted sets for finer granularity.
- **Hierarchical Limiting**: Per-user and per-IP limits.
- **Edge Cases**: Clock skew, bursts, DDoS, token exhaustion, misconfigs, multi-region sync, graceful degradation.
- **Monitoring**: Rejection rate, refill latency, Redis throughput, accuracy.

**Architecture Diagram**: Client -> API (Redis rate limiter) -> Backend (protected service).

**Trade-offs**:
- Token vs Leaky: Token allows bursts, suits APIs.
- Redis vs In-Memory: Redis durable, distributed.
- Global vs Per-User: Both for full control.

**Why Token Bucket + Redis?** Handles bursts, scales distributedly.

**Scalability**: ~100K QPS checks, ~1M users, ~10K limits/sec.

**Latency**: <10ms checks, ~50K checks/sec, <50ms refills.

**CAP Context**: CP (consistency for token counts, availability via replication).

**Failure Modes**: Redis outages; mitigated by fallback to in-memory. Clock skew; mitigated by synchronized clocks.

**Cost Considerations**: Redis memory costs; optimize with eviction. Lua scripts reduce network overhead.

**Security**: Rate limiting prevents DDoS, HTTPS for API.

### 11. Log Collection/Analysis
**Improvements**:
- **Ingestion**: Kafka for high-throughput logs, partitioned by source/timestamp.
- **Processing**: ELK stack (Elasticsearch, Logstash, Kibana). Spark for batch analytics.
- **Buffering**: Kafka for spikes, consumer groups for parallel processing.
- **Querying**: Time-based indices in Elasticsearch with rollover policies.
- **Enrichment**: Add metadata via Logstash.
- **Anomaly Detection**: ML-based alerting.
- **Alternative**: ClickHouse for high-speed analytics.
- **Compression**: Gzip for storage reduction.
- **Edge Cases**: Spikes, malformed logs, retention, consumer lag, duplicates, schema evolution.
- **Monitoring**: Ingestion latency, query time, partition lag, processing success.

**Architecture Diagram**: Source -> Kafka -> Logstash -> Elasticsearch -> Kibana/Spark (analytics).

**Trade-offs**:
- Kafka vs RabbitMQ: Kafka for high throughput.
- Elasticsearch vs ClickHouse: Elasticsearch for text search.
- Real-time vs Batch: Both for balance.

**Why Kafka + ELK?** Reliable ingestion, robust search/viz.

**Scalability**: ~1M logs/sec, ~1PB storage, ~10K QPS queries.

**Latency**: <100ms ingestion, <500ms queries, ~100K logs/sec.

**CAP Context**: AP (availability for ingestion, eventual consistency for queries).

**Failure Modes**: Kafka lag; mitigated by consumer scaling. Elasticsearch outages; mitigated by replication.

**Cost Considerations**: Kafka storage costs; optimize with retention policies. Elasticsearch compute costs; use reserved instances.

**Security**: HTTPS, log encryption, access controls.

### 12. Voting System
**Improvements**:
- **Idempotency**: Unique vote_id (user_id + poll_id) in Redis. Cleanup expired votes.
- **Fraud**: Rate-limiting, CAPTCHA. Log votes in PostgreSQL.
- **Aggregation**: Redis for real-time counts, Spark for final tallies.
- **Audit Logging**: Kafka for vote events.
- **Duplicate Check**: Bloom Filter for duplicate votes.
- **Edge Cases**: Reversals, network failures, disputes, impersonation, spikes, vote tampering, vote expiration.
- **Monitoring**: Throughput, fraud rate, count accuracy, latency.

**Architecture Diagram**: Client -> API (Redis counts, PostgreSQL votes) -> Kafka (audit) -> Spark (aggregation).

**Trade-offs**:
- Real-time vs Eventual: Real-time with periodic reconciliation.
- Redis vs DB: Redis for speed, DB for durability.
- Rate-Limiting vs CAPTCHA: Both for layered security.

**Why Redis + PostgreSQL?** Fast counting, durable storage.

**Scalability**: ~10K votes/sec, ~1M voters, ~100K QPS counts.

**Latency**: <50ms submission, <100ms counts, ~5K votes/sec.

**CAP Context**: CP (consistency for votes, availability via replication).

**Failure Modes**: Redis memory exhaustion; mitigated by cleanup. PostgreSQL contention; mitigated by indexing.

**Cost Considerations**: Redis memory costs; optimize with TTL. PostgreSQL storage; use partitioning.

**Security**: CAPTCHA, rate limiting, HTTPS, tamper detection.

### 13. Trending Topics
**Improvements**:
- **Counting**: Count-min sketch in Redis for approx mentions. Periodic exact counts via Spark.
- **Sliding Window**: Redis sorted sets for time-based tracking.
- **Ranking**: Background job for trending scores, cached in Redis.
- **Clustering**: NLP-based grouping of similar terms.
- **Geo-Trends**: Redis Geo for location-specific trends.
- **Ingestion**: Kafka for mentions.
- **Edge Cases**: Spikes, stale trends, low memory, ambiguity, skew, spam filtering, trend decay.
- **Monitoring**: Sketch accuracy, ranking latency, cache hit, refresh rate.

**Architecture Diagram**: Source -> Kafka -> Redis (count-min, sorted sets) -> Spark (exact counts) -> API (rankings).

**Trade-offs**:
- Count-Min vs Exact: Count-min for memory efficiency.
- Sliding vs Fixed: Sliding for real-time.
- Redis vs In-Memory: Redis for durability.

**Why Count-Min + Redis?** Scales high cardinality, low-latency updates.

**Scalability**: ~100K mentions/sec, ~1M topics, ~10K QPS rankings.

**Latency**: <50ms updates, <200ms rankings, ~10K trends/sec.

**CAP Context**: AP (availability for updates, eventual consistency for rankings).

**Failure Modes**: Count-min overestimation; mitigated by periodic exact counts. Redis outages; mitigated by replication.

**Cost Considerations**: Redis memory costs; optimize with eviction. Spark compute costs; use spot instances.

**Security**: Spam filtering, HTTPS, rate limiting.

### 14. Facebook Messenger
**Improvements**:
- **Ingestion**: Kafka partitioned by chat_id.
- **Storage**: Cassandra: {chat_id, message_id, content, timestamp}.
- **Delivery**: WebSockets for real-time, APNs/GCM for push. Fallback to polling for unreliable connections.
- **Encryption**: End-to-end with Signal Protocol.
- **Search**: Elasticsearch for full-text message search.
- **Group Chats**: Redis Pub/Sub for fanout.
- **Edge Cases**: Offline users, duplicates, group chats, expiration, failures, message retraction, read receipt syncing.
- **Monitoring**: Delivery latency, notification success, queue backlog, processing success.

**Architecture Diagram**: Client -> WebSockets -> API (Cassandra storage, Redis receipts) -> Kafka (ingestion) -> Elasticsearch (search).

**Trade-offs**:
- Kafka vs RabbitMQ: Kafka for durability, scale.
- WebSockets vs Polling: WebSockets for UX.
- Cassandra vs DynamoDB: Cassandra for cost-effective writes.

**Why Kafka + Cassandra?** Reliable ingestion, scalable storage.

**Scalability**: ~100M DAU, ~1B msgs/day, ~50K QPS delivery.

**Latency**: <100ms delivery, <200ms notifications, ~10K msgs/sec.

**CAP Context**: AP (availability for delivery, eventual consistency for receipts).

**Failure Modes**: Kafka lag; mitigated by consumer scaling. Cassandra outages; mitigated by replication.

**Cost Considerations**: Kafka storage costs; optimize with retention. Cassandra cost-effective for writes.

**Security**: End-to-end encryption, HTTPS, access controls.

### 15. Local Delivery (Gopuff)
**Improvements**:
- **Orders**: Kafka partitioned by region.
- **Inventory**: Redis for real-time, PostgreSQL for durability.
- **Routing**: OSRM for routes, cached in Redis.
- **Traffic-Aware Lookup**: Sync DC data, filter by drive time.
- **Orders (Consistency)**: SERIALIZABLE transaction in PostgreSQL. Optimistic locking for high concurrency.
- **Availability Lookup**: Cache results in Redis by item + DC.
- **Inventory Sync**: Kafka Streams for real-time updates.
- **Demand Forecasting**: ML-based stock predictions.
- **Driver Tracking**: Redis Geo for locations.
- **Edge Cases**: Out-of-stock, delays, cancellations, driver unavailability, traffic, order fraud, driver assignment failures.
- **Monitoring**: Order latency, inventory accuracy, ETA, success rate.

**Architecture Diagram**: Client -> API (Redis inventory, PostgreSQL orders) -> Kafka (orders) -> OSRM (routing) -> ML (forecasting).

**Trade-offs**:
- Redis vs DB: Redis for speed, DB for durability.
- Real-time vs Batch Routing: Real-time for UX.
- Kafka vs RabbitMQ: Kafka for high throughput.

**Why Kafka + Redis?** High-throughput orders, fast inventory checks.

**Scalability**: ~10K orders/sec, ~1M daily orders, ~100K QPS inventory.

**Latency**: <100ms inventory, <500ms routing, ~5K orders/sec.

**CAP Context**: CP (consistency for orders, availability via Redis).

**Failure Modes**: Redis outages; mitigated by PostgreSQL fallback. Kafka lag; mitigated by consumer scaling.

**Cost Considerations**: Redis memory costs; optimize with TTL. Kafka storage; use retention policies.

**Security**: HTTPS, fraud detection, access controls.

### 16. Strava
**Improvements**:
- **Ingestion**: Kafka for GPS/fitness data, partitioned by user_id.
- **Storage**: Cassandra: {activity_id, user_id, gps_points, timestamp}.
- **Analytics**: Spark for leaderboards, cache in Redis.
- **Offline Tracking**: Local storage, upload on completion. Timestamp-based conflict resolution.
- **Real-time Sharing**: Polling every 2-5s, smart buffering for smoothness.
- **Leaderboard**: Redis Sorted Sets for real-time rankings.
- **Social Features**: Redis for follower feeds.
- **Route Analysis**: ML-based segment detection.
- **Search**: Elasticsearch for activity search.
- **Edge Cases**: GPS errors, large uploads, privacy, duplicates, incomplete data, data validation, privacy controls.
- **Monitoring**: Ingestion latency, analytics accuracy, cache hit, processing success.

**Architecture Diagram**: Client -> API (Cassandra storage, Redis leaderboards) -> Kafka (ingestion) -> Spark (analytics) -> Elasticsearch (search).

**Trade-offs**:
- Cassandra vs DynamoDB: Cassandra for cost-effective writes.
- Real-time vs Batch: Batch with periodic updates.
- Kafka vs RabbitMQ: Kafka for high throughput.

**Why Kafka + Cassandra?** Reliable ingestion, scalable storage.

**Scalability**: ~1M DAU, ~10M activities/day, ~10K QPS analytics.

**Latency**: <200ms ingestion, <500ms analytics, ~5K activities/sec.

**CAP Context**: AP (availability for ingestion, eventual consistency for analytics).

**Failure Modes**: Cassandra outages; mitigated by replication. GPS errors; mitigated by validation.

**Cost Considerations**: Kafka storage costs; optimize with retention. Cassandra cost-effective for writes.

**Security**: HTTPS, privacy controls, data validation.

### 17. FB Live Comments
**Improvements**:
- **Ingestion**: Kafka partitioned by video_id.
- **Storage**: DynamoDB: {liveVideoId, commentId, message}.
- **Delivery**: SSE for real-time broadcasting. WebSockets for interactive features (e.g., likes).
- **Moderation**: ML-based spam detection.
- **Analytics**: Flink for real-time comment trends.
- **Caching**: Redis for hot comments.
- **Edge Cases**: Spam, deleted comments, high QPS, ordering, viewer lag, comment throttling, viewer scaling.
- **Monitoring**: Comment latency, delivery success, queue backlog, processing success.

**Architecture Diagram**: Client -> SSE -> API (DynamoDB storage, Redis cache) -> Kafka (ingestion) -> Flink (analytics).

**Trade-offs**:
- Kafka vs RabbitMQ: Kafka for scale.
- WebSockets vs SSE: SSE simpler for one-way.
- Cassandra vs DynamoDB: DynamoDB for simplicity.

**Why Kafka + SSE?** High-throughput ingestion, real-time delivery.

**Scalability**: ~100K comments/sec, ~1M viewers/video, ~50K QPS delivery.

**Latency**: <100ms delivery, <200ms ingestion, ~10K comments/sec.

**CAP Context**: AP (availability for delivery, eventual consistency for analytics).

**Failure Modes**: Kafka lag; mitigated by consumer scaling. DynamoDB throttling; mitigated by reserved capacity.

**Cost Considerations**: Kafka storage costs; optimize with retention. DynamoDB write costs; use caching.

**Security**: HTTPS, spam detection, rate limiting.

## Strong Consistency Systems
Focus: transactional integrity, failure handling.

### 18. Ticket Booking
**Improvements**:
- **Concurrency**: Distributed lock with TTL for reservations. Pessimistic locking for high-demand events.
- **High Demand**: Virtual queue for popular events.
- **Reservation Flow**: Reserve in Redis (TTL), process payment, confirm in DB.
- **Inventory Pre-allocation**: Reserve seats in Redis before payment.
- **Fraud Detection**: ML-based anomaly detection.
- **Logging**: Kafka for booking events.
- **Edge Cases**: Payment failures, expired reservations, overselling, double bookings, high demand, partial bookings, timeout handling.
- **Monitoring**: Booking success, lock contention, payment latency, reservation accuracy.

**Architecture Diagram**: Client -> API (Redis reservations, PostgreSQL bookings) -> Kafka (logging) -> Payment Service.

**Trade-offs**:
- Optimistic vs Pessimistic: Optimistic for scale, pessimistic for high-demand.
- Redis vs DB: Redis for speed, DB for durability.
- Queue vs No Queue: Queue for spikes.

**Why Optimistic + Redis/DB?** Scales for typical loads, fast reservations.

**Scalability**: ~10K bookings/sec, ~1M daily tickets, ~50K QPS reservations.

**Latency**: <200ms reservation, <500ms booking, ~5K bookings/sec.

**CAP Context**: CP (consistency for reservations, availability via replication).

**Failure Modes**: Redis lock contention; mitigated by sharding. Payment failures; mitigated by retries.

**Cost Considerations**: Redis memory costs; optimize with TTL. PostgreSQL storage; use partitioning.

**Security**: HTTPS, fraud detection, access controls.

### 19. E-Commerce (Amazon)
**Improvements**:
- **Catalog**: DynamoDB for products, Elasticsearch for search.
- **Cart**: Redis for carts (TTL for abandoned).
- **Orders**: Saga pattern (Kafka + microservices) for transactions. Strong consistency for inventory via conditional writes.
- **Idempotency**: Unique order_id for checkout.
- **Forecasting**: ML-based demand prediction.
- **Cart Merging**: Merge carts across devices.
- **Processing**: Kafka Streams for order processing.
- **Edge Cases**: Out-of-stock, failed payments, partial orders, cart conflicts, fraud, cart abandonment, price change conflicts.
- **Monitoring**: Checkout success, search latency, inventory accuracy, order latency.

**Architecture Diagram**: Client -> API (DynamoDB catalog, Redis carts) -> Kafka (orders) -> Microservices (Saga) -> Elasticsearch (search).

**Trade-offs**:
- Monolith vs Microservices: Microservices for scale.
- Eventual vs Strong Consistency: Strong for inventory.
- DynamoDB vs Cassandra: DynamoDB simpler.

**Why Microservices + DynamoDB?** Independent scaling, high read/write loads.

**Scalability**: ~100M DAU, ~1M orders/sec, ~100K QPS searches.

**Latency**: <100ms searches, <500ms checkout, ~10K orders/sec.

**CAP Context**: CP (consistency for inventory, availability for searches).

**Failure Modes**: DynamoDB throttling; mitigated by caching. Saga failures; mitigated by compensation.

**Cost Considerations**: DynamoDB read/write costs; optimize with caching. Kafka storage; use retention policies.

**Security**: HTTPS, fraud detection, access controls.

### 20. Messaging App (WhatsApp/Slack)
**Improvements**:
- **Queues**: Kafka partitioned by chat_id.
- **Delivery Receipts**: Redis for status, sync to Cassandra.
- **Retries**: Exponential backoff, dead-letter queues.
- **Offline**: Store in Cassandra, push via APNs/GCM.
- **Media**: Separate attachment system, presigned URLs.
- **Scaling**: Horizontal Chat Servers, Pub/Sub (Redis) for routing.
- **Multiple Clients**: Clients table, clientId topics.
- **Encryption**: End-to-end with Signal Protocol.
- **Search**: Elasticsearch for message search.
- **Group Chats**: Redis Pub/Sub for fanout.
- **Archiving**: S3 for cold storage.
- **Edge Cases**: Ordering, duplicates, group chats, expiration, delays, message retraction, read receipt syncing.
- **Monitoring**: Delivery latency, notification success, queue backlog, processing success.

**Architecture Diagram**: Client -> WebSockets -> API (Cassandra storage, Redis receipts) -> Kafka (queues) -> Elasticsearch (search).

**Trade-offs**:
- Kafka vs RabbitMQ: Kafka for durability.
- Push vs Pull: Push for UX.
- Cassandra vs DynamoDB: Cassandra for cost.

**Why Kafka + Cassandra?** Reliable delivery, scalable storage.

**Scalability**: ~100M DAU, ~1B msgs/day, ~50K QPS delivery.

**Latency**: <100ms delivery, <200ms notifications, ~10K msgs/sec.

**CAP Context**: AP (availability for delivery, eventual consistency for receipts).

**Failure Modes**: Kafka lag; mitigated by consumer scaling. Cassandra outages; mitigated by replication.

**Cost Considerations**: Kafka storage costs; optimize with retention. Cassandra cost-effective for writes.

**Security**: End-to-end encryption, HTTPS, access controls.

### 21. Task Management
**Improvements**:
- **APIs**: REST with PostgreSQL: {task_id, user_id, status, due_date}.
- **Auth**: OAuth2, RBAC.
- **Jobs**: Celery + RabbitMQ for reminders.
- **Audit**: Log changes in separate table. Partition for scale.
- **Dependencies**: DAG-based execution with Celery.
- **Notifications**: Kafka for reminder events.
- **Caching**: Redis for task metadata.
- **Edge Cases**: Concurrent edits, deleted users, overdue tasks, duplicates, permission errors, task reassignment, priority scheduling.
- **Monitoring**: API latency, queue length, log size, update success.

**Architecture Diagram**: Client -> API (PostgreSQL tasks, Redis cache) -> Celery/RabbitMQ (jobs) -> Kafka (notifications).

**Trade-offs**:
- SQL vs NoSQL: SQL for relations.
- Celery vs Custom: Celery for reliability.
- REST vs GraphQL: REST for simplicity.

**Why PostgreSQL + Celery?** Relational data, reliable jobs.

**Scalability**: ~1M DAU, ~10M tasks/day, ~10K QPS APIs.

**Latency**: <100ms CRUD, <500ms reminders, ~5K tasks/sec.

**CAP Context**: CP (consistency for tasks, availability via replication).

**Failure Modes**: PostgreSQL contention; mitigated by indexing. Celery failures; mitigated by retries.

**Cost Considerations**: PostgreSQL storage; use partitioning. RabbitMQ compute; optimize with clustering.

**Security**: OAuth2, HTTPS, RBAC.

### 22. Tinder
**Improvements**:
- **Profiles**: DynamoDB: {user_id, preferences, location}.
- **Matchmaking**: Redis Geo for location-based swipes.
- **Transactions**: Optimistic locking for swipes.
- **Swipes**: Dedicated Swipe Service, Cassandra for storage.
- **Ranking**: ML-based compatibility scoring.
- **Abuse Detection**: Rate limiting swipes.
- **Ingestion**: Kafka for swipe events.
- **Edge Cases**: Double swipes, high volume, location errors, privacy, stale data, profile staleness, geo-fencing.
- **Monitoring**: Match latency, swipe throughput, geo accuracy, success rate.

**Architecture Diagram**: Client -> API (Redis Geo, DynamoDB profiles, Cassandra swipes) -> Kafka (ingestion) -> ML (ranking).

**Trade-offs**:
- Redis Geo vs PostGIS: Redis for speed.
- Optimistic vs Pessimistic: Optimistic for scale.
- Cassandra vs DynamoDB: Cassandra for writes.

**Why Redis + Cassandra?** Fast geo-matching, scalable swipes.

**Scalability**: ~10M DAU, ~1B swipes/day, ~50K QPS matches.

**Latency**: <100ms matches, <50ms swipes, ~10K swipes/sec.

**CAP Context**: AP (availability for swipes, eventual consistency for matches).

**Failure Modes**: Redis outages; mitigated by fallback to Cassandra. Location errors; mitigated by validation.

**Cost Considerations**: Redis memory costs; optimize with TTL. Cassandra cost-effective for writes.

**Security**: HTTPS, rate limiting, privacy controls.

### 23. LeetCode
**Improvements**:
- **Submissions**: RabbitMQ for queuing, partitioned by problem_id.
- **Judging**: Docker containers for isolation, PostgreSQL for results. Cgroups/seccomp for security.
- **Scheduling**: Celery for test runs.
- **Feedback**: Run code in language-specific containers, return results.
- **Isolation**: Read-only mounts, resource limits, timeouts.
- **Leaderboard**: Redis Sorted Sets for real-time rankings.
- **Test Cases**: PostgreSQL for test data.
- **Analytics**: ML-based difficulty scoring.
- **Logging**: Kafka for submission events.
- **Edge Cases**: Malicious code, timeouts, high rates, test failures, disputes, resource exhaustion, submission retries.
- **Monitoring**: Submission latency, judge accuracy, queue backlog, success rate.

**Architecture Diagram**: Client -> API (PostgreSQL results, Redis leaderboards) -> RabbitMQ/Celery (submissions) -> Docker (judging) -> Kafka (logging).

**Trade-offs**:
- RabbitMQ vs Kafka: RabbitMQ simpler for tasks.
- Docker vs VMs: Docker lightweight, sandboxed.
- Celery vs Custom: Celery for reliability.

**Why RabbitMQ + Docker?** Reliable queuing, scalable judging.

**Scalability**: ~10K submissions/sec, ~1M daily, ~10K QPS judging.

**Latency**: <1s judging, <500ms queuing, ~5K submissions/sec.

**CAP Context**: CP (consistency for results, availability via replication).

**Failure Modes**: Docker resource exhaustion; mitigated by limits. RabbitMQ lag; mitigated by scaling.

**Cost Considerations**: Docker compute costs; optimize with spot instances. PostgreSQL storage; use partitioning.

**Security**: Sandboxing, HTTPS, rate limiting.

### 24. Ad Click Aggregator
**Improvements**:
- **Ingestion**: Kafka partitioned by ad_id.
- **Aggregation**: Flink for real-time, Redis for fast reads, PostgreSQL for durability. Spark Streaming for micro-batch alternative.
- **Scheduling**: Cron for daily/weekly reports.
- **Fraud Detection**: ML-based anomaly detection.
- **Geo-Aggregation**: Redis Geo for location stats.
- **Search**: Elasticsearch for ad performance.
- **Edge Cases**: Fraud, delayed events, high throughput, duplicates, skew, event deduplication, late event handling.
- **Monitoring**: Click latency, aggregation accuracy, Kafka lag, processing success.

**Architecture Diagram**: Source -> Kafka -> Flink (aggregation) -> Redis/PostgreSQL (storage) -> Elasticsearch (search).

**Trade-offs**:
- Flink vs Spark: Flink for real-time.
- Redis vs DB: Redis for speed, DB for durability.
- Kafka vs RabbitMQ: Kafka for scale.

**Why Kafka + Flink?** High-throughput clicks, real-time aggregation.

**Scalability**: ~100K clicks/sec, ~1B daily, ~10K QPS aggregates.

**Latency**: <100ms ingestion, <500ms aggregates, ~50K clicks/sec.

**CAP Context**: AP (availability for ingestion, eventual consistency for aggregates).

**Failure Modes**: Kafka lag; mitigated by consumer scaling. Flink failures; mitigated by checkpointing.

**Cost Considerations**: Kafka storage costs; optimize with retention. Flink compute costs; use spot instances.

**Security**: HTTPS, fraud detection, access controls.

## Trie / Proximity Systems
Focus: efficient data structures, low-latency retrieval.

### 25. Search Autocomplete
**Improvements**:
- **Structure**: Trie in Redis for prefix lookups, frequency scores.
- **Debouncing**: Client-side (300ms).
- **Caching**: Redis for popular queries (LRU).
- **Typos**: Levenshtein or n-grams for fuzzy matching. N-grams faster for prefixes.
- **Personalization**: User history in Redis.
- **Logging**: Kafka for query analytics.
- **Fallback**: Elasticsearch for full-text search.
- **Edge Cases**: Long queries, empty results, high QPS, spam, stale suggestions, query sanitization, suggestion ranking.
- **Monitoring**: Query latency, cache hit, suggestion accuracy, processing success.

**Architecture Diagram**: Client -> API (Redis Trie) -> Kafka (logging) -> Elasticsearch (fallback).

**Trade-offs**:
- Trie vs TST: Trie simpler, Redis-integrated.
- Exact vs Fuzzy: Exact fast, fuzzy fallback.
- Redis vs In-Memory: Redis durable, scalable.

**Why Trie + Redis?** Fast prefix lookups, scalable caching.

**Scalability**: ~100K QPS, ~1M DAU, ~10M suggestions/day.

**Latency**: <50ms suggestions, <100ms caching, ~50K queries/sec.

**CAP Context**: AP (availability for suggestions, eventual consistency for analytics).

**Failure Modes**: Redis outages; mitigated by replication. High QPS; mitigated by caching.

**Cost Considerations**: Redis memory costs; optimize with TTL. Kafka storage; use retention policies.

**Security**: Query sanitization, HTTPS, rate limiting.

### 26. Ride-Sharing (Uber/Lyft)
**Improvements**:
- **Matchmaking**: Redis Geo or PostGIS for nearby drivers. Geohashing for query optimization.
- **Location Tracking**: Kafka for real-time streams, Flink for processing.
- **ETA**: OSRM with traffic data.
- **Surge**: Supply/demand ratios in Redis.
- **Location Updates**: Redis for fast geospatial storage.
- **Adaptive Updates**: Adjust frequency by movement.
- **Simultaneous Requests**: Distributed lock with TTL.
- **Peak Demand**: Queue with dynamic scaling.
- **Geo-Sharding**: Partition by region for low latency.
- **Incentives**: ML-based surge pricing.
- **Analytics**: Flink for trip trends.
- **Streams**: Redis Streams for location events.
- **Edge Cases**: Cancellations, GPS errors, high demand, surge abuse, network failures, driver unavailability, ride cancellation refunds.
- **Monitoring**: Match latency, ETA accuracy, surge multiplier, match success.

**Architecture Diagram**: Client -> API (Redis Geo, PostGIS) -> Kafka/Flink (tracking) -> OSRM (routing) -> ML (pricing).

**Trade-offs**:
- Redis Geo vs PostGIS: Redis faster for simple queries.
- Real-time vs Batch ETA: Real-time for UX.
- Kafka vs RabbitMQ: Kafka for scale.

**Why Redis + Kafka?** Fast geospatial queries, real-time streams.

**Scalability**: ~10M DAU, ~1M rides/day, ~50K QPS matching.

**Latency**: <100ms matching, <500ms ETA, ~10K matches/sec.

**CAP Context**: AP (availability for matching, eventual consistency for analytics).

**Failure Modes**: Redis outages; mitigated by PostGIS fallback. Kafka lag; mitigated by consumer scaling.

**Cost Considerations**: Redis memory costs; optimize with TTL. Kafka storage; use retention policies.

**Security**: HTTPS, GPS validation, rate limiting.

### 27. Typeahead Suggestion
**Improvements**:
- **Structure**: Trie in Redis for prefixes, frequency scores. Optimized for entity names (users, products).
- **Caching**: Redis for popular queries (LRU).
- **Typos**: N-grams or Levenshtein for fuzzy matching. N-grams faster for prefixes.
- **Context-Aware**: Suggestions based on user role/location.
- **Testing**: A/B testing for suggestion algorithms.
- **Logging**: Kafka for query analytics.
- **Edge Cases**: Long queries, empty results, high QPS, duplicates, stale suggestions, suggestion throttling, offline caching.
- **Monitoring**: Suggestion latency, cache hit, accuracy, success rate.

**Architecture Diagram**: Client -> API (Redis Trie) -> Kafka (logging) -> A/B Testing Service.

**Trade-offs**:
- Trie vs Inverted Index: Trie for prefixes.
- Exact vs Fuzzy: Exact with fuzzy fallback.
- Redis vs In-Memory: Redis for durability.

**Why Trie + Redis?** Fast prefix lookups, scalable caching.

**Scalability**: ~100K QPS, ~1M DAU, ~10M suggestions/day.

**Latency**: <50ms suggestions, <100ms caching, ~50K queries/sec.

**CAP Context**: AP (availability for suggestions, eventual consistency for analytics).

**Failure Modes**: Redis outages; mitigated by replication. High QPS; mitigated by caching.

**Cost Considerations**: Redis memory costs; optimize with TTL. Kafka storage; use retention policies.

**Security**: HTTPS, rate limiting, query sanitization.

### 28. Twitter Search
**Improvements**:
- **Index**: Elasticsearch for tweets: {tweet_id, content, user_id, timestamp}. Recent tweets cached in Redis.
- **Caching**: Redis for recent/popular searches (TTL).
- **Ranking**: Recency, relevance, engagement via weighted scoring.
- **Hashtags**: Inverted index in Redis for hashtag search.
- **Trends**: Flink for real-time trend detection.
- **Ingestion**: Kafka for tweets.
- **Edge Cases**: Trending topics, deleted tweets, high QPS, spam, stale results, search throttling, result pagination.
- **Monitoring**: Search latency, ranking accuracy, cache hit, success rate.

**Architecture Diagram**: Client -> API (Elasticsearch index, Redis cache) -> Kafka (ingestion) -> Flink (trends).

**Trade-offs**:
- Elasticsearch vs Solr: Elasticsearch managed, costlier.
- Real-time vs Batch: Real-time for UX.
- Caching vs No Caching: Caching with TTL for balance.

**Why Elasticsearch + Redis?** Complex searches, fast caching.

**Scalability**: ~100M DAU, ~1B tweets/day, ~100K QPS searches.

**Latency**: <100ms results, <200ms caching, ~50K searches/sec.

**CAP Context**: AP (availability for searches, eventual consistency for indexing).

**Failure Modes**: Elasticsearch latency; mitigated by caching. Kafka lag; mitigated by consumer scaling.

**Cost Considerations**: Elasticsearch compute costs; optimize with reserved instances. Redis memory; use TTLs.

**Security**: HTTPS, spam filtering, rate limiting.

### 29. FB Post Search (No Search Engine)
**Improvements**:
- **Inverted Index**: Map keywords to post IDs in Redis.
- **Sorting**: Lists for recency, sorted sets for likes.
- **Caching**: CDN for search results.
- **Phrases**: Bigrams for multi-keyword queries. Trigrams for fuzzy matching.
- **High Writes**: Kafka for ingestion, shard indexes, approx likes.
- **Storage**: Cap indexes, move rare keywords to blob storage.
- **Privacy**: Restrict search to authorized users.
- **Ranking**: ML-based relevance scoring.
- **Updates**: Redis Streams for index updates.
- **Edge Cases**: Duplicates, slow subscribers, high throughput, unavailable subscribers, large lists, index rebuilding, search timeouts.
- **Monitoring**: Request latency, delivery success, storage perf, cache hit, customer metrics.

**Architecture Diagram**: Client -> CDN -> API (Redis index) -> Kafka (ingestion) -> ML (ranking).

**Trade-offs**:
- Kafka vs RabbitMQ: Kafka for scale.
- Cassandra vs DynamoDB: Cassandra for cost.
- Task-based vs Sequential: Task-based for isolation.

**Why Kafka + Task-Based Sender?** Scalable ingestion, parallel delivery.

**Scalability**: ~1M msgs/sec, ~1B daily, ~10K QPS ops.

**Latency**: <100ms ingestion, <1s delivery, <50ms metadata, ~1M msgs/sec.

**CAP Context**: AP (availability for searches, eventual consistency for indexing).

**Failure Modes**: Redis memory exhaustion; mitigated by capping indexes. Kafka lag; mitigated by consumer scaling.

**Cost Considerations**: Redis memory costs; optimize with eviction. Kafka storage; use retention policies.

**Security**: HTTPS, privacy filtering, rate limiting.

## Aggregation Systems
Focus: processing, ranking high-cardinality data.

### 30. Top K (YouTube Views, Keywords, Songs)
**Improvements**:
- **Counting**: Count-min sketch in Redis for approx counts. Periodic exact counts via Spark.
- **Ranking**: Spark for top K, cache in Redis.
- **Sliding Window**: Redis sorted sets for time-based tracking.
- **Time-Based Ranking**: Sliding window in Redis for recency.
- **Geo-Aggregation**: Redis Geo for regional trends.
- **Alternative**: Flink for real-time processing.
- **Edge Cases**: Spikes, stale data, high cardinality, skew, duplicates, data skew, late event handling.
- **Monitoring**: Sketch accuracy, ranking latency, cache hit, ranking success.

**Architecture**: Lambda Architecture for fast (approx) + slow (accurate) paths.
- **Ingestion**: API Gateway -> logs -> Kafka (random partitions).
- **Fast Path**: Count-Min Sketch for approx top K, flush to Storage.
- **Slow Path**: MapReduce for accurate top K, store in Storage.
- **Retrieval**: API Gateway queries Storage for top K lists.

**Scalability**: Horizontal scaling, Kafka partitioning, consistent hashing.

**Fault Tolerance**: Kafka replication, MapReduce retries.

**Edge Cases**: Hot partitions, delayed events, duplicates, large k.

**Trade-offs**: Fast vs slow path, Kafka vs Kinesis, MapReduce vs Spark.

**Applications**: Trending services, DDoS protection, financial systems.

**Why Count-Min + Redis?** Scales high cardinality, low-latency ranking.

**Scalability**: ~100K events/sec, ~1B daily, ~10K QPS rankings.

**Latency**: <50ms updates, <500ms rankings, ~50K events/sec.

**CAP Context**: AP (availability for updates, eventual consistency for rankings).

**Failure Modes**: Count-min overestimation; mitigated by slow path. Kafka lag; mitigated by consumer scaling.

**Cost Considerations**: Redis memory costs; optimize with eviction. Spark/Flink compute; use spot instances.

**Security**: HTTPS, rate limiting, access controls.

## Distributed Systems

### Distributed Message Queue
Async communication, decouples producers/consumers.

**Requirements**:
- APIs: send, receive, create/delete queues.
- Scalable, available, fast, durable, observable.

**Architecture**:
- **VIP -> Load Balancer -> FrontEnd (stateless)**: SSL, auth, validation, dedup, rate limit, routing, caching, encryption.
- **Metadata Service**: Queue config, DB-backed, cached.
- **Backend**: Message storage, replication, delivery.
- **Prioritization**: High-priority queues.
- **Dead-Letter Queues**: Store failed messages.

**Models**:
- **Single Leader**: Leader per queue, replicates to followers.
- **Leaderless**: Clusters handle queues, any instance serves.

**Lifecycle**: Visibility timeouts, deletion policies (explicit, timeout, retention).

**Delivery**: At-least-once, no strict ordering. Exactly-once with idempotency and transactional outbox.

**Scalability**: Horizontal scaling, sharding, replication.

**Security**: SSL/TLS, encryption at rest.

**Observability**: Metrics, logs for health, client visibility.

**Applications**: Microservices, log pipelines, task scheduling.

**Reference**: Kafka for high-throughput queues.

**Scalability**: ~1M msgs/sec, ~1B daily.

**Latency**: <100ms ingestion, <1s delivery.

**CAP Context**: AP (availability for ingestion, eventual consistency for delivery).

**Failure Modes**: Queue overflow; mitigated by backpressure. Consumer failures; mitigated by retries.

**Cost Considerations**: Kafka storage costs; optimize with retention. Compute costs for frontends; use auto-scaling.

**Security**: HTTPS, encryption, access controls.

### Notification Service
Delivers messages from publishers to subscribers.

**Architecture**:
- **FrontEnd**: Validation, auth, SSL, caching, throttling, dedup.
- **Metadata Service**: Topics/subscribers, cached in Redis.
- **Temporary Storage**: Kafka for messages.
- **Sender Service**: Retrieves messages, delivers via tasks.
- **Templates**: Dynamic content for notifications.
- **Preferences**: User opt-out settings in Redis.
- **Alternative**: Flink for real-time processing.

**Delivery**: At-least-once, retries, confirmation. WebSockets/APNs/GCM for delivery.

**Security**: Auth, SSL, encryption.

**Monitoring**: Health, customer metrics.

**Scalability**: ~1M msgs/sec, ~1B daily.

**Latency**: <100ms ingestion, <1s delivery.

**Edge Cases**: Notification deduplication, delivery throttling.

**CAP Context**: AP (availability for delivery, eventual consistency for metadata).

**Failure Modes**: Kafka lag; mitigated by consumer scaling. Delivery failures; mitigated by retries.

**Cost Considerations**: Kafka storage costs; optimize with retention. Redis memory; use TTLs.

**Security**: HTTPS, encryption, user preferences.

### Distributed Cache
Speeds data retrieval with in-memory storage.

**Architecture**:
- **Load Balancer -> Cache Cluster**: Sharded via consistent hashing.
- **LRU Cache**: Hash table + doubly linked list for O(1) ops.
- **Replication**: Master-slave for hot shards. Sync vs async replication.
- **Configuration Service**: Host discovery, leader election.
- **Expiration**: TTL with passive/active cleanup.
- **Eviction**: LFU as alternative to LRU.
- **Warming**: Preload predictable workloads.
- **Reference**: Redis Cluster for scalability.

**Scalability**: Horizontal scaling, replication.

**Security**: Firewalls, optional encryption.

**Monitoring**: Hit/miss rates, latency, resource usage.

**Trade-offs**: Consistent hashing vs Jump Hash, async vs sync replication, dedicated vs co-located cache.

**Scalability**: ~1M req/sec, ~1B daily ops.

**Latency**: <1ms hits, <10ms shard selection.

**Edge Cases**: Cache stampede, hot shard mitigation.

**CAP Context**: AP (availability for reads, eventual consistency for replication).

**Failure Modes**: Cache stampede; mitigated by probabilistic early expiration. Hot shards; mitigated by rebalancing.

**Cost Considerations**: Redis memory costs; optimize with eviction. Compute costs for load balancers; use auto-scaling.

**Security**: HTTPS, encryption, access controls.

### Distributed File System
Scalable storage for large files with high availability and durability.

**What**: Stores and retrieves files, supports partial reads/writes, replicates data.

**Does**: Provides scalable, durable file storage with fault tolerance.

**How**: Metadata service (DynamoDB) for file locations, data nodes (S3) for storage, consistent hashing for sharding, replication for fault tolerance.

**Improvements**:
- **Metadata**: DynamoDB for {file_id, path, location}.
- **Storage**: S3 for data chunks.
- **Replication**: Multi-region S3 buckets.
- **Access**: Presigned URLs for secure reads/writes.
- **Compression**: Gzip/Brotli for bandwidth efficiency.
- **Deduplication**: Content hashing for storage savings.

**Pros**: Scalable, durable, fault-tolerant, supports large files.

**Cons**: Complex to manage, eventual consistency for some operations, high latency for small files.

**Examples**:
- Hadoop HDFS: Distributed storage for big data analytics.
- Google GFS: Backend for Google services.
- AWS EFS: Elastic file system for cloud workloads.

**Trade-offs**:
- Centralized vs Decentralized Metadata: Centralized simpler, decentralized scales better.
- Strong vs Eventual Consistency: Strong for metadata, eventual for data.
- S3 vs Custom: S3 managed, custom cheaper but complex.

**Scalability**: ~1PB storage, ~10K QPS reads/writes.

**Latency**: <100ms metadata, <500ms data retrieval.

**Edge Cases**: File conflicts, partial writes, corruption.

**Monitoring**: Access latency, storage usage, replication health.

**CAP Context**: CP (consistency for metadata, availability via replication).

**Failure Modes**: S3 outages; mitigated by multi-region buckets. Metadata inconsistencies; mitigated by conditional writes.

**Cost Considerations**: S3 storage costs; optimize with lifecycle policies. DynamoDB read/write costs; use caching.

**Security**: HTTPS, encryption, access controls.

**Architecture Diagram**: Client -> API (DynamoDB metadata, S3 storage) -> CDN (caching).

### Distributed Lock Manager
Coordinates access to shared resources in distributed systems.

**What**: Grants exclusive or shared locks with timeouts to prevent deadlocks.

**Does**: Ensures safe concurrent access to resources.

**How**: Uses ZooKeeper/etcd or Redis with Redlock. Locks stored as ephemeral nodes or key-value pairs with TTL.

**Improvements**:
- **Locking**: Redis Redlock for lightweight locks.
- **Timeouts**: TTL to prevent stale locks.
- **Election**: ZooKeeper for leader election.
- **Monitoring**: Lock contention, acquisition latency.

**Pros**: Prevents race conditions, scalable, fault-tolerant with proper implementation.

**Cons**: Complex to implement correctly, performance overhead, lease expiration risks.

**Examples**:
- ZooKeeper: Used in Apache Hadoop for coordination.
- Chubby (Google): Lock service for GFS and Bigtable.
- Redis Redlock: Lightweight locking for microservices.

**Trade-offs**:
- ZooKeeper vs Redis: ZooKeeper reliable, Redis faster but less robust.
- Exclusive vs Shared Locks: Exclusive simpler, shared for read-heavy.
- Lease vs Permanent: Lease prevents stale locks, requires renewal.

**Scalability**: ~10K lock requests/sec, ~1M concurrent locks.

**Latency**: <10ms lock acquisition, <50ms release.

**Edge Cases**: Lock contention, stale locks, network partitions.

**Monitoring**: Lock contention, acquisition latency, lease expirations.

**CAP Context**: CP (consistency for locks, availability via replication).

**Failure Modes**: Network partitions; mitigated by quorum-based systems. Stale locks; mitigated by TTLs.

**Cost Considerations**: ZooKeeper compute costs; optimize with clustering. Redis memory; use TTLs.

**Security**: HTTPS, access controls, encryption.

**Architecture Diagram**: Client -> API (ZooKeeper/Redis locks) -> Backend (protected resource).

### Event Sourcing System
Persists application state as a sequence of events.

**What**: Stores events (e.g., user actions) and reconstructs state by replaying them.

**Does**: Provides auditability and flexibility for state reconstruction.

**How**: Events in Kafka/EventStoreDB, projections via Flink/Spark, state cached in Redis/DB.

**Improvements**:
- **Storage**: Kafka for event logs.
- **Projections**: Flink for real-time state.
- **Snapshots**: Periodic snapshots to reduce replay latency.
- **Schema**: Versioned events for evolution.

**Pros**: Auditability, flexibility to rebuild state, supports CQRS.

**Cons**: Complex to implement, event schema evolution, replay latency.

**Examples**:
- E-commerce: Order history in Kafka for audit and analytics.
- Banking: Transaction logs for account balances.
- Microservices: Event-driven architectures with Axon/Eventuate.

**Trade-offs**:
- Kafka vs EventStoreDB: Kafka scalable, EventStoreDB optimized for events.
- Eventual vs Strong Consistency: Eventual for projections, strong for critical state.
- Snapshotting vs Full Replay: Snapshotting reduces latency, increases complexity.

**Scalability**: ~100K events/sec, ~1B events/day.

**Latency**: <100ms event ingestion, <500ms state reconstruction.

**Edge Cases**: Event duplicates, schema mismatches, replay failures.

**Monitoring**: Event latency, projection accuracy, storage usage.

**CAP Context**: CP (consistency for events, availability via replication).

**Failure Modes**: Kafka lag; mitigated by consumer scaling. Replay failures; mitigated by snapshots.

**Cost Considerations**: Kafka storage costs; optimize with retention. Flink compute; use spot instances.

**Security**: HTTPS, encryption, access controls.

**Architecture Diagram**: Source -> Kafka (events) -> Flink (projections) -> Redis/DB (state).

### API Gateway
Centralized entry point for client requests to microservices.

**What**: Handles routing, authentication, rate limiting, caching, and metrics.

**Does**: Simplifies client access and secures microservices.

**How**: Load balancer routes to gateway (AWS API Gateway/Kong), which authenticates (OAuth2/JWT), rate-limits (Redis), caches (Redis), and routes to services.

**Improvements**:
- **Auth**: OAuth2/JWT for secure access.
- **Rate Limiting**: Token bucket in Redis.
- **Caching**: Redis for API responses.
- **Metrics**: Prometheus for request monitoring.

**Pros**: Simplifies client access, improves security, reduces latency with caching.

**Cons**: Single point of failure, added latency, complex configuration.

**Examples**:
- AWS API Gateway: Managed gateway for serverless apps.
- Netflix Zuul: Custom gateway for microservices.
- Kong: Open-source gateway for APIs.

**Trade-offs**:
- Managed vs Custom: Managed simpler, custom flexible.
- Centralized vs Decentralized: Centralized easier to manage, decentralized scales better.
- Caching vs No Caching: Caching reduces backend load, increases staleness risk.

**Scalability**: ~1M req/sec, ~1B daily requests.

**Latency**: <10ms routing, <50ms with auth/caching.

**Edge Cases**: Rate limit exhaustion, cache staleness, auth failures.

**Monitoring**: Request latency, error rates, cache hit ratio.

**CAP Context**: AP (availability for routing, eventual consistency for caching).

**Failure Modes**: Gateway outages; mitigated by auto-scaling. Cache staleness; mitigated by TTLs.

**Cost Considerations**: API Gateway compute costs; optimize with reserved instances. Redis memory; use TTLs.

**Security**: HTTPS, OAuth2, rate limiting.

**Architecture Diagram**: Client -> Load Balancer -> API Gateway (Redis cache) -> Microservices.

## Security Best Practices
- **Encryption**: Use HTTPS/TLS for all APIs, encrypt data at rest (e.g., S3, DynamoDB).
- **Authentication**: Implement OAuth2/JWT for secure access.
- **Rate Limiting**: Use token bucket in Redis to prevent abuse.
- **Content Moderation**: Apply ML-based filtering for user-generated content.
- **Access Controls**: Use RBAC/IAM for fine-grained permissions.
- **DDoS Protection**: Use Cloudflare/Akamai for traffic filtering.
- **Input Validation**: Sanitize inputs to prevent injection attacks.
- **Audit Logging**: Log critical actions (e.g., Kafka) for traceability.