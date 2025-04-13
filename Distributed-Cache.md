# System Design: Distributed Cache Like Redis

This is a concise, interview-ready summary for designing a distributed cache like Redis, optimized for a 35-minute interview. It addresses all critical points to excel, aligning with your 20+ system design cheatsheet for efficient study. The design is "hard" due to high availability, low latency (<10ms), and scaling to 1TB data and 100K req/s with eventual consistency.

---

## Overview
**Name**: Distributed Cache (Redis-like)  
**Purpose**: Store key-value pairs in memory across multiple nodes for fast access.  
**Scope**: Core features (set/get/delete, TTL, LRU eviction). Excludes durability, strong consistency, complex queries.

---

## Functional Requirements
- **Set/Get/Delete**: Store, retrieve, and remove key-value pairs.  
- **TTL**: Configure expiration times for keys.  
- **LRU Eviction**: Evict least recently used keys when memory is full.

---

## Non-Functional Requirements
- **Availability**: Highly available, survives node failures.  
- **Consistency**: Eventual consistency acceptable.  
- **Latency**: <10ms for get/set operations.  
- **Scalability**: Handle 1TB data, 100K req/s.  
- **Fault Tolerance**: No data loss on node failure (within consistency model).

---

## Scale Estimations
- **Data**: 1TB total, ~1MB/key avg ≈ 1M keys.  
- **Requests**: 100K req/s (mixed get/set/delete).  
- **Nodes**: 1TB ÷ 24GB/node ≈ 50 nodes (32GB RAM, 24GB usable).  
- **Throughput**: 100K req/s ÷ 50 nodes ≈ 2K req/s/node.  
**Insight**: High read/write throughput, memory-bound, hot keys critical.

---

## API Design
RESTful APIs with client library handling distribution.

1. **Set Key-Value**:  
   - `POST /:key`  
   - Body: `{ value: string, ttl: int }`  
   - Returns: `{ ok: true }`  
   - Purpose: Store key-value with optional TTL.  

2. **Get Key-Value**:  
   - `GET /:key`  
   - Returns: `{ value: string }`  
   - Purpose: Retrieve value by key.  

3. **Delete Key**:  
   - `DELETE /:key`  
   - Returns: `{ ok: true }`  
   - Purpose: Remove key-value pair.

---

## Database
**Choice**: In-memory hash table + doubly-linked list per node.  
**Key Entities**:  
- **Key-Value**: key (string), value (string), expiry (timestamp), node pointers.  
**Indexing**:  
- Hash table: `key → Node` (O(1) lookup).  
- Doubly-linked list: Tracks LRU order (O(1) move/evict).  
**Replication**: Async across nodes.  
**Sharding**: Consistent hashing by `key`.

---

## High-Level Design
**Architecture**: Distributed nodes with consistent hashing, async replication.  
**Components**:  
1. **Client**: App/service using cache via library.  
2. **Client Library**: Handles sharding, routing, batching.  
3. **Cache Nodes**: Store key-value pairs, manage LRU/TTL.  
4. **Coordinator**: Zookeeper for node discovery, hash ring.  
5. **Load Balancer**: Distributes initial client requests.

**Data Flow**:  
- **Set**: Client → Library → Cache Node (hash(key)) → Hash Table + Linked List.  
- **Get**: Client → Library → Cache Node → Hash Table → Client.  
- **Delete**: Client → Library → Cache Node → Remove from Hash Table/Linked List.  
- **Replication**: Leader node → Replicas (async).  
- **Expiration**: Background janitor scans linked list.  
- **Eviction**: LRU via linked list tail.

**Diagram** (verbalize):  
- Client → Library → Load Balancer → Cache Nodes (hash ring).  
- Nodes: Hash Table + Linked List, async replication, Zookeeper for coordination.

---

## Deep Dives
Key challenges addressing non-functional requirements.

### 1. High Availability and Fault Tolerance
**Challenge**: Survive node failures without downtime, eventual consistency.  
**Solution**:  
- **Async Replication**:  
  - Each key has 1 leader, 2 replicas (3 copies total).  
  - Writes go to leader, async propagate to replicas.  
  - Reads can hit any replica (eventual consistency).  
- **Failure Detection**:  
  - Zookeeper monitors heartbeats, removes failed nodes from hash ring.  
  - Clients reroute to new leader/replica via library.  
