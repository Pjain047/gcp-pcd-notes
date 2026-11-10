## 📘 Study Notes: GKE Networking, Identity, & Security

Google Cloud heavily tests your ability to secure GKE clusters using the principle of least privilege, specifically focusing on eliminating long-lived credentials and securing network boundaries.

### 1. Workload Identity (The #1 GKE Security Topic)

Workload Identity is the Google-recommended way for applications running inside GKE to authenticate to Google Cloud services (like Cloud Storage, Spanner, or BigQuery).

* **The Core Concept:** It maps a Kubernetes Service Account (KSA) to a Google Service Account (GSA). The pod authenticates transparently without needing any downloaded keys.
* **The 3-Step Setup (Memorize this sequence):**
1. Create a GSA and grant it GCP IAM roles (e.g., `roles/storage.objectViewer`).
2. Create a KSA in your cluster namespace, and **annotate it**: `kubectl annotate ksa [KSA_NAME] iam.gke.io/gcp-service-account=[GSA_EMAIL]`.
3. Create an IAM policy binding that grants the KSA permission to impersonate the GSA using the role `roles/iam.workloadIdentityUser`.


* **Exam Traps:**
* *Never* choose an option that involves downloading a JSON Service Account key and storing it in a Kubernetes Secret.
* *Never* choose an option that grants broad permissions to the default Compute Engine Node Service Account (which affects all pods on the node).



### 2. GKE Networking & Load Balancing

The exam tests how traffic reaches your pods from the outside world efficiently.

* **Container-Native Load Balancing (NEGs):**
* *Concept:* Traditionally, traffic hits a Node IP, and `kube-proxy` routes it to a Pod (adding a network hop and masking the client IP). Container-Native Load Balancing uses **Network Endpoint Groups (NEGs)** to route traffic from the external Load Balancer *directly* to the Pod IP.
* *Trigger:* Enabled automatically on GKE Ingress, or manually by annotating a Kubernetes Service with `[cloud.google.com/neg](https://cloud.google.com/neg)`.


* **BackendConfig (CRITICAL):**
* *Concept:* A Custom Resource used to configure Google Cloud Load Balancer features that aren't natively supported by standard Kubernetes Ingress.
* *Exam Scenarios:* Use `BackendConfig` to extend HTTP timeouts (`timeoutSec: 600`), attach **Cloud Armor** security policies, or enable **Identity-Aware Proxy (IAP)** for internal dashboards.



### 3. Kubernetes NetworkPolicies (Pod-to-Pod Firewalls)

NetworkPolicies act as Layer 4 firewalls defining which pods can talk to each other.

* **Default Deny Trap:** A security best practice is to implement a "Default Deny All" policy for a namespace.
* **The Exam Trick:** If you apply a default-deny Egress policy, your pods will instantly lose the ability to resolve domain names. You **must** create an explicit Egress rule allowing UDP/TCP on Port 53 to the `kube-dns` or `CoreDNS` pods, plus explicitly allow outbound traffic to your external APIs.

### 4. Private GKE Clusters & API Access

A Private GKE Cluster ensures worker nodes only have internal RFC 1918 IP addresses (no public internet routing).

* **Control Plane Access:** In a private cluster, the Kubernetes API server (control plane) is also locked down.
* **Exam Scenario:** A developer working from home needs to run `kubectl get pods`. How do they connect?
* *Solution:* Enable **Master Authorized Networks** to whitelist specific IP CIDR blocks. Connect via a corporate VPN/Cloud Interconnect, or use a private-to-public **IAP Bastion Host** (Jump Server) deployed inside the VPC.



### 5. Binary Authorization (Supply Chain Security)

Binary Authorization is a deploy-time security control (an admission controller) that ensures only trusted container images are deployed on GKE.

* **Attestors:** Cryptographic entities that verify an image has completed a required pipeline step.
* **Exam Workflow:** Code is pushed -> Cloud Build runs tests -> Cloud Build signs the image using an **Attestor** -> GKE Binary Authorization checks for this exact signature before allowing the pod to start. If unsigned, the deployment is blocked.

---