## 📘 Study Notes: Caching Strategies & Content Delivery

Google Cloud tests your ability to optimize read-heavy workloads, manage session state, and deliver static assets globally with minimal latency.

### 1. Cloud Memorystore (Redis vs. Memcached)

Memorystore provides fully managed in-memory data stores. You must know when to choose which engine and how to configure memory management.

* **Redis vs. Memcached:**
* *Choose Redis:* You need complex data structures (hashes, lists, sets), persistence (saving to disk), High Availability (HA) with read replicas, or Pub/Sub messaging.
* *Choose Memcached:* You only need simple key-value caching, require multi-threading, and do *not* need persistence or HA.


* **Redis Eviction Policies (CRITICAL EXAM TRAP):**
* When Redis runs out of memory, its `maxmemory-policy` dictates what happens next.
* **`allkeys-lru`:** Evicts the Least Recently Used keys across the entire database. *Best for: General caching (like a product catalog).*
* **`volatile-lru`:** Evicts only keys that have an explicit Time-To-Live (TTL) set.
* **`noeviction`:** Never evicts any keys. Instead, it returns an Out of Memory (OOM) error for new write operations. *Best for: Critical session state data where dropping a key means logging a user out unexpectedly.*



### 2. Cloud CDN (Content Delivery Network)

Cloud CDN caches content at the edge of Google's network, drastically reducing latency for users and offloading traffic from your backend.

* **Cache Invalidation:**
* *The Scenario:* You deploy a new version of your frontend UI, but users still see the old CSS or JavaScript files.
* *The Solution:* Issue a **Cache Invalidation** request to Cloud CDN. This forces the edge nodes to drop the cached file and fetch the new one from your backend (Cloud Storage or HTTP Load Balancer).


* **Browser Caching vs. CDN Caching:**
* Even if you invalidate the CDN, a user's *local web browser* might still aggressively cache the file. To fix this, you must configure **HTTP `Cache-Control` headers** (e.g., `Cache-Control: no-cache, max-age=0`) directly on the backend Cloud Storage objects.



---

## 📘 Study Notes: Orchestration vs. Queues (Workflows vs. Tasks)

The exam heavily tests your ability to distinguish between asynchronous queueing and stateful workflow orchestration in microservice architectures.

### 1. Cloud Tasks (Asynchronous Queues)

Cloud Tasks is a fully managed service for distributed task queues.

* **Core Capabilities:**
* **Rate Limiting:** Granular control over how fast tasks are dispatched (e.g., `max_dispatches_per_second`). Perfect for calling third-party APIs with strict quotas.
* **Delayed Execution:** You can schedule a task to run at a specific future timestamp (e.g., "send a follow-up email in 2 hours").
* **Retries with Backoff:** Automatically retries failed HTTP calls with exponential backoff.


* **The Exam Trigger:** Use Cloud Tasks when the caller *does not need to wait* for the result (fire-and-forget), or when you need strict rate-limiting for outbound webhook dispatches.

### 2. Cloud Workflows (Stateful Orchestration)

Cloud Workflows executes a series of steps in a specific order, managing the state and passing data between services.

* **Core Capabilities:**
* **Step-by-Step Logic:** Step B depends on the JSON output of Step A.
* **Conditional Branching:** `switch` statements and `try/catch` error handling across different API calls.
* **Long-Running:** A single workflow execution can pause and wait for up to **1 year** (e.g., waiting for human approval via an HTTP callback).


* **The Exam Trigger:** Use Cloud Workflows when you need to coordinate a complex business process (e.g., Order Checkout: charge credit card -> if success, update inventory -> if out of stock, refund card).

---

## 📘 Study Notes: App Engine Configuration Files

If a scenario involves App Engine, you will often be tested on which specific YAML file is used to configure routing, indexes, or scheduled jobs.

* **`app.yaml` (The Core Config):**
* Configures the application's runtime (e.g., `python39`), instance class, environment variables, auto-scaling parameters (like `max_instances`), and basic URL handlers. Every App Engine service must have this file.


* **`dispatch.yaml` (Routing):**
* Used to override default routing rules.
* *Exam Scenario:* You want all requests starting with `[https://example.com/mobile/](https://example.com/mobile/)*` to route to a specific App Engine service named `mobile-backend`. You configure this in `dispatch.yaml`.


* **`cron.yaml` (Scheduled Jobs):**
* Configures scheduled tasks (cron jobs) that run at defined intervals (e.g., `every 24 hours`). It simply issues an HTTP GET request to a specific endpoint in your application.


* **`index.yaml` (Datastore Indexes):**
* Used to define composite indexes for Cloud Datastore (or Firestore in Datastore mode). If your application executes a complex query requiring an index that doesn't exist, you define it here and deploy the file.


* **`queue.yaml` (Legacy Task Queues):**
* Configures legacy App Engine task queues (push and pull). Note: Modern architectures should use Cloud Tasks instead, but this file still appears in legacy modernization questions.