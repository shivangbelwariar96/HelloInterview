# System Design Cheatsheet

## Read-Heavy Systems
These prioritize low-latency data retrieval, efficient caching, and scalability.

### 1. URL Shortener (Bitly)
**Improvements:**
- **Key Generation**: Use base62 encoding (a-z, A-Z, 0-9) for short keys (e.g., 7 characters yield ~62^7 URLs). Use a distributed counter (Redis is particularly well-suited for managing this counter because it's single-threaded and supports atomic operations.) or MD5 hash of the long URL with truncation to avoid collisions.
- **Collision Handling**: Add a Bloom filter to check for existing short URLs before insertion. If a collision occurs, append a unique suffix or retry with a new hash.
- **DB Storage**: Use a NoSQL DB like DynamoDB for high read/write throughput. Store {short_url: long_url, creation_time, expiration_time, user_id}.
- **301 vs 302**: Choose 302 (Temporary Redirect) ensuring that future requests for the short URL will always go through our server first.
- **Caching**: Use Redis for caching hot URLs (e.g., LRU eviction). Cache both short-to-long and long-to-short mappings to speed up lookups and prevent duplicate short URLs.
- **Sharding**: Shard by short_url prefix to distribute load across DB nodes. Avoid sharding by user_id to prevent hotspots from power users. (can create some partitionKey, SortKey, GSI, LSI)
- **Edge Cases**: Handle expired URLs (use TTL in Redis/DB), malicious URLs (blocklist), custom aliases (store separately), rate-limited requests (throttle excessive redirects), and invalid URLs (validate format on input).
- **Monitoring**: Track redirect latency, cache hit ratio, collision rate, and request rate per second.
- **Trade-offs:**
  - **Hashing vs. Counter**: Hashing (MD5) is stateless but risks collisions; a counter (Redis) is collision-free but requires coordination. Choose counter for guaranteed uniqueness and simplicity.
  - **SQL vs. NoSQL**: NoSQL (DynamoDB) is preferred for scalability and low-latency reads over SQL, which struggles with high write throughput.
  - **In-Memory vs. Persistent Storage**: Redis is fast for caching but volatile; DynamoDB ensures durability but is slower. Use Redis for hot data and DynamoDB for persistence.
- **Why Counter + NoSQL?**: Counters ensure uniqueness without retry logic, and NoSQL scales horizontally for traffic spikes.
- **Can USE CDN**: For low latency.
- **Scalability Estimates**: Supports ~10K QPS for redirects, ~1B unique URLs, and ~100M daily active users with horizontal scaling of DynamoDB and Redis clusters.
- **Latency/Throughput Targets**: Target <50ms for redirect latency, ~10K redirects/sec, and <100ms for URL creation.

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

### 3. Social Media Platform (Twitter/Facebook)
**Improvements:**
- **Posts**: Store in a NoSQL DB (Cassandra) with {post_id, user_id, content, timestamp}. Use time-based partitioning for efficient range queries.
- **Timelines**: Use a hybrid push-pull model: push posts to followers’ timelines (Redis lists) for active users; pull for inactive users to reduce write amplification.
- **Relationships**: Store follow/friend graphs in a graph DB (Neo4j) or adjacency lists in Redis for fast traversal.
- **Denormalization**: Store precomputed timelines to avoid joining user and post tables. Update timelines incrementally on new posts.
- **Sharding**: Shard posts by post_id and timelines by user_id to distribute load. Use consistent hashing to handle node failures.
- **Edge Cases**: Handle deleted posts (tombstone records), privacy settings (filter on read), viral posts (cache hot content), user bans (block content access), and timeline staleness (refresh periodically).
- **Monitoring**: Track timeline latency, fanout throughput, graph query performance, and post retrieval success rate.
- **Trade-offs:**
  - **Push vs. Pull**: Push ensures low read latency but high write cost for influencers; pull is write-efficient but slower for reads. Hybrid model balances both by pushing to active users only.
  - **Graph DB vs. Adjacency Lists**: Graph DBs are flexible for complex queries but slower; adjacency lists in Redis are faster for simple follow lookups. Choose adjacency lists for low-latency timeline generation.
  - **Cassandra vs. DynamoDB**: Cassandra handles high write throughput; DynamoDB is simpler but costlier. Choose Cassandra for write-heavy posts.
- **Why Hybrid + Cassandra?**: Hybrid model optimizes for both influencers and inactive users, and Cassandra handles high write throughput for posts.
- **Scalability Estimates**: Supports ~100M daily active users, ~1B posts/day, and ~50K QPS for timeline retrieval with Cassandra clusters and Redis caching.
- **Latency/Throughput Targets**: Target <200ms for timeline generation, <100ms for post retrieval, and ~10K posts/sec ingestion.

### 4. NewsFeed System
**Improvements:**
- **Push vs. Pull**: Use fanout-on-write for active users (push posts to Redis timelines) and fanout-on-read for inactive users to reduce write load.
- **Caching**: Cache recent posts in Redis and full timelines in Memcached to reduce DB load.
- **Pagination**: Use cursor-based pagination (timestamp + post_id) for efficient scrolling.
- **Ranking**: Use a weighted scoring algorithm (e.g., likes, recency, user affinity) computed via a background job (Spark) and stored in Redis.
- **Handle users with a large number of followers**: For Justin Bieber (and other high-follow accounts), instead of writing to 90+ million followers we can instead add a flag onto the Follow table which indicates that this particular follow isn't precomputed. In the async worker queue, we'll ignore requests for these users. On the read side, when users request their feed via the Feed Service, we can grab their (partially) precomputed feed from the Feed Table and merge it with recent posts from those accounts which aren't precomputed.
- **Edge Cases**: Handle feed staleness (refresh inactive users’ feeds periodically), deleted posts (remove from cache), skewed user activity (rate-limit influencers), privacy filters (apply on read), and cache inconsistencies (revalidate with DB).
- **Monitoring**: Track feed refresh latency, ranking accuracy, cache eviction rate, and feed generation success rate.
- **Trade-offs:**
  - **Fanout-on-Write vs. Read**: Write is faster for active users but expensive for influencers; read is write-efficient but slower. Choose hybrid to balance latency and cost.
  - **Real-time vs. Batch Ranking**: Real-time ranking is fresh but compute-heavy; batch ranking is efficient but stale. Choose batch with periodic updates for scalability.
  - **Redis vs. Memcached**: Redis supports complex data structures but is heavier; Memcached is lightweight but simpler. Use both for complementary caching.
- **Why Hybrid + Redis?**: Hybrid minimizes write amplification, and Redis ensures low-latency feed retrieval.
- **Scalability Estimates**: Handles ~50M daily active users, ~500M posts/day, and ~20K QPS for feed generation with Redis and Memcached scaling.
- **Latency/Throughput Targets**: Target <150ms for feed retrieval, <500ms for ranking updates, and ~5K feeds/sec generation.

### 5. Pastebin
**Improvements:**
- **Text Storage**: Store pastes in a NoSQL DB (DynamoDB) with {paste_id, content, creation_time, expiration_time, user_id}. Use S3 for large pastes (>1MB).
- **Key Generation**: Generate short paste IDs using base62 encoding (6-8 chars, ~62^6 URLs) via a distributed counter (ZooKeeper) or hash of content.
- **Caching**: Cache hot pastes in Redis (LRU eviction) for fast retrieval.
- **Expiration**: Set TTL in Redis and DB for auto-deletion of expired pastes.
- **Access Control**: Support public/private pastes with signed URLs for private access.
- **Edge Cases**: Handle duplicate pastes (hash-based deduplication), malicious content (content filters), high read QPS (CDN like Cloudflare), large pastes (multipart uploads to S3), and expired paste revival (archive option).
- **Monitoring**: Track paste creation rate, cache hit ratio, expiration cleanup, and paste retrieval latency.
- **Trade-offs:**
  - **Counter vs. Hashing**: Counter ensures unique IDs but requires coordination; hashing risks collisions. Choose counter for simplicity and guaranteed uniqueness.
  - **NoSQL vs. S3**: NoSQL is fast for metadata; S3 is better for large blobs. Use DynamoDB for metadata and S3 for content to balance speed and cost.
  - **Redis vs. CDN**: Redis is fast for hot data; CDN scales for global access. Use both for optimal performance.
- **Why Counter + DynamoDB/S3?**: Counter ensures unique IDs, DynamoDB provides low-latency metadata access, and S3 scales for large content.
- **Scalability Estimates**: Supports ~1K paste creations/sec, ~10M daily active users, and ~100K QPS for reads with DynamoDB and CDN scaling.
- **Latency/Throughput Targets**: Target <50ms for paste retrieval, <200ms for creation, and ~2K pastes/sec ingestion.

### 6. Instagram
**Improvements:**
- **Media Storage**: Store images/videos in S3 with metadata (post_id, user_id, timestamp) in Cassandra for high write throughput.
- **Feed Generation**: Use a hybrid push-pull model: push posts to active followers’ Redis timelines; pull for inactive users to reduce write load.
- **Caching**: Cache feeds and media in Redis and CDN (Cloudflare) for low-latency delivery. Apply smart caching: aggressively cache popular media, use shorter TTLs for less-accessed content, and pre-warm caches for trending items.
- **Sharding**: Shard posts by post_id and timelines by user_id using consistent hashing.
- **low latency (< 500ms )**: To reduce write amplification from celebrities, we use a hybrid approach: fanout-on-write for users with <100K followers and fanout-on-read for celebrities (>100K followers). Celebrity posts go only to the Posts DB, not follower feeds. When a user fetches their feed, we merge precomputed posts (from Redis) with recent celebrity posts (from DB). This balances fast reads and scalable writes. Use Redis Cluster for data sharding across multiple nodes for durability
- **Sort Chronologically**: To show a chronological feed of posts from followed users, query the Follow table (DynamoDB, with followerId as partition key, followedId as sort key) to get followed user IDs, then query the Post table (partition key: userId, sort key: createdAt+postId) for their recent posts. Combine, sort chronologically, and return posts to the client with cursor and limit, using indexes to avoid slow full table scans.
- **Edge Cases**: Handle deleted posts (tombstone records), privacy settings (filter on read), viral content (cache hot posts), media corruption (validate uploads), and feed inconsistencies (recompute timelines).
- **Monitoring**: Track feed latency, media load time, cache hit ratio, and post retrieval success rate.
- **Trade-offs:**
  - **Push vs. Pull**: Push ensures fast feeds but high write cost for influencers; pull is write-efficient but slower. Hybrid balances both for active users.
  - **Cassandra vs. DynamoDB**: Cassandra handles high write throughput; DynamoDB is simpler but costlier. Choose Cassandra for write-heavy posts.
  - **S3 vs. Custom Storage**: S3 is managed but costly; custom storage is cheaper but complex. Use S3 for reliability.
- **Why Hybrid + Cassandra?**: Hybrid optimizes for active/inactive users, and Cassandra scales for frequent posts.
- **Scalability Estimates**: Supports ~100M daily active users, ~1B posts/day, and ~50K QPS for feed retrieval with Cassandra and CDN scaling.
- **Latency/Throughput Targets**: Target <200ms for feed generation, <100ms for media retrieval, and ~10K posts/sec ingestion.

### 7. Dropbox
**Improvements:**
- **File Storage**: Store files in S3 with metadata (file_id, user_id, version, path) in PostgreSQL for relational queries. Use presigned URLs to let clients upload files directly to Blob Storage, bypassing the backend for faster, cheaper uploads. The client requests a URL with file metadata (saved as "uploading"), uploads the file via PUT. Backend generates the URL, saves metadata as "uploading", and updates it to "uploaded" upon receiving a notification from Blob Storage post-upload. These URLs are generated with a specific expiration time, after which they become invalid, offering a secure way to share files without altering permissions.
- **Downloading**: Use a CDN to cache files on global servers, serving them from the closest location to the user for reduced latency and faster downloads. Generate secure, time-limited URLs for users to download files from the CDN, similar to S3 presigned URLs.
- **Sharing files**: Create a SharedFiles table linking userId to fileId to track shared files. To find a user's files, check this table for their userId. No need to manage a separate sharelist. Downside: searching the table is a bit slower than a direct lookup.
- **Versioning**: Store file deltas in S3 to save space; maintain version history in DB.
- **Syncing**: Use WebSockets for real-time sync notifications and Kafka for file change events or polling(if realtime not important).
- **Make everything fast**: Speed up uploads, downloads, and syncing by using a CDN for low-latency downloads, chunking files for parallel uploads and selective syncing of changed chunks, and compressing files (using Gzip, Brotli, or Zstandard) based on file type, size, and network conditions. Compress before encrypting for better ratios, but skip compression for media files like images/videos with low compression gains.
- **Support Large Files**: Support large files by chunking them into 5-10MB pieces for upload to S3, using client-side fingerprinting (SHA-256) for unique file and chunk identification. Track upload progress in a FileMetadata table with a chunks field, updated via S3 event notifications. Enable resumable uploads by checking uploaded chunks and use AWS Multipart Upload API for efficient handling. Show progress indicators for better user experience.
- **Deduplication**: Compute file hashes to store identical files once, referencing via metadata.
- **File Security:**: Ensure file security with HTTPS for encryption in transit, S3 encryption at rest using unique keys, and access control via a shareList or table. Use signed URLs with short expiration times (e.g., 5 minutes) for downloads, compatible with CDNs like CloudFront, to prevent unauthorized access even if links are shared.
- **Edge Cases**: Handle concurrent edits (conflict resolution with versioning), large files (chunked uploads), offline access (local caching), file corruption (validate on upload), and storage quotas (enforce limits).
- **Monitoring**: Track sync latency, storage usage, deduplication rate, and file upload success rate.
- **Trade-offs:**
  - **S3 vs. Custom Storage**: S3 is managed but costly; custom storage is cheaper but complex. Choose S3 for reliability and scalability.
  - **PostgreSQL vs. NoSQL**: PostgreSQL excels for relational metadata; NoSQL is flexible but complex for joins. Choose PostgreSQL for file relationships.
  - **WebSockets vs. Polling**: WebSockets are real-time but complex; polling is simpler but delayed. Choose WebSockets for UX.
- **Why S3 + PostgreSQL?**: S3 ensures durable storage, and PostgreSQL handles complex metadata queries.
- **Scalability Estimates**: Supports ~10M daily active users, ~1PB storage, and ~1K file uploads/sec with S3 and PostgreSQL scaling.
- **Latency/Throughput Targets**: Target <500ms for file sync, <1s for uploads, and ~1K files/sec processing.

### 8. YouTube or Netflix
**Improvements:**
- **Video Storage**: Store videos in S3 with metadata (video_id, title, uploader_id) in DynamoDB. Use HLS/DASH for adaptive streaming. Allow video uploads by storing metadata (name, description) in a scalable database like Cassandra, partitioned by videoId for efficient point lookups. Store video data in S3 using presigned URLs for direct, multi-part uploads. Post-process videos by splitting them into short, playable segments in multiple formats for efficient streaming, managed by a complex pipeline that stores segment references for downstream use.
- **Video Watching**: Enable video watching with adaptive bitrate streaming: fetch VideoMetadata from the database, containing a URL to a manifest file in S3. The client downloads the manifest, selects video segments based on network conditions, and dynamically switches formats (e.g., lower resolution for slower networks) to ensure smooth playback. This relies on pre-segmented videos in multiple formats and a complex client-side logic.
- **CDN**: Use Akamai/Cloudflare to cache videos globally, reducing origin load.
- **Bitrate streaming**: Process videos for adaptive bitrate streaming by uploading the full video to S3, then using a DAG-based pipeline (orchestrated by tools like Temporal) to: split the video into segments with ffmpeg, transcode segments into multiple formats in parallel on worker nodes, generate audio/transcripts, and create manifest files referencing segments. Store segments and manifests in S3, using URLs to pass data between workers, and mark the upload as complete.
- Support **resumable video** uploads by chunking the video into 5-10MB pieces with fingerprints, tracking chunk status in VideoMetadata. Upload chunks to S3, update status via S3 event notifications, and resume by skipping already-uploaded chunks based on VideoMetadata.
- **Recommendation**: Compute personalized recommendations (collaborative filtering) via Spark jobs; cache in Redis.
- **Sharding**: Shard video metadata by video_id; partition user data by user_id.
- **Edge Cases**: Handle video buffering (pre-fetch chunks), geo-restrictions (IP-based filtering), high demand (scale CDN), video corruption (validate uploads), and recommendation staleness (periodic refresh).
- **Monitoring**: Track streaming latency, recommendation click-through rate, CDN hit ratio, and video playback success rate.
- **Trade-offs:**
  - **Pre-compute vs. Real-time Recs**: Pre-compute is efficient but stale; real-time is fresh but compute-heavy. Choose pre-compute with periodic updates.
  - **DynamoDB vs. Cassandra**: DynamoDB is simpler but costlier; Cassandra scales better for writes. Choose DynamoDB for ease of use.
  - **S3 vs. Custom Storage**: S3 is reliable but costly; custom storage is cheaper but complex. Use S3 for durability.
- **Why S3 + CDN?**: S3 provides durable storage, and CDN ensures low-latency video delivery.
- **Scalability Estimates**: Supports ~1B daily active users, ~10PB storage, and ~100K QPS for streaming with S3 and CDN scaling.
- **Latency/Throughput Targets**: Target <200ms for video start, <500ms for recommendations, and ~50K streams/sec.

### 9. Yelp or Nearby Friends
**Improvements:**
- **Geospatial Index**: Use Redis Geo or PostgreSQL PostGIS to store and query locations (business_id, lat, lon).
- **Search**: Index business metadata in Elasticsearch for fast text and geo-based searches.
- **Caching**: Cache popular searches and nearby results in Redis (TTL-based eviction).
- **Edge Cases**: Handle location spoofing (IP validation), stale data (periodic refresh), high QPS (load balancing), privacy settings (filter results), and inaccurate coordinates (validate inputs).
- Efficiently calculate and update business average **ratings** by storing a precomputed average in the business record, updated incrementally with each new review using a formula (e.g., (old_avg * old_count + new_rating) / (old_count + 1)). Avoid on-the-fly calculations or complex message queues, as the low write volume (~1 write/second for 100M users) is easily handled by modern databases.
- Enforce a **one-review-per-business** constraint by adding a unique composite key or index on userId and businessId in the reviews table at the database level (e.g., using a unique constraint in PostgreSQL or DynamoDB). This prevents duplicate reviews, ensures consistency at the persistence layer, and avoids complex application-layer checks.
- To efficiently handle **complex search queries**, use **quadtree indexing** for geospatial searches to avoid slow full-table scans on latitude/longitude. First, filter by distance using the Haversine formula (determines the great-circle distance between two points on a sphere given their longitudes and latitudes) to shrink the result set, then apply filters like name or category. Prioritize distance filtering to quickly reduce the search space for faster, scalable searches.
- To **search by predefined location names** like cities or neighborhoods, create a locations table mapping names (e.g., "San Francisco", "Mission") to polygons from datasets like GeoJSON, indexed for fast lookups. Pre-compute and store location identifiers (e.g., "san_francisco", "mission_district") in each business’s record using an inverted index (e.g., Elasticsearch keyword field). Query the locations table to get the polygon, then filter businesses by matching pre-computed location identifiers, avoiding costly per-request polygon checks.
- **Monitoring**: Track search latency, geo-query accuracy, cache hit ratio, and search success rate.
- **Trade-offs:**
  - **Redis Geo vs. PostGIS**: Redis is faster for simple queries; PostGIS handles complex geospatial ops. Choose Redis for low-latency lookups.
  - **Elasticsearch vs. DB**: Elasticsearch is optimized for search; DB is durable but slower. Use Elasticsearch for fast queries.
  - **Caching vs. No Caching**: Caching reduces latency but risks staleness; no caching is fresh but slower. Use caching with TTL for balance.
- **Why Redis + Elasticsearch?**: Redis ensures fast geo-queries, and Elasticsearch handles complex searches.
- **Scalability Estimates**: Supports ~10M daily active users, ~100K QPS for searches, and ~1M businesses with Redis and Elasticsearch scaling.
- **Latency/Throughput Targets**: Target <100ms for geo-queries, <200ms for search results, and ~5K searches/sec.

## Write-Heavy Systems
These prioritize high throughput, durability, and ingestion speed.

### 10. Rate Limiter
**Improvements:**
- **Algorithm**: Use token bucket for flexibility (allows bursts) over leaky bucket (fixed rate). Store tokens in Redis with {user_id, tokens, last_refill_timestamp}.
- **TTL Logic**: Set Redis TTL to bucket refresh interval (e.g., 1 minute) to auto-expire stale counters.
- **Distributed**: Use Redis cluster for scalability and Lua scripts for atomic updates to avoid race conditions. Each host has a Token Bucket. Requests for same client can hit different hosts. Hosts must communicate token consumption.
- **Negative Tokens:** In distributed systems, request bursts from a single client can be routed to different servers, potentially exceeding the intended rate limit. To address this, servers allow token buckets to have temporary negative balances. This ensures that throttling occurs for a sufficient period after the burst, effectively compensating for the initial excess and maintaining the overall rate limit.
- **Inter-Host Communication:** To enforce rate limits across a distributed system, servers must share information about client request counts. This can be achieved through methods like full-mesh communication (simple but not scalable), a more scalable gossip protocol, or a shared cache. Alternatively, a coordination service can elect a leader to manage the rate calculations. When considering how servers communicate, factors like reliability and speed must be balanced, often leading to a choice between TCP and UDP protocols.
- **Memory Management:** Storing token buckets for every client can consume significant memory. To optimize this, buckets for inactive clients can be removed after a defined timeout period. When an inactive client sends a new request, a fresh token bucket is then created. This approach prevents the system from being burdened with excessive storage for clients that are not actively using the service.
- **Rule Management:** Service teams require a tool to define and manage rate-limiting rules, specifying limits for various clients or API endpoints. This tool should provide functionality to create, update, and delete these rules, which are typically stored in a database. The rate-limiting system then retrieves and enforces these rules from the database.
- **Edge Cases**: Handle clock skew (use monotonic clocks), bursty traffic (adjust bucket size), DDoS attacks (global rate limits), token exhaustion (queue requests), and misconfigured limits (audit logs).
- **Monitoring**: Track rejection rate, token refill latency, Redis throughput, and rate limit accuracy.
- **Trade-offs:**
  - **Token vs. Leaky Bucket**: Token bucket allows bursts, suitable for APIs; leaky bucket enforces strict rates, better for hardware. Choose token bucket for API flexibility.
  - **Redis vs. In-Memory**: Redis is durable and distributed; in-memory is faster but volatile. Choose Redis for reliability.
  - **Global vs. Per-User Limits**: Global limits prevent abuse but are coarse; per-user limits are precise but complex. Use both for comprehensive control.
- **Why Token Bucket + Redis?**: Token bucket handles bursts gracefully, and Redis scales for distributed systems.
- **Scalability Estimates**: Supports ~100K QPS for rate checks, ~1M users, and ~10K limits/sec with Redis cluster scaling.
- **Latency/Throughput Targets**: Target <10ms for rate checks, ~50K checks/sec, and <50ms for token refills.

### 11. Log Collection and Analysis System
**Improvements:**
- **Ingestion**: Use Kafka for high-throughput log ingestion with topic partitioning by source or timestamp.
- **Processing**: Use ELK stack (Elasticsearch for storage, Logstash for parsing, Kibana for visualization). Add Spark for batch analytics.
- **Buffering**: Buffer logs in Kafka to handle spikes; use consumer groups for parallel processing.
- **Real-time Querying**: Index logs in Elasticsearch with time-based indices for fast searches.
- **Edge Cases**: Handle log spikes (scale Kafka partitions), malformed logs (validate before ingestion), data retention (TTL policies), consumer lag (scale workers), and log duplication (idempotency keys).
- **Monitoring**: Track ingestion latency, query response time, partition lag, and log processing success rate.
- **Trade-offs:**
  - **Kafka vs. RabbitMQ**: Kafka is better for high-throughput logs; RabbitMQ is simpler but less scalable. Choose Kafka for durability and partitioning.
  - **Elasticsearch vs. ClickHouse**: Elasticsearch is flexible for text search; ClickHouse is faster for analytics. Choose Elasticsearch for general-purpose querying.
  - **Real-time vs. Batch Processing**: Real-time is fast but resource-heavy; batch is efficient but delayed. Use both for balance.
- **Why Kafka + ELK?**: Kafka ensures reliable ingestion, and ELK provides robust search and visualization.
- **Scalability Estimates**: Handles ~1M logs/sec, ~1PB storage, and ~10K QPS for queries with Kafka and Elasticsearch scaling.
- **Latency/Throughput Targets**: Target <100ms for log ingestion, <500ms for queries, and ~100K logs/sec processing.

### 12. Voting System
**Improvements:**
- **Idempotency**: Use a unique vote_id (user_id + poll_id) to prevent duplicate votes. Store in Redis for fast checks.
- **Fraud Prevention**: Add rate-limiting and CAPTCHA for suspicious activity. Log votes in a relational DB (PostgreSQL) for auditing.
- **Aggregation**: Use Redis for real-time vote counts (increment counters) and batch jobs (Spark) for final tallies.
- **Edge Cases**: Handle vote reversals (store vote history), network failures (retry logic), result disputes (audit trails), voter impersonation (auth checks), and high vote spikes (scale Redis).
- **Monitoring**: Track vote throughput, fraud detection rate, count accuracy, and vote processing latency.
- **Trade-offs:**
  - **Real-time vs. Eventual Counts**: Real-time is user-friendly but risks inconsistency; eventual is consistent but delayed. Choose real-time with periodic reconciliation for UX.
  - **Redis vs. DB**: Redis is fast for counters but volatile; DB is durable but slower. Use Redis with DB backup for speed and reliability.
  - **Rate-Limiting vs. CAPTCHA**: Rate-limiting is lightweight but coarse; CAPTCHA is robust but intrusive. Use both for layered security.
- **Why Redis + PostgreSQL?**: Redis ensures low-latency vote counting, and PostgreSQL provides durable storage.
- **Scalability Estimates**: Supports ~10K votes/sec, ~1M voters, and ~100K QPS for counts with Redis and PostgreSQL scaling.
- **Latency/Throughput Targets**: Target <50ms for vote submission, <100ms for counts, and ~5K votes/sec processing.

### 13. Trending Topics System
**Improvements:**
- **Counting**: Use count-min sketch for approximate counting of topic mentions to save memory. Store in Redis for fast updates.
- **Sliding Window**: Implement a time-based sliding window (e.g., 1 hour) using Redis sorted sets to track recent mentions.
- **Ranking**: Compute trending scores (mentions + recency) via a background job and cache results in Redis.
- **Edge Cases**: Handle sudden spikes (use exponential decay in scoring), stale trends (expire old data), low-memory scenarios (evict cold topics), topic ambiguity (normalize terms), and skewed data (cap outliers).
- **Monitoring**: Track sketch accuracy, ranking latency, cache hit ratio, and trend refresh rate.
- **Trade-offs:**
  - **Count-Min vs. Exact Counting**: Count-min is memory-efficient but approximate; exact counting is accurate but memory-heavy. Choose count-min for scalability.
  - **Sliding vs. Fixed Window**: Sliding window is precise but complex; fixed window is simpler but less granular. Choose sliding for real-time trends.
  - **Redis vs. In-Memory**: Redis is durable and scalable; in-memory is faster but volatile. Use Redis for reliability.
- **Why Count-Min + Redis?**: Count-min scales for high cardinality, and Redis ensures low-latency updates.
- **Scalability Estimates**: Handles ~100K mentions/sec, ~1M topics, and ~10K QPS for rankings with Redis scaling.
- **Latency/Throughput Targets**: Target <50ms for mention updates, <200ms for rankings, and ~10K trends/sec processing.

### 14. Facebook Messenger
**Improvements:**
- **Message Ingestion**: Use Kafka for high-throughput message queues, partitioned by chat_id.
- **Storage**: Store messages in Cassandra {chat_id, message_id, content, timestamp} for scalability.
- **Delivery**: Use WebSockets for real-time delivery and APNs/GCM for push notifications.
- **Edge Cases**: Handle offline users (store messages), duplicates (idempotency keys), group chats (fanout logic), message expiration (TTL policies), and delivery failures (retry logic).
- **Monitoring**: Track message delivery latency, notification success rate, queue backlog, and message processing success rate.
- **Trade-offs:**
  - **Kafka vs. RabbitMQ**: Kafka is durable and scalable; RabbitMQ is simpler but less robust. Choose Kafka for high throughput.
  - **WebSockets vs. Polling**: WebSockets are real-time but complex; polling is simpler but delayed. Choose WebSockets for UX.
  - **Cassandra vs. DynamoDB**: Cassandra is cost-effective for writes; DynamoDB is simpler but costlier. Choose Cassandra for scalability.
- **Why Kafka + Cassandra?**: Kafka ensures reliable ingestion, and Cassandra scales for message storage.
- **Scalability Estimates**: Supports ~100M daily active users, ~1B messages/day, and ~50K QPS for delivery with Kafka and Cassandra scaling.
- **Latency/Throughput Targets**: Target <100ms for message delivery, <200ms for notifications, and ~10K messages/sec ingestion.

### 15. Local Delivery (Gopuff)
**Improvements:**
- **Order Ingestion**: Use Kafka to handle order events, partitioned by region.
- **Inventory**: Store inventory in Redis for real-time updates and PostgreSQL for durability.
- **Routing**: Use OSRM for delivery route optimization; cache routes in Redis.
- **Traffic-Aware Availability Lookup:** To meet the 1-hour delivery requirement, the Nearby Service enhances basic distance checks by periodically syncing DC location data into memory. On each request, it first filters DCs within a generous fixed radius (e.g. 60 miles), then sends those to an external Travel Time Service that accounts for traffic, roads, borders, and geography. Only DCs with actual drive times under 1 hour are considered for inventory lookup—ensuring more accurate, real-world availability results.
- **Placing Orders (Strong Consistency)**: Customers place orders via the Orders Service, which uses a SERIALIZABLE Postgres transaction for atomicity. The transaction checks inventory, records the order, and updates stock in one step to prevent double booking. All inventory and order data is colocated in the same ACID-compliant DB for simplicity and strong consistency. If any item is out of stock, the transaction fails and returns a clear error to the user. This approach favors correctness over partial order success.
- **Fast & Scalable Availability Lookup**: Estimating from 10M orders/day and accounting for browsing behavior, we may see ~20K availability queries/sec. Hitting the DB directly doesn’t scale, so we optimize by caching. The Nearby Service syncs DC data to memory, and availability results are cached (e.g. in Redis) by item ID + DC ID. Availability Service first checks the cache; on a miss, it queries Postgres, updates the cache, and returns results—reducing DB load and improving latency.
- **Edge Cases**: Handle out-of-stock items (real-time checks), delivery delays (re-routing), order cancellations (rollback), driver unavailability (reassign orders), and traffic disruptions (recompute routes).
- **Monitoring**: Track order latency, inventory accuracy, delivery ETA, and order success rate.
- **Trade-offs:**
  - **Redis vs. DB for Inventory**: Redis is fast but volatile; DB is durable but slower. Use Redis with DB backup.
  - **Real-time vs. Batch Routing**: Real-time is accurate but compute-heavy; batch is efficient but stale. Choose real-time for UX.
  - **Kafka vs. RabbitMQ**: Kafka is scalable for events; RabbitMQ is simpler but less robust. Use Kafka for high throughput.
- **Why Kafka + Redis?**: Kafka handles high-throughput orders, and Redis ensures fast inventory checks.
- **Scalability Estimates**: Supports ~10K orders/sec, ~1M daily orders, and ~100K QPS for inventory checks with Kafka and Redis scaling.
- **Latency/Throughput Targets**: Target <100ms for inventory checks, <500ms for routing, and ~5K orders/sec processing.

### 16. Strava
**Improvements:**
- **Activity Ingestion**: Use Kafka to process GPS and fitness data streams, partitioned by user_id.
- **Storage**: Store activities in Cassandra {activity_id, user_id, gps_points, timestamp} for high write throughput.
- **Analytics**: Use Spark for leaderboards and activity stats; cache results in Redis.
- **Offline Activity Tracking**: Since we don’t support real-time sharing, we record GPS data locally using on-device sensors and store it in memory, persisting to local storage (e.g. Core Data/Room DB) every ~10s to prevent loss. On activity completion or when online, data is uploaded to the server in bulk (optionally chunked). This reduces network reliance, saves bandwidth, and simplifies client UX—highlighting the importance of treating the client as an active system component.
- **Supporting Real-time Activity Sharing:** To enable friends to follow along in real-time, we send location updates every 2-5 seconds from the athlete's device to the server. Instead of complex WebSockets/SSE, we use a simpler polling mechanism where friends’ clients request updates at the same interval, with a slight offset. Since updates are predictable and slight delays are acceptable, this avoids over-engineering. To improve the user experience, we implement a smart buffering system, intentionally delaying updates by a few seconds for smoother, continuous movement, mimicking a live-stream experience while compensating for network latency.
- **Realtime Leaderboard:** Use Redis Sorted Sets for real-time leaderboards, storing user IDs as members and total distances as scores. For country-specific leaderboards, create sets like leaderboard:run:USA. For time range filtering, combine sorted sets for timestamps with hashes for activity data, using ZRANGEBYSCORE to retrieve activity IDs within the range. Aggregate distances by user ID and sort to generate the leaderboard. Cache results in Redis with TTL for efficiency. Ensure data consistency and handle Redis memory limitations by caching frequently accessed leaderboards.
- **Edge Cases**: Handle GPS errors (filter outliers), large uploads (chunked processing), privacy settings (filter on read), activity duplicates (idempotency keys), and incomplete data (validate inputs).
- **Monitoring**: Track ingestion latency, analytics accuracy, cache hit ratio, and activity processing success rate.
- **Trade-offs:**
  - **Cassandra vs. DynamoDB**: Cassandra is cost-effective for writes; DynamoDB is simpler but costlier. Choose Cassandra for scalability.
  - **Real-time vs. Batch Analytics**: Real-time is fresh but heavy; batch is efficient but stale. Choose batch with periodic updates.
  - **Kafka vs. RabbitMQ**: Kafka is durable for streams; RabbitMQ is simpler but less robust. Use Kafka for high throughput.
- **Why Kafka + Cassandra?**: Kafka ensures reliable data ingestion, and Cassandra scales for activity storage.
- **Scalability Estimates**: Supports ~1M daily active users, ~10M activities/day, and ~10K QPS for analytics with Kafka and Cassandra scaling.
- **Latency/Throughput Targets**: Target <200ms for activity ingestion, <500ms for analytics, and ~5K activities/sec processing.

### 17. FB Live Comments
**Improvements:**
- **Comment Ingestion**: Use Kafka for high-throughput comment streams, partitioned by video_id.
- **Storage**: Users can post comments on a live video by sending a POST request to POST /comments/:liveVideoId with the comment message. The request is validated by the server, and the comment is stored in a DynamoDB database. The commenter client sends the comment to the comment management service, which stores it and later retrieves comments for display. DynamoDB is used for its scalability and speed. While storing and querying comments is simple, complexities arise in how comments are delivered to viewers in real-time.
To allow viewers to **see comments made before they joined a live feed**, we use infinite scrolling with cursor-based pagination. Upon joining, users can fetch recent comments with GET /comments/:liveVideoId, using cursor pagination to retrieve the N most recent comments before a certain timestamp. Cursor pagination is more efficient than offset pagination, ensuring stable scrolling and consistent performance even as comment volume grows. DynamoDB's LastEvaluatedKey feature works well with this approach, ensuring scalability and reliability.
We use Server-Sent Events (SSE) instead of WebSockets for **real-time comment broadcasting** because SSE is simpler, more lightweight, and better suited for one-way communication (server to client). WebSockets require a persistent bi-directional connection and more complex management, which adds overhead. SSE is easier to implement, scales better for broadcast scenarios, and handles message delivery in a more efficient manner for our use case, where the server pushes new comments to multiple clients. However, scalability challenges still exist, which will need to be addressed in future improvements.
To **scale SSE for millions of concurrent viewers**, we must horizontally scale by adding multiple servers since a single server cannot handle the load. The challenge arises when viewers are connected to different servers, making it difficult to broadcast comments to all viewers of the same live video. For example, Server 1 can send a comment to UserA, but not to UserB connected to Server 2. To solve this, we can use a pub/sub system like Redis or Kafka to ensure messages are broadcasted to all servers handling viewer connections. Redis offers low latency but lacks message persistence, while Kafka provides guaranteed delivery and persistence but with higher latency. The choice between these systems depends on the tradeoffs of scalability, message guarantees, and operational complexity.
**But this is better:**
To allocate viewers of the same live video to the same server, we can use two strategies: 1) Intelligent Routing via Scripts or Configuration, where a Layer 7 load balancer like NGINX or Envoy uses consistent hashing based on liveVideoId to route viewers to the same server. 2) Dynamic Lookup via Coordination Service, where a service like Zookeeper stores liveVideoId-to-server mappings and updates as needed. While both approaches are valid, the former is simpler, while the latter provides more flexibility at the cost of added complexity.
- **Delivery**: Use WebSockets to push comments to viewers in real-time.
- **Edge Cases**: Handle comment spam (rate-limiting), deleted comments (tombstones), high QPS (scale WebSockets), comment ordering (sequence numbers), and viewer lag (buffer comments).
- **Monitoring**: Track comment latency, delivery success rate, queue backlog, and comment processing success rate.
- **Trade-offs:**
  - **Kafka vs. RabbitMQ**: Kafka scales for high throughput; RabbitMQ is simpler but less robust. Choose Kafka for live comments.
  - **WebSockets vs. Polling**: WebSockets are real-time but complex; polling is simpler but delayed. Choose WebSockets for UX.
  - **Cassandra vs. DynamoDB**: Cassandra is cost-effective for writes; DynamoDB is simpler but costlier. Choose Cassandra for scalability.
