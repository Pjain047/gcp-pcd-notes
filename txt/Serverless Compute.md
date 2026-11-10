## 📘 Study Notes: Serverless Compute (Cloud Run, App Engine, Cloud Functions)

Google loves testing your ability to choose the *exact right* serverless compute product based on subtle constraints like cold starts, custom OS dependencies, scaling requirements, and background processing.

### 1. The Ultimate Serverless Decision Tree

When reading a scenario, use this mental checklist to eliminate wrong answers instantly:

* **Do you have a Docker container?** → **Cloud Run**
* **Is it a single-purpose snippet of code reacting to a GCP event (like a file upload)?** → **Cloud Functions**
* **Is it a standard web app (source code only, e.g., Node.js/Python) and you want it to scale to zero?** → **App Engine Standard**
* **Does the web app require custom OS packages (e.g., `apt-get install lib-magick`) or custom background threads?** → **App Engine Flexible** (or Cloud Run).

---

### 2. Cloud Run (Highly Tested)

Cloud Run is Google's flagship serverless container platform. It handles HTTP requests (Services) and batch processes (Jobs).

* **Cloud Run Services vs. Cloud Run Jobs:**
* *Services:* Listens for web traffic/HTTP. Max timeout is usually 60 minutes.
* *Jobs:* Executes a run-to-completion script (no HTTP listener). Perfect for nightly BigQuery aggregations or database migrations. Max execution time is 24 hours.


* **CPU Allocation (The "Background Task" Trap):**
* By default, Cloud Run throttles (pauses) the CPU the millisecond it finishes responding to an HTTP request.
* *Exam Scenario:* If an app needs to run an asynchronous background thread or use OpenTelemetry exporters *after* returning the HTTP 200 response, you **must** configure **CPU is always allocated** (`--no-cpu-throttling`).


* **Concurrency:**
* Unlike Cloud Functions Gen 1 (which handles 1 request per instance), Cloud Run can handle up to **1000 concurrent requests** per container instance. This makes it highly cost-effective for spiky traffic.


* **Cold Starts & Connection Spikes:**
* If traffic spikes, Cloud Run creates new instances rapidly (Cold Starts).
* *Exam Trap:* This rapid scaling can overwhelm downstream databases (like Cloud SQL) with too many connections.
* *Fix:* Set `--min-instances` to keep some containers warm, and configure a connection pooler (like Cloud SQL Auth Proxy) or set a strict `--max-instances` limit.


* **Networking:**
* To connect to a private Cloud SQL instance or Redis Memorystore, use **Direct VPC Egress** (the modern, preferred way that avoids managing connector VMs) or a **Serverless VPC Access Connector**.



---

### 3. App Engine: Standard vs. Flexible

The exam will rigorously test if you know the exact boundary between these two environments.

| Feature | App Engine Standard | App Engine Flexible |
| --- | --- | --- |
| **Scaling** | Scales to **zero** instantly. | Minimum **1 instance** (does NOT scale to zero). |
| **Startup Time** | Milliseconds (fast cold starts). | Minutes (boots a Compute Engine VM underneath). |
| **Custom OS Packages** | **No.** Pre-configured sandboxes only. | **Yes.** Use `runtime: custom` and a `Dockerfile`. |
| **Background Threads** | **No.** (Requires Cloud Tasks/PubSub). | **Yes.** Can run persistent background threads. |
| **Local Disk Access** | Read-only (except for ephemeral `/tmp`). | Read/Write to local ephemeral disk. |

* **Exam Trap:** If a question says, *"Your app receives traffic once a week and needs to minimize costs to absolutely zero when idle,"* you **cannot** use App Engine Flexible. You must use App Engine Standard or Cloud Run.

---

### 4. Cloud Functions (Cloud Run Functions)

Used for lightweight, event-driven glue code.

* **Gen 1 vs. Gen 2:** Gen 2 is built on top of Cloud Run and Eventarc.
* **Gen 2 Superpowers:** Supports up to 60-minute timeouts for HTTP functions (Gen 1 was 9 mins), supports up to 1000 concurrent requests per instance, and supports traffic splitting.
* **Idempotency:** Cloud Functions guarantee *at-least-once* execution.
* *Exam Trap:* Because functions can retry on failure, your code **must be idempotent** (safe to execute multiple times with the same event payload).



---