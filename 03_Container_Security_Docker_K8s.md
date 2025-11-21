# 03. Container Security: Docker & Kubernetes

Containers changed the world. They also changed the attack surface.

## 1. Docker Security: The Foundation

### The "Golden Image" Strategy
Don't let developers pull random images from Docker Hub.
*   **Base Image**: Create a curated, hardened base image (e.g., `mycompany/alpine-base:v1`).
*   **Scan it**: Scan this base image nightly.
*   **Enforce**: Pipelines can only build `FROM mycompany/alpine-base`.

### Dockerfile Best Practices (Bad vs. Good)

#### ❌ The BAD Dockerfile
```dockerfile
FROM ubuntu:latest             # 1. Huge attack surface, unknown version
RUN apt-get update && \
    apt-get install -y python3 # 2. Installs unnecessary tools
COPY . /app
CMD ["python3", "/app/main.py"] # 3. Runs as ROOT!
```

#### ✅ The GOOD Dockerfile
```dockerfile
# 1. Use specific, minimal tag (Alpine or Distroless)
FROM python:3.9-alpine

# 2. Create a non-root user
RUN addgroup -S appgroup && adduser -S appuser -G appgroup

# 3. Set working directory
WORKDIR /app

# 4. Install dependencies
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# 5. Copy code
COPY . .

# 6. Switch to non-root user
USER appuser

# 7. Healthcheck (Critical for K8s)
HEALTHCHECK --interval=30s --timeout=3s \
  CMD curl -f http://localhost:8000/ || exit 1

CMD ["python3", "main.py"]
```

### Docker Content Trust (DCT)
Ensures you are running the image you think you are.
*   **Enable**: `export DOCKER_CONTENT_TRUST=1`
*   **Effect**: Docker will refuse to pull images that haven't been digitally signed by the publisher.

---

## 2. Kubernetes Security: The Orchestrator

### A. Authentication & Authorization (RBAC)
*   **Authentication**: "Who are you?" (Handled by AWS IAM, OIDC, Certificates).
*   **Authorization**: "What can you do?" (Handled by K8s RBAC).

#### RBAC Example: Read-Only Developer
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: dev
  name: pod-reader
rules:
- apiGroups: [""] # Core API group
  resources: ["pods", "pods/log"]
  verbs: ["get", "watch", "list"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods
  namespace: dev
subjects:
- kind: User
  name: jane
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

### B. Network Policies (The Firewall)
By default, K8s is a flat network. Any pod can talk to any pod (even across namespaces).

#### Policy 1: Default Deny All (Best Practice)
Apply this to every namespace. It forces you to whitelist traffic.
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

#### Policy 2: Allow Frontend to Backend
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-backend
spec:
  podSelector:
    matchLabels:
      app: backend
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend
    ports:
    - protocol: TCP
      port: 5432
```

### C. Pod Security Standards (PSS) & Admission Controllers
How do we prevent someone from deploying a "Bad" Dockerfile?
*   **Admission Controller**: A gatekeeper that intercepts requests to the API Server.
*   **Tools**: **Kyverno** or **OPA Gatekeeper**.

#### Kyverno Example: Block Root Containers
```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-non-root
spec:
  validationFailureAction: Enforce
  rules:
  - name: check-runasnonroot
    match:
      resources:
        kinds:
        - Pod
    validate:
      message: "Running as root is not allowed. Set runAsNonRoot: true."
      pattern:
        spec:
          securityContext:
            runAsNonRoot: true
```

---

## 3. Container Runtime Security (Runtime Defense)
What if the attacker gets inside the container?
*   **Falco**: The "Security Camera" for K8s. It detects abnormal behavior.
    *   *Rule*: "Terminal shell in a container".
    *   *Alert*: "Notice: A shell was spawned in pod nginx-789 by user root."

---

## 4. Troubleshooting "CrashLoopBackOff"
The most common K8s error.
1.  **Check Logs**: `kubectl logs <pod-name>`
    *   *Common*: Missing env var, app crashed.
2.  **Check Events**: `kubectl describe pod <pod-name>`
    *   *Common*: OOMKilled (Out of Memory), Liveness probe failed.
3.  **Check Previous Logs**: `kubectl logs <pod-name> --previous`
    *   See why it died the last time.

---

## 5. Interview Preparation

### Q1: How do you handle secrets in Kubernetes?
*   **Answer**:
    *   "Native K8s Secrets are just base64 encoded, which is not secure enough for GitOps.
    *   I use **External Secrets Operator (ESO)**.
    *   ESO fetches secrets from **AWS Secrets Manager** or **HashiCorp Vault** and injects them into the cluster as K8s Secrets at runtime.
    *   This way, no secrets are ever stored in Git or etcd."

### Q2: What is the difference between a Liveness and Readiness Probe?
*   **Answer**:
    *   **Liveness**: "Are you alive?" If fails, K8s **restarts** the pod. (Fixes deadlocks).
    *   **Readiness**: "Are you ready to take traffic?" If fails, K8s **removes IP from Service**. (Prevents traffic to starting apps).

### Q3: Explain "Container Breakout".
*   **Answer**:
    *   When an attacker escapes the container isolation and gains access to the Host OS.
    *   **Prevention**:
        1.  Don't run as root.
        2.  Use `gVisor` or `Kata Containers` for stronger isolation.
        3.  Keep the Kernel patched.

---

## 6. Hands-On Lab
1.  **Docker**: Write a Dockerfile for a simple Python app.
2.  **Scan**: Run `trivy image myapp`. Fix the CVEs.
3.  **K8s**: Deploy it to Minikube.
4.  **NetworkPolicy**: Apply a "Deny All" policy and watch the app fail. Then apply an "Allow" policy to fix it.
5.  **Kyverno**: Install Kyverno and apply a policy that blocks pods without a `app` label. Try to deploy a pod without a label.
