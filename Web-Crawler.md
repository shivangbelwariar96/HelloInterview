# System Design: Web Crawler

This is a concise, interview-ready summary for designing a web crawler, optimized for a 35-minute interview. It covers all critical points to excel, aligning with your 20+ system design cheatsheet for efficient study. The design is "hard" due to fault tolerance, politeness (robots.txt), efficiency (<5 days), and scalability (10B pages).

---

## Overview
**Name**: Web Crawler  
**Purpose**: Traverse the web to extract text data from pages for LLM training.  
**Scope**: Core features (crawl from seed URLs, extract/store text). Excludes dynamic content, non-text data, authentication, LLM training.

---

## Functional Requirements
- **Crawl Web**: Start from seed URLs, follow links to fetch pages.  
- **Extract and Store Text**: Parse pages, store text data.

---

## Non-Functional Requirements
- **Fault Tolerance**: Handle failures, resume without losing progress.  
- **Politeness**: Adhere to robots.txt, avoid overloading servers.  
- **Efficiency**: Crawl 10B pages in <5 days.  
- **Scalability**: Handle 10B pages, ~2MB/page.  
- **Durability**: Store text data reliably.

---

## Scale Estimations
- **Pages**: 10B pages × 2MB = 20PB raw data.  
- **Crawl Time**: 5 days = 432K seconds, 10B ÷ 432K ≈ 23K pages/s.  
- **Storage**: 10B pages × 100KB text/page ≈ 1PB processed.  
- **Requests**: 23K req/s (fetch, DNS, store).  
**Insight**: I/O-intensive, politeness constrains throughput, storage scales with pages.

---

## System Interface
- **Input**: List of seed URLs.  
- **Output**: Text data stored in blob storage.

---

## Data Flow
1. Fetch seed URL from frontier queue, resolve IP via DNS.  
2. Fetch HTML from external server.  
3. Store raw HTML in blob storage.  
4. Extract text and URLs from HTML.  
5. Store text in blob storage, enqueue new URLs to frontier.  
6. Repeat until queue is empty or goal is met.

---

## Database
**Choices**:  
- **Metadata**: DynamoDB for URLs and state (scalable).  
- **Queue**: SQS for frontier (fault-tolerant).  
- **Storage**: S3 for HTML and text (durable).  
- **Cache**: Redis for DNS and robots.txt (low latency).  
**Key Entities**:  
- **URL**: urlId, url, domain, status, htmlPath, textPath, depth, lastCrawled.  
- **Domain**: domain, robotsTxt, lastCrawled, crawlDelay.  
**Indexing**:  
- DynamoDB:  
  - URL: Partition `urlId`, GSI `domain` for rate limiting.  
  - Domain: Partition `domain`.  
- SQS: Messages with `urlId`.  
- S3: `html/{urlId}`, `text/{urlId}`.  
- Redis: `dns:{domain}`, `robots:{domain}`.

---

## High-Level Design
**Architecture**: Pipelined microservices with SQS for coordination.  
**Components**:  
1. **Frontier Queue (SQS)**: Stores URLs to crawl.  
2. **URL Fetcher**: Fetches HTML, stores in S3.  
3. **Text/URL Extractor**: Parses HTML, extracts text and URLs, stores text in S3.  
4. **Metadata DB (DynamoDB)**: Tracks URLs, domains, state.  
5. **Cache (Redis)**: Caches DNS, robots.txt.  
6. **Storage (S3)**: Stores HTML, text.  
7. **DNS**: Resolves IPs (external/cached).  
8. **Webpage**: External servers (out of scope).

**Data Flow**:  
- **Crawl**: SQS → Fetcher → S3 (HTML) → DynamoDB (update) → SQS (parsed).  
- **Extract**: SQS → Extractor → S3 (text) → DynamoDB (update) → SQS (new URLs).  
- **Politeness**: Check DynamoDB/Redis for robots.txt, rate limits.  

**Diagram** (verbalize):  
- SQS → Fetcher → S3/DynamoDB → SQS → Extractor → S3/DynamoDB → SQS.  
- Redis for DNS/robots.txt, DynamoDB for state.

---

## Deep Dives
Key challenges addressing non-functional requirements.

### 1. Fault Tolerance
**Challenge**: Handle failures (fetch, parse, crashes) without losing progress.  
**Solution**:  
- **Pipelined Stages**:  
  - Fetcher: Fetches HTML, stores in S3, updates DynamoDB (`status: fetched`).  
  - Extractor: Parses HTML, stores text in S3, enqueues URLs, updates DynamoDB (`status: processed`).  
  - Each stage retries independently (3x with backoff).  
- **SQS Resilience**:  
  - Visibility timeout (30s): Unfinished jobs reappear if Fetcher/Extractor crashes.  
  - Dead Letter Queue (DLQ): Failed jobs (post-retries) logged for analysis.  
- **Metadata DB**:  
  - Tracks `urlId`, `status`, `htmlPath`, `textPath`.  
  - Resumes by querying `status: pending` or `fetched`.  
- **Back-of-Envelope**:  
  - 23K req/s × 1% fail × 3 retries = 690 retries/s.  
  - 30s timeout × 23K jobs/s = 690K active jobs (SQS scales).  
