
<img width="1120" alt="Screenshot 2025-04-13 at 3 59 32 PM" src="https://github.com/user-attachments/assets/4f4fa80f-2578-42ae-a6af-6f5341a05a07" />




# System Design: URL Shortener Like Bit.ly

This is a concise, interview-ready summary for designing a URL shortener like Bit.ly, tailored for quick review and covering all critical points to ensure you succeed in the interview. It omits unnecessary details (e.g., full DB schemas, verbose explanations) but includes everything essential for a junior-to-mid-level candidate. The format is clear, focused, and optimized for studying alongside your 20+ other system designs.

---

## Overview
**Name**: URL Shortener (Bit.ly-like)  
**Purpose**: Convert long URLs into short, shareable links and redirect users to the original URL.  
**Scope**: Core features (shorten URLs, custom aliases, expiration, redirection). Excludes authentication, analytics, spam detection.

---

## Functional Requirements
- **Shorten URL**: Users submit a long URL and receive a short URL.  
- **Custom Alias**: Users can optionally specify a custom alias for the short URL.  
- **Expiration**: Users can optionally set an expiration date for the short URL.  
- **Redirect**: Users access the short URL to reach the original URL.

---

## Non-Functional Requirements
- **Uniqueness**: Short codes must be unique (no two long URLs map to the same short URL).  
- **Performance**: Redirection latency <100ms.  
- **Availability**: 99.99% uptime, prioritizing availability over consistency.  
- **Scalability**: Support 1B shortened URLs, 100M daily active users (DAU), with ~1000:1 read-to-write ratio (e.g., 100K writes/day, 100M reads/day).  

---

## API Design
Simple REST APIs aligned with functional requirements, ensuring clarity and security.

1. **Shorten URL**:  
   - `POST /urls`  
   - Body: `{ long_url: string, custom_alias: string (optional), expiration_date: string (optional) }`  
   - Returns: `{ short_url: "http://short.ly/abc123" }`  
   - Purpose: Create a short URL mapping.  

2. **Redirect**:  
   - `GET /{short_code}`  
   - Returns: HTTP 302 redirect to original URL.  
   - Purpose: Fetch and redirect to long URL.

---

## Database
**Choice**: SQL (e.g., Postgres) for simplicity and indexing efficiency, given low write throughput.  
**Key Entities**:  
- **URL Mapping**: Short code, long URL, custom alias (if any), expiration date, creation time.  
**Indexing**:  
- Primary key on `short_code` for fast redirects.  
- Index on `long_url` to check duplicates.  
**Size Estimate**: ~500 bytes/row × 1B URLs = 500GB, manageable with a single DB or sharding if needed.

---

## High-Level Design
**Architecture**: Monolith for simplicity, given low write volume and focused scope.  
**Components**:  
1. **Client**: Web/mobile app for URL submission and redirection.  
2. **Primary Server**: Handles URL creation, validation, and redirects.  
3. **Database**: Postgres for URL mappings.  
4. **Cache**: Redis for fast redirect lookups.  
5. **Counter Service**: Redis for generating unique short codes (optional, for scale).

**Data Flow**:  
- **Shorten**: Client → Primary Server (validate, generate short code) → Database → Client.  
- **Redirect**: Client → Primary Server → Cache/Database → 302 Redirect.  

**Diagram** (verbalize):  
- Client hits Primary Server.  
- Shorten: Server validates, stores in Postgres, returns short URL.  
- Redirect: Server checks Redis, falls back to Postgres, redirects.  
- Redis counter (optional) for unique codes.

---

## Deep Dives
Critical challenges and solutions for non-functional requirements, with clear explanations for interviewer probes.

### 1. Unique Short Codes
**Challenge**: Ensure short codes are unique, short (~6–8 chars), and efficiently generated.  
**Solution**:  
- **Base62 Counter**:  
  - Use a counter (1, 2, 3, …) encoded to Base62 ([0-9A-Za-z], 62 chars).  
  - Example: Counter 123 → Base62 "b9" (6 chars cover ~568M URLs, 7 chars ~35B).  
  - Store counter in Redis, increment atomically for uniqueness.  
  - Batch fetch (e.g., 1,000 values) to reduce Redis calls.  
- **Custom Alias**: Check alias uniqueness in DB before saving.  
- **Why Base62?**: Short codes, no special chars, predictable growth.  
- **Alternative**:  
  - Random Base62: Risks collisions, requires DB checks.  
  - UUID: Too long for short URLs.  
  - Zookeeper (coordinated counters): Overkill for modest write load.  
