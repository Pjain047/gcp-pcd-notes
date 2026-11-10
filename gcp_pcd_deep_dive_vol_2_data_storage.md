# Google Cloud Professional Cloud Developer

## Deep Dive Volume 2: Data Storage, Caching, and State Management

This volume covers the second largest domain of the Professional Cloud Developer exam. You must be able to select the correct database based on scalability, transaction requirements (ACID), regionality, and data structure. You must also understand how to design schemas (Primary Keys, Row Keys, Indexes) that avoid performance bottlenecks.

## 1. Relational Databases (OLTP)

### 1.1 Cloud SQL

Cloud SQL is a fully managed regional relational database service for MySQL, PostgreSQL, and SQL Server.

* **High Availability (HA) vs. Read Replicas:**
  * **HA (Zonal Failover):** Synchronously replicates data to a standby instance in a *different zone within the same region*. If the primary zone goes down, the standby automatically takes over. **Exam Trap:** HA is strictly for availability, *not* for read scaling. You cannot route read queries to the HA standby instance.
  * **Read Replicas:** Asynchronously replicates data. Used to offload read-heavy traffic (e.g., analytical queries, reporting dashboards) from the primary instance.
  * **Cross-Region Read Replicas:** Used for Disaster Recovery (DR). If an entire region goes down, you must manually promote the cross-region read replica to be a standalone primary instance.

* **Connection Management (Crucial for Serverless):**
  * Serverless platforms (Cloud Run, Cloud Functions) can scale out rapidly to thousands of instances. If each instance opens a new database connection, Cloud SQL will crash due to connection exhaustion.
  * **Solution:** Configure a connection pool in your application code (e.g., HikariCP for Java, `pg` pool for Node.js), limit the maximum connections per container (e.g., 5), and strictly set `max-instances` on the Cloud Run service.

* **Cloud SQL Auth Proxy:**
  * Often deployed as a sidecar container in GKE. 
  * The application connects to `localhost:5432` (because containers in the same pod share the network namespace). The Proxy securely encrypts the connection to Cloud SQL using the pod's Workload Identity.

### 1.2 Cloud Spanner

Cloud Spanner is a fully managed, globally distributed, horizontally scalable relational database with strict serializable ACID transactions.

* **When to choose Spanner over Cloud SQL:**
  * You need relational semantics (SQL, Joins, Foreign Keys) BUT your database is larger than Cloud SQL's limits (e.g., > 64TB or extremely high IOPS).
  * You need multi-region, global strong consistency with 99.999% availability.

* **Primary Key Hotspotting (Exam Critical):**
  * Spanner stores data lexicographically sorted by the Primary Key. It divides data into "splits" based on key ranges.
  * **The Trap:** Using a monotonically increasing value (like an auto-incrementing ID or a raw timestamp) as the first part of the Primary Key. All new writes will hit the exact same node (the "hot" split), while other nodes sit idle.
  * **The Solution:** 
    1. Use a Version 4 UUID.
    2. Bit-reverse sequential values.
    3. Prefix the key with a hash of the ID.
    4. Reorder the key to put a high-cardinality field first (e.g., use `(CustomerID, OrderTimestamp)` instead of `(OrderTimestamp, CustomerID)`).

* **Mock Exam Scenario (Testing Spanner in CI/CD):**
  * *Scenario:* You want to prevent performance issues by running automated tests against Spanner on every code commit in Cloud Build. Concurrent tests might overlap.
  * *Solution:* Start the **Cloud Spanner emulator** locally in the Cloud Build pipeline, load data, run tests, and shut down the emulator. Creating/destroying real Spanner instances per test is too slow and expensive.

## 2. NoSQL Databases

### 2.1 Firestore (Datastore Mode vs Native Mode)

Firestore is a serverless, NoSQL document database optimized for web and mobile applications with real-time sync and offline support.

* **Indexes:**
  * Firestore automatically creates single-field indexes for all fields in a document.
  * **Composite Indexes:** Required if a query filters on multiple fields (`where("city", "==", "Chicago")`) AND sorts by another field (`orderBy("age")`).
  * **Exam Trap:** Do not manually create single-field indexes for everything. You can use index exemptions to disable automatic indexing for large string fields or fields you never query, saving on write costs and storage.

* **Transactions and Batched Writes:**
  * **Transactions:** Atomic read-modify-write operations. They are optimistic and will automatically retry if another client modifies the document concurrently. *Never put non-idempotent external calls (like sending an email or billing a credit card) inside a transaction callback, as the callback may run multiple times.*
  * **Batched Writes:** Execute multiple writes atomically, but *cannot* read data.

* **Snapshot Listeners (Real-time):**
  * Clients can listen to document changes. The callback payload identifies if a document was `ADDED`, `MODIFIED`, or `REMOVED`.

### 2.2 Cloud Bigtable

Bigtable is a sparse, wide-column NoSQL database designed for massive throughput and sub-millisecond latency. Best for IoT telemetry, AdTech, and time-series data.