- **Recovery**:  
  - New node joins, syncs data from replicas.  
  - Partial sync for hot keys to minimize downtime.  
- **Back-of-Envelope**:  
  - 1TB ÷ 3 copies = 333GB/leader, ~7GB/node (50 nodes).  
  - 100K req/s ÷ 50 nodes = 2K req/s/node (manageable).  
- **Why?**: Async replication balances availability/simplicity; Zookeeper ensures coordination.  
- **Alternative**: Sync replication (higher latency); no replication (data loss).  
**Interview Tip**: Propose async replication, sketch failover. If probed, discuss replica lag or quorum reads.

### 2. Scalability with Consistent Hashing
**Challenge**: Scale to 1TB, 100K req/s, even key distribution.  
**Solution**:  
- **Consistent Hashing**:  
  - Keys hashed (e.g., MurmurHash) to a ring, mapped to nodes.  
  - Each node owns a range, ~20 virtual nodes/node for balance.  
  - Client library routes requests to correct node.  
- **Node Addition/Removal**:  
  - Add node: Reassign ~2% keys (1/50 nodes).  
  - Remove node: Replicas take over, sync to new nodes.  
- **Load Balancing**:  
  - Initial requests via L7 load balancer, then direct via library.  
- **Back-of-Envelope**:  
  - 1M keys ÷ 50 nodes = 20K keys/node.  
  - 100K req/s ÷ 50 nodes = 2K req/s/node, ~5ms/req (within 10ms).  
- **Why?**: Consistent hashing minimizes remapping; virtual nodes balance load.  
- **Alternative**: Modulo hashing (massive remapping); central router (SPOF).  
**Interview Tip**: Draw hash ring, explain key routing. If probed, discuss virtual nodes or rebalance cost.

### 3. Handling Hot Keys
**Challenge**: Hot keys (reads/writes) overwhelm single node.  
**Solution**:  
- **Hot Reads**:  
  - Cache hot keys on multiple replicas (e.g., 5 copies).  
  - Client library randomly selects replica for reads.  
  - Detect via request counters (e.g., >10K req/s/key).  
- **Hot Writes**:  
  - Append random suffix (e.g., `key:1`, `key:2`).  
  - Client writes to all suffixes, reads from any.  
  - Merge writes on leader (background job).  
- **Scaling**:  
  - Hot key metadata in Zookeeper, updated every 10s.  
  - Client library caches hot key config.  
- **Back-of-Envelope**:  
  - 1K hot keys × 5 copies = 5K key copies.  
  - 10K req/s/hot key ÷ 5 replicas = 2K req/s/replica.  
- **Why?**: Replication spreads read load; suffixes distribute writes; client library simplifies logic.  
- **Alternative**: Dedicated hot key nodes (complex); no mitigation (hotspots).  
**Interview Tip**: Propose replication for reads, suffixes for writes. If probed, discuss detection or merge logic.

---

## Interview Checklist
- **Structure** (35 min):  
  1. Requirements: Functional/non-functional (2 min).  
  2. Estimations: Data, QPS (2 min).  
  3. API: REST endpoints (1 min).  
  4. Database: Hash table + linked list (2 min).  
  5. High-Level: Nodes, sharding, replication (5 min).  
  6. Deep Dives: Availability, scalability, hot keys (20–23 min).  
- **Whiteboard** (verbalize):  
  - Draw: Client → Library → Cache Nodes (hash ring) → Hash Table + Linked List.  
  - Show: Set (hash → node → store), replication (leader → replicas), hot key (suffixes).  
- **Avoid Pitfalls**:  
  - Don’t skip replication (single node failure kills cache).  
  - Don’t use modulo hashing (poor scaling).  
  - Don’t ignore hot keys (hotspots crash nodes).  
  - Don’t propose disk storage (high latency).  
- **Trade-Offs**:  
  - Async vs. sync replication: Availability vs. consistency.  
  - Consistent vs. modulo hashing: Stability vs. simplicity.  
  - Hot key suffixes vs. dedicated nodes: Complexity vs. isolation.  
- **Mid-Level Expectations**:  
  - Define API, single-node hash table + linked list.  
  - Propose naive sharding (e.g., modulo), pivot to consistent hashing with hints.  
  - Miss hot keys, address replication naively.  
  - Reason through probes; expect guidance.

---

This summary is streamlined for your study. Send the next system design, and I’ll maintain the format!