- **Why?**: Pipelines isolate failures; SQS ensures retry; DynamoDB persists state.  
- **Alternative**: Monolithic crawler (full restart); no queue (lost jobs).  
**Interview Tip**: Sketch pipeline, explain SQS timeout. If probed, discuss DLQ or retry logic.

### 2. Politeness (robots.txt, Rate Limiting)
**Challenge**: Respect robots.txt, avoid overloading servers.  
**Solution**:  
- **robots.txt**:  
  - Fetch `/robots.txt` per domain, cache in Redis (`robots:{domain}`, TTL 24h).  
  - Store rules in DynamoDB (`Domain` table: `disallow`, `crawlDelay`).  
  - Fetcher checks rules before crawling: skip if disallowed, delay per `crawlDelay`.  
- **Rate Limiting**:  
  - Redis tracks requests/domain (`domain:{domain}:count`, TTL 1s).  
  - Sliding window: Increment count, check <1 req/s/domain.  
  - Jitter: Random delay (10–100ms) to avoid synchronized retries.  
  - If rate-limited, requeue URL to SQS with delay (1s).  
- **Domain Scheduling**:  
  - DynamoDB `Domain` tracks `lastCrawled`.  
  - Fetcher skips URLs if `lastCrawled` < `crawlDelay`, requeues.  
- **Back-of-Envelope**:  
  - 10M domains × 1 req/s = 10M req/s max, throttled to 23K req/s.  
  - Redis: 10M domains × 100B = 1GB (cached).  
- **Why?**: Redis for fast checks; DynamoDB for persistence; jitter prevents spikes.  
- **Alternative**: No rate limiting (server bans); local checks (inconsistent).  
**Interview Tip**: Explain robots.txt flow, rate limit with jitter. If probed, discuss cache refresh or domain sharding.

### 3. Scalability and Efficiency (<5 Days)
**Challenge**: Crawl 10B pages in <5 days, handle I/O and storage.  
**Solution**:  
- **Crawlers (Fetcher)**:  
  - Parallelize: ~4 EC2 instances (network-optimized, 400Gbps).  
  - Assume 30% bandwidth: 400Gbps ÷ 8 ÷ 2MB/page × 30% ≈ 7.5K pages/s/instance.  
  - 10B pages ÷ (7.5K × 4) ÷ 86400 ≈ 3.85 days.  
- **Extractors**:  
  - Auto-scale ECS tasks based on SQS depth (~100 tasks, 230 pages/s/task).  
  - CPU-bound, parallel parsing.  
- **DNS Optimization**:  
  - Cache in Redis (`dns:{domain}`, TTL 1h): ~10M domains × 100B = 1GB.  
  - Multiple providers, round-robin to avoid rate limits.  
- **Deduplication**:  
  - DynamoDB `URL` table checks `urlId` (hash of URL) before enqueuing.  
  - Optional: Content hash index for duplicate pages (trade-off: storage vs. efficiency).  
- **Crawler Traps**:  
  - DynamoDB `URL` tracks `depth`, max 100 to avoid infinite loops.  
- **Storage**:  
  - S3: 20PB HTML, 1PB text (multipart uploads).  
  - DynamoDB: 10B URLs × 1KB = 10TB, partitioned by `urlId`.  
- **Back-of-Envelope**:  
  - 10B pages ÷ 432K s = 23K pages/s.  
  - 4 instances × 7.5K = 30K pages/s (meets goal).  
  - S3: 20PB ÷ 5 days = 46TB/s (S3 scales).  
- **Why?**: EC2 for I/O; ECS for parsing; Redis/DynamoDB optimize lookups; S3 scales storage.  
- **Alternative**: Lambda (costly); no deduplication (redundant crawls).  
**Interview Tip**: Estimate bandwidth, justify 4 instances. If probed, discuss DNS caching or traps.

---

## Interview Checklist
- **Structure** (35 min):  
  1. Requirements: Functional/non-functional (2 min).  
  2. Estimations: Pages, time (2 min).  
  3. Interface: Input/output (1 min).  
  4. Data Flow: Crawl/extract (2 min).  
  5. High-Level: Pipeline, components (5 min).  
  6. Deep Dives: Fault tolerance, politeness, scalability (20–23 min).  
- **Whiteboard** (verbalize):  
  - Draw: SQS → Fetcher → S3/DynamoDB → SQS → Extractor → S3/DynamoDB.  
  - Show: robots.txt check, rate limiting, deduplication.  
- **Avoid Pitfalls**:  
  - Don’t store HTML in queue (slow, costly).  
  - Don’t skip politeness (bans, ethics).  
  - Don’t ignore traps (infinite loops).  
  - Don’t crawl sequentially (too slow).  
- **Trade-Offs**:  
  - SQS vs. Kafka: Simplicity vs. control.  
  - DynamoDB vs. MySQL: Scalability vs. relations.  
  - Content deduplication vs. URL-only: Efficiency vs. simplicity.  
- **Mid-Level Expectations**:  
  - Define flow, basic crawler with queue, S3 storage.  
  - Propose naive fetch, pivot to pipeline with hints.  
  - Miss politeness/traps, address scaling naively.  
  - Reason through probes; expect guidance.

---

This summary is streamlined for your preparation. Send the next system design, and I’ll keep the format consistent!
