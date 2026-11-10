## 📘 Study Notes: CI/CD, DevOps, & Supply Chain Security

Google Cloud’s DevOps philosophy on the exam revolves around **automation, secure supply chains, and zero-trust deployments**. You will be tested heavily on how to move code from a Git repository to a production GKE cluster or Cloud Run service securely.

### 1. Cloud Build (The CI/CD Engine)

Cloud Build executes your builds on Google's infrastructure using a series of containerized steps.

* **The Configuration:** Defined using a `cloudbuild.yaml` file. Every step in a Cloud Build pipeline is simply a Docker container executing a command (e.g., `name: 'gcr.io/cloud-builders/docker'`).
* **Triggers:** You automate pipelines by linking Cloud Build Triggers to Cloud Source Repositories, GitHub, or GitLab (e.g., trigger a build when code is pushed to the `main` branch).
* **Cloud Build Private Pools (CRITICAL EXAM TOPIC):**
* *The Problem:* By default, Cloud Build runs on public Google infrastructure. If your build needs to access a private GKE cluster, a private Cloud SQL instance, or an internal Nexus/Artifactory server on a private VM, the default Cloud Build workers cannot reach them.
* *The Solution:* Create a **Cloud Build Private Pool** and configure VPC Network Peering between the Private Pool's network and your VPC.
* *Exam Trap:* Never choose an option that suggests assigning public IP addresses to your internal servers or opening firewall rules to the public internet just so Cloud Build can reach them.



### 2. Artifact Registry (The Secure Vault)

Artifact Registry is the successor to Container Registry (GCR). It stores Docker images, plus language packages (npm, Maven, Python, etc.) and OS packages.

* **Vulnerability Scanning:**
* *Automatic Scanning:* Can be configured to automatically scan container images for OS and language package vulnerabilities (CVEs) the moment they are pushed to the registry.
* *Exam Scenario:* If a security team requires that no image with a "Critical" vulnerability is deployed, you integrate Artifact Registry scanning into your pipeline to fail the build, or use Binary Authorization to block it at deployment.



### 3. Binary Authorization (The Deployment Gatekeeper)

This is one of the most highly tested security concepts on the exam. It is a deploy-time security control (an admission controller) for GKE and Cloud Run.

* **The Core Concept:** It ensures that only trusted, verified container images are allowed to run in your production environment.
* **Attestors & Signatures:**
1. Your CI/CD pipeline (Cloud Build) finishes running unit tests and vulnerability scans successfully.
2. An **Attestor** (backed by Cloud KMS cryptographic keys) "signs" the container image digest, creating an attestation.
3. When Kubernetes tries to pull and run the image, **Binary Authorization** intercepts the request, verifies the cryptographic signature, and blocks the pod if the signature is missing or invalid.


* **Exam Trap:** If a question asks how to prevent unverified images from running on GKE, the answer is *always* Binary Authorization. It is *never* Kubernetes NetworkPolicies, IAM roles, or Cloud Armor.

### 4. Deployment Strategies (Canary & Blue/Green)

You need to know how to roll out traffic safely without breaking production.

* **Cloud Run Traffic Splitting:**
* Cloud Run makes canary deployments incredibly easy natively. You deploy a new revision with `--no-traffic`, and then explicitly route a percentage of traffic to it.
* *CLI Command:* `gcloud run services update-traffic my-service --to-revisions=v2=10,v1=90`


* **GKE Canary Deployments (Without Istio/Gateway):**
* If a question asks how to do a simple 10% canary rollout on native GKE *without* a service mesh, you achieve it by manipulating Replica counts.
* *The Solution:* Create two separate `Deployment` objects (one for v1 with 9 replicas, one for v2 with 1 replica). Ensure both deployments share the exact same pod labels so that a single Kubernetes `Service` routes traffic to all 10 pods, naturally resulting in a 90/10 traffic split.



---