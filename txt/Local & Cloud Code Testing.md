## 📘 Study Notes: Local & Cloud Code Testing

Google Cloud heavily tests your ability to optimize the "inner developer loop"—the cycle of writing code, testing it locally, and deploying it without incurring unnecessary cloud costs or waiting for long CI/CD pipelines to finish.

### 1. Google Cloud Emulators (Highly Tested)

Emulators allow you to run GCP services locally on your machine or inside a CI/CD pipeline (like Cloud Build or GitHub Actions) without provisioning real cloud resources or spending money.

* **Supported Emulators:**
* Cloud Pub/Sub, Cloud Spanner, Cloud Bigtable, and Firestore (Datastore mode).


* **How They Work:** You start the emulator (via `gcloud beta emulators ...`) and point your application to it by setting specific environment variables.
* *Example:* Setting `PUBSUB_EMULATOR_HOST=localhost:8085` forces the Google Cloud Client Library to route traffic to your local machine instead of the real GCP API.


* **The Exam Trap:** There is **no official local emulator for Cloud SQL or Cloud Storage**.
* If a question asks how to test Cloud SQL locally, the answer is to run a standard Docker container for MySQL/PostgreSQL, or use the Cloud SQL Auth Proxy to connect your local machine to a Dev database in the cloud.



### 2. Testing Serverless (Cloud Functions & Cloud Run)

You must know how to test serverless code locally before pushing it to GCP.

* **The Functions Framework (CRITICAL):**
* An open-source library provided by Google for Node.js, Python, Go, and Java.
* *Exam Scenario:* You are writing a Cloud Function that triggers on a Cloud Storage event. How do you test this on your laptop without deploying it?
* *The Solution:* Use the **Functions Framework** to spin up a local HTTP server that wraps your function, allowing you to send mock JSON event payloads to `localhost:8080`.


* **Cloud Run Local Testing:**
* Because Cloud Run is just a Docker container listening on a port (usually 8080), local testing simply involves running `docker build` and `docker run`. You do not need a special framework.



### 3. Kubernetes & GKE Development (Cloud Code & Skaffold)

Google expects you to use modern tooling to bridge the gap between local code and Kubernetes clusters.

* **Cloud Code (IDE Extension):**
* A plugin for VS Code and IntelliJ. It provides built-in YAML validation, GCP API integration, and one-click deployments to Cloud Run and GKE.
* *Exam Feature:* It allows you to **step-debug** applications running remotely in a GKE cluster directly from your local IDE.


* **Skaffold (The Continuous Dev Loop):**
* A command-line tool that handles the workflow for building, pushing, and deploying applications to Kubernetes.
* *The Scenario:* A developer complains that testing a microservice requires manually running `docker build`, `docker push`, and `kubectl apply` every time they change a single line of code.
* *The Solution:* Run **`skaffold dev`**. It watches your local file system. When you hit save, Skaffold automatically builds the image, pushes it to Artifact Registry, and deploys it to your Dev GKE cluster or local Minikube in seconds.



### 4. CI/CD Integration Testing

How to test code safely in an automated pipeline.

* **Testcontainers / Emulators in CI:**
* *Exam Scenario:* You want your Cloud Build pipeline to run integration tests for a microservice that writes to Pub/Sub. You want to ensure tests are isolated and don't pollute a shared cloud topic.
* *The Solution:* Configure your CI pipeline to spin up the Pub/Sub emulator as a background container step, run the tests against it, and tear it down.


* **Shadow Testing (Traffic Mirroring):**
* Testing in production with zero risk to the user.
* *How it works (GKE/Istio):* You deploy `v2` of your app. You configure the network to duplicate (mirror) all incoming HTTP traffic going to `v1` and send a copy to `v2`. The user only sees the response from `v1`, but developers can monitor `v2`'s logs and metrics to see how it handles real-world traffic before a full rollout.