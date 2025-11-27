# Kubernetes Advanced Study Guide

This guide provides a recommended learning path to master the advanced Kubernetes concepts documented in this folder. Follow this order to build your knowledge logically.

## Phase 1: Core Workloads & Scheduling
*Start here to understand how workloads run and where they go.*

1.  **[01_Pod_Spec_and_Security.md](01_Pod_Spec_and_Security.md)**
    *   **Why**: The Pod is the atomic unit. You must understand its lifecycle, probes, and security context before anything else.
2.  **[02_Advanced_Scheduling_and_Node_Management.md](02_Advanced_Scheduling_and_Node_Management.md)**
    *   **Why**: Once you have a Pod, you need to control *where* it runs (Taints, Affinity).
3.  **[08_Deployment_Strategies.md](08_Deployment_Strategies.md)**
    *   **Why**: Learn how to update stateless applications with zero downtime (Rolling, Blue/Green).
4.  **[03_Stateful_Workloads.md](03_Stateful_Workloads.md)**
    *   **Why**: Moving beyond stateless, learn how to manage databases and stateful apps (StatefulSets).
5.  **[09_Storage_Deep_Dive.md](09_Storage_Deep_Dive.md)**
    *   **Why**: Stateful apps need storage. Understand PVs, PVCs, and CSI.

## Phase 2: Networking & Security
*Connect your workloads and secure them.*

6.  **[04_Networking_CNI_and_Policies.md](04_Networking_CNI_and_Policies.md)**
    *   **Why**: Understand how Pods talk to each other (CNI) and how to restrict that traffic (Network Policies).
7.  **[05_Ingress_and_Service_Mesh.md](05_Ingress_and_Service_Mesh.md)**
    *   **Why**: Expose your apps to the outside world (Ingress) and manage advanced traffic flows (Service Mesh).
8.  **[10_RBAC_and_Security_Hardening.md](10_RBAC_and_Security_Hardening.md)**
    *   **Why**: Secure the cluster itself. Control who can do what (RBAC) and enforce policies (OPA/Kyverno).

## Phase 3: Operations & Scaling
*Day 2 operations: Scaling and Monitoring.*

9.  **[06_Autoscaling_Deep_Dive.md](06_Autoscaling_Deep_Dive.md)**
    *   **Why**: Automate resource management with HPA, VPA, and Cluster Autoscaler.
10. **[12_Observability_and_Logging.md](12_Observability_and_Logging.md)**
    *   **Why**: You can't fix what you can't see. Master Metrics, Logs, and Tracing.

## Phase 4: Advanced Architecture & Extensions
*Deep dives and extending the platform.*

11. **[11_CRDs_Operators_and_GitOps.md](11_CRDs_Operators_and_GitOps.md)**
    *   **Why**: Extend Kubernetes with custom logic (Operators) and manage it via Git (GitOps).
12. **[07_Cluster_Internals_and_Architecture.md](07_Cluster_Internals_and_Architecture.md)**
    *   **Why**: The deepest dive. Understand how the Kubelet, API Server, and etcd actually work under the hood.

## Phase 5: Review
*Test your knowledge.*

13. **[13_Advanced_Kubernetes_Interview_Questions.md](13_Advanced_Kubernetes_Interview_Questions.md)**
    *   **Why**: Apply everything you've learned to scenario-based interview questions.
