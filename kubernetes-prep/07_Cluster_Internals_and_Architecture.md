# 07. Cluster Internals & Architecture

## 1. Kubelet Internals

The Kubelet is the primary "node agent" that runs on each node.

### Key Components
- **SyncLoop**: The main control loop. Checks desired state (from API Server) vs actual state.
- **PLEG (Pod Lifecycle Event Generator)**: Relies on container runtime events to update Pod status.
- **CRI (Container Runtime Interface)**: gRPC interface to talk to the runtime (containerd/CRI-O).
- **CNI (Container Network Interface)**: Configures networking.
- **CSI (Container Storage Interface)**: Mounts volumes.

## 2. Container Runtime Internals

Kubernetes doesn't run containers directly; it uses a runtime.

### CRI-O vs containerd
Both are OCI-compliant runtimes.
- **containerd**: Originally from Docker, now a CNCF graduated project. Industry standard.
- **CRI-O**: Lightweight, built specifically for Kubernetes.

### OCI (Open Container Initiative)
- **Image Spec**: How images are built/stored.
- **Runtime Spec**: How to run a container (namespaces, cgroups). `runc` is the reference implementation used by both containerd and CRI-O.

## 3. API Server Internals

The brain of the cluster. The only component that talks to etcd.

### Request Flow
1.  **Authentication**: Who are you? (Client Certs, Bearer Token, OIDC).
2.  **Authorization**: Can you do this? (RBAC, ABAC, Node).
3.  **Admission Control**:
    - **Mutating**: Modify the request (e.g., add default storage class, inject sidecar).
    - **Validating**: Approve/Deny (e.g., check Pod Security Standards).
4.  **Validation**: Schema validation.
5.  **Persist**: Write to **etcd**.

## 4. High Availability (HA) Cluster Design

To ensure zero downtime for the Control Plane.

### Components
- **etcd**: Distributed key-value store. Needs odd number of nodes (3 or 5) for quorum (Raft consensus).
- **API Server**: Stateless. Can scale horizontally behind a Load Balancer.
- **Scheduler & Controller Manager**: Stateful (leader election). Only one active instance at a time per cluster.

### HA Topology
- **Stacked etcd**: etcd runs on the same nodes as the control plane components. Easier to manage.
- **External etcd**: etcd runs on dedicated nodes. Better isolation and security.
