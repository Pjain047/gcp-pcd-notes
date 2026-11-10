## 📘 Study Notes: API Management (API Gateway, Apigee, Cloud Endpoints)

Google Cloud offers three distinct API management tools. The Professional Cloud Developer exam will test your ability to choose the right one based on backend architecture, budget, and enterprise requirements.

### 1. The API Decision Tree

When a scenario asks you to expose a backend service securely, use this matrix:

* **Do you need to monetize the API, create a developer portal, or manage APIs across hybrid/multi-cloud environments?** → **Apigee**
* **Do you have a simple serverless backend (Cloud Run, Cloud Functions, App Engine) and want a fully managed, zero-maintenance gateway?** → **API Gateway**
* **Are you deploying a gRPC backend, or do you want to run the API proxy as a sidecar container inside a GKE pod?** → **Cloud Endpoints**

### 2. Apigee (Enterprise API Management)

Apigee is a heavy-duty, full-lifecycle API platform.

* **When to use it:** Enterprise-grade routing, complex traffic transformations (e.g., SOAP to REST), monetization (charging users per API call), and advanced analytics.
* **Exam Trap:** Apigee is expensive and complex. If a question asks for a "cost-effective, lightweight" way to secure a Cloud Function, Apigee is the *wrong* answer.

### 3. Cloud Endpoints (The Containerized Proxy)

Cloud Endpoints uses a distributed API gateway architecture using the **Extensible Service Proxy (ESP)** or **ESPv2**.

* **How it works:** ESPv2 is an Envoy-based proxy that you deploy as a container. In GKE, it is typically deployed as a **sidecar container** living inside the same Pod as your application.
* **Supported Protocols:** It fully supports **gRPC** and OpenAPI (Swagger).
* **Exam Scenario:** If the question mentions an application running on GKE or Compute Engine that requires deep, low-latency gRPC inspection, choose Cloud Endpoints.

### 4. API Gateway (The Serverless Champion)

API Gateway is a fully managed service built specifically for serverless backends.

* **No Infrastructure:** Unlike Cloud Endpoints, you do not deploy or manage ESPv2 containers. Google handles the proxy for you.
* **Exam Scenario:** If a team has dozens of Cloud Functions and Cloud Run services and wants to expose them under a single unified REST API endpoint (`[api.company.com/v1/](https://api.company.com/v1/)...`) with the least operational overhead, choose API Gateway.

### 5. API Security & JWT Authentication (CRITICAL EXAM TOPIC)

All three services support API Keys, Firebase Authentication, and OAuth 2.0. However, the exam specifically tests how you validate **JSON Web Tokens (JWTs)**.

* **The Concept:** You want the API gateway to cryptographically verify a JWT *before* the request is allowed to reach your backend code.
* **The Configuration:** You configure this inside the **OpenAPI specification (Swagger YAML)** file uploaded to the gateway.
* **The Magic Field:** You define the security definitions and provide the URL to the public keys using the exact field `x-google-jwks_uri`.
* **Exam Trap:** Never choose an option that suggests writing custom Node.js/Python middleware in your Cloud Run container to parse and validate the JWT signature. The architectural best practice is to offload validation to the API Gateway.

---