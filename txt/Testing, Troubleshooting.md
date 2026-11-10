## 📘 Study Notes: Testing, Troubleshooting, & Deployment Strategies

Google Cloud expects a Professional Cloud Developer to know how to safely roll out new features without downtime, manage infrastructure as code (IaC), test applications locally, and quickly diagnose production failures.

### 1. Deployment Strategies (Canary & Blue/Green)

The exam heavily tests how to perform zero-downtime rollouts across different serverless and container platforms.

* **Cloud Run Traffic Splitting (Highly Tested):**
* *The Safe Rollout:* Deploy the new version without routing any immediate traffic to it using `gcloud run deploy --no-traffic`.
* *The Canary Command:* Slowly shift traffic using the `update-traffic` command.
* *Exam Trap:* Do NOT use `--set-env-vars` or `update` to shift traffic. The exact command is: `gcloud run services update-traffic [SERVICE_NAME] --to-revisions=[REVISION_NAME]=10`


* **GKE Canary Deployments (Native Kubernetes):**
* *Without a Service Mesh:* If the question asks for a native Kubernetes way to do a 10% canary rollout without Istio, the answer is **Replica Math**.
* *The Solution:* Run two separate deployments (e.g., `v1` with 9 replicas and `v2` with 1 replica). Ensure both share the exact same `matchLabels` so a single Kubernetes `Service` routes traffic across all 10 pods, naturally resulting in a 90/10 split.


* **App Engine Traffic Splitting:**
* App Engine allows splitting traffic by **IP Address**, **Cookie**, or **Random**.
* *Exam Scenario:* If you are testing a new shopping cart feature and want to ensure a single user stays on the *same* version of the app for their entire session, split traffic by **Cookie** or **IP Address** (not Random).



### 2. Infrastructure as Code (Terraform & Kustomize)

Modern DevOps heavily relies on declarative configurations.

* **Terraform State Management (CRITICAL):**
* *The Problem:* Multiple CI/CD pipelines or developers running `terraform apply` concurrently can corrupt the state file.
* *The Solution:* Store the Terraform state remotely in a **Cloud Storage bucket (`gcs` backend)**.
* *Exam Trick:* To protect against accidental state corruption or deletion, you must enable **Object Versioning** on that Cloud Storage bucket.


* **Kustomize for Multi-Environment GKE:**
* *The Problem:* You have Dev, Staging, and Prod environments. You don't want to duplicate hundreds of lines of YAML for each.
* *The Solution:* Use a `base` directory for the common configuration. Create an **`overlays/prod`** directory containing a `kustomization.yaml` file that references `../../base` and applies specific production patches (like scaling replicas up to 10).



### 3. Local Testing & Emulators

You do not need a live Google Cloud Project to run automated integration tests in your CI/CD pipeline.

* **Google Cloud Local Emulators:**
* Google provides official local emulators for **Cloud Pub/Sub, Cloud Spanner, Cloud Bigtable, and Firestore/Datastore**.
* *Exam Scenario:* Your Cloud Build pipeline needs to run integration tests for a microservice that writes to Pub/Sub. How do you do this securely and cheaply without creating real topics?
* *The Solution:* Spin up the **Pub/Sub Emulator** as a background service within the CI/CD pipeline, point your application's `PUBSUB_EMULATOR_HOST` environment variable to it, and run the tests.



### 4. Rapid Troubleshooting Guide (Exam Cheat Sheet)

When a scenario describes an error, look for these specific keywords:

* **"My GKE application takes 3 minutes to process a report, but users get an HTTP `504 Gateway Timeout` after 30 seconds."**
* *Fix:* Create a **`BackendConfig`** Custom Resource Definition (CRD), set `timeoutSec: 600`, and attach it to the GKE Service.


* **"My application receives `DefaultCredentialsError` or `403 Forbidden` when trying to access Cloud Storage from GKE."**
* *Fix:* Check **Workload Identity**. Ensure the Kubernetes Service Account (KSA) is annotated with the Google Service Account (GSA) email, and the IAM binding `roles/iam.workloadIdentityUser` exists.


* **"During a traffic spike, my Cloud SQL database crashes with 'Too Many Connections'."**
* *Fix:* **PgBouncer** or a database connection pooler. Do *not* just increase max connections or infinite container scaling limits.


* **"My Cloud Run application is taking 5+ seconds to process the very first HTTP request after being idle."**
* *Fix:* This is a **Cold Start**. Set `--min-instances=1` (or higher) to keep a warm container ready in memory.



---