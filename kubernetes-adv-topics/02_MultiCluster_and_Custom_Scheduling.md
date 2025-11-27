# 02. Multi-Cluster & Custom Scheduling

## 1. Multi-Cluster Management

Managing applications across multiple Kubernetes clusters (e.g., Hybrid Cloud, DR, Edge).

### Cluster Federation (KubeFed)
- **Concept**: A "Host" cluster manages configuration for multiple "Member" clusters.
- **Federated Resources**: `FederatedDeployment`, `FederatedService`.
- **Status**: KubeFed v2 is the current standard, but complexity is high. Many prefer GitOps (ArgoCD) for multi-cluster config management.

### Multi-Cluster Service Mesh
- **Istio Multicluster**: Connects services across clusters transparently.
- **Submariner**: Connects overlay networks of different clusters (L3 connectivity).

## 2. Custom Scheduling

### Scheduler Extenders
- **Concept**: Webhook callout from the default scheduler to an external service to filter/prioritize nodes.
- **Pros**: Simple to implement.
- **Cons**: Performance overhead (HTTP round trip).

### Custom Schedulers
- **Concept**: Run a second scheduler binary alongside the default one.
- **Usage**: Set `schedulerName: my-scheduler` in Pod spec.
- **Use Case**: Batch jobs, specialized hardware needs.

### Node Feature Discovery (NFD)
- **Problem**: How to schedule pods on nodes with specific hardware (GPU, FPGA, specific CPU flags)?
- **Solution**: NFD runs as a DaemonSet, detects hardware features, and advertises them as **Node Labels**.
- **Example**: `feature.node.kubernetes.io/pci-10de.present=true` (NVIDIA GPU).

## 3. Topology Aware Scheduling

Optimizes placement based on network topology (Zone/Region) to reduce latency and cost.
- **Topology Manager**: Kubelet component that coordinates resource allocation (CPU, Device Manager) to align with NUMA nodes.
- **Service Topology**: Route traffic to endpoints in the same Node -> Zone -> Region.
