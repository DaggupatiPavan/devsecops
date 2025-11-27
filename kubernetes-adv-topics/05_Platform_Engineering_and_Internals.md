# 05. Platform Engineering & Internals

## 1. Advanced Helm Chart Design

Helm is the package manager for Kubernetes.

### Library Charts
- **Concept**: A chart that defines shared templates (e.g., a standard Deployment template) but deploys no resources itself.
- **Usage**: Application charts depend on the library chart and re-use the templates to ensure standardization (DRY).

### Umbrella Charts
- **Concept**: A chart that has no templates of its own, but lists many other charts as dependencies.
- **Usage**: Deploy a full stack (App + DB + Redis + Ingress) with a single `helm install`.

## 2. Network Load Balancing Internals

How does `type: LoadBalancer` actually work?

### Cloud Controller Manager (CCM)
- The CCM watches for Services with `type: LoadBalancer`.
- It calls the Cloud Provider API (AWS ELB, Azure LB) to provision a physical LB.
- It configures the LB to send traffic to the NodePorts of the cluster nodes.

### MetalLB (On-Prem)
- **Layer 2 Mode**: One node attracts traffic for the VIP (ARP/NDP).
- **BGP Mode**: All nodes advertise the VIP to the upstream router via BGP. True load balancing.

## 3. EKS Internals (AWS)

### Control Plane
- Managed by AWS. Hidden from the user.
- Runs across 3 AZs for HA.
- **VPC CNI**: Pods get real VPC IP addresses directly. High performance, but consumes IP space.

### Data Plane (Nodes)
- **Managed Node Groups**: AWS manages the EC2 instances (updates, scaling).
- **Fargate**: Serverless nodes. You don't see the EC2 instance at all.

## 4. Debugging with Ephemeral Containers

How to debug a distroless container (no shell, no curl)?

### `kubectl debug`
Launches a temporary container in the *same Pod namespaces* (Process, Network, IPC) as the target container.

```bash
# Attach a debug container with a shell and tools
kubectl debug -it my-pod --image=nicolaka/netshoot --target=my-container
```
- **Netshoot**: A popular image containing tcpdump, curl, dig, etc.
- **Target**: The container you want to troubleshoot (shares Process ID namespace).