- **Security Note**: Sequential codes could leak URL count; mitigate with random offset or hashing if private URLs are a concern (not core requirement).  
**Interview Tip**: Propose Base62 counter, mention Redis batching. If probed, discuss random vs. sequential trade-offs or DB fallback.

### 2. Fast Redirects (<100ms)
**Challenge**: Redirect lookups must be fast despite 100M daily reads (~1,200 queries/second average, higher at peak).  
**Solution**:  
- **Redis Cache**:  
  - Cache `short_code` → `long_url` mappings (TTL based on expiration).  
  - Hit rate ~90%+, serving most redirects in <10ms.  
  - Evict expired URLs to save space.  
- **Database Fallback**: Postgres with index on `short_code` for cache misses (~100ms worst case).  
- **Read Replicas**: Scale Postgres reads if cache misses spike.  
- **Why?**: Cache handles high read volume; DB ensures correctness.  
- **Alternative**: DB-only (slow full-table scans); in-memory DB (costly for 500GB).  
**Interview Tip**: Lead with Redis cache, estimate read QPS (~1,200). If probed, mention eviction or replicas. Avoid overcomplicating with CDN unless asked.

### 3. Scalability (1B URLs, 100M DAU)
**Challenge**: Handle 100M daily redirects, 100K daily writes, 500GB data.  
**Solution**:  
- **Read Scaling**:  
  - Redis cluster for redirects, auto-scaling for peak loads (~10K QPS).  
  - Postgres read replicas per region to handle cache misses.  
- **Write Scaling**:  
  - Low write QPS (~1–2 writes/second from 100K/day).  
  - Single Postgres instance suffices; shard by `short_code` if growth exceeds 1TB.  
  - Redis counter scales writes via batching (1,000 IDs per fetch).  
- **High Availability**:  
  - Redis replication (multi-AZ) for counter and cache.  
  - Postgres leader-follower replication, failover to follower if leader dies.  
  - Persist counter to DB periodically for recovery.  
- **Why?**: Cache absorbs reads; DB handles writes easily; replication ensures 99.99% uptime.  
- **Alternative**: Microservices (overkill for low writes); DynamoDB (less indexing flexibility).  
**Interview Tip**: Emphasize cache for reads, low write QPS. If probed, mention sharding or replication. Avoid Zookeeper unless explicitly required.

### 4. Handling Redis Failure
**Challenge**: Redis downtime (counter or cache) could halt writes or slow redirects.  
**Solution**:  
- **Counter Failure**:  
  - Persist last counter batch to Postgres daily.  
  - On Redis restart, query DB for max `short_code`, decode Base62, resume counter.  
  - Multi-AZ replication minimizes downtime.  
- **Cache Failure**:  
  - Fallback to Postgres for redirects (slower but functional).  
  - Warm cache post-recovery by preloading recent URLs.  
- **Why?**: Replication and DB fallback ensure availability; low write QPS limits impact.  
- **Alternative**: No counter (random IDs, collision checks); DB-only (slow redirects).  
**Interview Tip**: Mention replication and DB fallback. If probed, explain counter recovery. Keep it simple, avoid niche failure scenarios.

---

## Interview Checklist
- **Structure** (35 min):  
  1. Requirements: Functional/non-functional (2 min).  
  2. API: List endpoints (2 min).  
  3. Database: SQL, key entity (2 min).  
  4. High-Level: Monolith, components, flow (5 min).  
  5. Deep Dives: Uniqueness, redirects, scalability, failure (15–20 min).  
- **Whiteboard** (verbalize):  
  - Draw: Client → Primary Server → Redis (cache/counter) → Postgres.  
  - Show: Shorten (counter → DB), redirect (cache → DB → 302).  
- **Avoid Pitfalls**:  
  - Don’t use 301 redirects (caching issues); use 302.  
  - Don’t skip cache (DB-only is slow).  
  - Don’t propose random IDs without collision handling.  
  - Don’t suggest microservices or Zookeeper (overkill).  
- **Trade-Offs**:  
  - Base62 counter vs. random: Counter simpler, no collisions.  
  - Cache vs. DB: Cache for speed, DB for durability.  
  - Postgres vs. NoSQL: Postgres for indexing, low writes.  
- **Junior/Mid-Level Expectations**:  
  - Define clear APIs, basic DB model.  
  - Propose Base62 counter, Redis cache, 302 redirects.  
  - Explain flow for shorten/redirect.  
  - Handle probes on uniqueness, speed; expect interviewer to guide deep dives.

---

This summary is lean and focused, covering all must-have points for a Bit.ly-like design, perfect for quick study. Send the next system design, and I’ll format it consistently for your cheatsheet!
