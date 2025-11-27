# 01. Pod Spec Deep Dive & Security Standards

## 1. Pod Lifecycle & Conditions

Understanding the Pod lifecycle is crucial for debugging and designing resilient applications.

### Phases
- **Pending**: Pod accepted by API server, but container image creation/download is pending or scheduling is in progress.
- **Running**: Pod bound to a node, all containers created. At least one is running, starting, or restarting.
- **Succeeded**: All containers terminated successfully (exit code 0) and will not restart.
- **Failed**: All containers terminated, at least one with failure (non-zero exit).
- **Unknown**: State cannot be obtained (usually node communication error).

### Pod Conditions
Check these via `kubectl describe pod <name>`:
- **PodScheduled**: Pod has been scheduled to a node.
- **Initialized**: All `initContainers` have completed successfully.
- **ContainersReady**: All containers in the Pod are ready.
- **Ready**: Pod is able to serve requests (passed readiness probes).

## 2. Container Probes (Health Checks)

Kubernetes uses probes to determine container health.

### Types of Probes
1.  **Liveness Probe**:
    - **Purpose**: "Is the container running?"
    - **Action if failed**: Kills and restarts the container.
    - **Use Case**: Deadlocks, infinite loops.
2.  **Readiness Probe**:
    - **Purpose**: "Is the container ready to accept traffic?"
    - **Action if failed**: Removes Pod IP from Service endpoints (stops traffic).
    - **Use Case**: App initializing, loading cache, waiting for DB.
3.  **Startup Probe**:
    - **Purpose**: "Has the container started?"
    - **Action if failed**: Kills and restarts container.
    - **Use Case**: Slow-starting legacy apps. Disables other probes until this passes.

### Example Configuration
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: probe-demo
spec:
  containers:
  - name: my-app
    image: my-app:v1
    # Startup Probe: Give it 30s to start (30 * 1s)
    startupProbe:
      httpGet:
        path: /healthz
        port: 8080
      failureThreshold: 30
      periodSeconds: 1
    # Liveness Probe: Check every 10s if it's alive
    livenessProbe:
      httpGet:
        path: /healthz
        port: 8080
      initialDelaySeconds: 5
      periodSeconds: 10
    # Readiness Probe: Check every 5s if it can take traffic
    readinessProbe:
      exec:
        command:
        - cat
        - /tmp/healthy
      initialDelaySeconds: 5
      periodSeconds: 5
```

## 3. Resource Management & QoS Classes

### Requests vs Limits
- **Requests**: Guaranteed minimum resources. Used for scheduling.
- **Limits**: Hard cap. CPU is throttled; Memory triggers OOMKill.

### Quality of Service (QoS) Classes
Kubernetes assigns a QoS class based on requests/limits. This determines eviction priority when the node is under pressure.

1.  **Guaranteed** (Highest Priority):
    - Every container has CPU & Memory Limits and Requests.
    - Limits must equal Requests.
2.  **Burstable**:
    - At least one container has Requests or Limits.
    - Does not meet Guaranteed criteria.
3.  **BestEffort** (Lowest Priority - First to be evicted):
    - No Requests or Limits set for any container.

```yaml
resources:
  requests:
    memory: "64Mi"
    cpu: "250m"
  limits:
    memory: "128Mi"
    cpu: "500m"
```

## 4. InitContainers & Sidecar Patterns

### InitContainers
Run to completion *before* app containers start.
- **Use Cases**: Database migration, waiting for service availability, permission setup.
- **Behavior**: Run sequentially. If one fails, Pod restarts (unless RestartPolicy=Never).

### Sidecar Pattern
Helper container running alongside the main app container.
- **Use Cases**: Log shipping (Fluentd), Proxy (Envoy/Istio), Config watcher.
- **Native Sidecar Support (K8s 1.28+)**: Use `restartPolicy: Always` in `initContainers` to create native sidecars that start before the main app and terminate after it.

## 5. Pod Security Context & Standards

Control privilege and access control settings for a Pod or Container.

### SecurityContext Example
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: security-context-demo
spec:
  securityContext:
    runAsUser: 1000
    runAsGroup: 3000
    fsGroup: 2000
  containers:
  - name: sec-ctx-demo
    image: busybox
    securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      capabilities:
        drop: ["ALL"]
        add: ["NET_ADMIN"] # Only if absolutely necessary
```

### Pod Security Standards (PSS)
Defines three policies for Pod security:
1.  **Privileged**: Unrestricted. (e.g., CNI plugins, storage drivers).
2.  **Baseline**: Minimally restrictive. Prevents known privilege escalations. Default for most.
3.  **Restricted**: Heavily restricted. Follows hardening best practices (no root, drop capabilities).

### Pod Security Admission (PSA)
Enforce PSS via Namespace labels:
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: my-secure-ns
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/enforce-version: latest
    pod-security.kubernetes.io/warn: restricted
```
