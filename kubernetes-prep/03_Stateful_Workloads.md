# 03. Stateful Workloads & Patterns

## 1. Deployment vs StatefulSet

| Feature | Deployment | StatefulSet |
| :--- | :--- | :--- |
| **Primary Use Case** | Stateless apps (Web Servers, APIs) | Stateful apps (Databases, Kafka, Zookeeper) |
| **Pod Identity** | Random hash (e.g., `web-789d7c9-xyz`) | Sticky, ordinal index (e.g., `web-0`, `web-1`) |
| **Network Identity** | Service IP (ClusterIP) | Stable Hostname (`web-0.service.ns`) |
| **Storage** | Shared or Ephemeral | Stable, Persistent per Pod (`volumeClaimTemplates`) |
| **Scaling** | Parallel (all at once) | Ordered (0->1->2) and Graceful (2->1->0) |

## 2. StatefulSet Deep Dive

### Stable Network Identity
StatefulSets require a **Headless Service** to control the domain of its Pods.
- A Headless Service has `clusterIP: None`.
- DNS returns A records for *each* Pod IP, not a single VIP.

### Stable Storage
StatefulSets use `volumeClaimTemplates` to create a unique PersistentVolumeClaim (PVC) for each Pod.
- If `web-0` dies and is rescheduled, it reattaches to `pvc-web-0`.
- PVCs are **not** deleted when the StatefulSet is deleted (data safety).

### Example Configuration
```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  ports:
  - port: 80
    name: web
  clusterIP: None # Headless Service
  selector:
    app: nginx
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
spec:
  serviceName: "nginx"
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: k8s.gcr.io/nginx-slim:0.8
        ports:
        - containerPort: 80
          name: web
        volumeMounts:
        - name: www
          mountPath: /usr/share/nginx/html
  volumeClaimTemplates:
  - metadata:
      name: www
    spec:
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: 1Gi
```

## 3. Advanced Stateful Patterns

### Leader Election
Since StatefulSets provide stable identities (`web-0`), you can hardcode logic:
- `web-0` is always the Writer/Master.
- `web-1`, `web-2` are Readers/Slaves.
- Sidecars can be used to automate failover if `web-0` goes down.

### Rolling Updates
StatefulSets support `RollingUpdate` strategy.
- **Partition**: You can stage an update. If `replicas=3` and `partition=2`, only `web-2` is updated. `web-0` and `web-1` remain on the old version. Useful for canarying stateful apps.

### Pod Management Policy
- **OrderedReady** (Default): Sequential startup/teardown.
- **Parallel**: Launch/terminate all pods simultaneously (like a Deployment). Useful if state is managed externally (e.g., Cassandra/Elasticsearch handles its own clustering).
