# 08. Deployment Strategies

## 1. Rolling Update (Default)

Updates Pods in a rolling fashion, ensuring zero downtime.

### Configuration
- **maxSurge**: How many pods can be created *above* the desired replica count. (e.g., 25% or 1).
- **maxUnavailable**: How many pods can be unavailable during the update. (e.g., 25% or 0).

### Example: Zero Downtime
```yaml
spec:
  replicas: 4
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1        # Create 1 new pod first (Total 5)
      maxUnavailable: 0  # Don't kill any old pods until new one is Ready
```

## 2. Recreate Strategy

Kills all existing Pods before creating new ones.
- **Pros**: Simple. No version mismatch (v1 and v2 never run together).
- **Cons**: **Downtime** implies service interruption.

```yaml
spec:
  strategy:
    type: Recreate
```

## 3. Blue/Green Deployment

Run two identical environments (Blue = v1, Green = v2). Switch traffic instantly.

### Implementation
1.  **Blue Deployment** (v1) is running. Service points to Blue.
2.  **Green Deployment** (v2) is deployed.
3.  **Test Green** via a temporary service/port-forward.
4.  **Switch Traffic**: Update the main Service selector to point to Green.
5.  **Cleanup**: Delete Blue.

### Pros/Cons
- **Pros**: Instant rollback (switch service back). No mixed versions.
- **Cons**: Requires double resources (2x cost).

## 4. Canary Deployment

Release a new version to a small subset of users (e.g., 10%) before a full rollout.

### Implementation Options
1.  **Native (Replica-based)**:
    - Deployment A (v1): 9 replicas.
    - Deployment B (v2): 1 replica.
    - Service selects both. Traffic split is roughly 90/10.
    - *Downside*: No fine-grained control.

2.  **Ingress Controller (Nginx)**:
    - Use annotations to split traffic by weight or header.
    - See `05_Ingress_and_Service_Mesh.md` for example.

3.  **Service Mesh (Istio)**:
    - Most advanced control (weight, headers, cookies).
    - See `05_Ingress_and_Service_Mesh.md` for example.

4.  **Argo Rollouts**:
    - A custom controller that automates Canary analysis (e.g., check Prometheus metrics) and promotion.