- **Why Kafka + WebSockets?**: Kafka handles comment ingestion, and WebSockets ensure real-time delivery.
- **Scalability Estimates**: Supports ~100K comments/sec, ~1M viewers/video, and ~50K QPS for delivery with Kafka and WebSocket scaling.
- **Latency/Throughput Targets**: Target <100ms for comment delivery, <200ms for ingestion, and ~10K comments/sec processing.

## Strong Consistency Systems
These prioritize transactional integrity and failure handling.

### 18. Online Ticket Booking System
**Improvements:**
- **Concurrency**: To improve the booking experience by reserving tickets, we can implement a distributed lock mechanism with a Time-To-Live (TTL). When a user selects a ticket, the Booking Service attempts to acquire a lock on that ticket in a distributed lock system (like Redis) with a 10-minute TTL. Simultaneously, a booking record is created in the database with an "in-progress" status, and the user is redirected to the payment page with a bookingId. If the user completes payment within the TTL, a webhook from the payment processor triggers a database transaction, updating both the ticket status to "sold" and the booking status to "confirmed." If the user abandons the process or the TTL expires, the lock is automatically released, making the ticket available again. This approach prevents overselling and provides a better user experience by temporarily reserving tickets during checkout.
- To **handle tens of millions of concurrent view requests during peak events**, a multi-faceted approach combining caching, load balancing, and horizontal scaling is crucial. We'll heavily cache static and infrequently changing data like event details and venue information in a fast in-memory store (Redis/Memcached) using a read-through strategy with appropriate Time-to-Live (TTL) policies. Database triggers will ensure cache invalidation upon data updates. Load balancers will distribute incoming traffic evenly across multiple stateless instances of the view API service (horizontal scaling). While maintaining cache consistency and managing a large number of instances present operational challenges, this combination will significantly reduce database load and ensure high availability.
- To ensure a **good user experience during high-demand booking events with millions of concurrent users**, a virtual waiting queue system can be implemented for extremely popular events. When a user attempts to access the booking page, they are placed in a queue, and a persistent WebSocket connection is established. Users are dequeued based on a set schedule or criteria and notified via their WebSocket connection when it's their turn to proceed to the ticket selection process. Simultaneously, the database is updated to grant these users access. While long wait times could be a concern, providing real-time updates on their queue position and estimated wait time via the WebSocket connection can help manage user expectations and mitigate frustration. This approach controls the flow of users, preventing system overload and ensuring a smoother experience for everyone.
- **Reservation Flow**: Reserve seats in Redis (with TTL) → process payment → confirm in DB (PostgreSQL).
- **Edge Cases**: Handle payment failures (rollback reservation), expired reservations (auto-release seats), oversold tickets (audit logs), double bookings (idempotency keys), and high demand (queue requests).
- **Monitoring**: Track booking success rate, lock contention, payment latency, and seat reservation accuracy.
- **Trade-offs:**
  - **Optimistic vs. Pessimistic Locking**: Optimistic is scalable but risks conflicts; pessimistic is safe but slower. Choose optimistic for most cases, with pessimistic for high-demand events.
  - **Redis vs. DB for Reservation**: Redis is fast but volatile; DB is durable but slower. Use Redis with DB confirmation for speed and reliability.
  - **Queue vs. No Queue**: Queuing handles spikes but adds latency; no queuing is fast but risks failures. Use queuing for high-contention events.
