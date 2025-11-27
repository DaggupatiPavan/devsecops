# 04. Networking, CNI & Policies

## 1. CNI Deep Dive (Container Network Interface)

CNI is the specification that defines how plugins configure networking for containers.

### How CNI Works
1.  **Kubelet** calls the CNI plugin when a Pod is created.
2.  **CNI Plugin** (e.g., Cilium, Calico) allocates an IP address (via IPAM) and attaches the container interface to the network (e.g., via veth pairs).
3.  **Kubelet** calls CNI to clean up when the Pod is deleted.

### CNI Comparison: Cilium vs Calico

| Feature | Calico | Cilium |
| :--- | :--- | :--- |
| **Data Plane** | Linux Bridge + Iptables (Standard) or eBPF (Newer) | eBPF (Extended Berkeley Packet Filter) |
| **Performance** | Good (Standard Linux Networking) | Excellent (Bypasses parts of TCP/IP stack) |
| **Network Policy** | Standard + Extended (GlobalNetworkPolicy) | Standard + Extended (L7 HTTP/DNS Policy) |
| **Observability** | Basic (Flow logs) | Advanced (Hubble - L7 visibility) |
| **Encryption** | WireGuard / IPsec | WireGuard / IPsec (Transparent) |

### Why Cilium? (eBPF)
Cilium uses eBPF to program the kernel dynamically. It can filter packets at the socket layer, providing high-performance load balancing and deep visibility (L7) without sidecars for some use cases.

## 2. Network Policies

Network Policies act as a firewall for Pods.
- **Default Behavior**: All Pods can talk to all Pods (Allow All).
- **Isolation**: Once a NetworkPolicy selects a Pod, it rejects all traffic not explicitly allowed.

### 3-Tier Application Example

**Scenario**:
- **Frontend**: Accessible from anywhere (Ingress). Can talk to Backend.
- **Backend**: Accessible ONLY from Frontend. Can talk to DB.
- **DB**: Accessible ONLY from Backend. Cannot talk to internet.

#### 1. Default Deny All (Best Practice)
Start by blocking everything in the namespace.
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
```

#### 2. Frontend Policy
Allow Ingress from anywhere (or specific Ingress Controller CIDR) and Egress to Backend.
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: frontend-policy
spec:
  podSelector:
    matchLabels:
      app: frontend
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - {} # Allow all ingress (e.g. from LoadBalancer)
  egress:
  - to:
    - podSelector:
        matchLabels:
          app: backend
    ports:
    - protocol: TCP
      port: 8080
  - to: # Allow DNS
    - namespaceSelector: {}
      podSelector:
        matchLabels:
          k8s-app: kube-dns
    ports:
    - protocol: UDP
      port: 53
```

#### 3. Backend Policy
Allow Ingress ONLY from Frontend. Allow Egress to DB.
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-policy
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend
    ports:
    - protocol: TCP
      port: 8080
  egress:
  - to:
    - podSelector:
        matchLabels:
          app: db
    ports:
    - protocol: TCP
      port: 5432
```

#### 4. Database Policy
Allow Ingress ONLY from Backend. No Egress (except maybe backup).
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: db-policy
spec:
  podSelector:
    matchLabels:
      app: db
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: backend
    ports:
    - protocol: TCP
      port: 5432
```
