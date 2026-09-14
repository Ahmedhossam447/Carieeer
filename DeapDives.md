# Technical Deep Dives: Architectural Trade-Offs & Edge Cases

This document provides in-depth technical justifications, mathematical formulations, and trade-off analyses for the core architectural decisions of the platform.

---

## 1. Database Replication & Read Consistency Guarantees

The persistence tier utilizes PostgreSQL deployed in a **Single-Primary, Multi-Replica** topology. The Primary DB absorbs all state-modifying write operations (`INSERT`, `UPDATE`, `DELETE`), while read operations are distributed across asynchronous read replicas to satisfy the 7,500 Read QPS requirement.

```
                      ┌──────────────────────────────────────┐
                      ▼                                      │
               [ Primary DB ]                                │
               (Writes Only)                                 │
                     │                                       │
     Replication     ├───────────────────────────┐           │ 13s Cache Bypass Window
     Stream (WAL)    ▼                           ▼           │ (Read-Your-Own-Writes)
             [ Read Replica A ]          [ Read Replica B ]  │
             (LSN: 0/3004F20)            (LSN: 0/3004D10)    │
                     ▲                           ▲           │
                     └─────────────┬─────────────┘           │
                                   │                         │
                         [ Replication Router ] ─────────────┘
                                   ▲
                                   │ Session Checkpoint Headers
                         [ Application Gateway ]

```

### 1.1. Read-Your-Own-Writes via 13-Second Redis TTL Pinning

Under asynchronous replication, changes committed on the Primary node are propagated via Write-Ahead Logging (WAL) to read replicas with a non-zero replication lag (typically 50ms to 2000ms under standard operational conditions).

If an employer advances an application stage via `PATCH /Applications/{jobID}/status` (e.g., from `APPLIED` to `INTERVIEW`), an immediate browser redirect or page reload hitting an asynchronous replica would present stale data, making it appear as though the update failed.

To guarantee **Read-Your-Own-Writes** consistency without routing all general traffic to the Primary DB:

1. Upon committing any application status mutation, the write handler sets an ephemeral key in Redis:
```text
SET user:{candidate_id}:primary_read_lock "true" EX 13

```


2. Any subsequent read request originating from that authenticated session checks Redis. If the key exists, the routing middleware forces the read query to execute directly against the **Primary DB**.
3. The 13-second TTL was calculated based on system SLAs: it comfortably exceeds the $99.9\text{th}$ percentile replication lag window while guaranteeing that general read traffic naturally falls back to the read replicas without administrative intervention.

---

### 1.2. Monotonic Replica Reads ("Time-Travel" Prevention)

In a multi-replica setup, asynchronous replication can drift unevenly across nodes due to network jitter or localized I/O contention. Replica $A$ might process WAL records up to Log Sequence Number `LSN: 0/3004F20`, while Replica $B$ lags behind at `LSN: 0/3004D10`.

If a user issues two consecutive read requests:

* **Request 1** hits Replica $A$ (sees latest application state).
* **Request 2** hits Replica $B$ (sees older state).

The user observes a perceived time-travel regression where updated records disappear. To prevent this, each replica transaction state is tagged via its current LSN version checkpoint, injected into client session headers ($V_{\text{rep}}$). When a client presents $V_{\text{prev}}$, the routing proxy inspects whether the target replica has caught up ($V_{\text{new}} \ge V_{\text{prev}}$).

#### Evaluated Architectural Options (Decision Pending Load Testing)

Due to competition timeline constraints, two options remain under active evaluation:

* **Option A: Dynamic Primary Fallback Routing**
If the selected replica's LSN is lower than the client's observed version checkpoint, the router flags a lag violation and automatically reroutes the read query to the **Primary DB**.
* *Advantage:* Absolute consistency guarantee across arbitrary replica clusters.
* *Trade-off:* Spikes in replication lag can cause a stampede of read traffic onto the Primary DB, threatening write availability.


* **Option B: Sticky Session Replica Pinning**
The application gateway uses consistent hashing on `user_id` to bind a user session to a single designated replica node. Because the user continuously queries the same node, their view of time is strictly monotonic.
* *Advantage:* Shields the Primary DB completely from read spillover.
* *Trade-off:* Requires re-hashing and session migration logic if a replica node becomes unhealthy or restarts.



---

## 2. In-Memory Set Math & Native In-Database Vector Matching

To prevent polyglot operational overhead, the architecture deliberately avoids introducing external Python microservices for analytics and matchmaking.