- **Why Optimistic + Redis/DB?**: Optimistic locking scales for typical loads, and Redis ensures fast reservations with DB for durability.
- **Scalability Estimates**: Supports ~10K bookings/sec, ~1M daily tickets, and ~50K QPS for reservations with Redis and PostgreSQL scaling.
- **Latency/Throughput Targets**: Target <200ms for seat reservation, <500ms for booking, and ~5K bookings/sec processing.

### 19. E-Commerce Website (Amazon)
**Improvements:**
- **Catalog**: Store products in a NoSQL DB (DynamoDB) for flexible schemas. Index in Elasticsearch for fast search.
- **Cart**: Use Redis for in-memory carts (user_id: cart_items) with TTL to handle abandoned carts.
- **Order Processing**: Use a saga pattern (Kafka + microservices) for distributed transactions (inventory, payment, shipping).
- **Idempotency**: Ensure checkout is idempotent using a unique order_id.
- **Edge Cases**: Handle out-of-stock items (real-time inventory checks), failed payments (retries), partial orders (rollback), cart conflicts (optimistic locking), and order fraud (anomaly detection).
- **Monitoring**: Track checkout success rate, search latency, inventory accuracy, and order processing latency.
- **Trade-offs:**
  - **Monolith vs. Microservices**: Monolith is simpler but less scalable; microservices are flexible but complex. Choose microservices for scalability.
  - **Eventual vs. Strong Consistency**: Eventual is scalable but risks overselling; strong is safe but slower. Use strong consistency for inventory.
  - **DynamoDB vs. Cassandra**: DynamoDB is simpler but costlier; Cassandra is scalable but complex. Choose DynamoDB for ease of use.
