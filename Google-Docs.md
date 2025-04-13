<img width="675" alt="Screenshot 2025-04-13 at 4 46 20 PM" src="https://github.com/user-attachments/assets/fdf54e77-bacb-4be0-b263-98319518bb77" />



# System Design: Collaborative Document Editor Like Google Docs

This is a concise, interview-ready summary for designing a collaborative document editor like Google Docs, optimized for a 35-minute interview. It covers all critical points to excel, aligning with your 20+ system design cheatsheet for efficient study. The design is "hard" due to real-time collaboration, eventual consistency, and scaling to millions of users with low latency (<100ms).

---

## Overview
**Name**: Collaborative Document Editor (Google Docs-like)  
**Purpose**: Enable users to create and edit documents collaboratively in real-time.  
**Scope**: Core features (create documents, concurrent editing, real-time updates, cursor presence). Excludes versioning, permissions, complex formatting.

---

## Functional Requirements
- **Create Document**: Users create new documents.  
- **Concurrent Editing**: Multiple users edit the same document simultaneously.  
- **Real-Time Updates**: Users see others’ changes instantly.  
- **Cursor Presence**: Users see others’ cursor positions and presence.

---

## Non-Functional Requirements
- **Consistency**: Eventual consistency for document state.  
- **Latency**: Updates delivered in <100ms.  
- **Scalability**: Support millions of concurrent users, billions of documents, ≤100 editors/document.  
- **Durability**: Documents persist across server restarts.  
- **Availability**: System remains operational despite failures.

---

## Scale Estimations
- **Users**: 10M concurrent users, ~100 editors/document.  
- **Documents**: 1B documents, ~50KB/doc avg.  
- **Edits**: 10M users × 10 edits/min × 100B/edit ≈ 16K edits/s.  
- **Storage**: 1B docs × 50KB ≈ 50TB (ops); 1B × 1KB metadata ≈ 1TB.  
**Insight**: High write (edits), stateful connections (WebSockets), storage growth critical.

---

## API Design
REST for document creation, WebSocket for real-time collaboration, JWT for auth.

1. **Create Document**:  
   - `POST /docs`  
   - Header: `Authorization: Bearer <JWT>`  
   - Body: `{ title: string }`  
   - Returns: `{ docId: string }`  
   - Purpose: Create a new document.  

2. **Collaborate on Document**:  
   - `WS /docs/{docId}`  
   - Header: `Authorization: Bearer <JWT>`  
   - Messages:  
     - **Send**: `{ type: "insert", position: int, text: string, version: int }` (insert text).  
     - **Send**: `{ type: "delete", position: int, length: int, version: int }` (delete text).  
     - **Send**: `{ type: "cursor", position: int, userId: string }` (update cursor).  
     - **Recv**: `{ type: "update", operation: { type: string, ... }, version: int }` (receive edit).  
     - **Recv**: `{ type: "cursor", userId: string, position: int }` (receive cursor).  
   - Purpose: Real-time edits and cursor updates.

---

## Database
**Choices**:  
- **Metadata**: Postgres for document metadata (query flexibility).  
- **Operations**: Cassandra for edit ops (high write throughput).  
- **Cache**: Redis for cursor state (ephemeral).  
- **Storage**: S3 for snapshots (durability).  
**Key Entities**:  
- **Document**: docId, title, createdAt.  
- **Edit**: docId, version, type (insert/delete), position, text/length, timestamp.  
- **Cursor**: userId, docId, position (in-memory).  
**Indexing**:  
- Postgres:  
  - PK: `docId`.  
- Cassandra:  
  - Partition: `docId`, sort: `version`.  
- Redis:  
  - `doc:{docId}:cursors` (hash, userId → position).  
- S3:  
  - `snapshots/{docId}/{version}`.

---

## High-Level Design
**Architecture**: Microservices with WebSocket-based collaboration, OT for consistency.  
**Components**:  
1. **Client**: Browser app for editing, WebSocket connection.  
2. **API Gateway**: Routes REST/WebSocket, handles auth.  
3. **Document Service**: Manages metadata, coordinates edits.  
4. **Edit Service**: Processes operations, applies OT, broadcasts updates.  
5. **Database**: Postgres for metadata, Cassandra for ops.  
6. **Cache**: Redis for cursor state.  
7. **Storage**: S3 for snapshots.  
8. **Coordinator**: Zookeeper for consistent hashing.

**Data Flow**:  
- **Create**: Client → API Gateway → Document Service → Postgres → Client.  
- **Edit**: Client → API Gateway → Edit Service → Cassandra → WebSocket broadcast → Clients.  
- **Cursor**: Client → API Gateway → Edit Service → Redis → WebSocket broadcast → Clients.  
- **Load**: Client → API Gateway → Edit Service → Cassandra/S3 → Client.

**Diagram** (verbalize):  
- Client → API Gateway → Document/Edit Service → Postgres/Cassandra/Redis/S3.  
- WebSocket: Client ↔ Edit Service for edits/cursors.

---

## Deep Dives
Key challenges addressing non-functional requirements.

### 1. Scaling WebSocket Connections
**Challenge**: Millions of concurrent WebSocket connections, ≤100 editors/doc.  
**Solution**:  
- **Consistent Hashing**:  
  - Shard Edit Service by `docId` using consistent hash ring (~100 servers).  
  - Zookeeper manages ring, maps `docId` to server.  
  - Clients connect to correct server via redirect (HTTP → WebSocket).  
