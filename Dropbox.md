<img width="1149" alt="Screenshot 2025-04-13 at 4 00 38 PM" src="https://github.com/user-attachments/assets/bc2ec97c-6f1b-4231-a6a5-c501cc74ce65" />


# System Design: File Storage Service Like Dropbox

This is a concise, interview-ready summary for designing a file storage service like Dropbox, tailored for quick review and covering all critical points to succeed in a 35-minute interview. It omits unnecessary details (e.g., full DB schemas, verbose trade-offs) but includes everything essential for a junior-to-mid-level candidate. The format is clear, focused, and optimized for studying alongside your 20+ other system designs.

---

## Overview
**Name**: File Storage Service (Dropbox-like)  
**Purpose**: Cloud-based service for uploading, downloading, sharing, and syncing files across devices.  
**Scope**: Core features (upload, download, share, sync). Excludes editing, previewing, versioning, virus scanning.

---

## Functional Requirements
- **Upload File**: Users can upload files from any device.  
- **Download File**: Users can download files to any device.  
- **Share File**: Users can share files with others and view shared files.  
- **Sync Files**: Files auto-sync across devices (local ↔ remote).  

---

## Non-Functional Requirements
- **Availability**: High availability (99.99%), prioritizing over consistency (eventual consistency OK for sync).  
- **File Size**: Support files up to 50GB.  
- **Security**: Secure storage and access; recover lost/corrupted files.  
- **Performance**: Minimize upload/download/sync latency.  
- **Scalability**: Handle millions of users, billions of files (~TB–PB storage).

---

## API Design
RESTful APIs for core functionality, using secure headers for user authentication (JWT/session).

1. **Upload File**:  
   - `POST /files`  
   - Body: `{ file: binary, metadata: { name: string, size: int, mimeType: string } }`  
   - Returns: `{ fileId: string, status: string }`  
   - Purpose: Upload file and metadata.  

2. **Download File**:  
   - `GET /files/{fileId}`  
   - Returns: File binary, metadata.  
   - Purpose: Retrieve file and details.  

3. **Share File**:  
   - `POST /files/{fileId}/share`  
   - Body: `{ users: string[] }`  
   - Returns: `{ status: string }`  
   - Purpose: Grant access to specified users.  

4. **Sync Changes**:  
   - `GET /files/changes?since={timestamp}`  
   - Returns: `{ files: [{ fileId: string, metadata: object, updatedAt: string }] }`  
   - Purpose: Fetch updates for client sync.

---

## Database
**Choices**:  
- **Metadata**: NoSQL (e.g., DynamoDB) for flexible schema, fast user-based queries.  
- **Files**: Blob storage (e.g., AWS S3) for scalable file storage.  
**Key Entities**:  
- **FileMetadata**: File ID, name, size, mimeType, uploadedBy, sharedWith (user IDs), updatedAt.  
- **SharedFiles**: File ID, user IDs (for access control).  
**Indexing**:  
- Metadata: Partition key on `userId`, index on `fileId`.  
- SharedFiles: Index on `fileId`, `userId` for sharing queries.

---

## High-Level Design
**Architecture**: Monolith for simplicity, with stateless services for scale. Leverages cloud storage for files.  
**Components**:  
1. **Client**: Web/mobile/desktop app with sync agent for local monitoring.  
2. **API Gateway**: Routes requests, handles SSL, rate limiting.  
3. **File Service**: Manages metadata, generates S3 presigned URLs.  
4. **Metadata DB**: DynamoDB for file metadata.  
5. **Blob Storage**: S3 for raw files.  
6. **CDN**: CloudFront for fast downloads.  
7. **Sync Service**: Tracks changes, notifies clients via WebSocket/polling.  

**Data Flow**:  
- **Upload**: Client → API Gateway → File Service → S3 (file), DynamoDB (metadata).  
- **Download**: Client → API Gateway → File Service → CDN/S3 → Client.  
- **Share**: Client → API Gateway → File Service → DynamoDB (update SharedFiles).  
- **Sync**: Client ↔ Sync Service (WebSocket/polling) ↔ DynamoDB → S3 for file updates.  

**Diagram** (verbalize):  
- Client talks to API Gateway → File Service.  
- Files go to S3 via presigned URLs; metadata to DynamoDB.  
- CDN serves downloads; Sync Service pushes/polls changes.  
- WebSocket for fresh files, polling for stale ones.

---

## Deep Dives
Key challenges addressing non-functional requirements, with clear solutions for interviewer probes.

### 1. Supporting Large Files (50GB)
**Challenge**: Upload/download 50GB files without timeouts, network failures, or poor UX.  
**Solution**:  
- **Chunking**:  
  - Client splits files into 5–10MB chunks (e.g., 50GB → ~10,000 chunks).  
  - Upload chunks in parallel to maximize bandwidth.  
  - Store chunk fingerprints (SHA-256 hashes) for deduplication/resumption.  
- **Resumable Uploads**:  
  - Track chunks in DynamoDB (`fileId`, chunk ID, status: uploaded/not-uploaded).  
  - Client queries metadata to resume failed uploads, skipping uploaded chunks.  
  - S3 Multipart Upload API handles chunk assembly server-side.  
