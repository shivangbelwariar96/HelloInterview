<img width="1133" alt="Screenshot 2025-04-13 at 4 37 29 PM" src="https://github.com/user-attachments/assets/c4862d99-f351-4692-a015-0c7aebdd0112" />

# System Design: Coding Platform Like LeetCode

This is a concise, interview-ready summary for designing a coding platform like LeetCode, optimized for a 35-minute interview. It covers all critical points to succeed, omitting unnecessary details (e.g., full DB schemas) while aligning with your 20+ system design cheatsheet for quick study.

---

## Overview
**Name**: Coding Platform Service (LeetCode-like)  
**Purpose**: Platform for users to solve coding problems, submit solutions, and view live competition leaderboards.  
**Scope**: Core features (view problems, submit solutions, get feedback, view leaderboards). Excludes authentication, profiles, payments.

---

## Functional Requirements
- **View Problems**: Users can browse a list of coding problems.  
- **View/Code Problem**: Users can view a problem and code a solution in multiple languages.  
- **Submit Solution**: Users can submit code and get instant feedback.  
- **View Leaderboard**: Users can see live competition leaderboards.

---

## Non-Functional Requirements
- **Availability**: Prioritize availability over consistency.  
- **Security**: Isolate user code execution for safety.  
- **Performance**: Submission results in <5s; problem list/leaderboard <500ms.  
- **Scalability**: Support competitions with 100K users (~4K problems total).

---

## API Design
RESTful APIs with JWT for authentication.

1. **List Problems**:  
   - `GET /problems?page={int}&limit={int}`  
   - Returns: `{ problems: [{ id, title, level, tags }] }`  
   - Purpose: Fetch paginated problem list.  

2. **View Problem**:  
   - `GET /problems/{id}?language={string}`  
   - Returns: `{ id, title, statement, codeStub, testCases }`  
   - Purpose: Fetch problem details and code stub.  

3. **Submit Solution**:  
   - `POST /problems/{id}/submit`  
   - Body: `{ code: string, language: string }`  
   - Returns: `{ submissionId: string, results: { passed: bool, output: string[] } }`  
   - Purpose: Submit code, get test case results.  

4. **View Leaderboard**:  
   - `GET /leaderboard/{competitionId}?page={int}&limit={int}`  
   - Returns: `{ entries: [{ userId, score, time }] }`  
   - Purpose: Fetch ranked competition leaderboard.

---

## Database
**Choices**:  
- **Problems/Submissions**: DynamoDB for scalability, simple queries.  
- **Cache**: Redis for leaderboards.  
**Key Entities**:  
- **Problem**: ID, title, statement, level, tags, codeStubs {language: code}, testCases [{input, output}].  
- **Submission**: ID, userId, problemId, competitionId, code, language, results, timestamp.  
- **Leaderboard**: competitionId, userId, score, completionTime.  
**Indexing**:  
- DynamoDB: Partition key `problemId` (problems), `competitionId` (submissions).  
- GSI: `userId` for submissions.  
- Redis: Sorted set (`leaderboard:{competitionId}`).

---

## High-Level Design
**Architecture**: Monolith for simplicity; microservices for scale if needed.  
**Components**:  
1. **Client**: Web app with code editor (e.g., Monaco).  
2. **API Server**: Handles requests, routes to services.  
3. **Problem Service**: Manages problem list/details.  
4. **Submission Service**: Processes submissions, triggers execution.  
5. **Execution Service**: Runs code in isolated containers.  
6. **Leaderboard Service**: Computes live rankings.  
7. **Database**: DynamoDB for problems/submissions.  
8. **Cache**: Redis for leaderboards.  
9. **Queue**: SQS for submission execution.

**Data Flow**:  
- **List/View Problem**: Client → API Server → Problem Service → DynamoDB → Client.  
- **Submit**: Client → API Server → Submission Service → SQS → Execution Service → DynamoDB → Client.  
- **Leaderboard**: Client → API Server → Leaderboard Service → Redis/DynamoDB → Client.  

**Diagram** (verbalize):  
- Client → API Server → Problem/Submission/Leaderboard Service.  
- Problem → DynamoDB; Submission → SQS → Containers → DynamoDB; Leaderboard → Redis.  

---

## Deep Dives
Key challenges addressing non-functional requirements.