- **Connection Management**:  
  - Edit Service maintains `docId → Set<WebSocket>` in-memory.  
  - Heartbeats detect disconnects, remove cursors.  
- **Scaling**:  
  - Add servers, rebalance ~1% connections (consistent hashing).  
  - Max 100 connections/doc ≈ 10K docs/server (1M connections/100).  
- **Back-of-Envelope**:  
  - 10M users ÷ 100 servers = 100K connections/server.  
  - 16K edits/s ÷ 100 servers = 160 edits/s/server (manageable).  
- **Why?**: Consistent hashing minimizes reshuffling; WebSockets enable real-time; Zookeeper ensures coordination.  
- **Alternative**: Redis pub/sub (higher latency); single server (SPOF).  
**Interview Tip**: Propose consistent hashing, sketch redirect flow. If probed, discuss rebalancing or heartbeat.

### 2. Ensuring Eventual Consistency with OT
**Challenge**: Guarantee same document state across clients with concurrent edits.  
**Solution**:  
- **Operational Transformation (OT)**:  
  - Edit Service transforms operations (e.g., insert/delete) based on document version.  
  - Client sends `{ type, position, text/length, version }`.  
  - Server applies OT, increments version, stores in Cassandra.  
  - Broadcasts transformed op to all clients (except sender).  
- **Client-Side OT**:  
  - Clients apply local edits immediately, queue for server.  
  - On server update, transform queued ops against received op.  
- **Example**:  
  - Doc: "Hi". User A inserts "!" at pos 2 (version 0). User B deletes "i" at pos 1 (version 0).  
  - Server gets A’s op, makes "Hi!", version 1.  
  - B’s op transforms to delete pos 1 on "Hi!", yielding "H!", version 2.  
  - Both clients converge to "H!".  
- **Back-of-Envelope**:  
  - 16K edits/s × 1KB/op = 16MB/s (Cassandra handles).  
  - OT compute: ~1μs/op × 16K/s = 16ms/s/server.  
- **Why?**: OT ensures convergence; Cassandra scales writes; client OT reduces latency.  
- **Alternative**: CRDT (higher storage); locking (blocks concurrency).  
**Interview Tip**: Explain OT briefly, use example. If probed, discuss version conflicts or CRDT trade-offs.

### 3. Controlling Storage Growth
**Challenge**: 50TB of operations across 1B documents, memory constraints.  
**Solution**:  
- **Compaction**:  
  - Periodically snapshot documents to S3 (e.g., every 1K ops or 1h).  
  - Single op: `insert(0, full_text)`.  
  - Delete old ops from Cassandra post-snapshot.  
- **Lazy Loading**:  
  - Load snapshot from S3 + recent ops from Cassandra on first client connect.  
  - Cache in Edit Service memory for active docs.  
- **Eviction**:  
  - Evict inactive docs from memory (LRU, TTL 1h).  
  - Max 10K active docs/server × 50KB = 500MB memory.  
- **Back-of-Envelope**:  
  - 1B docs × 50KB/snapshot = 50TB (S3 scales).  
  - 16K ops/s × 1KB × 1h = 57GB/h pre-compaction.  
  - Post-compaction: ~1B snapshots × 50KB = 50TB total.  
- **Why?**: Snapshots reduce ops storage; S3 is cost-effective; eviction manages memory.  
- **Alternative**: Keep all ops (unbounded growth); in-memory only (no durability).  
**Interview Tip**: Propose snapshots, estimate storage. If probed, discuss compaction triggers or versioning.

---

## Interview Checklist
- **Structure** (35 min):  
  1. Requirements: Functional/non-functional (2 min).  
  2. Estimations: QPS, storage (2 min).  
  3. API: REST/WebSocket (2 min).  
  4. Database: Postgres/Cassandra/Redis (2 min).  
  5. High-Level: Microservices, OT flow (5 min).  
  6. Deep Dives: WebSockets, OT, storage (18–22 min).  
- **Whiteboard** (verbalize):  
  - Draw: Client → API Gateway → Document/Edit Service → Postgres/Cassandra/Redis/S3.  
  - Show: Edit (WebSocket → OT → Cassandra → broadcast), cursor (Redis → broadcast).  
- **Avoid Pitfalls**:  
  - Don’t use snapshots for every edit (bandwidth waste).  
  - Don’t skip OT/CRDT (inconsistent state).  
  - Don’t store cursors in DB (ephemeral data).  
  - Don’t ignore memory (OOM risk).  
- **Trade-Offs**:  
  - OT vs. CRDT: Simplicity vs. decentralization.  
  - Cassandra vs. DynamoDB: Throughput vs. simplicity.  
  - WebSocket vs. SSE: Bidirectionality vs. lightweight.  
- **Mid-Level Expectations**:  
  - Define APIs, basic Document Service, Postgres for docs.  
  - Propose naive edits (e.g., snapshots), pivot to ops with hints.  
  - Miss OT details, address scaling naively.  
  - Reason through probes; expect guidance.

---

This summary is lean for your preparation. Send the next system design, and I’ll keep the format consistent!