- **Why Microservices + DynamoDB?**: Microservices scale independently, and DynamoDB handles high read/write loads.
- **Scalability Estimates**: Supports ~100M daily active users, ~1M orders/sec, and ~100K QPS for searches with DynamoDB and Elasticsearch scaling.
- **Latency/Throughput Targets**: Target <100ms for searches, <500ms for checkout, and ~10K orders/sec processing.

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

### 32. Ride-Sharing App (Uber/Lyft)
**Improvements:**
- **Matchmaking**: Use a geospatial index (Redis Geo or PostgreSQL PostGIS) to match riders with nearby drivers.
- **Location Tracking**: Stream driver locations via Kafka, processed by a real-time engine (Flink).
- **ETA Algorithms**: Use graph-based routing (OSRM) with traffic data for accurate ETAs.
- **Surge Pricing**: Compute dynamic pricing based on supply/demand ratios, cached in Redis.
- **Edge Cases**: Handle driver cancellations (re-match logic), GPS errors (filter outliers), high demand (queue riders), surge abuse (cap multipliers), and network failures (retry logic).
- **Monitoring**: Track match latency, ETA accuracy, surge multiplier, and match success rate.
- **Trade-offs:**
  - **Redis Geo vs. PostGIS**: Redis is faster for simple queries; PostGIS is robust for complex geospatial operations. Choose Redis for low-latency matching.
  - **Real-time vs. Batch ETA**: Real-time is accurate but compute-heavy; batch is efficient but stale. Choose real-time for UX.
  - **Kafka vs. RabbitMQ**: Kafka is scalable for streams; RabbitMQ is simpler but less robust. Use Kafka for high throughput.
- **Why Redis + Kafka?**: Redis ensures fast geospatial queries, and Kafka handles real-time location streams.
- **Scalability Estimates**: Supports ~10M daily active users, ~1M rides/day, and ~50K QPS for matching with Redis and Kafka scaling.
- **Latency/Throughput Targets**: Target <100ms for driver matching, <500ms for ETA, and ~10K matches/sec processing.

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

## Aggregation Systems
These focus on processing and ranking high-cardinality data.

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
