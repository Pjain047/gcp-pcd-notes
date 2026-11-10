# Google Cloud Professional Cloud Developer

## Deep Dive Volume 4: Google Kubernetes Engine (GKE) Master Notes

GKE is one of the most heavily tested technologies on the Professional Cloud Developer exam. You must understand not only how to deploy containerized workloads, but also how GKE interacts with native Google Cloud infrastructure (such as Cloud Load Balancing, Secret Manager, Filestore, and IAM).

## 1. GKE Architecture & Workload Selection

### 1.1 Cluster Modes: Standard vs. Autopilot

* **GKE Standard:** You manage the worker nodes (Compute Engine VMs). You have root access to nodes, can install custom daemonsets, adjust kernel settings, and configure custom node pools. You pay for the virtual machines whether they run pods or sit idle.

* **GKE Autopilot (Google-Recommended for Most Microservices):** Google manages the underlying node infrastructure, scaling, security hardening, and updates. You only pay for the resources (CPU/RAM) consumed by your pods. 

  * *Exam Rule:* Default to Autopilot unless the scenario explicitly requires custom node pools, GPU attachments, node-level daemonsets, or low-level kernel configuration.

### 1.2 Core Kubernetes Workload Objects

* **Pod:** The smallest deployable unit. Contains one or more tightly coupled containers sharing storage and network namespaces (`localhost`).

* **Deployment:** Manages stateless pods. Ensures a desired replica count and handles **Rolling Updates** seamlessly.

* **StatefulSet:** For stateful applications (databases, distributed systems). Guarantees stable, unique network identifiers (`pod-0`, `pod-1`) and persistent storage binding using `volumeClaimTemplates`.

* **DaemonSet:** Ensures a copy of a pod runs on *all* (or selected) cluster nodes. Used for cluster-level logging agents (Fluent Bit) or monitoring agents (Prometheus node-exporter).

* **Job & CronJob:** Runs pods to completion (batch processing, migrations) or on a scheduled cron-like timeline.

## 2. Pod Lifecycle, Probes, and Initialization

### 2.1 Probes (The Triad of Health)

Understanding when and how Kubernetes probes check container health is vital for zero-downtime deployments.

* **Startup Probe:** Validates whether the application within the container has started up. 

  * *Behavior:* While a startup probe is running, **all other probes (liveness and readiness) are disabled**. If the startup probe fails, Kubernetes kills the container.

  * *Exam Use Case:* Slow-starting applications (e.g., pulling a large machine learning model or database schema migration on boot).

* **Readiness Probe:** Determines whether a pod is ready to accept user traffic.

  * *Behavior:* If a readiness probe fails, the endpoint controller removes the pod's IP address from all matching Kubernetes **Services** and Load Balancers. The pod is *not* restarted; it is simply isolated until it recovers.

* **Liveness Probe:** Determines whether the application is running healthily or is stuck in a deadlock.

  * *Behavior:* If a liveness probe fails, the `kubelet` terminates the container and restarts it based on the pod's restart policy.

  * *Exam Trap:* Never point a liveness probe to a heavy database check. If the database drops momentarily, your entire GKE cluster's pods will trigger restart storms simultaneously.

## 3. Storage Architecture in GKE

Pods are ephemeral; data written to a container's local filesystem is lost when the pod restarts. GKE solves this using CSI (Container Storage Interface) drivers integrated with Google Cloud storage.

* **PersistentVolume (PV):** The cluster-wide storage resource provisioned by an administrator or dynamically via a StorageClass.

* **PersistentVolumeClaim (PVC):** The developer's request for storage (e.g., "I need 50Gi of ReadWriteOnce storage").

* **StorageClasses:** Define the underlying Google Cloud storage backing (e.g., `standard-rwo` for standard Persistent Disks, `premium-rwo` for SSDs, or `enterprise-filestore` for NFS).

* **Shared Filesystem (ReadWriteMany - RWX):**

  * *Exam Scenario:* "You are migrating a monolithic legacy application to microservices on GKE. Multiple pods need simultaneous read and write access to a single configuration file on a shared filesystem without rewriting code."

  * *Solution:* Provision a Google Cloud **Filestore** instance and mount it in your GKE pods using an NFS PersistentVolumeClaim with `accessModes: [ReadWriteMany]`.

## 4. Autoscaling: HPA, VPA, and Cluster Autoscaler

To balance cost and performance, GKE offers three independent autoscaling layers:

