## 📘 Study Notes: Cloud Storage, Bigtable, & BigQuery

Google Cloud tests your ability to select the right storage or database solution based on latency, throughput, schema requirements, and cost-optimization strategies.

### 1. Cloud Storage (Object Storage)

Cloud Storage is for unstructured data (images, videos, backups, static web assets). The exam focuses heavily on lifecycle management, security, and temporary access.

* **Temporary Access (Highly Tested):**
* *Signed URLs:* Allows users *without* a Google account to download or upload a file for a limited time (e.g., 15 minutes). Requires a Service Account to generate the cryptographic signature.
* *Signed Policy Documents:* Used specifically when a user needs to upload a file directly via an HTML form (`HTTP POST`) and you need to enforce strict constraints (e.g., file size must be < 5MB, content type must be `image/jpeg`).


* **Cost Optimization & Retention:**
* *Object Lifecycle Management:* Automatically transition objects to cheaper storage classes (Nearline -> Coldline -> Archive) or delete them based on `Age` or `IsLive` status.
* *Bucket Lock (Retention Policies):* WORM (Write Once, Read Many) compliance. Once locked, **nobody** (not even Project Owners) can delete or modify the file until the retention period expires.


* **Static Web Hosting & Caching:**
* *Exam Trap:* If a browser is aggressively caching old versions of a single-page app (like `index.html`), the solution is to set HTTP `Cache-Control` metadata headers (e.g., `Cache-Control: no-cache`) on the Cloud Storage object.



### 2. Cloud Bigtable (Wide-Column NoSQL)

Bigtable is designed for massive scale, single-digit millisecond latency, and extremely high throughput (millions of reads/writes per second). Use it for IoT time-series data, ad-tech, and financial trading ledgers.

* **Row Key Design (The #1 Bigtable Exam Topic):**
* Data is lexicographically sorted by the row key.
* *The Anti-Pattern:* Using a sequential ID or a monotonically increasing timestamp at the *front* of the row key (e.g., `2026-11-09#DeviceA`). This causes **write hotspotting** because all new data hits the exact same physical node.
* *The Fix:* Use a hash prefix, reverse the domain name, or swap the order (e.g., `DeviceA#2026-11-09`).


* **Time-Series Sorting Trick:**
* *Exam Scenario:* You need to retrieve the most recent events first (newest-to-oldest) as quickly as possible.
* *The Solution:* Invert the timestamp (`[MAX_INT - TIMESTAMP]`) so that the newest records naturally sort to the top of a lexicographical scan.


* **No Multi-Row Transactions:** Bigtable does not support multi-row ACID transactions (unlike Spanner). It only guarantees atomicity at the single-row level.

### 3. BigQuery (Enterprise Data Warehouse)

BigQuery is an analytical SQL database (OLAP) meant for petabyte-scale data crunching. It is not for transactional workloads (OLTP).

* **Performance & Cost Optimization (CRITICAL):**
* BigQuery charges by the amount of data *scanned*.
* *Partitioning:* Divides a table into segments (usually by `DATE` or `TIMESTAMP`). This drastically reduces cost because queries can skip scanning entire days/months of data.
* *Clustering:* Sorts data within a partition based on frequently filtered columns (e.g., `customer_id`).
* *Exam Rule:* If an exam question asks how to optimize costs for time-based analytics, the answer is almost always **Partitioning by date/time**.


* **Authorized Views (Security):**
* *Exam Scenario:* You have a table with PII (Personally Identifiable Information) and aggregated sales data. Marketing needs the sales data but must be blocked from seeing the PII.
* *The Solution:* Create an **Authorized View** that queries only the aggregated data. Grant Marketing access to the View, but *do not* grant them access to the underlying dataset.


* **Data Ingestion:**
* *Batch Load:* Free. Best for daily CSV/JSON uploads from Cloud Storage.
* *Streaming Inserts (`tabledata.insertAll`):* Charges a fee per GB but makes data available for querying almost instantly. Best for real-time dashboards.



---