# 02. Advanced Scheduling & Node Management

## 1. Taints and Tolerations

Taints allow a **Node** to repel a set of Pods. Tolerations allow a **Pod** to be scheduled onto a Node with matching taints.

### Taint Effects
- **NoSchedule**: Pod will not be scheduled on the node unless it has a matching toleration.
- **PreferNoSchedule**: System will try to avoid placing the pod on the node, but it's not guaranteed.
- **NoExecute**: Pod will be evicted from the node if it is already running, unless it has a matching toleration.

### Example
**Taint a Node:**
```bash
kubectl taint nodes node1 key=value:NoSchedule
```

**Pod Toleration:**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
  - name: nginx
    image: nginx
  tolerations:
  - key: "key"
    operator: "Equal"
    value: "value"
    effect: "NoSchedule"
```

## 2. Affinity and Anti-Affinity

Affinity expands on `nodeSelector` by allowing more expressive rules.

### Node Affinity
Constrains which nodes your Pod is eligible to be scheduled on, based on node labels.

- **requiredDuringSchedulingIgnoredDuringExecution**: Hard rule. Must be met.
- **preferredDuringSchedulingIgnoredDuringExecution**: Soft rule. Scheduler tries to find a matching node.

```yaml
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: kubernetes.io/e2e-az-name
            operator: In
            values:
            - e2e-az1
            - e2e-az2
```

### Pod Affinity & Anti-Affinity
Constrains which nodes your Pod is scheduled on based on **labels of Pods already running on that node** (or other topology domain).

**Use Case: Co-location (Affinity)**
Keep frontend and backend pods in the same availability zone.

**Use Case: High Availability (Anti-Affinity)**
Ensure replicas of the same app are NOT on the same node.

```yaml
spec:
  affinity:
    podAntiAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchExpressions:
          - key: app
            operator: In
            values:
            - web-store
        topologyKey: "kubernetes.io/hostname"
```

## 3. Topology Aware Scheduling

Topology Aware Hints (beta in recent versions) allow the control plane to guide traffic to endpoints in the same zone as the client, reducing latency and cross-zone costs.

It relies on the `topology.kubernetes.io/zone` label.

## 4. Node Management Operations

Essential commands for cluster maintenance.

### Cordon
Marks a node as unschedulable. No new pods will be scheduled there. Existing pods remain.
```bash
kubectl cordon <node-name>
```

### Uncordon
Marks a node as schedulable again.
```bash
kubectl uncordon <node-name>
```

### Drain
Safely evicts all pods from a node to prepare for maintenance (e.g., kernel upgrade).
- Respects `PodDisruptionBudgets`.
- Ignores DaemonSets (by default).

```bash
# Evict pods, ignore DaemonSets, and delete local data if necessary
kubectl drain <node-name> --ignore-daemonsets --delete-emptydir-data
```
