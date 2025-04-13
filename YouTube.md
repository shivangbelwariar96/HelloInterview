# System Design: Video Streaming Platform Like YouTube

This is a concise, interview-ready summary for designing a video streaming platform like YouTube, optimized for a 35-minute interview. It covers all critical points to excel, aligning with your 20+ system design cheatsheet for efficient study. The design is "hard" due to high availability, low-latency streaming, large video support (10s of GB), and scaling to ~1M uploads and 100M views/day.

---

## Overview
**Name**: Video Streaming Platform (YouTube-like)  
**Purpose**: Allow users to upload and stream videos.  
**Scope**: Core features (upload, stream videos). Excludes search, comments, recommendations, subscriptions.

---

## Functional Requirements
- **Upload Videos**: Users upload videos with metadata.  
- **Stream Videos**: Users watch videos with adaptive playback.

---

## Non-Functional Requirements
- **Availability**: Highly available, prioritizes availability over consistency.  
- **Scalability**: ~1M uploads/day, 100M views/day.  
- **Latency**: Low-latency streaming, even on low bandwidth.  
- **Large Videos**: Support 10s of GB uploads.  
- **Resumable Uploads**: Handle interrupted uploads.  
- **Durability**: Videos persist across failures.

---

## Scale Estimations
- **Uploads**: 1M videos/day ≈ 11.6 uploads/s, ~10GB/video avg ≈ 10TB/day.  
- **Views**: 100M views/day ≈ 1.2K views/s, ~1MB/s/view (adaptive) ≈ 1.2TB/s peak.  
- **Metadata**: 1M videos × 1KB/record ≈ 1GB/day, ~365GB/year.  
- **Storage**: 10TB/day × 365 days ≈ 3.65PB/year (pre-transcoding).  
**Insight**: High storage and bandwidth, metadata lightweight, views dominate traffic.

---

## API Design
RESTful APIs with client handling chunked uploads and streaming.

1. **Get Presigned URL**:  
   - `POST /presigned_url`  
   - Header: `Authorization: Bearer <JWT>`  
   - Body: `{ videoId: string, metadata: { title: string, ... }, chunks: [{ fingerprint: string, size: long }] }`  
   - Returns: `{ urls: [{ chunkId: string, url: string }] }`  
   - Purpose: Initiate upload, get S3 URLs for chunks.  

2. **Stream Video**:  
   - `GET /videos/{videoId}`  
   - Header: `Authorization: Bearer <JWT>`  
   - Returns: `{ metadata: { title: string, ... }, primaryManifestUrl: string }`  
   - Purpose: Fetch metadata and manifest for streaming.

---

## Database
**Choices**:  
- **Metadata**: Cassandra for video metadata (scalable, partitioned).  
- **Cache**: Redis for hot metadata (low latency).  
- **Storage**: S3 for videos and manifests (durability, scale).  
- **Queue**: Kafka for processing jobs (reliability).  
**Key Entities**:  
- **VideoMetadata**: videoId, title, uploaderId, chunks [{ fingerprint, status }], manifestUrls, status.  
- **VideoSegment**: Stored in S3 (e.g., `videos/{videoId}/{format}/{segmentId}`).  
- **Manifest**: Stored in S3 (e.g., `manifests/{videoId}/{format}.m3u8`).  
**Indexing**:  
- Cassandra:  
  - Partition: `videoId`.  
- Redis:  
  - `video:{videoId}:metadata` (hash).  
- S3:  
  - `videos/{videoId}/raw`, `videos/{videoId}/{format}`, `manifests/{videoId}`.  
- Kafka: Topic `video_jobs`, partitioned by `videoId`.

---

## High-Level Design
**Architecture**: Microservices with S3 for storage, CDN for streaming, DAG for processing.  
**Components**:  
1. **Client**: App/browser for upload/streaming.  
2. **API Gateway**: Routes requests, handles auth.  
3. **Video Service**: Manages metadata, presigned URLs.  
4. **Processing Service**: Transcodes videos, generates segments/manifests.  
5. **Database**: Cassandra for metadata.  
6. **Cache**: Redis for hot metadata.  
7. **Storage**: S3 for raw videos, segments, manifests.  
8. **Queue**: Kafka for job orchestration.  
9. **CDN**: Edge caching for segments/manifests.

**Data Flow**:  
- **Upload**: Client → API Gateway → Video Service → Cassandra (metadata) → S3 (chunks via presigned URLs) → Kafka (job).  
- **Processing**: Kafka → Processing Service → S3 (segments, manifests) → Cassandra (update metadata).  
- **Stream**: Client → API Gateway → Video Service → Cassandra/Redis → CDN/S3 (manifest, segments) → Client.  

**Diagram** (verbalize):  
- Client → API Gateway → Video Service → Cassandra/Redis → S3/CDN.  
- Upload: Client → S3 → Kafka → Processing Service → S3.  
- Stream: Client ↔ CDN/S3.

---

## Deep Dives
Key challenges addressing non-functional requirements.

### 1. Adaptive Bitrate Streaming
**Challenge**: Low-latency streaming across varying bandwidths.  
**Solution**:  
- **Processing Pipeline (DAG)**:  
  - Split raw video into segments (~5s each) using FFmpeg.  
  - Transcode segments to multiple formats (e.g., H.264/720p, H.265/1080p).  
  - Generate media manifests (.m3u8) per format, primary manifest linking all.  
  - Store segments/manifests in S3.  
