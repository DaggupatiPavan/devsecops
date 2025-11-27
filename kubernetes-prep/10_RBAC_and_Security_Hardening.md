# 10. RBAC & Security Hardening

## 1. Role-Based Access Control (RBAC)

Controls "Who can do What" in the cluster.

### Core Components
- **Role**: Permissions within a specific Namespace.
- **ClusterRole**: Permissions across the entire Cluster (or for non-namespaced resources like Nodes).
- **RoleBinding**: Grants a Role to a User/Group in a Namespace.
- **ClusterRoleBinding**: Grants a ClusterRole to a User/Group cluster-wide.

### Example: Read-Only User
```yaml
# 1. Define the permissions
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: default
  name: pod-reader
rules:
- apiGroups: [""] # "" indicates the core API group
  resources: ["pods", "pods/log"]
  verbs: ["get", "watch", "list"]

---
# 2. Bind them to a user
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods
  namespace: default
subjects:
- kind: User
  name: jane
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

### ServiceAccounts
Identities for processes running in Pods.
- **AutomountServiceAccountToken**: Should be `false` if not needed.
- **Workload Identity**: Map K8s ServiceAccount to Cloud IAM Role (IRSA in AWS, Workload Identity in GKE) to avoid long-lived keys.

## 2. Authentication (AuthN)

Kubernetes does not have a built-in user database. It relies on external providers.
- **X509 Client Certs**: Common for admins (kubeconfig). Hard to revoke.
- **OIDC (OpenID Connect)**: Standard for humans (Okta, Google, Azure AD).
- **Webhook Token Auth**: Verify bearer tokens via a remote webhook.

## 3. Policy Engines (Policy as Code)

RBAC controls *who* can do it. Policy engines control *what* the object looks like (content validation).

### OPA Gatekeeper
- Uses **Rego** language (complex but powerful).
- **ConstraintTemplate**: Defines the logic.
- **Constraint**: Applies the logic to resources.

### Kyverno
- Kubernetes-native (policies are YAML).
- Easier to learn than OPA.
- Supports **Validation**, **Mutation**, and **Generation**.

**Kyverno Example: Require Labels**
```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-labels
spec:
  validationFailureAction: enforce
  rules:
  - name: check-team
    match:
      resources:
        kinds:
        - Pod
    validate:
      message: "label 'team' is required"
      pattern:
        metadata:
          labels:
            team: "?*"
```

## 4. Secret Management

Kubernetes Secrets are base64 encoded, not encrypted (by default).

### Best Practices
1.  **Encryption at Rest**: Enable `EncryptionConfiguration` in API Server to encrypt secrets in etcd.
2.  **External Secrets Operator (ESO)**:
    - Syncs secrets from external vaults (AWS Secrets Manager, HashiCorp Vault, Azure Key Vault) into K8s Secrets.
    - **Benefit**: Source of truth is the secure vault, not git or etcd.

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: database-credentials
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: aws-store
    kind: SecretStore
  target:
    name: db-secret # Creates this K8s Secret
  data:
  - secretKey: username
    remoteRef:
      key: prod/db
      property: username
```