```
Candidate Profile               Target Job
 [10 (Linux), 12 (Git)]          [10 (Linux), 12 (Git), 42 (Docker), 104 (K8s)]
          │                               │
          └───────────────┬───────────────┘
                          ▼
           [Step 1: Set-Difference Engine]
           Missing Skills: [42, 104] (Microsecond RAM diff)
                          │
                          ▼
           [Step 2: DAG Topological Sort]
           Ordered Sequence: [42 -> 104]
                          │
                          ▼
           [Step 3: Curriculum Hydration]
           Attach project tasks & time estimates
                          │
                          ▼
           [Step 4: Persist & Return Roadmap]

```

### 2.1. In-Memory Integer Set Difference ($O(N)$)

Because candidate and job skill arrays are mapped strictly to canonical integer identifiers at ingestion boundaries, computing a skill gap does not require heavy natural language processing. It is executed via an in-memory set-difference operation directly within the core service runtime:


$$\text{Skill Gap} = \text{Job.RequiredSkillIDs} \setminus \text{Candidate.SkillIDs}$$

* **Computational Complexity:** For arrays containing 10 to 30 elements, a hash-set difference executes in **$< 10\mu\text{s}$ in RAM**.
* **Operational Rationale:** Making an inter-service RPC/HTTP call to a standalone Python container would introduce 15ms to 30ms of serialization and network overhead for a calculation that the core backend resolves in memory.

---

### 2.2. Native Database Vector Search via `pgvector`

Semantic matching between candidate profiles and job requirements is delegated directly to the database engine using PostgreSQL's `pgvector` extension with Hierarchical Navigable Small World (**HNSW**) indexing.

Cosine distance is calculated natively in C inside the storage engine:


$$\text{distance} = 1 - \cos(\theta) = 1 - \frac{u \cdot v}{\Vert{}u\Vert{}_2 \Vert{}v\Vert{}_2}$$

```sql
SELECT candidate_id, 1 - (embedding <=> :job_vector) AS match_score
FROM candidate_profiles
WHERE is_active = TRUE 
  AND years_of_experience >= :min_exp
ORDER BY embedding <=> :job_vector ASC
LIMIT 500;

```

* **Index Performance:** HNSW indexing maintains logarithmic search complexity ($O(\log N)$) across high-dimensional vector spaces, returning the top 500 candidate vectors in under 40ms directly from database buffer memory.
* **Storage Footprint:** For 20 million registered candidates with 1536-dimensional float vectors, vector storage fits within standard high-memory database instances, eliminating the need for dedicated external vector databases (e.g., Pinecone, Milvus).

---

## 3. Skill Standardization & Graph-Based Roadmap Generation

### 3.1. Master Skill Table vs. Junction Dependency DAG

To support dependency traversal, skill information is partitioned into two relational tables:

1. **`skills` (Master Entity):** Stores canonical metadata (`id`, `name`, `category`).
2. **`skill_dependencies` (Self-Referencing Graph Junction):** Represents directed prerequisite edges.

```sql
CREATE TABLE skill_dependencies (
    prerequisite_skill_id INT REFERENCES skills(id) ON DELETE CASCADE,
    target_skill_id       INT REFERENCES skills(id) ON DELETE CASCADE,
    is_mandatory          BOOLEAN DEFAULT TRUE,
    PRIMARY KEY (prerequisite_skill_id, target_skill_id)
);

```

---

### 3.2. Synchronous Inspection vs. Asynchronous Roadmap Enrollment

| Operational Attribute | Skill-Gap Inspection | Roadmap Generation & Enrollment |
| --- | --- | --- |
| **Trigger** | Candidate views a job posting | Candidate clicks "Enroll in Roadmap" |
| **Execution Path** | Synchronous HTTP (`GET /jobs/{id}/skill-gap`) | Asynchronous Event (`POST /candidates/{id}/roadmaps`) |
| **Response Contract** | `200 OK` ($< 50\text{ms}$) | `202 Accepted` (Task queued via RabbitMQ) |
| **Processing Workload** | RAM-based integer array set subtraction | Recursive DAG traversal + milestone persistence |

When full roadmap generation is requested:

1. The request thread returns a `202 Accepted` response with a pending tracking ID.
2. A task is enqueued to RabbitMQ.
3. The **`Roadmap Service`** consumes the message and executes a recursive query across `skill_dependencies` to retrieve the prerequisite hierarchy:
```sql
WITH RECURSIVE SkillChain AS (
    SELECT prerequisite_skill_id, target_skill_id, 1 AS depth
    FROM skill_dependencies
    WHERE target_skill_id = :target_skill_id
    UNION ALL
    SELECT sd.prerequisite_skill_id, sd.target_skill_id, sc.depth + 1
    FROM skill_dependencies sd
    JOIN SkillChain sc ON sd.target_skill_id = sc.prerequisite_skill_id
)
SELECT prerequisite_skill_id, depth FROM SkillChain ORDER BY depth DESC;

```


