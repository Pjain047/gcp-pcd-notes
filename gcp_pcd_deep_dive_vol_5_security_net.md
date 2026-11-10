# Google Cloud Professional Cloud Developer

## Deep Dive Volume 5: Security, IAM, Networking, and Encryption

Security and networking make up a substantial portion of the exam. Google Cloud operates on a "Defense in Depth" model. You must know how to apply the principle of least privilege, authenticate applications without storing long-lived credentials, protect perimeters against data exfiltration, and establish secure hybrid networking paths.

---

## 1. Identity and Access Management (IAM)

IAM answers three fundamental questions: **Who** can do **What** on **Which** resource?

### 1.1 IAM Components
* **Principal (Who):** A Google Account, a Google Group, a Service Account, or a Domain (e.g., `user@company.com`, `my-sa@project.iam.gserviceaccount.com`).
* **Role (What):** A collection of permissions (e.g., `roles/storage.objectViewer`, `roles/run.invoker`).
    * *Predefined Roles:* Managed by Google, granular, fine-tuned for specific services. Always prefer predefined over basic roles.
    * *Basic Roles (Legacy):* Owner, Editor, Viewer. **Exam Trap:** Never choose basic roles for production applications or service accounts. They are far too broad (e.g., Editor grants write access across almost every service in the project).
    * *Custom Roles:* Created when predefined roles do not meet your exact requirements.
* **Binding (The Link):** Attaches one or more Principals to a Role on a specific Resource (Project, Folder, Organization, or individual bucket/dataset).

### 1.2 Service Accounts & Keyless Authentication
* **Service Accounts:** Special accounts used by applications, workloads, or VMs rather than humans.
* **Application Default Credentials (ADC):** A strategy used by Google Cloud client libraries to automatically discover runtime credentials. 
    * When running on GCP (Compute Engine, Cloud Run, GKE), ADC automatically picks up the attached service account or workload identity without requiring local environment variables.
* **Workload Identity Federation (External Workloads):**
    * *Scenario:* Your CI/CD pipeline running in **GitHub Actions**, AWS, or on-premises needs to deploy to GCP or push images to Artifact Registry.
    * *Solution:* Configure **Workload Identity Federation**. This allows external systems to authenticate via OIDC token exchange and assume a Google Service Account temporarily, **eliminating the need to download and store long-lived JSON service account keys in GitHub repository secrets**.
* **Service Account Impersonation:**
    * Allows a user or service account with `roles/iam.serviceAccountTokenCreator` to temporarily borrow the identity and permissions of another service account without holding a static key file.

---

## 2. API Security & Front-Door Access

### 2.1 Identity-Aware Proxy (IAP)
* **Purpose:** Secures web applications and virtual machines by establishing a central authentication and authorization layer.
* **How it works:** When a user attempts to access an application behind an external HTTP(S) Load Balancer, IAP intercepts the request, prompts the user to sign in with Google (or corporate SSO), checks their IAM permissions (`roles/iap.httpsResourceAccessor`), and only forwards traffic if authorized.
* **Exam Scenario:** "You are hosting a web application on GKE behind a global load balancer with SSL. You do not have time to write custom login code in your application, but you want to restrict access to authorized employees." $\rightarrow$ **Deploy IAP on the Load Balancer backend.**

### 2.2 VPC Service Controls (VPC-SC)
* **Purpose:** Prevents **data exfiltration** from Google Cloud services.
* **The Problem:** IAM controls *who* has access to a resource. However, an authorized user or compromised service account with `roles/bigquery.admin` or `roles/storage.admin` can still copy sensitive enterprise data out of your project to an unauthorized public bucket or external project.
* **The Solution:** **VPC Service Controls** creates a cryptographic **Service Perimeter** around Google Cloud APIs (BigQuery, Cloud Storage, Cloud SQL, Secret Manager). Requests crossing the perimeter boundary are blocked, regardless of IAM permissions, unless explicitly allowed via Access Levels or Ingress/Egress rules.

---

## 3. Cryptography & Key Management (Cloud KMS)

### 3.1 Cloud KMS & CMEK
* **Cloud KMS (Key Management Service):** A cloud-hosted service that lets you manage cryptographic keys and use them for cryptographic operations.
* **CMEK (Customer-Managed Encryption Keys):** By default, Google encrypts data at rest using Google-managed keys. With CMEK, you create and control the encryption keys inside Cloud KMS for supported services (Cloud Storage, BigQuery, Compute Engine disks).
* **Key Hierarchy & Envelope Encryption:**
    1. **Data Encryption Key (DEK):** Encrypts the actual application data.
    2. **Key Encryption Key (KEK):** Encrypts (wraps) the DEK. Managed in Cloud KMS.
    3. *Process:* Data is encrypted locally by the app/service using a DEK. The DEK is then wrapped by the KEK in KMS. Only ciphertext + wrapped DEK are stored. To decrypt, KMS unwraps the DEK, and the payload is decrypted.

### 3.2 Secret Manager
* **Purpose:** Centralized secure storage for sensitive strings: database passwords, API keys, TLS certificates, and private tokens.
* **Features:** Versioning, automatic rotation triggers (via Pub/Sub and Cloud Run Functions), and fine-grained IAM access control per secret.
* **Exam Rule:** Never store secrets in plaintext environment variables in YAML files or bake them into container image layers. Fetch them securely at runtime from **Secret Manager**.

---

## 4. Virtual Private Cloud (VPC) Networking & Hybrid Connectivity

### 4.1 VPC Design Patterns
* **Custom Mode VPC:** Recommended for production. You explicitly define your own subnets across regions, giving you full control over IP address ranges.
* **Shared VPC:** Allows an organization to share subnets from a central host VPC project across multiple service projects, enabling centralized network administration while maintaining workload separation.
* **VPC Network Peering:** Connects two separate VPC networks privately using internal IP addresses, regardless of whether they belong to the same project or organization.

### 4.2 Hybrid Connectivity (On-Premises to Cloud)
* **HA VPN (High Availability VPN):** Encrypted IPsec VPN connection over the public internet providing a 99.99% service availability SLA.
* **Cloud Interconnect:**
    * *Dedicated Interconnect:* A direct, high-speed, physical private connection between your on-premises data center and Google’s network (bypasses the public internet entirely).
    * *Partner Interconnect:* Connects through a supported supported third-party network service provider.
* **Cloud Router:** Dynamically exchanges routing information between your VPC and your on-premises network using BGP (Border Gateway Protocol). Used alongside VPN and Interconnect.

### 4.3 Egress and API Access Controls
* **Cloud NAT (Network Address Translation):** Allows private VMs and GKE nodes (which lack public IP addresses) to initiate outbound connections to the internet (e.g., to download software packages or security patches) while remaining completely protected from inbound internet traffic.
* **Private Google Access:** Allows instances with only private IP addresses (no public IPs) to reach Google APIs and services (like Cloud Storage, BigQuery, Pub/Sub) over internal Google IP addresses rather than traversing the public internet.
* **Private Service Connect (PSC):** Allows private consumption of services across VPCs or Google-managed services (like Cloud SQL or partner APIs) using internal IP addresses.

---

*End of Volume 5: Security, IAM, Networking, and Encryption.*