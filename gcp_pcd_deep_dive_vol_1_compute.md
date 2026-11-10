# Google Cloud Professional Cloud Developer
## Deep Dive Volume 1: Compute & Cloud-Native Architecture

This volume covers the most heavily weighted domain on the PCD exam: designing highly scalable, secure, and reliable cloud-native applications. You must be able to decisively choose between Cloud Run, GKE, App Engine, and Compute Engine based on granular constraints (e.g., WebSockets, background threads, OS dependencies, scaling speed, and operational overhead).

---

## 1. The Compute Decision Matrix (Exam Blueprint)

When presented with a compute scenario on the exam, always read the final constraints first (e.g., "minimize operational overhead," "scale to zero," "custom Linux binaries"). 

### Decision Tree
1. **Does it require custom kernel modules, specialized OS licenses, or is it a legacy monolith that cannot be containerized?**
   * *Yes* → **Compute Engine**.
2. **Does it require specific Kubernetes APIs, sidecar containers (like Istio/Service Mesh), deep network topology control, or specific hardware accelerators (TPUs/custom GPUs)?**
   * *Yes* → **Google Kubernetes Engine (GKE)**. (Default to Autopilot unless OS/node-level control is explicitly requested).
3. **Is it a simple, source-based web application where the team refuses to write Dockerfiles and wants out-of-the-box scaling?**
   * *Yes* → **App Engine Standard**.
4. **Is it a stateless containerized microservice, web API, or event-driven webhook that experiences spiky traffic and needs to scale to zero?**
   * *Yes* → **Cloud Run**.

---

## 2. Cloud Run Exhaustive Deep Dive

Cloud Run is Google's fully managed serverless container platform (Knative under the hood). 

### 2.1 Concurrency & CPU Allocation
Unlike AWS Lambda or Cloud Functions Gen 1 (which handle 1 request per instance), Cloud Run can handle up to **1000 concurrent requests per instance** (default is 80).

* **I/O-Bound Workloads (Waiting on DB/Network):** High concurrency (e.g., 80-200). One container can process many requests simultaneously while waiting for a database to respond.
* **CPU-Bound Workloads (Image/Video Processing):** Low concurrency (e.g., 1-5). Prevents requests from starving each other of CPU cycles.

**CPU Allocation Billing Models:**
1. **Request-based billing (Default):** CPU is only allocated while an HTTP request is actively being processed. If you return an HTTP 200 OK and then try to run a background thread to process data, the container's CPU will be severely throttled, and the background task will likely fail or stall.
2. **Instance-based billing (`--no-cpu-throttling`):** CPU is always allocated as long as the container is alive. 
   * *Exam Trap:* Use instance-based billing ONLY when you have background threads that must continue running *after* the HTTP response is sent.

### 2.2 Scaling Controls (Min/Max Instances)
* **`min-instances`:** Keeps idle instances warm. Use this to prevent **Cold Starts** for latency-sensitive applications (e.g., Java Spring Boot apps that take 10 seconds to boot). Note: This incurs baseline costs even when there is no traffic.
* **`max-instances`:** Limits the maximum number of containers. 
   * *Exam Scenario:* "Your Cloud Run app scales up massively and crashes your Cloud SQL database due to connection exhaustion." 
   * *Solution:* Configure `max-instances` on Cloud Run to act as a backpressure mechanism, and implement a connection pool (e.g., maximum of 5 connections per instance).

### 2.3 Traffic Splitting & Revisions
Every deployment creates an immutable **Revision**. You can split traffic across revisions for Blue/Green or Canary deployments.

* **Targeted Beta Testing (Mock Exam Scenario):** 
  * *Scenario:* You want to test a new version of a Cloud Run app with *only specific registered beta users* based on application logic, without affecting normal users.
  * *Solution:* Deploy the new revision with a URL tag (e.g., `gcloud run deploy --tag=beta --no-traffic`). This creates a dedicated URL (`https://beta---myservice.run.app`). Your frontend code evaluates the user's beta status and routes their API calls to the tagged URL. 
  * *Why other options fail:* Standard traffic splitting (`--to-revisions NEW_REV=10`) routes 10% of *all* users randomly, which violates the requirement to only test registered beta users.

