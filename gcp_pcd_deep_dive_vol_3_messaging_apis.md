# Google Cloud Professional Cloud Developer

## Deep Dive Volume 3: Messaging, Orchestration, and API Management

This volume tackles the integration domain. Modern cloud-native applications rely on asynchronous communication and robust API front-doors. You must be able to choose the correct decoupling mechanism based on constraints like rate limiting, event fan-out, workflow state, and developer tiers.

## 1. Asynchronous Messaging: Pub/Sub vs. Cloud Tasks

The exam *always* tests your ability to distinguish when to use Cloud Pub/Sub versus Cloud Tasks. 

### 1.1 Cloud Pub/Sub (Event-Driven / Fan-Out)

Pub/Sub is designed for high-throughput, implicit event-driven architectures. The publisher doesn't know who is consuming the message.

*   **Pattern: Fan-Out.** One publisher sends an `OrderCreated` event to a Topic. Three separate Subscriptions (Billing, Inventory, Shipping) receive their own copy of the message and process it independently.
*   **Delivery Guarantees:** **At-least-once delivery**. Because messages can be delivered more than once (especially during network timeouts), your consumer application **must be idempotent** (safe to process the same message twice).
*   **Ordering:** Global ordering is not guaranteed. If sequence matters, you must publish messages using an **Ordering Key** (e.g., `customer_id`).
*   **Push vs. Pull Subscriptions:**
    *   **Pull:** The consumer (e.g., a GKE worker pod) explicitly requests messages. Best when the consumer wants to control its own processing rate (flow control).
    *   **Push:** Pub/Sub sends an HTTP POST request to an endpoint (e.g., a Cloud Run service). Best for serverless applications. Pub/Sub handles the delivery pacing based on endpoint success rates.
*   **Dead Letter Queue (DLQ):** If a message fails to process (unacknowledged) after a configured number of delivery attempts (e.g., 5 attempts), it is forwarded to a Dead Letter Topic to prevent it from blocking the queue.

*   *Mock Exam Scenario:* An e-commerce app sends confirmation emails via a Cloud Run function. The 3rd-party email provider is returning HTTP 500 errors, causing data loss.
    *   *Solution:* Decouple the system. Have the main app publish to a **Pub/Sub topic**. Have the Cloud Run function consume via a Push subscription with a Dead Letter Topic configured.

### 1.2 Cloud Tasks (Targeted Execution / Rate Limiting)

Cloud Tasks is designed for explicit, point-to-point asynchronous execution. The creator of the task *knows* exactly which webhook or App Engine service will execute it.

*   **Pattern: Rate Limiting & Scheduling.** 
*   **Key Features (Why it beats Pub/Sub for specific use cases):**
    *   **Rate Limits:** You can explicitly set `max_dispatches_per_second` and `max_concurrent_dispatches`. This is strictly required when calling fragile downstream 3rd-party APIs that enforce strict rate limits (e.g., "10 requests per second").
    *   **Scheduling:** You can set a `schedule_time` to execute a task in the future (e.g., "send a follow-up email exactly 2 hours from now"). Pub/Sub cannot do this.
    *   **Task Management:** You can pause queues or delete individual tasks before they execute.
*   **Retry Configuration (`retryConfig`):**
    *   `maxAttempts`: Total allowed tries (e.g., 10).
    *   `minBackoff` / `maxBackoff`: Bounds for exponential backoff (e.g., 10s to 300s).
    *   `maxDoublings`: How many times the delay doubles before capping out.

*   *Decision Rule:* If the prompt mentions "API rate limits," "throttling," or "scheduled execution delay," the answer is **Cloud Tasks**. If it mentions "event streaming," "multiple consumers," or "data ingestion," the answer is **Pub/Sub**.

## 2. Orchestration vs. Choreography

### 2.1 Cloud Workflows (Orchestration)

Workflows is a fully managed service for orchestrating stateful, multi-step sequences.