4. The worker applies a **Topological Sort** to serialize prerequisites into an un-conflicted linear milestone curriculum.
5. Milestones are bulk-inserted into `roadmap_milestones`, and a completion event is published to RabbitMQ to alert the user.

---

## 4. Event-Driven Messaging, Batching & Broker Resilience

```
[ Application / Job Service ]
             │
             ├── 1. Writes Entity (Application / Job Record)
             │
             └── 2. Emits Domain Event
                         │
                         ▼
                  [ RabbitMQ Exchange ]
                         │
         ┌───────────────┼───────────────┐
         ▼ (urgent)      ▼ (standard)    ▼ (bulk)
   [Interview Queue]  [Roadmap Queue]  [Batch Match Queue]
         │               │               │
         └───────────────┼───────────────┘
                         ▼
              [ Notification Service ]
                         │
             ┌───────────┴───────────┐
             ▼                       ▼
      [ Primary DB ]         [ External Gateways ]
   (In-App Feed INSERT)        (FCM / APNs Push)

```

### 4.1. Event-Driven Push Model (vs. Batch Polling)

Matching is triggered strictly via an event-driven **Push Model** upon job publication (`JobPublishedEvent`). Periodic polling crons were eliminated because they repeatedly evaluate static job records, leading to wasted database I/O and latency spikes.

---

### 4.2. Producer-Side Chunking & Consumer Batching

When a newly published job matches 500 candidates, publishing 500 individual messages creates excessive network roundtrips and broker channel overhead.

* **Producer-Side Chunking:** The matching worker bundles matched candidate IDs into discrete batches of **50 candidates per message payload**:
```json
{
  "job_id": "job_9412",
  "batch_index": 1,
  "candidate_ids": [101, 204, 308, 412, "...up to 50 IDs"]
}

```


* **Consumer-Side Bulk Operations:**
1. The `Notification Service` consumes the chunked payload and executes a single multi-row `INSERT` into `candidate_notifications` on the Primary DB.
2. The worker transmits device tokens to external notification gateways (e.g., Firebase Cloud Messaging, APNs) using their bulk multicast APIs in a single HTTP POST request.
3. Once both the database write and external dispatch succeed, the consumer confirms delivery in RabbitMQ via `basic.ack(multiple: true)`.



---

### 4.3. Broker Network Failure Resilience

To ensure reliability during transient network hiccups or broker restarts:

* Services publishing to RabbitMQ maintain long-lived TCP connections with active connection pooling.
* RabbitMQ **Publisher Confirms** are enforced: write handlers hold published messages in an in-memory retry buffer with exponential backoff until a positive acknowledgment (`basic.ack`) is returned by the broker.
* *Outbox Pattern Boundary:* The platform intentionally relies on direct broker publishing with retry buffers rather than the Transactional Outbox pattern to minimize schema complexity and avoid background outbox polling overhead under current project constraints.

---

## 5. Architectural Topology: Modular Monolith vs. Microservices

The system is partitioned into clear bounded contexts (`Profile`, `Job`, `Application`, `Skill-Gap`, `Roadmap`, `Job Matching`, and `Notification`).

### Trade-Off Analysis & Current Stance

| Evaluation Metric | Modular Monolith Deployment | Distributed Microservices Deployment |
| --- | --- | --- |
| **Inter-Service Latency** | Near zero (in-memory function calls & shared memory) | 5ms to 25ms per network RPC/HTTP hop |
| **Operational Overhead** | Low (single CI/CD pipeline, unified logging) | High (independent clusters, service mesh, tracing) |
| **Independent Scalability** | Low (scales the entire application process) | High (scale matching workers independently from APIs) |
| **Data Consistency** | Easy (in-process coordination) | Complex (distributed sagas, eventual consistency) |

**Current Decision:**

Because resource bottlenecks cannot be accurately predicted prior to real-world traffic testing, the application services are deployed within a unified modular codebase that interacts asynchronously via RabbitMQ.

Whether high-load workers (such as the `Job Matching Service` or `Roadmap Service`) will be extracted into isolated microservice containers will be decided based on production CPU and memory metrics observed during horizontal scale benchmarking.