### 2.4 Cloud Run Jobs
Used for run-to-completion scripts, database migrations, or batch processing. They do not listen on an HTTP port. Triggered manually or via **Cloud Scheduler** (for cron jobs).

### 2.5 VPC Connectivity
Cloud Run lives in a Google-managed VPC, not your VPC. To connect to private resources (like a private Cloud SQL instance or Memorystore Redis):
1. **Serverless VPC Access Connector:** Provisions a `/28` subnet of VMs bridging Cloud Run to your VPC. Supports scaling up, but *not* scaling down.
2. **Direct VPC Egress:** The modern approach. Sends traffic directly into your VPC without provisioning bridge VMs.

---

## 3. Google Kubernetes Engine (GKE) Deep Dive

GKE is tested heavily in the context of workload configurations, probes, and autoscaling.

### 3.1 Autopilot vs Standard
* **GKE Standard:** You manage the underlying Compute Engine instances (Node Pools). You pay per node. Required if you need privileged pods, daemonsets that modify host kernels, or specific older versions of Kubernetes.
* **GKE Autopilot:** Google manages the nodes. You pay per pod (based on CPU/Memory requests). 
  * *Exam Scenario:* You need to run a highly CPU-intensive video encoding pod on an Autopilot cluster. How do you get CPU-optimized hardware if you can't create Node Pools?
  * *Solution:* Use a `nodeSelector` in your Pod specification requesting a specific **Compute Class** (e.g., `cloud.google.com/compute-class: "Scale-Out"` or a CPU-optimized class). Autopilot intercepts this request and automatically provisions the correct underlying hardware for that specific pod.

### 3.2 Probes (Health Checks) - Exam Critical
1. **Startup Probe:** Validates if the application has started. *Disables liveness and readiness checks until it succeeds.*
   * *Mock Exam Scenario:* A Node.js Machine Learning app takes 45-50 seconds to download a 2GB model file from Cloud Storage on startup. Traffic must not be served until it's done, and Kubernetes must not kill it.
   * *Solution:* Configure a `startupProbe` that checks for the file's existence (or hits an initialization endpoint). Configure a `readinessProbe` to allow traffic only after. 
   * *Why Liveness fails here:* If you use a `livenessProbe` with `initialDelaySeconds: 45`, and the download takes 46 seconds, Kubernetes kills the pod and restarts it, causing an infinite CrashLoopBackOff.
2. **Readiness Probe:** Validates if the pod can accept user traffic. If it fails, the pod is removed from the Service (Load Balancer) endpoints, but the pod is *not* killed. Use this when the app is temporarily overloaded or lost DB connection.
3. **Liveness Probe:** Validates if the pod is alive. If it fails (e.g., deadlock), the `kubelet` kills and restarts the container.

```yaml
# Example of the 45-second startup scenario
apiVersion: v1
kind: Pod
metadata:
  name: ml-model-app
spec:
  containers:
  - name: ml-container
    image: my-ml-app:v1
    startupProbe:
      httpGet:
        path: /init-status
        port: 8080
      failureThreshold: 30
      periodSeconds: 5 # Gives the app up to 150 seconds to start (30 * 5)
    readinessProbe:
      httpGet:
        path: /ready
        port: 8080
      periodSeconds: 10
```

### 3.3 Autoscaling in GKE
* **Horizontal Pod Autoscaler (HPA):** Scales the *number of pod replicas*. Usually based on CPU utilization.
  * *Rule:* HPA requires `requests` (CPU/Memory) to be defined in the pod spec to calculate utilization percentages.
* **Vertical Pod Autoscaler (VPA):** Automatically adjusts the CPU and Memory `requests` and `limits` of a pod. It generally recreates the pod to apply new sizes.
  * *Exam Trap:* Never use VPA and HPA simultaneously on the same metric (e.g., CPU). They will fight each other.
* **Cluster Autoscaler:** Scales the *number of nodes*. Triggers when pods are in a `Pending` state due to insufficient cluster resources.