- **Progress UI**: Client tracks chunk uploads, shows progress bar.  
- **Why?**: Chunking avoids timeouts (e.g., 50GB at 100Mbps takes ~1.1 hours), supports resumption, improves UX.  
- **Alternative**: Single POST (times out, no resumption); server-side chunking (full upload first, defeats purpose).  
**Interview Tip**: Propose chunking and resumption, mention S3 Multipart. If probed, explain fingerprints or parallel uploads. Avoid assuming interviewer wants S3 internals.

### 2. Fast Upload/Download/Sync
**Challenge**: Minimize latency for uploads, downloads, and sync across millions of users.  
**Solution**:  
- **Uploads**:  
  - Parallel chunk uploads to S3 (maximize bandwidth).  
  - Client-side compression for text files (e.g., Gzip, reduces 5GB to ~1GB).  
  - Avoid compression for media (low ratio, high CPU cost).  
- **Downloads**:  
  - CDN (CloudFront) caches files near users, reducing latency (<100ms).  
  - Presigned S3 URLs for secure, direct access.  
- **Sync**:  
  - Hybrid sync: WebSocket for fresh files (real-time), polling (every 5–10s) for stale files.  
  - Sync only changed chunks (use fingerprints to detect).  
  - “Last write wins” for conflicts (versioning out of scope).  
- **Why?**: CDN and chunking optimize bandwidth; hybrid sync balances speed and resources.  
- **Alternative**: Polling-only (slow for active files); WebSocket-only (resource-heavy).  
**Interview Tip**: Lead with CDN and chunking, mention hybrid sync. If probed, discuss compression trade-offs or conflict resolution.

### 3. File Security
**Challenge**: Ensure files are accessible only to authorized users, recoverable if lost.  
**Solution**:  
- **Encryption**:  
  - **In Transit**: HTTPS for all transfers (client ↔ S3/CDN).  
  - **At Rest**: S3 server-side encryption (AES-256) with managed keys.  
- **Access Control**:  
  - Store `sharedWith` in DynamoDB metadata or separate SharedFiles table.  
  - Generate S3 presigned URLs (5-min TTL) for downloads, validated by File Service.  
  - CDN (CloudFront) honors presigned URLs, blocks unauthorized access.  
- **Recovery**:  
  - S3 durability (99.999999999%) ensures files aren’t lost.  
  - Metadata backups in DynamoDB streams to another region.  
- **Why?**: Encryption and presigned URLs secure access; S3 durability simplifies recovery.  
- **Alternative**: Custom ACLs (complex); no encryption (insecure).  
**Interview Tip**: Propose HTTPS, S3 encryption, presigned URLs. If probed, mention CDN validation or backups. Avoid deep crypto unless asked.

### 4. Scalability (Millions of Users, PB Storage)
**Challenge**: Handle millions of uploads/downloads, billions of files, PB-scale storage.  
**Solution**:  
- **Storage**:  
  - S3 scales infinitely for files; no sharding needed.  
  - DynamoDB auto-scales for metadata; partition by `userId` for even distribution.  
- **Services**:  
  - File Service stateless, auto-scales behind API Gateway (e.g., AWS Lambda/ECS).  
  - Sync Service scales WebSocket connections via load balancers.  
- **Performance**:  
  - CDN absorbs download traffic (99%+ cache hits).  
  - Redis cache for metadata queries (e.g., recent files), reducing DynamoDB load.  
  - Batch sync updates to minimize polling overhead.  
- **Why?**: S3/DynamoDB handle storage; stateless services scale horizontally; CDN/cache reduce latency.  
- **Alternative**: Custom storage (complex, costly); single DB (bottleneck).  
**Interview Tip**: Highlight S3 scalability, CDN for downloads. If probed, mention DynamoDB partitioning or caching. Avoid overcomplicating with microservices.

---

## Interview Checklist
- **Structure** (35 min):  
  1. Requirements: Functional/non-functional (2 min).  
  2. API: List endpoints (2 min).  
  3. Database: NoSQL + S3, key entities (2 min).  
  4. High-Level: Monolith, components, flow (5 min).  
  5. Deep Dives: Large files, speed, security, scalability (15–20 min).  
- **Whiteboard** (verbalize):  
  - Draw: Client → API Gateway → File Service → S3, DynamoDB, CDN; Sync Service ↔ WebSocket.  
  - Show: Upload (chunks → S3), download (CDN → S3), sync (WebSocket/polling).  
- **Avoid Pitfalls**:  
  - Don’t chunk on server (defeats purpose).  
  - Don’t skip CDN (slow downloads).  
  - Don’t pass user ID in body (use headers).  
  - Don’t propose single DB for files (use S3).  
- **Trade-Offs**:  
  - S3 vs. custom storage: S3 for scalability, simplicity.  
  - WebSocket vs. polling: Hybrid for balance.  
  - Compression vs. none: Compress text, skip media.  
- **Mid-Level Expectations**:  
  - Define APIs, data model (metadata + blob).  
  - Propose S3 for files, DynamoDB for metadata, chunking for large files.  
  - Explain upload/download/sync flow.  
  - Reason through probes (e.g., chunking, security); expect interviewer guidance.

---

This summary is lean and comprehensive, covering all must-have points for a Dropbox-like design, perfect for quick study. Send the next system design, and I’ll format it consistently for your cheatsheet!
