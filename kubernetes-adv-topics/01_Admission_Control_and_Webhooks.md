# 01. Admission Control & Webhooks

## 1. Admission Controllers

Admission Controllers are plugins that intercept requests to the Kubernetes API server after authentication and authorization, but before the object is persisted to etcd.

### Phases
1.  **Mutating Phase**: Can modify the object (e.g., inject a sidecar, set default storage class).
2.  **Validating Phase**: Can accept or reject the object (e.g., enforce security policies).

### Built-in Controllers
- **LimitRanger**: Enforces default resource limits.
- **ResourceQuota**: Enforces quota constraints.
- **AlwaysPullImages**: Modifies every Pod to force image pull.

## 2. Dynamic Admission Control (Webhooks)

Allows you to extend admission control via HTTP callbacks to external services.

### MutatingAdmissionWebhook
- **Use Case**: Sidecar Injection (Istio/Linkerd), Defaulting fields.
- **Flow**: API Server -> HTTP POST -> Webhook Service -> JSON Patch -> API Server.

### ValidatingAdmissionWebhook
- **Use Case**: Enforcing CRD validation, preventing root users (OPA/Kyverno).
- **Flow**: API Server -> HTTP POST -> Webhook Service -> Allow/Deny -> API Server.

### Example: Validating Webhook Configuration
```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingWebhookConfiguration
metadata:
  name: "pod-policy.example.com"
webhooks:
- name: "pod-policy.example.com"
  rules:
  - apiGroups:   [""]
    apiVersions: ["v1"]
    operations:  ["CREATE"]
    resources:   ["pods"]
    scope:       "Namespaced"
  clientConfig:
    service:
      namespace: "default"
      name: "pod-admission-server"
      path: "/validate-pods"
  admissionReviewVersions: ["v1"]
  sideEffects: None
  timeoutSeconds: 5
```

## 3. API Aggregation Layer

Allows you to extend the Kubernetes API with additional APIs that are not part of the core Kubernetes API.
- **AggregationLayer**: Proxies requests to your custom Extension API Server.
- **Use Case**: Metrics Server (metrics.k8s.io), Service Catalog.
- **APIService Object**: Registers the new API group/version.

```yaml
apiVersion: apiregistration.k8s.io/v1
kind: APIService
metadata:
  name: v1beta1.metrics.k8s.io
spec:
  service:
    name: metrics-server
    namespace: kube-system
  group: metrics.k8s.io
  version: v1beta1
  insecureSkipTLSVerify: true
  groupPriorityMinimum: 100
  versionPriority: 100
```