### 3.4 GKE Networking
* **ClusterIP (Default):** Internal to the cluster only.
* **NodePort:** Opens a port (30000-32767) on every node's external IP. Mostly used for testing or legacy integrations.
* **LoadBalancer:** Provisions an External Network Load Balancer (L4) in GCP.
* **Ingress:** A Kubernetes object that provisions a Google Cloud External Application Load Balancer (L7). Routes HTTP(S) traffic to internal ClusterIP services based on paths (e.g., `/api/v1` to `api-svc`).
* **Gateway API:** The modern, role-oriented successor to Ingress. Separates infrastructure management (GatewayClass) from routing logic (HTTPRoute).

---

## 4. App Engine Deep Dive

App Engine is Google's original PaaS. It runs one application per GCP Project, divided into Services and Versions.

### 4.1 Standard vs Flexible
* **App Engine Standard:**
  * Runs in a secure sandbox.
  * Rapid scaling (milliseconds). Scales to zero.
  * Strict language support (Python, Java, Node, Go, PHP, Ruby).
  * Cannot write to local disk (ephemeral `/tmp` only).
  * Cannot run background threads or WebSockets (requires Flex).
* **App Engine Flexible:**
  * Runs application inside Docker containers on Compute Engine VMs.
  * Slow scaling (minutes). Requires at least 1 instance running at all times (does not scale to zero).
  * Full OS control (can use `apt-get` for custom binaries like `ffmpeg` or `libgeos`).
  * Supports WebSockets and background processing.
  * *Exam Trap:* If a question asks for a fast, scale-to-zero application that requires a custom C++ library installed via Linux packages, you *cannot* use App Engine Standard. You must use Cloud Run or App Engine Flex. Cloud Run is always preferred for modern architectures.

---

## 5. Compute Engine & VMs (GCE)

You will see Compute Engine on the Developer exam primarily in the context of legacy migrations, Lift-and-Shift, and Managed Instance Groups.

### 5.1 Interactive Voice Response (IVR) Scenario (Mock Exam Breakdown)
* *Scenario:* You are migrating a backend for an IVR payroll system. Characteristics:
  * Creates a *separate persistent gRPC connection* to the backend for each call session.
  * Call duration ranges from 1 to 30 minutes.
  * Connection interruptions cause latency (connections must be highly stable).
  * Calls spike around paydays.
* *Analysis:*
  * Cloud Run has a maximum timeout of 60 minutes, which could theoretically handle 30 minutes, but serverless environments can proactively scale down or shift connections, disrupting persistent gRPC streams.
  * Cloud Run Functions has a max timeout of 9 minutes (fails).
  * Compute Engine requires manual scaling and OS management (high effort).
  * **GKE Standard** is the correct answer. It perfectly handles persistent gRPC connections, allows long-running pods without arbitrary timeouts, handles auto-scaling via HPA during paydays, and offers lower operational overhead than raw VMs.

### 5.2 Managed Instance Groups (MIGs)
* **Instance Templates:** Immutable templates defining the VM (machine type, image, startup script). If you need to update a MIG, you create a *new* instance template and trigger a Rolling Update.
* **Autohealing:** Uses an HTTP/TCP health check. If a VM fails the check, the MIG deletes and recreates it.
* **Autoscaling:** Adds/removes VMs based on CPU, load balancing capacity, or custom Cloud Monitoring metrics.

---

## 6. Website Testing Scenario (Traffic Management)

* *Scenario:* You have a website deployed on Compute Engine. Marketing wants to test conversion rates between three different designs. You cannot change application code. How do you implement this?
* *Solution:* **Deploy the website on Cloud Run as three separate revisions and use traffic splitting.**
* *Why:* Cloud Run natively supports splitting traffic natively at the control-plane level (e.g., 33% / 33% / 33%) across deployed revisions. Doing this on Compute Engine requires complex Load Balancer URL maps and multiple MIGs. Cloud Run Functions is for event-driven snippets, not hosting full frontend websites.

---
*End of Volume 1: Compute & Cloud-Native Architecture.*