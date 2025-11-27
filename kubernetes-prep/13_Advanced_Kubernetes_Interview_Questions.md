# 13. Advanced Kubernetes Interview Questions

This guide focuses on **scenario-based** questions that test deep understanding, rather than simple definition recall.

## 1. Troubleshooting & Operations

**Q1: A Pod is in `CrashLoopBackOff`. How do you debug it?**
*   **Answer**:
    1.  `kubectl describe pod <name>`: Check events (OOMKilled? Liveness probe failed?).
    2.  `kubectl logs <name>`: Check app logs for stack traces or startup errors.
    3.  `kubectl logs <name> --previous`: Check logs of the *previous* crashed instance.
    4.  **Scenario**: If logs are empty, it might be a command/entrypoint issue. Debug with `kubectl debug` or override entrypoint.

**Q2: A developer complains their service is unreachable. Walk me through your debugging flow.**
*   **Answer**: "Follow the packet":
    1.  **Pod Level**: Is the Pod Running and Ready? (`kubectl get pods`).
    2.  **Service Level**: Does the Service exist? Does it have Endpoints? (`kubectl get ep`). If no endpoints, check Selector labels.
    3.  **DNS Level**: Can I resolve the service name from another pod? (`nslookup my-service`).
    4.  **Network Policy**: Is there a `NetworkPolicy` blocking traffic?
    5.  **Ingress**: If external, check Ingress Controller logs and rules.

**Q3: A Node is marked as `NotReady`. What could be the reasons?**
*   **Answer**:
    1.  **Kubelet stopped**: Check systemd status (`systemctl status kubelet`).
    2.  **Disk Pressure**: Node is out of disk space. Kubelet stops accepting pods.
    3.  **Network Partition**: Node cannot reach API Server.
    4.  **CNI Plugin failure**: Check CNI logs.

## 2. Architecture & Internals

**Q4: What happens when I run `kubectl run nginx --image=nginx`? (The "Life of a Pod")**
*   **Answer**:
    1.  **kubectl**: Validates command, sends HTTP POST to API Server.
    2.  **API Server**: AuthN -> AuthZ -> Admission Control (Mutating/Validating) -> Writes to **etcd**.
    3.  **Scheduler**: Watches for new Pods (unbound). Selects best Node (Filtering/Scoring). Updates API Server (binds Pod to Node).
    4.  **Kubelet (on Node)**: Watches for Pods bound to itself.
    5.  **CRI**: Kubelet tells Container Runtime (containerd) to pull image and start container.
    6.  **CNI**: Kubelet tells CNI plugin to configure networking (IP address).
    7.  **Status**: Kubelet updates Pod status to "Running" via API Server.

**Q5: We lost a Control Plane node in a 3-node cluster. What is the impact?**
*   **Answer**:
    *   **Availability**: Cluster remains functional (Quorum is 2/3). API Server works.
    *   **Risk**: If another node fails, quorum is lost (1/3), and the cluster becomes Read-Only (etcd cannot write).
    *   **Action**: Replace the failed node immediately to restore redundancy.

## 3. Networking & CNI

**Q6: Explain how Pod-to-Pod communication works across different nodes.**
*   **Answer**:
    *   **Requirement**: K8s mandates a flat network (no NAT between pods).
    *   **CNI Role**: CNI plugin (Calico/Cilium) configures routing.
    *   **Encapsulation (Overlay)**: VXLAN/IPIP tunnels wrap packets. Node A wraps packet -> Physical Network -> Node B unwraps -> Pod B.
    *   **Direct Routing (No Overlay)**: BGP (Calico) advertises Pod CIDRs to physical routers. Faster, but requires network support.

**Q7: Why would you choose Cilium over Calico?**
*   **Answer**:
    *   **Performance**: Cilium uses eBPF to bypass parts of the iptables/TCP stack.
    *   **Observability**: Hubble provides deep L7 visibility (HTTP/DNS) without sidecars.
    *   **Scale**: Iptables rules degrade performance at high scale (thousands of services). eBPF uses hashmaps (O(1) lookup).

## 4. Security

**Q8: How do you prevent a Pod from running as Root?**
*   **Answer**:
    *   **Pod Security Context**: Set `runAsUser: 1000`, `runAsNonRoot: true`.
    *   **Enforcement**: Use **Pod Security Admission (PSA)** at the Namespace level (`pod-security.kubernetes.io/enforce: restricted`) or a policy engine like **Kyverno/OPA**.

**Q9: I have a secret in Git. Is it safe?**
*   **Answer**: No. K8s Secrets are just base64 encoded.
    *   **Solution**: Use **External Secrets Operator** to sync from Vault/AWS Secrets Manager, or use **Sealed Secrets** (encrypts secret into a CRD that only the cluster controller can decrypt).

## 5. Scaling & Storage

**Q10: Can I use HPA and VPA together?**
*   **Answer**: Generally **No**, if both scale on CPU/Memory. They will fight (HPA adds pods because CPU is high; VPA increases CPU request because CPU is high).
    *   **Exception**: You *can* use VPA in "Off" mode (recommendations only) or use HPA on *custom metrics* (e.g., QPS) while VPA manages CPU/Mem.

**Q11: How does a StatefulSet ensure data safety during a node failure?**
*   **Answer**:
    *   **Sticky Identity**: Pod `web-0` is always associated with `pvc-web-0`.
    *   **No Force Reschedule**: If a node is "Unknown" (partitioned), K8s will NOT reschedule the StatefulSet pod until the old pod is confirmed dead (deleted). This prevents "Split Brain" (two pods writing to the same volume). You might need to force delete the pod manually if the node is permanently dead.
