## 📘 Study Notes: Cloud Migration, Strangler Fig, & Legacy Modernization

Google Cloud emphasizes iterative, low-risk migrations over risky "big bang" rewrites. You will be tested on how to architect a safe transition from an on-premise monolithic application to a cloud-native microservices architecture on GCP.

### 1. The Core Migration Strategies

When an exam scenario presents a migration project, you must identify the correct strategy based on the business constraints (time, budget, risk).

* **Rehost (Lift and Shift):** Moving the application exactly as-is to Google Cloud.
* *Target:* **Compute Engine (VMs)** or VMware Engine.
* *When to use:* The data center lease is expiring in 30 days, or you have zero developer resources to rewrite code. It is fast but does not gain cloud-native benefits like auto-scaling to zero.


* **Replatform (Lift and Optimize):** Upgrading the underlying infrastructure without changing the core application logic.
* *Target:* Migrating a self-hosted MySQL database to **Cloud SQL**, or containerizing a legacy web app and running it on **GKE**.
* *When to use:* You want managed services (automated backups, patching) without a massive code rewrite.


* **Refactor (Re-architect):** Completely redesigning the application to be cloud-native.
* *Target:* **Cloud Run, Cloud Functions, Pub/Sub, Firestore/Spanner**.
* *When to use:* The business requires extreme scalability, global reach, and developer agility, and has the time/budget to rewrite the code.



### 2. The Strangler Fig Pattern (CRITICAL EXAM TOPIC)

This is the most highly tested architectural design pattern for legacy modernization. It is named after a vine that slowly grows around a host tree, eventually replacing it.

* **The Problem:** Rewriting a massive legacy monolith all at once (a "Big Bang" rewrite) is incredibly risky, takes years, and halts feature development.
* **The Solution (How Strangler Fig Works):**
1. **The Facade:** Place an **API Gateway** or **HTTP(S) Load Balancer** (the facade) in front of the legacy system. Initially, it routes 100% of traffic to the legacy monolith.
2. **Extract a Slice:** Identify one specific bounded business capability (e.g., the "Billing" module). Build that single module as a brand new microservice on **Cloud Run** or **GKE**.
3. **Route Traffic:** Update the API Gateway to route `/billing` requests to the new Cloud Run service, while routing all other requests to the legacy monolith.
4. **Repeat & Retire:** Continue extracting modules one by one. Once all traffic is routed to the new microservices, shut down the legacy monolith.


* **The Exam Trigger:** If a scenario asks how to migrate a monolith to microservices **"incrementally," "with minimal downtime,"** or **"without a big-bang cutover,"** the answer is always the Strangler Fig pattern.

### 3. Data Migration & Coexistence During Modernization

The hardest part of the Strangler Fig pattern is keeping the data synchronized while both the legacy and new systems are running simultaneously.

* **Change Data Capture (CDC):**
* *The Scenario:* Your new microservice uses Cloud Spanner, but the legacy monolith still uses an on-premise Oracle database. They must both reflect the same inventory levels during the 6-month migration.
* *The Solution:* Use **Datastream** to read the transaction logs of the legacy database in real-time and stream changes to Pub/Sub or Dataflow, which then updates the new Cloud Spanner database.


* **Database Strangling:**
* *Anti-Pattern:* Having the new microservices write directly to the legacy database. This creates tight coupling.
* *Best Practice:* Each new microservice should own its own modern database (Firestore, Cloud SQL, Spanner). Use asynchronous events (Pub/Sub) to keep the legacy DB updated if legacy modules still need the data.



### 4. Legacy Modernization (Monolith to Microservices)

When refactoring a monolith into microservices on Google Cloud, the exam tests your understanding of distributed system design.

* **Domain-Driven Design (DDD):** Use "Bounded Contexts" to decide where to split the monolith. (e.g., User Management is one service, Checkout is another).
* **Event-Driven Decoupling:**
* *The Trap:* Breaking a monolith into microservices but making them all call each other synchronously via HTTP. If one fails, they all fail (a "distributed monolith").
* *The Solution:* Use **Cloud Pub/Sub** or **Eventarc** to decouple them. (e.g., The Checkout service publishes an `OrderPlaced` event; the Inventory and Shipping services listen and react asynchronously).



---