<img width="1159" alt="Screenshot 2025-04-13 at 4 38 04 PM" src="https://github.com/user-attachments/assets/16a74b02-68c2-495c-a7f8-ac13455d8e2a" />


# System Design: Messaging App Like WhatsApp

This is a concise, interview-ready summary for designing a messaging app like WhatsApp, tailored for a 35-minute interview. It covers all critical points to excel, omitting extraneous details (e.g., exhaustive schemas) while aligning with your 20+ system design cheatsheet for efficient study.

---

## Overview
**Name**: Messaging Service (WhatsApp-like)  
**Purpose**: Mobile/desktop app for encrypted group chats with low-latency message delivery and offline support.  
**Scope**: Core features (create group chats, send/receive messages, offline delivery, media support). Excludes calls, business interactions, profiles.

---

## Functional Requirements
- **Create Group Chats**: Users can start chats with 2–100 participants.  
- **Send/Receive Messages**: Users can send/receive text messages in chats.  
- **Offline Delivery**: Users receive messages sent while offline (stored up to 30 days).  
- **Media Support**: Users can send/receive media (images, videos) in messages.

---

## Non-Functional Requirements
- **Latency**: Message delivery <500ms for online users.  
- **Durability**: Guarantee message delivery (no loss).  
- **Scalability**: Support billions of users (~200M concurrent, 100B messages/day).  
- **Storage**: Minimize server-side message retention (delete after delivery or 30 days).  
- **Resilience**: Handle component failures gracefully.

---

## API Design
**Protocol**: WebSocket for real-time, bidirectional communication (TLS-secured).  
**Commands** (JSON over WebSocket, userId via JWT):  

1. **Create Chat**:  
   - `-> createChat { participants: string[], name: string }`  
   - `<- { chatId: string }`  
   - Purpose: Initialize group chat.  

2. **Send Message**:  
   - `-> sendMessage { chatId: string, message: string, attachments: string[] }`  
   - `<- "SUCCESS" | "FAILURE"`  
   - `<- newMessage { chatId: string, userId: string, message: string, attachments: string[], timestamp: int }`  
   - Purpose: Send message, notify recipients.  

3. **Create Attachment**:  
   - `-> createAttachment { body: binary, hash: string }`  
   - `<- { attachmentId: string }`  
   - Purpose: Upload media, get ID for message inclusion.  

4. **Modify Participants**:  
   - `-> modifyChatParticipants { chatId: string, userId: string, operation: "ADD" | "REMOVE" }`  
   - `<- "SUCCESS" | "FAILURE"`  
   - `<- chatUpdate { chatId: string, participants: string[] }`  
   - Purpose: Update chat membership.  

5. **Acknowledge Receipt**:  
   - `-> ack { messageId: string }`  
   - Purpose: Confirm message delivery to delete from inbox.

---

## Database
**Choices**:  
- **Metadata**: DynamoDB for chats/participants (scalable key-value).  
- **Messages**: DynamoDB for messages/inbox (high write throughput).  
- **Media**: S3 for attachments (cost-efficient storage).  
- **Cache**: Redis for connection routing.  
**Key Entities**:  
- **Chat**: chatId, name.  
- **ChatParticipant**: chatId, userId.  
- **Message**: messageId, chatId, userId, content, attachments, timestamp.  
- **Inbox**: clientId, messageId (per-device).  
- **Client**: userId, clientId, lastConnected.  
- **Attachment**: attachmentId, hash, S3 path.  
**Indexing**:  
- DynamoDB:  
  - Chat: PK `chatId`.  
  - ChatParticipant: PK `chatId`, SK `userId`; GSI PK `userId`, SK `chatId`.  
  - Message: PK `chatId`, SK `messageId`.  
  - Inbox: PK `clientId`, SK `messageId`.  
  - Client: PK `userId`, SK `clientId`.  
- S3: `attachmentId` as key.  
- Redis: Hash `userId:clientId → chatServer`.

---

## High-Level Design
**Architecture**: Microservices with WebSocket for real-time messaging.  
**Components**:  
1. **Client**: Mobile/desktop app with WebSocket connection.  
2. **Load Balancer**: L4 (TCP) for WebSocket routing.  
3. **Chat Service**: Manages WebSocket connections, routes messages.  
4. **Message Service**: Persists messages, handles inbox.  
5. **Attachment Service**: Uploads/downloads media to S3.  
6. **Database**: DynamoDB for metadata/messages.  
7. **Storage**: S3 for media.  
8. **Cache**: Redis for user-to-server routing.  
9. **Cron Job**: Deletes messages/inbox entries >30 days.

**Data Flow**:  
- **Create Chat**: Client → Chat Service → DynamoDB (Chat, ChatParticipant) → Client.  
- **Send Message**: Client → Chat Service → Message Service → DynamoDB (Message, Inbox) → Redis → Chat Service → Clients; offline clients sync on reconnect.  
- **Attachment**: Client → Attachment Service → S3 → Client (attachmentId).  
- **Offline Sync**: Client connects → Chat Service → Message Service → DynamoDB (Inbox) → Client.

