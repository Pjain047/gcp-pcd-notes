## 📘 Study Notes: Cloud Spanner (Databases Domain)

**Core Use Case & Positioning**
Cloud Spanner is a globally distributed, horizontally scalable, strictly consistent relational database (SQL).

* **Choose Spanner when:** You need both relational semantics (ACID transactions, SQL, foreign keys) **AND** horizontal scaling (multi-region, massive throughput).
* **Do NOT choose Spanner when:** You only need regional relational data (use Cloud SQL), or you don't need relational queries/ACID (use Bigtable/Firestore).

**Primary Key Design & Hotspotting (CRITICAL EXAM TOPIC)**
Spanner divides data into "splits" based on primary key ranges. If writes are concentrated in one range, a single node gets overwhelmed (a "hotspot"), while others sit idle.

* **The Anti-Pattern (Exam Trap):** Never use monotonically increasing values (like sequential IDs or pure `TIMESTAMP` columns) as the **first/leading** column of a primary key. This forces all new writes into the same physical split.
* **How to Fix Hotspots:**
1. **UUID (v4):** Distributes writes randomly.
2. **Bit-reverse sequential IDs:** Reverses the bits of a sequential ID so they distribute evenly.
3. **Hash Prefix:** Hash the sequential ID/timestamp and prepend it to the primary key.
4. **Swap column order:** If you have a composite key like `(Timestamp, UserID)`, swap it to `(UserID, Timestamp)` so writes are distributed across different users.



**Interleaved Tables vs. Foreign Keys**
This is how Spanner optimizes parent-child relationships (e.g., `Customers` and `Orders`).

* **Interleaved Tables:** Physically co-locates child rows with their parent row on the exact same underlying storage node. This makes reads and writes of the parent-child hierarchy incredibly fast.
* *Syntax:* `INTERLEAVE IN PARENT Customers ON DELETE CASCADE`


* **Foreign Keys:** Enforces referential integrity (ensures an order belongs to a valid customer) but **does not** physically co-locate the data.
* *Exam Trap:* If a question asks how to optimize performance and physical co-location for parent-child data, the answer is *always* Interleaving, not Foreign Keys.



**Secondary Indexes & The STORING Clause**
Used to look up data by a column other than the primary key.

* **Index-to-Base-Table Lookup:** By default, an index only contains the indexed column and the primary key. If your `SELECT` statement asks for columns not in the index, Spanner must do a costly secondary hop back to the base table.
* **The STORING (Covering) Clause:** You can store copies of non-indexed columns directly inside the index.
* *Example:* `CREATE INDEX UserEmailIdx ON Users(Email) STORING (FirstName, LastName)`
* *Exam Scenario:* If a query using an index is too slow, the solution is usually to add the requested columns to a `STORING` clause to create a covering index.



**Transactions & TrueTime**
Spanner uses Google's atomic clocks (TrueTime) to guarantee external consistency.

* **Read-Write Transactions:** Use pessimistic locking. High overhead, can fail and require retries.
* **Read-Only Transactions:** Lock-free. They perform much faster.
* **Stale Reads (Bounded Staleness):** If your application can tolerate data that is slightly out of date (e.g., up to 15 seconds old), configure a read-only transaction with bounded staleness. Spanner can serve this read from a local replica without coordinating globally, massively reducing latency.

---