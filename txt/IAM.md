## 📘 Study Notes: Firestore (Datastore) & Cloud SQL

Google Cloud tests your ability to choose the right database based on relational vs. NoSQL needs, regional vs. global scale, and how to handle concurrency and connection limits.

### 1. Firestore (Serverless NoSQL Document DB)

Firestore is a fully managed, serverless NoSQL document database. It is the successor to Cloud Datastore (and can run in Datastore mode). Use it for mobile/web apps, real-time syncing, offline support, and hierarchical data.

* **The 1-Write-Per-Second Limit (CRITICAL EXAM TRAP):**
* *The Problem:* A single Firestore document can only be updated about once per second. If you have a global "likes" counter for a viral video, concurrent writes will fail.
* *The Solution:* **Distributed Counters**. Shard the counter into multiple sub-documents (e.g., 10 shards), write to them randomly, and sum them up when reading.


* **Transactions vs. Batched Writes:**
* **Transactions (`db.runTransaction`):** Use when a write depends on a previous read (e.g., read stock level, check if > 0, decrement stock). Must perform *all reads before any writes*. Automatically retries on contention.
* **Batched Writes (`db.batch`):** Executes multiple write operations atomically, but **cannot contain reads**. Faster and less network overhead if you only need to bulk-insert or bulk-delete.


* **Query Limitations (Highly Tested):**
* You cannot chain multiple `array-contains` filters in a single query. (Fix: Use a map `roles: {admin: true, editor: true}` and query `.where('roles.admin', '==', true)`).
* You can only perform range filters (inequality: `<`, `>`, `<=`) on a **single field** per query.


* **Indexes:**
* Single-field indexes are created automatically.
* Queries with multiple fields require a **Composite Index**. If you test a complex query without one, Firestore returns an error containing a direct URL to build the exact index needed in the console. You deploy these via the `firestore.indexes.json` file.



### 2. Cloud SQL (Regional Relational DB)

Cloud SQL is a fully managed, regional relational database (MySQL, PostgreSQL, SQL Server). It scales vertically (up to massive RAM/CPU limits) but is fundamentally a single-node write architecture.

* **Cloud SQL vs. Cloud Spanner (The Ultimate Showdown):**
* *Use Cloud SQL when:* You are doing a "lift and shift" of a legacy app, your data fits in a single region, and you rely on traditional stored procedures or standard ORMs.
* *Use Spanner when:* You need global horizontal scaling, active-active multi-region writes, and petabytes of capacity.


* **High Availability (HA) vs. Read Replicas (KNOW THE DIFFERENCE):**
* **High Availability (HA):** Configures a synchronous standby instance in a *different zone within the same region*. Used strictly for Disaster Recovery (automated failover). **You cannot route read queries to the HA standby.**
* **Read Replicas:** Asynchronous copies of the primary instance. Used to offload heavy `SELECT` queries (like analytical reporting). *Exam Trap:* Never route writes (`INSERT/UPDATE`) to a read replica.


* **Serverless Connection Exhaustion (CRITICAL):**
* *The Problem:* Cloud Run or Cloud Functions scale out massively during a traffic spike. Every new container opens a new direct database connection, instantly crashing Cloud SQL with "Too many connections" errors.
* *The Solution:* Use a connection pooler (like **PgBouncer**) combined with the **Cloud SQL Auth Proxy**. Do *not* just increase the connection limit to infinity.


* **Cloud SQL Auth Proxy:**
* A daemon/sidecar you run alongside your app. It provides secure, encrypted access to Cloud SQL without needing to whitelist IP addresses or configure SSL manually. It uses IAM for authentication.
* *Exam Trap:* The Auth Proxy secures the connection, but by default, it does *not* provide connection pooling. You must pair it with a pooler or use specific pooling flags if available for your driver.



### 3. Disaster Recovery & Backups

* **Point-in-Time Recovery (PITR):** Allows you to recover a Cloud SQL database to a specific millisecond in the past (e.g., right before a developer accidentally dropped a table). It requires enabling **automated backups** and **binary logging** (MySQL) or **write-ahead logging (WAL)** (PostgreSQL).

---