- **Orchestration**:  
  - Temporal manages DAG: split → transcode (parallel) → manifest → update metadata.  
  - Workers scale elastically (~100 nodes for 11.6 uploads/s).  
- **Streaming**:  
  - Client fetches primary manifest, selects format based on bandwidth.  
  - Streams segments via HLS/DASH from CDN.  
- **Back-of-Envelope**:  
  - 1M uploads/day × 10GB = 10TB/day raw.  
  - 5 formats × 1GB/format = 5TB/day post-transcode.  
  - 1.2K views/s × 1MB/s = 1.2GB/s (CDN handles).  
- **Why?**: DAG parallelizes transcoding; CDN reduces latency; HLS/DASH adapts to bandwidth.  
- **Alternative**: Single format (no adaptation); no CDN (high latency).  
**Interview Tip**: Sketch DAG, explain HLS. If probed, discuss transcode cost or segment size.

### 2. Resumable Uploads
**Challenge**: Support interrupted uploads for 10s of GB videos.  
**Solution**:  
- **Chunked Upload**:  
  - Client splits video into ~10MB chunks, generates fingerprints (hashes).  
  - POST `/presigned_url` with metadata, chunk fingerprints.  
  - Video Service stores metadata in Cassandra (`chunks: [{ fingerprint, status: NotUploaded }]`).  
  - Returns presigned S3 URLs per chunk.  
- **Upload Process**:  
  - Client uploads chunks to S3 concurrently.  
  - S3 triggers Lambda via event notifications, updates Cassandra (`status: Uploaded`).  
- **Resume**:  
  - Client queries `/videos/{videoId}`, checks uploaded chunks.  
  - Requests URLs for remaining chunks, continues upload.  
- **Back-of-Envelope**:  
  - 10GB video ÷ 10MB/chunk = 1K chunks.  
  - 11.6 uploads/s × 1K chunks = 11.6K chunk req/s (S3 scales).  
- **Why?**: Chunking reduces failure impact; Cassandra tracks progress; S3/Lambda automates updates.  
- **Alternative**: Single upload (no resume); client-side tracking (unreliable).  
**Interview Tip**: Explain chunk flow, highlight Lambda. If probed, discuss fingerprint collisions or timeout.

### 3. Scalability
**Challenge**: Handle 1M uploads, 100M views/day, avoid hotspots.  
**Solution**:  
- **Video Service**:  
  - Stateless, horizontally scaled (~50 servers for 11.6 req/s).  
  - Load balancer distributes requests.  
- **Cassandra**:  
  - Partition by `videoId`, replicate 3x for availability.  
  - Cache hot metadata in Redis (LRU, `videoId` key).  
- **Processing Service**:  
  - Kafka queues jobs, ~100 workers for parallel transcoding.  
  - Auto-scale based on queue depth.  
- **S3**:  
  - Scales to 10TB/day uploads, 5TB/day segments.  
- **CDN**:  
  - Caches segments/manifests at edge (~1.2GB/s views).  
  - Reduces S3 load, lowers latency for global users.  
- **Hot Videos**:  
  - Redis cache for metadata (~10K hot videos × 1KB = 10MB).  
  - CDN replicates popular segments across edges.  
- **Back-of-Envelope**:  
  - 100M views/day ÷ 50 servers = 23 req/s/server (metadata).  
  - 1.2K views/s × 1MB/s = 1.2GB/s (CDN scales).  
- **Why?**: Cassandra/Redis handle metadata; S3/CDN scale storage/bandwidth; Kafka ensures processing reliability.  
- **Alternative**: Single DB (hotspots); no CDN (high latency).  
**Interview Tip**: Highlight CDN/Redis, estimate traffic. If probed, discuss cache eviction or CDN costs.

---

## Interview Checklist
- **Structure** (35 min):  
  1. Requirements: Functional/non-functional (2 min).  
  2. Estimations: Uploads, views, storage (2 min).  
  3. API: REST endpoints (2 min).  
  4. Database: Cassandra/S3/Redis (2 min).  
  5. High-Level: Services, flow (5 min).  
  6. Deep Dives: Streaming, uploads, scalability (20–22 min).  
- **Whiteboard** (verbalize):  
  - Draw: Client → API Gateway → Video Service → Cassandra/Redis → S3/CDN.  
  - Show: Upload (S3 → Kafka → Processing), stream (CDN → segments).  
- **Avoid Pitfalls**:  
  - Don’t store videos in DB (slow, costly).  
  - Don’t skip CDN (high latency).  
  - Don’t ignore chunking (upload failures).  
  - Don’t process in real-time (slow uploads).  
- **Trade-Offs**:  
  - S3 vs. custom storage: Scalability vs. control.  
  - Cassandra vs. DynamoDB: Flexibility vs. simplicity.  
  - CDN vs. direct S3: Latency vs. cost.  
- **Mid-Level Expectations**:  
  - Define APIs, basic upload/stream flow, S3 for storage.  
  - Propose naive processing (e.g., single format), pivot to segments with hints.  
  - Miss DAG/CDN, address scaling naively.  
  - Reason through probes; expect guidance.

---

This summary is streamlined for your preparation. Send the next system design, and I’ll keep the format consistent!