* **Row Key Design (Exam Critical):**
  * Bigtable only has ONE index: the Row Key. Designing this key is the most important part of Bigtable architecture.
  * Data is sorted lexicographically by the Row Key.
  * **Time-Series Pattern:** If you frequently query the *latest* data (e.g., latest 10 temperatures for a sensor), use a reverse timestamp in the row key: `sensor_id#reverse_timestamp`. This forces the newest data to the top of the read operation.
  * **Hotspotting:** Just like Spanner, sequential row keys (e.g., raw timestamps) cause hotspotting. Prepend a high-cardinality identifier (e.g., `device_id#timestamp`) or use "salting" (adding a calculated prefix) to distribute writes.

## 3. Object Storage: Cloud Storage (GCS)

Cloud Storage is used for unstructured data: images, videos, backups, and static website assets.

### 3.1 Storage Classes & Cost Optimization

* **Standard:** Frequent access (served instantly).
* **Nearline:** Accessed less than once a month (30-day minimum storage duration).
* **Coldline:** Accessed less than once a quarter (90-day minimum).
* **Archive:** Accessed less than once a year (365-day minimum). Best for regulatory backups.

### 3.2 Mock Exam Scenarios (Lifecycle & Retention)

* **Scenario A (Regulatory Compliance):** A financial app stores trade receipts. SEC regulations require receipts to be un-deletable for 7 years. After 3 years, they are rarely accessed.
  * *Solution:* 
    1. Apply a **Retention Policy** of 7 years on the bucket and explicitly lock it using **Bucket Lock** (WORM compliance - Write Once, Read Many). 
    2. Add an **Object Lifecycle Management** rule to transition objects to `Archive` storage after 3 years (1095 days).
  * *Why other options fail:* Lifecycle policies alone *do not* prevent a user with `storage.admin` permissions from manually deleting the file before 7 years. Only a locked Retention Policy physically prevents deletion.

* **Scenario B (Direct Client Uploads):** An app allows anonymous users (no Google Accounts) to view private images, and authenticated users to upload raw 5GB video files directly to GCS without proxying through a Cloud Run backend.
  * *Viewing Private Images:* The backend generates a **Signed URL (HTTP GET)** valid for a short time (e.g., 15 minutes) and passes it to the anonymous client.
  * *Uploading with Constraints:* The backend generates a **Signed Policy Document (HTTP POST)**. Unlike a standard PUT Signed URL, a POST Policy Document allows the backend to enforce strict constraints, such as `content-length-range: [0, 5368709120]` and `Content-Type: video/mp4`.

### 3.3 Security Boundaries (VPC-SC)

* **VPC Service Controls (VPC-SC):** 
  * *Scenario:* Prevent an application in Project A from using compromised credentials to read a GCS bucket in Project B.
  * *Solution:* Implement a VPC-SC perimeter around Project A. Even if the service account is granted IAM permissions in Project B, the perimeter blocks the API request from crossing the organizational network boundary, preventing **Data Exfiltration**.

## 4. In-Memory Caching & State Management

### 4.1 Cloud Memorystore (Redis & Memcached)

Memorystore provides fully managed Redis and Memcached. Used for sub-millisecond data retrieval.

* **Use Cases:** Session storage, leaderboards, rate-limiting counters, and caching DB query results.
* **Eviction Policies (Redis):**
  * `noeviction`: Returns errors when memory is full (never deletes data).
  * `allkeys-lru`: Evicts the Least Recently Used keys to make space. Best for general caching.
* **Connectivity (Serverless VPC Access):**
  * Memorystore instances are provisioned with private IP addresses inside your VPC.
  * *Exam Trap:* If a Cloud Run service gets a `502 Bad Gateway` trying to reach Memorystore, the most likely cause is a missing or misconfigured **Serverless VPC Access Connector** (or Direct VPC Egress). Cloud Run is serverless and outside your VPC by default; it needs a connector to route traffic to private internal IPs.

### 4.2 Handling Application State (Mock Exam Scenario)

* **Scenario:** An e-commerce platform needs to store user sessions and shopping cart data. Users log out after 30 minutes, but their shopping cart must remain intact when they log back in days later.
* *Solution:* 
  * Store the **session information** in **Memorystore for Redis** (ephemeral, fast access, short TTL).
  * Store the **shopping cart information** in **Firestore** (durable, persistent, user-specific data).
* *Why other options fail:* Storing shopping carts in Redis risks data loss if the cache is flushed or evicted. Storing sessions in BigQuery is an anti-pattern (BigQuery is for analytics, not low-latency OLTP).

## 5. Quick Decision Matrix for the Exam

| Database Requirement | Best Google Cloud Service |
| :--- | :--- |
| Ad-hoc SQL analytics, petabyte-scale, data warehousing | **BigQuery** |
| Global scale, relational SQL, strong consistency | **Cloud Spanner** |
| Regional scale, standard relational SQL (MySQL/PostgreSQL) | **Cloud SQL** |
| Serverless, document data, mobile/web real-time sync | **Firestore** |
| Massive throughput, time-series, IoT, flat wide-column | **Cloud Bigtable** |
| Sub-millisecond latency, ephemeral session state, caching | **Memorystore (Redis)** |
| Unstructured blobs, backups, videos, regulatory archives | **Cloud Storage** |

*End of Volume 2: Data Storage, Caching, and State Management.*