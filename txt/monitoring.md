## 📘 Study Notes: Observability & Cloud Ops (Logging, Monitoring, Trace, Profiler)

Google Cloud’s observability tools are heavily tested on the Professional Cloud Developer exam. You must know exactly which tool to reach for when diagnosing latency, memory leaks, or setting up alerts, as well as how to integrate them efficiently to save costs.

### 1. The Observability Decision Tree (Know Your Tools)

When a scenario asks you to troubleshoot a performance issue, use this breakdown:

* **"Why is my microservice architecture slow?"** → **Cloud Trace**. Tracks a single request as it hops across multiple services (e.g., API Gateway → Cloud Run → Cloud SQL). Identifies *network latency* and *service bottlenecks*.
* **"Which specific function in my code is eating 100% CPU or leaking memory?"** → **Cloud Profiler**. Continuously analyzes your application's call stack with low overhead. Uses **Flame Graphs** to show exactly which line of code or library is consuming resources.
* **"My application crashed, how many times has this exact stack trace happened?"** → **Error Reporting**. Automatically groups, deduplicates, and alerts on identical stack traces across your fleet.
* **"I need to track system health over time and trigger a pager."** → **Cloud Monitoring**.

### 2. Log-to-Trace Correlation (CRITICAL EXAM TOPIC)

The exam loves asking how to seamlessly jump from viewing a distributed trace in Cloud Trace directly to the specific application logs generated during that exact request in Logs Explorer.

* **The Requirement:** Your application must write logs in **Structured JSON format**.
* **The Magic Field:** You must inject a specific trace ID field into your JSON log payload. The exact field name is `[logging.googleapis.com/trace](https://logging.googleapis.com/trace)`.
* **The Format:** The value must be formatted exactly as: `projects/[PROJECT_ID]/traces/[TRACE_ID]`.

### 3. Cloud Logging & Cost Optimization

Logs are expensive to store. The exam tests how you manage log volume without losing operational visibility.

* **Log-Based Metrics:** You can create custom metrics based on incoming log streams (e.g., counting how many times "CheckoutFailed" appears).
* **The Exam Trick (Exclusion Filters):** You have a web app generating millions of `HTTP 200 OK` logs, costing too much money. You still want to monitor the *total count* of 200s, but you don't want to store the text logs.
* *The Solution:* Create a **Log-based Metric** to count the HTTP 200s. Then, create an **Exclusion Filter** on the `_Default` log sink to drop logs with `status: 200`. The metric is calculated *before* the exclusion filter drops the log, giving you the data without the storage cost!



### 4. Cloud Monitoring & Alerting

You must understand how metrics are aggregated to trigger alerts.

* **Alignment:** Grouping raw data points over a specific time window (e.g., taking the average CPU every 1 minute using `ALIGN_MEAN`).
* **Managed Service for Prometheus (GKE):**
* *Scenario:* You have a GKE app exposing standard Prometheus `/metrics`, but you don't want the operational burden of running Prometheus servers.
* *The Solution:* Enable **Managed Service for Prometheus** and deploy a `PodMonitoring` Custom Resource Definition (CRD). Google manages the scraping and ingestion directly into Cloud Monitoring.



### 5. Cloud Profiler Deep Dive

* **How it works:** You install a lightweight language-specific agent (e.g., `@google-cloud/profiler` for Node.js or the native `pprof` for Go) into your application code.
* **Exam Trap:** If a question says a container is hitting 100% CPU and Cloud Trace only shows that the time is being spent *somewhere inside* the container, Trace is the wrong tool to go deeper. You must switch to **Cloud Profiler** to look at the heap and CPU utilization of the internal functions.

---