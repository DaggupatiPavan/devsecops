# 03a. Container Security Deep Dive: Distroless, NetworkPolicy & Kyverno

You asked for "All". These are the advanced topics that separate Junior from Senior DevSecOps.

## 1. Distroless Images (No Shell, No Pain)
**Concept**: Most Docker images (like `ubuntu` or `alpine`) contain a full OS (bash, ls, curl, apt). Hackers love these tools.
**Distroless**: Google created images that contain *only* your app and its runtime. Nothing else.

### Comparison
*   **Standard**: `FROM python:3.9-slim` (Has bash, apt, curl). Size: ~120MB.
*   **Distroless**: `FROM gcr.io/distroless/python3` (No bash, no apt). Size: ~50MB.

### How to Debug? (The Catch)
Since there is no shell, you cannot do `docker exec -it my-container bash`.
*   **Solution**: Use `kubectl debug` (Ephemeral Containers).
    ```bash
    kubectl debug -it my-pod --image=busybox --target=my-container
    ```
    This attaches a *sidecar* container with tools to your running pod.

---

## 2. Kubernetes Network Policies (The Firewall)
**Problem**: By default, if a hacker compromises your Frontend, they can talk to your Database directly.
**Solution**: NetworkPolicy.

### The "Deny All" Policy (Start Here)
Put this in every namespace. It blocks ALL traffic.
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: default
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
```

### The "Allow DNS" Policy (Essential)
If you block Egress, your pods can't look up DNS (like `google.com` or `db-service`). You must allow UDP port 53.
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns
  namespace: default
spec:
  podSelector: {}
  policyTypes:
  - Egress
  egress:
  - to:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: kube-system
    ports:
    - protocol: UDP
      port: 53
```

---

## 3. Admission Controllers (Kyverno)
**Problem**: Developers keep deploying containers running as `root`.
**Solution**: Kyverno (Policy as Code). It watches the API Server and rejects bad YAMLs.

### Install Kyverno
```bash
kubectl create -f https://github.com/kyverno/kyverno/releases/download/v1.10.0/install.yaml
```

### The Policy (Block Root)
Save as `block-root.yaml` and apply.
```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-non-root
spec:
  validationFailureAction: Enforce  # Block it! (Use 'Audit' to just log)
  rules:
  - name: check-runasnonroot
    match:
      resources:
        kinds:
        - Pod
    validate:
      message: "Running as root is forbidden!"
      pattern:
        spec:
          securityContext:
            runAsNonRoot: true
```

### Test It
Try to deploy a bad pod.
```bash
kubectl run bad-pod --image=nginx
# Error: admission webhook "validate.kyverno.svc-fail" denied the request: Running as root is forbidden!
```

---

## 4. Image Signing (Cosign)
**Problem**: How do you know the image `my-app:v1` wasn't modified by a hacker on Docker Hub?
**Solution**: Sign it with Cosign.

### Workflow
1.  **Generate Key**:
    ```bash
    cosign generate-key-pair
    # Creates cosign.key (private) and cosign.pub (public)
    ```
2.  **Sign Image**:
    ```bash
    cosign sign --key cosign.key user/demo:v1
    # Pushes a signature to the registry
    ```
3.  **Verify Image**:
    ```bash
    cosign verify --key cosign.pub user/demo:v1
    ```
4.  **Enforce in K8s**: Use Kyverno to check the signature before pulling.