### 1. Secure Code Execution
**Challenge**: Run user code safely without crashing or compromising the system.  
**Solution**:  
- **Docker Containers**:  
  - Run code in isolated containers (read-only filesystem, no network).  
  - Set CPU/memory limits (e.g., 1 vCPU, 512MB).  
  - Timeout after 4s to meet 5s SLA.  
  - Use `seccomp` to restrict syscalls.  
- **SQS Queue**: Decouple submission from execution, retry on failure.  
- **Why?**: Containers ensure isolation; queue adds reliability.  
- **Alternative**: VMs (slow); serverless (cold starts); no isolation (security risk).  
**Interview Tip**: Propose containers with limits, mention SQS. If probed, discuss `seccomp` or sandboxing.

### 2. Efficient Leaderboard Fetching
**Challenge**: Live leaderboards for 100K users strain DB with frequent queries.  
**Solution**:  
- **Redis Sorted Set**:  
  - Store leaderboard (`leaderboard:{competitionId}` → `{userId, score, time}`, TTL post-competition).  
  - Update on submission: Increment score, set time.  
  - Fetch with `ZRANGE` for top N entries (<10ms).  
- **Polling**: Client polls every 5s; no WebSockets (overkill).  
- **Why?**: Redis for low-latency ranking; polling for simplicity.  
- **Alternative**: DB query (slow); WebSockets (complex).  
**Interview Tip**: Lead with Redis sorted set, justify polling. If probed, discuss TTL or WebSockets trade-offs.

### 3. Scalability for Competitions (100K Users)
**Challenge**: Handle ~1K QPS peak submissions (100K users, 10 submissions each over 90min).  
**Solution**:  
- **Horizontal Scaling**:  
  - Auto-scale API Server, Execution Service (10 instances, 32 vCPUs each).  
  - DynamoDB auto-scales with `competitionId` partitioning.  
- **SQS Queue**: Buffer submissions, smooth spikes (~300 vCPUs needed).  
- **Pre-Scaling**: Spin up containers pre-competition based on sign-ups.  
- **Why?**: Queue prevents overload; scaling handles bursts.  
- **Alternative**: No queue (execution bottleneck); single server (crash).  
**Interview Tip**: Estimate QPS (~1K), propose queue/scaling. If probed, discuss pre-scaling or retries.

### 4. Running Test Cases
**Challenge**: Execute test cases across languages efficiently.  
**Solution**:  
- **Standardized Test Cases**:  
  - Store in DynamoDB as JSON (`[{type, input, output}]`).  
  - Example: Tree input as array `[3,9,20,null,null,15,7]`, output `3`.  
- **Test Harness**:  
  - Per-language harness deserializes input, runs code, compares output.  
  - Example: Python harness converts array to `TreeNode`, calls `maxDepth`.  
- **Container Setup**: Preload harness, problem-specific classes (e.g., `TreeNode`).  
- **Why?**: Single test case set reduces maintenance; harness ensures portability.  
- **Alternative**: Language-specific tests (high maintenance); no harness (inconsistent).  
**Interview Tip**: Describe harness, give example. If probed, discuss serialization or edge cases.

---

## Interview Checklist
- **Structure** (35 min):  
  1. Requirements: Functional/non-functional (2 min).  
  2. API: Endpoints (2 min).  
  3. Database: DynamoDB/Redis (2 min).  
  4. High-Level: Monolith, flow (5 min).  
  5. Deep Dives: Security, leaderboard, scale, test cases (15–20 min).  
- **Whiteboard** (verbalize):  
  - Draw: Client → API Server → Services → DynamoDB/Redis/SQS/Containers.  
  - Show: Problem (read), submission (queue → execute), leaderboard (cache).  
- **Avoid Pitfalls**:  
  - Don’t skip isolation (security risk).  
  - Don’t query DB for leaderboard (slow).  
  - Don’t overcomplicate (e.g., WebSockets for small scale).  
  - Don’t ignore test case standardization (maintenance nightmare).  
- **Trade-Offs**:  
  - Containers vs. serverless: Control vs. simplicity.  
  - Redis vs. DB: Speed vs. durability.  
  - Queue vs. direct: Reliability vs. latency.  
- **Mid-Level Expectations**:  
  - Define APIs, data model.  
  - Propose DynamoDB for storage, containers for execution, Redis for leaderboard.  
  - Address security naively (e.g., containers).  
  - Reason through probes; expect guidance.

---

This summary is lean and tailored for your study. Send the next system design, and I’ll maintain the format!