* **Horizontal Pod Autoscaler (HPA):** Scales the **number of pod replicas** up or down based on CPU, memory, or custom/external metrics (e.g., Pub/Sub queue backlog).

  * *Requirement:* HPA requires resource `requests` to be defined in the pod specification. Utilization is calculated as `Current Usage / Requested Amount`.

* **Vertical Pod Autoscaler (VPA):** Automatically adjusts the CPU and Memory `requests` and `limits` of running pods based on historical usage.

  * *Exam Trap:* **Never run HPA and VPA on the same metric (such as CPU) simultaneously.** If VPA scales up the CPU request, HPA sees lower utilization percentage and scales down the pods, creating an infinite, unstable oscillation loop.

* **Cluster Autoscaler:** Scales the **number of worker nodes** in the cluster. It triggers automatically when pods remain in a `Pending` state because no node has enough spare capacity to schedule them.

## 5. Advanced Scheduling, Taints, and Tolerations

Kubernetes provides mechanisms to control exactly which node a pod lands on:

* **NodeSelector & Node Affinity:** Force or prefer pods to land on nodes with specific labels (e.g., placing memory-heavy pods on high-RAM nodes).

* **Taints and Tolerations:** The "repel" mechanism. A node can be **tainted** (e.g., `dedicated=gpu:NoSchedule`) so that ordinary pods cannot run on it. A pod can specify a **toleration** matching that taint to run there.

  * *Exam Rule:* A toleration allows a pod to land on a tainted node, but it does *not* force it there. To guarantee placement, combine a taint/toleration with explicit Node Affinity.

* **Pod Anti-Affinity:** Prevents pods of the same application from landing on the same node or zone, ensuring high availability.

## 6. GKE Networking, Services, and Ingress

### 6.1 Service Types

* **ClusterIP (Default):** Exposes the service on an internal IP address reachable only from within the cluster.

* **NodePort:** Exposes the service on a static port (30000–32767) across all cluster node IPs.

* **LoadBalancer:** Provisions a Google Cloud external Network Load Balancer (L4) that routes external traffic to the node ports.

### 6.2 Container-Native Load Balancing (NEGs)

* Traditionally, external load balancers routed traffic to NodePorts, which then bounced traffic via `kube-proxy` to pods. This added network hops and latency.

* **Network Endpoint Groups (NEGs):** GKE creates NEGs that map pod IP addresses directly to the Google Cloud HTTP(S) Application Load Balancer backend service. This achieves true container-native load balancing.

### 6.3 NetworkPolicies

* By default, all pods in a GKE cluster can communicate with all other pods across namespaces.

* **NetworkPolicy:** A Kubernetes resource used to enforce firewall-like restrictions at the pod level (Ingress and Egress).

* *Best Practice:* Apply a **Default-Deny** NetworkPolicy to your production namespaces, and explicitly write allow-rules for authorized microservice paths (e.g., frontend can talk to backend; backend can talk to database).

## 7. GKE Security Best Practices

* **Workload Identity:** The gold standard for GKE security. Maps a Kubernetes Service Account (KSA) to a Google Cloud Service Account (GSA) using IAM token impersonation. **Never use long-lived JSON keys inside containers.**

* **Binary Authorization:** A deploy-time security control that enforces that only container images signed by trusted authorities (attestors, vulnerability scanners) can be deployed to the cluster.

* **Application-layer Secrets Encryption:** Encrypts Kubernetes secrets at rest in `etcd` using a customer-managed key from Cloud KMS.

## 8. The GKE Troubleshooting Ladder (Exam Rapid-Fire Reference)

When troubleshooting GKE scenarios on the exam, follow this diagnostic sequence:

1. **Pod is `Pending`:**
   * Run `kubectl describe pod`. Check for insufficient CPU/Memory `requests`, unmet node affinity rules, unmatched taints/tolerations, or unresolvable PersistentVolumeClaims.

2. **Pod is in `CrashLoopBackOff`:**
   * Run `kubectl logs <pod-name> --previous`. Check application startup errors, missing configuration variables, or failing startup probes.

3. **Service has No Endpoints (`kubectl get endpoints`):**
   * Check if the Service `selector` labels match the Pod's `labels` precisely. Also verify that the Pod's `readinessProbe` is passing.

4. **External Load Balancer returns `503 Service Unavailable`:**
   * Verify that Container-Native NEGs are healthy, the Cloud Load Balancer health check path matches an active application endpoint, and firewall rules/Cloud Armor policies aren't blocking traffic.

*End of Volume 4: Google Kubernetes Engine (GKE) Master Notes.*