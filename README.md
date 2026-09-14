# Carieeer: System Architecture & Technical Specification

A scalable, performance-optimized career marketplace platform engineered for high-scale data handling, deterministic skill evaluation, and event-driven asynchronous processing.

---

## 1. Requirements & System Scope

### 1.1. Functional Requirements

1. **Profile Management:** Ingestion and lifecycle management for Candidate and Employer profiles.
2. **Job Management:** Employers create, configure, and post job listings with required skill specifications.
3. **Matching Algorithm:** Automated discovery matching qualified candidates to newly published job listings.
4. **Browse & Apply Pipeline:** Candidates browse active job listings and submit applications; employers manage candidates through structured pipeline stages (`APPLIED`, `SCREENED`, `INTERVIEW`, `OFFER`, `REJECTED`).
5. **Skill-Gap Analysis:** Identifies missing technical proficiencies between a candidate's profile and target job requirements.
6. **Search Platform:** High-throughput search and filtering across active jobs and employer profiles.
7. **Multi-Channel Notifications:** Alerts dispatched for job matches, pipeline updates (interviews), and roadmap milestones.
8. **Roadmap Generation:** Creation of step-by-step learning paths derived from prerequisite dependencies of missing skills.

### 1.2. Non-Functional Requirements

* **Latency SLAs:**
* Standard CRUD and Search: $P99 < 150\text{ms}$.
* Skill-Gap Analysis, Matching Engine, and Roadmap Generation: $P95 < 2\text{s}$.


* **Deduplication:** The platform strictly prevents duplicate job applications.
* **Strong Consistency for Applications:** Application submissions and pipeline status updates require immediate, strong consistency.
* **Guaranteed Notification Delivery:** At-least-once delivery guarantee for all notification events.
* **Read Scalability:** The system is heavily read-skewed, requiring dedicated read replicas and multi-tier caching in Redis.

### 1.3. Out of Scope

* **Privacy & Security Deep-Dives:** Enterprise RBAC, detailed compliance frameworks (GDPR, SOC2), and cryptographic key management are excluded from the current scope.
* **Unstructured Resume Parsing:** PDF and document parsing are omitted; all skill inputs are entered and standardized at ingestion boundaries.

---

## 2. Core Domain Entities

* **Candidate:** Personal profile metadata, years of experience, standardized skill links, and profile vector embedding.
* **Employer:** Company information, verification status, and organization profile.
* **Job:** Role specifications, listing status (`DRAFT`, `OPEN`, `CLOSED`), standardized required skills, and job vector embedding.
* **Standardized Skills:** Canonical master taxonomy entries mapping technical skills to unique integer identifiers.
* **Skill Dependencies:** Self-referencing graph table (`prerequisite_skill_id`, `target_skill_id`) defining prerequisite learning paths.
* **Application:** Application lifecycle entity tracking candidate progression through the hiring pipeline (`APPLIED`, `SCREENED`, `INTERVIEW`, etc.).
* **Skill-Gap Analysis:** Computed set differences highlighting missing skills between candidates and jobs.
* **Roadmap:** Ordered, milestone-based learning curricula generated via recursive prerequisite queries.

---

## 3. Core API Interface

### 3.1. Submit Job Application

* **Endpoint:** `POST /Applications/{jobID}/apply`
* **Description:** Creates an application record for the authenticated candidate. Enforces strong consistency and prevents duplicate submissions.
* **Response:** `201 Created`

### 3.2. Update Application Status

* **Endpoint:** `PATCH /Applications/{jobID}/status`
* **Description:** Invoked by employers to transition a candidate's pipeline status (e.g., move to `INTERVIEW`).
* **Request Payload:**
```json
{
  "candidate_id": "cand_102",
  "status": "INTERVIEW",
  "note": "Technical screen scheduled"
}

```


* **Response:** `200 OK`

### 3.3. Search Jobs & Employers

* **Endpoint:** `GET /jobs/search?queryparameter=...`
* **Description:** High-throughput read endpoint querying active jobs and employers via indexed filters. Routed to read replicas.
* **Response:** `200 OK`

### 3.4. Candidate Recommendations

* **Endpoint:** `GET /Candidates/{CandidateID}/recommendations`
* **Description:** Retrieves pre-computed, ranked job matches produced by the matching engine.
* **Response:** `200 OK`

---

## 4. Architectural Topology: Monolith vs. Microservices (Pending Decision)

The system boundaries are logically split into distinct operational domains:

* `Profile Service`
* `Job Service`
* `Application Service`
* `Skill Gap Service`
* `Roadmap Service`
* `Job Matching Service`
* `Batch Service & Notification Service`

**Evaluation Status:**

Due to competition timeline constraints, the final architectural deployment model remains open. We have not finalized whether these bounded contexts will run as independently deployed **Microservices** or stay packaged together inside a highly modular **Monolith**. The final decision will be driven by future load testing and horizontal scaling requirements on individual bottlenecks (e.g., separating compute-heavy roadmap generation from lightweight job search).

---

## 5. Database Replication, Read Consistency & Caching

The persistence layer uses a relational **Primary DB** (PostgreSQL) paired with **Read Replicas** to satisfy read scalability requirements while maintaining write integrity.

### 5.1. Status Change Consistency: The 13-Second Redis TTL Bypass

