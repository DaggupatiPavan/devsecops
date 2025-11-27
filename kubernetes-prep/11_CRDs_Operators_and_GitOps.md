# 11. CRDs, Operators & GitOps

## 1. Custom Resource Definitions (CRDs)

CRDs allow you to extend the Kubernetes API with your own types.

### Structure
- **Kind**: The name of the resource (e.g., `PrometheusRule`).
- **Group**: API group (e.g., `monitoring.coreos.com`).
- **Version**: API version (e.g., `v1`).
- **Schema**: OpenAPI v3 schema defining the fields (validation).

### Example CRD
```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: crontabs.stable.example.com
spec:
  group: stable.example.com
  versions:
  - name: v1
    served: true
    storage: true
    schema:
      openAPIV3Schema:
        type: object
        properties:
          spec:
            type: object
            properties:
              cronSpec:
                type: string
              image:
                type: string
  scope: Namespaced
  names:
    plural: crontabs
    singular: crontab
    kind: CronTab
    shortNames:
    - ct
```

## 2. The Operator Pattern

An Operator is a software extension that uses custom resources to manage applications and their components.
**Operator = Controller + CRD + Domain Knowledge**.

### How it works
1.  **Watch**: The controller watches for changes to the Custom Resource (CR).
2.  **Reconcile Loop**:
    - Compare **Desired State** (CR Spec) vs **Actual State** (Cluster).
    - Take action to fix the drift (e.g., create a Deployment, update a ConfigMap).
3.  **Update Status**: Update the CR Status with the result.

### Tools
- **Kubebuilder / Operator SDK**: Frameworks to scaffold operators in Go/Ansible/Helm.

## 3. GitOps

GitOps is a set of practices to manage infrastructure and application configurations using Git.

### Principles
1.  **Declarative**: Entire system described in Git.
2.  **Versioned & Immutable**: Git is the single source of truth.
3.  **Pulled Automatically**: Software agents (ArgoCD, Flux) pull changes from Git and apply them to the cluster.
4.  **Continuously Reconciled**: Agents detect drift and correct it.

### ArgoCD Architecture
- **Application Controller**: Compares live state (K8s) vs target state (Git).
- **Repo Server**: Clones Git repo and generates manifests (Helm/Kustomize).
- **API Server**: Web UI and CLI.

### App of Apps Pattern
Manage the ArgoCD configuration itself using ArgoCD. A root "Application" points to a folder containing other "Application" manifests.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: guestbook
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/argoproj/argocd-example-apps.git
    targetRevision: HEAD
    path: guestbook
  destination:
    server: https://kubernetes.default.svc
    namespace: guestbook
```