*   **When to use:** You need to orchestrate independent microservices in a specific order, pass data between them, handle conditional branching (switch statements), or manage complex retry/compensation logic if step 3 fails.
*   *Mock Exam Scenario (Image Processing Migration):* You need to migrate an on-premise pipeline that processes images: 1. Remove Watermark -> 2. Resize -> 3. Apply Filter. 
    *   *Solution:* Use **Cloud Workflows** to orchestrate sequential HTTP calls to three distinct **Cloud Run** services. 

### 2.2 Eventarc (Choreography)

Eventarc routes `CloudEvents` from Google Cloud sources (like Cloud Storage, BigQuery, or Audit Logs) to destinations like Cloud Run, GKE, or Workflows.

*   *Exam Scenario:* You want a Cloud Run application to execute *only* when a new object is finalized in a specific Cloud Storage bucket.
    *   *Solution:* Create an **Eventarc trigger** filtering on `type=google.cloud.storage.object.v1.finalized` pointing to your Cloud Run service.

## 3. Data Processing Pipelines (Dataflow)

While heavily tested in the Data Engineer exam, PCD expects you to know when to use Dataflow for application integration.

*   **Cloud Dataflow (Apache Beam):** Fully managed streaming and batch data processing.
    *   *Use Case:* Windowed aggregation. 
    *   *Mock Exam Scenario:* A mobile app collects global user votes. You need to count votes within every 30-minute window and store them for analysis. 
    *   *Solution:* App -> Pub/Sub -> **Dataflow Pipeline** (processes 30-minute windows) -> BigQuery.

## 4. API Management

The exam tests your ability to choose between Apigee, Cloud Endpoints, and API Gateway, as well as knowing Apigee's specific policies.

### 4.1 Service Selection

*   **Apigee:** Enterprise-grade API management. Use when you need Developer Portals, API Monetization, complex XML/JSON payload transformations, and advanced security.
*   **Cloud Endpoints:** Distributed API management based on the ESP (Extensible Service Proxy). Best for gRPC APIs, OpenAPI specs on Compute Engine/GKE, and low-latency internal Google Cloud routing.
*   **API Gateway:** A fully managed, simpler version of Endpoints. Best for serverless backends (Cloud Run, Cloud Functions, App Engine).

### 4.2 Apigee Policies & Developer Tiers

Apigee allows you to manage consumers via **API Products**, Developer Apps, and API Keys.

*   **Quota Policy:** Enforces limits over a longer time period (e.g., 10,000 requests per month).
*   **SpikeArrest Policy:** Smooths out traffic bursts by strictly limiting the rate at a given moment (e.g., 100 requests per second). *Trap: SpikeArrest is NOT for billing quotas.*
*   **VerifyAPIKey Policy:** Validates the key and identifies the app/product.

*   *Mock Exam Scenario (Apigee Tiers):* You have a GKE backend and want to offer a **Basic Tier (100 req/min)** and a **Premium Tier (1000 req/min)** to external developers.
    *   *Solution:* Deploy a *single* GKE backend. In **Apigee**, create two **API Products** (Basic and Premium) and attach a **Quota Policy** to them. Register developers to generate their API Key. Apigee enforces the quotas at the edge before traffic ever hits GKE.
    *   *Why other options fail:* Deploying two separate GKE endpoints wastes compute resources. GKE HPA cannot scale based on API keys, nor can it enforce billing quotas.

## 5. Security for Web Apps (IAP)

### 5.1 Identity-Aware Proxy (IAP)

IAP establishes a central authorization layer for applications accessed by HTTPS, enabling an organizational Zero Trust model.

*   **When to use:** You have an internal web application (e.g., HR portal, admin dashboard) deployed on GKE, Cloud Run, or App Engine, and you want to authenticate users using their Google Workspace accounts *without* writing custom login code in the application.
*   *Mock Exam Scenario:* You are deploying a containerized app behind a global load balancer. You do not have time to develop authentication at the application level.
    *   *Solution:* Use **Identity-Aware Proxy (IAP)** combined with Cloud Load Balancing. 

*End of Volume 3: Messaging, Orchestration, and API Management.*