Asynchronous database replication introduces eventual consistency lag. If a candidate applies for a job or an employer updates a status to `INTERVIEW`, reading from a lagging replica would show stale data (e.g., appearing as though the status never changed).

To prevent this without introducing global database locking:

1. When an application state change commits to the Primary DB, the backend writes a temporary key to Redis:
`SET user:{candidate_id}:primary_read_lock "true" EX 13`
2. For the next **13 seconds**, any read request from that user bypasses all read replicas and reads directly from the **Primary DB**.
3. Once the 13-second TTL expires (a duration exceeding the $99.9\text{th}$ percentile replication lag window), subsequent read traffic returns to reading from the read replicas.

### 5.2. Monotonic Replica Reads ("Time-Travel" Prevention)

When multiple read replicas are deployed, they may pull changes from the replication stream at different speeds. If a user makes two consecutive read requests:

* Request 1 hits **Replica A** (version/LSN: 1000).
* Request 2 hits **Replica B** (version/LSN: 950).

The user experiences a "time-travel" regression where newly visible data disappears. Each replica tracks a version/Log Sequence Number (LSN). When a user reads from a replica with a lower version than previously observed, the system must intervene.

**Two Architectural Options Under Evaluation:**

* **Option 1: Route to Primary DB on Lower Version**
If the assigned replica has an LSN lower than the client's session checkpoint, the routing proxy detects the lag and automatically routes that request to the Primary DB.
* **Option 2: Sticky Replica Pinning**
Pin each user session to a single, dedicated replica via consistent hashing on `user_id`. The user always reads from the same replica node, guaranteeing a strictly monotonic view of time.

*Evaluation Status:* Both options are designed, but due to time constraints, we have not finalized which approach is superior for production. Option 1 prevents stale reads at the cost of potential Primary DB load spikes during replica lag; Option 2 protects the Primary DB but requires failover balancing if a replica crashes.

---

## 6. End-to-End Component Workflow

```
                             [ API Gateway ]
                     (Rate Limiting, Load Balancer, Auth)
                      │          │             │            │
          ┌───────────┘          │             │            └───────────┐
          ▼ (post job)           ▼ (app status)▼ (profile)              ▼
    [Job Service]     [Application Service] [Profile Service]   [Skill Gap Service]
      │        │             │        │         │        │          │        │
      │        │             │        │         ▼        ▼          │        │
      │        │             │        │    [Embedding] [Redis]      │        │
      │        │             │        │      [Model]                │        │
      ▼        ▼             ▼        ▼         │                   ▼        ▼
   [RabbitMQ] ──► [Job Matching]   [Primary DB] ◄───────────────────────┘        │
      │                 │               ▲                                        │
      │ (consume jobs)  │ (write feed)  │ (recursive query)                      │
      │                 ▼               │                                        │
      ├───────────► [Primary DB] ───────┤                                        │
      │                 ▲               │                                        │
      │                 │ (feed batches)│                                        │
      ▼ (consume queue) │               │                                        │
  [Batch Service &  ────┘               └───────────── [Roadmap Service] ◄───────┘
   Notification Svc]                                   (consumes jobs from queue)

```

### 6.1. Ingestion & Profile Management

* **`Profile Service`:** Receives candidate profile submissions. Queries the master skill table in `primary DB` for standardization, stores frequently used skills in `Redis`, calls `embedding model` to generate the candidate vector, and writes the standardized profile (including years of experience) to `primary DB`.
* **`Job Service`:** Receives job postings from employers. Checks `Redis` for job status and cached standardized skills, resolves un-cached skills from the master table in `primary DB`, calls `embedding model` for job vectors, and stores job listings in `primary DB`. It publishes newly created jobs to `rabbitMq` (`job matching queue`).

### 6.2. Application Pipeline & Deduplication

* **`Application Service`:** Handles job applications and status transitions.
* **Deduplication Constraint:** A unique composite database index (`UNIQUE(candidate_id, job_id)`) guarantees candidates cannot apply for the same job twice.
* **Status Updates:** General status updates (`APPLIED`, `SCREENED`, `OFFER`, `REJECTED`) are persisted directly to `primary DB`. When an interview is scheduled, `Application Service` sends an interview notification directly to `rabbitMq` for candidate alerting.

### 6.3. Job Matching Engine

* **`Job matching Service`:** Consumes new job events from `rabbitMq`.
* Queries `primary DB` using vector similarity to find eligible candidate matches.
* Writes the resulting matches directly into the notification feed in `primary DB`.

### 6.4. Notifications & Batch Delivery

* **`Batch Service & notification service`:** Consumes the notification queue from `rabbitMq` for event alerts and reads from `primary DB` to retrieve notification feeds in batches.
* Dispatches aggregated push and email notifications while maintaining user activity feeds.

### 6.5. Skill-Gap & Roadmap Engine

* **`Skill gap service`:** Queries standardized candidate skills and standardized job requirements from `primary DB`. It computes the missing skills via set difference, persists the missing skills, and if requested, publishes an `"add roadmap job"` event to `rabbitMq`.
* **`Roadmap service`:** Consumes roadmap generation jobs from `rabbitMq`. It runs recursive queries on the self-referencing `skill_dependencies` table in `primary DB` to build the full prerequisite path, persists the resulting roadmap milestones, and sends a `"roadmap completion"` notification event to `rabbitMq`.
