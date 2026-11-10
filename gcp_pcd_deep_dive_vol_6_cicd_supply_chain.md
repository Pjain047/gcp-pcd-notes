# Google Cloud Professional Cloud Developer

## Deep Dive Volume 6: CI/CD, DevOps, and Software Supply Chain

The software supply chain and automated deployment pipeline represent a critical domain on the Professional Cloud Developer exam. You must understand how to construct repeatable build pipelines, enforce secure artifact governance, guarantee container integrity, and streamline the local developer workflow.

## 1. Cloud Build & Pipeline Orchestration

Cloud Build is Google Cloud's native, serverless CI/CD service that executes builds as a series of containerized steps defined in a `cloudbuild.yaml` or `cloudbuild.json` file.

### 1.1 Anatomy of a `cloudbuild.yaml` File

Each step in a build executes inside an ephemeral container. 

* **Execution Flow:** By default, build steps run **sequentially** (each step waits for the previous step to complete successfully). 

* **Parallel Execution (`waitFor`):** You can decouple steps by specifying the `waitFor` property. Using `waitFor: ['-']` allows a step to start immediately in parallel when the build initializes.

* **Persistent Workspace (`/workspace`):** All files generated or downloaded in the workspace directory persist across build steps.

```yaml
steps:
  - id: 'run-unit-tests'
    name: 'python:3.12-slim'
    entrypoint: 'bash'
    args: ['-c', 'pip install -r requirements.txt && pytest']
    waitFor: ['-'] # Starts immediately in parallel if other setup steps exist

  - id: 'build-container'
    name: 'gcr.io/cloud-builders/docker'
    args: ['build', '-t', 'us-central1-docker.pkg.dev/$PROJECT_ID/app/backend:$COMMIT_SHA', '.']
    waitFor: ['run-unit-tests'] # Waits for tests to pass successfully

images:
  - 'us-central1-docker.pkg.dev/$PROJECT_ID/app/backend:$COMMIT_SHA'
```

### 1.2 Default Pools vs. Private Pools

* **Default Pool:** Multi-tenant, shared serverless execution environment managed by Google. Fully sufficient for standard public builds and source building.

* **Private Pool:** Customer-specific, dedicated worker pool. 

  * *Exam Use Case:* Use a Private Pool when your CI/CD pipeline needs to securely reach private resources inside your managed VPC (such as an internal database, private Git repository, or on-premises system connected via Cloud Interconnect/VPN) during the build process.

### 1.3 Secure Secret Handling in Cloud Build

* **The Anti-Pattern:** Never store passwords, API keys, or tokens in plaintext within environment variables or substitution variables inside `cloudbuild.yaml`.

* **The Correct Pattern:** Use Cloud Build's integration with **Secret Manager**. Retrieve the secret at runtime using the `availableSecrets` block or mount it securely using Docker BuildKit (`--mount=type=secret`).

## 2. Artifact Registry & Software Supply Chain Security

Artifact Registry is the evolution of Container Registry, providing secure storage for Docker containers, Maven packages, npm modules, and Python packages.

### 2.1 Immutable Digests vs. Mutable Tags

* **Mutable Tags (e.g., `:latest`, `:v1.0`):** Can be overwritten in the repository. If a tag is updated maliciously or accidentally, different clusters could run different versions of code under the same name.

* **Immutable Digests (e.g., `@sha256:abc123xyz...`):** Cryptographically binds the image to its exact contents. 

  * *Exam Rule:* Always reference container images by their **immutable SHA256 digest** in production deployment manifests (Kubernetes Deployments, Cloud Run configurations) to guarantee reproducibility and prevent supply chain substitution attacks.

### 2.2 Artifact Analysis & Vulnerability Scanning

Artifact Registry automatically integrates with **Artifact Analysis** to scan container images for known Common Vulnerabilities and Exposures (CVEs) upon push.

### 2.3 Binary Authorization

Binary Authorization is a deploy-time security control that ensures only trusted, verified container images are deployed to GKE or Cloud Run.

* **How it works:** 

  1. Code is built in Cloud Build.
  
  2. Artifact Analysis scans the image for vulnerabilities.
  
  3. If tests and scans pass, a cryptographic **Attestation** is signed and attached to the image.
  
  4. Binary Authorization checks the cluster policy; if an image lacks a valid attestation from an approved authority, deployment is strictly blocked.

* *Mock Exam Scenario:* "You need to ensure your production GKE cluster automatically rejects any container images that were not built by your approved CI/CD pipeline or that contain CRITICAL vulnerabilities." $\rightarrow$ **Enforce Binary Authorization policies backed by Artifact Analysis vulnerability scans.**

## 3. Progressive Delivery & Release Management (Cloud Deploy)

Cloud Deploy is a managed continuous delivery service that automates the promotion of releases across multiple target environments (e.g., `dev` $\rightarrow$ `staging` $\rightarrow$ `production`).

### 3.1 Deployment Strategies

* **Rolling Update:** Gradually replaces old pods/instances with new ones. Minimizes infrastructure cost but offers less fine-grained user traffic splitting.

* **Canary Deployment:** Routes a small percentage of production traffic (e.g., 5%) to a new version to evaluate error rates and latency in Cloud Monitoring before rolling out to 100%.

* **Blue-Green Deployment:** Maintains two identical production environments. Traffic switches instantly via load balancer configuration. If errors occur, rollback is instantaneous.

* *Exam Scenario:* "You want to test a new version of an app on GKE with 5% of users. If exceptions spike, you need an instant rollback mechanism with minimal steps." $\rightarrow$ **Use Cloud Service Mesh or Gateway API weighted traffic routing for a canary deployment.**

## 4. Local Development & Testing Workflows

### 4.1 The Developer Inner-Loop (Cloud Code + Skaffold)

* **The Problem:** Manually running `docker build`, pushing to Artifact Registry, and updating Kubernetes manifests takes 10–15 minutes per code change, destroying developer productivity.

* **The Solution:** **Cloud Code** integrated with **Skaffold** in your IDE. Skaffold watches your local source files, automatically builds container layers on file save, syncs changes directly to a development cluster, and streams container logs back to your IDE in real time.

### 4.2 Local GCP Emulators

* **Purpose:** To run fast, deterministic, offline unit and integration tests without hitting live cloud APIs or incurring GCP costs.

* **Supported Services:** Pub/Sub, Firestore, Bigtable, Spanner.

* **Implementation:** Set environment variables in your test runner configuration (e.g., `PUBSUB_EMULATOR_HOST=localhost:8085`, `SPANNER_EMULATOR_HOST=localhost:9010`). The official Google Cloud client libraries automatically detect these variables and redirect API calls to the local emulator.

*End of Volume 6: CI/CD, DevOps, and Software Supply Chain.*