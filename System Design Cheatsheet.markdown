https://github.com/shivangbelwariar96/HelloInterview/blob/main/Numbers-To-Remember.md



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


### 35. FB Post Search (we're not allowed to use a search engine like Elasticsearch or a pre-built full-text index (like Postgres Full-Text)):
- To enable users to **search posts** by keyword efficiently, we'll create an inverted index. This index will map keywords to the IDs of the posts that contain them. When a new post is ingested, the Ingestion Service will tokenize its content into individual keywords and then append the post's ID to the list associated with each of these keywords in our inverted index. For fast lookups, we'll store this inverted index in Redis, leveraging its in-memory capabilities. When a user searches for a keyword, we can directly retrieve the list of matching post IDs from Redis and return the corresponding posts. This approach provides significantly faster search compared to querying an unindexed database.
- To enable **sorting search results by recency or like count**, we'll maintain multiple indexes in Redis. For recency-based sorting, we'll use Redis lists. When a new post is created, its ID will be appended to the list associated with each of its keywords. For sorting by like count, we'll use Redis sorted sets. The score in the sorted set will represent the post's like count, and the value will be the post's ID. When a new post is created, its ID will be added to the sorted set for each of its keywords with an initial score of zero. When a like event occurs, we'll update the score of the corresponding post ID in the relevant sorted sets. This approach allows for efficient retrieval of results sorted by either criterion directly from the appropriate index.
- To **handle the large volume of search requests**, we'll implement aggressive caching at the edge using a Content Delivery Network (CDN) like Cloudflare or CloudFront. We'll configure our /search endpoint to include cache-control headers, instructing the CDN to cache search results. When a user performs a search, their request will first hit a geographically close CDN node. If the result is cached, it will be served with very low latency. If it's a cache miss, the CDN will forward the request to our API gateway and search service as usual. Combined with our in-memory Redis search cache, the CDN will significantly reduce the load on our origin servers and improve response times for the majority of search queries, especially for popular and repeated searches, leveraging our non-personalized search requirement.
- To **handle multi-keyword phrase** queries efficiently, we can implement bigrams (or shingles). During post ingestion, we'll create tokens for each consecutive pair of words (e.g., "Taylor Swift" becomes "Taylor Swift"). These bigram tokens will also be indexed in our "Likes" and "Creation" Redis indexes, alongside single-word keywords. When a user searches for a phrase like "Taylor Swift", we can directly look up the bigram "Taylor Swift" in our indexes and retrieve the relevant post IDs, rather than performing an intersection of the results for "Taylor" and "Swift". While this increases the index size, it significantly speeds up phrase queries, a common search pattern. We might consider probabilistic methods to index only frequent bigrams to mitigate the size increase.
- To address the **high volume of writes**, particularly for likes, we'll implement a two-stage architecture. For post creation, we'll use Kafka to buffer and distribute ingestion requests across multiple service instances, and we'll shard our Redis indexes by keyword to distribute the write load. For likes, instead of updating the index on every like, we'll only update the like count in our Redis "Likes" index when the count reaches specific milestones (e.g., powers of two). This significantly reduces write frequency, at the cost of the index being an approximation. To provide accurate, up-to-date like counts for sorting, when retrieving search results, we'll fetch a larger set of potential results from the approximate "Likes" index and then query a dedicated Like Service for the precise, current like counts for these posts before the final sorting and ranking. This two-stage approach balances write efficiency with read accuracy.
- To **optimize storage**, we can implement a tiered approach. First, we'll cap the size of our inverted indexes (e.g., to 1k-10k post IDs per keyword), significantly reducing storage for common terms. Second, we'll identify rarely searched keywords based on usage analytics and move their corresponding indexes from our in-memory Redis to cheaper, less frequently accessed blob storage like S3 or R2. When a search query comes in, we'll first check Redis. If the keyword's index isn't found, we'll retrieve it from blob storage, incurring a slight latency penalty for these less common searches. This strategy balances query performance for popular terms with cost-effective storage for the vast majority of less frequently accessed data.



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



