## 📘 Study Notes: Data Processing (Dataflow, Dataproc) & Pub/Sub

Google Cloud tests your ability to decouple microservices, ingest streaming data, and process massive datasets. You must know exactly when to choose native GCP tools versus managed open-source ecosystems.

### 1. Cloud Pub/Sub (Asynchronous Messaging)

Pub/Sub is a global, horizontally scalable message broker. It is the glue that connects microservices and data pipelines.

* **Delivery Guarantees & Idempotency (CRITICAL):**
* By default, Pub/Sub guarantees **At-Least-Once** delivery. This means duplicates *will* happen.
* *The Exam Rule:* Your subscriber application **must be idempotent** (processing the same message twice must not corrupt data). Use a database (like Firestore/Spanner) to track processed message IDs.
* *Exactly-Once Delivery:* This is a newer feature you can enable on a subscription, but best practice still dictates building idempotent consumers.


* **Message Ordering:**
* Pub/Sub does not guarantee order by default.
* *The Solution:* Enable **Message Ordering** on the subscription and include an `OrderingKey` (e.g., `customer_id`) in the publisher's message payload. Messages with the same key are delivered in strict order.


* **Push vs. Pull Subscriptions:**
* **Pull:** The subscriber actively asks for messages. Best for backend workers handling heavy processing where they need to control their own throughput.
* **Push:** Pub/Sub sends an HTTP POST request to a webhook endpoint (like Cloud Run or Cloud Functions). Best for serverless environments. *Exam Trap:* The push endpoint must have a valid SSL certificate (HTTPS).


* **Poison Pills & Dead Letter Topics (DLT):**
* If a message consistently crashes your application (a "poison pill"), it will block ordered processing or waste CPU cycles retrying indefinitely.
* *The Fix:* Configure a **Dead Letter Topic**. Set `max_delivery_attempts` (e.g., 5). After 5 failed acks, Pub/Sub automatically routes the bad message to the DLT for manual inspection.


* **Pub/Sub Schemas:**
* Enforces data structure (Avro or Protobuf) *before* the message is accepted by the topic, preventing malformed JSON from breaking downstream pipelines.



### 2. Cloud Dataflow (Apache Beam)

Dataflow is a fully managed, auto-scaling runner for **Apache Beam** pipelines. It is the GCP-native choice for data transformation.

* **Batch vs. Stream (The Unified Model):**
* Dataflow processes both batch (files in Cloud Storage) and streaming (real-time Pub/Sub messages) using the exact same programming model.


* **The Streaming Decision Tree:**
* *Exam Scenario:* You are receiving raw IoT telemetry via Pub/Sub. You need to filter out invalid readings, group them by device ID, calculate a 5-minute average, and write the result to Bigtable.
* *The Answer:* **Cloud Dataflow**. It is the *only* tool designed for complex real-time transformations, aggregations, and windowing of streaming data.


* **Windowing & Watermarks (Highly Tested):**
* Because streaming data can arrive late (e.g., a mobile device loses cell service and uploads data an hour later), Dataflow uses **Windows** (grouping data by time) and **Watermarks** (the system's heuristic for "I think all data for this window has arrived").
* You must configure **Triggers** to tell Dataflow what to do with late-arriving data (e.g., update the BigQuery aggregate or discard it).


* **Dataflow Templates:**
* Allows non-developers to execute Dataflow pipelines via the console or API without writing Java/Python code (e.g., the "Pub/Sub to BigQuery" Google-provided template).



### 3. Cloud Dataproc (Managed Hadoop/Spark)

Dataproc is a fully managed Apache Hadoop and Apache Spark environment.

* **The "Lift and Shift" Rule:**
* *Exam Scenario:* A company has existing Apache Spark, Hadoop, or Hive workloads running on-premise and wants to migrate them to GCP with the *least amount of code rewrite*.
* *The Answer:* **Cloud Dataproc**.
* *Exam Trap:* Do *not* choose Dataflow if the prompt specifically mentions migrating *existing* Spark/Hadoop jobs, as rewriting them in Apache Beam would be too much effort.


* **Ephemeral Clusters (Cost Optimization):**
* On-premise Hadoop clusters are usually long-running. On GCP, you should decouple compute from storage.
* *Best Practice:* Store your data in **Cloud Storage** (using the Cloud Storage connector), not in HDFS on the cluster nodes. Create a Dataproc cluster just to run the job, and **delete the cluster** as soon as the job finishes to save money.