**Diagram** (verbalize):  
- Client → L4 LB → Chat Service → Redis/DynamoDB/S3.  
- Chat Service ↔ Redis (routing); Message Service → DynamoDB; Attachment Service → S3.

---

## Deep Dives
Key challenges addressing non-functional requirements.

### 1. Scalability for Billions of Users
**Challenge**: Handle ~200M concurrent users, ~1M QPS (100B messages/day ÷ 86,400s).  
**Solution**:  
- **Chat Service Scaling**:  
  - Horizontal scaling (100s of servers, ~1M connections each).  
  - Redis pub/sub for routing: Chat servers subscribe to `userId` channels; publish messages to recipients’ channels.  
  - Example: User A sends message → Chat Server A publishes to Redis `userId:B` → Chat Server B (subscribed) delivers to B’s WebSocket.  
- **DynamoDB**:  
  - Partition by `chatId` (messages), `clientId` (inbox).  
  - Auto-scale write capacity (~1M writes/s).  
- **Back-of-Envelope**:  
  - 200M connections ÷ 1M/server = 200 servers.  
  - 100B messages × 100 bytes = 10TB/day; DynamoDB/S3 handles with partitioning.  
- **Why?**: Redis pub/sub avoids point-to-point server connections; DynamoDB scales writes.  
- **Alternative**: Consistent hashing (complex orchestration); Kafka (topic limits).  
**Interview Tip**: Propose Redis pub/sub, estimate QPS. If probed, discuss sharding or Zookeeper.

### 2. Multiple Clients per User
**Challenge**: Sync messages across devices (phone, laptop); track delivery per device.  
**Solution**:  
- **Client Table**: Store `userId`, `clientId` in DynamoDB.  
- **Per-Client Inbox**: Inbox keyed by `clientId`, not `userId`.  
- **Delivery**:  
  - Message Service queries `userId` → `clientIds`, writes to each client’s inbox.  
  - Chat Service sends to active clients via WebSocket; offline clients sync on connect.  
- **Limits**: Cap at 3 clients/user to bound storage/QPS.  
- **Cleanup**: Delete inactive clients (no connection >30 days).  
- **Why?**: Per-client inbox ensures sync; limits prevent abuse.  
- **Alternative**: Single inbox (sync issues); no limits (storage explosion).  
**Interview Tip**: Highlight per-client inbox, mention limits. If probed, discuss cleanup or sync edge cases.

### 3. Low Latency (<500ms)
**Challenge**: Deliver messages to online users quickly.  
**Solution**:  
- **WebSocket**: Persistent connections avoid handshake overhead.  
- **Redis Pub/Sub**: In-memory routing (<10ms).  
- **DynamoDB**: Fast writes with eventual consistency for offline users.  
- **Geo-Distribution**: Deploy Chat Services in multiple regions, route via closest LB.  
- **Why?**: WebSocket/Redis minimize hops; DynamoDB scales writes.  
- **Alternative**: Polling (high latency); REST (overhead).  
**Interview Tip**: Lead with WebSocket/Redis, mention geo-routing. If probed, discuss consistency trade-offs.

### 4. Message Durability
**Challenge**: Ensure no message loss, even for offline users.  
**Solution**:  
- **Transactional Writes**: Message Service writes to Message + Inbox tables atomically.  
- **Client Ack**: Clients send `ack` on receipt; Message Service deletes from Inbox.  
- **Offline Sync**: On connect, query Inbox for undelivered messages, re-send.  
- **Timeout**: Retry delivery for non-acked messages (e.g., network drop) until 30 days.  
- **Why?**: Transactions ensure persistence; acks confirm delivery.  
- **Alternative**: No acks (risk duplicates); no transactions (data loss).  
**Interview Tip**: Propose transactions/acks, explain sync. If probed, discuss retries or ordering.

---

## Interview Checklist
- **Structure** (35 min):  
  1. Requirements: Functional/non-functional (2 min).  
  2. API: WebSocket commands (3 min).  
  3. Database: DynamoDB/S3/Redis (2 min).  
  4. High-Level: Microservices, flow (5 min).  
  5. Deep Dives: Scale, clients, latency, durability (15–20 min).  
- **Whiteboard** (verbalize):  
  - Draw: Client → L4 LB → Chat Service → Redis/DynamoDB/S3.  
  - Show: Chat creation (write), message (pub/sub → inbox), attachment (S3).  
- **Avoid Pitfalls**:  
  - Don’t use REST for messaging (high latency).  
  - Don’t skip inbox (offline delivery fails).  
  - Don’t ignore pub/sub (scaling bottleneck).  
  - Don’t store media in DB (costly).  
- **Trade-Offs**:  
  - WebSocket vs. polling: Latency vs. simplicity.  
  - Redis vs. consistent hashing: Speed vs. control.  
  - DynamoDB vs. SQL: Scale vs. joins.  
- **Mid-Level Expectations**:  
  - Define WebSocket APIs, data model.  
  - Propose DynamoDB for messages, S3 for media, Redis for routing.  
  - Address offline delivery naively (e.g., inbox table).  
  - Reason through probes; expect guidance.

---

This summary is streamlined for your preparation. Send the next system design, and I’ll keep the format consistent!
